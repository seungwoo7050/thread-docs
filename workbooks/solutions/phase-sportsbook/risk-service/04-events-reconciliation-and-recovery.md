# 이벤트 경계·수락 베팅 재조정·Kafka 실패 복구

이 문서는 Kafka에서 들어온 수락 베팅을 신뢰 가능한 도메인 값으로 바꾸는 경계, HTTP 예약 lifecycle과 first-seen 이벤트 projection을 하나의 멱등 흐름으로 합치는 재조정, source offset과 DLT 확인 순서를 다룬다. Thread 13·14·15의 핵심은 "외부 사실을 언제 영구 처리 완료로 간주할 것인가"다.

---

## P14. [Thread 13 / `feat(events): validate accepted event envelopes`] Avro 바이트를 신뢰 가능한 도메인 이벤트로 바꾸는 엄격한 경계

### 면접 질문

Kafka에서 Avro payload를 역직렬화한 뒤에도 `AcceptedBetEnvelope`에서 다시 검증한 이유는 무엇이며, 어떤 항목을 경계에서 확인했나요?

꼬리 질문:

- decoder가 정상 record를 만든 뒤 trailing bytes가 남아 있어도 허용하면 어떤 문제가 생깁니까?
- Kafka record key와 payload의 user ID가 다른 경우 어느 쪽을 신뢰해야 합니까?
- `requestedAt`과 consumer가 관찰한 시각을 모두 보관하되, 위험 창의 score에는 어느 시각을 사용했나요?
- UUID 문자열을 parse할 수 있다는 것과 canonical 문자열이라는 것은 어떻게 다릅니까?

### 30초 모범 답변

Avro schema 통과는 타입 모양만 보장하고 도메인 계약까지 보장하지 않습니다. payload 전체가 정확히 소비됐는지 확인하고, canonical ID·금액·통화·선택 고유성·베팅 shape·idempotency key·Kafka key와 user 일치를 검증해 하나의 `RiskCheckCommand`로 만듭니다. 이벤트의 `requestedAt`은 감사 정보로 남기되, 오래된 이벤트가 과거 창에 삽입되어 현재 한도를 왜곡하지 않도록 consumer가 관찰한 시각을 처리 시각으로 사용합니다. 하나라도 모호하면 정상 projection을 만들지 않습니다.

### 답변 핵심 키워드

untrusted bytes · full payload consumption · schema vs domain validation · canonical UUID · Kafka key binding · observedAt · unique selections · strict envelope

### 백지 구현

#### 구현 목표

이미 Avro decoder를 통과한 단순 이벤트 DTO를 받아 Kafka key와 도메인 규칙을 검증하고, 내부 명령과 원래 요청 시각을 담은 envelope를 만든다.

#### 인터페이스 또는 함수 시그니처

