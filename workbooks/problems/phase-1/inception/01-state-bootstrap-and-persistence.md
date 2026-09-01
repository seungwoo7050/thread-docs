# 상태 수렴, 초기화, 영속성

이 문서는 중단 가능한 초기화와 영속 상태의 경계를 다룬다. 백지 구현은 실제 Docker·MariaDB·WordPress 전체를 재현하는 문제가 아니라, 원본 구현이 지켜야 했던 상태 분류와 invariant를 10~30분 안에 확인할 수 있도록 축소했다.

---

<a id="s-01"></a>
## S-01 · [Thread 04 / `fix(init): 중단된 단계별 초기화를 수렴`] MariaDB crash-safe 상태 공개

### 면접 질문

MariaDB 데이터 디렉터리를 바로 초기화하지 않고 형제 staging 디렉터리에서 만든 뒤 공개한 이유를 설명해 보세요. 완료 표식 파일만 남기면 충분한지, `sync`와 디렉터리 rename은 각각 어떤 실패를 막는지도 설명해 보세요.

꼬리 질문:

- 프로세스가 시스템 테이블 생성 직후, 계정 변경 직후, marker 생성 직후, rename 직후에 강제 종료되면 다음 실행은 무엇을 관찰해야 합니까?
- 데이터 디렉터리는 존재하지만 완료 표식이 없다면 자동으로 이어서 초기화하는 것과 실패하는 것 중 무엇을 선택하겠습니까?
- POSIX rename의 원자성이 보장되어도 왜 부모 디렉터리 `fsync`가 필요합니까?
- staging과 live 디렉터리가 서로 다른 파일시스템에 있으면 설계가 어떻게 달라집니까?
- 기존 live 데이터가 있으면 marker만 확인하면 됩니까, 실제 DB 인증과 스키마도 다시 확인해야 합니까?

### 30초 모범 답변

초기화 도중 죽었을 때 부분 완성 디렉터리를 정상 상태로 오인하지 않게 하려는 설계입니다. 새 상태는 같은 볼륨의 staging에서 만들고 실제 계정·DB 접근을 검증한 다음 완료 표식을 기록합니다. 파일과 staging 디렉터리를 동기화한 뒤 live 경로로 원자적으로 rename하고 부모 디렉터리까지 동기화합니다. 그래서 관찰 가능한 live 상태는 없거나 완성된 상태 둘 중 하나가 됩니다. 기존 데이터가 있는데 marker가 없으면 의미가 모호하므로 임의 복구보다 실패하고, marker가 있어도 인증과 핵심 상태를 재검증하는 편이 안전합니다.

### 답변 핵심 키워드

staging · same filesystem · atomic rename · completion marker · verify before publish · file fsync · directory fsync · ambiguous partial state · fail closed · idempotent re-entry

### 백지 구현

#### 구현 목표

같은 부모 디렉터리 안의 staging 디렉터리를 live 디렉터리로 게시하는 함수를 구현한다. 함수는 게시 직전의 구조적 조건과 완료 표식을 검증하고, durable publication에 필요한 동기화 순서를 지켜야 한다.

#### 인터페이스

