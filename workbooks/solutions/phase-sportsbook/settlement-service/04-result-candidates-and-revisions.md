# 결과 후보·인과 순서·revision 계획 면접 워크북

이 문서는 공식 결과를 덮어쓰는 값이 아니라 immutable candidate로 축적하고, predecessor·sequence·database time으로 인과 순서를 지킨 뒤 correction plan을 영속화하는 과정을 다룬다.

<a id="p16"></a>
<!-- POINT:P16 -->
## P16 — [Thread 13 / `feat(correction): fingerprint semantic resolutions`] 결과 후보의 의미 fingerprint와 결정 상태

### 면접 질문

공식 경기 결과를 바로 `match_result`에 덮어쓰지 않고 immutable `ResultCandidate`로 저장한 이유는 무엇입니까? `VOIDED` mode의 fingerprint에서 selection outcomes를 제외한 이유도 설명해 보세요.

꼬리 질문:

- 같은 event의 payload 순서만 다른 outcomes map은 같은 후보여야 합니까?
- exact replay, already accepted replay, late candidate, future candidate를 왜 구분합니까?
- fingerprint unique constraint만으로 첫 결과 race까지 해결됩니까?

### 30초 모범 답변

결과 변경은 금전 correction의 원인이므로 원본 후보와 판단 이력을 덮어쓰지 않고 immutable candidate로 남겼습니다. fingerprint는 event ID, mode, 그리고 selection ID 기준으로 정렬한 outcome 의미를 hash해 map 순서와 무관하게 만들었습니다. `VOIDED` mode는 개별 outcome이 의사결정 의미에 참여하지 않으므로 제외했습니다. unique `(event_id, fingerprint)`로 semantic replay를 합치고, candidate state와 수신·settled 시각을 이용해 accepted replay, future hold, late hold, stale supersede를 구분합니다.

### 답변 핵심 키워드

immutable evidence, semantic fingerprint, sorted map, mode-dependent identity, exact replay, candidate state, future/late hold

### 백지 구현

#### 구현 목표

