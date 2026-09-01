# Outbox, Inbox, Kafka 실패 분류와 복구

이 문서는 로컬 트랜잭션과 메시지 브로커 사이의 간극을 다루는 transactional outbox, 소비자의 중복·충돌을 영속화하는 inbox, 재시도 가능한 오류와 영구 오류를 분리하고 DLT 게시까지 확인한 뒤 offset을 복구하는 정책을 다룬다.

<a id="t09-transactional-outbox"></a>
## [Thread 09 / `feat(outbox): publish acknowledged pending events`] 승인 상태와 이벤트를 원자적으로 남기고 ack 뒤 게시 완료 처리하기

### 면접 질문

bet을 ACCEPTED로 바꾸는 transaction 안에서 Kafka에 직접 publish하지 않고 outbox row를 함께 저장한 뒤 별도 publisher가 보내게 한 이유는 무엇입니까? publisher가 `send()` 호출 직후가 아니라 broker acknowledgement를 받은 뒤에만 `publishedAt`을 기록해야 하는 이유도 설명해 보세요.

꼬리 질문:

- DB commit은 성공했지만 Kafka publish가 실패하면 어떻게 됩니까?
- Kafka publish는 성공했지만 `publishedAt` DB commit이 실패하면 어떤 중복이 생깁니까?
- 이 설계가 exactly-once delivery를 보장합니까?
- unpublished row를 오래된 순서로 제한된 batch만 읽는 이유는 무엇입니까?
- publisher가 기다리는 시간을 무한대로 두면 어떤 resource 문제가 생깁니까?
- `InterruptedException`을 잡은 뒤 interrupted flag를 복원해야 하는 이유는 무엇입니까?
- 여러 publisher 인스턴스가 같은 row를 보내는 문제는 현재 계약에서 어떻게 다뤄야 합니까?

### 30초 모범 답변

DB 상태 전이와 Kafka publish는 하나의 원자적 transaction으로 묶을 수 없으므로, ACCEPTED 전이와 outbox row 저장만 같은 DB transaction에 둡니다. 별도 publisher는 pending row를 보내고 broker ack를 확인한 뒤에만 게시 완료 시각을 기록합니다. publish 후 DB 갱신이 실패하면 중복 전송될 수 있으므로 소비자는 멱등해야 하고, 이 패턴은 유실을 막는 at-least-once 전달입니다. batch, 전용 scheduler, bounded wait로 backlog와 thread 점유를 통제합니다.

### 답변 핵심 키워드

`transactional outbox`, `atomic local write`, `broker acknowledgement`, `at-least-once`, `duplicate delivery`, `consumer idempotency`, `bounded batch`, `bounded wait`, `interrupt preservation`

### 백지 구현

**구현 목표**

pending outbox event를 제한된 개수만 읽어 broker에 전송하고, 명시적 ack를 받은 event만 published로 표시하는 publisher를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
record OutboxEvent(UUID id, String topic, String key, byte[] payload, Instant publishedAt) {}

interface OutboxStore {
  List<OutboxEvent> oldestPending(int limit);
  void markPublished(UUID eventId, Instant at);
}

interface Broker {
  CompletionStage<Void> send(String topic, String key, byte[] payload);
}

