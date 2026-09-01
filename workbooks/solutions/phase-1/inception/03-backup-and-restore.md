# 일관된 백업과 새 프로젝트 복원

이 문서는 MariaDB와 WordPress 파일을 하나의 검증 가능한 백업 세트로 만들고, 적대적 입력을 거부한 뒤 새 프로젝트에 복원하는 흐름을 다룬다. 핵심은 "파일 두 개를 복사한다"가 아니라, 여러 자원의 시점·소유권·무결성·부분 실패를 하나의 상태 전이로 다루는 것이다.

---

<a id="s-05"></a>
## S-05 · [Thread 07 / `feat(backup): DB 덤프와 WordPress 볼륨 수집` + `feat(backup): 백업 세트를 원자적으로 게시`] 일관된 백업 트랜잭션

### 면접 질문

MariaDB dump와 WordPress 파일 archive를 같은 백업 세트로 묶을 때 어떤 시점 일관성 문제가 생깁니까? 이 프로젝트에서 nginx와 WordPress를 멈추고 MariaDB는 동작시킨 채 `--single-transaction` dump를 만든 이유와 trade-off를 설명해 보세요.

꼬리 질문:

- 세 서비스가 모두 실행 중인지 정확한 집합으로 확인한 이유는 무엇입니까?
  - 모범답변: `mariadb`, `wordpress`, `nginx` 중 하나가 빠지면 이미 비정상 상태일 수 있고, 예상 밖 writer 서비스가 있으면 쓰기 정지 범위가 불완전할 수 있습니다. 원본은 부분 집합 검사가 아니라 실행 서비스 집합의 정확한 동등성을 확인해 백업 시작 상태를 고정합니다.
- WordPress를 멈췄는데도 MariaDB dump가 반드시 같은 논리 시점이라고 말할 수 있는 근거는 무엇입니까?
  - 모범답변: nginx를 먼저 막아 새 요청을 차단하고 WordPress 프로세스도 중지해 애플리케이션 writer를 quiesce한 뒤 dump를 시작합니다. 그 이후 DB를 바꾸는 다른 writer가 없다는 운영 계약과 `--single-transaction`의 DB 내부 일관 snapshot을 결합한 것입니다. 외부 DB writer가 있다면 이 근거는 성립하지 않습니다.
- dump·archive·manifest를 최종 디렉터리에 직접 쓰면 어떤 실패 상태가 노출됩니까?
  - 모범답변: dump만 있거나 archive가 잘렸거나 checksum이 맞지 않는 manifest가 최종 백업처럼 보일 수 있습니다. 원본은 private staging에 세 산출물과 검증을 모두 완료한 뒤 디렉터리 rename으로 공개해 최종 경로가 없거나 완성된 세트인 상태만 노출합니다.
- 임시 디렉터리를 최종 출력의 형제 경로로 만들어야 하는 이유는 무엇입니까?
  - 모범답변: 같은 부모 아래 만들면 같은 파일시스템에서 디렉터리 rename을 사용할 수 있어 교차 장치 복사와 부분 공개를 피합니다. 원본은 최종 경로 자체도 먼저 예약하고 inode를 기억해 경로 교체 경쟁을 확인합니다.
- 파일 `fsync`, 임시 디렉터리 `fsync`, `os.replace`, 부모 디렉터리 `fsync`는 각각 무엇을 보장합니까?
  - 모범답변: 파일 `fsync`는 각 산출물의 내용과 필요한 메타데이터를, staging 디렉터리 `fsync`는 그 파일 이름들을 지속시킵니다. `os.replace`는 검증된 staging을 최종 이름으로 한 번에 전환하고, 부모 `fsync`는 그 이름 전환이 전원 장애 뒤에도 남도록 요구합니다.
- 백업 중 실패하거나 SIGTERM을 받았을 때 서비스 재시작 실패까지 발생하면 어떤 오류를 우선 보고해야 합니까?
  - 모범답변: 백업 또는 신호 오류를 원래 원인으로 보존하면서 서비스 복구 실패를 별도 문맥으로 함께 보고해야 합니다. 원본도 원래 예외를 chain에 남기고 복구 오류를 최종 `BackupError`에 포함해, 실패 원인과 현재 서비스 위험을 둘 다 드러냅니다.
