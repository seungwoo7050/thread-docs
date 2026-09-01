# 예약 상태 머신·파생 집계 무결성·bounded history

이 문서는 예약이 승인된 뒤 commit·release·expiry로 이동하는 상태 머신, lifecycle과 여러 파생 footprint 사이의 invariant, 패턴 이력의 시간·개수 기반 보존 정책을 다룬다. Thread 11의 전이와 Thread 12의 하드닝은 별개 문제가 아니라 "상태를 바꾸기 전에 기존 상태 전체를 신뢰할 수 있는가"라는 한 흐름으로 연결된다.

---

## P11. [Thread 11 / `feat(reservation): define atomic commit lifecycle`] replay를 포함한 예약 commit·release·expiry 상태 머신

### 면접 질문

예약 lifecycle의 commit과 release를 멱등하게 만들면서도, 잘못된 토큰이나 이미 확정된 예약의 release는 어떻게 구분했나요?

꼬리 질문:

- 최초 적용과 replay를 둘 다 성공으로 보되 결과 enum을 나눈 이유는 무엇입니까?
- 만료 시각과 정확히 같은 시각에 commit이 오면 어떤 상태가 되어야 합니까?
- lifecycle을 즉시 삭제하지 않고 `EXPIRED`, `RELEASED`, `REJECTED` tombstone으로 보존하는 이유는 무엇입니까?
- release의 내부 transition 결과와 HTTP에서의 idempotent success가 다를 수 있는 이유는 무엇입니까?

### 30초 모범 답변

commit은 `RESERVED`이고 lease 안이며 토큰이 일치할 때만 active footprint를 제거하고 committed counter와 확정 이력을 반영한 뒤 `COMMITTED`로 바뀝니다. 같은 토큰으로 다시 commit하면 아무것도 더하지 않고 replay 성공을 반환합니다. release는 `RESERVED`만 active 용량을 회수하고, 이미 `RELEASED`면 replay, `COMMITTED`면 conflict입니다. 만료·거절·release 상태는 retention 동안 tombstone으로 남겨 재시도의 의미를 결정하고, TTL이 끝난 뒤에만 replay 기억을 잃습니다.

### 답변 핵심 키워드

상태 머신 · token-bound transition · APPLIED vs REPLAYED · tombstone · lease/retention · active→committed 이동 · idempotency

### 백지 구현

#### 구현 목표

