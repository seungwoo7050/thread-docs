# 도메인 invariant와 운영 경계

이 문서는 마스터 인덱스의 W13~W16을 다룬다. 도메인별 필드명을 외우는 것보다, 여러 값의 조합이 하나의 유효한 상태를 이루는지 증명하고 운영 경계에서 불완전한 입력·출력을 차단하는 판단을 연습한다. 백지 구현은 실제 코드를 복원하지 않으며, 확인된 invariant를 10~30분 안에 검증할 수 있도록 축소했다.

<a id="W13"></a>
## [Thread 17 / `feat(risk): verify complete limits snapshots`; `feat(risk): clear one user limit override`] 완전한 위험 한도 집합과 scope invariant

### 면접 질문

위험 한도 조회가 정확히 7개의 항목을 반환해야 할 때, 단순히 `list.size() == 7`만 검사하면 왜 충분하지 않습니까? 이 프로젝트에서 한도 종류와 통화의 조합을 target으로 보고, 누락·중복·지원하지 않는 조합을 함께 거부한 이유를 설명해 보세요.

꼬리 질문:

- `STAKE_DAILY`, `STAKE_WEEKLY`, `STAKE_MONTHLY`에는 통화가 필요하고 `SELECTIONS_PER_MINUTE`에는 통화가 없어야 하는 조건을 어디에서 강제하는 것이 좋습니까?
- 목록의 순서가 달라도 같은 snapshot으로 인정해야 한다면 어떤 자료구조가 적합합니까?
- 중복 항목 하나와 누락 항목 하나가 동시에 있으면 size 검사가 통과할 수 있는 이유는 무엇입니까?
- 요청한 user ID와 응답 user ID가 다르면 HTTP 200이어도 왜 계약 위반입니까?
- 값 상한을 `9_007_199_254_740_991`로 둔 선택은 JSON/JavaScript 소비자와 어떤 관련이 있습니까?
- offset이나 override를 변경하는 endpoint에서 type과 currency scope를 다시 검증해야 하는 이유는 무엇입니까?

### 30초 모범 답변

완전성은 개수보다 집합의 동일성으로 확인해야 합니다. 이 구현은 한도 종류와 선택적 통화를 하나의 target으로 만들고, 응답의 각 항목을 검증하면서 `Set`에 추가해 중복을 막은 뒤 요구되는 7개 target 집합과 정확히 비교합니다. 통화가 필요한 stake 한도와 통화가 금지되는 selection 한도를 교차 필드 invariant로 묶고, user ID·값 범위·source까지 확인합니다. 그래서 7개라는 숫자만 맞춘 중복·누락 snapshot이나 다른 사용자의 응답을 성공으로 오인하지 않습니다.

### 답변 핵심 키워드

`set equality` · `completeness` · `uniqueness` · `target identity` · `type/currency invariant` · `requested ID correlation` · `safe integer` · `order independent` · `fail closed`

### 백지 구현

#### 구현 목표

요청한 사용자의 위험 한도 snapshot이 지원되는 target을 정확히 한 번씩 포함하는지 검증한다. 목록 순서에는 의존하지 않는다.

#### 인터페이스 또는 함수 시그니처

