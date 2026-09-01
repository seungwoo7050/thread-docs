# 기본 정산, 전체 무효화, 순서화된 revision

이 문서는 처음 도착한 정산 snapshot을 검증해 투영하는 규칙과, 이후 full replacement revision을 순서·중복·갭·충돌에 따라 적용하는 규칙을 분리한다. 두 문제 모두 payload identity, actor ownership, 금액 invariant, 영구 충돌 격리가 핵심이다.

<a id="t15-base-settlement"></a>
## [Thread 15 / `feat(settlement): project base resolution snapshots`] 기본 정산과 whole-slip void를 검증 가능한 snapshot으로 투영하기

### 면접 질문

첫 `BetSettled` 또는 `BetVoided` event를 적용할 때 bet을 잠그고 canonical ID, owner, event ID와 payload hash를 검증한 이유는 무엇입니까? SYSTEM whole-slip void의 refund가 원래 unit stake가 아니라 committed total exposure와 같아야 하는 이유도 설명해 보세요.

꼬리 질문:

- base resolution이 아직 없음을 revision number `-1`, 적용됨을 `0`, revision 적용됨을 `1+`로 표현하면 어떤 판정이 가능합니까?
- 같은 event ID와 같은 hash가 다시 오면 왜 no-op입니까?
- 같은 revision 0 상태에서 event ID나 hash가 다르면 왜 덮어쓰지 않고 충돌로 격리합니까?
- 이미 higher revision이 적용된 뒤 늦은 base event가 오면 어떻게 해야 합니까?
- event userId가 bet owner와 다르면 retry로 회복될 수 있습니까?
- whole-slip void refund를 event 값만 믿지 않고 다시 계산한 이유는 무엇입니까?
- SYSTEM의 event stake가 unit stake라는 계약을 잊으면 어떤 과다·과소 환불이 생깁니까?
- `MARKET_VOID`를 별도 whole-slip void event가 아니라 settled VOID result로 제한한 이유는 무엇입니까?

### 30초 모범 답변

정산 event는 돈을 확정하는 외부 입력이므로 bet root를 잠그고 canonical ID와 owner를 먼저 검증합니다. base가 없는 경우만 적용하고, 동일 event·동일 payload는 안전한 중복으로 무시하지만 revision 0에서 다른 내용은 producer 충돌이므로 덮어쓰지 않습니다. higher revision 뒤에 도착한 base는 superseded라 무시합니다. whole-slip void의 refund는 SYSTEM의 한 줄 stake가 아니라 `unit stake × 조합 수`인 committed exposure와 같아야 하므로 도메인 계산기로 재검증합니다.

### 답변 핵심 키워드

`base snapshot`, `canonical identity`, `actor ownership`, `payload hash`, `duplicate vs conflict`, `superseded base`, `whole-slip void`, `committed exposure`, `pessimistic lock`

### 백지 구현

**구현 목표**

현재 projection 상태와 base resolution 명령을 받아 APPLY, DUPLICATE, SUPERSEDED, CONFLICT를 판정하고, 적용 시 금액·소유권 invariant를 검증한다.

**인터페이스 또는 함수 시그니처**

