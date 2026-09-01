# 조정과 복구 상태 머신

결제 결과가 뒤늦게 바뀌면 한 번의 즉시 이체로 끝나지 않을 수 있다. 이 문서는 최초 조정 요청을 영속 proof로 판정하는 단계와, 부족 자금으로 막힌 음수 조정을 FIFO로 회수하는 단계를 분리해 연습한다.

<a id="w06"></a>
## W06 — [Thread 10 / `feat(adjustment): choose locked correction outcomes`] 조정 최초 판정과 revision 선점

### 면접 질문

정산 시스템이 같은 베팅의 지급액을 `previousPayout`에서 `newPayout`으로 수정할 때, 최초 요청을 `APPLIED`, `BLOCKED`, `REJECTED` 중 하나로 결정하는 순서를 설명해 보세요. 왜 idempotency key 잠금 외에 `(betId, revisionNumber)` 잠금이 또 필요합니까?

꼬리 질문:

- 서로 다른 revision ID와 idempotency key로 같은 `(betId, revisionNumber)`가 동시에 들어오면 어떻게 해야 합니까?
- 음수 delta가 현재 잔액으로 가능하더라도 기존 blocked head가 있으면 왜 즉시 적용하지 않습니까?
- `BLOCKED` proof를 만들 때 잔액과 ledger를 변경하면 안 되는 이유는 무엇입니까?
- 계정 없음·통화 불일치는 왜 단순 예외가 아니라 영속 `REJECTED` proof가 될 수 있습니까?
- `new - previous`와 `abs(delta)`를 계산할 때 어떤 overflow 경계를 확인해야 합니까?

### 30초 모범 답변

먼저 요청 자체와 canonical idempotency key를 검증하고, idempotency key 경쟁과 별도로 `(betId, revisionNumber)`를 직렬화해 같은 업무 revision을 하나만 소유하게 합니다. 계정과 가장 오래된 blocked head를 함께 잠근 뒤 양수 delta는 즉시 credit하고, 음수 delta는 선행 head가 없고 available이 충분할 때만 즉시 회수합니다. 그렇지 않으면 잔액·ledger 없이 recovery debt와 queue sequence를 가진 `BLOCKED` proof를 저장합니다. 계정 없음이나 통화 불일치 같은 확정 업무 실패는 `REJECTED` facts로 남기며, 모든 최초 판정은 operation outcome과 proof가 일치하게 한 트랜잭션에서 끝나야 합니다.

### 답변 핵심 키워드

`revision identity` · `별도 advisory namespace` · `account row lock` · `FIFO head` · `positive apply` · `negative apply-or-block` · `durable proof` · `no partial ledger`

### 백지 구현

#### 구현 목표

검증된 최초 조정 요청과 잠긴 현재 상태를 받아 `APPLIED_INCREASE`, `APPLIED_DECREASE`, `BLOCKED`, `REJECTED`, `CONFLICT` 중 하나의 효과 계획을 만든다. 실제 DB 저장은 문제 범위 밖이며, 계획은 서로 모순되는 효과를 동시에 요구해서는 안 된다.

#### 면접용 축소 인터페이스

