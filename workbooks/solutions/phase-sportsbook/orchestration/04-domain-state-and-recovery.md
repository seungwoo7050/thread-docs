# 도메인 상태, revision, replay, 감사 상관관계

이 문서는 마스터 인덱스의 IM-13, IM-14, IM-15, IM-16을 다룬다. Thread 10~13은 현재 프로젝트 요약에서 문서명과 주제는 확인되지만 개별 커밋 메시지나 원본 상태명은 노출되지 않았다. 따라서 아래 상태명과 축소 규칙은 원본 세부를 추측하지 않고 면접용으로 새로 정의한 것이다.

<a id="im-13"></a>
## [Thread 10 / 개별 커밋 메시지 미노출] 베팅 배치 상태 머신과 실패 복구

### 면접 질문

베팅 배치처럼 외부 I/O와 실패 복구가 있는 흐름을 상태 머신으로 표현할 때 어떤 invariant를 코드 경계에 강제해야 합니까?

꼬리 질문:

- 동일 요청이나 동일 이벤트가 재전달되면 상태와 외부 효과가 어떻게 되어야 합니까?
- terminal state에서 과거 state로 돌아가는 전이를 허용할 수 있습니까?
- 상태 저장은 성공했지만 외부 효과가 실패한 경우와 그 반대는 어떻게 구분합니까?
- retry 가능한 실패와 영구 실패를 같은 state로 표현하면 어떤 문제가 생깁니까?

### 30초 모범 답변

상태 머신의 핵심은 현재 상태와 입력 이벤트 조합마다 허용 전이와 금지 전이를 명시하는 것입니다. 중복 입력은 같은 결과로 수렴하고 외부 효과를 다시 만들지 않아야 하며, terminal state가 임의로 이전 상태로 회귀해서는 안 됩니다. 상태 기록과 외부 I/O의 실패 순서를 구분해 복구 가능한 중간 상태를 남기고, retry 가능 여부를 오류 원인과 함께 표현해야 합니다. 이렇게 하면 예외 처리 분기보다 invariant를 테스트하기 쉬워집니다.

### 답변 핵심 키워드

finite state machine, explicit transition, illegal transition, idempotency, terminal state, side-effect boundary, retryable vs terminal failure, convergence

### 백지 구현

아래 상태명은 **면접용 축소 문제 전용**이며 원본 상태명을 재구성한 것이 아니다.

**구현 목표**

현재 상태와 이벤트를 받아 다음 상태와 새로 실행할 effect를 반환하는 순수 transition 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
enum PlacementState { RECEIVED, RESERVED, CONFIRMED, FAILED }
enum PlacementEvent { RESERVE_OK, RESERVE_FAILED, CONFIRM_OK, CONFIRM_FAILED, RETRY }
enum Effect { REQUEST_RESERVATION, REQUEST_CONFIRMATION, NONE }
record Transition(PlacementState nextState, Effect effect) {}

// 직접 구현
Transition transition(PlacementState current, PlacementEvent event) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 현재 상태와 하나의 event
- 출력: 다음 상태와 새로 실행해야 할 effect

**반드시 만족해야 할 조건**

- 허용하지 않은 상태·event 조합은 명시적 오류다.
- `CONFIRMED`는 terminal이며 다른 상태로 돌아가지 않는다.
- 같은 성공 event가 재전달돼도 새로운 effect를 만들지 않는다.
- `FAILED`에서의 `RETRY` 허용 여부와 다음 상태를 명시한다.
- 상태 전이 결정 함수는 외부 I/O를 수행하지 않는다.

**경계 조건**

- 첫 event가 예상 순서와 다름
- 성공 event 중복
- failure event 중복
- terminal state에서 retry
- null 입력

**실패 조건**

- illegal transition
- 정의되지 않은 event
- 복구 정책이 없는 failed state

**필요한 제약**

- effect 실행 결과는 새 event로 돌아오며 transition 함수 내부에서 직접 호출하지 않는다.
- 원본과 다른 상태 모델도 invariant를 만족하면 가능한 해법으로 본다.

### 구현 후 자가 검증

- 모든 상태·event 조합이 허용 또는 거부로 명시돼 있는가?
- 중복 성공 event에서 상태와 effect가 안정적인가?
- terminal state가 회귀하지 않는가?
- 실패와 retry 경로가 무한 effect 반복을 만들지 않는가?
- 순수 함수라서 동일 입력에 동일 출력이 나오는가?
- 외부 effect 실행과 state transition 책임이 분리됐는가?
- 상태 수와 event 수가 늘어날 때 분기 구조를 설명할 수 있는가?