- 이 설계의 downtime을 줄이려면 어떤 대안이 있고, 일관성·복잡도 trade-off는 무엇입니까?
  - 모범답변: 스토리지 snapshot, DB replication/backup API, 애플리케이션 write lock과 변경 journal을 결합하면 중단 시간을 줄일 수 있습니다. 대신 DB와 파일 snapshot의 공통 시점 조정, snapshot lifecycle, 복원 순서가 복잡해집니다. 원본은 짧은 write 중단을 받아들이고 단순하고 검증 가능한 일관성 경계를 선택했습니다.

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
    import os

    if destination.parent != staging.parent or destination == staging:
        raise BackupCreationError("destination and staging must be distinct siblings")
    if os.path.lexists(destination):
        raise BackupCreationError("backup destination already exists")

    expected = frozenset({"mariadb", "wordpress", "nginx"})
    try:
        if operations.running_services() != expected:
            raise BackupCreationError("all and only the expected services must be running")
    except BackupCreationError:
        raise
    except BaseException as error:
        raise BackupCreationError("could not verify the initial service state") from error

    writers_may_be_stopped = False
    published = False
    original_error: BaseException | None = None
    cleanup_errors: list[BaseException] = []
    restart_errors: list[BaseException] = []
    try:
        # stop callback이 일부 writer를 멈춘 뒤 실패할 수도 있으므로 호출 전부터 복구 대상으로 본다.
        writers_may_be_stopped = True
        operations.stop_writers()
        operations.dump_database(staging / "database.sql")
        operations.archive_files(staging / "wordpress.tar.gz")
        operations.write_and_verify_manifest(staging)
        operations.publish_directory(staging, destination)
        published = True
        operations.restart_services()
        writers_may_be_stopped = False
        return
    except BaseException as error:
        original_error = error

    # 게시가 끝났다면 destination은 증거이므로 절대 제거하지 않는다.
    if not published:
        try:
            operations.remove_staging(staging)
        except BaseException as error:
            cleanup_errors.append(error)
    if writers_may_be_stopped:
        try:
            operations.restart_services()
            writers_may_be_stopped = False
        except BaseException as error:
            restart_errors.append(error)

    failure = BackupCreationError(
        "backup creation failed"
        + ("; staging cleanup also failed" if cleanup_errors else "")
        + ("; service recovery also failed" if restart_errors else "")
    )
    failure.cleanup_errors = tuple(cleanup_errors)  # type: ignore[attr-defined]
    failure.restart_errors = tuple(restart_errors)  # type: ignore[attr-defined]
    failure.published = published  # type: ignore[attr-defined]
    raise failure from original_error
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
   - 외부 요청과 WordPress writer를 멈추면 백업 중 애플리케이션 변경이 더 생기지 않는다는 계약을 세울 수 있습니다. MariaDB는 계속 실행해 `--single-transaction`으로 내부 일관 snapshot을 dump하므로 DB crash recovery 없이 짧은 write downtime만 사용합니다.
2. 임시 형제 디렉터리와 원자 rename을 사용한 이유
   - 산출물 생성과 manifest 검증 중에는 staging만 보이고, 완성된 뒤 같은 파일시스템 rename으로 최종 경로를 한 번에 공개할 수 있습니다. 형제 경로는 이 rename의 원자성과 부모 디렉터리 동기화 경계를 보장하기 위한 구조입니다.
3. publish 성공과 전체 작업 성공을 구분한 이유
   - 디렉터리가 게시된 뒤에도 서비스 재시작과 health 확인이 실패할 수 있습니다. 이때 백업 artifact는 보존해야 하지만 운영 작업은 실패로 보고해야 하므로 `published`와 전체 성공 상태를 따로 추적합니다.
4. 원래 오류와 서비스 복구 오류를 함께 보존하는 방법
   - 원래 오류를 예외 원인으로 chain하고 cleanup·restart 오류를 별도 목록이나 구조화 속성에 보존합니다. 최종 메시지는 secret이나 artifact 내용을 출력하지 않고 어느 복구 단계가 실패했는지만 알려야 합니다.