```python
from pathlib import Path

class PublicationError(RuntimeError):
    pass


def publish_staged_directory(
    parent: Path,
    *,
    staging_name: str,
    live_name: str,
    marker_name: str,
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- `parent`: staging과 live가 함께 존재해야 하는 부모 디렉터리
- `staging_name`: 공개할 임시 디렉터리 이름
- `live_name`: 최종 디렉터리 이름
- `marker_name`: staging 내부의 완료 표식 이름
- 성공 시 반환값은 없고, live 경로가 staging의 완성 상태를 가리킨다.
- 실패 시 `PublicationError`를 발생시킨다.

#### 반드시 만족해야 할 조건

- `parent`, staging, marker는 심볼릭 링크가 아닌 예상 종류여야 한다.
- staging과 live는 `parent`의 직접 자식이어야 하며 `.`·`..`·경로 구분자를 이름으로 허용하지 않는다.
- live가 이미 존재하면 덮어쓰지 않는다.
- marker는 staging 안의 일반 파일이어야 한다.
- marker와 staging 디렉터리의 내용을 게시 전에 동기화한다.
- staging을 live로 같은 파일시스템 안에서 rename한다.
- rename 뒤 `parent` 디렉터리를 동기화한다.
- 함수가 성공을 반환한 뒤에는 staging 이름이 남아 있지 않아야 한다.

#### 경계 조건

- 빈 이름, `.` 또는 `..`, `/`가 포함된 이름
- marker 누락, marker가 디렉터리이거나 symlink인 경우
- staging이 없거나 일반 디렉터리가 아닌 경우
- live가 파일·디렉터리·dangling symlink 중 어떤 형태로든 이미 존재하는 경우
- `fsync` 또는 rename 실패
- rename 직전과 직후에 예외가 발생하는 경우

#### 실패 조건과 제약

- 다른 파일시스템으로 복사해서는 안 된다. 이 함수의 계약은 같은 `parent` 아래 원자 rename이다.
- 기존 live 상태를 자동 삭제하거나 교체하지 않는다.
- 실패 후 어느 경로가 공개되었는지 호출자가 판별할 수 있도록 원래 OS 오류를 원인으로 보존한다.
- 디렉터리 전체를 메모리에 읽거나 재귀 복사하는 방식은 허용하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 staging과 marker를 넘기면 live로 한 번만 게시된다.
- [ ] live가 미리 존재하면 내용이 바뀌지 않은 채 실패한다.
- [ ] marker 누락·symlink·디렉터리를 모두 거부한다.
- [ ] staging 또는 live 이름으로 `../x`, `a/b`, 빈 문자열을 거부한다.
- [ ] 성공 뒤 staging 경로가 사라지고 live 경로가 존재한다.
- [ ] marker 동기화, staging 디렉터리 동기화, rename, 부모 디렉터리 동기화 순서를 테스트 대역으로 검증할 수 있다.
- [ ] rename 뒤 부모 `fsync`가 실패하면 성공으로 보고하지 않는다.
- [ ] 실패해도 기존 live 디렉터리를 삭제하지 않는다.
- [ ] 모든 파일 디스크립터가 성공·실패 경로에서 닫힌다.
- [ ] 시간 복잡도는 디렉터리 내용 크기에 선형으로 증가하지 않고, 게시 단계 자체는 메타데이터 연산 중심이다.

### 구현 후 설명할 것

1. marker를 생성하는 것과 완성 상태를 검증하는 것의 차이
2. rename의 원자성과 저장장치에 대한 지속성 보장의 차이
3. live가 존재할 때 overwrite 대신 거부한 이유
4. staging과 live를 같은 부모에 둔 이유
5. 게시 전후에 프로세스가 죽었을 때 허용되는 관찰 상태

### 원본 확인 위치

- Thread 04
- 커밋: `fix(init): 중단된 단계별 초기화를 수렴`
- 파일: `srcs/requirements/mariadb/tools/docker-entrypoint.sh`
- 함수·컴포넌트: `start_temporary_server`, `stop_temporary_server`, `write_option_file`, `verify_database`, `runtime`, `bootstrap`
- 상태 경로·단계: staging 데이터 디렉터리, `.container-stack-initialized`, `system-tables`, `temporary-server`, `database-state`, `database-marker`, `database-publish`
- 관련 Thread: 02, 03, 06, 13

---

<a id="s-02"></a>
## S-02 · [Thread 05 / `fix(init): 중단된 단계별 초기화를 수렴`] WordPress reconciliation과 설정 격리

### 면접 질문

WordPress entrypoint가 "설치되어 있으면 건너뛴다" 수준의 초기화가 아니라 현재 상태를 검사하고 수렴시키는 구조여야 하는 이유를 설명해 보세요. 웹 루트와 설정 볼륨을 분리하고 `wp-config.php`를 symlink로 게시한 설계의 장단점도 설명해 보세요.

꼬리 질문:

- 웹 루트에 기존 일반 파일 `wp-config.php`가 있고 전용 설정 볼륨은 비어 있다면 어떤 순서로 이동해야 합니까?
- 완료 marker는 있는데 설정 파일의 DB 사용자나 비밀번호가 기대값과 다르면 자동 덮어쓰기해야 합니까?
- WordPress core 파일과 사용자 콘텐츠를 같은 정책으로 동기화하면 어떤 문제가 생깁니까?
- URL 변경은 수렴 가능한 구성 변경인데 DB 자격증명 변경은 별도 회전 절차로 보낸 이유는 무엇입니까?
- 설정 파일을 임시 파일에 쓴 뒤 rename할 때 symlink 공격과 부분 쓰기를 어떻게 막습니까?

### 30초 모범 답변

WordPress 상태는 core 파일, 사용자 콘텐츠, 설정, DB 설치, 계정, 완료 marker가 서로 다른 저장소에 걸쳐 있어서 단순 존재 여부만으로 정상 여부를 판단할 수 없습니다. 그래서 관찰 상태를 신규·레거시·부분 완료·완료·드리프트로 나누고 안전한 전이만 수행해야 합니다. 설정은 전용 볼륨의 일반 파일로 유지하고 웹 루트에는 정확한 symlink만 공개해 콘텐츠와 자격증명의 수명을 분리합니다. URL처럼 의도적으로 수렴 가능한 값은 갱신하지만 DB 자격증명 불일치는 별도 회전이 필요한 상태로 보고 자동 덮어쓰지 않습니다. 모든 파일 게시 후 사용자 인증까지 확인한 다음 marker를 마지막에 공개합니다.

### 답변 핵심 키워드

reconciliation · observed state · desired state · idempotence · legacy migration · config volume · exact symlink target · atomic file replace · credential drift · fail closed · marker last

### 백지 구현

#### 구현 목표

관찰된 WordPress 설정 상태를 안전한 작업 계획으로 분류하는 순수 함수를 구현한다. 실제 파일 수정이나 WordPress 명령 실행은 하지 않는다. 면접관은 상태 분류의 완전성과 위험한 자동 복구를 피하는 판단을 본다.

#### 인터페이스

```python
from dataclasses import dataclass
from enum import Enum, auto

