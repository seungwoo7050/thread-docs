# 종료와 검증 전략

이 문서는 서버가 종료 요청을 받은 뒤 어떤 보장을 제공할지, 드물고 위험한 실패 경로를 어떻게 결정적으로 재현할지, 실제 TCP 경계에서 어떤 계약을 검증할지를 다룬다.

## [Thread 14 / `feat(shutdown): 종료 전 송신 대기열 처리`] P15. 제한된 drain을 포함한 graceful shutdown

### 면접 질문

이 프로젝트는 종료 신호를 받으면 client에게 종료 ERROR를 queue하고, 제한된 횟수의 `pollOnce`로 송신 대기열 drain을 시도한 뒤 연결과 backend를 정리합니다. graceful shutdown의 단계와 각 단계에서 보장하는 것·보장하지 못하는 것을 설명해 보세요.

꼬리 질문:

- signal handler 안에서 application shutdown이나 로그 출력을 직접 하면 왜 위험합니까?
  - 답변: 대부분의 C++ 객체 연산, 할당, iostream, socket workflow는 async-signal-safe가 아니어서 lock 재진입과 상태 손상을 일으킬 수 있습니다. 원본 handler는 `volatile sig_atomic_t gRunning`만 0으로 바꾸고 정상 loop가 종료 작업을 수행합니다.
- 종료 drain 중 listen socket을 계속 열어 두면 어떤 일이 생길 수 있습니까?
  - 답변: 원본의 제한된 `pollOnce`가 listen fd의 readable 이벤트도 처리하므로 drain 중 새 연결을 accept할 수 있고 새 명령도 들어올 수 있습니다. 더 강한 계약은 첫 단계에서 listen 관심을 제거하고 socket을 닫아야 합니다.
- `pollOnce(50)`을 8번 호출한다고 해서 wall-clock 400ms 안에 종료된다고 단정할 수 없는 이유는 무엇입니까?
  - 답변: 50ms는 backend wait의 최대 대기 인자일 뿐 이벤트 callback, recv·send, 명령 처리와 scheduler 지연 시간은 포함하지 않습니다. 이벤트가 즉시 준비돼도 처리 시간이 길 수 있어 8회는 횟수 budget이지 hard deadline이 아닙니다.
- 모든 client에게 ERROR를 queue하다가 일부 queue가 실패해 연결이 제거되면 순회를 어떻게 안전하게 유지합니까?
  - 답변: 원본 `IrcApplication::shutdown`처럼 먼저 fd vector snapshot을 얻어 순회하고 각 send·close 요청은 fd로 다시 찾게 합니다. live registry iterator나 `Connection&`를 send 경계 너머로 유지하지 않습니다.
- 출력 완전 전달과 종료 지연 상한은 어떤 trade-off 관계입니까?
  - 답변: drain 시간과 재시도를 늘리면 slow peer에게 마지막 ERROR가 전달될 가능성은 높지만 자원 회수와 process 종료가 늦어집니다. hard deadline을 우선하면 남은 queue는 강제 close 때 손실될 수 있습니다.
- shutdown을 여러 번 요청해도 안전하게 만들려면 어떤 상태가 필요합니까?
  - 답변: 현재 phase, 최초 시작 시각, 사용한 drain 횟수와 알림 수행 여부를 보존하고 phase가 뒤로 가지 않게 해야 합니다. `Stopped` 이후에는 모든 action이 false인 같은 결과를 반환합니다.

### 30초 모범 답변

signal handler는 `sig_atomic_t` 같은 최소 상태만 바꾸고 실제 종료 작업은 event loop로 돌아와 수행해야 합니다. 현재 구현은 client snapshot에 종료 메시지와 close 요청을 넣고 제한된 `pollOnce` drain 뒤 전체 정리를 수행하지만, drain 동안 listen socket이 열려 있어 새 연결·명령을 완전히 차단하지는 못합니다. 따라서 drain은 최선 노력이지 전달 보장이나 hard wall-clock 상한이 아닙니다. 더 강한 설계라면 먼저 accept를 중단하고, 종료 중 send 실패가 client 제거를 유발할 수 있으므로 fd snapshot과 단계별 존재성 재검사를 사용합니다.

### 답변 핵심 키워드

signal-safe handoff, shutdown phase, stop accepting, fd snapshot, queue close notice, bounded drain, best effort, hard deadline 아님, idempotence, forced cleanup

### 백지 구현

**구현 목표**

외부 signal handler와 분리된 shutdown coordinator를 구현한다. coordinator는 단계별 action을 반환하고, 제한된 poll 횟수 또는 절대 deadline 뒤 강제 종료 단계로 넘어간다. 이 면접용 축소 문제는 원본보다 강한 계약을 연습하도록 첫 종료 전이에 `stopAccepting`을 포함한다.

**면접용 축소 인터페이스**

```cpp
#include <chrono>
#include <cstddef>

enum class ShutdownPhase {
    Running,
    NotifyAndClose,
    Draining,
    ForceClose,
    Stopped,
};

struct ShutdownConfig {
    std::size_t maxDrainPolls;
    std::chrono::milliseconds hardDeadline;
};

struct ShutdownObservation {
    bool requestObserved;
    std::size_t openConnections;
    std::size_t connectionsWithPendingOutput;
    std::chrono::steady_clock::time_point now;
};

struct ShutdownDecision {
    ShutdownPhase phase;
    bool stopAccepting;
    bool notifyClients;
    bool pollOnce;
    bool forceCloseAll;
    bool stopBackend;
};

class ShutdownCoordinator {
public:
    explicit ShutdownCoordinator(const ShutdownConfig& config);

    ShutdownDecision step(const ShutdownObservation& observation);
    ShutdownPhase phase() const;

private:
    ShutdownConfig config_;
    ShutdownPhase phase_ = ShutdownPhase::Running;
    std::size_t drainPolls_ = 0;
    std::chrono::steady_clock::time_point startedAt_{};
};
```

