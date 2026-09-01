# 종료와 검증 전략

이 문서는 서버가 종료 요청을 받은 뒤 어떤 보장을 제공할지, 드물고 위험한 실패 경로를 어떻게 결정적으로 재현할지, 실제 TCP 경계에서 어떤 계약을 검증할지를 다룬다.

## [Thread 14 / `feat(shutdown): 종료 전 송신 대기열 처리`] P15. 제한된 drain을 포함한 graceful shutdown

### 면접 질문

이 프로젝트는 종료 신호를 받으면 client에게 종료 ERROR를 queue하고, 제한된 횟수의 `pollOnce`로 송신 대기열 drain을 시도한 뒤 연결과 backend를 정리합니다. graceful shutdown의 단계와 각 단계에서 보장하는 것·보장하지 못하는 것을 설명해 보세요.

꼬리 질문:

- signal handler 안에서 application shutdown이나 로그 출력을 직접 하면 왜 위험합니까?
- 종료 drain 중 listen socket을 계속 열어 두면 어떤 일이 생길 수 있습니까?
- `pollOnce(50)`을 8번 호출한다고 해서 wall-clock 400ms 안에 종료된다고 단정할 수 없는 이유는 무엇입니까?
- 모든 client에게 ERROR를 queue하다가 일부 queue가 실패해 연결이 제거되면 순회를 어떻게 안전하게 유지합니까?
- 출력 완전 전달과 종료 지연 상한은 어떤 trade-off 관계입니까?
- shutdown을 여러 번 요청해도 안전하게 만들려면 어떤 상태가 필요합니까?

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
    // 직접 구현
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
- 종료 phase와 commit 지점
- poll 횟수 budget과 절대 deadline을 함께 둔 이유
- drain 중 listen socket을 언제 닫을지에 대한 선택과 원본 구현의 한계
- 전달 보장, 종료 지연, 구현 복잡도 사이의 trade-off

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
- fail-next 방식과 특정 fd·호출 번호 실패 방식의 차이는 무엇입니까?
- callback이 disconnect한 뒤 예외까지 던지는 시나리오가 왜 필요한가요?
- private 상태를 직접 보는 white-box test와 public 관찰만 보는 black-box test를 어떻게 나눕니까?
- allocation failure까지 같은 방식으로 다루기 어려운 이유는 무엇입니까?

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
    // 필요한 상태를 직접 정의
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
    // 직접 구현
}

void FakeEventBackend::failNextUpdateFor(int fd) {
    // 직접 구현
}

void FakeEventBackend::add(int fd, bool write) {
    // 직접 구현
}

void FakeEventBackend::update(int fd, bool write) {
    // 직접 구현
}

void FakeEventBackend::remove(int fd) noexcept {
    // 직접 구현
}

std::vector<Ready> FakeEventBackend::wait() {
    // 직접 구현
}

void registrationRollbackTest() {
    // 직접 구현
}

void callbackRemovalTest() {
    // 직접 구현
}

void updateRollbackTest() {
    // 직접 구현
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
- fake가 모델링하는 계약과 모델링하지 않는 OS 동작
- 호출 횟수 assertion보다 사후 invariant를 우선한 이유
- fail-next, 특정 fd, scripted sequence 방식의 trade-off
- 단위 실패 주입과 실제 TCP 통합 테스트가 서로 대체 관계가 아닌 이유

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
- CRLF와 frame 순서를 어떻게 검증해야 합니까?
- 임시 port를 먼저 reserve한 뒤 서버를 띄우는 방식에도 어떤 race가 있습니까?
- slow receiver의 receive buffer를 작게 하고 읽지 않는 이유는 무엇입니까?
- probe client가 PONG을 받았다는 사실로 무엇을 증명하고 무엇은 증명하지 못합니까?
- shutdown 로그의 event 순서를 검증할 때 fd 재사용 가능성을 어떻게 고려합니까?
- CI에서 Linux epoll과 macOS kqueue를 모두 돌리는 이유는 무엇입니까?

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
import socket
import subprocess
from typing import Optional


@dataclass
class Peer:
    sock: socket.socket
    label: str
    buffer: bytes = b""
    frames: list[tuple[str, bytes]] = field(default_factory=list)

    def send_line(self, line: str) -> None:
        # 직접 구현
        pass

    def send_raw(self, data: bytes) -> None:
        # 직접 구현
        pass

    def read_available(self, timeout: float) -> None:
        # 직접 구현
        pass

    def expect_next_exact(self, expected: str, timeout: float) -> str:
        # 직접 구현
        raise NotImplementedError

    def wait_closed(self, timeout: float) -> None:
        # 직접 구현
        pass


def check_split_frame(host: str, port: int) -> None:
    # 직접 구현
    pass


def check_slow_receiver_progress(host: str, port: int) -> None:
    # 직접 구현
    pass


def stop_and_verify(process: subprocess.Popen[str], peer: Peer) -> None:
    # 직접 구현
    pass
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
- exact next-frame matcher를 사용한 이유
- timeout·transcript·cleanup을 test harness의 기능으로 둔 이유
- slow receiver scenario가 만드는 backpressure 조건과 false positive 가능성
- Linux·macOS 교차 실행과 sanitizer가 서로 잡는 결함의 차이

### 원본 확인 위치

- Thread 09, 13, 14, 15
- 커밋: `test(smoke): 실제 TCP 등록과 채널 흐름 검증`, `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`, `test(irc): 실행 조건과 오류 동작 계약 검증`, `ci: Linux·macOS 회귀와 새니타이저 자동화`, `ci: harden cross-platform verification`
- 파일: `tests/irc_smoke.sh`, `tools/irc_smoke_client.py`, `tests/irc_contract.py`, `tests/irc_event_fairness.py`, `.github/workflows/cpp-ft-irc-ci.yml`, `Makefile`
- 함수·컴포넌트: `IrcPeer`, `Peer`, `check_cli_contract`, `check_wire_contract`, `validate_shutdown_log`, `check_many_connections`, `check_slow_receiver_isolation`, `stop_server`
- 관련 Thread: 01, 02, 09, 11, 12, 13, 14, 15
