# 매칭·결과 확정·토너먼트 오케스트레이션 면접 워크북

이 문서는 여러 사용자가 동시에 상태를 선점하는 실시간 오케스트레이션에서 예약, 보상 처리, 재시도, 멱등성, 진행 상태를 연결하는 문제를 다룬다.

---

## IM-19. [Thread 11 / `refactor(game): rating 기반 closest-pair queue 구현`, `refactor(game): AI fallback과 reservation lifecycle 구현`] closest-pair 매칭과 예약 수명주기

### 면접 질문

대기열의 첫 사용자와 아무나 매칭하지 않고 rating 차이가 가장 작은 pair를 찾되, 방 생성 전 두 사용자를 예약 상태로 옮긴 이유는 무엇인가요? AI fallback 타이머와 예약 실패 롤백까지 포함해 설명해 주세요.

꼬리 질문:

- 가장 가까운 pair를 찾는 단순 구현의 시간 복잡도는 얼마이고, 어느 규모부터 개선이 필요합니까?
  - 모범답변: 전체 대기 pair를 매번 비교하면 O(n²)이고, 원본처럼 새 entrant의 상대만 찾으면 join당 O(n)입니다. 현재처럼 큐가 작고 rating 차이 200 제한이면 단순성이 유리하며, queue 길이·matching latency가 SLO를 위협할 때 정렬 구조나 rating bucket을 고려합니다.
- rating 차이가 같은 pair가 여러 개면 결정적인 tie-breaker를 왜 정해야 합니까?
  - 모범답변: iteration·삽입 순서에 우연히 의존하면 같은 입력에서도 상대가 달라져 테스트와 장애 재현이 어렵고 오래 기다린 사용자의 공정성도 설명하기 어렵습니다. joinedAt, 그 다음 user ID 순으로 고정하면 결과가 안정적입니다.
- 큐에서 제거한 뒤 방 생성이 실패하면 사용자는 어떤 상태로 돌아가야 합니까?
  - 모범답변: 두 사용자의 원래 player 정보와 joinedAt을 보존한 reservation을 해제하고 각각 한 번만 queued로 복원해야 합니다. 원본 GameHub도 방 생성 실패 시 room과 client roomId를 정리하고 matchmaking reservation을 release할 lifecycle을 둡니다.
- AI fallback 타이머가 실행되는 순간 사람 상대가 매칭되는 경쟁 상태를 어떻게 해결합니까?
  - 모범답변: timer callback도 `claimAiFallback`에서 현재 status가 queued인지 확인하고 동기적으로 matched로 선점합니다. 사람 match가 먼저 status를 바꿨으면 fallback은 unavailable이고, fallback이 먼저 선점했으면 사람 enqueue가 duplicate/matched를 봐 두 방이 생기지 않습니다.

### 30초 모범 답변

매칭 선택과 방 생성 사이에는 비동기 작업이 있으므로 사용자를 바로 소비하면 실패 시 유실되고, 큐에 그대로 두면 중복 매칭될 수 있습니다. 그래서 `queued → reserved → matched` 같은 상태를 두고 pair를 원자적으로 예약한 뒤 방을 만듭니다. 성공하면 예약을 확정하고, 실패하면 원래 대기 순서와 timeout 정책을 보존해 복구합니다. AI fallback도 동일한 예약 경계를 통과하게 해 사람 매칭과 동시에 두 경기가 만들어지지 않도록 합니다. 작은 큐에서는 전체 pair 탐색이 단순하고 충분하지만, tie-breaker는 항상 고정합니다.

### 답변 핵심 키워드

closest pair · reservation · queued/reserved/matched · rollback · AI fallback race · deterministic tie-break · timer ownership · 중복 매칭 방지

### 백지 구현

**구현 목표**

