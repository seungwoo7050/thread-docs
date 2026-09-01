# 사가 체크포인트, 복구 lease, 집계 잠금 경계

이 문서는 여러 원격 부작용을 하나의 로컬 트랜잭션처럼 보이게 만들 수 없을 때 상태를 어떻게 남기고 복구하는지, 여러 worker가 복구 작업을 공정하고 안전하게 나누는지, JPA 집계 전이를 어떤 잠금 경계에서 직렬화하는지를 다룬다.

<a id="t08-saga-checkpoints"></a>
## [Thread 08 / `feat(placement): checkpoint external side effects`] 재시작 가능한 사가와 비가역 보상 상태 머신

### 면접 질문

리스크 reserve, 지갑 debit, 리스크 commit, bet acceptance가 서로 다른 시스템에 걸쳐 있을 때 왜 하나의 긴 메서드와 try/catch로 충분하지 않습니까? 각 외부 side effect 뒤에 durable checkpoint를 두고 보상 action까지 상태로 저장한 이유를 설명해 보세요.

꼬리 질문:

- `CREATED → RISK_RESERVED → WALLET_CONFIRMED → RISK_COMMITTED` 순서가 바뀌면 어떤 문제가 생깁니까?
- 지갑 debit 이후 리스크 commit이 확정 실패하면 어떤 보상이 필요합니까?
- reserve 이후 debit이 확정 실패하면 어떤 보상이 필요합니까?
- 보상 상태를 `REQUIRED → IN_PROGRESS → COMPLETED`로 나누는 이유는 무엇입니까?
- 같은 checkpoint 저장 메서드가 여러 번 호출되어도 상태가 후퇴하지 않아야 하는 이유는 무엇입니까?
- acceptance와 outbox 저장을 같은 DB transaction에 둔 이유는 무엇입니까?
- recovery loop에 최대 step 수를 두는 이유는 무엇입니까?

### 30초 모범 답변

분산 호출 사이에는 process crash와 응답 유실이 생기므로 메모리의 실행 흐름은 복구 근거가 될 수 없습니다. 각 확정된 side effect 직후 로컬 DB에 checkpoint와 operation proof를 저장하고, 재실행은 현재 상태에서 다음 한 단계만 수행합니다. 실패가 발생하면 이미 일어난 비가역 효과에 따라 `RISK_RELEASE`나 `WALLET_REFUND`를 선택해 durable compensation 상태로 전환합니다. 모든 전이는 멱등적이고 단조로워야 하며, 최종 acceptance와 outbox는 한 트랜잭션으로 묶어 승인된 bet만 이벤트를 남기게 합니다.

### 답변 핵심 키워드

`saga`, `durable checkpoint`, `monotonic transition`, `at-least-once replay`, `operation proof`, `compensation action`, `terminal state`, `atomic local transaction`, `bounded progress loop`

### 백지 구현

**구현 목표**

외부 호출 자체는 하지 않고, 현재 placement/compensation 상태와 직전 관측 결과를 받아 다음 durable 전이 또는 다음 실행 action을 결정하는 순수 상태 머신을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
enum Phase { CREATED, RISK_RESERVED, WALLET_CONFIRMED, RISK_COMMITTED, ACCEPTED, REJECTED }
enum CompensationState { NONE, REQUIRED, IN_PROGRESS, COMPLETED }
enum CompensationAction { RISK_RELEASE, WALLET_REFUND }
enum NextAction { RESERVE_RISK, DEBIT_WALLET, COMMIT_RISK, ACCEPT, RELEASE_RISK, REFUND_WALLET, FINISH_REJECTION, NONE }

record SagaState(
    Phase phase,
    CompensationState compensationState,
    CompensationAction compensationAction) {}

final class PlacementMachine {
  NextAction next(SagaState state) {
    // 직접 구현
    return null;
  }

  SagaState checkpoint(SagaState state, NextAction completedAction) {
    // 직접 구현
    return null;
  }

