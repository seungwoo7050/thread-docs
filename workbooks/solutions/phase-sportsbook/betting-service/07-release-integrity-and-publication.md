# 고정된 릴리스 입력, 격리 빌드, 원자적 산출물 게시

이 문서는 애플리케이션 코드 밖에서도 일반화되는 릴리스 엔지니어링 문제를 다룬다. 첫 문제는 여러 저장소의 source identity를 정확한 commit으로 고정하고 안전한 detached worktree로 구체화하는 과정이며, 두 번째는 격리된 의존성 저장소에서 완전한 generation을 만든 뒤 실패 시 기존 generation을 보존하는 게시 과정이다.

<a id="ops01-pinned-materialization"></a>
## [OPS Thread 01 / `build(source): materialize detached release worktrees`] 릴리스 manifest를 먼저 검증하고 정확한 source commit만 구체화하기

### 면접 질문

여러 서비스 저장소를 조합해 릴리스할 때 branch 이름만 기록하지 않고 `logical-name | branch | 40자 commit | artifact` manifest를 만들고, 각 commit을 detached worktree로 구체화한 이유는 무엇입니까? manifest 전체를 side effect 전에 preflight하고 실패 시 생성한 worktree를 역순으로 정리한 이유도 설명해 보세요.

꼬리 질문:

- branch tip이 manifest commit과 달라졌으면 commit object가 로컬에 있어도 왜 실패해야 합니까?
- detached checkout이 일반 branch checkout보다 릴리스 입력 증명에 유리한 이유는 무엇입니까?
- target이 `/`, repository root, 상위 디렉터리, symlink인 경우 왜 거부해야 합니까?
- cleanup 전에 모든 worktree의 owner·HEAD·dirty 상태를 먼저 확인하는 이유는 무엇입니까?
- 일부 manifest row만 읽고 build를 시작하면 어떤 partial release가 생깁니까?
- local branch가 없고 remote archive ref만 있는 경우 어떤 ref resolution 정책이 필요합니까?
- orchestration repository 자신을 manifest에 넣지 않아 lock cycle을 피한 이유는 무엇입니까?
- script의 현재 작업 디렉터리가 아니라 script path에서 repository root를 찾은 이유는 무엇입니까?

### 30초 모범 답변

branch는 움직이는 이름이므로 재현 가능한 릴리스 입력이 아닙니다. manifest에 정확한 commit과 기대 artifact를 고정하고, branch 또는 archive ref가 여전히 그 commit을 가리키는지 확인한 뒤 detached worktree를 만듭니다. manifest 형식과 전체 서비스 수를 먼저 검증해 partial build를 막고, target 경로·symlink·기존 worktree의 소유권을 확인해 cleanup이 사용자 파일을 지우지 않게 합니다. 중간 실패는 trap으로 이번 실행이 만든 경로만 역순 정리해 원자적인 source materialization처럼 동작시킵니다.

### 답변 핵심 키워드

`immutable release input`, `exact commit`, `detached worktree`, `manifest preflight`, `fail closed`, `owned cleanup`, `symlink defense`, `partial side effect`, `moving branch`

### 백지 구현

**구현 목표**

manifest 문자열을 검증해 typed entry 목록으로 만들고, 모든 entry와 target이 안전하다고 확인된 뒤에만 materializer가 checkout을 시작하도록 하는 Python 함수를 구현한다. 실제 Git 명령은 주입된 인터페이스로 대체한다.

**인터페이스 또는 함수 시그니처**

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Protocol

@dataclass(frozen=True)
class Entry:
    logical: str
    branch: str
    commit: str
    artifact: str

class Git(Protocol):
    def is_commit(self, commit: str) -> bool: ...
    def resolve_branch(self, branch: str) -> str | None: ...
    def add_detached_worktree(self, path: Path, commit: str) -> None: ...
    def remove_worktree(self, path: Path) -> None: ...


def parse_manifest(text: str, expected_logicals: set[str]) -> list[Entry]:
    # 직접 구현
    ...


def materialize(entries: list[Entry], target: Path, repo_root: Path, git: Git) -> None:
    # 직접 구현
    ...