```java
record SelectionDto(
    String marketId,
    String selectionId,
    String odds) {}

record AcceptedEventDto(
    String eventId,
    String userId,
    String betId,
    String idempotencyKey,
    SlipType slipType,
    long stakeAmount,
    String currency,
    Integer systemMinWins,
    Integer systemTotalSelections,
    List<SelectionDto> selections,
    Instant requestedAt) {}

record AcceptedEnvelope(
    Candidate command,
    Instant requestedAt) {}

public static AcceptedEnvelope from(
    String kafkaKey,
    AcceptedEventDto event,
    Instant observedAt) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: Kafka key, decoded event, consumer 관찰 시각
- 출력: 모든 도메인 invariant를 만족하는 envelope
- 잘못된 입력: 명시적 예외

#### 반드시 만족해야 할 조건

- 필수 식별자는 canonical UUID 문자열이어야 한다.
- Kafka key는 payload user ID와 정확히 같아야 한다.
- idempotency key는 비어 있지 않고 길이 상한과 허용 문자 범위를 만족해야 한다.
- stake와 currency는 지원 도메인으로 변환 가능해야 한다.
- 선택 수는 1~15이며 selection ID가 고유해야 한다.
- 각 selection의 필수 market·selection·odds 필드를 검증한다.
- slip type과 시스템 필드 shape를 P03 규칙에 맞게 검증한다.
- 내부 command의 평가 시각은 `observedAt`이다.
- envelope의 `requestedAt`은 event 값을 보존한다.
- 입력 목록을 신뢰 가능한 불변 목록으로 복사한다.

#### 경계 조건

- canonical UUID와 대소문자·축약 등 비정규 UUID
- Kafka key 누락과 mismatch
- idempotency key의 최소·최대 길이, 비출력 문자
- 선택 1개, 15개, 중복 선택
- 시스템 필드 누락·불일치
- `requestedAt`이 `observedAt`보다 훨씬 과거 또는 미래

#### 실패 조건

- null event·observedAt·필수 필드
- 식별자 parse 실패 또는 비정규 표현
- key와 payload identity 불일치
- 잘못된 금액·통화·odds
- 잘못된 선택 집합 또는 slip shape

#### 필요한 제약

- Avro codec 구현은 제공된 것으로 가정한다.
- 잘못된 필드를 기본값으로 채우지 않는다.
- payload user ID와 Kafka key 중 하나를 임의로 우선하지 않는다.
- 시간 보정이나 재정렬은 하지 않는다.
- 20~30분 안에 핵심 검증과 테스트를 작성한다.

### 구현 후 자가 검증

- [ ] 정상 이벤트가 observedAt 기반 command로 변환된다.
- [ ] requestedAt은 별도 필드에 원본 그대로 남는다.
- [ ] parse 가능한 비정규 UUID를 정책에 따라 거부한다.
- [ ] Kafka key mismatch가 실패한다.
- [ ] idempotency key 경계와 비출력 문자가 검증된다.
- [ ] 중복 selection과 selection 수 범위 위반이 실패한다.
- [ ] slip shape와 노출액 계산 입력이 일관된다.
- [ ] 원본 selection 리스트를 바꿔도 envelope가 변하지 않는다.
- [ ] 실패한 이벤트에서 부분 command가 외부로 노출되지 않는다.
- [ ] codec 계층에서는 trailing bytes를 거부하는 별도 테스트가 있다.

### 구현 후 설명할 것

1. Avro schema 검증 뒤에도 도메인 envelope가 필요한 이유
2. Kafka key를 payload identity와 결합한 이유
3. requestedAt 대신 observedAt을 Redis score에 쓰는 trade-off
4. canonical 문자열 검증이 replay 지문 안정성에 미치는 영향
5. malformed 입력을 DLT 대상으로 분류하는 기준

### 원본 확인 위치

- Thread 13
- 커밋: `feat(events): validate accepted event envelopes`
- 파일: `src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java`
- 클래스: `AcceptedBetEnvelope`, `AvroCodec`
- 테스트: `AcceptedBetEnvelopeTest`, `AvroCodecTest`
- 관련 Thread: 02, 09, 14, 15

---

## P15. [Thread 14 / `feat(events): reconcile accepted reservations`] 예약 commit과 first-seen projection을 결합한 멱등 재조정

### 면접 질문

수락 이벤트가 도착했을 때 예약 lifecycle이 있으면 commit하고, 없으면 직접 projection하는 흐름을 어떻게 멱등하게 만들었나요?

꼬리 질문:

- commit 결과가 `NOT_FOUND`일 때만 direct projection으로 넘어가야 하는 이유는 무엇입니까?
- projection 도중 같은 bet의 reservation lifecycle이 갑자기 나타났다면 왜 그냥 commit으로 전환하지 않았습니까?
- 같은 fingerprint replay와 다른 fingerprint conflict는 Redis에 어떤 marker로 구분합니까?
- accepted projection 뒤 같은 bet가 다시 reservation admission으로 들어오는 것을 왜 막아야 합니까?
- 이 설계를 "정확히 한 번"이라고 부를 수 있습니까?

### 30초 모범 답변

재조정은 먼저 reservation commit을 시도합니다. 정상 적용이나 replay면 끝내고, lifecycle이 전혀 없는 `NOT_FOUND`에서만 first-seen projection을 실행합니다. projection은 bet별 accepted fingerprint marker와 committed windows·history 갱신을 한 Lua 실행으로 묶어 같은 fingerprint는 replay, 다른 fingerprint는 conflict로 만듭니다. projection 시작 시 lifecycle이 나타나면 두 경로가 경쟁한 것이므로 error로 중단하고 재시도합니다. admission도 accepted marker를 확인해 이미 확정된 identity가 active 용량으로 다시 들어오지 못하게 합니다.

### 답변 핵심 키워드

commit-first · `NOT_FOUND` fallback · first-seen projection · fingerprint marker · APPLIED/REPLAYED/CONFLICT · cross-ingress identity · atomic projection

### 백지 구현

#### 구현 목표

예약 store와 accepted projection store의 결과를 조합해 하나의 재조정 결과를 반환한다. 임의 fallback으로 영구 실패를 숨기지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```java
enum CommitResult {
  APPLIED, REPLAYED, NOT_FOUND, EXPIRED, TOMBSTONED, CONFLICT
}