### 구현 후 설명할 것

1. 상태 전이와 effect 실행을 분리한 이유
2. 중복 event를 오류가 아니라 멱등 처리할 수 있는 조건
3. terminal state와 recoverable failed state의 차이
4. illegal transition을 조용히 무시하지 않는 이유
5. 상태 머신 table, switch, polymorphism 중 선택한 표현의 trade-off

### 원본 확인 위치

- Thread 10 — 베팅 배치 상태 머신과 실패 복구
- 현재 프로젝트 요약에서 개별 커밋 메시지는 노출되지 않음
- 파일: `10-bet-placement-state-machine-and-failure-recovery-01.md`
- 파일: `10-bet-placement-state-machine-and-failure-recovery-02.md`
- 관련 Thread: 4, 5, 9, 12

---

<a id="im-14"></a>
## [Thread 11 / 개별 커밋 메시지 미노출] 결과 후보 선택과 revision 복구

### 면접 질문

여러 settlement result candidate와 revision이 들어올 수 있을 때 어떤 규칙으로 유효 결과를 선택하고, stale·duplicate·conflicting revision을 어떻게 구분합니까?

꼬리 질문:

- 더 낮은 revision이 늦게 도착하면 무시, 기록, 오류 중 무엇을 선택합니까?
- 같은 revision인데 payload가 다르면 멱등 재전달입니까, 데이터 충돌입니까?
- 높은 revision 적용 중 실패한 뒤 재시작하면 어디에서 복구합니까?
- revision만 높으면 무조건 신뢰할 수 있습니까?

### 30초 모범 답변

revision 기반 복구에서는 적용된 revision이 뒤로 가지 않는 단조성이 핵심입니다. 낮은 revision은 stale로 분리하고, 같은 revision의 동일 내용은 duplicate로 수렴시키되 내용이 다르면 충돌로 실패시켜야 합니다. 더 높은 revision은 유효성 검사를 통과한 뒤에만 현재 상태를 대체하고, 적용 여부를 durable하게 남겨 재시작 후에도 같은 판단을 해야 합니다. revision 숫자는 순서 기준일 뿐 신뢰성 검증을 대신하지 않습니다.

### 답변 핵심 키워드

monotonic revision, stale, duplicate, same-revision conflict, validation before replace, durable recovery point, deterministic reducer

### 백지 구현

아래 revision 규칙은 **면접용 축소 조건**이며 원본 candidate 구조를 복원한 것이 아니다.

**구현 목표**

현재 선택된 candidate와 새 candidate를 비교해 `ACCEPT_NEW`, `DUPLICATE`, `STALE`, `CONFLICT`를 판정한다.

**인터페이스 또는 함수 시그니처**

