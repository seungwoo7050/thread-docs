# 런타임 기동, 소유권 lifecycle, 비밀과 evidence 경계

이 문서는 마스터 인덱스의 IM-03, IM-07, IM-17, IM-18을 다룬다. 백지 구현은 Compose나 원본 스크립트를 복원하지 않고 dependency, ownership, cleanup, security invariant를 순수 로직으로 축소한다.

<a id="im-03"></a>
## [Thread 3 / `build(startup): enforce full dependency DAG`] 기동 DAG와 semantic readiness

### 면접 질문

Docker Compose에서 `depends_on`으로 프로세스 시작 순서만 맞추는 것과 persistence → preflight → application → consumer-assignment → settlement → admin의 의미 기반 기동 DAG를 보장하는 것은 어떻게 다릅니까?

꼬리 질문:

- 서로 독립적인 서비스는 병렬 기동해도 됩니까?
- cycle이나 존재하지 않는 dependency가 있으면 언제 실패시켜야 합니까?
- healthcheck가 통과했지만 consumer assignment가 아직 없으면 준비된 것입니까?
- 선행 단계가 실패했을 때 후행 서비스를 시작하지 않는 이유는 무엇입니까?

### 30초 모범 답변

프로세스가 시작됐다는 사실은 의존 기능을 사용할 수 있다는 뜻이 아닙니다. 기동 관계를 DAG로 모델링하고 각 노드는 단순 PID가 아니라 그 단계에 맞는 health 또는 semantic readiness를 만족한 뒤 다음 노드를 열어야 합니다. cycle과 누락 dependency는 실행 전에 거부하고, 선행 실패는 후행 기동을 막아 원인과 피해 범위를 제한합니다. 독립 노드는 같은 wave에서 병렬화할 수 있지만 결정적 진단 가능성과 과도한 동시 부하 사이를 조절해야 합니다.

### 답변 핵심 키워드

DAG, topological order, startup wave, semantic readiness, healthcheck, consumer assignment, cycle detection, failure propagation, bounded parallelism

### 백지 구현

**구현 목표**

서비스별 dependency 집합을 받아 동시에 시작할 수 있는 startup wave를 계산한다. 실제 프로세스 실행과 health polling은 제외한다.

**인터페이스 또는 함수 시그니처**

