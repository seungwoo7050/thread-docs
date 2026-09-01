# 매칭·결과 확정·토너먼트 오케스트레이션 면접 워크북

이 문서는 여러 사용자가 동시에 상태를 선점하는 실시간 오케스트레이션에서 예약, 보상 처리, 재시도, 멱등성, 진행 상태를 연결하는 문제를 다룬다.

---

## IM-19. [Thread 11 / `refactor(game): rating 기반 closest-pair queue 구현`, `refactor(game): AI fallback과 reservation lifecycle 구현`] closest-pair 매칭과 예약 수명주기

### 면접 질문

대기열의 첫 사용자와 아무나 매칭하지 않고 rating 차이가 가장 작은 pair를 찾되, 방 생성 전 두 사용자를 예약 상태로 옮긴 이유는 무엇인가요? AI fallback 타이머와 예약 실패 롤백까지 포함해 설명해 주세요.

꼬리 질문:

- 가장 가까운 pair를 찾는 단순 구현의 시간 복잡도는 얼마이고, 어느 규모부터 개선이 필요합니까?
- rating 차이가 같은 pair가 여러 개면 결정적인 tie-breaker를 왜 정해야 합니까?
- 큐에서 제거한 뒤 방 생성이 실패하면 사용자는 어떤 상태로 돌아가야 합니까?
- AI fallback 타이머가 실행되는 순간 사람 상대가 매칭되는 경쟁 상태를 어떻게 해결합니까?

### 30초 모범 답변

매칭 선택과 방 생성 사이에는 비동기 작업이 있으므로 사용자를 바로 소비하면 실패 시 유실되고, 큐에 그대로 두면 중복 매칭될 수 있습니다. 그래서 `queued → reserved → matched` 같은 상태를 두고 pair를 원자적으로 예약한 뒤 방을 만듭니다. 성공하면 예약을 확정하고, 실패하면 원래 대기 순서와 timeout 정책을 보존해 복구합니다. AI fallback도 동일한 예약 경계를 통과하게 해 사람 매칭과 동시에 두 경기가 만들어지지 않도록 합니다. 작은 큐에서는 전체 pair 탐색이 단순하고 충분하지만, tie-breaker는 항상 고정합니다.

### 답변 핵심 키워드

closest pair · reservation · queued/reserved/matched · rollback · AI fallback race · deterministic tie-break · timer ownership · 중복 매칭 방지

### 백지 구현

**구현 목표**

작은 인메모리 매칭 대기열에서 rating 차이가 가장 작은 두 사용자를 예약하고, 성공·실패에 따라 확정 또는 복원할 수 있는 API를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface Player {
  id: string;
  rating: number;
  joinedAtMs: number;
}

interface Reservation {
  id: string;
  players: readonly [Player, Player];
}

export class MatchQueue {
  enqueue(player: Player): void {
    // 직접 구현
  }

  reserveClosestPair(): Reservation | null {
    // 직접 구현
    throw new Error("not implemented");
  }

  commit(reservationId: string): void {
    // 직접 구현
  }

  rollback(reservationId: string): void {
    // 직접 구현
  }

  remove(playerId: string): boolean {
    // 직접 구현
    throw new Error("not implemented");
  }

