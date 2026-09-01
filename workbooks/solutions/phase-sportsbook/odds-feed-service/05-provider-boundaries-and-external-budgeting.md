# Provider 경계·외부 요청 예산·변경 감지 워크북

이 문서는 외부 odds 공급자를 내부 시스템에 연결할 때 필요한 네 가지 기본기를 다룬다.

- 내부 모델을 변경 불가능한 계약으로 닫기
- 짧은 시간 창의 요청량 제한
- process 밖에 남는 월간 quota
- polling snapshot을 안정적 identity와 변경 이벤트로 변환하기

<a id="p17"></a>
## [Thread 01 / `feat(provider): define provider events`] sealed 이벤트 모델과 방어적 복사

### 면접 질문

외부 provider adapter가 바로 Kafka Avro 객체를 만들지 않고 `OddsProvider`, `EventSummary`, sealed `ProviderEvent` 계층을 거치게 한 이유는 무엇입니까? record를 썼는데도 `MatchOutcome.detail`을 별도로 방어 복사해야 하는 이유도 설명해 주세요.

꼬리 질문:

- sealed hierarchy가 일반 interface보다 pattern matching에 주는 이점은 무엇입니까?
- 필수 필드 null 검증을 consumer마다 하지 않고 생성자에서 하는 이유는 무엇입니까?
- `reason`처럼 선택적인 값은 `null`과 `Optional` 중 무엇이 더 적절합니까?
- record의 참조 필드는 자동으로 immutable해집니까?
- `Flux<ProviderEvent>`를 provider contract에 노출한 trade-off는 무엇입니까?

### 30초 모범 답변

provider 경계는 외부 DTO와 내부 전달 계약 사이의 anti-corruption layer입니다. `OddsProvider`는 event 목록, event stream, 선택적 결과라는 최소 포트를 제공하고, sealed `ProviderEvent`는 허용된 변화 종류를 닫아 consumer가 빠짐없이 분기할 수 있게 합니다. record 생성 시 필수 값을 검증해 invalid event가 시스템 안으로 들어오지 못하게 했습니다. 다만 record는 얕은 불변성만 주므로 mutable map 같은 필드는 `Map.copyOf`로 방어 복사해야 합니다. 대가로 새 이벤트 종류 추가 시 모든 switch와 테스트가 함께 바뀝니다.

### 답변 핵심 키워드

port and adapter, anti-corruption layer, sealed hierarchy, record, shallow immutability, defensive copy, constructor invariant, exhaustive handling

### 백지 구현

**구현 목표**

외부 adapter가 반환할 수 있는 세 종류의 내부 이벤트와 immutable 결과 모델을 정의한다. framework annotation은 사용하지 않는다.

**인터페이스**

