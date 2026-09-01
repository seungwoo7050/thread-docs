# correction proof·모호한 응답 복구·원자적 revision 확정 면접 워크북

이 문서는 이미 지급된 금액을 수정하는 correction 경계에 집중한다. Wallet response를 business truth로 확정하는 exact proof, 응답 손실 후 GET-first 복구, revision과 revised outbox event의 원자적 확정을 다룬다.

<a id="p20"></a>
<!-- POINT:P20 -->
## P20 — [Thread 15 / `feat(correction): validate wallet adjustment proofs`] adjustment proof의 identity·금액 대수 검증

### 면접 질문

Wallet adjustment proof를 받을 때 status만 확인하지 않고 revision ID, bet ID, revision number, user ID, previous/new payout, delta, currency를 모두 비교한 이유는 무엇입니까? proof가 2xx body로 왔지만 필드 하나가 다르면 어떻게 처리해야 합니까?

꼬리 질문:

- `delta == new - previous`를 Wallet 값만 믿지 않고 다시 계산해야 하는 이유는 무엇입니까?
- negative delta도 `BLOCKED`가 될 수 있다면 validator는 무엇을 가정하면 안 됩니까?
- zero-delta correction에 proof가 전달되면 왜 거부해야 합니까?

### 30초 모범 답변

proof는 단순 응답 DTO가 아니라 이 revision의 금전 효과를 확정할 권한 증거입니다. 그래서 stable revision identity와 대상 bet/user/revision number, previous/new amount·currency, 그리고 overflow-safe `new - previous`가 모두 plan과 정확히 같아야 합니다. status별 queue 또는 applied evidence shape도 함께 검증합니다. 2xx라도 mismatch나 누락이 있으면 semantic rejection을 만들어내지 않고 malformed success로 취급해 복구 경로로 보냅니다. zero delta는 Wallet 작업 자체가 없어야 하므로 proof도 허용하지 않습니다.

### 답변 핵심 키워드

exact operation proof, identity binding, payout algebra, currency preservation, status-dependent shape, malformed success, zero-delta no-op

### 백지 구현

#### 구현 목표

