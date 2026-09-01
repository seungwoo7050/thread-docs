# Redis 카운터·원자적 승인·스냅샷

이 문서는 Redis의 정렬 집합과 파생 합계, 동시 요청을 직렬화하는 원자적 승인, 패턴 임계값의 정확한 계산, 하나의 시점으로 묶인 진단 스냅샷을 다룬다. Thread 08·10·12에서 반복된 카운터 불변식은 P06과 P07에, Thread 06·10의 패턴 로직은 P08과 P09에 통합했다.

---

## P06. [Thread 08 / `feat(counter): expose sliding window counters`] 정렬 집합과 파생 합계로 만드는 슬라이딩 윈도 카운터

### 면접 질문

시간 창 안의 금액 합계를 Redis에서 계산할 때, 왜 정렬 집합만 매번 합산하지 않고 `entries` 정렬 집합과 `sum` 문자열 키를 함께 유지했나요?

꼬리 질문:

- 같은 시각에 여러 이벤트가 들어오면 정렬 집합 member를 어떻게 고유하게 만들겠습니까?
- 재전달된 같은 이벤트가 합계를 두 번 올리지 않게 하려면 무엇을 원자적으로 검사해야 합니까?
- `entries`는 없는데 `sum`만 남았거나, member 합과 `sum`이 다르면 어떻게 처리하겠습니까?
- 한 번의 추가 연산의 시간 복잡도는 무엇에 비례합니까?

### 30초 모범 답변

정렬 집합은 시간순 만료와 고유 이벤트 추적에 적합하지만 현재 합계를 매번 전체 순회하면 hot user에서 비용이 커집니다. 그래서 `<betId>|<amount>` member를 timestamp score로 저장하고 정확한 합계를 별도 키에 유지합니다. 한 Lua 실행에서 만료 member를 제거하고 합계를 차감한 뒤 `ZADD NX` 성공 시에만 새 금액을 더합니다. 두 구조가 불일치하면 승인 계산을 계속하지 않고 실패시켜 파생 집계 오류가 위험 한도를 우회하지 못하게 합니다.

### 답변 핵심 키워드

sorted set · timestamp score · 고유 member · `ZADD NX` · 파생 합계 · 슬라이딩 만료 · 원자적 갱신 · 무결성 검사

### 백지 구현

#### 구현 목표

하나의 Redis Lua 실행으로 슬라이딩 윈도 금액 카운터를 갱신한다. 만료된 항목을 제거하고, 동일 이벤트는 중복 반영하지 않으며, 성공 후 현재 합계를 반환한다.

#### 인터페이스 또는 함수 시그니처

```lua
-- KEYS[1]: 시간순 entry를 보관할 sorted set
-- KEYS[2]: 정확한 현재 합계를 보관할 string
--
-- ARGV[1]: 현재 시각(epoch millis)
-- ARGV[2]: window millis
-- ARGV[3]: event id
-- ARGV[4]: 양의 금액 decimal text
-- ARGV[5]: key retention millis

-- 직접 구현
```

#### 입력과 출력

- 입력: 현재 시각, 창 길이, 이벤트 ID, 금액, 보존 시간
- 출력: 갱신 뒤 현재 합계와 신규 반영 여부
- 저장 구조가 손상되었거나 숫자 계약을 위반하면 Redis error

#### 반드시 만족해야 할 조건

- 창 밖의 member를 제거하고 그 금액만큼 합계를 줄인다.
- 이벤트 ID와 금액을 함께 포함한 member는 같은 timestamp에서도 고유해야 한다.
- 같은 member의 재처리는 합계를 다시 올리지 않는다.
- 새 member가 실제 추가된 경우에만 합계를 증가시킨다.
- 결과 합계는 정확 정수 범위 안이며 음수가 될 수 없다.
- entry가 모두 사라지면 orphan `sum`을 남기지 않는다.
- 비어 있지 않은 키에는 창보다 긴 유한 TTL을 적용한다.
- 잘못된 Redis type, 파싱 불가능한 member, 합계 underflow를 조용히 보정하지 않는다.

#### 경계 조건

- 비어 있는 최초 상태
- 만료 항목이 없는 상태
- 모든 항목이 만료되는 상태
- 같은 이벤트의 재전달
- 서로 다른 이벤트가 같은 timestamp를 갖는 상태
- 합계가 정확히 상한에 도달하는 상태
- cutoff와 score가 같은 member의 포함 규칙

