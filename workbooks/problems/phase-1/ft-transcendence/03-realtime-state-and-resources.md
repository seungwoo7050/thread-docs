# 실시간 상태·연결·리소스 수명주기 면접 워크북

이 문서는 WebSocket 기반 게임 서버에서 상태 머신, 타이머, 연결 교체, 재접속, 전송 혼잡, 서버 종료를 서로 충돌하지 않게 다루는 문제를 다룬다.

---

## IM-11. [Thread 12 / `feat(game): 게임 방 상태를 RoomSession에 연결`, `test(game): reconnect 복구 동작 검증`] 경기 방 상태 머신과 재접속 유예

### 면접 질문

경기 방 상태를 여러 boolean과 조건문으로 관리하지 않고 `RoomSession` 상태 머신으로 분리한 이유는 무엇인가요? 연결이 끊긴 뒤 15초 동안 이전 `playing` 또는 `paused` 상태를 보존하고, 만료 시 몰수패 또는 무승부로 끝내는 흐름을 설명해 주세요.

꼬리 질문:

- 양쪽이 거의 동시에 끊겼을 때 승자를 정하지 않아야 하는 이유와 판정 시점은 무엇입니까?
- 재접속 직전에 만료 타이머가 실행되는 경쟁 상태를 어떻게 막겠습니까?
- `pause()`나 `resume()`을 잘못된 상태에서 호출했을 때 예외와 멱등 반환 중 무엇을 선택하겠습니까?
- 상태 머신과 WebSocket·스케줄러 같은 부수 효과를 어떻게 분리해 테스트합니까?

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
  readonly state: SessionState;
  readonly reconnectDeadline: number | null;

  markReady(side: Side): SessionState {
    // 직접 구현
    throw new Error("not implemented");
  }

  pause(): SessionState {
    // 직접 구현
    throw new Error("not implemented");
  }

  resume(): SessionState {
    // 직접 구현
    throw new Error("not implemented");
  }

  disconnect(side: Side, nowMs: number): SessionState {
    // 직접 구현
    throw new Error("not implemented");
  }

  reconnect(side: Side, nowMs: number): boolean {
    // 직접 구현
    throw new Error("not implemented");
  }

  expireReconnect(nowMs: number): ExpiryResult | null {
    // 직접 구현
    throw new Error("not implemented");
  }

  finish(): SessionState {
    // 직접 구현
    throw new Error("not implemented");
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
2. 이전 상태를 별도로 보존하는 이유
3. deadline 판정을 순수 시간 값으로 만든 이유
4. 양쪽 disconnect 정책과 사용자 경험 trade-off
5. 상태 전이와 외부 부수 효과를 분리한 테스트 전략

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
- 기존 연결의 `close` 이벤트가 나중에 도착해 새 연결 상태를 지우는 문제를 어떻게 막습니까?
- 교체 연결은 일반 disconnect와 달리 왜 즉시 몰수패 타이머를 시작하면 안 됩니까?
- heartbeat의 interval과 timeout을 각각 어떤 기준으로 정합니까?

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
  attach(connection: ClientConnection): ClientConnection | null {
    // 이전 연결을 반환하거나 null
    throw new Error("not implemented");
  }

  detach(connection: ClientConnection): boolean {
    // 현재 authoritative 연결만 제거
    throw new Error("not implemented");
  }

  get(userId: string): ClientConnection | null {
    // 직접 구현
    throw new Error("not implemented");
  }
}

export class Heartbeat {
  constructor(
    target: Pick<SocketLike, "ping" | "terminate">,
    options?: { pingIntervalMs?: number; timeoutMs?: number }
  ) {}

  start(): void {
    // 직접 구현
  }

  acknowledge(): void {
    // 직접 구현
  }

  stop(): void {
    // 직접 구현
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
2. 새 연결 등록과 이전 연결 종료의 순서
3. heartbeat가 애플리케이션 상태 정리에 연결되는 방식
4. 교체 close code를 일반 네트워크 종료와 구분하는 이유
5. heartbeat 주기와 오탐·복구 시간의 trade-off

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
- 긴 정지 뒤 모든 누락 tick을 처리하면 어떤 문제가 생깁니까?
- 남은 accumulator를 버리는 것과 유지하는 것의 trade-off는 무엇입니까?
- 여러 방마다 타이머를 둘 때와 공유 스케줄러를 둘 때 어떤 차이가 있습니까?

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
  constructor(
    readonly stepMs: number,
    readonly maxStepsPerAdvance: number,
    readonly maxAccumulatedMs: number
  ) {}

  advance(elapsedMs: number): StepResult {
    // 직접 구현
    throw new Error("not implemented");
  }

  reset(): void {
    // 직접 구현
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
2. catch-up 상한이 필요한 이유
3. 지연을 버릴 때 발생하는 시간 왜곡과 대안
4. 시뮬레이션 tick과 네트워크 snapshot cadence 분리
5. 방별 타이머와 공유 스케줄러의 리소스 trade-off

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
- soft limit과 hard limit을 따로 두면 어떤 정책을 만들 수 있습니까?
- 장시간 혼잡한 연결을 종료하지 않으면 서버 전체에 어떤 위험이 생깁니까?
- drop, delivery delay, congestion 종료를 어떤 메트릭으로 관측하겠습니까?

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
  constructor(
    socket: SnapshotSocket,
    options?: {
      now?: () => number;
      onDelivered?: (delayMs: number) => void;
      onDropped?: (reason: DropReason) => void;
    }
  ) {}

  enqueue(payload: string): void {
    // 직접 구현
  }

  close(reason?: DropReason): void {
    // 직접 구현
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
2. callback 완료와 transport buffer 혼잡을 분리한 이유
3. soft/hard threshold 및 혼잡 timeout의 역할
4. 연결 종료가 과격해 보여도 필요한 자원 보호 이유
5. drop·지연 메트릭을 이용한 threshold 튜닝 방법

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
- 여러 종료 신호가 연속으로 오면 종료 절차가 두 번 실행되지 않게 어떻게 합니까?
- 진행 중 경기의 DB 확정 재시도가 남아 있을 때 drain 완료를 어떻게 정의합니까?
- `stop_grace_period`가 애플리케이션 drain timeout보다 짧으면 무슨 일이 생깁니까?

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
  beginTask(): { release(): void } | null {
    // draining 중이면 null
    throw new Error("not implemented");
  }

  beginDrain(timeoutMs: number): Promise<DrainResult> {
    // 직접 구현
    throw new Error("not implemented");
  }

  close(): void {
    // 직접 구현
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
  // 직접 구현
  throw new Error("not implemented");
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
2. 대기 작업과 진행 중 작업을 다르게 처리한 이유
3. 결과 저장 재시도를 active work에 포함할지에 대한 판단
4. timeout 이후 강제 종료에서 감수하는 데이터 위험
5. 애플리케이션 timeout과 컨테이너 `stop_grace_period` 정렬

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
