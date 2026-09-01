# 비밀 파일, 관리 작업 잠금, 자격증명 회전

이 문서는 비밀값이 호스트 파일에서 일회성 bootstrap으로 전달되고, 장기 실행 컨테이너에는 남지 않도록 만든 경계를 다룬다. 백지 구현에서는 보안 조건을 생략한 "동작하는 코드"보다, 공격 가능한 파일시스템 상태와 부분 실패를 명시적으로 거부하는 코드를 우선한다.

---

<a id="s-03"></a>
## S-03 · [Thread 02 / `refactor(secrets): 비밀 파일 로딩 경계 공통화`] 안전한 비밀 파일 읽기

### 면접 질문

`Path.read_text()` 대신 `lstat`·`os.open(O_NOFOLLOW)`·`fstat`을 사용해 비밀 파일을 읽은 이유를 설명해 보세요. 경로를 열기 전 검사만으로 충분하지 않은 이유와, 열고 난 파일 디스크립터에서 다시 확인해야 할 항목을 말해 보세요.

꼬리 질문:

- 상위 디렉터리가 현재 사용자 소유의 0700이 아니면 왜 파일 자체가 0600이어도 거부해야 합니까?
- 심볼릭 링크만 막으면 하드링크 공격도 막을 수 있습니까?
- `stat(path)`와 `fstat(fd)` 사이의 차이는 무엇입니까?
- FIFO나 장치 파일을 일반 파일처럼 읽으면 어떤 장애가 생길 수 있으며 `O_NONBLOCK`은 어떤 역할을 합니까?
- 파일 크기 제한과 비밀번호 형식 제한은 각각 보안과 운영 측면에서 어떤 이점이 있습니까?
- 서로 다른 secret 이름이 같은 canonical path를 가리키는 것을 왜 거부해야 합니까?

### 30초 모범 답변

경로 검사 뒤 open 사이에는 대상이 바뀔 수 있으므로, 보안 판단은 실제로 연 파일 디스크립터를 기준으로 해야 합니다. 먼저 상위 디렉터리가 현재 사용자 소유의 비공개 일반 디렉터리인지 `lstat`으로 확인하고, 파일은 `O_NOFOLLOW`와 `O_NONBLOCK`으로 엽니다. 이어 `fstat`으로 일반 파일, 현재 사용자 소유, 정확한 private mode, 링크 수 1, 제한된 크기를 확인한 뒤 값을 읽고 형식을 검증합니다. 모든 경로에서 descriptor를 닫고, secret 이름들이 같은 실제 파일을 공유하면 회전과 권한 경계가 모호해지므로 canonical 중복도 거부합니다.

### 답변 핵심 키워드

TOCTOU · lstat · O_NOFOLLOW · O_NONBLOCK · fstat · regular file · owner UID · 0600 · link count 1 · bounded read · canonical path uniqueness · resource cleanup

### 백지 구현

#### 구현 목표

현재 사용자만 접근할 수 있는 디렉터리 아래의 일반 파일에서 짧은 비밀 문자열 하나를 안전하게 읽는 함수를 구현한다.

#### 인터페이스

```python
from pathlib import Path

class SecretFileError(RuntimeError):
    pass


def read_private_secret(
    path: Path,
    *,
    max_bytes: int = 1024,
    min_length: int = 24,
    max_length: int = 128,
) -> str:
    # 직접 구현
    ...
```

#### 입력과 출력

- `path`: 읽을 비밀 파일 경로
- 성공 시 검증된 문자열 하나를 반환한다.
- 구조·권한·소유권·크기·형식이 계약과 다르면 `SecretFileError`를 발생시킨다.

#### 반드시 만족해야 할 조건