#### 실패 조건

- `entries` 또는 `sum`의 Redis type 불일치
- member에서 금액을 복원할 수 없음
- 저장 합계가 없거나 정수가 아님
- 만료 금액의 합이 저장 합계보다 큼
- 신규 합계가 정확 정수 상한을 초과

#### 필요한 제약

- 전체 살아 있는 entry를 매번 모두 합산하는 구현은 피한다.
- 네트워크 왕복을 여러 번 나누지 않는다.
- Redis 트랜잭션과 애플리케이션 잠금 대신 한 Lua 호출로 끝낸다.
- 실제 프로젝트 스크립트를 복사하지 않고 단일 차원으로 축소한다.

### 구현 후 자가 검증

- [ ] 최초 추가 후 합계와 entry 수가 일치한다.
- [ ] 같은 event ID와 금액을 재처리해도 합계가 증가하지 않는다.
- [ ] 같은 timestamp의 서로 다른 이벤트가 모두 보존된다.
- [ ] cutoff 경계의 포함·제외 규칙이 테스트와 일치한다.
- [ ] 일부 만료 후 합계가 정확히 감소한다.
- [ ] 전부 만료되면 `entries`와 `sum`이 함께 사라지거나 일관된 빈 상태가 된다.
- [ ] 잘못된 member나 underflow에서는 어떤 mutation도 완료된 정상 결과로 보이지 않는다.
- [ ] 합계는 member 금액의 합과 항상 같다는 invariant를 설명할 수 있다.
- [ ] 복잡도가 제거된 항목 수를 `r`이라 할 때 대략 `O(r log n)`임을 설명할 수 있다.

### 구현 후 설명할 것

1. 정렬 집합 단독 합산과 파생 `sum` 조합의 읽기 비용·무결성 trade-off
2. member에 이벤트 ID와 금액을 함께 넣은 이유
3. score가 동일해도 member 고유성으로 동시 이벤트를 잃지 않는 방식
4. TTL을 창보다 길게 두는 이유와 너무 길거나 짧을 때의 문제
5. 파생 합계 불일치를 자동 재계산하지 않고 fail-closed로 처리하는 이유

### 원본 확인 위치

- Thread 08
- 커밋: `feat(counter): expose sliding window counters`
- 파일: `src/main/resources/scripts/sliding-window.lua`
- 클래스: `SlidingWindowCounter`, `LimitKeys`, `LimitType`
- 테스트: `SlidingWindowScriptTest`
- 관련 Thread: 10, 11, 12, 14

---

## P07. [Thread 10 / `feat(reservation): define atomic reservation entrypoint`] 한도 확인과 용량 예약을 한 번에 수행하는 원자적 승인

### 면접 질문

"현재 사용량을 읽고 한도를 확인한 뒤 예약을 쓴다"는 세 단계를 애플리케이션 코드로 나누면 어떤 race가 생기며, 이 프로젝트는 왜 하나의 Redis Lua 스크립트로 묶었나요?

꼬리 질문:

- 이미 확정된 사용량만 보지 않고 활성 예약까지 포함해야 하는 이유는 무엇입니까?
- 100 한도에서 10짜리 요청 20개가 동시에 오면 무엇을 보장해야 합니까?
- 동일 지문 재시도와 같은 `betId`의 다른 요청을 어떻게 구분합니까?
- 진단 endpoint의 승인 결과를 실제 결제 권한으로 사용할 수 없는 이유는 무엇입니까?

### 30초 모범 답변

읽기와 쓰기를 분리하면 여러 요청이 같은 여유 용량을 동시에 보고 모두 승인되는 TOCTOU race가 생깁니다. 예약 스크립트는 만료 footprint 정리, committed와 active 합산, 후보 한도 검사, lifecycle과 active footprint 기록을 한 Redis 실행으로 처리합니다. 그래서 승인된 lifecycle만 있고 합계가 빠진 순간이나 그 반대가 없습니다. 같은 지문은 저장 결과를 replay하고 다른 지문은 conflict로 처리하며, Redis 오류 때 기본 승인으로 우회하지 않습니다.