```java
public sealed interface IngressEvent
    permits PriceChanged, MarketStateChanged, LifecycleChanged {

  String eventId();
  Instant occurredAt();
}

public record PriceChanged(
    String eventId,
    String marketId,
    String selectionId,
    BigDecimal previous,
    BigDecimal current,
    Instant occurredAt)
    implements IngressEvent {
  // 직접 구현
}

public record MarketStateChanged(
    String eventId,
    String marketId,
    MarketStatus previous,
    MarketStatus current,
    String reason,
    Instant occurredAt)
    implements IngressEvent {
  // 직접 구현
}

public record LifecycleChanged(
    String eventId,
    String status,
    Instant scheduledStartAt,
    Instant occurredAt)
    implements IngressEvent {
  // 직접 구현
}

public record Outcome(
    String eventId,
    String score,
    Map<String, String> detail,
    Instant settledAt) {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 생성자 인자
- 출력: 생성 이후 외부 변경으로 상태가 달라지지 않는 value object

**반드시 만족해야 할 조건**

- 허용 이벤트 종류가 sealed permits 목록에 명시
- 공통 `eventId`, `occurredAt` 계약
- 필수 필드 null 거부
- `MarketStateChanged.reason`만 선택적이라는 정책
- odds/price의 유효 범위는 값 객체 또는 생성자에서 검증
- `Outcome.detail` 방어 복사 및 수정 불가
- 전달받은 mutable map을 이후 변경해도 outcome 불변
- equality가 값 기반

**경계 조건**

- 빈 detail map
- null reason
- blank ID
- previous와 current가 같은 price
- scheduled time과 occurred time의 관계
- detail key/value null

**실패 조건**

- record이므로 자동 deep immutable이라고 가정
- mutable map 그대로 보관
- 필수 필드 일부만 검증
- 새 event subtype이 permits 밖에서 임의 추가됨

**필요한 제약**

- 15~20분
- serialization 코드는 구현하지 않음
- 선택 필드 정책을 명시적으로 테스트

### 구현 후 자가 검증

- [ ] 각 subtype이 공통 interface 계약을 만족
- [ ] permits 목록 밖 subtype 생성 불가
- [ ] 필수 null 입력 거부
- [ ] reason null만 허용
- [ ] 원본 detail map 수정 후 outcome 불변
- [ ] outcome detail 수정 시 예외
- [ ] 동일 값 record equality 검증
- [ ] switch에서 모든 subtype을 처리할 수 있음

### 구현 후 설명할 것

1. sealed interface와 enum payload 하나의 trade-off
2. record의 얕은 불변성과 방어 복사
3. domain 값 객체가 primitive/string보다 안전한 이유
4. provider contract에 Reactor 타입을 노출하는 결합도
5. 외부 DTO를 내부 모델로 정규화하는 위치

### 원본 확인 위치

- Thread 01
- 커밋: `feat(provider): define provider events`
- 커밋: `feat(provider): define odds provider contract`
- 파일/컴포넌트: `ProviderEvent`, `EventSummary`, `MatchOutcome`, `OddsProvider`
- subtype: `OddsUpdated`, `MarketStatusUpdated`, `LifecycleUpdated`
- 관련 Thread: 04, 06, 07

---

<a id="p18"></a>
## [Thread 04 / `feat(real): enforce request rate limits`] sliding-window rate limiter

### 면접 질문

분당 최대 N건을 제한하는 `RateLimiter`를 고정된 minute bucket이 아니라 timestamp deque 기반 sliding window로 구현한 이유는 무엇입니까? 동시 호출과 복잡도도 설명해 주세요.

꼬리 질문:

- 10:00:59에 N건, 10:01:00에 N건이 들어오는 fixed window의 burst 문제는 무엇입니까?
- 오래된 timestamp를 언제 제거해야 합니까?
- `currentUsage()`도 같은 정리 과정을 거쳐야 합니까?
- synchronized 전체 메서드와 lock-free 구조 중 어느 쪽이 적절합니까?
- instance가 여러 개면 이 limiter가 전체 quota를 보장합니까?

### 30초 모범 답변

fixed minute bucket은 경계 양쪽에 요청이 몰리면 짧은 시간에 거의 2N건을 허용합니다. sliding window는 승인된 요청 시각을 deque에 저장하고 매 호출마다 `now - window` 이전 항목을 앞에서 제거한 뒤 size가 limit보다 작을 때만 현재 시각을 추가합니다. 각 timestamp는 한 번 들어오고 한 번 빠지므로 amortized `O(1)`, 공간은 최대 limit 수준입니다. 작은 local limiter라 synchronized로 check와 append를 한 critical section에 묶었지만, 다중 instance 전체 제한은 Redis 같은 공유 원자 counter가 필요합니다.

### 답변 핵심 키워드

sliding window, deque, boundary burst, atomic check-and-add, amortized O(1), synchronized, local vs distributed limit, injected Clock

### 백지 구현

**구현 목표**

주어진 시간 창 안에서 승인된 요청 수를 제한하는 thread-safe sliding-window limiter를 구현한다.

**인터페이스**

```java
public final class SlidingWindowRateLimiter {
  public SlidingWindowRateLimiter(
      int maxRequests,
      Duration window,
      Clock clock) {
    // 직접 구현
  }

  public boolean tryAcquire() {
    // 직접 구현
  }

