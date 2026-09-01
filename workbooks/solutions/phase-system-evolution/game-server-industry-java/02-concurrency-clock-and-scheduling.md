# 동시성, 시계와 스케줄링

이 문서는 single-owner 상태 모델이 단일 Room에서 fixed clock을 거쳐 many-Room scheduler로 확장되는 흐름을 다룬다. 백지 구현은 원본 코드를 재현하는 문제가 아니라 invariant를 작은 인터페이스로 다시 세우는 문제다.

---

## [G03 / test: verify identity lifecycle and room ownership] Single-owner command gateway와 identity 경계

### 면접 질문

이 서버는 왜 `Room`의 모든 변경을 하나의 owner thread에서 직렬화하고 Netty callback은 mailbox에 command만 넣도록 했습니까? 모든 필드에 lock을 거는 방식과 비교해 얻는 장점과 잃는 점을 설명하십시오.

꼬리 질문:

- Connection, Session, Player, Room ID를 왜 분리해야 하며, packet에 들어온 `session_id`와 `player_id`를 어떤 실제 connection 문맥과 비교해야 합니까?
  - 모범답변: connection은 transport 수명, session은 인증된 논리 접속, player는 game 참가자, room은 state owner라 교체 시점이 다르다. packet claim은 source channel로 찾은 실제 `Peer -> Session -> membership/player`와 비교해야 한다.
- mailbox capacity 1,024가 가득 찼을 때 command를 조용히 버리는 것과 terminal 오류를 보내고 연결을 닫는 것의 차이는 무엇입니까?
  - 모범답변: 조용한 drop은 client가 요청 상태를 알 수 없고 재시도 중복을 만든다. 원본은 bounded admission 실패를 안정된 오류와 close 정책으로 드러내되, 그 오류 자체도 무한 queue에 넣지 않는다.
- network intent가 mailbox에 대기 중일 때 Room snapshot이 바뀌지 않아야 한다는 관측은 어떤 happens-before 경계를 보여 줍니까?
  - 모범답변: I/O callback의 enqueue 완료와 owner의 dequeue/apply 사이에 명시적 serialization boundary가 있다는 뜻이다. snapshot mutation은 owner drain 이후에만 관찰돼 callback thread가 Room을 직접 바꾸지 않았음을 증명한다.
- `Room.assertOwner()` 같은 runtime assertion이 테스트와 운영에서 주는 가치가 무엇입니까?
  - 모범답변: 잘못된 thread의 mutation을 race가 우연히 성공하기 전에 호출 지점에서 실패시킨다. 원본은 조회 일부에도 owner 검사를 둬 테스트가 confinement 계약을 지속적으로 확인하게 한다.
- shutdown 때 listener·channels·owner state·executors 순서를 잘못 잡으면 어떤 late callback과 leak이 생길 수 있습니까?
  - 모범답변: owner를 먼저 종료하면 channelInactive가 닫힌 executor에 command를 보내고, channel을 먼저 drain하지 않으면 parser/outbound buffer와 session registry가 남을 수 있다. 원본은 새 accept 차단, TCP/UDP channel close, I/O callback drain, owner cleanup, event-loop 종료 순서를 지킨다.

### 30초 모범 답변

게임 상태를 한 owner가 변경하면 복잡한 lock 조합 없이 command 순서와 tick 사이의 invariant를 한 곳에서 유지할 수 있습니다. Netty thread는 connection 문맥을 붙여 bounded mailbox에 넣고, owner가 실제 connection에 귀속된 session·player 권한을 확인한 뒤 상태를 바꿉니다. 대신 mailbox가 병목이 되므로 capacity와 overflow 정책, 한 번에 처리할 양, shutdown drain 경계를 명시해야 합니다. foreign-thread mutation을 즉시 실패시키면 우연히 동작하는 race를 테스트에서 숨기지 않을 수 있습니다.

### 답변 핵심 키워드

owner confinement, serialization, actor-style mailbox, bounded admission, identity authority, connection context, FIFO, happens-before, lifecycle, shutdown drain

### 백지 구현

#### 구현 목표

여러 I/O thread가 command를 제출하지만 실제 게임 상태는 owner thread만 변경하는 축소 gateway를 구현한다. connection에 실제로 연결된 session·player와 command가 주장하는 ID를 검증하고, bounded mailbox overflow가 상태를 바꾸지 않게 한다.

#### 면접용 인터페이스