**입력과 출력**

- 입력: 종료 요청 관찰 여부, 열린 연결 수, pending output 연결 수, 현재 단조 시각
- 출력: 이번 loop에서 수행할 단계별 action
- 내부 상태: 현재 phase, drain 시도 수, 종료 시작 시각

**반드시 만족해야 할 조건**

- 종료 요청 전에는 shutdown action을 만들지 않는다.
- 첫 종료 전이에서 새 연결 수락 중단과 client 알림 요청이 한 번만 발생한다.
- pending output이 모두 비면 budget을 다 쓰기 전에 정리 단계로 진행한다.
- poll 횟수나 hard deadline 중 먼저 도달한 조건에서 force-close로 진행한다.
- 이미 `Stopped`인 coordinator는 추가 요청에 같은 최종 상태를 반환한다.
- 단계가 뒤로 되돌아가지 않는다.
- openConnections가 0인 경우 불필요한 drain을 생략할 수 있다.

**경계 조건**

- 종료 요청 없음
- 연결 0개에서 첫 종료 요청
- pending output 0인 여러 연결
- 한 번의 poll 뒤 모두 drain
- maxDrainPolls 0
- deadline이 이미 지난 시각
- poll budget과 deadline이 같은 step에서 동시에 도달
- shutdown 요청 반복
- `ForceClose` 뒤 관찰상 연결이 아직 남아 있는 경우

**실패 조건**

- notify 단계에서 일부 client queue 실패
- poll 호출 자체 실패
- force-close 중 개별 cleanup 실패
- backend stop 실패를 상위에 어떻게 보고할지에 대한 정책

**제약**

- 20~25분 안에 상태 전이만 구현한다.
- signal API, socket close, 실제 poll은 구현하지 않는다.
- hard deadline은 `steady_clock`으로 판단한다.
- action은 idempotent 호출자가 소비한다고 가정하되, 각 action이 한 번만 필요한지 명시한다.

```cpp
ShutdownCoordinator::ShutdownCoordinator(
    const ShutdownConfig& config)
    : config_(config) {}

ShutdownDecision ShutdownCoordinator::step(
    const ShutdownObservation& observation) {
    ShutdownDecision decision{
        phase_, false, false, false, false, false};

    if (phase_ == ShutdownPhase::Stopped) {
        return decision;
    }
    if (phase_ == ShutdownPhase::Running) {
        if (!observation.requestObserved) {
            return decision;
        }
        startedAt_ = observation.now;
        phase_ = ShutdownPhase::NotifyAndClose;
        decision.phase = phase_;
        decision.stopAccepting = true;
        decision.notifyClients = observation.openConnections != 0;
        return decision;
    }

    const bool deadlineReached =
        observation.now < startedAt_ ||
        observation.now - startedAt_ >= config_.hardDeadline;

    if (phase_ == ShutdownPhase::NotifyAndClose ||
        phase_ == ShutdownPhase::Draining) {
        if (observation.openConnections == 0) {
            phase_ = ShutdownPhase::Stopped;
            decision.phase = phase_;
            decision.stopBackend = true;
            return decision;
        }
        if (observation.connectionsWithPendingOutput == 0 ||
            drainPolls_ >= config_.maxDrainPolls || deadlineReached) {
            phase_ = ShutdownPhase::ForceClose;
            decision.phase = phase_;
            decision.forceCloseAll = true;
            return decision;
        }

        phase_ = ShutdownPhase::Draining;
        ++drainPolls_;
        decision.phase = phase_;
        decision.pollOnce = true;
        return decision;
    }

    // ForceClose action은 cleanup 일부가 실패해 연결이 남아도 반복 가능하다.
    if (observation.openConnections != 0) {
        decision.forceCloseAll = true;
        return decision;
    }
    phase_ = ShutdownPhase::Stopped;
    decision.phase = phase_;
    decision.stopBackend = true;
    return decision;
}

ShutdownPhase ShutdownCoordinator::phase() const {
    return phase_;
}
```

### 구현 후 자가 검증

- [ ] phase가 단조롭게 앞으로만 진행하는가?
- [ ] notifyClients가 반복 step마다 다시 true가 되지 않는가?
- [ ] pending output 0이면 즉시 force-close/stop 단계로 넘어가는가?
- [ ] poll 횟수와 deadline을 각각 독립적으로 테스트했는가?
- [ ] maxDrainPolls 0에서 drain loop가 실행되지 않는가?
- [ ] 반복 shutdown 요청이 상태를 초기화하지 않는가?
- [ ] `Stopped` 뒤 action이 다시 발생하지 않는가?
- [ ] deadline 계산이 wall clock에 의존하지 않는가?
- [ ] 실제 호출자가 fd snapshot과 존재성 재검사를 해야 함을 분리했는가?
- [ ] "최선 노력 전달"과 "종료 완료"의 의미를 구분했는가?

### 구현 후 설명할 것

