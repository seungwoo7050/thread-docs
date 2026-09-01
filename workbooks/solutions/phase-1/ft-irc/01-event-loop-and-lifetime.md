# 이벤트 루프와 객체 수명

이 문서는 운영체제 이벤트를 공통 모델로 바꾸는 경계, 단일 스레드 이벤트 루프의 상태 전이, 콜백과 실패가 객체 수명에 미치는 영향을 다룬다.

## [Thread 01 / `feat(event): kqueue 준비 이벤트 변환 구현` · `feat(event): epoll 준비 이벤트 변환 구현`] P01. 플랫폼 이벤트를 공통 계약으로 정규화

### 면접 질문

`epoll`과 `kqueue`는 준비 이벤트와 오류·종료를 서로 다른 플래그와 구조로 표현합니다. 이 프로젝트의 `EventManager`처럼 상위 서버가 플랫폼 차이를 모르도록 만들 때, 공통 `Event`에 어떤 의미를 보존해야 합니까?

꼬리 질문:

- readable, writable, error, hangup이 한 번에 함께 보고되면 하나만 선택해야 합니까, 모두 보존해야 합니까?
  - 답변: 모두 독립적으로 보존해야 합니다. 이 프로젝트의 `Event`도 관심 비트와 `error`·`hangup`을 별도 필드로 두므로, 마지막 입력을 읽거나 남은 출력을 처리할 기회를 상위 계층이 정책에 따라 판단할 수 있습니다.
- `EINTR`은 연결 오류와 어떻게 다르게 취급해야 합니까?
  - 답변: `EINTR`은 특정 연결의 실패가 아니라 wait 시스템 호출이 신호로 중단된 경우입니다. 이 프로젝트의 두 backend는 빈 이벤트 목록을 반환하고 다음 event-loop 반복에서 다시 기다립니다.
- backend가 현재 관심 상태를 별도 자료구조에 기록하는 이유는 무엇입니까?
  - 답변: `kqueue`는 read/write filter를 개별 추가·삭제하므로 이전 상태와 새 상태의 차이가 필요하고, 두 backend 모두 등록 여부와 `add`·`update` 계약을 검증하는 데 shadow map을 사용합니다.
- `removeFd`에서 이미 닫혔거나 등록되지 않은 descriptor를 어느 수준까지 멱등적으로 처리할 수 있습니까?
  - 답변: 프로젝트에서는 shadow map에 없으면 즉시 반환하고, `kqueue` 정리 중 `ENOENT`·`EBADF`도 무시합니다. 일반 원칙으로 cleanup 경로의 반복 호출은 안전하게 만들되, 그 밖의 예상하지 못한 backend 오류까지 숨기지는 않아야 합니다.

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
    // 준비·종료·오류는 배타적인 상태가 아니므로 각각 그대로 보존한다.
    ReadyEvent event;
    event.fd = fd;
    event.readable = native.readable;
    event.writable = native.writable;
    event.hangup = native.eof;
    event.error = native.failed;
    event.errorCode = native.errorCode;
    return event;
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
  - 답변: fd, read/write 준비, EOF 계열 hangup, 오류 여부와 오류 코드를 보존했습니다. 반면 `EPOLLIN`, `EVFILT_READ` 같은 native 상수와 backend 전용 payload는 상위 서버가 해석하지 않으므로 제외했습니다.
- hangup을 즉시 close 명령으로 바꾸지 않고 사실로 보존한 이유
  - 답변: hangup과 readable이 함께 올 수 있고 송신 대기열도 남을 수 있기 때문입니다. 프로젝트의 서버는 이 사실들을 I/O 결과와 함께 본 뒤 대기열이 없을 때 최종 disconnect를 결정합니다.
- backend별 차이를 factory 아래에 숨길 때 테스트해야 할 계약
  - 답변: 동일 관심 상태가 동일한 공통 비트로 변환되는지, 복수 비트와 오류 코드가 손실되지 않는지, `EINTR`과 add/update/remove 실패가 같은 상위 계약으로 드러나는지를 backend별로 검증해야 합니다.
- 관심 상태 shadow map을 둘 때 생기는 중복 상태와 얻는 검증 가능성
  - 답변: kernel 상태와 사용자 공간 map이 어긋날 위험이 추가되므로 성공한 시스템 호출 뒤에만 map을 갱신해야 합니다. 대신 중복 add, 미등록 update, 멱등 remove를 결정적으로 검사하고 kqueue filter 차이를 계산할 수 있습니다.

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
  - 답변: 프로젝트는 `error`를 먼저 영구 실패로 처리하고, 그 외에는 read, write, 마지막 hangup 순서로 진행하며 각 수명 경계 뒤 fd로 다시 찾습니다. 이 순서는 읽은 마지막 frame과 이미 queue된 출력을 최대한 보존하면서 오류에는 즉시 반응합니다.