한 예약의 lifecycle과 active/committed 금액을 메모리 모델로 축소해 commit·release·expiry transition을 구현한다. 같은 명령의 재시도는 수치를 다시 변경하지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```java
enum State {
  RESERVED, COMMITTED, RELEASED, EXPIRED, REJECTED
}

enum Transition {
  APPLIED, REPLAYED, CONFLICT, NOT_FOUND, TOMBSTONED, EXPIRED
}

record Reservation(
    String token,
    State state,
    long amount,
    Instant expiresAt) {}

record Capacity(long active, long committed) {}

record Result(
    Reservation reservation,
    Capacity capacity,
    Transition transition) {}

public static Result commit(
    Reservation reservation,
    Capacity capacity,
    String suppliedToken,
    Instant now) {
  // 직접 구현
}

public static Result release(
    Reservation reservation,
    Capacity capacity,
    Instant now) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 현재 lifecycle, active/committed 수치, 토큰, 처리 시각
- 출력: 새 lifecycle, 새 수치, transition 결과
- 존재하지 않는 lifecycle은 `reservation == null`로 표현

#### 반드시 만족해야 할 조건

- commit은 올바른 토큰의 유효한 `RESERVED`에서만 최초 적용된다.
- commit 최초 적용 시 active는 amount만큼 감소하고 committed는 amount만큼 증가한다.
- `COMMITTED`에 대한 같은 commit은 `REPLAYED`이며 수치를 바꾸지 않는다.
- 토큰 불일치는 상태와 관계없이 기존 데이터를 바꾸지 않고 `CONFLICT`다.
- `now >= expiresAt`인 `RESERVED` commit은 `EXPIRED`로 tombstone 처리하고 active만 회수한다.
- release 최초 적용은 `RESERVED → RELEASED`이며 active만 회수한다.
- `RELEASED` 재호출은 `REPLAYED`, `COMMITTED` release는 `CONFLICT`다.
- `EXPIRED`와 `REJECTED`는 commit 대상이 아니며 committed를 바꾸지 않는다.
- active와 committed는 음수가 되거나 정확 정수 상한을 넘지 않는다.
- 실패 또는 replay에서 수치가 두 번 변하지 않는다.

#### 경계 조건

- `now`가 `expiresAt` 직전, 정확히 같음, 직후
- amount와 active가 정확히 같음
- active가 amount보다 작은 손상 상태
- commit을 두 번 호출
- release를 두 번 호출
- commit 뒤 release, release 뒤 commit
- lifecycle 없음

#### 실패 조건

- 토큰 null/불일치
- lifecycle과 active 수치 불일치
- committed 덧셈 범위 초과
- 알 수 없는 상태
- 필수 시각 누락

#### 필요한 제약

- 상태 전이는 순수 함수로 작성해 테스트 가능하게 한다.
- HTTP status mapping은 구현하지 않는다.
- 영속화·락은 다루지 않고 전이 의미와 수치 invariant에 집중한다.
- 자동으로 손상 수치를 0으로 보정하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 commit이 `RESERVED → COMMITTED`로 한 번만 적용된다.
- [ ] 두 번째 commit은 replay이며 active/committed가 그대로다.
- [ ] 잘못된 토큰은 어떤 수치도 바꾸지 않는다.
- [ ] 만료 경계 `now == expiresAt`에서 정책과 테스트가 일치한다.
- [ ] 정상 release가 active만 회수하고 committed는 바꾸지 않는다.
- [ ] 두 번째 release가 replay다.
- [ ] commit 뒤 release는 conflict이며 committed가 유지된다.
- [ ] release 뒤 commit은 committed를 증가시키지 않는다.
- [ ] 모든 성공 결과에서 `active >= 0`, `committed >= 0`이다.
- [ ] 최초 적용과 replay를 호출자가 구분할 수 있다.

### 구현 후 설명할 것

1. `APPLIED`와 `REPLAYED`를 모두 성공으로 보면서 구분한 이유
2. 토큰을 commit에 요구하고 release에는 요구하지 않는 계약의 의미
3. tombstone을 retention 동안 보관하는 비용과 replay 안정성
4. lease 만료 경계의 포함 규칙을 명시해야 하는 이유
5. 내부 transition과 HTTP idempotency mapping을 분리한 이유

### 원본 확인 위치

- Thread 11
- 커밋: `feat(reservation): define atomic commit lifecycle`
- 파일: `src/main/resources/scripts/risk-commit.lua`, `src/main/resources/scripts/risk-release.lua`
- 클래스: `RiskReservationStore`, `RedisRiskReservationStore`, `ReservationTransition`
- 테스트: `ReservationCommitLifecycleScriptTest`, `ReservationReleaseLifecycleScriptTest`, `ReservationExpiryCleanupScriptTest`
- 관련 Thread: 09, 10, 12, 14, 17

---

## P12. [Thread 12 / `feat(snapshot): validate active aggregate consistency`] validate-before-mutate와 fail-closed 파생 집계 검증

### 면접 질문

active reservation의 lifecycle, 사용자별 bet set, 금액 entry와 sum, 선택 entry와 sum, 선택별 set이 함께 존재할 때 어떤 invariant를 검사해야 하며, 왜 만료 cleanup 전에 전부 검사했나요?

꼬리 질문:

- 하나의 selection footprint가 누락된 만료 예약을 나머지만 정리하면 어떤 문제가 생깁니까?
- 저장된 `sum`이 entry를 합산한 값보다 크면 서비스가 다시 계산해 고쳐도 됩니까?
- 글로벌 active gauge는 사용자별 active count와 어떤 관계만 검사할 수 있습니까?
- Lua 스크립트가 중간에 error를 반환하면 앞선 Redis mutation은 어떻게 됩니까?

### 30초 모범 답변

파생 구조는 lifecycle을 원본으로 보고 각 active bet에 금액·선택 entry와 선택별 footprint가 정확히 하나씩 있는지, entry 합과 저장 sum이 같은지, cardinality가 맞는지 먼저 계산합니다. 검증 중에는 mutation하지 않고 cleanup plan만 만듭니다. 하나라도 누락되거나 orphan이면 스크립트를 error로 끝내 Redis의 원자적 rollback 성질에 기대고, 잘못된 상태 일부만 지워 정상처럼 보이게 하지 않습니다. 자동 재계산은 동시 트래픽에서 손상을 은폐할 수 있어 운영 복구 영역으로 남깁니다.

### 답변 핵심 키워드

source vs derived state · cardinality · sum equality · footprint completeness · orphan detection · preflight plan · no partial mutation · fail-closed

### 백지 구현

#### 구현 목표

한 사용자의 active reservation 파생 상태를 검증하고, 만료되었거나 더 이상 active가 아닌 항목의 cleanup 대상만 계획한다. 모든 검증이 성공하기 전에는 입력 상태를 변경하지 않는다.

#### 인터페이스 또는 함수 시그니처

```java
record Lifecycle(
    String betId,
    String userId,
    String currency,
    long amount,
    List<String> selections,
    State state,
    Instant expiresAt) {}

