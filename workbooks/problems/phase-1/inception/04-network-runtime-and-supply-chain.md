# 종단 요청 경로, 런타임 격리, 재현 가능한 공급망

이 문서는 HTTPS 요청이 실제 데이터 저장까지 이어지는지를 검증하는 방법, 여러 Compose 프로젝트가 같은 호스트에서 안전하게 공존하기 위한 런타임 정책, 그리고 이미지 빌드 입력과 런타임 산출물을 재현 가능하게 고정하는 방법을 다룬다. 세 항목 모두 설정 문법을 외우는 문제가 아니라, 어떤 경계를 어떤 증거로 검증할지 설명하는 데 초점을 둔다.

---

<a id="a-05"></a>
## A-05 · [Thread 03 / `test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증`] 종단 요청 경로 검증

### 면접 질문

`/healthz`와 PHP-FPM ping이 모두 성공해도 전체 WordPress 스택이 정상이라고 단정할 수 없는 이유는 무엇입니까? 이 프로젝트의 종단 검증이 HTTPS 요청, WordPress 쓰기, MariaDB 상태 확인을 함께 사용한 이유를 설명해 보세요.

꼬리 질문:

- nginx 상태 응답, PHP-FPM ping, WordPress 페이지 조회, DB 조회는 각각 어느 경계까지만 증명합니까?
- 테스트마다 nonce를 넣어 새 게시물을 만든 이유는 무엇입니까?
- `curl --resolve`로 도메인을 loopback 주소에 연결한 이유는 무엇입니까?
- HTTPS 포트가 이미 사용 중일 때 다른 포트로 재시도하되, 모든 시작 실패를 포트 충돌로 취급하면 왜 위험합니까?
- 포트를 잠시 예약했다가 해제한 뒤 Docker가 bind하는 사이의 race를 완전히 제거할 수 있습니까?
- 외부 페이지에 값이 보이는 것과 DB에 값이 저장된 것을 모두 확인하면 어떤 종류의 오탐을 줄일 수 있습니까?
- 테스트 실패 뒤 이전 시도의 컨테이너·네트워크·포트 상태를 정리하지 않으면 다음 재시도에 어떤 영향을 줍니까?

### 30초 모범 답변

healthcheck는 각 프로세스의 일부 기능만 확인하므로 nginx, FastCGI, WordPress bootstrap, DB 인증, 실제 쓰기·읽기 경로가 모두 이어졌다는 증거가 아닙니다. 그래서 고유한 nonce로 게시물을 생성하고 HTTPS로 같은 값을 읽은 뒤 DB에서도 결과를 확인해 요청 경로 전체를 검증합니다. 로컬 DNS에 의존하지 않도록 `curl --resolve`를 사용하고, 포트 bind 경쟁은 완전히 제거하기 어렵기 때문에 알려진 bind 충돌만 분류해 제한 횟수로 재시도합니다. 다른 시작 오류는 즉시 실패시키고, 각 실패 시도의 프로젝트 자원을 정리해 원인을 숨기지 않습니다.

### 답변 핵심 키워드

healthcheck scope · end-to-end evidence · unique nonce · write/read round trip · HTTPS termination · FastCGI · database verification · `curl --resolve` · bind race · bounded retry · failure classification · attempt cleanup

### 백지 구현

#### 구현 목표

외부 실행기를 직접 호출하지 않고, 주입된 callback으로 후보 포트를 선택해 스택을 시작하고 종단 검증을 수행한다. 오직 명확한 포트 bind 충돌만 새 포트로 재시도하고, 실패한 시도의 자원을 정리한다.

#### 인터페이스

```python
from collections.abc import Callable


class StackStartError(RuntimeError):
    pass


class EndToEndVerificationError(RuntimeError):
    pass


def start_and_verify_with_port_retry(
    *,
    reserve_candidate_port: Callable[[], int],
    start_stack: Callable[[int], None],
    verify_end_to_end: Callable[[int], None],
    cleanup_attempt: Callable[[int], None],
    is_bind_conflict: Callable[[BaseException], bool],
    max_attempts: int,
) -> int:
    # 직접 구현
    ...
```

