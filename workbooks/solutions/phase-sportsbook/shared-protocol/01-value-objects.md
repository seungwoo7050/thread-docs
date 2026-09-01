# 값 객체와 수치 경계 면접 워크북

이 문서는 통화 금액과 배당처럼 작은 타입 안에 정확성, 표현 규칙, 실패 조건을 밀어 넣는 설계를 다룬다. 두 문제 모두 10~30분 백지 구현과 설계 설명에 적합하다.

<a id="i01"></a>
## [Thread 03 / `feat(money): define monetary amounts`] 통화 안전 금액 값 객체

### 면접 질문

`Money`를 `long amount` 하나가 아니라 `long` minor unit과 `Currency`의 조합으로 만든 이유는 무엇인가요? 덧셈·비교·곱셈에서 반드시 막아야 하는 실패는 무엇인가요?

꼬리 질문:

- 음수 금액을 값 객체 단계에서 금지하지 않은 이유는 무엇인가요?
- `BigDecimal` 대신 정수 minor unit을 쓰면 얻는 이점과 잃는 유연성은 무엇인가요?
- `isZero()`, `isPositive()` 같은 메서드가 JSON 필드로 새어 나가면 왜 계약 문제가 되나요?
- `Long.MAX_VALUE + 1`을 조용히 wrap-around시키면 원장이나 정산에서 어떤 문제가 생기나요?

### 30초 모범 답변

금액은 부동소수점 오차를 피하기 위해 통화의 minor unit을 `long`으로 저장하고, 통화를 타입에 함께 넣어 서로 다른 통화의 산술과 비교를 차단했습니다. 음수는 debit·credit 원장 표현에 필요하므로 구조적으로 허용하되, 잔액이 음수가 되면 안 된다는 정책은 wallet 서비스가 책임집니다. 산술은 overflow를 조용히 허용하지 않고 즉시 실패해야 하며, JSON 계약은 `amount`와 `currency`만 노출되도록 고정합니다.

### 답변 핵심 키워드

minor unit, 통화 판별자, cross-currency 차단, exact arithmetic, overflow fail-fast, immutable value, 원장 부호, 구조와 정책 분리, JSON shape

### 백지 구현

**구현 목표**

통화가 다른 값끼리의 산술·비교를 거부하고, 정수 overflow를 숨기지 않는 불변 금액 타입을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public enum Currency {
  KRW,
  USD
}

public record Money(long amount, Currency currency) implements Comparable<Money> {
  public Money add(Money other) {
    // 직접 구현
  }

  public Money subtract(Money other) {
    // 직접 구현
  }

  public Money multiply(long multiplier) {
    // 직접 구현
  }

  public Money negate() {
    // 직접 구현
  }

  @Override
  public int compareTo(Money other) {
    // 직접 구현
  }

  public boolean isZero() {
    // 직접 구현
  }

  public boolean isPositive() {
    // 직접 구현
  }