record AmountEntry(String betId, long amount) {}
record SelectionEntry(String betId, int count) {}

record ActiveState(
    Set<String> activeBetIds,
    Map<String, Lifecycle> lifecycles,
    Map<String, List<AmountEntry>> stakeEntriesByCurrency,
    Map<String, Long> stakeSumsByCurrency,
    List<SelectionEntry> selectionEntries,
    long selectionSum,
    Map<String, Set<String>> betIdsBySelection,
    long globalActiveGauge) {}

record CleanupPlan(List<String> betIdsToRemove) {}

public static CleanupPlan validateAndPlan(
    String userId,
    ActiveState state,
    Instant now) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 한 사용자 범위의 lifecycle과 모든 파생 구조, 현재 시각
- 출력: 검증이 성공한 경우에만 cleanup 대상 bet ID 목록
- 불일치: 명시적 예외, 입력은 변경되지 않음

#### 반드시 만족해야 할 조건

- `activeBetIds`의 모든 ID에 lifecycle이 존재한다.
- lifecycle의 userId가 검사 사용자와 일치한다.
- 각 bet의 통화별 금액 entry가 정확히 하나 존재하고 금액이 lifecycle과 같다.
- 각 bet의 selection entry가 정확히 하나 존재하고 count가 lifecycle 선택 수와 같다.
- lifecycle의 모든 선택마다 해당 selection set에 bet ID가 존재한다.
- 통화별 저장 sum은 해당 entry 금액 합과 정확히 같다.
- selection sum은 selection entry count 합과 정확히 같다.
- entry cardinality와 active bet cardinality 관계가 일관된다.
- 검사 사용자 active 수는 global gauge보다 클 수 없다.
- active set에 없는 orphan entry와 orphan selection footprint를 탐지한다.
- `state != RESERVED`이거나 만료된 항목만 cleanup plan에 들어간다.
- 검증 중 입력 컬렉션을 변경하지 않는다.

#### 경계 조건

- active bet 0개
- 한 통화만 존재
- 여러 통화가 섞인 상태
- 한 bet에 여러 선택
- lifecycle은 있지만 한 footprint만 누락
- 저장 sum만 1 크게 오염
- global gauge가 사용자 active 수와 같거나 더 큼
- `expiresAt == now`

#### 실패 조건

- lifecycle 누락 또는 사용자 불일치
- 금액·선택 entry 누락, 중복, 값 불일치
- sum 불일치
- per-selection cardinality 불일치
- orphan entry/footprint
- global gauge가 사용자 active 수보다 작음
- 수치 합산 오버플로