#### 입력과 출력

- `reserve_candidate_port`는 다음 시도에 사용할 후보 포트를 반환한다.
- `start_stack`은 해당 포트로 프로젝트를 시작한다.
- `verify_end_to_end`는 고유한 데이터를 생성하고 HTTPS와 DB 양쪽의 관찰 결과를 검증한다.
- `cleanup_attempt`는 실패한 시도에서 생성된 프로젝트 자원을 제거한다.
- 성공 시 실제 사용한 포트를 반환한다.
- 재시도 불가능한 오류나 시도 한도 초과 시 원인을 보존한 예외를 발생시킨다.

#### 반드시 만족해야 할 조건

- `max_attempts`가 1보다 작은 입력은 side effect 전에 거부한다.
- 각 시도마다 새 후보 포트를 얻는다.
- 시작이 성공한 뒤에만 종단 검증을 호출한다.
- 시작 오류가 `is_bind_conflict`에서 참일 때만 재시도한다.
- 포트 충돌이 아닌 시작 오류는 추가 후보를 시도하지 않고 실패한다.
- 시작 또는 종단 검증이 실패한 시도는 cleanup을 정확히 한 번 시도한다.
- cleanup 실패가 원래 오류를 덮어쓰지 않으며 두 오류를 모두 보고할 수 있어야 한다.
- 종단 검증 실패는 새 포트로 재시도하지 않는다. 포트가 아니라 서비스 계약 실패이기 때문이다.
- 마지막 포트 충돌에서도 cleanup을 시도한 뒤 시도 횟수와 마지막 원인을 포함해 실패한다.
- 성공한 시도에는 cleanup을 호출하지 않는다.

#### 종단 검증 callback의 계약

`verify_end_to_end` 구현은 최소한 다음을 검증한다고 가정한다.

- 테스트마다 새로운 nonce를 생성한다.
- WordPress를 통해 nonce가 포함된 데이터를 쓴다.
- HTTPS 요청으로 같은 nonce가 포함된 응답을 읽는다.
- MariaDB 관찰 결과에서도 같은 논리 데이터가 확인된다.
- 단순 `/healthz` 성공만으로 정상 처리하지 않는다.

#### 경계 조건

- 첫 번째 후보만 충돌하고 두 번째 후보는 성공하는 경우
- 모든 후보가 포트 충돌인 경우
- 포트 충돌처럼 보이는 문자열이 포함됐지만 분류 callback은 거짓인 경우
- 시작 callback이 일부 자원을 만든 뒤 오류를 던지는 경우
- cleanup이 실패한 뒤 다음 시도를 계속할 수 있는지 여부
- 시작은 성공하지만 HTTPS 검증이 실패하는 경우
- HTTPS 읽기는 성공하지만 DB 검증이 실패하는 경우
- 후보 포트 함수가 같은 포트를 반복 반환하는 경우

#### 실패 조건과 제약

- 오류 메시지의 모든 실패를 단순 부분 문자열 하나로 포트 충돌이라고 판단하지 않는다.
- 포트 예약이 Docker bind 시점까지 유지된다고 가정하지 않는다.
- 무한 재시도를 허용하지 않는다.
- 이전 실패 시도의 자원이 남은 상태에서 다음 시작을 조용히 진행하지 않는다.
- 종단 검증 결과를 캐시된 이전 데이터로 통과시키지 않도록 nonce 사용을 계약에 포함한다.

### 구현 후 자가 검증

