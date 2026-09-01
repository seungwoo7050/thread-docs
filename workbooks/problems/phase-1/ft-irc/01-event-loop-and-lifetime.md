# 이벤트 루프와 객체 수명

이 문서는 운영체제 이벤트를 공통 모델로 바꾸는 경계, 단일 스레드 이벤트 루프의 상태 전이, 콜백과 실패가 객체 수명에 미치는 영향을 다룬다.

## [Thread 01 / `feat(event): kqueue 준비 이벤트 변환 구현` · `feat(event): epoll 준비 이벤트 변환 구현`] P01. 플랫폼 이벤트를 공통 계약으로 정규화

### 면접 질문

`epoll`과 `kqueue`는 준비 이벤트와 오류·종료를 서로 다른 플래그와 구조로 표현합니다. 이 프로젝트의 `EventManager`처럼 상위 서버가 플랫폼 차이를 모르도록 만들 때, 공통 `Event`에 어떤 의미를 보존해야 합니까?

꼬리 질문:

- readable, writable, error, hangup이 한 번에 함께 보고되면 하나만 선택해야 합니까, 모두 보존해야 합니까?
- `EINTR`은 연결 오류와 어떻게 다르게 취급해야 합니까?
- backend가 현재 관심 상태를 별도 자료구조에 기록하는 이유는 무엇입니까?
- `removeFd`에서 이미 닫혔거나 등록되지 않은 descriptor를 어느 수준까지 멱등적으로 처리할 수 있습니까?

### 30초 모범 답변

공통 이벤트는 OS 플래그를 그대로 노출하는 대신 read·write·error·hangup과 실제 오류 코드를 독립적으로 보존해야 합니다. 이 값들은 동시에 참일 수 있으므로 우선순위를 정해 하나로 축약하면 읽을 수 있는 마지막 데이터나 송신 drain 기회를 잃을 수 있습니다. `EINTR`은 대상 연결 실패가 아니라 wait 자체가 신호로 중단된 것이므로 빈 결과나 재시도로 처리합니다. 관심 상태를 backend가 추적하면 add와 update의 의미를 검증하고, kqueue처럼 read/write filter를 개별 추가·삭제해야 하는 API에서도 현재 상태와의 차이를 계산할 수 있습니다.

### 답변 핵심 키워드

이벤트 정규화, 다중 플래그 보존, error code, EOF와 hangup, `EINTR`, 관심 상태 shadow map, add/update/remove 의미, 상위 계층의 플랫폼 독립성

### 백지 구현

**구현 목표**

면접용 중립 이벤트 구조를 받아 상위 이벤트 루프가 소비할 공통 이벤트로 변환한다. 운영체제 API 호출 자체가 아니라 의미 보존에 집중한다.

**면접용 축소 인터페이스**

```cpp
#include <cstdint>

struct NativeReady {
    bool readable;
    bool writable;
    bool eof;
    bool failed;
    int errorCode;
};

struct ReadyEvent {
    int fd;
    bool readable;
    bool writable;
    bool hangup;
    bool error;
    int errorCode;
};

ReadyEvent normalizeReadyEvent(int fd, const NativeReady& native);
```

**입력과 출력**

- 입력: descriptor와 backend에서 이미 추출한 준비·EOF·오류 정보
- 출력: 플랫폼 이름이나 native flag가 없는 공통 이벤트

**반드시 만족해야 할 조건**

- read와 write가 동시에 준비된 상태를 잃지 않는다.
- EOF/hangup과 read 준비가 동시에 나타날 수 있음을 허용한다.
- 오류 여부와 오류 코드를 구분한다.
- 입력 구조를 변경하지 않는다.

**경계 조건**

- 어떤 준비 비트도 없는 이벤트
- EOF이면서 readable인 이벤트
- failed이지만 오류 코드가 0인 이벤트
- readable·writable·EOF·failed가 모두 참인 이벤트