enum ProjectionResult {
  APPLIED, REPLAYED, CONFLICT
}

enum Reconciliation {
  APPLIED,
  REPLAYED,
  PERMANENT_CONFLICT,
  PERMANENT_TERMINAL,
  TRANSIENT_FAILURE
}

interface ReservationStore {
  CommitResult commit(String betId, String fingerprint, Instant now);
  ProjectionResult projectAccepted(AcceptedEnvelope envelope, String fingerprint);
}

public static Reconciliation reconcile(
    ReservationStore store,
    AcceptedEnvelope envelope,
    String fingerprint) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 검증된 accepted envelope와 canonical fingerprint
- 출력: source consumer가 ack·DLT·retry를 결정할 수 있는 재조정 결과
- store의 기술 오류: 예외 또는 명시적 transient 결과

#### 반드시 만족해야 할 조건

- commit `APPLIED`는 projection 없이 성공한다.
- commit `REPLAYED`는 projection 없이 replay 성공이다.
- commit `NOT_FOUND`에서만 `projectAccepted`를 호출한다.
- projection `APPLIED`와 `REPLAYED`는 성공으로 구분해 반환한다.
- projection `CONFLICT`는 영구 identity conflict다.
- commit의 `EXPIRED`, `TOMBSTONED`, `CONFLICT`를 `NOT_FOUND`처럼 projection하지 않는다.
- store 오류를 영구 입력 실패로 오분류하지 않는다.
- fingerprint는 envelope의 의미 필드에서 계산된 값이어야 한다.
- 같은 호출에서 commit과 projection이 둘 다 적용되는 경로가 없어야 한다.

#### 경계 조건

- 예약이 정상 존재
- 예약 commit replay
- 예약이 처음부터 없음
- projection marker가 같은 fingerprint로 이미 존재
- projection marker가 다른 fingerprint로 존재
- expired/released/rejected lifecycle
- commit과 projection 사이에 경쟁이 발생해 store가 오류를 내는 경우

#### 실패 조건

- null store/envelope/fingerprint
- 잘못된 fingerprint 형식
- store가 알 수 없는 결과를 반환
- projection 도중 lifecycle 경쟁
- Redis 연결·timeout 등 기술 실패

#### 필요한 제약

- retry loop를 이 함수 안에 넣지 않는다.
- Kafka acknowledgment와 DLT publish는 다루지 않는다.
- `NOT_FOUND` 외 결과를 편의상 direct projection으로 보내지 않는다.
- 결과 enum이 후속 실패 분류에 충분해야 한다.

### 구현 후 자가 검증

