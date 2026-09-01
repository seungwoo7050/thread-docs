# 분산 전달, 멱등성, 정산 정정 면접 워크북

이 문서는 동기 승인과 비동기 전파의 분리, at-least-once 전달에서의 중복 처리, Redis와 Kafka 사이의 전달 간극, 역순으로 도착할 수 있는 정산 정정 스냅샷을 다룬다. 현재 프로젝트에서 확인된 것은 계약과 아키텍처 규칙이며, 확인되지 않은 서비스 내부 클래스명은 사용하지 않는다.

<a id="i09"></a>
## [Thread 06·16 / `feat(idempotency): define request keys` · `docs(project): document shared protocol`] 요청 키 검증과 durable idempotency

### 면접 질문

`IdempotencyKey` 값 객체가 문자열 형식만 검증한다고 해서 요청 처리가 멱등해지는 것은 아닌 이유를 설명해 주세요. 같은 키로 다른 payload가 들어오면 어떻게 처리해야 하나요?

꼬리 질문:

- DB unique constraint와 Redis cache를 함께 사용할 때 각각의 역할은 무엇인가요?
- 최초 요청이 처리 중일 때 동일 키 요청이 동시에 들어오면 어떤 race가 생기나요?
- 같은 키·같은 payload에는 왜 같은 `betId`와 결과를 반환해야 하나요?
- 성공 결과뿐 아니라 업무 거절 결과도 저장해야 하나요?
- printable ASCII와 최대 길이 제한은 어떤 경계를 보호하나요?

### 30초 모범 답변

키 값 객체는 HTTP·Kafka header·DB column을 오갈 수 있는 안정적인 wire shape만 보장합니다. 실제 멱등성은 키와 요청 payload fingerprint, 처리 결과를 durable store에 원자적으로 기록하고 unique constraint로 경쟁 요청을 막아야 합니다. 같은 키와 같은 payload는 저장된 결과를 재생하고, 같은 키에 다른 payload는 충돌로 거부합니다. Redis는 빠른 조회 경로일 수 있지만 정합성의 최종 근거는 durable store여야 합니다.

### 답변 핵심 키워드

wire validation, durable outcome, payload fingerprint, unique constraint, replay, conflict, race, cache fast path, source of truth, same key same result

### 백지 구현

**구현 목표**

요청 키의 wire 조건을 검증하고, 저장된 결과와 새 요청을 비교해 실행·재생·충돌 중 하나를 결정하는 작은 멱등성 판정기를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public record IdempotencyKey(String value) {
  public static final int MAX_LENGTH = 128;

  public IdempotencyKey {
    // 직접 구현
  }
}

public record StoredOutcome(
    IdempotencyKey key,
    String payloadFingerprint,
    String responseBody) {}

public sealed interface IdempotencyDecision
    permits IdempotencyDecision.Execute,
            IdempotencyDecision.Replay,
            IdempotencyDecision.Conflict {

  record Execute() implements IdempotencyDecision {}
  record Replay(String responseBody) implements IdempotencyDecision {}
  record Conflict() implements IdempotencyDecision {}
}

