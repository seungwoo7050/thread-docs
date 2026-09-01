# 시간·입력 승인·결정성 면접 워크북

이 문서는 G04~G07에서 확인되는 고정 틱 시간 모델, 입력 identity와 tick window, abuse bound, 결정적 replay와 state hash를 묶는다. G05와 G06은 같은 입력 승인 파이프라인을 확장하므로 하나의 대표 문제로 통합했다.

---

<a id="w02-g04"></a>

## [G04 / `feat(time): bound fixed ticks with monotonic accumulation`] 고정 틱 누적기와 bounded catch-up

### 면접 질문

50ms 고정 틱 서버에서 wake-up 간격이 50ms, 125ms, 225ms처럼 불규칙할 때 실행할 틱 수를 어떻게 계산했습니까? 한 iteration에서 catch-up을 4틱으로 제한하면서 남은 시간을 버리지 않은 이유는 무엇입니까?

꼬리 질문:

- wall clock 대신 monotonic clock을 사용해야 하는 이유는 무엇입니까?
- `sleep_for(50ms)`를 반복하는 방식이 drift를 만드는 이유는 무엇입니까?
- catch-up cap을 초과한 backlog를 버리면 어떤 gameplay 문제가 생깁니까?
- backlog를 무제한 처리하면 I/O와 다른 Room에 어떤 영향을 줍니까?
- monotonic 입력이 이전 값보다 작아졌을 때 clamp와 fail-fast 중 무엇을 선택했습니까?
- "OVERLOADED"와 `Room`의 gameplay lifecycle을 왜 별도 상태로 보아야 합니까?

### 30초 모범 답변

매 iteration에서 monotonic 시간 차이를 정수 밀리초 accumulator에 더하고, `remaining / 50ms`만큼 due tick을 계산하되 최대 4개만 실행합니다. 실행한 시간만 빼고 나머지는 보존하므로 지연이 사라지지 않고 다음 iteration에서 계속 따라잡습니다. cap은 한 Room의 catch-up이 I/O나 다른 작업을 독점하지 못하게 하고, 한 틱 이상 backlog가 남으면 운영 상태만 OVERLOADED로 표시합니다. wall clock 조정은 시뮬레이션에 영향을 주지 않으며, monotonic 역행은 계약 위반으로 즉시 드러냅니다.

### 답변 핵심 키워드

steady clock, integer accumulator, elapsed time, drift 방지, catch-up cap, backlog 보존, starvation 방지, operational overload

### 백지 구현

#### 구현 목표

주어진 monotonic timestamp를 받아 이번 호출에서 실행할 고정 틱 수, 남은 시간, overload 여부를 계산하는 누적기를 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct TickBatch {
    int ticks;
    std::int64_t remaining_ms;
    bool overloaded;
};

class FixedTickAccumulator {
public:
    FixedTickAccumulator(std::int64_t tick_ms, int max_catch_up_ticks);