- [ ] 정상 경로에서 후보 선택→시작→종단 검증 순서로 한 번씩 호출된다.
- [ ] 알려진 bind 충돌만 cleanup 뒤 새 포트로 재시도한다.
- [ ] 비포트 시작 오류는 첫 시도에서 중단된다.
- [ ] 종단 검증 오류는 포트를 바꿔 숨기지 않고 즉시 실패한다.
- [ ] 실패한 각 시도의 cleanup 호출 횟수가 정확하다.
- [ ] cleanup 오류가 있어도 시작·검증의 원래 오류가 보존된다.
- [ ] 성공 포트를 반환하며 성공 시도의 자원을 정리하지 않는다.
- [ ] 시도 한도를 초과하면 마지막 충돌과 총 시도 횟수를 설명할 수 있다.
- [ ] nonce 기반 write/read/DB 검증 계약이 테스트 double로 확인된다.
- [ ] 알고리즘의 callback 호출 수는 최대 `O(max_attempts)`이고 추가 공간은 `O(1)`이다.

### 구현 후 설명할 것

1. 포트 예약과 실제 container bind 사이 race를 제거하지 않고 분류된 재시도로 흡수한 이유
2. healthcheck와 E2E 검증이 서로 대체 관계가 아니라 계층별 증거인 이유
3. 종단 검증 실패를 새 포트로 재시도하지 않는 이유
4. nonce가 캐시·이전 실행 데이터에 의한 오탐을 줄이는 방식
5. 원래 오류와 cleanup 오류를 함께 보존하는 예외 설계

### 원본 확인 위치

- **Thread:** 03
- **커밋 메시지:** `test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증`
- **파일:** `tests/runtime_stack.py`
- **클래스·함수:** `RuntimeStack`, `RuntimeStack.fetch`, `RuntimeStack.verify_e2e`
- **관련 Thread:** 01의 nginx·PHP-FPM·MariaDB 요청 경계, 05의 WordPress bootstrap, 11의 loopback 포트·네트워크 격리, 13의 격리 실행 하네스

---

<a id="a-06"></a>
## A-06 · [Thread 11 / `feat(runtime): 프로젝트·이미지·포트·URL 격리` + `feat(network): DB 트래픽을 내부 backend로 격리` + `feat(runtime): 서비스 자원과 종료 한계 적용`] 런타임 격리 정책

### 면접 질문

같은 호스트에서 여러 Compose 프로젝트를 실행할 때 `container_name`을 고정하지 않고 프로젝트 이름, 이미지 이름, 포트, URL을 외부 입력으로 분리한 이유는 무엇입니까? 여기에 frontend/backend 네트워크 분리와 resource limit을 추가하면 어떤 실패 범위를 줄일 수 있습니까?

꼬리 질문:

- Compose 프로젝트 이름과 고정 `container_name`이 동시에 존재하면 어떤 충돌이 생길 수 있습니까?
- nginx만 frontend에, MariaDB는 internal backend에, WordPress는 두 네트워크에 연결한 이유는 무엇입니까?
- backend를 `internal: true`로 설정하는 것과 DB 포트를 publish하지 않는 것은 어떻게 다릅니까?
- HTTPS 기본 bind 주소를 loopback으로 둔 이유와 외부 공개 환경에서 바꿔야 할 점은 무엇입니까?
- CPU, memory, PID, `nofile` 제한은 각각 어떤 자원 고갈을 막습니까?
- `no-new-privileges`, 로그 회전, stop signal, grace period는 성능 제한과 어떤 다른 위험을 다룹니까?
- 설정 파일에 정책이 존재하는 것만 확인하지 않고 실제 container inspect와 network inspect까지 검증한 이유는 무엇입니까?
- 제한값을 너무 낮게 두면 어떤 오탐 또는 장애가 생기며, 기본값을 정하는 근거는 무엇이어야 합니까?

### 30초 모범 답변