  SagaState requireCompensation(SagaState state, String definitiveFailure) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: 현재 durable 상태, 완료된 action 또는 확정 실패
- 출력: 수행해야 할 다음 action이나 새 durable 상태

**반드시 만족해야 할 조건**

- placement phase는 뒤로 가지 않는다.
- terminal 상태에서는 외부 action을 제안하지 않는다.
- 이미 완료된 action을 다시 checkpoint해도 상태가 더 변하지 않는다.
- 보상 action은 이미 발생한 side effect에 따라 한 번 결정되면 바뀌지 않는다.
- `IN_PROGRESS` 없이 완료되도록 허용할지 계약을 정하고 일관되게 지킨다.
- 보상 완료 전에는 terminal rejection으로 가지 않는다.
- 모순된 state 조합을 생성하거나 수용하지 않는다.

**경계 조건**

- 각 phase에서 재시작
- 같은 action 완료 통지를 두 번 받음
- CREATED에서 definitive validation failure
- RISK_RESERVED에서 wallet rejection
- WALLET_CONFIRMED에서 risk commit conflict
- compensation 중 process 재시작
- 이미 ACCEPTED 또는 REJECTED

**실패 조건**

불가능한 전이, 누락된 compensation action, terminal 상태 변경 시도를 명시적으로 거부한다. 오류 때문에 상태를 초기화하거나 이전 checkpoint로 되돌리지 않는다.

**제약**

외부 I/O, database, framework annotation은 제외한다. 상태 머신의 정확성과 invariant만 구현한다.

### 구현 후 자가 검증

- [ ] 정상 경로가 정확한 순서로 ACCEPTED에 도달한다.
- [ ] 각 phase에서 재시작해도 다음 action이 동일하다.
- [ ] 중복 checkpoint가 상태를 후퇴시키지 않는다.
- [ ] reserve 뒤 실패는 risk release로 연결된다.
- [ ] debit 뒤 실패는 wallet refund로 연결된다.
- [ ] 보상 완료 전에 REJECTED가 되지 않는다.
- [ ] terminal 상태에서 side effect가 다시 실행되지 않는다.
- [ ] 모순된 상태 조합이 거부된다.
- [ ] 무한 전이 없이 bounded loop로 진행 가능하다.

### 구현 후 설명할 것

1. 외부 호출과 checkpoint의 순서를 어떻게 정했는지
2. "정확히 한 번" 대신 멱등적 at-least-once를 택한 이유
3. compensation action을 동적으로 다시 계산하지 않고 저장하는 이유
4. local atomicity가 필요한 최종 전이 경계
5. 상태 수 증가와 복구 가능성 사이의 trade-off

### 원본 확인 위치

- Thread: 08, 체크포인트 기반 베팅 접수 사가와 보상
- 커밋: `feat(database): add compensation verdict schema`, `feat(placement): checkpoint external side effects`, `feat(placement): commit risk before atomic acceptance`, `test(recovery): repeat commit and compensation checkpoints`
- 파일: `Bet.java`, `BetPlacementService.java`, `BetStore.java`, `V6__placement_compensation_and_verdict.sql`
- 함수·컴포넌트: `BetPlacementService.advance(...)`, `BetPlacementService.reconcile(...)`, `BetStore.recordRiskReservation(...)`, `confirmWallet(...)`, `commitRisk(...)`, `beginCompensation(...)`, `completeRiskRelease(...)`, `completeWalletRefund(...)`, `acceptAndEnqueue(...)`
- 관련 Thread: 05, 06, 07, 09, 10, 11

---

<a id="t11-owner-fenced-leases"></a>
## [Thread 11 / `feat(recovery): claim fair reconciliation batches`] `SKIP LOCKED`와 owner fencing으로 복구 작업 분배하기

### 면접 질문

여러 인스턴스가 stale PENDING bet를 복구할 때 단순히 같은 조회 결과를 읽어 각자 처리하면 어떤 문제가 생깁니까? `FOR UPDATE SKIP LOCKED`, owner, lease expiry, retry eligibility를 함께 사용한 이유를 설명해 보세요.

꼬리 질문:

- lease 시간에 애플리케이션 clock 대신 DB `CURRENT_TIMESTAMP`를 사용한 이유는 무엇입니까?
- claim query가 `ORDER BY eligibleAt, requestedAt, createdAt, betId` 같은 안정적 순서를 가져야 하는 이유는 무엇입니까?
- owner 조건 없이 claim을 clear하면 어떤 race가 생깁니까?
- worker가 죽으면 claim은 어떻게 회수됩니까?
- 처리 실패 뒤 즉시 재claim하지 않고 `reconciliation_eligible_at`을 미래로 미는 이유는 무엇입니까?
- `SKIP LOCKED`가 starvation을 완전히 막아 줍니까?
- transaction이 claim된 작업 전체를 수행할 때까지 열린 상태여야 합니까?

### 30초 모범 답변

복구 대상 조회와 소유권 획득을 분리하면 여러 worker가 같은 row를 잡습니다. 그래서 짧은 DB transaction 안에서 eligible row를 공정한 순서로 고르고 `FOR UPDATE SKIP LOCKED`로 겹치지 않게 한 뒤 owner와 만료 시각을 원자적으로 기록합니다. 실제 복구는 transaction 밖에서 수행하고, 마지막 clear는 `betId + owner` 조건으로 fencing합니다. worker가 죽어도 lease가 DB 시간 기준으로 만료되면 다른 worker가 회수할 수 있고, retry eligibility를 미뤄 hot loop를 막습니다.

### 답변 핵심 키워드

`work claiming`, `FOR UPDATE SKIP LOCKED`, `bounded batch`, `lease expiry`, `owner fencing`, `database time`, `fair ordering`, `crash recovery`, `retry eligibility`

### 백지 구현

**구현 목표**

메모리 내 작업 목록을 대상으로 DB claim 계약과 동일한 의미를 갖는 bounded lease claim 함수를 구현한다. 동시 호출을 고려해 메서드 단위 원자성을 보장한다.

**인터페이스 또는 함수 시그니처**

```java
record WorkItem(
    UUID id,
    boolean pending,
    Instant eligibleAt,
    String claimOwner,
    Instant claimUntil) {}

final class LeaseQueue {
  List<UUID> claim(
      String owner,
      Instant databaseNow,
      Duration lease,
      Duration retryDelay,
      int batchSize) {
    // 직접 구현
    return null;
  }

  boolean clear(UUID id, String owner) {
    // 직접 구현
    return false;
  }
}
```

**입력과 출력**

- 입력: owner, 신뢰할 수 있는 기준 시각, lease, retry delay, batch size
- 출력: 이번 호출이 독점적으로 claim한 ID 목록

**반드시 만족해야 할 조건**

- pending이며 eligible하고 claim이 없거나 만료된 항목만 대상이다.
- 가장 오래 기다린 항목부터 안정적으로 정렬한다.
- 최대 batch size만 claim한다.
- claim 시 owner, claimUntil, 다음 eligibleAt을 함께 갱신한다.
- 같은 시각의 두 owner가 같은 항목을 동시에 얻지 못한다.
- `clear`는 현재 owner가 일치할 때만 성공한다.
- 만료된 lease는 다른 owner가 회수할 수 있다.
- terminal 항목은 claim되지 않는다.

**경계 조건**

- 빈 queue
- batch size 1, 전체보다 큰 batch
- claimUntil이 정확히 now인 항목
- eligibleAt이 정확히 now인 항목
- lease가 만료되기 직전과 직후
- 오래된 owner가 늦게 clear를 시도
- 동일 eligibleAt을 가진 여러 항목

**실패 조건**

owner 공백, 0 이하 duration, 0 이하 batch size를 거부한다. 중간 실패로 일부 항목만 owner 없이 시간만 바뀌는 상태가 생기지 않아야 한다.

**제약**

한 JVM에서 `synchronized` 또는 lock을 사용할 수 있다. 구현 후 실제 DB에서는 왜 row lock과 atomic update가 필요한지도 설명한다.

### 구현 후 자가 검증

- [ ] 두 owner의 claim 결과가 겹치지 않는다.
- [ ] 오래된 eligible 항목부터 선택된다.
- [ ] batch 상한을 넘지 않는다.
- [ ] 아직 유효한 lease는 다른 owner가 가져가지 못한다.
- [ ] 만료된 lease는 회수된다.
- [ ] 이전 owner의 늦은 clear가 새 owner의 claim을 지우지 못한다.
- [ ] pending이 아닌 항목이 제외된다.
- [ ] retry eligibility가 즉시 재claim hot loop를 막는다.
- [ ] claim 변경이 원자적으로 일어난다.

### 구현 후 설명할 것

1. claim transaction과 실제 작업 transaction을 분리한 이유
2. DB 시간을 기준으로 삼은 이유
3. owner fencing이 lease만 사용하는 것보다 안전한 이유
4. `SKIP LOCKED`의 처리량 장점과 공정성 한계
5. lease 길이, retry delay, batch size의 운영 trade-off

### 원본 확인 위치

- Thread: 11, 소유자 펜싱 리스를 이용한 공정한 접수 복구
- 커밋: `feat(placement): resume stale placement checkpoints`, `feat(recovery): add owner-fenced reconciliation leases`, `feat(recovery): claim fair reconciliation batches`, `feat(recovery): consume owner-fenced reconciliation claims`
- 파일: `Bet.java`, `BetRepository.java`, `BetReconciliationJob.java`, `V10__reconciliation_lease.sql`, `PostgresRecoveryClaimIntegrationTest.java`
- 함수·컴포넌트: `BetRepository.claimReconciliationBatch(...)`, `BetRepository.clearReconciliationClaim(...)`, `BetReconciliationJob.reconcile()`
- 관련 Thread: 08, 10, 17

---

<a id="t17-root-locking"></a>
## [Thread 17 / `fix(persistence): lock bet roots before loading legs`] 쓰기 잠금과 연관 그래프 로딩을 분리하기

### 면접 질문

집계 전이를 직렬화하려고 `PESSIMISTIC_WRITE`와 `EntityGraph(legs)`를 한 repository method에 함께 둔 구현을, 루트 row를 먼저 잠그고 같은 transaction에서 legs를 초기화하는 두 단계로 바꾼 이유는 무엇입니까?

꼬리 질문:

- join fetch나 entity graph가 있는 select에 `FOR UPDATE`를 붙이면 DB별로 어떤 문제가 생길 수 있습니까?
- 루트만 잠근 뒤 자식 컬렉션을 읽으면 다른 transaction의 자식 변경까지 안전합니까?
- 이 프로젝트에서 자식 변경은 왜 루트 전이와 같은 경계로 제한되어야 합니까?
- lazy collection 초기화는 반드시 어느 transaction 범위에서 일어나야 합니까?
- optimistic locking만으로 대체할 수 있는 상황은 언제입니까?
- read-only 조회는 왜 별도의 graph-loading method를 유지해야 합니까?

### 30초 모범 답변

전이를 직렬화할 대상은 집계 루트 row입니다. 루트와 one-to-many graph를 한 번에 join해 pessimistic lock을 걸면 DB가 outer join 대상까지 잠그려 하거나 `FOR UPDATE` 제약으로 실패할 수 있고, 예상보다 넓은 잠금이 생깁니다. 그래서 명시적 JPQL로 root만 `PESSIMISTIC_WRITE`로 잠근 뒤 같은 transaction 안에서 legs를 초기화합니다. 읽기 전용 조회는 entity graph를 쓰고, 전이 경로는 루트 잠금과 graph loading을 분리해 잠금 범위를 명확히 합니다.

### 답변 핵심 키워드

`pessimistic root lock`, `entity graph`, `join fetch`, `lock amplification`, `same transaction`, `lazy initialization`, `aggregate serialization`, `optimistic vs pessimistic`

### 백지 구현

**구현 목표**

repository 계약을 루트 잠금용 쿼리와 전체 집계 조회용 쿼리로 분리하고, service가 같은 transaction에서 root lock 후 children을 초기화하도록 skeleton을 완성한다.

**인터페이스 또는 함수 시그니처**

```java
interface OrderRepository {
  Optional<Order> findLockedRoot(UUID orderId);
  Optional<Order> findWithItems(UUID orderId);
}

final class OrderTransitionStore {
  Order loadForUpdate(UUID orderId) {
    // 직접 구현
    return null;
  }
}
```

Spring Data/JPA annotation을 사용한다면 다음 계약을 만족하도록 선언한다.

```java
// findLockedRoot: 루트만 선택하고 PESSIMISTIC_WRITE
// findWithItems: 읽기용 graph loading, 쓰기 잠금 없음
// loadForUpdate: 하나의 쓰기 transaction 안에서 루트 잠금 후 items 초기화
```

**입력과 출력**

- 입력: 집계 ID
- 출력: 루트가 쓰기 잠금된 상태이며 child collection이 transaction 안에서 초기화된 집계

**반드시 만족해야 할 조건**

- 잠금 쿼리는 root entity만 선택한다.
- 전체 graph fetch와 pessimistic lock annotation을 같은 쿼리에 결합하지 않는다.
- child 초기화가 transaction 종료 전에 끝난다.
- ID가 없으면 일관된 not-found 오류를 낸다.
- read-only 경로는 불필요한 쓰기 잠금을 획득하지 않는다.
- 전이 코드는 항상 `loadForUpdate` 경로를 사용한다.

**경계 조건**

- child가 0개인 집계
- 많은 child를 가진 집계
- 동시에 같은 ID를 전이하려는 두 transaction
- transaction 밖에서 lazy collection 접근
- read-only 조회 뒤 상태 변경 시도

**실패 조건**

잠금 실패·timeout과 not-found를 구분한다. transaction 밖 lazy loading에 의존하지 않는다.

**제약**

실제 DB 통합 테스트는 작성하지 않아도 되지만, annotation 또는 쿼리 계약을 코드에 드러낸다.

### 구현 후 자가 검증

- [ ] root lock 쿼리에 graph fetch가 붙지 않았다.
- [ ] read query에는 불필요한 `PESSIMISTIC_WRITE`가 없다.
- [ ] children이 transaction 안에서 초기화된다.
- [ ] 같은 집계의 두 전이가 직렬화된다.
- [ ] 다른 집계 ID끼리는 불필요하게 막지 않는다.
- [ ] not-found와 lock timeout을 구분할 수 있다.
- [ ] 모든 쓰기 전이 진입점이 같은 잠금 경계를 사용한다.
- [ ] DB별 `FOR UPDATE`와 join 제약을 설명할 수 있다.

### 구현 후 설명할 것

1. 잠금 대상이 root row인 이유
2. eager graph와 lock을 분리함으로써 줄어드는 잠금 범위
3. child 변경 규칙이 aggregate 경계와 맞아야 하는 이유
4. optimistic locking을 선택할 수 있는 대안 조건
5. lazy initialization을 명시적으로 수행한 이유와 N+1 trade-off

### 원본 확인 위치

- Thread: 17, 집계 루트 잠금과 그래프 로딩 분리
- 커밋: `feat(persistence): load owned bets with evidence`, `fix(persistence): lock bet roots before loading legs`, `test(persistence): separate root locking from graph loading`
- 파일: `BetRepository.java`, `BetRepositoryContractTest.java`
- 함수·컴포넌트: `findLockedByBetId(...)`, `findLockedRootByBetId(...)`, `findWithLegsByBetId(...)`
- 관련 Thread: 08, 11, 15, 16
