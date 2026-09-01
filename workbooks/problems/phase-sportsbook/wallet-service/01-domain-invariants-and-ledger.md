# 도메인 invariant와 이중부기 원장

이 문서는 계정 상태 전이와 append-only 원장 쌍을 하나의 면접 축으로 묶는다. 프로젝트의 실제 구현 명칭은 `원본 확인 위치`에만 적었고, 백지 구현 인터페이스는 면접용으로 축소한 별도 문제다.

<a id="w01"></a>
## W01 — [Thread 4 / `test(domain): verify Long.MAX aggregate boundaries`] 계정 잔액 전이와 표현 범위 invariant

### 면접 질문

`available`과 `locked`를 각각 비음수로 유지하는 것만으로는 왜 충분하지 않습니까? 이 프로젝트의 계정 모델에서 한 번의 상태 전이가 반드시 보존해야 하는 invariant를 설명해 보세요.

꼬리 질문:

- `available + locked <= Long.MAX_VALUE`를 검사할 때 실제 합을 먼저 계산하면 어떤 문제가 생깁니까?
- `available → locked` 이동과 외부 입금은 총액 관점에서 어떻게 다릅니까?
- 통화 불일치나 잔액 부족이 발생했을 때 객체가 부분 변경되지 않게 하려면 어떤 순서로 검증해야 합니까?
- 도메인 객체와 DB `CHECK` 제약을 둘 다 두는 이유는 무엇입니까?

### 30초 모범 답변

계정은 하나의 통화와 비음수인 `available`, `locked` 두 버킷을 가지며 두 값의 합도 signed 64-bit 범위 안에 있어야 합니다. 그래서 증가 연산은 합을 직접 더하기보다 `available <= Long.MAX_VALUE - locked` 같은 형태로 먼저 표현 가능성을 확인해야 합니다. 버킷 간 이동은 총액을 보존하고, 입금·몰수 같은 외부 이동은 총액을 바꿉니다. 모든 조건을 먼저 검증한 뒤 한 번에 새 상태를 만들면 실패 시 부분 변경을 피할 수 있고, DB 제약은 애플리케이션 밖의 쓰기와 버그까지 마지막으로 차단합니다.

### 답변 핵심 키워드

`단일 통화` · `비음수 버킷` · `합계 표현 범위` · `subtract-before-add` · `검증 후 변경` · `총액 보존 전이` · `도메인 + DB 이중 방어`

### 백지 구현

#### 구현 목표

불변 객체 `BalanceState`에 한 건의 잔액 이동을 적용한다. 성공하면 새 상태를 반환하고, 실패하면 원본 상태를 전혀 바꾸지 않은 채 예외를 던진다.

#### 면접용 축소 인터페이스