```java
import java.util.Optional;

public record ConnectionId(String value) {}
public record SessionId(String value) {}
public record PlayerId(String value) {}

public record ConnectionContext(
        ConnectionId connectionId,
        SessionId sessionId,
        PlayerId playerId) {}

public record Command(
        ConnectionId sourceConnection,
        SessionId claimedSession,
        PlayerId claimedPlayer,
        String operation,
        String payload) {}

public enum SubmitResult {
    ENQUEUED,
    MAILBOX_FULL,
    CLOSED
}

public sealed interface ApplyResult {
    record Applied() implements ApplyResult {}
    record Rejected(String code) implements ApplyResult {}
}

public final class OwnerCommandGateway implements AutoCloseable {
    public OwnerCommandGateway(int capacity, Thread ownerThread) {
        if (capacity < 0) throw new IllegalArgumentException("negative capacity");
        this.capacity = capacity;
        this.ownerThread = java.util.Objects.requireNonNull(ownerThread);
    }

    public void register(ConnectionContext context) {
        assertOwner();
        java.util.Objects.requireNonNull(context);
        if (closed) throw new IllegalStateException("gateway closed");
        connections.put(context.connectionId(), context);
    }

    public SubmitResult submit(Command command) {
        java.util.Objects.requireNonNull(command);
        synchronized (monitor) {
            if (closed) return SubmitResult.CLOSED;
            if (mailbox.size() == capacity) return SubmitResult.MAILBOX_FULL;
            mailbox.addLast(command); // submit은 game state를 변경하지 않는다.
            return SubmitResult.ENQUEUED;
        }
    }

    public int drain(int maxCommands) {
        assertOwner();
        if (maxCommands < 0) throw new IllegalArgumentException("negative drain bound");
        int drained = 0;
        while (drained < maxCommands) {
            Command command;
            synchronized (monitor) {
                if (closed || mailbox.isEmpty()) break;
                command = mailbox.removeFirst();
            }
            ConnectionContext actual = connections.get(command.sourceConnection());
            ApplyResult result;
            if (actual == null || !actual.sessionId().equals(command.claimedSession()))
                result = new ApplyResult.Rejected("SESSION_INVALID");
            else if (!actual.playerId().equals(command.claimedPlayer()))
                result = new ApplyResult.Rejected("PLAYER_MISMATCH");
            else {
                appliedCommands++;
                result = new ApplyResult.Applied();
            }
            lastResults.put(command.sourceConnection(), result);
            drained++;
        }
        return drained;
    }

    public Optional<ApplyResult> lastResult(ConnectionId connectionId) {
        assertOwner();
        return Optional.ofNullable(lastResults.get(connectionId));
    }

    @Override
    public void close() {
        assertOwner();
        synchronized (monitor) {
            if (closed) return;
            closed = true;
            mailbox.clear();
        }
        connections.clear();
        lastResults.clear();
    }

    private final int capacity;
    private final Thread ownerThread;
    private final Object monitor = new Object();
    private final java.util.ArrayDeque<Command> mailbox = new java.util.ArrayDeque<>();
    private final java.util.Map<ConnectionId, ConnectionContext> connections = new java.util.HashMap<>();
    private final java.util.Map<ConnectionId, ApplyResult> lastResults = new java.util.HashMap<>();
    private long appliedCommands;
    private volatile boolean closed;

    private void assertOwner() {
        if (Thread.currentThread() != ownerThread)
            throw new IllegalStateException("mutation outside owner thread");
    }
}
```

#### 입력과 출력

- 입력: connection 등록, I/O thread의 command 제출, owner thread의 bounded drain.
- 출력: enqueue 여부와 owner가 기록한 apply/rejection 결과.
- 실제 게임 로직 대신 성공 시 단순 counter나 문자열 상태를 바꾸도록 축소해도 된다.

#### 반드시 만족해야 할 조건

- `submit`은 게임 상태를 직접 변경하지 않는다.
- `drain`과 state mutation은 생성 시 지정한 owner thread에서만 허용한다.
- source connection에 등록된 session·player가 claim과 모두 일치해야 apply한다.
- foreign session은 `SESSION_INVALID`, foreign player는 `PLAYER_MISMATCH`처럼 구분 가능한 결과를 낸다.
- mailbox가 가득 차면 새 command를 enqueue하지 않고 기존 queue와 상태를 보존한다.
- enqueue된 command는 최대 한 번, FIFO로 처리된다.
- unregister/close 뒤 늦게 도착한 command가 상태를 변경하지 않는다.
- `close` 뒤 queue와 registry를 비우고 추가 submit을 거절한다.

#### 경계 조건

- capacity만큼 command가 대기 중인 상태에서 한 개를 더 제출한다.
- valid command 뒤에 spoofed session command가 같은 connection에서 들어온다.
- command 제출 직후 connection이 닫히고 owner가 나중에 drain한다.
- owner가 아닌 thread가 `drain`을 호출한다.
- close가 queue에 command가 남은 상태에서 호출된다.