    void reset(std::int64_t monotonic_ms);
    TickBatch advance(std::int64_t monotonic_ms);
    std::int64_t remaining_ms() const noexcept;

private:
    // 직접 구현
};
```

#### 입력과 출력

- 입력: 절대 monotonic millisecond 값
- 출력: 이번 iteration의 실행 틱 수, 실행 후 누적 잔여 시간, 여전히 한 틱 이상 밀렸는지 여부

#### 반드시 만족해야 할 조건

- 시간 계산은 정수 연산만 사용한다.
- 현재 timestamp가 이전보다 작으면 계약 위반으로 처리한다.
- 실행 틱 수는 `0..max_catch_up_ticks` 범위다.
- 실행되지 않은 elapsed time은 버리지 않는다.
- `overloaded`는 실행 후 잔여 시간이 적어도 한 틱 이상일 때만 참이다.
- `reset`은 이전 anchor와 accumulator를 명확히 초기화한다.
- wall clock 또는 실제 sleep은 사용하지 않는다.

#### 경계 조건

- elapsed 0, tick-1, tick, tick+1
- 정확히 cap개 틱 분량
- cap개를 초과하는 큰 elapsed
- 여러 호출에 걸쳐 fractional remainder가 합쳐져 한 틱이 되는 경우
- reset 직후 advance
- timestamp 역행
- `tick_ms <= 0`, `max_catch_up_ticks <= 0` 생성 인자
- 큰 timestamp 차이에서 정수 overflow 가능성

#### 실패 조건

- cap 초과분을 잃으면 실패다.
- 한 번의 호출에서 cap보다 많은 틱을 반환하면 실패다.
- remainder가 음수가 되면 실패다.
- wall clock 변화가 결과에 들어오면 요구사항 위반이다.

#### 필요한 제약

- 10~15분 범위로 제한한다.
- 스케줄러 loop, sleep, 실제 게임 tick 호출은 범위에서 제외한다.
- 입력 timestamp와 누적 합의 overflow 정책을 명시해야 한다.

### 구현 후 자가 검증

- 정상 경로: `[50, 50, 125, 0, 225, 50]` 형태의 delta에서 누적 결과 확인
- 경계값: 정확히 4틱과 5틱 분량의 elapsed 비교
- 실패 경로: 역행 timestamp와 잘못된 생성 인자 확인
- 상태 변화: cap으로 남은 backlog가 다음 호출에서 실행되는지 확인
- invariant: `총 유입 시간 = 실행한 총 틱 시간 + 현재 remainder`
- 시간 복잡도: timestamp 크기와 무관하게 호출당 O(1)인지 확인
- 요구사항 충족: overload가 gameplay 상태를 직접 바꾸지 않는지 설명 가능한지 확인

### 구현 후 설명할 것

1. sleep 기반 loop보다 accumulator 방식이 drift에 강한 이유
2. catch-up cap과 backlog 보존을 동시에 선택한 이유
3. 역행 monotonic clock을 조용히 보정하지 않은 이유
4. manual clock 주입이 테스트 결정성을 높이는 방식
5. G13처럼 Room이 여러 개일 때 이 누적기를 Room별로 가져야 하는 이유

### 원본 확인 위치

- Thread: G04
- 커밋 메시지: `feat(time): bound fixed ticks with monotonic accumulation`
- 파일: `src/game.hpp`, `src/game.cpp`, `src/transport.cpp`, `src/lifecycle_scenario.cpp`, `server.example.json`
- 함수·클래스·컴포넌트: `TickBatch`, `FixedTickAccumulator`, `FixedTickAccumulator::reset`, `FixedTickAccumulator::advance`, `monotonic_milliseconds`, `Server::run_scheduler`, `Server::advance_one_tick`
- 관련 Thread: G03의 single owner, G13의 Room별 scheduler와 hot-room 격리

---

<a id="w02-g05-g06"></a>

## [G05 / 기록 제목: Sequence identity and tick-window admission + G06 / 기록 제목: Authoritative intent and four-attempt bound] 입력 승인 상태기계

### 면접 질문

입력에 `seq`와 `target_tick`을 함께 둔 이유는 무엇이며, duplicate·conflict·stale·late·too-early를 어떤 순서와 invariant로 판정했습니까? G06에서 malformed 입력까지 attempt limit에 포함하려고 검증 위치를 owner admission으로 옮긴 이유는 무엇입니까?

꼬리 질문:

- 같은 sequence와 같은 payload 재전송은 왜 duplicate로 성공 취급할 수 있고, 다른 payload는 왜 conflict입니까?
- rejected sequence를 예약해 버리면 클라이언트 재시도에 어떤 문제가 생깁니까?
- target tick을 부동소수점으로 읽거나 음수를 0으로 clamp하면 어떤 우회가 가능합니까?
- `next_tick..next_tick+4` 비교에서 unsigned overflow를 어떻게 피합니까?
- 같은 tick에 여러 accepted 입력이 있다면 어느 입력을 적용해야 합니까?
- rate limit을 parser 단계에서 먼저 적용하지 않고 authenticated owner 단계에서 적용한 이유는 무엇입니까?
- malformed 4회 뒤 5번째 유효 요청은 decoder까지 도달해야 합니까?
- 실제 simulation tick이 없는데 attempt count를 wall time으로 reset하면 어떤 문제가 생깁니까?

### 30초 모범 답변

`seq`는 재전송 identity이고 `target_tick`은 적용 시점 계약입니다. 마지막 accepted payload와 sequence를 기준으로 동일 재전송은 duplicate, 같은 sequence의 다른 의미는 conflict, 더 낮은 sequence는 stale로 분류합니다. target은 정확한 정수로 다음 tick부터 네 tick 앞까지만 허용하고, accepted 항목만 sequence와 pending 상태를 갱신합니다. G06의 attempt limit은 인증된 player에 귀속된 뒤 decoder 전에 증가시켜 malformed 요청도 비용을 내게 하며, 다섯 번째는 decode·sequence 예약·queue 삽입 없이 거부합니다. count는 실제 게임 tick에서만 reset해 시간 모델과 abuse 정책을 일치시킵니다.

### 답변 핵심 키워드

idempotency key, duplicate vs conflict, accepted-only mutation, exact integer, tick window, bounded pending, highest due sequence, authenticated admission, malformed counts, tick-scoped rate limit

### 백지 구현

#### 구현 목표

한 player의 입력 승인 상태를 관리한다. sequence 중복·충돌, target tick window, pending 상한, tick당 attempt 상한을 하나의 일관된 순서로 처리한다.

#### 인터페이스 또는 함수 시그니처

```cpp
using ExactInteger = std::variant<std::int64_t, std::uint64_t>;

