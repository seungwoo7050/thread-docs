# 동시성·결정성·lifecycle 워크북

이 문서는 실시간 feed를 오래 실행할 때 발생하는 세 종류의 어려움을 묶는다.

- 구독 객체의 생성·종료·교체가 경쟁하는 **resource ownership**
- 외부 장애 중 재시도가 몰리는 **retry control**
- 시간과 난수를 통제해 동일 실패를 재현하는 **deterministic execution**

<a id="p12"></a>
## [Thread 07 / `feat(feed): manage provider subscriptions`] 단일 구독 소유권과 race-safe cleanup

### 면접 질문

event별 provider stream을 `ConcurrentHashMap<EventId, Disposable>`로 관리할 때 어떤 race가 발생할 수 있습니까? 특히 구독이 map에 들어가기 전에 동기 종료되거나, 오래된 구독의 `doFinally`가 새 구독을 제거하는 상황을 설명해 주세요.

꼬리 질문:

- `containsKey` 후 `put`을 따로 호출하면 왜 중복 구독이 생깁니까?
- `doFinally(() -> subscriptions.remove(eventId))`가 왜 위험합니까?
- `putIfAbsent`에서 패한 구독은 어떻게 정리해야 합니까?
- application shutdown 시 모든 `Disposable`을 어떤 순서로 처리합니까?
- provider stream이 retry 중일 때 map에는 무엇이 들어 있어야 합니까?

### 30초 모범 답변

구독은 event별로 한 객체만 소유해야 하지만 subscribe 자체가 side effect라 map 연산만 원자적이어도 충분하지 않습니다. 두 thread가 동시에 subscribe할 수 있고, stream이 동기 종료되면 disposable을 map에 넣기 전에 cleanup callback이 실행될 수 있습니다. 또 오래된 callback이 key만 보고 remove하면 교체된 새 구독까지 지웁니다. 그래서 종료 여부와 자기 disposable 참조를 추적하고, `remove(key, sameDisposable)`로 자기 소유분만 제거하며, `putIfAbsent`에서 진 구독은 즉시 dispose합니다. shutdown에서는 map을 비우며 모든 disposable을 정리합니다.

### 답변 핵심 키워드

resource ownership, subscribe side effect, putIfAbsent, synchronous termination, compare-and-remove, loser disposal, `doFinally`, `@PreDestroy`

### 백지 구현

**구현 목표**

event별 active subscription을 최대 하나로 유지하는 registry를 구현한다. 실제 provider 로직은 factory로 주입한다.

**인터페이스**

