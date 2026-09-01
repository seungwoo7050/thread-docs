# 트랜잭셔널 아웃박스 기록과 전달

이 문서는 메시지 내용을 만드는 일보다 더 중요한 세 경계를 다룬다. 업무 트랜잭션과 이벤트 사실의 원자성, 다중 worker의 strict FIFO 선점, lease가 바뀐 뒤의 stale completion 차단이다.

<a id="w08"></a>
## W08 — [Thread 12 / `feat(outbox): append one event per operation`] 업무 트랜잭션 안의 아웃박스 append

### 면접 질문

금전 트랜잭션이 커밋된 뒤 Kafka로 직접 전송하는 방식과, 같은 DB 트랜잭션에 outbox row를 기록하는 방식의 실패 차이를 설명해 보세요. 이 프로젝트는 왜 `(topic, partitionKey)`별 `streamSequence`를 DB에서 할당합니까?

꼬리 질문:

- DB 커밋은 성공하고 프로세스가 Kafka send 전에 죽으면 직접 전송 방식에서 무엇을 잃습니까?
- Kafka send를 DB 트랜잭션 안에서 기다리면 왜 원자성이 완성되지 않으며 어떤 리소스 문제가 생깁니까?
- 같은 stream의 sequence를 `MAX + 1`로 계산하면 동시성에서 어떤 문제가 생깁니까?
- 성공 debit과 잔액 부족 debit의 outbox facts는 각각 무엇과 같은 트랜잭션에 있어야 합니까?
- `byte[]` payload를 entity 밖으로 그대로 노출하면 append-only 의미가 어떻게 깨질 수 있습니까?

### 30초 모범 답변

DB와 Kafka는 하나의 로컬 트랜잭션을 공유하지 않으므로 직접 dual write는 어느 한쪽만 성공하는 창을 만듭니다. 업무 변경과 outbox row를 PostgreSQL 한 트랜잭션에 저장하면 커밋된 업무에는 반드시 전달할 사실이 남고, 별도 publisher가 나중에 재시도할 수 있습니다. 스트림별 sequence도 같은 트랜잭션에서 잠긴 stream row를 통해 증가시켜 created timestamp가 아니라 명시적 순서를 권위로 사용합니다. payload와 identity는 불변이어야 하며, 실제 Kafka send는 트랜잭션 밖의 전달 단계가 담당합니다.

### 답변 핵심 키워드

`dual-write gap` · `same DB transaction` · `durable event fact` · `stream row lock` · `monotonic sequence` · `immutable payload` · `send later`

### 백지 구현

#### 구현 목표

현재 업무 트랜잭션 안에서 호출된다고 가정하고, 스트림별 다음 sequence를 배정해 immutable outbox event 하나를 저장한다.

#### 면접용 축소 인터페이스

