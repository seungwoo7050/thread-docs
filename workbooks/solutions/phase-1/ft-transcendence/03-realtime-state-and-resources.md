# 실시간 상태·연결·리소스 수명주기 면접 워크북

이 문서는 WebSocket 기반 게임 서버에서 상태 머신, 타이머, 연결 교체, 재접속, 전송 혼잡, 서버 종료를 서로 충돌하지 않게 다루는 문제를 다룬다.

---

## IM-11. [Thread 12 / `feat(game): 게임 방 상태를 RoomSession에 연결`, `test(game): reconnect 복구 동작 검증`] 경기 방 상태 머신과 재접속 유예

### 면접 질문

경기 방 상태를 여러 boolean과 조건문으로 관리하지 않고 `RoomSession` 상태 머신으로 분리한 이유는 무엇인가요? 연결이 끊긴 뒤 15초 동안 이전 `playing` 또는 `paused` 상태를 보존하고, 만료 시 몰수패 또는 무승부로 끝내는 흐름을 설명해 주세요.

꼬리 질문:

- 양쪽이 거의 동시에 끊겼을 때 승자를 정하지 않아야 하는 이유와 판정 시점은 무엇입니까?
  - 모범답변: 한쪽 disconnect 직후 바로 몰수패를 확정하면 반대쪽도 같은 장애로 끊긴 상황에서 임의 승자가 생깁니다. `reconnecting` 동안 끊긴 side 집합을 모으고 deadline 만료 시 집합 크기가 정확히 1일 때만 상대를 승자로 정합니다.
- 재접속 직전에 만료 타이머가 실행되는 경쟁 상태를 어떻게 막겠습니까?
  - 모범답변: 타이머 callback이 현재 room identity와 `reconnectDeadline`을 다시 확인한 뒤 `expireReconnect(now)`를 호출해야 합니다. 재접속이 먼저 side를 제거하고 deadline을 비우면 만료는 null을 반환하며, 만료가 먼저 finished로 전이하면 reconnect는 false입니다.
- `pause()`나 `resume()`을 잘못된 상태에서 호출했을 때 예외와 멱등 반환 중 무엇을 선택하겠습니까?
  - 모범답변: 현재 `RoomSession`은 잘못된 상태에서 예외 대신 현재 상태를 반환합니다. 네트워크 중복 명령을 무해하게 만들고, 상위 GameHub가 좌석·room 권한 오류를 별도로 응답하게 하는 선택입니다.
- 상태 머신과 WebSocket·스케줄러 같은 부수 효과를 어떻게 분리해 테스트합니까?
  - 모범답변: `RoomSession`에는 side·명시적 now 값·상태 집합만 두고, timer 생성·socket close·결과 저장은 GameHub가 반환 상태와 expiry 결과를 보고 수행합니다. 상태 전이는 순수 단위 테스트, timer와 재접속 wiring은 fake timer 기반 허브 테스트로 나눕니다.

### 30초 모범 답변

방 상태를 명시적 상태 머신으로 만들면 허용된 전이와 금지된 전이를 한곳에서 검증할 수 있습니다. 연결이 끊기면 현재 상태를 보존하고 `reconnecting`으로 전환한 뒤 deadline을 기록합니다. 기한 안에 같은 사용자가 돌아오면 이전 상태를 복원하고 타이머를 해제합니다. 기한이 지나 한쪽만 없으면 몰수패, 양쪽이 없으면 승자 없이 종료하며, 이미 종료된 세션은 만료 처리가 다시 실행돼도 결과를 한 번만 냅니다.

### 답변 핵심 키워드

명시적 상태 머신 · 허용 전이 · 이전 상태 보존 · deadline · 단일 만료 결과 · 양쪽 disconnect · 타이머 취소 · 부수 효과 분리

### 백지 구현

**구현 목표**

순수 상태 객체로 경기 준비, 일시정지, 재접속, 만료, 종료 전이를 구현한다. 타이머와 WebSocket 호출은 포함하지 않는다.

**인터페이스 또는 함수 시그니처**

```ts
type Side = "left" | "right";
type SessionState = "waiting" | "playing" | "paused" | "reconnecting" | "finished";

interface ExpiryResult {
  forfeitingSide: Side | null;
  winnerSide: Side | null;
}

export class MatchSession {
  private currentState: SessionState = "waiting";
  private resumeState: Exclude<SessionState, "reconnecting" | "finished"> = "waiting";
  private readonly ready = new Set<Side>();
  private readonly disconnected = new Set<Side>();
  private reconnectDeadlineMs: number | null = null;

  get state(): SessionState {
    return this.currentState;
  }

  get reconnectDeadline(): number | null {
    return this.reconnectDeadlineMs;
  }

  markReady(side: Side): SessionState {
    if (this.currentState !== "waiting") return this.currentState;
    this.ready.add(side);
    if (this.ready.size === 2) this.currentState = "playing";
    return this.currentState;
  }

  pause(): SessionState {
    if (this.currentState === "playing") this.currentState = "paused";
    return this.currentState;
  }

  resume(): SessionState {
    if (this.currentState === "paused") this.currentState = "playing";
    return this.currentState;
  }

  disconnect(side: Side, nowMs: number): SessionState {
    if (!Number.isFinite(nowMs)) throw new RangeError("invalid time");
    if (this.currentState === "finished") return this.currentState;
    if (this.currentState !== "reconnecting") this.resumeState = this.currentState;
    this.disconnected.add(side);
    // 원본은 마지막 disconnect 시점부터 15초의 복구 창을 다시 계산한다.
    this.reconnectDeadlineMs = nowMs + 15_000;
    this.currentState = "reconnecting";
    return this.currentState;
  }

  reconnect(side: Side, nowMs: number): boolean {
    if (
      this.currentState !== "reconnecting"
      || this.reconnectDeadlineMs === null
      || nowMs > this.reconnectDeadlineMs
      || !this.disconnected.has(side)
    ) {
      return false;
    }
    this.disconnected.delete(side);
    if (this.disconnected.size === 0) {
      this.currentState = this.resumeState;
      this.reconnectDeadlineMs = null;
    }
    return true;
  }

  expireReconnect(nowMs: number): ExpiryResult | null {
    if (
      this.currentState !== "reconnecting"
      || this.reconnectDeadlineMs === null
      || nowMs < this.reconnectDeadlineMs
    ) {
      return null;
    }
    const [firstDisconnected] = this.disconnected;
    const forfeitingSide = this.disconnected.size === 1 ? firstDisconnected ?? null : null;
    this.finish();
    return {
      forfeitingSide,
      winnerSide: forfeitingSide === null
        ? null
        : forfeitingSide === "left" ? "right" : "left"
    };
  }

  finish(): SessionState {
    this.currentState = "finished";
    this.disconnected.clear();
    this.reconnectDeadlineMs = null;
    return this.currentState;
  }
}
```