작은 인메모리 매칭 대기열에서 rating 차이가 가장 작은 두 사용자를 예약하고, 성공·실패에 따라 확정 또는 복원할 수 있는 API를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
import { randomUUID } from "node:crypto";

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
  private readonly queued = new Map<string, Player>();
  private readonly reservations = new Map<string, Reservation>();
  private readonly reservationByPlayer = new Map<string, string>();

  constructor(private readonly createId: () => string = randomUUID) {}

  enqueue(player: Player): void {
    if (!player.id.trim() || !Number.isSafeInteger(player.rating) || !Number.isFinite(player.joinedAtMs)) {
      throw new Error("invalid player");
    }
    if (this.queued.has(player.id) || this.reservationByPlayer.has(player.id)) return;
    this.queued.set(player.id, { ...player });
  }

  reserveClosestPair(): Reservation | null {
    const players = this.listQueued();
    if (players.length < 2) return null;
    let best: readonly [Player, Player] | null = null;
    let bestKey: readonly [number, number, string, string] | null = null;
    for (let left = 0; left < players.length - 1; left += 1) {
      for (let right = left + 1; right < players.length; right += 1) {
        const pair = [players[left], players[right]] as const;
        const key = [
          Math.abs(pair[0].rating - pair[1].rating),
          Math.min(pair[0].joinedAtMs, pair[1].joinedAtMs),
          pair[0].id < pair[1].id ? pair[0].id : pair[1].id,
          pair[0].id < pair[1].id ? pair[1].id : pair[0].id
        ] as const;
        if (!bestKey || key.some((value, index) => value < bestKey![index]
          && key.slice(0, index).every((prefix, prefixIndex) => prefix === bestKey![prefixIndex]))) {
          best = pair;
          bestKey = key;
        }
      }
    }
    if (!best) return null;
    const reservation: Reservation = { id: this.createId(), players: best };
    if (this.reservations.has(reservation.id)) throw new Error("reservation id collision");
    for (const player of best) {
      this.queued.delete(player.id);
      this.reservationByPlayer.set(player.id, reservation.id);
    }
    this.reservations.set(reservation.id, reservation);
    return { id: reservation.id, players: [{ ...best[0] }, { ...best[1] }] };
  }

  commit(reservationId: string): void {
    const reservation = this.reservations.get(reservationId);
    if (!reservation) return;
    this.reservations.delete(reservationId);
    for (const player of reservation.players) this.reservationByPlayer.delete(player.id);
  }

  rollback(reservationId: string): void {
    const reservation = this.reservations.get(reservationId);
    if (!reservation) return;
    this.reservations.delete(reservationId);
    for (const player of reservation.players) {
      this.reservationByPlayer.delete(player.id);
      if (!this.queued.has(player.id)) this.queued.set(player.id, { ...player });
    }
  }

  remove(playerId: string): boolean {
    if (this.queued.delete(playerId)) return true;
    const reservationId = this.reservationByPlayer.get(playerId);
    if (!reservationId) return false;
    const reservation = this.reservations.get(reservationId);
    if (!reservation) {
      this.reservationByPlayer.delete(playerId);
      return false;
    }
    this.reservations.delete(reservationId);
    for (const player of reservation.players) {
      this.reservationByPlayer.delete(player.id);
      // 연결이 남은 상대만 원래 joinedAt으로 대기열에 복원한다.
      if (player.id !== playerId) this.queued.set(player.id, { ...player });
    }
    return true;
  }

  listQueued(): Player[] {
    return [...this.queued.values()]
      .sort((left, right) => left.joinedAtMs - right.joinedAtMs || left.id.localeCompare(right.id))
      .map((player) => ({ ...player }));
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
   - 모범답변: 상대 선택과 room 생성 사이에 실패가 있으므로 queued를 바로 소모하면 사용자를 잃고 그대로 두면 다른 match가 중복 선점합니다. reservation은 외부 작업 동안 단일 소유권과 rollback 정보를 함께 보존합니다.
2. pair 품질과 대기 시간 공정성 사이의 trade-off
   - 모범답변: rating 차이만 최소화하면 rating이 희귀한 사용자는 오래 기다릴 수 있습니다. 원본은 허용 차이와 6초 AI fallback을 두며, 일반적으로 대기 시간에 따라 rating window를 넓히는 방식으로 품질과 공정성을 조절합니다.
3. AI fallback과 사람 매칭이 같은 예약 경계를 써야 하는 이유
   - 모범답변: 둘은 같은 사용자의 queued ownership을 소비하는 경쟁자입니다. 각각 별도 flag를 쓰면 timer와 사람 join이 동시에 성공할 수 있으므로 하나의 status transition만 matched를 선점해야 합니다.
4. 방 생성 실패 시 복원해야 하는 정보
   - 모범답변: player ID·rating·kind, 원래 queuedAt, queue entry의 fallback timer 정책, matchmaking status를 복원해야 합니다. 새 joinedAt을 주면 실패 때문에 대기 우선순위가 뒤로 밀립니다.
5. 정렬 구조나 rating bucket으로 확장할 시점
   - 모범답변: queue 길이와 join rate가 커져 O(n) 또는 O(n²) 탐색이 event-loop latency에 유의미하게 나타날 때입니다. balanced tree나 rating bucket은 근접 조회를 줄이지만 대기 시간 tie-break와 삭제·rollback 복잡도가 늘어납니다.

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
  - 모범답변: room의 `finishing`에 첫 `finalizeRoom` Promise를 저장하고 이후 `finishRoom` 호출은 그 Promise를 그대로 반환합니다. 실패한 Promise와 room identity가 여전히 같을 때만 표시를 비워 정책상 다시 시작할 수 있게 합니다.
- 저장 성공 전 `game.finished`를 방송하면 어떤 거짓 상태가 생깁니까?
  - 모범답변: client는 persisted match ID와 rating 변경이 확정됐다고 보지만 DB transaction이 실패해 전적에 없는 결과가 됩니다. 원본은 저장 성공 결과를 받은 뒤에만 `persisted: true` 이벤트를 보냅니다.
- 영구 실패와 일시 실패를 구분하지 않고 무한 재시도하면 어떤 운영 문제가 생깁니까?
  - 모범답변: validation 같은 영구 오류에도 room·client·retry timer가 계속 남고 DB 장애 중 재시도가 부하를 증폭합니다. 원본은 delay를 5초로 제한하지만 횟수는 무한이므로, 운영 확장에서는 오류 분류·최대 시도·dead-letter 또는 관리 개입 정책이 필요합니다.
- shutdown drain은 재시도 타이머와 in-flight 저장을 어떻게 기다려야 합니까?
  - 모범답변: finalization Promise와 retry timer를 room lifecycle에 포함하고 성공 또는 명시적 포기 전에는 room을 active map에서 제거하지 않습니다. drain은 `rooms.size`가 0이 되거나 외부 timeout에 도달할 때까지 기다립니다.

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
  private readonly operation: (key: string) => Promise<T>;
  private readonly policy: RetryPolicy;
  private readonly sleep: (ms: number) => Promise<void>;
  private readonly onAttempt: (input: { key: string; attempt: number; error?: unknown }) => void;
  private readonly inFlight = new Map<string, Promise<FinalizationOutcome<T>>>();

  constructor(
    operation: (key: string) => Promise<T>,
    policy: RetryPolicy,
    dependencies?: {
      sleep?: (ms: number) => Promise<void>;
      onAttempt?: (input: { key: string; attempt: number; error?: unknown }) => void;
    }
  ) {
    if (
      !Number.isFinite(policy.baseDelayMs)
      || policy.baseDelayMs < 0
      || !Number.isFinite(policy.maxDelayMs)
      || policy.maxDelayMs < policy.baseDelayMs
      || !Number.isInteger(policy.maxAttempts)
      || policy.maxAttempts <= 0
    ) {
      throw new RangeError("invalid retry policy");
    }
    this.operation = operation;
    this.policy = { ...policy };
    this.sleep = dependencies?.sleep ?? ((ms) => new Promise((resolve) => setTimeout(resolve, ms)));
    this.onAttempt = dependencies?.onAttempt ?? (() => undefined);
  }

  finalize(key: string): Promise<FinalizationOutcome<T>> {
    const existing = this.inFlight.get(key);
    if (existing) return existing;
    const task = this.run(key).finally(() => {
      if (this.inFlight.get(key) === task) this.inFlight.delete(key);
    });
    this.inFlight.set(key, task);
    return task;
  }

  inFlightCount(): number {
    return this.inFlight.size;
  }

  private async run(key: string): Promise<FinalizationOutcome<T>> {
    for (let attempt = 1; attempt <= this.policy.maxAttempts; attempt += 1) {
      try {
        const value = await this.operation(key);
        this.observe({ key, attempt });
        return { value, attempts: attempt };
      } catch (error) {
        this.observe({ key, attempt, error });
        if (attempt === this.policy.maxAttempts) throw error;
        const delayMs = Math.min(
          this.policy.maxDelayMs,
          this.policy.baseDelayMs * 2 ** Math.min(attempt - 1, 30)
        );
        await this.sleep(delayMs);
      }
    }
    throw new Error("unreachable finalization state");
  }

  private observe(input: { key: string; attempt: number; error?: unknown }): void {
    try {
      this.onAttempt(input);
    } catch {
      // 관측 hook은 확정 결과와 retry 제어 흐름을 바꾸지 않는다.
    }
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
   - 모범답변: single-flight는 현재 프로세스의 같은 room 호출을 Promise 하나로 합칠 뿐 재시작·다중 인스턴스를 넘지 못합니다. DB `resultKey` unique constraint는 실제 경기·rating 효과가 재호출돼도 한 번만 commit되게 합니다.
2. 완료 이벤트를 commit 뒤에 발행해야 하는 이유
   - 모범답변: 저장 전에 `game.finished`를 보내면 client는 영속 match ID와 rating 반영이 있다고 믿지만 DB는 실패할 수 있습니다. 원본은 `finalizeMatch` 성공 결과를 받은 뒤에만 persisted result를 broadcast하고 finally에서 room을 제거합니다.
3. backoff 상한과 최대 시도 횟수의 운영 trade-off
   - 모범답변: 짧고 많은 재시도는 DB 장애를 증폭하고, 긴 backoff는 room과 client 자원을 오래 붙잡습니다. 원본 지연은 250ms부터 5초 상한이며, 워크북 조정기는 최대 횟수까지 둬 영구 실패를 호출자 정책으로 넘깁니다.
4. 재시도 가능 오류와 영구 오류를 분류하는 확장 방법
   - 모범답변: timeout·connection reset·serialization failure는 backoff 재시도하고 validation·foreign key·권한 오류는 즉시 실패시키는 predicate를 주입할 수 있습니다. 오류 code 기준 분류와 unknown 기본 정책을 명시해야 합니다.
5. drain 완료 조건에 in-flight·sleeping retry를 포함하는 방법
   - 모범답변: 원본은 finalization loop와 retry timer가 room 객체에 속하고 성공 후에만 room을 제거합니다. 따라서 active room 0은 operation 실행과 sleeping retry 모두 끝났다는 뜻이며 drain waiter도 그때 resolve됩니다.

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
  - 모범답변: 현재 GameHub는 실제로 메모리 room을 먼저 만들고 `startTournamentMatch`를 호출하며, DB 실패를 catch하면 `abandonRoom`으로 보상합니다. 다만 create와 DB 호출 사이에 프로세스가 중단되면 DB에는 없는 room이 잠시 존재할 수 있어, 더 강한 복구가 필요하면 영속 reservation을 먼저 두는 편이 낫습니다.
- 보상 동작 자체도 실패하면 어떤 상태와 운영 도구가 필요합니까?
  - 모범답변: matchId와 resourceId, reservation 단계, 원래 오류와 cleanup 오류를 모두 남기고 `stuck`/`reconcile_required` 같은 상태를 운영자가 조회할 수 있어야 합니다. idempotent release/destroy를 재시도하는 reconciler나 관리 명령이 필요합니다.
- 같은 match ID에 두 사용자가 동시에 참가할 때 방이 하나만 만들어지게 어떻게 합니까?
  - 모범답변: 단일 프로세스의 원본은 match별 waiter 배열을 지운 뒤 한 room만 만들지만 다중 인스턴스에는 충분하지 않습니다. DB에서 `ready → running` 조건부 전이를 한 resourceId만 성공시키고 unique/lock으로 match 단일 소유권을 정해야 합니다.
- 준결승 완료와 결승 생성·진행을 하나의 거대한 트랜잭션으로 묶는 것이 항상 좋은가요?
  - 모범답변: 준결승 결과와 결승 row 생성은 같은 영속 invariant라 짧은 트랜잭션에 묶는 것이 좋지만, 두 준결승 경기 실행과 WebSocket room 생명주기 전체를 묶으면 장기 lock이 됩니다. 각 경기 완료를 작은 멱등 트랜잭션으로 확정하고 다음 단계는 상태 전이로 연결합니다.

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
  const reservation = await store.reserve(input.matchId, input.resourceId);
  if (reservation !== "reserved") {
    // 이 계약만으로는 기존 예약이 같은 resource인지 판별할 수 없으므로 생성하지 않고 충돌로 둔다.
    throw new Error("match resource already reserved");
  }

  let resource: Resource;
  try {
    resource = await factory.create(input.resourceId);
  } catch (createError) {
    try {
      await store.release(input.matchId, input.resourceId);
    } catch (releaseError) {
      throw new AggregateError([createError, releaseError], "resource creation and reservation release failed");
    }
    throw createError;
  }

  try {
    await store.confirm(input.matchId, input.resourceId);
    return resource;
  } catch (confirmError) {
    const errors: unknown[] = [confirmError];
    try {
      await factory.destroy(resource);
    } catch (destroyError) {
      errors.push(destroyError);
    }
    try {
      await store.release(input.matchId, input.resourceId);
    } catch (releaseError) {
      errors.push(releaseError);
    }
    if (errors.length > 1) {
      throw new AggregateError(errors, "confirmation and compensation failed");
    }
    throw confirmError;
  }
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
   - 모범답변: DB transaction manager는 SQL row만 commit/rollback하며 JavaScript Map, scheduler timer, WebSocket send를 되돌릴 수 없습니다. 두 자원 사이에는 중단 지점이 생기므로 식별 가능한 상태 전이와 idempotent 보상이 필요합니다.
2. 예약 후 생성 순서를 선택한 이유
   - 모범답변: 일반 원칙상 영속 reservation을 먼저 선점하면 같은 match의 다른 worker가 resource를 만들기 전에 충돌을 봅니다. 다만 현재 프로젝트는 room을 먼저 만들고 DB start 실패 시 room을 버리는 반대 순서를 쓰므로, 보장은 단일 GameHub 프로세스의 waiter ownership에 더 의존합니다.
3. 보상 처리가 rollback과 다른 점
   - 모범답변: rollback은 commit 전 변경이 외부에 보이지 않게 원자적으로 취소되지만, 보상은 이미 끝난 별도 작업을 새 명령으로 되돌리므로 실패·중복·관측 가능성이 있습니다. 그래서 release와 destroy는 멱등해야 합니다.
4. 중단 지점마다 남는 불완전 상태와 복구 전략
   - 모범답변: reserve 뒤 중단이면 stale reservation, create 뒤 중단이면 reservation과 미확정 resource, confirm 뒤 응답 전 중단이면 정상 확정인데 호출자는 결과를 모르는 상태입니다. match/resource ID와 단계·만료 시각을 조회해 reconciler가 release, destroy 또는 기존 확정을 반환해야 합니다.
5. 더 복잡해질 때 outbox·작업 큐·reconciler를 도입할 기준
   - 모범답변: 다중 인스턴스에서 중단 상태가 실제로 누적되거나 보상을 request thread에서 끝내기 어렵고 감사 가능한 재시도가 필요할 때입니다. DB 상태와 outbox를 함께 commit하고 worker가 resource 작업을 멱등 실행하면 복구 지점을 영속화할 수 있습니다.

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
