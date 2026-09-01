# 계약·상태·메시지 경계 면접 워크북

이 문서는 입력 계약, aggregate 상태 전이, Kafka 처리 경계, 순서가 뒤바뀐 이벤트 복구를 다룬다. 프로젝트에서 확인된 이름과 동작만 원본 확인 위치에 적었다.

<a id="p01"></a>
<!-- POINT:P01 -->
## P01 — [Thread 1 / `feat(bet): create pending placement aggregate`] 정산 aggregate의 단방향 terminal 전이

### 면접 질문

`Bet`은 왜 `PENDING`에서 terminal 상태로 한 번만 전이하도록 만들었습니까? 경기 결과가 `VOID`인 경우와 경기 취소로 전체 베팅을 무효화한 경우를 같은 상태로 표현하지 않은 이유도 설명해 보세요.

꼬리 질문:

- 도메인 객체에서 전이를 막는 것만으로 동시 요청까지 안전합니까?
- payout의 통화와 부호를 terminal 전이 시점에 다시 검사해야 하는 이유는 무엇입니까?
- 이미 terminal인 베팅에 같은 명령이 다시 오면 예외, 무시, 멱등 성공 중 무엇이 적절합니까?

### 30초 모범 답변

베팅은 `PENDING → SETTLED 또는 VOIDED`의 단방향 상태 기계로 두고, 모든 terminal 변경을 한 경계에서 검증했습니다. 결과 파이프라인에서 선택 결과가 모두 `VOID`라면 정산 결과는 `VOID`지만 베팅 자체는 정상 정산된 것이므로 상태는 `SETTLED`입니다. 반면 경기 취소·연기처럼 전체 슬립을 환불한 경우만 `VOIDED`입니다. 전이 시 payout의 비음수성과 통화 일치를 확인하고, 실제 동시성은 행 잠금과 한 번만 생성되는 정산 시도까지 함께 사용해 막습니다.

### 답변 핵심 키워드

단방향 상태 기계, result와 aggregate status 분리, terminal 불변성, 통화 보존, 행 잠금, 내구성 있는 단일 시도

### 백지 구현

#### 구현 목표