**입력과 출력**

- 입력: 좌석, 현재 시각, 사용자 명령
- 출력: 새 상태, 재접속 성공 여부, 만료 결과

**반드시 만족해야 할 조건**

- 양쪽 준비 전에는 `playing`이 되지 않는다.
- `pause`와 `resume`은 허용된 상태에서만 의미 있는 전이를 만든다.
- 첫 disconnect 시점부터 고정된 재접속 기한을 관리한다.
- 재접속 성공 시 disconnect 이전 상태를 복원한다.
- deadline 이전 만료 호출은 결과가 없다.
- deadline 이후 만료 결과는 최대 한 번만 생성된다.
- 한쪽만 없으면 상대가 승자이고, 양쪽이 없으면 승자가 없다.
- `finished` 이후 어떤 호출도 경기를 다시 살리지 않는다.

**경계 조건**

- deadline 정확히 직전과 정확히 같은 시각
- 이미 끊긴 좌석의 중복 disconnect
- 한쪽이 끊긴 상태에서 다른 쪽도 끊김
- paused 상태에서 disconnect 후 reconnect
- finish와 expire가 연속 호출됨

**실패 조건**

- 정의되지 않은 좌석
- 시간이 뒤로 가는 입력을 허용할지 여부
- 이미 다른 사용자가 좌석을 차지했다는 검증은 이 클래스 밖의 책임으로 둔다.

**필요한 제약**

- 현재 시각은 함수 인자로 받아 테스트 가능하게 한다.
- 네트워크 전송, 저장, 타이머 생성은 상태 객체에 넣지 않는다.

### 구현 후 자가 검증

- [ ] 가능한 상태와 모든 허용 전이를 표로 설명할 수 있다.
- [ ] 양쪽 준비 순서가 바뀌어도 같은 결과가 나온다.
- [ ] paused 상태가 재접속 뒤 정확히 복원된다.
- [ ] 경계 시각에서 만료 규칙이 일관된다.
- [ ] 양쪽 disconnect는 임의 승자를 만들지 않는다.
- [ ] 만료와 종료가 중복 결과를 만들지 않는다.
- [ ] 상태 객체에 해제되지 않는 타이머나 소켓 참조가 없다.

### 구현 후 설명할 것

1. 여러 boolean보다 상태 머신이 유리한 이유
   - 모범답변: `isPaused`, `isDisconnected`, `isFinished` 조합은 paused이면서 finished 같은 불가능 상태를 만들 수 있습니다. 단일 state와 허용 전이 메서드는 가능한 상태 공간과 멱등 동작을 한곳에서 제한합니다.
2. 이전 상태를 별도로 보존하는 이유
   - 모범답변: reconnecting은 연결 복구 과정이지 경기의 이전 진행 의미를 대체하지 않습니다. `resumeState`를 보관해야 playing에서 끊긴 방은 playing으로, paused에서 끊긴 방은 paused로 정확히 돌아갑니다.
3. deadline 판정을 순수 시간 값으로 만든 이유
   - 모범답변: 상태 객체가 `Date.now()`나 timer를 만들지 않고 `nowMs`를 받으면 경계 시각을 결정적으로 테스트할 수 있습니다. timer 지연이 있어도 실제 callback 시각과 저장된 deadline을 비교해 같은 규칙을 적용합니다.
4. 양쪽 disconnect 정책과 사용자 경험 trade-off
   - 모범답변: 한쪽만 없으면 상대 승리로 경기를 끝내 자원 점유를 제한하고, 양쪽이 없으면 공통 네트워크 장애 가능성이 있어 임의 승자를 만들지 않습니다. 무승부 대신 경기 무효·재경기를 택하는 것은 제품 정책입니다.
5. 상태 전이와 외부 부수 효과를 분리한 테스트 전략
   - 모범답변: side와 시간을 넣어 state·expiry 결과만 검증하는 빠른 table test를 만들고, 별도 GameHub 테스트에서 timer 취소, socket 교체, result finalization, room cleanup이 해당 결과에 한 번만 반응하는지 확인합니다.

### 원본 확인 위치

- Thread 12
- 커밋: `feat(game): 게임 방 상태를 RoomSession에 연결`
- 커밋: `test(game): reconnect 복구 동작 검증`
- `apps/api/src/game/roomSession.ts`
  - `RoomSession`
- `apps/api/src/game/roomSession.test.ts`
- `apps/api/src/gameHub.ts`
- `apps/api/src/gameHub.reconnect.test.ts`
- 관련 Thread: 07, 22

---

## IM-12. [Thread 12 / `feat(game): WebSocket heartbeat 추가`, `feat(game): 사용자별 active connection 교체`] 연결 생존성, 단일 소유권, 교체 시 정리

### 면접 질문

사용자별 WebSocket 연결을 하나만 허용하면서 새 연결이 들어오면 기존 연결을 교체한 이유는 무엇인가요? heartbeat와 연결 교체가 큐, 입력 상태, 방 재접속 판정과 어떻게 연결되는지도 설명해 주세요.

꼬리 질문:

- TCP 연결이 열려 있어도 애플리케이션 관점에서 죽은 연결일 수 있는 이유는 무엇입니까?
  - 모범답변: 네트워크 단절, NAT·프록시 idle timeout, 절전 상태에서는 서버가 즉시 FIN/RST를 받지 못해 socket이 open처럼 남을 수 있습니다. 주기적 ping에 대한 pong deadline으로 실제 왕복 생존성을 확인해야 합니다.
- 기존 연결의 `close` 이벤트가 나중에 도착해 새 연결 상태를 지우는 문제를 어떻게 막습니까?
  - 모범답변: `clientsByUser.get(userId) === closingClient`처럼 현재 map 값과 닫히는 객체 identity가 같을 때만 제거합니다. ID나 userId만 비교하면 이미 등록된 새 연결까지 지울 수 있습니다.
- 교체 연결은 일반 disconnect와 달리 왜 즉시 몰수패 타이머를 시작하면 안 됩니까?
  - 모범답변: 새 연결이 이미 authoritative owner로 도착한 상태라 실제 부재가 아닙니다. 기존 socket close를 일반 disconnect로 처리하면 재접속 중 상태와 몰수 timer를 잘못 만들므로 전용 교체 경로에서 room side를 새 client로 넘깁니다.