#### 실패 조건

- `submit` 시 상태가 먼저 바뀐 뒤 enqueue가 실패한다.
- claimed ID만 보고 connection ownership을 확인하지 않는다.
- overflow된 command가 나중에 유령처럼 실행된다.
- 같은 command가 두 번 apply된다.
- close 후 registry나 pending command가 남는다.

#### 필요한 제약

- queue 구현은 `ArrayBlockingQueue` 또는 명시적인 bounded 구조를 사용할 수 있다.
- `lastResult`는 면접용 관측 API다. 실제 transport reply 구현은 범위 밖이다.
- 처리량 최적화보다 ownership·identity·overflow invariant가 우선이다.

### 구현 후 자가 검증

- [ ] submit 전후의 게임 상태가 같고, owner drain 뒤에만 바뀐다.
- [ ] valid connection/session/player 조합만 apply된다.
- [ ] foreign session과 foreign player가 서로 다른 rejection으로 남는다.
- [ ] capacity+1번째 command가 queue·state를 바꾸지 않는다.
- [ ] enqueue된 N개 command가 FIFO로 정확히 N번 처리된다.
- [ ] foreign thread의 drain이 즉시 실패하고 상태가 보존된다.
- [ ] disconnect/close 뒤 늦은 command가 apply되지 않는다.
- [ ] close 후 pending count와 registry count가 0이다.
- [ ] submit은 O(1), drain은 실제 처리한 command 수에 선형이다.

### 구현 후 설명할 것

1. coarse lock 대신 single-owner serialization을 선택한 이유.
   - 모범답변: Room의 여러 field와 lifecycle을 한 순서로 바꿔 복합 lock 순서와 중간 상태 노출을 피한다. 대가로 owner mailbox가 병목이므로 capacity와 per-iteration quantum이 필요하다.
2. Connection·Session·Player를 별도 식별자로 둔 권한 모델.
   - 모범답변: 실제 source connection에서 session을 찾고 그 session의 player·room membership을 request claim과 대조한다. 각 수명을 분리해 reconnect 때 transport만 교체하고 stable player state를 보존할 수 있다.
3. mailbox full을 backpressure 또는 terminal failure로 표현한 기준.
   - 모범답변: 명령을 안전하게 재시도할 수 있고 transport가 흐름 제어를 제공하면 backpressure가 가능하지만, 이미 hard capacity에 도달해 응답 queue까지 위험하면 terminal close가 bounded하다. 어느 경우든 command state는 변경하지 않는다.
4. command enqueue와 owner apply 사이의 관측 가능한 경계.
   - 모범답변: `submit` 성공은 work 보관만 의미하고 Room snapshot은 그대로다. owner `drain`이 FIFO로 identity를 재검증하고 apply한 뒤에만 결과와 state가 바뀐다.
5. shutdown에서 새 입력 차단, channel close, owner cleanup, executor 종료 순서를 정한 이유.
   - 모범답변: 새 work 생성원을 먼저 없애고 channel close callback을 모두 발생·drain한 다음 owner가 session/Room을 정리해야 late mutation이 없다. 마지막에 executor를 종료해야 callback 유실과 resource leak을 피한다.

### 원본 확인 위치

- 대표 Thread: G03
- 커밋: `test: verify identity lifecycle and room ownership`
- 통합 Thread: G01 — `feat: establish single-room Netty arena baseline`
- 파일·컴포넌트: `src/main/java/arena/ArenaServer.java`, `Room.java`, `src/test/java/arena/IdentityScenario.java`
- 함수·테스트: `Room.assertOwner`, `RoomTest.frozenG03IdentityLifecycle`, `ArenaServer.enqueue` 경계
- 확인된 identity 오류: `ROOM_NOT_JOINABLE`, `SESSION_INVALID`, `PLAYER_MISMATCH`
- 관련 Thread: G05, G11, G13

---

## [G04 / feat: add bounded monotonic fixed clock] Fixed-step accumulator와 catch-up 상한

### 면접 질문

50ms마다 한 tick을 실행하려는 서버가 timer wake-up마다 무조건 한 tick만 실행하면 어떤 drift가 생깁니까? 반대로 밀린 tick을 한 번에 전부 실행하면 어떤 starvation이 생깁니까? accumulator와 한 iteration당 최대 4 tick이라는 상한으로 두 문제를 어떻게 절충합니까?

꼬리 질문:

- wall clock이 아니라 `System.nanoTime` 계열 monotonic source를 사용해야 하는 이유는 무엇입니까?
  - 모범답변: wall clock은 NTP나 관리자 조정으로 역행·도약할 수 있어 elapsed simulation time에 부적합하다. `nanoTime`은 절대 시각이 아니라 단조 elapsed 차이를 계산하는 용도다.
- Room이 RUNNING이 되기 전 idle 시간을 accumulator 시작점에서 제외해야 하는 이유는 무엇입니까?
  - 모범답변: lobby 대기 시간은 game simulation이 실행될 시간이 아니므로 시작과 동시에 대량 catch-up tick으로 바뀌어서는 안 된다. 원본은 Room이 RUNNING이 될 때 새 `FixedTickClock`을 만들고 그 시각을 anchor로 둔다.
- cap 이후 한 tick 이상이 더 남은 상태를 `OVERLOADED`로 표시하되 Room lifecycle을 종료하지 않은 이유는 무엇입니까?
  - 모범답변: 한 번의 wake 지연은 운영 backlog이지 game rule상 종료가 아니다. remaining full tick을 관측하고 G13의 지속 miss 정책처럼 별도 임계치에서만 failure로 승격한다.
- backlog를 버리는 방식과 보존하는 방식의 게임 체감·서버 보호 trade-off는 무엇입니까?
  - 모범답변: 버리면 server는 빨리 안정되지만 이동·cooldown·duration이 느려지는 time dilation이 생긴다. 보존하면 simulation 계약은 유지되지만 과부하가 지속될 수 있어 cap과 Room-local termination 같은 보호가 필요하다.
- manual test clock과 production timer가 같은 accumulator를 거쳐야 하는 이유는 무엇입니까?
  - 모범답변: 테스트만 다른 tick 계산을 쓰면 경계값을 검증해도 production drift를 잡지 못한다. 원본은 source만 주입하고 `poll(now, serviceReady)` 상태 전이는 동일하게 사용한다.

### 30초 모범 답변

고정 tick은 "timer가 몇 번 깼는가"가 아니라 monotonic elapsed time을 기준으로 해야 합니다. 이전 sample과의 차이를 accumulator에 더하고 50ms 단위로 tick을 실행하되 한 iteration에서는 최대 4개만 처리해 다른 Room·I/O 작업을 굶기지 않습니다. 남은 시간은 버리지 않고 다음 iteration으로 넘기며, cap 뒤에도 full tick이 남으면 operational overload로 관측합니다. 시작 전 idle 시간과 wall-clock 보정은 gameplay elapsed time에 섞지 않습니다.

### 답변 핵심 키워드

monotonic clock, fixed timestep, accumulator, drift, backlog, catch-up cap, starvation, overload metric, start boundary, deterministic test clock

### 백지 구현

#### 구현 목표

외부에서 주어진 monotonic nanosecond sample을 받아 fixed-duration tick을 실행하는 accumulator를 구현한다. backlog는 보존하되 한 번의 sample에서 실행할 tick 수를 제한한다.

#### 면접용 인터페이스

```java
public record ClockIteration(
        int executedTicks,
        long remainingNanos,
        boolean overloaded) {}

public final class FixedStepClock {
    public FixedStepClock(long tickNanos, int maxCatchUpTicks) {
        if (tickNanos <= 0 || maxCatchUpTicks <= 0)
            throw new IllegalArgumentException("clock bounds must be positive");
        this.tickNanos = tickNanos;
        this.maxCatchUpTicks = maxCatchUpTicks;
    }

    public void start(long nowNanos) {
        if (active) throw new IllegalStateException("clock already started");
        active = true;
        previousNanos = nowNanos;
        remainingNanos = 0;
        lastExecutedTicks = 0;
    }

    public ClockIteration sample(long nowNanos, Runnable tick) {
        if (!active) throw new IllegalStateException("clock is not active");
        java.util.Objects.requireNonNull(tick);
        long elapsed = nowNanos - previousNanos;
        if (elapsed < 0) throw new IllegalStateException("monotonic source moved backwards");
        previousNanos = nowNanos;
        remainingNanos = Math.addExact(remainingNanos, elapsed);

        int due = (int) Math.min(remainingNanos / tickNanos, maxCatchUpTicks);
        int executed = 0;
        while (executed < due) {
            tick.run();
            remainingNanos -= tickNanos; // 성공한 callback의 시간만 소비한다.
            executed++;
        }
        lastExecutedTicks = executed;
        return view();
    }

    public ClockIteration view() {
        return new ClockIteration(lastExecutedTicks, remainingNanos,
                active && remainingNanos >= tickNanos);
    }

    public void stop() {
        active = false;
        remainingNanos = 0;
        lastExecutedTicks = 0;
    }

    private final long tickNanos;
    private final int maxCatchUpTicks;
    private boolean active;
    private long previousNanos;
    private long remainingNanos;
    private int lastExecutedTicks;
}
```

