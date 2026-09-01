# 비공개 진단, 격리 검증, CI 안전장치

이 문서는 장애 증거를 남기면서도 비밀값을 노출하지 않는 방법, 중단·신호·자원 누수를 결정적으로 재현하는 런타임 하네스, 소유한 자원만 정리하는 정책, 그리고 코드·workflow의 운영 계약을 구조적으로 검사하는 방법을 다룬다. 진단과 테스트는 실패를 관찰하기 위해 더 많은 정보를 다루므로, 정상 경로보다 오히려 강한 보안·수명주기 경계가 필요하다.

---

<a id="s-10"></a>
## S-10 · [Thread 12 / `feat(diagnostics): Compose 비밀값과 민감 항목 마스킹`] fail-closed 진단 마스킹

### 면접 질문

장애 진단에서 비밀번호 문자열만 정규식으로 가리면 충분하지 않은 이유는 무엇입니까? 이 프로젝트가 렌더링된 Compose 모델에서 비밀 파일의 원문 경로·해석된 경로·실제 값을 모두 수집하고, 그중 하나라도 안전하게 읽지 못하면 진단 자체를 중단한 이유를 설명해 보세요.

꼬리 질문:

- 실제 비밀값과 `password=...`, `token: ...` 같은 구조적 민감 항목을 함께 마스킹하는 이유는 무엇입니까?
- 여러 비밀값이 서로 prefix 관계일 때 긴 문자열부터 치환해야 하는 이유는 무엇입니까?
- 빈 문자열을 치환 대상에 넣으면 어떤 문제가 생깁니까?
- 비밀 파일 경로도 민감 정보로 취급한 이유는 무엇입니까?
- 로그에 URL encoding, JSON escaping, shell quoting 등 변형된 비밀값이 남으면 단순 exact replacement가 놓치는 부분은 무엇입니까?
- 마스킹할 비밀 파일 하나를 읽지 못했을 때 읽은 값만 가리고 일부 진단을 게시하면 왜 위험합니까?
- 마스킹 함수의 idempotence는 어떤 의미이며 왜 테스트할 가치가 있습니까?
- 정규식 기반 assignment 마스킹의 false positive와 false negative를 어떻게 관리하겠습니까?

### 30초 모범 답변

진단 자료에는 실제 값뿐 아니라 비밀 파일의 상대·절대 경로와 `password=`, `token:` 형태의 파생 표현이 섞일 수 있습니다. 먼저 렌더링된 설정에서 모든 비밀 원본을 찾고 안전한 파일 reader로 값을 읽은 뒤, 긴 exact secret부터 치환하고 민감 assignment 패턴을 추가로 마스킹합니다. 비밀 하나라도 읽지 못하면 무엇을 가려야 하는지 완전하게 알 수 없으므로 출력 디렉터리를 만들기 전에 중단해야 합니다. 이 방식은 일부 진단을 잃는 대신 알려지지 않은 비밀이 게시되는 위험을 막는 fail-closed 선택입니다.

### 답변 핵심 키워드

secret inventory · raw path · canonical path · exact value redaction · longest first · sensitive assignment · fail-closed · no partial publication · idempotence · encoded variants · false positive/negative · diagnostic threat model

### 백지 구현

#### 구현 목표

여러 exact secret과 민감 assignment 정규식을 이용해 진단 문자열을 마스킹한다. prefix가 겹치는 값, 빈 값, 반복 실행, 구조적 key-value 표현을 안전하게 처리한다.

#### 인터페이스

```python
import re
from collections.abc import Iterable


class RedactionError(RuntimeError):
    pass


def redact_diagnostic_text(
    text: str,
    exact_secrets: Iterable[str],
    *,
    assignment_pattern: re.Pattern[str],
    replacement: str = "<redacted>",
) -> str:
    # 직접 구현
    ...
```

#### 입력과 출력

- 원본 진단 문자열, exact secret 집합, 구조적 민감 assignment를 찾는 compiled pattern을 받는다.
- 모든 알려진 exact secret과 pattern이 가리키는 값 부분을 `replacement`로 바꾼 새 문자열을 반환한다.
- 잘못된 계약 입력은 원본을 부분 처리하지 않고 `RedactionError`로 거부한다.

#### 반드시 만족해야 할 조건

- `replacement`가 빈 문자열이면 거부한다.
- exact secret의 빈 문자열은 조용히 전체 문자열 사이에 삽입 치환하지 않도록 명시적으로 거부하거나 제외하는 정책을 구현한다.
- 중복 secret은 한 번만 처리한다.
- prefix 관계의 secret은 길이가 긴 값부터 처리한다.
- exact secret 문자열 자체를 오류 메시지에 포함하지 않는다.
- assignment pattern은 민감 key와 구분자 형식은 유지하되 값 부분만 replacement로 바꾸는 계약을 가져야 한다.
- exact replacement 뒤 assignment replacement를 수행해 이미 가려진 결과가 다시 노출되지 않게 한다.
- 입력 문자열 객체를 변경하지 않고 새 문자열을 반환한다.
- 같은 입력에 함수를 두 번 적용해도 의미상 같은 결과가 나와야 한다.
- 반환 문자열에 non-empty exact secret이 남아 있지 않은지 최종 확인한다.