- peer가 half-close했지만 서버 송신 대기열이 남아 있다면 어떤 정책이 가능합니까?
  - 답변: 프로젝트처럼 새 read는 중단하고 write-only로 남은 출력을 drain한 뒤 닫을 수 있고, 보안·지연 상한이 더 중요하면 즉시 닫을 수도 있습니다. 현재 구현은 `peerClosed`가 close 요청을 만들고 pending output이 있으면 drain합니다.
- write 관심을 항상 켜 두면 왜 문제가 됩니까?
  - 답변: socket이 대개 writable이므로 보낼 데이터가 없어도 event loop가 계속 깨어나는 busy loop가 됩니다. 프로젝트는 `wantsWrite()`가 참일 때만 write 관심을 파생합니다.
- `send`가 더 진행되지 않는 상태와 영구 오류를 어떻게 구분합니까?
  - 답변: `EINTR`은 재시도하고, `EAGAIN`·`EWOULDBLOCK`과 프로젝트의 0 반환 정책은 현재 flush를 멈추되 queue를 유지합니다. 그 밖의 오류나 요청량보다 큰 반환은 close를 요청하는 영구 실패입니다.
- close 요청 뒤에도 read 관심을 유지할 이유가 있습니까?
  - 답변: 현재 프로젝트 정책에는 없습니다. close 요청 뒤에는 새 명령이 상태를 더 바꾸지 않도록 read를 끄고, pending output이 있을 때만 write 관심을 유지합니다.

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
    const bool closing = state.closeRequested || state.peerClosed;
    if (closing && state.pendingBytes == 0) {
        return NextStep{true, InterestPlan::None};
    }
    if (closing) {
        // 종료가 정해진 연결은 새 입력을 받지 않고 기존 출력만 drain한다.
        return NextStep{false, InterestPlan::WriteOnly};
    }
    if (state.pendingBytes != 0) {
        return NextStep{false, InterestPlan::ReadWrite};
    }
    return NextStep{false, InterestPlan::ReadOnly};
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
  - 답변: 미전송 데이터가 없는 socket도 계속 writable로 보고될 수 있어 CPU를 소모하는 busy loop가 되기 때문입니다. write 관심은 `pendingBytes > 0`에서 파생해야 합니다.
- close 요청과 실제 descriptor close를 분리한 이유
  - 답변: ERROR나 마지막 응답처럼 이미 queue된 출력을 논블로킹으로 drain할 기회를 주기 위해서입니다. 실제 close는 close 요청과 빈 queue가 함께 성립할 때 수행합니다.
- hangup과 peer EOF를 정책에서 어떻게 해석했는지
  - 답변: 둘 다 더 읽을 수 없다는 종료 사실로 해석해 `closing` 상태로 합쳤습니다. 다만 pending output이 있으면 즉시 제거하지 않고 write-only drain을 허용합니다.
- 실제 이벤트 루프에서는 각 I/O·콜백 뒤 어떤 존재성 검사가 추가로 필요한지
  - 답변: callback, `sendTo`, interest 갱신 실패는 현재 연결을 제거할 수 있으므로 안정 키인 fd를 보존하고 매 경계 뒤 `findConnection(fd)`로 다시 찾아야 합니다. 이전 pointer나 reference는 재사용하지 않습니다.
- drain 우선 정책과 즉시 종료 정책의 trade-off
  - 답변: drain 우선은 프로토콜 응답 전달 가능성을 높이지만 느린 peer가 자원을 더 오래 점유합니다. 즉시 종료는 자원 회수가 빠르지만 이미 생성한 응답을 잃으며, 운영 환경에서는 drain 시간·횟수 상한을 함께 두는 편이 안전합니다.

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
  - 답변: map 항목을 지우고 backend에 부분 등록이 남았을 가능성도 멱등 `remove`로 정리해야 합니다. 원본 `acceptReadyClients`도 map 삽입 뒤 `addFd`가 실패하면 삽입한 항목을 erase합니다.
- `disconnect` callback에 객체를 넘기려면 map에서 언제 erase해야 안전합니까?
  - 답변: 원본처럼 `unique_ptr`를 지역 변수로 옮기고 map에서는 먼저 erase한 뒤 callback을 호출해야 합니다. 그러면 callback 동안 객체는 살아 있지만 재진입한 lookup에는 이미 비활성으로 보입니다.
- 방송 함수가 송신자까지 제거할 수 있을 때 channel iterator는 어떻게 다뤄야 합니까?
  - 답변: channel 이름과 recipient fd 목록을 먼저 복사하고 방송 뒤 기존 iterator·pointer를 버립니다. 이후 상태가 필요하면 `_channels.find(channelName)`으로 다시 찾아야 합니다.
- 예외를 잡는 것만으로 use-after-free가 해결되지 않는 이유는 무엇입니까?
  - 답변: callback이 정상 반환하면서 객체를 제거할 수도 있고, catch 블록 자체가 이미 dangling reference를 읽을 수도 있기 때문입니다. 예외 처리와 별개로 외부 호출 뒤 재탐색 규칙이 필요합니다.