```java
record ResultCandidate(String resultId, long revision, String payloadHash) {}
enum CandidateDecision { ACCEPT_NEW, DUPLICATE, STALE, CONFLICT }

// 직접 구현
CandidateDecision decideCandidate(
        Optional<ResultCandidate> current,
        ResultCandidate incoming) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 현재 candidate, 새 candidate
- 출력: 네 가지 판정 중 하나

**반드시 만족해야 할 조건**

- 현재 값이 없으면 유효한 incoming은 `ACCEPT_NEW`다.
- incoming revision이 낮으면 `STALE`다.
- 같은 revision과 같은 identity/content는 `DUPLICATE`다.
- 같은 revision인데 identity 또는 content가 다르면 `CONFLICT`다.
- 더 높은 revision은 기본적으로 `ACCEPT_NEW`지만 별도 payload validation이 선행된다고 가정한다.

**경계 조건**

- revision 0
- 매우 큰 revision
- 같은 payload hash지만 result ID가 다름
- 같은 result ID지만 payload hash가 다름
- current 없음

**실패 조건**

- 음수 revision
- 빈 result identity
- 빈 payload hash

**필요한 제약**

- 이 함수는 current state를 직접 변경하지 않는다.
- accept 판정 뒤 durable update의 원자성은 별도 계층 책임이다.

### 구현 후 자가 검증

- revision 대소와 동일성 조합이 네 판정으로 정확히 나뉘는가?
- lower revision이 current를 덮지 않는가?
- same revision conflict를 duplicate로 오판하지 않는가?
- malformed incoming이 판정 전에 거부되는가?
- 동일 입력에 동일 판정이 나오는가?
- current update와 decision 책임을 분리했는가?
- O(1) 비교라고 설명할 수 있는가?

### 구현 후 설명할 것

1. monotonic revision invariant
2. duplicate와 same-revision conflict를 구분하는 기준
3. higher revision도 payload validation이 필요한 이유
4. pure decision과 durable state update를 분리한 이유
5. stale input을 완전히 버릴지 audit에 남길지의 trade-off

### 원본 확인 위치

- Thread 11 — 정산 결과 후보와 revision 복구
- 현재 프로젝트 요약에서 개별 커밋 메시지는 노출되지 않음
- 파일: `11-settlement-result-candidates-and-revision-recovery-01.md`
- 파일: `11-settlement-result-candidates-and-revision-recovery-02.md`
- 관련 Thread: 9, 12, 13

---

<a id="im-15"></a>
## [Thread 12 / 개별 커밋 메시지 미노출] event ordering, replay, DLT 분류 invariant

### 면접 질문

이벤트 처리에서 duplicate, out-of-order, missing gap, poison event를 왜 하나의 재시도 오류로 처리하면 안 됩니까?

꼬리 질문:

- 이미 적용한 event가 replay되면 side effect를 다시 실행하지 않으면서 무엇을 갱신할 수 있습니까?
- 미래 순서 event가 먼저 왔을 때 무한 대기와 즉시 DLT 사이 기준은 무엇입니까?
- DLT로 보낼 non-retriable 오류와 일시적 infrastructure 오류를 어떻게 구분합니까?
- ordering state가 유실된 뒤 replay하면 어떤 invariant로 복구합니까?

### 30초 모범 답변

duplicate는 이미 처리된 입력이고, out-of-order나 gap은 아직 선행 event가 오지 않은 상태이며, poison event는 같은 입력을 다시 처리해도 성공할 수 없는 문제라 복구 전략이 다릅니다. 적용된 순서를 durable하게 추적하고 replay는 같은 상태로 수렴시켜 side effect를 중복 생성하지 않아야 합니다. gap은 bounded retry나 보류 정책을 두고, 구조적으로 유효하지 않거나 비재시도 오류만 DLT로 보냅니다. 분류를 섞으면 정상 지연이 데이터 손실로 바뀌거나 poison event가 무한 재시도됩니다.

### 답변 핵심 키워드

duplicate, out-of-order, gap, replay, durable ordering state, idempotent effect, bounded retry, poison event, DLT, error taxonomy

### 백지 구현

아래 sequence 규칙은 **면접용 축소 조건**이며 원본 event envelope을 복원한 것이 아니다.

**구현 목표**

현재까지 적용한 sequence와 incoming event를 보고 `APPLY`, `DUPLICATE`, `DEFER`, `DEAD_LETTER`를 판정한다.

**인터페이스 또는 함수 시그니처**

```java
record EventEnvelope(String streamId, long sequence, boolean structurallyValid, boolean retryableFailure) {}
record ProcessingState(String streamId, long lastAppliedSequence) {}
enum EventDecision { APPLY, DUPLICATE, DEFER, DEAD_LETTER }

// 직접 구현
EventDecision classify(EventEnvelope event, ProcessingState state) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: event와 stream별 last applied sequence
- 출력: 처리 분류

**반드시 만족해야 할 조건**

- structurally invalid event는 `DEAD_LETTER`다.
- `sequence <= lastAppliedSequence`는 `DUPLICATE`다.
- `sequence == lastAppliedSequence + 1`은 `APPLY`다.
- 더 큰 gap은 `DEFER`다.
- stream identity가 다르면 입력 오류다.
- retryableFailure flag를 이 순수 분류에 어떻게 반영할지 명시한다.

**경계 조건**

- 첫 event의 sequence 시작값
- sequence 0
- 매우 큰 gap
- 같은 sequence 재전달
- structurally invalid duplicate

**실패 조건**

- 음수 sequence
- stream identity mismatch
- malformed state

**필요한 제약**

- 실제 side effect 실행과 lastAppliedSequence 갱신은 별도 원자적 경계가 필요하다.
- `DEFER`의 보관 한도와 timeout은 caller 정책이다.

### 구현 후 자가 검증

- apply, duplicate, gap, poison 경로가 명확히 분리되는가?
- duplicate가 side effect 재실행 대상으로 분류되지 않는가?
- gap을 즉시 DLT로 보내지 않는가?
- invalid event와 stream mismatch를 구분하는가?
- overflow 가능성을 고려했는가?
- 동일 event replay가 같은 판정으로 수렴하는가?
- 분류는 O(1)이고 deferred buffer는 별도 공간 비용임을 설명할 수 있는가?

### 구현 후 설명할 것