class ConfigAction(Enum):
    CREATE_NEW = auto()
    MIGRATE_LEGACY = auto()
    REPAIR_LINK = auto()
    UPDATE_URL_ONLY = auto()
    KEEP = auto()
    REFUSE = auto()

@dataclass(frozen=True)
class ConfigObservation:
    marker_exists: bool
    web_entry_kind: str
    web_link_target: str | None
    config_entry_kind: str
    syntax_valid: bool
    database_identity_matches: bool
    database_password_matches: bool
    url_matches: bool
    legacy_and_config_contents_equal: bool | None


def plan_config_reconciliation(observed: ConfigObservation) -> ConfigAction:
    # 직접 구현
    ...
```

#### 입력과 출력

- `web_entry_kind`, `config_entry_kind`는 `missing`, `regular`, `symlink`, `directory`, `other` 중 하나다.
- web symlink의 허용 대상은 호출자가 정한 전용 설정 경로 하나뿐이라고 가정한다.
- 반환값은 실행할 작업의 종류이며, 실제 side effect는 만들지 않는다.

#### 반드시 만족해야 할 조건

- 완료 marker가 있는 상태에서는 문법 오류, 비정상 파일 종류, DB identity/password 드리프트를 자동 복구하지 않고 `REFUSE`한다.
- 신규 상태에서 두 위치가 모두 비어 있으면 `CREATE_NEW`가 가능하다.
- web에만 안전한 일반 설정 파일이 있고 전용 설정 경로가 비어 있으면 `MIGRATE_LEGACY`가 가능하다.
- 두 위치 모두 파일이 있으면 동일성이 입증되지 않는 한 하나를 덮어쓰지 않는다.
- 설정 파일이 정상이고 링크만 빠졌거나 잘못되었을 때만 `REPAIR_LINK`를 선택한다.
- DB identity와 password가 일치하고 URL만 다를 때만 `UPDATE_URL_ONLY`를 선택할 수 있다.
- 알 수 없는 `kind` 값은 오류로 처리한다.

#### 경계 조건

- marker는 없지만 DB 설치가 이미 존재할 수 있는 부분 완료 상태
- web 경로가 dangling symlink인 경우
- web 링크가 전용 설정 파일이 아닌 다른 정상 파일을 가리키는 경우
- 설정 파일 문법은 맞지만 DB 이름·사용자·호스트 중 하나가 다른 경우
- 비밀번호만 다른 경우
- 레거시 파일과 전용 파일의 내용 비교 결과를 얻지 못한 경우

#### 실패 조건과 제약

- `REFUSE`를 반환해야 할 상태를 자동 수리 상태로 분류하면 안 된다.
- 함수는 파일을 열거나 외부 명령을 실행하지 않는다.
- 작업 우선순위가 입력 필드 순서에 의존해서는 안 된다.
- 반환값만 보고 호출자가 destructive action을 실행할 수 있으므로 모호한 상태는 보수적으로 거부한다.

### 구현 후 자가 검증

- [ ] 신규 상태는 `CREATE_NEW`로 분류된다.
- [ ] web 일반 파일만 있는 레거시 상태는 안전 조건을 만족할 때만 `MIGRATE_LEGACY`가 된다.
- [ ] 완료 marker와 자격증명 드리프트가 함께 있으면 `REFUSE`한다.
- [ ] URL만 다른 정상 상태는 `UPDATE_URL_ONLY`가 된다.
- [ ] 정상 설정과 올바른 링크, 일치하는 값은 `KEEP`이다.
- [ ] dangling symlink, 디렉터리, 알 수 없는 파일 종류를 거부한다.
- [ ] 두 설정 복사본이 다르거나 비교할 수 없으면 어느 쪽도 덮어쓰지 않는다.
- [ ] 모든 boolean·kind 조합 중 반환하지 못하는 경로가 없는지 확인한다.
- [ ] 분류 함수는 입력을 변경하지 않고 side effect가 없다.

### 구현 후 설명할 것

1. idempotent 실행과 상태 reconciliation이 같은 말이 아닌 이유
2. 완료 marker가 있어도 실제 설정·사용자 인증을 다시 확인해야 하는 이유
3. URL 변경과 자격증명 변경을 다른 정책으로 분류한 이유
4. 설정 파일과 사용자 콘텐츠의 수명을 분리한 이유
5. 모호한 상태에서 자동 수리보다 거부를 선택한 기준

### 원본 확인 위치

- Thread 05
- 커밋: `fix(init): 중단된 단계별 초기화를 수렴`
- 파일: `srcs/requirements/wordpress/tools/docker-entrypoint.sh`
- 함수·컴포넌트: `prepare_config_location`, `publish_config_link`, `write_wordpress_config`, `config_value`, `validate_wordpress_config`, `update_config_urls`, `converge_wordpress_config`, `install_wordpress`, `ensure_author`, `verify_user_password`, `runtime`, `bootstrap`, `install_core_files`, `install_content_files`
- 관련 파일: `srcs/docker-compose.yml`
- 관련 Thread: 02, 03, 06, 09, 10, 13

---

<a id="a-01"></a>
## A-01 · [Thread 02 / `refactor(runtime): Compose 프로젝트 실행 경계 공통화`] 프로젝트 범위 오케스트레이션

### 면접 질문

각 관리 도구가 직접 `docker compose` 명령을 조합하지 않고 `ComposeProject`와 `run_action` 같은 공통 경계를 사용한 이유를 설명해 보세요. 이 추상화가 책임져야 하는 것과 책임지지 말아야 하는 것을 구분해 보세요.

꼬리 질문:

- `input_data`와 `input_stream`을 동시에 허용하면 어떤 모호성이 생깁니까?
- 모든 subprocess 호출에 타임아웃을 강제하는 이유와, 단일 기본값만 쓰기 어려운 이유는 무엇입니까?
- stale bootstrap 컨테이너를 이름만 보고 삭제하면 어떤 위험이 있습니까?
- `start`, `database`, `application` 동작을 한 함수가 처리할 때 잘못된 pause stage를 어떻게 거부해야 합니까?
- 오류 메시지를 공통 예외로 감싸되 원래 예외를 보존해야 하는 이유는 무엇입니까?

### 30초 모범 답변

공통 실행 경계는 프로젝트 이름, env·compose 파일, 작업 디렉터리, 타임아웃, 입력 방식, 출력 캡처, 오류 변환을 모든 도구에서 동일하게 강제합니다. 오케스트레이터는 DB→애플리케이션 순서와 선택한 action에 맞는 stage만 허용하고, 같은 프로젝트의 관리 작업 잠금 안에서 실행합니다. 반면 실제 서비스의 초기화 invariant는 각 entrypoint가 책임집니다. stale 컨테이너도 이름만 믿지 않고 Compose project와 bootstrap label을 확인한 뒤 제거해야 다른 프로젝트 자원을 침범하지 않습니다.

### 답변 핵심 키워드

command boundary · dependency injection · project scope · timeout · mutually exclusive input · typed config · error chaining · ownership label · stage validation · separation of concerns

### 백지 구현

#### 구현 목표

외부 명령 실행기를 주입받아 프로젝트 범위 명령을 구성하고 실행하는 최소 어댑터를 구현한다. 실제 Docker는 호출하지 않는다.

#### 인터페이스

```python
from pathlib import Path
from typing import BinaryIO, Protocol

