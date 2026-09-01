# 알고리즘·정산 금액·Wallet 계약 면접 워크북

이 문서는 SYSTEM 조합 생성, 배당 계산, 금액 보존식, Wallet 응답 판정을 다룬다. 계산 문제는 구현 정답이 아니라 만족해야 할 invariant와 검증 기준만 제공한다.

<a id="p07"></a>
<!-- POINT:P07 -->
## P07 — [Thread 6 / `feat(resolver): expand deterministic system combinations`] K-of-N 조합의 결정적 생성

### 면접 질문

SYSTEM 베팅의 K-of-N line을 어떤 순서로 생성했으며, 순서가 결정적이어야 하는 이유는 무엇입니까? 시간·공간 복잡도도 설명해 보세요.

꼬리 질문:

- 모든 조합을 메모리에 올리지 않고 스트리밍하면 어떤 장단점이 있습니까?
- 입력 selection 순서를 먼저 정렬해야 합니까?
- 재귀 구현에서 같은 mutable buffer를 결과에 그대로 넣으면 어떤 문제가 생깁니까?

### 30초 모범 답변

입력 selection의 상대 순서를 유지하는 lexicographic 조합 순서로 K개를 선택했습니다. 각 단계에서 다음 시작 인덱스를 넘기는 backtracking을 쓰면 중복 없이 `C(N,K)`개를 만들 수 있습니다. 결과 line 순서는 payout 합계 자체에는 영향을 주지 않지만 테스트, snapshot, 재현성, 장애 분석에 영향을 주므로 결정적이어야 합니다. 시간과 결과 공간은 모두 `O(C(N,K) × K)`이고, 탐색 stack은 `O(K)`입니다.

### 답변 핵심 키워드

backtracking, start index, lexicographic order, deterministic output, defensive copy, `C(N,K)`, output-sensitive complexity

### 백지 구현

#### 구현 목표

