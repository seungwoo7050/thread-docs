# 내구 큐·중단 복구·readiness 워크북

이 문서는 Thread 09의 critical delivery와 Thread 14의 operator delivery에 공통으로 쓰인 Redis Stream 전달 모델을 하나의 문제로 통합한다. 두 경로의 payload는 다르지만, **unread → pending → reclaim → 성공 후 ack/cleanup**이라는 생명주기는 같다.

<a id="p05"></a>
## [Thread 09, 14 / `feat(delivery): consume unread critical events`, `feat(commands): reclaim interrupted operator deliveries`] Redis Stream pending reclaim과 교체 consumer 복구

### 면접 질문

Redis Stream consumer group에서 새 메시지와 이전 process가 읽고 완료하지 못한 pending 메시지를 어떻게 함께 처리했습니까? process가 죽은 뒤 replacement consumer가 누락 없이 복구하는 과정을 설명해 주세요.

꼬리 질문:

- `ReadOffset.lastConsumed()`만 읽으면 이전 consumer의 pending은 왜 복구되지 않습니까?
- pending message의 idle time을 기준으로 claim하는 이유는 무엇입니까?
- claim idle을 너무 짧게 잡으면 어떤 중복 처리가 생깁니까?
- consumer group 생성 중 `BUSYGROUP`은 왜 정상 경쟁으로 취급할 수 있습니까?
- pending과 unread에 같은 record가 동시에 들어오는 상황을 어떻게 방어하겠습니까?

### 30초 모범 답변

consumer group의 unread 읽기만으로는 crash 이전 consumer의 pending이 남습니다. poll마다 group을 확인하고, 먼저 pending summary를 조회해 claim idle을 넘긴 레코드를 현재 consumer로 가져온 뒤 남은 batch만큼 unread를 읽습니다. claim은 유실을 막지만 원래 consumer가 아직 처리 중이면 중복이 생길 수 있으므로 idle 기준과 downstream idempotency가 필요합니다. Redis 오류가 나면 cached `groupReady`를 false로 되돌려 재접속 뒤 group 존재 여부를 다시 확인합니다.

### 답변 핵심 키워드

consumer group, PEL, pending, idle claim, replacement consumer, batch limit, duplicate delivery, BUSYGROUP, groupReady reset

### 백지 구현

**구현 목표**

Redis API 대신, pending과 unread snapshot을 받아 이번 poll에서 전달할 레코드를 선택하는 순수 함수를 구현한다. Thread 09와 Thread 14 모두에 적용 가능한 형태로 만든다.

**인터페이스**