5. downtime을 줄이는 대안과 복잡도·복구 가능성 trade-off
   - 볼륨 snapshot과 DB snapshot 좌표, replica backup, 애플리케이션 변경 journal을 사용할 수 있습니다. writer 중단은 줄지만 두 저장소의 공통 시점을 기록하고 복원 시 재조정하는 코드와 운영 의존성이 늘어나므로 복구 검증 비용이 커집니다.

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
  - 모범답변: 경로를 검사한 뒤 open하기 전에 symlink나 파일이 교체될 수 있습니다. 원본은 디렉터리 fd를 먼저 열고 그 fd를 기준으로 각 파일을 `O_NOFOLLOW`로 열며, 실제 읽을 inode의 종류·소유권·모드·링크 수를 `fstat`으로 확인합니다.
- 백업 파일의 링크 수가 1보다 크면 왜 거부할 수 있습니까?
  - 모범답변: 다른 경로가 같은 inode를 공유하면 외부 주체가 백업 디렉터리 밖 이름으로 파일을 수정할 가능성과 ownership 경계가 생깁니다. 이 백업 형식은 독립된 private 파일을 요구하므로 원본은 `st_nlink == 1`로 계약을 단순화합니다.
- restore가 읽는 동안 다른 프로세스가 백업 파일을 교체하거나 수정하지 못하게 하려면 어떤 잠금이 필요합니까?
  - 모범답변: 열린 각 파일 descriptor에 non-blocking shared `flock`을 잡고 검증부터 실제 restore가 끝날 때까지 유지합니다. 협력하는 writer는 exclusive lock을 사용해야 하며, advisory lock이므로 모든 생산·변경 도구가 같은 계약을 지켜야 합니다.
- 정확한 파일 집합을 요구하는 이유는 무엇이며, 예상하지 못한 추가 파일을 무시하면 어떤 문제가 있습니까?
  - 모범답변: format version이 정의한 입력 전체를 명확히 해 잘못된 디렉터리나 공격자가 섞은 payload를 복원 대상으로 오인하지 않기 위해서입니다. 추가 파일을 무시하면 이후 기능이 그것을 새 metadata로 해석하거나 운영자가 함께 검증됐다고 착각할 수 있습니다.
- tar에서 절대 경로와 `..`만 막으면 symlink·hardlink·device 항목은 안전합니까?
  - 모범답변: 아닙니다. 안전한 상대 이름이어도 symlink와 hardlink는 추출 경로를 다른 대상으로 연결할 수 있고, FIFO·device는 추출 후 I/O 동작을 바꿉니다. 원본은 추출 전에 일반 파일과 디렉터리만 허용하고 정규화 경로 중복도 거부합니다.
- manifest의 크기와 SHA-256을 둘 다 확인하는 이유는 무엇입니까?
  - 모범답변: 크기는 잘림·추가 바이트를 빠르고 명시적으로 검출하고 예상 전송량을 고정합니다. digest는 같은 크기의 내용 변경을 검출합니다. 둘을 함께 두면 오류 설명과 무결성 계약이 분명해지며, 어느 하나라도 다르면 side effect 전에 거부합니다.

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
    import re
    from pathlib import PurePosixPath

    expected_files = {"manifest.json", "database.sql", "wordpress.tar.gz"}
    data_files = {"database.sql", "wordpress.tar.gz"}
    errors: list[str] = []

    by_name: dict[str, FileObservation] = {}
    for observed in files:
        if observed.name in by_name:
            errors.append(f"duplicate backup file: {observed.name}")
            continue
        by_name[observed.name] = observed
    present = set(by_name)
    for name in sorted(expected_files - present):
        errors.append(f"missing backup file: {name}")
    for name in sorted(present - expected_files):
        errors.append(f"unexpected backup file: {name}")

    digest_pattern = re.compile(r"[0-9a-f]{64}")
    for name in sorted(expected_files & present):
        observed = by_name[name]
        if observed.kind != "regular":
            errors.append(f"backup entry is not regular: {name}")
        if observed.owner_uid != expected_uid:
            errors.append(f"backup entry owner differs: {name}")
        if observed.mode & 0o077:
            errors.append(f"backup entry is not private: {name}")
        if observed.link_count != 1:
            errors.append(f"backup entry link count differs: {name}")
        if observed.size < 0:
            errors.append(f"backup entry size is negative: {name}")

    if set(manifest) != data_files:
        for name in sorted(data_files - set(manifest)):
            errors.append(f"manifest entry missing: {name}")
        for name in sorted(set(manifest) - data_files):
            errors.append(f"manifest entry unexpected: {name}")
    for name in sorted(data_files & set(manifest)):
        declared = manifest[name]
        if declared.size < 0 or digest_pattern.fullmatch(declared.sha256) is None:
            errors.append(f"manifest entry format invalid: {name}")
            continue
        actual = by_name.get(name)
        if actual is None:
            continue
        if digest_pattern.fullmatch(actual.sha256) is None:
            errors.append(f"observed digest format invalid: {name}")
        if actual.size != declared.size:
            errors.append(f"backup size mismatch: {name}")
        if actual.sha256 != declared.sha256:
            errors.append(f"backup digest mismatch: {name}")

    if not tar_members:
        errors.append("wordpress archive is empty")
    seen_members: set[str] = set()
    for member in tar_members:
        path = PurePosixPath(member.name)
        normalized = path.as_posix()
        if (
            not member.name
            or path.is_absolute()
            or ".." in path.parts
            or normalized in {"", "."}
        ):
            errors.append(f"unsafe archive path: {member.name}")
            continue
        if normalized in seen_members:
            errors.append(f"duplicate archive path: {normalized}")
        else:
            seen_members.add(normalized)
        if member.kind not in {"regular", "directory"}:
            errors.append(f"unsupported archive member: {normalized}")

    if errors:
        raise BackupValidationError("; ".join(sorted(errors)))
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
   - `resolve()`는 한 시점의 경로 해석 결과일 뿐 이후 open 대상이 같다는 보장이 없습니다. 디렉터리 fd 상대 open과 `fstat`은 검증할 metadata와 실제로 hash·복원에 읽는 inode를 같은 descriptor에 묶습니다.