결과 candidate의 semantic fingerprint를 계산하고, 현재 accepted candidate와 시간 정책을 기준으로 intake decision을 반환하는 순수 컴포넌트를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class ResultCandidateIntakePolicy {
  public CandidateDecision decide(
      IncomingCandidate incoming,
      Optional<AcceptedCandidate> accepted,
      Instant databaseNow,
      Duration correctionWindow,
      Set<String> knownFingerprints) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: event/mode/outcomes/settledAt/receivedAt, 현재 accepted 후보, DB 시각, correction window, 기존 fingerprints
- 출력: fingerprint와 `EXACT_REPLAY`, `FUTURE_HELD`, `FIRST_CANDIDATE`, `REPLACEMENT_CANDIDATE`, `LATE_HELD` 중 하나

#### 반드시 만족해야 할 조건

- fingerprint는 소문자 SHA-256 형식이다.
- outcomes는 selection ID 기준의 결정적 순서로 canonicalize한다.
- mode가 `VOIDED`이면 개별 outcomes가 fingerprint 의미에 참여하지 않는다.
- 알려진 fingerprint면 새 candidate를 만들지 않는 exact replay다.
- `settledAt > databaseNow`인 후보는 자동 승인 대상이 아니다.
- accepted 후보가 없으면 first candidate 후보가 된다.
- accepted 후보가 있고 correction window를 넘으면 late held다.
- 그 밖의 새 후보는 replacement 후보일 뿐 이 함수가 accepted 상태를 직접 변경하지 않는다.
- 입력 map을 변경하지 않는다.

#### 경계 조건

- outcomes map 순서만 다른 입력
- `VOIDED` mode에서 서로 다른 outcomes
- settledAt이 databaseNow와 같은 후보
- receivedAt이 correction window 경계와 같은 후보
- accepted candidate가 없는 exact replay
- 빈 outcomes가 허용되는 mode와 허용되지 않는 mode

#### 실패 조건

- event ID 또는 필수 시각 누락
- 결과 map iteration 순서에 따라 fingerprint 변경
- future candidate 자동 승인
- late candidate 자동 replacement
- fingerprint 충돌을 서로 다른 의미의 정상 replay로 간주

#### 필요한 제약

- hash input은 길이 또는 구분 경계를 명확히 인코딩한다.
- 이 문제에서는 persistence race를 해결하지 않고 결정 후보만 반환한다.
- 시간 기준은 caller의 system clock이 아니라 전달된 database time이다.

### 구현 후 자가 검증

- map 삽입 순서가 달라도 같은 fingerprint인가?
- `VOIDED` mode에서 outcomes 차이가 fingerprint를 바꾸지 않는가?
- mode나 event ID가 바뀌면 fingerprint가 달라지는가?
- exact replay가 새 candidate decision으로 바뀌지 않는가?
- future와 late 경계의 등호 처리가 일관적인가?
- 입력 outcomes map이 수정되지 않는가?
- accepted 상태 변경은 별도 atomic store operation이 필요하다고 설명했는가?

### 구현 후 설명할 것

1. overwrite 대신 candidate evidence를 쌓은 이유
2. semantic identity에서 map 순서를 제거한 방법
3. `VOIDED` mode의 의미가 fingerprint 구조에 미친 영향
4. replay·future·late를 서로 다른 상태로 유지한 이유
5. fingerprint deduplication과 acceptance race의 책임 차이

### 원본 확인 위치

- Thread: 13 — 결과 후보 결정과 인과 순서
- 대표 커밋: `feat(correction): fingerprint semantic resolutions`
- 관련 커밋: `feat(correction): capture immutable result snapshots`, `feat(correction): deduplicate semantic candidates`, `feat(correction): hold late result candidates`, `fix(correction): fence future result decisions`, `feat(correction): gate candidate decisions by database time`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidate.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateState.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateFingerprinter.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java`
- 관련 메서드: `ResultCandidateIntake.ingest`, `ResultCandidateStore.record`, `ResultCandidateStore.holdWhileFuture`
- 관련 Thread: 2, 16, 19

<a id="p17"></a>
<!-- POINT:P17 -->
## P17 — [Thread 13 / `feat(correction): enforce candidate causal order`] predecessor CAS와 단조 증가 sequence

### 면접 질문

새 candidate가 현재 accepted result를 교체하려면 왜 `expectedAcceptedId`, `replacesCandidateId`, 더 큰 `candidateSequence`, `settledAt <= current_timestamp`를 모두 검사해야 합니까?

꼬리 질문:

- expected accepted ID만 비교하면 어떤 ABA 또는 역전 문제가 남습니까?
- 두 replacement가 같은 predecessor를 가리키며 동시에 승인되면 어떻게 됩니까?
- 관리자 승인도 자동 correction과 같은 predecessor fence를 사용해야 합니까?

### 30초 모범 답변

교체는 단순 update가 아니라 인과 chain의 CAS입니다. `expectedAcceptedId`는 읽은 뒤 다른 결과가 승인된 lost update를 막고, candidate가 그 predecessor를 명시해야 엉뚱한 base에서 계산된 correction을 막습니다. 더 큰 sequence는 과거 후보로 되돌아가는 역전을 막고, source settled 시각은 DB 현재보다 미래일 수 없습니다. 이 조건을 한 SQL predicate와 transaction 안에 넣어 competing replacement 중 하나만 성공시키며, 관리자 승인도 같은 fence를 통과해야 합니다.

### 답변 핵심 키워드

compare-and-set, predecessor fence, monotonic sequence, causal chain, lost update, future fence, one winner

### 백지 구현

#### 구현 목표

현재 accepted candidate와 교체 후보를 비교해 인과적으로 교체 가능한지 판정하고, 거부 이유를 반환한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class CandidateReplacementFence {
  public ReplaceDecision evaluate(
      AcceptedCandidate current,
      ResultCandidate next,
      UUID expectedAcceptedId,
      Instant databaseNow) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 accepted candidate, 새 candidate, caller가 본 expected accepted ID, DB 시각
- 출력: `ALLOWED` 또는 구체적인 stale/future/state/predecessor/sequence 거부 이유

#### 반드시 만족해야 할 조건

- current ID와 expected accepted ID가 같아야 한다.
- next는 `PENDING`이어야 한다.
- current와 next는 같은 event에 속해야 한다.
- next의 `replacesCandidateId`가 current ID와 같아야 한다.
- next sequence는 current sequence보다 커야 한다.
- next settledAt은 databaseNow보다 미래일 수 없다.
- 함수는 상태를 변경하지 않고 atomic update predicate를 위한 판단만 한다.

#### 경계 조건

- sequence가 정확히 1 큰 후보
- sequence가 같은 후보
- current가 평가 직후 다른 candidate로 바뀌는 race
- settledAt이 databaseNow와 같은 후보
- accepted replay candidate

#### 실패 조건

- predecessor가 null 또는 다른 후보
- 다른 event candidate
- stale expected ID
- future source
- pending이 아닌 candidate 교체
- sequence 역전

#### 필요한 제약

- 실제 적용은 이 검사를 다시 표현한 단일 transaction/SQL CAS여야 한다.
- 평가 후 update 사이의 시간 간격을 안전하다고 가정하지 않는다.
- 거부 이유에 민감한 payload를 포함하지 않는다.

### 구현 후 자가 검증

- 각 조건을 하나씩 깨뜨렸을 때 정확한 거부 이유가 나오는가?
- 경계 시각의 등호가 허용되는가?
- sequence가 작거나 같은 후보가 거부되는가?
- current ID가 바뀐 race를 CAS 실패로 처리할 수 있는가?
- same predecessor의 두 후보가 DB update에서 하나만 성공한다고 설명했는가?
- 관리자 명령도 이 fence를 우회하지 않는가?

### 구현 후 설명할 것

1. 낙관적 CAS와 row lock을 함께 또는 대안으로 쓰는 방법
2. predecessor ID와 sequence가 각각 막는 오류
3. future source를 DB time으로 막는 이유
4. 자동·수동 승인 경로가 같은 invariant를 공유해야 하는 이유
5. race loser를 실패가 아닌 stale/superseded 상태로 남기는 이점

### 원본 확인 위치

- Thread: 13 — 결과 후보 결정과 인과 순서
- 대표 커밋: `feat(correction): enforce candidate causal order`
- 관련 커밋: `feat(correction): auto accept replacement results`, `feat(correction): supersede stale candidates`, `fix(correction): fence future result decisions`, `test(correction): verify PostgreSQL first result race`
- Thread 16 관련 커밋: `feat(admin): approve result candidates`
- Thread 19 관련 커밋: `feat(correction): gate candidate decisions by database time`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminCandidateApproval.java`
- 관련 메서드: `ResultCandidateStore.acceptFirst`, `ResultCandidateStore.replaceAccepted`, `ResultCandidateStore.supersedeStale`, `ResultCandidateStore.lockForAdmin`
- 관련 Thread: 16, 19

<a id="p18"></a>
<!-- POINT:P18 -->
## P18 — [Thread 14 / `feat(correction): persist plans before wallet`] immutable revision snapshot과 payout delta

### 면접 질문

correction에서 현재 bet을 바로 변경하지 않고 `RevisionTarget`과 `RevisionPlan`을 만들어 Wallet 호출 전에 영속화한 이유는 무엇입니까? plan에 previous payout과 new payout을 모두 보존하는 이유도 설명해 보세요.

꼬리 질문:

- retry 때 accepted result와 bet을 다시 읽어 plan을 재계산하면 어떤 문제가 생깁니까?
- delta가 0이면 왜 Wallet을 호출하지 않습니까?
- 같은 source candidate로 revision plan이 두 개 생기지 않게 어떤 경계가 필요합니까?

### 30초 모범 답변

revision은 이미 지급된 금액을 바꾸므로 외부 호출 전에 계산 기준을 완전히 고정해야 합니다. target에 bet·user·event·source candidate·현재 revision number·previous result/payout·slip과 selection snapshot·source settled 시각을 담고, plan에 새 result/payout과 독립 revision ID를 배정해 저장했습니다. retry는 이 plan을 reload해 같은 idempotency identity와 delta를 사용합니다. `(bet, revision number)`와 `(bet, source candidate)` 유일성이 중복 계획을 막고, zero delta는 외부 금전 작업 없이 상태만 원자적으로 확정합니다.

### 답변 핵심 키워드

immutable correction intent, previous/new snapshot, persist-before-Wallet, stable revision ID, exact delta, zero-delta fast path, unique source candidate

### 백지 구현

#### 구현 목표

현재 correction target과 새 settlement outcome을 받아 immutable revision plan을 생성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class RevisionPlanAllocator {
  public RevisionPlan allocate(
      UUID revisionId,
      RevisionTarget target,
      SettlementOutcome outcome,
      Instant createdAt) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 사전에 배정된 revision ID, immutable target, 새 result/payout, 생성 시각
- 출력: delta와 zero-delta 여부를 계산할 수 있는 immutable plan

#### 반드시 만족해야 할 조건

- 모든 식별자와 시각이 존재해야 한다.
- target revision number는 유효한 양수 순서여야 한다.
- previous payout과 new payout은 같은 통화다.
- new payout은 음수일 수 없다.
- delta는 `new - previous`이며 overflow를 감지한다.
- target의 selections는 defensive copy한다.
- plan 생성 후 target과 outcomes를 외부에서 변경할 수 없다.
- source candidate ID와 source result settled 시각을 보존한다.
- zero delta를 명시적으로 판정할 수 있어야 한다.

#### 경계 조건

- positive delta
- negative delta
- zero delta
- previous payout이 0
- 최대·최소 범위에 가까운 minor-unit 값
- selection이 여러 event에 걸친 snapshot

#### 실패 조건

- 통화 불일치
- 음수 new payout
- delta overflow
- revision ID 또는 source candidate 누락
- mutable selection list 공유
- previous/current 상태 정보 누락으로 retry 재검증 불가

#### 필요한 제약

- UUID 생성 자체보다 입력 revision ID의 안정성을 검증하는 데 집중한다.
- persistence 시에는 current bet revision과 source candidate를 다시 fence해야 한다.
- source settled 시각이 DB 현재보다 미래인 plan은 저장 대상이 아니다.

### 구현 후 자가 검증

- positive/negative/zero delta가 정확한가?
- 통화가 다르면 생성되지 않는가?
- 입력 selection list 수정이 plan에 영향을 주지 않는가?
- overflow가 wrap되지 않는가?
- 같은 target/outcome/revision ID로 같은 의미의 plan이 만들어지는가?
- zero delta가 Wallet 미호출 경로로 분기될 수 있는가?
- persistence에서 expected revision과 source candidate unique fence가 필요함을 설명했는가?

### 구현 후 설명할 것

1. correction 계산 기준을 snapshot으로 고정한 이유
2. previous와 new 값을 모두 저장한 이유
3. plan ID를 Wallet idempotency identity로 재사용하는 장점
4. zero-delta를 외부 호출에서 제외한 이유
5. snapshot 저장 비용과 재현 가능성의 trade-off

### 원본 확인 위치

- Thread: 14 — immutable revision planning과 payout delta
- 대표 커밋: `feat(correction): persist plans before wallet`
- 관련 커밋: `feat(correction): capture revision targets`, `feat(correction): allocate immutable revision plans`, `feat(correction): capture immutable revision snapshots`, `build(flyway): add V9 revision plans`
- Thread 19 관련 커밋: `fix(correction): persist plans with database time`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionTarget.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionPlan.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionSnapshot.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java`
- 클래스·컴포넌트: `CorrectionRevisionPreparer`
- 관련 메서드: `RevisionPlan.allocate`, `RevisionPlan.deltaAmount`, `RevisionPlan.hasZeroDelta`, `RevisionPlanRepository.persist`
- 관련 Thread: 13, 15, 17, 19

<a id="p19"></a>
<!-- POINT:P19 -->
## P19 — [Thread 14 / `feat(correction): define revision lifecycle`] revision 상태 기계와 DB-level invariant

### 면접 질문

`PENDING`, `BLOCKED`, `EXHAUSTED`, `APPLIED`, `REJECTED` 상태를 어떻게 구분했고, 애플리케이션의 `canTransitionTo`만 두지 않고 데이터베이스 CHECK constraints까지 둔 이유는 무엇입니까?

꼬리 질문:

- `BLOCKED → PENDING`은 허용하면서 `EXHAUSTED → PENDING`을 자동 상태 기계에서 허용하지 않은 이유는 무엇입니까?
- terminal state에 lease나 next retry가 남아 있으면 어떤 문제가 생깁니까?
- 같은 상태로의 전이를 허용하는 것이 멱등성에 어떤 도움을 줍니까?

### 30초 모범 답변

`PENDING`은 worker가 소유할 수 있는 실행 상태, `BLOCKED`는 Wallet queue proof를 보존하며 기다리는 상태, `EXHAUSTED`는 자동 재시도 상한에 도달한 일시 정지, `APPLIED`와 `REJECTED`는 terminal입니다. 애플리케이션 상태 기계는 의도를 읽기 쉽게 만들고, DB constraints는 race·수동 SQL·버그가 lease pair, proof shape, retry schedule, terminal 정리를 깨뜨리지 못하게 합니다. `EXHAUSTED`의 재개는 일반 자동 전이가 아니라 별도 관리자 command와 audit를 통해서만 이뤄집니다.

### 답변 핵심 키워드

explicit state machine, blocked proof, retry exhaustion, terminal states, defense in depth, lease/proof/schedule constraints, operator-controlled resume

### 백지 구현

#### 구현 목표

revision 상태 전이 가능 여부와 row 전체 invariant를 검증하는 순수 validator를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class RevisionInvariantValidator {
  public boolean canTransitionTo(RevisionState from, RevisionState to) {
    // 직접 구현
  }

  public void validate(RevisionRow row) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: source/target state 또는 revision row 전체
- 출력: 전이 가능 여부, invariant 위반 시 명시적 실패

#### 반드시 만족해야 할 조건

- `PENDING`은 허용된 대기·terminal 상태로만 전이한다.
- `BLOCKED`는 재개 가능한 실행 상태 또는 terminal 상태로만 전이한다.
- `EXHAUSTED`, `APPLIED`, `REJECTED`는 일반 자동 전이의 terminal이다.
- lease token과 lease until은 쌍으로 존재한다.
- active execution에 필요한 state만 lease를 가질 수 있다.
- terminal state에는 live lease나 next retry schedule이 남지 않는다.
- `BLOCKED` row는 보존해야 할 Wallet queue proof와 schedule의 의미가 일치해야 한다.
- attempt count와 revision number는 음수가 될 수 없다.
- payout 통화와 delta의 의미가 plan과 일치해야 한다.

#### 경계 조건

- same-state 멱등 전이
- `BLOCKED → PENDING`
- `PENDING → EXHAUSTED`
- zero-delta `PENDING → APPLIED`
- terminal row에 null proof
- lease가 막 만료된 PENDING row

#### 실패 조건

- terminal에서 재실행 상태로 자동 전이
- token만 있고 expiry가 없음
- BLOCKED인데 queue evidence가 없음
- APPLIED인데 retry schedule이 남음
- 음수 attempt count 또는 revision number
- state와 proof status 불일치

#### 필요한 제약

- 상태 전이 규칙과 row shape 검증을 분리해도 된다.
- 상태별 proof 세부 규칙은 제공된 contract policy를 호출하도록 설계할 수 있다.
- 관리자 재시도는 별도의 명시적 command로 모델링한다.

### 구현 후 자가 검증

- 모든 상태 조합을 표로 만들어 허용·거부가 의도와 일치하는가?
- same-state 정책을 의식적으로 결정했는가?
- terminal row에서 lease/schedule 누락 검사가 동작하는가?
- BLOCKED proof와 next retry 의미가 함께 검증되는가?
- 애플리케이션 validator를 우회해도 DB constraint가 필요한 이유를 설명했는가?
- 관리자 재개와 자동 전이를 혼동하지 않는가?

### 구현 후 설명할 것

1. 상태를 예외 코드가 아니라 durable lifecycle로 둔 이유
2. `BLOCKED`와 `EXHAUSTED`의 의미 차이
3. 애플리케이션 상태 기계와 DB constraint의 중복 가치
4. terminal cleanup invariant가 recovery scanner를 단순하게 하는 방식
5. 수동 재개를 별도 audited command로 제한한 이유

### 원본 확인 위치

- Thread: 14 — immutable revision planning과 payout delta
- 대표 커밋: `feat(correction): define revision lifecycle`
- 관련 커밋: `build(flyway): add V9 revision plans`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionState.java`
- 클래스·컴포넌트: `RevisionPlan`, `RevisionPlanRepository`
- 데이터베이스 위치: `settlement_revision`, `settlement_revision_selection` 테이블의 state·lease·proof·schedule constraints
- 관련 메서드: `RevisionState.canTransitionTo`, `RevisionState.requireTransitionTo`
- 관련 Thread: 15, 16, 19, 21