#### 필요한 제약

- 검증과 실제 cleanup을 한 메서드에서 섞지 않는다.
- 검증 성공 후 적용할 plan만 반환한다.
- 컬렉션 전체를 스캔하는 `O(n + totalSelections)` 풀이면 충분하다.
- 자동 수정이나 "최소값 사용" 방식으로 진행하지 않는다.

### 구현 후 자가 검증

- [ ] 완전한 정상 상태는 올바른 cleanup plan을 반환한다.
- [ ] footprint 하나를 제거하면 실패하고 입력은 그대로다.
- [ ] sum을 1만 바꿔도 실패한다.
- [ ] orphan entry와 orphan selection set member를 잡는다.
- [ ] 만료되지 않은 `RESERVED`는 plan에 들어가지 않는다.
- [ ] 만료 또는 비-`RESERVED` 항목은 검증 성공 후에만 plan에 들어간다.
- [ ] active가 없는 빈 상태에서 orphan sum이 있으면 실패한다.
- [ ] global gauge가 더 큰 것은 다른 사용자 때문일 수 있어 허용하고, 더 작은 것은 실패한다.
- [ ] 실패 경로에서 부분 삭제나 lifecycle 전이가 없다.
- [ ] 합산 과정의 정확 정수 상한을 검사한다.

### 구현 후 설명할 것

1. lifecycle을 기준 사실로 보고 나머지를 파생 구조로 본 이유
2. 모든 검증 뒤에 mutation을 배치하는 이유
3. 자동 self-healing과 fail-closed 사이의 운영 trade-off
4. 글로벌 gauge를 사용자 범위에서 완전히 검증할 수 없는 한계
5. 검증 비용이 hot user latency에 미치는 영향과 별도 ledger가 없는 설계의 대가

### 원본 확인 위치

- Thread 12
- 커밋: `feat(snapshot): validate active aggregate consistency`
- 파일: `src/main/resources/scripts/risk-snapshot.lua`, `src/main/resources/scripts/risk-reserve.lua`, `src/main/resources/scripts/risk-commit.lua`
- 테스트: `RiskSnapshotAggregateConsistencyScriptTest`, `ReservationCorruptFootprintScriptTest`, `ReservationAggregateConsistencyScriptTest`
- 관련 Thread: 08, 10, 11, 14

---

## P13. [Thread 06 / `feat(history): configure bounded retention`] 시간 창과 표본 상한으로 제한하는 패턴 이력

### 면접 질문

빠른 베팅, stake 중앙값, 반복 선택에 필요한 이력을 Redis에 무기한 저장하지 않고 어떻게 bounded state로 만들었나요?

꼬리 질문:

- 시간 기반 정리와 최대 표본 수 기반 정리가 각각 필요한 이유는 무엇입니까?
- 동일 accepted event가 재전달될 때 이력이 중복되지 않게 하려면 어떤 member와 명령을 사용합니까?
- idle retention이 패턴 window보다 짧으면 어떤 문제가 생깁니까?
- active reservation 사실과 confirmed history를 왜 구분합니까?

### 30초 모범 답변

빠른 베팅과 반복 선택은 각 규칙 window 밖의 score를 제거하고, stake 이력은 중앙값 계산 비용을 제한하기 위해 최근 최대 표본 수도 둡니다. member에 bet ID를 포함하고 `ZADD NX`를 사용해 같은 사실의 재투영을 중복 반영하지 않습니다. 비어 있지 않은 키에는 가장 긴 규칙 window를 덮는 idle TTL을 갱신하고 빈 키는 삭제합니다. confirmed history는 commit이나 first-seen accepted projection에서만 기록하고, 승인 전 동시 후보는 별도 active footprint로 다룹니다.

### 답변 핵심 키워드

bounded state · score window · sample cap · `ZADD NX` · idle TTL · confirmed vs active · hot-user memory

### 백지 구현

#### 구현 목표