### 답변 핵심 키워드

TOCTOU · check-and-set · Redis Lua 원자성 · committed + active · lifecycle/footprint 동시 생성 · replay · conflict · fail-closed

### 백지 구현

#### 구현 목표

단일 통화·단일 일간 한도로 축소한 예약 승인 Lua 스크립트를 작성한다. 동시에 실행된 요청도 총 승인 금액이 한도를 넘지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```lua
-- KEYS[1]: bet별 lifecycle hash
-- KEYS[2]: 사용자의 active entry sorted set
-- KEYS[3]: 사용자의 active sum string
-- KEYS[4]: 사용자의 committed entry sorted set
-- KEYS[5]: 사용자의 committed sum string
--
-- ARGV에는 version, now, window, lease, retention,
-- fingerprint, betId, amount, limit가 주어진다.

-- 직접 구현
```

#### 입력과 출력

- 입력: 요청 식별자와 금액, 현재 시각, 한도, lease/retention, 관련 Redis 키
- 출력: `APPROVED`, `REJECTED`, `REPLAYED`, `CONFLICT` 중 하나와 필요한 최소 데이터
- 상태 손상이나 잘못된 인자는 Redis error

#### 반드시 만족해야 할 조건

- committed 창을 먼저 정리해 현재 합계를 계산한다.
- 만료된 active 예약은 승인 계산 전에 제거한다.
- `committed + active + candidate <= limit`일 때만 승인한다.
- 승인 시 lifecycle과 active entry·sum이 모두 같은 호출에서 기록된다.
- 거절 시 active 용량을 만들지 않지만 bounded replay에 필요한 결과는 남긴다.
- 같은 `betId`와 같은 fingerprint는 기존 결과를 replay한다.
- 같은 `betId`와 다른 fingerprint는 conflict다.
- 수치·타입·footprint 오류에서 기본 승인하지 않는다.
- retention은 lease보다 길어야 한다.

#### 경계 조건

- 현재 사용량 0
- 후보를 더하면 정확히 한도
- 후보를 더하면 한도보다 1 초과
- 동일 요청 재시도
- 동일 `betId`에 금액만 다른 요청
- 막 만료된 active 예약
- committed와 active 중 하나만 존재하는 상태

#### 실패 조건

- Redis key type 불일치
- 저장 합계나 member 손상
- 정확 정수 범위 초과
- lifecycle과 active footprint 불일치
- retention/lease 계약 위반

#### 필요한 제약

- 애플리케이션의 `synchronized`, 분산 락, 여러 Redis round trip을 사용하지 않는다.
- 한 차원 한도만 구현한다.
- 실제 프로젝트 응답 JSON이나 전체 키 배열을 복사하지 않는다.
- 동시성 테스트가 가능한 결정적 결과를 반환한다.

### 구현 후 자가 검증

- [ ] 빈 상태의 정상 요청은 승인되고 lifecycle과 active 합계가 함께 생긴다.
- [ ] 한도와 정확히 같은 후보는 승인되고 1 초과 후보는 거절된다.
- [ ] 같은 요청 재시도는 합계를 다시 올리지 않고 replay된다.
- [ ] 같은 `betId`의 다른 fingerprint는 conflict이며 기존 합계가 유지된다.
- [ ] 동시 20개 요청에서 승인 금액의 합이 한도를 넘지 않는다.
- [ ] 만료 예약 정리 후 회수된 용량을 새 요청이 사용할 수 있다.
- [ ] 거절된 요청은 active entry를 만들지 않는다.
- [ ] 실패 경로에서 lifecycle만 또는 합계만 바뀌는 부분 mutation이 없다.
- [ ] 승인 뒤 `activeSum == active entries의 금액 합` invariant가 유지된다.

### 구현 후 설명할 것

1. Lua 원자성과 분산 락 방식의 차이
2. active 예약을 committed와 함께 한도에 포함한 이유
3. lifecycle과 파생 footprint를 한 호출에서 만드는 이유
4. 거절 결과도 replay용으로 보존하는 trade-off
5. bet-scoped 키와 user-scoped 키를 함께 다루어 Redis Cluster를 지원하지 않는 설계 영향

### 원본 확인 위치