```java
enum MovementType {
  CREDIT_AVAILABLE,
  DEBIT_AVAILABLE,
  LOCK,
  UNLOCK,
  FORFEIT_LOCKED
}

record BalanceState(long available, long locked, Currency currency) {}
record Movement(MovementType type, long amount, Currency currency) {}

final class BalanceTransitions {
  static BalanceState apply(BalanceState before, Movement movement) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 두 버킷과 통화, 이동 종류, 양의 금액, 요청 통화
- 출력: 모든 invariant를 만족하는 새 `BalanceState`
- 실패: 원본 상태를 변경하지 않고 적절한 예외 발생

#### 반드시 만족해야 할 조건

- 현재 상태와 결과 상태의 두 버킷은 모두 0 이상이다.
- 요청 금액은 0보다 커야 한다.
- 요청 통화는 계정 통화와 같아야 한다.
- `available + locked`는 어떤 성공 상태에서도 `Long.MAX_VALUE`를 넘지 않는다.
- `LOCK`은 `available`을 줄이고 `locked`를 같은 양만큼 늘린다.
- `UNLOCK`은 `locked`를 줄이고 `available`을 같은 양만큼 늘린다.
- `CREDIT_AVAILABLE`은 총액을 늘리고, `DEBIT_AVAILABLE`과 `FORFEIT_LOCKED`는 총액을 줄인다.
- 잔액 부족, 통화 불일치, 표현 범위 초과를 검증하기 전에 상태를 쓰지 않는다.

#### 경계 조건

- 요청 금액이 0 또는 음수인 경우
- 요청 금액과 해당 버킷 잔액이 정확히 같은 경우
- 성공 결과의 총액이 정확히 `Long.MAX_VALUE`인 경우
- 1만 더하면 표현 범위를 넘는 경우
- `available`은 충분하지만 `locked`가 부족한 `FORFEIT_LOCKED`
- 계정 통화와 요청 통화가 다른 경우

#### 실패 조건

- 잘못된 현재 상태가 전달된 경우
- 금액 부족
- 통화 불일치
- 합계 범위 초과
- 알 수 없는 이동 종류

#### 필요한 제약

- `BigDecimal`이나 부동소수점으로 우회하지 않는다.
- 성공 경로의 시간 복잡도는 O(1), 추가 공간은 O(1)이어야 한다.
- 전이 실패 후 `before`의 관찰값이 그대로여야 한다.

### 구현 후 자가 검증

- [ ] 1원 입금, 출금, 잠금, 잠금 해제, 몰수의 정상 경로를 각각 확인했다.
- [ ] 정확한 잔액을 전부 이동한 뒤 소스 버킷이 0이 된다.
- [ ] `Long.MAX_VALUE` 경계에서 허용되는 마지막 증가와 거부되는 다음 증가를 구분한다.
- [ ] 부족 잔액·통화 불일치·0원 요청 뒤에 입력 상태가 변하지 않는다.
- [ ] `LOCK`과 `UNLOCK` 전후 총액이 같다.
- [ ] 결과의 두 버킷이 항상 비음수다.
- [ ] 합계를 먼저 더해 overflow된 값을 비교하지 않는다.
- [ ] 모든 분기가 O(1)이다.

### 구현 후 설명할 것

1. 왜 가변 객체를 조금씩 고치기보다 검증 완료 후 새 상태를 반환했는가.
2. 합계를 직접 계산하지 않고 남은 여유 공간을 비교한 이유.
3. 도메인 검증과 DB 제약이 중복이 아니라 서로 다른 실패 경계를 담당하는 이유.
4. 버킷 간 이동과 외부 상대 계정이 있는 이동을 총액 변화 기준으로 구분한 방식.
5. 향후 금액 표현을 `long`보다 넓게 바꿀 때 영향을 받는 경계.

### 원본 확인 위치

- Thread: **4 — 계정 잔액 불변식**
- 커밋: `feat(domain): model account identity and reject system UUIDs`
- 커밋: `test(domain): verify Long.MAX aggregate boundaries`
- 커밋: `test(domain): verify locked-bucket invariants`
- 파일: `src/main/java/com/sportsbook/wallet/domain/Account.java`
- 파일: `src/main/java/com/sportsbook/wallet/domain/EmbeddedMoney.java`
- 파일: `src/main/resources/db/migration/V1__account_and_ledger.sql`
- 클래스·메서드: `Account`, `Account.openFor()`, `Account.total()` 및 available/locked 변경 메서드
- 테스트: `AccountIdentityTest`, `AccountBalanceLimitTest`, `AccountLockedFundsTest`
- 관련 Thread: 1, 5, 7, 11, 14

---

<a id="w02"></a>
## W02 — [Thread 5 / `feat(service): derive results from complete ledger pairs`] 이중부기 쌍 복원과 업무 토폴로지

### 면접 질문

저장된 원장 행이 정확히 두 개이고 `DEBIT`, `CREDIT`이 하나씩 있다는 사실만 확인하면 성공 결과를 안전하게 복원할 수 있습니까? 이 프로젝트에서 추가로 검증해야 하는 공통 필드와 업무 이유별 토폴로지를 설명해 보세요.

꼬리 질문:

- 같은 금액이라도 `DEPOSIT`의 상대 계정이 `HOUSE`라면 왜 잘못된 원장입니까?
- `BET_DEBIT`는 같은 사용자 계정 안에서 어떤 버킷 사이를 이동합니까?
- 원장 결과 복원 시 목록 순서에 의존하면 어떤 문제가 생깁니까?
- 원장 행을 append-only로 두면서도 손상 여부를 어떻게 검출할 수 있습니까?
- 이 프로젝트의 `DEBIT`가 destination, `CREDIT`이 source를 나타내는 규칙을 일반 회계 용어와 혼동하지 않으려면 어떻게 설명하겠습니까?

### 30초 모범 답변

두 행은 하나의 `operationGroupId`, 멱등성 키, 금액, 통화, 이유, 시각을 공유하고 side는 정확히 하나씩 달라야 합니다. 그 다음 이유별로 계정과 버킷 방향을 검증해야 합니다. 예를 들어 `BET_DEBIT`는 사용자 `AVAILABLE`에서 같은 사용자의 `LOCKED`로 가고, `BET_FORFEIT`는 사용자 `LOCKED`에서 `HOUSE`로 갑니다. 목록 순서가 아니라 side로 행을 찾고, 공통 필드와 토폴로지를 모두 통과한 경우에만 성공 결과를 복원해야 손상된 원장을 정상 결과로 오인하지 않습니다.

### 답변 핵심 키워드

`정확히 두 행` · `한 DEBIT/한 CREDIT` · `공통 operation group` · `순서 비의존` · `이유별 상대 계정` · `버킷 방향` · `append-only 검증`

### 백지 구현

#### 구현 목표

두 원장 행이 하나의 완전한 자금 이동을 표현하는지 검증하고, 성공 결과를 복원한다. 이 문제에서는 프로젝트의 최종 토폴로지 규칙을 사용한다.

#### 면접용 축소 인터페이스

```java
enum Side { DEBIT, CREDIT }
enum Bucket { AVAILABLE, LOCKED }
enum Reason {
  DEPOSIT, WITHDRAW, BET_DEBIT, BET_PAYOUT,
  BET_REFUND, BET_FORFEIT, BET_ADJUSTMENT
}