#### 입력과 출력

- 입력: monotonic `nowNanos`, 한 tick의 실제 상태 변경 callback.
- 출력: 이번 iteration에서 실행한 tick 수, 남은 accumulator, overload 여부.
- 프로젝트 값으로 연습할 때 `tickNanos=50ms`, `maxCatchUpTicks=4`를 사용한다.

#### 반드시 만족해야 할 조건

- `start` 전 sample은 tick을 실행하지 않거나 명시적으로 거절한다.
- 첫 `start` sample은 기준점만 설정하고 이전 idle 시간을 누적하지 않는다.
- elapsed time 전체를 accumulator에 보존한다.
- 한 sample에서 callback 호출은 최대 `maxCatchUpTicks`회다.
- cap 처리 후 accumulator에 `tickNanos` 이상이 남으면 overloaded다.
- elapsed가 0이면 tick을 실행하지 않는다.
- `stop` 후 backlog를 정리하고 추가 sample 정책을 명확히 한다.
- wall-clock API를 내부에서 직접 읽지 않는다.

#### 경계 조건

- 정확히 49ms, 50ms, 51ms가 경과한다.
- 125ms 뒤 0ms sample이 이어진다.
- 225ms가 한 번에 들어와 cap 4를 채운다.
- cap 뒤 남은 25ms·50ms를 다음 sample에서 처리한다.
- `start`를 두 번 호출하거나 `stop` 뒤 sample한다.
- monotonic contract를 위반한 감소 sample이 들어올 때 정책을 설명한다.

#### 실패 조건

- cap을 적용하면서 초과 backlog를 버린다.
- callback이 cap보다 많이 호출된다.
- overload가 Room 종료나 tick 자체의 성공·실패와 섞인다.
- start 전 idle 시간이 gameplay tick으로 변환된다.
- manual test 경로와 production 경로가 서로 다른 tick 계산식을 쓴다.

#### 필요한 제약

- source가 정상적으로 단조 증가한다는 전제를 둘 수 있다. 감소 입력 정책은 예외·clamp 중 하나를 선택하고 이유를 설명한다.
- callback 내부 실패 정책은 과제 밖이지만, accumulator를 먼저 차감할지 성공 후 차감할지는 명시해야 한다.

### 구현 후 자가 검증

- [ ] 50, 50, 125, 0, 225, 50ms delta에서 tick 수가 `1, 1, 2, 0, 4, 2`다.
- [ ] 같은 입력에서 remaining이 `0, 0, 25, 25, 50, 0ms`다.
- [ ] 225ms sample에서 cap을 넘지 않고 overloaded가 true다.
- [ ] 다음 sample에서 남은 backlog를 이어서 처리한다.
- [ ] start 이전 큰 timestamp가 tick으로 변환되지 않는다.
- [ ] stop 후 remaining과 active 상태가 일관된다.
- [ ] callback 호출 횟수와 reported executedTicks가 항상 같다.
- [ ] sample 한 번의 시간 복잡도는 최대 catch-up cap에 제한된다.

### 구현 후 설명할 것

1. monotonic source와 wall clock의 역할 분리.
   - 모범답변: monotonic source는 tick 간 elapsed와 deadline을 계산하고 wall clock은 로그·사용자 표시 같은 달력 시각에만 쓴다. simulation 결과에는 wall-clock 조정이 들어오지 않는다.
2. backlog 보존과 overload 시 drop·slowdown 대안의 trade-off.
   - 모범답변: 원본은 backlog를 보존해 고정-step 의미를 지키고 cap으로 CPU를 제한한다. drop은 가용성은 높지만 simulation 시간을 잃고, slowdown은 시간 체감을 바꾸므로 별도 game 정책이 필요하다.
3. 한 iteration의 catch-up cap이 latency 격리에 주는 효과.
   - 모범답변: callback 횟수가 최대 4로 제한돼 한 Room의 큰 elapsed가 owner loop를 무한 점유하지 않는다. I/O와 다른 Room이 같은 scheduler cycle에서 기회를 얻는다.
4. clock operational state와 Room gameplay lifecycle을 분리한 이유.
   - 모범답변: `OVERLOADED`는 accumulator backlog 관측이고 `RUNNING/FINISHED/CLOSED`는 game 상태다. scheduler jitter가 즉시 game transition을 만들지 않게 두 state machine을 분리한다.