**실패 조건**

- 유효하지 않은 fd를 허용할지 거부할지 계약을 정하고 일관되게 처리한다.
- 상위 계층이 해석할 수 없는 native 상태를 조용히 버리지 않는다.

**제약**

- 10~15분 안에 구현한다.
- `epoll`·`kqueue` 헤더를 사용하지 않는다.
- 한 종류의 이벤트만 남기는 `if/else if` 구조로 축약하지 않는다.

```cpp
ReadyEvent normalizeReadyEvent(int fd, const NativeReady& native) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] readable과 writable이 동시에 보존되는가?
- [ ] EOF가 read 가능 데이터의 존재를 덮어쓰지 않는가?
- [ ] error boolean과 `errorCode`의 관계를 명시했는가?
- [ ] 아무 비트도 없는 입력의 결과가 정의되어 있는가?
- [ ] 변환 결과가 특정 backend 상수에 의존하지 않는가?
- [ ] 각 필드 변환이 O(1)이며 추가 동적 할당이 없는가?

### 구현 후 설명할 것

- 공통 모델에 넣은 정보와 의도적으로 제외한 정보
- hangup을 즉시 close 명령으로 바꾸지 않고 사실로 보존한 이유
- backend별 차이를 factory 아래에 숨길 때 테스트해야 할 계약
- 관심 상태 shadow map을 둘 때 생기는 중복 상태와 얻는 검증 가능성

### 원본 확인 위치

- Thread 01
- 커밋: `feat(event): kqueue 관심 상태 등록 구현`, `feat(event): kqueue 준비 이벤트 변환 구현`, `feat(event): epoll 관심 상태 등록 구현`, `feat(event): epoll 준비 이벤트 변환 구현`
- 파일: `include/EventManager.hpp`, `src/KqueueEventManager.cpp`, `src/EpollEventManager.cpp`
- 컴포넌트: `EventInterest`, `Event`, `EventManager`, `EventManager::createDefault`
- 관련 Thread: 03, 15

---

## [Thread 03 / `feat(server): 논블로킹 연결 수락과 등록 구현` · `feat(server): 준비 이벤트 루프 구동` · `feat(server): 송신 큐와 쓰기 관심 상태 연결` · `feat(server): 연결 해제와 오류 정리 구현`] P02. 관심 상태와 close-drain 상태 전이

### 면접 질문

이 프로젝트의 서버는 연결마다 송신 대기열이 생길 때만 write 관심을 등록하고, close가 요청되어도 남은 출력이 있으면 즉시 descriptor를 닫지 않습니다. `Server::handleClientEvent`와 `Server::refreshInterest`가 지켜야 할 상태 전이를 설명해 보세요.

꼬리 질문:

- error, readable, writable, hangup이 동시에 왔을 때 처리 순서를 어떻게 정하겠습니까?
- peer가 half-close했지만 서버 송신 대기열이 남아 있다면 어떤 정책이 가능합니까?
- write 관심을 항상 켜 두면 왜 문제가 됩니까?
- `send`가 더 진행되지 않는 상태와 영구 오류를 어떻게 구분합니까?
- close 요청 뒤에도 read 관심을 유지할 이유가 있습니까?

### 30초 모범 답변

연결의 write 관심은 "보낼 데이터가 남아 있음"에서 파생되어야 하며, 대기열이 비면 제거해야 busy loop를 피할 수 있습니다. close 요청이 있고 대기열이 비면 즉시 제거하지만, 출력이 남으면 read는 끄고 write만 유지해 제한된 drain을 시도할 수 있습니다. 이벤트 하나에 여러 사실이 함께 올 수 있으므로 각 단계 뒤 연결이 여전히 존재하는지와 close 상태를 다시 확인해야 합니다. hangup도 읽을 마지막 데이터나 보낼 데이터가 남을 수 있어 정책상 즉시 삭제와 동일시하지 않고, I/O 결과와 대기열 상태를 함께 보고 최종 결정을 내립니다.

### 답변 핵심 키워드

파생 관심 상태, write busy loop 방지, close-drain, read 차단, partial progress, 다중 이벤트 플래그, 단계별 재검증, half-close, idempotent disconnect

### 백지 구현

**구현 목표**

연결의 현재 상태를 받아 다음 event interest와 즉시 제거 여부를 결정하는 순수 정책 함수를 작성한다. 실제 socket 호출은 하지 않는다.

**면접용 축소 인터페이스**

```cpp
#include <cstddef>