  public boolean isNegative() {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: `long` minor unit, `Currency`, 다른 `Money`, 정수 multiplier
- 출력: 새 `Money`, 비교 결과, 부호 판정
- 모든 연산은 원본 객체를 변경하지 않는다.

**반드시 만족해야 할 조건**

- `currency`는 `null`일 수 없다.
- 덧셈·뺄셈·비교는 통화가 같을 때만 가능하다.
- 산술 overflow와 `Long.MIN_VALUE` negate는 명시적 실패가 되어야 한다.
- 결과 통화는 피연산자의 통화를 유지한다.
- 음수와 0은 값 객체가 표현할 수 있다.
- JSON 직렬화 결과에는 금액과 통화 외의 계산형 프로퍼티가 포함되지 않아야 한다.

**경계 조건**

- `0`, `-1`, `Long.MAX_VALUE`, `Long.MIN_VALUE`
- 같은 금액이지만 통화가 다른 경우
- multiplier가 `0`, 음수, 매우 큰 값인 경우
- `null` 피연산자와 `null` 통화
- helper 메서드가 bean getter로 인식되는 직렬화 설정

**실패 조건**

- 통화 불일치
- 정수 범위 초과
- 필수 값 누락
- JSON wire shape 변경

**필요한 제약**

- balance, payout, stake 양수 여부 같은 서비스 정책을 이 타입에 넣지 않는다.
- 시간·공간 복잡도는 각 연산 `O(1)`이어야 한다.
- 예외를 삼키거나 포화 연산으로 바꾸지 않는다.

### 구현 후 자가 검증

- [ ] 같은 통화의 덧셈·뺄셈·곱셈·부호 반전이 새 값을 반환한다.
- [ ] 서로 다른 통화의 덧셈과 비교가 실패한다.
- [ ] `Long.MAX_VALUE + 1`, 큰 수의 곱셈, `Long.MIN_VALUE` 반전이 조용히 wrap-around되지 않는다.
- [ ] 0·양수·음수 helper가 amount와 일치한다.
- [ ] 음수 금액 자체는 생성할 수 있다.
- [ ] `currency == null`과 `other == null`의 실패가 예측 가능하다.
- [ ] JSON에는 `amount`, `currency`만 존재한다.
- [ ] 객체가 불변이며 연산 후 원본 값이 바뀌지 않는다.
- [ ] 금액 객체가 wallet의 잔액 정책까지 떠맡지 않는다.

### 구현 후 설명할 것

1. 정수 minor unit이 부동소수점·소수 scale 문제를 어떻게 줄이는지
2. cross-currency 검사를 모든 이항 연산에 두는 이유
3. exact arithmetic과 wrap-around 허용 사이의 trade-off
4. 음수 허용과 "잔액은 음수가 아니어야 한다" 정책을 분리한 이유
5. helper 메서드와 JSON wire shape를 분리한 방법

### 원본 확인 위치

- Thread 03
- 커밋: `feat(money): define supported currencies`
- 커밋: `feat(money): define monetary amounts`
- 커밋: `test(money): verify arithmetic and currency safety`
- 커밋: `test(money): verify monetary JSON shape`
- 파일: `src/main/java/com/sportsbook/protocol/value/Currency.java`
- 파일: `src/main/java/com/sportsbook/protocol/value/Money.java`
- 클래스: `Currency`, `Money`
- 함수: `Money.add`, `Money.subtract`, `Money.multiply`, `Money.negate`, `Money.compareTo`
- 테스트: `CurrencyTest`, `MoneyArithmeticTest`, `MoneyJsonTest`
- 관련 Thread: 10, 13, 15, 16

---

<a id="i02"></a>
## [Thread 04 / `feat(odds): define normalized decimal odds` · `feat(odds): convert display formats`] 배당 정규화와 표시 변환

### 면접 질문

`BigDecimal` 기반 배당에서 `1.85`와 `1.8500`을 같은 값으로 취급하면서 `equals`와 `hashCode` 계약을 지키려면 어떻게 해야 하나요? decimal, American, fractional 표현을 오갈 때 반올림과 유효 범위는 어디에서 관리해야 하나요?

꼬리 질문:

- 팩터리에서 scale을 4로 고정해도 생성자를 직접 호출할 수 있다면 무엇을 더 고려해야 하나요?
- decimal odds `1.0000`은 값 객체에서 허용되지만 American 변환 공식의 분모가 0이 될 수 있습니다. 이 경계를 어떻게 명시하겠습니까?
- fractional odds를 기약분수로 만들 때 어떤 자료구조와 알고리즘을 쓰나요?
- American odds `+99`, `-99`, `0`을 거부해야 하는 이유는 무엇인가요?
- 표시 변환을 핵심 저장 표현과 분리하면 어떤 장점이 있나요?

### 30초 모범 답변

decimal odds를 기준 표현으로 두고 scale과 반올림 규칙을 한곳에서 정규화하면 서비스마다 값이 달라지는 일을 막을 수 있습니다. `BigDecimal.equals`는 scale까지 비교하므로 숫자 비교 기반 동등성을 쓰면 hash도 같은 정규화 규칙으로 계산해야 합니다. American과 fractional은 표시용 변환이므로 유효 범위, 0으로 나누는 경계, 반올림을 명시하고, 원본 decimal 값을 손실 없이 유지하는 것이 핵심입니다.

### 답변 핵심 키워드

BigDecimal scale, HALF_EVEN, compareTo equality, equals/hashCode 계약, 기준 표현, American formula, gcd, 기약분수, 1.0000 경계, 표시 변환, 유효 범위

### 백지 구현

**구현 목표**

scale이 다른 동일 배당을 같은 값으로 비교하고, decimal 배당과 American·fractional 표시를 일관된 규칙으로 변환하는 불변 타입을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public record Odds(BigDecimal decimal) {
  public static final int SCALE = 4;

  public static Odds ofDecimal(BigDecimal value) {
    // 직접 구현
  }

  public static Odds ofDecimal(String value) {
    // 직접 구현
  }

  public String toAmerican() {
    // 직접 구현
  }

  public String toFractional() {
    // 직접 구현
  }

  public static Odds ofAmerican(int american) {
    // 직접 구현
  }

  @Override
  public boolean equals(Object other) {
    // 직접 구현
  }

  @Override
  public int hashCode() {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: decimal 문자열 또는 `BigDecimal`, 정수 American odds
- 출력: scale 4 decimal odds, `+150`/`-200` 형식, `17/20` 같은 기약분수
- 모든 변환은 고정된 반올림 규칙을 사용한다.

**반드시 만족해야 할 조건**

- decimal odds는 `1.00` 미만일 수 없다.
- 팩터리 결과는 scale 4로 정규화한다.
- 숫자가 같은 `1.85`와 `1.8500`은 동등하고 hash도 같아야 한다.
- decimal 2.0 이상과 미만의 American 변환 공식을 구분한다.
- fractional 결과는 최대공약수로 약분한다.
- American odds 절댓값 100 미만은 거부한다.
- decimal `1.0000`의 표시 변환 동작을 명시적으로 정의하고, 암묵적 divide-by-zero에 맡기지 않는다.
- JSON round trip 후 숫자 동등성이 유지되어야 한다.

**경계 조건**

- `1.0000`, `1.99995`, `2.0000`, 매우 큰 배당
- scale이 0, 2, 4, 8인 같은 숫자
- 반올림 경계와 HALF_EVEN 결과
- `+100`, `-100`, `+99`, `-99`, `0`
- fractional numerator가 0인 경우
- 직접 생성자 호출과 정규화 팩터리 호출의 차이

**실패 조건**

- decimal odds가 1 미만
- American odds 유효 범위 위반
- 0으로 나누기 또는 비정의 표시 결과
- `equals`와 `hashCode` 불일치
- 반올림 규칙이 호출 경로마다 달라짐

**필요한 제약**

- core 저장 표현은 decimal로 유지한다.
- 표시 변환이 가격 source나 drift 허용 정책을 소유하지 않는다.
- 문자열을 `double`로 우회 변환하지 않는다.
- 고정 scale 분수 약분은 정수화한 numerator/denominator에 대해 수행한다.

### 구현 후 자가 검증

- [ ] `1.85`, `1.850`, `1.8500`이 같은 값이고 같은 hash를 가진다.
- [ ] 팩터리가 scale 4와 동일한 반올림 규칙을 적용한다.
- [ ] decimal 1.5, 1.85, 2.0, 2.5의 American 결과를 손으로 계산해 확인했다.
- [ ] `+150`, `-200`을 decimal로 되돌렸을 때 예상 scale과 값이 나온다.
- [ ] `17/20`, `2/1`처럼 fractional 결과가 기약분수다.
- [ ] `±99`, 0이 거부된다.
- [ ] decimal `1.0000`의 American·fractional 처리 규칙이 테스트에 드러난다.
- [ ] scale이 다른 직접 생성 값에서도 equals/hashCode 계약이 깨지지 않는다.
- [ ] JSON round trip 후 값 동등성이 유지된다.
- [ ] 변환 중 `double` 정밀도 손실이 없다.

### 구현 후 설명할 것

1. `BigDecimal.equals`를 그대로 쓰지 않은 이유와 hash 정규화 방식
2. scale 4와 HALF_EVEN을 선택한 효과와 손실
3. decimal을 기준 표현으로 두고 American/fractional을 표시 표현으로 둔 이유
4. fractional 약분의 시간 복잡도와 `gcd` 사용 이유
5. decimal `1.0000`처럼 값 불변식과 표시 공식이 충돌하는 경계를 처리한 방법

### 원본 확인 위치

- Thread 04
- 커밋: `feat(odds): define normalized decimal odds`
- 커밋: `test(odds): verify decimal odds invariants`
- 커밋: `feat(odds): convert display formats`
- 커밋: `test(odds): verify American and fractional conversions`
- 파일: `src/main/java/com/sportsbook/protocol/value/Odds.java`
- 클래스: `Odds`
- 함수: `Odds.ofDecimal`, `Odds.toAmerican`, `Odds.toFractional`, `Odds.ofAmerican`
- 테스트: `OddsTest`, `OddsConversionTest`
- 관련 Thread: 12, 13, 15, 16