- signal handler와 정상 제어 흐름을 분리한 이유
  - 답변: signal context에서는 `sig_atomic_t` 변경처럼 async-signal-safe한 최소 동작만 보장됩니다. 원본처럼 handler가 flag만 바꾸고 event loop가 application·socket·로그 객체를 정상 순서로 다뤄야 합니다.
- 종료 phase와 commit 지점
  - 답변: 최초 요청 관찰 시 시작 시각을 고정하고 `NotifyAndClose`로 단조 전이하는 지점이 shutdown commit입니다. 이후 새 요청은 상태를 초기화하지 않으며 drain 성공·budget 소진에 따라 ForceClose와 Stopped로만 전진합니다.
- poll 횟수 budget과 절대 deadline을 함께 둔 이유
  - 답변: 횟수 budget은 무한 event-loop 반복을 막고, deadline은 각 poll·callback 시간이 길어지는 경우의 경과 시간 제한을 제공합니다. 둘 중 먼저 도달한 조건에서 best-effort drain을 끝냅니다.
- drain 중 listen socket을 언제 닫을지에 대한 선택과 원본 구현의 한계
  - 답변: 답안은 첫 전이에 `stopAccepting`을 반환해 새 작업 유입을 막습니다. 원본은 application 알림 뒤 같은 `Server::pollOnce`를 8회 호출하면서 listen socket을 유지하므로 그 사이 accept와 read를 완전히 차단하지 못합니다.
- 전달 보장, 종료 지연, 구현 복잡도 사이의 trade-off
  - 답변: ACK 기반 전달 확인은 가장 강하지만 protocol과 상태가 복잡하고 종료가 peer에 종속됩니다. bounded drain은 단순하고 지연을 제한하지만 최선 노력만 보장하며, 즉시 close는 가장 빠른 대신 마지막 frame 손실을 감수합니다.

### 원본 확인 위치

- Thread 14
- 커밋: `feat(shutdown): 종료 전 송신 대기열 처리`, `test(irc): 실행 조건과 오류 동작 계약 검증`
- 파일: `src/main.cpp`, `src/IrcApplication.cpp`, `src/Server.cpp`, `tests/irc_contract.py`, `architecture/shutdown-metrics-and-runtime-contract.md`
- 함수·컴포넌트: `IrcApplication::shutdown`, `Server::pollOnce`, `Server::stop`, `gRunning`, `validate_shutdown_log`
- 관련 Thread: 03, 10, 15

---

## [Thread 10 / `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` · Thread 15 / 결정적 검증 계층] P16. 실패 주입으로 수명·rollback 검증

### 면접 질문

event backend의 `addFd`·`updateFd` 실패나 callback 내부 disconnect는 실제 환경에서 드물고 재현도 어렵습니다. 이 프로젝트의 `FakeEventManager`와 작은 송신 한도 테스트처럼 실패를 결정적으로 주입하려면 test double에 어떤 기능이 필요하며, 결과를 무엇으로 검증해야 합니까?

꼬리 질문:

- "예외가 발생했다"만 확인하면 왜 부족합니까?
  - 답변: 예외를 잡았어도 map·backend 한쪽에 fd가 남거나 객체가 두 번 파괴되고 무관한 연결이 제거될 수 있습니다. 원본 테스트는 connection count, fake 관심 map, 로그와 다른 peer/channel 보존까지 사후 invariant로 검사합니다.
- fail-next 방식과 특정 fd·호출 번호 실패 방식의 차이는 무엇입니까?
  - 답변: fail-next는 설정이 간단하지만 준비 코드의 호출 하나가 추가되면 다른 지점에서 실패할 수 있습니다. 특정 fd나 호출 번호 방식은 목표 경계를 안정적으로 겨냥하지만 테스트가 내부 호출 순서에 더 결합될 수 있습니다.
- callback이 disconnect한 뒤 예외까지 던지는 시나리오가 왜 필요한가요?
  - 답변: 정상 반환 경로뿐 아니라 catch에서 오류 보고나 close 요청을 하며 제거된 reference를 다시 쓰는 결함을 드러냅니다. 원본 connect·line lifetime 테스트가 자기 제거 후 throw를 함께 사용합니다.
- private 상태를 직접 보는 white-box test와 public 관찰만 보는 black-box test를 어떻게 나눕니까?
  - 답변: mode·client/channel invariant처럼 외부에서 만들기 어려운 정밀 상태는 작은 white-box fixture로 검증하고, wire·로그·process 종료 계약은 실제 TCP black-box로 검증합니다. 같은 성질을 한 계층에만 의존하지 않도록 나눕니다.
- allocation failure까지 같은 방식으로 다루기 어려운 이유는 무엇입니까?
  - 답변: 할당은 STL·문자열·callback 등 매우 많은 지점에서 암묵적으로 발생하고 전역 allocator 주입은 테스트 격리와 구현 결합 문제가 큽니다. 명시적 backend·sender interface의 실패보다 정확한 한 지점을 지정하기 어렵습니다.

### 30초 모범 답변

실패 주입 대역은 관심 상태 map, 준비 이벤트 queue, 호출 횟수와 "다음 add/update 또는 특정 fd update 실패" 스위치를 제공하면 원하는 경계를 정확히 재현할 수 있습니다. 검증은 예외 자체보다 실패 뒤 invariant가 핵심입니다. connection map과 backend 양쪽에 찌꺼기가 없는지, object가 한 번만 파괴됐는지, sender만 제거되고 무관한 peer·channel은 남는지, 성공 로그가 잘못 기록되지 않았는지를 확인합니다. callback disconnect 후 throw를 함께 넣으면 catch 이후 stale reference를 다시 쓰는 결함도 드러낼 수 있습니다.