```java
record StreamId(long milliseconds, long sequence)
    implements Comparable<StreamId> {
  // compareTo 직접 구현
}

record PendingRecord(
    StreamId id,
    String owner,
    Instant lastDeliveredAt,
    String payload) {}

record UnreadRecord(
    StreamId id,
    String payload) {}

record DeliveryRecord(
    StreamId id,
    String payload,
    boolean reclaimed) {}

final class StreamDeliverySelector {
  static List<DeliveryRecord> select(
      List<PendingRecord> pending,
      List<UnreadRecord> unread,
      Instant now,
      Duration claimIdle,
      int batchSize) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: group PEL snapshot, unread snapshot, 현재 시각, claim idle, batch size
- 출력: 이번 consumer가 처리할 delivery 목록

**반드시 만족해야 할 조건**

- `lastDeliveredAt + claimIdle <= now`인 pending만 reclaim 후보
- reclaim 후보를 먼저 선택하고 남은 수만 unread에서 채움
- 전체 결과 수는 `batchSize` 이하
- 같은 Stream ID는 한 번만 포함
- reclaim 여부를 정확히 표시
- 각 범주 안에서는 Stream ID 순서를 안정적으로 유지
- 입력 목록을 수정하지 않음

**경계 조건**

- `batchSize`가 0 이하
- `claimIdle`이 0
- pending만 있는 경우
- unread만 있는 경우
- 같은 ID가 pending과 unread 양쪽에 있는 경우
- idle 경계와 정확히 같은 시각
- 매우 오래된 pending이 batch보다 많은 경우

**실패 조건**

- null ID/payload
- 음수 idle
- 비교 불가능한 malformed Stream ID
- 중복 ID를 조용히 서로 다른 payload로 허용함

**필요한 제약**

- 20~25분
- 외부 상태 변경 없이 deterministic
- 정렬이 필요한지 입력 정렬 계약으로 둘지 명시하고 복잡도를 설명

### 구현 후 자가 검증

- [ ] claim idle 미만 pending은 선택되지 않음
- [ ] 경계 시각의 pending은 정책대로 일관되게 선택
- [ ] expired pending이 unread보다 먼저 batch를 차지
- [ ] 결과 수가 batch 상한을 넘지 않음
- [ ] 같은 ID가 두 입력에 있어도 한 번만 반환
- [ ] reclaimed flag가 pending/unread 출처와 일치
- [ ] 입력 순서가 보장되지 않는다는 전제라면 정렬 테스트가 있음
- [ ] large pending에서 불필요한 전체 복사를 줄였는지 설명 가능
- [ ] claim은 exactly-once가 아니라 at-least-once 복구라는 점을 설명 가능

### 구현 후 설명할 것

1. unread와 pending을 별도로 읽어야 하는 이유
2. claim idle의 중복 처리와 복구 지연 trade-off
3. batch를 pending이 독점할 때 unread starvation 가능성
4. consumer name을 process별로 안정적으로 부여해야 하는 이유
5. critical event와 operator action이 같은 queue 알고리즘을 공유해도 payload codec은 분리해야 하는 이유

### 원본 확인 위치

- Thread 09
- 커밋: `feat(delivery): consume unread critical events`
- 파일/컴포넌트: `CriticalEventQueue`, `QueuedCriticalEvent`
- Thread 14
- 커밋: `feat(commands): reclaim interrupted operator deliveries`
- 파일/컴포넌트: `OperatorActionQueue`, `QueuedOperatorMarketAction`, `OperatorActionCodec`
- 함수/경계: `poll`, pending 조회, `claimExpired`, consumer group 생성
- 관련 Thread: 15

---

<a id="p06"></a>
## [Thread 09 / `feat(delivery): acknowledge completed deliveries`] apply 성공 이후 ack와 at-least-once 실패 경계

### 면접 질문

queued event를 처리할 때 `apply`, broker acknowledgement, Redis Stream `XACK`/`XDEL`의 순서는 어떻게 잡았습니까? 각 단계 사이에서 process가 죽으면 어떤 일이 생깁니까?

꼬리 질문:

- apply 전에 ack하면 어떤 종류의 유실이 생깁니까?
- Kafka ack 뒤 Redis completion/cleanup 전에 죽으면 왜 중복 발행이 가능한가요?
- `XACK`과 `XDEL`을 하나의 Lua script로 묶은 이유는 무엇입니까?
- poison record가 계속 실패할 때 현재 구조에 무엇을 추가해야 합니까?
- pending meter를 local 증감만으로 정확히 유지할 수 있습니까?

### 30초 모범 답변

기본 규칙은 **side effect가 성공한 뒤에만 ack**입니다. apply나 Kafka ack가 실패하면 Stream record를 pending에 남겨 replacement consumer가 reclaim하도록 합니다. 성공 뒤에는 `XACK`과 `XDEL`을 한 Redis script로 묶어 group pending과 실제 stream 보관 상태를 같이 정리합니다. 다만 Kafka ack 후 Redis cleanup 전에 crash하면 동일 이벤트가 다시 발행될 수 있으므로 보장은 at-least-once이고, consumer idempotency나 event identity가 필요합니다. 유실보다 중복을 선택한 설계입니다.

### 답변 핵심 키워드

ack-after-success, at-least-once, crash window, XACK, XDEL, atomic cleanup, idempotent consumer, poison record

### 백지 구현

**구현 목표**

한 queued event를 적용하고 성공한 경우에만 acknowledgement port를 호출하는 processor를 구현한다. 재시도 가능한 실패를 숨기지 않는다.

**인터페이스**

```java
record QueuedEvent(String recordId, String payload, boolean reclaimed) {}

interface EventApplier {
  void apply(String payload) throws Exception;
}

interface StreamAcknowledger {
  void acknowledgeAndDelete(String recordId) throws Exception;
}

enum ProcessOutcome {
  COMPLETED,
  APPLY_FAILED,
  CLEANUP_FAILED
}

