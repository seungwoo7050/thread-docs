# 숫자 경계·도메인 불변식·정규 식별자

이 문서는 Java와 Redis Lua 사이의 수치 경계, 입력 객체가 보장해야 하는 불변식, 조합식 노출액 계산, 멱등성 식별자, 결과 타입의 상태 불변식을 묶는다. 서로 다른 Thread에서 반복된 "경계에서 한 번 검증하고 내부에서는 신뢰한다"는 역량을 다섯 문제로 통합했다.

---

## P01. [Thread 02 / `test(numeric): verify Redis integer boundaries`] Redis Lua와 Java 사이의 정확한 정수 범위

### 면접 질문

Java의 `long`은 훨씬 큰 정수를 표현할 수 있는데, 이 서비스는 왜 금액과 집계값의 상한을 `9,007,199,254,740,991`로 제한했나요?

꼬리 질문:

- 덧셈과 곱셈에서 상한을 넘는지 결과를 계산하기 전에 어떻게 확인하겠습니까?
- Redis에 이미 상한 밖의 값이나 정수가 아닌 문자열이 저장되어 있다면 보정할까요, 실패할까요?
- JSON 숫자 대신 문자열로 수치를 전달하는 경계가 있는 이유는 무엇입니까?

### 30초 모범 답변

Redis Lua의 숫자는 배정밀도 부동소수점이라 `2^53-1`까지만 모든 정수를 정확히 표현합니다. Java에서 더 큰 `long`을 허용하면 Java에서는 맞아 보여도 Lua에서 반올림되어 금액, 합계, 비교 결과가 달라질 수 있습니다. 그래서 입력, 저장값, 스크립트 인자와 결과를 같은 정확 정수 도메인으로 제한하고, 덧셈·곱셈은 연산 전에 범위를 검사합니다. 저장값이 이 계약을 어기면 임의 보정하지 않고 실패시켜 잘못된 승인보다 가용성 저하를 선택합니다.

### 답변 핵심 키워드

`IEEE-754` · `2^53-1` · 교차 런타임 계약 · 연산 전 범위 검사 · 정밀도 손실 방지 · fail-closed · 문자열 수치 경계

### 백지 구현

#### 구현 목표

