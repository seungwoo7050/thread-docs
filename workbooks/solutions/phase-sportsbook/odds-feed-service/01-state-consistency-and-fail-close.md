# 상태 정합성·fail-close 워크북

이 문서는 Redis projection, Kafka acknowledgement, terminal lifecycle에 반복해서 등장하는 하나의 핵심 원칙을 다룬다.

> **더 제한적인 상태는 늦은 완화 전이나 전달 실패가 되돌리지 못해야 한다.**

<a id="p01"></a>
## [Thread 05 / `feat(cache): project feed availability holds`] 상태 우선순위와 원자적 유효 상태 계산

### 면접 질문

`RedisOddsCache`에는 provider 상태, 운영자 override, feed hold, event/market terminal latch가 동시에 존재합니다. 단순히 마지막으로 도착한 값을 `SET`하지 않고 명시적인 우선순위를 계산한 이유는 무엇입니까?

꼬리 질문:

- provider가 `OPEN`을 늦게 보내도 terminal market이 열리지 않아야 하는 invariant를 어떻게 표현하겠습니까?
- 운영자가 `OPEN`을 요청했지만 feed hold가 남아 있으면 effective 상태는 무엇이어야 합니까?
- 여러 Redis key를 Java에서 순서대로 갱신하는 방식과 Lua 원자 갱신의 차이는 무엇입니까?
- terminal latch에 일반 projection TTL을 적용하면 어떤 문제가 생깁니까?

### 30초 모범 답변

이 상태는 last-write-wins가 아니라 **안전 우선순위**로 계산해야 합니다. event나 market이 terminal이면 항상 `CLOSED`, 그다음 운영자 제한, 그다음 feed hold의 `SUSPENDED`, 마지막이 provider 상태입니다. 특히 terminal은 늦은 `OPEN`이나 odds 갱신으로 되돌아가면 안 되는 단조 invariant입니다. 관련 key를 따로 갱신하면 중간 상태가 노출되므로 Redis Lua 한 번으로 원본 상태, effective 상태, registry를 함께 갱신했습니다. 대가로 script가 복잡해지지만 금전적 오류보다 fail-close를 우선했습니다.

### 답변 핵심 키워드

terminal monotonicity, restrictive precedence, effective state, stale update, Redis Lua, atomic projection, fail-close, durable latch

### 백지 구현

**구현 목표**

여러 상태 원천을 받아 하나의 effective market 상태를 결정하는 순수 함수를 구현한다. Redis나 Spring 코드는 작성하지 않는다.

**인터페이스**