```java
record PendingMessage(
    UUID eventId,
    String operationKey,
    String topic,
    String partitionKey,
    String schemaName,
    String deduplicationKey,
    byte[] payload,
    Instant createdAt) {}

record OutboxEvent(
    UUID eventId,
    String operationKey,
    String topic,
    String partitionKey,
    String schemaName,
    String deduplicationKey,
    long streamSequence,
    byte[] payload,
    Instant createdAt) {}

interface TransactionContext {
  boolean isActive();
}

interface StreamSequenceStore {
  long next(String topic, String partitionKey);
}

interface OutboxEventStore {
  OutboxEvent insert(OutboxEvent event);
}

final class OutboxAppender {
  OutboxEvent append(PendingMessage message) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 업무 operation과 연결된 pending message
- 출력: 양의 stream sequence를 가진 영속 outbox event
- 저장 실패: 호출자 업무 트랜잭션 전체가 롤백될 수 있도록 예외 전달

#### 반드시 만족해야 할 조건

- 활성 트랜잭션이 없으면 append를 거부한다.
- topic, partition key, schema, operation key, deduplication key는 비어 있지 않아야 한다.
- payload는 비어 있지 않아야 한다.
- `(topic, partitionKey)`마다 sequence가 1부터 단조 증가한다.
- sequence 할당과 event insert는 호출자 업무 트랜잭션 안에 있다.
- event가 보관하는 payload와 getter가 돌려주는 payload 모두 외부 배열 변경의 영향을 받지 않는다.
- 같은 operation에 허용된 이벤트 수와 같은 semantic deduplication key의 중복은 저장소 제약으로 차단할 수 있어야 한다.
- `availableAt` 같은 전달 마감 시각은 저장소가 DB 기준으로 부여할 수 있으며 호출자가 임의로 덮어쓰지 않는다.
- 이 메서드는 Kafka producer를 호출하지 않는다.

통합된 debit 사실:

- 성공 debit에서는 잔액·ledger·SUCCEEDED outcome과 성공 event가 함께 커밋된다.
- 잔액 부족 debit에서는 잔액·ledger 없이 REJECTED outcome과 실패 event가 함께 커밋된다.
- 인프라 실패에서는 업무 상태와 outbox row가 모두 롤백된다.

#### 경계 조건

- 새 stream의 첫 event
- 기존 stream의 연속 event
- 서로 다른 partition key의 독립 sequence
- 같은 operation key 중복
- 같은 semantic deduplication key 중복
- 호출자가 넘긴 payload 배열을 append 뒤 변경
- getter로 받은 배열을 변경
- sequence store가 0, 음수, null에 해당하는 잘못된 값을 반환
- event insert 실패

#### 실패 조건

- 비활성 트랜잭션
- null·blank identity
- 빈 payload
- 비양수 sequence
- uniqueness 위반
- 저장소 오류

#### 필요한 제약

- `SELECT MAX(sequence) + 1`만으로 동시 sequence를 배정하지 않는다.
- event identity와 payload는 append 뒤 불변이다.
- 한 event append의 애플리케이션 복잡도는 O(payload length)이며, sequence 경쟁 범위는 해당 stream에 한정한다.

### 구현 후 자가 검증

- [ ] 트랜잭션 밖 호출을 거부한다.
- [ ] 같은 stream에서 1, 2, 3의 연속 sequence가 나온다.
- [ ] 다른 stream의 sequence는 독립적으로 시작한다.
- [ ] 동일 operation·deduplication 중복을 거부한다.
- [ ] 입력 배열과 반환 배열을 변경해도 저장 event payload가 변하지 않는다.
- [ ] event insert 실패 시 호출자 트랜잭션이 롤백될 수 있게 예외가 전달된다.
- [ ] append 경로에 Kafka send가 없다.
- [ ] 성공·업무 거절·인프라 실패 세 경우의 outbox 존재 조건을 구분했다.

### 구현 후 설명할 것

1. outbox가 해결하는 것은 exactly-once delivery가 아니라 dual-write 원자성이라는 점.
2. stream row 기반 sequence 할당과 DB contention의 trade-off.
3. created timestamp 대신 sequence를 순서 권위로 사용한 이유.
4. mutable `byte[]`에 defensive copy가 필요한 이유.
5. 업무 거절 이벤트를 남기되 infrastructure failure 이벤트를 남기지 않는 기준.

### 원본 확인 위치

- Thread: **12 — 트랜잭셔널 아웃박스 기록**
- 커밋: `feat(outbox): map durable stream positions`
- 커밋: `test(outbox): verify durable stream positions`
- 커밋: `feat(outbox): append one event per operation`
- 커밋: `test(outbox): preserve stream sequence ordering`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/PendingOutboxMessage.java`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/OutboxEvent.java`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/OutboxAppender.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/OutboxStreamLock.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/OutboxEventRepository.java`
- 클래스·메서드: `OutboxAppender.append()`, `OutboxStreamLock.nextSequence()`, `OutboxEvent.pending()`, `OutboxEvent.payload()`
- 관련 Thread: 1, 8, 9, 13

---

<a id="w09"></a>
## W09 — [Thread 13 / `feat(repository): claim strict FIFO outbox stream heads`] 다중 작업자의 스트림별 FIFO 선점

### 면접 질문

여러 publisher worker가 동시에 outbox를 처리할 때 서로 다른 사용자 stream은 병렬로 진행시키면서 같은 stream은 반드시 순서대로 보내려면 claim query가 어떤 조건을 가져야 합니까?

꼬리 질문:

- 단순히 `ORDER BY streamSequence LIMIT N FOR UPDATE SKIP LOCKED`만 사용하면 strict FIFO가 깨질 수 있는 경우는 무엇입니까?
- 앞선 event가 아직 `availableAt`에 도달하지 않았으면 뒤 event를 보내도 됩니까?
- lease가 만료된 행은 어떤 조건에서 다시 claim할 수 있습니까?
- 애플리케이션 시계와 DB 시계가 다르면 due·lease 판정에 어떤 문제가 생깁니까?
- `SKIP LOCKED`가 처리량을 높이는 대신 주는 공정성 trade-off는 무엇입니까?

### 30초 모범 답변

