# 저장소 격리, Kafka 준비 계약, 실행 artifact 검증

이 문서는 마스터 인덱스의 IM-04, IM-05, IM-06, IM-10을 다룬다. 원본의 Docker·PostgreSQL·Redis·Kafka·JAR 배선을 그대로 재현하지 않고, 면접에서 구현 가능한 판정기와 planner로 축소한다.

<a id="im-04"></a>
## [Thread 4 / `build(postgres): bootstrap service databases`] 격리된 저장소 bootstrap과 migration plan

### 면접 질문

서비스별 PostgreSQL database bootstrap과 Flyway 실행, Redis별 persistent state 격리를 함께 설계할 때 어떤 invariant를 먼저 정해야 합니까?

꼬리 질문:

- 재시작할 때 create-if-missing과 migration을 어떻게 구분합니까?
- 두 서비스가 실수로 같은 database나 Redis state를 가리키면 언제 탐지합니까?
- migration 일부만 성공한 뒤 재실행되면 무엇을 신뢰합니까?
- idempotent bootstrap이 schema drift를 무시한다는 뜻입니까?

### 30초 모범 답변

각 서비스가 소유한 PostgreSQL database와 Redis persistent state가 겹치지 않는다는 소유권 invariant를 먼저 둬야 합니다. bootstrap은 없는 리소스 생성과 기존 schema의 versioned migration을 구분하고, 재실행해도 같은 목표 상태로 수렴해야 하지만 예상하지 않은 drift는 조용히 덮지 말고 실패시켜야 합니다. 부분 실패 뒤에는 migration metadata와 실제 상태를 다시 검증하고, persistent state를 편의상 초기화하지 않습니다. 격리는 장애와 테스트 오염의 범위를 줄이는 대신 운영 리소스 수와 관리 비용을 늘립니다.

### 답변 핵심 키워드

store ownership, isolated database, isolated Redis state, create-if-missing, Flyway, versioned migration, idempotency, drift detection, partial failure

### 백지 구현

**구현 목표**

서비스별 원하는 PostgreSQL database와 Redis namespace를 받아 이름 충돌을 검사하고, 현재 상태에 대해 `CREATE`, `MIGRATE`, `NOOP` 또는 `DRIFT` plan을 만든다. 실제 DB 연결은 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record StoreSpec(String service, String postgresDatabase, String redisStateId, int targetSchemaVersion) {}
record ExistingStoreState(Set<String> postgresDatabases, Set<String> redisStateIds, Map<String, Integer> schemaVersions) {}
record BootstrapAction(String service, String kind, String detail) {}

// 직접 구현
List<BootstrapAction> planBootstrap(
        List<StoreSpec> desired,
        ExistingStoreState actual) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 서비스별 desired store identity와 schema version, 현재 존재 상태
- 출력: 결정적인 bootstrap action 목록

**반드시 만족해야 할 조건**

- 서로 다른 서비스가 같은 PostgreSQL database나 Redis state ID를 소유할 수 없다.
- 없는 database는 create 대상으로 분류한다.
- 현재 version이 목표보다 낮으면 migrate 대상으로 분류한다.
- 현재 version이 목표보다 높거나 metadata가 모순되면 drift로 실패한다.
- 동일 입력에서 action 순서가 결정적이어야 한다.

**경계 조건**

- 원하는 서비스가 없음
- database는 있지만 version metadata가 없음
- Redis state만 충돌
- target version과 current version이 같음
- 서비스 이름은 다르지만 store identity가 같음

**실패 조건**

- 소유권 충돌
- 음수 version 또는 malformed spec
- backward migration이 필요한 상태
- 실제 상태 metadata 불일치

**필요한 제약**

- action planner와 실제 I/O executor를 분리한다.
- destructive reset action은 이 문제의 허용 action에 포함하지 않는다.

### 구현 후 자가 검증

- 신규, 동일 version, migration 필요 상태가 올바르게 분류되는가?
- database와 Redis 각각의 충돌을 독립적으로 찾는가?
- current version이 더 높을 때 자동 downgrade하지 않는가?
- action 순서가 입력 list 순서에 우연히 의존하지 않는가?
- 빈 plan과 malformed state를 명확히 구분하는가?
- 재실행 시 `CREATE`가 `NOOP` 또는 `MIGRATE`로 수렴하는가?
- planner가 실제 DB side effect를 수행하지 않는가?

### 구현 후 설명할 것

