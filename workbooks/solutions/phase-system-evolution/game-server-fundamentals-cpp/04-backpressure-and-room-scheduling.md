# 역압·bounded outbound·다중 Room 스케줄링 면접 워크북

이 문서는 G12와 G13에서 확인되는 느린 소비자 격리, 이동 전용 전송 버퍼 수명, snapshot coalescing, Room별 고정 틱 스케줄링과 hot-room 실패 격리를 묶는다.

---

<a id="w04-g12"></a>

## [G12 / `feat(cpp): bound and coalesce outbound snapshots`] 느린 소비자와 bounded outbound work

### 면접 질문

한 UDP 소비자가 오랫동안 전송 가능 상태가 아니어도 simulation과 다른 클라이언트가 계속 진행되게 하면서, 해당 연결의 메모리를 어떻게 상한 안에 유지했습니까? control message와 FULL·DELTA snapshot을 같은 FIFO로 무한 누적하지 않은 이유는 무엇입니까?

꼬리 질문:

- control은 순서와 개별 의미가 중요하지만 snapshot은 최신 상태가 더 중요한 이유는 무엇입니까?
  - 모범답변: ACK·오류·lifecycle control은 각각의 사건과 순서가 의미를 가지므로 FIFO가 필요하다. snapshot은 같은 stream에서 더 새 FULL/DELTA가 이전 queued 상태를 대체할 수 있어 최신 bounded slot을 유지하는 편이 적합하다.
- 연결당 pending FULL 1개·DELTA 1개로 제한했을 때 superseded buffer 수명은 어떻게 끝나야 합니까?
  - 모범답변: 아직 queue가 소유한 old slot을 `reset`해 즉시 소멸시키고 memory counter를 한 번 감소시킨 뒤 새 move-only buffer를 넣는다. 원본도 FULL은 queued FULL·DELTA를, DELTA는 queued DELTA를 supersede한다.
- 비차단 `send`가 partial write 또는 `EAGAIN`을 반환하면 unsent suffix를 누가 소유합니까?
  - 모범답변: `PendingWrite`가 vector와 offset을 계속 소유한다. partial이면 offset만 전진하고 EAGAIN이면 그대로 보존해 readiness가 돌아왔을 때 `remaining()` suffix만 다시 보낸다.
- move-only `PendingWrite`가 별도 `OutboundMemory` 계측과 결합될 때 이동 생성·대입에서 어떤 invariant가 필요합니까?
  - 모범답변: accounting 책임 포인터도 buffer와 함께 정확히 한 객체로 이동하고 원본은 null이 돼야 한다. 이동 대입 대상이 기존 buffer를 가졌다면 먼저 release해 항상 `created = released + live_buffers`를 유지한다.
- readiness event가 꺼진 동안 snapshot을 drop했다고 기록하는 것과 coalesce하는 것은 왜 다릅니까?
  - 모범답변: drop은 새 state가 보존되지 않은 것이고, coalesce는 queue가 소유한 이전 대기본을 더 최신본으로 교체해 다음 writable 시 보낼 work가 남아 있다. 원본은 이를 `snapshots_coalesced`와 slot state로 구분한다.
- TCP close가 도착했을 때 transport buffer 해제와 owner mailbox의 player disconnect mutation 중 어느 것이 먼저 관찰될 수 있습니까?
  - 모범답변: connection erase로 transport-owned queue와 buffer가 먼저 해제되고, disconnect notification은 owner drain에서 player를 DISCONNECTED로 바꿀 수 있다. 두 수명을 분리하므로 중간에 buffer는 0이지만 game state는 아직 이전 상태일 수 있다.
- control queue가 포화됐을 때 계속 오류 메시지를 추가하려고 하면 어떤 feedback loop가 생깁니까?
  - 모범답변: 포화 오류를 같은 포화 queue에 넣으려는 시도가 다시 실패해 오류와 allocation을 무한 생성할 수 있다. 상한 도달을 terminal decision으로 만들고 새 work를 받지 않아야 한다.