- `path`의 바로 위 디렉터리는 symlink가 아닌 일반 디렉터리이며 현재 사용자 소유여야 한다.
- 상위 디렉터리의 group/other 권한 비트가 하나라도 설정되어 있으면 거부한다.
- 파일은 symlink를 따라가지 않고 non-blocking 방식으로 연다.
- 실제로 연 descriptor의 대상이 일반 파일인지 `fstat`으로 확인한다.
- 파일 소유자는 현재 사용자여야 하고, 권한은 정확히 소유자 읽기·쓰기만 허용해야 한다.
- 하드링크 수가 1이 아니면 거부한다.
- 최대 크기보다 큰 파일을 전체 메모리에 읽지 않고 거부할 수 있어야 한다.
- 반환 문자열은 길이 범위와 허용 문자 집합을 만족해야 한다.
- 성공·실패 모든 경로에서 descriptor가 닫혀야 한다.
- 하위 예외를 원인으로 보존하되 오류 메시지에는 비밀값을 넣지 않는다.

#### 경계 조건

- 부모 디렉터리 누락, 파일 누락
- 부모가 symlink·파일·다른 사용자 소유·0750·0701인 경우
- 파일이 symlink, dangling symlink, FIFO, socket, device, directory인 경우
- 파일 모드 0400, 0640, 0660, 0600
- 하드링크 수 2 이상
- 정확히 `max_bytes`, `max_bytes + 1` 크기
- 빈 값, 최소 길이 바로 아래·이상, 최대 길이·초과
- 허용하지 않은 공백·제어 문자·NUL이 포함된 경우
- open 뒤 path가 교체되어도 descriptor 검증 결과를 사용해야 하는 경우

#### 실패 조건과 제약

- `Path.resolve()`와 `Path.read_text()`만으로 구현하지 않는다.
- 검증 전에 파일 전체를 읽지 않는다.
- 파일 내용을 로그·예외·repr에 포함하지 않는다.
- 권한이 더 제한적인 0400을 허용할지는 인터페이스 계약과 일치시켜야 한다. 이 문제에서는 정확한 0600을 요구한다.
- 플랫폼에서 `O_NOFOLLOW`가 제공되지 않는 경우를 조용히 안전하다고 가정하지 말고 계약에 맞게 처리한다.

### 구현 후 자가 검증

- [ ] 0700 부모 아래 현재 사용자 소유 0600 일반 파일의 정상 값을 읽는다.
- [ ] symlink와 dangling symlink를 모두 거부한다.
- [ ] FIFO를 전달해도 무기한 block하지 않는다.
- [ ] 부모 디렉터리의 group/other 권한을 각각 거부한다.
- [ ] 파일 소유권, 모드, 링크 수 위반을 각각 구분해 거부한다.
- [ ] 최대 크기보다 1바이트 큰 파일을 제한된 read로 감지한다.
- [ ] 최소·최대 길이 경계와 허용 문자 집합을 검증한다.
- [ ] open 이후 경로 교체를 모의해도 열린 descriptor의 메타데이터를 기준으로 판단한다.
- [ ] 오류 문자열과 로그에 비밀값이 포함되지 않는다.
- [ ] 모든 예외 경로에서 파일 디스크립터가 닫힌다.
- [ ] 시간·공간 복잡도는 최대 입력 크기에 대해 `O(n)`, 추가 공간 `O(n)` 이하이며 `n`은 상한으로 제한된다.

### 구현 후 설명할 것

1. 경로 기반 선행 검사와 descriptor 기반 사후 검증을 함께 둔 이유
2. symlink와 하드링크가 만드는 위협의 차이
3. FIFO에 대한 `O_NONBLOCK`과 크기 제한의 역할
4. 정확한 0600·현재 UID·private parent를 동시에 요구한 이유
5. canonical path 중복을 회전·백업 도구에서 거부해야 하는 이유

### 원본 확인 위치

- Thread 02
- 커밋: `refactor(secrets): 비밀 파일 로딩 경계 공통화`
- 파일: `tools/stack_runtime.py`
- 함수·상수: `_private_directory`, `read_private_secret`, `secret_source_paths`, `load_secret_values`, `SECRET_FILENAMES`, `PASSWORD_PATTERN`, `NOFOLLOW`, `NONBLOCK`
- 관련 Thread: 06, 09, 12

---

<a id="a-03"></a>
## A-03 · [Thread 02 / `refactor(runtime): 프로젝트 관리 작업 잠금 공통화`] 프로세스 간 직렬화