5. manual clock과 real timer가 같은 코어를 재사용하도록 한 테스트 설계.
   - 모범답변: production은 `System.nanoTime`, 테스트는 수동 supplier를 넣지만 둘 다 같은 accumulator 연산을 호출한다. 따라서 sleep 없이 정확한 delta sequence로 실제 경계를 재현한다.

### 원본 확인 위치

- Thread: G04
- 커밋: `feat: add bounded monotonic fixed clock`
- 파일·컴포넌트: `src/main/java/arena/FixedTickClock.java`, `ArenaServer.java`
- 함수·경계: `ArenaServer.runClockIteration`, `FixedTickClock.view`, `FixedTickClock.stop`
- 시나리오: `src/main/java/arena/ClockScenario.java`
- 관련 Thread: G01, G13

---

## [G13 / feat: isolate bounded many-room scheduling] Many-Room 공정성과 hot-room 격리

### 면접 질문

single-owner 서버가 여러 Room을 서비스할 때 Room 하나의 mailbox 폭주나 clock backlog가 다른 Room의 tick을 지연시키지 않게 하려면 scheduler iteration을 어떻게 구성해야 합니까? 이 프로젝트가 Room별 mailbox drain 64개, tick 4개, iteration당 공유 monotonic sample 하나를 둔 이유를 설명하십시오.

꼬리 질문:

- 각 Room마다 `now`를 따로 읽는 것보다 iteration 시작 시 한 번 읽어 공유하는 것이 공정성과 재현성에 어떤 영향을 줍니까?
  - 모범답변: 뒤에 순회한 Room만 더 늦은 시각을 받아 tick이 하나 늘어나는 order bias를 없앤다. 같은 입력 iteration은 같은 sample을 사용하므로 테스트 재현성도 높아진다.
- 특정 Room이 225ms 동안 서비스되지 않아 backlog가 생겨도 lifecycle을 RUNNING으로 유지하고 operational overload만 표시하는 이유는 무엇입니까?
  - 모범답변: 지연은 scheduler 상태이고 authoritative game 종료 조건이 아니다. backlog를 보존해 다음 quantum에서 회복시키고 지속 deadline miss가 한도에 도달할 때만 해당 Room을 닫는다.
- 20회 연속 deadline을 회복하지 못한 Room만 닫을 때 어떤 상태와 queue를 함께 제거해야 합니까?
  - 모범답변: Room registry와 scheduler clock, Room mailbox·pending input·replay, recoverable session credential, 그 Room의 active connection/outbound를 owner cleanup에서 정리한다. 다른 Room registry와 miss count는 유지한다.
- "첫 번째 Room"을 가리키는 역사적 field를 routing authority로 계속 쓰면 multi-room에서 어떤 버그가 생깁니까?
  - 모범답변: 두 번째 이후 Room의 command·snapshot·disconnect가 첫 Room state로 잘못 전달된다. 원본은 first-Room field를 호환 관측에만 두고 실제 session membership과 Room registry를 routing authority로 쓴다.
- Room 수·connection 수·pending input 수를 곱한 전역 상한을 왜 계산해야 합니까?
  - 모범답변: 각 객체의 local limit을 지켜도 최대 개수만큼 동시에 존재하면 합산 메모리가 커진다. `rooms × players × per-player pending`과 connection limit을 함께 계산해야 instance worst-case capacity가 명시된다.

### 30초 모범 답변

여러 Room을 한 owner가 돌리면 Room별 작업량을 quantum으로 잘라야 hot Room이 전체 iteration을 독점하지 않습니다. iteration마다 monotonic time을 한 번 읽고 각 `RoomRuntime`에 같은 sample을 주며, mailbox는 최대 64개, clock catch-up은 최대 4 tick만 처리합니다. backlog와 missed deadline은 Room별로 누적하고, 지속적으로 회복하지 못한 Room만 close·unregister해 다른 Room의 상태와 schedule을 보존합니다. 실제 session membership이 routing authority이고 첫 Room alias는 관측 용도로만 남겨야 합니다.

### 답변 핵심 키워드

fair quantum, per-Room runtime, bounded mailbox drain, bounded ticks, shared time sample, hot-room isolation, deadline streak, failure domain, registry routing, global capacity

### 백지 구현

#### 구현 목표

여러 Room runtime을 한 owner thread에서 공정하게 순회하는 scheduler를 구현한다. Room마다 bounded mailbox와 fixed-step clock이 있고, 지속적인 deadline miss는 해당 Room만 제거한다.

#### 면접용 인터페이스