- healthy client가 slow client 때문에 publication을 놓치지 않았음을 어떻게 독립적으로 검증합니까?
  - 모범답변: 두 connection의 readiness를 독립 주입해 slow 쪽 slot과 coalesce count만 bounded로 변하고 healthy 쪽 sent sequence·snapshot count와 replica hash가 reference와 일치하는지 확인한다.

### 30초 모범 답변

느린 소비자는 simulation을 멈추게 하지 않고 연결별 outbound state만 제한해야 합니다. control은 최대 64개의 bounded FIFO로 보존하지만, snapshot은 상태 대체 가능성이 있으므로 pending FULL과 DELTA를 각각 최대 하나만 두고 새 generation이 오면 superseded buffer를 해제합니다. partial write는 move-only `PendingWrite`가 byte vector와 offset을 계속 소유하고, readiness가 돌아올 때 suffix만 전송합니다. 연결 종료 시 모든 queue와 retained bytes를 먼저 해제하고 owner mailbox가 이후 player disconnect 상태를 적용하게 해 transport 수명과 game 수명을 분리합니다.

### 답변 핵심 키워드

backpressure, bounded queue, latest-state coalescing, control vs state semantics, nonblocking partial write, move-only buffer, readiness interest, slow-consumer isolation, cleanup-before-owner-mutation

### 백지 구현

#### 구현 목표

연결별 control FIFO와 FULL·DELTA 최신 슬롯을 관리하는 bounded outbound queue를 구현한다. 각 buffer는 이동 전용이며 생성·해제·현재 byte 수 계측이 정확해야 한다.

#### 인터페이스 또는 함수 시그니처

```cpp
struct OutboundMemory {
    std::size_t live_buffers = 0;
    std::size_t live_bytes = 0;
    std::size_t high_water_buffers = 0;
    std::size_t high_water_bytes = 0;
    std::uint64_t created = 0;
    std::uint64_t released = 0;
};

class PendingWrite {
public:
    PendingWrite(std::vector<std::byte> bytes,
                 OutboundMemory* memory)
        : bytes_(std::move(bytes)), memory_(memory) {
        if (memory_) {
            ++memory_->created;
            ++memory_->live_buffers;
            memory_->live_bytes += bytes_.capacity();
            memory_->high_water_buffers =
                std::max(memory_->high_water_buffers, memory_->live_buffers);
            memory_->high_water_bytes =
                std::max(memory_->high_water_bytes, memory_->live_bytes);
        }
    }
    ~PendingWrite() noexcept { release_accounting(); }

    PendingWrite(PendingWrite&& other) noexcept
        : bytes_(std::move(other.bytes_)),
          offset_(other.offset_),
          memory_(std::exchange(other.memory_, nullptr)) {}

    PendingWrite& operator=(PendingWrite&& other) noexcept {
        if (this == &other) return *this;
        release_accounting();
        bytes_ = std::move(other.bytes_);
        offset_ = other.offset_;
        memory_ = std::exchange(other.memory_, nullptr);
        return *this;
    }
    PendingWrite(const PendingWrite&) = delete;
    PendingWrite& operator=(const PendingWrite&) = delete;

    std::span<const std::byte> remaining() const noexcept {
        return std::span<const std::byte>(bytes_).subspan(offset_);
    }

    void consume(std::size_t count) {
        if (count > remaining().size())
            throw std::logic_error("write offset exceeds owned buffer");
        offset_ += count;
    }

    bool complete() const noexcept { return offset_ == bytes_.size(); }

private:
    void release_accounting() noexcept {
        if (!memory_) return;
        --memory_->live_buffers;
        memory_->live_bytes -= bytes_.capacity();
        ++memory_->released;
        memory_ = nullptr;
    }

    std::vector<std::byte> bytes_;
    std::size_t offset_ = 0;
    OutboundMemory* memory_ = nullptr;
};

enum class SnapshotKind { Full, Delta };

enum class EnqueueCode {
    Queued,
    Coalesced,
    ControlBackpressure,
    Terminal
};

class OutboundQueue {
public:
    static constexpr std::size_t kMaxControls = 64;

    EnqueueCode enqueue_control(PendingWrite write) {
        if (terminal_) return EnqueueCode::Terminal;
        if (controls_.size() == kMaxControls) {
            terminal_ = true;
            return EnqueueCode::ControlBackpressure;
        }
        controls_.push_back(std::move(write));
        return EnqueueCode::Queued;
    }

    EnqueueCode enqueue_snapshot(SnapshotKind kind, PendingWrite write) {
        if (terminal_) return EnqueueCode::Terminal;
        bool replaced = false;
        if (kind == SnapshotKind::Full) {
            replaced = full_.has_value() || delta_.has_value();
            // FULL은 이전 queued base와 그 base의 DELTA를 함께 대체한다.
            full_.reset();
            delta_.reset();
            full_.emplace(std::move(write));
        } else {
            replaced = delta_.has_value();
            delta_.reset();
            delta_.emplace(std::move(write));
        }
        return replaced ? EnqueueCode::Coalesced : EnqueueCode::Queued;
    }

    PendingWrite* front_control() {
        return controls_.empty() ? nullptr : &controls_.front();
    }
    PendingWrite* pending_full() { return full_ ? &*full_ : nullptr; }
    PendingWrite* pending_delta() { return delta_ ? &*delta_ : nullptr; }

    void complete_front_control() {
        if (controls_.empty() || !controls_.front().complete())
            throw std::logic_error("control write is not complete");
        controls_.pop_front();
    }

    void complete_snapshot(SnapshotKind kind) {
        auto& slot = kind == SnapshotKind::Full ? full_ : delta_;
        if (!slot || !slot->complete())
            throw std::logic_error("snapshot write is not complete");
        slot.reset();
    }

    void clear() noexcept {
        controls_.clear();
        full_.reset();
        delta_.reset();
        terminal_ = true;
    }

    std::size_t retained_buffers() const noexcept {
        return controls_.size() + full_.has_value() + delta_.has_value();
    }

    std::size_t retained_bytes() const noexcept {
        std::size_t bytes = 0;
        for (const auto& item : controls_) bytes += item.remaining().size();
        if (full_) bytes += full_->remaining().size();
        if (delta_) bytes += delta_->remaining().size();
        return bytes;
    }

private:
    std::deque<PendingWrite> controls_;
    std::optional<PendingWrite> full_;
    std::optional<PendingWrite> delta_;
    bool terminal_ = false;
};
```