2. shared lock이 검증과 실제 복원 사이의 변경을 어떻게 줄이는지
   - 검증 후 descriptor를 닫지 않고 shared lock을 유지하면 같은 계약의 writer가 exclusive lock을 얻지 못해 내용을 바꿀 수 없습니다. 다만 advisory이므로 비협력 writer까지 강제로 막지는 못하며 private directory와 ownership 검사가 함께 필요합니다.
3. exact file allowlist를 사용한 이유
   - manifest, DB dump, WordPress archive라는 format 경계를 정확히 고정해 다른 디렉터리나 미래·악성 항목을 현재 버전이 조용히 해석하지 않게 합니다. 누락과 추가를 모두 오류로 만들면 검증 성공의 의미가 명확합니다.
4. tar 경로뿐 아니라 member type과 중복을 검사한 이유
   - 상대 경로여도 link나 device는 추출 의미가 위험하고, `a`와 `./a` 같은 중복은 뒤 항목이 앞 항목을 덮어쓰게 할 수 있습니다. 추출 전에 정규 경로의 유일성과 일반 파일·디렉터리 종류를 함께 제한해야 합니다.
5. 크기와 digest를 모두 manifest 계약에 넣은 이유
   - 크기는 truncation과 예상 복원량을 명시하고 digest는 바이트 내용의 동일성을 확인합니다. 두 값은 전송·진단에서 서로 다른 오류를 빠르게 구분하며, digest 형식까지 고정해야 느슨한 parser 차이를 피할 수 있습니다.

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
  - 모범답변: 맞습니다. 그래서 원본은 project label 조회와 별도로 렌더링된 예상 볼륨·네트워크 이름, 기본·명시적 컨테이너 이름 전체를 Docker의 전체 name 목록과 교차합니다. label 없는 수동 자원도 이름이 같으면 복원을 거부합니다.
- 반대로 이름만 보고 기존 자원을 삭제하면 어떤 위험이 있습니까?
  - 모범답변: 이름은 소유권 증명이 아니어서 다른 사용자가 만든 정상 자원을 지울 수 있습니다. 이 프로젝트의 preflight는 충돌을 보고할 뿐 삭제하지 않고, 실패 cleanup도 이번 시도와 project 범위를 증명할 수 있는 자원만 대상으로 합니다.