### 답변 핵심 키워드

deterministic fault injection, fake backend, scripted outcome, fail-next, call targeting, post-failure invariant, stale state, unrelated-state preservation, white-box vs black-box

### 백지 구현

**구현 목표**

작은 `EventBackend` fake를 작성하고 다음 세 회귀 테스트의 빈 본문을 완성한다.

1. add 실패 뒤 등록 rollback
2. callback 자기 제거 뒤 dispatch 안전성
3. update 실패 뒤 연결·관심 상태 정리

**면접용 축소 인터페이스**

```cpp
#include <functional>
#include <stdexcept>
#include <unordered_map>
#include <vector>

struct Ready {
    int fd;
    bool readable;
};

class EventBackend {
public:
    virtual ~EventBackend() = default;
    virtual void add(int fd, bool write) = 0;
    virtual void update(int fd, bool write) = 0;
    virtual void remove(int fd) noexcept = 0;
    virtual std::vector<Ready> wait() = 0;
};

class FakeEventBackend final : public EventBackend {
public:
    void failNextAdd();
    void failNextUpdateFor(int fd);
    void queueReadable(int fd);

    bool contains(int fd) const;
    std::size_t addCalls() const;
    std::size_t updateCalls() const;

    void add(int fd, bool write) override;
    void update(int fd, bool write) override;
    void remove(int fd) noexcept override;
    std::vector<Ready> wait() override;

private:
    std::unordered_map<int, bool> interests_;
    std::vector<Ready> ready_;
    std::size_t addCalls_ = 0;
    std::size_t updateCalls_ = 0;
    bool failNextAdd_ = false;
    int failUpdateFd_ = -1;
};

void registrationRollbackTest();
void callbackRemovalTest();
void updateRollbackTest();
```

**입력과 출력**

- 입력: 실패 주입 설정과 준비 이벤트
- 출력: backend 호출 결과 또는 의도된 예외
- 테스트 관찰: registry·backend 상태와 callback 호출 기록

**반드시 만족해야 할 조건**

- 실패는 설정한 한 번 또는 지정 fd에서만 발생한다.
- 실패 후 스위치가 자동 소비되는지 정책을 명시한다.
- 존재하지 않는 fd update는 정상 성공으로 위장하지 않는다.
- `remove`는 cleanup 경로에서 사용 가능하도록 예외를 던지지 않는다.
- `wait`는 queue된 이벤트를 결정적 순서로 반환한다.
- 각 테스트는 서로 상태를 공유하지 않는다.
- 테스트는 예외뿐 아니라 map/backend의 사후 상태를 검증한다.

**경계 조건**

- 실패 설정 없이 정상 add/update/remove
- 첫 add 실패와 두 번째 add 성공
- 특정 fd update 실패, 다른 fd update 성공
- callback이 자기 fd 제거
- callback이 제거 뒤 예외 발생
- remove 두 번
- wait queue 비어 있음

**실패 조건**

- 의도하지 않은 호출까지 연쇄 실패
- fake 상태와 production registry 상태가 서로 다르게 남음
- 테스트 순서에 따라 결과가 달라짐
- cleanup 경로에서 fake가 추가 예외를 던져 원래 결함을 가림

**제약**

- 25~30분 안에 fake 핵심과 테스트 하나 이상을 완성한다.
- 실제 OS event API와 sleep을 사용하지 않는다.
- production 구현을 fake 안에 복제하지 않는다.
- 테스트가 구현 세부 호출 순서에 과도하게 결합되지 않도록 최종 invariant를 우선한다.