#### 입력과 출력

- 입력: 직렬화된 control 또는 snapshot buffer, partial write 소비량, 완료·연결 종료 신호
- 출력: enqueue 결과, 다음에 쓸 buffer view, queue·memory 계측

#### 반드시 만족해야 할 조건

- control buffer는 FIFO이며 최대 64개다.
- control 포화 이후 새 allocation을 무한히 시도하지 않도록 terminal 정책을 명시한다.
- pending FULL은 최대 1개, pending DELTA는 최대 1개다.
- 같은 종류의 새 snapshot은 기존 슬롯을 대체하고 기존 buffer 수명을 즉시 끝낸다.
- FULL과 DELTA 사이 전송 순서·상호 무효화 정책은 명시적이어야 하며, 어느 정책이든 base contract를 깨지 않아야 한다.
- `PendingWrite`는 vector와 offset을 함께 이동한다.
- 이동 후 원본은 memory counter를 더 이상 감소시키지 않는다.
- `consume(count)`는 remaining보다 큰 값을 허용하지 않는다.
- partial write 후 unsent suffix는 같은 buffer가 보존한다.
- `clear`와 소멸은 모든 live buffer를 정확히 한 번 해제한다.
- 항상 `created = released + live_buffers`다.
- `live_bytes`는 실제 보유 buffer capacity 또는 문제에서 정한 byte accounting 기준과 일치한다.
- readiness가 false인 동안 enqueue는 bounded state 갱신만 하고 block하지 않는다.

#### 경계 조건

- 빈 buffer와 1바이트 buffer
- partial consume 0, 정확한 remaining, remaining 초과
- control 64번째와 그 다음 요청
- FULL 100개 연속, DELTA 100개 연속
- FULL과 DELTA가 교차해 들어오는 경우
- 이동 대입 대상이 이미 live buffer를 소유하는 경우
- queue를 여러 번 `clear`하는 경우
- connection close 중 pending control·FULL·DELTA가 모두 있는 경우
- `OutboundMemory*`가 null인 계측 비활성 정책