claim 대상은 미발행이고 due이며 lease가 없거나 만료된 행이어야 합니다. 여기에 같은 `(topic, partitionKey)`에서 더 작은 sequence의 미발행 행이 하나도 없다는 조건을 넣어야 strict FIFO가 됩니다. 선행 행이 아직 due가 아니거나 다른 worker에게 lease된 경우에도 뒤 행은 막혀야 합니다. 후보는 짧은 `REQUIRES_NEW` 트랜잭션에서 `FOR UPDATE SKIP LOCKED`로 잡고 owner·leaseUntil·leaseVersion·attempt를 갱신합니다. due와 expiry는 한 query의 DB 시계로 비교해 worker 간 clock skew를 제거합니다.

### 답변 핵심 키워드

`unpublished` · `due` · `lease expired` · `NOT EXISTS earlier unpublished` · `strict stream head` · `SKIP LOCKED` · `DB clock` · `short claim transaction`

### 백지 구현

#### 구현 목표

DB query 대신 메모리 목록으로 claim 의미를 재현한다. 입력 snapshot에서 batch 크기만큼 안전한 stream head를 골라 lease를 부여하고, 나머지는 변경하지 않는다.

#### 면접용 축소 인터페이스

```java
record EventState(
    UUID eventId,
    String topic,
    String partitionKey,
    long streamSequence,
    Instant createdAt,
    Instant availableAt,
    Instant publishedAt,
    String leaseOwner,
    Instant leaseUntil,
    long leaseVersion,
    int attemptCount,
    boolean rowLockedByAnotherWorker) {}

record ClaimResult(
    List<EventState> updatedEvents,
    List<EventState> claimedEvents) {}

final class OutboxClaimer {
  static ClaimResult claim(
      List<EventState> events,
      String owner,
      int batchSize,
      Instant databaseNow,
      Duration leaseDuration) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: outbox snapshot, owner, batch 크기, DB 기준 시각, lease 기간
- 출력: 선택된 행에 새 lease가 반영된 전체 snapshot과 claim 목록

#### 반드시 만족해야 할 조건

후보 자격:

- `publishedAt == null`
- `availableAt <= databaseNow`
- lease가 없거나 `leaseUntil <= databaseNow`
- 다른 작업자가 현재 row lock을 보유한 행은 건너뛴다.
- 같은 `(topic, partitionKey)`에 더 작은 sequence의 **미발행 행**이 하나라도 있으면 후보가 아니다. 그 선행 행이 아직 due가 아니거나 lease 중이어도 동일하다.

선택과 갱신:

- 후보를 `availableAt`, `createdAt`, `eventId`의 안정적인 순서로 정렬해 최대 batch 크기만 선택한다.
- 선택 행의 `leaseOwner`를 owner로, `leaseUntil`을 `databaseNow + leaseDuration`으로 바꾼다.
- `leaseVersion`과 `attemptCount`를 각각 1 증가시킨다.
- 같은 stream에서 한 번의 claim에 두 행을 선택하지 않는다.
- 만료 lease를 가져온 경우에도 동일하게 새 version을 발급한다.
- 선택되지 않은 행은 값이 그대로다.

#### 경계 조건

- 빈 목록과 batch size 0
- stream 하나에 연속된 미발행 행 여러 개
- 선행 행이 미래 `availableAt`인 경우
- 선행 행이 다른 worker lease 중인 경우
- 선행 행이 published이고 다음 행이 due인 경우
- 만료 시각과 `databaseNow`가 정확히 같은 경우
- 서로 다른 stream head 여러 개
- row lock된 후보와 그 뒤의 같은 stream 행
- 동일한 정렬 시각을 가진 후보 여러 개

#### 실패 조건

- blank owner
- 음수 batch size
- 0 이하 lease duration
- 비양수 stream sequence
- `databaseNow + leaseDuration` 표현 범위 오류

#### 필요한 제약

- 입력 객체를 제자리에서 일부만 고쳐 실패 시 반쪽 결과를 만들지 않는다.
- 메모리 구현은 O(n log n) 이내로 작성한다.
- 실제 DB 구현에서는 claim과 lease 갱신을 짧은 독립 트랜잭션에 둔다는 점을 설명할 수 있어야 한다.

### 구현 후 자가 검증

- [ ] 같은 stream에서는 가장 작은 미발행 sequence만 claim된다.
- [ ] 선행 행이 아직 due가 아니어도 뒤 행은 막힌다.
- [ ] 선행 행이 lease 중이어도 뒤 행은 막힌다.
- [ ] 선행 행이 published된 뒤에만 다음 행이 후보가 된다.
- [ ] 서로 다른 stream head는 batch 범위에서 함께 claim된다.
- [ ] 다른 worker가 row lock한 행은 건너뛰고 독립 stream을 진행한다.
- [ ] 만료 lease takeover에서 version과 attempt가 증가한다.
- [ ] 선택되지 않은 행은 변경되지 않는다.
- [ ] batch 크기를 넘지 않는다.

### 구현 후 설명할 것

1. 단순 정렬보다 `선행 미발행 행 부재` 조건이 핵심인 이유.
2. 앞선 행이 future due여도 뒤 행을 막아야 strict FIFO가 되는 이유.
3. `FOR UPDATE SKIP LOCKED`가 독립 stream의 병렬성을 만드는 방식.
4. DB 시계 CTE를 한 번 읽어 query 전체의 시간 기준을 고정하는 이유.
5. claim 트랜잭션과 실제 Kafka send를 분리해 lock 시간을 줄인 이유.

### 원본 확인 위치

- Thread: **13 — 리스 기반 FIFO 아웃박스 전달**
- 커밋: `feat(repository): claim strict FIFO outbox stream heads`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java`
- 클래스·메서드: `OutboxDeliveryRepository.claim()`
- 테스트: `OutboxDeliveryRepositoryTest`
- 관련 Thread: 1, 3, 12