멀티 프로젝트 환경에서는 이름·이미지 태그·포트가 전역 자원이라 고정값이 있으면 서로 덮어쓰거나 시작에 실패할 수 있습니다. 그래서 Compose 프로젝트 범위를 유지하고 고정 컨테이너 이름을 없애며, 외부에 필요한 nginx 포트만 loopback에 publish합니다. nginx와 WordPress는 frontend, WordPress와 MariaDB는 internal backend로 연결해 DB 접근 경로를 줄입니다. CPU·메모리·PID·파일 디스크립터 제한은 noisy neighbor와 폭주를 완화하고, `no-new-privileges`, 로그 회전, graceful stop은 권한 상승·디스크 고갈·강제 종료 손상을 줄입니다. 정적 모델과 실제 inspect 결과를 함께 확인해야 선언과 런타임 적용의 차이를 잡을 수 있습니다.

### 답변 핵심 키워드

project namespace · global Docker resources · no fixed container name · loopback publish · frontend/backend segmentation · internal network · least connectivity · CPU/memory/PID/nofile limits · no-new-privileges · log rotation · graceful shutdown · rendered model · runtime inspect

### 백지 구현

#### 구현 목표

Docker나 Compose에 직접 의존하지 않는 정규화된 런타임 모델을 받아, 세 서비스의 이름·포트·네트워크·자원·보안 정책 위반을 모두 수집해 반환하는 validator를 구현한다.

#### 인터페이스

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ServiceRuntimePolicy:
    name: str
    image: str
    fixed_container_name: str | None
    published_addresses: tuple[str, ...]
    networks: frozenset[str]
    cpu_limit: float | None
    memory_limit_bytes: int | None
    pid_limit: int | None
    nofile_soft: int | None
    nofile_hard: int | None
    security_options: frozenset[str]
    stop_grace_seconds: int | None
    log_max_bytes: int | None
    log_max_files: int | None


@dataclass(frozen=True)
class NetworkRuntimePolicy:
    name: str
    internal: bool
    members: frozenset[str]


def isolation_violations(
    services: tuple[ServiceRuntimePolicy, ...],
    networks: tuple[NetworkRuntimePolicy, ...],
    *,
    project_image_prefix: str,
) -> list[str]:
    # 직접 구현
    ...