class Runner(Protocol):
    def __call__(self, argv: list[str], **kwargs: object) -> object:
        ...

class ProjectCommandError(RuntimeError):
    pass

class ProjectCommand:
    def __init__(
        self,
        project: str,
        env_file: Path,
        compose_file: Path,
        *,
        default_timeout: int,
        runner: Runner,
    ) -> None:
        # 직접 구현
        ...

    def command(self, *arguments: str) -> list[str]:
        # 직접 구현
        ...

    def run(
        self,
        *arguments: str,
        input_data: bytes | None = None,
        input_stream: BinaryIO | None = None,
        capture: bool = False,
        timeout: int | None = None,
    ) -> object:
        # 직접 구현
        ...
```

#### 입력과 출력

- 프로젝트 이름, 존재하는 env 파일, 존재하는 Compose 파일, 기본 타임아웃을 받는다.
- `command`는 프로젝트 범위 옵션을 포함한 argv 리스트를 반환한다.
- `run`은 주입된 runner의 결과를 반환한다.

#### 반드시 만족해야 할 조건

- 프로젝트 이름은 소문자·숫자·밑줄·하이픈으로 구성된 3~63자여야 한다.
- env와 Compose 파일은 생성 시점에 존재하는 실제 경로로 확정한다.
- 기본 타임아웃과 호출별 타임아웃은 양수이며 합리적인 상한 안에 있어야 한다.
- `input_data`와 `input_stream`이 동시에 지정되면 runner를 호출하지 않고 실패한다.
- 캡처 여부와 입력 방식이 runner 인자로 정확히 전달되어야 한다.
- 타임아웃 오류는 프로젝트 맥락이 있는 `ProjectCommandError`로 변환하되 원인을 보존한다.
- shell 문자열이 아니라 argv 리스트를 만든다.

#### 경계 조건

- 존재하지 않는 파일, 디렉터리를 env 파일로 전달, symlink가 가리키는 실제 파일
- 프로젝트 이름 최소·최대 길이와 허용되지 않는 문자
- 호출별 타임아웃 0, 음수, 지나치게 큰 값
- 빈 추가 인자
- runner가 일반 OS 오류 또는 timeout을 발생시키는 경우

#### 실패 조건과 제약

- `shell=True`를 사용하지 않는다.
- 민감한 입력을 argv에 자동으로 추가하지 않는다.
- runner가 호출되기 전 검증 실패와 호출 후 실행 실패를 구분할 수 있어야 한다.
- 이 클래스 안에서 서비스별 시작 순서를 하드코딩하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 입력의 argv에 project, env file, compose file이 정확히 한 번씩 포함된다.
- [ ] 잘못된 프로젝트 이름과 타임아웃은 runner 호출 전에 거부된다.
- [ ] 두 입력 방식을 동시에 지정하면 실패한다.
- [ ] capture 설정이 runner의 stdout/stderr 정책에 반영된다.
- [ ] 호출별 타임아웃이 없으면 기본값을, 있으면 명시값을 사용한다.
- [ ] timeout 예외의 원인이 예외 체인에 남는다.
- [ ] 경로에 공백이 있어도 하나의 argv 원소로 유지된다.
- [ ] 비밀번호 같은 값이 argv에 들어가지 않는다.

### 구현 후 설명할 것

1. argv 리스트와 shell 문자열의 보안·정확성 차이
2. 외부 실행 경계와 서비스 상태 머신을 분리한 이유
3. 기본 타임아웃과 작업별 타임아웃을 함께 둔 이유
4. stale 자원 삭제 전에 label 소유권을 확인해야 하는 이유
5. 공통 예외와 원래 예외 체인을 모두 유지한 이유

### 원본 확인 위치

- Thread 02
- 커밋: `refactor(runtime): Compose 프로젝트 실행 경계 공통화`
- 파일: `tools/stack_runtime.py`
- 클래스·함수: `ComposeProject`, `command`, `run`, `config`, `running_services`
- 연관 커밋: `fix(init): 중단된 단계별 초기화를 수렴`
- 연관 파일·함수: `tools/start_stack.py`, `run_action`, `start_database`, `start_application`, `run_bootstrap`, `remove_stale_bootstrap`, `pause_arguments`
- 관련 Thread: 04, 05, 07, 08, 09, 12, 13

---

<a id="a-02"></a>
## A-02 · [Thread 03 / `test(persistence): 재시작·재생성 뒤 상태 보존 검증`] 영속성 invariant

### 면접 질문

컨테이너 `restart`와 `down/up` 재생성을 둘 다 테스트한 이유를 설명해 보세요. 데이터가 보존되었다는 사실을 어떤 관찰값으로 증명해야 하며, 볼륨 이름만 같으면 충분한지도 설명해 보세요.

꼬리 질문:

- DB 게시물, WordPress option, 업로드 파일을 각각 검증한 이유는 무엇입니까?
- 컨테이너가 재생성되어도 볼륨 식별자는 같아야 하는 이유는 무엇입니까?
- healthcheck가 성공했지만 게시물만 사라졌다면 어떤 계층의 invariant가 깨진 것입니까?
- 테스트 데이터에 nonce를 사용하는 이유는 무엇입니까?
- 영속성 테스트가 이전 실행의 잔여 데이터 때문에 거짓 양성이 되지 않게 하려면 어떻게 해야 합니까?

### 30초 모범 답변

restart는 같은 컨테이너 프로세스 수명만 바꾸지만 `down/up`은 컨테이너를 새로 만들기 때문에 서로 다른 실패를 드러냅니다. 그래서 DB에 저장된 게시물과 option, 파일 볼륨의 업로드를 각각 만들고 두 lifecycle 뒤 모두 조회합니다. 또한 프로젝트가 기대한 세 영구 볼륨을 계속 사용하며 식별자가 바뀌지 않는지 확인합니다. health만 확인하면 빈 새 설치도 정상처럼 보일 수 있으므로 실제 사용자 데이터와 저장소 identity를 함께 봐야 합니다. 매 실행마다 nonce를 써서 이전 잔여 상태와 구분합니다.

### 답변 핵심 키워드

restart vs recreation · logical data · file data · volume identity · nonce fixture · false positive · health is insufficient · cross-layer invariant

### 백지 구현

#### 구현 목표

재시작·재생성 전후의 영속 상태 스냅샷을 비교해 invariant 위반 목록을 반환하는 순수 함수를 구현한다.

#### 인터페이스

```python
from dataclasses import dataclass
from collections.abc import Mapping