```cpp
void FakeEventBackend::failNextAdd() {
    failNextAdd_ = true;
}

void FakeEventBackend::failNextUpdateFor(int fd) {
    failUpdateFd_ = fd;
}

void FakeEventBackend::queueReadable(int fd) {
    ready_.push_back(Ready{fd, true});
}

bool FakeEventBackend::contains(int fd) const {
    return interests_.find(fd) != interests_.end();
}

std::size_t FakeEventBackend::addCalls() const {
    return addCalls_;
}

std::size_t FakeEventBackend::updateCalls() const {
    return updateCalls_;
}

void FakeEventBackend::add(int fd, bool write) {
    ++addCalls_;
    if (failNextAdd_) {
        failNextAdd_ = false;
        throw std::runtime_error("injected add failure");
    }
    interests_[fd] = write;
}

void FakeEventBackend::update(int fd, bool write) {
    ++updateCalls_;
    if (fd == failUpdateFd_) {
        failUpdateFd_ = -1;
        throw std::runtime_error("injected update failure");
    }
    const std::unordered_map<int, bool>::iterator found = interests_.find(fd);
    if (found == interests_.end()) {
        throw std::runtime_error("updated an unregistered descriptor");
    }
    found->second = write;
}

void FakeEventBackend::remove(int fd) noexcept {
    interests_.erase(fd);
}

std::vector<Ready> FakeEventBackend::wait() {
    std::vector<Ready> result;
    result.swap(ready_);
    return result;
}

namespace {

void require(bool condition, const char* message) {
    if (!condition) {
        throw std::runtime_error(message);
    }
}

class LifetimeRegistry {
public:
    explicit LifetimeRegistry(EventBackend& backend) : backend_(backend) {}

    bool add(int fd) {
        if (connections_.find(fd) != connections_.end()) {
            return false;
        }
        connections_[fd] = true;
        try {
            backend_.add(fd, false);
        } catch (...) {
            backend_.remove(fd);
            connections_.erase(fd);
            return false;
        }
        return true;
    }

    void disconnect(int fd) noexcept {
        if (connections_.erase(fd) != 0) {
            backend_.remove(fd);
        }
    }

    void dispatch(int fd, const std::function<void()>& callback) {
        if (connections_.find(fd) == connections_.end()) {
            return;
        }
        callback(); // 반환 뒤 기존 iterator나 객체를 다시 읽지 않는다.
    }

    bool enableWrite(int fd) {
        if (connections_.find(fd) == connections_.end()) {
            return false;
        }
        try {
            backend_.update(fd, true);
        } catch (...) {
            disconnect(fd);
            return false;
        }
        return true;
    }

    bool contains(int fd) const {
        return connections_.find(fd) != connections_.end();
    }

private:
    EventBackend& backend_;
    std::unordered_map<int, bool> connections_;
};

} // namespace

void registrationRollbackTest() {
    FakeEventBackend backend;
    LifetimeRegistry registry(backend);
    backend.failNextAdd();

    require(!registry.add(10), "injected add unexpectedly succeeded");
    require(!registry.contains(10), "add failure left registry state");
    require(!backend.contains(10), "add failure left backend state");
    require(backend.addCalls() == 1, "add failure was not targeted once");
}

void callbackRemovalTest() {
    FakeEventBackend backend;
    LifetimeRegistry registry(backend);
    require(registry.add(20), "fixture registration failed");

    bool threw = false;
    try {
        registry.dispatch(20, [&]() {
            registry.disconnect(20);
            throw std::runtime_error("callback failure after disconnect");
        });
    } catch (const std::runtime_error&) {
        threw = true;
    }
    require(threw, "callback exception was not observed");
    require(!registry.contains(20), "callback removal left registry state");
    require(!backend.contains(20), "callback removal left backend state");
}

void updateRollbackTest() {
    FakeEventBackend backend;
    LifetimeRegistry registry(backend);
    require(registry.add(30), "target fixture registration failed");
    require(registry.add(31), "peer fixture registration failed");
    backend.failNextUpdateFor(30);

    require(!registry.enableWrite(30), "injected update unexpectedly succeeded");
    require(!registry.contains(30), "update failure left registry state");
    require(!backend.contains(30), "update failure left backend state");
    require(registry.contains(31), "update failure removed unrelated registry state");
    require(backend.contains(31), "update failure removed unrelated backend state");
}
```

### 구현 후 자가 검증

- [ ] fail-next가 정확히 한 호출에만 적용되는가?
- [ ] 지정 fd 실패가 다른 fd에 전파되지 않는가?
- [ ] add 실패 뒤 backend와 connection registry 모두 비어 있는가?
- [ ] callback 자기 제거 뒤 dispatch가 제거된 객체를 다시 읽지 않는가?
- [ ] callback이 예외를 던져도 cleanup 결과가 같은가?
- [ ] update 실패 뒤 양쪽 registry에서 fd가 제거되는가?
- [ ] 무관한 fd의 상태가 보존되는가?
- [ ] remove가 반복 호출에 안전한가?
- [ ] 테스트마다 새 fixture를 사용해 순서 의존성이 없는가?
- [ ] 실행 시간이 scheduler나 실제 네트워크 상태에 의존하지 않는가?

### 구현 후 설명할 것

- 실패 지점을 선택한 기준과 production 위험
  - 답변: map/backend가 갈라지는 `add`, queue 뒤 관심 갱신인 `update`, callback 자기 제거를 골랐습니다. 모두 원본에서 드물지만 dangling reference나 반등록 fd로 이어지는 수명·rollback 경계입니다.
- fake가 모델링하는 계약과 모델링하지 않는 OS 동작
  - 답변: 등록 관심 상태, one-shot 실패, 미등록 update 거절, 멱등 remove와 준비 이벤트 순서는 모델링합니다. 실제 epoll/kqueue flag 조합, fd 재사용, kernel socket buffer와 scheduler timing은 모델링하지 않습니다.
- 호출 횟수 assertion보다 사후 invariant를 우선한 이유
  - 답변: 내부 최적화로 호출 수가 바뀌어도 실패 뒤 양쪽 registry가 정리되고 무관 상태가 보존되는 production 계약은 같아야 합니다. 횟수는 실패가 정확히 소비됐는지 보조 확인에만 사용했습니다.
- fail-next, 특정 fd, scripted sequence 방식의 trade-off
  - 답변: fail-next는 간단하고 한 호출 회귀에 적합하며, 특정 fd는 다중 연결 격리를 안정적으로 검사합니다. scripted sequence는 복합 재시도까지 표현하지만 fixture가 production 호출 순서에 가장 강하게 결합됩니다.
- 단위 실패 주입과 실제 TCP 통합 테스트가 서로 대체 관계가 아닌 이유
  - 답변: fake는 희귀 분기를 결정적으로 만들지만 kernel framing·backpressure·backend 변환을 재현하지 않습니다. 실제 TCP는 통합 동작을 보지만 특정 실패와 모든 invariant를 정밀하게 유발·관찰하기 어려워 두 계층이 상호 보완합니다.

### 원본 확인 위치