```

#### 입력과 출력

- 정규화된 서비스 세 개와 네트워크 두 개를 받는다.
- 위반이 없으면 빈 목록을 반환한다.
- 여러 위반이 있으면 첫 오류에서 멈추지 않고 결정적인 순서의 메시지 목록을 반환한다.

#### 반드시 만족해야 할 조건

- 서비스 이름 집합이 정확히 `nginx`, `wordpress`, `mariadb`인지 확인한다.
- 어떤 서비스에도 고정 container name이 없어야 한다.
- 모든 image 이름이 비어 있지 않은 `project_image_prefix`로 시작해야 한다.
- 외부 publish는 nginx에만 허용하고, 모든 publish 주소는 loopback 주소여야 한다.
- nginx의 네트워크 집합은 frontend만, WordPress는 frontend와 backend, MariaDB는 backend만이어야 한다.
- frontend와 backend의 실제 member 집합도 서비스 선언과 일치해야 한다.
- backend는 internal이어야 하고 frontend는 외부 요청을 받을 수 있어야 한다.
- 세 서비스 모두 CPU, memory, PID 제한이 양수여야 한다.
- `nofile` soft/hard가 존재하고 `0 < soft <= hard`를 만족해야 한다.
- 세 서비스 모두 `no-new-privileges:true` 보안 옵션을 가져야 한다.
- stop grace period가 양수여야 한다.
- 로그 최대 크기와 파일 수가 양수여야 한다.
- 같은 의미의 위반 메시지는 중복하지 않는다.

#### 경계 조건

- 서비스 또는 네트워크가 누락되거나 하나 더 있는 경우
- image prefix가 빈 문자열인 경우
- nginx가 `0.0.0.0`이나 빈 host 주소로 publish된 경우
- MariaDB가 포트를 publish한 경우
- network 선언과 network member 관찰값이 서로 다른 경우
- backend가 internal이 아니거나 nginx가 backend에 연결된 경우
- 제한값이 `None`, 0, 음수, NaN인 경우
- `nofile` soft가 hard보다 큰 경우
- security option의 철자나 형식이 다른 경우
- 동일한 이름의 서비스·네트워크가 중복된 경우

#### 실패 조건과 제약

- 첫 위반만 반환하지 않는다. 운영 정책 검사는 한 번에 수정할 수 있도록 전체 위반을 수집한다.
- image 이름의 세부 Docker 문법 parser를 구현하지 않는다. 이 문제에서는 prefix 계약만 확인한다.
- loopback 판별을 단순히 문자열이 `127.`로 시작하는지에만 의존하지 말고, 표준 IP parser 사용을 고려한다.
- 네트워크 선언만 보고 실제 member 집합을 추정하지 않는다. 두 입력을 독립적으로 대조한다.
- 구체적인 기본 제한값을 코드에 임의로 만들지 않는다. 존재·순서·양수 조건만 검증한다.

### 구현 후 자가 검증

- [ ] 정상 모델이 빈 위반 목록을 반환한다.
- [ ] 고정 container name, 잘못된 image prefix, 비loopback publish를 각각 검출한다.
- [ ] nginx·WordPress·MariaDB의 정확한 network membership을 양쪽 모델에서 검증한다.
- [ ] backend의 internal 정책이 빠진 경우를 검출한다.
- [ ] CPU·memory·PID·nofile·로그 제한의 누락과 비정상 값을 모두 찾는다.
- [ ] `no-new-privileges:true`와 stop grace 누락을 찾는다.
- [ ] 여러 위반을 동시에 넣어도 결과 순서가 매 실행에서 같다.
- [ ] 중복 서비스·네트워크 이름을 모호하게 덮어쓰지 않고 위반으로 보고한다.
- [ ] IPv4·IPv6 loopback 처리 정책이 테스트에 드러난다.
- [ ] 시간 복잡도는 서비스·네트워크 수에 대해 `O(n)` 또는 `O(n log n)`, 추가 공간은 `O(n)` 이내다.

### 구현 후 설명할 것

1. 이름 격리, 네트워크 격리, 자원 격리를 서로 다른 방어층으로 본 이유
2. nginx만 publish하고 MariaDB backend를 internal로 둔 최소 연결 원칙
3. 정적 Compose 모델과 실제 런타임 inspect를 모두 검증해야 하는 이유
4. 제한값이 너무 낮을 때의 가용성 비용과 관찰 기반 튜닝 방법
5. 여러 위반을 모아 반환하는 validator와 fail-fast validator의 trade-off

### 원본 확인 위치

- **Thread:** 11
- **커밋 메시지:** `feat(runtime): 프로젝트·이미지·포트·URL 격리`, `feat(network): DB 트래픽을 내부 backend로 격리`, `feat(runtime): 서비스 자원과 종료 한계 적용`
- **파일:** `.env.example`, `srcs/docker-compose.yml`, `srcs/requirements/wordpress/tools/docker-entrypoint.sh`, `tests/runtime_stack.py`, `tests/validate_stack.py`
- **관련 컴포넌트:** nginx, WordPress, MariaDB, frontend 네트워크, backend 네트워크
- **관련 Thread:** 01의 기본 서비스 토폴로지, 02의 프로젝트 실행 경계, 03의 포트 충돌·E2E 검증, 12의 운영 상태 증거 수집

---

<a id="a-07"></a>
## A-07 · [Thread 10 / `build(wordpress): WordPress 산출물을 고정해 게시` + `test(supply-chain): 불변 image 입력 검증`] 재현 가능한 이미지와 검증된 산출물

### 면접 질문

Dockerfile에서 버전 문자열만 고정하는 것과 base image digest, 불변 package snapshot, 다운로드 산출물 SHA-256까지 고정하는 것은 어떻게 다릅니까? 이 프로젝트가 빌드 시 검증한 WordPress core를 런타임에 다시 checksum으로 확인해 게시한 이유를 설명해 보세요.

꼬리 질문:

- `FROM image:tag`만 사용하면 같은 Dockerfile을 다시 빌드했을 때 결과가 달라질 수 있는 이유는 무엇입니까?
- base image digest를 고정해도 package repository가 변하면 무엇이 달라질 수 있습니까?
- URL과 버전을 고정했는데도 다운로드 SHA-256 검증이 필요한 이유는 무엇입니까?
- 빌드 시 checksum을 통과한 파일을 런타임에 다시 검증하는 것은 중복입니까?
- core 파일과 사용자 `wp-content`를 같은 정책으로 덮어쓰면 어떤 데이터 손실이 생깁니까?
- 손상된 core 파일만 검증된 이미지 산출물로 교체하고 사용자 콘텐츠는 보존하는 reconciliation의 장점은 무엇입니까?
- 고정된 버전은 보안 업데이트를 자동으로 받지 못합니다. 어떤 업데이트 절차가 필요합니까?
- 런타임에서 설치된 패키지·PHP·MariaDB 최소 버전을 확인하는 것은 build-time pinning과 어떤 다른 오류를 잡습니까?

### 30초 모범 답변

태그와 일반 패키지 저장소는 시간이 지나면 같은 이름이 다른 바이트를 가리킬 수 있습니다. 그래서 base image digest와 불변 Debian snapshot을 고정하고, WP-CLI와 WordPress archive는 버전뿐 아니라 SHA-256까지 검증해 빌드 입력을 바이트 수준으로 묶습니다. 런타임에서는 이미지에 포함된 core manifest로 persistent volume의 core 파일을 다시 확인하고, 손상되거나 다른 파일만 임시 파일과 원자 rename으로 복구합니다. 사용자 콘텐츠는 별도 정책으로 보존합니다. 재현성과 자동 업데이트는 상충하므로, 버전 갱신은 checksum·호환성·E2E 검증을 포함한 명시적 변경으로 수행해야 합니다.

### 답변 핵심 키워드

mutable tag · image digest · immutable package snapshot · version pin · content digest · SHA-256 · build-time verification · runtime attestation · core/content boundary · selective reconciliation · atomic publish · explicit update process · compatibility floor

### 백지 구현

#### 구현 목표

텍스트 manifest에 기록된 상대 경로와 SHA-256을 검증해, 산출물 트리가 기대한 바이트와 일치하는지 확인하는 스트리밍 verifier를 구현한다. 절대 경로, 상위 디렉터리 이동, 중복 경로, 심볼릭 링크, 비일반 파일을 거부한다.

#### 인터페이스

```python
from dataclasses import dataclass
from pathlib import Path
from collections.abc import Iterable