```java
enum AdjustmentStatus { APPLIED, BLOCKED, REJECTED }

enum EffectKind {
  APPLY_INCREASE,
  APPLY_DECREASE,
  QUEUE_RECOVERY,
  RECORD_REJECTION
}

record AdjustmentRequest(
    UUID revisionId,
    UUID betId,
    long revisionNumber,
    UUID userId,
    long previousPayout,
    long newPayout,
    Currency currency,
    String idempotencyKey) {}

record LockedAdjustmentContext(
    boolean accountExists,
    Currency accountCurrency,
    long available,
    long locked,
    boolean blockedHeadExists,
    boolean betRevisionAlreadyClaimed) {}

record AdjustmentPlan(
    AdjustmentStatus status,
    EffectKind effect,
    long absoluteDelta,
    String rejectionCode) {}

final class AdjustmentDecider {
  static AdjustmentPlan decide(
      AdjustmentRequest request,
      LockedAdjustmentContext context) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 지급 전·후 snapshot과 업무 identity, 잠긴 계정·queue 상태
- 출력: 최초 writer가 한 트랜잭션에서 수행해야 할 단 하나의 효과 계획
- 충돌: 같은 `(betId, revisionNumber)`가 이미 다른 요청에 의해 소유된 경우

#### 반드시 만족해야 할 조건

요청 검증:

- `revisionNumber >= 1`
- 이전·새 지급액은 0 이상이고 통화가 같다.
- delta는 0이 아니어야 한다.
- `newPayout - previousPayout`과 절댓값 계산이 overflow하지 않아야 한다.
- idempotency key는 `settlement:revision:{revisionId}`와 정확히 일치해야 한다.
- 시스템 상대 계정 ID를 사용자로 사용할 수 없다.

판정 규칙:

- `(betId, revisionNumber)`가 이미 소유되었다면 balance·proof·ledger 계획 없이 conflict다.
- 계정 없음 또는 통화 불일치는 영속 `REJECTED` 계획이다.
- 양수 delta는 계정 총액 표현 범위를 넘지 않을 때 `APPLIED` 증가 계획이다.
- 음수 delta는 blocked head가 없고 available이 충분할 때만 `APPLIED` 감소 계획이다.
- 음수 delta에 선행 blocked head가 있거나 available이 부족하면 `BLOCKED` recovery queue 계획이다.
- `BLOCKED` 계획은 즉시 balance 감소나 ledger 쓰기를 요구하지 않는다.
- 각 결과는 operation status와 adjustment proof status가 서로 대응해야 한다.

#### 경계 조건

- `previous=0`, `new=1`
- `previous=1`, `new=0`
- delta가 정확히 available과 같은 음수 조정
- available은 충분하지만 blocked head가 있는 음수 조정
- `new == previous`
- 서로 다른 revision ID로 같은 bet/revision을 선점한 요청
- 양수 반영 뒤 계정 총액이 정확히 `Long.MAX_VALUE`
- subtraction 또는 absolute value가 표현 범위를 벗어나는 입력

#### 실패 조건

- 요청 구조 자체가 잘못된 경우
- 업무 revision conflict
- 계정 없음·통화 불일치·잔액 표현 범위 초과 같은 영속 업무 거절
- 잠긴 account와 queue head의 recovery 상태가 서로 모순되는 경우

#### 필요한 제약

- 이 함수는 하나의 최초 판정만 만들며 recovery worker 동작을 수행하지 않는다.
- conflict와 rejected를 구분한다.
- `BLOCKED`를 성공 원장 행이 있는 상태로 표현하지 않는다.
- 시간·공간 복잡도는 O(1)이다.

### 구현 후 자가 검증

- [ ] 양수 delta가 `APPLIED/INCREASE`가 된다.
- [ ] 음수 delta가 잔액 충분·head 없음일 때만 `APPLIED/DECREASE`가 된다.
- [ ] 잔액이 충분해도 head가 있으면 `BLOCKED`가 된다.
- [ ] `BLOCKED` 계획에 즉시 ledger 또는 balance 감소 효과가 없다.
- [ ] 계정 없음과 통화 불일치가 동일한 conflict로 섞이지 않고 `REJECTED`가 된다.
- [ ] 같은 bet/revision 중복은 idempotency key가 달라도 conflict다.
- [ ] 0 delta, 잘못된 canonical key, 잘못된 revision number를 거부한다.
- [ ] delta와 absolute delta 계산에서 overflow를 놓치지 않는다.

### 구현 후 설명할 것

1. idempotency key lock과 `(betId, revisionNumber)` lock을 별도 namespace로 둔 이유.
2. account와 FIFO head를 함께 잠근 snapshot에서 판정해야 하는 이유.
3. `BLOCKED`를 실패 예외가 아니라 조회 가능한 durable proof로 만든 이유.
4. 업무 거절과 conflict의 HTTP·재시도 의미 차이.
5. 판정 함수와 실제 효과 writer를 분리했을 때 얻는 테스트 가능성과 주의점.

### 원본 확인 위치

- Thread: **10 — 조정 증명과 최초 판정**
- 커밋: `feat(command): validate correction payout snapshots`
- 커밋: `feat(adjustment): serialize bet revision claims`
- 커밋: `feat(adjustment): apply positive corrections atomically`
- 커밋: `feat(adjustment): queue blocked negative corrections`
- 커밋: `feat(adjustment): persist rejected correction proofs`
- 커밋: `feat(adjustment): choose locked correction outcomes`
- 파일: `src/main/java/com/sportsbook/wallet/service/command/AdjustmentCommand.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/AdjustmentPairLock.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/WalletAdjustmentRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/AdjustmentFirstWriter.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletAdjustmentService.java`
- 클래스·메서드: `AdjustmentCommand.deltaAmount()`, `AdjustmentCommand.absoluteDelta()`, `AdjustmentPairLock.acquire()`, `AdjustmentFirstWriter.write()`
- 관련 Thread: 4, 6, 7, 11, 16

---

<a id="w07"></a>
## W07 — [Thread 11 / `feat(recovery): claim one FIFO head transactionally`] 복구 부채 FIFO 회수와 bounded retry

### 면접 질문

부족 자금 때문에 `BLOCKED`가 된 음수 조정을 자동 회수할 때, 두 worker가 같은 계정을 동시에 처리해도 중복 회수가 일어나지 않고 FIFO가 유지되게 하려면 무엇을 잠그고 어떤 상태만 바꿔야 합니까?

꼬리 질문:

- recovery debt가 있는 동안 outbound를 막되 입금·환불·몰수 같은 동작을 허용하는 이유는 무엇입니까?
- 자금이 여전히 부족한 retry에서 account version·ledger·operation status까지 바꾸면 어떤 문제가 생깁니까?
- 충분한 자금으로 head를 완료한 뒤 다음 head를 깨우는 순서는 왜 같은 트랜잭션에 있어야 합니까?
- 두 worker 중 하나가 `APPLIED`를 완료한 뒤 다른 worker는 어떤 결과를 관찰해야 합니까?
- `base * 2^retryCount` 계산을 그대로 하면 어떤 overflow가 생기며 어떻게 cap에 포화시킵니까?

### 30초 모범 답변

worker는 due 상태인 계정의 가장 오래된 blocked proof 하나만 선택하고 proof와 account를 같은 트랜잭션에서 잠급니다. available이 부족하면 retry count와 `nextAttemptAt`만 갱신하고 잔액·ledger·operation은 그대로 둡니다. 충분하면 available에서 회수하고 matched adjustment ledger를 기록한 뒤 proof를 `APPLIED`, blocked operation을 `SUCCEEDED`로 완료하고 recovery debt를 줄입니다. debt가 0이면 freeze를 해제하고, 다음 head가 있으면 그 head만 깨웁니다. backoff는 overflow 전에 cap에 포화시켜야 하며, worker 한 번은 head 하나만 처리해야 lock 보유 시간을 제한할 수 있습니다.

### 답변 핵심 키워드

`oldest due head` · `proof + account lock` · `one head per transaction` · `insufficient = metadata only` · `debt/freeze invariant` · `wake next` · `saturating backoff` · `two-worker exclusion`

### 백지 구현

#### 구현 목표

저장소가 행 잠금과 트랜잭션 롤백을 제공한다고 가정하고, due recovery head를 최대 하나 처리하는 worker와 포화형 retry delay를 작성한다.

#### 면접용 축소 인터페이스

```java
enum RecoveryResult { NO_WORK, DEFERRED_FUNDS, APPLIED }