- heartbeat의 interval과 timeout을 각각 어떤 기준으로 정합니까?
  - 모범답변: interval은 idle proxy보다 충분히 짧되 트래픽 부담을 제한하고, timeout은 일시적 지터와 몇 번의 누락을 견디도록 interval보다 길게 둡니다. 프로젝트는 15초 ping과 45초 timeout으로 세 주기 정도의 여유를 둡니다.

### 30초 모범 답변

연결 하나가 여러 상태 구조에 동시에 등록되므로 사용자별 단일 소유권을 정하지 않으면 두 소켓이 같은 좌석과 입력을 경쟁할 수 있습니다. 새 연결을 authoritative connection으로 먼저 등록하고 기존 소켓을 전용 코드로 닫으며, 이후 이벤트는 객체 identity를 확인해 오래된 연결이 새 상태를 삭제하지 못하게 합니다. heartbeat는 pong이 일정 시간 없으면 타이머와 전송 버퍼를 정리하고 연결을 종료합니다. 연결 교체는 사용자가 이미 돌아온 상황이므로 일반 disconnect의 재접속 유예와 구분해야 합니다.

### 답변 핵심 키워드

single owner · clientsByUser · connection replacement · stale close event · object identity · ping/pong · timer cleanup · disconnect 의미 구분

### 백지 구현

**구현 목표**

사용자별 하나의 활성 연결만 유지하는 레지스트리와 독립적인 heartbeat 수명주기를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface SocketLike {
  ping(): void;
  close(code: number, reason: string): void;
  terminate(): void;
}

interface ClientConnection {
  id: string;
  userId: string;
  socket: SocketLike;
}

export class ActiveConnectionRegistry {
  private readonly byUser = new Map<string, ClientConnection>();

  attach(connection: ClientConnection): ClientConnection | null {
    // 이전 연결을 반환하거나 null
    const previous = this.byUser.get(connection.userId) ?? null;
    this.byUser.set(connection.userId, connection);
    return previous === connection ? null : previous;
  }

  detach(connection: ClientConnection): boolean {
    // 현재 authoritative 연결만 제거
    if (this.byUser.get(connection.userId) !== connection) return false;
    this.byUser.delete(connection.userId);
    return true;
  }

  get(userId: string): ClientConnection | null {
    return this.byUser.get(userId) ?? null;
  }
}

export class Heartbeat {
  private readonly pingIntervalMs: number;
  private readonly timeoutMs: number;
  private pingTimer: ReturnType<typeof setInterval> | null = null;
  private timeoutTimer: ReturnType<typeof setTimeout> | null = null;

  constructor(
    private readonly target: Pick<SocketLike, "ping" | "terminate">,
    options?: { pingIntervalMs?: number; timeoutMs?: number }
  ) {
    this.pingIntervalMs = options?.pingIntervalMs ?? 15_000;
    this.timeoutMs = options?.timeoutMs ?? 45_000;
    if (
      !Number.isFinite(this.pingIntervalMs)
      || this.pingIntervalMs <= 0
      || !Number.isFinite(this.timeoutMs)
      || this.timeoutMs <= this.pingIntervalMs
    ) {
      throw new RangeError("invalid heartbeat timing");
    }
  }

  start(): void {
    if (this.pingTimer || this.timeoutTimer) return;
    this.armTimeout();
    this.pingTimer = setInterval(() => {
      try {
        this.target.ping();
      } catch {
        this.terminate();
      }
    }, this.pingIntervalMs);
  }

  acknowledge(): void {
    if (!this.pingTimer && !this.timeoutTimer) return;
    this.armTimeout();
  }

  stop(): void {
    if (this.pingTimer) clearInterval(this.pingTimer);
    if (this.timeoutTimer) clearTimeout(this.timeoutTimer);
    this.pingTimer = null;
    this.timeoutTimer = null;
  }

  private armTimeout(): void {
    if (this.timeoutTimer) clearTimeout(this.timeoutTimer);
    this.timeoutTimer = setTimeout(() => this.terminate(), this.timeoutMs);
  }