```

**입력과 출력**

- 입력: manifest text, 기대 logical 집합, target, repository root, Git adapter
- 출력: 성공 시 `target/<logical>` detached worktree 집합, 실패 시 이번 실행이 만든 경로 없음

**반드시 만족해야 할 조건**

- 주석·빈 줄을 제외한 각 row는 정확히 4개 field다.
- commit은 lowercase 40자 hex다.
- logical 이름이 중복되지 않고 기대 집합과 정확히 같다.
- artifact 이름과 branch가 빈 값이 아니다.
- 모든 entry 검증을 끝내기 전 worktree를 만들지 않는다.
- target은 root·repo root·repo parent가 아니고 symlink가 아니며 기존 경로가 없어야 한다.
- 각 commit object가 존재하고 branch 또는 허용된 remote ref가 정확히 그 commit을 가리킨다.
- checkout 뒤 HEAD가 commit과 같고 detached인지 확인할 수 있어야 한다.
- 실패 시 생성한 worktree만 역순 제거한다.
- cleanup 실패가 원래 오류를 숨기지 않도록 정책을 정한다.

**경계 조건**

- 빈 manifest
- row field 부족·초과
- duplicate logical
- 기대 서비스 누락·추가
- uppercase·짧은 commit
- branch ref 없음
- branch tip mismatch
- target symlink
- 두 번째 worktree 생성 중 실패
- cleanup 중 한 경로 제거 실패

**실패 조건**

검증 실패는 side effect 전에 종료한다. materialization 중 실패하면 완성된 것처럼 target을 남기지 않는다. 안전을 확인할 수 없는 cleanup target은 삭제하지 않는다.

**제약**

실제 subprocess 호출은 구현하지 않아도 된다. `Git` fake로 호출 순서와 rollback을 테스트한다.

### 구현 후 자가 검증

- [ ] valid manifest가 기대 순서의 typed entries로 변환된다.
- [ ] 서비스 누락·중복·추가가 build 전에 거부된다.
- [ ] commit 형식과 object type이 검증된다.
- [ ] branch tip mismatch가 거부된다.
- [ ] 위험한 target과 symlink가 거부된다.
- [ ] 첫 side effect는 모든 preflight가 끝난 뒤 발생한다.
- [ ] 중간 실패 시 이번 실행의 worktree만 역순 제거된다.
- [ ] 외부에서 이미 있던 경로는 삭제하지 않는다.
- [ ] 호출 위치가 repository 밖이어도 root resolution 계약이 유지된다.

### 구현 후 설명할 것

1. branch와 commit을 모두 manifest에 둔 이유
2. preflight-all-before-side-effects 원칙
3. detached worktree가 source provenance를 명확히 하는 방식
4. cleanup ownership 검증과 symlink 방어의 보안 의미
5. rollback이 최선 노력이어도 기존 사용자 상태를 지우지 않는 것이 우선인 이유

### 원본 확인 위치

- Thread: OPS 01, 릴리스 입력 잠금과 소스 구체화
- 커밋: `build(lock): pin service release inputs`, `build(source): materialize detached release worktrees`, `build(source): resolve repository from script path`, `build(source): preflight cleanup ownership`, `fix(source): resolve archive remote refs`, `test(lock): reject partial manifest consumption`
- 파일: `services.lock`, `scripts/materialize-sources.sh`, `tests/test_services_lock.py`, `tests/test_materializer.py`, `tests/test_release_lock_fail_closed.py`, `tests/test_materializer_remote_refs.py`
- 함수·컴포넌트: manifest entry parser, `materialize`, `cleanup`, target validation, branch-ref resolution
- 관련 Thread: OPS 02, NT-20

---

<a id="ops02-atomic-publication"></a>
## [OPS Thread 02 / `build(jars): stage exact release artifacts atomically`] 격리된 빌드와 generation 단위 원자적 게시

### 면접 질문

각 서비스 JAR를 현재 `docker/jars` 디렉터리에 바로 덮어쓰지 않고 새 generation 디렉터리에 모두 stage한 뒤 검증하고 managed symlink를 한 번에 교체한 이유는 무엇입니까? 빌드에 격리된 Maven repository와 exact artifact 검증을 사용한 이유도 설명해 보세요.

꼬리 질문:

- 7개 중 6개만 새 버전이고 하나는 이전 버전인 혼합 generation이 왜 위험합니까?
- 실패한 rebuild가 active generation과 checksum을 그대로 보존해야 하는 이유는 무엇입니까?
- source가 detached이지만 dirty하면 왜 빌드하면 안 됩니까?
- 기대 파일명만 존재하면 충분하지 않고 executable JAR 구조를 확인한 이유는 무엇입니까?
- `SHA256SUMS`를 generation 안에 함께 두는 이유는 무엇입니까?
- symlink가 관리 영역 밖을 가리키는 경우 교체·삭제를 거부해야 하는 이유는 무엇입니까?
- 새 symlink를 임시 이름으로 만든 뒤 rename하는 이유는 무엇입니까?
- Linux와 macOS의 `mv` 차이를 감싼 이유는 무엇입니까?
- 이전 generation은 언제 삭제해야 합니까?
- protocol artifact를 먼저 격리 Maven repository에 설치하는 단계가 downstream build의 재현성에 어떤 역할을 합니까?

### 30초 모범 답변

여러 서비스 산출물은 하나의 호환 가능한 release set이므로 파일별 덮어쓰기를 하면 중간에 혼합 generation이 노출됩니다. 고정된 detached·clean source를 격리 Maven repo로 빌드하고, exact executable JAR와 전체 개수·checksum을 새 generation에서 검증한 뒤 symlink rename으로 active set을 한 번에 전환합니다. 어떤 빌드나 검증이 실패해도 새 staging만 지우고 기존 symlink와 파일은 건드리지 않습니다. 이전 generation은 전환 성공 후에만 삭제해 rollback 가능한 경계를 유지합니다.

### 답변 핵심 키워드

`hermetic build`, `isolated dependency repository`, `complete generation`, `atomic symlink swap`, `checksum manifest`, `fail-safe publication`, `dirty source rejection`, `managed path ownership`

### 백지 구현

**구현 목표**

filesystem adapter를 사용해 완전한 artifact generation을 staging하고 검증한 뒤, active link를 원자적으로 새 generation으로 교체한다. 실패 시 기존 active generation은 바뀌지 않아야 한다.

**인터페이스 또는 함수 시그니처**

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Protocol

@dataclass(frozen=True)
class Artifact:
    service: str
    source: Path
    expected_name: str

class BuildSystem(Protocol):
    def build(self, artifact: Artifact, isolated_repo: Path) -> Path: ...
    def is_executable_jar(self, path: Path) -> bool: ...

class FileSystem(Protocol):
    def make_generation(self, root: Path) -> Path: ...
    def copy(self, source: Path, target: Path) -> None: ...
    def sha256(self, path: Path) -> str: ...
    def atomic_link_swap(self, link: Path, new_target: Path) -> Path | None: ...
    def remove_tree(self, path: Path) -> None: ...


def publish_generation(
    artifacts: list[Artifact],
    isolated_repo: Path,
    generations_root: Path,
    active_link: Path,
    builder: BuildSystem,
    fs: FileSystem,
) -> Path:
    # 직접 구현
    ...
```

