# 멱등성·마켓별 순서·안전한 reopen 워크북

이 문서는 운영 명령이 HTTP 요청으로 들어온 뒤 durable queue에 접수되고, 마켓별 순서를 따라 Kafka acknowledgement까지 완료되는 전 과정을 다룬다.

핵심은 다음 세 경계를 혼동하지 않는 것이다.

1. **동일 요청인가:** canonical fingerprint와 idempotency key
2. **내구 접수됐는가:** 원자 dedup, Stream append, fail-close projection
3. **최신 명령으로 적용 가능한가:** per-market predecessor와 completion CAS

<a id="p08"></a>
## [Thread 13 / `feat(commands): fingerprint operator requests`] canonical fingerprint와 ambiguous concatenation 방지

### 면접 질문

운영 명령의 멱등성을 위해 요청 fingerprint를 만들 때 단순히 `eventId + marketId + status + reason`을 이어 붙이지 않은 이유는 무엇입니까? 어떤 필드를 어떤 규칙으로 정규화해야 합니까?

꼬리 질문:

- `"ab" + "c"`와 `"a" + "bc"` 같은 모호성을 어떻게 없앱니까?
- idempotency key 자체와 request fingerprint를 같은 hash domain에서 처리하면 어떤 문제가 있습니까?
- reason의 앞뒤 공백은 같은 요청으로 봐야 합니까?
- caller나 request version을 fingerprint에 넣은 이유는 무엇입니까?
- SHA-256 fingerprint가 authentication을 대신할 수 있습니까?

### 30초 모범 답변

멱등성은 "같아 보이는 문자열"이 아니라 **정확히 같은 canonical request**를 판별해야 합니다. 그래서 reason을 API 규칙대로 정규화하고, version·caller·action·event·market·status·reason을 고정 순서로 UTF-8 인코딩한 뒤 각 필드 길이를 함께 넣어 경계를 보존했습니다. idempotency key는 별도 domain prefix로 hash해 request hash와 의미를 분리합니다. 같은 key와 같은 fingerprint면 최초 action을 replay하고, 같은 key에 다른 fingerprint면 conflict입니다. 이 hash는 비교용 identity이지 인증 수단은 아닙니다.

### 답변 핵심 키워드

canonicalization, length-prefix framing, fixed field order, normalization, versioning, caller domain, SHA-256, domain separation, replay vs conflict

### 백지 구현

**구현 목표**

운영 명령을 canonical byte sequence로 만들고 SHA-256 hex fingerprint를 계산한다. 특정 정답 hash 값은 제공하지 않는다.

**인터페이스**