확정 베팅 하나를 세 종류의 Redis history에 기록하는 단순 Lua 스크립트를 작성한다. 시간 창과 최대 stake 표본 수를 지키고 재전달에 멱등해야 한다.

#### 인터페이스 또는 함수 시그니처

```lua
-- KEYS[1]: confirmed bet history sorted set
-- KEYS[2]: confirmed stake history sorted set
-- KEYS[3..n]: selection별 confirmed history sorted set
--
-- ARGV에는 now, rapidWindow, repeatedWindow, idleRetention,
-- maxStakeSamples, betId, stakeAmount가 주어진다.

-- 직접 구현
```

#### 입력과 출력

- 입력: 확정 시각, window/retention, 표본 상한, bet identity와 stake
- 출력: bet/stake가 신규 추가되었는지 나타내는 최소 결과
- 잘못된 type·인자·member: Redis error

#### 반드시 만족해야 할 조건

- bet history는 rapid window 밖 항목을 제거한다.
- selection history는 repeated window 밖 항목을 제거한다.
- stake history는 동일 bet의 재전달을 중복 추가하지 않는다.
- stake history가 최대 표본 수를 넘으면 가장 오래된 표본부터 제거한다.
- 모든 입력 키의 Redis type을 mutation 전에 검증한다.
- 비어 있는 키는 삭제하고, 비어 있지 않은 키에는 idle TTL을 적용한다.
- idle retention은 필요한 규칙 window보다 짧을 수 없다.
- 같은 bet의 재전달로 cardinality와 표본 수가 증가하지 않는다.
- 정확 정수 범위를 벗어난 stake를 거부한다.

#### 경계 조건

- 최초 기록
- 같은 bet 재기록
- score가 같은 서로 다른 bet
- 표본 수가 cap과 정확히 같거나 하나 초과
- 모든 오래된 항목이 제거되는 경우
- 선택이 여러 개인 bet
- idle retention과 repeated window가 정확히 같은 경우

#### 실패 조건

- wrong Redis type
- 빈 bet ID
- stake member 파싱 실패 또는 비양수
- window/retention/cap 비정상
- retention이 필요한 window보다 짧음

#### 필요한 제약

- 하나의 Lua 실행으로 끝낸다.
- 전체 history를 애플리케이션으로 가져와 자르지 않는다.
- selection 수는 이미 상위 경계에서 제한되었다고 가정한다.
- 패턴 판정 자체는 구현하지 않는다.

### 구현 후 자가 검증

- [ ] 최초 기록 후 bet·stake·각 selection history에 한 항목이 있다.
- [ ] 같은 bet 재기록 후 cardinality가 증가하지 않는다.
- [ ] 오래된 bet/selection 항목이 각 window에 맞게 제거된다.
- [ ] stake 표본이 cap을 넘으면 오래된 것부터 잘린다.
- [ ] 같은 timestamp의 서로 다른 bet가 모두 남는다.
- [ ] 비어 있는 키가 orphan TTL 없이 삭제된다.
- [ ] 비어 있지 않은 모든 키의 TTL이 양수다.
- [ ] 잘못된 key type에서는 정상 결과를 반환하지 않는다.
- [ ] hot user의 stake history 공간이 `O(maxStakeSamples)`로 제한된다.

### 구현 후 설명할 것

1. rapid/repeated는 시간 기반, stake는 개수 기반 상한도 둔 이유
2. event time 대신 서비스 관찰/확정 시각을 score로 쓸 때의 의미
3. `ZADD NX`가 제공하는 멱등성과 fingerprint marker와의 역할 차이
4. idle TTL과 규칙 window의 관계
5. active 후보를 confirmed history에 미리 넣지 않는 이유

### 원본 확인 위치

- Thread 06
- 커밋: `feat(history): configure bounded retention`
- 파일: `src/main/resources/scripts/history-record.lua`
- 클래스: `RiskHistoryProperties`, `HistoryKeys`
- 테스트: `HistoryProjectionScriptTest`
- 관련 Thread: 07, 10, 11, 14
