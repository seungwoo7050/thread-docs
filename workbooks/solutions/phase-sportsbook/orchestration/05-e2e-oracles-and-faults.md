# Live probe, cross-store oracle, fault restoration

이 문서는 마스터 인덱스의 IM-11, IM-12를 다룬다. 실제 Kafka client나 Toxiproxy API 사용법을 암기하는 대신, correlation, timeout, 다중 상태 oracle, fault ownership과 cleanup을 구현 대상으로 축소한다.

<a id="im-11"></a>
## [Thread 8 / `build(fixtures): stage executable publisher`] correlation 기반 live Kafka publish/probe

### 면접 질문

fixture publisher로 실제 Kafka에 publish한 뒤 probe가 성공했다고 판정하려면 단순히 "메시지 하나를 받았다"보다 어떤 조건이 더 필요합니까?

꼬리 질문:

- 이전 테스트가 남긴 stale record를 현재 성공으로 오판하지 않으려면 어떻게 합니까?
- correlation이 같은 응답이 두 개 왔는데 내용이 다르면 어떻게 처리합니까?
- schema 또는 shared protocol identity가 기대와 다르면 timeout과 구분해 어떻게 보고합니까?
- unrelated traffic이 많은 topic에서 probe의 시간·메모리 사용을 어떻게 제한합니까?

### 30초 모범 답변

live probe는 publish한 요청의 고유 correlation과 protocol identity를 기준으로 정확한 응답을 찾아야 합니다. unrelated 또는 이전 실행 record는 무시하고, 기대 응답이 timeout 안에 한 번의 일관된 결과로 관찰돼야 성공입니다. 같은 correlation의 상충 record는 중복 성공이 아니라 invariant 위반으로 보고, protocol mismatch와 timeout을 다른 오류로 분류해야 진단할 수 있습니다. poll 범위와 보관량을 제한해 테스트가 트래픽 양에 따라 무한히 커지지 않게 합니다.

### 답변 핵심 키워드

live I/O, unique correlation, stale record, unrelated traffic, protocol identity, timeout, conflicting duplicate, bounded polling

### 백지 구현

**구현 목표**

관찰된 record 목록에서 기대 correlation과 protocol identity에 맞는 probe 결과를 판정한다. 실제 Kafka poll loop는 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record ObservedRecord(String correlationId, String protocolIdentity, String payloadHash, long observedAtMillis) {}
enum ProbeStatus { SUCCESS, TIMEOUT, PROTOCOL_MISMATCH, CONFLICT }
record ProbeResult(ProbeStatus status, Optional<ObservedRecord> matched) {}

// 직접 구현
ProbeResult evaluateProbe(
        String expectedCorrelationId,
        String expectedProtocolIdentity,
        long publishStartedAtMillis,
        long deadlineMillis,
        List<ObservedRecord> observed) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 기대 correlation/protocol, publish 시작 시각, deadline, 관찰 record
- 출력: 성공·timeout·protocol mismatch·상충 duplicate 판정

**반드시 만족해야 할 조건**

- correlation이 다른 record는 무시한다.
- publish 시작 전 record는 stale로 간주한다.
- correlation은 같지만 protocol identity가 다르면 별도 오류다.
- 같은 correlation과 protocol에서 payload hash가 서로 다르면 conflict다.
- deadline 안의 일관된 record 하나 이상만 성공이다.

**경계 조건**

- 관찰 record 없음
- stale record만 있음
- unrelated traffic만 있음
- 동일 record 중복
- 동일 correlation의 상충 payload
- deadline 경계 시각에 관찰됨

**실패 조건**

- invalid time range
- blank correlation 또는 protocol identity
- 동일 correlation의 protocol mismatch와 valid record가 함께 존재할 때의 정책 미정

**필요한 제약**

- record 전체 정렬이 필요한지 단일 순회로 가능한지 설명한다.
- 실제 poll loop는 cancellation과 timeout을 별도로 책임진다.

### 구현 후 자가 검증

- stale·unrelated record가 성공으로 오판되지 않는가?
- valid record 하나와 동일 중복 여러 개가 같은 성공으로 수렴하는가?
- 상충 payload를 conflict로 찾는가?
- protocol mismatch와 timeout이 구분되는가?
- deadline 경계 정책이 일관적인가?
- 큰 record 목록을 한 번의 순회와 제한된 추가 메모리로 처리 가능한가?

### 구현 후 설명할 것

1. correlation과 publish 시작 시각을 함께 쓰는 이유
2. duplicate와 conflicting duplicate의 차이
3. protocol mismatch와 timeout을 분리하는 진단 가치
4. batch evaluator와 live poll loop의 책임 경계
5. 관찰량을 bounded하게 유지하는 방법

### 원본 확인 위치

- Thread 8 — 결정적 Avro 픽스처와 Kafka 프로브
- 커밋: `269cf445cb2a — build(fixtures): stage executable publisher`
- 확인 가능한 컴포넌트: executable fixture publisher, shared protocol identity, live Kafka publish/probe
- 관련 Thread: 5, 9, 12

---

<a id="im-12"></a>
## [Thread 9 / `test(e2e): …`] cross-store E2E oracle과 Toxiproxy fault restoration

### 면접 질문

13개 cross-store E2E scenario에서 HTTP 응답만 확인하지 않고 여러 저장소의 최종 상태를 oracle로 검증해야 하는 이유와, Toxiproxy fault를 테스트 실패와 무관하게 복구해야 하는 이유를 설명해 보세요.

꼬리 질문:

- API는 성공했지만 PostgreSQL, Redis, Kafka 중 하나가 예상 상태가 아니면 테스트 결과는 무엇입니까?
- fault 적용 후 scenario assertion이 실패하면 restoration 오류와 assertion 오류 중 무엇을 주 오류로 보존합니까?
- fault가 실제로 제거됐다는 사실을 어떻게 검증합니까?
- 여러 test가 같은 proxy를 건드릴 수 있다면 어떤 isolation 또는 ownership이 필요합니까?
- eventual consistency 때문에 즉시 상태가 맞지 않을 때 oracle은 어떻게 기다립니까?

### 30초 모범 답변

분산 흐름에서는 API 성공이 모든 저장소와 downstream effect의 정합성을 뜻하지 않으므로, scenario별로 PostgreSQL·Redis·Kafka 등 관찰 가능한 상태를 하나의 기대 결과와 대조해야 합니다. 일시적 불일치는 bounded polling으로 기다리되 timeout 시 마지막 관찰과 불일치 항목을 남깁니다. Toxiproxy fault는 resource처럼 한 owner가 적용 handle을 갖고 성공·실패·취소 모두에서 복구하며, restoration 실패가 원래 scenario 실패를 덮지 않게 보존합니다. 테스트 종료 전에는 정상 연결이 회복됐는지도 다시 확인해야 다음 scenario 오염을 막을 수 있습니다.

### 답변 핵심 키워드

cross-store oracle, observable state, eventual consistency, bounded polling, fault ownership, unconditional restoration, primary failure, cleanup failure, test isolation

### 백지 구현

**구현 목표**

두 개의 작은 경계를 구현한다.

1. 기대 상태와 관찰된 여러 store 상태를 비교해 모든 불일치를 반환하는 pure oracle
2. fault를 적용한 뒤 scenario를 실행하고 모든 종료 경로에서 restoration을 시도하는 fault scope

**인터페이스 또는 함수 시그니처**

```java
record ExpectedState(Map<String, String> facts) {}
record ObservedState(Map<String, String> facts) {}
record OracleResult(boolean matches, Map<String, String> mismatches) {}

// 직접 구현
OracleResult evaluateOracle(ExpectedState expected, ObservedState observed) {
    throw new UnsupportedOperationException("TODO");
}

interface FaultHandle {
    void restore() throws Exception;
    boolean isRestored() throws Exception;
}

@FunctionalInterface
interface ThrowingSupplier<T> { T get() throws Exception; }

// 직접 구현
<T> T runWithFault(
        Supplier<FaultHandle> applyFault,
        ThrowingSupplier<T> scenario) throws Exception {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- oracle 입력: store 이름을 포함한 기대 fact와 관찰 fact
- oracle 출력: 전체 일치 여부와 모든 mismatch
- fault scope 입력: fault 적용 함수와 scenario body
- fault scope 출력: scenario 결과 또는 primary failure

**반드시 만족해야 할 조건**

- oracle은 첫 불일치에서 중단하지 않고 모든 확인 가능한 mismatch를 반환한다.
- 예상 fact 누락과 예상 밖 fact를 어떻게 다룰지 정책을 명시한다.
- fault 적용이 성공한 뒤에는 scenario 성공·실패와 무관하게 restoration을 시도한다.
- restoration 후 `isRestored()` 검증을 수행한다.
- scenario failure와 restoration failure가 모두 있으면 둘 다 잃지 않는다.
- fault 적용 자체가 실패하면 존재하지 않는 handle을 복구하려 하지 않는다.

**경계 조건**

- 기대·관찰 fact가 모두 비어 있음
- 하나의 store만 불일치
- 여러 store가 동시에 불일치
- scenario 성공, restoration 실패
- scenario 실패, restoration 성공
- 둘 다 실패
- restore는 성공을 반환했지만 `isRestored()`가 false

**실패 조건**

- fault 적용 실패
- scenario failure
- restore failure
- restoration verification failure
- malformed oracle input

**필요한 제약**

- 동일 proxy를 공유한다면 외부 lock 또는 test별 proxy ownership이 필요하다.
- eventual consistency polling은 pure oracle 바깥에서 bounded하게 수행한다.
- 오류 메시지에 secret이나 전체 payload를 무제한 복사하지 않는다.

### 구현 후 자가 검증

- API 성공과 store mismatch를 분리해 실패로 판정할 수 있는가?
- 여러 mismatch를 한 번에 보고하는가?
- scenario의 모든 종료 경로에서 restore가 시도되는가?
- apply failure 시 restore가 잘못 호출되지 않는가?
- primary failure가 cleanup failure에 덮이지 않는가?
- restore 호출 성공뿐 아니라 실제 복구 상태도 검증하는가?
- 두 테스트가 같은 fault target을 공유할 때의 경쟁 조건을 설명할 수 있는가?
- oracle 비교와 bounded polling의 책임이 분리됐는가?

### 구현 후 설명할 것

1. endpoint 응답과 cross-store invariant를 별도 oracle로 보는 이유
2. 모든 mismatch를 모아서 반환하는 진단 가치
3. fault를 resource handle로 소유하는 이유
4. primary failure와 restoration failure의 예외 보존 방식
5. eventual consistency polling의 timeout·interval trade-off

### 원본 확인 위치

- Thread 9 — 다중 저장소 E2E 오라클과 장애 주입
- 확인 가능한 대표 커밋: `28a36bf8d802 — test(e2e): …`; 현재 프로젝트 요약에 전체 메시지는 노출되지 않음
- 확인 가능한 컴포넌트: 13개 cross-store E2E scenario, Toxiproxy fault mutation, restoration
- 관련 Thread: 4, 5, 8, 10, 11, 12, 13
