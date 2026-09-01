# 일관된 백업과 새 프로젝트 복원

이 문서는 MariaDB와 WordPress 파일을 하나의 검증 가능한 백업 세트로 만들고, 적대적 입력을 거부한 뒤 새 프로젝트에 복원하는 흐름을 다룬다. 핵심은 "파일 두 개를 복사한다"가 아니라, 여러 자원의 시점·소유권·무결성·부분 실패를 하나의 상태 전이로 다루는 것이다.

---

<a id="s-05"></a>
## S-05 · [Thread 07 / `feat(backup): DB 덤프와 WordPress 볼륨 수집` + `feat(backup): 백업 세트를 원자적으로 게시`] 일관된 백업 트랜잭션

### 면접 질문

MariaDB dump와 WordPress 파일 archive를 같은 백업 세트로 묶을 때 어떤 시점 일관성 문제가 생깁니까? 이 프로젝트에서 nginx와 WordPress를 멈추고 MariaDB는 동작시킨 채 `--single-transaction` dump를 만든 이유와 trade-off를 설명해 보세요.

꼬리 질문:

- 세 서비스가 모두 실행 중인지 정확한 집합으로 확인한 이유는 무엇입니까?
- WordPress를 멈췄는데도 MariaDB dump가 반드시 같은 논리 시점이라고 말할 수 있는 근거는 무엇입니까?
- dump·archive·manifest를 최종 디렉터리에 직접 쓰면 어떤 실패 상태가 노출됩니까?
- 임시 디렉터리를 최종 출력의 형제 경로로 만들어야 하는 이유는 무엇입니까?
- 파일 `fsync`, 임시 디렉터리 `fsync`, `os.replace`, 부모 디렉터리 `fsync`는 각각 무엇을 보장합니까?
- 백업 중 실패하거나 SIGTERM을 받았을 때 서비스 재시작 실패까지 발생하면 어떤 오류를 우선 보고해야 합니까?
- 이 설계의 downtime을 줄이려면 어떤 대안이 있고, 일관성·복잡도 trade-off는 무엇입니까?

### 30초 모범 답변

DB와 업로드 파일이 함께 바뀌면 따로 복사한 두 결과가 서로 다른 시점을 나타낼 수 있습니다. 그래서 시작 전 실행 서비스 집합을 확인하고 외부 요청과 WordPress writer를 멈춘 뒤, MariaDB는 일관 스냅샷 dump가 가능하도록 유지합니다. dump와 파일 archive를 private 임시 형제 디렉터리에 스트리밍하고 크기·SHA-256을 담은 manifest까지 검증한 다음 디렉터리를 원자적으로 최종 경로에 게시합니다. 실패나 신호에서는 임시 결과를 제거하고 서비스를 복구하며, 원래 오류와 복구 오류를 모두 보존합니다. 대가로 짧은 쓰기 중단이 생기지만 구현과 복원 의미가 명확해집니다.

### 답변 핵심 키워드

cross-resource consistency · quiesce writers · single-transaction dump · private staging directory · manifest · streaming hash · atomic directory publish · fsync · service recovery · bounded downtime

### 백지 구현

#### 구현 목표

실제 Docker 명령 대신 주입된 의존성을 사용해 "사전 상태 확인 → writer 정지 → 두 산출물 생성 → manifest 검증 → 원자 게시 → 서비스 복구"를 수행하는 coordinator를 구현한다.

#### 인터페이스