#### 경계 조건

- `abc`와 `abcdef`가 모두 secret인 경우
- 같은 secret이 여러 위치와 여러 줄에 나타나는 경우
- secret이 정규식 메타문자를 포함하는 경우
- replacement 문자열 자체가 민감 assignment pattern과 일치하는 경우
- `PASSWORD = value`, `token:value`, `secret\t=\tvalue` 같은 공백 변형
- punctuation, 쉼표, 세미콜론, 줄 끝 직전의 값
- Unicode secret과 매우 긴 로그
- exact secret이 이미 `<redacted>`인 경우
- assignment pattern의 capture group 계약이 잘못된 경우

#### 실패 조건과 제약

- secret을 정규식으로 직접 보간하지 않는다. exact 값은 literal replacement로 다룬다.
- 마스킹 대상 inventory가 불완전할 수 있는 상황을 이 함수의 성공으로 포장하지 않는다. 호출자가 모든 비밀을 수집했다는 사전 조건을 명확히 둔다.
- URL encoding·base64·부분 문자열 변형까지 자동 추측하지 않는다. 필요한 변형은 별도 inventory로 명시한다.
- 마스킹 전후 문자열을 디버그 로그에 출력하지 않는다.
- 처리 성능을 이유로 secret 값을 해시만 비교하고 실제 출력을 그대로 두지 않는다.

### 구현 후 자가 검증

- [ ] 겹치는 secret에서 긴 값 전체가 한 번에 가려진다.
- [ ] 중복·빈 secret 정책이 명확하고 빈 값 때문에 출력이 폭증하지 않는다.
- [ ] 정규식 메타문자가 포함된 exact secret도 literal로 치환된다.
- [ ] 민감 assignment의 key·구분자는 유지되고 값만 가려진다.
- [ ] 여러 줄과 Unicode 입력에서 secret이 남지 않는다.
- [ ] 함수를 두 번 적용해도 추가 손상이나 secret 재등장이 없다.
- [ ] 잘못된 assignment pattern 계약은 부분 결과를 반환하지 않고 실패한다.
- [ ] 오류 문자열에 secret 원문이 포함되지 않는다.
- [ ] replacement가 secret 목록에 포함된 경우의 정책을 테스트한다.
- [ ] 시간 복잡도와 secret 수가 커질 때의 비용을 설명할 수 있다.

### 구현 후 설명할 것

1. 알려진 exact 값과 구조적 assignment를 두 층으로 마스킹한 이유
2. 모든 비밀을 읽지 못하면 진단을 중단하는 fail-closed trade-off
3. 긴 값부터 치환해야 하는 prefix 충돌 문제
4. encoding·escaping 변형을 다루는 방법과 이 구현의 한계
5. false positive를 줄이면서도 비밀 누출을 우선 차단하는 정책

### 원본 확인 위치

- **Thread:** 12
- **커밋 메시지:** `feat(diagnostics): Compose 비밀값과 민감 항목 마스킹`
- **파일:** `tools/diagnose_stack.py`
- **함수:** `rendered_compose_config`, `secret_values`, `redact`
- **관련 위치:** `tools/stack_runtime.py`의 `read_private_secret`, `secret_source_paths`
- **관련 Thread:** 02의 안전한 비밀 파일 reader, 06의 런타임 비노출 검사, 11의 로그 정책, 13의 실패 진단 artifact allowlist

---

<a id="a-08"></a>
## A-08 · [Thread 12 / `feat(diagnostics): 컨테이너 런타임 상태 수집`] 비공개 증거 묶음 게시

### 면접 질문

장애 진단 결과를 일반 디렉터리에 덮어쓰는 방식이 왜 위험합니까? 이 프로젝트가 출력 디렉터리 0700, 파일 0600, 고정 파일 집합, 기존 경로·dangling symlink 거부를 검증한 이유를 설명해 보세요.

꼬리 질문:

- `Path.exists()`만으로 dangling symlink 출력 경로를 안전하게 거부할 수 있습니까?
- 기존 진단 경로를 덮어쓰지 않는 것이 보안과 운영 측면에서 각각 어떤 의미가 있습니까?
- 파일을 0600으로 만들더라도 상위 디렉터리가 공개되어 있으면 어떤 정보가 노출될 수 있습니까?
- 수집 명령이 실패했을 때 전체 진단을 버리는 대신 exit code·stdout·stderr를 증거 파일에 남기는 장점은 무엇입니까?
- 고정 allowlist 외의 파일을 생성하지 않는 이유는 무엇입니까?
- 파일 write와 `fsync`, 디렉터리 `fsync`는 crash 시 어떤 차이를 만듭니까?
- 다섯 파일 중 세 번째를 쓰다가 실패하면 부분 디렉터리를 남길지 제거할지 어떤 정책이 적절합니까?
- 진단 경로 자체가 비밀 파일 경로를 포함하지 않는지 어떻게 확인하겠습니까?