  public int currentUsage() {
    // 직접 구현
  }
}
```

**입력과 출력**

- 생성자: 최대 요청 수, window 길이, clock
- `tryAcquire`: 이번 요청 승인 여부
- `currentUsage`: 현재 window 안 승인 건수

**반드시 만족해야 할 조건**

- 승인된 요청 timestamp만 저장
- window를 벗어난 timestamp를 앞에서 제거
- check와 append가 원자적
- `currentUsage`도 stale timestamp 정리
- system clock 직접 호출 금지
- `maxRequests > 0`, `window > 0`
- deque가 시간순을 유지
- 동일 시각의 여러 요청 처리

**경계 조건**

- 정확히 `now - window`인 timestamp의 포함/제외 정책
- maxRequests 1
- clock이 갑자기 뒤로 이동
- 많은 thread 동시 호출
- 오랫동안 호출이 없다가 다시 호출
- `currentUsage`만 반복 호출

**실패 조건**

- check와 append 사이 race
- 거부 요청 timestamp까지 저장
- stale timestamp를 한 개만 제거
- currentUsage가 오래된 수치를 반환
- 무제한 deque 성장

**필요한 제약**

- 15~20분
- 표준 JDK collection 사용
- lock 범위를 설명하고 단위 테스트는 mutable/fixed Clock 사용

### 구현 후 자가 검증

- [ ] limit까지 승인되고 다음 요청 거부
- [ ] window 경과 후 다시 승인
- [ ] 경계 timestamp 정책 테스트
- [ ] 거부 요청이 usage를 늘리지 않음
- [ ] currentUsage가 stale 항목 제거
- [ ] 동시 N+K 요청에서 승인 수가 N을 넘지 않음
- [ ] invalid constructor 인자 거부
- [ ] amortized 시간 `O(1)`, 공간 `O(maxRequests)` 설명 가능

### 구현 후 설명할 것

1. fixed window와 sliding window 차이
2. deque가 적합한 이유
3. synchronized를 선택한 처리량 가정
4. backward clock 문제와 monotonic clock 대안
5. local limiter와 distributed quota를 함께 둔 이유

### 원본 확인 위치

- Thread 04
- 커밋: `feat(real): enforce request rate limits`
- 파일/컴포넌트: `RateLimiter`
- 함수: `tryAcquire`, `currentUsage`
- 상태: `Deque<Instant>`, 주입된 `Clock`
- 관련 Thread: 15, 16

---

<a id="p19"></a>
## [Thread 04 / `feat(real): persist monthly request quotas`] UTC 월간 quota reservation과 요청 전 차단

### 면접 질문

짧은 분당 rate limit와 월간 provider quota를 왜 분리했습니까? `RedisQuotaCounter`가 UTC `yyyy-MM` key와 35일 TTL을 사용하고, HTTP 요청 전에 count를 증가시키는 흐름을 설명해 주세요.

꼬리 질문:

- quota counter를 local memory에 두면 어떤 재시작·다중 instance 문제가 생깁니까?
- 월말 23:59:59와 다음 달 00:00:00의 key는 어떻게 나뉩니까?
- `INCR` 뒤 quota 초과를 확인하면 초과 시도도 사용량에 포함됩니다. 장단점은 무엇입니까?
- `INCR`과 `EXPIRE`가 완전히 원자적이지 않으면 어떤 key가 남을 수 있습니까?
- rate token을 먼저 소비한 뒤 quota가 거부하면 그 token을 돌려줘야 합니까?

### 30초 모범 답변

rate limit는 짧은 burst로 provider를 압박하지 않기 위한 local admission이고, 월 quota는 재시작과 여러 process를 넘어 유지해야 하는 비용 예산입니다. 그래서 UTC 월 key를 Redis에서 원자 증가하고, 증가 결과가 quota를 넘으면 네트워크 요청을 시작하지 않습니다. 35일 TTL은 지난달 key가 월 경계 뒤 잠시 남아 관측 가능하면서 영구 누적되지 않게 합니다. 현재 순서는 rate 승인 후 quota를 예약하므로 quota 초과 시 rate token과 초과 count가 소비되며, 단순성과 보수적 제한을 택한 것입니다.

### 답변 핵심 키워드

short-term rate vs long-term quota, Redis INCR, UTC period key, reserve-before-I/O, TTL, multi-instance, conservative accounting

### 백지 구현

**구현 목표**

local rate limiter와 distributed 월 counter를 조합해 외부 HTTP 호출 가능 여부를 결정한다.

**인터페이스**

```java
interface LocalRateLimiter {
  boolean tryAcquire();
}