  private terminate(): void {
    this.stop();
    try {
      this.target.terminate();
    } catch {
      // 타이머 ownership은 이미 정리됐으므로 소켓 종료 예외를 heartbeat 밖으로 전파하지 않는다.
    }
  }
}
```

**입력과 출력**

- 입력: 사용자·연결 객체, ping/pong 신호
- 출력: 교체된 이전 연결 또는 제거 성공 여부

**반드시 만족해야 할 조건**

- 사용자마다 활성 연결은 최대 하나다.
- 새 연결 등록은 이전 연결을 식별해 반환한다.
- 이전 연결의 늦은 detach는 새 연결을 제거하지 못한다.
- heartbeat `start`는 중복 타이머를 만들지 않는다.
- `acknowledge`는 timeout을 갱신한다.
- ping 실패나 timeout 시 정확히 한 번 종료한다.
- `stop` 뒤 남은 interval·timeout이 없다.

**경계 조건**

- 같은 연결을 두 번 attach
- 교체 직후 이전 연결의 close 이벤트
- `start`, `stop`, `start` 반복
- ping 호출 자체가 예외를 던짐
- timeout과 acknowledge가 같은 event-loop turn에 발생

**실패 조건**

- 소켓 종료 함수가 예외를 던지는 경우의 처리 정책
- 음수 또는 timeout보다 긴 ping interval 설정

**필요한 제약**

- 실제 시간 대신 fake timer로 검증할 수 있어야 한다.
- 레지스트리는 방 상태 변경을 직접 수행하지 않고 호출자에게 교체 사실을 반환한다.

### 구현 후 자가 검증

- [ ] 새 연결이 등록된 뒤 오래된 close가 새 연결을 지우지 않는다.
- [ ] 한 사용자에 대한 authoritative 연결이 항상 하나다.
- [ ] heartbeat 타이머가 중복 생성되지 않는다.
- [ ] pong 수신 시 timeout이 정확히 재설정된다.
- [ ] stop과 terminate 뒤 타이머가 남지 않는다.
- [ ] 교체와 실제 단절을 상위 계층에서 구분할 수 있다.

### 구현 후 설명할 것

1. 연결 ID가 아니라 연결 객체 identity까지 확인하는 이유
   - 모범답변: userId는 교체 전후에 같고 외부 ID도 재사용될 수 있지만 객체 identity는 그 callback이 소유한 정확한 connection generation을 나타냅니다. 현재 map 값과 동일할 때만 detach해야 stale close가 무해합니다.
2. 새 연결 등록과 이전 연결 종료의 순서
   - 모범답변: 새 연결을 먼저 `clientsByUser`의 authoritative 값으로 등록한 뒤 room·queue ownership을 넘기고 이전 socket을 전용 코드로 닫습니다. 그래야 그 사이 들어온 오래된 이벤트가 identity 검사에서 탈락합니다.
3. heartbeat가 애플리케이션 상태 정리에 연결되는 방식
   - 모범답변: heartbeat는 pong이 없으면 timer를 모두 정리하고 socket을 terminate하며, 실제 close handler가 GameHub의 queue·input·snapshot·room reconnect cleanup을 실행합니다. heartbeat 자체는 게임 상태를 소유하지 않습니다.
4. 교체 close code를 일반 네트워크 종료와 구분하는 이유
   - 모범답변: 교체는 사용자가 이미 돌아온 정상 ownership 이전이고 일반 close는 실제 부재일 수 있습니다. 원본의 4001 전용 코드와 별도 `replaceConnection` 경로가 불필요한 reconnect 유예와 몰수 판정을 막습니다.
5. heartbeat 주기와 오탐·복구 시간의 trade-off
   - 모범답변: 짧은 timeout은 죽은 연결을 빨리 회수하지만 모바일 지터·event-loop 정지에 오탐이 늘고, 긴 timeout은 좌석·queue 자원을 오래 붙잡습니다. 트래픽과 장애 관측값으로 15/45초 같은 값을 조정해야 합니다.

### 원본 확인 위치

- Thread 12
- 커밋: `feat(game): WebSocket heartbeat 추가`
- 커밋: `feat(game): 사용자별 active connection 교체`
- `apps/api/src/game/heartbeat.ts`
  - `ConnectionHeartbeat`
  - `HEARTBEAT_PING_INTERVAL_MS`
  - `HEARTBEAT_TIMEOUT_MS`
- `apps/api/src/gameHub.ts`
  - `clientsByUser`
- `apps/api/src/gameHub.reconnect.test.ts`
- 관련 Thread: 07, 08

---

## IM-13. [Thread 13 / `feat(game): fixed-step scheduler 추가`, `feat(game): fixed-step scheduler를 GameHub에 연결`] 고정 시간 간격 실행과 지연 누적 제한

### 면접 질문

실시간 게임 시뮬레이션을 실제 콜백 간격만큼 움직이지 않고 fixed-step accumulator로 실행한 이유는 무엇인가요? 이벤트 루프가 잠시 밀렸을 때 catch-up 횟수와 누적 지연을 제한해야 하는 이유도 설명해 주세요.

꼬리 질문:

- `setInterval(50)`만 사용하는 것과 accumulator 방식은 무엇이 다릅니까?
  - 모범답변: `setInterval` callback은 event-loop 지터로 실제 간격이 달라지고 누락 횟수를 알려주지 않습니다. accumulator는 monotonic 경과 시간을 더한 뒤 항상 50ms 단위 step 수로 바꿔 물리 계산량을 일정하게 만듭니다.
- 긴 정지 뒤 모든 누락 tick을 처리하면 어떤 문제가 생깁니까?
  - 모범답변: 한 loop에서 수백 tick을 돌며 event loop를 다시 막고 다음 callback도 늦추는 spiral of death가 생깁니다. 원본은 누적 250ms와 loop당 5 tick으로 작업량을 제한합니다.
- 남은 accumulator를 버리는 것과 유지하는 것의 trade-off는 무엇입니까?
  - 모범답변: 유지하면 짧은 지터를 다음 loop에서 따라잡아 simulation 시간이 정확하지만 과부하가 지속되면 계속 뒤처집니다. 상한 초과분을 버리면 실시간 응답성을 지키는 대신 wall time 대비 simulation 시간이 느려지는 왜곡을 감수합니다.
- 여러 방마다 타이머를 둘 때와 공유 스케줄러를 둘 때 어떤 차이가 있습니까?
  - 모범답변: 방별 타이머는 구현이 독립적이지만 방 수만큼 timer callback과 지터가 늘어납니다. 프로젝트는 `SharedRoomScheduler`로 활성 방을 한 loop에서 순회해 timer 자원을 제한하는 대신 한 방의 무거운 step이 전체 방에 영향을 주지 않도록 step 비용을 제한해야 합니다.

### 30초 모범 답변

게임 규칙은 실제 콜백 지터가 아니라 일정한 tick에 대해 계산해야 결정성과 재현성이 좋아집니다. 그래서 경과 시간을 accumulator에 더하고 고정 step만큼 여러 번 처리합니다. 다만 프로세스가 오래 멈춘 뒤 모든 tick을 따라잡으면 event loop를 더 막는 spiral of death가 생기므로 최대 누적 시간과 한 번의 catch-up step 수를 제한합니다. 렌더링·스냅샷 주기는 시뮬레이션 tick과 별도로 조절할 수 있습니다.

### 답변 핵심 키워드

fixed timestep · accumulator · event-loop jitter · catch-up cap · spiral of death · monotonic clock · simulation/render 분리 · 공유 scheduler

### 백지 구현

**구현 목표**

경과 시간을 누적해 고정 크기 step 개수를 반환하되, 한 번에 처리할 최대 step과 최대 누적 시간을 제한하는 순수 accumulator를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface StepResult {
  steps: number;
  remainingMs: number;
  droppedMs: number;
}

export class FixedStepAccumulator {
  private accumulatedMs = 0;

  constructor(
    readonly stepMs: number,
    readonly maxStepsPerAdvance: number,
    readonly maxAccumulatedMs: number
  ) {
    if (!Number.isFinite(stepMs) || stepMs <= 0) {
      throw new RangeError("stepMs must be positive");
    }
    if (!Number.isInteger(maxStepsPerAdvance) || maxStepsPerAdvance <= 0) {
      throw new RangeError("maxStepsPerAdvance must be a positive integer");
    }
    if (!Number.isFinite(maxAccumulatedMs) || maxAccumulatedMs < stepMs) {
      throw new RangeError("maxAccumulatedMs must cover at least one step");
    }
  }

  advance(elapsedMs: number): StepResult {
    if (!Number.isFinite(elapsedMs) || elapsedMs < 0) {
      throw new RangeError("elapsedMs must be finite and non-negative");
    }
    const uncapped = this.accumulatedMs + elapsedMs;
    const capped = Math.min(this.maxAccumulatedMs, uncapped);
    const droppedMs = uncapped - capped;
    const availableSteps = Math.floor(capped / this.stepMs);
    const steps = Math.min(this.maxStepsPerAdvance, availableSteps);
    this.accumulatedMs = capped - steps * this.stepMs;
    return { steps, remainingMs: this.accumulatedMs, droppedMs };
  }

  reset(): void {
    this.accumulatedMs = 0;
  }
}
```