### 면접 질문

백업, 복원, 시작, 자격증명 회전이 모두 같은 프로젝트 잠금을 공유해야 하는 이유를 설명해 보세요. 잠금 파일 경로를 프로젝트 이름 그대로 사용하지 않고 해시한 이유와, 잠금 디렉터리의 소유권·권한을 검사한 이유도 말해 보세요.

꼬리 질문:

- 스레드 잠금만으로 충분하지 않은 이유는 무엇입니까?
- 잠금 파일 자체가 symlink라면 어떤 공격이 가능합니까?
- blocking lock과 non-blocking lock 중 운영 명령에는 무엇이 더 적합합니까?
- 프로세스가 비정상 종료되면 advisory file lock은 어떻게 해제됩니까?
- 같은 프로젝트의 읽기 작업도 모두 배타 잠금을 잡아야 합니까?
- 잠금을 획득한 뒤 오래 걸리는 subprocess가 hang하면 어떤 문제가 생기며 어떻게 제한합니까?

### 30초 모범 답변

이 작업들은 DB·볼륨·설정·비밀 파일 같은 동일한 프로젝트 상태를 여러 단계에 걸쳐 변경하므로 서로 겹치면 각각의 rollback 가정이 깨집니다. 따라서 프로세스 안의 mutex가 아니라 여러 CLI 프로세스가 공유하는 advisory file lock이 필요합니다. 잠금 디렉터리는 사용자별 0700 디렉터리로 만들고 소유권과 종류를 확인하며, 프로젝트 이름은 고정 길이 해시로 바꿔 경로 조작과 이름 길이 문제를 피합니다. 잠금 파일도 안전하게 열고 `flock`을 유지한 descriptor의 수명 동안만 임계 구역을 실행합니다. 각 외부 작업에는 별도 timeout을 둬 잠금 독점을 제한합니다.

### 답변 핵심 키워드

cross-process lock · advisory flock · per-user private directory · hashed lock name · O_NOFOLLOW · descriptor lifetime · timeout · critical section · fail fast

### 백지 구현

#### 구현 목표

프로젝트 이름별 비차단 프로세스 잠금 context manager를 구현한다. 같은 사용자의 같은 프로젝트는 동시에 하나만 진입할 수 있고, 다른 프로젝트는 독립적으로 실행할 수 있어야 한다.

#### 인터페이스