1. 소유권 충돌을 runtime보다 bootstrap plan 단계에서 막는 이유
2. idempotency와 drift tolerance의 차이
3. PostgreSQL schema version과 Redis persistent state identity를 함께 검사하는 이유
4. destructive reset을 자동 복구 수단으로 쓰지 않는 이유
5. planner와 executor 분리의 테스트 이점

### 원본 확인 위치

- Thread 4 — 저장소 격리와 마이그레이션 무결성
- 커밋: `f57b610f2637 — build(postgres): bootstrap service databases`
- 확인 가능한 컴포넌트: PostgreSQL database bootstrap, Flyway, Redis별 isolated persistent state
- 관련 Thread: 9, 10, 11

---

<a id="im-05"></a>
## [Thread 5 / `build(kafka): provision topics without mutation`] Kafka topic drift를 자동 수정하지 않고 판정하기

### 면접 질문

`topic-init`이 없는 topic만 만들고 기존 topic의 partition, replication, retention drift를 자동 수정하지 않도록 한 이유는 무엇입니까?

꼬리 질문:

- partition 수 증가는 가능한 변경인데도 왜 무조건 자동 적용하지 않을 수 있습니까?
- retention 값만 다르면 warning으로 둘지 실패로 둘지 무엇을 기준으로 정합니까?
- broker가 일시적으로 잘못된 metadata를 반환하면 어떻게 재확인합니까?
- desired spec과 actual spec 비교 결과를 운영자가 이해하기 쉽게 어떻게 표현합니까?

### 30초 모범 답변

missing resource 생성과 기존 resource mutation은 위험도가 다릅니다. 새 topic은 원하는 spec으로 만들 수 있지만 기존 partition, replication, retention을 바꾸면 ordering, 부하, 보존 정책에 운영 영향을 줄 수 있으므로 drift를 명시적으로 보여 주고 gate를 실패시키는 편이 안전합니다. 비교는 `CREATE`, `UNCHANGED`, `DRIFT`처럼 부작용 없는 판정으로 분리하고 mutation은 별도 승인 경로로 둡니다. 이렇게 하면 bootstrap의 편의성과 운영 변경 통제를 동시에 얻습니다.

### 답변 핵심 키워드

create-if-missing, no implicit mutation, desired vs actual, partition, replication, retention, drift, fail closed, change approval

### 백지 구현

**구현 목표**

원하는 topic spec과 현재 spec을 비교해 생성, 일치, drift를 판정하고 drift field를 반환한다.

**인터페이스 또는 함수 시그니처**