@dataclass(frozen=True)
class PersistenceSnapshot:
    volumes: frozenset[str]
    database_values: Mapping[str, str]
    option_values: Mapping[str, str]
    file_hashes: Mapping[str, str]


def persistence_violations(
    before: PersistenceSnapshot,
    after_restart: PersistenceSnapshot,
    after_recreate: PersistenceSnapshot,
    *,
    expected_volume_count: int,
) -> list[str]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 세 시점의 불변 스냅샷과 기대 볼륨 개수를 받는다.
- 위반이 없으면 빈 리스트, 있으면 사람이 이해할 수 있는 독립 오류 목록을 반환한다.

#### 반드시 만족해야 할 조건

- 시작 시점의 볼륨 수가 기대값과 다르면 위반이다.
- restart와 recreate 뒤 볼륨 집합은 시작 시점과 같아야 한다.
- 세 저장소 범주에서 시작 시점에 있던 모든 key와 값이 두 시점 뒤 모두 같아야 한다.
- 추가 key가 생기는 것은 기본 계약상 허용하되, 같은 key의 값 변경·누락은 위반으로 보고한다.
- 한 범주의 첫 오류에서 중단하지 않고 가능한 위반을 모두 수집한다.
- 오류 메시지에 실제 비밀값이나 전체 콘텐츠를 그대로 포함하지 않는다.