```java
import java.util.List;
import java.util.Optional;

public interface RoomRuntime {
    String roomId();
    int drainCommands(int maxCommands);
    int advanceClock(long nowNanos, int maxTicks);
    boolean hasTickBacklog();
    boolean missedDeadline(long nowNanos);
    void close();
}

public record RoomService(
        String roomId,
        int commands,
        int ticks,
        boolean overloaded,
        boolean closed) {}

public final class ManyRoomScheduler implements AutoCloseable {
    public ManyRoomScheduler(
            int maxRooms,
            int maxCommandsPerRoom,
            int maxTicksPerRoom,
            int maxConsecutiveMisses) {
        if (maxRooms <= 0 || maxCommandsPerRoom <= 0 || maxTicksPerRoom <= 0
                || maxConsecutiveMisses <= 0)
            throw new IllegalArgumentException("scheduler bounds must be positive");
        this.maxRooms = maxRooms;
        this.maxCommandsPerRoom = maxCommandsPerRoom;
        this.maxTicksPerRoom = maxTicksPerRoom;
        this.maxConsecutiveMisses = maxConsecutiveMisses;
        this.owner = Thread.currentThread();
    }

    public void register(RoomRuntime room) {
        assertOwner();
        java.util.Objects.requireNonNull(room);
        if (closed) throw new IllegalStateException("scheduler closed");
        if (rooms.containsKey(room.roomId()))
            throw new IllegalArgumentException("duplicate room id");
        if (rooms.size() == maxRooms)
            throw new IllegalStateException("room capacity reached");
        rooms.put(room.roomId(), room);
        missed.put(room.roomId(), 0);
    }

    public List<RoomService> runIteration(long sharedNowNanos) {
        assertOwner();
        if (closed) return List.of();
        List<RoomService> services = new java.util.ArrayList<>(rooms.size());
        List<String> closeAfterIteration = new java.util.ArrayList<>();
        // LinkedHashMap snapshot은 stable insertion order와 deferred erase를 보장한다.
        for (RoomRuntime room : List.copyOf(rooms.values())) {
            int commands = room.drainCommands(maxCommandsPerRoom);
            int ticks = room.advanceClock(sharedNowNanos, maxTicksPerRoom);
            boolean overloaded = room.hasTickBacklog();
            int streak = room.missedDeadline(sharedNowNanos)
                    ? missed.get(room.roomId()) + 1 : 0;
            missed.put(room.roomId(), streak);
            boolean closeRoom = streak >= maxConsecutiveMisses;
            services.add(new RoomService(room.roomId(), commands, ticks,
                    overloaded, closeRoom));
            if (closeRoom) closeAfterIteration.add(room.roomId());
        }
        for (String roomId : closeAfterIteration) {
            RoomRuntime removed = rooms.remove(roomId);
            missed.remove(roomId);
            if (removed != null) removed.close();
        }
        return List.copyOf(services);
    }

    public Optional<RoomRuntime> room(String roomId) {
        assertOwner();
        return Optional.ofNullable(rooms.get(roomId));
    }

    @Override
    public void close() {
        assertOwner();
        if (closed) return;
        closed = true;
        rooms.values().forEach(RoomRuntime::close);
        rooms.clear();
        missed.clear();
    }

    private final int maxRooms;
    private final int maxCommandsPerRoom;
    private final int maxTicksPerRoom;
    private final int maxConsecutiveMisses;
    private final Thread owner;
    private final java.util.LinkedHashMap<String, RoomRuntime> rooms = new java.util.LinkedHashMap<>();
    private final java.util.Map<String, Integer> missed = new java.util.HashMap<>();
    private boolean closed;

    private void assertOwner() {
        if (Thread.currentThread() != owner)
            throw new IllegalStateException("scheduler access outside owner");
    }
}
```

#### 입력과 출력

- 입력: Room 등록, iteration마다 외부에서 한 번 읽은 `sharedNowNanos`.
- 출력: Room별 처리 command 수, tick 수, overload, close 여부.
- 프로젝트 값으로 연습할 때 command quantum 64, tick quantum 4, consecutive miss 한도 20을 사용한다.

#### 반드시 만족해야 할 조건

- 한 Room이 한 iteration에서 command·tick quantum을 넘지 않는다.
- 모든 active Room이 iteration마다 최대 한 번 서비스 기회를 얻는다.
- 같은 iteration의 Room들은 동일한 `sharedNowNanos`를 받는다.
- backlog가 남으면 overloaded로 관측하지만 즉시 close하지 않는다.
- deadline miss streak는 Room별로 독립적이며 회복 시 정책을 명시한다.
- 한도가 된 Room만 close하고 registry에서 제거한다.
- 제거된 Room의 mailbox·clock·pending resource 정리는 그 Room의 `close` 책임으로 완료된다.
- 다른 Room의 state·miss streak·service order는 실패 Room 때문에 초기화되지 않는다.
- 등록 한도를 넘는 Room은 기존 registry를 바꾸지 않고 거절한다.