```java
record TopicSpec(String name, int partitions, int replicationFactor, long retentionMs) {}
enum TopicAction { CREATE, UNCHANGED, DRIFT }
record TopicDecision(TopicAction action, Set<String> driftFields) {}

// 직접 구현
TopicDecision reconcileTopic(TopicSpec desired, Optional<TopicSpec> actual) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: desired topic spec, optional actual spec
- 출력: action과 drift가 난 field 집합

**반드시 만족해야 할 조건**

- actual이 없으면 `CREATE`다.
- 모든 비교 대상이 같으면 `UNCHANGED`다.
- 하나라도 다르면 `DRIFT`이며 자동 수정하지 않는다.
- topic 이름 불일치를 정상 drift로 볼지 입력 오류로 볼지 명시한다.
- drift field 집합 순서가 결정적이어야 한다.

**경계 조건**

- partition 1
- retention 0 또는 정책상 허용되는 특수값
- replication factor가 broker 수보다 큰지 여부는 이 함수의 책임인지 구분
- actual spec의 일부 값이 조회되지 않음

**실패 조건**

- 음수 또는 0 이하의 허용되지 않은 spec
- desired/actual topic identity 불일치
- 불완전한 actual metadata

**필요한 제약**

- 이 함수는 Kafka mutation API를 호출하지 않는다.
- 비교 필드 추가 시 기존 caller가 조용히 무시하지 않도록 설계한다.

### 구현 후 자가 검증

- missing, exact match, 각 단일 field drift가 구분되는가?
- 여러 field drift를 모두 보고하는가?
- drift인데 `UNCHANGED`로 오판하는 default 값이 없는가?
- invalid spec과 정상 drift가 구분되는가?
- 결과가 입력 객체 순서나 hash iteration에 흔들리지 않는가?
- 비교 복잡도가 field 수에 선형임을 설명할 수 있는가?

### 구현 후 설명할 것

1. create와 mutate의 위험 차이
2. drift를 fail-fast로 다루는 이유
3. 부작용 없는 reconciler와 승인된 mutation executor를 분리하는 이유
4. 모든 drift field를 한 번에 반환하는 진단 이점

### 원본 확인 위치

- Thread 5 — Kafka 토픽과 소비자 준비 계약
- 커밋: `f9e15158d474 — build(kafka): provision topics without mutation`
- 확인 가능한 컴포넌트: Kafka KRaft state, `topic-init`, partition·replication·retention drift
- 관련 Thread: 3, 8, 9, 12

---

<a id="im-06"></a>
## [Thread 5 / consumer-assignment gap] 소비 가능한 상태를 readiness로 판정하기

### 면접 질문

Kafka broker와 topic이 준비됐더라도 consumer assignment가 확인되기 전에는 왜 후행 단계를 시작하면 안 됩니까?

꼬리 질문:

- assignment가 한 번 관찰된 직후 rebalance가 시작되면 readiness를 유지합니까?
- required consumer가 여러 개일 때 일부만 준비된 상태를 어떻게 보고합니까?
- fixed sleep 대신 무엇을 사용합니까?
- timeout 오류에서 마지막 snapshot을 어떻게 진단에 남깁니까?

### 30초 모범 답변

broker가 연결 가능하고 topic이 존재해도 실제 consumer가 partition을 배정받지 못했다면 이벤트를 처리할 준비가 된 것이 아닙니다. required assignment를 semantic readiness 조건으로 두고 snapshot을 polling하되, rebalance나 불안정한 관찰을 피하려면 일정 횟수 연속 만족 같은 안정성 조건을 둘 수 있습니다. timeout에서는 단순 실패만 내지 말고 누락된 consumer와 마지막 assignment를 남겨야 합니다. fixed sleep보다 빠르고 flapping에도 더 정확한 방식입니다.

### 답변 핵심 키워드

semantic readiness, consumer assignment, rebalance, stable samples, timeout, missing requirement, last snapshot, no fixed sleep

### 백지 구현

**구현 목표**

시간 순서대로 들어오는 assignment snapshot을 평가해 required assignment가 지정 횟수 연속 만족되면 ready를 반환한다. 실제 Kafka polling과 sleep은 범위에서 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record ConsumerRequirement(String consumer, String topic) {}
record AssignmentSnapshot(long observedAtMillis, Set<ConsumerRequirement> assignments, boolean rebalancing) {}
record ReadinessResult(boolean ready, Set<ConsumerRequirement> missing, int consecutiveSatisfied) {}

// 직접 구현
ReadinessResult evaluateAssignments(
        Set<ConsumerRequirement> required,
        List<AssignmentSnapshot> snapshots,
        int stableSamples) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: required assignment, 시간순 snapshot, 필요한 연속 만족 횟수
- 출력: ready 여부, 마지막 기준 누락 assignment, 연속 만족 횟수

**반드시 만족해야 할 조건**

- rebalance 중인 snapshot은 안정적인 성공으로 세지 않는다.
- required가 모두 보이는 snapshot만 연속 성공으로 센다.
- 중간에 하나라도 빠지면 연속 성공 수를 재설정한다.
- snapshot 시간 순서가 뒤섞였을 때의 정책을 명시한다.
- required가 빈 경우 의미를 명시한다.

**경계 조건**

- snapshot 없음
- stableSamples가 1
- 마지막 snapshot에서만 준비됨
- 준비 후 rebalance
- unrelated assignment가 많이 포함됨

**실패 조건**

- stableSamples가 0 이하
- malformed requirement
- 시간 역전이 금지된 정책에서 out-of-order snapshot

**필요한 제약**

- 실제 timeout과 polling interval은 caller 책임으로 둔다.
- snapshot 전체를 보관하지 않는 streaming reducer로 바꿀 수 있는지 설명한다.

### 구현 후 자가 검증

- 연속 성공 횟수가 정확히 stableSamples일 때 ready가 되는가?
- rebalance와 누락에서 카운터가 리셋되는가?
- unrelated consumer가 readiness를 방해하지 않는가?
- missing 집합이 마지막 평가 상태와 일치하는가?
- snapshot이 많을 때 한 번의 순회로 처리 가능한가?
- fixed sleep 없이 readiness semantics를 설명할 수 있는가?

### 구현 후 설명할 것

1. broker readiness와 consumer readiness를 분리한 이유
2. 연속 관찰 조건이 flapping을 줄이는 방법
3. timeout 진단에 last snapshot과 missing set이 필요한 이유
4. batch evaluator와 live polling reducer의 차이

### 원본 확인 위치

- Thread 5 — Kafka 토픽과 소비자 준비 계약
- 현재 프로젝트 요약에서 consumer-assignment 관련 개별 커밋 메시지는 분리 노출되지 않음
- 확인 가능한 컴포넌트: consumer assignment readiness
- 관련 Thread: 3, 8, 12

---

<a id="im-10"></a>
## [Thread 8 / `build(fixtures): stage executable publisher`] 실행 가능한 fixture JAR 계약 검증

### 면접 질문

fixture publisher JAR을 staging하기 전에 Java 17, shared protocol identity, shaded dependencies, main class를 검사한 이유는 무엇입니까?

꼬리 질문:

- JAR이 존재하고 hash도 계산되는데 실행 계약이 깨질 수 있는 경우는 무엇입니까?
- protocol identity mismatch는 compile time이 아니라 runtime에서 어떤 장애를 만듭니까?
- 모든 검증 오류를 한 번에 보여 줄지 첫 오류에서 중단할지 어떤 기준으로 선택합니까?
- main class와 dependency bundling은 배포 artifact의 어떤 성질을 증명합니까?

### 30초 모범 답변

파일이 존재한다는 사실만으로 실행 가능한 fixture라는 보장은 없습니다. 목표 runtime인 Java 17과 맞고, 서비스가 기대하는 shared protocol identity를 사용하며, 필요한 dependency가 함께 패키징되고, 진입점이 명시돼 있어야 실제 cold 환경에서 동일하게 실행됩니다. staging 전에 이 계약을 검증하면 Kafka probe 단계에서 뒤늦게 classpath나 schema identity 문제를 발견하는 비용을 줄일 수 있습니다. hash는 내용 동일성을 증명하지만 실행 가능성까지 증명하지는 않습니다.

### 답변 핵심 키워드

Java 17, executable JAR, main class, shaded dependencies, protocol identity, preflight validation, hash vs executability

### 백지 구현

**구현 목표**

JAR 검사기가 추출했다고 가정한 metadata를 기대 계약과 비교해 staging 가능 여부와 모든 위반 사항을 반환한다. 실제 ZIP/JAR parsing은 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record ExecutableJarFacts(
        int javaMajor,
        String protocolIdentity,
        boolean dependenciesBundled,
        Optional<String> mainClass) {}
record FixtureContract(int requiredJavaMajor, String requiredProtocolIdentity) {}

// 직접 구현
List<String> validateFixtureJar(
        ExecutableJarFacts facts,
        FixtureContract contract) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 관찰된 JAR facts, 기대 fixture contract
- 출력: staging을 막아야 하는 위반 메시지 목록

**반드시 만족해야 할 조건**

- Java major 불일치, protocol identity 불일치, dependency 미포함, main class 누락을 각각 검출한다.
- 여러 위반이 있으면 모두 보고한다.
- 오류 메시지는 결정적 순서를 가진다.
- 빈 protocol identity와 blank main class를 정상으로 보지 않는다.

**경계 조건**

- 요구 버전과 정확히 같음
- main class 문자열은 있으나 공백뿐임
- protocol identity의 대소문자 차이를 허용할지 여부
- 모든 위반이 동시에 발생

**실패 조건**

- malformed facts
- 지원하지 않는 Java major
- contract 자체가 비어 있음

**필요한 제약**

- 이 validator가 통과하기 전에는 staging/publish를 호출하지 않는다.
- artifact hash 검증은 별도 책임으로 둔다.

### 구현 후 자가 검증

- 정상 facts에서 오류가 없는가?
- 각 단일 위반과 복합 위반이 모두 검출되는가?
- blank 값이 누락으로 처리되는가?
- 오류 순서가 입력이나 collection 순서에 흔들리지 않는가?
- validator가 파일 I/O나 staging side effect를 수행하지 않는가?
- 새 계약 field를 추가할 때 누락 검사를 방지할 구조인가?

### 구현 후 설명할 것

1. 존재·hash 검증과 실행 계약 검증의 차이
2. 모든 오류를 모아서 반환하는 진단 trade-off
3. protocol identity를 artifact 경계에서 확인하는 이유
4. validator와 JAR inspector를 분리한 테스트 이점

### 원본 확인 위치

- Thread 8 — 결정적 Avro 픽스처와 Kafka 프로브
- 커밋: `269cf445cb2a — build(fixtures): stage executable publisher`
- 확인 가능한 컴포넌트: Java 17, shared protocol identity, shaded dependencies, main class, executable publisher
- 관련 Thread: 2, 5, 16