### 30초 모범 답변

진단에는 로그·설정·컨테이너 상태가 들어가므로 결과 자체를 민감 자산으로 봐야 합니다. 목적지가 기존 파일이나 dangling symlink까지 포함해 이미 존재하면 거부하고, 새 디렉터리를 0700으로 만든 뒤 허용된 이름의 일반 파일만 0600·배타 생성으로 씁니다. 수집 명령의 비정상 종료도 exit code와 출력으로 기록하되, 게시 전에 전체 텍스트를 마스킹합니다. 기존 결과는 덮어쓰지 않고 파일과 디렉터리를 동기화해 증거의 변경·부분 게시 가능성을 줄입니다. 중간 실패 시 부분 결과 처리 정책도 명시적으로 정해야 합니다.

### 답변 핵심 키워드

private evidence · 0700 directory · 0600 files · lexists · dangling symlink · exclusive creation · fixed allowlist · no overwrite · command failure as evidence · redact before publish · fsync · partial bundle policy

### 백지 구현

#### 구현 목표

정확히 정해진 진단 파일 다섯 개를 새 비공개 디렉터리에 게시한다. 기존 경로와 symlink를 거부하고, 각 파일을 배타적으로 생성하며, 실패 시 부분 결과를 정해진 정책대로 처리한다.

#### 인터페이스