---

<a id="w10"></a>
## W10 — [Thread 13 / `feat(outbox): model fenced delivery leases`] lease fencing과 at-least-once 완료

### 면접 질문

worker A의 lease가 만료되어 worker B가 같은 event를 다시 claim한 뒤, 늦게 끝난 A가 `published` 완료를 시도하면 무엇으로 차단해야 합니까? owner 문자열만 비교해서는 왜 부족합니까?

꼬리 질문:

- `leaseVersion`이 fencing token으로 동작하는 과정을 설명해 보세요.
- Kafka ACK는 받았지만 DB `publishedAt` 기록 전에 프로세스가 죽으면 어떤 delivery semantics가 됩니까?
- 이 중복 창을 publisher에서 완전히 제거할 수 없다면 소비자는 무엇을 해야 합니까?
- Kafka send를 claim 트랜잭션 밖에서 수행하는 이유는 무엇입니까?
- lease duration과 producer의 delivery timeout·max blocking time 사이에는 어떤 안전 조건이 필요합니까?
- 비동기 send에 semaphore permit을 사용한다면 어느 경로에서 반드시 반환해야 합니까?

### 30초 모범 답변

claim할 때마다 증가하는 `leaseVersion`을 owner와 함께 token으로 발급하고, publish·retry 완료 UPDATE가 event ID, owner, version, unpublished 조건을 모두 만족할 때만 한 행을 바꾸게 합니다. 그러면 만료 뒤 새 version을 받은 B가 소유권을 갖고, 늦은 A의 완료는 0행 update로 fence됩니다. Kafka ACK 뒤 DB mark 전에 죽으면 event가 다시 전송될 수 있으므로 전달은 at-least-once이고 소비자는 `event-id`로 중복 제거해야 합니다. send는 DB lock을 오래 잡지 않도록 트랜잭션 밖에서 하고, lease는 producer의 최대 완료 시간보다 안전 여유를 두어야 합니다.

### 답변 핵심 키워드

`fencing token` · `owner + version` · `conditional update` · `stale completion` · `ack/mark crash window` · `at-least-once` · `consumer dedup` · `permit cleanup`

### 백지 구현

#### 구현 목표

immutable lease token을 사용해 publish 성공 또는 retry 해제를 적용한다. stale token은 예외로 최신 상태를 덮지 않고 `false`를 반환한다. 함께 포화형 retry delay와 오류 문자열 경계를 구현한다.

#### 면접용 축소 인터페이스