**입력과 출력**

- 입력: 기대 artifact 목록, 격리 dependency repo, generation root, active link, build·filesystem adapter
- 출력: 새 active generation 경로

**반드시 만족해야 할 조건**

- artifact logical 이름이 중복되지 않고 기대 개수와 일치한다.
- source는 사전에 pinned·detached·clean으로 검증됐다고 가정하거나 함수 안에서 검증한다.
- 모든 build는 동일한 isolated dependency repo를 사용한다.
- built path는 exact expected name의 regular file이며 symlink가 아니다.
- 각 JAR는 executable 구조 검증을 통과한다.
- staging generation에는 모든 서비스 JAR와 checksum manifest가 있어야 한다.
- complete 검증 전 active link를 바꾸지 않는다.
- active link는 없거나 managed generation만 가리켜야 한다.
- link 교체는 원자적이어야 한다.
- 교체 성공 전에는 old generation을 삭제하지 않는다.
- 실패 시 staging과 임시 link만 정리하고 기존 active generation은 보존한다.

**경계 조건**

- 첫 publish로 active link가 없음
- 한 서비스 build 실패
- 잘못된 artifact 이름
- non-executable JAR
- checksum 생성 실패
- incomplete artifact set
- active path가 regular directory
- active symlink가 managed root 밖을 가리킴
- atomic swap 직전 실패
- swap 성공 뒤 old generation cleanup 실패

**실패 조건**

불완전하거나 검증되지 않은 generation을 active로 노출하지 않는다. swap 뒤 old cleanup이 실패한 경우 새 generation을 되돌릴지, orphan을 남기고 경고할지 정책을 명시한다.

**제약**

실제 Maven·JAR·OS 명령은 adapter로 대체한다. 핵심은 호출 순서, complete-set invariant, rollback 경계다.

### 구현 후 자가 검증

- [ ] 모든 artifact가 성공한 뒤에만 active link가 바뀐다.
- [ ] 한 build 실패 시 active link와 기존 checksum이 그대로다.
- [ ] incomplete generation이 노출되지 않는다.
- [ ] exact 파일명과 executable JAR를 모두 검증한다.
- [ ] checksum manifest가 실제 staged bytes와 일치한다.
- [ ] active link가 managed root 밖이면 거부한다.
- [ ] swap은 임시 link와 atomic rename으로 한 번에 보인다.
- [ ] old generation은 swap 성공 뒤에만 삭제 대상이 된다.
- [ ] staging과 임시 link가 실패 경로에서 정리된다.
- [ ] 격리 repo가 host cache 오염과 우연한 dependency 사용을 줄이는 이유를 설명할 수 있다.

### 구현 후 설명할 것

1. artifact 파일별 atomicity가 아니라 generation-level atomicity가 필요한 이유
2. hermetic build가 완전한 hermeticity는 아니어도 재현성을 높이는 방식
3. checksum, exact filename, executable structure가 각각 잡는 실패 유형
4. symlink swap을 선택한 이유와 platform 차이
5. publish 성공과 old cleanup 사이의 장애에서 선택한 정책

### 원본 확인 위치

- Thread: OPS 02, 격리 빌드와 원자적 산출물 게시
- 커밋: `build(shared): install the locked protocol artifact`, `build(shared): canonicalize release inputs`, `build(jars): stage exact release artifacts atomically`, `test(jars): verify complete release generation`, `test(jars): preserve atomic generation on failure`
- 파일: `scripts/install-shared.sh`, `scripts/stage-release-jars.sh`, `tests/test_jar_staging_completeness.py`, `tests/test_jar_staging_atomicity.py`, `tests/staging_fixture.py`
- 함수·컴포넌트: shared artifact install/verification, staging generation creation, checksum generation, managed symlink swap, failure cleanup
- 관련 Thread: OPS 01, OPS 04, NT-20