```java
enum RequestedStatus {
  OPEN, SUSPENDED, CLOSED
}

record OperatorCommandRequest(
    String caller,
    String eventId,
    String marketId,
    RequestedStatus requestedStatus,
    String reason) {}

final class CommandFingerprint {
  static String requestFingerprint(
      int requestVersion,
      OperatorCommandRequest request) {
    // 직접 구현
  }

  static String idempotencyIdentity(String rawIdempotencyKey) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: request version과 명령 필드
- 출력: 소문자 hex SHA-256 문자열
- idempotency key는 request와 다른 domain에서 별도 identity 생성

**반드시 만족해야 할 조건**

- 필드 순서가 고정
- 각 문자열의 UTF-8 byte length와 bytes를 함께 hash
- reason은 trim 후 1~256자
- status에 대응하는 action 의미가 fingerprint에 반영
- request version과 caller 포함
- idempotency identity에 고정 domain tag 포함
- 같은 canonical 입력은 같은 출력
- 서로 다른 필드 분할이 같은 byte sequence가 되지 않음

**경계 조건**

- 다국어 reason
- 빈 문자열, 공백만 있는 reason
- 256자와 257자
- 같은 표시 문자열이지만 Unicode 표현이 다른 경우의 normalization 정책
- caller 변경
- version 변경
- status 변경
- 매우 긴 idempotency key

**실패 조건**

- null 필수 필드
- default charset 사용
- delimiter 한 글자에만 의존
- hash collision을 business conflict 처리와 혼동
- secret 비교에 일반 문자열 equality 사용

**필요한 제약**

- 15~20분
- 표준 JDK API만 사용
- digest 객체를 전역 mutable singleton으로 공유하지 않음

### 구현 후 자가 검증

- [ ] 앞뒤 공백만 다른 reason이 정책대로 같은 fingerprint
- [ ] status, caller, version 중 하나만 바뀌어도 fingerprint가 달라짐
- [ ] `["ab","c"]`와 `["a","bc"]`가 다름
- [ ] UTF-8 다국어 입력이 platform과 무관하게 안정적
- [ ] request fingerprint와 idempotency identity가 같은 raw text에서도 다름
- [ ] invalid reason은 hash 전에 거부
- [ ] 결과 길이가 SHA-256 hex 길이와 일치
- [ ] mutable buffer나 shared digest로 인한 동시성 문제가 없음

### 구현 후 설명할 것

1. length-prefix framing과 delimiter framing의 차이
2. normalization 정책 변경 시 request version을 올려야 하는 이유
3. 원문 idempotency key를 Redis key에 직접 쓰지 않은 이유
4. hash collision 가능성과 practical conflict 처리
5. fingerprint와 authorization의 책임 분리

### 원본 확인 위치

- Thread 13
- 커밋: `feat(commands): fingerprint operator requests`
- 커밋: `test(commands): verify canonical request fingerprints`
- 파일/컴포넌트: `MarketActionFingerprint`
- 함수: `request`, `idempotencyKey`
- 관련 컴포넌트: `OperatorActionQueue`, `MarketAdminController`
- 관련 Thread: 12, 14

---

<a id="p09"></a>
## [Thread 13 / `feat(commands): define atomic operator submissions`] dedup·Stream append·fail-close를 한 접수 경계로 묶기

### 면접 질문

`OperatorSubmissionScript`가 idempotency mapping, action mapping, Stream `XADD`, restrictive projection을 한 Lua script에서 처리한 이유를 설명해 주세요.

꼬리 질문:

- idempotency key가 같고 action ID만 다른 동일 요청은 어떤 결과를 반환해야 합니까?
- key가 같지만 payload가 다르면 왜 새 action으로 접수하면 안 됩니까?
- restrictive action의 Stream append는 성공했지만 override write가 실패하면 어떤 상태입니까?
- terminal event의 reopen은 어느 시점에 거부하고, Stream에는 무엇이 남아야 합니까?
- HTTP 202 응답은 실제 market transition 완료와 어떻게 다릅니까?

### 30초 모범 답변

접수 단계의 성공 조건은 "요청을 기억했다"가 아니라 dedup identity, 원래 action ID, durable Stream record, 필요한 fail-close projection이 모두 함께 만들어진 것입니다. 이를 여러 Redis 호출로 나누면 재시도 중 중복 record나 mapping만 남는 부분 성공이 생깁니다. Lua로 한 원자 경계에서 같은 fingerprint는 최초 결과를 replay하고, 다른 payload나 action 충돌은 conflict로 끝내며 아무것도 바꾸지 않습니다. suspend/close는 접수와 동시에 제한하고, reopen은 terminal이면 enqueue하지 않으며 기존 제한은 broker ack까지 유지합니다. 202는 이 durable 접수만 의미합니다.

### 답변 핵심 키워드

atomic acceptance, idempotency replay, conflict, action identity, XADD, no partial state, restrictive projection, deferred reopen, HTTP 202

### 백지 구현

**구현 목표**

Redis 대신 in-memory immutable state를 사용해 원자 명령 접수 규칙을 구현한다. 메서드가 실패하면 입력 state는 전혀 변경되지 않아야 한다.

**인터페이스**

```java
record Command(
    String idempotencyIdentity,
    String fingerprint,
    String actionId,
    String eventId,
    String marketId,
    RequestedStatus requestedStatus,
    String normalizedReason,
    Instant occurredAt) {}