```python
from pathlib import Path
from collections.abc import Mapping


ALLOWED_EVIDENCE_FILES = frozenset(
    {
        "versions.txt",
        "compose-ps.txt",
        "compose-logs.txt",
        "compose-model.txt",
        "container-state.txt",
    }
)


class EvidencePublicationError(RuntimeError):
    pass


def publish_evidence_bundle(
    destination: Path,
    artifacts: Mapping[str, str],
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- 목적지 디렉터리와 파일명→이미 마스킹된 텍스트 mapping을 받는다.
- 성공 시 목적지에는 allowlist와 정확히 같은 일반 파일 집합만 존재해야 한다.
- 기존 목적지나 게시 실패 시 `EvidencePublicationError`를 발생시킨다.

#### 반드시 만족해야 할 조건

- artifact key 집합이 allowlist와 정확히 일치하는지 side effect 전에 확인한다.
- 파일명에 `/`, `..`, 빈 문자열이 들어갈 수 없어야 한다.
- destination은 일반 존재 여부뿐 아니라 dangling symlink까지 포함해 어떤 디렉터리 엔트리도 없어야 한다.
- 새 destination을 0700으로 생성한다.
- 각 파일은 0600, write-only, create, exclusive 방식으로 생성하며 symlink follow를 허용하지 않는다.
- 텍스트 encoding은 명시적으로 UTF-8을 사용한다.
- 각 파일을 flush·fsync한 뒤 닫는다.
- 모든 파일이 생성된 뒤 destination 디렉터리를 fsync한다.
- 기존 파일을 truncate하거나 replace하지 않는다.
- 파일 생성 도중 실패하면 지금까지 자신이 만든 파일과 디렉터리만 정리한다.
- cleanup 실패가 있으면 원래 publication 오류와 함께 보고한다.

#### 경계 조건

- destination이 빈 디렉터리, 일반 파일, symlink, dangling symlink인 경우
- allowlist 파일 하나가 누락되거나 추가 파일이 있는 경우
- 부모 디렉터리가 없거나 쓰기 불가능한 경우
- 첫 파일, 중간 파일, 마지막 파일 쓰기에서 실패하는 경우
- UTF-8로 표현할 수 없는 surrogate가 포함된 문자열
- 매우 큰 로그 문자열
- 파일 생성 직전에 공격자가 같은 이름을 만든 경우
- fsync가 실패하는 경우

#### 실패 조건과 제약

- 이 함수 안에서 원본 로그를 다시 마스킹하려 하지 않는다. 입력이 마스킹됐다는 계약과 호출 순서를 분리한다.
- `write_text()` 한 줄로 모드·배타 생성·fsync를 생략하지 않는다.
- cleanup을 위해 destination 부모의 다른 파일을 탐색하거나 삭제하지 않는다.
- partial bundle을 성공으로 반환하지 않는다.
- publication 오류 메시지에 artifact 본문을 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 게시 후 디렉터리 모드는 0700, 파일 모드는 모두 0600이다.
- [ ] 파일 집합이 allowlist와 정확히 같다.
- [ ] 일반 파일·디렉터리·정상 symlink·dangling symlink 목적지를 모두 거부한다.
- [ ] 기존 경로의 바이트와 메타데이터를 변경하지 않는다.
- [ ] 중간 실패 시 자신이 만든 부분 파일만 제거한다.
- [ ] cleanup 실패와 원래 write/fsync 실패가 함께 보존된다.
- [ ] 파일명 path traversal이 side effect 전에 거부된다.
- [ ] 각 파일과 디렉터리 fsync 호출을 test double로 확인한다.
- [ ] 큰 입력을 처리할 때 불필요한 복제 횟수가 제한된다.
- [ ] 오류 출력에 artifact 본문이나 비밀값이 나타나지 않는다.

### 구현 후 설명할 것

1. 진단 결과를 일반 로그가 아니라 비공개 artifact로 취급한 이유
2. `exists`와 dangling symlink를 포함한 lexically existing 경로 검사의 차이
3. allowlist와 no-overwrite가 증거 신뢰성을 높이는 방식
4. 중간 실패 시 partial bundle을 제거하는 정책의 장단점
5. 파일 fsync와 디렉터리 fsync를 모두 수행하는 이유

### 원본 확인 위치

- **Thread:** 12
- **커밋 메시지:** `feat(diagnostics): 컨테이너 런타임 상태 수집`
- **파일:** `tools/diagnose_stack.py`, `tests/runtime_stack.py`
- **함수:** `run`, `container_state`
- **출력 파일:** `versions.txt`, `compose-ps.txt`, `compose-logs.txt`, `compose-model.txt`, `container-state.txt`
- **관련 Thread:** 11의 stdout/stderr·로그 회전 정책, 13의 실패 artifact 업로드 allowlist

---

<a id="a-09"></a>
## A-09 · [Thread 13 / `test(bootstrap): 격리된 런타임 하네스 추가` + 런타임 복구 시나리오] 결정적 실패 주입 하네스

### 면접 질문

초기화·백업·회전 복구 테스트에서 임의의 `sleep` 뒤 프로세스를 죽이는 방식이 왜 불안정합니까? 이 프로젝트가 stage별 pause 지점과 ready file을 사용해 정확한 상태에서 SIGKILL·SIGTERM을 주입한 이유를 설명해 보세요.

꼬리 질문:

- 테스트 프로젝트 이름, 임시 디렉터리, 비밀 파일, 포트를 실행마다 격리해야 하는 이유는 무엇입니까?
- ready marker가 나타나기 전에 child process가 종료되면 어떤 결과로 처리해야 합니까?
- wall clock 대신 monotonic clock을 사용해야 하는 이유는 무엇입니까?
- marker가 symlink이거나 다른 사용자가 만든 파일이면 왜 신뢰하면 안 됩니까?
- SIGTERM과 SIGKILL은 정리·복구 로직에서 무엇을 각각 검증합니까?
- 프로세스에 신호를 보낸 뒤 종료를 기다리지 않으면 어떤 자원 누수가 생길 수 있습니까?
- 테스트 본문 실패와 cleanup 실패가 동시에 발생하면 어떤 정보를 남겨야 합니까?
- `--keep` 같은 디버그 옵션은 CI 기본 동작과 어떻게 분리해야 합니까?

### 30초 모범 답변

`sleep`은 머신 부하와 실행 속도에 따라 다른 단계에서 신호를 보내므로 재현성이 없습니다. 운영 코드에 테스트 전용 pause stage와 private ready marker를 두면 하네스가 "DB 상태 작성 후, publish 전"처럼 정확한 경계를 관찰한 뒤 신호를 한 번 보낼 수 있습니다. 대기에는 monotonic timeout과 child 조기 종료 검사를 사용하고, marker의 종류·소유권을 확인합니다. 신호 뒤 child 종료와 재실행 수렴을 확인하며, `finally`에서 프로젝트 자원을 정리하고 원래 실패와 cleanup 실패를 모두 보존합니다.

### 답변 핵심 키워드

deterministic failure injection · explicit stage · ready marker · private test channel · monotonic timeout · child liveness · SIGTERM · SIGKILL · wait/reap · isolated project · finally cleanup · reproducibility

### 백지 구현

#### 구현 목표

child process가 private ready file로 목표 단계 도달을 알릴 때까지 기다린 뒤 지정한 signal을 정확히 한 번 보내고 종료 결과를 회수하는 interruption controller를 구현한다.

#### 인터페이스

```python
from pathlib import Path
import signal
import subprocess
from collections.abc import Callable


class StageInterruptionError(RuntimeError):
    pass


def interrupt_when_stage_ready(
    process: subprocess.Popen[bytes],
    ready_file: Path,
    *,
    signal_number: signal.Signals,
    timeout_seconds: float,
    monotonic: Callable[[], float],
    sleep: Callable[[float], None],
    poll_interval_seconds: float = 0.05,
) -> int:
    # 직접 구현
    ...