- 안정 키(fd, channel name)로 재탐색하는 방식과 `shared_ptr`로 수명을 늘리는 방식의 trade-off는 무엇입니까?
  - 답변: 안정 키 재탐색은 논리적 활성 상태와 객체 수명을 함께 확인하고 단일 소유권을 유지하지만 조회가 반복됩니다. `shared_ptr`는 메모리 수명은 쉽게 늘리지만 map에서 제거된 객체를 계속 사용해 논리적으로 종료된 상태를 변경할 위험과 순환 참조 비용이 있습니다.

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
    if (!connection) {
        return false;
    }
    const int fd = connection->fd;
    if (connections_.find(fd) != connections_.end()) {
        return false;
    }

    const std::pair<
        std::unordered_map<int, std::unique_ptr<Connection>>::iterator,
        bool> inserted = connections_.emplace(fd, std::move(connection));
    if (!inserted.second) {
        return false;
    }

    try {
        backend_.add(fd);
    } catch (...) {
        // 등록의 commit은 map과 backend가 모두 성공한 뒤다.
        backend_.remove(fd);
        connections_.erase(fd);
        return false;
    }
    return true;
}

void ConnectionTable::disconnect(int fd) noexcept {
    const std::unordered_map<int, std::unique_ptr<Connection>>::iterator found =
        connections_.find(fd);
    if (found == connections_.end()) {
        return;
    }

    std::unique_ptr<Connection> owned = std::move(found->second);
    connections_.erase(found);
    backend_.remove(fd);
}

void ConnectionTable::dispatch(
    int fd,
    const std::function<void(Connection&)>& callback) {
    const std::unordered_map<int, std::unique_ptr<Connection>>::iterator found =
        connections_.find(fd);
    if (found == connections_.end()) {
        return;
    }

    // callback은 disconnect(fd)를 호출할 수 있으므로 반환 뒤 found를 읽지 않는다.
    callback(*found->second);
}

bool ConnectionTable::contains(int fd) const {
    return connections_.find(fd) != connections_.end();
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
  - 답변: callback에 reference를 넘기기 전에 fd를 값으로 보존하고, 반환 뒤에는 그 reference와 기존 iterator를 폐기합니다. 후속 작업이 필요할 때만 `connections_.find(fd)` 결과로 새 수명을 시작합니다.
- map erase와 callback 호출 순서를 선택한 이유
  - 답변: 원본 `Server::disconnect`는 객체 소유권을 지역 `unique_ptr`로 옮긴 뒤 map에서 먼저 지웁니다. callback 중 재진입해도 같은 fd가 활성 연결로 보이지 않으면서 callback 인자로 준 객체는 함수가 끝날 때까지 살아 있습니다.
- transactional registration에서 commit 지점을 어디로 잡았는지
  - 답변: map 삽입과 backend `add`가 모두 성공한 반환 직전을 commit으로 잡았습니다. 그 전 예외는 새 fd의 map 항목과 backend 부분 상태를 제거해 외부에 반등록 상태를 노출하지 않습니다.
- rollback 함수가 예외를 던지지 않아야 하는 이유
  - 답변: cleanup 예외가 최초 실패를 가리거나 남은 정리를 중단하면 invariant와 소유권 복구가 불가능해집니다. 그래서 축소 인터페이스의 `remove`와 `disconnect`를 `noexcept` 계약으로 두었습니다.
- `unique_ptr`+재탐색과 `shared_ptr` 기반 수명 연장의 trade-off
  - 답변: `unique_ptr`는 소유자가 명확하고 map 제거 즉시 논리적 비활성을 표현하지만 경계마다 lookup이 필요합니다. `shared_ptr`는 callback 중 메모리 생존을 보장하기 쉽지만 제거된 연결의 논리 상태가 남고 참조 주기와 파괴 시점이 복잡해질 수 있습니다.

### 원본 확인 위치

- Thread 10
- 커밋: `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`, `test(app): 작은 송신 한도에서 상태 정리 검증`, `fix(app): 응답 실패 뒤 명령 처리를 중단`
- 파일: `src/Server.cpp`, `src/ChannelCommands.cpp`, `src/ApplicationSupport.cpp`, `tests/server_lifetime_test.cpp`, `tests/application_lifetime_test.cpp`
- 함수·테스트: `Server::disconnect`, `Server::handleClientEvent`, `Server::refreshInterest`, `IrcApplication::sendRaw`, `IrcApplication::sendNumeric`, `broadcastMode`, `registrationRollbackTest`, `connectCallbackLifetimeTest`, `lineCallbackLifetimeTest`, `interestUpdateRollbackTest`, `modeStopsAfterSenderCleanupTest`
- 컴포넌트: `FakeEventManager`, `CapturedStderr`
- 관련 Thread: 03, 06, 07, 15