- 렌더링된 Compose 모델에서 자원 이름을 계산해야 하는 이유는 무엇입니까?
  - 모범답변: env interpolation, project name, 명시적 `name`과 Compose 기본 naming 규칙이 적용된 실제 이름은 YAML 원문만 봐서는 정확하지 않을 수 있습니다. 원본은 `docker compose config --format json`의 결과에서 볼륨·네트워크 이름을 읽습니다.
- 사전 검사와 실제 생성 사이에 다른 프로세스가 자원을 만들 수 있는 경쟁은 어떻게 다뤄야 합니까?
  - 모범답변: 같은 관리 도구끼리는 project operation lock으로 직렬화합니다. 외부 Docker 사용자는 그 lock을 따르지 않을 수 있으므로 생성 시 name conflict도 정상 실패로 처리하고, 이번 시도에서 생성된 범위만 cleanup한 뒤 원래 충돌을 보고합니다.
- 기존 프로젝트에 복원하는 기능이 필요하다면 어떤 추가 안전장치가 필요합니까?
  - 모범답변: 명시적 사용자 승인, 복원 전 현재 상태 백업, 서비스 완전 중지, 대상 project·volume identity 확인, 교체 대상 ledger, 단계별 검증과 되돌리기 계획이 필요합니다. in-place merge인지 전체 교체인지도 계약으로 분리해야 합니다.

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
    expected_by_kind = {
        "container": expected.containers,
        "volume": expected.volumes,
        "network": expected.networks,
    }
    labelled_by_kind = {
        "container": observed.project_labeled_containers,
        "volume": observed.project_labeled_volumes,
        "network": observed.project_labeled_networks,
    }
    all_names_by_kind = {
        "container": observed.all_container_names,
        "volume": observed.all_volume_names,
        "network": observed.all_network_names,
    }

    if not any(expected_by_kind.values()):
        return ["configuration: all expected resource sets are empty"]

    conflicts: list[str] = []
    for kind in sorted(expected_by_kind):
        # label orphan과 label 없는 이름 충돌을 합쳐 같은 자원의 중복 보고를 제거한다.
        names = set(labelled_by_kind[kind])
        names.update(expected_by_kind[kind].intersection(all_names_by_kind[kind]))
        conflicts.extend(f"{kind}: {name}" for name in sorted(names))
    return conflicts
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
   - 기존 자원에는 사용자 데이터가 있고 현재 백업과의 관계를 증명할 수 없습니다. 자동 삭제하면 rollback이 복원 전 상태를 되살릴 근거도 없어지므로, 원본은 fresh-target 위반을 side effect 전에 명시적으로 거부합니다.
2. label 기반 소유권과 이름 충돌 검사를 함께 한 이유
   - label 조회는 프로젝트의 orphan 자원을 찾지만 label 없는 수동 자원과 충돌 이름을 놓칠 수 있습니다. 전체 이름 검사는 충돌을 잡지만 소유권을 증명하지 못하므로, 두 관찰을 합쳐 삭제 없이 보수적으로 거부합니다.
3. rendered Compose model에서 이름을 얻는 이유
   - 환경 변수와 project scope가 적용된 최종 이름이 Docker가 실제 생성할 이름입니다. 렌더링 모델을 사용해야 사용자 지정 `name`과 Compose의 naming 결과를 preflight가 정확히 반영합니다.
4. preflight 이후 TOCTOU 경쟁을 operation lock과 생성 실패로 다루는 방법
   - project lock은 협력하는 관리 명령 사이의 검사→생성 구간을 직렬화합니다. 외부 생성 경쟁은 없앨 수 없으므로 Docker의 충돌 오류를 실패로 받아들이고, 생성 ledger 범위만 되돌린 뒤 재시도하게 합니다.
5. overwrite restore를 지원하려면 추가로 필요한 backup·승인·rollback 정책
   - 기존 자원의 identity와 소유권을 확정하고, 변경 전 백업과 검증 가능한 복귀 지점을 만든 뒤, 사용자의 명시적 destructive 승인을 받아야 합니다. 교체 단계별 ledger, 서비스 중지, 실패 보상과 최종 데이터 검증도 필요합니다.

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
  - 모범답변: 비밀번호 계약에서 줄바꿈을 금지하고 UTF-8 비밀번호 뒤에 정확히 한 개의 `\n`을 붙입니다. child shell은 `IFS= read -r password`로 첫 줄만 소비하므로 남은 바이트가 그대로 SQL dump의 시작이 됩니다.