final class OutboxPublisher {
  void publishBatch(OutboxStore store, Broker broker, Clock clock, int batchSize, Duration timeout) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: outbox 저장소, broker, clock, batch 크기, ack timeout
- 출력: 반환값 없음. ack가 확인된 row만 저장소 상태가 변한다.

**반드시 만족해야 할 조건**

- oldest pending event를 최대 batch size만 처리한다.
- 각 event의 ack를 timeout 안에 기다린다.
- ack 성공 뒤에만 `markPublished`를 호출한다.
- send 실패·timeout·DB mark 실패가 다음 event 처리에 미치는 정책을 명확히 정한다.
- interrupted 상태를 삼키지 않는다.
- payload 배열이 외부에서 변경되지 않도록 소유권을 고려한다.
- 이미 published인 event는 조회 대상이 아니다.

**경계 조건**

- pending 0개
- 정확히 batch size, batch size+1개
- 즉시 완료된 future
- timeout
- exceptional completion
- interruption
- send 성공 후 mark 실패
- 한 event 실패 후 다음 event 처리 여부

**실패 조건**

실패한 event를 published로 표시하지 않는다. 오류를 로깅하고 남겨 재시도할 수 있어야 하며, thread interruption은 호출자에게 보존한다.

**제약**

병렬 publish는 제외하고 순차 처리한다. exactly-once를 만들기 위한 분산 transaction은 사용하지 않는다.

### 구현 후 자가 검증

- [ ] ack 성공한 event만 published 처리된다.
- [ ] send 실패와 timeout은 pending 상태를 유지한다.
- [ ] publish 성공·mark 실패 시 재시도 중복 가능성을 설명할 수 있다.
- [ ] batch 상한과 oldest-first 순서가 지켜진다.
- [ ] interruption 후 현재 thread의 flag가 유지된다.
- [ ] payload가 호출 전후 임의 변경되지 않는다.
- [ ] consumer 멱등성이 필요한 이유가 테스트 시나리오에 드러난다.
- [ ] scheduler thread가 무한히 block되지 않는다.

### 구현 후 설명할 것

1. DB 상태와 broker publish 사이의 atomicity gap
2. at-least-once를 선택하고 duplicate를 downstream에 맡긴 이유
3. ack 확인과 `publishedAt` 기록 순서
4. 순차 batch와 병렬 publish의 처리량·순서 trade-off
5. 다중 publisher를 지원할 때 row claim 또는 lock이 추가로 필요한 이유

### 원본 확인 위치

- Thread: 09, transactional outbox와 승인 기반 최소 한 번 전달
- 커밋: `feat(database): add transactional outbox`, `feat(placement): finalize outcomes with outbox atomically`, `feat(outbox): publish acknowledged pending events`, `feat(messaging): configure idempotent Kafka producer`
- 파일: `OutboxEvent.java`, `OutboxEventRepository.java`, `OutboxPublisher.java`, `BetStore.java`, `KafkaConfig.java`, `V2__outbox.sql`
- 함수·컴포넌트: `BetStore.acceptAndEnqueue(...)`, `OutboxPublisher.publishPending()`, `OutboxEvent.markPublished(...)`, `OutboxEventRepository.findUnpublished(...)`
- 관련 Thread: 05, 08, 10, 12

---

<a id="t10-durable-inbox"></a>
## [Thread 10 / `feat(wallet-events): deduplicate reconciliation hints`] 동일 event 재생과 동일 ID 충돌을 구분하는 durable inbox

### 면접 질문

Kafka consumer가 wallet event를 받았을 때 event ID만 unique하게 저장하고 중복이면 무조건 무시하는 대신, topic·betId·userId·payload hash까지 비교한 이유는 무엇입니까? receipt를 먼저 저장하고 bet에 reconciliation wake hint를 남긴 뒤 처리 완료를 표시한 순서도 설명해 보세요.

꼬리 질문:

- 같은 event ID와 같은 payload는 왜 no-op이어야 합니까?
- 같은 event ID와 다른 payload를 왜 단순 중복이 아니라 영구 충돌로 봅니까?
- unknown bet 또는 actor mismatch를 retry하면 해결될 가능성이 있습니까?
- receipt를 저장하기 전에 bet를 먼저 바꾸면 crash 시 어떤 문제가 생깁니까?
- event를 최종 상태 전이 명령이 아니라 "복구를 깨우는 힌트"로 사용한 이유는 무엇입니까?
- `processedAt`이 이미 있으면 다시 쓰지 않는 이유는 무엇입니까?
- consumer offset과 DB transaction 사이에는 여전히 어떤 간극이 있습니까?

### 30초 모범 답변

at-least-once 소비에서는 같은 event가 반복되므로 event ID를 영속적으로 소유해야 합니다. 다만 ID만 같고 내용이 다르면 producer나 routing 계약이 깨진 것이므로 topic, actor, bet, payload hash를 비교해 충돌로 격리합니다. 새 event는 bet을 잠가 소유권을 확인하고 receipt를 먼저 flush한 뒤 reconciliation 요청 시각을 남깁니다. event 자체로 최종 상태를 추정하지 않고 기존 HTTP 증거 조회 사가를 깨우는 힌트로 사용해, 순서 역전이나 부분 정보에도 안전하게 복구합니다.

### 답변 핵심 키워드

`durable inbox`, `event identity`, `payload hash`, `duplicate vs conflict`, `receipt before effect`, `reconciliation hint`, `actor ownership`, `at-least-once consumer`

### 백지 구현

**구현 목표**

동일 event의 정확한 재생은 기존 receipt를 반환하고, 동일 ID의 다른 내용은 거부하며, 새 event는 대상 소유권을 검증한 뒤 receipt와 wake hint를 원자적으로 남긴다.

**인터페이스 또는 함수 시그니처**

```java
record EventIdentity(UUID eventId, String topic, UUID aggregateId, UUID actorId, String payloadHash) {}
record Receipt(EventIdentity identity, Instant receivedAt, Instant processedAt) {}

interface InboxStore {
  Optional<Receipt> find(UUID eventId);
  void insert(Receipt receipt);
  void markProcessed(UUID eventId, Instant at);
}

interface AggregateStore {
  Optional<Aggregate> findLocked(UUID aggregateId);
}

final class Inbox {
  Receipt record(EventIdentity event, InboxStore inbox, AggregateStore aggregates, Clock clock) {
    // 직접 구현
    return null;
  }

  void markProcessed(UUID eventId, InboxStore inbox, Clock clock) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: canonical event identity, receipt 저장소, aggregate 저장소, clock
- 출력: 새 receipt 또는 동일 재생의 기존 receipt

**반드시 만족해야 할 조건**

- event ID가 이미 있으면 모든 identity field와 hash를 비교한다.
- 완전 동일하면 aggregate를 다시 건드리지 않는다.
- 하나라도 다르면 conflict로 실패한다.
- 새 event는 aggregate를 잠그고 actor 소유권을 검사한다.
- receipt insert와 wake hint 변경은 하나의 transaction 경계를 가정한다.
- receipt가 먼저 영속되는 순서를 보장한다.
- `markProcessed`는 첫 처리 시각을 보존한다.
- 허용 topic과 hash 형식을 검증한다.

**경계 조건**

- 첫 수신
- 동일 event의 두 번째 수신
- hash만 다른 재생
- topic만 다른 재생
- unknown aggregate
- actor mismatch
- receipt insert 충돌
- 이미 processed인 receipt

**실패 조건**

identity conflict, unknown aggregate, actor mismatch를 조용히 중복으로 삼키지 않는다. receipt가 사라진 상태에서 processed 표시를 허용하지 않는다.

**제약**

Kafka API는 제외한다. transaction 순서는 mock 또는 in-memory log로 검증할 수 있다.

### 구현 후 자가 검증

- [ ] 동일 event 재생은 aggregate side effect를 반복하지 않는다.
- [ ] 동일 ID의 hash·topic·actor·aggregate 차이를 모두 검출한다.
- [ ] 새 event에서 aggregate lock과 actor 검증이 수행된다.
- [ ] receipt 저장이 wake hint보다 먼저다.
- [ ] receipt와 wake hint 중 하나만 저장되는 부분 성공이 없다.
- [ ] 첫 `processedAt`이 중복 호출로 덮이지 않는다.
- [ ] unknown topic과 잘못된 hash가 거부된다.
- [ ] inbox가 있어도 consumer offset과 DB commit 간 중복 가능성은 남는다고 설명할 수 있다.

### 구현 후 설명할 것

1. ID만 unique하게 두는 것보다 payload hash를 함께 검증한 이유
2. duplicate와 conflict의 운영 대응 차이
3. event를 최종 명령이 아니라 wake hint로 사용한 이유
4. receipt-first 순서와 transaction 경계
5. outbox와 inbox가 서로 보완하지만 end-to-end exactly-once는 아닌 이유

### 원본 확인 위치

- Thread: 10, 내구성 있는 지갑 이벤트 inbox와 복구 wake hint
- 커밋: `feat(wallet-events): mark reconciliation wake requests`, `feat(wallet-events): deduplicate reconciliation hints`, `feat(wallet-events): preserve raw record identity`
- 파일: `WalletEventReceipt.java`, `WalletEventReceiptRepository.java`, `WalletEventInbox.java`, `WalletEventListener.java`, `Bet.java`
- 함수·컴포넌트: `WalletEventInbox.record(...)`, `WalletEventInbox.markProcessed(...)`, `WalletEventReceipt.pending(...)`, `WalletEventReceipt.markProcessed(...)`, `Bet.requestReconciliation(...)`
- 관련 Thread: 07, 08, 09, 11, 12

---

<a id="t12-kafka-recovery"></a>
## [Thread 12 / `feat(messaging): require acknowledged permanent recovery`] 영구 오류만 DLT로 보내고 DLT ack 전에는 offset을 복구하지 않기

### 면접 질문

Kafka consumer 오류를 `PermanentKafkaException`과 그 외 transient failure로 나누고, 영구 오류는 원본 raw key/value를 같은 partition의 DLT로 보낸 뒤 send result를 확인해야만 recovered로 처리한 이유는 무엇입니까?

꼬리 질문:

- malformed Avro, canonical UUID key mismatch, unsupported topic은 왜 retry해도 낫지 않습니까?
- DB unavailable은 왜 즉시 DLT로 보내면 안 됩니까?
- DLT publish가 실패했는데 source offset을 commit하면 어떤 데이터 손실이 생깁니까?
- raw byte serializer를 별도 producer에 사용한 이유는 무엇입니까?
- DLT를 source와 같은 partition 번호로 보낼 때의 장점과 전제는 무엇입니까?
- 무한 retry 정책에는 어떤 운영 위험이 있으며 무엇으로 보완해야 합니까?
- topic auto-create를 끈 이유는 무엇입니까?
- DLT consumer가 다시 실패하면 어떤 별도 운영 절차가 필요합니까?

### 30초 모범 답변

재시도로 회복되는 인프라 오류와 레코드 자체가 잘못된 영구 오류를 분리해야 합니다. transient 오류는 원본 partition에서 계속 retry해 순서와 처리를 보존하고, 잘못된 bytes·key·계약 위반만 raw 형태로 같은 partition의 DLT에 게시합니다. DLT send가 broker에 승인되기 전 source record를 recovered로 표시하면 레코드가 어느 곳에도 남지 않으므로, publish failure는 미복구로 두고 offset을 진행하지 않습니다. topic은 사전 provisioning해 오타나 잘못된 partition 구성이 자동 생성으로 숨지 않게 합니다.

### 답변 핵심 키워드

`permanent vs transient`, `raw DLT`, `same partition`, `acknowledged recovery`, `offset safety`, `non-retryable classification`, `preprovisioned topics`, `poison record`

### 백지 구현

**구현 목표**

레코드 처리 실패를 분류하고, 영구 오류는 DLT 게시가 확인된 경우에만 ACK하며, 일시 오류와 DLT 게시 실패는 RETRY하는 순수 coordinator를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
record RawRecord(String topic, int partition, byte[] key, byte[] value) {}
enum Decision { ACK, RETRY }

interface Handler {
  void handle(RawRecord record) throws Exception;
}

interface DeadLetterSink {
  CompletionStage<Void> publish(String dltTopic, int partition, byte[] key, byte[] value, Throwable cause);
}

interface FailureClassifier {
  boolean isPermanent(Throwable failure);
}

final class ConsumerRecovery {
  Decision process(
      RawRecord record,
      Handler handler,
      DeadLetterSink dlt,
      FailureClassifier classifier,
      Duration dltTimeout) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: raw record, 실제 handler, DLT sink, 실패 분류기, timeout
- 출력: source offset을 진행해도 되는지 나타내는 `ACK` 또는 `RETRY`

**반드시 만족해야 할 조건**

- handler 성공은 ACK다.
- transient failure는 DLT에 보내지 않고 RETRY다.
- permanent failure는 `<source>.DLT`의 같은 partition으로 raw bytes를 보낸다.
- DLT ack 성공 뒤에만 ACK다.
- DLT exceptional completion, timeout, interruption은 RETRY다.
- raw key/value를 재직렬화하거나 변경하지 않는다.
- null payload 등의 영구 오류 분류 규칙을 호출자가 주입할 수 있다.

**경계 조건**

- 정상 처리
- permanent parsing failure
- transient database failure
- DLT send 즉시 성공
- DLT timeout
- DLT topic/partition 없음
- thread interruption
- null key 또는 null value

**실패 조건**

DLT publish 실패를 ACK로 바꾸지 않는다. classifier 자체가 실패하거나 알 수 없는 예외가 오면 안전한 기본값을 RETRY로 둔다.

**제약**

실제 Kafka offset API는 제외한다. byte array는 방어적으로 다룬다.

### 구현 후 자가 검증

- [ ] 정상 record는 ACK다.
- [ ] transient failure가 DLT로 가지 않는다.
- [ ] permanent failure만 DLT로 간다.
- [ ] DLT topic과 partition이 올바르다.
- [ ] raw key/value가 동일하게 보존된다.
- [ ] DLT ack 전에 ACK가 반환되지 않는다.
- [ ] DLT timeout·실패·interruption은 RETRY다.
- [ ] 알 수 없는 오류의 기본 분류가 데이터 손실 쪽으로 기울지 않는다.
- [ ] 무한 retry가 운영상 감시와 backoff를 필요로 한다고 설명할 수 있다.

### 구현 후 설명할 것

1. retry 가능성과 record 영구 오류를 나누는 기준
2. DLT publish acknowledgement가 offset 안전성에 미치는 영향
3. raw bytes와 같은 partition을 보존한 이유
4. 무한 retry와 서비스 가용성 사이의 trade-off
5. topic auto-create 금지가 배포 책임을 어디로 이동시키는지

### 원본 확인 위치

- Thread: 12, Kafka 영구 오류 분류와 승인 기반 DLT 복구
- 커밋: `feat(messaging): configure raw DLT publication`, `feat(messaging): require acknowledged permanent recovery`, `feat(messaging): require preprovisioned topics`
- 파일: `KafkaMessageValidator.java`, `PermanentKafkaException.java`, `KafkaRecoveryConfig.java`, `KafkaConfig.java`, `application.yml`, `KafkaRecoveryIntegrationTest.java`
- 함수·컴포넌트: `KafkaRecoveryConfig.errorHandler(...)`, `KafkaRecoveryConfig.recoverer(...)`, `KafkaMessageValidator.decode(...)`, `KafkaMessageValidator.requireKey(...)`
- 관련 Thread: 09, 10, 15, 16