**입력과 출력**

- 입력: 직전 실행 이후의 경과 밀리초
- 출력: 이번에 실행할 고정 step 수, 남은 누적 시간, 버린 지연 시간

**반드시 만족해야 할 조건**

- step보다 작은 지연은 누적만 하고 0 step을 반환한다.
- 정확한 배수는 그 수만큼 step을 반환한다.
- 반환 step 수는 설정한 최대값을 넘지 않는다.
- 누적 시간은 설정한 상한을 넘지 않는다.
- 음수·NaN·무한대 입력을 안전하게 처리하거나 명시적으로 거부한다.
- `reset` 뒤 이전 지연이 다음 게임에 섞이지 않는다.

**경계 조건**

- `0`, `stepMs - 1`, `stepMs`, 큰 지연
- 반복되는 작은 지연의 합
- 매우 큰 지연 뒤 정상 지연
- 부동소수점 경과 시간

**실패 조건**

- `stepMs <= 0`
- 최대 step 또는 누적 상한이 잘못된 설정
- 시간이 뒤로 간 값

**필요한 제약**

- 시계 호출과 타이머 생성은 별도 스케줄러 책임으로 둔다.
- 한 번의 `advance`는 입력 크기와 무관하게 제한된 작업만 수행한다.

### 구현 후 자가 검증

- [ ] 작은 지연이 여러 번 누적돼 정확한 step으로 변환된다.
- [ ] 긴 정지 뒤 step 수가 상한을 넘지 않는다.
- [ ] 버려진 시간과 남은 시간이 음수가 아니다.
- [ ] reset 뒤 누적값이 0이다.
- [ ] 호출 한 번의 시간 복잡도가 제한된다.
- [ ] wall clock 대신 monotonic clock이 필요한 이유를 설명할 수 있다.

### 구현 후 설명할 것

1. fixed timestep이 물리 결정성에 주는 효과
   - 모범답변: 매 update가 동일한 50ms를 사용하면 callback 지터와 무관하게 같은 초기 상태·입력 순서에서 같은 tick 결과를 계산하기 쉽습니다. replay와 테스트도 wall-clock 간격 대신 tick sequence로 재현할 수 있습니다.
2. catch-up 상한이 필요한 이유
   - 모범답변: 긴 정지의 모든 step을 한 번에 처리하면 CPU를 독점해 지연을 더 키웁니다. 누적 시간과 loop당 tick 수를 함께 제한해 한 callback의 작업량을 bounded하게 유지합니다.
3. 지연을 버릴 때 발생하는 시간 왜곡과 대안
   - 모범답변: 버린 시간만큼 simulation이 실제 시간보다 덜 진행되어 공·paddle 움직임이 순간적으로 느려집니다. 대안은 서버 용량 확장, 방당 비용 축소, 일시적 품질 저하이며 임의의 큰 variable step은 충돌 안정성과 결정성을 해칠 수 있습니다.
4. 시뮬레이션 tick과 네트워크 snapshot cadence 분리
   - 모범답변: 원본은 simulation을 고정 step으로 진행하면서 snapshot은 delivery divisor로 더 낮은 주기에 보냅니다. 게임 규칙의 해상도는 유지하고 직렬화·네트워크 비용은 별도로 조절하기 위해서입니다.
5. 방별 타이머와 공유 스케줄러의 리소스 trade-off
   - 모범답변: 공유 스케줄러는 timer 수와 wake-up을 줄이고 일괄 관측하기 쉽지만 한 loop의 총 방 step 시간이 중요해집니다. 방별 timer는 격리는 단순해 보여도 많은 방에서 callback 폭주와 scheduling overhead가 커집니다.

### 원본 확인 위치

- Thread 13
- 커밋: `feat(game): fixed-step scheduler 추가`
- 커밋: `feat(game): fixed-step scheduler를 GameHub에 연결`
- `apps/api/src/game/fixedStepScheduler.ts`
  - `FixedStepAccumulator`
  - `FixedStepScheduler`
- `apps/api/src/gameHub.ts`
- 관련 Thread: 09, 21, 23

---

## IM-14. [Thread 13 / `test(game): snapshot replacement와 congestion 검증`, `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`] 최신 상태 전송과 WebSocket backpressure

### 면접 질문

서버가 게임 snapshot을 생성하는 속도보다 클라이언트 전송이 느릴 때 모든 snapshot을 무한히 큐에 쌓지 않은 이유는 무엇인가요? 또한 `send` callback이 아직 오지 않았다는 사실과 실제 `bufferedAmount` 혼잡을 구분해야 하는 이유를 설명해 주세요.

꼬리 질문:

- 게임 snapshot에서 중간 프레임을 버릴 수 있지만 채팅 메시지는 버리면 안 되는 이유는 무엇입니까?
  - 모범답변: snapshot은 전체 최신 상태라 새 값이 이전 값을 포함해 대체하지만, 채팅은 각 메시지가 독립 사건이라 하나를 버리면 의미가 영구히 사라집니다. 그래서 latest-state buffer는 snapshot에만 쓰고 채팅은 순서·전달 정책이 있는 별도 경로로 보냅니다.
- soft limit과 hard limit을 따로 두면 어떤 정책을 만들 수 있습니까?
  - 모범답변: 256KiB soft 초과에서는 pending을 최신 하나로 합치고 50ms 뒤 재시도해 일시 혼잡을 견딥니다. 1MiB hard 도달 또는 5초 지속 시에는 회복 가능성이 낮다고 보고 연결을 종료해 자원을 회수합니다.
- 장시간 혼잡한 연결을 종료하지 않으면 서버 전체에 어떤 위험이 생깁니까?
  - 모범답변: 느린 client마다 pending payload·retry timer·room/heartbeat 상태가 계속 남고 socket 전송 버퍼도 커져 메모리와 event-loop 작업을 잠식합니다. 다수의 느린 연결이 정상 사용자까지 지연시키는 slow-consumer 장애가 됩니다.
- drop, delivery delay, congestion 종료를 어떤 메트릭으로 관측하겠습니까?
  - 모범답변: `replaced`, `connection_closed`, `congestion` reason별 drop counter, enqueue부터 callback 성공까지 delivery latency histogram, hard/timeout terminate counter를 봅니다. room 수·bufferedAmount 분포와 함께 p95/p99를 비교해 threshold를 조정합니다.