#### 실패 조건

- coalesced buffer가 memory counter에 live로 남으면 실패다.
- move 후 원본 소멸자가 counter를 다시 감소시키면 실패다.
- readiness가 false라고 simulation 호출이 block되면 실패다.
- control queue 포화에 대한 오류를 같은 포화 queue에 무한 추가하면 실패다.
- partial write에서 이미 보낸 prefix를 다시 전송하면 실패다.

#### 필요한 제약

- 25~30분 범위다.
- 실제 socket·event registration은 범위에서 제외한다.
- snapshot base 유효성은 metadata callback으로 제공된다고 가정해도 된다.
- cross-kind coalescing 정책은 구현 전에 한 문장으로 선언해야 한다.

### 구현 후 자가 검증

- 정상 경로: control FIFO와 snapshot slot을 각각 enqueue·consume·complete하는지 확인
- 경계값: control 64/초과, FULL·DELTA 각 1개 상한 확인
- 실패 경로: oversized consume, terminal 이후 enqueue, 중복 clear 확인
- 상태 변화: snapshot 교체 직후 old buffer release와 new buffer live 계측 확인
- invariant: 모든 시점에 `created == released + live_buffers`, retained count가 실제 queue slot 수와 같은지 확인
- resource cleanup: connection close 후 live buffer·byte가 0인지 확인
- 중복·누락 처리: partial write suffix가 정확히 한 번씩 이어지는지 확인
- 동시성·비동기 문제: readiness callback에서 owner simulation을 직접 block하지 않는지 설명
- 시간·공간 복잡도: snapshot enqueue O(1), control enqueue O(1), 메모리 상한 계산
- 요구사항 충족: slow connection 1개가 다른 queue의 상태를 변경하지 않는지 확인

### 구현 후 설명할 것

1. control과 snapshot에 서로 다른 queue 정책을 적용한 이유
   - 모범답변: control은 각 응답과 순서가 protocol 의미를 가지므로 bounded FIFO가 필요하고, snapshot은 최신 상태로 이전 queued 상태를 대체할 수 있다. 같은 정책을 쓰면 control 손실 또는 snapshot 무한 backlog 중 하나가 생긴다.
2. snapshot coalescing이 packet loss로 가장하는 것과 다른 이유
   - 모범답변: coalescing은 아직 전송되지 않아 queue가 소유한 work를 replication 규칙에 따라 새 work로 교체하고 그 사실을 계측한다. network loss는 전송 뒤 client 도달 여부의 문제이며 ACK/retention 복구 대상이다.
3. move-only buffer와 lifetime 계측을 함께 설계한 방법
   - 모범답변: vector·offset·memory counter 책임을 하나의 move-only 객체에 묶고 이동 시 counter 책임 포인터를 원본에서 null로 넘긴다. 소멸·교체·clear 어느 경로도 같은 release helper를 거친다.
4. UDP write interest를 pending 상태에 따라 켜고 끄는 이유
   - 모범답변: pending slot이 있을 때만 writable notification을 받아 EAGAIN suffix를 재시도하고, 비어 있을 때는 계속 발생하는 writable event와 owner work를 피한다. 원본의 `update_udp_write_interest`가 전체 connection slot을 보고 토글한다.
5. transport close에서 buffer 정리를 owner player mutation보다 분리한 이유
   - 모범답변: socket과 outbound buffer는 connection 수명에 속해 close callback에서 즉시 회수돼야 하고, player disconnect는 owner 순서로 처리돼야 한다. 분리하면 late callback이 Room을 직접 건드리지 않고 reconnect state도 일관되게 만든다.

### 원본 확인 위치

- Thread: G12
- 커밋 메시지: `feat(cpp): bound and coalesce outbound snapshots`
- 파일: `src/transport.hpp`, `src/transport.cpp`, `tests/g12.hpp`, `tests/g12.cpp`, `tests/g12_queue.cpp`
- 함수·클래스·컴포넌트: `OutboundMemory`, `PendingWrite`, `Server::write_datagrams`, `Server::update_udp_write_interest`, `Server::outbound_ready`, `Server::outbound_state`, `Server::observe_outbound`, `OutboundFixture`
- 관련 Thread: G01의 RAII·shutdown, G08·G10의 snapshot base contract, G11의 disconnect grace, G13의 Room 간 격리, G14의 fixed-load 관측