- [ ] commit 적용 성공 시 projection을 호출하지 않는다.
- [ ] commit replay 시 projection을 호출하지 않는다.
- [ ] `NOT_FOUND`에서만 projection을 정확히 한 번 호출한다.
- [ ] projection replay가 중복 counter 갱신 없이 성공으로 반환된다.
- [ ] fingerprint conflict가 영구 conflict로 반환된다.
- [ ] expired/tombstoned/conflict commit 결과에서 projection을 호출하지 않는다.
- [ ] store 예외가 성공이나 영구 입력 오류로 바뀌지 않는다.
- [ ] mock 호출 순서가 commit → 필요한 경우 projection이다.
- [ ] 어떤 결과에서도 두 경로가 동시에 적용됐다고 보고하지 않는다.

### 구현 후 설명할 것

1. commit-first 순서가 active 용량과 committed counter를 연결하는 방식
2. `NOT_FOUND`만 fallback 가능한 이유
3. bet별 accepted fingerprint marker의 보존 기간과 멱등성 범위
4. projection과 admission 양쪽에서 accepted identity를 확인하는 이유
5. exactly-once가 아니라 bounded idempotency와 at-least-once delivery를 조합한 설계라는 점

### 원본 확인 위치

- Thread 14
- 커밋: `feat(events): reconcile accepted reservations`
- 파일: `src/main/resources/scripts/risk-project-accepted.lua`
- 클래스: `ReservationAcceptedBetReconciler`, `AcceptedBetReconciler`, `AcceptedBetReconciliation`, `AcceptedProjectionRequest`
- 테스트: `AcceptedProjectionIdentityScriptTest`, `AcceptedReservationBoundaryScriptTest`
- 관련 Thread: 06, 09, 11, 13, 15, 17

---

## P16. [Thread 15 / `feat(events): consume accepted bet events`] source offset, retry, DLT broker 확인의 순서

### 면접 질문

accepted bet consumer가 source offset을 언제 acknowledge하며, transient failure와 permanent failure를 어떻게 다르게 처리했나요?

꼬리 질문:

- DLT 전송 메서드를 호출한 직후 source offset을 ack하면 어떤 유실이 생깁니까?
- DLT broker 확인 후 source ack 전에 프로세스가 죽으면 어떤 중복이 생깁니까?
- transient failure를 무제한 1초 재시도하면 partition 전체에 어떤 영향이 있습니까?
- `InterruptedException`을 잡은 뒤 interrupt flag를 복원해야 하는 이유는 무엇입니까?
- framework의 automatic recovery가 source offset을 대신 commit하지 못하게 한 설정 의도는 무엇입니까?

### 30초 모범 답변

정상 재조정은 바로 source offset을 ack합니다. malformed·key mismatch·fingerprint conflict 같은 영구 실패는 DLT publish의 broker acknowledgment를 제한 시간 안에 확인한 뒤에만 source를 ack합니다. DLT publish 실패나 Redis·Kafka 일시 장애는 source를 ack하지 않아 같은 record가 재시도됩니다. 따라서 유실은 막지만 DLT 확인 후 source ack 전 crash에서는 DLT가 중복될 수 있습니다. transient retry는 partition을 막을 수 있으므로 운영 지표와 별도 격리 전략을 함께 고려해야 합니다.

### 답변 핵심 키워드

manual acknowledgment · ack ordering · confirmed DLT · no ack on transient · at-least-once · duplicate DLT window · partition blocking · interrupt restoration

### 백지 구현

#### 구현 목표

재조정 결과에 따라 source acknowledgment, DLT publish, retry를 정확한 순서로 수행하는 consumer 메서드를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
enum OutcomeKind {
  SUCCESS, PERMANENT_FAILURE, TRANSIENT_FAILURE
}

record Outcome(
    OutcomeKind kind,
    String reason) {}

interface Reconciler {
  Outcome reconcile(String key, byte[] payload);
}