### 30초 모범 답변

snapshot은 최신 전체 상태이므로 오래된 중간 값을 모두 전달하는 것보다 최신 값으로 대체하는 편이 지연을 줄입니다. 실제 혼잡 신호는 소켓의 전송 버퍼 크기이고, callback 지연만으로 전송 정체라고 단정하면 정상 연결에서도 불필요하게 snapshot을 버립니다. soft limit에서는 최신 pending 하나만 유지하며 재시도하고, hard limit 또는 일정 시간 이상 혼잡하면 연결을 종료해 메모리와 타이머를 회수합니다. 손실 허용 메시지와 손실 불가 메시지는 별도 경로를 사용해야 합니다.

### 답변 핵심 키워드

backpressure · latest-state coalescing · bufferedAmount · callback latency 구분 · soft/hard threshold · congestion timeout · bounded memory · drop metrics

### 백지 구현

**구현 목표**

최신 상태 메시지 전용 버퍼를 구현한다. 정상 전송 시 불필요하게 callback을 기다려 직렬화하지 않고, 실제 전송 버퍼가 혼잡할 때만 최신 pending 상태로 합친다.

**인터페이스 또는 함수 시그니처**

```ts
interface SnapshotSocket {
  readonly bufferedAmount: number;
  send(payload: string, callback: (error?: Error) => void): void;
  terminate(): void;
}

type DropReason = "replaced" | "connection_closed" | "congestion";

export class LatestStateSender {
  private readonly now: () => number;
  private readonly onDelivered: (delayMs: number) => void;
  private readonly onDropped: (reason: DropReason) => void;
  private pending: { payload: string; enqueuedAtMs: number } | null = null;
  private congestionStartedAtMs: number | null = null;
  private retryTimer: ReturnType<typeof setTimeout> | null = null;
  private closed = false;

  constructor(
    private readonly socket: SnapshotSocket,
    options?: {
      now?: () => number;
      onDelivered?: (delayMs: number) => void;
      onDropped?: (reason: DropReason) => void;
    }
  ) {
    this.now = options?.now ?? (() => performance.now());
    this.onDelivered = options?.onDelivered ?? (() => undefined);
    this.onDropped = options?.onDropped ?? (() => undefined);
  }

  enqueue(payload: string): void {
    if (this.closed) return;
    if (this.pending) this.onDropped("replaced");
    this.pending = { payload, enqueuedAtMs: this.now() };
    this.drain();
  }

  close(reason?: DropReason): void {
    if (this.closed) return;
    this.closed = true;
    if (this.pending) this.onDropped(reason ?? "connection_closed");
    this.pending = null;
    if (this.retryTimer) clearTimeout(this.retryTimer);
    this.retryTimer = null;
  }

  private drain(): void {
    if (this.closed) return;
    const nowMs = this.now();
    if (this.socket.bufferedAmount >= 1_024 * 1_024) {
      this.terminate("congestion");
      return;
    }
    if (this.socket.bufferedAmount > 256 * 1_024) {
      this.congestionStartedAtMs ??= nowMs;
      if (nowMs - this.congestionStartedAtMs >= 5_000) {
        this.terminate("congestion");
        return;
      }
      this.armRetry();
      return;
    }

    this.congestionStartedAtMs = null;
    const snapshot = this.pending;
    if (!snapshot) return;
    this.pending = null;
    try {
      // callback 대기 중이어도 transport buffer가 낮으면 다음 snapshot을 즉시 보낼 수 있다.
      this.socket.send(snapshot.payload, (error) => {
        if (error) {
          this.onDropped("connection_closed");
          this.terminate("connection_closed");
          return;
        }
        this.onDelivered(Math.max(0, this.now() - snapshot.enqueuedAtMs));
      });
    } catch {
      this.onDropped("connection_closed");
      this.terminate("connection_closed");
    }
  }

  private armRetry(): void {
    if (this.retryTimer || this.closed) return;
    this.retryTimer = setTimeout(() => {
      this.retryTimer = null;
      this.drain();
    }, 50);
  }

  private terminate(reason: DropReason): void {
    if (this.closed) return;
    this.close(reason);
    try {
      this.socket.terminate();
    } catch {
      // sender의 timer와 pending ownership은 이미 정리됐다.
    }
  }
}
```

**입력과 출력**

- 입력: 순서대로 생성되는 snapshot 문자열
- 출력: 소켓 전송, drop·delivery 관측 콜백

**반드시 만족해야 할 조건**

- 닫힌 뒤 새 입력을 무시한다.
- 정상적인 낮은 `bufferedAmount`에서는 callback 대기만을 이유로 혼잡 처리하지 않는다.
- soft threshold 초과 중에는 오래된 pending 대신 가장 최신 snapshot만 남긴다.
- hard threshold 이상이거나 혼잡 지속 시간이 상한을 넘으면 연결을 종료한다.
- send가 동기 예외를 던지거나 callback으로 실패해도 pending·retry timer가 정리된다.
- `close`는 멱등이고 남은 pending drop을 한 번만 기록한다.
- retry timer는 동시에 하나만 존재한다.

**경계 조건**

- enqueue 직후 close
- soft threshold 경계값과 hard threshold 경계값
- 혼잡 중 여러 snapshot 도착
- callback이 매우 늦지만 `bufferedAmount`는 낮은 경우
- send callback 실패와 close가 거의 동시에 발생

**실패 조건**

- 동기 `send` 예외
- 비동기 callback 오류
- 장기 혼잡
- 소켓 종료 함수 실패에 대한 정책

**필요한 제약**

- 대기 메모리는 입력 수와 무관하게 상수 크기로 제한한다.
- snapshot 외 메시지에는 이 버퍼를 재사용하지 않는다고 명시한다.
- fake clock과 fake socket으로 테스트 가능해야 한다.

### 구현 후 자가 검증

- [ ] 정상 전송에서 callback 지연만으로 snapshot이 drop되지 않는다.
- [ ] 혼잡 중 pending 수가 1을 넘지 않는다.
- [ ] 가장 최신 상태가 최종적으로 선택된다.
- [ ] hard limit과 지속 시간 만료에서 연결이 종료된다.
- [ ] send 실패와 close 뒤 retry timer가 남지 않는다.
- [ ] drop 이유가 중복 기록되지 않는다.
- [ ] 메모리 사용량이 O(1)임을 설명할 수 있다.

### 구현 후 설명할 것