```

#### 입력과 출력

- 실행 중인 child process, stage ready file, 보낼 signal, timeout과 시간 callback을 받는다.
- marker를 안전하게 확인한 뒤 signal을 한 번 보내고 child의 최종 return code를 반환한다.
- marker 이전 종료, timeout, 잘못된 marker, 신호 전송 실패, 종료 대기 실패는 `StageInterruptionError`로 보고한다.

#### 반드시 만족해야 할 조건

- timeout과 poll interval이 양수인지 side effect 전에 확인한다.
- wall clock이 아니라 주입된 monotonic 시간을 기준으로 deadline을 계산한다.
- 각 polling cycle에서 child가 이미 종료됐는지 확인한다.
- ready path가 존재하면 symlink가 아닌 일반 파일인지 확인한다.
- ready file의 소유자가 현재 사용자이고 group/other 접근 비트가 없는지 확인하는 정책을 포함한다.
- 안전한 marker가 확인된 경우에만 signal을 한 번 보낸다.
- signal을 보낸 뒤 child를 wait하여 zombie를 남기지 않는다.
- wait에도 남은 timeout을 적용한다.
- timeout이 발생하면 child의 현재 상태와 목표 marker를 포함하되 파일 내용은 출력하지 않는다.
- 자신이 생성하지 않은 ready file이나 주변 경로를 삭제하지 않는다.

#### 경계 조건

- 함수 호출 직후 marker가 이미 존재하는 경우
- marker 생성 직전에 child가 종료되는 경우
- marker가 생성된 직후 child가 자연 종료되는 경우
- marker가 symlink, 디렉터리, FIFO인 경우
- marker 권한 또는 소유자가 잘못된 경우
- signal 전송 시 process가 이미 사라진 경우
- signal을 무시해 timeout까지 종료하지 않는 child
- `poll_interval_seconds`보다 작은 timeout
- monotonic callback이 동일 값을 여러 번 반환하는 경우

#### 실패 조건과 제약

- marker 대기를 위해 busy loop를 만들지 않는다.
- marker의 내용으로 임의 shell 명령을 만들지 않는다.
- signal 전송과 wait 사이에 같은 process 객체를 다른 controller가 조작하지 않는다는 계약을 명시한다.
- PID를 문자열 파일에서 다시 읽어 신호를 보내지 않는다. 전달받은 process handle을 사용한다.
- timeout 후 child를 어떻게 최종 정리할지는 상위 scenario runner의 책임으로 분리한다.

### 구현 후 자가 검증

- [ ] marker가 안전하게 생성되면 signal을 정확히 한 번 보낸다.
- [ ] marker 이전 child 종료를 timeout과 구분해 즉시 보고한다.
- [ ] symlink·비일반 파일·공개 권한 marker를 거부한다.
- [ ] monotonic deadline을 사용하고 polling이 bounded다.
- [ ] signal 후 wait를 호출해 return code를 회수한다.
- [ ] signal 전송 race와 wait timeout 오류가 원인별로 구분된다.
- [ ] 오류 메시지에 marker 내용이나 비밀값이 포함되지 않는다.
- [ ] 이미 존재하는 안전한 marker도 처리한다.
- [ ] 함수가 주변 파일이나 project 자원을 삭제하지 않는다.
- [ ] polling 횟수는 대략 `O(timeout / interval)`이고 추가 공간은 `O(1)`이다.

### 구현 후 설명할 것

1. 임의 sleep과 explicit ready marker의 결정성 차이
2. SIGTERM과 SIGKILL로 서로 다른 복구 경계를 검증하는 이유
3. process handle·liveness·wait를 사용해 PID 재사용과 zombie 위험을 줄이는 방법
4. 테스트 전용 pause hook이 운영 코드 복잡도에 주는 비용
5. interruption controller와 전체 scenario cleanup 책임을 분리한 이유

### 원본 확인 위치

- **Thread:** 13
- **커밋 메시지:** `test(bootstrap): 격리된 런타임 하네스 추가`, `ci(stack): 정적·런타임·복구 검증 자동화`
- **파일:** `tests/runtime_stack.py`, `tools/verify_stack.py`
- **클래스·함수:** `RuntimeStack`, 런타임 scenario 실행 경계
- **관련 위치:** Thread 02의 `pause_arguments`, Thread 04·05의 bootstrap stage, Thread 07의 `pause_for_test`, Thread 09의 회전 pause·rollback ready file
- **관련 Thread:** 02, 04, 05, 07, 08, 09, 13

---

<a id="a-10"></a>
## A-10 · [Thread 13 / `ci(stack): 정적·런타임·복구 검증 자동화`] 기록 기반 범위 제한 정리와 증거 수명주기

### 면접 질문

통합 테스트 뒤 `docker system prune` 같은 광역 정리를 사용하면 왜 안 됩니까? 이 프로젝트가 실행 중 생성한 project record를 남기고, 그 범위와 label을 확인해 컨테이너·네트워크·볼륨·이미지만 정리한 이유를 설명해 보세요.

꼬리 질문:

- 테스트 시작 전에 project identifier를 기록해야 하는 이유는 무엇입니까?
- 이름이 record에 있다는 이유만으로 삭제하지 않고 Compose project label까지 확인해야 하는 이유는 무엇입니까?
- 컨테이너, 네트워크, 볼륨, 이미지의 제거 순서는 왜 중요합니까?
- 이미 사라진 자원은 실패입니까, 멱등 성공입니까?
- record가 손상되거나 예상 형식이 아니면 보수적으로 어떤 행동을 해야 합니까?
- cleanup 일부가 실패해도 나머지 자원을 계속 정리해야 합니까?
- cleanup report를 별도 파일로 남기는 이유는 무엇입니까?
- 정리에 실패했을 때 임시 디렉터리와 진단 자료를 보존하고, 성공했을 때 제거하는 정책의 장단점은 무엇입니까?

### 30초 모범 답변

공유 CI 호스트에서 광역 prune은 다른 job이나 사용자의 자원을 삭제할 수 있습니다. 그래서 scenario가 사용할 project 이름과 image reference를 side effect 전에 기록하고, cleanup은 record에 있는 값만 후보로 삼습니다. 실제 자원은 Compose project label과 기대한 이름을 다시 확인해 소유권이 불명확하면 삭제하지 않고 보고합니다. 의존 관계에 맞춰 컨테이너부터 정리하고, 누락 자원은 멱등하게 처리하되 각 실패를 집계합니다. cleanup 실패 시 report와 진단 디렉터리를 보존해야 원래 테스트 실패뿐 아니라 누수 원인도 추적할 수 있습니다.

### 답변 핵심 키워드

scoped cleanup · ownership record · label verification · no broad prune · least destructive · dependency order · idempotence · anomaly report · aggregate failures · evidence preservation · CI shared host

### 백지 구현

#### 구현 목표

테스트가 기록한 프로젝트와 이미지 목록, 현재 관찰한 Docker 자원을 받아 안전하게 삭제 가능한 항목만 결정적인 순서로 계획한다. 실제 삭제는 하지 않고 계획과 거부 사유를 반환한다.

#### 인터페이스

```python
from dataclasses import dataclass
from collections.abc import Iterable, Mapping