1. duplicate, gap, poison event의 복구 전략 차이
2. ordering state를 durable하게 가져야 하는 이유
3. DLT를 재시도 횟수 초과 쓰레기통으로만 쓰면 안 되는 이유
4. classification과 side effect/update 원자성을 분리한 이유
5. gap buffer의 메모리·timeout trade-off

### 원본 확인 위치

- Thread 12 — 이벤트 순서, replay, DLT invariant
- 현재 프로젝트 요약에서 개별 커밋 메시지는 노출되지 않음
- 파일: `12-event-ordering-replay-and-dlt-invariants-01.md`
- 파일: `12-event-ordering-replay-and-dlt-invariants-02.md`
- 관련 Thread: 5, 8, 9, 10, 11, 13

---

<a id="im-16"></a>
## [Thread 13 / 개별 커밋 메시지 미노출] admin audit와 downstream correlation

### 면접 질문

admin 동작의 감사 기록과 downstream 처리 상관관계를 남길 때 current state log만으로 충분하지 않은 이유는 무엇입니까?

꼬리 질문:

- correlation ID와 causation ID는 각각 무엇을 연결합니까?
- 동일 명령이 재전달되면 audit event도 중복 저장해도 됩니까?
- audit payload에 secret이나 불필요한 개인정보가 들어가지 않도록 어디에서 제한합니까?
- downstream 실패와 재시도를 한 요청 계보로 어떻게 추적합니까?

### 30초 모범 답변

current state는 최종 결과만 보여 주고 누가 어떤 명령을 실행해 어떤 downstream 효과를 만들었는지는 설명하지 못합니다. audit event는 append-only 사실로 남기고, correlation ID로 하나의 업무 흐름을, causation ID로 바로 앞 원인을 연결해야 합니다. 재전달은 event identity로 중복을 통제하되 실패와 재시도 자체는 별도 사실로 남길 수 있습니다. audit에는 추적에 필요한 최소 필드만 넣고 secret은 Thread 15의 evidence 경계와 같은 원칙으로 차단해야 합니다.

### 답변 핵심 키워드

append-only audit, event identity, actor, action, target, correlation, causation, retry lineage, minimal data, secret exclusion

### 백지 구현

**구현 목표**

감사 event가 downstream correlation에 필요한 최소 계약을 만족하는지 검사한다. 저장소 구현은 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record AuditEvent(
        String eventId,
        String actorId,
        String action,
        String targetId,
        String correlationId,
        Optional<String> causationId,
        Instant occurredAt,
        Map<String, String> attributes) {}

// 직접 구현
List<String> validateAuditEvent(AuditEvent event, Set<String> forbiddenAttributeKeys) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: audit event와 금지 attribute key 집합
- 출력: 모든 계약 위반 목록

**반드시 만족해야 할 조건**

- event, actor, action, target, correlation identity가 비어 있지 않아야 한다.
- causation ID가 자기 event ID와 같을 수 없다.
- occurredAt이 누락되면 안 된다.
- 금지 attribute key와 blank key를 거부한다.
- 검증 오류에 민감 attribute value를 복사하지 않는다.

**경계 조건**

- 최초 root event라 causation ID가 없음
- attributes가 비어 있음
- 같은 event ID가 다시 입력됨: validator와 dedup store 책임을 구분
- timestamp가 미래 또는 지나치게 오래됨

**실패 조건**

- 필수 identity 누락
- self-causation
- forbidden key
- malformed timestamp

**필요한 제약**

- append-only와 unique event ID enforcement는 저장 계층에서 별도로 보장한다.
- correlation graph cycle 검출은 단일 event validator의 범위를 벗어난다.

### 구현 후 자가 검증

- root event와 child event가 모두 유효하게 표현되는가?
- self-causation과 필수 field 누락을 찾는가?
- forbidden attribute가 있더라도 value를 오류에 노출하지 않는가?
- validation과 dedup, append-only persistence 책임이 구분되는가?
- 동일 입력에 동일 오류 순서가 나오는가?
- 최소 필드만으로 downstream lineage를 설명할 수 있는가?

### 구현 후 설명할 것

1. current state와 audit event의 역할 차이
2. correlation과 causation을 분리하는 이유
3. duplicate delivery와 retry fact를 동시에 보존하는 방법
4. validator, dedup store, append-only storage의 책임 경계
5. audit 최소화와 조사 가능성 사이의 trade-off

### 원본 확인 위치

- Thread 13 — 관리자 감사와 downstream 상관관계
- 현재 프로젝트 요약에서 개별 커밋 메시지는 노출되지 않음
- 파일: `13-admin-audit-and-downstream-correlation.md`
- 관련 Thread: 9, 11, 12, 15, 16