@dataclass(frozen=True)
class ArtifactRequirement:
    relative_path: str
    sha256: str


class ArtifactVerificationError(RuntimeError):
    pass


def verify_artifact_tree(
    root: Path,
    requirements: Iterable[ArtifactRequirement],
    *,
    chunk_size: int = 1024 * 1024,
) -> None:
    # 직접 구현
    ...
```

#### 입력과 출력

- 검증할 root와 상대 경로·digest 요구사항을 받는다.
- 모든 항목이 안전하고 digest가 일치하면 반환한다.
- 하나라도 잘못되면 해당 상대 경로와 실패 종류를 포함한 `ArtifactVerificationError`를 발생시킨다.

#### 반드시 만족해야 할 조건

- `root`가 실제 디렉터리이고 심볼릭 링크가 아닌지 확인한다.
- requirement 집합이 비어 있으면 거부한다.
- 상대 경로는 비어 있지 않고 절대 경로가 아니며 `.`·`..`로 root를 벗어날 수 없어야 한다.
- 경로 정규화 뒤 같은 대상을 가리키는 중복 requirement를 거부한다.
- SHA-256 문자열은 정확히 64자리 16진수여야 한다.
- 대상은 존재하는 일반 파일이어야 하며 심볼릭 링크를 허용하지 않는다.
- 파일은 `chunk_size` 단위로 읽어 전체를 메모리에 적재하지 않는다.
- 계산한 digest를 대소문자와 무관한 문자열 편의가 아니라 정규화된 값으로 비교한다.
- `chunk_size`가 1보다 작은 경우 파일을 열기 전에 거부한다.
- 파일 읽기 실패를 digest 불일치와 구분해 보고한다.

#### 경계 조건

- `a/../b`, `./b`, `b`처럼 정규화 뒤 충돌하는 경로
- 절대 경로, 빈 경로, trailing slash로 표현된 디렉터리
- dangling symlink와 root 밖을 가리키는 symlink
- FIFO, socket, device, directory
- 같은 파일이 manifest에 두 번 나타나는 경우
- 대문자 SHA-256, 길이가 짧거나 비16진 문자가 있는 digest
- 빈 파일과 매우 큰 파일
- 검증 도중 파일이 교체되거나 길이가 바뀌는 경우
- root 자체가 사라지거나 권한이 바뀌는 경우

#### 실패 조건과 제약

- `Path.resolve()` 결과만 확인한 뒤 일반 `open()`으로 다시 열어 안전하다고 가정하지 않는다. 구현 범위에 따라 descriptor 기준 검증 또는 명확한 잔여 race 설명이 필요하다.
- digest 불일치 파일을 자동으로 수정하지 않는다. 이 문제는 verifier만 구현한다.
- 사용자 콘텐츠처럼 manifest 관리 대상이 아닌 파일을 임의로 삭제하지 않는다.
- 모든 파일 바이트를 하나의 큰 `bytes`로 합치지 않는다.
- SHA-256을 비밀값 검증용 MAC처럼 설명하지 않는다. 여기서는 공급망 산출물 무결성 식별자다.

### 구현 후 자가 검증

- [ ] 정상 manifest의 여러 파일이 통과한다.
- [ ] 누락·digest 불일치·읽기 실패가 서로 구분된다.
- [ ] 절대 경로, `..`, 정규화 중복 경로가 파일 open 전에 거부된다.
- [ ] symlink와 일반 파일이 아닌 항목을 거부한다.
- [ ] 빈 파일과 chunk보다 큰 파일의 digest를 정확히 계산한다.
- [ ] `chunk_size`가 작아도 결과가 같고 전체 파일을 메모리에 올리지 않는다.
- [ ] malformed SHA-256을 검증 시작 전에 거부한다.
- [ ] requirement iterator를 한 번만 순회해도 동작하도록 필요 자료를 안전하게 정리한다.
- [ ] 오류 메시지에 실제 파일 내용이나 불필요한 민감 경로를 포함하지 않는다.
- [ ] 시간 복잡도는 총 파일 바이트 수를 `B`라 할 때 `O(B)`, 추가 버퍼는 `O(chunk_size + n)`이다.

### 구현 후 설명할 것

1. tag·version·digest·snapshot이 각각 고정하는 범위
2. build-time verification과 runtime verification이 잡는 실패가 다른 이유
3. core 산출물과 사용자 콘텐츠를 분리한 reconciliation 정책
4. 스트리밍 digest의 메모리 복잡도와 파일 교체 race에 대한 한계
5. 자동 업데이트를 끄는 대신 안전하게 버전을 갱신하는 운영 절차

### 원본 확인 위치

- **Thread:** 10
- **커밋 메시지:** `build(wordpress): WordPress 산출물을 고정해 게시`, `test(supply-chain): 불변 image 입력 검증`
- **파일:** `srcs/requirements/nginx/Dockerfile`, `srcs/requirements/mariadb/Dockerfile`, `srcs/requirements/wordpress/Dockerfile`, `srcs/requirements/wordpress/tools/docker-entrypoint.sh`, `tests/validate_stack.py`, `tests/runtime_stack.py`
- **함수·컴포넌트:** `install_core_files`, `install_content_files`, WordPress core checksum manifest, 런타임 버전 검증
- **관련 Thread:** 05의 WordPress 산출물 게시, 11의 이미지 이름 격리, 13의 정적·런타임 검증 gate