- Thread 10
- 커밋: `feat(reservation): define atomic reservation entrypoint`
- 파일: `src/main/resources/scripts/risk-reserve.lua`
- 클래스: `ReservationScriptRequest`, `RedisRiskReservationStore`
- 테스트: `RiskReserveScriptTest`, `ReservationRollingCapacityScriptTest`
- 관련 Thread: 01, 08, 09, 11, 12, 17

---

## P08. [Thread 06 / `test(pattern): verify sudden stake boundaries`] 오버플로 없이 홀수·짝수 중앙값 임계값 비교하기

### 면접 질문

최근 stake 중앙값의 배수 이상이면 급격한 증가로 판단한다고 할 때, 짝수 개 표본의 중앙값과 큰 금액을 정확하게 비교하는 방법을 설명해 주세요.

꼬리 질문:

- 표본이 lookback 수보다 적으면 왜 판단하지 않았습니까?
- `(a+b)/2`를 먼저 계산하면 어떤 정밀도 문제가 생깁니까?
- `candidate * 2 >= (a+b) * multiplier`로 바꾸면 모든 문제가 해결됩니까?
- 최근 표본 선택과 금액 정렬은 각각 어떤 순서를 사용해야 합니까?

### 30초 모범 답변

먼저 시간순으로 가장 최근 lookback개를 고르고, 그 복사본을 금액순으로 정렬해 중앙값을 구합니다. 표본이 부족하면 규칙을 적용하지 않습니다. 홀수는 가운데 값의 배수와 비교하고, 짝수는 두 가운데 값의 평균을 정확히 표현해야 하므로 정수 나눗셈으로 버리지 않고 동치 비교식을 사용합니다. 다만 양쪽 곱셈도 상한을 넘을 수 있어 사전 범위 검사나 더 넓은 정확 표현을 사용해야 하며, 프로젝트의 Lua 경계에서는 십진 문자열 산술로 이를 피했습니다.

### 답변 핵심 키워드

최근 lookback · 복사 후 정렬 · 홀수 중앙값 · 짝수 평균 · 정수 절삭 금지 · 동치 비교 · 곱셈 오버플로 · 정확한 문자열 산술

### 백지 구현

#### 구현 목표

최근 stake 목록으로 급격한 증가 여부를 판정한다. 충분한 표본이 있을 때만 판단하고, 짝수 중앙값과 배수 비교에서 오버플로·절삭이 없어야 한다.

#### 인터페이스 또는 함수 시그니처