- child process가 password를 읽은 뒤 남은 stdin을 DB client에 넘기려면 어떤 조건을 지켜야 합니까?
  - 모범답변: shell이 첫 줄 외의 바이트를 미리 읽거나 command substitution으로 소비하지 않아야 하고, DB client가 같은 stdin을 상속해야 합니다. 인증 정보는 별도 0600 option file로 넘기며 client argv에는 비밀번호를 넣지 않습니다.
- `shutil.copyfileobj` 같은 stream copy에서 chunk size가 왜 중요합니까?
  - 모범답변: chunk는 peak 메모리와 syscall 횟수의 trade-off를 결정합니다. 원본은 1 MiB 단위로 복사해 백업 크기에 비례한 메모리 사용을 피하면서 너무 작은 read/write 반복도 줄입니다.
- WordPress archive를 추출할 때 대상 경로가 비어 있음을 다시 확인해야 하는 이유는 무엇입니까?
  - 모범답변: fresh-target preflight 이후 bootstrap이 볼륨에 파일을 만들었거나 외부 경쟁이 개입했을 수 있습니다. 원본의 one-off 추출 shell은 tar 실행 직전에 data와 config 디렉터리가 모두 비었는지 확인해 기존 파일과 archive가 섞이는 것을 막습니다.
- producer·consumer 중 하나가 먼저 실패하면 pipe와 subprocess cleanup을 어떻게 해야 합니까?
  - 모범답변: 반대편 stream을 닫아 block을 풀고 child의 종료 상태를 유한 timeout 안에 회수해야 합니다. partial write나 broken pipe를 성공으로 처리하지 않고, 임시 파일과 one-off 컨테이너는 `finally`·trap으로 정리하며 원래 producer/consumer 오류를 보존합니다.
- 임시 option file은 어떤 디렉터리와 모드로 만들고 언제 삭제해야 합니까?
  - 모범답변: 컨테이너의 비공개 runtime 디렉터리인 `/run` 아래에서 `umask 077`과 `mktemp`로 만들고 client가 끝나는 즉시 `EXIT HUP INT TERM` trap으로 삭제합니다. 장기 볼륨이나 host 공유 경로에는 남기지 않습니다.

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
    import hashlib
    import re

    if re.fullmatch(r"[A-Za-z0-9_.~!@#%^+=,-]{24,128}", secret) is None:
        raise StreamTransferError("secret does not satisfy the transfer format")
    if not 1 <= chunk_size <= 16 * 1024 * 1024:
        raise StreamTransferError("chunk size is outside the allowed range")

    def write_all(data: bytes | memoryview) -> None:
        view = memoryview(data)
        while view:
            written = destination.write(view)
            if written is None or written <= 0 or written > len(view):
                raise StreamTransferError("destination made no valid write progress")
            view = view[written:]

    digest = hashlib.sha256()
    transferred = 0
    try:
        write_all(secret.encode("utf-8") + b"\n")
        while True:
            chunk = source.read(chunk_size)
            if chunk == b"":
                break
            if not isinstance(chunk, (bytes, bytearray, memoryview)):
                raise StreamTransferError("source returned a non-binary chunk")
            chunk_view = memoryview(chunk)
            if len(chunk_view) > chunk_size:
                raise StreamTransferError("source exceeded the requested chunk size")
            digest.update(chunk_view)
            write_all(chunk_view)
            transferred += len(chunk_view)
    except StreamTransferError:
        raise
    except BaseException as error:
        raise StreamTransferError("prefixed stream transfer failed") from error
    return transferred, digest.hexdigest()
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
   - stdin은 비밀번호를 process argv, inspect 환경, shell history에 남기지 않고 초기화 프로세스 수명에 전달 범위를 묶습니다. 다만 수신 코드가 first-line 경계를 정확히 지켜야 하고 pipe 접근·오류 로그·core dump 같은 실행 환경 위험까지 없애지는 않습니다.
2. partial write와 backpressure를 고려한 이유
   - pipe나 custom stream의 `write`는 요청한 일부만 받아 producer에 속도 조절을 걸 수 있습니다. 남은 view를 반복 전송해야 바이트 손실이 없고, 0바이트 진행은 consumer 종료나 고장으로 보고 무한 루프를 막아야 합니다.