---

<a id="w04-g13"></a>

## [G13 / 기록 제목: Many-room scheduler and hot-room isolation] Room별 accumulator와 failure containment

### 면접 질문

하나의 서버 인스턴스가 여러 Room을 서비스할 때 shared monotonic timestamp를 한 번 읽고도 Room별 simulation time과 catch-up backlog를 독립적으로 유지하려면 scheduler state를 어떻게 구성해야 합니까? 한 hot Room의 unrecovered deadline이 다른 Room을 멈추지 않게 한 경계는 무엇입니까?

꼬리 질문:

- 모든 Room이 하나의 global accumulator를 공유하면 어떤 오류가 생깁니까?
  - 모범답변: 시작 시각과 backlog가 다른 Room의 elapsed가 섞여 hot Room의 지연이 정상 Room tick으로 소비되거나 반대가 된다. 원본은 shared `now`만 사용하고 accumulator·deadline·miss는 `RoomContext`별로 둔다.
- 한 iteration에서 각 scheduled Room을 한 번씩만 방문하고 최대 4틱만 실행한 이유는 무엇입니까?
  - 모범답변: Room별 service quantum을 고정해 앞의 hot Room이 owner loop를 독점하지 못하게 한다. backlog는 버리지 않고 남기므로 다음 iteration에서 공정하게 다시 기회를 얻는다.
- Room 순회 중 새 Room이 추가되거나 hot Room이 닫히면 registry iteration을 어떻게 안전하게 처리합니까?
  - 모범답변: iteration 시작 시 scheduled ID snapshot을 만들거나 close 목록을 모아 순회 후 erase한다. 원본처럼 owner 한 곳에서 registry를 변경하더라도 현재 iterator를 무효화하지 않는 deferred cleanup 경계를 둬야 한다.
- `OVERLOADED` 운영 상태와 `RUNNING` gameplay 상태를 분리한 이유는 무엇입니까?
  - 모범답변: backlog가 남았다는 사실은 일시적인 scheduler 상태이고 game 규칙상 종료 조건은 아니다. 원본은 반복 unrecovered deadline이 20회가 될 때만 `ROOM_OVERLOAD`로 그 Room을 닫는다.
- unrecovered deadline 20회를 어디에 저장하고 어떤 event에서 reset해야 합니까?
  - 모범답변: 각 `RoomContext::missed_deadlines`에 저장한다. 해당 Room의 due backlog가 실제로 회복돼 remaining full tick이 없어지는 service event에서 0으로 reset하고, 다른 Room 성공이나 wall-time 경과로 reset하지 않는다.
- hot Room 종료 시 pending input, resume record, replay journal, scheduler entry, connection을 어떤 경계에서 정리해야 합니까?
  - 모범답변: scheduler는 close 지시만 만들고 owner의 `close_overloaded`가 scheduled entry·deadline을 제거하고 Room pending, resume, replay를 정리한 뒤 그 Room connection에 terminal 오류를 제한적으로 보내고 닫는다.
- instance-wide total pending 상한과 per-player pending 상한을 함께 둔 이유는 무엇입니까?
  - 모범답변: player당 64만으로는 최대 Room·player 수를 곱한 전체 메모리를 제한하지 못하고, 전역 한도만으로는 한 actor 독점을 막지 못한다. 계층별 상한이 local fairness와 instance capacity를 함께 보장한다.
- 31개 정상 Room이 reference replay와 hash까지 일치한다는 검증은 무엇을 보여 줍니까?
  - 모범답변: hot Room의 backlog와 terminal cleanup이 다른 Room의 tick 순서·accepted event·canonical state에 영향을 주지 않았음을 보여 준다. 단순 생존이 아니라 deterministic simulation 결과까지 failure containment됐다는 증거다.

### 30초 모범 답변