```python
from dataclasses import dataclass
from pathlib import Path
from collections.abc import Callable

@dataclass(frozen=True)
class BackupOperations:
    running_services: Callable[[], frozenset[str]]
    stop_writers: Callable[[], None]
    dump_database: Callable[[Path], None]
    archive_files: Callable[[Path], None]
    write_and_verify_manifest: Callable[[Path], None]
    publish_directory: Callable[[Path, Path], None]
    restart_services: Callable[[], None]
    remove_staging: Callable[[Path], None]

class BackupCreationError(RuntimeError):
    pass


def create_backup_set(
    destination: Path,
    staging: Path,
    operations: BackupOperations,
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- 최종 목적지와 같은 부모에 예약된 staging 경로, 부작용 callback 집합을 받는다.
- 성공 시 staging이 destination으로 게시되고 서비스가 다시 정상 상태여야 한다.
- 실패 시 `BackupCreationError`에 원래 오류와 정리·복구 오류 정보가 남아야 한다.

#### 반드시 만족해야 할 조건

- 시작 시 실행 서비스 집합이 정확히 `mariadb`, `wordpress`, `nginx`인지 확인한다.
- 사전 조건 실패 시 writer 정지나 파일 생성 callback을 호출하지 않는다.
- writer 정지 뒤에는 어떤 후속 단계가 실패해도 서비스 복구를 시도한다.
- DB dump와 파일 archive가 모두 성공한 뒤에만 manifest를 만들고 검증한다.
- manifest 검증이 성공한 뒤에만 최종 게시를 시도한다.
- 게시 성공 전 staging cleanup을 실행하지 않는다.
- 게시 성공 후 destination을 삭제하지 않는다.
- 원래 실패가 있어도 staging cleanup과 서비스 복구를 모두 시도한다.
- cleanup 또는 restart 오류가 발생해도 원래 오류를 잃지 않는다.
- 성공을 반환하기 전에 서비스 복구가 성공했는지 확인한다.

#### 경계 조건

- 시작 서비스가 하나 부족하거나 예상하지 못한 서비스가 추가된 경우
- stop 단계가 일부 side effect 후 실패하는 경우
- DB dump만 성공, archive만 성공, manifest 검증 실패
- publish callback이 실제 rename 후 오류를 던지는 경우
- cleanup과 restart가 동시에 실패하는 경우
- destination이 이미 존재하거나 staging이 없는 경우
- restart가 성공했지만 health 검증이 실패하는 경우

#### 실패 조건과 제약

- callback의 내부 구현을 추측해 중복 호출하지 않는다. 각 callback의 idempotence 요구를 계약으로 명시한다.
- 원래 실패를 cleanup 실패로 덮어쓰지 않는다.
- 최종 destination이 게시된 뒤 자동으로 제거하지 않는다.
- 서비스 중단 여부를 단순 boolean 하나로만 추정하지 말고, 실제로 어떤 단계가 시작·완료되었는지 추적한다.
- coordinator는 산출물 전체를 메모리에 적재하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로의 callback 순서가 사전 확인→정지→DB→파일→manifest→publish→복구다.
- [ ] 사전 조건 실패 시 side effect callback이 호출되지 않는다.
- [ ] DB·archive·manifest·publish 각각의 실패 지점에서 cleanup과 restart가 시도된다.
- [ ] publish 성공 뒤 restart 실패가 발생해도 destination은 보존되고 실패로 보고된다.
- [ ] 원래 오류, staging cleanup 오류, restart 오류가 모두 최종 예외에서 구분된다.
- [ ] writer를 멈추기 전에 dump나 archive가 시작되지 않는다.
- [ ] manifest 전에 두 산출물이 모두 존재해야 한다는 순서가 깨지지 않는다.
- [ ] 성공 반환 시 destination 게시와 서비스 정상화가 둘 다 완료되었다.
- [ ] 같은 destination 재시도 정책이 명확하다. 이 문제에서는 기존 destination을 덮어쓰지 않는다.
- [ ] callback 수와 산출물 크기에 대해 coordinator 자체의 추가 공간은 `O(1)` 수준이다.

### 구현 후 설명할 것

1. DB를 멈추지 않고 writer만 정지한 일관성 모델
2. 임시 형제 디렉터리와 원자 rename을 사용한 이유
3. publish 성공과 전체 작업 성공을 구분한 이유
4. 원래 오류와 서비스 복구 오류를 함께 보존하는 방법
5. downtime을 줄이는 대안과 복잡도·복구 가능성 trade-off

### 원본 확인 위치

- Thread 07
- 커밋: `feat(backup): 백업 무결성과 비공개 파일 I/O 정의`
- 커밋: `feat(backup): 관리 작업 신호와 테스트 중단 경계 추가`
- 커밋: `feat(backup): DB 덤프와 WordPress 볼륨 수집`
- 커밋: `feat(backup): 백업 출력 경로를 안전하게 예약`
- 커밋: `feat(backup): 백업 세트를 원자적으로 게시`
- 파일: `tools/stack_backup.py`
- 함수: `sha256_stream`, `fsync_directory`, `write_private`, `private_output`, `operation_signal_handlers`, `database_dump`, `wordpress_archive`, `normalize_backup_output`, `same_directory`, `create_backup`
- 관련 Thread: 02, 03, 08, 13

---

<a id="s-06"></a>
## S-06 · [Thread 08 / `feat(restore): 백업 입력의 형식과 체크섬 검증`] 적대적 입력 방어

### 면접 질문

복원 입력 디렉터리를 신뢰할 수 없는 데이터로 보고 어떤 검증 순서를 적용해야 하는지 설명해 보세요. 경로를 `resolve()`한 뒤 일반 파일인지 확인하는 것만으로 충분하지 않은 이유와, tar member 검증을 추출 전에 해야 하는 이유도 말해 보세요.

꼬리 질문:

- 디렉터리와 파일을 `O_NOFOLLOW`로 열고 descriptor 기준으로 검사해야 하는 이유는 무엇입니까?
- 백업 파일의 링크 수가 1보다 크면 왜 거부할 수 있습니까?
- restore가 읽는 동안 다른 프로세스가 백업 파일을 교체하거나 수정하지 못하게 하려면 어떤 잠금이 필요합니까?
- 정확한 파일 집합을 요구하는 이유는 무엇이며, 예상하지 못한 추가 파일을 무시하면 어떤 문제가 있습니까?
- tar에서 절대 경로와 `..`만 막으면 symlink·hardlink·device 항목은 안전합니까?
- manifest의 크기와 SHA-256을 둘 다 확인하는 이유는 무엇입니까?

### 30초 모범 답변

백업은 외부 입력이므로 경로 문자열이 아니라 실제로 연 descriptor를 검증합니다. 디렉터리는 symlink를 따라가지 않고 열고, 정확한 파일 집합만 허용합니다. 각 파일도 `O_NOFOLLOW`와 non-blocking으로 열어 일반 파일, 소유자, private mode, 링크 수 1을 `fstat`으로 확인하고 읽는 동안 공유 잠금을 유지합니다. manifest 구조와 선언 크기·SHA-256을 실제 stream과 비교합니다. WordPress tar는 추출 전에 비어 있지 않은지, 상대 경로인지, `..`가 없는지, 중복 경로가 없는지, 일반 파일·디렉터리만 포함하는지 검사합니다. 어느 검증도 불확실하면 복원을 시작하지 않습니다.

### 답변 핵심 키워드

untrusted input · directory fd · O_NOFOLLOW · fstat · shared lock · exact allowlist · owner/mode/link count · size + SHA-256 · pre-extraction tar validation · path traversal · special file rejection · fail before side effect

### 백지 구현

#### 구현 목표

이미 수집된 백업 파일 메타데이터, manifest, tar member 목록을 검증해 복원을 시작해도 되는지 판정하는 순수 validator를 구현한다. 파일을 실제로 여는 코드는 면접 범위에서 제외하되, descriptor 기반 수집 결과라고 가정한다.

#### 인터페이스

```python
from dataclasses import dataclass
from collections.abc import Mapping, Sequence