enum class Direction { Stop, North, East, South, West };

struct RawIntent {
    ExactInteger seq;
    ExactInteger target_tick;
    std::string direction;
    std::optional<std::string> tag_target_player_id;
};

struct Intent {
    std::uint64_t seq;
    std::uint64_t target_tick;
    Direction direction;
    std::optional<std::string> tag_target_player_id;
};

enum class AdmitCode {
    Accepted,
    Duplicate,
    MessageInvalid,
    InputStale,
    SequenceConflict,
    InputLate,
    InputTooEarly,
    InputQueueFull,
    InputRateExceeded
};

struct AdmitResult {
    AdmitCode code;
    std::optional<std::uint64_t> accepted_seq;
};

class PlayerInputState {
public:
    AdmitResult admit(const RawIntent& raw, std::uint64_t next_tick);
    std::optional<Intent> take_for_tick(std::uint64_t tick);
    void begin_next_simulation_tick();

private:
    // 직접 구현
};
```

#### 입력과 출력

- 입력: 인증된 player에 귀속된 raw intent와 다음 simulation tick
- 출력: 안정된 admission code, accepted sequence 정보, tick 시 적용할 intent

#### 반드시 만족해야 할 조건

- tick당 첫 네 번의 시도는 malformed 여부와 관계없이 count된다.
- 다섯 번째 시도는 full decode 전에 `InputRateExceeded`를 반환한다.
- rate 거부는 sequence, last accepted payload, pending queue를 바꾸지 않는다.
- signed 음수와 `uint64_t` 범위 밖 표현을 조용히 clamp하거나 float로 변환하지 않는다.
- duplicate 판단은 알려진 typed logical field만 비교하며, 무시하기로 한 unknown field는 identity에 포함하지 않는다.
- 동일 accepted sequence·동일 payload는 duplicate이며 새 pending 항목을 만들지 않는다.
- 동일 accepted sequence·다른 payload는 conflict다.
- accepted sequence보다 낮은 sequence는 stale다.
- target tick은 `next_tick`부터 최대 네 tick 앞까지만 허용한다.
- target window 비교는 unsigned overflow 없이 수행한다.
- accepted pending은 최대 64개다.
- 실패한 요청은 sequence를 예약하지 않아 수정 후 같은 sequence를 재사용할 수 있다.
- 같은 tick에 due인 accepted 입력이 여러 개면 가장 높은 sequence 하나만 적용한다.
- 미래 tick 입력은 그대로 남고, 과거 due 항목을 무한히 보존하지 않는다.
- 실제 simulation tick 경계에서 attempt count를 reset한다.

#### 경계 조건

- sequence 0, 1, `uint64_t` 최대값
- target이 `next-1`, `next`, `next+4`, `next+5`
- `next_tick`이 `uint64_t` 최대값에 가까운 경우
- 동일 sequence의 unknown field만 다른 재전송
- duplicate가 pending limit 64에서 들어오는 경우
- 64번째 accepted와 65번째 새로운 accepted
- malformed 4회 후 유효 5번째, tick reset 후 동일 유효 요청 재시도
- 같은 target tick에 sequence가 증가하는 여러 입력
- future 입력과 current 입력이 섞인 상태

#### 실패 조건

- malformed 또는 rate-limited 요청이 last accepted sequence를 올리면 실패다.
- duplicate가 pending 항목 수를 증가시키면 실패다.
- conflict 요청이 기존 accepted payload를 덮어쓰면 실패다.
- 다섯 번째 요청이 decoder 또는 queue allocation을 수행하면 실패다.
- wall clock만 흘렀다고 attempt count가 reset되면 실패다.

#### 필요한 제약

- 25~30분 범위다.
- player membership, TAG 규칙, 이동 계산은 범위에서 제외한다.
- JSON 라이브러리 대신 `RawIntent`가 제공되지만 exact signed/unsigned 분류는 구현해야 한다.
- pending 자료구조와 목표 복잡도를 설명해야 한다.

### 구현 후 자가 검증

- 정상 경로: 서로 다른 future tick 입력이 승인되고 각 tick에 맞게 남고 적용되는지 확인
- 경계값: window의 양 끝, pending 64/65, attempt 4/5 확인
- 실패 경로: malformed, stale, conflict, late, too early 후 상태가 그대로인지 확인
- 상태 변화: accepted → duplicate → conflict 순서에서 last accepted와 pending 확인
- invariant: pending의 모든 항목은 accepted된 고유 sequence이며 상한 이내인지 확인
- 중복·누락 처리: 같은 tick의 낮은 sequence가 적용되거나 future 입력이 누락되지 않는지 확인
- 보안: unknown field로 duplicate identity를 우회하거나 음수 clamp로 window를 우회할 수 없는지 확인
- 시간·공간 복잡도: admission과 tick 선택의 비용, pending 64 상한이 주는 의미 설명
- 요구사항 충족: rejected sequence가 수정 후 재사용 가능한지 확인

### 구현 후 설명할 것

1. 검증 순서가 observable error code와 상태 mutation에 미치는 영향
2. duplicate를 성공적인 idempotent 재전송으로 보는 이유
3. attempt counting을 decoder 전·인증 후에 둔 이유
4. tick window와 pending bound가 서버 메모리·예측 가능성을 지키는 방식
5. 같은 tick에서 highest accepted sequence를 선택한 결정성 trade-off

### 원본 확인 위치

- 대표 Thread: G05
- 통합 Thread: G06
- G05 기록 제목: Sequence identity and tick-window admission (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- G06 기록 제목: Authoritative intent and four-attempt bound (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- 파일: `src/game.hpp`, `src/game.cpp`, `src/transport.hpp`, `src/transport.cpp`, `src/lifecycle_scenario.cpp`
- 함수·클래스·컴포넌트: `Input`, `InputResult`, `Player`, `Room::input`, `Room::tick`, `decode_input`, `admit_input`, `Room::begin_input_attempt`, `Server::drain_mailbox`
- 관련 Thread: G02의 strict decoding, G04의 실제 tick 경계, G07의 accepted-event 기록, G13의 instance-wide pending bound

---

<a id="w02-g07"></a>

## [G07 / 기록 제목: Accepted-input replay and exact state hash] 결정적 canonical state와 첫 divergence 탐지

### 면접 질문

같은 accepted input journal을 여러 프로세스에서 replay했을 때 동일한 hash를 만들려면 canonical state에 무엇을 포함하고 무엇을 제외해야 합니까? hash가 다를 때 첫 divergent tick을 어떻게 진단 가능하게 만들었습니까?

꼬리 질문:

- JSON 객체를 그대로 dump해서 hash하면 왜 위험합니까?
- player iteration 순서, signed/unsigned 표기, delimiter, final newline까지 계약에 넣은 이유는 무엇입니까?
- connection ID, session credential, input attempt count, pending effect를 canonical state에서 제외한 이유는 무엇입니까?
- accepted되었지만 같은 tick의 더 높은 sequence에 밀린 입력도 journal에 남겨야 하는 이유는 무엇입니까?
- rejected 입력을 replay artifact에서 제거해도 결과가 같아야 한다는 검증은 무엇을 보여 줍니까?
- capture가 4MiB를 넘었을 때 gameplay를 중단하지 않고 export만 거부한 이유는 무엇입니까?
- expected hash와 actual hash만 보여 주는 것보다 canonical record를 함께 남기는 이점은 무엇입니까?

### 30초 모범 답변

결정성 검증은 의미가 같은 상태가 항상 같은 바이트열이 되는 canonicalization부터 필요합니다. 그래서 player를 명시적 순서로 정렬하고 정수 표기, 필드 순서, delimiter와 마지막 개행까지 고정하며, 연결·credential처럼 simulation 결과가 아닌 값은 제외합니다. journal에는 accepted event를 원래 admission boundary에 기록하고 tick마다 production 규칙을 다시 실행한 뒤 hash를 비교합니다. 첫 불일치에서 tick, expected·actual hash와 actual canonical record를 반환하면 원인을 좁힐 수 있습니다. capture 한도를 넘으면 기록만 incomplete로 latch하고 export를 거부하되 authoritative simulation과 hash 계산은 계속합니다.

### 답변 핵심 키워드

canonical bytes, deterministic ordering, semantic state only, admission boundary, accepted-event journal, per-tick hash, first divergence, bounded capture, fail-open gameplay/fail-closed export

### 백지 구현

#### 구현 목표

정렬된 논리 상태를 안정된 바이트 문자열로 직렬화하고, 제공된 deterministic step·hash 콜백으로 replay의 첫 divergent tick을 찾는다. SHA-256 자체는 구현하지 않는다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct CanonicalPlayer {
    std::string player_id;
    int slot;
    int x;
    int y;
    Direction direction;
    int score;
    std::string connectivity;
    std::uint64_t last_seq;
    std::int64_t last_tag_tick;
};

struct CanonicalRoom {
    std::string room_id;
    std::int64_t tick;
    std::string status;
    std::uint64_t owner_epoch;
    std::vector<CanonicalPlayer> players;
};

std::string canonical_state(const CanonicalRoom& room);

struct ReplayTick {
    std::uint64_t tick;
    std::vector<Event> events;
    std::string expected_hash;
};

struct Divergence {
    std::uint64_t first_divergent_tick;
    std::string expected_hash;
    std::string actual_hash;
    std::string actual_canonical_record;
};

std::optional<Divergence> first_divergence(
    SimulationState initial,
    std::span<const ReplayTick> ticks,
    const std::function<void(SimulationState&, const Event&)>& apply_event,
    const std::function<void(SimulationState&)>& execute_tick,
    const std::function<CanonicalRoom(const SimulationState&)>& project,
    const std::function<std::string(std::string_view)>& hash);
```