서버는 monotonic 시간을 iteration당 한 번 읽되, 각 `RoomContext`가 자기 accumulator, deadline, miss count, replay, resume, model을 소유해야 합니다. scheduler는 scheduled Room을 공정하게 한 번씩 방문하고 각 Room에서 최대 4틱만 실행해 hot Room이 owner loop를 독점하지 못하게 합니다. 서비스되지 못한 due work는 Room별 backlog와 miss로 남고, 20회 회복되지 않으면 그 Room만 `ROOM_OVERLOAD`로 닫아 pending·credential·transport·scheduler 자원을 정리합니다. 다른 Room context와 연결 routing은 그대로 유지돼 failure containment가 보장됩니다.

### 답변 핵심 키워드

per-room context, shared observation/private simulation time, bounded service quantum, fair iteration, backlog retention, missed deadline, room-local terminal, failure containment, aggregate bounds

### 백지 구현

#### 구현 목표

여러 Room의 고정 틱 누적 상태를 한 owner loop에서 공정하게 처리하고, 연속 service miss가 한도에 도달한 Room만 종료하는 scheduler 핵심을 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
using RoomId = std::string;

struct RoomScheduleState {
    RoomId room_id;
    FixedTickAccumulator accumulator;
    bool scheduled;
    bool terminal;
    int missed_deadlines;
    bool operationally_overloaded;
    std::size_t pending_inputs;
};

struct RoomWork {
    RoomId room_id;
    int ticks_to_execute;
    bool close_for_overload;
    std::int64_t remaining_ms;
};

struct ScheduleResult {
    std::vector<RoomWork> work;
    std::size_t rooms_visited;
    std::size_t ticks_scheduled;
};

ScheduleResult schedule_iteration(
    std::span<RoomScheduleState> rooms,
    std::int64_t shared_monotonic_ms,
    const std::function<bool(const RoomId&)>& service_ready,
    int max_catch_up_ticks,
    int max_unrecovered_deadlines);