```java
enum BaseDecision { APPLIED, DUPLICATE, SUPERSEDED }
enum Result { WON, LOST, VOID }

record Projection(
    UUID betId,
    UUID ownerId,
    long revisionNumber,
    UUID resolutionEventId,
    String payloadHash,
    String slipType,
    int minWins,
    int totalSelections,
    long unitStake,
    String currency) {}

record BaseResolution(
    UUID betId,
    UUID actorId,
    UUID eventId,
    Result result,
    long payoutOrRefund,
    String currency,
    String payloadHash) {}

final class BaseSettlement {
  BaseDecision apply(Projection current, BaseResolution incoming) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: 현재 projection, base event
- 출력: 적용·중복·superseded 판정. 충돌은 예외 또는 별도 결과로 표현한다.

**반드시 만족해야 할 조건**

- bet ID와 actor가 정확히 일치한다.
- payload hash는 canonical lowercase SHA-256 형식이다.
- `revisionNumber > 0`이면 base는 SUPERSEDED다.
- `revisionNumber == 0`에서 event ID와 hash가 모두 같으면 DUPLICATE다.
- `revisionNumber == 0`에서 내용이 다르면 CONFLICT다.
- base 미적용 상태에서 payout/refund는 음수가 아니고 currency가 일치한다.
- whole-slip VOID의 refund는 committed exposure와 같다.
- SYSTEM exposure는 조합 수와 unit stake로 계산한다.

**경계 조건**

- base 미적용 상태
- 동일 replay
- 같은 event ID·다른 hash
- 다른 event ID·같은 hash
- higher revision 뒤 늦은 base
- owner mismatch
- SYSTEM 1-of-N, N-of-N
- 환불 1 차이, currency mismatch

**실패 조건**

owner, ID, hash, 금액, projection 충돌은 영구 오류로 분류할 수 있어야 한다. 충돌 시 현재 projection을 변경하지 않는다.

**제약**

실제 event schema와 DB는 제외한다. canonical UUID 문자열 검사까지 포함할 경우 문자열 입력 overload를 추가할 수 있다.

### 구현 후 자가 검증

- [ ] 미적용 projection에 정상 base가 한 번만 적용된다.
- [ ] 동일 event·hash replay는 no-op이다.
- [ ] 동일 revision의 다른 내용은 덮어쓰지 않는다.
- [ ] higher revision 이후 base가 상태를 되돌리지 않는다.
- [ ] actor mismatch와 unknown bet 성격을 영구 오류로 설명할 수 있다.
- [ ] SYSTEM whole-slip refund가 total exposure와 일치한다.
- [ ] unit stake를 refund와 직접 비교하는 실수를 하지 않는다.
- [ ] 실패 시 projection이 부분 변경되지 않는다.

### 구현 후 설명할 것

1. event ID와 payload hash를 함께 저장한 이유
2. duplicate, conflict, superseded의 의미 차이
3. base와 revision 번호 체계가 단조성에 주는 이점
4. 외부 refund를 내부 committed exposure로 재검증한 이유
5. root lock이 같은 bet의 동시 resolution을 직렬화하는 방식

### 원본 확인 위치

- Thread: 15, 검증된 기본 정산과 전체 슬립 무효화 투영
- 커밋: `feat(settlement): project base resolution snapshots`, `feat(settlement): preserve raw resolution keys`
- 파일: `BetSettlementService.java`, `SettlementResultListener.java`, `Bet.java`, `SystemBetCalculator.java`
- 함수·컴포넌트: `BetSettlementService.apply(BetSettled, ...)`, `BetSettlementService.apply(BetVoided, ...)`, `Bet.settleBase(...)`, `Bet.voidBase(...)`, `SystemBetCalculator.totalStake(...)`
- 관련 Thread: 02, 12, 16, 17

---

<a id="t16-ordered-revisions"></a>
## [Thread 16 / `feat(settlement): apply full revision snapshots`] 낮은 revision, 중복, 갭, 충돌을 서로 다르게 처리하기

### 면접 질문

정산 수정 event를 delta가 아니라 이전·새 결과와 payout을 모두 가진 full snapshot으로 받고, revision number에 따라 IGNORED, DUPLICATE, APPLIED, APPLIED_WITH_GAP, CONFLICT를 구분한 이유는 무엇입니까?

꼬리 질문:

- incoming revision이 current보다 낮으면 왜 오류가 아니라 IGNORED입니까?
- 같은 revision number에서 revision ID와 hash가 같으면 왜 DUPLICATE이고, 다르면 왜 CONFLICT입니까?
- current+1보다 큰 revision을 무조건 거부하지 않고 적용하되 gap metric을 남긴 이유는 무엇입니까?
- gap이 없을 때만 incoming `previousResult/previousPayout`과 현재 projection의 일치를 강제한 이유는 무엇입니까?
- gap이 있는 full snapshot도 적용 가능한 전제는 무엇입니까?
- 수정 event의 `eventId`가 실제 slip 선택 event 중 하나인지 확인한 이유는 무엇입니까?
- `sourceSettledAt <= revisedAt` chronology를 검증한 이유는 무엇입니까?
- VOIDED terminal bet에 revision을 금지한 이유는 무엇입니까?
- invalid revision을 permanent Kafka error로 감싸는 이유는 무엇입니까?

### 30초 모범 답변

revision은 순서가 뒤섞이고 중복될 수 있으므로 번호와 immutable identity를 함께 봅니다. 낮은 번호는 늦은 전달이라 무시하고, 같은 번호의 같은 ID·hash는 중복, 다른 내용은 split-brain 충돌입니다. 다음 번호는 이전 projection과 `previous` snapshot을 비교해 연속성을 검증합니다. 번호가 건너뛰면 중간 이력을 검증할 수 없지만 event가 full replacement snapshot이므로 최신 상태는 적용하고 gap을 계측합니다. actor, 선택 event, 금액·통화, chronology가 맞지 않는 입력은 재시도로 회복되지 않는 영구 오류로 격리합니다.

### 답변 핵심 키워드

`ordered revision`, `full snapshot`, `monotonic projection`, `duplicate identity`, `split-brain conflict`, `gap tolerance`, `continuity check`, `chronology`, `permanent isolation`, `observability metric`

### 백지 구현

**구현 목표**

현재 정산 projection에 full revision snapshot을 적용하는 순수 함수를 구현한다. 반환 결과는 `IGNORED`, `DUPLICATE`, `APPLIED`, `APPLIED_WITH_GAP` 중 하나이며 충돌·invalid input은 명시적으로 실패한다.

**인터페이스 또는 함수 시그니처**

```java
enum RevisionDecision { IGNORED, DUPLICATE, APPLIED, APPLIED_WITH_GAP }
enum SettlementResult { WON, LOST, VOID }

