# 도메인 상태와 불변식 면접 워크북

이 문서는 베팅 조합을 닫힌 타입 계층으로 표현하는 방법, 복합 객체의 구조적 불변식, 상태별 정산 필드 조합을 다룬다. 정책을 공통 계약에 과도하게 넣지 않으면서도 모든 소비자가 해석 가능한 상태만 만들도록 하는 판단이 핵심이다.

<a id="i03"></a>
## [Thread 07 / `feat(bet): classify bet slips`] 봉인 타입과 K-of-N 베팅 조합

### 면접 질문

단일·다중·시스템 베팅을 하나의 enum과 nullable 필드로 표현하지 않고 sealed interface와 하위 record로 모델링한 이유는 무엇인가요?

꼬리 질문:

- `SYSTEM`을 Trixie, Yankee 같은 상품명별 타입이 아니라 `minWins`, `totalSelections`의 K-of-N으로 일반화한 이유는 무엇인가요?
- `totalSelections`의 상한을 구조적 계약에 둘 때 얻는 장점과 단점은 무엇인가요?
- Jackson 다형성 태그가 바뀌면 어떤 소비자가 영향을 받나요?
- 조합 수가 커질 때 계산량은 어떤 형태로 증가하나요?

### 30초 모범 답변

세 조합은 가능한 상태와 필요한 데이터가 다르므로 닫힌 타입 계층으로 표현하면 타입별 필드를 명확히 하고 잘못된 조합을 생성 단계에서 차단할 수 있습니다. 시스템 베팅은 상품명을 계속 추가하기보다 K-of-N으로 일반화해 프로토콜 변경 없이 여러 상품을 표현합니다. 다만 N이 커질수록 조합 수가 급증하므로 구조적으로 허용할 최대 범위와 서비스 운영 정책의 더 엄격한 한도를 분리해야 합니다.

### 답변 핵심 키워드

sealed hierarchy, exhaustive modeling, nullable state 제거, tagged union, K-of-N, 조합 폭발, 구조 한도, 운영 정책 분리, wire discriminator

### 백지 구현

**구현 목표**

세 종류의 베팅 조합을 닫힌 타입 계층으로 표현하고, 시스템 베팅의 구조적 범위를 생성 시점에 검증한다.

**인터페이스 또는 함수 시그니처**

```java
public sealed interface BetSlipType
    permits BetSlipType.Single, BetSlipType.Multiple, BetSlipType.System {

  record Single() implements BetSlipType {}

  record Multiple() implements BetSlipType {}

  record System(int minWins, int totalSelections) implements BetSlipType {
    public static final int MIN_TOTAL_SELECTIONS = 2;
    public static final int MAX_TOTAL_SELECTIONS = 15;

    public System {
      // 직접 구현
    }
  }
}
```

**입력과 출력**

- 입력: 시스템 베팅의 최소 적중 수 `minWins`, 전체 선택 수 `totalSelections`
- 출력: 유효한 `BetSlipType.System` 값 또는 유효하지 않은 입력에 대한 예외

**반드시 만족해야 할 조건**

- `totalSelections`는 2 이상 15 이하이다.
- `minWins`는 1 이상 `totalSelections` 이하이다.
- `Single`, `Multiple`, `System` 이외의 구현은 허용하지 않는다.
- 타입별 JSON 표현을 설계한다면 안정적인 구분 태그를 가져야 한다.
- Trixie 같은 상품명은 프로토콜 타입으로 추가하지 않는다.

**경계 조건**

- `(minWins=1, totalSelections=2)`
- `(minWins=15, totalSelections=15)`
- `minWins=0`
- `minWins > totalSelections`
- `totalSelections=1`
- `totalSelections=16`

**실패 조건**

- 범위를 벗어난 값이 정상 객체로 생성됨
- 타입 구분 없이 nullable 필드 조합으로 되돌아감
- 운영 환경의 최대 선택 수 정책까지 이 공통 타입이 소유함

**필요한 제약**

- Java 17의 sealed type과 record를 사용한다.
- 조합 생성 알고리즘은 구현하지 않는다.
- 검증은 생성 직후부터 항상 성립해야 한다.

### 구현 후 자가 검증