#### 경계 조건

- Room 1개, 최대 Room 수, 0개인 iteration.
- 첫 Room만 매 iteration backlog가 있고 나머지는 가볍다.
- hot Room이 quantum을 항상 소진한다.
- miss streak가 한도 직전, 한도 도달, 중간 회복 상태다.
- iteration 도중 Room이 close 대상이 된다.
- scheduler 전체 close와 Room별 close가 반복 호출된다.

#### 실패 조건

- hot Room의 while-loop가 backlog를 모두 비우느라 다음 Room을 굶긴다.
- Room마다 다른 시각 sample을 읽어 순회 순서가 simulation 결과에 영향을 준다.
- 한 Room 실패가 전체 registry나 shared owner를 종료한다.
- 제거된 Room으로 session routing이 계속 가능하다.
- close 대상 Room의 pending command나 clock backlog가 남는다.

#### 필요한 제약

- scheduler mutation은 single owner thread에서만 실행된다고 가정한다.
- Room 내부 command 의미와 clock 수학은 제공된 `RoomRuntime`의 책임이다.
- iteration 순서는 insertion order, stable ID order 등 하나를 선택하고 결정성을 설명한다.

### 구현 후 자가 검증

- [ ] 모든 active Room이 한 iteration에서 한 번씩 관찰된다.
- [ ] hot Room command·tick 수가 각각 quantum을 넘지 않는다.
- [ ] normal Room의 tick 진척이 hot Room backlog 유무와 무관하게 유지된다.
- [ ] 모든 RoomRuntime이 같은 `sharedNowNanos`를 받는다.
- [ ] miss streak가 Room별로 독립이며 회복 정책대로 갱신된다.
- [ ] 20회 연속 미회복 Room만 close·remove된다.
- [ ] 제거 후 lookup·routing에서 해당 Room이 보이지 않는다.
- [ ] scheduler close 후 모든 Room의 close가 정확히 한 번 효과를 낸다.
- [ ] iteration 시간은 Room 수 × 각 quantum으로 상한이 잡힌다.
- [ ] 전역 pending 상한을 Room·connection·per-player bound의 곱으로 설명할 수 있다.

### 구현 후 설명할 것

1. Room별 quantum을 command와 tick에 각각 둔 이유.
   - 모범답변: mailbox 폭주와 clock backlog는 서로 독립적인 work source라 하나만 제한하면 다른 쪽이 owner를 독점할 수 있다. 원본은 command 64와 tick 4를 별도로 제한한다.
2. iteration당 단일 monotonic sample이 공정성·결정성에 주는 효과.
   - 모범답변: 모든 Room이 동일 elapsed boundary를 보고 순회 위치에 따른 시간 차이를 없앤다. Room별 accumulator는 유지하므로 shared observation과 independent simulation state를 동시에 얻는다.
3. 일시 backlog와 지속 실패를 다른 상태로 처리한 기준.
   - 모범답변: backlog는 다음 iteration에 회복될 수 있어 operational overload로만 기록한다. 실제 deadline이 계속 누적되고 20회 동안 due work가 회복되지 않을 때 terminal failure로 본다.
4. failure domain을 Room 하나로 제한하기 위해 제거한 registry·queue·clock 자원.
   - 모범답변: 해당 Room의 registry membership, runtime mailbox, clock, pending input, replay, session credential와 connection만 close한다. shared owner와 다른 Room context는 계속 서비스한다.
5. session membership을 routing authority로 삼고 역사적 alias를 배제한 이유.
   - 모범답변: membership은 현재 session이 실제로 속한 Room을 나타내며 reconnect·close 때 함께 갱신된다. 첫 Room alias는 다중 Room에서 유일하지 않아 command routing 근거가 될 수 없다.

### 원본 확인 위치

- Thread: G13
- 커밋: `feat: isolate bounded many-room scheduling`
- 파일·컴포넌트: `src/main/java/arena/RoomRuntime.java`, `ArenaServer`의 Room registry·scheduler 경계
- 테스트·시나리오: `src/test/java/arena/ManyRoomScenario.java`, `src/test/resources/G13.json`
- 확인된 bound: Room 최대 64, connection 최대 512, Room별 mailbox drain 64, tick quantum 4, 지속 deadline 20회
- 관련 Thread: G03, G04, G12, G14