record LedgerRow(
    UUID entryId,
    UUID accountId,
    Bucket bucket,
    Side side,
    long amount,
    Currency currency,
    Reason reason,
    String idempotencyKey,
    UUID operationGroupId,
    Instant createdAt) {}

record TransferView(
    UUID operationGroupId,
    UUID userId,
    long amount,
    Currency currency,
    Reason reason,
    Instant createdAt) {}

final class LedgerPairReconstructor {
  static TransferView reconstruct(
      UUID userId,
      UUID houseId,
      UUID externalPaymentId,
      List<LedgerRow> rows) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 사용자 ID, 두 시스템 상대 계정 ID, 원장 행 목록
- 출력: 검증된 `TransferView`
- 실패: 불완전 쌍, 공통 필드 불일치, 잘못된 상대 계정·버킷·방향이면 예외

#### 반드시 만족해야 할 조건

공통 조건:

- 행은 정확히 두 개다.
- `DEBIT`, `CREDIT`이 각각 정확히 하나다.
- 두 `entryId`는 다르다.
- 금액은 양수이고 두 행의 금액·통화가 같다.
- `reason`, `idempotencyKey`, `operationGroupId`, `createdAt`이 같다.
- 입력 목록의 정렬 순서와 무관하게 같은 결과가 나온다.

통합된 최종 토폴로지:

| Reason | DEBIT 행 | CREDIT 행 |
| --- | --- | --- |
| `DEPOSIT` | 사용자 / `AVAILABLE` | `EXTERNAL_PAYMENT` / `AVAILABLE` |
| `WITHDRAW` | `EXTERNAL_PAYMENT` / `AVAILABLE` | 사용자 / `AVAILABLE` |
| `BET_DEBIT` | 사용자 / `LOCKED` | 사용자 / `AVAILABLE` |
| `BET_PAYOUT` | 사용자 / `AVAILABLE` | `HOUSE` / `AVAILABLE` |
| `BET_REFUND` | 사용자 / `AVAILABLE` | 사용자 / `LOCKED` 또는 `HOUSE` / `AVAILABLE` |
| `BET_FORFEIT` | `HOUSE` / `AVAILABLE` | 사용자 / `LOCKED` |
| `BET_ADJUSTMENT` | 사용자 / `AVAILABLE` ↔ `HOUSE` / `AVAILABLE` 중 한 방향 |

#### 경계 조건

- 빈 목록, 한 행, 세 행
- 두 행이 모두 같은 side인 경우
- 공통 group은 같지만 amount나 currency가 다른 경우
- 동일한 transfer leg가 양쪽에 반복되는 경우
- `BET_REFUND`의 두 허용 source 중 하나
- `BET_ADJUSTMENT`의 양수·음수 두 방향
- 목록 순서를 뒤집은 입력
- 시스템 ID와 사용자 ID가 우연히 같게 전달된 잘못된 입력

#### 실패 조건

- 불완전하거나 중복된 원장 쌍
- 지원하지 않는 이유
- 이유와 맞지 않는 상대 계정 또는 버킷
- 공통 식별자·시각 불일치
- 비양수 금액

#### 필요한 제약

- 첫 번째 행을 무조건 기준 행으로 삼아 side를 추론하지 않는다.
- 두 행만 처리하므로 핵심 검증은 O(1)이어야 한다.
- 오류 메시지에 민감한 멱등성 키 전체를 불필요하게 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 모든 허용 토폴로지를 한 번씩 통과시켰다.
- [ ] 행 순서를 바꿔도 결과가 같다.
- [ ] 같은 side 두 개를 거부한다.
- [ ] group은 같지만 amount·currency·reason·timestamp 중 하나가 다른 쌍을 각각 거부한다.
- [ ] `DEPOSIT` 상대 계정을 `HOUSE`로 바꾼 쌍을 거부한다.
- [ ] `BET_DEBIT`의 AVAILABLE/LOCKED 방향을 뒤집은 쌍을 거부한다.
- [ ] `BET_REFUND`의 사용자 locked source와 house-funded source를 모두 허용한다.
- [ ] 반환 결과가 원장 행의 공유 facts에서만 만들어진다.

### 구현 후 설명할 것

1. 목록 순서가 아니라 side를 기준으로 행을 식별한 이유.
2. 구조적 완전성 검증과 업무 토폴로지 검증을 두 단계로 나눈 이유.
3. 이유별 `switch`와 데이터 기반 매핑 테이블 중 선택한 방식과 변경 비용.
4. append-only 원장에서 결과를 다시 만들 때 검증을 엄격하게 해야 하는 이유.
5. 새로운 `Reason` 추가 시 domain, DB constraint, 결과 복원, integrity scan을 함께 바꿔야 하는 이유.

### 원본 확인 위치

- Thread: **5 — 이중부기 원장 토폴로지**
- 커밋: `feat(domain): construct matched debit-credit pairs`
- 커밋: `feat(service): derive results from complete ledger pairs`
- 커밋: `test(service): reject reason topology mismatches`
- 파일: `src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/LedgerEntryRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java`
- 클래스·메서드: `LedgerEntry.pair()`, `LedgerEntry.TransferLeg`, `LedgerEntry.Pair`, `WalletOperationResult.fromExisting()`
- 테스트: `LedgerEntryTest`, `WalletTransferTopologyTest`
- 관련 Thread: 4, 7, 9, 14