입력 순서를 기준으로 K개 원소의 모든 조합을 중복 없이 결정적인 순서로 반환한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class Combinations {
  public static <T> List<List<T>> choose(List<T> values, int k) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 중복되지 않는 원소 목록과 선택 개수 K
- 출력: 입력 상대 순서를 보존한 K개 조합 목록

#### 반드시 만족해야 할 조건

- 각 결과는 정확히 K개 원소를 가진다.
- 같은 인덱스 집합이 두 번 나오지 않는다.
- 결과 순서는 매 실행마다 동일하다.
- 입력의 상대 순서를 조합 내부에서 유지한다.
- 결과 목록과 각 조합은 탐색용 mutable buffer와 분리한다.
- 입력 목록을 변경하지 않는다.

#### 경계 조건

- K=1
- K=N
- N이 작은 최소 유효 입력
- 결과 수가 빠르게 증가하는 입력
- null 원소를 허용할지 거부할지 명시

#### 실패 조건

- K<1
- K>N
- null 입력 목록
- 결과에 같은 mutable list 참조를 반복 저장
- 정수 overflow가 발생하는 예상 조합 수 계산

#### 필요한 제약

- 구현 후 시간 복잡도를 결과 크기에 비례해 설명한다.
- 사전에 `C(N,K)`를 계산한다면 overflow를 감지한다.
- 정렬로 원래 계약 순서를 바꾸지 않는다.

### 구현 후 자가 검증

- N=4, K=2에서 6개 조합이 생성되는가?
- 첫 조합과 마지막 조합이 입력 순서 기준으로 예상되는가?
- K=1과 K=N이 별도 특수 처리 없이 동작하는가?
- 결과 한 조합을 수정해도 다른 조합에 영향이 없는가?
- 같은 입력으로 반복 호출해 동일한 순서가 나오는가?
- 시간·공간 복잡도가 반환 결과 크기보다 비현실적으로 커지지 않는가?

### 구현 후 설명할 것

1. 시작 인덱스로 중복 선택을 막는 원리
2. 결과 순서를 결정적으로 만든 이유
3. mutable 탐색 buffer를 복사해야 하는 이유
4. 모든 조합 materialization과 lazy iteration의 trade-off
5. 조합 수 폭증을 호출자가 통제해야 하는 이유

### 원본 확인 위치

- Thread: 6 — 베팅 line 확장과 payout 계산
- 대표 커밋: `feat(resolver): expand deterministic system combinations`
- 관련 커밋: `feat(resolver): expand single and multiple lines`, `test(resolver): verify system combination ordering`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/SettlementLineFactory.java`
- 클래스·컴포넌트: `SettlementLineFactory`, `SettlementLine`, `ResolvedSelection`
- 관련 메서드: `SettlementLineFactory.lines`
- 관련 Thread: 5, 7, 12, 19

<a id="p08"></a>
<!-- POINT:P08 -->
## P08 — [Thread 6 / `feat(resolver): calculate unit stake line payouts`] line 합산 후 한 번만 수행하는 payout 반올림

### 면접 질문

각 line의 payout을 먼저 minor unit으로 내림한 뒤 합산하지 않고, line의 정확한 값을 모두 합한 다음 최종 payout에서 한 번만 내림한 이유는 무엇입니까? `PUSH`와 `VOID` selection은 odds 계산에서 어떻게 다뤘습니까?

꼬리 질문:

- line별 반올림이 장기적으로 어떤 편향을 만듭니까?
- LOST selection이 하나 있는 line의 나머지 odds를 계속 곱할 필요가 있습니까?
- decimal 곱셈에서 overflow와 scale을 어떻게 다루겠습니까?

### 30초 모범 답변

각 SYSTEM line은 unit stake에 살아남은 WON odds를 곱하고, PUSH와 VOID는 배율 1로 유지하며, LOST가 있으면 line payout은 0입니다. line마다 내림하면 fractional remainder를 여러 번 버려 총 payout이 작아지는 편향이 생기므로, 정확한 line 금액들을 합산한 뒤 통화 minor unit에서 한 번만 `FLOOR`했습니다. 동시에 total lines와 surviving lines를 계산해 이후 Wallet movement의 총 노출액과 반환 stake를 분리할 수 있게 했습니다.

### 답변 핵심 키워드

unit stake per line, LOST zero, PUSH/VOID neutral, exact decimal accumulation, one final floor, rounding bias, surviving lines

### 백지 구현

#### 구현 목표

settlement lines와 unit stake를 받아 총 payout, surviving line 수, total line 수를 계산한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class PayoutCalculator {
  public PayoutCalculation calculate(List<SettlementLine> lines, Money unitStake) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: selection outcome과 odds를 가진 line 목록, 양수 unit stake
- 출력: 통화 minor unit의 payout, surviving line 수, total line 수

#### 반드시 만족해야 할 조건

- LOST selection을 포함한 line payout은 0이다.
- WON selection만 odds 배율에 기여한다.
- PUSH와 VOID는 배율 1로 처리한다.
- 각 line의 정확한 decimal 금액을 합한 뒤 최종 합계에만 내림을 적용한다.
- payout 통화는 unit stake 통화와 같다.
- surviving line 수는 payout에 참여한 line의 정의와 일치해야 한다.
- total line 수는 입력 line 수와 일치해야 한다.

#### 경계 조건

- 모든 line이 LOST
- 모든 selection이 VOID
- PUSH만 있는 line
- 소수 부분이 minor unit 경계 바로 아래 또는 위인 합계
- 매우 큰 odds와 line 수

#### 실패 조건

- 빈 line 목록
- 0 또는 음수 unit stake
- null selection/outcome/odds
- 음수 또는 허용 범위 밖 odds
- 곱셈·합산·minor-unit 변환 overflow

#### 필요한 제약

- 부동소수점 `double`을 사용하지 않는다.
- 반올림 위치를 코드 구조에서 명확히 드러낸다.
- 모든 intermediate 값의 통화 의미가 일관돼야 한다.

### 구현 후 자가 검증

- line별 내림과 최종 한 번 내림이 다른 예제를 만들어 차이를 확인했는가?
- LOST line이 0이고 surviving count에서 제외되는가?
- PUSH와 VOID가 odds를 0으로 만들지 않는가?
- 모든 VOID일 때 반환 stake에 해당하는 payout을 계산할 수 있는가?
- 결과 payout이 음수가 될 수 없는가?
- 큰 입력에서 overflow가 조용히 wrap되지 않는가?

### 구현 후 설명할 것

1. 반올림 시점이 금액 invariant에 미치는 영향
2. PUSH/VOID를 배율 1로 모델링한 이유
3. line payout과 전체 Wallet movement를 분리한 이유
4. decimal 타입과 minor-unit 정수 타입의 역할
5. 계산량이 `line 수 × line 길이`에 비례하는 이유

### 원본 확인 위치

- Thread: 6 — 베팅 line 확장과 payout 계산
- 대표 커밋: `feat(resolver): calculate unit stake line payouts`
- 관련 커밋: `test(resolver): verify line sums and final rounding`, `feat(resolver): classify base settlement outcomes`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/SettlementPayoutCalculator.java`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/PayoutCalculation.java`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java`
- 클래스·컴포넌트: `SettlementOutcome`, `SettlementLine`
- 관련 메서드: `SettlementPayoutCalculator.calculate`, `SettlementResolver.resolve`
- 관련 Thread: 7, 12, 14

<a id="p09"></a>
<!-- POINT:P09 -->
## P09 — [Thread 7 / `feat(settlement): enforce money plan conservation`] stake·payout 보존식과 Wallet movement 순서

### 면접 질문

정산 금액 계획에 왜 다음 두 보존식을 동시에 두었습니까?

```text
lockedRelease + lockedForfeit = committed
lockedRelease + houseProfit = payout
```

또 Wallet 호출을 release, forfeit, profit payout 순서로 나눈 이유와 재시도 안전성을 설명해 보세요.

꼬리 질문:

- `committed`와 `payout`을 하나의 식으로 합치면 무엇을 잃습니까?
- SYSTEM 베팅에서 committed는 unit stake와 어떤 관계입니까?
- 중간 Wallet 호출이 성공하고 다음 호출이 timeout되면 어떻게 복구합니까?

### 30초 모범 답변

첫 식은 잠긴 사용자 stake의 귀속을 보존하고, 둘째 식은 최종 사용자 payout의 자금 출처를 보존합니다. 반환 stake는 두 식에 공통이고, 잃은 stake는 forfeit되며, 반환 stake를 넘는 이익만 house 자금에서 지급됩니다. SYSTEM의 committed는 unit stake가 아니라 전체 line 노출액입니다. 각 movement는 0이면 생략하고 bet 기반의 안정된 idempotency key를 써서 release, forfeit, profit 순으로 재호출해도 같은 작업으로 식별되게 했습니다.

### 답변 핵심 키워드

double-entry-like conservation, committed exposure, returned stake, forfeited stake, house-funded profit, ordered movements, stable idempotency key

### 백지 구현

#### 구현 목표

unit stake, line 수, surviving line 수, 최종 payout을 받아 보존식을 만족하는 금액 계획과 실행 순서가 있는 movement 목록을 만든다.

#### 인터페이스 또는 함수 시그니처

```java
public final class SettlementMoneyPlanner {
  public PlannedMovements plan(
      UUID betId,
      SettlementAction action,
      Money unitStake,
      int totalLines,
      int survivingLines,
      Money payout) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: bet ID, 정산 동작, unit stake, total/surviving line 수, 최종 payout
- 출력: immutable money plan과 순서가 고정된 Wallet movements

#### 반드시 만족해야 할 조건

- `committed = unitStake × totalLines`다.
- `lockedRelease = unitStake × survivingLines`다.
- `lockedForfeit = committed - lockedRelease`다.
- `houseProfit = payout - lockedRelease`다.
- 모든 금액은 비음수이고 같은 통화다.
- 두 보존식이 모두 성립한다.
- movement 순서는 release → forfeit → house profit이다.
- 금액이 0인 movement는 외부 호출 대상으로 만들지 않는다.
- 각 movement key는 동일 bet와 목적에 대해 결정적이며 서로 충돌하지 않는다.
- 전체 슬립 무효화는 총 노출액을 release하고 forfeit와 profit이 0이어야 한다.

#### 경계 조건

- 전부 패배해 payout과 returned stake가 0인 경우
- 전부 PUSH/VOID라서 committed 전액이 반환되는 경우
- payout이 returned stake와 정확히 같은 경우
- house profit이 매우 큰 경우
- totalLines=1인 SINGLE/MULTIPLE

#### 실패 조건

- survivingLines<0 또는 survivingLines>totalLines
- totalLines<1
- payout이 returned stake보다 작아 house profit이 음수
- 통화 불일치
- 곱셈·뺄셈 overflow
- 동일 목적의 key가 다른 금액 계획에도 재사용됨

#### 필요한 제약

- 금액 계산과 외부 호출을 분리한다.
- plan은 생성 후 변경할 수 없어야 한다.
- key 문자열의 구체 형식보다 semantic namespace 충돌 방지가 중요하다.

### 구현 후 자가 검증

- 두 보존식을 모든 정상 사례에서 다시 계산해 확인했는가?
- SYSTEM total exposure가 unit stake만으로 축소되지 않는가?
- 전패·전부 반환·일부 승리·큰 이익 사례를 모두 통과하는가?
- 0원 movement가 생략되는가?
- movement 순서와 key가 반복 호출에서도 동일한가?
- 금액 객체의 통화와 부호가 각 중간 단계에서 검증되는가?
- overflow가 예외로 드러나는가?

### 구현 후 설명할 것

1. stake 보존식과 payout 보존식을 분리한 이유
2. returned stake가 두 식에 공통으로 등장하는 의미
3. 외부 movement 순서를 고정한 이유
4. 안정된 idempotency key가 crash recovery에 주는 효과
5. 계산 plan을 Wallet 호출 전에 내구 저장해야 하는 이유

### 원본 확인 위치

- Thread: 7 — 정산 금액 보존과 Wallet 자금 이동 순서
- 대표 커밋: `feat(settlement): enforce money plan conservation`
- 관련 커밋: `feat(resolver): split wallet settlement movements`, `feat(settlement): release locked stake first`, `feat(settlement): forfeit lost locked stake`, `feat(settlement): pay house funded profit`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java`
- 파일: `src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementMoneyPlan.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementWalletExecutor.java`
- 관련 메서드: `WalletMovementPlanner.plan`, `SettlementWalletExecutor.releaseLocked`, `SettlementWalletExecutor.forfeitLocked`, `SettlementWalletExecutor.payHouseProfit`
- 관련 Thread: 5, 6, 8, 10, 14

<a id="p10"></a>
<!-- POINT:P10 -->
## P10 — [Thread 8 / `feat(wallet): classify dependency failures`] HTTP 상태와 operation proof를 함께 판정하는 Wallet 경계

### 면접 질문

Wallet이 2xx를 반환했는데 body가 없거나 operation identity가 요청과 다르면 왜 영구 business rejection으로 처리하지 않고 재시도 가능한 실패로 봤습니까? 429, 5xx, 그 밖의 non-2xx는 어떻게 분류했습니까?

꼬리 질문:

- timeout 이후 같은 요청을 다시 보내는 것이 안전하려면 무엇이 필요합니까?
- HTTP status만 보고 "적용 완료"를 확정하면 안 되는 이유는 무엇입니까?
- connect/read timeout에 상한을 둔 이유는 무엇입니까?

### 30초 모범 답변

HTTP status는 transport 결과일 뿐 금전 작업의 정확한 증거가 아닙니다. 429와 5xx는 일시 실패로, 그 밖의 명시적 non-2xx는 안전하게 추출한 error code를 가진 영구 실패로 분류했습니다. 반면 2xx라도 body가 없거나 ID·사용자·금액·통화가 요청과 다르면 실제 효과가 불명확하므로 transient malformed success로 처리합니다. 성공 확정은 exact operation proof가 있어야 하고, 요청은 안정된 idempotency identity와 bounded connect/read timeout을 사용해야 합니다.

### 답변 핵심 키워드

transport vs business truth, transient 429/5xx, permanent problem, malformed success, exact proof, bounded timeout, idempotent retry

### 백지 구현

#### 구현 목표

HTTP status, 선택적 problem body, 선택적 operation proof를 받아 retry, permanent failure, exact success 중 하나로 판정한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class WalletResponsePolicy {
  public WalletDecision classify(
      int httpStatus,
      Optional<ProblemBody> problem,
      Optional<OperationProof> proof,
      ExpectedOperation expected) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: HTTP status, 오류 body, 성공 proof, 요청 시 기대한 operation identity와 금액
- 출력: `EXACT_SUCCESS`, `RETRY`, `PERMANENT_FAILURE`와 안전한 오류 정보

#### 반드시 만족해야 할 조건

- 429와 5xx는 retryable이다.
- 그 밖의 허용되지 않은 non-2xx는 permanent failure다.
- 오류 body가 malformed여도 dependency의 원문 body를 그대로 노출하지 않는다.
- 허용된 성공 status라도 proof가 없으면 retryable malformed success다.
- proof의 operation ID, 대상 ID, 사용자 ID, 목적, 금액, 통화가 expected와 정확히 일치해야 한다.
- status와 proof status의 허용 조합을 검증한다.
- 예상하지 못한 HTTP status는 성공으로 간주하지 않는다.
- 판단 결과에 secret, 인증 header, 전체 dependency body를 넣지 않는다.

#### 경계 조건

- 200/202 등 서로 다른 허용 성공 status
- 빈 body인 2xx
- 숫자 금액은 같지만 통화가 다른 proof
- 한 식별자만 다른 proof
- malformed problem JSON
- 429와 500, 404, 예상 밖 3xx

#### 실패 조건

- 2xx를 proof 없이 성공 처리
- mismatch proof를 성공 처리
- timeout을 semantic rejection으로 변환
- dependency 원문을 로그나 예외 메시지에 노출
- status와 proof status 결합 규칙 누락

#### 필요한 제약

- classifier는 순수 함수로 만들고 HTTP client와 분리한다.
- connect/read timeout 값은 양수이고 구성된 상한 안이어야 한다.
- retry 여부와 idempotency 안전성은 별도 조건임을 설명한다.

### 구현 후 자가 검증

- 429와 모든 5xx가 retry로 분류되는가?
- malformed 2xx가 영구 rejection으로 위장되지 않는가?
- expected field 하나씩 바꾼 proof가 모두 거부되는가?
- 오류 body가 없어도 안전한 fallback code를 만들 수 있는가?
- 예상 밖 status가 성공하지 않는가?
- 반환 메시지와 `toString`에 secret이나 dependency body가 없는가?
- timeout 상한이 설정 오류를 startup에 드러내는가?

### 구현 후 설명할 것

1. transport status와 금전 작업 증거를 분리한 이유
2. malformed success를 retryable로 본 이유
3. exact proof에서 비교해야 할 identity와 금액 필드
4. 오류 응답 redaction과 운영 진단성 사이의 trade-off
5. bounded timeout이 worker 처리량과 recovery에 미치는 영향

### 원본 확인 위치

- Thread: 8 — Wallet HTTP 계약과 작업 증거 검증
- 대표 커밋: `feat(wallet): classify dependency failures`
- 관련 커밋: `feat(wallet): bound dependency timeouts`, `feat(wallet): enforce bounded HTTP transport`, `feat(wallet): reject malformed success responses`, `feat(wallet): reject unexpected HTTP statuses`, `feat(wallet): couple adjustment statuses and proofs`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletClient.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletHttpProperties.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletHttpConfiguration.java`
- 클래스·컴포넌트: `WalletProblemDetail`, `WalletAuthenticationHeaders`
- 관련 Thread: 3, 7, 15, 18