interface DeadLetterPublisher {
  void publishAndAwait(String key, byte[] payload, String reason)
      throws Exception;
}

interface Acknowledgment {
  void acknowledge();
}

public final class Consumer {
  public void onMessage(
      String key,
      byte[] payload,
      Acknowledgment acknowledgment) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: source key/payload와 manual acknowledgment
- 출력: 없음
- transient 또는 DLT publish 실패: 호출이 실패해 framework retry 대상이 됨

#### 반드시 만족해야 할 조건

- `SUCCESS`: source를 한 번 ack하고 DLT를 호출하지 않는다.
- `PERMANENT_FAILURE`: DLT publish가 성공적으로 확인된 뒤 source를 한 번 ack한다.
- DLT publish가 실패하면 source를 ack하지 않고 예외를 전파한다.
- `TRANSIENT_FAILURE`: source를 ack하지 않고 retry 가능한 실패를 전파한다.
- decode 실패처럼 consumer 내부에서 확인되는 영구 입력 오류도 같은 DLT 순서를 사용한다.
- `InterruptedException`을 직접 처리하는 경로가 있으면 interrupt flag를 복원한다.
- ack는 DLT보다 먼저 호출될 수 없다.
- failure reason은 bounded한 안정적 값이어야 하며 payload 전체나 비밀을 header에 넣지 않는다.

#### 경계 조건

- 정상 결과
- 각 영구 실패 reason
- DLT future가 성공, timeout, broker error, interruption
- reconciler가 runtime exception을 던짐
- acknowledgment 자체가 실패
- 같은 source record가 재전달됨

#### 실패 조건

- null payload/acknowledgment
- reconciler 기술 실패
- DLT broker 확인 실패
- timeout 또는 interruption
- 알 수 없는 outcome

#### 필요한 제약

- consumer 메서드 안에서 sleep/retry loop를 직접 구현하지 않는다.
- source와 DLT를 하나의 분산 트랜잭션으로 가장하지 않는다.
- mock 기반 호출 순서 테스트가 가능하도록 의존성을 주입한다.
- 최대 20분 구현 크기로 제한한다.

### 구현 후 자가 검증

- [ ] 성공 경로는 source ack 1회, DLT 0회다.
- [ ] 영구 실패 경로의 호출 순서는 DLT 확인 → source ack다.
- [ ] DLT 실패 경로에서 source ack가 호출되지 않는다.
- [ ] transient 경로에서 DLT와 source ack가 모두 호출되지 않는다.
- [ ] reconciler 예외가 조용히 삼켜지지 않는다.
- [ ] interruption을 잡는 구현이면 thread interrupt 상태가 복원된다.
- [ ] 같은 record 재처리 시 downstream의 멱등성에 의존함을 테스트 이름과 설명에서 드러낸다.
- [ ] framework error handler가 recovered record를 자동 commit하지 않도록 필요한 설정을 설명할 수 있다.
- [ ] DLT 중복 가능 crash window를 인정하고 유실 가능성과 구분한다.

### 구현 후 설명할 것

1. source ack보다 DLT broker 확인이 먼저여야 하는 이유
2. at-least-once 처리에서 중복 DLT를 완전히 없애기 어려운 이유
3. transient 무제한 retry가 partition 처리량에 주는 영향
4. retry와 DLT 분류를 도메인 결과에서 명시적으로 만든 이유
5. transaction/outbox 없이 택한 현재 보장과 대안 설계

### 원본 확인 위치

- Thread 15
- 커밋: `feat(events): consume accepted bet events`
- 클래스: `BetPlacedConsumer`, `BetPlacedDeadLetterPublisher`, `KafkaConsumerConfiguration`, `AcceptedBetReconciliation`
- 테스트: `BetPlacedConsumerReconciliationTest`, `BetPlacedDeadLetterPublisherTest`
- 관련 Thread: 01, 13, 14, 16, 17