interface MonthlyQuotaCounter {
  long incrementAndGet(String periodKey);
}

enum BudgetDenial {
  NONE, RATE_LIMIT, MONTHLY_QUOTA
}

record BudgetDecision(
    boolean allowed,
    BudgetDenial denial,
    long monthlyUsage) {}

final class ExternalRequestBudget {
  BudgetDecision reserve(
      Instant now,
      int monthlyQuota,
      LocalRateLimiter rateLimiter,
      MonthlyQuotaCounter quotaCounter) {
    // 직접 구현
  }

  static String utcMonthKey(Instant now) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 현재 시각, 월 quota, 두 admission port
- 출력: HTTP 호출 허용 여부, 거부 원인, 관측된 월 사용량

**반드시 만족해야 할 조건**

- rate limit 거부 시 quota counter 호출 없음
- rate 승인 시 UTC month key로 counter 증가
- 증가 결과가 quota 이하일 때만 허용
- 거부 결정 뒤 HTTP port를 호출하지 않는 것은 caller 계약
- quota 양수 검증
- month key가 timezone default에 의존하지 않음
- counter null/비정상 응답 정책
- 동일 process 재시작과 무관한 port 계약

**경계 조건**

- quota 1
- 정확히 quota와 같은 count
- quota+1
- UTC 월 경계 직전/직후
- local timezone이 UTC가 아님
- Redis 오류
- rate limiter 승인 후 quota 오류

**실패 조건**

- HTTP 후 counter 증가
- local memory만으로 월 quota 보장
- server default timezone 사용
- rate 거부인데 quota 소비
- Redis 오류를 fail-open

**필요한 제약**

- 10~20분
- Redis 구현은 port 밖
- `reserve` 호출과 실제 HTTP 호출 사이 race/실패 의미를 설명

### 구현 후 자가 검증

- [ ] rate 거부 시 quota port 미호출
- [ ] count 1부터 quota까지 허용
- [ ] quota+1부터 거부
- [ ] 월 경계에서 key 변경
- [ ] JVM timezone 변경에도 UTC key 동일
- [ ] Redis 오류가 허용으로 바뀌지 않음
- [ ] invalid quota 거부
- [ ] 호출자가 denial별 metric을 남길 수 있는 결과

### 구현 후 설명할 것

1. rate limiter와 quota counter의 서로 다른 scope
2. quota reservation과 실제 provider billable request 차이
3. `INCR`+TTL 원자성 개선 방법
4. 초과 시도 count 포함 여부의 정책
5. 다중 instance에서 local rate limit가 총합을 제한하지 못하는 점

### 원본 확인 위치

- Thread 04
- 커밋: `feat(real): persist monthly request quotas`
- 커밋: `test(real): verify monthly quota accounting`
- 파일/컴포넌트: `QuotaCounter`, `RedisQuotaCounter`, `TheOddsApiProvider`
- 함수: `increment`, `current`, `currentKey`, provider fetch budget 검사
- Redis key: `oddsfeed:provider-quota:` prefix, UTC 월, 35일 TTL
- 관련 Thread: 15

---

<a id="p20"></a>
## [Thread 04 / `test(real): verify changed-only polling`] 안정적 identity와 snapshot diff

### 면접 질문

polling API가 매번 새 JSON 객체를 반환하는데 내부 `EventId`, `MarketId`, `SelectionId`를 어떻게 안정적으로 만들고, 실제 odds가 바뀐 selection만 `OddsUpdated`로 발행했습니까?

꼬리 질문:

- list 순서나 bookmaker 응답 순서가 바뀌어도 ID가 같아야 하는 이유는 무엇입니까?
- random UUID를 polling 때마다 만들면 어떤 cache·Kafka 문제가 생깁니까?
- 첫 snapshot은 baseline입니까, 전체 change event입니까?
- selection이 새로 생기거나 사라지는 경우 정책은 무엇이어야 합니까?
- `lastSeen`을 process memory에만 두는 trade-off는 무엇입니까?

### 30초 모범 답변

polling adapter에서는 객체 identity가 아니라 provider의 안정 필드를 canonical key로 만들어야 합니다. upstream event ID에서 event ID를, event와 market key에서 market ID를, event·market·selection key에서 selection ID를 결정적으로 파생하면 응답 순서와 process 내 객체 생성에 흔들리지 않습니다. 매 poll은 selection key별 odds map으로 정규화하고 이전 map과 비교해 값이 달라진 항목만 이벤트로 냅니다. 같은 snapshot은 아무것도 발행하지 않습니다. `lastSeen`이 memory에 있으므로 재시작 첫 poll의 baseline 정책과 신규/삭제 selection 정책은 명확히 해야 합니다.

### 답변 핵심 키워드

stable identity, canonical key, name-based UUID, snapshot normalization, map diff, changed-only emission, baseline, process-local state

### 백지 구현

**구현 목표**

canonical parts로 안정 ID를 만들고, 이전/현재 odds snapshot의 변경 집합을 계산한다.

**인터페이스**

```java
record SelectionKey(
    String marketKey,
    String selectionName) {}

record Snapshot(
    String upstreamEventId,
    Map<SelectionKey, BigDecimal> oddsBySelection,
    Instant observedAt) {}

record PriceChange(
    UUID eventId,
    UUID marketId,
    UUID selectionId,
    BigDecimal previous,
    BigDecimal current,
    Instant observedAt) {}

final class ProviderIdentity {
  static UUID stableId(
      String domain,
      List<String> canonicalParts) {
    // 직접 구현
  }
}

final class SnapshotDiffer {
  static List<PriceChange> changedPrices(
      Snapshot previous,
      Snapshot current) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 이전/현재 normalized snapshot
- 출력: deterministic 순서의 changed price 목록

**반드시 만족해야 할 조건**

- 동일 upstream event ID는 동일 event UUID
- market과 selection identity에 서로 다른 domain 사용
- canonical part 순서와 encoding 고정
- map iteration 순서와 결과 무관
- 기존 selection의 동일 가격은 change 없음
- 기존 selection 가격 변경은 정확히 한 건
- 출력 순서 deterministic
- BigDecimal 비교 정책을 명시: scale 차이와 수치 동등성
- 신규/삭제 selection 처리 정책을 명시하고 테스트

**경계 조건**

- previous 없음
- 빈 market
- 신규 selection
- 삭제 selection
- 같은 수치, 다른 scale
- selection name 대소문자/공백 normalization
- upstream ID 충돌
- current observedAt이 previous보다 과거

**실패 조건**

- random UUID
- list index를 identity로 사용
- map 순서에 따라 ID나 event 순서 변화
- 동일 snapshot 반복 시 중복 발행
- stale current snapshot을 무조건 적용

**필요한 제약**

- 20~30분
- cryptographic collision 저항보다 deterministic domain identity가 목적
- 외부 JSON parsing은 구현하지 않음

### 구현 후 자가 검증

- [ ] 같은 canonical parts는 같은 UUID
- [ ] domain이 다르면 같은 parts라도 다른 UUID
- [ ] map 삽입 순서를 바꿔도 결과 동일
- [ ] 동일 snapshot은 빈 change
- [ ] 두 selection만 바꾸면 두 change
- [ ] scale 비교 정책 테스트
- [ ] 신규/삭제 selection 정책 테스트
- [ ] stale observedAt 정책 테스트
- [ ] 출력 list가 deterministic
- [ ] 전체 diff 시간 복잡도 `O(n)` 또는 정렬 포함 `O(n log n)` 설명

### 구현 후 설명할 것

1. provider identity에 어떤 필드를 포함해야 하는지
2. name-based UUID와 별도 mapping table의 trade-off
3. process-local `lastSeen` 재시작 의미
4. 첫 poll baseline과 full snapshot 발행 정책
5. 삭제 selection을 별도 상태 이벤트로 모델링할 필요

### 원본 확인 위치

- Thread 04
- 커밋: `test(real): verify changed-only polling`
- 파일/컴포넌트: `TheOddsApiProvider`, `SelectionKey`
- 상태: `lastSeen`
- 함수: `pollSport`, `emitChanges`, `deriveEventId`, `deriveMarketId`, `deriveSelectionId`
- 관련 Thread: 01, 07
