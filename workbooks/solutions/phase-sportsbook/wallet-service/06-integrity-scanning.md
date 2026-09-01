# 영속 상태 무결성 스캔

정상 쓰기 경로에 invariant가 있어도 운영 중 손상 가능성을 0으로 가정할 수는 없다. 이 문서는 account snapshot과 append-only ledger를 독립적으로 대사하고, 여러 invariant query를 하나의 반복 가능한 DB 뷰에서 실행하는 방법을 다룬다.

<a id="w11"></a>
## W11 — [Thread 14 / `feat(integrity): scan durable wallet invariants`] repeatable-read 원장 대사

### 면접 질문

계정 테이블의 `available`, `locked` 값이 정상이라고 어떻게 독립적으로 검증할 수 있습니까? 여러 drift query를 각각 별도 read-committed 트랜잭션에서 실행하면 어떤 거짓 양성이나 거짓 음성이 생길 수 있습니까?

꼬리 질문:

- append-only ledger에서 계정·버킷별 순증감을 계산할 때 이 프로젝트의 DEBIT/CREDIT 부호는 어떻게 적용합니까?
- 합계 계산에 `long` 대신 더 넓은 정수 표현이 필요한 이유는 무엇입니까?
- 사용자 account가 없는 ledger row는 모두 orphan입니까? 시스템 상대 계정은 어떻게 처리합니까?
- DB `CHECK`와 foreign key가 있는데도 사후 integrity scan이 필요한 이유는 무엇입니까?
- 모든 drift ID를 metric label로 내보내면 어떤 문제가 생깁니까?
- Prometheus scrape 때마다 DB scan을 실행하지 않고 마지막 완료 snapshot을 노출한 이유는 무엇입니까?

### 30초 모범 답변

원장을 계정과 버킷별로 다시 합산해 account snapshot과 비교합니다. 이 프로젝트 모델에서는 destination인 DEBIT를 증가, source인 CREDIT를 감소로 해석하고, 합계 중간값은 `long`을 넘을 수 있으므로 넓은 정수로 계산합니다. 사용자 account가 없는 행은 `HOUSE`, `EXTERNAL_PAYMENT` 같은 알려진 시스템 상대 계정을 제외하고 orphan으로 봅니다. account, operation, recovery, adjustment를 검사하는 여러 query는 하나의 repeatable-read 트랜잭션에서 실행해야 모두 같은 DB snapshot을 봅니다. 결과는 ID 전체가 아니라 최신 완료 scan의 drift count로 metric·health에 내보내 cardinality와 scrape 비용을 제한합니다.

### 답변 핵심 키워드

`ledger recomputation` · `signed aggregation` · `wide integer` · `orphan exclusion` · `repeatable read` · `independent verification` · `bounded metrics` · `snapshot health`

### 백지 구현

#### 구현 목표

메모리의 account snapshot과 ledger row를 대사해 금액·통화 drift와 orphan ledger account를 찾는다. 이후 실제 DB scanner에서 이 계산을 어떤 isolation으로 묶을지 설명한다.

#### 면접용 축소 인터페이스