record RecoveryHead(
    UUID revisionId,
    UUID userId,
    long queueSequence,
    long amount,
    int retryCount,
    Instant nextAttemptAt) {}

interface RecoveryStore {
  Optional<RecoveryHead> claimOldestDueHead(Instant now);
  Account lockAccount(UUID userId);
  void defer(RecoveryHead head, int nextRetryCount, Instant nextAttemptAt);
  void insertRecoveryLedger(RecoveryHead head, UUID operationGroupId, Instant now);
  void completeProof(RecoveryHead head, UUID operationGroupId, Instant now);
  void completeBlockedOperation(RecoveryHead head, UUID operationGroupId, Instant now);
  void updateAccountAfterRecovery(Account account, long amount, Instant now);
  void wakeNextBlockedHead(UUID userId, Instant now);
}

final class RecoveryProcessor {
  RecoveryResult recoverOne(Instant now) {
    // 직접 구현
  }

  static Duration retryDelay(int retryCount, Duration base, Duration cap) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: DB 기준 현재 시각, due blocked proof, 잠긴 계정, retry 설정
- 출력: 작업 없음, 자금 부족으로 연기, 회수 완료 중 하나
- 실패: 영속 상태 모순이나 인프라 오류는 트랜잭션 롤백

#### 반드시 만족해야 할 조건

선점과 FIFO:

- 한 호출은 due head를 최대 하나만 처리한다.
- 계정별 가장 작은 `queueSequence`인 blocked proof만 대상이다.
- 동일 proof를 두 worker가 동시에 완료할 수 없다.
- head를 얻지 못하면 `NO_WORK`이며 다른 상태를 쓰지 않는다.

자금 부족:

- `available < amount`이면 account balance, recovery debt, ledger, proof status, operation status를 바꾸지 않는다.
- retry count와 다음 시각만 갱신한다.
- 다음 시각은 DB 기준 현재 시각에 bounded delay를 더한 값이다.

회수 성공:

- `available >= amount`일 때만 회수한다.
- available과 recovery debt를 같은 금액만큼 줄인다.
- 사용자에서 `HOUSE`로 향하는 `BET_ADJUSTMENT` matched ledger를 쓴다.
- proof는 `APPLIED`, 대응 blocked operation은 `SUCCEEDED`가 된다.
- debt가 0이면 outbound freeze가 해제되어야 한다.
- 다음 blocked head가 있으면 그 head만 due 상태로 깨운다.
- 위 효과가 모두 같은 트랜잭션에서 커밋된다.

backoff:

- `base > 0`, `cap >= base`, `retryCount >= 0`
- 지수 증가가 overflow하기 전에 cap에 포화된다.
- 반환값은 항상 `[base, cap]` 범위다.

#### 경계 조건

- due head가 없음
- available이 amount보다 1 작음
- available이 amount와 정확히 같음
- 회수 뒤 debt가 정확히 0
- 같은 계정에 두 번째 head가 있음
- 다른 계정의 due head가 동시에 있음
- retry count가 매우 커서 shift·multiplication이 overflow할 수 있음
- 두 worker가 같은 head를 동시에 시도
- 첫 worker가 성공한 직후 두 번째 worker가 관찰하는 상태

#### 실패 조건

- account의 recovery debt와 blocked queue가 서로 맞지 않음
- head 금액이 비양수
- queue sequence가 비양수
- proof와 blocked operation의 identity가 맞지 않음
- ledger 저장·proof 완료·operation 완료·account update 중 인프라 오류
- 잘못된 retry 설정

#### 필요한 제약

- 긴 반복문으로 한 트랜잭션에서 계정의 모든 head를 처리하지 않는다.
- 애플리케이션 시계가 아니라 저장소 선점에 사용된 DB 기준 시각을 사용한다.
- 자금 부족 retry는 무효과 경로여야 한다.
- 정상 경로의 처리량은 한 head당 O(1)이며 queue 탐색은 인덱스가 지원한다고 가정한다.

### 구현 후 자가 검증

- [ ] due head가 없으면 어떤 write도 없다.
- [ ] 자금 부족 시 retry metadata 외 필드가 바뀌지 않는다.
- [ ] 정확한 잔액에서 한 번만 회수되고 available이 0이 된다.
- [ ] ledger, proof, operation, account 중 하나의 저장을 실패시키면 전체가 롤백된다.
- [ ] 두 worker가 경쟁해도 `APPLIED` 완료는 하나뿐이다.
- [ ] queue sequence가 더 큰 head가 먼저 처리되지 않는다.
- [ ] debt가 0이면 freeze가 해제되고, debt가 남으면 유지된다.
- [ ] 다음 head만 깨우며 뒤의 여러 head를 한꺼번에 due로 만들지 않는다.
- [ ] 큰 retry count에서도 음수 delay나 overflow가 나오지 않고 cap에 머문다.

### 구현 후 설명할 것

1. 한 트랜잭션에서 head 하나만 처리해 lock hold time을 제한한 이유.
2. 자금 부족을 실패 exception이 아니라 retry metadata 전이로 처리한 이유.
3. `queueSequence`가 timestamp보다 FIFO 권위로 적합한 이유.
4. freeze가 모든 동작을 막지 않고 outbound만 막는 이유.
5. recovery worker와 최초 adjustment writer가 공유하는 invariant와 서로 다른 책임.

### 원본 확인 위치

- Thread: **11 — 복구 부채와 FIFO 회수**
- 커밋: `test(account): allow inflows and forfeit while recovery frozen`
- 커밋: `feat(recovery): wake the oldest blocked adjustment`
- 커밋: `feat(recovery): define full debt collection transfers`
- 커밋: `feat(recovery): bound automatic retry delays`
- 커밋: `feat(recovery): claim one FIFO head transactionally`
- 커밋: `feat(recovery): schedule automatic collection`
- 파일: `src/main/java/com/sportsbook/wallet/domain/Account.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/RecoveryWakeService.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/RecoveryRetryPolicy.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/RecoveryHeadProcessor.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/RecoveryWorker.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/RecoveryScheduler.java`
- 클래스·메서드: `RecoveryWorker.recoverOne()`, `RecoveryHeadProcessor.process()`, `RecoveryRetryPolicy`, `RecoveryWakeService.wake()`
- 관련 Thread: 4, 7, 10, 13, 14