1. 최신 전체 상태와 순서 보장 메시지의 전달 정책 차이
   - 모범답변: snapshot은 최신 값 하나가 이전 상태를 대체하므로 혼잡 시 coalesce해 지연을 줄입니다. 채팅·경기 종료 같은 사건은 누락과 재정렬이 의미를 바꾸므로 별도 ordered 전송 또는 저장 후 재조회 경로가 필요합니다.
2. callback 완료와 transport buffer 혼잡을 분리한 이유
   - 모범답변: `send` callback은 event-loop scheduling 때문에 늦을 수 있어도 OS/WebSocket buffer는 비어 있을 수 있습니다. 실제 `bufferedAmount`가 낮다면 callback을 기다리지 않고 다음 snapshot을 보내야 정상 연결의 불필요한 drop을 막습니다.
3. soft/hard threshold 및 혼잡 timeout의 역할
   - 모범답변: soft는 일시 혼잡에서 최신 pending 하나만 유지하며 회복 기회를 주고, hard는 즉시 메모리 보호를 수행합니다. timeout은 soft 구간에 영원히 머무는 느린 연결도 유한 시간 안에 정리합니다.
4. 연결 종료가 과격해 보여도 필요한 자원 보호 이유
   - 모범답변: 실시간 서버는 한 client의 완전 전달보다 전체 방의 bounded memory와 event-loop 지연이 중요합니다. 회복하지 못하는 slow consumer를 종료해야 다른 연결의 simulation과 heartbeat를 보호할 수 있습니다.
5. drop·지연 메트릭을 이용한 threshold 튜닝 방법
   - 모범답변: 정상 부하에서 delivery p95/p99와 replaced 비율을 보고 soft threshold가 너무 낮은지 판단합니다. congestion 종료가 특정 네트워크에 과도하면 timeout을 조정하되 heap, bufferedAmount, event-loop lag가 함께 악화되지 않는지 확인합니다.

### 원본 확인 위치

- Thread 13
- 커밋: `test(game): snapshot replacement와 congestion 검증`
- 커밋: `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`
- `apps/api/src/game/latestSnapshotBuffer.ts`
  - `LatestSnapshotBuffer`
- `apps/api/src/game/latestSnapshotBuffer.test.ts`
- `apps/api/src/gameHub.ts`
- 관련 Thread: 21, 23

---

## IM-15. [Thread 22 / `feat(game): 새 작업 차단과 active room drain 추가`, `feat(ops): graceful shutdown 절차 추가`] 새 작업 차단, 진행 중 작업 drain, 강제 종료 상한

### 면접 질문

실시간 게임 서버가 SIGTERM을 받았을 때 바로 프로세스를 닫지 않고, 새 경기만 차단한 뒤 진행 중인 방을 기다리는 drain 절차를 둔 이유는 무엇인가요? 무한 대기를 막는 timeout과 컨테이너 종료 유예의 관계도 설명해 주세요.

꼬리 질문:

- 대기열 사용자와 토너먼트 대기자는 drain 시작 시 어떻게 처리해야 합니까?
  - 모범답변: 새 경기 admission을 먼저 닫고 일반 queue entry의 fallback timer·reservation을 정리한 뒤 `server_draining` 오류를 보냅니다. 토너먼트 waiter도 같은 오류를 보내고 map을 비워, 시작되지 않은 작업이 drain 중 새 room으로 바뀌지 않게 합니다.
- 여러 종료 신호가 연속으로 오면 종료 절차가 두 번 실행되지 않게 어떻게 합니까?
  - 모범답변: signal listener가 공유하는 `started` flag를 첫 callback에서 동기적으로 true로 바꾸고 shutdown Promise를 한 번만 시작합니다. disposer는 SIGTERM·SIGINT listener를 모두 제거합니다.
- 진행 중 경기의 DB 확정 재시도가 남아 있을 때 drain 완료를 어떻게 정의합니까?
  - 모범답변: 원본에서는 room이 finalization 성공 후 `removeFinishedRoom`될 때까지 active이므로 retry timer와 확정 Promise도 room 생명주기에 포함됩니다. `rooms.size === 0`이 되어야 drained true이고, timeout이면 남은 room 수를 반환합니다.
- `stop_grace_period`가 애플리케이션 drain timeout보다 짧으면 무슨 일이 생깁니까?
  - 모범답변: 애플리케이션이 room 종료·DB 확정·socket/DB close를 마치기 전에 컨테이너 런타임이 SIGKILL로 강제 종료할 수 있습니다. 프로젝트는 내부 60초 drain보다 긴 70초 grace를 둡니다.

### 30초 모범 답변

즉시 종료하면 진행 중 경기와 결과 저장이 끊기므로 먼저 서버를 `accepting`에서 `draining`으로 전환해 새 큐·토너먼트 참가를 거부하고, 대기 중인 사용자에게 명시적 오류를 보냅니다. 이미 시작된 방은 완료와 결과 확정까지 기다리되, 운영 환경이 영원히 종료되지 않도록 timeout을 둡니다. 종료 신호 핸들러는 단 한 번만 시작되고, 컨테이너의 종료 유예는 애플리케이션 drain 상한보다 길어야 합니다. 마지막에는 타이머·heartbeat·소켓·DB pool을 모두 닫습니다.

### 답변 핵심 키워드

draining state · admission control · in-flight completion · bounded shutdown · idempotent signal handler · timer cleanup · DB close · orchestrator grace period

### 백지 구현

**구현 목표**

활성 작업 수를 추적하는 서비스에서 새 작업을 차단하고, 작업이 모두 끝나거나 timeout이 되면 결과를 반환하는 drain 조정기를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface DrainResult {
  drained: boolean;
  activeTasks: number;
}

export class DrainCoordinator {
  private accepting = true;
  private activeTasks = 0;
  private waiter: {
    promise: Promise<DrainResult>;
    resolve: (result: DrainResult) => void;
    timer: ReturnType<typeof setTimeout>;
  } | null = null;

  beginTask(): { release(): void } | null {
    // draining 중이면 null
    if (!this.accepting) return null;
    this.activeTasks += 1;
    let released = false;
    return {
      release: () => {
        if (released) return;
        released = true;
        this.activeTasks -= 1;
        if (this.activeTasks < 0) throw new Error("active task invariant violated");
        if (this.waiter && this.activeTasks === 0) {
          this.finishDrain({ drained: true, activeTasks: 0 });
        }
      }
    };
  }