- Thread 10, Thread 15
- 커밋: `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`, `test(app): 작은 송신 한도에서 상태 정리 검증`
- 파일: `tests/server_lifetime_test.cpp`, `tests/application_lifetime_test.cpp`, `tests/connection_test.cpp`, `src/Server.cpp`, `src/Connection.cpp`
- 컴포넌트·테스트: `FakeEventManager`, `ScriptedSender`, `CapturedStderr`, `registrationRollbackTest`, `connectCallbackLifetimeTest`, `lineCallbackLifetimeTest`, `interestUpdateRollbackTest`, `queueLimitCloseTest`, `registrationQueueFailureTest`, `modeStopsAfterSenderCleanupTest`
- 관련 Thread: 03, 10, 15

---

## [Thread 09 / `test(smoke): 실제 TCP 등록과 채널 흐름 검증` · Thread 14 / `test(irc): 실행 조건과 오류 동작 계약 검증` · Thread 15 / `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`] P17. 실제 TCP 계약과 진행성 검증

### 면접 질문

이 프로젝트에는 단위 테스트 외에 실제 TCP smoke, 정확한 CLI·wire·shutdown 계약, 160개 연결과 slow receiver 테스트가 있습니다. 각 검증 계층이 어떤 결함을 잡고, 어떤 주장을 해서는 안 되는지 설명해 보세요.

꼬리 질문:

- 응답에 substring이 포함됐는지만 확인하는 테스트가 왜 위험합니까?
  - 답변: 잘못된 prefix·numeric·parameter가 붙어도 우연히 substring이 맞을 수 있고, 앞선 예상 밖 frame을 건너뛰어 순서 회귀를 숨깁니다. 계약 검증은 다음 frame 전체를 정확히 비교해야 합니다.
- CRLF와 frame 순서를 어떻게 검증해야 합니까?
  - 답변: byte buffer를 LF에서 나누되 각 frame이 실제로 `\r\n`으로 끝났는지 별도 기록하고, queue의 0번째 frame만 기대값과 비교합니다. 원본 `expect_next_exact`가 내용·ending·순서를 함께 검사합니다.
- 임시 port를 먼저 reserve한 뒤 서버를 띄우는 방식에도 어떤 race가 있습니까?
  - 답변: reserve socket을 닫고 server가 bind하기 전 다른 process가 같은 port를 차지할 수 있습니다. 가능하면 server가 port 0에 직접 bind하고 실제 port를 부모에게 알리게 하거나, 실패 시 새 port로 재시도해야 합니다.
- slow receiver의 receive buffer를 작게 하고 읽지 않는 이유는 무엇입니까?
  - 답변: kernel receive buffer를 빨리 채워 server의 nonblocking send가 partial/EAGAIN과 outbound queue를 실제로 거치게 하려는 것입니다. 단순히 느리게 읽는 것보다 backpressure 조건을 재현하기 쉽습니다.
- probe client가 PONG을 받았다는 사실로 무엇을 증명하고 무엇은 증명하지 못합니까?
  - 답변: 지정한 flood와 timeout 아래 slow peer와 무관한 연결이 실제로 진행했다는 성질은 보입니다. 모든 부하에서의 최대 지연, 각 fd의 균등 처리, starvation 부재 같은 strict fairness는 증명하지 못합니다.
- shutdown 로그의 event 순서를 검증할 때 fd 재사용 가능성을 어떻게 고려합니까?
  - 답변: 원본은 shutdown client의 `client_registered`에서 fd를 얻고 그 이전의 가장 가까운 같은 fd `client_connected`, 그 이후 metrics와 정확한 disconnect를 순서대로 찾습니다. fd 숫자만 전역 identity로 보지 않고 nick과 시간적 구간을 함께 묶습니다.
- CI에서 Linux epoll과 macOS kqueue를 모두 돌리는 이유는 무엇입니까?
  - 답변: 공통 `Event` 계약이 같아도 native read/write filter, EOF, 오류 code와 remove 동작이 다르기 때문입니다. 한 OS의 fake나 compile만으로 다른 backend의 실제 변환·등록 회귀를 잡을 수 없습니다.

### 30초 모범 답변

단위 테스트는 framing·queue·rollback을 결정적으로 검증하고, 실제 TCP smoke는 socket 조각화와 여러 peer 상호작용을 확인합니다. 계약 테스트는 정확한 CRLF, numeric, 응답 순서, CLI exit·stderr, shutdown log 순서를 외부 관찰자로 고정합니다. 다중 연결·slow receiver 테스트는 읽지 않는 수신자가 있어도 무관한 probe가 진행한다는 bounded scenario를 검증합니다. 다만 유한한 peer 수와 timeout으로 엄격한 공정성, 모든 scheduler interleaving, 실제 성능 상한을 증명할 수는 없습니다. epoll과 kqueue를 각각 실행해야 공통 추상화 아래 플랫폼별 변환 결함을 잡을 수 있습니다.

### 답변 핵심 키워드

layered verification, exact wire contract, CRLF, frame order, black-box process test, split frame, multi-peer, slow receiver, progress property, finite scenario limitation, cross-platform backend

### 백지 구현

**구현 목표**

이미 실행 중인 축소 line protocol 서버를 대상으로 다음을 확인하는 Python 테스트 harness의 핵심을 작성한다.

1. 분할 전송한 요청이 한 frame으로 처리되는지
2. 응답 frame이 정확한 CRLF와 순서를 가지는지
3. 한 peer가 읽지 않아도 무관한 probe가 제한 시간 안에 응답받는지
4. SIGTERM 뒤 종료 frame과 process 종료를 관찰하는지