- [ ] 최소·최대 `totalSelections`가 허용된다.
- [ ] 상한 바로 위와 하한 바로 아래가 거부된다.
- [ ] `minWins=1`, `minWins=totalSelections`가 허용된다.
- [ ] `minWins=0`, `minWins>totalSelections`가 거부된다.
- [ ] 하위 타입 집합이 닫혀 있다.
- [ ] 값 동등성이 record 의미와 일치한다.
- [ ] 공통 구조 한도와 서비스 정책 한도를 혼동하지 않았다.

### 구현 후 설명할 것

1. enum과 여러 nullable 필드보다 sealed hierarchy가 유효하지 않은 상태를 줄이는 이유
2. K-of-N 일반화가 새 상품 추가 시 프로토콜 변경을 줄이는 방식
3. 최대 N을 무제한으로 두지 않은 이유와 조합 복잡도
4. JSON 구분 태그가 장기 wire 계약이 되는 이유
5. 구조적 허용 범위와 운영 정책을 서로 다른 계층에 두는 이유

### 원본 확인 위치

- Thread 07
- 커밋: `feat(bet): classify bet slips`
- 커밋: `test(bet): verify slip type invariants`
- 파일: `src/main/java/com/sportsbook/protocol/domain/BetSlipType.java`
- 클래스·컴포넌트: `BetSlipType`, `BetSlipType.Single`, `BetSlipType.Multiple`, `BetSlipType.System`
- 테스트: `BetSlipTypeTest`
- 관련 Thread: 08, 13, 15, 16

---

<a id="i04"></a>
## [Thread 08 / `feat(bet): compose self-consistent slips`] BetSlip 구조적 불변식과 defensive copy

### 면접 질문

`BetSlip` 생성자가 선택 개수, stake, 타입 일치 여부를 검증하도록 한 이유는 무엇인가요? 같은 이벤트 조합 제한이나 배당 drift까지 이 생성자에서 검증하지 않은 이유도 설명해 주세요.

꼬리 질문:

- Java record인데도 `List<BetSelection>`을 복사해야 하는 이유는 무엇인가요?
- 입력 리스트를 복사하지 않으면 객체의 hash/equality 또는 소비자 관점에서 어떤 문제가 생길 수 있나요?
- `SYSTEM.totalSelections`와 실제 리스트 크기가 다르면 어느 경계에서 거부해야 하나요?
- 구조적 불변식과 서비스 정책을 나누는 기준은 무엇인가요?

### 30초 모범 답변

공통 프로토콜은 모든 소비자가 안전하게 해석할 수 있는 자기 일관된 데이터 모양만 허용해야 합니다. 그래서 선택 목록이 비어 있지 않고, stake가 양수이며, 베팅 타입과 선택 개수가 맞는지는 생성 시점에 강제합니다. 반면 같은 이벤트 허용 여부, 최대 선택 수, 배당 drift 같은 규칙은 설정과 업무 흐름에 따라 바뀌므로 betting 서비스가 소유합니다. record도 내부에 가변 리스트 참조를 가질 수 있으므로 defensive copy로 생성 후 상태 변화를 막습니다.

### 답변 핵심 키워드

aggregate invariant, fail-fast constructor, structural consistency, policy boundary, defensive copy, shallow immutability, aliasing, self-consistent wire shape

### 백지 구현

**구현 목표**