#### 경계 조건

- 빈 저장소 맵
- 시작 시점에 중복을 표현할 수 없는 mapping 구조
- restart는 정상이나 recreate에서만 누락되는 경우
- 파일 hash만 바뀌고 DB는 같은 경우
- 볼륨 수는 같지만 식별자 하나가 교체된 경우
- after snapshot에 예상하지 못한 추가 데이터가 있는 경우

#### 실패 조건과 제약

- 함수는 네트워크나 파일시스템을 직접 조회하지 않는다.
- mapping iteration 순서에 따라 오류 순서가 달라지지 않도록 결정적 출력을 만든다.
- 실제 데이터 값을 오류 문자열에 노출하지 않는다.
- 비교 비용은 전체 key 수에 선형이어야 한다.

### 구현 후 자가 검증

- [ ] 세 스냅샷이 같으면 빈 리스트를 반환한다.
- [ ] restart에서만 값이 바뀐 경우와 recreate에서만 바뀐 경우를 구분해 보고한다.
- [ ] 볼륨 개수 불일치와 같은 개수의 식별자 교체를 모두 잡는다.
- [ ] DB·option·file 범주의 누락과 변경을 각각 보고한다.
- [ ] 추가 데이터만 있는 경우 계약대로 처리한다.
- [ ] 여러 오류가 동시에 있어도 모두 반환한다.
- [ ] 오류 메시지에 원래 값이 포함되지 않는다.
- [ ] 결과 순서가 입력 mapping의 삽입 순서에 의존하지 않는다.
- [ ] 시간 복잡도는 전체 비교 항목 수 `O(n)` 수준이다.

### 구현 후 설명할 것

1. restart와 recreate를 별도 단계로 둔 이유
2. 논리 데이터와 볼륨 identity를 함께 검증한 이유
3. hash로 파일을 비교한 이유와 충돌 가능성에 대한 판단
4. 추가 데이터 허용 여부를 계약으로 명시한 이유
5. 테스트 fixture를 nonce로 격리한 이유

### 원본 확인 위치

- Thread 03
- 커밋: `test(persistence): 재시작·재생성 뒤 상태 보존 검증`
- 파일: `tests/runtime_stack.py`
- 클래스·함수: `RuntimeStack`, `project_volumes`, `_verify_persistent_values`, `verify_persistence`
- 관련 위치: `srcs/docker-compose.yml`의 `mariadb_data`, `wordpress_data`, `wordpress_config`
- 관련 Thread: 04, 05, 07, 08
