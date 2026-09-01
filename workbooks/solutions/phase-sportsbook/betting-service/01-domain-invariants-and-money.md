# 도메인 invariant, 조합 계산, 접수 검증, 시간 순서 조회

이 문서는 베팅 도메인의 구조적 invariant, 시스템 베팅의 조합·금액 의미론, 실패 폐쇄형 접수 검증, 시간 순서 식별자와 키셋 조회를 다룬다. 각 백지 구현은 원본 코드를 재현하는 문제가 아니라 같은 요구사항을 만족하는 축소 문제다.

<a id="t01-aggregate-invariants"></a>
## [Thread 01 / `feat(domain): attach structurally valid legs`] 집계 루트가 자식 구조를 소유하도록 만들기

### 면접 질문

베팅 슬립의 종류가 SINGLE, MULTIPLE, SYSTEM으로 나뉠 때, 왜 `Bet` 생성 이후에 `BetLeg`를 임의로 추가하게 두지 않고 집계 생성 시점에 구조를 검증하고 소유 관계와 순서를 확정해야 합니까?

꼬리 질문:

- 애플리케이션 검증과 데이터베이스 제약을 동시에 두는 이유는 무엇입니까?
- `legs()`가 내부 가변 컬렉션을 그대로 반환하면 어떤 invariant가 깨질 수 있습니까?
- SYSTEM의 `totalSelections`와 실제 leg 수가 다른 상태를 허용하면 뒤의 금액 계산과 정산에 어떤 문제가 생깁니까?
- JPA 양방향 관계에서 자식의 back-reference와 순번을 누가 설정해야 합니까?

### 30초 모범 답변

집계 루트는 유효한 상태만 외부에 노출해야 합니다. 그래서 생성 시 slip 종류와 leg 수를 검사하고, 각 leg의 소유자와 순번을 한 번에 고정합니다. 그렇지 않으면 부분적으로 조립된 집계가 저장되거나, SYSTEM의 조합 수와 실제 선택 수가 어긋나 금액·리스크·정산 결과가 서로 달라질 수 있습니다. 애플리케이션 invariant는 빠른 오류와 명확한 도메인 표현을 담당하고, 데이터베이스 제약은 다른 쓰기 경로나 버그까지 막는 마지막 방어선입니다.

### 답변 핵심 키워드

`aggregate root`, `valid-by-construction`, `bidirectional ownership`, `ordered child`, `defensive copy`, `application invariant`, `database constraint`

### 백지 구현

**구현 목표**

슬립 종류와 자식 leg 목록을 받아 항상 구조적으로 유효하고, 순서와 소유 관계가 확정된 집계를 생성한다.

**인터페이스 또는 함수 시그니처**