Java `long` 값을 Redis Lua의 정확 정수 도메인 안에서만 사용하도록 검증하고, 덧셈과 곱셈이 범위를 넘지 않을 때만 결과를 반환하는 유틸리티를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class ExactInteger {
  public static final long MAX = 9_007_199_254_740_991L;

  public static long requireNonNegative(long value, String name) {
    // 직접 구현
  }

  public static long requirePositive(long value, String name) {
    // 직접 구현
  }

  public static long add(long left, long right, String name) {
    // 직접 구현
  }

  public static long multiply(long left, long right, String name) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: `long` 피연산자와 오류 메시지에 사용할 이름
- 출력: 계약을 만족하는 동일 값 또는 정확한 연산 결과
- 잘못된 값: 명시적인 예외

#### 반드시 만족해야 할 조건

- 허용 범위는 `0..MAX`, 양수 요구 시 `1..MAX`다.
- `add`와 `multiply`는 Java `long` 오버플로뿐 아니라 `MAX` 초과도 거부한다.
- 잘못된 입력에 대해 포화 연산, 자동 절삭, 음수 보정, 부동소수점 변환을 하지 않는다.
- 예외 메시지에서 어느 값 또는 연산이 실패했는지 식별할 수 있어야 한다.

#### 경계 조건

- `0`, `1`, `MAX`
- `MAX + 0`, `MAX + 1`
- `0 × MAX`, `1 × MAX`
- 결과가 정확히 `MAX`인 곱셈
- 음수 피연산자

#### 실패 조건

- 입력 자체가 도메인 밖
- 덧셈 또는 곱셈 결과가 `MAX`를 초과
- 계산 도중 Java `long` 오버플로 가능성이 존재

#### 필요한 제약

- `BigInteger`, `double`, 문자열 산술 없이 구현한다.
- 연산 결과를 먼저 계산한 뒤 사후 비교하는 방식에 의존하지 않는다.
- 10~15분 안에 구현하고 경계 테스트를 작성할 수 있는 크기로 제한한다.

### 구현 후 자가 검증

- [ ] `requireNonNegative(0)`과 `requirePositive(1)`이 성공한다.
- [ ] `requirePositive(0)`, 음수, `MAX + 1`이 거부된다.
- [ ] 결과가 정확히 `MAX`인 덧셈과 곱셈은 허용된다.
- [ ] `MAX + 1`, `2 × MAX`, 큰 두 수의 곱셈이 계산 전에 거부된다.
- [ ] 실패 후 반환되는 잘못된 값이나 부분 상태가 없다.
- [ ] 모든 성공 결과가 `0..MAX` 안에 있다는 invariant가 유지된다.
- [ ] 시간 복잡도와 공간 복잡도가 모두 `O(1)`이다.

### 구현 후 설명할 것

1. Java `long` 범위가 아니라 가장 약한 실행 경계인 Redis Lua에 도메인을 맞춘 이유
2. 사후 오버플로 감지보다 연산 전 비교를 택한 이유
3. 포화 연산이나 자동 보정보다 예외를 선택한 이유
4. 수치 계약을 한 유틸리티에 집중시켰을 때 얻는 일관성과 결합도
5. 범위를 넓히려면 저장 형식과 Lua 연산 모델까지 함께 바꿔야 한다는 점

### 원본 확인 위치

- Thread 02
- 커밋: `test(numeric): verify Redis integer boundaries`
- 파일: `src/main/java/com/sportsbook/risk/policy/SafeRedisNumber.java`
- 클래스: `SafeRedisNumber`
- 관련 Thread: 06, 07, 08, 10, 12, 13

---

## P02. [Thread 02 / `feat(command): define typed risk candidates`] 유효하지 않은 후보를 경계에서 차단하는 불변 객체

### 면접 질문

위험 평가와 예약 승인에 동일하게 쓰이는 후보 객체에 어떤 invariant를 넣어야 하며, 왜 서비스 메서드마다 다시 검증하지 않았나요?

꼬리 질문:

- 선택 목록을 그대로 보관하면 어떤 문제가 생깁니까?
- 중복 선택을 허용했을 때 용량 계산과 멱등성 지문에는 어떤 영향이 있습니까?
- 생성자 검증이 API 검증을 완전히 대체할 수 있습니까?

### 30초 모범 답변

후보 객체는 사용자·베팅 식별자, 양수이면서 정확 표현 가능한 금액, 1~15개의 null 없는 고유 선택, 평가 시점을 한 번에 보장해야 합니다. 생성 시 목록을 방어적으로 복사하면 생성 뒤 외부 변경으로 invariant가 깨지는 것도 막을 수 있습니다. API 검증은 좋은 오류 응답을 만드는 역할이고, 도메인 생성자 검증은 HTTP·Kafka·테스트 등 어느 진입점에서도 잘못된 상태가 내부로 들어오지 못하게 하는 최종 방어선입니다.

### 답변 핵심 키워드

불변 객체 · 생성자 invariant · 방어적 복사 · 고유 집합 · 다중 진입점 · API 검증과 도메인 검증의 역할 분리

### 백지 구현

#### 구현 목표

예약과 진단에서 공통으로 사용할 불변 후보 객체를 작성한다. 객체가 생성된 뒤에는 내부 필드의 유효성을 다시 검사하지 않고 사용할 수 있어야 한다.

#### 인터페이스 또는 함수 시그니처

```java
public record Candidate(
    String userId,
    String betId,
    long amount,
    String currency,
    List<String> selectionIds,
    Instant evaluatedAt) {

  public Candidate {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 후보를 구성하는 식별자, 금액, 통화, 선택 목록, 평가 시각
- 출력: 모든 invariant를 만족하는 `Candidate`
- 잘못된 입력: 생성 실패

#### 반드시 만족해야 할 조건

- 모든 필수 값은 null이 아니며 식별자 문자열은 비어 있지 않다.
- 금액은 P01의 정확 정수 양수 도메인을 만족한다.
- 선택 수는 1~15다.
- 선택 목록에 null, 빈 문자열, 중복이 없다.
- 생성 후 호출자가 원본 목록을 변경해도 객체 내용은 변하지 않는다.
- 접근자가 반환한 목록을 통해 내부 상태를 변경할 수 없다.

#### 경계 조건

- 선택 1개, 15개
- 선택 0개, 16개
- 같은 선택이 인접하거나 떨어져 중복된 경우
- 수정 가능한 `ArrayList`를 입력한 뒤 외부에서 변경하는 경우
- 금액 `1`, `MAX`, `0`, `MAX + 1`

#### 실패 조건

- 누락된 식별자나 시각
- 잘못된 금액
- 선택 개수 위반, null 선택, 중복 선택

#### 필요한 제약

- 저장 순서는 보존한다.
- 내부에서 계속 검증하는 getter나 서비스 코드는 작성하지 않는다.
- 정규식으로 실제 UUID 형식까지 검증하는 것은 선택 사항이며, 구현 시 그 책임을 설명한다.

### 구현 후 자가 검증

- [ ] 정상 후보가 생성되고 입력 순서가 보존된다.
- [ ] 1개와 15개 선택은 성공하고 0개와 16개는 실패한다.
- [ ] 중복과 null 선택이 실패한다.
- [ ] 생성 후 원본 리스트를 수정해도 객체가 변하지 않는다.
- [ ] 반환된 목록을 직접 수정할 수 없거나 수정이 객체에 영향을 주지 않는다.
- [ ] 금액 invariant가 P01과 동일하다.
- [ ] 어느 성공 인스턴스에서도 후속 코드가 선택 개수나 null을 다시 확인할 필요가 없다.

### 구현 후 설명할 것

1. API 계층 검증과 도메인 생성자 검증을 둘 다 둔 이유
2. `List.copyOf`에 해당하는 방어적 복사가 필요한 이유
3. 중복 선택이 단순 입력 오류를 넘어 집계·패턴·지문을 왜곡하는 방식
4. record를 사용했을 때 얻는 값 의미와 생성자 제약
5. 생성 비용과 내부 코드 단순화 사이의 trade-off

### 원본 확인 위치

- Thread 02
- 커밋: `feat(command): define typed risk candidates`
- 파일: `src/main/java/com/sportsbook/risk/service/RiskCheckCommand.java`
- 클래스: `RiskCheckCommand`
- 관련 Thread: 04, 07, 09, 10, 13, 14

---

## P03. [Thread 13 / `feat(events): calculate accepted bet exposure`] 시스템 베팅의 정확한 조합식 노출액

### 면접 질문

`SINGLE`, `MULTIPLE`, `SYSTEM` 베팅의 노출액을 어떻게 계산했고, 조합 수와 금액의 오버플로를 어떻게 다뤘나요?

꼬리 질문:

- `SYSTEM`에서 `C(n, k)`를 계산할 때 왜 `min(k, n-k)`를 사용할 수 있습니까?
- 홀수·짝수나 조합 계산 중간값에서 나눗셈 순서가 잘못되면 어떤 문제가 생깁니까?
- 이벤트가 이미 수락된 뒤 도착했다면 노출액 계산 실패를 임의 값으로 대체해도 됩니까?

### 30초 모범 답변

단일과 복수 베팅은 이벤트 stake가 전체 노출액이고, 시스템 베팅은 stake에 `C(totalSelections, minWins)`를 곱합니다. 선택 수와 시스템 필드의 모양을 먼저 검증하고, 조합 수와 최종 곱셈은 정수 정확성을 유지하면서 매 단계 오버플로를 확인합니다. 수락 이벤트의 노출액이 모호하거나 범위를 넘으면 잘못된 금액을 Redis에 반영하는 것보다 영구 입력 실패로 분류하는 편이 안전합니다.

### 답변 핵심 키워드

조합 `nCk` · 대칭성 `k=min(k,n-k)` · 정확한 정수 나눗셈 · 중간 오버플로 · 이벤트 shape 검증 · fail-closed

### 백지 구현

#### 구현 목표

세 베팅 형태의 입력 규칙을 검증하고, 시스템 베팅의 조합 수를 정확히 계산해 전체 노출액을 반환한다.

#### 인터페이스 또는 함수 시그니처

```java
enum SlipType {
  SINGLE, MULTIPLE, SYSTEM
}

public static long exposure(
    SlipType type,
    long unitStake,
    int selectionCount,
    Integer systemMinWins,
    Integer systemTotalSelections) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 베팅 형태, 단위 stake, 실제 선택 수, 선택적인 시스템 필드
- 출력: P01의 양수 정확 정수 범위에 있는 전체 노출액
- 잘못된 shape나 범위 초과: 예외

#### 반드시 만족해야 할 조건

- 선택 수는 1~15다.
- `SINGLE`은 선택 1개이며 시스템 필드가 없어야 한다.
- `MULTIPLE`은 선택 2개 이상이며 시스템 필드가 없어야 한다.
- `SYSTEM`은 총 선택 수가 실제 선택 수와 같고 2~15이며, 최소 적중 수는 1 이상 총 선택 수 이하여야 한다.
- 시스템 노출액은 `unitStake × C(totalSelections, systemMinWins)`다.
- 조합 계산과 최종 곱셈에서 정수 손실과 상한 초과를 허용하지 않는다.

#### 경계 조건

- 최소 적중 수 0은 거절하고, 최소 적중 수가 전체 선택 수와 같은 `C(n, n)`은 허용
- `C(15, 7)`처럼 조합 수가 큰 경우
- 최소 적중 수 1과 전체 선택 수와 같은 경우
- 시스템 총 선택 수와 실제 목록 크기가 다른 경우
- 최종 결과가 정확히 `MAX` 또는 `MAX`를 초과하는 경우

#### 실패 조건

- 형태와 필드 조합 불일치
- 선택 수 범위 위반
- 조합 중간값 또는 최종 금액의 범위 초과
- null 형태 또는 누락 필드

#### 필요한 제약

- 부동소수점, 팩토리얼의 무조건적 선계산, 근삿값을 사용하지 않는다.
- 15개 선택이라는 상한은 사용해도 되지만, 경계 검증을 생략하는 근거로 쓰지 않는다.
- 정답 코드 대신 테스트 가능한 작은 함수 하나로 완성한다.

### 구현 후 자가 검증

- [ ] `SINGLE`과 `MULTIPLE`의 정상 shape가 unit stake를 반환한다.
- [ ] 비시스템 형태에 시스템 필드가 있으면 실패한다.
- [ ] 시스템 총 선택 수와 실제 선택 수가 다르면 실패한다.
- [ ] 대칭인 `C(n,k)`와 `C(n,n-k)`가 같은 값을 낸다.
- [ ] 조합 결과가 작은 알려진 사례에서 정확하다.
- [ ] 중간 계산과 최종 곱셈이 범위를 넘을 때 실패한다.
- [ ] 모든 성공 결과가 양수이며 정확 정수 상한 이하다.
- [ ] 시간 복잡도가 선택 수 상한에 대해 설명 가능하다.

### 구현 후 설명할 것

1. 팩토리얼 방식보다 단계적 조합 계산을 선택한 이유
2. 조합 대칭성을 사용해 반복 횟수와 중간값을 줄인 이유
3. 베팅 shape 검증을 계산 전에 끝내는 이유
4. 이미 수락된 이벤트라도 잘못된 노출액을 추정하지 않는 이유
5. 선택 수 상한이 바뀔 때 현재 자료형과 알고리즘이 받는 영향

### 원본 확인 위치

- Thread 13
- 커밋: `feat(events): calculate accepted bet exposure`
- 파일: `src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java`
- 클래스: `AcceptedBetExposure`
- 관련 Thread: 02, 10, 14

---

## P04. [Thread 09 / `feat(reservation): fingerprint reservation requests`] 순서 독립적이고 모호하지 않은 요청 지문

### 면접 질문

같은 `betId` 재요청을 replay로 처리하면서 다른 요청은 conflict로 막으려면 어떤 필드를 어떻게 정규화해 지문을 만들겠습니까?

꼬리 질문:

- 단순 문자열 이어붙이기가 위험한 이유는 무엇입니까?
- 선택 순서를 지문에서 무시해야 하는 근거는 무엇입니까?
- 평가 시각을 지문에 넣지 않은 이유는 무엇입니까?
- 해시 알고리즘 버전을 입력에 포함하는 이유는 무엇입니까?

### 30초 모범 답변

지문은 사용자, bet, 금액, 통화, 선택 집합처럼 요청의 의미를 결정하는 필드만 포함합니다. 선택은 순서가 의미 없으므로 정렬하고, 각 필드는 길이를 먼저 넣어 단순 연결의 경계 모호성을 없앱니다. 평가 시각처럼 재시도마다 바뀔 값은 제외하고, 버전 문자열을 포함해 정규화 규칙 변경을 구분합니다. 결과는 SHA-256 소문자 16진수로 보관해 같은 의미의 재요청은 같은 토큰, 다른 의미는 conflict가 되게 합니다.

### 답변 핵심 키워드

정규 직렬화 · 필드 길이 prefix · 선택 정렬 · 의미 필드만 포함 · 버전 태그 · SHA-256 · replay와 conflict

### 백지 구현

#### 구현 목표

의미가 같은 요청은 필드 순서나 선택 입력 순서와 무관하게 같은 64자리 소문자 SHA-256 지문을 만들고, 의미가 달라지면 지문이 달라지도록 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public record FingerprintInput(
    String userId,
    String betId,
    long amount,
    String currency,
    List<String> selectionIds) {}

public static String fingerprint(FingerprintInput input) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 이미 도메인 검증을 통과한 요청 의미 필드
- 출력: `[0-9a-f]{64}` 형식의 문자열

#### 반드시 만족해야 할 조건

- 버전 식별자를 해시 입력에 포함한다.
- 사용자, bet, 금액, 통화, 모든 선택 ID가 반영된다.
- 선택 목록의 순서는 결과에 영향을 주지 않는다.
- 서로 다른 필드 조합이 단순 연결 경계 때문에 같은 바이트열이 되지 않는다.
- 현재 시각, 요청 수신 시각 등 replay마다 달라질 수 있는 값은 제외한다.
- 입력 목록을 변경하지 않는다.

#### 경계 조건

- 선택 1개와 여러 개
- 같은 선택 집합의 서로 다른 순서
- 문자열 길이가 다른 필드 조합
- 금액 또는 통화만 다른 요청
- 빈 선택이나 중복 선택은 상위 도메인에서 차단된다는 전제와 이 함수의 방어 수준

#### 실패 조건

- null 입력 또는 필수 필드
- SHA-256을 사용할 수 없는 런타임
- 입력이 사전 invariant를 위반하는 경우 처리 정책을 명확히 정하지 못함

#### 필요한 제약

- JSON 직렬화 라이브러리에 정규화 책임을 넘기지 않는다.
- 플랫폼 기본 문자셋에 의존하지 않는다.
- 지문에 비밀값을 넣지 않는다.
- 실제 원본 지문 값이나 테스트 고정값을 복사하지 않는다.

### 구현 후 자가 검증

- [ ] 같은 입력을 여러 번 처리하면 같은 결과다.
- [ ] 선택 순서만 뒤집어도 같은 결과다.
- [ ] 사용자, bet, 금액, 통화, 선택 중 하나라도 달라지면 결과가 달라진다.
- [ ] 시간 값이 입력 모델에 없으며 replay 안정성이 유지된다.
- [ ] 결과가 정확히 64자리 소문자 16진수다.
- [ ] `"ab"+"c"`와 `"a"+"bc"` 같은 경계 모호성이 발생하지 않는다.
- [ ] 원본 선택 목록의 순서나 내용이 변경되지 않는다.
- [ ] 정렬 때문에 시간 복잡도가 `O(n log n)`임을 설명할 수 있다.

### 구현 후 설명할 것

1. 지문이 인증용 비밀 토큰이 아니라 요청 의미 결합값이라는 점
2. 길이 prefix를 사용한 이유와 다른 정규 직렬화 대안
3. 선택 정렬로 얻는 replay 안정성과 추가 비용
4. 버전 필드를 바꿀 때 기존 lifecycle과의 호환 전략
5. SHA-256 충돌 가능성과 실제 시스템에서의 허용 판단

### 원본 확인 위치

- Thread 09
- 커밋: `feat(reservation): fingerprint reservation requests`
- 파일: `src/main/java/com/sportsbook/risk/reservation/ReservationFingerprint.java`
- 클래스: `ReservationFingerprint`
- 테스트: `ReservationFingerprintTest`
- 관련 Thread: 10, 11, 13, 14

---

## P05. [Thread 09 / `feat(reservation): define reservation decisions`] 불가능한 결과 조합을 차단하는 결과 타입

### 면접 질문

예약 결과를 하나의 record로 표현하면서 `APPROVED`, `REJECTED`, `CONFLICT`마다 허용되는 필드를 어떻게 강제했나요? 그냥 nullable 필드를 두는 것과 무엇이 다릅니까?

꼬리 질문:

- 승인 결과에 거절 사유가 들어가면 어디서 막아야 합니까?
- conflict 결과가 replay나 패턴 목록을 갖지 못하게 한 이유는 무엇입니까?
- Java sealed hierarchy로 바꾸면 어떤 장단점이 있습니까?

### 30초 모범 답변

결과 상태마다 유효한 데이터 모양이 다르므로 생성자에서 조합 invariant를 강제했습니다. 승인에는 상태·만료·토큰이 필요하고 거절에는 거절 사유만 필요하며 conflict에는 outcome 데이터가 없어야 합니다. 목록은 복사해 불변성을 보장합니다. 이러면 컨트롤러나 replay 로직이 nullable 필드의 우연한 조합을 방어하지 않아도 되고, Redis wire 매핑 오류도 경계에서 바로 드러납니다.

### 답변 핵심 키워드

합 타입 흉내 · 상태별 shape · 생성자 검증 · nullable 조합 제거 · 불변 컬렉션 · wire 매핑 방어

### 백지 구현

#### 구현 목표

세 결과 상태를 하나의 값 타입으로 표현하되, 상태별로 불가능한 필드 조합은 생성할 수 없도록 한다.

#### 인터페이스 또는 함수 시그니처

```java
public record AdmissionResult(
    Status status,
    Instant expiresAt,
    String token,
    String rejectionReason,
    boolean replayed,
    List<String> flags) {

  public AdmissionResult {
    // 직접 구현
  }

  enum Status {
    APPROVED, REJECTED, CONFLICT
  }
}
```

#### 입력과 출력

- 입력: 상태와 상태별 부가 정보
- 출력: 내부 조합이 일관된 결과 값
- 잘못된 조합: 생성 실패

#### 반드시 만족해야 할 조건

- `APPROVED`: 만료 시각과 비어 있지 않은 토큰 필수, 거절 사유 금지
- `REJECTED`: 비어 있지 않은 거절 사유 필수, 만료와 토큰 금지
- `CONFLICT`: 만료·토큰·거절 사유 금지, replay는 false, flags는 빈 목록
- flags는 null이 아니고 생성 뒤 외부 변경의 영향을 받지 않는다.
- 편의 팩터리 메서드를 추가하더라도 canonical constructor를 우회하지 않는다.

#### 경계 조건

- 빈 토큰과 공백 거절 사유
- 수정 가능한 flags 목록
- 승인 replay와 거절 replay
- conflict에 우연히 부가 데이터가 들어온 경우

#### 실패 조건

- null 상태 또는 목록
- 상태별 필수 필드 누락
- 상태별 금지 필드 존재
- conflict에 replay/flags 존재

#### 필요한 제약

- getter에서 매번 조합을 확인하지 않는다.
- 잘못된 값을 조용히 제거해 정상화하지 않는다.
- 10~20분 안에 생성자와 대표 테스트를 끝낸다.

### 구현 후 자가 검증

- [ ] 각 상태의 정상 인스턴스를 만들 수 있다.
- [ ] 승인에 거절 사유를 넣으면 실패한다.
- [ ] 거절에 토큰이나 만료를 넣으면 실패한다.
- [ ] conflict에 replay 또는 flags를 넣으면 실패한다.
- [ ] 공백 토큰과 공백 거절 사유가 거부된다.
- [ ] 입력 flags 목록을 나중에 변경해도 결과가 변하지 않는다.
- [ ] 결과 소비 코드가 상태 분기 뒤 필수 필드의 존재를 신뢰할 수 있다.

### 구현 후 설명할 것

1. 하나의 record와 sealed hierarchy 중 현재 문제 규모에서의 선택
2. 잘못된 조합을 자동 정리하지 않고 예외로 드러내는 이유
3. Redis/JSON wire 검증과 도메인 결과 검증을 둘 다 두는 이유
4. replay를 상태와 분리된 축으로 둔 이유
5. 향후 상태가 늘어날 때 switch 완전성과 호환성을 유지하는 방법

### 원본 확인 위치

- Thread 09
- 커밋: `feat(reservation): define reservation decisions`
- 파일: `src/main/java/com/sportsbook/risk/reservation/ReservationDecision.java`
- 클래스: `ReservationDecision`
- 테스트: `ReservationDecisionTest`
- 관련 Thread: 04, 10, 11, 14, 16