  listQueued(): Player[] {
    // 직접 구현
    throw new Error("not implemented");
  }
}
```

**입력과 출력**

- 입력: 플레이어, 예약 ID, 연결 종료 제거 요청
- 출력: 가장 가까운 pair 예약 또는 없음

**반드시 만족해야 할 조건**

- 동일 플레이어가 큐나 예약에 중복 존재하지 않는다.
- pair 선택은 rating 차이를 최소화한다.
- 동률이면 가입 시각과 ID 등 미리 정한 안정적 규칙으로 결정한다.
- 예약된 사용자는 다른 예약에 포함되지 않는다.
- commit 뒤 사용자는 큐에서 완전히 제거된다.
- rollback 뒤 사용자는 다시 한 번만 대기 상태가 된다.
- 존재하지 않거나 이미 완료된 예약에 대한 처리는 멱등하거나 명확히 거부한다.

**경계 조건**

- 사용자 0명, 1명, 2명
- 같은 rating과 같은 가입 시각
- 예약 중 한 사용자의 연결 종료
- rollback 직후 새 사용자 enqueue
- 같은 사용자의 중복 enqueue

**실패 조건**

- 유효하지 않은 rating
- 예약 ID 불일치
- 방 생성 성공 후 commit 실패를 어떤 계층에서 막을지 설명

**필요한 제약**

- 작은 큐를 가정해 명확한 구현을 우선한다.
- 상태 전이는 한 메서드 호출 안에서 동기적으로 완료한다.
- AI fallback은 별도 타이머가 이 예약 API를 호출한다고 가정한다.

### 구현 후 자가 검증

- [ ] 모든 후보 중 rating 차이가 최소인 pair가 선택된다.
- [ ] tie-breaker가 반복 실행에서 같은 결과를 낸다.
- [ ] 예약된 사용자가 중복 예약되지 않는다.
- [ ] rollback 뒤 큐 순서와 참가자 수가 복구된다.
- [ ] commit·rollback 중복 호출이 invariant를 깨지 않는다.
- [ ] 연결 종료 시 queued·reserved 상태를 모두 처리할 정책이 있다.
- [ ] 현재 구현의 시간 복잡도와 더 큰 규모의 대안을 설명할 수 있다.

### 구현 후 설명할 것

1. 큐에서 즉시 제거하는 대신 reservation을 둔 이유
2. pair 품질과 대기 시간 공정성 사이의 trade-off
3. AI fallback과 사람 매칭이 같은 예약 경계를 써야 하는 이유
4. 방 생성 실패 시 복원해야 하는 정보
5. 정렬 구조나 rating bucket으로 확장할 시점

### 원본 확인 위치

- Thread 11
- 커밋: `refactor(game): matchmaking player와 fallback 계약 정의`
- 커밋: `refactor(game): rating 기반 closest-pair queue 구현`
- 커밋: `refactor(game): AI fallback과 reservation lifecycle 구현`
- 커밋: `refactor(game): Matchmaker queue reservation을 GameHub에 연결`
- `apps/api/src/game/matchmaker.ts`
  - `Matchmaker`
  - `MatchmakingPlayer`
  - `MatchmakerJoinResult`
  - `AiFallbackResult`
  - `MatchmakerPlayerStatus`
  - `MATCHMAKER_AI_FALLBACK_MS`
  - `findClosestOpponent`
- `apps/api/src/game/matchmaker.test.ts`
- `apps/api/src/gameHub.ts`
- 관련 Thread: 10, 12, 16

---

## IM-20. [Thread 15 / `refactor(game): 경기 결과 확정 boundary 사용`] 단일 in-flight 확정과 재시도 가능한 영속화

### 면접 질문

경기가 끝났을 때 `finalizeMatch`가 실패하면 방을 바로 삭제하거나 종료 이벤트를 보내지 않고, room별 in-flight Promise와 지수형 지연 재시도를 유지한 이유는 무엇인가요? 정확히 한 번의 효과와 최소 한 번의 호출을 어떻게 함께 다뤘습니까?

꼬리 질문:

- 같은 방에서 종료 판정이 여러 tick에 걸쳐 반복되면 저장 호출 중복을 어떻게 막습니까?
- 저장 성공 전 `game.finished`를 방송하면 어떤 거짓 상태가 생깁니까?
- 영구 실패와 일시 실패를 구분하지 않고 무한 재시도하면 어떤 운영 문제가 생깁니까?
- shutdown drain은 재시도 타이머와 in-flight 저장을 어떻게 기다려야 합니까?

### 30초 모범 답변

프로세스 내부에서는 한 방의 확정 작업을 하나의 Promise로 대표해 중복 실행을 막고, 저장소에서는 `resultKey` 유일 제약으로 재시도의 부수 효과를 한 번으로 제한합니다. 저장이 실패하면 in-flight 표시를 해제하고 제한된 backoff로 다시 시도하지만, 성공 전에는 완료 이벤트를 보내거나 방을 제거하지 않습니다. 즉 호출 자체는 최소 한 번 이상 일어날 수 있지만 상태 변화는 멱등합니다. 운영에서는 재시도 횟수·지연 상한·관측·종료 정책을 함께 둬야 합니다.

### 답변 핵심 키워드

single-flight · retry · exponential backoff · idempotent sink · publish-after-commit · room lifecycle · bounded delay · drain coordination

### 백지 구현

**구현 목표**

키별로 동시에 하나의 비동기 작업만 실행하고, 실패 시 제한된 backoff로 재시도하는 조정기를 구현한다. 실제 데이터베이스 로직은 의존성으로 주입한다.

**인터페이스 또는 함수 시그니처**

```ts
interface RetryPolicy {
  baseDelayMs: number;
  maxDelayMs: number;
  maxAttempts: number;
}