final class AtLeastOnceProcessor {
  ProcessOutcome processOne(
      QueuedEvent event,
      EventApplier applier,
      StreamAcknowledger acknowledger) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: queued event, 실제 side effect를 수행하는 applier, Stream cleanup port
- 출력: 성공·apply 실패·cleanup 실패 구분

**반드시 만족해야 할 조건**

- `apply`가 정상 완료되기 전에는 acknowledgement 호출 금지
- `apply` 예외 시 record가 pending에 남는다는 계약
- `apply` 성공 후 cleanup 실패를 성공으로 위장하지 않음
- cleanup은 논리적으로 `XACK`과 `XDEL`을 한 연산으로 제공
- 동일 record 재처리를 허용
- interrupt 계열 예외를 별도로 받을 경우 interrupt 상태 복구 정책 명시

**경계 조건**

- 이미 acknowledgement된 record 재처리
- applier는 성공했지만 응답 직전에 예외가 난 경우
- `XACK=0`, `XDEL=1` 같은 멱등 cleanup 결과
- reclaimed record
- 빈 payload 또는 decode가 먼저 실패하는 경우

**실패 조건**

- apply 예외를 삼키고 ack
- cleanup 실패를 `COMPLETED`로 반환
- duplicate 처리 가능성을 문서화하지 않음
- 동일 record를 무한 tight loop로 재시도

**필요한 제약**

- 10~20분
- retry scheduler나 DLQ는 구현 범위 밖
- recording fake로 호출 순서를 검증

### 구현 후 자가 검증

- [ ] 정상 경로가 `apply → acknowledgeAndDelete`
- [ ] apply 실패 시 cleanup 미호출
- [ ] cleanup 실패 시 `CLEANUP_FAILED`
- [ ] 동일 event를 두 번 처리해도 processor가 두 번째 호출을 금지하지 않음
- [ ] duplicate에 안전한 책임은 applier/downstream에도 있음을 명시
- [ ] cleanup 결과 0건을 무조건 치명 오류로 볼지 멱등 완료로 볼지 정책이 있음
- [ ] poison record에 backoff/시도 횟수/DLQ가 필요함을 설명 가능
- [ ] 시간 복잡도는 한 record당 상수이나 외부 I/O latency가 지배함

### 구현 후 설명할 것

1. exactly-once 대신 at-least-once를 선택한 이유
2. Kafka ack와 Redis cleanup 사이 crash window
3. event ID 또는 action ID로 downstream dedup하는 방법
4. `XACK`만 하고 `XDEL`하지 않는 보존 정책과 현재 방식의 차이
5. poison record를 readiness와 별도 운영 신호로 다루는 방법

### 원본 확인 위치

- Thread 09
- 커밋: `feat(delivery): acknowledge completed deliveries`
- 파일/컴포넌트: `CriticalEventQueue`, `CriticalEventProcessor`, `QueuedCriticalEvent`
- 함수: `CriticalEventQueue.acknowledge`, processor poll/apply 흐름
- Redis 경계: Lua `XACK` + `XDEL`
- 관련 Thread: 10, 14

---

<a id="p07"></a>
## [Thread 15 / `feat(delivery): report durable queue health`] readiness 집계와 Redis 재접속 상태 복구

### 면접 질문

서비스 process가 살아 있는 것과 odds feed가 트래픽을 받을 준비가 된 것은 어떻게 다릅니까? `CriticalDeliveryHealthIndicator`와 queue의 `healthy`/`groupReady` 상태가 각각 무엇을 나타냅니까?

꼬리 질문:

- pending count가 1 이상이면 readiness를 무조건 DOWN으로 해야 합니까?
- Redis `DataAccessException` 뒤 `groupReady=true`를 유지하면 어떤 문제가 생깁니까?
- broker metadata probe가 성공하면 publisher health를 바로 UP으로 돌려도 됩니까?
- health detail을 anonymous 요청에 모두 공개하면 어떤 위험이 있습니까?
- 현재 readiness가 provider freshness나 scheduler lag까지 증명합니까?

### 30초 모범 답변

liveness는 process가 재시작 대상인지, readiness는 지금 요청을 받아도 도메인 전달 계약을 지킬 수 있는지를 봅니다. 이 서비스는 Redis Stream, Kafka publisher, critical/operator processor의 health를 AND로 집계합니다. pending 수는 backlog 관찰값이지 그 자체로 장애는 아닙니다. Redis 예외가 나면 `healthy=false`와 함께 cached `groupReady`도 버려서 재접속 후 consumer group을 다시 확인합니다. 성공한 I/O 뒤 health를 회복하되, readiness가 provider freshness나 poison record 부재까지 보장한다고 과장하면 안 됩니다.

### 답변 핵심 키워드

liveness vs readiness, dependency aggregation, pending telemetry, groupReady invalidation, reconnect, false positive, health detail exposure

### 백지 구현

**구현 목표**

전달 dependency snapshot을 readiness로 집계하고, Redis 호출 성공/실패에 따라 queue connection state를 갱신하는 작은 모델을 구현한다.

**인터페이스**

```java
record DeliveryHealthSnapshot(
    boolean redisStreamHealthy,
    boolean kafkaPublisherHealthy,
    boolean criticalProcessorHealthy,
    boolean operatorQueueHealthy,
    boolean operatorProcessorHealthy,
    long criticalPending,
    long operatorPending) {}

record Readiness(
    boolean up,
    Map<String, Object> details) {}

final class DeliveryReadiness {
  static Readiness evaluate(DeliveryHealthSnapshot snapshot) {
    // 직접 구현
  }
}

final class RedisQueueConnectionState {
  private boolean healthy = true;
  private boolean groupReady;

  void onSuccessfulCall(boolean groupConfirmed) {
    // 직접 구현
  }

  void onDataAccessFailure() {
    // 직접 구현
  }

  boolean isHealthy() {
    // 직접 구현
  }

  boolean isGroupReady() {
    // 직접 구현
  }
}
```

**입력과 출력**

- readiness 입력: 필수 전달 dependency별 boolean과 backlog 수치
- readiness 출력: 전체 UP/DOWN과 진단 detail
- connection state 입력: Redis I/O 성공 또는 DataAccess 실패
- connection state 출력: 다음 poll에서 group을 다시 확인할지 여부

**반드시 만족해야 할 조건**

- 필수 dependency 하나라도 unhealthy이면 readiness DOWN
- pending count는 detail에 포함하되 단순히 0보다 크다는 이유만으로 DOWN하지 않음
- 음수 pending은 거부
- Redis DataAccess 실패 시 `healthy=false`, `groupReady=false`
- group이 실제 확인된 성공 경로에서만 `groupReady=true`
- 후속 정상 I/O가 health를 회복할 수 있음
- detail map은 외부에서 수정할 수 없음

**경계 조건**

- operator delivery component가 비활성인 profile
- pending이 매우 크지만 dependency는 healthy
- group 생성 경쟁에서 `BUSYGROUP`
- Redis 연결은 회복했지만 group 확인은 아직 안 됨
- publisher probe는 성공했지만 직전 실제 publish는 실패

**실패 조건**

- 하나의 dependency 실패를 무시하고 UP
- Redis 실패 뒤 groupReady 유지
- backlog 수치만으로 liveness까지 DOWN
- credential 없이 내부 topology detail 노출

**필요한 제약**

- 15~20분
- Spring `HealthIndicator` API를 몰라도 풀 수 있는 순수 모델
- optional component 처리 정책을 명시

### 구현 후 자가 검증

- [ ] 모든 dependency healthy일 때 UP
- [ ] 각 dependency를 하나씩 false로 바꿀 때 DOWN
- [ ] pending이 양수여도 healthy이면 UP
- [ ] 음수 pending 거부
- [ ] DataAccess failure가 health와 groupReady를 모두 false로 변경
- [ ] 단순 성공과 group 확인 성공을 구분
- [ ] reconnect 이후 group 확인 전에는 groupReady가 false
- [ ] details가 immutable
- [ ] readiness가 증명하지 않는 freshness/lag 항목을 설명 가능

### 구현 후 설명할 것

1. pending count를 상태와 지표로 분리한 이유
2. cached readiness flag가 연결 교체 후 무효가 되는 이유
3. broker probe와 실제 publish acknowledgement의 차이
4. readiness dependency가 너무 많을 때 연쇄 장애를 만드는 trade-off
5. health detail의 인증 정책

### 원본 확인 위치

- Thread 15
- 커밋: `feat(delivery): report durable queue health`
- 커밋: `test(health): verify operator readiness details`
- 파일/컴포넌트: `CriticalDeliveryHealthIndicator`, `KafkaBrokerProbe`, `BrokerAvailability`, `CriticalEventQueue`, `OperatorActionQueue`, `OddsFeedPublisher`, `CriticalEventProcessor`, `OperatorActionProcessor`
- 함수/상태: `health`, `isHealthy`, `pendingCount`, `groupReady`, `updatePendingCount`, `KafkaTemplate.partitionsFor`
- 관련 Thread: 06, 08, 09, 14