```java
public static boolean isSuddenIncrease(
    long candidate,
    List<Long> chronologicalStakes,
    int lookback,
    int multiplier) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 양수 후보 금액, 오래된 순서에서 최신 순서로 정렬된 과거 금액, lookback, 2 이상의 배수
- 출력: 임계값 이상이면 `true`, 그 외 `false`

#### 반드시 만족해야 할 조건

- 과거 표본이 lookback보다 적으면 `false`다.
- 전체 과거가 아니라 가장 최근 lookback개만 사용한다.
- 입력 목록을 변경하지 않는다.
- 홀수 표본은 가운데 값, 짝수 표본은 가운데 두 값의 정확한 평균을 기준으로 한다.
- 임계값과 정확히 같은 후보는 match다.
- 중간 덧셈·곱셈에서 `long` 오버플로와 `2^53-1` 도메인 초과를 허용하지 않는다.
- 부동소수점 근사로 비교하지 않는다.

#### 경계 조건

- 표본 수가 lookback보다 하나 적거나 정확히 같은 경우
- 홀수 lookback과 짝수 lookback
- 중앙값이 0인 저장 사실을 허용할지 명시
- 임계값보다 1 작거나 정확히 같은 후보
- 모든 값이 정확 정수 상한에 가까운 경우
- 이미 정렬된 입력과 역순 입력

#### 실패 조건

- candidate가 양수가 아님
- lookback이 1 미만
- multiplier가 2 미만
- null 표본 또는 음수 표본
- 선택한 정확 비교 전략으로 표현할 수 없는 값

#### 필요한 제약

- `double`과 평균의 정수 절삭을 사용하지 않는다.
- `chronologicalStakes.sort(...)`처럼 호출자 목록을 변경하지 않는다.
- `BigInteger`를 사용할지 여부는 면접 시작 시 정하고, 사용하지 않는 경우 오버플로 안전한 비교를 직접 설계한다.
- 정렬 기반 `O(k log k)` 풀이면 충분하다.

### 구현 후 자가 검증

- [ ] 표본 부족 시 후보가 커도 match하지 않는다.
- [ ] 홀수 중앙값의 바로 아래와 정확한 임계값을 구분한다.
- [ ] 짝수 중앙값이 정수가 아닌 경우도 절삭 없이 판단한다.
- [ ] 최근 lookback 밖의 극단값이 결과에 영향을 주지 않는다.
- [ ] 입력 목록의 순서와 내용이 유지된다.
- [ ] 상한 근처 값에서 음수 wraparound나 잘못된 `true`가 나오지 않는다.
- [ ] equality가 포함되는 임계 비교인지 테스트와 설명이 일치한다.
- [ ] 시간 복잡도 `O(k log k)`, 추가 공간 `O(k)`를 설명할 수 있다.

### 구현 후 설명할 것

1. 시간순 최근 선택과 금액순 중앙값 계산을 분리한 이유
2. 표본 부족을 정상적인 "미판정"으로 본 이유
3. 짝수 중앙값을 정수 나눗셈하지 않는 이유
4. Java와 Redis Lua에서 사용할 수 있는 정확 산술 도구의 차이
5. 정렬 대신 선택 알고리즘을 사용할 때 얻는 복잡도 이점과 구현 복잡성

### 원본 확인 위치

- Thread 06
- 커밋: `test(pattern): verify sudden stake boundaries`
- 파일: `src/main/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRule.java`
- 클래스: `SuddenStakeIncreaseRule`
- 테스트: `SuddenStakeIncreaseRuleTest`
- 관련 Thread: 02, 07, 10

---

## P09. [Thread 10 / `feat(reservation): evaluate currency stake patterns`] 동시 후보가 패턴 임계값을 우회하지 못하게 하는 active 사실

### 면접 질문

빠른 반복 베팅이나 같은 선택 반복 규칙을 확정 이력만으로 검사하면, 동시에 들어온 후보들이 어떻게 모두 임계값을 우회할 수 있습니까?

꼬리 질문:

- 후보 자신을 count에 포함하는 시점과 비교 연산은 어떻게 정의했습니까?
- `BLOCK`과 `SUSPECT`·`REVIEW`의 상태 변경 차이는 무엇입니까?
- 반복 선택 용량이 통화별이 아니라 선택 ID별로 공유되는 이유는 무엇입니까?
- 패턴 평가를 진단 Java 규칙과 예약 Lua 양쪽에 둔 비용은 무엇입니까?

### 30초 모범 답변

확정 이력만 보면 동시에 실행된 요청은 서로를 보지 못해 모두 같은 낮은 count를 기준으로 승인될 수 있습니다. 권위 승인에서는 confirmed history와 아직 만료되지 않은 active reservation을 합쳐 후보까지 포함한 값을 한 Lua 실행 안에서 검사합니다. `BLOCK`이면 거절 lifecycle만 남기고 active footprint는 만들지 않으며, `SUSPECT`와 `REVIEW`는 플래그를 반환하되 용량을 예약합니다. 반복 선택은 금액 통화와 무관한 선택 행위이므로 통화 간에도 같은 count를 공유합니다.

### 답변 핵심 키워드

confirmed + active + candidate · 동시 임계값 우회 · 후보 포함 · BLOCK vs advisory · 선택 ID 전역성 · 권위 경계

### 백지 구현

#### 구현 목표

빠른 베팅 `BLOCK` 규칙 하나로 축소한 원자적 예약 스크립트를 작성한다. 병렬 후보가 서로를 무시해 임계값 이상 승인되지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```lua
-- KEYS[1]: confirmed bet history sorted set
-- KEYS[2]: active reservation sorted set
-- KEYS[3]: candidate lifecycle hash
--
-- ARGV에는 now, window, lease, retention,
-- betId, fingerprint, threshold가 주어진다.