```java
record EventId(String value) {}

interface Subscription {
  void dispose();
  boolean isDisposed();
}

interface SubscriptionFactory {
  Subscription subscribe(EventId eventId, Runnable onFinally);
}

final class EventSubscriptionRegistry implements AutoCloseable {
  private final ConcurrentHashMap<EventId, Subscription> active =
      new ConcurrentHashMap<>();

  boolean startIfAbsent(
      EventId eventId,
      SubscriptionFactory factory) {
    // 직접 구현
  }

  int activeCount() {
    // 직접 구현
  }

  @Override
  public void close() {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: event ID와 subscription factory
- 출력: 이 호출이 최종 active subscription을 설치했는지 여부

**반드시 만족해야 할 조건**

- event별 active subscription 최대 1개
- 두 thread가 동시에 호출해도 최종 승자 하나
- 설치 경쟁에서 진 subscription은 dispose
- `onFinally`는 자기 subscription이 현재 map 값일 때만 제거
- subscribe 중 동기 종료되어도 종료된 subscription을 map에 남기지 않음
- `close()` 후 모든 active subscription dispose
- cleanup과 close가 동시에 실행돼도 예외 없이 수렴
- null ID/factory 거부

**경계 조건**

- factory가 subscribe 중 예외
- factory가 callback을 동기 실행한 뒤 subscription 반환
- 반환 직후 이미 disposed
- old subscription 종료와 replacement 설치 동시 발생
- `close()`와 `startIfAbsent()` 경쟁
- `dispose()`가 여러 번 호출되어도 안전한지에 대한 계약

**실패 조건**

- `containsKey`/`put` check-then-act
- key만으로 unconditional remove
- 경쟁 패배 subscription 누수
- shutdown 뒤 새 subscription 허용 여부가 불명확
- factory 예외 후 placeholder 잔존

**필요한 제약**

- 25~30분
- lock-free 또는 짧은 critical section 중 하나를 선택하고 이유 설명
- 테스트에서 barrier/latch를 사용해 경쟁을 재현할 수 있어야 함

### 구현 후 자가 검증

- [ ] 단일 thread에서 최초 start 성공, 두 번째 false
- [ ] 다중 thread 동시 start 후 active 1개
- [ ] 패자 subscription이 dispose됨
- [ ] 오래된 callback이 replacement를 제거하지 않음
- [ ] 동기 종료 subscription이 map에 남지 않음
- [ ] factory 예외 후 active state 불변
- [ ] `close()`가 모든 subscription을 정확히 한 번 이상 dispose
- [ ] close/cleanup 경쟁 뒤 active 0
- [ ] resource cleanup이 callback 실행 순서에 의존하지 않음

### 구현 후 설명할 것

1. map의 원자 연산과 subscribe side effect를 하나로 묶기 어려운 이유
2. identity 기반 `remove(key, value)`가 필요한 이유
3. synchronized 방식과 CAS/atomic-reference 방식의 trade-off
4. shutdown 이후 신규 start 정책
5. Reactor `doFinally`가 complete/error/cancel 모두에서 필요한 이유

### 원본 확인 위치

- Thread 07
- 커밋: `feat(feed): manage provider subscriptions`
- 파일/컴포넌트: `FeedOrchestrator`
- 상태/타입: `ConcurrentHashMap<EventId, Disposable>`, `AtomicBoolean`, `AtomicReference`
- lifecycle 경계: `doFinally`, `@PreDestroy`
- 관련 Thread: 02, 15

---

<a id="p13"></a>
## [Thread 07 / `feat(feed): retry failed provider streams`] bounded exponential backoff와 jitter

### 면접 질문

provider stream 재접속에 1초부터 시작해 30초 상한의 exponential backoff와 jitter를 사용한 이유를 설명해 주세요. fixed delay나 즉시 무한 재시도와 비교해 주세요.

꼬리 질문:

- 여러 instance가 동시에 끊겼을 때 jitter가 없으면 무슨 일이 생깁니까?
- attempt가 매우 커질 때 duration overflow를 어떻게 막습니까?
- 재접속 성공 뒤 다음 장애의 backoff는 어디서 다시 시작해야 합니까?
- 인증 오류처럼 재시도해도 의미 없는 오류를 어떻게 분류하겠습니까?
- 무한 retry 중 readiness는 어떤 상태여야 합니까?

### 30초 모범 답변

즉시 재시도는 장애 난 provider와 자기 scheduler를 동시에 압박하고, fixed delay는 짧은 장애 복구와 장기 장애 부하를 같이 만족시키기 어렵습니다. exponential backoff는 초기 복구는 빠르게, 반복 실패는 느리게 하며 30초 상한으로 복구 지연을 제한합니다. jitter는 여러 구독이나 instance가 같은 순간에 다시 몰리는 thundering herd를 줄입니다. 성공한 새 subscription은 새로운 retry cycle이므로 attempt를 초기화하고, 영구 오류는 별도 분류가 필요합니다.

### 답변 핵심 키워드

exponential backoff, max cap, jitter, thundering herd, retry classification, overflow, reset-on-success, scheduler pressure

### 백지 구현

**구현 목표**

attempt 번호와 난수 공급자를 받아 bounded jittered delay를 계산하는 순수 함수를 구현한다.

**인터페이스**

```java
interface UnitRandom {
  double nextDouble(); // [0.0, 1.0)
}