```java
enum MarketStatus {
  OPEN, SUSPENDED, CLOSED
}

record MarketSnapshot(
    boolean eventTerminal,
    boolean marketTerminal,
    MarketStatus providerStatus,
    MarketStatus operatorOverride, // 없으면 null
    boolean feedHeld) {}

final class MarketStateResolver {
  static MarketStatus resolve(MarketSnapshot snapshot) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: terminal 여부, provider 상태, 선택적 운영자 override, feed hold 여부
- 출력: 외부 조회와 발행 판단에 사용할 단 하나의 effective `MarketStatus`

**반드시 만족해야 할 조건**

- event 또는 market terminal이면 다른 입력과 무관하게 `CLOSED`
- terminal이 아닌 상태에서 운영자 `CLOSED`/`SUSPENDED`가 provider와 feed hold보다 우선
- override가 없고 feed hold가 있으면 `SUSPENDED`
- 위 조건이 없으면 provider 상태 사용
- 같은 입력은 항상 같은 출력
- 입력 객체를 변경하지 않음

**경계 조건**

- provider 상태가 `CLOSED`인 경우
- override가 `OPEN`으로 전달되는 경우를 허용할지, "override 제거"로만 표현할지 명시
- event terminal과 market terminal이 동시에 참인 경우
- 모든 제한 원천이 동시에 존재하는 경우

**실패 조건**

- 필수 provider 상태가 없음
- override에 허용하지 않은 값이 들어옴
- 상태 enum 이외의 문자열을 외부에서 바로 전달함

**필요한 제약**

- 10~15분 안에 구현 가능한 순수 함수
- 우선순위가 `if` 순서에 암묵적으로만 숨어 있지 않도록 테스트 이름이나 주석으로 invariant를 드러낼 것

### 구현 후 자가 검증

- [ ] provider `OPEN`만 있을 때 `OPEN`
- [ ] feed hold가 provider `OPEN`을 `SUSPENDED`로 제한
- [ ] 운영자 `CLOSED`가 feed hold와 provider를 모두 이김
- [ ] terminal이 모든 override를 이기고 `CLOSED`
- [ ] terminal 이후 provider `OPEN` 입력을 다시 넣어도 결과가 변하지 않음
- [ ] 입력 객체를 수정하거나 전역 상태에 의존하지 않음
- [ ] 각 우선순위 조합 테스트가 한 가지 invariant를 명확히 검증함
- [ ] 시간·공간 복잡도가 모두 상수임

### 구현 후 설명할 것

1. 왜 `OPEN` override를 값으로 저장하기보다 override 삭제로 표현하는 편이 단순한지
2. pure resolver와 Redis 원자 갱신 script의 책임을 어떻게 나눌지
3. terminal latch를 일반 cache TTL과 분리해야 하는 이유
4. 새로운 상태 원천이 추가될 때 우선순위를 어디에 명시할지
5. fail-open 대신 fail-close를 선택한 도메인 비용

### 원본 확인 위치

- Thread 05
- 커밋: `feat(cache): project feed availability holds`
- 커밋: `feat(cache): preserve terminal market closures`
- 파일/컴포넌트: `RedisOddsCache`, `PROJECT_LATEST_ODDS`, `CacheKeys.eventTerminal`, `CacheKeys.marketTerminal`
- 함수: `holdLatestOdds`, `projectLatestOdds`, `storeProviderMarketStatus`, `storeOdds`
- 관련 Thread: 08, 10, 13, 14

---

<a id="p02"></a>
## [Thread 08 / `feat(feed): project acknowledged odds`] Kafka ack 이후에만 projection을 완화하는 흐름

### 면접 질문

새 odds를 Kafka로 발행하는 과정과 Redis current projection 갱신의 순서를 설명해 주세요. 왜 broker가 불안정할 때 최신 odds를 Redis에 먼저 노출하지 않고 market을 hold해야 합니까?

꼬리 질문:

- `KafkaTemplate.send()`가 future를 반환한 시점과 broker acknowledgement 시점은 어떻게 다릅니까?
- 기존 feed hold가 있는 상태에서 threshold 미만 변화가 들어오면 어떻게 복구 snapshot을 보장합니까?
- publish 성공 뒤 Redis projection 갱신이 실패하면 어떤 불일치가 남습니까?
- 오래된 release 이벤트가 더 최신 hold를 지우지 못하게 하려면 어떤 값이 필요합니까?

### 30초 모범 답변

외부 consumer가 받지 못한 가격을 current API에 먼저 노출하면 두 관측면이 갈라집니다. 그래서 publisher가 unhealthy이거나 ack 대기가 실패하면 최신 가격과 관측 시각을 hold로 기록하고 effective 상태를 `SUSPENDED`로 유지합니다. hold가 있으면 다음 변화는 threshold를 우회한 복구 snapshot으로 발행하고, broker ack 뒤에만 projection과 hold 해제를 원자적으로 적용합니다. 또한 hold 시각보다 오래된 update는 release하지 못하게 비교해 out-of-order 복구를 막습니다.

### 답변 핵심 키워드

ack-before-project, dual-write boundary, feed hold, forced snapshot, observedAt, stale release rejection, broker health, fail-close

### 백지 구현

**구현 목표**

publisher와 projection store를 port로 두고, odds update 한 건을 안전하게 처리하는 coordinator를 구현한다. 실제 Kafka·Redis 코드는 작성하지 않는다.

**인터페이스**

```java
record OddsUpdate(
    String eventId,
    String marketId,
    String selectionId,
    BigDecimal previousOdds,
    BigDecimal newOdds,
    Instant observedAt) {}

enum PublishOutcome {
  ACKNOWLEDGED,
  SUPPRESSED
}

interface AckPublisher {
  PublishOutcome publish(OddsUpdate update, boolean forceSnapshot) throws Exception;
  boolean isHealthy();
}

interface ProjectionStore {
  boolean isFeedHeld(String eventId, String marketId);
  void hold(OddsUpdate update);
  void projectAndRelease(OddsUpdate update);
  void projectWithoutRelease(OddsUpdate update);
}