#### 입력과 출력

- 입력: 초기 simulation state, 순서가 고정된 tick journal, event·tick·projection·hash 콜백
- 출력: 모든 tick이 일치하면 없음, 다르면 첫 divergence의 진단 정보

#### 반드시 만족해야 할 조건

- canonical player는 `player_id` 기준으로 명시적으로 정렬한다.
- 필드 순서, 구분자, 줄 순서, 정수 표기, 마지막 LF가 입력 순서와 무관하게 고정된다.
- credential, connection descriptor, process ID, wall clock 같은 비결정적 값은 canonical 구조에 없다.
- 각 replay record의 tick 번호가 현재 실행 tick과 정확히 맞아야 한다.
- event의 admission boundary가 해당 tick보다 앞뒤로 이동하지 않는다.
- 한 tick의 event를 모두 적용한 뒤 정확히 한 simulation tick을 실행하고 hash한다.
- 첫 mismatch에서 즉시 멈추고 actual canonical bytes를 반환한다.
- 콜백이 실패하거나 journal 구조가 잘못되면 hash mismatch와 구분되는 명시적 오류 정책을 둔다.

#### 경계 조건

- player 0명과 1명
- 입력 vector 순서가 뒤섞여도 같은 canonical bytes가 나오는 경우
- `uint64_t` 최대 sequence와 음수 cooldown tick
- 빈 문자열, delimiter 문자 포함 식별자의 허용 여부
- final newline이 있거나 없는 문자열 비교
- tick 0에서 divergence, 마지막 tick에서 divergence
- accepted event가 없는 tick
- 동일 tick에 여러 event
- journal tick이 건너뛰거나 중복되는 경우