interface FinalizationOutcome<T> {
  value: T;
  attempts: number;
}

export class KeyedFinalizer<T> {
  constructor(
    operation: (key: string) => Promise<T>,
    policy: RetryPolicy,
    dependencies?: {
      sleep?: (ms: number) => Promise<void>;
      onAttempt?: (input: { key: string; attempt: number; error?: unknown }) => void;
    }
  ) {}

  finalize(key: string): Promise<FinalizationOutcome<T>> {
    // 직접 구현
    throw new Error("not implemented");
  }

  inFlightCount(): number {
    // 직접 구현
    throw new Error("not implemented");
  }
}
```

**입력과 출력**

- 입력: 작업 키
- 출력: 성공 결과와 시도 횟수, 또는 최종 실패

**반드시 만족해야 할 조건**

- 같은 키의 동시 호출은 같은 in-flight 결과를 공유한다.
- 서로 다른 키는 독립적으로 진행된다.
- 실패 시 delay는 상한을 넘지 않는다.
- 성공하거나 최종 실패하면 in-flight map에서 제거된다.
- 관측 콜백의 실패가 본 작업을 실패시키지 않도록 할지 정책을 명시한다.
- 호출자가 작업 성공 뒤에만 완료 이벤트를 발행할 수 있는 반환 경계를 제공한다.

**경계 조건**

- 첫 시도 성공
- 마지막 허용 시도에서 성공
- 모든 시도 실패
- retry sleep 중 같은 키 재호출
- operation이 동기적으로 throw하는 Promise 생성 함수를 쓴 경우

**실패 조건**

- 잘못된 retry 설정
- sleep 의존성 실패
- operation이 취소를 지원하지 않음

**필요한 제약**

- 정답 저장소가 멱등하지 않다면 이 클래스만으로 exactly-once를 보장한다고 주장하지 않는다.
- 테스트에서는 실제 대기 없이 sleep을 주입한다.
- 무한 재시도는 금지한다.

### 구현 후 자가 검증

- [ ] 같은 키 동시 호출이 operation을 한 번만 시작한다.
- [ ] 실패 뒤 정해진 횟수와 지연으로 재시도한다.
- [ ] 지연이 최대값을 넘지 않는다.
- [ ] 완료 뒤 in-flight 엔트리가 제거된다.
- [ ] 한 키의 실패가 다른 키를 막지 않는다.
- [ ] 완료 이벤트가 저장 성공 이후에만 나가도록 호출 순서를 설명할 수 있다.
- [ ] 프로세스 재시작을 넘는 보장은 저장소의 멱등성 키에 의존함을 설명할 수 있다.

### 구현 후 설명할 것

1. single-flight와 데이터베이스 멱등성이 서로 다른 계층인 이유
2. 완료 이벤트를 commit 뒤에 발행해야 하는 이유
3. backoff 상한과 최대 시도 횟수의 운영 trade-off
4. 재시도 가능 오류와 영구 오류를 분류하는 확장 방법
5. drain 완료 조건에 in-flight·sleeping retry를 포함하는 방법

### 원본 확인 위치

- Thread 15
- 커밋: `refactor(game): 경기 결과 확정 boundary 사용`
- `apps/api/src/gameHub.ts`
  - `finishRoom`
  - `finalizeRoom`
  - room의 `finishing` 상태
- `packages/db/src/index.ts`
  - `finalizeMatch`
- 관련 Thread: 13, 16, 21, 22

---

## IM-21. [Thread 16 / `feat(tournament): 대진 경기 schema 추가`] 토너먼트 경기 예약과 방 생성 실패 보상

### 면접 질문

토너먼트 경기 참가에서 DB의 경기를 `running`으로 예약한 뒤 실시간 방 생성이 실패하면 왜 예약을 되돌려야 하나요? 데이터베이스 트랜잭션으로 묶을 수 없는 메모리 방 생성과 영속 상태를 어떻게 조정하겠습니까?

꼬리 질문:

- 방을 먼저 만든 뒤 DB를 갱신하는 순서에는 어떤 유령 방 문제가 있습니까?
- 보상 동작 자체도 실패하면 어떤 상태와 운영 도구가 필요합니까?
- 같은 match ID에 두 사용자가 동시에 참가할 때 방이 하나만 만들어지게 어떻게 합니까?
- 준결승 완료와 결승 생성·진행을 하나의 거대한 트랜잭션으로 묶는 것이 항상 좋은가요?

### 30초 모범 답변

DB 트랜잭션은 프로세스 메모리의 방 생성까지 원자적으로 포함할 수 없으므로 단계별 상태 전이와 보상 처리가 필요합니다. 먼저 저장소에서 해당 토너먼트 경기가 시작 가능한지 확인하고 room ID를 예약한 뒤 방을 만듭니다. 방 생성이 실패하면 같은 예약 식별자를 기준으로 DB 상태를 이전 값으로 되돌립니다. 중복 참가에는 match 단위 단일 소유권이나 조건부 상태 전이를 사용하고, 보상 실패는 숨기지 말고 재조정 가능한 상태와 로그로 남겨야 합니다.

### 답변 핵심 키워드

cross-resource atomicity · reservation · compensating action · conditional transition · ghost room · stuck reservation · reconciliation · saga 성격

### 백지 구현

**구현 목표**

영속 저장소의 작업 예약과 외부 리소스 생성을 연결하고, 생성 실패 시 예약을 보상하는 작은 오케스트레이터를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface ReservationStore {
  reserve(matchId: string, resourceId: string): Promise<"reserved" | "already_reserved">;
  confirm(matchId: string, resourceId: string): Promise<void>;
  release(matchId: string, resourceId: string): Promise<void>;
}

interface ResourceFactory<Resource> {
  create(resourceId: string): Promise<Resource>;
  destroy(resource: Resource): Promise<void>;
}

export async function provisionMatchResource<Resource>(
  input: { matchId: string; resourceId: string },
  store: ReservationStore,
  factory: ResourceFactory<Resource>
): Promise<Resource> {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 영속 match ID와 새 resource ID
- 출력: 생성된 리소스

**반드시 만족해야 할 조건**

- 저장소 예약이 성공하기 전에는 리소스를 만들지 않는다.
- 동일 match가 이미 다른 resource에 예약되어 있으면 중복 생성하지 않는다.
- 리소스 생성 실패 시 해당 예약을 해제한다.
- confirm 실패 시 생성한 리소스를 정리하고, 예약 상태를 어떻게 다룰지 명시한다.
- release 또는 destroy 실패가 본래 오류를 덮지 않도록 한다.
- 모든 보상은 동일한 match/resource 식별자를 사용한다.

**경계 조건**

- 이미 같은 resource로 예약된 재시도
- reserve 성공 직후 프로세스 종료
- create 성공 후 confirm 실패
- cleanup 두 단계가 모두 실패
- 두 호출이 같은 match에 다른 resource ID를 제안

**실패 조건**

- 예약 충돌
- 리소스 생성 실패
- confirm 실패
- 보상 실패

**필요한 제약**

- 완벽한 분산 트랜잭션을 가정하지 않는다.
- 불완전 상태를 나중에 탐지할 수 있는 식별자와 상태를 남긴다.
- 오류의 원인과 보상 오류를 함께 보존한다.

### 구현 후 자가 검증

- [ ] reserve 실패 시 create가 호출되지 않는다.
- [ ] create 실패 시 release가 호출된다.
- [ ] confirm 실패 시 생성 리소스 cleanup이 시도된다.
- [ ] 같은 match의 동시 호출이 리소스 두 개를 확정하지 못한다.
- [ ] 본래 오류와 cleanup 오류가 모두 관측 가능하다.
- [ ] 프로세스 중단 지점별로 남을 수 있는 상태를 열거할 수 있다.
- [ ] 사후 reconciliation이 확인할 키와 상태를 설명할 수 있다.

### 구현 후 설명할 것

1. DB와 메모리 리소스를 하나의 트랜잭션으로 묶을 수 없는 이유
2. 예약 후 생성 순서를 선택한 이유
3. 보상 처리가 rollback과 다른 점
4. 중단 지점마다 남는 불완전 상태와 복구 전략
5. 더 복잡해질 때 outbox·작업 큐·reconciler를 도입할 기준

### 원본 확인 위치

- Thread 16
- 커밋: `feat(tournament): 대진 경기 schema 추가`
- `packages/db/src/index.ts`
  - `getTournamentMatch`
  - `startTournamentMatch`
  - `completeTournamentMatch`
  - `ensureFinalMatch`
- `apps/api/src/gameHub.ts`
  - `joinTournamentMatch`
  - `leaveTournamentWaiters`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 11, 15, 17