static IdempotencyDecision decide(
    IdempotencyKey key,
    String payloadFingerprint,
    StoredOutcome stored) {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 요청 키, 정규화된 payload fingerprint, 같은 키의 저장 결과 또는 null
- 출력: 새 side effect 실행, 기존 결과 재생, 키 재사용 충돌 중 하나

**반드시 만족해야 할 조건**

- 키는 null·blank가 아니다.
- 키 길이는 128자를 넘지 않는다.
- 키는 0x20~0x7E 범위의 printable ASCII만 포함한다.
- 저장 결과가 없으면 실행 후보로 판정한다.
- 같은 키·같은 fingerprint는 저장된 결과를 재생한다.
- 같은 키·다른 fingerprint는 충돌이다.
- 실제 side effect와 결과 저장은 durable transaction 및 unique constraint로 경쟁을 조정해야 한다.
- Redis cache는 정합성의 유일한 근거가 아니다.

**경계 조건**

- 길이 1과 128
- 길이 129
- 공백만 있는 문자열
- 문자열 내부의 일반 공백
- newline·tab 같은 제어 문자
- 비ASCII 문자
- 저장 결과 없음
- 동일 fingerprint
- 다른 fingerprint
- 두 요청이 동시에 저장 결과 없음으로 관찰한 경우

**실패 조건**

- 키가 같다는 이유만으로 다른 payload 결과를 재생함
- cache miss를 곧바로 최초 요청으로 간주함
- side effect 후 결과 저장 전에 장애가 나 재시도 시 side effect가 반복됨
- unique constraint 충돌 뒤 기존 결과를 다시 읽지 않음
- 응답을 새로 계산해 최초 요청과 다른 식별자나 결과를 반환함

**필요한 제약**

- fingerprint 계산 방식 자체는 이 문제에서 제공된 것으로 가정한다.
- 메모리 Map만으로 분산 환경의 정합성을 해결했다고 주장하지 않는다.
- 저장소 구현은 쓰지 않되 필요한 transaction 경계를 설명한다.

### 구현 후 자가 검증

- [ ] null, blank, 길이 상한, 비ASCII, 제어 문자를 각각 검증했다.
- [ ] 길이 128은 허용하고 129는 거부한다.
- [ ] 저장 결과가 없는 정상 경로를 처리한다.
- [ ] 같은 payload의 재시도는 side effect 없이 결과를 재생한다.
- [ ] 다른 payload의 키 재사용은 충돌로 분류한다.
- [ ] 동시 최초 요청 race에서 unique constraint가 필요한 이유를 설명한다.
- [ ] side effect와 결과 저장 사이 장애를 검토했다.
- [ ] cache와 durable store의 역할을 구분했다.
- [ ] 판정 자체는 상수 시간 비교로 끝난다.

### 구현 후 설명할 것

1. wire 수준 키 검증과 서비스 수준 멱등성 보장의 차이
2. payload fingerprint를 함께 저장해야 하는 이유
3. DB unique constraint가 분산 race의 최종 조정자가 되는 이유
4. Redis를 fast path로만 사용하는 trade-off
5. 성공·거절 결과를 동일하게 재생할 때 얻는 API 안정성

### 원본 확인 위치

- Thread 06
- 커밋: `feat(idempotency): define request keys`
- 커밋: `test(idempotency): verify canonical request keys`
- 커밋: `test(idempotency): reject malformed request keys`
- 파일: `src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java`
- 클래스·컴포넌트: `IdempotencyKey`
- 테스트: `IdempotencyKeyTest`, `IdempotencyKeyValidationTest`
- Thread 16
- 커밋: `docs(project): document shared protocol`
- 파일: `architecture/contract-ownership-and-representation-boundaries.md`
- 파일: `architecture/event-flow-and-consumer-map.md`
- 관련 Thread: 10, 13, 15
- 서비스 내부의 실제 멱등 저장소·handler 파일명은 현재 프로젝트 기록에서 확인되지 않는다.

---

<a id="i10"></a>
## [Thread 16 / `docs(project): document shared protocol`] 동기 승인과 비동기 이벤트 전파의 분리

### 면접 질문

베팅 접수에서 risk와 wallet 결과를 동기로 확인하면서도, 완료 후에는 이벤트를 비동기로 전파하는 구조를 선택한 이유는 무엇인가요?

꼬리 질문:

- Avro record가 있다는 사실이 해당 명령을 Kafka로 처리한다는 뜻이 아닌 이유는 무엇인가요?
- `EventLifecycle`과 `MatchResult`를 별도 이벤트로 둔 이유는 무엇인가요?
- acceptance 응답 전에 wallet debit이 확인되지 않으면 어떤 사용자 경험과 정합성 문제가 생기나요?
- 동기 의존 서비스가 느리거나 실패할 때 retry와 timeout을 어디에서 결정해야 하나요?
- 최초 placement보다 lifecycle이나 settlement 관련 이벤트가 먼저 도착할 가능성을 어떻게 다뤄야 하나요?

### 30초 모범 답변

사용자에게 베팅 승인 여부를 즉시 확정하려면 한도 확인과 자금 차감 같은 승인 조건은 동기 경로에서 확인해야 합니다. 승인이 끝난 뒤 다른 서비스의 projection·알림·정산 준비는 비동기 이벤트로 분리해 결합과 응답 지연을 줄입니다. 이벤트 스키마는 전달 계약일 뿐 명령 처리 방식을 결정하지 않으며, 서비스별 outbox와 consumer 멱등성이 전달 실패를 흡수합니다. 서로 다른 topic 사이에는 전역 순서가 없으므로 consumer 상태 전이가 역순 도착을 허용해야 합니다.

### 답변 핵심 키워드

synchronous decision, asynchronous propagation, user-visible commitment, command vs event, transactional outbox, no global order, timeout ownership, projection, eventual consistency

### 백지 구현

이 항목은 코딩보다 흐름과 실패 경계 설명이 적절하다. 다음 시퀀스와 실패 표를 작성한다.

**구현 목표**

베팅 요청부터 승인 응답, 이벤트 발행까지의 동기·비동기 경계를 그리고 각 단계의 실패 처리를 설명한다.

**작성할 인터페이스 또는 산출물**

```text
참여자:
client | betting | risk | wallet | betting DB/outbox | Kafka | downstream consumer

작성할 것:
1. 정상 승인 시퀀스
2. risk 거절 시퀀스
3. wallet 실패 시퀀스
4. DB commit 성공 후 publish 지연 시퀀스
5. event redelivery 시퀀스
```

**입력과 출력**

- 입력: 베팅 요청, risk 판단, wallet debit 결과
- 출력: 사용자에게 반환되는 승인·거절 결과와 후속 비동기 이벤트

**반드시 만족해야 할 조건**

- risk와 wallet 결과는 승인 결정 전에 확인한다.
- accepted placement 이벤트는 승인 조건이 충족되고 durable state가 확정된 뒤 발행 대상으로 기록한다.
- 업무 state와 outbox는 일관된 경계에서 저장되어야 한다.
- Kafka publish 성공 여부와 사용자 승인 조건을 혼동하지 않는다.
- consumer는 중복 이벤트를 허용한다.
- `EventLifecycle`은 경기 단계 변화, `MatchResult`는 정산에 필요한 결과 데이터라는 의미 차이를 표시한다.
- topic 간 전역 순서를 가정하지 않는다.
- timeout·retry·보상 정책은 해당 서비스의 책임으로 둔다.

**경계 조건**

- risk 승인 후 wallet 거절
- wallet 응답 직후 betting 저장 실패
- 업무 commit 성공 후 producer 장애
- publish 성공 후 ack 기록 전 producer 장애
- consumer side effect 성공 후 offset commit 전 장애
- lifecycle이 placement보다 먼저 도착
- settlement 또는 revision이 초기 projection보다 먼저 도착

**실패 조건**

- 승인 응답 후 실제 debit이 없는데 정상 베팅으로 노출됨
- Kafka가 일시 중단됐다는 이유로 이미 확정된 업무 transaction을 잃음
- 이벤트 record 존재를 근거로 모든 동기 명령을 Kafka에 넣음
- topic 간 순서를 전제로 consumer가 초기 이벤트 없이는 아무것도 처리하지 못함
- 공통 protocol JAR이 서비스별 saga 실행을 소유함

**필요한 제약**

- 현재 프로젝트에서 확인된 컴포넌트명만 사용한다.
- 구체적인 timeout 수치나 retry 횟수는 만들지 않는다.
- 서비스 내부 클래스나 테이블 이름은 가정하지 않는다.

### 구현 후 자가 검증

- [ ] 사용자 승인 시점과 이벤트 publish 시점을 구분했다.
- [ ] risk·wallet 실패 경로가 각각 있다.
- [ ] 업무 commit과 outbox 기록의 관계를 표시했다.
- [ ] publish 재시도와 consumer redelivery를 모두 다뤘다.
- [ ] topic 간 역순 도착을 포함했다.
- [ ] lifecycle과 result의 의미가 구분된다.
- [ ] 서비스 정책과 공통 계약의 책임을 섞지 않았다.
- [ ] 확인되지 않은 구현명을 쓰지 않았다.

### 구현 후 설명할 것

1. 승인 조건을 동기 경로에 둔 사용자·정합성 이유
2. 후속 전파를 비동기로 분리해 얻는 결합도와 가용성 trade-off
3. transactional outbox가 DB commit과 publish 사이 간극을 다루는 방식
4. 명령과 사실 이벤트를 구분하는 기준
5. 전역 순서가 없다는 전제에서 consumer를 설계하는 방법

### 원본 확인 위치

- Thread 16
- 커밋: `docs(project): document shared protocol`
- 파일: `architecture/event-flow-and-consumer-map.md`
- 파일: `architecture/contract-ownership-and-representation-boundaries.md`
- 컴포넌트: betting, risk, wallet, settlement, odds-feed, gateway
- 관련 record: `BetPlacedRequested`, `EventLifecycle`, `MatchResult`, `BetSettled`, `BetVoided`, `BetResolutionRevised`
- 관련 Thread: 10, 11, 12, 13, 14, 15

---

<a id="i11"></a>
## [Thread 16 / `docs(project): document shared protocol`] At-least-once 전달과 멱등 consumer

### 면접 질문

consumer가 업무 side effect를 완료한 뒤 offset commit 전에 죽으면 왜 동일 이벤트가 다시 전달되며, 이를 정상 경로로 취급하려면 handler를 어떻게 설계해야 하나요?

꼬리 질문:

- "Kafka exactly-once"라는 표현만으로 DB side effect까지 중복이 없어지는 것은 왜 아닌가요?
- event identity와 업무 idempotency key 중 어떤 것을 dedup 기준으로 선택하나요?
- 처리 완료 마커와 projection update를 같은 transaction에 넣지 않으면 어떤 race가 생기나요?
- 중복 이벤트를 무시할 때도 payload 충돌을 검사해야 하는 경우가 있나요?
- 외부 시스템 호출처럼 로컬 DB transaction에 묶을 수 없는 side effect는 어떻게 다루겠나요?

### 30초 모범 답변

메시지 broker와 서비스 DB는 하나의 원자적 transaction이 아니므로 side effect 후 offset commit 전에 장애가 나면 redelivery가 발생합니다. 따라서 handler는 이벤트 식별자나 업무 멱등 키로 처리 여부를 확인하고, projection 변경과 처리 완료 기록을 같은 DB transaction에 저장해야 합니다. offset은 그 이후 commit합니다. exactly-once 설정에만 기대지 않고 redelivery를 정상 입력으로 설계하며, 외부 side effect는 별도 idempotency key나 outbox·inbox 패턴이 필요합니다.

### 답변 핵심 키워드

at-least-once, redelivery, inbox/dedup, atomic local transaction, side effect before offset, event identity, business key, external idempotency, exactly-once boundary

### 백지 구현

**구현 목표**

동일 이벤트가 여러 번 전달되어도 projection side effect가 한 번만 적용되도록 단순 consumer handler를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public record DomainEvent(
    String eventId,
    String aggregateId,
    String payload) {}

public interface ConsumerUnitOfWork {
  boolean wasProcessed(String eventId);
  String loadProjection(String aggregateId);
  void saveProjection(String aggregateId, String newValue);
  void markProcessed(String eventId);
}

public enum ProcessOutcome {
  APPLIED,
  DUPLICATE
}

public final class IdempotentEventHandler {
  public ProcessOutcome handle(DomainEvent event, ConsumerUnitOfWork uow) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: event identity, aggregate identity, payload와 하나의 로컬 transaction으로 동작하는 unit of work
- 출력: 새로 적용됐는지 중복이었는지 나타내는 결과

**반드시 만족해야 할 조건**

- 이미 처리한 event identity면 projection을 다시 변경하지 않는다.
- 새 이벤트면 현재 projection을 읽고 한 번 갱신한다.
- projection 저장과 처리 완료 마커는 같은 local transaction에 속한다.
- handler 성공 뒤 offset을 commit한다고 가정한다.
- 중복 판단과 적용 사이 경쟁을 storage의 unique constraint 또는 transaction 격리로 막아야 한다.
- `eventId`가 재시도마다 바뀌는 설계라면 업무 idempotency key를 추가로 사용해야 한다.

**경계 조건**

- 최초 전달
- 같은 이벤트의 즉시 중복
- side effect 후 offset commit 전 장애에 따른 재전달
- 두 consumer 실행이 같은 event identity를 동시에 처음 관찰
- projection이 아직 존재하지 않는 경우
- 같은 aggregate의 서로 다른 이벤트
- 외부 API 호출이 projection update에 포함되는 경우

**실패 조건**

- side effect를 먼저 수행하고 dedup 마커를 별도 transaction에 저장함
- process-local Set만으로 재시작·다중 인스턴스 중복을 막음
- event identity가 안정적이지 않은데 이를 유일한 dedup 키로 사용함
- 중복 메시지를 예외로 취급해 무한 retry를 유발함
- offset commit을 업무 DB commit보다 먼저 수행함

**필요한 제약**

- broker client 코드는 구현하지 않는다.
- unit of work 메서드는 하나의 transaction에서 호출된다고 가정한다.
- payload 해석 로직은 단순화한다.
- 외부 side effect는 별도 설계가 필요하다고 명시한다.

### 구현 후 자가 검증

- [ ] 최초 이벤트는 projection을 한 번 변경한다.
- [ ] 동일 event identity의 재전달은 no-op이다.
- [ ] 처리 완료 후 offset 전 장애 시 재전달을 견딘다.
- [ ] projection과 dedup 마커의 transaction 원자성을 설명한다.
- [ ] 동시 처리 race에서 unique constraint 필요성을 검토했다.
- [ ] 서로 다른 이벤트는 독립적으로 적용된다.
- [ ] process-local 메모리에만 의존하지 않는다.
- [ ] 외부 side effect의 추가 멱등성 요구를 설명한다.
- [ ] 평균 처리 복잡도와 저장소 조회 횟수를 설명할 수 있다.

### 구현 후 설명할 것

1. broker offset과 업무 DB가 만드는 이중 기록 문제
2. inbox/dedup 마커와 projection을 같은 transaction에 두는 이유
3. event identity와 업무 idempotency key의 선택 기준
4. 중복을 오류가 아니라 정상 입력으로 보는 이유
5. exactly-once가 적용되는 경계와 적용되지 않는 외부 side effect

### 원본 확인 위치

- Thread 16
- 커밋: `docs(project): document shared protocol`
- 파일: `architecture/event-flow-and-consumer-map.md`
- 확인된 규칙: durable outbox, 중복 side effect 방지, side effect와 offset commit 사이 장애를 redelivery로 처리, exactly-once 가정 배제
- 관련 Thread: 06, 10, 11, 12, 13, 14
- 실제 consumer handler·inbox repository 파일명은 현재 프로젝트 기록에서 확인되지 않는다.

---

<a id="i12"></a>
## [Thread 16 / `docs(project): document shared protocol`] Redis projection과 Kafka 전달 간극

### 면접 질문

odds-feed가 Redis projection 저장에는 성공했지만 Kafka publish에는 실패했다면, betting 서비스와 gateway는 서로 다른 현실을 보게 됩니다. 이 간극을 어떻게 탐지하고 복구해야 하나요?

꼬리 질문:

- Redis write와 Kafka publish를 같은 성공으로 간주할 수 없는 이유는 무엇인가요?
- critical-event queue에서 Kafka publish 확인 후 ack해야 하는 이유는 무엇인가요?
- readiness가 단순 프로세스 생존 여부만 확인하면 왜 부족한가요?
- risk 알림을 best-effort로 두고 odds critical event는 더 강하게 다루는 기준은 무엇인가요?
- 중복 publish가 발생해도 안전하려면 downstream은 무엇을 해야 하나요?

### 30초 모범 답변

Redis는 betting의 동기 가격 조회 projection이고 Kafka는 gateway와 비동기 consumer의 전달 경로이므로 한쪽 성공이 다른 쪽 성공을 보장하지 않습니다. critical event는 재시도 가능한 queue에 남겨 두고 Kafka publish 확인 후에만 ack해야 하며, backlog나 오래된 미전달 항목을 readiness에 반영해 delivery gap을 드러내야 합니다. 재시도로 중복 publish가 생길 수 있으므로 consumer는 event identity나 업무 키로 멱등해야 합니다. 모든 이벤트에 같은 비용을 쓰지 않고 업무 영향에 따라 best-effort와 durable 경로를 나눕니다.

### 답변 핵심 키워드

dual write gap, Redis projection, Kafka fan-out, critical queue, ack after publish, backlog readiness, retry, duplicate publish, delivery class, observability

### 백지 구현

이 항목은 특정 client API를 외우는 구현보다 실패 매트릭스와 운영 신호 설계가 중요하다.

**구현 목표**

Redis update와 Kafka publish의 가능한 결과 조합을 열거하고, 각 경우의 데이터 관측 차이·재시도·readiness 상태를 정의한다.

**작성할 인터페이스 또는 산출물**

```text
열:
- Redis write 결과
- critical queue 기록 결과
- Kafka publish 결과
- betting 동기 조회가 보는 값
- gateway/consumer가 보는 값
- 재시도 대상
- ack 가능 여부
- readiness 영향

행:
- 모두 성공
- Redis 성공 / Kafka 실패
- Redis 실패 / queue 기록 성공
- publish 성공 / ack 전 장애
- 중복 재시도 publish
```

**입력과 출력**

- 입력: Redis, critical queue, Kafka 각 단계의 성공·실패
- 출력: 복구 행동과 운영 신호가 포함된 실패 매트릭스

**반드시 만족해야 할 조건**

- Redis write 성공과 Kafka publish 성공을 별도로 기록한다.
- critical event는 Kafka publish 확인 전에 queue에서 제거하지 않는다.
- publish 성공 후 ack 전 장애는 중복 publish 가능 경로로 취급한다.
- backlog와 전달 지연을 readiness 또는 운영 신호에서 드러낸다.
- downstream consumer의 멱등성을 전제로 한다.
- risk best-effort 알림과 odds-feed critical event의 보장 수준 차이를 설명한다.
- 구체적인 threshold 수치는 만들지 않는다.

**경계 조건**

- Redis에는 새 가격, Kafka 소비자는 이전 가격
- Kafka에는 전달됐지만 Redis 동기 조회는 실패한 상태
- queue 기록 자체 실패
- 장기간 Kafka 장애로 backlog 증가
- 재시작 후 미ack 항목 재처리
- 같은 event의 중복 publish
- readiness는 green이지만 backlog가 계속 증가하는 잘못된 구현

**실패 조건**

- Redis 성공만으로 전체 처리 성공 응답
- publish 호출 반환 전에 queue ack
- 중복 publish 가능성을 숨김
- backlog·age를 관측하지 않아 전달 단절을 늦게 발견함
- 모든 이벤트에 무조건 같은 내구성 비용을 부과하거나, 반대로 모두 best-effort로 처리함

**필요한 제약**

- 실제 Redis Stream consumer group 이름이나 metric 이름은 가정하지 않는다.
- 거래 원자성을 제공하지 않는 두 시스템의 실패를 명시적으로 모델링한다.
- 기술 제품명보다 보장과 복구 원리를 설명한다.

### 구현 후 자가 검증

- [ ] 성공·부분 성공·재시도 중복 경로를 모두 열거했다.
- [ ] betting과 gateway가 서로 다른 값을 볼 수 있는 경우를 표시했다.
- [ ] ack 시점을 publish 확인 이후로 두었다.
- [ ] 재시작 후 미ack 항목 복구를 포함했다.
- [ ] backlog와 지연이 운영 상태에 반영된다.
- [ ] downstream 멱등성 필요성을 적었다.
- [ ] best-effort와 critical delivery 분류 기준을 설명한다.
- [ ] 확인되지 않은 metric·class·threshold를 만들지 않았다.

### 구현 후 설명할 것

1. Redis와 Kafka 사이에서 원자적 dual write가 되지 않는 이유
2. queue를 delivery intent의 durable 근거로 사용하는 방식
3. ack-after-publish가 유실 대신 중복을 선택하는 trade-off
4. readiness에 전달 backlog를 포함해야 하는 이유
5. 이벤트 중요도에 따라 보장 수준을 다르게 적용하는 판단 기준

### 원본 확인 위치

- Thread 16
- 커밋: `docs(project): document shared protocol`
- 파일: `architecture/event-flow-and-consumer-map.md`
- 문서 내 컴포넌트: odds-feed Redis projection, Redis Stream critical-event queue, Kafka, gateway, betting-service
- 관련 Thread: 11, 12, 15
- 실제 queue handler·readiness 구현 파일명은 현재 프로젝트 기록에서 확인되지 않는다.

---

<a id="i13"></a>
## [Thread 14·16 / `feat(events): define resolution revision snapshots` · `docs(project): document shared protocol`] Full snapshot 정산 정정 reducer

### 면접 질문

정산 결과 정정 이벤트를 payout delta만 보내지 않고 이전·새 결과와 payout의 전체 스냅샷으로 만든 이유는 무엇인가요? consumer는 중복과 역순 전달을 어떻게 처리해야 하나요?

꼬리 질문:

- 최초 `BetSettled`를 논리 revision 0으로 보는 이유는 무엇인가요?
- revision 2가 최초 settlement보다 먼저 도착해도 처리할 수 있어야 하는 이유는 무엇인가요?
- 같은 revision number에 다른 payload가 오면 왜 단순 last-write-wins가 위험한가요?
- revision event를 publish하기 전에 wallet adjustment가 확인돼야 하는 이유는 무엇인가요?
- Kafka key를 `betId`로 선택하면 무엇을 보장하고 무엇을 보장하지 못하나요?

### 30초 모범 답변

topic 간 전역 순서가 없고 재전달도 가능하므로 delta 이벤트는 이전 상태 누락이나 역순 도착에 취약합니다. 정정 이벤트가 새 결과와 payout의 전체 replacement snapshot을 가지면 consumer는 현재 revision number만 비교해 더 큰 revision을 적용할 수 있습니다. 더 작은 revision은 무시하고, 같은 revision·같은 payload는 no-op, 같은 revision·다른 payload는 producer invariant 위반으로 격리합니다. 최초 정산은 revision 0이며, 정정이 먼저 도착하면 이후의 revision 0을 무시할 수 있습니다.

### 답변 핵심 키워드

full snapshot, monotonic revision, logical revision 0, out-of-order, duplicate no-op, conflict quarantine, replacement projection, betId partition key, wallet adjustment proof, outbox

### 백지 구현

**구현 목표**

현재 정산 projection과 새 정정 snapshot을 비교해 적용·구버전 무시·중복·충돌을 판정하는 순수 reducer를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public record ResolutionSnapshot(
    String revisionId,
    long revisionNumber,
    String betId,
    String result,
    long payoutMinorUnits,
    String currency) {}

public sealed interface ApplyResult
    permits ApplyResult.Applied,
            ApplyResult.IgnoredOlder,
            ApplyResult.Duplicate,
            ApplyResult.Conflict {

  record Applied(ResolutionSnapshot current) implements ApplyResult {}
  record IgnoredOlder(ResolutionSnapshot current) implements ApplyResult {}
  record Duplicate(ResolutionSnapshot current) implements ApplyResult {}
  record Conflict(ResolutionSnapshot current, ResolutionSnapshot incoming)
      implements ApplyResult {}
}

static ApplyResult applyRevision(
    ResolutionSnapshot current,
    ResolutionSnapshot incoming) {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 현재 projection 또는 null, 새 full snapshot
- 출력: 적용된 새 상태, 구버전 무시, 중복 no-op, 충돌 중 하나

**반드시 만족해야 할 조건**

- 같은 `betId`의 snapshot만 비교한다.
- revision number는 음수가 아니다.
- 최초 settlement snapshot은 논리 revision 0으로 표현할 수 있다.
- incoming revision이 더 작으면 현재 상태를 유지한다.
- 같은 revision과 같은 전체 payload면 duplicate no-op이다.
- 같은 revision과 다른 payload면 conflict다.
- incoming revision이 더 크면 full snapshot으로 현재 projection을 교체한다.
- current가 없고 revision 1 이상이 먼저 도착해도 full snapshot이므로 적용 가능하다.
- 이후 도착한 revision 0은 더 오래된 상태로 무시한다.
- payout delta를 누적하지 않는다.
- VOIDED lifecycle 정정은 현재 V1 정정 계약 범위로 포함하지 않는다.

**경계 조건**

- current null + revision 0
- current null + revision 2
- current revision 2 + incoming revision 0
- current revision 2 + incoming revision 1
- 같은 revision·완전히 같은 payload
- 같은 revision·result만 다름
- 같은 revision·payout만 다름
- revision 2에서 revision 3으로 payout 증가
- revision 2에서 revision 3으로 payout 감소
- 서로 다른 `betId`

**실패 조건**

- revision number를 비교하지 않고 도착 순서대로 적용함
- 같은 revision의 다른 payload를 last-write-wins로 덮음
- delta를 누적해 중복 전달 때 payout이 두 번 반영됨
- 초기 settlement가 반드시 먼저 온다고 가정함
- `betId`가 다른 상태를 비교·교체함
- consumer가 previous snapshot과 정확히 일치해야만 새 snapshot을 적용할 수 있게 해 복구를 막음

**필요한 제약**

- reducer는 순수 함수로 작성하고 외부 I/O를 하지 않는다.
- payload equality는 projection에 필요한 전체 snapshot을 기준으로 한다.
- conflict의 격리·알림 방식은 호출자 책임이다.
- producer 측 wallet adjustment와 revision/outbox transaction은 설명만 하고 구현하지 않는다.

### 구현 후 자가 검증

- [ ] 더 작은 revision이 상태를 되돌리지 않는다.
- [ ] 같은 revision·같은 payload가 no-op이다.
- [ ] 같은 revision·다른 payload가 conflict다.
- [ ] 더 큰 revision이 전체 상태를 교체한다.
- [ ] current가 없는 상태에서 revision이 먼저 도착해도 처리한다.
- [ ] 나중에 도착한 revision 0을 무시한다.
- [ ] payout 증가와 감소를 모두 처리한다.
- [ ] 중복 전달로 payout을 누적하지 않는다.
- [ ] 서로 다른 `betId`를 거부한다.
- [ ] VOIDED 정정이 범위 밖임을 유지한다.
- [ ] reducer의 시간·공간 복잡도가 snapshot 크기에 비례하는지 설명한다.

### 구현 후 설명할 것

1. delta보다 full snapshot이 역순·누락·재처리에 강한 이유
2. revision number와 stable revision id가 각각 제공하는 의미
3. 같은 revision의 다른 payload를 격리해야 하는 이유
4. `betId` partition key가 bet 내부 순서에는 도움을 주지만 topic 간 전역 순서를 주지 않는다는 점
5. wallet adjustment 확인 후 revision과 outbox를 저장해야 하는 정합성 이유

### 원본 확인 위치

- Thread 14
- 커밋: `feat(events): define settled bet outcomes`
- 커밋: `build(avro): order shared named schemas`
- 커밋: `feat(events): define resolution revision snapshots`
- 커밋: `test(events): verify revision schema contract`
- 커밋: `test(events): verify payout increase revisions`
- 커밋: `test(events): verify payout decrease revisions`
- 파일: `src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc`
- 파일: `src/main/avro/com/sportsbook/protocol/event/BetResolutionRevised.avsc`
- 파일: `src/test/java/com/sportsbook/protocol/event/BetResolutionRevisedSchemaTest.java`
- 파일: `src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutIncreaseTest.java`
- 파일: `src/test/java/com/sportsbook/protocol/event/BetResolutionPayoutDecreaseTest.java`
- record·컴포넌트: `BetSettled`, `BetResolutionRevised`, `SettlementResultAvro`, Avro `Money`
- Thread 16 문서: `README.md`, `architecture/event-flow-and-consumer-map.md`
- 관련 Thread: 02, 10, 12, 13, 15, 16
- 실제 projection reducer·consumer 클래스 파일명은 현재 프로젝트 기록에서 확인되지 않는다.