3. stream 소유권을 호출자에게 남긴 이유
   - source와 destination은 subprocess·파일 context의 더 긴 lifecycle 일부일 수 있습니다. helper가 닫으면 호출자가 종료 상태를 회수하거나 추가 flush·검증을 할 수 없으므로, 이 함수는 전달만 하고 close 책임을 생성한 쪽에 둡니다.
4. WordPress 대상 경로의 empty check를 추출 직전 다시 해야 하는 이유
   - preflight와 추출 사이 bootstrap 또는 외부 작업이 파일을 만들 수 있습니다. 추출 직전 검사로 기존 사용자 데이터와 archive가 합쳐지는 것을 막고 fresh-target invariant를 side effect 경계에서 다시 확인합니다.
5. 대용량 fixture로 checksum과 DB payload 길이를 검증한 이유
   - 작은 입력은 우연히 단일 read/write로 끝나 streaming·partial transfer 결함을 드러내지 못합니다. chunk보다 큰 파일의 hash와 MariaDB `LONGTEXT` 길이를 확인하면 실제 payload가 잘리거나 변형되지 않고 복원됐음을 검증할 수 있습니다.

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
  - 모범답변: 아닙니다. 전자는 DB 컨테이너·볼륨·네트워크까지만 생겼을 수 있고, 후자는 WordPress 볼륨과 one-off 컨테이너까지 추가됐을 수 있습니다. 생성 직후 ledger에 기록하면 각 실패 시점에 실제 만들어진 자원만 역순으로 정리할 수 있습니다.
- cleanup 단계 하나가 실패하면 이후 자원 정리를 중단해야 합니까?
  - 모범답변: 중단하면 더 많은 누수가 남으므로 나머지를 계속해야 합니다. 원본도 `down -v` 결과와 container·volume·network의 잔존 여부를 함께 검사하고, 축약 ledger에서는 모든 remove·verify 오류를 집계합니다.
- SIGINT가 데이터 주입 중 들어왔을 때 cleanup을 보장하려면 신호를 예외로 바꾸는 위치가 어디여야 합니까?
  - 모범답변: 프로젝트 잠금과 복원 `try/except/finally` 경계에 들어가기 전에 handler를 설치해, 자원 생성 이후 어느 단계의 신호도 같은 실패 경로로 들어오게 해야 합니다. 원본의 `operation_signal_handlers`가 SIGINT·SIGTERM을 `BackupError`로 바꿉니다.
- pre-existing 자원과 이번 시도에서 만든 자원을 어떻게 구분합니까?
  - 모범답변: 먼저 fresh-target preflight로 기존 label 자원과 이름 충돌이 없음을 증명하고, 그 뒤 생성 성공 직후 `(kind, name)`을 ledger에 추가합니다. 삭제는 그 ledger와 기대 project scope를 모두 만족하는 항목으로 제한합니다.
- cleanup이 완전히 성공했는지 재검사하는 이유는 무엇입니까?
  - 모범답변: 삭제 명령이 일부 자원을 건너뛰거나 daemon 오류 뒤 불완전하게 끝날 수 있고, 성공 코드만으로 실제 부재를 보장하지 못할 수 있습니다. 이름과 label 조회로 잔존 자원을 다시 확인해야 재시도 가능한 fresh target으로 돌아왔는지 알 수 있습니다.