베팅 조합과 선택 목록이 구조적으로 일치하고, 생성 후 외부 리스트 변경의 영향을 받지 않는 불변 `BetSlip`을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public record BetSlip(
    BetId id,
    UserId userId,
    BetSlipType type,
    List<BetSelection> selections,
    Money stake,
    BetStatus status,
    Instant placedAt,
    SettlementResult settlementResult,
    Instant settledAt,
    Money payout) {

  public static final int MULTIPLE_MIN_SELECTIONS = 2;

  public BetSlip {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 식별자, 베팅 타입, 선택 목록, stake, 상태와 정산 관련 값
- 출력: 자기 일관된 불변 `BetSlip` 또는 잘못된 구조에 대한 예외

**반드시 만족해야 할 조건**

- 필수 필드는 null이 아니다.
- 선택 목록은 비어 있지 않다.
- stake는 양수다.
- `Single`은 선택 1개만 가진다.
- `Multiple`은 선택 2개 이상을 가진다.
- `System`은 실제 선택 수와 `totalSelections`가 같다.
- 입력 선택 목록을 복사하고, 외부에서 수정할 수 없는 형태로 보관한다.
- 같은 이벤트 조합, 마켓 조합, 배당 drift, 운영 최대 선택 수는 여기서 검증하지 않는다.

**경계 조건**

- Single 0개·1개·2개
- Multiple 1개·2개
- System 선언 N과 실제 목록 크기의 일치·불일치
- stake가 음수·0·양수
- 변경 가능한 `ArrayList`를 생성자에 전달한 뒤 원본을 수정하는 경우
- 선택 목록 자체 또는 필수 필드가 null인 경우

**실패 조건**

- 잘못된 선택 수를 가진 slip 생성
- 0 또는 음수 stake 허용
- 외부 리스트 수정으로 생성된 slip의 내용이 바뀜
- 공통 계약이 betting 서비스의 변동 가능한 정책까지 소유함

**필요한 제약**

- 선택 원소인 `BetSelection`은 별도의 불변 값으로 가정한다.
- 전체 리스트를 순회하는 것 이상의 불필요한 계산을 하지 않는다.
- 정산 상태 조합의 세부 검증은 다음 문제에서도 별도로 다룬다.

### 구현 후 자가 검증

- [ ] 각 베팅 타입의 최소 유효 선택 수를 통과시켰다.
- [ ] 타입과 선택 수가 맞지 않는 모든 대표 조합을 거부했다.
- [ ] stake의 0과 음수를 거부했다.
- [ ] null 필수 값이 생성 이후까지 남지 않는다.
- [ ] 생성자 호출 뒤 원본 리스트를 변경해도 slip이 바뀌지 않는다.
- [ ] 반환된 선택 목록을 직접 수정할 수 없다.
- [ ] 검증 복잡도가 선택 수 N에 대해 불필요하게 증가하지 않는다.
- [ ] 구조 규칙과 서비스 정책의 경계를 지켰다.

### 구현 후 설명할 것

1. 생성자에서 불변식을 강제하면 호출 지점마다 검증을 반복하지 않아도 되는 이유
2. record가 참조하는 컬렉션까지 자동으로 불변으로 만들지는 않는다는 점
3. `List.copyOf` 계열 선택이 aliasing을 차단하는 방식
4. 공통 계약이 소유할 규칙과 betting 서비스가 소유할 규칙을 나눈 기준
5. 선택 수 검증의 시간·공간 비용

### 원본 확인 위치

- Thread 08
- 커밋: `feat(bet): define bet selections`
- 커밋: `test(bet): verify selection invariants`
- 커밋: `feat(bet): compose self-consistent slips`
- 커밋: `test(bet): verify structural slip invariants`
- 커밋: `test(bet): isolate slip selection state`
- 파일: `src/main/java/com/sportsbook/protocol/domain/BetSelection.java`
- 파일: `src/main/java/com/sportsbook/protocol/domain/BetSlip.java`
- 클래스·컴포넌트: `BetSelection`, `BetSlip`
- 테스트: `BetSelectionTest`, `BetSlipStructureTest`, `BetSlipIsolationTest`
- 관련 Thread: 03, 04, 07, 13, 15, 16

---

<a id="i05"></a>
## [Thread 08 / `test(bet): verify settlement slip invariants`] 상태에 따른 정산 필드 조합

### 면접 질문

`status`, `settlementResult`, `settledAt`, `payout`을 독립 nullable 필드로 두면 어떤 잘못된 상태들이 생길 수 있나요? 현재 모델은 그 조합을 어떻게 제한하나요?

꼬리 질문:

- `LOST`에 payout 0을 넣는 것과 payout을 두지 않는 것 중 계약 관점의 차이는 무엇인가요?
- `SettlementResult.VOID`와 `BetStatus.VOIDED`는 왜 다른 개념인가요?
- `SETTLED`가 아닌 상태에 정산 필드가 들어오면 무시할지 실패시킬지 어떻게 판단하나요?
- 정산 금액의 정확한 계산까지 이 값 객체가 검증해야 하나요?

### 30초 모범 답변

nullable 필드가 독립적으로 움직이면 미정산 베팅에 payout이 있거나, 정산 완료인데 결과와 시각이 없는 모순이 생깁니다. 따라서 `SETTLED`는 결과와 정산 시각을 반드시 갖고, 그 외 상태는 정산 필드를 갖지 못하게 합니다. `WON`, `PUSH`, `VOID`는 실제 지급 또는 환급 스냅샷이 필요하고 `LOST`는 payout을 두지 않습니다. 다만 payout 산식과 잔액 반영은 settlement·wallet 서비스의 정책과 트랜잭션 책임입니다.

### 답변 핵심 키워드

state-dependent invariant, illegal state, terminal snapshot, null semantics, LOST absence, VOID vs VOIDED, fail-fast, calculation ownership

### 백지 구현

**구현 목표**

베팅 상태와 정산 필드의 조합을 한 곳에서 검증해 모순된 객체가 생성되지 않도록 한다.

**인터페이스 또는 함수 시그니처**

```java
static void validateSettlementState(
    BetStatus status,
    SettlementResult settlementResult,
    Instant settledAt,
    Money payout) {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 베팅 상태와 선택적 정산 결과·시각·지급액
- 출력: 반환값 없음. 조합이 유효하지 않으면 예외

**반드시 만족해야 할 조건**

- `SETTLED`는 `settlementResult`와 `settledAt`을 반드시 가진다.
- `SETTLED`가 아닌 상태는 `settlementResult`, `settledAt`, `payout`을 가지지 않는다.
- `WON`, `PUSH`, `VOID` 결과는 payout을 가진다.
- `LOST` 결과는 payout을 가지지 않는다.
- payout의 정확한 액수, stake와의 관계, 통화 일치는 이 문제의 범위를 벗어난다.

**경계 조건**

- `SETTLED + WON + payout`
- `SETTLED + PUSH + payout`
- `SETTLED + VOID + payout`
- `SETTLED + LOST + payout 없음`
- `SETTLED + 결과 없음`
- `SETTLED + 시각 없음`
- 비정산 상태에 결과만 존재, 시각만 존재, payout만 존재
- `LOST + Money.zero(...)`

**실패 조건**

- 모순된 필드 조합을 조용히 보정하거나 무시함
- `LOST`와 payout 0의 존재를 동일하게 취급해 wire 의미를 바꿈
- 정산 산식이나 wallet 잔액 정책을 검증 함수에 끌어옴

**필요한 제약**

- 기존 enum 값 집합만 사용한다.
- 예외 타입과 메시지는 호출자가 원인을 식별할 수 있어야 한다.
- 검증은 상수 시간에 끝나야 한다.

### 구현 후 자가 검증

- [ ] 네 가지 정산 결과의 유효 조합을 각각 확인했다.
- [ ] 결과·시각 누락을 각각 독립적으로 거부했다.
- [ ] 모든 비정산 상태에서 정산 필드의 존재를 거부했다.
- [ ] `LOST + null payout`과 `LOST + zero payout`을 구분했다.
- [ ] VOID 결과와 VOIDED 상태의 의미를 섞지 않았다.
- [ ] 검증이 상태를 변경하거나 값을 보정하지 않는다.
- [ ] payout 산식 같은 외부 정책을 포함하지 않았다.

### 구현 후 설명할 것

1. nullable 조합을 허용할 때 생기는 대표적인 불법 상태
2. 누락과 0 값이 wire 계약에서 다른 의미가 될 수 있는 이유
3. 생성 시점 실패와 나중 검증 중 생성 시점 실패를 선택한 이유
4. `SettlementResult.VOID`와 `BetStatus.VOIDED`의 경계
5. 이 검증이 책임지지 않는 정산·지갑 정책

### 원본 확인 위치

- Thread 08
- 커밋: `test(bet): verify state semantics`
- 커밋: `feat(bet): compose self-consistent slips`
- 커밋: `test(bet): verify settlement slip invariants`
- 파일: `src/main/java/com/sportsbook/protocol/domain/BetStatus.java`
- 파일: `src/main/java/com/sportsbook/protocol/domain/SettlementResult.java`
- 파일: `src/main/java/com/sportsbook/protocol/domain/BetSlip.java`
- 클래스·컴포넌트: `BetStatus`, `SettlementResult`, `BetSlip`
- 테스트: `DomainEnumsTest`, `BetSlipSettlementTest`
- 관련 Thread: 13, 14, 15, 16