enum class InterestPlan {
    None,
    ReadOnly,
    WriteOnly,
    ReadWrite,
};

struct ConnectionSnapshot {
    bool closeRequested;
    bool peerClosed;
    std::size_t pendingBytes;
};

struct NextStep {
    bool disconnectNow;
    InterestPlan interests;
};

NextStep decideNextStep(const ConnectionSnapshot& state);
```

**입력과 출력**

- 입력: close 요청 여부, peer 종료 관찰 여부, 미전송 byte 수
- 출력: 즉시 제거 여부와 다음 read/write 관심 계획

**반드시 만족해야 할 조건**

- 미전송 데이터가 없는데 write 관심만 남는 상태를 만들지 않는다.
- close가 요청되고 미전송 데이터도 없으면 제거 결정을 내린다.
- close가 요청됐지만 미전송 데이터가 있으면 새 입력을 더 받지 않는 정책을 표현한다.
- `disconnectNow`와 `InterestPlan::None`의 관계를 명확히 정의한다.

**경계 조건**

- 정상 연결이며 pending 0
- 정상 연결이며 pending 1
- close 요청과 pending 0
- close 요청과 pending 양수
- peerClosed와 pending 양수
- peerClosed와 closeRequested가 모두 참인 상태

**실패 조건**

- 서로 모순되는 결과를 만들지 않는다. 예: 즉시 제거하면서 read/write 관심을 요구하는 결과
- pending 0인데 write-only로 남겨 무한 wakeup을 유발하지 않는다.

**제약**

- 10~15분 안에 구현한다.
- 상태 전이 정책을 순수 함수로 작성한다.
- 실제 repo의 구현을 재현할 필요는 없지만, 선택한 half-close 정책을 설명할 수 있어야 한다.

```cpp
NextStep decideNextStep(const ConnectionSnapshot& state) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 가능한 boolean 조합을 표로 적어 모두 확인했는가?
- [ ] write 관심이 pending byte의 존재와 일치하는가?
- [ ] close 요청 뒤 새 입력을 허용할지 금지할지 일관적인가?
- [ ] peerClosed 상태에서 남은 출력의 처리 방침이 정의되어 있는가?
- [ ] 즉시 제거 결과가 다른 관심 상태와 모순되지 않는가?
- [ ] 함수가 외부 상태를 변경하지 않아 단위 테스트하기 쉬운가?

### 구현 후 설명할 것

- write 관심을 항상 등록하지 않는 이유
- close 요청과 실제 descriptor close를 분리한 이유
- hangup과 peer EOF를 정책에서 어떻게 해석했는지
- 실제 이벤트 루프에서는 각 I/O·콜백 뒤 어떤 존재성 검사가 추가로 필요한지
- drain 우선 정책과 즉시 종료 정책의 trade-off

### 원본 확인 위치

- Thread 03
- 커밋: `feat(server): 논블로킹 연결 수락과 등록 구현`, `feat(server): 준비 이벤트 루프 구동`, `feat(server): 클라이언트 입력 이벤트 전달`, `feat(server): 송신 큐와 쓰기 관심 상태 연결`, `feat(server): 연결 해제와 오류 정리 구현`
- 파일: `include/Server.hpp`, `src/Server.cpp`
- 함수: `Server::pollOnce`, `Server::acceptReadyClients`, `Server::handleClientEvent`, `Server::refreshInterest`, `Server::sendTo`, `Server::disconnect`
- 관련 Thread: 01, 02, 10, 14