record SettlementProjection(
    Set<UUID> selectedEventIds,
    long currentRevision,
    UUID currentRevisionId,
    String currentPayloadHash,
    SettlementResult result,
    long payout,
    String currency,
    Instant settledAt) {}

record RevisionSnapshot(
    UUID eventId,
    UUID revisionId,
    long revisionNumber,
    SettlementResult previousResult,
    long previousPayout,
    SettlementResult newResult,
    long newPayout,
    String currency,
    Instant sourceSettledAt,
    Instant revisedAt,
    String payloadHash) {}

final class RevisionProjector {
  RevisionDecision apply(SettlementProjection current, RevisionSnapshot revision) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: 현재 projection, incoming full revision snapshot
- 출력: 적용 판정. 구현 방식에 따라 새 projection을 함께 반환하는 result record를 만들어도 된다.

**반드시 만족해야 할 조건**

- revision number는 1 이상이다.
- revision event는 selected event 집합에 속한다.
- previous/new payout은 음수가 아니고 projection currency와 같다.
- `sourceSettledAt`과 `revisedAt`은 존재하며 source가 revised보다 늦지 않다.
- incoming number < current면 IGNORED다.
- incoming number == current이고 ID·hash가 모두 같으면 DUPLICATE다.
- 같은 번호에서 ID 또는 hash가 다르면 CONFLICT다.
- incoming number == current+1이면 현재 result·payout과 incoming previous가 일치해야 한다.
- incoming number > current+1이면 full snapshot을 적용하고 APPLIED_WITH_GAP을 반환한다.
- 어떤 적용도 revision number를 낮추지 않는다.

**경계 조건**

- current 0에 revision 1
- current 3에 revision 2, 3, 4, 6
- 같은 ID·다른 hash
- 다른 ID·같은 hash
- 연속 revision의 previous mismatch
- gap revision의 previous mismatch
- payout 0
- source와 revised 시각이 같음
- selected event가 아닌 ID

**실패 조건**

충돌과 invalid snapshot에서 projection을 바꾸지 않는다. 영구 오류로 보내야 할 failure와 transient storage failure를 구분할 수 있게 예외 경계를 둔다.

**제약**

metric registry와 Kafka listener는 제외한다. `APPLIED_WITH_GAP` 반환을 호출자가 계측한다고 가정한다.

### 구현 후 자가 검증

- [ ] 낮은 revision이 projection을 되돌리지 않는다.
- [ ] 같은 revision의 정확한 replay만 DUPLICATE다.
- [ ] 같은 번호의 다른 identity가 충돌한다.
- [ ] 연속 revision에서 previous snapshot을 검증한다.
- [ ] gap revision은 full snapshot으로 적용되고 별도 결과를 낸다.
- [ ] gap이 아닌데 previous mismatch인 입력은 거부된다.
- [ ] 선택 event 소속, currency, 음수 payout, chronology를 검증한다.
- [ ] 실패 시 current state가 그대로다.
- [ ] gap metric은 실제 APPLIED_WITH_GAP에만 증가해야 한다고 설명할 수 있다.

### 구현 후 설명할 것

1. revision number와 revision ID·hash를 함께 쓰는 이유
2. 낮은 revision을 ignore하고 같은 revision 충돌을 격리하는 이유
3. full snapshot이 gap 적용을 가능하게 하는 조건
4. 연속 revision에서 previous snapshot을 optimistic precondition처럼 사용하는 이유
5. gap을 가용성 측면에서 적용하면서 관측 가능성으로 보완한 trade-off

### 원본 확인 위치

- Thread: 16, 순서화된 정산 수정, 갭, 충돌 격리
- 커밋: `feat(database): persist resolution revisions`, `feat(settlement): apply full revision snapshots`, `feat(settlement): consume ordered resolution revisions`, `feat(settlement): isolate permanent projection conflicts`, `feat(settlement): count revision projection gaps`, `feat(settlement): bind revisions to selected events`, `fix(settlement): validate revision chronology`
- 파일: `Bet.java`, `BetSettlementService.java`, `SettlementResultListener.java`, `V9__resolution_revision_projection.sql`
- 함수·컴포넌트: `Bet.applyRevision(...)`, `BetSettlementService.apply(BetResolutionRevised, ...)`, `SettlementResultListener.onResolution(...)`, revision gap metric
- 관련 Thread: 10, 12, 15, 17