@dataclass(frozen=True)
class FileObservation:
    name: str
    kind: str
    owner_uid: int
    mode: int
    link_count: int
    size: int
    sha256: str

@dataclass(frozen=True)
class TarMemberObservation:
    name: str
    kind: str

@dataclass(frozen=True)
class ManifestEntry:
    size: int
    sha256: str

class BackupValidationError(RuntimeError):
    pass


def validate_backup_observations(
    files: Sequence[FileObservation],
    manifest: Mapping[str, ManifestEntry],
    tar_members: Sequence[TarMemberObservation],
    *,
    expected_uid: int,
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- 백업 디렉터리에서 descriptor 기준으로 얻은 파일 관찰값
- manifest가 선언한 파일별 크기·SHA-256
- WordPress tar의 member 이름·종류
- 문제가 없으면 반환값이 없고, 하나라도 위반하면 side effect 없이 실패한다.

#### 반드시 만족해야 할 조건

- 허용하는 백업 파일 이름 집합은 `manifest.json`, DB dump, WordPress archive의 정확한 세 개로 고정한다.
- 파일 이름 중복과 추가·누락을 모두 거부한다.
- 각 항목은 일반 파일이고, expected UID 소유이며, private mode와 링크 수 1을 만족해야 한다.
- manifest 자신을 제외한 데이터 파일의 실제 크기·digest가 선언값과 일치해야 한다.
- digest는 정해진 길이의 소문자 16진수 형식이어야 한다.
- tar는 비어 있으면 안 된다.
- tar 경로는 상대 경로이고 `..` segment가 없어야 한다.
- 정규화된 tar 경로가 중복되면 거부한다.
- tar 종류는 일반 파일과 디렉터리만 허용한다.
- 오류 메시지에는 안전한 파일·member 이름은 넣을 수 있지만 파일 내용은 넣지 않는다.

#### 경계 조건

- `a/../b`, `/abs`, `./a`, `a//b`, 빈 member 이름
- `a`와 `./a`처럼 정규화 결과가 같은 두 member
- symlink, hardlink, FIFO, block·character device tar member
- 파일 mode 0600, 0400, 0640
- link count 0, 1, 2
- manifest에 데이터 파일 하나가 없거나 추가 key가 있는 경우
- 실제 크기만 다름, digest만 다름, digest 형식이 틀림
- 같은 파일 이름 관찰값이 두 번 들어온 경우

#### 실패 조건과 제약

- 첫 위반 뒤 복원 가능한 부분을 추출하지 않는다.
- tar member 이름을 호스트 경로와 단순 문자열 결합해 안전성을 판단하지 않는다.
- symlink와 hardlink를 일반 파일로 취급하지 않는다.
- 추가 파일을 조용히 무시하지 않는다.
- validator는 실제 파일 내용을 메모리에 보유하지 않는다.

### 구현 후 자가 검증

- [ ] 정확한 세 파일과 일치하는 manifest, 안전한 tar는 통과한다.
- [ ] 백업 파일 추가·누락·중복을 모두 거부한다.
- [ ] owner·mode·kind·link count 위반을 각각 잡는다.
- [ ] 크기 또는 digest 하나만 다른 경우도 거부한다.
- [ ] 절대 경로, `..`, 중복 정규 경로를 거부한다.
- [ ] symlink·hardlink·FIFO·device tar member를 모두 거부한다.
- [ ] 빈 archive를 거부한다.
- [ ] member 순서가 바뀌어도 결과가 동일하다.
- [ ] 위반 오류에 파일 내용이나 secret 값이 나타나지 않는다.
- [ ] 전체 파일·member 수에 대해 시간 `O(n)`, 중복 검사용 공간 `O(n)`이다.

### 구현 후 설명할 것

1. `resolve()` 후 path 검증과 descriptor 기반 검증의 차이
2. shared lock이 검증과 실제 복원 사이의 변경을 어떻게 줄이는지
3. exact file allowlist를 사용한 이유
4. tar 경로뿐 아니라 member type과 중복을 검사한 이유
5. 크기와 digest를 모두 manifest 계약에 넣은 이유

### 원본 확인 위치

- Thread 08
- 커밋: `feat(restore): 백업 입력의 형식과 체크섬 검증`
- 파일: `tools/stack_backup.py`
- 클래스·함수: `VerifiedBackup`, `open_regular_file`, `load_and_verify_backup`
- 연관 커밋: Thread 07 `feat(backup): WordPress 아카이브 입력 검증`
- 연관 함수: `validate_archive_stream`, `validate_archive`
- 관련 Thread: 02, 07, 12

---

<a id="s-07"></a>
## S-07 · [Thread 08 / `feat(restore): 대상 프로젝트 자원 충돌 사전 차단`] fresh-target invariant

### 면접 질문

복원 도구가 대상 프로젝트를 자동 정리한 뒤 덮어쓰지 않고, 컨테이너·볼륨·네트워크 충돌을 모두 사전 검사해 "완전히 새 프로젝트"만 허용한 이유를 설명해 보세요.

꼬리 질문:

- Compose label로 찾은 자원만 검사하면 이름이 충돌하지만 label이 없는 자원을 놓칠 수 있지 않습니까?
- 반대로 이름만 보고 기존 자원을 삭제하면 어떤 위험이 있습니까?
- 렌더링된 Compose 모델에서 자원 이름을 계산해야 하는 이유는 무엇입니까?
- 사전 검사와 실제 생성 사이에 다른 프로세스가 자원을 만들 수 있는 경쟁은 어떻게 다뤄야 합니까?
- 기존 프로젝트에 복원하는 기능이 필요하다면 어떤 추가 안전장치가 필요합니까?

### 30초 모범 답변

복원은 여러 새 자원을 만든 뒤 데이터를 주입하므로 대상이 비어 있지 않으면 기존 상태와 백업 상태가 섞이거나 rollback이 사용자 자원을 지울 수 있습니다. 그래서 side effect 전에 렌더링된 Compose 모델로 기대 컨테이너·볼륨·네트워크 이름을 구하고, 프로젝트 label로 보이는 자원과 이름 자체가 충돌하는 자원을 모두 검사합니다. 하나라도 존재하면 자동 삭제하지 않고 거부합니다. 이후 프로젝트 operation lock 안에서 생성해 같은 도구끼리의 경쟁을 막고, Docker가 생성 시 반환하는 name conflict도 실패로 처리해 rollback 범위를 이번 시도에서 만든 자원으로 제한합니다.

### 답변 핵심 키워드

preflight · fresh target · rendered resource names · label ownership · name collision · no implicit deletion · TOCTOU window · operation lock · creation ledger

### 백지 구현

#### 구현 목표

복원 예정 자원과 현재 관찰된 Docker 자원 집합을 비교해, side effect 전에 거부해야 할 충돌을 결정적으로 반환하는 함수를 구현한다.

#### 인터페이스

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ExpectedResources:
    containers: frozenset[str]
    volumes: frozenset[str]
    networks: frozenset[str]

@dataclass(frozen=True)
class ObservedResources:
    project_labeled_containers: frozenset[str]
    project_labeled_volumes: frozenset[str]
    project_labeled_networks: frozenset[str]
    all_container_names: frozenset[str]
    all_volume_names: frozenset[str]
    all_network_names: frozenset[str]


def fresh_target_conflicts(
    expected: ExpectedResources,
    observed: ObservedResources,
) -> list[str]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 렌더링된 구성에서 얻은 기대 자원 이름과 현재 Docker 관찰값을 받는다.
- 충돌이 없으면 빈 리스트, 있으면 종류와 이름을 포함한 결정적 목록을 반환한다.

#### 반드시 만족해야 할 조건

- 해당 project label이 붙은 자원은 이름이 기대 집합에 없더라도 충돌로 본다.
- 기대 이름과 같은 unlabeled 자원도 충돌로 본다.
- container·volume·network를 모두 검사한다.
- 한 자원이 label 집합과 이름 집합 양쪽에 있어도 중복 보고하지 않는다.
- 결과 정렬은 자원 종류와 이름 기준으로 결정적이어야 한다.
- 함수는 자원을 삭제하거나 생성하지 않는다.
- 빈 기대 집합은 구성 오류로 볼지 허용할지 계약을 명확히 하고 일관되게 처리한다. 이 문제에서는 세 범주 모두 비어 있으면 구성 오류로 본다.

#### 경계 조건

- label은 맞지만 예상 이름과 다른 orphan 자원
- 이름은 같지만 label이 없는 수동 생성 자원
- 동일 문자열 이름이 서로 다른 자원 종류에 존재
- 일부 기대 집합만 빈 경우
- 관찰 결과에 예상하지 못한 project-labeled image가 있는 경우는 이 인터페이스 범위 밖임을 명시
- 대소문자와 정규화 정책

#### 실패 조건과 제약

- 충돌을 해결하기 위해 자동 삭제하지 않는다.
- label 정보만 또는 전체 이름 정보만 사용하지 않는다.
- first conflict에서 중단하지 않고 전체 목록을 반환한다.
- 실제 Docker 조회를 함수 내부에서 수행하지 않는다.

### 구현 후 자가 검증

- [ ] 완전히 빈 대상은 통과한다.
- [ ] label orphan과 unlabeled name collision을 각각 잡는다.
- [ ] container·volume·network 충돌을 모두 보고한다.
- [ ] 동일 자원의 중복 보고를 제거한다.
- [ ] 서로 다른 종류의 같은 이름은 별도 충돌로 유지한다.
- [ ] 결과 순서가 입력 set의 순서에 의존하지 않는다.
- [ ] 구성 오류와 실제 충돌을 구분한다.
- [ ] 함수가 입력 객체를 변경하지 않는다.
- [ ] 집합 크기에 대해 평균 시간 `O(n)`이다.

### 구현 후 설명할 것

1. 기존 자원 자동 삭제보다 거부를 선택한 이유
2. label 기반 소유권과 이름 충돌 검사를 함께 한 이유
3. rendered Compose model에서 이름을 얻는 이유
4. preflight 이후 TOCTOU 경쟁을 operation lock과 생성 실패로 다루는 방법
5. overwrite restore를 지원하려면 추가로 필요한 backup·승인·rollback 정책

### 원본 확인 위치

- Thread 08
- 커밋: `feat(restore): 대상 프로젝트 자원 충돌 사전 차단`
- 파일: `tools/stack_backup.py`
- 함수: `rendered_resource_names`, `existing_named_resources`, `expected_container_names`, `existing_named_containers`, `ensure_fresh_project`
- 테스트 관련 위치: `tests/runtime_stack.py`, `_verify_restore_resource_refusal`
- 관련 Thread: 02, 11, 13

---

<a id="s-08"></a>
## S-08 · [Thread 08 / `feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입`] 보안 스트리밍

### 면접 질문

큰 MariaDB dump와 WordPress archive를 복원할 때 전체 데이터를 메모리에 올리지 않고 스트리밍한 이유를 설명해 보세요. DB root 비밀번호를 command argument나 환경 변수 대신 표준 입력의 prefix와 임시 option file로 전달한 이유도 말해 보세요.

꼬리 질문:

- 비밀번호 한 줄과 dump stream을 같은 stdin에 넣을 때 경계를 어떻게 안정적으로 정의합니까?
- child process가 password를 읽은 뒤 남은 stdin을 DB client에 넘기려면 어떤 조건을 지켜야 합니까?
- `shutil.copyfileobj` 같은 stream copy에서 chunk size가 왜 중요합니까?
- WordPress archive를 추출할 때 대상 경로가 비어 있음을 다시 확인해야 하는 이유는 무엇입니까?
- producer·consumer 중 하나가 먼저 실패하면 pipe와 subprocess cleanup을 어떻게 해야 합니까?
- 임시 option file은 어떤 디렉터리와 모드로 만들고 언제 삭제해야 합니까?

### 30초 모범 답변

백업 크기는 상한이 작다고 가정할 수 없으므로 dump와 archive는 bounded chunk로 전달해 메모리 사용을 제한합니다. DB 복원 입력은 첫 줄에 검증된 root 비밀번호를 두고 그 뒤에 dump bytes를 붙입니다. 컨테이너 shell은 첫 줄만 읽어 0600 임시 option file을 만들고 trap을 설치한 뒤, 남은 stdin을 MariaDB client가 소비합니다. 그래서 비밀번호가 argv나 장기 환경에 남지 않습니다. WordPress archive는 사전 검증 후 one-off 컨테이너에서 빈 data·config 경로에만 추출합니다. timeout, partial write, consumer 조기 종료를 오류로 처리하고 모든 stream과 임시 파일을 닫습니다.

### 답변 핵심 키워드

bounded streaming · backpressure · stdin prefix protocol · argv secrecy · temporary option file · 0600 · trap cleanup · empty destination · partial write · timeout · large fixture

### 백지 구현

#### 구현 목표

짧은 secret prefix와 큰 payload stream을 하나의 binary output stream으로 제한된 메모리에서 전송하는 함수를 구현한다. 실제 subprocess는 호출하지 않는다.

#### 인터페이스

```python
from typing import BinaryIO

class StreamTransferError(RuntimeError):
    pass


def write_prefixed_stream(
    secret: str,
    source: BinaryIO,
    destination: BinaryIO,
    *,
    chunk_size: int = 1024 * 1024,
) -> tuple[int, str]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 검증할 secret 문자열, payload source, destination stream을 받는다.
- destination에는 UTF-8 secret 한 줄과 payload bytes가 순서대로 기록되어야 한다.
- 반환값은 payload 바이트 수와 payload SHA-256 digest다. secret prefix는 바이트 수·digest에 포함하지 않는다.

#### 반드시 만족해야 할 조건

- secret은 허용 문자와 길이 계약을 만족하고 줄바꿈을 포함하지 않아야 한다.
- prefix는 정확히 `secret.encode("utf-8") + b"\n"` 한 번만 기록한다.
- payload는 `chunk_size` 이하의 블록으로 읽는다.
- `destination.write`가 일부 바이트만 쓸 수 있음을 처리해야 한다.
- EOF까지 전달한 payload의 바이트 수와 digest를 반환한다.
- `chunk_size`는 양수이며 합리적인 상한 이하여야 한다.
- source나 destination을 함수가 소유하지 않으므로 닫지 않는다.
- 오류 메시지에 secret을 포함하지 않는다.

#### 경계 조건

- 빈 payload
- 1바이트 payload, 정확히 chunk size, chunk size보다 1바이트 큰 payload
- source가 한 번에 요청량보다 적게 반환
- destination이 partial write 또는 0바이트 write를 반환
- read·write 중 예외
- secret의 빈 값, 줄바꿈, 최대 길이 초과, 비ASCII 허용 여부
- digest 계산과 전송 바이트 불일치

#### 실패 조건과 제약

- payload 전체를 bytes 객체 하나로 만들지 않는다.
- `write()` 한 번이 전체 buffer를 쓴다고 가정하지 않는다.
- 전송 실패를 성공한 바이트 수로 조용히 처리하지 않는다.
- secret을 반환값, 로그, 예외 메시지에 포함하지 않는다.
- 호출자는 실패한 destination을 재사용할 수 없다고 가정하므로 partial output 가능성을 문서화한다.

### 구현 후 자가 검증

- [ ] destination의 첫 줄 뒤에 원본 payload가 byte-for-byte 이어진다.
- [ ] 반환 바이트 수와 별도 계산한 digest가 일치한다.
- [ ] 빈 payload에서도 prefix는 한 번 쓰이고 digest가 올바르다.
- [ ] source short read와 destination partial write를 처리한다.
- [ ] destination 0-byte write를 무한 루프로 만들지 않고 실패한다.
- [ ] invalid secret과 chunk size를 side effect 전에 거부한다.
- [ ] 큰 가상 stream에서 peak 메모리가 payload 크기에 비례하지 않는다.
- [ ] 오류 메시지에 secret이 없다.
- [ ] source와 destination의 close 여부를 함수가 바꾸지 않는다.
- [ ] 시간 `O(n)`, 추가 공간 `O(chunk_size)`이다.

### 구현 후 설명할 것

1. secret prefix protocol이 argv·환경 변수보다 나은 점과 남는 위험
2. partial write와 backpressure를 고려한 이유
3. stream 소유권을 호출자에게 남긴 이유
4. WordPress 대상 경로의 empty check를 추출 직전 다시 해야 하는 이유
5. 대용량 fixture로 checksum과 DB payload 길이를 검증한 이유

### 원본 확인 위치

- Thread 08
- 커밋: `feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입`
- 파일: `tools/stack_backup.py`
- 함수: `restore_database`, `restore_wordpress`
- 관련 함수: `tools/stack_runtime.py`의 `secret_payload`, `ComposeProject.run`
- 테스트 관련 위치: `tests/runtime_stack.py`의 큰 WordPress 파일 checksum과 MariaDB `LONGTEXT` 길이 검증
- 관련 Thread: 06, 07, 09

---

<a id="s-09"></a>
## S-09 · [Thread 08 / `feat(restore): 실패한 복원 자원을 정리하고 롤백`] 부분 성공 복구

### 면접 질문

복원 실패 시 `docker compose down -v` 같은 한 명령만 실행하지 않고, 이번 복원 시도에서 만들어진 자원을 추적해 정리한 이유를 설명해 보세요. 원래 오류와 cleanup 오류를 어떻게 함께 보존해야 합니까?

꼬리 질문:

- DB volume 생성 뒤 실패한 경우와 WordPress 데이터 주입 뒤 실패한 경우 cleanup 범위가 같습니까?
- cleanup 단계 하나가 실패하면 이후 자원 정리를 중단해야 합니까?
- SIGINT가 데이터 주입 중 들어왔을 때 cleanup을 보장하려면 신호를 예외로 바꾸는 위치가 어디여야 합니까?
- pre-existing 자원과 이번 시도에서 만든 자원을 어떻게 구분합니까?
- cleanup이 완전히 성공했는지 재검사하는 이유는 무엇입니까?
- rollback 중 또 신호가 오면 즉시 중단하는 것과 지연하는 것의 trade-off는 무엇입니까?

### 30초 모범 답변

복원은 네트워크·볼륨·컨테이너를 단계적으로 만들기 때문에 실패 시 일부만 존재할 수 있습니다. broad prune이나 이름 추정 삭제는 다른 프로젝트 자원을 지울 수 있으므로 fresh-target preflight 이후 이번 시도에서 실제 생성한 자원을 ledger에 기록합니다. 실패나 신호를 예외 경로로 통일하고, ledger의 자원을 의존성 역순으로 best-effort 정리합니다. 한 정리 실패가 나머지를 막지 않게 모든 오류를 모은 뒤 누수 여부를 재검사합니다. 최종 오류에는 최초 실패를 주원인으로 두고 정리 오류와 남은 자원을 함께 넣어 재시도와 수동 복구가 가능하게 합니다.

### 답변 핵심 키워드

partial success · resource ledger · reverse dependency order · best effort cleanup · no prune · ownership boundary · signal-to-exception · error aggregation · leak verification · retryability

### 백지 구현

#### 구현 목표

복원 중 생성된 자원을 기록하고, 실패 시 역순으로 모두 정리하며 오류를 집계하는 작은 resource ledger를 구현한다. 실제 Docker 삭제는 주입된 callback이 수행한다.

#### 인터페이스

```python
from dataclasses import dataclass
from collections.abc import Callable

@dataclass(frozen=True)
class CreatedResource:
    kind: str
    name: str
    remove: Callable[[], None]
    still_exists: Callable[[], bool]

class CleanupIncomplete(RuntimeError):
    pass

class RestoreLedger:
    def record(self, resource: CreatedResource) -> None:
        # 직접 구현
        ...

    def cleanup(self) -> None:
        # 직접 구현
        ...
```

#### 입력과 출력

- 생성 직후 `CreatedResource`를 순서대로 기록한다.
- `cleanup`은 기록의 역순으로 모든 `remove`를 시도하고, 남은 자원을 검사한다.
- 오류나 누수가 없으면 반환값이 없고, 있으면 모든 정보를 담은 `CleanupIncomplete`를 발생시킨다.

#### 반드시 만족해야 할 조건

- 같은 `(kind, name)`을 두 번 기록하면 side effect 없이 거부한다.
- cleanup은 역순으로 실행한다.
- 하나의 remove 실패가 이후 cleanup을 막지 않는다.
- remove가 성공했어도 `still_exists`가 참이면 누수로 보고한다.
- remove가 실패했지만 `still_exists`가 거짓이면 오류 기록 정책을 명확히 한다. 이 문제에서는 remove 오류도 보존한다.
- `still_exists` 자체의 오류도 별도로 수집한다.
- cleanup은 여러 번 호출해도 이미 정리된 자원을 다시 위험하게 삭제하지 않도록 설계한다.
- 오류 메시지에는 kind와 name, 단계는 포함하되 secret은 포함하지 않는다.
- ledger에 기록되지 않은 자원은 절대 삭제하지 않는다.

#### 경계 조건

- 빈 ledger
- 첫 remove부터 실패
- 모든 remove 성공 후 존재 재검사 하나가 실패
- 일부 자원은 이미 외부에서 삭제된 경우
- 동일 이름이 container와 volume에 각각 존재
- cleanup 호출 중 새 record를 시도하는 경우
- cleanup을 두 번 호출하는 경우
- callback이 매우 오래 걸리거나 hang할 수 있는 경우

#### 실패 조건과 제약

- `docker system prune` 같은 broad cleanup을 모델링하지 않는다.
- 첫 오류에서 return하지 않는다.
- 입력 callback의 원래 예외를 문자열 하나로만 소실하지 않는다.
- 정리 순서를 set iteration에 맡기지 않는다.
- timeout은 callback 바깥 실행 경계에서 강제해야 하며 그 이유를 설명한다.

### 구현 후 자가 검증

- [ ] 정상 cleanup은 기록 역순으로 각 remove와 존재 검사를 수행한다.
- [ ] remove 여러 개가 실패해도 모든 자원을 시도한다.
- [ ] ledger에 없는 자원 callback은 호출되지 않는다.
- [ ] 같은 이름의 다른 kind는 별도 자원으로 처리된다.
- [ ] 중복 record를 거부한다.
- [ ] remove 성공 뒤 남은 자원을 누수로 보고한다.
- [ ] 이미 없는 자원에 대한 idempotent cleanup 정책이 일관된다.
- [ ] 두 번째 cleanup 호출이 위험한 중복 삭제를 하지 않는다.
- [ ] 최종 예외에서 remove 오류, verify 오류, 남은 자원을 구분할 수 있다.
- [ ] cleanup 상태 변경이 예외 경로에서도 일관된다.

### 구현 후 설명할 것

1. 생성 즉시 ledger에 기록해야 하는 이유
2. 의존성 역순 cleanup의 근거
3. best-effort 전체 정리와 오류 집계 방식을 선택한 이유
4. pre-existing 자원을 절대 삭제하지 않는 경계
5. 원래 restore 오류와 cleanup 오류를 최종 사용자에게 함께 전달하는 방식

### 원본 확인 위치

- Thread 08
- 커밋: `feat(restore): 실패한 복원 자원을 정리하고 롤백`
- 파일: `tools/stack_backup.py`
- 함수·컴포넌트: `cleanup_failed_restore`, 복원 오케스트레이션, `operation_signal_handlers`
- 테스트 관련 위치: `tests/runtime_stack.py`의 DB 복원 후 실패, SIGINT 중단, 반복 복원 거부, 정리 실패 전파 검증
- 관련 Thread: 07, 09, 13