---

## [Thread 10 / `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` · `fix(app): 응답 실패 뒤 명령 처리를 중단`] P03. 콜백 중 객체 제거와 실패 경계 rollback

### 면접 질문

이 프로젝트에서는 connect/line callback이 현재 연결을 제거할 수 있고, 응답을 queue하는 과정의 event interest 갱신 실패도 연결 제거로 이어질 수 있습니다. 이런 환경에서 `Connection&`나 `Channel*`를 함수 끝까지 계속 사용하는 것이 왜 위험하며, 어떤 코딩 규칙으로 수명을 보호해야 합니까?

꼬리 질문:

- event backend 등록이 실패했는데 connection map에는 객체가 들어간 상태라면 무엇을 rollback해야 합니까?
- `disconnect` callback에 객체를 넘기려면 map에서 언제 erase해야 안전합니까?
- 방송 함수가 송신자까지 제거할 수 있을 때 channel iterator는 어떻게 다뤄야 합니까?
- 예외를 잡는 것만으로 use-after-free가 해결되지 않는 이유는 무엇입니까?
- 안정 키(fd, channel name)로 재탐색하는 방식과 `shared_ptr`로 수명을 늘리는 방식의 trade-off는 무엇입니까?

### 30초 모범 답변

콜백과 송신 함수는 단순 호출이 아니라 현재 연결을 삭제할 수 있는 수명 경계입니다. 따라서 그 경계를 넘기 전에 필요한 fd나 channel 이름 같은 안정 값을 복사하고, 호출 뒤에는 이전 reference·pointer·iterator를 폐기한 다음 registry에서 다시 찾습니다. 등록 과정은 socket, event backend, connection map이 함께 성공해야 완료된 것이므로 중간 실패 시 이미 성공한 단계를 역순으로 정리해야 합니다. `disconnect`는 소유권을 map 밖으로 옮긴 뒤 map에서 먼저 제거하고 callback 동안 지역 `unique_ptr`로 객체를 살려 두면 재진입에도 같은 fd가 활성 상태로 보이지 않습니다.

### 답변 핵심 키워드

수명 경계, reentrancy, 안정 키, 재탐색, dangling reference, `unique_ptr` 소유권 이동, 역순 rollback, idempotent cleanup, 송신 실패 전파, 무관 상태 보존

### 백지 구현

**구현 목표**

작은 연결 registry에서 다음 두 동작을 구현한다.

1. backend 등록과 map 삽입이 모두 성공해야 연결을 공개하는 transactional registration
2. callback이 자기 자신을 제거해도 이후에 제거된 객체를 다시 접근하지 않는 dispatch

**면접용 축소 인터페이스**

```cpp
#include <functional>
#include <memory>
#include <string>
#include <unordered_map>

struct Connection {
    explicit Connection(int value) : fd(value) {}
    int fd;
};

class EventBackend {
public:
    virtual ~EventBackend() = default;
    virtual void add(int fd) = 0;
    virtual void remove(int fd) noexcept = 0;
};

class ConnectionTable {
public:
    explicit ConnectionTable(EventBackend& backend);

    bool registerConnection(std::unique_ptr<Connection> connection);
    void disconnect(int fd) noexcept;
    void dispatch(int fd, const std::function<void(Connection&)>& callback);
    bool contains(int fd) const;

private:
    EventBackend& backend_;
    std::unordered_map<int, std::unique_ptr<Connection>> connections_;
};
```

**입력과 출력**

- 등록 입력: 소유권을 가진 새 연결
- 등록 출력: 완전 등록 성공 여부
- dispatch 입력: fd와 임의 callback
- dispatch 출력: 없음. callback 뒤 연결이 없어도 정상 종료해야 한다.