final class RetryDelay {
  static Duration nextDelay(
      int attempt,
      Duration initial,
      Duration maximum,
      double jitterRatio,
      UnitRandom random) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 0부터 시작하는 attempt, initial/max delay, jitter 비율, 주입 난수
- 출력: 0 이상 max 이하의 delay

**반드시 만족해야 할 조건**

- jitter 전 base delay는 지수 증가
- base delay가 maximum을 넘지 않음
- jitter 결과도 0 미만이나 maximum 초과 금지
- 같은 난수 sequence와 입력은 같은 결과
- attempt 증가 중 overflow 없음
- jitter ratio 허용 범위 검증
- initial이 maximum보다 큰 경우 정책 명시

**경계 조건**

- attempt 0
- 매우 큰 attempt
- initial 0
- jitter 0
- jitter 1
- random이 0 또는 1에 매우 가까움
- nanosecond 단위 Duration

**실패 조건**

- `Math.pow` 결과를 무검사로 long 변환
- 음수 delay
- max 상한 초과
- 전역 `Random`에 의존해 테스트 비결정적
- invalid configuration 묵인

**필요한 제약**

- 10~15분
- retry 실행이나 sleep은 구현하지 않음
- 정수 overflow를 방어할 것

### 구현 후 자가 검증

- [ ] attempt 0의 base가 initial
- [ ] jitter 0일 때 정확한 지수 backoff
- [ ] max에 도달한 뒤 더 증가하지 않음
- [ ] 매우 큰 attempt에도 overflow/예외 없음
- [ ] 난수 하한·상한에서 결과 범위 유지
- [ ] 주입 난수로 테스트가 결정적
- [ ] invalid negative duration/jitter 거부
- [ ] 성공 후 attempt reset은 caller 책임임을 계약에 명시

### 구현 후 설명할 것

1. fixed delay 대비 exponential backoff의 장점
2. full jitter, equal jitter 등 정책 차이
3. max backoff가 너무 크거나 작을 때 영향
4. retry 가능한 오류와 영구 오류 분류
5. retry 중 subscription ownership과 readiness 처리

### 원본 확인 위치

- Thread 07
- 커밋: `feat(feed): retry failed provider streams`
- 커밋: `test(feed): verify bounded retry backoff`
- 파일/컴포넌트: `FeedOrchestrator`
- 사용 API/값: `Retry.backoff`, initial 1초, max 30초, jitter 0.2
- 관련 Thread: 15

---

<a id="p14"></a>
## [Thread 07 / `feat(feed): discover and seed provider events`] 재시작 시 cached projection 우선 hydration

### 면접 질문

application 시작 시 provider가 반환한 event summary와 Redis에 남아 있는 cached summary가 다를 때 어느 쪽을 catalog에 넣어야 합니까? 왜 provider 값을 무조건 최신으로 보면 안 됩니까?

꼬리 질문:

- Redis에는 terminal 상태가 있고 provider는 `SCHEDULED`를 다시 주면 어떻게 합니까?
- catalog `putIfAbsent`와 cache write 순서가 왜 중요합니까?
- 두 refresh thread가 같은 event를 동시에 발견하면 어떤 side effect가 생길 수 있습니까?
- malformed cached JSON은 fail-fast, skip, fallback 중 무엇이 적절합니까?
- cache TTL 만료 뒤 provider만 남는 경우 어떤 상태가 손실될 수 있습니까?

### 30초 모범 답변

시작 직후 provider summary는 외부 원본의 현재 snapshot일 수 있지만, 이 서비스가 이전에 처리한 terminal·운영 상태보다 덜 진행된 값일 수 있습니다. 그래서 catalog에 없는 event를 발견하면 Redis cached summary가 있으면 그것을 먼저 hydrate하고, cache가 없을 때만 provider summary를 seed합니다. `putIfAbsent` 승자만 새 provider 값을 cache에 쓰면 중복 refresh도 idempotent합니다. 이 선택은 durable projection을 재시작 복구의 기준으로 삼는 대신 cache TTL과 손상 처리 정책을 중요하게 만듭니다.

### 답변 핵심 키워드

state hydration, durable projection, restart recovery, stale provider snapshot, putIfAbsent, idempotent discovery, cache corruption

### 백지 구현

**구현 목표**

provider summary 한 건을 catalog/cache에 seed하는 결정을 구현한다. 실제 serialization은 port 밖에 둔다.

**인터페이스**

```java
record EventSummary(
    String eventId,
    String status,
    Instant scheduledStartAt) {}

interface EventCache {
  Optional<EventSummary> get(String eventId);
  void put(EventSummary summary);
}

interface EventCatalog {
  boolean contains(String eventId);
  boolean putIfAbsent(EventSummary summary);
}

enum HydrationSource {
  ALREADY_PRESENT, CACHE, PROVIDER, LOST_RACE
}

record HydrationResult(
    HydrationSource source,
    EventSummary selected,
    boolean cacheWritten) {}

final class EventHydrator {
  HydrationResult hydrate(
      EventSummary providerSummary,
      EventCache cache,
      EventCatalog catalog) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: provider summary와 cache/catalog port
- 출력: 선택한 source, catalog 값, cache write 여부

**반드시 만족해야 할 조건**

- catalog에 이미 있으면 변경 없음
- cache hit이면 cached summary를 catalog 후보로 사용
- cache miss이면 provider summary 사용
- `putIfAbsent`에서 이긴 provider seed만 cache write
- cached summary를 선택했을 때 불필요하게 provider로 cache overwrite하지 않음
- 경쟁에서 진 호출은 기존 catalog를 덮지 않음
- provider summary 필수 필드 검증

**경계 조건**

- cache get 실패
- cache JSON 손상
- cache hit 후 다른 thread가 먼저 catalog 삽입
- provider와 cache kickoff/status 불일치
- cache TTL 만료
- 같은 event ID에 완전히 다른 teams 같은 identity 충돌

**실패 조건**

- provider 값으로 terminal cache 퇴행
- `contains` 결과만 믿고 비원자 put
- 경쟁에서 진 thread가 cache overwrite
- malformed cache를 조용히 provider로 덮어 원인 은폐

**필요한 제약**

- 15~20분
- port 호출 순서를 fake로 검증
- cache 오류 정책을 명시하되 한 가지를 선택해 일관되게 구현

### 구현 후 자가 검증

- [ ] catalog hit 경로에서 cache 접근/쓰기 최소화
- [ ] cache hit가 provider보다 우선
- [ ] cache miss가 provider를 catalog와 cache에 seed
- [ ] putIfAbsent 경쟁 패배 시 cache write 없음
- [ ] terminal cached summary가 scheduled provider로 바뀌지 않음
- [ ] cache failure 정책 테스트
- [ ] 동일 입력 반복 호출이 idempotent
- [ ] source/result가 실제 side effect와 일치

### 구현 후 설명할 것

1. durable cache를 복구 source로 선택한 이유
2. provider가 진짜 더 최신일 수 있는 경우 reconciliation 방법
3. cache TTL과 terminal latch 수명의 차이
4. corrupt cache를 fail-fast할지 격리할지
5. catalog와 Redis 사이 이중 저장의 일관성 한계

### 원본 확인 위치

- Thread 07
- 커밋: `feat(feed): discover and seed provider events`
- 커밋: `test(feed): verify discovery projections`
- 파일/컴포넌트: `FeedOrchestrator`, `RedisOddsCache`, `EventCatalog`
- 함수: `refresh`, `seedProjection`, `getEvent`, `storeEvent`, `putIfAbsent`
- 관련 Thread: 05

---

<a id="p15"></a>
## [Thread 02 / `feat(mock): seed deterministic event fixtures`] 난수 소비 경로 분리와 재현성

### 면접 질문

같은 seed를 썼는데도 odds tick 횟수가 달라지면 event ID나 최종 결과가 바뀌는 테스트 fixture는 왜 나쁜가요? `structureRandom`과 `oddsRandom`을 분리한 설계 의도를 설명해 주세요.

꼬리 질문:

- 하나의 `Random`을 모든 기능이 공유하면 새 기능 추가가 기존 golden test를 왜 깨뜨립니까?
- seed에 salt를 적용할 때 어떤 요구가 있습니까?
- deterministic UUID를 실제 business ID로 사용해도 됩니까?
- `Clock` 주입과 random seed 주입은 각각 어떤 비결정성을 제거합니까?
- seed 0을 특별값으로 두는 정책의 장단점은 무엇입니까?

### 30초 모범 답변

하나의 RNG를 공유하면 어느 기능이 난수를 몇 번 소비했는지가 전체 시뮬레이션 결과를 결정합니다. odds tick 로직 한 줄을 추가했는데 event ID나 match result까지 바뀌면 실패 재현과 테스트 의미가 무너집니다. 그래서 하나의 run seed에서 구조, odds 같은 용도별 독립 seed를 domain salt로 파생하고 각 전용 RNG만 사용합니다. 같은 seed와 clock이면 같은 구조·이벤트 순서를 만들고, 한 채널의 추가 소비가 다른 채널 결과를 바꾸지 않아야 합니다.

### 답변 핵심 키워드

deterministic fixture, RNG stream isolation, domain salt, reproducibility, random consumption coupling, injected Clock, stable UUID

### 백지 구현

**구현 목표**

하나의 run seed에서 구조·odds·result용 독립 난수 채널을 만들고, 한 채널의 추가 소비가 다른 채널 출력에 영향을 주지 않는 작은 simulation을 구현한다.

**인터페이스**

```java
record SimulationRandoms(
    RandomGenerator structure,
    RandomGenerator odds,
    RandomGenerator result) {}

final class SimulationSeeds {
  static SimulationRandoms fromRunSeed(long runSeed) {
    // 직접 구현
  }
}

record SimulationSnapshot(
    List<UUID> eventIds,
    List<BigDecimal> odds,
    String result) {}

final class DeterministicSimulation {
  static SimulationSnapshot run(
      long seed,
      int oddsTicks,
      Clock clock) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: run seed, odds tick 수, 고정 가능한 clock
- 출력: 구조 ID, odds sequence, 결과 snapshot

**반드시 만족해야 할 조건**

- 같은 seed/clock/tick 수는 동일 snapshot
- 구조 ID는 odds tick 수 변화와 무관
- 결과는 odds RNG 소비량 변화와 무관하다는 정책
- 서로 다른 domain seed가 명시적
- UUID 생성도 전용 structure RNG 사용
- 전역 RNG/현재 시각 사용 금지
- salt 값 충돌 방지 전략 설명

**경계 조건**

- seed 0
- 음수 seed
- oddsTicks 0
- 매우 많은 tick
- 같은 domain salt를 실수로 재사용
- concurrent simulation 실행

**실패 조건**

- 모든 채널이 같은 RNG 인스턴스 공유
- `UUID.randomUUID()` 사용
- `Instant.now()` 직접 호출
- 테스트가 호출 순서에 의존
- result가 odds tick 수에 따라 바뀜

**필요한 제약**

- 20~25분
- 실제 odds 수학은 단순한 bounded 값으로 축소 가능
- random generator를 외부에서 관찰 가능한 mutable singleton으로 두지 않음

### 구현 후 자가 검증

- [ ] 같은 seed 두 실행이 완전히 동일
- [ ] oddsTicks를 바꿔도 event IDs 동일
- [ ] oddsTicks를 바꿔도 result 동일
- [ ] seed가 달라지면 적어도 한 domain 결과가 달라짐
- [ ] structure RNG 추가 소비가 odds/result를 바꾸지 않음
- [ ] parallel run이 서로 간섭하지 않음
- [ ] fixed Clock으로 scheduled times 재현
- [ ] 전역 random/time 호출 없음

### 구현 후 설명할 것

1. 독립 RNG가 테스트 안정성을 높이는 이유
2. seed 파생에 단순 XOR를 사용할 때의 한계
3. cryptographic randomness와 simulation randomness의 차이
4. deterministic ID가 환경 간 fixture 연결에 주는 장점
5. 재현성 때문에 구현 변경 탐지가 어려워질 수 있는 trade-off

### 원본 확인 위치

- Thread 02
- 커밋: `feat(mock): seed deterministic event fixtures`
- 커밋: `feat(mock): publish deterministic odds ticks`
- 파일/컴포넌트: `MockOddsProvider`, `OddsSimulator`
- 상태/상수: `structureRandom`, `oddsRandom`, `ODDS_STREAM_SALT`, `runSeed`
- 함수: `seed`, `nextUuid`
- 관련 Thread: 03, 04

---

<a id="p16"></a>
## [Thread 02 / `feat(mock): advance event lifecycles`] 시간 점프와 terminal 단조성을 다루는 상태 머신

### 면접 질문

`tick(now)` 한 번이 kickoff와 종료 시각을 모두 지난 경우 어떤 lifecycle event를 내야 합니까? terminal 상태에서 반복 tick이 들어올 때의 invariant도 설명해 주세요.

꼬리 질문:

- `SCHEDULED → FINISHED`로 한 번에 건너뛰는 것과 중간 `IN_PLAY`를 발행하는 것의 차이는 무엇입니까?
- kickoff 시각과 정확히 같은 `now`는 어느 상태입니까?
- 같은 `now`로 tick을 두 번 호출하면 이벤트가 중복돼도 됩니까?
- terminal event의 odds tick을 계속 돌리면 어떤 문제가 생깁니까?
- replay buffer를 무제한으로 두면 어떤 resource 위험이 있습니까?

### 30초 모범 답변

상태 전이는 현재 상태와 시간 경계를 기준으로 단조롭게 진행해야 합니다. `now`가 kickoff 이상이면 `SCHEDULED`에서 `IN_PLAY`, 같은 tick에서 종료 시각도 지났다면 이어서 `FINISHED`까지 필요한 중간 전이를 순서대로 발행합니다. 이미 `FINISHED`, `CANCELLED`, `POSTPONED`면 아무것도 하지 않습니다. 상태를 먼저 바꾸고 그 전이에 대응하는 event를 한 번만 만들면 동일 tick 재호출도 idempotent합니다. replay는 늦은 subscriber에 유용하지만 반드시 bounded history여야 합니다.

### 답변 핵심 키워드

time-driven state machine, boundary instant, monotonic transition, multi-step catch-up, idempotent tick, terminal no-op, bounded replay

### 백지 구현

**구현 목표**

현재 lifecycle과 kickoff/end 시각, `now`를 받아 필요한 transition 목록과 새 상태를 반환한다.

**인터페이스**

```java
enum LifecycleStatus {
  SCHEDULED, IN_PLAY, FINISHED, CANCELLED, POSTPONED
}

record LifecycleState(
    LifecycleStatus status,
    Instant kickoffAt,
    Instant endAt) {}

record LifecycleTransition(
    LifecycleStatus previous,
    LifecycleStatus next,
    Instant occurredAt) {}

record AdvanceResult(
    LifecycleState newState,
    List<LifecycleTransition> transitions) {}

final class LifecycleMachine {
  static AdvanceResult advance(
      LifecycleState state,
      Instant now) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 현재 상태, kickoff, end, now
- 출력: 새 상태와 순서 있는 transition 목록

**반드시 만족해야 할 조건**

- terminal 상태는 항상 no-op
- `now < kickoff`면 no-op
- kickoff 경계에서 `SCHEDULED → IN_PLAY`
- end 경계에서 `IN_PLAY → FINISHED`
- `SCHEDULED` 상태에서 `now >= end`면 필요한 두 전이를 순서대로 반환
- 동일 결과를 다시 advance하면 새 transition 없음
- `endAt >= kickoffAt` 검증
- 입력 state immutable

**경계 조건**

- `now == kickoffAt`
- `now == endAt`
- kickoff와 end가 같은 시각
- 매우 큰 시간 점프
- clock이 뒤로 이동
- terminal 상태에 과거 now
- CANCELLED/POSTPONED

**실패 조건**

- FINISHED에서 OPEN 성격의 상태로 역행
- 같은 transition 중복 발행
- `SCHEDULED → FINISHED`로 건너뛰어 downstream이 IN_PLAY를 반드시 기대하는 계약 위반
- invalid time range 허용

**필요한 제약**

- 10~20분
- scheduler와 replay sink는 구현 범위 밖
- transition 순서를 리스트로 명시

### 구현 후 자가 검증

- [ ] kickoff 전 no-op
- [ ] kickoff 정확한 시각에 IN_PLAY
- [ ] end 정확한 시각에 FINISHED
- [ ] 큰 점프에서 IN_PLAY, FINISHED 두 전이 순서
- [ ] 같은 now 재호출 시 중복 없음
- [ ] terminal 세 상태 모두 no-op
- [ ] invalid end-before-kickoff 거부
- [ ] 입력 객체 불변
- [ ] 각 호출 시간/공간 복잡도 상수

### 구현 후 설명할 것

1. 중간 상태를 생략하지 않은 이유
2. 상태 변경과 event 생성의 원자적 관계
3. scheduler fixed-rate가 겹칠 때 동시 호출 방어
4. replay history 상한을 정하는 기준
5. provider가 out-of-order lifecycle을 보낼 때 수용/거부 정책

### 원본 확인 위치

- Thread 02
- 커밋: `feat(mock): advance event lifecycles`
- 커밋: `test(mock): verify lifecycle progression`
- 파일/컴포넌트: `MockOddsProvider`
- 함수: `scheduledTick`, `tick`, `advance`, `transitionTo`
- 상태: `EventLifecycleStatus`
- 관련 Thread: 03, 07, 10