- rollback 중 또 신호가 오면 즉시 중단하는 것과 지연하는 것의 trade-off는 무엇입니까?
  - 모범답변: 즉시 중단하면 사용자 의도에는 빠르게 반응하지만 자원 누수와 모호한 상태가 늘어납니다. 지연하면 cleanup 완료 가능성이 높지만 종료가 늦어집니다. 삭제 callback에 유한 timeout을 두고 추가 신호는 기록한 뒤 best-effort cleanup을 마치는 정책이 일반적으로 안전합니다.

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
        if getattr(self, "_sealed", False) or getattr(self, "_cleanup_active", False):
            raise RuntimeError("cannot record resources after cleanup has started")
        resources = getattr(self, "_resources", None)
        if resources is None:
            resources = []
            self._resources = resources
            self._keys = set()
            self._cleaned = set()
        key = (resource.kind, resource.name)
        if key in self._keys:
            raise ValueError(f"duplicate resource: {resource.kind}:{resource.name}")
        self._keys.add(key)
        resources.append(resource)

    def cleanup(self) -> None:
        resources = getattr(self, "_resources", [])
        cleaned = getattr(self, "_cleaned", set())
        self._cleaned = cleaned
        self._sealed = True
        if getattr(self, "_cleanup_active", False):
            raise RuntimeError("cleanup is already active")

        errors: list[tuple[str, str, str, BaseException]] = []
        leaks: list[tuple[str, str]] = []
        self._cleanup_active = True
        try:
            for resource in reversed(resources):
                key = (resource.kind, resource.name)
                if key in cleaned:
                    continue

                exists_before: bool | None = None
                try:
                    exists_before = resource.still_exists()
                except BaseException as error:
                    errors.append((resource.kind, resource.name, "precheck", error))

                if exists_before is not False:
                    try:
                        resource.remove()
                    except BaseException as error:
                        errors.append((resource.kind, resource.name, "remove", error))

                try:
                    remains = resource.still_exists()
                except BaseException as error:
                    errors.append((resource.kind, resource.name, "verify", error))
                    continue
                if remains:
                    leaks.append(key)
                else:
                    # 두 번째 cleanup에서는 이미 사라진 자원을 다시 삭제하지 않는다.
                    cleaned.add(key)
        finally:
            self._cleanup_active = False

        if errors or leaks:
            phases = ", ".join(
                f"{kind}:{name}:{phase}" for kind, name, phase, _ in errors
            )
            leaked = ", ".join(f"{kind}:{name}" for kind, name in leaks)
            message = "restore cleanup incomplete"
            if phases:
                message += f"; callback errors=[{phases}]"
            if leaked:
                message += f"; remaining=[{leaked}]"
            failure = CleanupIncomplete(message)
            failure.callback_errors = tuple(errors)  # type: ignore[attr-defined]
            failure.remaining_resources = tuple(leaks)  # type: ignore[attr-defined]
            raise failure
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
   - 생성과 record 사이에 다음 단계로 넘어가면 그 사이 실패 시 cleanup 대상이 누락됩니다. Docker가 자원 생성을 확정한 직후 기록해야 이후 모든 예외 지점에서 이번 시도의 소유 범위를 재구성할 수 있습니다.
2. 의존성 역순 cleanup의 근거
   - 나중에 만든 컨테이너가 앞서 만든 네트워크·볼륨을 참조합니다. 생성의 역순으로 제거하면 consumer를 먼저 없애고 기반 자원을 지워 `resource in use` 실패를 줄일 수 있습니다.
3. best-effort 전체 정리와 오류 집계 방식을 선택한 이유
   - 첫 remove 실패에서 멈추면 독립적으로 지울 수 있는 뒤 자원까지 남습니다. 모든 callback과 존재 검사를 시도하고 원래 예외 객체를 단계별로 보존하면 누수를 최소화하면서 수동 복구 정보도 제공합니다.
4. pre-existing 자원을 절대 삭제하지 않는 경계
   - fresh-target 검사 뒤 이번 실행이 생성 성공을 확인해 ledger에 기록한 `(kind, name)`만 삭제합니다. 이름 추정이나 광역 prune은 사용하지 않으며, 외부 자원은 충돌로 거부할 뿐 cleanup 범위에 넣지 않습니다.
5. 원래 restore 오류와 cleanup 오류를 최종 사용자에게 함께 전달하는 방식
   - restore 오류를 주 예외 원인으로 유지하고 `CleanupIncomplete`의 remove·verify 오류와 잔존 자원 목록을 별도 구조로 붙입니다. 메시지는 단계와 자원 identity만 포함해 최초 실패와 추가 운영 조치를 함께 판단하게 합니다.

### 원본 확인 위치

- Thread 08
- 커밋: `feat(restore): 실패한 복원 자원을 정리하고 롤백`
- 파일: `tools/stack_backup.py`
- 함수·컴포넌트: `cleanup_failed_restore`, 복원 오케스트레이션, `operation_signal_handlers`
- 테스트 관련 위치: `tests/runtime_stack.py`의 DB 복원 후 실패, SIGINT 중단, 반복 복원 거부, 정리 실패 전파 검증
- 관련 Thread: 07, 09, 13