record AcceptedMetadata(
    String fingerprint,
    String actionId,
    long sequence,
    long predecessor,
    String recordId) {}

record StreamEntry(
    String recordId,
    Command command,
    long sequence,
    long predecessor) {}

record SubmissionState(
    Map<String, AcceptedMetadata> byIdempotency,
    Map<String, String> idempotencyByAction,
    Map<String, Long> tailByMarket,
    Map<String, MarketStatus> overrideByMarket,
    Map<String, MarketStatus> effectiveByMarket,
    Set<String> terminalEvents,
    Set<String> terminalMarkets,
    List<StreamEntry> stream) {}

enum SubmissionOutcome {
  CREATED, REPLAYED, CONFLICT, TERMINAL_REOPEN
}

record SubmissionResult(
    SubmissionOutcome outcome,
    AcceptedMetadata metadata,
    SubmissionState newState) {}

final class AtomicCommandAcceptor {
  static SubmissionResult submit(SubmissionState state, Command command) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 현재 durable state snapshot과 정규화된 command
- 출력: outcome, 최초 metadata 또는 새 metadata, 전체 새 state
- conflict/terminal 실패 시 `newState`는 입력과 동등해야 함

**반드시 만족해야 할 조건**

- 같은 idempotency identity + 같은 fingerprint: 최초 metadata를 `REPLAYED`
- 같은 idempotency identity + 다른 fingerprint: `CONFLICT`, no mutation
- 같은 action ID가 다른 idempotency identity에 이미 연결됨: `CONFLICT`
- 새 요청은 market tail을 1 증가시키고 predecessor를 이전 tail로 설정
- Stream entry와 idempotency/action mapping이 함께 생성
- `SUSPENDED`/`CLOSED`는 override/effective를 접수 결과 state에 반영
- `OPEN`은 terminal event/market이면 `TERMINAL_REOPEN`, Stream append 없음
- terminal이 아니어도 `OPEN` 접수 시 기존 제한 override를 즉시 제거하지 않음
- 입력 map/list/set과 결과 state는 서로 mutable alias를 공유하지 않음

**경계 조건**

- 최초 market command
- 같은 canonical request를 다른 action ID로 재시도
- 같은 action ID로 다른 market 요청
- restrictive command가 terminal market에 들어옴
- reopen 접수 직전 terminal latch가 생김
- 연속 두 명령의 sequence
- record ID 생성 실패

**실패 조건**

- mapping만 쓰고 Stream entry 누락
- conflict 뒤 tail 증가
- terminal reopen을 Stream에 남김
- replay 시 새 action ID를 반환
- restrictive action 접수 후 effective가 여전히 `OPEN`
- state 일부가 mutable alias로 외부에서 변경됨

**필요한 제약**

- 25~30분
- transaction은 "새 immutable state를 한 번에 반환"하는 방식으로 표현
- record ID 생성기는 주입하거나 deterministic fake 사용
- 네트워크·Redis API는 구현하지 않음

### 구현 후 자가 검증

- [ ] 최초 요청은 `CREATED`, Stream size +1
- [ ] 같은 요청 재시도는 `REPLAYED`, Stream size 불변, 최초 action ID 반환
- [ ] 같은 key 다른 payload는 `CONFLICT`, 모든 state 불변
- [ ] action ID 충돌도 no mutation
- [ ] 첫 sequence는 1, predecessor는 0
- [ ] 같은 market 두 번째 명령은 sequence 2
- [ ] 다른 market은 독립 sequence
- [ ] restrictive 접수 직후 effective가 제한 상태
- [ ] reopen 접수 직후 기존 override가 남음
- [ ] terminal reopen은 Stream을 만들지 않음
- [ ] 결과 state를 수정해도 입력 state가 변하지 않음

### 구현 후 설명할 것

1. Java multi-call 대신 Redis Lua를 사용한 실제 이유
2. replay가 새 action ID가 아니라 최초 action ID를 반환해야 하는 이유
3. restrictive projection을 접수 transaction에 포함한 도메인 판단
4. OPEN override를 ack까지 유지하는 이유
5. HTTP 202와 최종 Kafka delivery 상태를 조회할 별도 API 필요성

### 원본 확인 위치

- Thread 13
- 커밋: `feat(commands): define atomic operator submissions`
- 커밋: `feat(commands): persist idempotent submissions`
- 커밋: `feat(commands): defer operator reopens`
- 커밋: `feat(api): accept durable market controls`
- 파일/컴포넌트: `OperatorSubmissionScript`, `OperatorActionQueue`, `OperatorActionSubmission`, `TerminalMarketReopenException`, `MarketAdminController`
- 함수: `OperatorActionQueue.submit`
- 관련 Thread: 05, 12, 14

---

<a id="p10"></a>
## [Thread 14 / `feat(commands): chain per-market operator actions`] predecessor 기반 마켓별 순서 보장

### 면접 질문

운영 명령에 전체 전역 sequence가 아니라 `(eventId, marketId)`별 `sequence`와 `predecessor`를 둔 이유는 무엇입니까? 어느 명령이 지금 publish 가능한지 어떻게 판단합니까?

꼬리 질문:

- 같은 market의 sequence 2가 sequence 1보다 먼저 poll되면 어떻게 합니까?
- 서로 다른 market의 명령은 동시에 처리해도 됩니까?
- committed 값보다 sequence가 작거나 같으면 어떤 상태입니까?
- tail과 committed를 왜 따로 저장합니까?
- queue의 Stream 순서만 믿으면 충분하지 않은 이유는 무엇입니까?

### 30초 모범 답변

필요한 순서는 전체 시스템이 아니라 같은 market 내부입니다. 접수 시 market tail을 원자 증가해 sequence를 만들고 predecessor는 직전 sequence로 둡니다. 전달 시 committed가 predecessor와 같을 때만 READY이고, 이미 committed 이상이면 COMPLETED, predecessor가 아직 안 끝났으면 BLOCKED입니다. 이렇게 하면 같은 market은 직렬화하면서 다른 market은 병렬 처리할 수 있습니다. tail은 최신 접수 명령을, committed는 ack 후 완료된 경계를 나타내므로 superseded 판단에도 둘 다 필요합니다.

### 답변 핵심 키워드

per-key ordering, sequence, predecessor, committed, tail, head-of-line blocking, cross-key concurrency, supersession

### 백지 구현

**구현 목표**

여러 market의 queued action에서 현재 publish 가능한 action을 선택한다. 같은 market에서는 한 번에 하나만 READY로 반환하고, 다른 market은 병렬 후보가 될 수 있다.

**인터페이스**

```java
record MarketKey(String eventId, String marketId) {}

record QueuedAction(
    String actionId,
    MarketKey market,
    long sequence,
    long predecessor,
    String streamId) {}

record OrderingSnapshot(
    Map<MarketKey, Long> committedByMarket,
    List<QueuedAction> queued) {}

enum OrderingState {
  READY, BLOCKED, COMPLETED, INVALID
}

record OrderedCandidate(
    QueuedAction action,
    OrderingState state) {}

final class PerMarketOrdering {
  static List<OrderedCandidate> classify(OrderingSnapshot snapshot) {
    // 직접 구현
  }

  static List<QueuedAction> selectReady(
      OrderingSnapshot snapshot,
      int limit) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: market별 committed sequence와 queued actions
- 출력: 각 action의 state 또는 이번 cycle의 READY 목록

**반드시 만족해야 할 조건**

- `sequence > 0`
- `predecessor == sequence - 1`
- `committed >= sequence`이면 `COMPLETED`
- `committed == predecessor`이면 `READY`
- `committed < predecessor`이면 `BLOCKED`
- 동일 market에서 한 cycle에 가장 이른 READY action 하나만 선택
- 서로 다른 market의 READY는 함께 선택 가능
- 결과 순서는 deterministic
- `limit` 준수

**경계 조건**

- committed 값이 없는 새 market
- sequence가 빠진 queue
- duplicate sequence
- 같은 market의 action이 Stream에서 뒤섞임
- limit보다 READY market이 많음
- 이미 완료된 record가 Stream에 남음

**실패 조건**

- predecessor invariant 위반을 BLOCKED로 숨김
- 같은 market에서 sequence 1과 2를 동시에 READY로 반환
- 전역 최소 sequence 때문에 다른 market까지 막음
- mutable 입력 map을 변경

**필요한 제약**

- 15~25분
- 실제 Redis CAS는 구현하지 않음
- 정렬 기준을 명시하고 복잡도를 설명

### 구현 후 자가 검증

- [ ] 새 market의 sequence 1이 READY
- [ ] sequence 2는 committed 0에서 BLOCKED
- [ ] committed 1이 되면 sequence 2가 READY
- [ ] committed 2에서 sequence 1/2는 COMPLETED
- [ ] market A의 BLOCKED가 market B의 READY를 막지 않음
- [ ] 같은 market에서 READY 하나만 선택
- [ ] malformed predecessor가 INVALID
- [ ] duplicate sequence 탐지
- [ ] limit과 deterministic ordering 검증
- [ ] 시간 복잡도와 market별 grouping 비용 설명 가능

### 구현 후 설명할 것

1. per-market ordering과 Kafka partition key ordering의 역할 차이
2. tail과 committed를 분리한 이유
3. blocked action이 pending backlog를 만들 때의 운영 지표
4. 하나의 poison action이 같은 market 후속 명령을 막는 방식
5. 전역 순서보다 per-key 순서가 처리량에 유리한 이유

### 원본 확인 위치

- Thread 14
- 커밋: `feat(commands): chain per-market operator actions`
- 파일/컴포넌트: `OperatorActionQueue`, `OperatorMarketAction`
- key/필드: `sequenceKey`, `committedKey`, `sequence`, `predecessor`
- 함수/상태: `deliveryState`, `DeliveryState`
- 관련 Thread: 13

---

<a id="p11"></a>
## [Thread 14 / `feat(commands): define acknowledged completion CAS`] publish 직전 재검사와 오래된 reopen 차단

### 면접 질문

reopen action이 접수된 뒤 Kafka publish 전이나 broker ack 후 completion 전까지 더 최신 `CLOSED`, terminal latch, feed hold가 생길 수 있습니다. 오래된 reopen이 새 제한을 지우지 못하게 한 방법을 설명해 주세요.

꼬리 질문:

- delivery decision과 completion을 왜 둘 다 Redis script로 검사합니까?
- 접수 당시 READY였던 reopen을 publish 시점에 SKIP해야 하는 경우는 무엇입니까?
- `tail != sequence`인데 predecessor는 맞는 경우 completion 결과는 무엇이어야 합니까?
- reopen 완료 시 feed hold까지 지워도 됩니까?
- completion이 성공했지만 Stream cleanup 전에 crash하면 어떤 재처리가 일어납니까?

### 30초 모범 답변

접수 시 검사는 충분하지 않습니다. queue에서 기다리는 동안 terminal이나 더 최신 close가 생길 수 있어 publish 직전 delivery decision이 현재 committed, tail, terminal, provider, feed hold를 다시 봅니다. ack 후 completion도 CAS로 predecessor를 확인하고 committed를 전진시킵니다. action이 현재 tail이 아니면 `SUPERSEDED`로 완료만 기록하고 projection 제한은 건드리지 않습니다. reopen이 여전히 최신일 때만 operator override를 제거하며, feed hold가 있으면 effective는 계속 `SUSPENDED`입니다. 따라서 오래된 완화 전이가 새 제한을 지우지 못합니다.

### 답변 핵심 키워드

check-at-use-time, delivery decision, completion CAS, committed vs tail, superseded, terminal recheck, safe reopen, feed hold preservation

### 백지 구현

**구현 목표**

acknowledged action 한 건의 completion을 pure state transition으로 구현한다. 최신성·terminal·feed hold invariant를 모두 지켜야 한다.

**인터페이스**

```java
record AcknowledgedAction(
    String actionId,
    long sequence,
    long predecessor,
    RequestedStatus requestedStatus) {}

record CompletionState(
    long committed,
    long tail,
    MarketStatus providerStatus,
    MarketStatus operatorOverride, // 없으면 null
    boolean feedHeld,
    boolean eventTerminal,
    boolean marketTerminal) {}

enum CompletionOutcome {
  APPLIED, SUPERSEDED, COMPLETED, BLOCKED
}

record CompletionResult(
    CompletionOutcome outcome,
    CompletionState newState,
    MarketStatus effectiveStatus) {}

final class OperatorCompletion {
  static CompletionResult complete(
      CompletionState state,
      AcknowledgedAction action) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: ack된 action과 현재 market completion/projection 상태
- 출력: outcome, 새 상태, 최종 effective status

**반드시 만족해야 할 조건**

- `committed >= sequence`: 멱등 `COMPLETED`
- `committed != predecessor`: `BLOCKED`, no mutation
- 유효 completion은 committed를 sequence까지 전진
- `tail != sequence`: `SUPERSEDED`; 최신 override를 제거하거나 덮지 않음
- terminal이면 effective는 항상 `CLOSED`
- 최신 restrictive action은 해당 제한을 유지
- 최신 reopen만 operator override 제거 가능
- reopen 뒤 feed hold가 있으면 effective `SUSPENDED`
- provider `CLOSED`를 reopen으로 이기지 못함
- 입력 상태를 수정하지 않음

**경계 조건**

- 동일 action completion 재호출
- predecessor보다 committed가 더 앞선 경우
- ack 뒤 terminal 발생
- reopen 뒤 더 최신 close 접수
- feed hold와 provider `OPEN`
- provider `CLOSED`인데 terminal key가 아직 없음
- tail/committed가 비정상적으로 역전

**실패 조건**

- BLOCKED 상태에서 committed 변경
- superseded reopen이 override 삭제
- terminal인데 `OPEN` 반환
- feed hold를 reopen과 함께 삭제
- completion 결과와 effective registry가 서로 다른 의미

**필요한 제약**

- 25~30분
- Redis script를 직접 작성하지 않고 state machine으로 축소
- 모든 outcome의 상태 변화 표를 테스트로 표현

### 구현 후 자가 검증

- [ ] 정상 sequence 1 completion이 committed를 1로 전진
- [ ] 동일 completion 재호출은 `COMPLETED`
- [ ] predecessor 미완료는 `BLOCKED`, state 불변
- [ ] 뒤에 close가 접수된 오래된 reopen은 `SUPERSEDED`
- [ ] superseded reopen 후 `CLOSED` override 유지
- [ ] terminal 이후 reopen completion도 effective `CLOSED`
- [ ] 최신 reopen이 override를 제거하되 feed hold는 유지
- [ ] feed hold가 없고 provider `OPEN`일 때만 실제 `OPEN`
- [ ] provider `CLOSED`는 reopen으로 열리지 않음
- [ ] 입력/출력 state가 mutable alias를 공유하지 않음

### 구현 후 설명할 것

1. delivery decision과 completion CAS를 분리한 이유
2. publish가 이미 끝난 superseded action도 committed를 전진해야 하는 이유
3. override와 feed hold가 서로 다른 원인 상태인 이유
4. completion 후 Stream cleanup을 별도 단계로 둔 at-least-once 영향
5. Redis Lua CAS와 데이터베이스 optimistic locking의 공통점

### 원본 확인 위치

- Thread 14
- 커밋: `feat(commands): define acknowledged completion CAS`
- 커밋: `feat(commands): evaluate queued operator actions`
- 파일/컴포넌트: `OperatorCompletionScript`, `OperatorDeliveryDecisionScript`, `OperatorActionQueue`, `OperatorDeliveryDecision`, `OperatorActionProcessor`
- 함수: `deliveryDecision`, `complete`, `cleanup`
- 관련 Thread: 05, 10, 13