final class AcknowledgedOddsCoordinator {
  void handle(OddsUpdate update, AckPublisher publisher, ProjectionStore store) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: odds update, publisher 상태와 결과, 현재 feed hold 여부
- 출력: 반환값보다 port 호출 순서와 최종 상태가 핵심

**반드시 만족해야 할 조건**

- publisher가 unhealthy이면 publish를 시도하지 않고 hold
- publish 예외 또는 ack 실패이면 hold
- hold 중에는 `forceSnapshot=true`
- hold 중 복구 snapshot이 ack되어야만 `projectAndRelease`
- hold가 없고 정책상 publish가 억제된 경우에만 `projectWithoutRelease` 허용
- projection 완화 호출은 acknowledgement보다 먼저 일어나지 않음
- 동일 update 재처리가 안전하도록 store port의 idempotency 전제를 문서화

**경계 조건**

- publisher health가 확인 직후 바뀌는 경우
- threshold 억제와 hold가 동시에 있는 경우
- ack 성공 뒤 projection 실패
- 동일 `observedAt`의 중복 update
- 더 오래된 update가 나중에 도착하는 경우

**실패 조건**

- publisher timeout·execution failure
- thread interruption
- projection store failure
- 잘못된 odds 값 또는 필수 ID 누락

**필요한 제약**

- 20~30분
- coordinator는 Kafka/Redis 세부 API에 의존하지 않음
- port 호출 순서를 검증할 수 있도록 단위 테스트 가능한 구조

### 구현 후 자가 검증

- [ ] unhealthy publisher 경로에서 publish 호출이 없음
- [ ] publish 실패 후 market이 hold됨
- [ ] 정상 상태의 threshold 억제는 불필요한 Kafka 발행을 하지 않음
- [ ] hold 상태에서는 threshold와 무관하게 복구 snapshot을 강제
- [ ] ack보다 먼저 `projectAndRelease`가 호출되지 않음
- [ ] ack 성공 뒤에만 hold가 해제됨
- [ ] stale timestamp release를 store가 거부한다는 계약이 드러남
- [ ] ack 성공·projection 실패의 재시도/중복 가능성을 숨기지 않음
- [ ] 예외가 health와 도메인 상태에 각각 어떤 영향을 주는지 테스트함

### 구현 후 설명할 것

1. Kafka와 Redis를 하나의 ACID transaction으로 묶지 않은 이유
2. ack 성공 후 projection 실패가 만드는 복구 전략
3. health probe와 도메인 feed hold를 별개로 유지하는 이유
4. threshold 억제와 복구 snapshot을 분리한 이유
5. `observedAt` 비교가 sequence 번호보다 적합하거나 부족한 상황

### 원본 확인 위치

- Thread 08
- 커밋: `feat(feed): project acknowledged odds`
- 커밋: `feat(feed): suspend markets during broker outages`
- 파일/컴포넌트: `FeedOrchestrator`, `OddsFeedPublisher`, `RedisOddsCache`, `BrokerAvailability`
- 함수: `FeedOrchestrator.handleOdds`, `RedisOddsCache.projectLatestOdds`, `holdLatestOdds`, `isFeedHeld`
- 관련 Thread: 05, 06, 15

---

<a id="p03"></a>
## [Thread 10 / `test(delivery): verify restrictive-first ordering`] 제한 전이와 완화 전이의 비대칭 순서

### 면접 질문

market `SUSPENDED`/`CLOSED`와 `OPEN`을 같은 순서로 처리하지 않은 이유를 설명해 주세요. durable queue 기록, Redis effective 상태, Kafka 발행은 각각 언제 일어나야 합니까?

꼬리 질문:

- restrictive event의 queue enqueue가 실패했는데 cache만 닫히면 어떤 문제가 생깁니까?
- 반대로 cache를 먼저 닫고 enqueue하면 더 안전해 보이는데 왜 기록 유실 위험을 함께 봐야 합니까?
- `OPEN`을 cache에 먼저 반영했는데 Kafka가 실패하면 어떤 관측이 생깁니까?
- 동일 transition이 재처리될 때 previous/effective 상태를 어떻게 다룹니까?

### 30초 모범 답변

제한 전이는 늦어지는 것보다 유실되는 것이 위험하고, 완화 전이는 너무 빨리 보이는 것이 위험합니다. 그래서 provider의 suspend/close는 먼저 durable queue에 기록이 성공한 뒤 cache를 fail-close로 제한합니다. queue 기록 자체가 실패하면 부분 상태를 만들지 않습니다. 반면 `OPEN`은 queue에만 넣고 current projection은 유지하며, processor가 Kafka ack를 받은 뒤에만 엽니다. 이 비대칭으로 전달 장애 시에도 시장이 잘못 열리는 방향으로 실패하지 않습니다.

### 답변 핵심 키워드

restrictive-first, deferred OPEN, durable intent, no partial mutation, asymmetric risk, broker ack, fail-close

### 백지 구현

**구현 목표**

transition 종류에 따라 durable 기록과 projection 변경 순서를 제어하는 작은 application service를 구현한다.

**인터페이스**

```java
record MarketTransition(
    String eventId,
    String marketId,
    MarketStatus previousStatus,
    MarketStatus requestedStatus,
    Instant occurredAt) {}

interface DurableTransitionQueue {
  String enqueue(MarketTransition transition);
}

interface MarketProjection {
  void restrict(MarketTransition transition);
  void openAfterAcknowledgement(MarketTransition transition);
}

record Acceptance(String recordId, boolean projectionChanged) {}

final class CriticalTransitionAcceptor {
  Acceptance accept(
      MarketTransition transition,
      DurableTransitionQueue queue,
      MarketProjection projection) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 현재/요청 상태를 가진 transition과 두 port
- 출력: durable record ID와 접수 시 projection 변경 여부

**반드시 만족해야 할 조건**

- `SUSPENDED`/`CLOSED`: enqueue 성공 후에만 `restrict`
- restrictive enqueue 실패 시 projection 호출 없음
- `OPEN`: enqueue만 하고 접수 단계에서 projection을 열지 않음
- unsupported transition은 side effect 전에 거부
- queue record ID가 없으면 성공으로 취급하지 않음
- 같은 입력 재시도 시 queue의 idempotency 계약을 요구

**경계 조건**

- 이미 `CLOSED`인 market에 `SUSPENDED`
- 이미 `OPEN`인 market에 `OPEN`
- terminal market의 `OPEN`
- queue 성공 후 projection 실패
- projection 제한 중 event terminal이 생기는 경우

**실패 조건**

- queue unavailable
- projection unavailable
- invalid state transition
- record ID 누락

**필요한 제약**

- 15~20분
- side-effect 호출 순서를 recording fake로 검증할 수 있어야 함
- processor의 ack 이후 완화 로직은 이 문제 범위 밖이지만 계약을 명시

### 구현 후 자가 검증

- [ ] restrictive 정상 경로가 `enqueue → restrict`
- [ ] restrictive enqueue 실패 시 `restrict` 미호출
- [ ] `OPEN` 정상 접수 시 projection 미변경
- [ ] unsupported/terminal reopen이 queue 앞에서 차단되는지 정책을 명시
- [ ] queue 성공·projection 실패가 재처리 가능한 상태로 남음
- [ ] duplicate enqueue가 새 record를 만들지 않는다는 port 계약을 적음
- [ ] 상태 변화가 완화 방향으로 선반영되지 않음
- [ ] 호출 순서 테스트가 단순 결과값뿐 아니라 side effect 순서를 검증함

### 구현 후 설명할 것

1. "더 제한적인 상태"와 "더 완화된 상태"의 위험 비대칭
2. durable intent를 먼저 남기는 이유
3. queue 성공 후 cache 실패를 어떻게 reconciliation할지
4. at-least-once 전달이 previous status 정확도에 미치는 영향
5. command 접수의 원자 Lua 방식(Thread 13)과 provider transition 방식의 차이

### 원본 확인 위치

- Thread 10
- 커밋: `test(delivery): verify restrictive-first ordering`
- 파일/컴포넌트: `FeedOrchestrator`, `CriticalEventQueue`, `CriticalEventProcessor`, `RedisOddsCache`
- 함수: `FeedOrchestrator.handleMarketStatus`, `CriticalEventProcessor.applyMarketTransition`
- 관련 Thread: 05, 08, 09, 13

---

<a id="p04"></a>
## [Thread 05, 10 / `feat(cache): fail close registered event markets`] terminal fan-out, snapshot, 단조 종료

### 면접 질문

경기가 `FINISHED`, `CANCELLED`, `POSTPONED` 같은 terminal 상태가 될 때 등록된 모든 market을 어떻게 닫고, 어떤 정보를 durable event에 담아야 합니까?

꼬리 질문:

- JVM의 현재 subscription 목록 대신 Redis market registry를 사용한 이유는 무엇입니까?
- market을 먼저 닫은 뒤 previous status를 읽으면 어떤 정보를 잃습니까?
- terminal lifecycle 재처리 시 동일 market closure가 다시 발행될 수 있는데 허용 가능한가요?
- 결과 조회가 실패하거나 아직 없을 때 terminal 처리 전체를 지연해야 합니까?

### 30초 모범 답변

terminal 전이는 market 전체에 퍼지는 aggregate 전이입니다. 먼저 Redis registry에서 아직 닫히지 않은 market과 이전 상태를 snapshot해 durable terminal envelope에 넣고, enqueue가 성공하면 event·market terminal latch와 effective `CLOSED`를 원자적으로 반영합니다. registry가 Redis에 있어 process 재시작 뒤에도 inventory를 복구할 수 있습니다. processor는 envelope만으로 lifecycle과 각 market closure를 재발행할 수 있고, 중복은 가능하지만 terminal reopen은 불가능합니다. 결과는 가능한 경우 함께 snapshot하되 종료 안전성을 깨지 않도록 다룹니다.

### 답변 핵심 키워드

aggregate fan-out, registry source of truth, snapshot-before-mutation, terminal latch, idempotent closure, durable envelope, at-least-once

### 백지 구현

**구현 목표**

terminal lifecycle 입력과 등록 market snapshot을 받아, mutation 전에 보존해야 할 정보를 담은 deterministic terminal plan을 만든다. 실제 Redis·Kafka 코드는 작성하지 않는다.

**인터페이스**

```java
record MarketId(String value) {}

record MatchOutcome(
    String score,
    Map<String, String> detail,
    Instant settledAt) {}

record TerminalLifecycle(
    String eventId,
    String terminalStatus,
    Instant occurredAt) {}

record MarketClosure(MarketId marketId, MarketStatus previousStatus) {}

record TerminalPlan(
    TerminalLifecycle lifecycle,
    List<MarketClosure> closures,
    MatchOutcome outcome) {} // 없으면 null

final class TerminalPlanner {
  static TerminalPlan plan(
      TerminalLifecycle lifecycle,
      Map<MarketId, MarketStatus> registeredMarkets,
      MatchOutcome outcome) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: terminal lifecycle, 현재 registry snapshot, 선택적 결과
- 출력: 이후 queue enqueue와 atomic close에 사용할 immutable plan

**반드시 만족해야 할 조건**

- terminal status만 허용
- 이미 `CLOSED`인 market은 새 closure 목록에서 제외
- 각 market의 previous status를 mutation 전에 보존
- market ID 기준 deterministic 순서
- 입력 map과 outcome detail을 방어적으로 복사
- 같은 snapshot으로 여러 번 호출해도 같은 plan
- 빈 registry도 정상 처리

**경계 조건**

- 모든 market이 이미 `CLOSED`
- registry가 비어 있음
- 같은 ID가 입력 표현상 중복될 수 있는 경우
- `FINISHED`지만 outcome이 없음
- `CANCELLED`/`POSTPONED`에 outcome이 들어온 경우 정책
- 매우 많은 market을 한 script에서 닫을 때 실행 시간

**실패 조건**

- non-terminal lifecycle
- null ID/status
- 잘못된 previous status
- mutable outcome detail이 생성 후 변경됨

**필요한 제약**

- 15~25분
- 계획 생성만 구현하고 queue/Kafka/Redis 실행은 설명으로 분리
- 정렬의 시간 복잡도를 설명할 것

### 구현 후 자가 검증

- [ ] OPEN/SUSPENDED market만 closure에 포함
- [ ] previous status가 정확히 보존됨
- [ ] 입력 map iteration 순서와 무관하게 출력 순서가 안정적
- [ ] 빈 registry가 빈 closure plan을 생성
- [ ] 이미 CLOSED인 market을 중복 closure로 만들지 않음
- [ ] outcome detail을 외부에서 수정해도 plan이 변하지 않음
- [ ] non-terminal status가 side effect 없이 거부됨
- [ ] 시간 복잡도가 정렬 사용 시 `O(n log n)`임
- [ ] 실제 적용 단계에서는 enqueue 성공 뒤 atomic close가 필요함을 설명 가능

### 구현 후 설명할 것

1. snapshot-before-mutation이 previous status 이벤트에 필요한 이유
2. Redis registry가 local memory보다 복구에 유리한 이유
3. 한 Lua script fan-out의 원자성 이점과 큰 aggregate에서의 실행 시간 위험
4. 중복 terminal event를 허용하는 at-least-once 설계와 downstream idempotency
5. result 부재를 terminal closure와 분리할지 함께 재시도할지의 trade-off

### 원본 확인 위치

- Thread 05
- 커밋: `feat(cache): fail close registered event markets`
- Thread 10
- 커밋: `test(delivery): verify terminal delivery ordering`
- 커밋: `feat(feed): snapshot registered terminal markets`
- 파일/컴포넌트: `RedisOddsCache`, `CLOSE_EVENT_MARKETS`, `FeedOrchestrator`, `CriticalEvent`, `CriticalEventProcessor`
- 함수: `closeEventMarkets`, `getRegisteredMarkets`, `FeedOrchestrator.handleLifecycle`, `CriticalEventProcessor.applyLifecycle`
- 관련 Thread: 09