```python
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path

class OperationBusyError(RuntimeError):
    pass

@contextmanager
def project_operation_lock(
    project_name: str,
    *,
    root: Path = Path("/tmp"),
) -> Iterator[None]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 유효한 프로젝트 이름을 입력받는다.
- 잠금 획득 후 context body를 실행하고, 종료 시 자동 해제한다.
- 이미 같은 프로젝트 잠금이 잡혀 있으면 기다리지 않고 `OperationBusyError`를 발생시킨다.

#### 반드시 만족해야 할 조건

- 잠금 디렉터리는 현재 UID별로 분리되고 현재 사용자 소유의 private 일반 디렉터리여야 한다.
- 기존 경로가 symlink·파일·다른 사용자 소유·group/other 접근 가능이면 거부한다.
- 잠금 파일 이름은 프로젝트 이름의 안정적인 cryptographic hash로 만든다.
- lock file은 symlink를 따라가지 않도록 연다.
- descriptor를 연 상태에서 non-blocking exclusive `flock`을 시도한다.
- context body가 정상 반환하거나 예외를 던져도 descriptor를 닫고 잠금을 해제한다.
- 다른 프로젝트 이름은 서로 다른 lock file을 사용한다.
- 오류 메시지에는 lock path와 project 맥락은 포함할 수 있지만 민감 데이터는 포함하지 않는다.

#### 경계 조건

- 빈 프로젝트 이름과 매우 긴 이름
- 같은 프로젝트의 재진입
- 두 프로세스가 동시에 디렉터리를 만들려는 경쟁
- 기존 잠금 디렉터리가 안전하지 않은 경우
- lock file이 dangling symlink인 경우
- context body에서 예외가 발생하는 경우
- 프로세스가 descriptor를 상속한 채 child를 실행하는 경우

#### 실패 조건과 제약

- broad global lock 하나로 모든 프로젝트를 직렬화하지 않는다.
- lock file 존재 여부만으로 잠금 상태를 판단하지 않는다.
- context 종료 전에 descriptor를 닫지 않는다.
- 무한 대기하지 않는다.
- 사용자가 소유하지 않은 경로의 권한을 자동 수정하지 않는다.

### 구현 후 자가 검증

- [ ] 같은 프로젝트의 두 번째 비차단 획득은 실패한다.
- [ ] 서로 다른 프로젝트는 동시에 획득할 수 있다.
- [ ] context body 예외 뒤 다시 잠금을 획득할 수 있다.
- [ ] 안전하지 않은 lock directory와 symlink lock file을 거부한다.
- [ ] 두 프로세스의 디렉터리 생성 경쟁을 안전하게 처리한다.
- [ ] lock file 이름에 원래 프로젝트 이름이나 경로 구분자가 직접 들어가지 않는다.
- [ ] 잠금 유지 중 descriptor가 열린 상태이며 종료 후 닫힌다.
- [ ] child process로 descriptor가 불필요하게 상속되지 않도록 고려했다.
- [ ] 실패가 busy인지 구조·권한 오류인지 구분된다.

### 구현 후 설명할 것

1. advisory lock이 동작하려면 모든 관리 도구가 같은 계약을 따라야 하는 이유
2. lock file 존재와 실제 잠금 소유가 다른 이유
3. 비차단 획득을 선택한 운영 UX상의 이유
4. 프로젝트 이름 해시가 해결하는 경로 조작·길이·문자 문제
5. 잠금과 subprocess timeout을 함께 설계해야 하는 이유

### 원본 확인 위치

- Thread 02
- 커밋: `refactor(runtime): 프로젝트 관리 작업 잠금 공통화`
- 파일: `tools/stack_runtime.py`
- 함수: `project_operation_lock`
- 관련 위치: `tools/start_stack.py`, `tools/stack_backup.py`, `tools/rotate_secrets.py`
- 관련 Thread: 07, 08, 09

---

<a id="a-04"></a>
## A-04 · [Thread 06 / `feat(secrets): 런타임 비밀 노출 경계 검사`] 비밀값 lifecycle

### 면접 질문

비밀 파일을 0600으로 안전하게 보관하는 것만으로 충분하지 않고, 실행 중인 컨테이너를 inspect해 비밀 마운트와 환경 변수 노출을 검사한 이유를 설명해 보세요. bootstrap 컨테이너와 장기 실행 컨테이너의 비밀 lifecycle을 어떻게 나눴습니까?

꼬리 질문:

- 초기화에 비밀번호가 필요한데 runtime에는 필요하지 않다면 어떤 전달 경로가 적절합니까?
- 환경 변수 이름만 검사하면 충분합니까? 임의 이름에 실제 secret 값이 들어간 경우는 어떻게 찾습니까?
- `/run/secrets` 디렉터리 자체와 그 하위 mount를 모두 검사해야 하는 이유는 무엇입니까?
- `docker inspect` 결과가 예상 구조가 아니거나 컨테이너를 하나로 식별하지 못하면 왜 실패해야 합니까?
- DB client 비밀번호를 command argument로 넘기면 inspect 외에 어떤 경로로 노출될 수 있습니까?

### 30초 모범 답변

비밀은 초기화 순간에만 필요하므로 호스트의 검증된 파일에서 읽어 one-off bootstrap 프로세스의 표준 입력으로 전달하고, 컨테이너 안에서는 0600 임시 option file로만 사용한 뒤 trap으로 삭제합니다. 장기 실행 서비스에는 secret mount나 비밀번호 환경 변수를 남기지 않습니다. 이 경계를 추측하지 않고 각 서비스의 inspect 결과에서 `/run/secrets` mount, 금지된 환경 변수 이름, 실제 secret 값의 포함 여부를 검사합니다. 컨테이너를 정확히 하나로 식별하지 못하거나 inspect 구조가 예상과 다르면 검증을 건너뛰지 않고 실패합니다.

### 답변 핵심 키워드

secret lifetime · one-off bootstrap · stdin · temporary option file · trap cleanup · no runtime mount · no env secret · inspect verification · fail closed · argv exposure

### 백지 구현

#### 구현 목표

이미 파싱된 컨테이너 inspect 자료에서 런타임 비밀 경계 위반을 모두 찾아 반환하는 순수 함수를 구현한다.

#### 인터페이스

```python
from collections.abc import Mapping, Sequence