```java
enum Side { DEBIT, CREDIT }
enum Bucket { AVAILABLE, LOCKED }

record AccountSnapshot(
    UUID userId,
    long available,
    long locked,
    Currency currency) {}

record LedgerRow(
    UUID accountId,
    Bucket bucket,
    Side side,
    long amount,
    Currency currency) {}

record AccountDrift(
    UUID userId,
    BigInteger expectedAvailable,
    BigInteger storedAvailable,
    BigInteger expectedLocked,
    BigInteger storedLocked,
    String reason) {}

record ReconciliationResult(
    List<AccountDrift> accountDrifts,
    List<UUID> orphanLedgerAccountIds) {}

final class AccountLedgerReconciler {
  static ReconciliationResult reconcile(
      List<AccountSnapshot> accounts,
      List<LedgerRow> ledger,
      Set<UUID> systemAccountIds) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 account snapshot, 전체 또는 범위 내 ledger rows, 알려진 시스템 상대 계정 ID
- 출력: 안정적으로 정렬된 account drift와 orphan account ID

#### 반드시 만족해야 할 조건

원장 합산:

- 금액은 양수라고 가정하거나 그렇지 않으면 별도 drift로 거부한다.
- 이 프로젝트 규칙에서 `DEBIT`는 해당 account/bucket의 증가, `CREDIT`은 감소다.
- `(accountId, bucket, currency)`별 합계를 `BigInteger`로 계산한다.
- ledger가 없는 계정의 기대 값은 두 버킷 모두 0이다.
- 한 account에 여러 통화의 ledger가 섞이면 currency drift다.

snapshot 비교:

- 기대 available/locked와 저장값을 각각 비교한다.
- account의 두 버킷 통화가 하나라는 전제를 검증한다.
- 저장 금액이 비음수이고 합계가 `long` 범위라는 별도 상태 invariant도 확인할 수 있어야 한다.
- drift 출력 순서는 user ID 기준으로 안정적이어야 한다.

orphan:

- ledger account ID가 사용자 account 목록에 없고 system ID 집합에도 없으면 orphan이다.
- 같은 orphan ID는 한 번만 출력한다.

scanner 경계:

- 실제 account drift, operation group, recovery queue, adjustment outcome/failure/fingerprint/ledger query는 하나의 `REPEATABLE_READ`, read-only scan에 묶는다.
- metric은 개별 ID가 아니라 최신 완료 snapshot의 count를 사용한다.
- scrape 요청 자체가 DB query를 실행하지 않는다.

#### 경계 조건

- account와 ledger가 모두 비어 있음
- ledger 없는 0원 account
- 입금·출금이 상쇄되어 net 0인 account
- available과 locked 이동만 있어 총액은 같지만 버킷이 다른 경우
- 합계 중간값이 `Long.MAX_VALUE`를 넘는 많은 ledger rows
- system account ledger만 존재
- 알려지지 않은 orphan account ID가 여러 행에 반복
- 같은 account에 KRW와 USD ledger가 섞임
- scan 도중 정상 거래가 커밋되는 상황을 isolation 관점에서 설명

#### 실패 조건

- 음수·0 ledger amount
- null account·bucket·side·currency
- 중복 account snapshot
- 저장 account가 표현 불가능한 상태
- scanner query 중 하나가 실패한 경우

#### 필요한 제약

- `double`, `BigDecimal` scale 비교로 정수 원장을 합산하지 않는다.
- account마다 ledger 전체를 다시 순회하는 O(A×L) 구현을 피한다.
- 목표 시간 복잡도는 O(A+L), 추가 공간은 O(A+고유 원장 계정 수)다.
- 미완료 scan의 일부 count를 최신 정상 snapshot처럼 노출하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 account/ledger가 drift 0을 만든다.
- [ ] available 또는 locked 한쪽만 틀린 경우를 각각 찾는다.
- [ ] 버킷 간 이동을 총액만 비교해 놓치지 않는다.
- [ ] ledger가 없는 0원 account를 정상으로 본다.
- [ ] 큰 합계를 `BigInteger`로 안전하게 처리한다.
- [ ] system IDs는 orphan에서 제외하고 미등록 ID는 한 번만 보고한다.
- [ ] 통화가 섞인 ledger를 drift로 찾는다.
- [ ] 결과 순서가 입력 순서와 무관하게 안정적이다.
- [ ] O(A+L) 구조로 구현했다.
- [ ] 실제 scanner의 모든 query가 같은 repeatable-read view를 사용해야 하는 이유를 설명할 수 있다.

### 구현 후 설명할 것

1. 정상 write path와 별도 SQL/reconciliation 경로로 invariant를 다시 계산한 이유.
2. read committed가 아닌 repeatable read를 택한 이유와 snapshot이 오래 유지될 때의 비용.
3. DB constraint가 막는 손상과 사후 scan만 찾을 수 있는 cross-table semantic drift의 차이.
4. 개별 drift ID 대신 count snapshot을 metric으로 노출한 cardinality·성능 이유.
5. health를 `UNKNOWN`, `DOWN`, `UP`으로 나누고 마지막 완료 scan만 신뢰하는 이유.

### 원본 확인 위치

- Thread: **14 — 영속 상태 무결성 스캔**
- 커밋: `feat(integrity): reconcile account ledger snapshots`
- 커밋: `feat(integrity): project operation ledger groups`
- 커밋: `feat(integrity): verify adjustment fingerprints`
- 커밋: `feat(integrity): reconcile adjustment outcomes`
- 커밋: `feat(integrity): scan durable wallet invariants`
- 커밋: `feat(integrity): summarize scan drift counts`
- 커밋: `feat(integrity): publish scan metrics`
- 커밋: `feat(integrity): report scan health`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/AccountIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/OperationIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/RecoveryQueueIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/AdjustmentOperationIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/AdjustmentFailureIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/AdjustmentFingerprintIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/AdjustmentLedgerIntegrityRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScanner.java`
- 클래스·메서드: `AccountIntegrityRepository.findSnapshotDrift()`, `findOrphanLedgerAccountIds()`, `WalletIntegrityScanner.scan()`
- 관련 컴포넌트: `WalletIntegritySnapshot`, `WalletIntegrityMetrics`, `WalletIntegrityHealth`, `WalletIntegrityScheduler`
- 관련 Thread: 1, 4, 5, 6, 10, 11