```java
// key: service, value: 해당 service가 시작되기 전에 준비되어야 하는 service 집합
// 직접 구현
List<Set<String>> startupWaves(Map<String, Set<String>> dependencies) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 서비스 dependency graph
- 출력: 앞 wave가 모두 준비된 뒤 시작할 수 있는 서비스 집합의 순서

**반드시 만족해야 할 조건**

- 같은 wave 안의 서비스끼리는 서로 의존하지 않는다.
- 모든 서비스가 정확히 한 번만 출력된다.
- dependency가 없는 서비스는 첫 wave 후보가 된다.
- cycle이 있으면 부분 결과를 반환하지 않고 실패한다.
- graph에 선언되지 않은 dependency를 허용할지 거부할지 정책을 명시한다.

**경계 조건**

- 빈 graph
- 서비스 하나
- 모든 서비스가 독립적
- 긴 선형 chain
- diamond dependency
- 자기 자신을 dependency로 가짐

**실패 조건**

- cycle
- 누락 dependency를 금지하는 정책에서의 외부 노드 참조
- null 또는 빈 서비스 이름

**필요한 제약**

- 결과 안의 집합 iteration order까지 결정적으로 만들 필요가 있는지 설명한다.
- 목표 시간 복잡도는 노드와 edge 수에 선형이어야 한다.

### 구현 후 자가 검증

- 선형, 병렬, diamond graph의 wave가 올바른가?
- cycle에서 무한 루프나 부분 성공이 발생하지 않는가?
- 같은 서비스가 중복 출력되지 않는가?
- 독립 노드가 불필요하게 직렬화되지 않는가?
- dependency edge 수가 커져도 불필요한 전체 재탐색을 하지 않는가?
- 계산된 순서와 실제 readiness 확인 책임을 구분해 설명할 수 있는가?

### 구현 후 설명할 것

1. process start와 semantic readiness의 차이
2. 위상 정렬 결과를 wave로 반환한 이유
3. cycle·누락 dependency를 실행 전에 실패시키는 이유
4. 병렬 기동이 진단 가능성과 리소스 사용에 미치는 trade-off
5. consumer assignment가 별도 readiness 단계인 이유

### 원본 확인 위치

- Thread 3 — 풀스택 토폴로지와 기동 DAG
- 커밋: `4b3c66663326 — build(startup): enforce full dependency DAG`
- 확인 가능한 컴포넌트: Docker Compose dependency graph, persistence, preflight, application, consumer-assignment, settlement, admin
- 관련 Thread: 5, 14

---

<a id="im-07"></a>
## [Thread 6 / `build(gate): generate isolated runtime secrets`] 실행 단위 secret의 소비 경계 검증

### 면접 질문

cold-gate마다 service key, RSA keypair, PostgreSQL/Grafana password를 새로 만들었다면, 안전성의 핵심은 생성 알고리즘 외에 무엇입니까?

꼬리 질문:

- 한 서비스가 다른 서비스의 secret을 environment에서 볼 수 있으면 어떤 문제가 생깁니까?
- secret 이름은 로그에 남겨도 되고 값은 남기면 안 된다는 경계를 어떻게 구현합니까?
- 프로세스가 실패한 뒤 secret 파일이나 environment dump를 누가 정리합니까?
- 동일 실행 안에서 재시작이 필요할 때 secret을 재사용할지 재발급할지 어떤 기준으로 정합니까?

### 30초 모범 답변

secret은 강하게 생성하는 것만으로 충분하지 않고 실행 단위 소유권과 소비자 allowlist가 필요합니다. 각 secret은 필요한 서비스에만 주입하고, 진단에는 값이 아니라 식별자나 fingerprint만 사용하며, evidence와 로그는 별도 redaction gate를 통과시켜야 합니다. 생성부터 주입, 폐기까지 cold-gate lifecycle 한 곳이 소유해야 orphan secret을 막을 수 있습니다. 재시작 정책은 동일 실행 identity를 유지해야 하는지와 노출 의심 여부를 기준으로 명시합니다.

### 답변 핵심 키워드

per-run secret, least privilege, consumer allowlist, environment injection, ownership, fingerprint, no plaintext diagnostics, rotation, cleanup

### 백지 구현

**구현 목표**

생성된 secret과 실제 consumer binding을 비교해 허용되지 않은 노출을 검출한다. 암호학적 secret 생성은 범위에서 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record SecretBinding(String secretId, String consumer, String channel) {}
record ExposureViolation(String secretId, String consumer, String reason) {}

// 직접 구현
List<ExposureViolation> validateSecretBindings(
        Set<String> generatedSecretIds,
        List<SecretBinding> actualBindings,
        Map<String, Set<String>> allowedConsumers) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 생성된 secret ID, 실제 주입 대상, secret별 허용 consumer
- 출력: 허용되지 않은 노출 목록

**반드시 만족해야 할 조건**

- 생성되지 않은 secret ID의 binding을 오류로 본다.
- allowlist에 없는 consumer를 오류로 본다.
- 오류 메시지에 secret 값이 들어갈 수 있는 필드를 두지 않는다.
- 중복 binding을 허용할지 경고할지 정책을 명시한다.
- consumer 이름과 channel 값의 공백·대소문자 정규화 정책을 명시한다.

**경계 조건**

- 생성된 secret은 있지만 binding이 없음
- 허용 consumer가 빈 집합
- 같은 binding이 여러 번 들어옴
- 허용된 consumer지만 예상하지 않은 channel을 사용함

**실패 조건**

- unknown secret
- unauthorized consumer
- malformed binding
- allowlist 자체가 secret 집합과 맞지 않음

**필요한 제약**

- 이 함수에는 plaintext secret을 전달하지 않는다.
- 값 검사와 노출 topology 검사를 분리한다.

### 구현 후 자가 검증

- 허용된 최소 binding만 있을 때 violation이 없는가?
- 다른 서비스로의 단일 오주입을 정확히 찾는가?
- unknown secret과 unbound secret을 구분할 수 있는가?
- 반환된 진단 정보만으로 secret 값이 추론되지 않는가?
- 중복과 정규화 정책이 결정적인가?
- consumer 수가 늘어날 때 탐색 복잡도를 설명할 수 있는가?

### 구현 후 설명할 것

1. secret generation과 exposure validation을 분리한 이유
2. plaintext 대신 ID/fingerprint만 진단에 쓰는 이유
3. environment injection의 편의성과 노출 면적 trade-off
4. secret 소유권을 Thread 14의 cold-gate lifecycle에 붙이는 이유

### 원본 확인 위치

- Thread 6 — 서비스 신뢰 경계와 제한 노출
- 커밋: `43d20c34e2eb — build(gate): generate isolated runtime secrets`
- 확인 가능한 컴포넌트: cold-gate별 service key, RSA keypair, PostgreSQL/Grafana password, environment injection
- 관련 Thread: 14, 15

---

<a id="im-17"></a>
## [Thread 14 / `build(gate): own cold release lifecycle`] 소유권 기반 cleanup과 예외 안전성

### 면접 질문

context 생성, secrets, build, checks, 성공 cleanup, 실패 cleanup을 하나의 cold release lifecycle이 소유해야 하는 이유는 무엇입니까?

꼬리 질문:

- resource를 세 개 확보한 뒤 네 번째 단계에서 실패하면 어떤 순서로 정리합니까?
- cleanup 자체가 실패하면 원래 실패를 어떻게 보존합니까?
- 성공 시 보존해야 할 evidence와 반드시 지워야 할 secret을 어떻게 구분합니까?
- 프로세스 중단이나 취소에서도 같은 cleanup 계약을 지키려면 무엇이 필요합니까?

### 30초 모범 답변

resource를 만든 주체와 지우는 주체가 다르면 누락과 과잉 삭제가 생기기 쉽습니다. cold-gate가 context부터 secret, build resource, checks까지 획득 즉시 소유권과 cleanup action을 등록하고, 성공과 실패 모두에서 역순으로 정리해야 합니다. cleanup 오류는 원래 실패를 덮지 말고 부가 오류로 보존하며, cleanup은 재호출돼도 안전해야 합니다. evidence처럼 보존 대상과 secret처럼 폐기 대상을 resource 유형별 정책으로 나누는 것도 중요합니다.

### 답변 핵심 키워드

single owner, acquire/register, reverse cleanup, exception safety, idempotent cleanup, primary error preservation, cancellation, retained evidence

### 백지 구현

**구현 목표**

resource를 획득할 때 cleanup을 등록하고, body의 성공·실패와 무관하게 등록된 cleanup을 수행하는 lifecycle scope를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
@FunctionalInterface
interface ThrowingRunnable { void run() throws Exception; }

final class LifecycleScope implements AutoCloseable {
    // 직접 구현
    void register(String resourceId, ThrowingRunnable cleanup) { }

    // 직접 구현
    @Override
    public void close() throws Exception { }
}
```