```java
public enum LimitType {
  STAKE_DAILY,
  STAKE_WEEKLY,
  STAKE_MONTHLY,
  SELECTIONS_PER_MINUTE
}

public enum Currency {
  KRW,
  USD
}

public enum Source {
  POLICY,
  OVERRIDE
}

public record LimitEntry(
    LimitType type,
    Currency currency,
    Long value,
    Source source) {}

public record LimitSnapshot(UUID userId, List<LimitEntry> limits) {}

public final class LimitSnapshotVerifier {
  public static LimitSnapshot verify(UUID requestedUserId, LimitSnapshot snapshot) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 요청한 user ID와 하위 서비스가 반환한 snapshot
- 출력: 모든 조건을 만족하면 같은 snapshot 반환
- 실패 출력: 하나의 계약 위반 예외

#### 반드시 만족해야 할 조건

지원하는 target은 다음 7개뿐이다.

| 종류 | 통화 |
| --- | --- |
| `STAKE_DAILY` | `KRW` |
| `STAKE_DAILY` | `USD` |
| `STAKE_WEEKLY` | `KRW` |
| `STAKE_WEEKLY` | `USD` |
| `STAKE_MONTHLY` | `KRW` |
| `STAKE_MONTHLY` | `USD` |
| `SELECTIONS_PER_MINUTE` | 없음 |

- snapshot과 `userId`, `limits`는 null이 아니어야 한다.
- 응답 user ID는 요청 user ID와 정확히 같아야 한다.
- 각 entry와 `type`, `value`, `source`는 null이 아니어야 한다.
- stake 계열 type에는 currency가 반드시 있어야 한다.
- `SELECTIONS_PER_MINUTE`에는 currency가 없어야 한다.
- value는 `0..9_007_199_254_740_991` 범위여야 한다.
- `(type, currency)` target은 중복될 수 없다.
- 실제 target 집합은 위 7개 요구 집합과 정확히 같아야 한다.
- entry 순서는 결과의 유효성에 영향을 주지 않는다.

#### 경계 조건

- value가 `0`
- value가 최대 허용값
- 최대 허용값보다 1 큰 값
- 같은 target이 두 번 있고 다른 target이 하나 빠진 7개 목록
- 6개 또는 8개 목록
- selection 한도에 currency가 붙은 경우
- stake 한도에 currency가 빠진 경우
- 순서만 섞인 유효 목록
- 요청 user와 다른 user의 완전한 목록

#### 실패 조건

- 필수 값 누락
- 잘못된 type/currency 조합
- 음수 또는 안전 범위 초과 값
- 중복 target
- 지원 target 누락 또는 알 수 없는 target 추가
- 요청·응답 user 불일치

#### 필요한 제약

- 목록을 정렬하지 않아도 되며 전체 검증은 평균 `O(n)`이어야 한다.
- 입력 목록을 수정하지 않는다.
- 첫 번째 오류에서 중단하거나 모든 오류를 모으는 방식 중 하나를 선택하되, 외부 예외에는 민감하거나 불필요한 전체 응답을 포함하지 않는다.

### 구현 후 자가 검증

- [ ] 7개 target이 다른 순서로 와도 통과한다.
- [ ] 중복 1개와 누락 1개가 함께 있는 size 7 목록을 거부한다.
- [ ] 모든 stake type에서 KRW와 USD가 각각 한 번씩 필요하다.
- [ ] selection target은 currency가 없을 때만 통과한다.
- [ ] 요청 user ID와 응답 user ID 불일치를 거부한다.
- [ ] `0`과 최대 안전값은 허용하고 음수·상한 초과는 거부한다.
- [ ] null entry와 null source를 거부한다.
- [ ] 검증이 목록 순서에 의존하지 않는다.
- [ ] 시간 복잡도가 평균 `O(n)`, 추가 공간이 `O(k)`이며 여기서 `k`는 target 수다.

### 구현 후 설명할 것

1. `size == 7` 대신 target 집합의 정확한 동일성을 확인한 이유
2. `(type, currency)`를 하나의 식별 단위로 만든 이유
3. type/currency invariant를 payload 생성과 응답 검증 양쪽에서 지키는 이유
4. JSON 안전 정수 상한을 서비스 경계에서 강제한 trade-off
5. 목록 순서를 계약으로 만들지 않은 이유

### 원본 확인 위치

- Thread 17
- `feat(risk): model scoped limit updates`
- `feat(risk): verify complete limits snapshots`
- `test(risk): reject incomplete limits snapshots`
- `feat(risk): replace one user limit`
- `feat(risk): clear one user limit override`
- `RiskLimitType`
- `RiskLimitPayload`
- `RiskLimitsResponse`
- `RiskClient`
- `RiskAdminController`
- 관련 Thread: 08

---

<a id="W14"></a>
## [Thread 20 / `feat(settlement): verify revision lifecycle proof`; Thread 19 / `feat(settlement): model candidate evidence`] 정산 상태와 재시도 증거 검증

### 면접 질문

정산 개정 응답의 필드가 각각 올바른 타입으로 역직렬화되었다고 해도 왜 신뢰할 수 없습니까? `state`, 시도 횟수, lease, 다음 재시도 시각, wallet 상태와 증거 필드의 조합을 하나의 상태 증명으로 검증한 이유를 설명해 보세요.

꼬리 질문:

- `APPLIED` 상태와 `appliedAt` 존재 여부가 정확히 대응해야 하는 이유는 무엇입니까?
- active 상태에서 lease가 잡혀 있을 때 `nextRetryAt`을 함께 허용하면 어떤 모호함이 생깁니까?
- queue sequence와 queued 시각이 둘 중 하나만 존재하는 응답을 왜 거부해야 합니까?
- wallet `BLOCKED`, `APPLIED`, `REJECTED`가 각각 요구하거나 금지해야 할 증거는 무엇입니까?
- `EXHAUSTED` 상태에서 error code가 필요하고 wallet 상태가 없어야 하는 이유는 무엇입니까?
- 재시도 영수증의 `QUEUED`와 `REPLAY`를 어떻게 구분하며, 요청 idempotency key를 왜 다시 대조합니까?
- 후보 상태에서 `accepted == (state == ACCEPTED)`와 pending/decision 시각 대응을 확인하는 이유는 무엇입니까?
- 이처럼 nullable field가 많은 모델을 더 강한 합 타입이나 상태별 타입으로 바꿀 때의 장단점은 무엇입니까?

### 30초 모범 답변

이 응답은 필드 집합이 아니라 상태에 대한 증거이므로 교차 필드 조건을 검증해야 합니다. 요청 ID, 시각 순서, 시도 횟수 범위를 먼저 확인하고, active·terminal 상태마다 lease와 다음 재시도 시각의 허용 조합을 제한합니다. wallet 상태가 있으면 queue·operation group·적용 시각 같은 증거가 해당 상태와 일치해야 하며, `APPLIED`와 `appliedAt`도 양방향으로 대응해야 합니다. 재시도 영수증은 요청 key를 대조하고 `QUEUED`일 때만 새 일정과 초기 시도 상태를 요구합니다. 불가능하거나 불완전한 2xx 응답은 계약 위반으로 거부합니다.

### 답변 핵심 키워드

`state proof` · `cross-field invariant` · `temporal ordering` · `retry budget` · `lease exclusivity` · `nullable evidence` · `queued/applied correlation` · `idempotency correlation` · `QUEUED vs REPLAY` · `impossible state`

### 백지 구현

#### 구현 목표

실제 19개 필드를 모두 재현하지 않고, 정산 개정 lifecycle의 핵심인 schedule·lease·wallet 증거를 보존한 축소 모델을 검증한다. 25~30분 문제다.

#### 인터페이스 또는 함수 시그니처

```java
public enum RevisionState {
  PENDING,
  BLOCKED,
  EXHAUSTED,
  APPLIED,
  REJECTED
}