```java
record LeaseToken(UUID eventId, String owner, long version) {}

record DeliveryState(
    UUID eventId,
    Instant publishedAt,
    String leaseOwner,
    Instant leaseUntil,
    long leaseVersion,
    Instant availableAt,
    int attemptCount,
    String lastError) {}

record CompletionResult(boolean applied, DeliveryState state) {}

final class DeliveryCompletion {
  static CompletionResult markPublished(
      DeliveryState current,
      LeaseToken token,
      Instant databaseNow) {
    // 직접 구현
  }

  static CompletionResult releaseForRetry(
      DeliveryState current,
      LeaseToken token,
      Instant databaseNow,
      Duration delay,
      String error) {
    // 직접 구현
  }

  static Duration retryDelay(int attemptCount, Duration base, Duration cap) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 event 상태, 완료를 시도하는 lease token, DB 시각, retry 정보
- 출력: 적용 여부와 새 상태
- stale token: 적용하지 않고 원본과 동등한 상태 반환

#### 반드시 만족해야 할 조건

fencing:

- current event ID와 token event ID가 같아야 한다.
- event가 아직 unpublished여야 한다.
- owner와 version이 모두 정확히 같아야 한다.
- lease expiry 시각만 같다는 사실은 소유권 증명이 아니다.
- 조건이 하나라도 다르면 상태를 바꾸지 않고 `applied=false`다.

publish 성공:

- `publishedAt=databaseNow`
- lease owner·until과 last error를 비운다.
- availableAt과 attempt count 같은 과거 facts를 불필요하게 되돌리지 않는다.

retry 해제:

- delay는 음수가 아니어야 한다.
- `availableAt=databaseNow+delay`
- lease owner·until을 비운다.
- 오류 문자열의 CR/LF를 한 줄로 정리하고 최대 1024자로 제한한다.
- version을 과거 값으로 되돌리지 않는다.

backoff·lifecycle:

- delay는 attempt 증가에 따라 지수 증가하되 cap에서 포화된다.
- arithmetic overflow가 cap을 뚫거나 음수가 되지 않는다.
- 비동기 send가 성공·실패·동기 예외·completion executor 거절 중 어느 경로로 끝나도 획득한 permit은 한 번 반환되어야 한다.
- lease 설정은 producer의 최악 완료 경계보다 커야 한다.

#### 경계 조건

- 정확히 일치하는 token
- owner는 같지만 version이 오래된 token
- version은 같지만 owner가 다른 token
- 이미 published된 event
- lease takeover 직후 이전 worker 완료
- delay 0
- 1024자 오류와 그보다 긴 오류
- CR/LF가 포함된 오류
- 매우 큰 attempt count
- Kafka ACK 직후 DB 완료 전 crash

#### 실패 조건

- null·blank token owner
- 음수 version 또는 attempt count
- 음수 delay
- 시각 덧셈 overflow
- retry error가 null인 경우
- 안전하지 않은 lease duration 설정

#### 필요한 제약

- stale completion은 최신 owner의 lease를 clear하거나 retry 시각을 덮어쓰지 않는다.
- 반환 상태는 입력과 별도 값으로 다뤄 부분 mutation을 피한다.
- 상태 전이는 O(error length)이며 error 외에는 O(1)이다.

### 구현 후 자가 검증

- [ ] 정확한 owner+version에서만 publish가 적용된다.
- [ ] owner가 같아도 이전 version이면 아무 상태도 바뀌지 않는다.
- [ ] retry와 publish 모두 이미 published된 행에 적용되지 않는다.
- [ ] publish 성공 후 lease와 error가 비워진다.
- [ ] retry 후 lease가 비워지고 availableAt만 미래로 이동한다.
- [ ] 오류 문자열이 한 줄, 최대 1024자로 저장된다.
- [ ] 큰 attempt에서도 delay가 cap을 넘거나 음수가 되지 않는다.
- [ ] 모든 비동기 종료 경로에서 permit 반환 횟수가 정확히 한 번이다.
- [ ] ACK/DB mark 사이 crash가 중복 가능성을 만든다는 점을 테스트 설명에 포함했다.

### 구현 후 설명할 것

1. owner 외에 monotonic version이 필요한 이유.
2. conditional update의 0행 결과를 정상적인 fenced completion으로 취급하는 이유.
3. at-least-once를 받아들이고 `event-id` 소비자 dedup을 요구한 이유.
4. claim, send, completion을 서로 다른 트랜잭션·실행 단계로 나눈 이유.
5. lease duration·producer timeout·bounded in-flight 사이의 안전 trade-off.

### 원본 확인 위치

- Thread: **13 — 리스 기반 FIFO 아웃박스 전달**
- 커밋: `feat(outbox): model fenced delivery leases`
- 커밋: `feat(outbox): schedule indefinitely retried bounded backoff`
- 커밋: `config(outbox): validate delivery lease safety`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/OutboxLease.java`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/LeasedOutboxMessage.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/OutboxRetryPolicy.java`
- 파일: `src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java`
- 클래스·메서드: `OutboxDeliveryRepository.markPublished()`, `OutboxDeliveryRepository.releaseForRetry()`, `OutboxPublisher`
- 관련 Thread: 1, 3, 12, 17