```java
sealed interface SlipType permits Single, Multiple, System {}
record Single() implements SlipType {}
record Multiple() implements SlipType {}
record System(int minWins, int totalSelections) implements SlipType {}

final class Leg {
  void attachTo(Slip owner, int index) {
    // 직접 구현
  }
}

final class Slip {
  static Slip create(SlipType type, List<Leg> legs) {
    // 직접 구현
    return null;
  }

  List<Leg> legs() {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: slip 종류, 순서를 가진 leg 목록
- 출력: 모든 leg가 해당 집계에 소속되고 0부터 연속된 순번을 가진 `Slip`

**반드시 만족해야 할 조건**

- SINGLE은 leg가 정확히 1개다.
- MULTIPLE은 leg가 2개 이상이다.
- SYSTEM은 `totalSelections`와 leg 수가 같다.
- `System.minWins`는 의미 있는 범위에 있어야 한다.
- null 목록, null leg, 동일 leg 객체의 중복 소유를 허용하지 않는다.
- 외부가 반환된 목록을 바꿔 집계 내부를 수정할 수 없어야 한다.

**경계 조건**

- 빈 목록
- SINGLE에 0개 또는 2개
- MULTIPLE에 1개
- SYSTEM의 `minWins`가 0, 음수, `totalSelections` 초과
- 이미 다른 집계에 붙은 leg

**실패 조건**

구조가 잘못되면 부분 집계를 반환하지 말고 예외로 종료한다. 실패 뒤 어떤 leg도 새 집계에 절반만 연결된 상태로 남지 않아야 한다.

**제약**

10~20분 안에 구현할 수 있도록 영속성 매핑은 제외한다. 컬렉션 복사와 조립 순서까지 구현 범위에 포함한다.

### 구현 후 자가 검증

- [ ] SINGLE, MULTIPLE, SYSTEM의 정상 구조가 모두 생성된다.
- [ ] 각 leg의 owner와 index가 정확하다.
- [ ] 잘못된 leg 수가 생성 전에 거부된다.
- [ ] 중간 leg가 null이어도 부분 연결 상태가 남지 않는다.
- [ ] 입력 목록을 나중에 변경해도 집계가 변하지 않는다.
- [ ] 반환된 목록을 수정해 내부 상태를 바꿀 수 없다.
- [ ] SYSTEM 파라미터와 실제 leg 수가 항상 일치한다.

### 구현 후 설명할 것

1. 검증을 생성자, 정적 팩터리, 별도 validator 중 어디에 두었는지와 이유
2. 모든 검증을 끝낸 뒤 소유 관계를 연결하도록 한 이유
3. 방어적 복사가 메모리 비용보다 중요한 이유
4. 애플리케이션 invariant와 DB 제약이 중복이 아니라 서로 다른 방어층인 이유

### 원본 확인 위치

- Thread: 01, 베팅 슬립 집계와 영속 표현
- 커밋: `feat(domain): model persisted bet legs`, `feat(domain): attach structurally valid legs`, `feat(database): create bet aggregate schema`
- 파일: `Bet.java`, `BetLeg.java`, `V1__bet_and_leg.sql`
- 함수·컴포넌트: `Bet.pending(...)`, `BetLeg.create(...)`, `BetLeg.assignTo(...)`, `Bet.legs()`
- 관련 Thread: 02, 03, 15, 16

---

<a id="t02-system-combinatorics"></a>
## [Thread 02 / `feat(domain): calculate system exposure`] 조합 수, 총 노출액, 최대 지급액을 안전하게 계산하기

### 면접 질문

K-of-N 시스템 베팅에서 unit stake, 총 stake, 최대 payout을 각각 어떻게 정의했고, 왜 같은 `stake`라는 이름으로 섞어 쓰면 안 됩니까? 조합 수와 조합별 배당 곱을 계산할 때 overflow와 반올림은 어디에서 통제해야 합니까?

꼬리 질문:

- `nCk`를 팩토리얼로 계산하지 않은 이유는 무엇입니까?
- `k`와 `n-k` 중 작은 값을 사용하는 이유는 무엇입니까?
- 조합별 최대 payout 계산의 시간 복잡도는 얼마입니까?
- `BigDecimal` 계산 뒤 정수 통화 단위로 내릴 때 반올림 규칙을 계약으로 고정해야 하는 이유는 무엇입니까?
- SYSTEM 2-of-4의 unit stake가 1,000원이면 총 stake는 얼마이며 왜 그렇습니까?

### 30초 모범 답변

SYSTEM의 unit stake는 조합 한 줄에 거는 금액이고, 총 stake는 unit stake에 `nCk`를 곱한 값입니다. 최대 payout은 가능한 모든 k개 조합의 배당 곱을 합한 뒤 unit stake를 곱합니다. 조합 수는 팩토리얼 대신 작은 쪽 k만 순회해 중간값을 줄이고, 정수 변환에는 exact 연산을 써 overflow를 조용히 허용하지 않습니다. 배당 계산은 고정 정밀도 숫자로 하고 최종 통화 단위의 내림 규칙을 한곳에 둬 리스크·이벤트·정산이 같은 금액 의미를 사용하게 합니다.

### 답변 핵심 키워드

`nCk`, `unit stake`, `total exposure`, `combination enumeration`, `BigDecimal`, `overflow`, `exact conversion`, `rounding contract`, `O(C(n,k)·k)`

### 백지 구현

**구현 목표**

K-of-N 시스템 베팅의 line 수, 총 stake, 모든 조합이 적중했을 때의 최대 payout을 계산한다.

**인터페이스 또는 함수 시그니처**

```java
record Money(long amount, String currency) {}