terminal 명령을 받아 베팅 상태를 새 불변 객체로 전이시키는 순수 함수를 작성한다. 결과 기반 정산과 전체 슬립 무효화를 구분하고, 이미 terminal인 상태를 다시 변경하지 못하게 한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class BetTerminalTransition {
  public BetTerminalState apply(BetTerminalState current, TerminalCommand command) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 상태, 정산 또는 전체 무효화 명령
- 출력: 검증을 통과한 새 terminal 상태

#### 반드시 만족해야 할 조건

- 현재 상태가 `PENDING`일 때만 전이한다.
- 결과 기반 정산 명령은 결과가 반드시 있고 전체 무효화 사유가 없어야 한다.
- 전체 무효화 명령은 결과가 없어야 하고 허용된 사유가 있어야 한다.
- payout은 0 이상이고 기존 stake와 같은 통화여야 한다.
- 결과 값이 `VOID`여도 aggregate 상태는 `SETTLED`다.
- 전체 무효화만 aggregate 상태를 `VOIDED`로 만든다.
- terminal 시각과 갱신 시각의 의미가 어긋나지 않아야 한다.

#### 경계 조건

- payout이 0인 정상 정산
- 결과가 `VOID`인 정상 정산
- 전체 노출액 전액 환불
- 이미 `SETTLED` 또는 `VOIDED`인 상태

#### 실패 조건

- 잘못된 명령 조합
- 음수 payout
- 통화 불일치
- 두 번째 terminal 전이
- 필수 식별자나 시각 누락

#### 필요한 제약

- 외부 저장소나 프레임워크 없이 순수 Java로 구현한다.
- 입력 객체를 변경하지 않는다.
- 정상 경로와 실패 경로를 명시적인 타입 또는 예외로 구분한다.

### 구현 후 자가 검증

- `PENDING + SETTLE`가 `SETTLED`가 되는가?
- 결과 `VOID`가 `VOIDED`로 잘못 매핑되지 않는가?
- 전체 무효화가 `VOIDED`와 전액 환불을 보존하는가?
- terminal 상태에서 어떤 후속 명령도 상태를 바꾸지 못하는가?
- payout의 부호와 통화 invariant가 모든 생성 경로에서 유지되는가?
- 입력 객체나 selections 컬렉션이 몰래 변경되지 않는가?

### 구현 후 설명할 것

1. 결과 값과 aggregate 상태를 분리한 이유
2. 전이 검증을 한 메서드에 모은 이유
3. 순수 도메인 검증과 데이터베이스 동시성 제어의 역할 차이
4. 중복 명령 처리 정책과 그 trade-off

### 원본 확인 위치

- Thread: 1 — 이벤트 소싱 베팅 스냅샷과 정산 상태 모델
- 커밋: `feat(bet): create pending placement aggregate`
- 파일: `src/main/java/com/sportsbook/settlement/domain/Bet.java`
- 클래스·컴포넌트: `Bet`, `BetSelection`, `SettlementStatus`
- 관련 메서드: `Bet.pending`, `Bet.recordSettled`, `Bet.recordVoided`
- 관련 Thread: 5, 9, 10

<a id="p02"></a>
<!-- POINT:P02 -->
## P02 — [Thread 2 / `feat(placement): persist exact replays idempotently`] 배치 계약 검증과 의미 기반 replay 멱등성

### 면접 질문

같은 `betId`의 배치 이벤트가 다시 왔을 때 단순히 "이미 존재하니 성공"으로 처리하지 않고, incoming placement와 저장된 snapshot의 의미를 fingerprint로 비교한 이유는 무엇입니까? 어떤 필드가 canonical identity에 포함되어야 합니까?

꼬리 질문:

- UUID 대소문자나 odds 소수점 표현을 느슨하게 정규화하면 어떤 문제가 생깁니까?
- selection 순서를 정렬해도 됩니까, 아니면 원계약의 순서를 보존해야 합니까?
- 동시에 같은 `betId`가 들어오는 race는 애플리케이션의 선조회만으로 막을 수 있습니까?

### 30초 모범 답변

멱등성은 "키가 같음"이 아니라 "같은 키가 같은 의미의 요청에만 재사용됨"을 보장해야 합니다. 그래서 SINGLE·MULTIPLE·SYSTEM 규칙, unit stake, 요청 시각, 선택 식별자와 submission odds를 손실 없이 검증하고, incoming 객체와 저장된 aggregate에서 같은 canonical fingerprint를 계산했습니다. 같은 `betId`와 같은 fingerprint는 exact replay이고, 다른 fingerprint는 계약 충돌입니다. 최종 race는 유일 제약과 재조회로 판정해야 하며, 선조회만으로는 충분하지 않습니다.

### 답변 핵심 키워드

semantic idempotency, canonical form, exact replay, conflicting reuse, 손실 없는 odds, 유일 제약, race 후 재조회

### 백지 구현

#### 구현 목표

원시 placement 입력을 검증하고, 의미가 같은 입력이면 항상 같은 SHA-256 fingerprint를 생성하는 경계 컴포넌트를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class PlacementBoundary {
  public ValidatedPlacement validateAndFingerprint(RawPlacement input) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: bet/user 식별자, slip 종류, system K/N, unit stake, 요청 시각, selections
- 출력: 검증된 불변 placement와 소문자 64자리 fingerprint

#### 반드시 만족해야 할 조건

- stake는 양수다.
- selection 수는 15개를 넘지 않는다.
- `SINGLE`은 selection이 정확히 1개다.
- `MULTIPLE`은 selection이 2개 이상이다.
- `SYSTEM`은 K와 N이 모두 있고, N은 실제 selection 수와 일치하며 K 범위가 유효하다.
- 비-SYSTEM 입력에는 system 전용 필드가 없어야 한다.
- selection ID는 중복될 수 없다.
- UUID와 odds는 계약이 요구하는 표현을 손실 없이 검증한다.
- odds는 요구된 소수 자릿수를 임의 반올림해 받아들이지 않는다.
- fingerprint 인코딩은 필드 경계를 모호하게 만들지 않아야 한다.
- incoming 입력과 영속 snapshot에서 동일한 필드 집합과 순서 규칙을 사용한다.

#### 경계 조건

- selection 1개인 SINGLE
- selection 2개인 MULTIPLE
- K=1 또는 K=N인 SYSTEM
- 최대 15 selections
- 동일 숫자를 서로 다른 문자열 표현으로 받은 odds
- UUID 대소문자 차이

#### 실패 조건

- slip 종류와 selection 수 불일치
- system 필드 일부만 존재
- 중복 selection
- 0 또는 음수 stake
- 손실을 수반하는 odds 변환
- canonical 인코딩 불가능

#### 필요한 제약

- hash 충돌을 해결책처럼 다루지 말고, hash 이전의 canonical encoding을 명확히 설계한다.
- 원본 selection 순서가 계약 의미에 포함되는지 먼저 결정하고 일관되게 적용한다.
- fingerprint에 비밀값이나 환경별 값은 포함하지 않는다.

### 구현 후 자가 검증

- 같은 객체를 여러 번 계산해 항상 같은 fingerprint가 나오는가?
- incoming 모델과 persisted 모델의 동일 의미가 같은 fingerprint가 되는가?
- stake, odds, selection 하나만 바뀌어도 fingerprint가 달라지는가?
- 유효하지 않은 입력을 fingerprint 계산 전에 거부하는가?
- selection 경계가 이어 붙이기 충돌을 만들지 않는가?
- exact replay와 semantic conflict를 명확히 구분할 수 있는가?
- 동시에 insert가 발생해도 저장소의 유일 제약으로 최종 판정할 수 있는가?

### 구현 후 설명할 것

1. idempotency key와 request fingerprint의 역할 차이
2. canonicalization에서 순서와 숫자 표현을 다루는 방법
3. 애플리케이션 검증과 데이터베이스 유일 제약을 함께 쓰는 이유
4. hash를 저장하는 방식의 장점과 디버깅 비용
5. 느슨한 정규화보다 계약 위반을 거부한 이유

### 원본 확인 위치

- Thread: 2 — 베팅 접수 계약과 멱등 intake
- 대표 커밋: `feat(placement): persist exact replays idempotently`
- 관련 커밋: `feat(placement): validate placement boundary invariants`, `feat(placement): fingerprint replay semantics`, `feat(placement): map the fixed Avro placement contract`
- 파일: `src/main/java/com/sportsbook/settlement/readmodel/BetPlacementValidator.java`
- 파일: `src/main/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinter.java`
- 파일: `src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java`
- 클래스·컴포넌트: `BetPlacement`, `BetPlacedMapper`, `PlacementContractException`
- 관련 메서드: `BetReadModelWriter.record`
- 관련 Thread: 1, 4, 13, 16

<a id="p03"></a>
<!-- POINT:P03 -->
## P03 — [Thread 2 / `test(placement): verify listener key and ack boundaries`] raw Kafka 입력의 key·decode·ack 경계

### 면접 질문

Kafka value 안에 `userId`가 있는데도 record key가 같은 `userId`인지 별도로 검증한 이유는 무엇입니까? 수동 acknowledgment는 정확히 어느 시점에 호출해야 합니까?

꼬리 질문:

- decode 성공 후 영속화가 실패하면 offset을 커밋해도 됩니까?
- 영속화는 성공했지만 catch-up fanout이 실패한 경우에는 어떻게 합니까?
- listener에서 이미 도메인 객체로 역직렬화된 값을 받는 것과 raw bytes를 받는 것의 차이는 무엇입니까?

### 30초 모범 답변

record key는 partition ordering과 라우팅 identity이고 payload의 actor identity와 독립된 계약입니다. 둘이 다르면 잘못된 partition에 들어간 메시지이므로 영속화 전에 영구 계약 오류로 분류해야 합니다. listener는 raw bytes를 엄격하게 decode하고 key를 검증한 뒤, placement 기록과 필요한 catch-up이 모두 끝난 다음 한 번만 acknowledge합니다. 중간 어느 단계라도 실패하면 ack하지 않아 재시도나 DLT 경계가 작동하게 합니다.

### 답변 핵심 키워드

raw boundary, partition identity, payload actor, strict decode, manual ack, durable boundary, no ack on failure

### 백지 구현

#### 구현 목표

raw Kafka record를 처리하면서 decode, key 검증, 영속 기록, catch-up, acknowledgment의 호출 순서를 보장하는 listener 함수를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class PlacementRecordHandler {
  public void handle(RawRecord record, Acknowledger acknowledger) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: raw key/value/headers를 가진 record, acknowledgment 인터페이스
- 출력: 없음. 성공 시 acknowledgment가 정확히 한 번 호출된다.

#### 반드시 만족해야 할 조건

- value가 없거나 decode할 수 없으면 이후 단계를 수행하지 않는다.
- record key를 canonical UUID로 해석하고 payload의 `userId`와 정확히 비교한다.
- key 검증 전에 저장소를 호출하지 않는다.
- placement 기록이 완료된 뒤에만 catch-up을 수행한다.
- 모든 catch-up 단계가 성공한 뒤 acknowledgment를 한 번 호출한다.
- exact replay도 catch-up은 다시 수행할 수 있어야 한다.
- 예외가 발생하면 acknowledgment를 호출하지 않는다.

#### 경계 조건

- null 또는 빈 key
- 올바른 길이가 아닌 UUID key
- key와 payload ID 불일치
- exact replay
- placement에 여러 event ID가 포함된 경우

#### 실패 조건

- decode 오류
- key 계약 오류
- 영속화 오류
- catch-up fanout 오류
- acknowledgment 자체의 오류

#### 필요한 제약

- listener가 실패를 삼키지 않는다.
- 저장소·fanout·acknowledger는 테스트 대역으로 교체 가능해야 한다.
- 호출 순서를 테스트할 수 있도록 책임을 분리한다.

### 구현 후 자가 검증

- 잘못된 key일 때 writer가 호출되지 않는가?
- writer 실패 시 ack와 fanout이 호출되지 않는가?
- fanout 실패 시 ack가 호출되지 않는가?
- 정상 경로에서 ack가 마지막에 정확히 한 번 호출되는가?
- exact replay에서도 누락된 외부 사실을 catch-up할 수 있는가?
- raw value를 변형하거나 재인코딩하지 않는가?

### 구현 후 설명할 것

1. key와 payload identity를 둘 다 검증하는 이유
2. ack를 durable boundary 뒤에 두는 이유
3. raw listener를 선택했을 때 얻는 제어권과 구현 비용
4. listener 재시도와 도메인 멱등성의 관계

### 원본 확인 위치

- Thread: 2 — 베팅 접수 계약과 멱등 intake
- 대표 커밋: `test(placement): verify listener key and ack boundaries`
- 관련 커밋: `feat(placement): consume accepted placement events`
- 파일: `src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java`
- 클래스·컴포넌트: `StrictAvroDecoder`, `KafkaUuidKeyValidator`, `BetPlacedMapper`, `BetReadModelWriter`
- 관련 메서드: `BetPlacedListener.receive`
- 관련 Thread: 3, 4, 12

<a id="p04"></a>
<!-- POINT:P04 -->
## P04 — [Thread 3 / `feat(kafka): wire retry recovery handler`] bounded retry와 원본 보존 DLT 복구

### 면접 질문

DLT로 보낼 때 payload를 다시 deserialize/serialize하지 않고 source key와 value bytes를 그대로 복사한 이유는 무엇입니까? DLT publish가 실패했는데도 source offset을 recovered로 커밋하면 어떤 문제가 생깁니까?

꼬리 질문:

- 같은 source partition으로 DLT를 보내는 장점과 한계는 무엇입니까?
- `InterruptedException`을 처리할 때 interrupt flag를 복원해야 하는 이유는 무엇입니까?
- 영구 오류와 일시 오류를 어떤 기준으로 나누겠습니까?

### 30초 모범 답변

DLT는 실패한 입력의 정확한 증거여야 하므로 key/value bytes와 기존 headers를 그대로 보존하고, 원본 topic·partition·offset·timestamp·consumer group·예외 유형을 추가합니다. 영구 계약 오류만 bounded retry 뒤 DLT로 보내고, publish 성공을 broker acknowledgment로 확인한 경우에만 recovered offset을 커밋합니다. timeout이나 publish 실패라면 source를 잃지 않도록 실패를 다시 올리고, interrupt는 flag를 복원해 상위 shutdown 정책이 인식하게 합니다.

### 답변 핵심 키워드

exact evidence, raw byte preservation, bounded retry, broker ack, commitRecovered, interrupt restoration, permanent vs transient

### 백지 구현

#### 구현 목표

실패 분류와 exact DLT publication을 수행하는 복구기를 작성한다. DLT가 내구성 있게 publish된 경우에만 source record를 recovered로 판단한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class ExactDeadLetterRecovery {
  public RecoveryResult recover(RawRecord source, Throwable failure) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: source topic/partition/offset/timestamp/key/value/headers와 처리 실패
- 출력: DLT 완료 또는 재시도 필요를 나타내는 결과

#### 반드시 만족해야 할 조건

- source key와 value byte 배열의 의미를 바꾸지 않는다.
- 기존 headers를 보존하고 원본 위치와 실패 유형을 식별할 metadata를 추가한다.
- DLT topic은 source topic에 대응하며 source partition을 유지한다.
- 전송 완료를 bounded timeout 안에서 확인한다.
- DLT publish가 확인되기 전에는 recovered 성공을 반환하지 않는다.
- interrupt가 발생하면 현재 thread의 interrupt 상태를 복원한다.
- 영구 오류와 일시 오류의 정책이 명시적으로 분리되어야 한다.

#### 경계 조건

- null key
- 빈 payload
- 기존 header가 여러 개인 record
- 가장 큰 partition 번호와 offset
- timeout 직전에 완료되는 publish
- 이미 interrupt된 thread

#### 실패 조건

- publisher timeout
- 비동기 전송 실패
- interrupt
- 유효하지 않은 source metadata
- 일시 오류를 DLT로 잘못 격리하는 분류

#### 필요한 제약

- retry 횟수와 backoff는 상한이 있어야 한다.
- DLT 기록 생성과 publish 완료 확인을 분리해 테스트 가능하게 한다.
- 비밀이나 전체 내부 stack trace를 header에 넣지 않는다.

### 구현 후 자가 검증

- key/value가 byte 단위로 동일하게 보존되는가?
- source partition이 DLT record에 유지되는가?
- 원본 topic·partition·offset·timestamp를 복구할 수 있는가?
- timeout/전송 실패 시 recovered 성공이 반환되지 않는가?
- interrupt flag가 복원되는가?
- 영구 오류만 bounded retry 후 DLT로 가는가?
- DLT 성공 후에만 source offset 커밋이 가능하다고 설명할 수 있는가?

### 구현 후 설명할 것

1. raw bytes를 증거로 보존하는 이유
2. DLT publish와 source offset commit의 순서
3. 같은 partition 보존의 운영상 이점
4. retryable 분류가 잘못됐을 때 생기는 두 종류의 장애
5. 동기 대기와 bounded timeout의 trade-off

### 원본 확인 위치

- Thread: 3 — 원시 Kafka 경계와 정확한 DLT 복구
- 대표 커밋: `feat(kafka): wire retry recovery handler`
- 관련 커밋: `feat(kafka): configure bounded raw DLT producer`
- 파일: `src/main/java/com/sportsbook/settlement/event/ExactDeadLetterRecoverer.java`
- 파일: `src/main/java/com/sportsbook/settlement/config/KafkaRecoveryConfiguration.java`
- 파일: `src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java`
- 클래스·컴포넌트: `KafkaRetryPolicy`, `MessageFailureClassifier`, `DefaultErrorHandler`
- 관련 메서드: `ExactDeadLetterRecoverer.recover`
- 관련 Thread: 2, 8, 11

<a id="p05"></a>
<!-- POINT:P05 -->
## P05 — [Thread 4 / `feat(lifecycle): catch up tombstones before results`] topic 간 순서 역전과 재전달 catch-up

### 면접 질문

placement보다 lifecycle terminal event나 accepted result가 먼저 도착할 수 있는 상황을 어떻게 처리했습니까? placement가 도착했을 때 lifecycle catch-up을 result catch-up보다 먼저 수행한 이유는 무엇입니까?

꼬리 질문:

- Kafka가 topic마다 순서를 보장하는데 왜 이 문제가 생깁니까?
- placement exact replay 때 catch-up을 다시 실행하는 것이 중복 효과를 만들지 않습니까?
- 한 placement에 같은 event가 여러 selection으로 등장하면 fanout을 몇 번 해야 합니까?

### 30초 모범 답변

서로 다른 topic 사이에는 전역 순서가 없으므로 먼저 온 lifecycle tombstone과 accepted result를 각각 내구 저장합니다. placement가 기록된 뒤 distinct event IDs를 안정된 순서로 훑고, 먼저 terminal lifecycle을 적용한 다음 accepted result를 적용합니다. 전체 슬립 무효화가 이미 성립한 경우 result 정산이 선점하지 않게 하는 의도입니다. exact replay도 catch-up을 다시 수행하지만 실제 자금 효과는 한 베팅당 하나의 내구 시도와 상태 제약이 막습니다.

### 답변 핵심 키워드

cross-topic ordering, durable early fact, replay-driven catch-up, lifecycle precedence, distinct event IDs, one-attempt invariant

### 백지 구현

#### 구현 목표

placement가 생성되거나 exact replay된 뒤, 이전에 저장된 lifecycle와 result 사실을 안정된 순서로 찾아 fanout하는 catch-up 조정기를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class PlacementCatchupCoordinator {
  public CatchupReport catchUp(Placement placement) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: selections를 포함한 placement snapshot
- 출력: lifecycle/result fanout 시도 수를 담은 불변 보고서

#### 반드시 만족해야 할 조건

- placement가 먼저 durable하게 기록됐다는 전제에서 실행한다.
- event IDs를 중복 제거하고 결정적인 순서로 처리한다.
- 모든 event에 대한 lifecycle pass를 먼저 끝낸 뒤 result pass를 시작한다.
- tombstone이나 accepted result가 없으면 해당 fanout을 생략한다.
- exact replay에서도 같은 절차를 다시 실행할 수 있다.
- fanout 실패를 숨기지 않고 이후 acknowledgment가 발생하지 않게 한다.

#### 경계 조건

- selection이 하나인 placement
- 동일 event에 여러 selections
- lifecycle만 존재
- result만 존재
- lifecycle과 result가 모두 존재
- 어떤 외부 사실도 존재하지 않음

#### 실패 조건

- lookup 저장소 오류
- lifecycle fanout 실패
- result fanout 실패
- 중복 제거가 잘못되어 같은 event를 여러 번 처리
- result pass가 lifecycle pass보다 먼저 실행

#### 필요한 제약

- catch-up 자체가 정산 완료를 보장한다고 가정하지 않는다.
- 금전 중복 방지는 downstream의 내구 claim에 의존함을 명시한다.
- 실행 순서를 테스트 대역으로 관찰할 수 있어야 한다.

### 구현 후 자가 검증

- event ID가 정렬되고 중복 제거되는가?
- lifecycle pass 전체가 result pass보다 앞서는가?
- exact replay에서도 누락됐던 사실을 다시 발견할 수 있는가?
- fanout 하나가 실패하면 이후 성공으로 위장하지 않는가?
- lifecycle와 result가 모두 있어도 downstream 단일 claim invariant를 전제로 하는가?
- 빈 lookup 결과를 정상 경로로 처리하는가?

### 구현 후 설명할 것

1. 같은 Kafka cluster에서도 topic 간 전역 순서가 없는 이유
2. 먼저 온 사실을 버퍼가 아니라 durable table에 보관한 이유
3. lifecycle 우선순위가 도메인 의미에 미치는 영향
4. replay를 catch-up 트리거로 재사용한 장점과 추가 부하

### 원본 확인 위치

- Thread: 4 — 순서가 뒤바뀐 이벤트 catch-up
- 대표 커밋: `feat(lifecycle): catch up tombstones before results`
- 관련 커밋: `feat(placement): catch up accepted results`, `test(placement): verify result catchup redelivery`
- 파일: `src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java`
- 클래스·컴포넌트: `LifecycleStore`, `LifecycleFanout`, `AcceptedResultRepository`, `ResultFanout`
- 관련 메서드: `LifecycleStore.findTombstone`, `AcceptedResultRepository.findByEventId`
- 테스트 위치: `BetPlacedCatchupPassOrderTest`, `BetPlacedResultReplayTest`
- 관련 Thread: 2, 5, 9, 10

<a id="p06"></a>
<!-- POINT:P06 -->
## P06 — [Thread 5 / `feat(lifecycle): capture semantic observations`] first-terminal latch와 전체 슬립 무효화

### 면접 질문

이벤트 lifecycle에서 최신 terminal 상태로 덮어쓰지 않고 첫 `CANCELLED` 또는 `POSTPONED`를 tombstone으로 latch한 이유는 무엇입니까? observation deduplication과 terminal latch는 왜 별도 개념입니까?

꼬리 질문:

- 동일 observation의 replay와 서로 다른 terminal observation은 어떻게 구분합니까?
- 두 terminal event가 동시에 들어오면 어느 쪽이 이겨야 합니까?
- SYSTEM 베팅 전체를 무효화할 때 unit stake만 환불하면 왜 잘못입니까?

### 30초 모범 답변

observation은 이력 증거이므로 의미 fingerprint로 replay를 식별해 저장하고, terminal tombstone은 이벤트당 한 행만 `ON CONFLICT DO NOTHING`으로 latch합니다. 첫 terminal 사실이 정산 의사결정을 시작한 뒤 후속 상태가 원인을 바꾸지 못하게 하기 위해서입니다. 동시에 들어와도 데이터베이스 유일 경계에서 하나만 latch됩니다. 전체 슬립 무효화는 SYSTEM의 unit stake가 아니라 `unit stake × C(N,K)`인 총 노출액을 환불하고, 한 베팅당 하나의 정산 시도로 금전 중복을 막습니다.

### 답변 핵심 키워드

semantic observation, exact replay, first-terminal latch, immutable tombstone, unique conflict, system total exposure, durable claim

### 백지 구현

#### 구현 목표

동시 호출에도 observation replay와 first-terminal latch를 구분하는 thread-safe in-memory lifecycle store를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class LifecycleRegistry {
  public RecordResult record(LifecycleObservation observation) {
    // 직접 구현
  }

  public Optional<LifecycleObservation> findTombstone(UUID eventId) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: event ID, status, 발생 시각, 선택적 예정 시작 시각, 수신 시각, semantic fingerprint
- 출력: `EXACT_REPLAY`, `OBSERVED`, `TERMINAL_LATCHED`, `TERMINAL_ALREADY_LATCHED` 중 하나

#### 반드시 만족해야 할 조건

- 동일 event와 fingerprint는 exact replay다.
- 비terminal observation은 저장되지만 tombstone을 만들지 않는다.
- `CANCELLED`와 `POSTPONED`만 terminal이다.
- event별 첫 terminal observation만 tombstone이 된다.
- 후속 terminal observation은 기존 tombstone을 덮어쓰지 않는다.
- 동시 terminal record에서도 하나만 `TERMINAL_LATCHED`를 반환한다.
- 조회 결과는 외부에서 변경할 수 없는 값이어야 한다.

#### 경계 조건

- 같은 observation 연속 replay
- status만 다른 두 observation
- scheduled start가 null인 terminal observation
- 동시에 들어오는 CANCELLED와 POSTPONED
- terminal 이후 비terminal observation

#### 실패 조건

- fingerprint 형식 오류
- 필수 시각 또는 event ID 누락
- nonterminal을 tombstone으로 저장
- race로 두 terminal이 모두 latch됨
- 기존 tombstone 덮어쓰기

#### 필요한 제약

- coarse-grained lock과 concurrent collection 중 하나를 선택하고 이유를 설명한다.
- fingerprint 생성 로직은 저장소와 분리해도 된다.
- 총 노출액 계산은 overflow를 감지해야 한다.

### 구현 후 자가 검증

- exact replay가 새 observation이나 tombstone을 만들지 않는가?
- first terminal만 latch되는가?
- 두 thread race에서 latch 성공이 하나뿐인가?
- 후속 terminal이 최초 원인과 시각을 바꾸지 않는가?
- 비terminal observation 이력이 보존되는가?
- SYSTEM 전체 환불액 계산에서 `C(N,K)`와 곱셈 overflow를 확인하는가?

### 구현 후 설명할 것

1. observation history와 decision tombstone을 분리한 이유
2. first-wins 정책과 latest-wins 정책의 차이
3. 애플리케이션 동기화와 데이터베이스 유일 제약의 대응 관계
4. SYSTEM unit stake와 total exposure의 차이
5. tombstone 재처리가 금전 중복으로 이어지지 않게 하는 하위 invariant

### 원본 확인 위치

- Thread: 5 — 이벤트 생명주기 terminal latch와 전체 슬립 무효화
- 대표 커밋: `feat(lifecycle): capture semantic observations`
- 관련 커밋: `feat(lifecycle): consume raw lifecycle events`, `feat(settlement): prepare whole slip void claims`, `feat(lifecycle): fan out terminal void claims`
- 파일: `src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFingerprinter.java`
- 파일: `src/main/java/com/sportsbook/settlement/lifecycle/LifecycleObservation.java`
- 파일: `src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java`
- 파일: `src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java`
- 클래스·컴포넌트: `EventLifecycleListener`
- 관련 메서드: `LifecycleStore.record`, `SettlementAttempt.wholeSlipVoid`, `LifecycleFanout.fanOut`
- 관련 Thread: 4, 6, 7, 10, 19