```

#### 입력과 출력

- 입력: Room별 scheduler 상태, iteration에서 한 번 관찰한 monotonic 시간, Room별 service 가능 여부
- 출력: Room별 실행 틱 수 또는 overload 종료 지시와 집계 정보

#### 반드시 만족해야 할 조건

- shared monotonic timestamp는 모든 Room에 동일하게 전달되지만 accumulator state는 Room별이다.
- terminal 또는 unscheduled Room은 work를 만들지 않는다.
- 각 active Room은 한 iteration에서 최대 한 번 방문된다.
- Room별 실행 틱 수는 `0..max_catch_up_ticks`다.
- catch-up cap 초과 backlog는 해당 Room accumulator에 남는다.
- 한 Room의 due work가 다른 Room의 accumulator나 miss count를 변경하지 않는다.
- service가 가능하고 due work가 처리되면 miss 회복 정책을 명시적으로 적용한다.
- service가 불가능한 due boundary는 해당 Room의 unrecovered miss만 증가시킨다.
- miss가 정확히 한도에 도달하면 그 Room만 `close_for_overload`가 된다.
- 종료 지시가 난 Room에는 같은 iteration의 추가 tick work를 주지 않는다.
- `operationally_overloaded`는 gameplay status와 별도 값이다.
- 함수 자체가 connection·replay·pending을 직접 삭제하지 않고 owner cleanup 단계에 명시적 close 지시를 전달한다.
- 전체 시간 복잡도는 Room 수와 bounded tick 수에 선형이어야 한다.

#### 경계 조건

- Room 0개, 1개, 최대 수에 가까운 Room
- 모든 Room due 0
- 한 Room만 매우 큰 backlog
- 모든 Room이 정확히 4틱 due
- service miss 19회 뒤 한 번 성공, 또는 20번째 연속 miss
- terminal Room과 active Room이 섞인 경우
- iteration 중 close 지시가 여러 Room에서 동시에 나는 경우
- timestamp 역행이 한 Room accumulator에서 발견되는 경우
- Room registry가 이후 cleanup에서 항목을 제거해야 하는 경우
- aggregate pending이 instance 상한에 도달한 경우는 scheduler 바깥 admission과 어떻게 연결할지

#### 실패 조건

- hot Room backlog 때문에 뒤의 Room을 방문하지 않으면 실패다.
- 한 Room의 miss가 다른 Room에 복사되면 실패다.
- close 지시 이후 같은 Room에 tick을 배정하면 실패다.
- cap 초과 backlog를 버리면 실패다.
- scheduler 함수가 registry를 순회하면서 직접 erase해 iterator를 무효화하면 실패 가능성이 있다.

#### 필요한 제약

- 20~30분 범위다.
- 실제 `Room::tick`, socket close, replay export는 callback 이후 owner 단계가 수행한다고 가정한다.
- miss의 "회복" 정책은 문제에 제시된 규칙을 따르거나, 후보자가 선언하고 테스트와 일치시켜야 한다.
- Room 수 상한과 total pending 상한은 별도 admission layer가 보장한다고 가정하되 연계를 설명한다.

### 구현 후 자가 검증

- 정상 경로: 여러 Room이 동일한 timestamp에서 각자 due tick을 받는지 확인
- 경계값: Room별 4틱 cap과 20번째 miss 종료 확인
- 실패 경로: 역행 timestamp, service unavailable, terminal Room 입력 확인
- 상태 변화: hot Room 종료 지시 후 정상 Room accumulator와 miss가 유지되는지 확인
- invariant: iteration당 각 Room 방문 수가 0 또는 1인지 확인
- resource cleanup: 반환된 close 목록을 owner cleanup에 적용했을 때 scheduler registry·pending·credential이 제거돼야 함을 확인
- 동시성·비동기 문제: 하나의 owner thread에서 registry snapshot 또는 deferred erase를 쓰는 이유 설명
- 중복·누락 처리: Room ID 중복, scheduled set과 context registry 불일치 검토
- 시간·공간 복잡도: O(R + R·K), `K <= 4`로 사실상 O(R)인지 설명
- 요구사항 충족: hot Room 하나의 종료가 정상 Room tick 결과를 바꾸지 않는지 확인

### 구현 후 설명할 것

1. shared monotonic read와 Room별 accumulator를 함께 사용한 이유
   - 모범답변: 한 iteration의 Room들이 같은 시간 관찰을 받아 순회 순서 편향을 줄이면서도 각자의 시작점과 backlog를 보존한다. 공유하는 것은 sample뿐이고 simulation time state는 공유하지 않는다.
2. Room당 bounded service quantum이 공정성과 latency를 지키는 방식
   - 모범답변: command와 tick 처리량에 상한을 두면 iteration 비용이 Room 수에 선형으로 제한되고 뒤 Room도 같은 wake에서 기회를 얻는다. hot Room work는 다음 iteration에 남는다.
3. operational overload와 gameplay lifecycle을 분리한 이유
   - 모범답변: 일시 backlog를 바로 game 종료로 만들면 scheduler jitter가 게임 규칙을 바꾼다. 관측 상태로 먼저 표시하고 지속 실패 한도에서만 명시적 Room-local terminal transition을 수행한다.
4. Room-local cleanup boundary가 failure containment를 만드는 방식
   - 모범답변: terminal Room의 scheduler entry, pending input, resume credential, replay, 해당 connection만 owner가 제거하고 다른 `RoomContext`와 routing은 건드리지 않는다. 그래서 실패 domain이 한 Room에 머문다.
5. per-player·per-Room·instance-wide resource bound를 함께 둔 이유
   - 모범답변: per-player는 actor 폭주, per-Room quantum/mailbox는 hot Room 독점, instance-wide limit은 많은 정상-looking Room의 합산 폭주를 막는다. 한 계층의 상한만으로는 다른 두 형태를 통제할 수 없다.

### 원본 확인 위치

- Thread: G13
- 기록 제목: Many-room scheduler and hot-room isolation (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- 파일: `src/transport.hpp`, `src/transport.cpp`, `tests/g13.hpp`, `tests/g13.cpp`
- 함수·클래스·컴포넌트: `RoomContext`, `Server::context`, `Server::room_metrics`, `Server::start_room`, `Server::execute_tick`, `Server::close_overloaded`, `rooms_`, `scheduled_rooms_`, `multi_room_fixture`, `run_isolation_scenario`
- 관련 Thread: G03의 single owner·identity routing, G04의 `FixedTickAccumulator`, G12의 connection별 outbound 격리, G14의 32-Room fixed-load 검증