final class SystemCalculator {
  long lineCount(int minWins, int totalSelections) {
    // 직접 구현
    return 0;
  }

  Money totalStake(int minWins, int totalSelections, Money unitStake) {
    // 직접 구현
    return null;
  }

  Money maxPayout(int minWins, Money unitStake, List<BigDecimal> decimalOdds) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: `minWins`, 선택 수, 한 줄 stake, 각 선택의 decimal odds
- 출력: 조합 수, 전체 노출액, 최대 payout

**반드시 만족해야 할 조건**

- `1 <= minWins <= totalSelections`
- odds 목록 크기가 `totalSelections`와 같다.
- 금액은 음수가 아니고 통화가 유지된다.
- 조합 수 또는 금액이 `long` 범위를 넘으면 명시적으로 실패한다.
- 최대 payout은 모든 크기 `minWins` 조합을 정확히 한 번씩 포함한다.
- 최종 payout은 소수 통화 단위를 버리는 명시된 내림 규칙을 사용한다.

**경계 조건**

- 1-of-1
- 1-of-N, N-of-N
- 대칭인 `nCk == nC(n-k)`
- odds가 1에 매우 가까운 경우
- 조합 수는 유효하지만 최종 금액이 overflow하는 경우
- 빈 odds, 0 또는 음수 odds

**실패 조건**

유효하지 않은 파라미터, currency 불일치, overflow, 정수로 정확히 표현할 수 없는 중간 결과 처리 규칙 위반은 예외로 종료한다.

**제약**

조합 목록 전체를 메모리에 저장하지 않아도 된다. 외부 조합 라이브러리는 사용하지 않는다.

### 구현 후 자가 검증

- [ ] `2-of-4`의 line 수가 6이다.
- [ ] `k=0`, `k>n`, 음수 입력을 조용히 0으로 처리하지 않고 계약대로 거부한다.
- [ ] `nCk` 대칭성이 유지된다.
- [ ] 조합이 누락되거나 중복되지 않는다.
- [ ] unit stake와 total stake를 혼동하지 않는다.
- [ ] 내림 직전까지 소수 정밀도가 보존된다.
- [ ] 중간 곱셈과 최종 변환의 overflow가 검출된다.
- [ ] 시간 복잡도와 재귀 깊이를 설명할 수 있다.

### 구현 후 설명할 것

1. 팩토리얼 대신 곱셈·나눗셈 누적 공식을 선택한 이유
2. 조합을 스트리밍으로 계산할지 목록으로 만들지에 대한 공간 trade-off
3. 금액 반올림 시점을 마지막으로 미룬 이유
4. unit stake를 이벤트에 유지하고 total exposure는 별도로 계산해야 하는 이유
5. 매우 큰 N을 허용할 때 정책 상한이 필요한 이유

### 원본 확인 위치

- Thread: 02, 시스템 베팅 조합과 금액 의미론
- 커밋: `feat(domain): calculate system exposure`, `feat(domain): calculate maximum system payout`, `feat(outbox): build unit-stake placement events`
- 파일: `SystemBetCalculator.java`, `BetEventFactory.java`
- 함수·컴포넌트: `lineCount(...)`, `totalStake(...)`, `maxPayout(...)`, `binomial(...)`
- 관련 Thread: 01, 06, 07, 09, 15

---

<a id="t03-fail-closed-admission"></a>
## [Thread 03 / `feat(odds): read effective market snapshots`] 외부 시세를 실패 폐쇄형으로 검증하기

### 면접 질문

베팅 접수 시 Redis의 market 상태나 odds 값이 없거나 잘못된 경우, 왜 "현재 값은 모르지만 일단 접수"가 아니라 market closed로 처리했습니까? 제출 odds와 현재 odds를 비교할 때 사용자를 보호하는 허용 경계는 어떻게 계산해야 합니까?

꼬리 질문:

- `OPEN` 외의 값, null, 파싱 불가능한 odds를 같은 실패 계열로 묶은 이유는 무엇입니까?
- current odds가 제출 odds보다 좋아진 경우에도 거부해야 합니까?
- tolerance 3%, 제출 2.00일 때 1.94와 1.9399의 판정은 어떻게 달라집니까?
- 검증 과정에서 읽은 snapshot과 실제 외부 부작용 사이에 시차가 생기는 문제를 어떻게 설명하겠습니까?
- 중복 selection, 동일 event·market 제한은 시세 검증 전후 어느 시점에 두는 편이 낫습니까?

### 30초 모범 답변

접수는 돈이 걸린 상태 전이이므로 외부 시세를 확인할 수 없으면 승인 근거가 없습니다. 따라서 market이 정확히 OPEN이고 selection odds가 정상 숫자로 존재할 때만 진행하고, 나머지는 실패 폐쇄형으로 거부합니다. 하락 허용치는 `quoted × (1 - tolerance)`를 하한으로 두고 current가 하한 이상이면 통과시킵니다. 상승은 사용자에게 불리하지 않으므로 거부하지 않습니다. 구조·stake 검증을 먼저 수행해 불필요한 I/O를 줄이고, snapshot 검증은 외부 side effect 전에 끝냅니다.

### 답변 핵심 키워드

`fail closed`, `authoritative snapshot`, `missing evidence`, `protective slippage`, `BigDecimal comparison`, `validation ordering`, `TOCTOU limitation`

### 백지 구현

**구현 목표**

요청 leg의 구조를 검사하고, snapshot 저장소에서 각 market과 selection의 현재 상태를 읽어 보호적 slippage 정책을 적용한다.

**인터페이스 또는 함수 시그니처**

```java
record Selection(UUID eventId, UUID marketId, UUID selectionId, BigDecimal quotedOdds) {}

interface SnapshotStore {
  String marketStatus(UUID eventId, UUID marketId);
  String currentOdds(UUID eventId, UUID marketId, UUID selectionId);
}

final class AdmissionValidator {
  void validate(List<Selection> selections, BigDecimal maxDropPercent, SnapshotStore store) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 요청 selections, 최대 허용 하락률, snapshot 저장소
- 출력: 정상일 때 반환값 없음, 거부 시 분류 가능한 예외

**반드시 만족해야 할 조건**

- 빈 요청, 중복 selection을 거부한다.
- 정책상 multi인 경우 동일 event 또는 market 중복을 거부할 수 있도록 구조를 분리한다.
- market 상태는 정확히 `OPEN`일 때만 유효하다.
- odds가 누락되거나 숫자로 파싱되지 않으면 거부한다.
- current odds가 허용 하한과 같으면 통과한다.
- current odds가 좋아진 경우 통과한다.
- 부동소수점 `double`이 아니라 십진 정밀도 타입을 사용한다.

**경계 조건**

- tolerance 0%
- 하한과 정확히 같은 current odds
- 하한보다 최소 단위만 낮은 odds
- status의 대소문자·공백 차이
- null과 빈 문자열
- 매우 큰 selections 목록

**실패 조건**

구조 오류와 snapshot 근거 부재를 구분 가능한 오류로 표현하되, 둘 다 승인으로 이어져서는 안 된다.

**제약**

snapshot store 호출 실패를 retry하는 로직은 제외한다. 한 요청 안에서 동일 키를 중복 조회하지 않는 정도의 최적화는 허용한다.

### 구현 후 자가 검증

- [ ] 정상 OPEN market과 유효 odds가 통과한다.
- [ ] status 누락, CLOSED, 임의 문자열이 모두 거부된다.
- [ ] odds 누락과 파싱 오류가 승인으로 바뀌지 않는다.
- [ ] 정확한 slippage 경계값을 테스트했다.
- [ ] current odds 상승은 거부하지 않는다.
- [ ] 중복 selection 검사가 O(n²)가 아닌지 확인했다.
- [ ] 검증 실패 뒤 외부 side effect가 실행될 여지가 없다.
- [ ] 오류 분류가 호출자에게 안정적으로 전달된다.

### 구현 후 설명할 것

1. 실패 폐쇄형이 매출 손실 가능성보다 중요한 이유
2. 구조 검증을 I/O보다 먼저 수행한 이유
3. 십진수 비교식과 경계 포함 여부
4. snapshot 검증만으로 TOCTOU를 완전히 제거할 수 없는 이유
5. 캐시 장애와 실제 market closed를 같은 사용자 응답으로 볼지 내부 관측은 분리할지

### 원본 확인 위치

- Thread: 03, 실패 폐쇄형 베팅 접수 검증
- 커밋: `feat(validation): enforce slip structure policy`, `feat(odds): read effective market snapshots`, `feat(odds): enforce user-protective slippage`, `feat(placement): assemble validated pending bets`
- 파일: `BetSlipValidator.java`, `OddsSnapshotReader.java`, `OddsSlippageChecker.java`, `BetAssembler.java`
- 함수·컴포넌트: `BetSlipValidator.validate(...)`, `OddsSnapshotReader.currentOdds(...)`, `OddsSlippageChecker.check(...)`, `BetAssembler.assemble(...)`
- 관련 Thread: 01, 02, 05, 06

---

<a id="t04-time-keyset"></a>
## [Thread 04 / `feat(api): query actor scoped bet history`] 시간 순서 ID와 actor-scoped 키셋 페이지

### 면접 질문

시간 순서가 포함된 UUIDv7을 식별자로 사용하고 `betId < cursor ORDER BY betId DESC` 형태의 키셋 페이지를 구성한 이유를 설명해 보세요. offset 페이지와 비교했을 때 무엇이 좋아지고, 어떤 가정이 필요합니까?

꼬리 질문:

- UUIDv7의 timestamp 영역만으로 전역적으로 완전한 생성 순서를 보장할 수 있습니까?
- 같은 밀리초에 생성된 ID들의 상대 순서는 무엇에 의해 정해집니까?
- 페이지에 `limit + 1`개를 조회하는 이유는 무엇입니까?
- 다음 cursor는 probe row가 아니라 마지막으로 노출한 row여야 하는 이유는 무엇입니까?
- actor 조건을 cursor 조건과 같은 쿼리에 넣지 않으면 어떤 보안 문제가 생깁니까?
- 중간에 새 row가 추가되는 동안 offset과 keyset의 결과는 어떻게 달라집니까?

### 30초 모범 답변

UUIDv7은 앞부분에 시간을 담아 대체로 삽입·조회 순서와 인덱스 지역성을 맞추면서 분산 환경에서도 중앙 sequence 없이 ID를 만들 수 있습니다. 조회는 actor를 항상 조건에 포함하고, 내림차순에서 cursor보다 작은 ID만 읽는 키셋 방식을 사용합니다. offset처럼 앞의 모든 row를 건너뛸 필요가 없고 새 데이터 삽입에도 이미 본 경계가 흔들리지 않습니다. `limit + 1`개로 다음 페이지 존재 여부를 확인하고, 외부에는 실제로 반환한 마지막 ID만 cursor로 노출합니다.

### 답변 핵심 키워드

`UUIDv7`, `48-bit timestamp`, `index locality`, `keyset pagination`, `seek method`, `limit + 1`, `tenant predicate`, `stable boundary`, `offset drift`

### 백지 구현

**구현 목표**

actor별로 분리된, 내림차순 UUID 키셋 페이지를 만든다. 저장소는 이미 `actorId`, `cursor`, `limit` 조건에 맞는 정렬 결과를 반환한다고 가정한다.

**인터페이스 또는 함수 시그니처**

```java
record BetRow(UUID betId, UUID actorId, String summary) {}
record CursorPage<T>(List<T> items, UUID nextCursor, boolean hasMore) {}

interface BetRows {
  List<BetRow> firstPage(UUID actorId, int probeLimit);
  List<BetRow> after(UUID actorId, UUID cursorExclusive, int probeLimit);
}

final class BetHistory {
  CursorPage<BetRow> page(UUID actorId, UUID cursor, int requestedLimit, BetRows rows) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: actor ID, nullable cursor, 요청 limit, 저장소
- 출력: 최대 limit개의 row, nullable next cursor, `hasMore`

**반드시 만족해야 할 조건**

- limit는 기본값과 최대값 정책을 가진다.
- 저장소에는 `limit + 1`을 요청한다.
- 반환 목록은 최대 limit개다.
- 더 읽을 row가 있을 때만 `hasMore=true`이고 next cursor가 있다.
- next cursor는 반환한 마지막 row의 ID다.
- 결과의 모든 row는 요청 actor의 소유여야 한다. 저장소가 잘못된 row를 주는 경우도 방어할지 결정하고 일관되게 구현한다.
- cursor는 exclusive다.

**경계 조건**

- 결과 0개, 정확히 limit개, limit+1개
- 요청 limit가 0, 음수, 최대 초과
- 마지막 페이지
- 존재하지 않는 cursor
- 같은 actor의 새 row가 첫 페이지 앞에 삽입된 경우
- 다른 actor의 ID가 cursor와 가까운 경우

**실패 조건**

actor가 null이거나 limit 정책을 위반하면 명시적으로 실패하거나 정규화하되 계약을 문서화한다. 다른 actor의 row가 노출되어서는 안 된다.

**제약**

offset 사용 금지. cursor 인코딩·서명은 구현 범위에서 제외하고 UUID 자체를 사용한다.

### 구현 후 자가 검증

- [ ] 첫 페이지와 후속 페이지 모두 내림차순이다.
- [ ] 페이지 사이에 중복이나 누락이 없다.
- [ ] 정확히 limit개일 때 불필요한 next cursor를 만들지 않는다.
- [ ] limit+1개일 때 probe row는 사용자에게 노출되지 않는다.
- [ ] 다음 cursor가 반환된 마지막 row와 일치한다.
- [ ] actor 조건이 모든 저장소 호출에 포함된다.
- [ ] 새 row 삽입 뒤에도 이미 읽은 경계가 흔들리지 않는다.
- [ ] 시간·공간 복잡도를 offset 방식과 비교할 수 있다.

### 구현 후 설명할 것

1. UUIDv7이 "대체로 시간 순서"이지 완전한 전역 순서가 아닌 이유
2. keyset이 깊은 페이지에서 offset보다 유리한 이유
3. cursor를 exclusive로 정의한 이유
4. actor predicate를 서비스 후처리가 아니라 조회 조건에 넣어야 하는 이유
5. 정렬 키가 유일하지 않을 때 복합 cursor가 필요한 이유

### 원본 확인 위치

- Thread: 04, 시간 순서 식별자와 키셋 조회
- 커밋: `feat(identifier): generate time-ordered bet ids`, `feat(identifier): generate human bet references`, `feat(api): query actor scoped bet history`
- 파일: `UuidV7.java`, `BetReferenceGenerator.java`, `CursorPage.java`, `BetQueryService.java`, `BetRepository.java`
- 함수·컴포넌트: `UuidV7.generate(...)`, `BetQueryService.byId(...)`, `BetQueryService.page(...)`, actor-scoped repository query methods
- 관련 Thread: 05, 13, 14, 17