revision plan과 Wallet adjustment proof를 비교해 exact proof만 반환하는 validator를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class AdjustmentProofValidator {
  public ValidatedAdjustmentProof requireExact(
      RevisionPlan plan,
      WalletAdjustmentProof proof,
      ProofShapePolicy shapePolicy) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: immutable revision plan, Wallet proof, status별 필수·금지 필드 정책
- 출력: 모든 검증을 통과한 proof wrapper

#### 반드시 만족해야 할 조건

- revision ID가 정확히 일치한다.
- bet ID와 user ID가 정확히 일치한다.
- revision number가 정확히 일치한다.
- previous payout과 new payout의 amount·currency가 정확히 일치한다.
- proof delta와 `new - previous`가 모두 plan delta와 일치한다.
- delta 계산은 overflow를 감지한다.
- proof status는 허용된 값이어야 한다.
- `APPLIED`, `BLOCKED`, `REJECTED` 각각의 필수·금지 증거 shape는 전달된 policy를 만족해야 한다.
- `BLOCKED`의 queue sequence는 양수이고 queue 시각 정보가 일관돼야 한다.
- zero-delta plan에는 proof가 필요하지 않으며 이 validator 호출 자체를 허용할지 명시한다.

#### 경계 조건

- positive, negative, zero delta
- previous payout이 0
- amount는 같지만 currency가 다름
- ID 하나만 다름
- status는 맞지만 status별 evidence가 누락됨
- 최대 범위에 가까운 minor-unit 값

#### 실패 조건

- identity mismatch
- payout 또는 delta mismatch
- overflow
- 허용하지 않은 status
- status와 evidence shape 불일치
- null proof
- zero-delta 경로에서 외부 proof 사용

#### 필요한 제약

- 검증 실패를 Wallet의 authoritative business rejection으로 바꾸지 않는다.
- 실패 메시지에 credential이나 전체 dependency body를 포함하지 않는다.
- status별 shape는 switch 분기 또는 별도 policy로 읽기 쉽게 유지한다.

### 구현 후 자가 검증

- 비교 대상 각 필드를 하나씩 바꿔 모두 거부되는가?
- negative delta도 대수 검증이 정확한가?
- overflow가 조용히 wrap되지 않는가?
- BLOCKED queue sequence 0 또는 음수를 거부하는가?
- status별 필수·금지 필드가 모두 검증되는가?
- malformed proof를 `REJECTED`로 오해하지 않는가?
- zero-delta가 Wallet proof 없이 처리되는가?

### 구현 후 설명할 것

1. proof를 business authorization evidence로 본 이유
2. identity와 금액 대수를 모두 비교한 이유
3. malformed success와 authoritative rejection의 차이
4. negative delta에서 status를 단순 추론하면 안 되는 이유
5. status별 evidence shape를 별도 policy로 분리한 trade-off

### 원본 확인 위치

- Thread: 15 — 내구성 있는 revision 실행, proof 복구, 원자적 확정
- 대표 커밋: `feat(correction): validate wallet adjustment proofs`
- 관련 커밋: `feat(correction): submit wallet adjustments`, `feat(correction): persist blocked adjustments`, `feat(correction): persist rejected adjustment proofs`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionProofValidator.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletAdjustmentProof.java`
- 관련 메서드: `RevisionProofValidator.requireExact`, `RevisionWalletGateway.submit`
- 관련 Thread: 8, 14, 19

<a id="p21"></a>
<!-- POINT:P21 -->
## P21 — [Thread 15 / `feat(correction): recover ambiguous adjustments`] GET-first 복구와 BLOCKED proof 보존

### 면접 질문

Wallet adjustment POST가 timeout되거나 응답을 잃은 뒤 recovery에서 왜 같은 POST를 즉시 다시 보내지 않고 먼저 GET을 호출했습니까? 어떤 경우에만 POST 재전송을 허용했습니까?

꼬리 질문:

- GET이 404여도 모든 404에서 POST해도 됩니까?
- durable `BLOCKED` proof가 있는 plan을 다시 POST하면 어떤 문제가 생깁니까?
- 자동 시도 상한에 도달한 no-proof plan과 BLOCKED plan을 같은 `EXHAUSTED` 상태로 만들지 않은 이유는 무엇입니까?

### 30초 모범 답변

POST timeout은 실패가 아니라 "효과는 적용됐지만 응답만 잃음"일 수 있으므로 blind retry를 하면 중복 금전 작업 위험이 있습니다. recovery는 persisted revision ID로 GET을 먼저 수행하고 exact proof가 있으면 검증 후 이어갑니다. Wallet이 정확히 adjustment-not-found를 뜻하는 404와 error code를 준 경우에만 같은 revision ID로 POST를 다시 보냅니다. `BLOCKED` proof는 queue identity와 schedule을 보존해 조회·대기해야 하며 재POST하지 않습니다. 자동 시도 상한 뒤 no-proof는 `EXHAUSTED`, proof가 있는 BLOCKED는 proof를 유지한 채 정지합니다.

### 답변 핵심 키워드

ambiguous outcome, GET-before-POST, exact not-found, stable idempotency identity, blocked queue preservation, capped attempts, proof-aware exhaustion

### 백지 구현

#### 구현 목표

현재 revision row와 Wallet lookup 결과를 받아 다음 복구 행동을 결정하는 순수 state reducer를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class RevisionRecoveryPolicy {
  public RecoveryAction next(
      RevisionRecoveryState state,
      WalletLookupOutcome lookup,
      int maxAutomaticAttempts) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: plan state·attempt count·durable proof 유무·next retry, GET lookup outcome
- 출력: `FINALIZE`, `WAIT_BLOCKED`, `SUBMIT_SAME_REVISION`, `RETRY_LATER`, `EXHAUST`, `REJECT` 중 하나

#### 반드시 만족해야 할 조건

- 복구 판단은 항상 GET lookup 결과를 먼저 해석한다.
- exact `APPLIED` proof면 finalization으로 간다.
- exact `BLOCKED` proof면 queue identity를 보존하고 재POST하지 않는다.
- authoritative `REJECTED` proof만 semantic reject로 간다.
- 특정 adjustment-not-found 404와 error code 조합에서만 동일 revision ID의 POST를 허용한다.
- timeout, malformed lookup response, 예상 밖 status는 즉시 POST 허용 근거가 아니다.
- durable BLOCKED proof가 있는 row는 missing lookup 때문에 proof를 지우거나 새 operation을 만들지 않는다.
- no-proof plan이 최대 자동 시도에 도달하면 exhaust한다.
- max attempt를 넘겨 자동 루프를 계속하지 않는다.
- plan의 amount, currency, target identity를 변경하지 않는다.

#### 경계 조건

- attempt count가 상한 바로 전 또는 정확히 상한
- durable BLOCKED proof와 아직 오지 않은 nextAttemptAt
- GET exact APPLIED
- 정확한 not-found 404
- 다른 404 error code
- malformed 2xx lookup proof

#### 실패 조건

- GET 없이 POST 재전송
- 일반 404를 not-found로 확대 해석
- BLOCKED proof를 덮어씀
- max attempts 이후 자동 재시도
- timeout을 REJECTED로 바꿈
- 새 revision ID나 새로운 금액으로 재전송

#### 필요한 제약

- reducer는 네트워크를 호출하지 않는다.
- 실제 runner가 action에 따라 하나의 외부 호출만 수행하도록 설계한다.
- backoff 시각 계산은 database time 기반 durable repository에 둔다.

### 구현 후 자가 검증

- exact APPLIED lookup이 POST 없이 finalize로 가는가?
- BLOCKED proof에서 `SUBMIT`이 절대 나오지 않는가?
- 정확한 not-found 조합에서만 same-ID submit이 나오는가?
- malformed/timeout lookup이 retryable로 남는가?
- attempt 상한에서 no-proof가 exhaust되는가?
- paused BLOCKED 상태가 queue evidence를 그대로 유지하는가?
- 모든 action이 immutable plan을 전제로 하는가?

### 구현 후 설명할 것

1. timeout 뒤 blind retry가 위험한 이유
2. GET-first가 idempotency와 결합되는 방식
3. 특정 404 계약을 좁게 해석한 이유
4. BLOCKED와 EXHAUSTED를 proof 유무에 따라 구분한 이유
5. 자동 retry 상한과 관리자 재개의 책임 분리

### 원본 확인 위치

- Thread: 15 — 내구성 있는 revision 실행, proof 복구, 원자적 확정
- 대표 커밋: `feat(correction): recover ambiguous adjustments`
- 관련 커밋: `feat(correction): execute durable revision plans`, `feat(recovery): protect blocked wallet proofs`, `feat(recovery): scan durable revision claims`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java`
- 관련 메서드: `RevisionWalletGateway.recoverAmbiguous`, `RevisionExecutionRunner.execute`, `RevisionRecoveryRepository.claimDue`
- 관련 Thread: 8, 14, 16, 19, 20

<a id="p22"></a>
<!-- POINT:P22 -->
## P22 — [Thread 17 / `feat(correction): finalize revisions atomically`] stale target를 막는 revision·bet·outbox 원자적 확정

### 면접 질문

Wallet에서 exact `APPLIED` proof를 받은 뒤에도 `RevisionFinalizer`가 bet을 다시 잠그고 target snapshot, current revision, candidate provenance, lease를 검증한 이유는 무엇입니까? 어떤 변경들이 한 transaction이어야 합니까?

꼬리 질문:

- Wallet은 이미 적용됐는데 DB finalization이 stale target 때문에 실패하면 어떻게 복구합니까?
- zero-delta revision은 proof가 없는데도 같은 finalization 경계를 어떻게 사용합니까?
- source result의 `settledAt`이 DB 현재보다 미래면 왜 적용하면 안 됩니까?

### 30초 모범 답변

Wallet proof는 금전 효과의 증거지만 bet이 아직 그 plan의 predecessor 상태라는 보장은 아닙니다. finalizer는 bet row를 잠그고 `SETTLED` 상태, expected revision number와 source candidate, 이전 result/payout·selection snapshot을 다시 확인합니다. nonzero delta는 exact `APPLIED` proof, zero delta는 null proof만 허용합니다. 그 뒤 bet result/payout/revision/provenance 변경, revision `APPLIED` 전이와 lease 소비, revised outbox insert를 한 DB transaction으로 커밋하고 DB time으로 시각을 확정합니다.

### 답변 핵심 키워드

proof is necessary not sufficient, stale-target revalidation, row lock, exact predecessor, zero-delta path, owner-fenced lease, atomic revised outbox

### 백지 구현

#### 구현 목표

transactional repository abstraction 위에서 revision finalization의 모든 fence를 검증하고, bet·revision·outbox를 원자적으로 변경하는 서비스를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface RevisionFinalizationService {
  // 직접 구현
  boolean finalizeRevision(FinalizeRevisionCommand command);
}
```

#### 입력과 출력

- 입력: persisted plan ID, exact lease token, 선택적 Wallet proof
- 출력: 모든 fence를 통과해 적용했으면 true, stale/lost ownership이면 false

#### 반드시 만족해야 할 조건

- persisted plan을 ID로 다시 읽는다.
- 대상 bet을 write lock으로 읽는다.
- bet 상태는 `SETTLED`여야 한다.
- current revision number와 source candidate가 plan predecessor와 일치해야 한다.
- previous result/payout과 필요한 selection snapshot이 plan target과 일치해야 한다.
- nonzero delta에는 exact `APPLIED` proof가 필요하다.
- zero delta에는 proof가 없어야 하고 Wallet 상태를 새로 만들지 않는다.
- lease token이 exact owner이고 DB time 기준으로 만료되지 않아야 한다.
- source result settled 시각은 DB 현재보다 미래일 수 없다.
- bet의 새 result/payout/revision/provenance, revision state·proof·appliedAt, revised outbox event가 한 transaction이다.
- fence 실패 시 부분 변경이 없어야 한다.

#### 경계 조건

- zero delta
- positive/negative delta
- lease 만료 경계
- 현재 bet revision이 plan보다 이미 앞선 경우
- proof는 exact하지만 target이 stale인 경우
- outbox 저장 직전 실패

#### 실패 조건

- proof만 믿고 stale bet에 적용
- bet만 변경되고 revision state/outbox가 누락
- revision만 APPLIED이고 bet이 이전 상태
- zero delta에서 Wallet proof 요구 또는 외부 호출
- caller clock으로 appliedAt 결정
- expired/stale token으로 적용

#### 필요한 제약

- method 전체는 하나의 DB transaction으로 실행된다고 가정한다.
- Wallet 호출은 finalizer transaction 밖에서 끝난 상태여야 한다.
- false와 invalid proof 예외를 구분한다. false는 stale/lost ownership, invalid proof는 계약 위반이다.

### 구현 후 자가 검증

- exact predecessor일 때만 모든 상태가 함께 변경되는가?
- stale revision·candidate·payout 각각이 부분 변경 없이 실패하는가?
- zero-delta가 null proof로 정상 적용되는가?
- nonzero delta에서 proof mismatch가 거부되는가?
- lease token과 DB-time expiry가 모두 검사되는가?
- outbox insert 실패가 bet/revision 변경을 rollback시키는가?
- appliedAt과 event payload가 같은 durable decision을 반영하는가?

### 구현 후 설명할 것

1. Wallet proof만으로 finalization할 수 없는 이유
2. target snapshot 재검증이 막는 stale correction
3. zero-delta와 nonzero-delta를 같은 transaction 경계에 둔 방식
4. bet·revision·outbox 원자성이 필요한 이유
5. Wallet 적용 후 DB fence 실패를 recovery가 다시 다루는 방식

### 원본 확인 위치

- 대표 Thread: 17 — correction settlement pipeline
- 대표 커밋: `feat(correction): finalize revisions atomically`
- Thread 15 관련 커밋: `feat(correction): execute durable revision plans`
- Thread 19 관련 커밋: `feat(correction): fence finalization with database time`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionFinalizer.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/persistence/BetRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/OutboxEventRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java`
- 관련 메서드: `RevisionFinalizer.apply`, `RevisionPlanRepository.markApplied`, `BetRepository.findForUpdateById`
- 관련 Thread: 11, 13, 14, 15, 19