def runtime_secret_violations(
    containers: Mapping[str, Mapping[str, object]],
    *,
    forbidden_env_names: frozenset[str],
    secret_values: frozenset[str],
) -> list[str]:
    # 직접 구현
    ...
```

#### 입력과 출력

- key는 서비스 이름, value는 단일 컨테이너의 정규화된 inspect 객체다.
- 위반이 없으면 빈 리스트, 있으면 서비스와 위반 종류를 나타내는 문자열 목록을 반환한다.

#### 반드시 만족해야 할 조건

- 각 컨테이너의 mount destination이 `/run/secrets` 또는 그 하위이면 위반이다.
- 환경 변수는 `NAME=value` 구조로 해석하고 금지 이름을 검사한다.
- 금지 이름이 아니어도 환경 변수 값, command, args, mount source·destination 등 검사 대상 문자열에 실제 secret 값이 포함되면 위반이다.
- 빈 secret 값은 검색 대상에서 제외한다.
- inspect의 필수 필드가 예상 타입이 아니면 조용히 건너뛰지 않고 구조 오류로 보고한다.
- 위반 메시지에는 실제 secret 값을 포함하지 않는다.
- 결과 순서는 서비스 이름과 위반 종류 기준으로 결정적이어야 한다.

#### 경계 조건

- mount destination `/run/secrets`, `/run/secrets/x`, `/run/secrets-old`
- 환경 변수 항목에 `=`가 없거나 값이 빈 경우
- secret 값이 다른 문자열의 부분 문자열로 들어간 경우
- 같은 secret이 여러 위치에서 발견되는 경우
- inspect 필드가 `None`, list 대신 dict, 문자열 대신 숫자인 경우
- 서비스가 빠졌거나 예상보다 추가된 경우

#### 실패 조건과 제약

- 실제 Docker 명령을 이 함수 안에서 호출하지 않는다.
- 위반을 발견한 첫 지점에서 중단하지 않고 가능한 범위를 모두 보고한다.
- secret 문자열을 오류 메시지, 디버그 출력, 정렬 key에 사용하지 않는다.
- `/run/secrets-old`처럼 단순 prefix가 같지만 하위 경로가 아닌 값은 오탐하지 않는다.

### 구현 후 자가 검증

- [ ] `/run/secrets`와 `/run/secrets/x` mount를 잡고 `/run/secrets-old`는 허용한다.
- [ ] 금지된 환경 변수 이름과 임의 이름에 들어간 실제 secret 값을 모두 잡는다.
- [ ] command·args에 들어간 secret 값을 잡는다.
- [ ] 빈 secret 값 때문에 모든 문자열이 위반으로 판정되지 않는다.
- [ ] 잘못된 inspect 구조를 명시적 오류로 보고한다.
- [ ] 동일 secret의 여러 노출 지점을 구분해 보고한다.
- [ ] 출력에 실제 secret 값이 나타나지 않는다.
- [ ] 입력 mapping 순서가 달라도 결과가 같다.
- [ ] 전체 문자열 길이에 선형인 범위에서 동작한다.

### 구현 후 설명할 것

1. 저장 시 보안과 runtime 비노출이 별도 요구사항인 이유
2. secret 이름 검사와 실제 값 검사 둘 다 필요한 이유
3. one-off bootstrap stdin과 장기 실행 mount의 trade-off
4. 임시 option file을 사용하고 command argument를 피한 이유
5. inspect 구조 오류를 fail-open으로 처리하지 않은 이유

### 원본 확인 위치

- Thread 06
- 커밋: `feat(secrets): 런타임 비밀 노출 경계 검사`
- 파일: `tools/rotate_secrets.py`, `tests/runtime_stack.py`
- 함수: `verify_runtime_secret_boundary`, `RuntimeStack.assert_runtime_secret_boundary`
- 관련 위치: `tools/start_stack.py`, MariaDB·WordPress `docker-entrypoint.sh`, `srcs/docker-compose.yml`의 `x-secret-files`
- 관련 Thread: 02, 04, 05, 09, 12

---

<a id="s-04"></a>
## S-04 · [Thread 09 / `feat(secrets): 스택 자격증명 회전 절차 연결` + `feat(secrets): 회전 실패 시 기존 자격증명 복구`] 보상 트랜잭션

### 면접 질문

MariaDB root·애플리케이션 계정, WordPress 설정, WordPress 사용자, 호스트 secret 파일을 한 번에 회전할 때 왜 일반적인 단일 DB 트랜잭션으로 처리할 수 없는지 설명해 보세요. 단계 순서와 실패 후 보상 전략을 설계해 보세요.

꼬리 질문:

- root 비밀번호 변경 직후 실패하면 현재 비밀번호와 새 비밀번호 중 무엇으로 DB에 접속해야 합니까?
- 호스트 secret 파일을 먼저 바꾸는 것과 마지막에 바꾸는 것의 trade-off는 무엇입니까?
- rollback 자체의 일부 단계가 실패하면 결과를 어떻게 보고해야 합니까?
- rollback 중 SIGTERM이 한 번 더 오면 즉시 중단해야 합니까?
- 새 secret 파일 세트가 기존 값과 일부 같거나 파일 경로를 공유하면 왜 거부해야 합니까?
- 최종 검증에서 "새 비밀번호가 동작한다" 외에 무엇을 확인해야 합니까?

### 30초 모범 답변

서로 다른 저장소와 프로세스에 걸친 변경이라 원자 커밋이 없으므로 saga에 가까운 보상 절차가 필요합니다. 프로젝트 잠금을 잡고 새 secret 세트의 소유권·경로 독립성·값 차이를 검증한 뒤, 서비스 쓰기를 제한하고 WordPress 사용자와 설정, DB 애플리케이션 계정, root 계정, 호스트 파일을 계획된 순서로 바꿉니다. 각 단계 뒤 검증하고 최종 재생성과 종단 쓰기·읽기, 이전 비밀번호 거부까지 확인합니다. 실패하면 현재·새 root 후보를 실제 인증으로 탐색해 기존 상태로 보상하며, rollback 중 추가 종료 신호는 기록만 하고 복구가 끝날 때까지 지연합니다. 원래 오류와 모든 보상 오류를 함께 남깁니다.

### 답변 핵심 키워드

saga · compensating transaction · project lock · staged credentials · current/new root discovery · atomic host file replace · verify each step · reject old credentials · signal deferral · error aggregation · retryability

### 백지 구현

#### 구현 목표

원자 트랜잭션이 없는 여러 자격증명 저장소를 순서대로 변경하고, 실패 시 적용 완료된 단계만 역순으로 보상하는 coordinator를 구현한다. 실제 DB나 파일 I/O는 주입된 step callback으로 대체한다.

#### 인터페이스

```python
from dataclasses import dataclass
from collections.abc import Callable, Sequence