@dataclass(frozen=True)
class ProjectRecord:
    project_name: str
    image_references: frozenset[str]


@dataclass(frozen=True)
class ObservedResource:
    kind: str  # container, network, volume, image
    name: str
    labels: Mapping[str, str]


@dataclass(frozen=True)
class Removal:
    kind: str
    name: str


def build_scoped_cleanup_plan(
    records: Iterable[ProjectRecord],
    observed: Iterable[ObservedResource],
) -> tuple[list[Removal], list[str]]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 테스트가 남긴 project record와 현재 관찰한 자원을 받는다.
- 첫 번째 반환값은 실제 삭제기에 전달할 안전한 제거 순서다.
- 두 번째 반환값은 삭제를 거부한 항목, 손상된 record, 소유권 불일치 등의 진단 메시지다.

#### 반드시 만족해야 할 조건

- project name은 소문자·숫자·밑줄·하이픈으로 구성된 허용 길이인지 검증한다.
- malformed record는 후보로 사용하지 않고 오류 목록에 추가한다.
- container·network·volume은 `com.docker.compose.project` label이 정확히 기록된 project와 일치할 때만 제거 후보가 된다.
- 이름이 비슷하거나 prefix만 같은 다른 project 자원은 제외한다.
- image는 record에 정확히 기록된 reference만 후보로 삼는다.
- 관찰됐지만 record에 없는 자원은 삭제하지 않는다.
- 같은 자원 중복 관찰은 제거 계획에 한 번만 포함한다.
- 제거 순서는 container → network → volume → image로 결정한다.
- 이미 관찰되지 않는 기록 항목은 계획에 넣지 않으며 오류로 볼지 정보로 볼지 정책을 명확히 한다.
- 반환 순서는 입력 순서와 무관하게 결정적이어야 한다.

#### 경계 조건

- record가 비어 있는 경우
- 같은 project가 여러 번 기록되고 image 집합이 다른 경우
- label이 없거나 다른 project를 가리키는 자원
- kind가 알려지지 않은 자원
- 서로 다른 kind에 같은 name이 있는 경우
- image reference가 빈 문자열이거나 태그가 없는 경우
- project 이름이 최대 길이 경계에 있는 경우
- 관찰 목록에 동일 객체가 여러 번 나타나는 경우
- record에는 있지만 현재 이미 삭제된 자원