**면접용 축소 인터페이스**

```python
from __future__ import annotations

from dataclasses import dataclass, field
import signal
import socket
import subprocess
import time
from typing import Optional


@dataclass
class Peer:
    sock: socket.socket
    label: str
    buffer: bytes = b""
    frames: list[tuple[str, bytes]] = field(default_factory=list)

    def send_line(self, line: str) -> None:
        payload = line.rstrip("\r\n").encode("utf-8") + b"\r\n"
        self.sock.sendall(payload)

    def send_raw(self, data: bytes) -> None:
        self.sock.sendall(data)

    def read_available(self, timeout: float) -> None:
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            self.sock.settimeout(max(0.01, deadline - time.monotonic()))
            try:
                chunk = self.sock.recv(65536)
            except socket.timeout:
                return
            if not chunk:
                raise ConnectionError(f"{self.label}: peer closed")
            self.buffer += chunk
            while b"\n" in self.buffer:
                raw, self.buffer = self.buffer.split(b"\n", 1)
                if raw.endswith(b"\r"):
                    self.frames.append((raw[:-1].decode("utf-8", "replace"), b"\r\n"))
                else:
                    self.frames.append((raw.decode("utf-8", "replace"), b"\n"))

    def expect_next_exact(self, expected: str, timeout: float) -> str:
        deadline = time.monotonic() + timeout
        while not self.frames and time.monotonic() < deadline:
            try:
                self.read_available(min(0.1, deadline - time.monotonic()))
            except ConnectionError as error:
                raise AssertionError(
                    f"{self.label}: closed before {expected!r}; "
                    f"recent={self.frames[-20:]!r}") from error
        if not self.frames:
            raise AssertionError(
                f"{self.label}: timed out waiting for {expected!r}; "
                f"recent={self.frames[-20:]!r}")

        actual, ending = self.frames.pop(0)
        if ending != b"\r\n" or actual != expected:
            raise AssertionError(
                f"{self.label}: expected next {(expected, b'CRLF')!r}, "
                f"got {(actual, ending)!r}; recent={self.frames[-20:]!r}")
        return actual

    def wait_closed(self, timeout: float) -> None:
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            self.sock.settimeout(max(0.01, deadline - time.monotonic()))
            try:
                chunk = self.sock.recv(65536)
            except socket.timeout:
                continue
            if not chunk:
                return
            self.buffer += chunk
        raise AssertionError(
            f"{self.label}: connection did not close; recent={self.frames[-20:]!r}")


def check_split_frame(host: str, port: int) -> None:
    sock = socket.create_connection((host, port), timeout=3.0)
    peer = Peer(sock, "split-frame")
    try:
        peer.send_raw(b"PI")
        peer.send_raw(b"NG :split-token\r")
        peer.send_raw(b"\n")
        peer.expect_next_exact(
            ":irc.relay.local PONG irc.relay.local split-token", 2.0)
    finally:
        sock.close()


def check_slow_receiver_progress(host: str, port: int) -> None:
    def connect(label: str, receive_buffer: Optional[int] = None) -> Peer:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        if receive_buffer is not None:
            sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, receive_buffer)
        sock.settimeout(5.0)
        sock.connect((host, port))
        return Peer(sock, label)

    def register(peer: Peer, nick: str) -> None:
        peer.send_raw(
            f"NICK {nick}\r\nUSER {nick} 0 * :{nick}\r\n".encode("ascii"))
        peer.expect_next_exact(
            f":irc.relay.local 001 {nick} :Welcome to irc-relay-server, {nick}", 3.0)
        peer.expect_next_exact(
            f":irc.relay.local 002 {nick} :Your host is irc.relay.local", 3.0)
        peer.expect_next_exact(
            f":irc.relay.local 003 {nick} :This server is running a C++17 event backend", 3.0)

    slow = connect("slow", 1024)
    sender = connect("sender")
    probe = connect("probe")
    try:
        register(slow, "slow")
        register(sender, "sender")
        register(probe, "probe")

        payload = "x" * 400
        sender.send_raw(
            (f"PRIVMSG slow :{payload}\r\n" * 4096).encode("ascii"))
        probe.send_line("PING :unrelated-progress")
        probe.expect_next_exact(
            ":irc.relay.local PONG irc.relay.local unrelated-progress", 15.0)
        # 의도적으로 slow에서는 flood 응답을 읽지 않는다.
    finally:
        slow.sock.close()
        sender.sock.close()
        probe.sock.close()


def stop_and_verify(process: subprocess.Popen[str], peer: Peer) -> None:
    process.send_signal(signal.SIGTERM)
    try:
        peer.expect_next_exact("ERROR :Server shutting down", 3.0)
        peer.wait_closed(3.0)
        return_code = process.wait(timeout=3.0)
        if return_code != 0:
            raise AssertionError(f"server exited with {return_code}")
    except BaseException:
        if process.poll() is None:
            process.kill()
            process.wait(timeout=3.0)
        raise
```

**입력과 출력**

- 입력: server 주소·port, 실행 중인 process, 여러 실제 socket peer
- 출력: 성공 시 정상 반환, 계약 위반 시 최근 transcript가 포함된 assertion

**반드시 만족해야 할 조건**