**입력과 출력**

- 입력: resource 식별자와 cleanup action
- 출력: 직접 반환값은 없으며 `close()`가 모든 등록 resource의 정리를 시도한다.

**반드시 만족해야 할 조건**

- 등록 역순으로 cleanup을 시도한다.
- 한 cleanup이 실패해도 나머지 cleanup을 계속 시도한다.
- 여러 cleanup 실패를 잃지 않는다.
- `close()` 재호출 정책을 명시하고 안전하게 처리한다.
- 등록되지 않은 resource를 임의로 삭제하지 않는다.

**경계 조건**

- 등록된 resource가 없음
- cleanup 하나만 있음
- 일부 cleanup만 실패함
- 같은 resource ID가 중복 등록됨
- `close()` 중 다시 `register()`가 호출됨

**실패 조건**

- cleanup action 예외
- lifecycle이 닫힌 뒤 등록 시도
- 중복 소유권이 금지된 경우의 중복 resource ID

**필요한 제약**

- body에서 발생한 primary failure와 `close()` failure를 호출자가 함께 보존할 수 있어야 한다.
- thread-safe scope가 필요한지 single-thread owner인지 전제를 명시한다.

### 구현 후 자가 검증

- 정상 종료와 body 실패 모두에서 cleanup이 실행되는가?
- 네 번째 획득 실패 시 앞서 획득한 세 resource만 정리되는가?
- cleanup 하나의 실패가 뒤 resource 정리를 막지 않는가?
- cleanup 순서가 dependency의 역순인가?
- `close()` 재호출로 이중 삭제 부작용이 생기지 않는가?
- 원래 예외와 cleanup 예외를 모두 관찰할 수 있는가?
- 닫힌 scope에 등록하는 오용을 차단하는가?

### 구현 후 설명할 것

1. 획득 즉시 cleanup을 등록하는 이유
2. 역순 정리가 dependency 해제에 유리한 이유
3. primary failure와 cleanup failure를 함께 보존하는 전략
4. idempotent cleanup과 exactly-once cleanup의 차이
5. evidence 보존과 secret 폐기를 같은 lifecycle에서 다르게 취급하는 방법

### 원본 확인 위치

- Thread 14 — 소유권 기반 콜드 릴리스 게이트
- 커밋: `5ef2d1349379 — build(gate): own cold release lifecycle`
- 확인 가능한 컴포넌트: context 생성, secrets, build, checks, success cleanup, failure cleanup
- 관련 Thread: 3, 6, 15, 16

---

<a id="im-18"></a>
## [Thread 15 / `build(evidence): redact runtime credentials`] evidence redaction과 rejection을 이중화하기

### 면접 질문

service key, password, PEM/JWT-like material이 evidence에 남지 않게 할 때 redaction만 두지 않고 rejection까지 두는 이유는 무엇입니까?

꼬리 질문:

- 키 이름은 평범하지만 값 모양이 PEM이나 JWT와 비슷한 경우를 어떻게 찾습니까?
- redaction 로그에 원문 secret을 넣는 실수를 어떻게 막습니까?
- 과탐으로 evidence가 저장되지 않는 문제와 누출 위험 중 어디에 무게를 둡니까?
- 중첩 JSON, 여러 줄 PEM, 큰 로그 스트림에서 같은 정책을 어떻게 유지합니까?

### 30초 모범 답변

redaction은 알려진 secret과 구조를 정상 경로에서 안전한 표기로 바꾸는 기능이고, rejection은 redaction 뒤에도 민감한 형태가 남았을 때 저장 자체를 막는 마지막 경계입니다. 키 이름과 실제 런타임 secret 값, PEM/JWT-like value shape를 함께 검사하고 finding에는 원문이 아니라 위치와 종류만 남겨야 합니다. 보안 evidence는 fail-closed가 기본이지만 과탐은 좁고 검토 가능한 예외 규칙으로 줄여야 합니다. 같은 sanitizer를 파일 저장, CI artifact, 오류 진단 앞에 재사용해야 우회 경로가 줄어듭니다.

### 답변 핵심 키워드

defense in depth, exact secret match, key-based detection, value-shape detection, redaction, rejection, fail closed, no secret in findings, idempotence

### 백지 구현

**구현 목표**

문자열 evidence에서 정확히 알려진 secret과 민감 패턴 후보를 찾아 안전한 placeholder로 바꾸고, 저장 가능 여부를 별도 단계에서 판정한다. 구체적인 정규식 정답은 제공하지 않는다.

**인터페이스 또는 함수 시그니처**

```java
record Finding(String kind, int startInclusive, int endExclusive) {}
record SanitizedEvidence(String text, List<Finding> findings) {}

// 직접 구현
SanitizedEvidence sanitizeEvidence(String raw, Set<String> exactSecrets) {
    throw new UnsupportedOperationException("TODO");
}

// 직접 구현
void assertPersistable(SanitizedEvidence sanitized) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: raw evidence 문자열, 실행 중 실제 생성된 secret 값 집합
- 출력: sanitized 문자열과 원문을 포함하지 않는 finding metadata

**반드시 만족해야 할 조건**

- exact secret은 길이와 종류에 관계없이 출력에서 사라져야 한다.
- PEM/JWT-like 후보를 탐지하되 finding에 원문을 복사하지 않는다.
- sanitizer를 두 번 적용해도 secret이 다시 나타나지 않는다.
- rejection 단계는 위험 finding 또는 잔존 민감값이 있으면 persistence를 막는다.
- 빈 문자열이나 지나치게 짧은 exact secret을 어떻게 처리할지 정책을 둔다.

**경계 조건**

- secret이 서로 겹치는 경우
- 여러 줄 값
- 같은 secret이 여러 번 등장
- placeholder 안에 민감 패턴처럼 보이는 문자열이 포함될 가능성
- 매우 큰 입력

**실패 조건**

- invalid input
- exact secret 집합에 빈 값 존재
- sanitization 후에도 실제 secret이 남음
- scanner가 안전하게 판정하지 못함

**필요한 제약**

- finding은 위치·종류·개수만 포함하고 raw token을 포함하지 않는다.
- 메모리 한계가 중요하면 streaming 버전과 batch 버전의 trade-off를 설명한다.

### 구현 후 자가 검증

- 정상 로그는 의미를 보존하면서 통과하는가?
- 알려진 password, service key, 여러 줄 secret이 모두 제거되는가?
- 중첩되거나 겹친 secret 처리에서 일부 문자가 남지 않는가?
- sanitizer 결과와 예외 메시지 어디에도 원문 secret이 없는가?
- 두 번 sanitize한 결과가 안정적인가?
- 의심 패턴을 놓쳤을 때 rejection이 마지막 경계가 되는가?
- 대형 입력에서 시간·공간 복잡도를 설명할 수 있는가?

### 구현 후 설명할 것

1. redaction과 rejection의 책임 차이
2. exact runtime secret과 pattern 기반 탐지를 함께 쓰는 이유
3. finding에 원문을 넣지 않는 진단 설계
4. 과탐을 broad bypass가 아닌 좁은 예외로 다루는 이유
5. CI artifact persistence 앞에 동일 정책을 두는 이유

### 원본 확인 위치

- Thread 15 — 증거 저장과 비밀 제거
- 커밋: `627c34edbd44 — build(evidence): redact runtime credentials`
- 확인 가능한 컴포넌트: service key, password, PEM/JWT-like material, evidence redaction과 rejection, CI artifact retention
- 관련 Thread: 6, 13, 14, 16