public enum WalletStatus {
  BLOCKED,
  APPLIED,
  REJECTED
}

public record RevisionProof(
    UUID revisionId,
    RevisionState state,
    int attemptCount,
    Instant nextRetryAt,
    String lastErrorCode,
    Instant leaseUntil,
    WalletStatus walletStatus,
    Long walletQueueSequence,
    UUID walletOperationGroupId,
    Instant walletQueuedAt,
    Instant walletAppliedAt,
    Instant walletNextAttemptAt,
    Instant createdAt,
    Instant updatedAt,
    Instant appliedAt) {}

public final class RevisionProofVerifier {
  public static RevisionProof verify(UUID requestedRevisionId, RevisionProof proof) {
    // 직접 구현
  }
}
```

면접 시간이 10~15분으로 제한되면 다음 영수증 검증만 구현한다.

```java
public enum RetryOutcome {
  QUEUED,
  REPLAY
}

public record RetryReceipt(
    UUID idempotencyKey,
    RetryOutcome outcome,
    RevisionState revisionState,
    Integer attemptCount,
    Instant nextRetryAt) {}

public final class RetryReceiptVerifier {
  public static RetryReceipt verify(UUID requestedKey, RetryReceipt receipt) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 개정 문제
  - 입력: 요청한 revision ID와 하위 서비스의 축소된 상태 증거
  - 출력: 유효하면 같은 증거 반환, 불가능하거나 불완전하면 계약 위반 예외
- 영수증 문제
  - 입력: 요청에 사용한 idempotency key와 재시도 영수증
  - 출력: 유효하면 같은 영수증 반환, 상관관계나 상태 조합이 틀리면 계약 위반 예외

#### 반드시 만족해야 할 조건

공통 조건:

- proof와 핵심 식별자·state·created/updated 시각은 null이 아니어야 한다.
- 응답 revision ID는 요청 ID와 같아야 한다.
- `attemptCount`는 `0..12` 범위여야 한다.
- `updatedAt`은 `createdAt`보다 빠를 수 없다.
- `APPLIED` 상태인 것과 `appliedAt`이 존재하는 것은 정확히 동치여야 한다.
- `appliedAt`은 `createdAt`보다 빠를 수 없다.

schedule·lease 조건:

- `leaseUntil`이 존재하면 state는 `PENDING` 또는 `BLOCKED`여야 한다.
- lease가 존재하는 동안 `nextRetryAt`은 없어야 한다.
- lease가 없는 `PENDING`은 retry budget이 남아 있고 `nextRetryAt`이 있어야 한다.
- lease가 없는 `BLOCKED`는 다음 일정이 있거나, wallet이 blocked이고 error code가 있는 명시적 중단 증거가 있어야 한다.
- `EXHAUSTED`, `APPLIED`, `REJECTED`에는 `nextRetryAt`이 없어야 한다.
- `EXHAUSTED`에는 error code가 필요하고 wallet 상태는 없어야 한다.
- `REJECTED`에는 wallet rejection 또는 error code 중 적어도 하나의 거절 증거가 있어야 한다.

wallet 증거 조건:

- `walletQueueSequence`와 `walletQueuedAt`은 함께 존재하거나 함께 없어야 한다.
- queue sequence가 있으면 양수여야 한다.
- wallet 상태가 없으면 queue·operation group·wallet 적용 시각·wallet 다음 시각도 모두 없어야 한다.
- wallet `BLOCKED`는 revision이 active 상태여야 하고, queue 증거와 `walletNextAttemptAt`이 필요하며 operation group과 적용 시각은 없어야 한다.
- wallet `APPLIED`는 revision이 `APPLIED`여야 하고 operation group과 적용 시각이 필요하며 wallet 다음 시각은 없어야 한다.
- queued 시각이 있다면 wallet 적용 시각은 그보다 빠를 수 없다.
- wallet `REJECTED`는 revision이 `REJECTED`여야 하고 queue·operation group·wallet 적용 시각·wallet 다음 시각이 없어야 한다.

짧은 영수증 문제 조건:

- 영수증과 모든 필수 필드는 null이 아니어야 한다.
- 영수증 key는 요청 key와 정확히 같아야 한다.
- 시도 횟수는 `0..12` 범위여야 한다.
- `QUEUED`이면 시도 횟수는 0이고, state는 `PENDING` 또는 `BLOCKED`이며, `nextRetryAt`이 있어야 한다.
- `REPLAY`는 이미 존재하는 결정의 재표현이므로 `QUEUED` 전용 조건을 강제하지 않는다.

#### 경계 조건

- 시도 횟수 0, 11, 12, 13
- `updatedAt == createdAt`
- `appliedAt == createdAt`
- active state + lease + next retry가 동시에 존재
- queue sequence만 있거나 queued 시각만 있는 경우
- wallet blocked인데 operation group이 존재하는 경우
- wallet applied인데 revision state가 blocked인 경우
- wallet applied 시각이 queued 시각보다 빠른 경우
- exhausted인데 wallet 상태가 남아 있는 경우
- rejected인데 거절 증거가 전혀 없는 경우
- 요청 key와 다른 `QUEUED`/`REPLAY` 영수증

#### 실패 조건

- 요청·응답 식별자 불일치
- retry budget 범위 위반
- 시각 역전
- state와 nullable 필드의 모순
- schedule과 lease의 중복 소유권 표현
- 불완전하거나 불가능한 wallet 증거
- 재시도 영수증의 key·outcome·state 조합 불일치

#### 필요한 제약

- 검증은 순수 함수로 작성하고 입력을 수정하지 않는다.
- 검증 비용은 필드 수에 선형이며 사실상 `O(1)`이어야 한다.
- boolean 식 하나에 모든 조건을 몰아넣기보다 schedule, wallet, 공통 조건을 이름 있는 단위로 나눠 설명 가능하게 만든다.
- 예외 메시지에 전체 하위 응답이나 민감한 원문을 넣지 않는다.

### 구현 후 자가 검증

- [ ] 요청 revision ID와 다른 응답을 거부한다.
- [ ] `APPLIED`와 `appliedAt`의 양방향 대응을 확인한다.
- [ ] 생성·수정·적용·wallet 시각의 역전을 거부한다.
- [ ] lease와 next retry가 동시에 존재하지 않는다.
- [ ] active 상태와 terminal 상태의 schedule 규칙이 다르게 적용된다.
- [ ] retry budget의 12/13 경계를 의도대로 처리한다.
- [ ] queue sequence와 queued 시각이 쌍으로 검증된다.
- [ ] wallet blocked/applied/rejected 각각의 필수·금지 증거를 확인한다.
- [ ] exhausted와 rejected의 error/wallet 증거 차이를 확인한다.
- [ ] `QUEUED` 영수증이 요청 key, 초기 시도 횟수, active state, 다음 시각을 모두 요구한다.
- [ ] `REPLAY`에 `QUEUED` 전용 조건을 잘못 적용하지 않는다.
- [ ] 검증 로직이 공통·schedule·wallet 단위로 설명 가능하다.

### 구현 후 설명할 것

1. nullable 필드를 각각 보지 않고 상태별 허용 조합으로 검증한 이유
2. lease와 예약 시각을 동시에 허용하지 않은 이유
3. wallet queue·적용 증거를 revision 상태와 연결한 이유
4. `QUEUED`와 `REPLAY` 영수증에 서로 다른 조건을 적용한 이유
5. 하나의 큰 record 대신 상태별 타입으로 모델링할 때 얻는 안전성과 직렬화 호환성 trade-off

### 원본 확인 위치

- Thread 20
- `feat(settlement): model revision evidence`
- `feat(settlement): verify revision lifecycle proof`
- `test(settlement): accept valid revision lifecycle proofs`
- `test(settlement): reject invalid revision lifecycle proofs`
- `test(settlement): accept valid revision wallet proofs`
- `test(settlement): reject invalid revision wallet proofs`
- `feat(settlement): verify revision retry receipts`
- `feat(settlement): retry paused revisions`
- `SettlementRevisionView`
- `SettlementRevisionProof`
- `SettlementRetryReceipt`
- `SettlementClient.retryRevision`
- `SettlementRevisionCommandController`
- 관련 Thread: 19, 08, 05, 07
- Thread 19 통합 위치
  - `feat(settlement): model candidate evidence`
  - `feat(settlement): verify candidate decisions`
  - `SettlementCandidateView`
  - `SettlementCandidateReceipt`
  - `SettlementRejectionPayload`

---

<a id="W15"></a>
## [Thread 14 / `feat(audit): expose filtered audit search`; `test(audit): page filtered actions newest first`] 안정적이고 제한된 감사 검색

### 면접 질문

감사 로그를 시간과 actor로 검색하는 endpoint에서 정렬을 `startedAt DESC` 하나만 두면 어떤 문제가 생깁니까? 시간 구간, page/size 정규화, 결정적 tie-breaker, offset pagination의 비용을 함께 설명해 보세요.

꼬리 질문:

- 조회 구간을 `[from, to)` 반개구간으로 두면 연속 구간 처리에 어떤 이점이 있습니까?
- 같은 `startedAt`을 가진 여러 행에서 `actionId DESC`를 두 번째 정렬 키로 둔 이유는 무엇입니까?
- 음수 page와 0 이하 size를 오류로 볼지 기본값으로 정규화할지 어떤 trade-off가 있습니까?
- 최대 page size를 두지 않으면 DB와 애플리케이션에 어떤 부하가 생깁니까?
- actor 문자열을 parameter binding으로 처리해야 하는 이유는 무엇입니까?
- 깊은 offset 페이지의 성능이 나빠지는 이유와 keyset pagination으로 바꿀 조건은 무엇입니까?
- 현재 시각을 함수 내부에서 직접 읽는 것과 인자로 받는 것의 테스트 차이는 무엇입니까?
- p95/p99 latency 목표와 오류율 목표를 함께 봐야 하는 이유는 무엇입니까?

### 30초 모범 답변

검색 조건은 기본적으로 `[from, to)`로 만들어 인접 구간이 겹치지 않게 하고, actor는 정규화한 뒤 parameter binding으로 exact match합니다. page는 0 이상, size는 기본 20·최대 200으로 제한합니다. 정렬은 `startedAt DESC, actionId DESC`처럼 유일한 tie-breaker까지 둬야 같은 시각의 행이 페이지 사이에서 흔들리지 않습니다. offset 방식은 단순하지만 깊은 페이지 비용과 동시 삽입에 약하므로, 대량 데이터나 연속 탐색이 중요해지면 마지막 정렬 키를 cursor로 쓰는 keyset pagination을 고려합니다.

### 답변 핵심 키워드

`half-open interval` · `bounded page size` · `deterministic ordering` · `tie-breaker` · `parameter binding` · `offset pagination` · `deep page cost` · `keyset pagination` · `clock injection` · `p95/p99`

### 백지 구현

#### 구현 목표

HTTP query parameter를 DB 검색에 사용할 수 있는 정규화된 계획으로 바꾼다. DB나 Spring 클래스는 사용하지 않고 순수 값 객체만 만든다.

#### 인터페이스 또는 함수 시그니처

```java
public record SortKey(String field, boolean descending) {}

public record AuditSearchPlan(
    Instant fromInclusive,
    Instant toExclusive,
    String actor,
    int page,
    int size,
    List<SortKey> sort) {}

public final class AuditSearchNormalizer {
  public static AuditSearchPlan normalize(
      Instant from,
      Instant to,
      String actor,
      int page,
      int size,
      Instant now) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 선택적 from/to/actor, page, size, 주입된 현재 시각
- 출력: 유효한 반개구간, 정규화된 actor, 제한된 page/size, 결정적 정렬을 가진 검색 계획

#### 반드시 만족해야 할 조건

- `from == null`이면 `Instant.EPOCH`을 사용한다.
- `to == null`이면 주입된 `now`를 사용한다.
- lower bound는 upper bound보다 반드시 빨라야 한다.
- actor가 null 또는 blank면 필터 없음으로 표현한다.
- actor가 있으면 앞뒤 공백을 제거한 exact 값으로 표현한다.
- page가 음수면 0으로 정규화한다.
- size가 1 미만이면 기본값 20을 사용한다.
- size는 최대 200을 넘을 수 없다.
- 정렬은 `startedAt DESC`, 이어서 `actionId DESC`를 정확히 포함한다.
- 시간 의미는 `startedAt >= from` 그리고 `startedAt < to`다.

#### 경계 조건

- from/to 모두 null
- `from == to`
- `from > to`
- actor가 null, 빈 문자열, 공백만 있는 문자열, 양끝 공백이 있는 값
- page가 음수·0·큰 양수
- size가 음수·0·1·20·200·201
- 여러 행의 `startedAt`이 같은 경우
- `now == Instant.EPOCH`

#### 실패 조건

- `now`가 null
- 정규화된 lower bound가 upper bound보다 빠르지 않음
- 지원하지 않는 정렬 필드나 방향이 계획에 들어감

#### 필요한 제약

- 순수 함수로 작성해 실제 clock이나 DB에 접근하지 않는다.
- 입력 문자열을 변경하지 않는다.
- 정규화는 `O(actorLength)` 이하여야 한다.
- SQL을 문자열 연결로 만들지 않는다. 실제 repository에서는 parameter binding을 사용한다고 설명할 수 있어야 한다.

### 구현 후 자가 검증

- [ ] 기본 구간이 epoch부터 주입된 now까지다.
- [ ] `from == to`와 역전 구간을 거부한다.
- [ ] upper bound가 exclusive임을 테스트한다.
- [ ] blank actor는 필터 없음, 유효 actor는 trim된 exact 값이다.
- [ ] 음수 page는 0이 된다.
- [ ] 0 이하 size는 20, 201 이상은 200으로 제한된다.
- [ ] 두 정렬 키의 순서와 방향이 고정된다.
- [ ] 같은 startedAt 행도 actionId로 결정적 순서를 가진다.
- [ ] 함수가 시스템 현재 시각에 의존하지 않는다.
- [ ] 깊은 offset 페이지의 비용을 코드 밖 설계 설명으로 다룰 수 있다.

### 구현 후 설명할 것

1. 시간 범위를 반개구간으로 선택한 이유
2. action ID를 tie-breaker로 추가한 이유
3. page/size를 오류 대신 정규화한 부분과 그 trade-off
4. offset pagination을 유지한 이유와 keyset으로 전환할 조건
5. 최대 size와 latency percentile을 운영 부하 경계로 본 이유

### 원본 확인 위치

- Thread 14
- `feat(audit): query actions by actor and time`
- `feat(audit): expose lifecycle read models`
- `feat(audit): expose filtered audit search`
- `test(audit): page filtered actions newest first`
- `feat(audit): look up actions by identifier`
- `test(load): add audit read fixture`
- `test(load): prevent persistent load evidence`
- `AuditLogRepository.search`
- `AuditLogController`
- `AuditLogView`
- `OffsetPage`
- `load-test/scenarios/audit-read.js`
- 관련 Thread: 10

---

<a id="W16"></a>
## [Thread 15 / `feat(logging): emit redacted structured events`; `test(logging): redact structured secrets`] 비밀값 안전한 구조화 로그

### 면접 질문

애플리케이션이 비밀값을 직접 로그하지 않도록 코딩 규칙을 두었는데도, 최종 logging pipeline에 redaction을 추가한 이유는 무엇입니까? 메시지뿐 아니라 stack trace와 MDC field까지 고려해야 하는 이유, 그리고 regex redaction만으로 완전한 보안을 보장할 수 없는 이유를 설명해 보세요.

꼬리 질문:

- `Authorization: Bearer ...`, `Idempotency-Key=...`, `X-API-Key`, `password`, `token`처럼 표기가 다양한 값을 어떻게 다룹니까?
- stack trace의 exception message가 secret을 포함할 수 있는 경로는 무엇입니까?
- 임의 MDC key를 모두 JSON field로 내보내지 않고 allowlist된 field만 노출한 이유는 무엇입니까?
- redaction 순서 때문에 label 기반 치환과 bare Bearer 치환이 충돌할 수 있습니까?
- 줄바꿈·comma·semicolon 같은 경계를 고려하지 않으면 어떤 과다 또는 과소 치환이 생깁니까?
- secret 자체를 hash해서 로그에 남기는 방법은 언제 위험합니까?
- idempotency key도 왜 운영상 민감할 수 있습니까?
- 구조화 로그 테스트에서 출력 JSON 전체에 원래 secret이 없는지 확인하는 것이 중요한 이유는 무엇입니까?

### 30초 모범 답변

비밀값은 정상 로그 호출뿐 아니라 예외 메시지, 라이브러리 출력, MDC를 통해 새어 나올 수 있으므로 최종 encoder 직전에도 방어합니다. 이 구현은 formatted message와 stack trace에 label 기반 secret과 bare Bearer token을 치환하고, JSON에는 trace·span·action ID처럼 승인된 MDC field만 넣습니다. 다만 regex는 알지 못하는 label이나 인코딩된 secret을 놓칠 수 있으므로 최종 방어선이지 유일한 통제는 아닙니다. 상류에서는 secret을 로그 인자로 전달하지 않고, credential 객체의 문자열 표현도 redacted하며, 테스트는 원문 secret이 전체 JSON 어디에도 없는지 확인해야 합니다.

### 답변 핵심 키워드

`defense in depth` · `formatted message` · `stack trace` · `MDC allowlist` · `labelled secret` · `Bearer token` · `redaction boundary` · `false positive/negative` · `no raw secret` · `final encoder`

### 백지 구현

#### 구현 목표

한 줄 또는 여러 줄의 로그 문자열에서 알려진 credential label 뒤의 값과 독립적인 Bearer token을 치환한다. 실제 정규식은 직접 설계한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class SecretRedactor {
  public static String redact(String input) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: formatted log message 또는 stack trace 문자열
- 출력: secret 값이 `[REDACTED]`로 치환된 문자열
- null 입력은 빈 문자열로 반환한다.

#### 반드시 만족해야 할 조건

다음 label은 대소문자를 구분하지 않고 인식해야 한다.

- `authorization`
- `idempotency-key`와 underscore/공백 변형
- `x-internal-api-key`와 separator 변형
- `x-api-key`와 separator 변형
- `api-key` 또는 `apiKey`에 해당하는 변형
- `password`
- `token`

추가 조건:

- label과 값 사이의 `:` 또는 `=`를 처리한다.
- 값 앞의 선택적 `Bearer` 접두어를 처리한다.
- double-quoted, single-quoted, unquoted 값을 처리한다.
- unquoted 값은 줄바꿈, comma, semicolon 같은 명확한 경계 너머까지 먹지 않는다.
- label 없이 등장하는 `Bearer <value>`도 치환한다.
- label과 separator는 가능하면 보존하고 값만 `[REDACTED]`로 바꾼다.
- 이미 redacted된 문자열을 다시 처리해도 secret이 복원되거나 출력이 비정상적으로 늘어나지 않아야 한다.
- 입력 원문을 별도 로그나 예외 메시지에 포함하지 않는다.

#### 경계 조건

- null과 빈 문자열
- label의 대소문자 변형
- dash, underscore, 공백 separator 변형
- 값이 따옴표로 둘러싸인 경우
- comma/semicolon 뒤에 일반 메시지가 이어지는 경우
- 여러 secret이 한 줄에 존재하는 경우
- stack trace 여러 줄에 secret이 존재하는 경우
- bare Bearer token
- `[REDACTED]`가 이미 들어 있는 입력
- secret label처럼 보이지만 값이 없는 경우

#### 실패 조건

- 알려진 label의 값이 출력에 그대로 남음
- bare Bearer 값이 남음
- secret 뒤의 일반 로그 내용까지 과도하게 제거함
- 줄 경계를 넘어 다른 stack frame을 함께 제거함
- null 처리 중 예외 발생

#### 필요한 제약

- 입력 길이에 대해 선형에 가까운 시간 복잡도를 목표로 한다.
- catastrophic backtracking이 가능한 정규식을 피한다.
- redaction 함수는 상태를 가지지 않고 thread-safe 해야 한다.
- regex만으로 모든 secret을 찾을 수 있다고 가정하지 않는다.

### 구현 후 자가 검증

- [ ] 각 label의 colon/equal 형식이 치환된다.
- [ ] 대소문자와 separator 변형을 처리한다.
- [ ] quoted/unquoted 값이 모두 치환된다.
- [ ] bare Bearer token이 치환된다.
- [ ] 한 줄의 여러 secret을 모두 제거한다.
- [ ] stack trace 여러 줄에서도 원문 secret이 남지 않는다.
- [ ] comma·semicolon·줄바꿈 뒤 일반 텍스트는 보존된다.
- [ ] null 입력은 빈 문자열이다.
- [ ] 이미 redacted된 입력에 대해 안정적이다.
- [ ] 긴 비정상 입력에서 실행 시간이 급격히 증가하지 않는다.
- [ ] 출력 JSON을 가정한 문자열 전체 검색에서 fixture secret이 하나도 발견되지 않는다.

### 구현 후 설명할 것

1. 상류 코딩 규칙 외에 encoder 단계 redaction을 둔 이유
2. message와 stack trace를 모두 처리한 이유
3. 임의 MDC field 대신 allowlist를 사용한 이유
4. 정규식의 false positive·false negative와 성능 trade-off
5. redaction을 최종 방어선으로만 보고 credential 최소 노출을 별도로 적용한 이유

### 원본 확인 위치

- Thread 15
- `feat(logging): emit redacted structured events`
- `test(logging): pin structured logger levels`
- `test(logging): redact structured secrets`
- `RedactedEventJsonProvider`
- `logback-spring.xml`
- `StructuredLoggingTest`
- 관련 Thread: 04, 05, 06, 13