#### 실패 조건과 제약

- `prune`, wildcard, prefix-only 삭제 계획을 만들지 않는다.
- label이 불명확한 자원을 "아마 테스트 것"이라고 추측하지 않는다.
- 실제 Docker 명령을 실행하지 않는다. 계획 생성과 실행을 분리한다.
- 한 malformed record 때문에 안전한 다른 record의 분석을 중단하지 않는다.
- 오류 메시지에 secret이나 환경 파일 내용을 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 record와 label이 일치하는 자원만 계획에 포함된다.
- [ ] 비슷한 이름의 다른 project 자원은 제외된다.
- [ ] label 누락·불일치는 제거하지 않고 오류로 보고된다.
- [ ] image는 정확한 recorded reference만 포함된다.
- [ ] 중복 관찰이 제거 계획을 중복시키지 않는다.
- [ ] 제거 순서가 container→network→volume→image다.
- [ ] malformed record와 알 수 없는 kind를 보수적으로 거부한다.
- [ ] 입력 순서를 섞어도 같은 계획과 오류 순서가 나온다.
- [ ] 이미 없는 자원에 대한 멱등 정책이 테스트로 고정된다.
- [ ] 알고리즘은 record·observed 수에 대해 `O(n log n)` 이내다.

### 구현 후 설명할 것

1. record와 런타임 label을 함께 사용한 이중 소유권 확인
2. broad prune을 금지한 공유 호스트 안전성 원칙
3. 제거 의존 순서와 멱등 cleanup의 관계
4. 일부 cleanup 실패를 집계하면서 나머지를 계속하는 이유
5. cleanup 실패 시 evidence를 보존하는 수명주기 정책

### 원본 확인 위치

- **Thread:** 13
- **커밋 메시지:** `ci(stack): 정적·런타임·복구 검증 자동화`
- **파일:** `tools/cleanup_test_resources.py`, `tools/verify_stack.py`, `.github/workflows/container-stack.yml`, `tests/runtime_stack.py`, `tests/validate_stack.py`
- **관련 컴포넌트:** project record 디렉터리, cleanup report, runtime scenario cleanup, 실패 진단 artifact
- **관련 Thread:** 02의 프로젝트 범위, 07·08·09의 operation lock과 cleanup, 12의 비공개 진단

---

<a id="a-11"></a>
## A-11 · [Thread 13 / `test(ci): workflow 검증 계약 추가`] AST·렌더링 모델 기반 회귀 계약

### 면접 질문

운영 도구에서 `subprocess.run`에 timeout이 빠졌는지 문자열 검색으로 검사하는 방식과 Python AST로 검사하는 방식의 차이는 무엇입니까? 이 프로젝트가 소스 문자열, AST, 렌더링된 Compose 모델, 실제 runtime scenario를 함께 사용한 이유를 설명해 보세요.

꼬리 질문:

- 문자열 검색은 주석·문자열 literal·줄바꿈 때문에 어떤 오탐과 누락을 만들 수 있습니까?
- AST는 어떤 구조적 사실을 확인할 수 있고, 반대로 실행 시 동적으로 만들어지는 값은 왜 알기 어렵습니까?
- Docker Compose YAML 원문 대신 `docker compose config`로 렌더링된 모델을 검사해야 하는 항목은 무엇입니까?
- workflow action을 tag가 아니라 commit SHA로 고정한 이유는 무엇입니까?
- workflow의 permissions, trigger, artifact path allowlist를 정적 계약으로 검사하는 이유는 무엇입니까?
- subprocess command 인자에 비밀값이 들어가지 않는지 AST만으로 완전히 증명할 수 있습니까?
- static validator가 repository 문구·README 정책과 functional 정책을 분리한 이유는 무엇입니까?
- 정적 검사와 런타임 테스트가 같은 계약을 중복 확인하는 것이 언제 가치가 있습니까?

### 30초 모범 답변

문자열 검사는 빠르지만 주석과 formatting에 민감하고 실제 호출 구조를 이해하지 못합니다. AST는 `subprocess.run` 호출과 keyword 인자를 구조적으로 찾아 timeout 누락 같은 계약을 비교적 정확히 잡을 수 있습니다. 다만 wrapper 내부나 동적 명령 의미까지는 증명하지 못하므로, Compose 변수·merge 결과는 렌더링 모델로 확인하고 실제 네트워크·자원·복구 동작은 런타임 scenario로 검증해야 합니다. CI workflow도 권한 최소화, action SHA, trigger, artifact allowlist를 정적으로 고정해 검증 경로 자체의 drift를 막습니다.

### 답변 핵심 키워드

text scan · AST · structural contract · false positive/negative · rendered Compose model · runtime verification · defense in depth · subprocess timeout · credential arguments · pinned action SHA · least permissions · artifact allowlist · policy separation