-- 직접 구현
```

#### 입력과 출력

- 입력: 시간 창, 임계값, candidate identity
- 출력: `APPROVED`, `REJECTED`, `REPLAYED`, `CONFLICT`
- 손상된 key나 lifecycle: Redis error

#### 반드시 만족해야 할 조건

- confirmed와 active에서 창 밖 또는 만료된 사실을 정리한다.
- 현재 count에 후보 1개를 포함해 임계값 도달 여부를 판단한다.
- 후보를 포함한 count가 임계값에 도달하면 `BLOCK`으로 거절한다.
- 승인 시 active set과 lifecycle을 함께 만든다.
- 거절 시 active set에는 추가하지 않고 replay 가능한 거절 lifecycle을 남긴다.
- 같은 fingerprint 재시도는 기존 결과를 반환하고 count를 바꾸지 않는다.
- 다른 fingerprint의 같은 `betId`는 conflict다.
- 병렬 요청에서도 임계값 직전까지만 승인된다.

#### 경계 조건

- count 0
- 후보 포함 count가 임계값보다 1 작음
- 후보 포함 count가 정확히 임계값
- 동일 요청 재시도
- 만료 active가 섞인 상태
- 같은 timestamp의 여러 후보

#### 실패 조건

- 잘못된 Redis type
- 임계값 또는 시간 값 오류
- lifecycle과 active set 불일치
- active 만료 정보를 판별할 수 없는 상태

#### 필요한 제약

- 하나의 규칙과 하나의 사용자만 다룬다.
- 애플리케이션 잠금이나 별도 count read를 사용하지 않는다.
- 승인/거절의 패턴 JSON 전체는 구현하지 않는다.
- 20~30분 안에 스크립트와 병렬 테스트 전략을 설명한다.

### 구현 후 자가 검증

- [ ] threshold가 5일 때 빈 상태에서 순차 후보 4개만 승인된다.
- [ ] 다섯 번째 후보는 거절되고 active count는 4다.
- [ ] 20개 병렬 요청에서도 승인 수가 4를 넘지 않는다.
- [ ] 같은 승인 요청 replay가 active count를 늘리지 않는다.
- [ ] 같은 거절 요청 replay가 계속 같은 거절을 반환한다.
- [ ] 만료 active를 정리한 뒤 용량이 회복된다.
- [ ] 거절 lifecycle과 active footprint가 동시에 존재하는 잘못된 상태가 없다.
- [ ] confirmed와 active를 중복 집계하는 경우가 없는지 확인했다.

### 구현 후 설명할 것

1. 진단용 순수 규칙과 권위 승인용 Lua 규칙을 별도로 유지한 이유
2. 후보 포함 비교를 `>= threshold`로 정의한 이유
3. active 사실을 패턴 용량으로 간주하는 장점과 취소 시 정리 책임
4. 반복 선택 규칙을 통화 중립으로 두는 도메인 판단
5. 두 구현의 규칙 drift를 테스트로 줄이는 방법

### 원본 확인 위치

- Thread 10
- 커밋: `feat(reservation): evaluate currency stake patterns`
- 파일: `src/main/resources/scripts/risk-reserve.lua`
- 테스트: `ReservationRapidPatternScriptTest`, `ReservationRepeatedPatternScriptTest`, `ReservationSuddenPatternScriptTest`
- 관련 Thread: 06, 07, 11, 12

---

## P10. [Thread 07 / `feat(snapshot): define combined risk facts`] 한 번의 Redis 실행으로 만드는 원자적 진단 스냅샷

### 면접 질문

진단은 상태를 변경하지 않는데도 왜 여러 Redis GET을 조합하지 않고 하나의 Lua 스냅샷을 사용했나요?

꼬리 질문:

- committed, active, override를 서로 다른 시점에 읽으면 어떤 불가능한 조합이 생깁니까?
- 스냅샷 읽기에서 만료 reservation 정리를 수행하는 것이 순수 read와 어떻게 다릅니까?
- Lua JSON 결과의 큰 정수를 문자열로 내보낸 이유는 무엇입니까?
- 한 slot이 손상된 경우 나머지 정상 slot만으로 판단해도 됩니까?

### 30초 모범 답변

여러 GET 사이에 예약·커밋·만료가 일어나면 committed와 active가 서로 다른 시점의 값이 되어 이중 계산하거나 누락할 수 있습니다. 스냅샷 스크립트는 필요한 한도와 패턴 사실을 한 Redis 실행에서 읽고, 만료 footprint도 같은 실행에서 정리합니다. 숫자는 JSON 정밀도 손실을 피하려고 canonical decimal string으로 전달하고 Java mapper가 version, 필수 slot, 범위와 결과 shape를 검증합니다. 필수 사실이 손상되면 부분 판단하지 않고 진단을 실패시킵니다.

### 답변 핵심 키워드

atomic snapshot · torn read 방지 · committed/active/override · side-effecting read · canonical decimal string · strict mapper · 필수 slot fail-closed

### 백지 구현

#### 구현 목표

하나의 스크립트가 반환한 JSON 문자열을 엄격하게 도메인 스냅샷으로 변환한다. 숫자 slot이 누락되거나 비정규 문자열이면 실패해야 한다.

#### 인터페이스 또는 함수 시그니처

```java
enum Dimension {
  DAILY, WEEKLY, MONTHLY, SELECTIONS
}