**반드시 만족해야 할 조건**

- backend `add` 실패 뒤 map과 backend 어디에도 새 연결이 남지 않는다.
- 중복 fd 등록 정책을 명확히 한다.
- callback이 `disconnect(fd)`를 호출해도 dispatch가 기존 reference를 다시 읽지 않는다.
- `disconnect`는 여러 번 호출해도 안전하다.
- 정리 중 발생한 예외 때문에 소유 객체가 누수되지 않는다.

**경계 조건**

- 존재하지 않는 fd dispatch
- callback 내부 자기 제거
- callback 내부 제거 후 예외 발생
- backend add 즉시 실패
- 동일 fd 중복 등록
- disconnect 중 backend remove가 이미 해제된 descriptor를 보는 경우

**실패 조건**

- map에는 있지만 backend에는 없는 반등록 상태
- backend에는 있지만 map에는 없는 활성 descriptor
- callback 이후 dangling reference 접근
- 실패한 등록의 소유권 누수

**제약**

- 20~30분 안에 구현한다.
- raw owning pointer를 사용하지 않는다.
- callback을 강제로 제한하지 말고 자기 제거가 가능하다고 가정한다.
- 정답에 가까운 별도 의사코드는 작성하지 않는다.

```cpp
ConnectionTable::ConnectionTable(EventBackend& backend)
    : backend_(backend) {}

bool ConnectionTable::registerConnection(
    std::unique_ptr<Connection> connection) {
    // 직접 구현
}

void ConnectionTable::disconnect(int fd) noexcept {
    // 직접 구현
}

void ConnectionTable::dispatch(
    int fd,
    const std::function<void(Connection&)>& callback) {
    // 직접 구현
}

bool ConnectionTable::contains(int fd) const {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] backend add 실패 전후의 map 크기가 같은가?
- [ ] callback이 자기 fd를 제거한 뒤에도 메모리 접근이 이어지지 않는가?
- [ ] callback이 제거 후 예외를 던져도 객체가 한 번만 파괴되는가?
- [ ] 중복 등록이 기존 연결을 덮어쓰지 않는가?
- [ ] disconnect를 두 번 호출해도 backend와 map 상태가 깨지지 않는가?
- [ ] 실패한 동작이 다른 fd의 연결을 건드리지 않는가?
- [ ] 등록 성공 시에만 map·backend 양쪽에 항목이 존재하는 invariant를 설명할 수 있는가?
- [ ] 소유권 이전 시점과 callback 동안 객체 생존 범위가 명확한가?

### 구현 후 설명할 것

- 외부 호출 전 안정 키를 복사하고 호출 후 재탐색하는 규칙
- map erase와 callback 호출 순서를 선택한 이유
- transactional registration에서 commit 지점을 어디로 잡았는지
- rollback 함수가 예외를 던지지 않아야 하는 이유
- `unique_ptr`+재탐색과 `shared_ptr` 기반 수명 연장의 trade-off

### 원본 확인 위치

- Thread 10
- 커밋: `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`, `test(app): 작은 송신 한도에서 상태 정리 검증`, `fix(app): 응답 실패 뒤 명령 처리를 중단`
- 파일: `src/Server.cpp`, `src/ChannelCommands.cpp`, `src/ApplicationSupport.cpp`, `tests/server_lifetime_test.cpp`, `tests/application_lifetime_test.cpp`
- 함수·테스트: `Server::disconnect`, `Server::handleClientEvent`, `Server::refreshInterest`, `IrcApplication::sendRaw`, `IrcApplication::sendNumeric`, `broadcastMode`, `registrationRollbackTest`, `connectCallbackLifetimeTest`, `lineCallbackLifetimeTest`, `interestUpdateRollbackTest`, `modeStopsAfterSenderCleanupTest`
- 컴포넌트: `FakeEventManager`, `CapturedStderr`
- 관련 Thread: 03, 06, 07, 15