- `send_line`은 CRLF를 정확히 한 번 붙인다.
- 수신 buffer는 LF 단위로 frame을 분리하고 실제 ending을 따로 보존한다.
- exact expectation은 다음 frame의 순서와 전체 문자열을 비교한다.
- timeout은 `time.monotonic()` 기준으로 계산한다.
- 연결 close와 socket timeout을 구분한다.
- 실패 메시지에는 최근 frame transcript가 포함된다.
- slow peer는 의도적으로 읽지 않고, sender와 probe는 별도 연결을 사용한다.
- probe 성공은 "해당 시나리오에서 진행했다"로만 해석한다.
- 테스트 종료 시 peer와 child process를 반드시 정리한다.

**경계 조건**

- `\r\n`이 두 recv로 분할됨
- 여러 frame이 한 recv에 들어옴
- LF만 오는 잘못된 ending
- expected frame 앞에 예상치 않은 frame이 옴
- peer가 expected 전 close
- slow receiver의 kernel buffer가 아직 가득 차지 않은 경우
- server startup 실패
- SIGTERM 뒤 종료 frame은 왔지만 process가 남음
- process는 끝났지만 종료 frame이 없음

**실패 조건**

- timeout
- frame 내용·순서·ending 불일치
- 예상 전 connection close
- probe 진행 실패
- child process 비정상 exit
- cleanup 뒤 process 또는 socket 누수

**제약**

- 25~30분 안에 `Peer` framing과 시나리오 하나 이상을 구현한다.
- substring matcher만 사용하지 않는다.
- arbitrary sleep만으로 readiness를 판단하지 않는다.
- 실제 서버 구현 내부에 접근하지 않는 black-box 테스트로 작성한다.
- 유한 테스트가 증명하지 못하는 성질을 결과 메시지에 과장하지 않는다.

### 구현 후 자가 검증

- [ ] split request와 combined response를 모두 처리하는가?
- [ ] exact frame 비교가 순서를 건너뛰지 않는가?
- [ ] CRLF가 아닌 ending을 실패로 감지하는가?
- [ ] timeout 계산이 wall-clock 변경에 영향받지 않는가?
- [ ] 실패 시 최근 transcript와 peer label을 보여 주는가?
- [ ] slow peer, sender, probe 역할이 분리되어 있는가?
- [ ] slow peer가 실제로 읽지 않는 동안 probe 응답을 검사하는가?
- [ ] process startup·정상 종료·강제 kill cleanup 경로가 모두 있는가?
- [ ] 반복 실행 시 고정 port 충돌을 피하는가?
- [ ] 테스트가 보장하는 진행성과 보장하지 않는 strict fairness를 구분했는가?
- [ ] OS별 backend를 실제 CI matrix에서 실행해야 하는 이유를 설명할 수 있는가?

### 구현 후 설명할 것

- unit, process contract, multi-peer/load test의 책임 분리
  - 답변: unit은 parser·queue·rollback 분기를 결정적으로 검증하고, process contract는 CLI exit·stderr·정확한 wire·로그 순서를 외부에서 고정합니다. multi-peer/load는 실제 socket buffer와 scheduler 아래 무관 연결의 진행을 제한된 시나리오로 확인합니다.
- exact next-frame matcher를 사용한 이유
  - 답변: substring 검색은 잘못된 numeric, prefix, parameter 추가와 앞선 예상 밖 frame을 놓칠 수 있습니다. 다음 frame의 전체 문자열·순서·CRLF를 함께 비교해야 protocol 계약 회귀를 정확히 잡습니다.
- timeout·transcript·cleanup을 test harness의 기능으로 둔 이유
  - 답변: 모든 scenario가 같은 단조 deadline과 진단 형식을 사용하고, 실패 지점과 무관하게 socket·child process를 회수할 수 있습니다. 테스트 본문은 행위와 기대 결과에 집중하게 됩니다.
- slow receiver scenario가 만드는 backpressure 조건과 false positive 가능성
  - 답변: 작은 receive buffer의 client를 읽지 않고 큰 direct-message flood를 보내 server outbound queue와 kernel buffer를 압박합니다. 다만 데이터가 실제 buffer를 충분히 채우지 못하면 probe 성공이 backpressure 격리를 거치지 않은 false positive일 수 있어 payload 크기와 queue metric도 보조 확인할 수 있습니다.
- Linux·macOS 교차 실행과 sanitizer가 서로 잡는 결함의 차이
  - 답변: 교차 실행은 epoll·kqueue의 filter, EOF, 오류 변환과 플랫폼 socket 옵션 차이를 잡습니다. ASan·UBSan은 실행된 경로의 use-after-free·overflow 같은 메모리/정의되지 않은 동작을 잡으므로 backend 호환성 테스트를 대체하지 않습니다.

### 원본 확인 위치

- Thread 09, 13, 14, 15
- 커밋: `test(smoke): 실제 TCP 등록과 채널 흐름 검증`, `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`, `test(irc): 실행 조건과 오류 동작 계약 검증`, `ci: Linux·macOS 회귀와 새니타이저 자동화`, `ci: harden cross-platform verification`
- 파일: `tests/irc_smoke.sh`, `tools/irc_smoke_client.py`, `tests/irc_contract.py`, `tests/irc_event_fairness.py`, `.github/workflows/cpp-ft-irc-ci.yml`, `Makefile`
- 함수·컴포넌트: `IrcPeer`, `Peer`, `check_cli_contract`, `check_wire_contract`, `validate_shutdown_log`, `check_many_connections`, `check_slow_receiver_isolation`, `stop_server`
- 관련 Thread: 01, 02, 09, 11, 12, 13, 14, 15