#### 실패 조건

- 컨테이너 iteration 순서에 따라 hash가 달라지면 실패다.
- mismatch 후 계속 실행해 "마지막 divergence"를 반환하면 실패다.
- malformed journal을 단순 hash mismatch로 보고하면 실패다.
- canonical bytes에 credential이나 process-local 값이 들어가면 요구사항 위반이다.

#### 필요한 제약

- 25~30분 범위다.
- simulation 규칙과 cryptographic hash는 콜백으로 제공된다.
- canonical text 포맷은 문제에 명시된 필드만 사용한다.
- replay capture의 4MiB 저장 정책은 구현 후 설명 문제로 다룬다.

### 구현 후 자가 검증

- 정상 경로: 같은 journal을 두 번 실행해 모든 hash가 일치하는지 확인
- 경계값: 최대 unsigned, 음수 signed, final LF 차이가 hash에 반영되는지 확인
- 실패 경로: tick 중복·누락과 malformed event가 divergence와 별도 오류인지 확인
- 상태 변화: event 적용 전후와 tick 실행 전후의 boundary가 섞이지 않는지 확인
- invariant: canonical bytes 생성은 입력 vector 순서와 독립적인지 확인
- 중복·누락 처리: accepted event가 journal에서 빠지면 예상 tick에서 처음 divergence가 나는지 확인
- 보안: credential과 connection 정보가 projection에 들어오지 않는지 확인
- 시간·공간 복잡도: player 정렬 O(P log P), tick 검증 O(T·step cost) 설명
- 요구사항 충족: 첫 mismatch의 actual canonical record를 재현 가능한 형태로 제공하는지 확인

### 구현 후 설명할 것

1. canonicalization을 hash 함수보다 먼저 설계해야 하는 이유
2. 어떤 상태를 "authoritative simulation 의미"로 포함·제외했는지
3. event 기록 시점을 admission boundary로 고정한 이유
4. 첫 divergence에서 중단하는 진단상의 이점
5. replay capture overflow가 gameplay를 중단하지 않되 incomplete export는 거부하는 trade-off

### 원본 확인 위치

- Thread: G07
- 기록 제목: Accepted-input replay and exact state hash (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- 파일: `src/replay.hpp`, `src/replay.cpp`, `src/transport.hpp`, `src/transport.cpp`, `tests/g07.hpp`, `tests/g07.cpp`, `tests/scenario_main.cpp`
- 함수·클래스·컴포넌트: `ReplayLog`, `ReplayLog::reserve`, `ReplayLog::append`, `ReplayLog::accepted_input`, `ReplayLog::finish_tick`, `canonical_state`, `sha256`, `parse_replay_artifact`, `verify_replay`, `ReplayRun`
- 관련 Thread: G05·G06의 accepted-only admission, G08·G10의 snapshot state hash, G11의 reconnect lifecycle record, G13의 Room별 replay journal