  beginDrain(timeoutMs: number): Promise<DrainResult> {
    if (!Number.isFinite(timeoutMs) || timeoutMs < 0) {
      throw new RangeError("timeoutMs must be non-negative");
    }
    // admission을 닫는 동기 전이가 active count 관측보다 먼저다.
    this.accepting = false;
    if (this.activeTasks === 0) {
      return Promise.resolve({ drained: true, activeTasks: 0 });
    }
    if (this.waiter) return this.waiter.promise;

    let resolveDrain: (result: DrainResult) => void = () => undefined;
    const promise = new Promise<DrainResult>((resolve) => {
      resolveDrain = resolve;
    });
    const timer = setTimeout(() => {
      this.finishDrain({ drained: false, activeTasks: this.activeTasks });
    }, timeoutMs);
    this.waiter = { promise, resolve: resolveDrain, timer };
    return promise;
  }

  close(): void {
    this.accepting = false;
    if (this.waiter) {
      this.finishDrain({ drained: false, activeTasks: this.activeTasks });
    }
  }

  private finishDrain(result: DrainResult): void {
    const waiter = this.waiter;
    if (!waiter) return;
    this.waiter = null;
    clearTimeout(waiter.timer);
    waiter.resolve(result);
  }
}

export function installSingleShotShutdown(
  signalSource: {
    on(event: "SIGTERM" | "SIGINT", listener: () => void): void;
    off(event: "SIGTERM" | "SIGINT", listener: () => void): void;
  },
  shutdown: (signal: "SIGTERM" | "SIGINT") => Promise<void>,
  onError: (error: unknown) => void
): () => void {
  let started = false;
  const start = (signal: "SIGTERM" | "SIGINT") => {
    if (started) return;
    started = true;
    void shutdown(signal).catch(onError);
  };
  const onSigterm = () => start("SIGTERM");
  const onSigint = () => start("SIGINT");
  signalSource.on("SIGTERM", onSigterm);
  signalSource.on("SIGINT", onSigint);

  return () => {
    signalSource.off("SIGTERM", onSigterm);
    signalSource.off("SIGINT", onSigint);
  };
}
```

**입력과 출력**

- 입력: 작업 시작·종료, drain timeout, 종료 신호
- 출력: 작업 lease 또는 drain 결과, signal listener 해제 함수

**반드시 만족해야 할 조건**

- drain 시작 뒤 새 작업은 받지 않는다.
- 기존 작업이 0개면 즉시 성공한다.
- 마지막 작업 release 시 대기 중 drain이 즉시 성공한다.
- timeout 시 `drained: false`와 남은 작업 수를 반환한다.
- `beginDrain` 중복 호출은 같은 진행 중 결과를 공유한다.
- lease의 `release`는 여러 번 호출돼도 카운트를 음수로 만들지 않는다.
- SIGTERM·SIGINT가 여러 번 와도 shutdown은 한 번만 시작한다.
- listener를 제거할 수 있고, close 뒤 timer가 남지 않는다.

**경계 조건**

- timeout 0
- 작업이 끝나는 시점과 timeout이 같은 turn
- drain 시작과 작업 시작이 거의 동시에 호출됨
- 같은 lease의 중복 release
- shutdown Promise가 reject

**실패 조건**

- 잘못된 음수 timeout
- 작업 카운터 invariant 위반
- signal callback 내부 예외

**필요한 제약**

- 전역 프로세스 객체에 직접 결합하지 않고 signal source를 주입한다.
- 종료 완료 판단에 포함할 작업의 정의를 호출자가 명시해야 한다.

### 구현 후 자가 검증

- [ ] drain 시작 순간부터 admission이 닫힌다.
- [ ] 기존 작업이 모두 release되면 timeout 전에 완료된다.
- [ ] timeout 뒤 Promise가 다시 resolve되지 않는다.
- [ ] 중복 drain과 중복 release가 무해하다.
- [ ] signal이 여러 번 와도 shutdown 호출은 한 번이다.
- [ ] 모든 timeout과 listener가 cleanup된다.
- [ ] 외부 종료 유예가 내부 timeout보다 길어야 함을 설명할 수 있다.

### 구현 후 설명할 것

1. readiness와 liveness가 drain 중 어떻게 달라져야 하는지
   - 모범답변: 프로세스는 drain을 수행할 만큼 살아 있으므로 liveness는 계속 200이어야 하지만 새 트래픽을 받으면 안 되므로 readiness는 lifecycle=`draining`, HTTP 503이 되어야 합니다. 원본 `/health/ready`가 이 둘을 분리합니다.
2. 대기 작업과 진행 중 작업을 다르게 처리한 이유
   - 모범답변: queue·tournament waiter는 아직 사용자 결과를 만드는 in-flight 경기가 아니어서 명시적 오류와 함께 즉시 해제할 수 있습니다. 생성된 room은 simulation과 최종 저장을 끝낼 기회를 줘 결과 유실을 줄입니다.
3. 결과 저장 재시도를 active work에 포함할지에 대한 판단
   - 모범답변: 이 프로젝트는 finalization retry가 끝나야 room을 제거하므로 active room에 포함합니다. 경기 화면 종료만 보고 drain을 완료하면 commit되지 않은 결과를 프로세스 종료로 잃을 수 있기 때문입니다.
4. timeout 이후 강제 종료에서 감수하는 데이터 위험
   - 모범답변: 남은 simulation·finalization이 중단되고 client가 종료 event를 받지 못할 수 있습니다. 대신 서버가 무한히 종료되지 않는 운영 상한을 보장하며, `resultKey` 멱등 저장으로 재처리 가능한 범위를 넓힙니다.
5. 애플리케이션 timeout과 컨테이너 `stop_grace_period` 정렬
   - 모범답변: 외부 grace에는 내부 60초 drain뿐 아니라 Fastify close, socket·timer cleanup, DB pool 종료 여유까지 포함해야 합니다. 현재 compose의 70초처럼 내부 상한보다 길게 두되 실제 종료 latency를 관측해 margin을 정합니다.

### 원본 확인 위치

- Thread 22
- 커밋: `feat(game): 새 작업 차단과 active room drain 추가`
- 커밋: `feat(ops): graceful shutdown 절차 추가`
- 커밋: `test(ops): GameHub drain과 graceful shutdown 검증`
- 커밋: `fix(runtime): container 종료 유예를 room drain과 정렬`
- `apps/api/src/gameHub.ts`
  - `beginDrain`
  - `close`
  - `notifyDrainProgress`
  - `finishDrain`
- `apps/api/src/gracefulShutdown.ts`
  - `installGracefulShutdown`
- `apps/api/src/gameHub.drain.test.ts`
- `docker-compose.yml`
- 관련 Thread: 12, 15, 21