@dataclass(frozen=True)
class RotationStep:
    name: str
    apply: Callable[[], None]
    verify_applied: Callable[[], None]
    compensate: Callable[[], None]
    verify_compensated: Callable[[], None]

class RotationFailed(RuntimeError):
    pass


def execute_rotation(
    steps: Sequence[RotationStep],
    *,
    final_verify: Callable[[], None],
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- 순서가 있는 회전 단계와 최종 검증 함수를 받는다.
- 성공하면 반환값이 없다.
- 적용 또는 검증이 실패하면 완료된 단계의 보상을 시도한 뒤 `RotationFailed`를 발생시킨다.

#### 반드시 만족해야 할 조건

- 각 단계는 `apply` 성공 후 `verify_applied`까지 성공해야 완료된 것으로 기록한다.
- 실패 시 완료된 단계만 역순으로 보상한다.
- 현재 단계의 `apply`가 side effect 후 예외를 낼 수 있다는 점을 고려해, 호출자가 그 단계를 보상 후보로 표시할 방법을 계약에 명확히 반영한다. 필요하면 데이터 모델을 조정해도 된다.
- 한 보상 단계가 실패해도 나머지 보상을 계속 시도한다.
- 보상 뒤 `verify_compensated` 실패도 별도 오류로 수집한다.
- 원래 적용 오류와 보상 오류 목록을 모두 보존한다.
- 최종 검증 실패도 전체 보상을 유발한다.
- step 이름 중복을 사전에 거부한다.
- 성공 경로에서는 compensate를 호출하지 않는다.

#### 경계 조건

- 빈 단계 목록
- 첫 단계 apply 실패
- apply 성공 후 verify 실패
- 마지막 단계까지 성공했지만 final verify 실패
- 여러 compensate와 verify_compensated가 연속 실패
- callback이 일반 예외가 아닌 사용자 정의 예외를 던지는 경우
- 같은 이름의 step 두 개
- 보상 callback이 재호출되어도 안전해야 하는 경우

#### 실패 조건과 제약

- 보상 오류 때문에 원래 오류가 사라지면 안 된다.
- 완료되지 않은 미래 단계를 보상하지 않는다.
- "모든 apply를 실행한 뒤 한 번 검증"하는 구조로 만들지 않는다.
- callback 실행 결과나 오류 문자열에 secret 값을 포함한다고 가정하지 말고, coordinator도 인자를 repr로 출력하지 않는다.
- coordinator는 lock·signal handler 자체를 구현하지 않아도 되지만, 그 바깥 경계가 필요한 이유를 설명해야 한다.

### 구현 후 자가 검증

- [ ] 모든 단계가 성공하면 각 apply·verify가 순서대로 한 번씩 호출되고 보상은 없다.
- [ ] 중간 단계 실패 시 그 이전 완료 단계만 역순으로 보상된다.
- [ ] apply 후 verify 실패한 단계가 실제로 변경되었을 가능성을 계약대로 처리한다.
- [ ] final verify 실패가 전체 보상을 시작한다.
- [ ] 하나의 보상 실패 뒤에도 나머지 보상이 계속된다.
- [ ] 원래 오류와 보상·보상 검증 오류가 모두 최종 예외에서 식별된다.
- [ ] step 이름 중복을 side effect 전에 거부한다.
- [ ] 성공·실패 경로의 callback 호출 순서가 결정적이다.
- [ ] retry 시 이미 보상된 상태에서도 callback 계약이 깨지지 않는지 확인한다.
- [ ] secret 값이 예외 메시지에 들어가지 않는다.

### 구현 후 설명할 것

1. 이 문제를 ACID 트랜잭션이 아니라 saga/보상 트랜잭션으로 본 이유
2. 단계별 검증과 최종 종단 검증을 모두 둔 이유
3. root 자격증명을 현재·새 후보 중 실제 인증으로 찾는 이유
4. rollback 중 추가 종료 신호를 지연해야 하는 이유
5. 원래 오류와 보상 오류를 함께 보존하면서 재시도 가능성을 유지하는 방법

### 원본 확인 위치

- Thread 09
- 커밋: `feat(secrets): WordPress 설정과 사용자 비밀번호 교체`
- 커밋: `feat(secrets): 회전 실패 시 기존 자격증명 복구`
- 커밋: `feat(secrets): 스택 자격증명 회전 절차 연결`
- 커밋: `test(secrets): 회전 롤백과 재시도 검증`
- 파일: `tools/rotate_secrets.py`, `tests/runtime_stack.py`
- 함수: `sql_literal`, `root_sql`, `alter_database_passwords`, `set_wordpress_db_config`, `set_wordpress_user`, `atomic_secret_write`, `verify_rotation`, `find_root_password`, `rollback_rotation`, `_rotate`, `rotate`, `RuntimeStack.verify_secret_rotation`
- 관련 Thread: 02, 05, 06, 07, 13