record LimitSlot(String committed, String active, String override, String error) {}

record SnapshotWire(
    String version,
    Map<String, LimitSlot> limits) {}

record LimitValue(long committed, long active, Long override) {}

public static Map<Dimension, LimitValue> decode(String json) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: Lua 스크립트가 반환한 JSON 문자열
- 출력: 모든 필수 dimension을 가진 불변 map
- 형식·버전·숫자·slot 오류: 예외

#### 반드시 만족해야 할 조건

- 지원하는 version만 허용한다.
- 네 dimension이 모두 존재해야 한다.
- `committed`, `active`는 canonical non-negative decimal string이다.
- override는 없거나 canonical non-negative decimal string이다.
- 선행 0, 부호, 소수점, 공백, 상한 초과를 거부한다.
- slot에 error가 있으면 해당 값을 임의 기본값으로 대체하지 않는다.
- 반환 map과 값은 외부 변경으로부터 안전하다.
- `committed + active`도 정확 정수 상한을 넘지 않아야 한다.

#### 경계 조건

- override 없음과 0 override
- 모든 합계 0
- 합계가 정확히 상한
- 필수 dimension 하나 누락
- `"0"`, `"00"`, `"-1"`, `"+1"`, `"1.0"`, `" 1"`
- 알 수 없는 추가 field

#### 실패 조건

- null/빈/잘못된 JSON
- 지원하지 않는 version
- slot 누락이나 error
- 정규 형식이 아닌 수치
- 합계 오버플로

#### 필요한 제약

- JSON 숫자를 `double`로 읽었다가 `long`으로 변환하지 않는다.
- 누락 slot을 0으로 채우지 않는다.
- Lua 스크립트 자체는 구현하지 않고 wire 경계에 집중한다.
- 파서 라이브러리 사용은 허용하되 검증 책임을 라이브러리에 넘기지 않는다.

### 구현 후 자가 검증

- [ ] 네 dimension의 정상 wire가 불변 map으로 변환된다.
- [ ] override 없음과 override 0을 구분한다.
- [ ] 선행 0, 부호, 소수, 공백 수치가 실패한다.
- [ ] 필수 slot 누락과 slot error가 실패한다.
- [ ] `committed + active` 상한 초과가 실패한다.
- [ ] 알 수 없는 추가 field를 허용할지 거부할지 정책이 명확하다.
- [ ] 실패한 decode에서 부분 snapshot이 반환되지 않는다.
- [ ] 호출자가 반환 map을 변경할 수 없다.

### 구현 후 설명할 것

1. 다중 GET과 Lua snapshot의 일관성 차이
2. 진단 read가 lazy expiry cleanup을 수행하는 설계의 장단점
3. 숫자를 문자열로 전송하고 Java에서 다시 검증하는 이유
4. 부분 slot 오류를 허용하지 않은 이유
5. 스냅샷은 원자적이어도 예약을 만들지 않으므로 권위 승인으로 쓸 수 없다는 점

### 원본 확인 위치

- Thread 07
- 커밋: `feat(snapshot): define combined risk facts`
- 파일: `src/main/resources/scripts/risk-snapshot.lua`
- 클래스: `LimitSnapshot`, `PatternSnapshot`, `RiskSnapshot`, `RiskSnapshotWire`, `RiskSnapshotWireMapper`, `RedisRiskSnapshotReader`
- 관련 Thread: 01, 02, 06, 10, 12