### 백지 구현

#### 구현 목표

Python 소스에서 직접 호출된 `subprocess.run`과 `from subprocess import run` 별칭 호출을 찾아, `timeout=` keyword가 없는 위치를 반환하는 AST validator를 구현한다.

#### 인터페이스

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Finding:
    line: int
    column: int
    call_name: str


class SourceValidationError(RuntimeError):
    pass


def find_subprocess_runs_without_timeout(source: str) -> list[Finding]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 하나의 Python source 문자열을 받는다.
- `subprocess.run(...)` 또는 import alias를 통한 동일 함수 호출 중 `timeout` keyword가 없는 위치를 source 순서대로 반환한다.
- 문법 오류나 분석할 수 없는 import alias 충돌은 `SourceValidationError`로 보고한다.

#### 반드시 만족해야 할 조건

- `import subprocess` 뒤의 `subprocess.run(...)`을 찾는다.
- `import subprocess as sp` 뒤의 `sp.run(...)`을 찾는다.
- `from subprocess import run`과 `from subprocess import run as execute` 호출을 찾는다.
- `timeout=...`이 존재하면 값의 종류와 관계없이 누락으로 보고하지 않는다.
- `**kwargs`만 전달된 호출은 timeout 존재를 확정할 수 없으므로 정책에 따라 별도 finding 또는 분석 불가로 표시한다.
- 주석·문자열 안의 `subprocess.run(` 텍스트는 무시한다.
- 다른 객체의 `.run()`을 subprocess 호출로 잘못 분류하지 않는다.
- 동일 위치를 중복 보고하지 않는다.
- finding은 line, column, 사람이 이해할 call name을 포함한다.
- 결과는 source 위치 순서로 결정적이어야 한다.

#### 경계 조건

- 여러 import 형식이 한 파일에 섞인 경우
- 함수 내부에서 import alias를 다른 값으로 재할당하는 경우
- `timeout = None`으로 명시한 경우
- `timeout`이 keyword가 아니라 `**options`에 포함될 가능성이 있는 경우
- `getattr(subprocess, "run")`이나 동적 import
- 중첩 함수와 class method 내부 호출
- syntax error가 있는 source
- `subprocess.run` 이름을 지역 변수로 shadowing한 경우
- 여러 줄 호출과 keyword 순서 변화

#### 실패 조건과 제약

- 정규식만으로 구현하지 않는다.
- 동적 import, `getattr`, arbitrary wrapper까지 완전하게 추론한다고 주장하지 않는다.
- timeout 값이 양수인지까지 검사하지 않는다. 이 문제의 계약은 keyword 존재 여부다.
- source를 실행하거나 import하지 않는다.
- 분석 한계를 조용히 정상으로 처리하지 말고 명시적인 정책을 둔다.

### 구현 후 자가 검증

- [ ] 네 가지 import·alias 형태의 timeout 누락을 찾는다.
- [ ] timeout이 있는 호출은 제외한다.
- [ ] 주석·문자열·다른 객체의 `.run()`은 오탐하지 않는다.
- [ ] 여러 줄 호출의 정확한 line·column을 반환한다.
- [ ] `**kwargs`, shadowing, 동적 호출에 대한 보수적 정책이 테스트로 고정된다.
- [ ] syntax error를 빈 finding으로 오해하지 않고 별도 오류로 보고한다.
- [ ] finding 순서가 source 위치 기준으로 결정적이다.
- [ ] source를 실행하지 않아 side effect가 없다.
- [ ] 대형 파일에서도 AST node 수에 비례해 동작한다.
- [ ] validator의 한계를 설명하는 테스트 또는 문서가 있다.

### 구현 후 설명할 것

1. 문자열 검색보다 AST가 줄이는 오탐·누락과 AST의 남은 한계
2. source policy, rendered configuration, runtime behavior를 계층별로 검증한 이유
3. `**kwargs`와 alias shadowing을 보수적으로 처리하는 방법
4. workflow 권한·action SHA·artifact allowlist를 코드 계약처럼 검사하는 이유
5. 정적 validator 자체가 과도하게 구현 세부사항에 결합될 때의 유지보수 비용

### 원본 확인 위치

- **Thread:** 13
- **커밋 메시지:** `test(ci): workflow 검증 계약 추가`, `ci(stack): 정적·런타임·복구 검증 자동화`, `ci: harden container stack validation`
- **파일:** `tests/validate_stack.py`, `.github/workflows/container-stack.yml`, `tools/check_commit_range.py`, `tools/verify_stack.py`
- **함수:** `validate_ci`, `require_subprocess_timeouts`, `validate_no_credential_arguments`
- **관련 Thread:** 02의 subprocess 실행 경계, 10의 공급망 정적 검사, 11의 런타임 정책 검사, 12의 artifact allowlist
