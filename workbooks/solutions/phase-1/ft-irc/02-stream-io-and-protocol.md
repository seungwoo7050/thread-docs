# 스트림 I/O와 프로토콜 문법

이 문서는 TCP가 메시지 경계를 보존하지 않는다는 사실에서 출발해, 수신 framing·부분 송신·IRC 문법 파싱을 서로 다른 책임으로 나누는 방법을 다룬다.

## [Thread 02 / `feat(connection): IRC 입력 프레임 추출 구현` · `feat(connection): 논블로킹 수신 상태 처리`] P04. TCP 스트림에서 완전한 IRC 프레임 추출

### 면접 질문

클라이언트가 `PING :token\r\n`을 한 번의 `send`로 보냈더라도 서버의 `recv`가 한 번에 같은 단위로 반환한다는 보장은 없습니다. `Connection::readAvailable`과 `extractLines`가 어떤 상태를 유지하고, 어떤 경우에 읽기를 멈추거나 연결 종료를 요청해야 합니까?

꼬리 질문:

- 한 `recv`에 여러 줄이 들어오거나 한 줄이 여러 `recv`로 나뉘면 어떻게 처리합니까?
  - 답변: 연결별 `input_`에 chunk를 누적하고 LF가 보일 때마다 앞에서부터 모든 완전한 frame을 순서대로 꺼냅니다. 마지막 불완전 suffix는 지우지 않고 다음 readable 호출까지 유지합니다.
- `recv`가 `-1`이고 `errno`가 `EINTR`인 경우와 `EAGAIN`인 경우의 차이는 무엇입니까?
  - 답변: `EINTR`은 같은 수신을 즉시 재시도할 수 있는 중단이고, `EAGAIN`·`EWOULDBLOCK`은 현재 읽을 byte를 모두 소비했다는 정상 종료 신호입니다. 후자는 오류나 close 요청으로 바꾸지 않습니다.
- `recv`가 0을 반환하면 이미 buffer에 들어온 완전한 줄은 버려야 합니까?
  - 답변: 버리지 않습니다. 원본도 EOF 전에 `extractLines`가 만든 `lines`를 결과에 남긴 채 `peerClosed`와 close 요청을 함께 반환합니다. 불완전 suffix는 더 완성될 수 없으므로 연결 종료와 함께 폐기됩니다.
- newline이 아직 없는데 partial buffer가 최대 길이를 넘으면 언제 거절해야 합니까?
  - 답변: 새 chunk를 붙이고 완전한 frame을 모두 제외한 suffix가 상한을 넘는 즉시 거절해야 합니다. newline을 기다리면 공격자가 한 연결의 메모리를 제한 없이 키울 수 있습니다.
- framing 계층과 IRC grammar parser를 분리하면 무엇이 좋아집니까?
  - 답변: framing은 TCP 분할·병합, LF/CRLF, 길이와 EOF만 검증하고 parser는 prefix·command·parameter만 다루게 됩니다. 두 경계를 독립적으로 테스트하고 문법 오류와 transport 상태를 혼동하지 않을 수 있습니다.

### 30초 모범 답변

TCP는 byte stream이므로 연결마다 partial input buffer를 보관하고 newline이 나타날 때만 완전한 frame을 꺼냅니다. 한 번의 readable 이벤트에서는 `recv`를 반복해 `EAGAIN`까지 비우되, `EINTR`은 같은 호출을 재시도하고 0은 peer EOF로 기록합니다. EOF 전에 이미 추출한 완전한 줄은 처리할 수 있지만, 남은 불완전 frame의 정책은 명시해야 합니다. 길이 제한은 newline이 온 뒤뿐 아니라 newline 없는 partial buffer에도 적용해야 무제한 메모리 증가를 막습니다. framing은 byte 경계만, parser는 prefix·command·parameter 문법만 맡기면 각각 독립적으로 테스트할 수 있습니다.

### 답변 핵심 키워드

TCP byte stream, 연결별 partial buffer, 여러 frame/분할 frame, `EINTR` 재시도, `EAGAIN` 정상 중단, EOF, 최대 frame 길이, framing과 parsing 분리, bounded memory

### 백지 구현

**구현 목표**

주입 가능한 수신 함수에서 chunk를 반복해서 받아 완전한 LF 종료 frame을 추출한다. CRLF인 경우 frame 끝의 CR은 제거한다.

**면접용 축소 인터페이스**

```cpp
#include <cstddef>
#include <functional>
#include <string>
#include <vector>

enum class ReceiveKind {
    Data,
    Interrupted,
    WouldBlock,
    PeerClosed,
    Fatal,
};

struct ReceiveStep {
    ReceiveKind kind;
    std::string bytes;
    int errorCode;
};

struct ReadBatch {
    std::vector<std::string> lines;
    bool peerClosed;
    bool closeRequested;
    std::string closeReason;
};

class LineReader {
public:
    explicit LineReader(std::size_t maxFrameBytes);
    ReadBatch readAvailable(const std::function<ReceiveStep()>& receive);
    const std::string& buffered() const;

private:
    std::size_t maxFrameBytes_;
    std::string input_;
};
```

**입력과 출력**

- 입력: 매 호출마다 `ReceiveStep`을 돌려주는 수신 함수
- 출력: 이번 호출에서 완성된 line 목록과 EOF·종료 요청 상태
- 내부 상태: 다음 호출로 넘길 불완전 frame

**반드시 만족해야 할 조건**

- chunk 하나에 여러 LF가 있으면 순서대로 모두 추출한다.
- 여러 chunk에 나뉜 frame은 합쳐서 한 번만 반환한다.
- LF 직전 CR만 frame 끝에서 제거한다.
- `Interrupted`는 진행 종료로 보지 않는다.
- `WouldBlock`은 오류가 아니며 현재 batch를 반환한다.
- fatal 오류와 frame 길이 초과는 종료 요청으로 표현한다.
- 반환한 frame byte를 내부 buffer에 다시 남기지 않는다.

**경계 조건**

- 빈 data chunk
- `"\n"`, `"\r\n"`, 연속된 두 newline
- CR 없이 LF만 오는 frame
- CR이 문자열 중간에 있는 frame
- 정확히 최대 길이인 frame과 한 byte 초과한 frame
- 완전한 여러 줄 뒤 EOF
- 불완전한 마지막 줄 뒤 EOF
- `Interrupted`가 여러 번 연속 발생

**실패 조건**

- newline 없는 partial buffer가 제한을 넘음
- fatal receive 결과
- 수신 함수가 `Data`를 반환하면서 계약상 허용하지 않은 빈 chunk를 반복해 무한 루프가 생기는 경우

**제약**

- 20~30분 안에 구현한다.
- grammar parsing은 하지 않는다.
- 전체 buffer를 매 frame마다 불필요하게 여러 번 복사하지 않도록 복잡도를 설명한다.
- 구현 선택에 따라 EOF 뒤 불완전 frame 처리 정책을 명시한다.

```cpp
LineReader::LineReader(std::size_t maxFrameBytes)
    : maxFrameBytes_(maxFrameBytes) {}

ReadBatch LineReader::readAvailable(
    const std::function<ReceiveStep()>& receive) {
    ReadBatch result{};
    std::size_t consumed = 0;

    while (true) {
        const ReceiveStep step = receive();
        if (step.kind == ReceiveKind::Interrupted) {
            continue;
        }
        if (step.kind == ReceiveKind::WouldBlock) {
            input_.erase(0, consumed);
            return result;
        }
        if (step.kind == ReceiveKind::PeerClosed) {
            input_.erase(0, consumed);
            result.peerClosed = true;
            result.closeRequested = true;
            result.closeReason = "peer closed connection";
            return result;
        }
        if (step.kind == ReceiveKind::Fatal) {
            input_.erase(0, consumed);
            result.closeRequested = true;
            result.closeReason = "receive failed: " + std::to_string(step.errorCode);
            return result;
        }
        if (step.bytes.empty()) {
            input_.erase(0, consumed);
            result.closeRequested = true;
            result.closeReason = "receive returned empty data";
            return result;
        }

        input_.append(step.bytes);
        while (true) {
            const std::size_t newline = input_.find('\n', consumed);
            if (newline == std::string::npos) {
                break;
            }
            // 원본과 같이 LF까지 포함한 frame 길이를 제한한다.
            if (newline - consumed + 1 > maxFrameBytes_) {
                input_.clear();
                result.closeRequested = true;
                result.closeReason = "incoming line exceeds maximum length";
                return result;
            }

            std::string line = input_.substr(consumed, newline - consumed);
            if (!line.empty() && line.back() == '\r') {
                line.pop_back();
            }
            result.lines.push_back(std::move(line));
            consumed = newline + 1;
        }

        if (input_.size() - consumed > maxFrameBytes_) {
            input_.clear();
            result.closeRequested = true;
            result.closeReason = "incoming line exceeds maximum length";
            return result;
        }
    }
}

const std::string& LineReader::buffered() const {
    return input_;
}
```

### 구현 후 자가 검증

- [ ] 한 frame이 1, 2, 3개 chunk로 나뉘어도 결과가 같은가?
- [ ] 여러 frame이 한 chunk로 들어오면 순서와 개수가 보존되는가?
- [ ] 완전한 frame만 반환하고 partial data는 다음 호출까지 남는가?
- [ ] 정확한 최대 길이와 최대 길이+1을 구분하는가?
- [ ] `EINTR` 대응 상태에서 이미 받은 데이터가 사라지지 않는가?
- [ ] `WouldBlock`을 연결 오류로 오인하지 않는가?
- [ ] EOF 전에 추출된 line을 누락하지 않는가?
- [ ] 종료 요청 뒤 추가 수신을 무한히 계속하지 않는가?
- [ ] 총 입력 N byte에 대해 불필요한 O(N²) 이동이 발생하지 않는가?

### 구현 후 설명할 것

- 연결별 partial buffer가 필요한 이유
  - 답변: TCP는 application frame 경계를 보존하지 않으므로 한 frame이 여러 `recv`로 나뉠 수 있습니다. 연결별 buffer가 있어야 다른 peer의 byte와 섞지 않고 불완전 suffix를 다음 호출로 넘길 수 있습니다.
- frame 길이 제한을 완성 frame과 partial frame 양쪽에 적용한 방식
  - 답변: LF를 찾았을 때는 LF까지의 frame 길이를 검사하고, LF가 없으면 이미 소비한 prefix를 뺀 suffix 길이를 검사했습니다. 어느 경우든 초과 즉시 close를 요청하고 원본처럼 입력 buffer를 비웁니다.
- EOF와 fatal 오류를 분리한 이유
  - 답변: EOF는 peer가 정상적으로 write 쪽을 닫았다는 protocol 상태여서 그 전에 완성된 lines를 처리할 수 있습니다. fatal은 `recv` 자체의 실패이며 오류 코드를 close 이유에 남겨 진단 가능하게 합니다.
- framing과 IRC grammar parser 사이의 입력 계약
  - 답변: framing은 LF를 소비하고 바로 앞 CR만 제거한 완전한 한 줄을 넘깁니다. parser는 TCP나 누적 buffer를 모르며, CRLF를 제외한 IRC 문법과 별도의 510-octet 계약만 검사합니다.
- buffer 앞부분 삭제 비용을 줄일 수 있는 대안과 현재 구현의 trade-off
  - 답변: 이번 구현은 호출 동안 `consumed` offset을 움직이고 반환 시 한 번만 erase해 frame마다 전체 suffix를 이동하지 않습니다. ring buffer나 chunk deque는 이동을 더 줄일 수 있지만 검색과 조립 로직이 복잡해집니다.

### 원본 확인 위치

- Thread 02
- 커밋: `feat(connection): IRC 입력 프레임 추출 구현`, `feat(connection): 논블로킹 수신 상태 처리`
- 파일: `include/Connection.hpp`, `src/Connection.cpp`, `tests/connection_test.cpp`
- 함수: `Connection::extractLines`, `Connection::readAvailable`, `Connection::requestClose`, `Connection::peerClosed`
- 관련 Thread: 03, 04, 09

---

## [Thread 02 / `feat(connection): 부분 송신 대기열 처리` · `feat(buffer): 송신 대기열 크기 제한`] P05. 부분 송신 대기열과 backpressure

### 면접 질문

논블로킹 socket의 `send`는 요청한 byte보다 적게 보낼 수 있고, 당장 보낼 수 없다는 의미로 `EAGAIN`을 반환할 수 있습니다. 이 프로젝트의 연결별 송신 대기열이 어떤 상태를 가져야 하며, queue limit을 넘는 enqueue가 실패했을 때 어떤 invariant를 지켜야 합니까?

꼬리 질문:

- 이미 일부 보낸 buffer의 앞부분을 매번 `erase`하면 어떤 문제가 생깁니까?
  - 답변: 남은 suffix를 매번 앞으로 이동하므로 작은 partial send가 반복될 때 누적 O(N²) 복사 비용이 생길 수 있습니다. 원본은 offset만 증가시키고 16 KiB를 넘었을 때만 compact합니다.
- `send`가 0을 반환하면 성공·재시도·오류 중 무엇으로 해석하겠습니까?
  - 답변: 원본은 진행 없는 상태로 보고 현재 flush를 멈추며 write 관심과 queue를 유지합니다. 이를 성공으로 처리하면 byte를 잃고, 같은 호출에서 무한 재시도하면 busy loop가 되므로 별도 no-progress 계약이 필요합니다.
- line enqueue 시 CRLF를 누가 붙이고, limit 계산에는 무엇을 포함해야 합니까?
  - 답변: 프로젝트에서는 `queueLine`이 입력 끝의 기존 CR/LF를 제거한 뒤 CRLF 하나를 붙입니다. limit은 wire에 실제 저장되는 본문과 CRLF 두 byte를 모두 포함합니다.
- queue limit 초과 시 기존 queued data를 유지해야 합니까, 전부 버려야 합니까?
  - 답변: 자료구조의 enqueue는 기존 queue를 그대로 유지하고 실패해야 합니다. 프로젝트는 여기에 close 요청을 추가해 해당 연결을 종료하지만, 이미 queue된 다른 byte를 임의로 지우지는 않습니다.
- `MSG_NOSIGNAL` 또는 `SIGPIPE` 처리는 왜 필요합니까?
  - 답변: 닫힌 peer에 `send`할 때 process 전체가 `SIGPIPE`로 종료되는 것을 막고 연결 하나의 오류로 처리하기 위해 필요합니다. 원본은 Linux에서 `MSG_NOSIGNAL`을 쓰고 main에서도 `SIGPIPE`를 무시하며, 지원 플랫폼에서는 `SO_NOSIGPIPE`도 설정합니다.
- 연결별 queue가 있어도 event loop의 엄격한 공정성이 자동 보장되지는 않는 이유는 무엇입니까?
  - 답변: queue는 메모리와 blocking send를 격리하지만, 한 readable fd에서 무제한 recv·frame·command를 처리하면 CPU 시간을 독점할 수 있습니다. 엄격한 공정성에는 이벤트당 byte·frame quantum이나 스케줄링 정책이 별도로 필요합니다.

### 30초 모범 답변

송신 상태는 byte buffer와 아직 보내지 않은 시작 offset으로 표현하면 부분 송신 뒤 이미 보낸 prefix를 매번 이동하지 않아도 됩니다. flush는 진행한 만큼 offset을 늘리고, `EINTR`은 재시도하며, `EAGAIN`은 상태를 그대로 둔 채 write 관심을 유지합니다. 대기열이 비면 offset과 buffer를 함께 초기화하고 write 관심을 제거합니다. 새 payload가 상한을 넘으면 기존 queue를 변경하지 않은 채 실패해야 하며, line의 CRLF도 실제 전송 byte이므로 limit에 포함해야 합니다. 상한 초과를 연결 종료 정책으로 연결할 수 있지만 그 결정은 queue 자료구조와 서버 정책을 분리해 설명하는 편이 좋습니다.

### 답변 핵심 키워드

partial send, buffer+offset, `EINTR`, `EAGAIN`, no-progress, write interest, CRLF 포함 byte 수, overflow-safe limit, 실패 시 무변경, per-connection backpressure, `SIGPIPE`

### 백지 구현

**구현 목표**

연결 하나의 미전송 byte를 관리하는 bounded outbound queue를 구현한다. 실제 socket 대신 주입 가능한 sender를 사용한다.

**면접용 축소 인터페이스**

```cpp
#include <cstddef>
#include <functional>
#include <string>
#include <string_view>

enum class SendKind {
    Written,
    Interrupted,
    WouldBlock,
    Closed,
    Fatal,
};

struct SendStep {
    SendKind kind;
    std::size_t count;
    int errorCode;
};

struct FlushResult {
    bool drained;
    bool closeRequested;
    std::string closeReason;
};

class OutboundQueue {
public:
    explicit OutboundQueue(std::size_t maxPendingBytes);

    bool queueRaw(std::string_view bytes);
    bool queueLine(std::string_view line);
    FlushResult flush(
        const std::function<SendStep(std::string_view)>& send);

    std::size_t pendingBytes() const;
    bool wantsWrite() const;

private:
    std::size_t maxPendingBytes_;
    std::string buffer_;
    std::size_t offset_ = 0;
};
```

**입력과 출력**

- enqueue 입력: raw byte 또는 line
- enqueue 출력: 기존 상태를 보존한 채 수락 여부
- flush 입력: 현재 미전송 view를 받는 sender
- flush 출력: drain 완료 여부와 종료 요청

**반드시 만족해야 할 조건**

- `pendingBytes()`는 이미 보낸 prefix를 제외한 논리 byte 수다.
- 부분 전송 후 남은 suffix만 다음 sender 호출에 전달한다.
- `Interrupted` 뒤에는 상태 변경 없이 재시도할 수 있다.
- `WouldBlock` 뒤에는 상태를 그대로 보존한다.
- queue limit 검사에서 `size_t` overflow가 발생하지 않는다.
- enqueue 거절 시 buffer와 offset이 바뀌지 않는다.
- `queueLine`은 입력의 기존 line ending 정책을 정하고 CRLF 하나를 포함해 계산한다.
- drain 완료 후 buffer와 offset을 일관되게 초기화한다.

**경계 조건**

- 빈 raw payload
- 빈 line
- 정확히 남은 한도와 한도+1
- offset이 0이 아닌 상태에서 추가 enqueue
- 1 byte씩 여러 번 부분 전송
- `Interrupted` 후 성공
- 일부 전송 후 `WouldBlock`
- sender의 `Written` count가 전달 view보다 큼
- sender가 `Written`이면서 count 0을 반환
- fatal 오류와 peer closed

**실패 조건**

- queue limit 초과
- sender 계약 위반
- 더 진행할 수 없는 0-byte 성공을 무한 반복
- fatal/closed 결과 뒤 queue를 계속 정상 상태로 취급

**제약**

- 25~30분 안에 구현한다.
- 매 partial send마다 전체 문자열을 앞에서 `erase`하지 않는다.
- compact가 필요하다면 언제 하는지 설명하되 필수 최적화로 만들지 않는다.
- queue 초과 시 연결을 끊는 정책은 `FlushResult` 또는 호출자 쪽에서 선택할 수 있도록 구분한다.

```cpp
OutboundQueue::OutboundQueue(std::size_t maxPendingBytes)
    : maxPendingBytes_(maxPendingBytes) {}

bool OutboundQueue::queueRaw(std::string_view bytes) {
    const std::size_t pending = pendingBytes();
    if (pending > maxPendingBytes_ || bytes.size() > maxPendingBytes_ - pending) {
        return false;
    }
    if (offset_ != 0) {
        buffer_.erase(0, offset_);
        offset_ = 0;
    }
    buffer_.append(bytes.data(), bytes.size());
    return true;
}

bool OutboundQueue::queueLine(std::string_view line) {
    std::size_t end = line.size();
    while (end != 0 && (line[end - 1] == '\r' || line[end - 1] == '\n')) {
        --end;
    }

    const std::size_t pending = pendingBytes();
    if (pending > maxPendingBytes_ || end > maxPendingBytes_ - pending) {
        return false;
    }
    const std::size_t afterLine = pending + end;
    if (2 > maxPendingBytes_ - afterLine) {
        return false;
    }

    if (offset_ != 0) {
        buffer_.erase(0, offset_);
        offset_ = 0;
    }
    buffer_.append(line.data(), end);
    buffer_.append("\r\n");
    return true;
}

FlushResult OutboundQueue::flush(
    const std::function<SendStep(std::string_view)>& send) {
    while (wantsWrite()) {
        const std::string_view pending(buffer_.data() + offset_, pendingBytes());
        const SendStep step = send(pending);

        if (step.kind == SendKind::Interrupted) {
            continue;
        }
        if (step.kind == SendKind::WouldBlock ||
            (step.kind == SendKind::Written && step.count == 0)) {
            return FlushResult{false, false, ""};
        }
        if (step.kind == SendKind::Closed) {
            return FlushResult{false, true, "peer closed during send"};
        }
        if (step.kind == SendKind::Fatal) {
            return FlushResult{
                false, true, "send failed: " + std::to_string(step.errorCode)};
        }
        if (step.count > pending.size()) {
            return FlushResult{
                false, true, "send returned more bytes than requested"};
        }
        offset_ += step.count;
    }

    buffer_.clear();
    offset_ = 0;
    return FlushResult{true, false, ""};
}

std::size_t OutboundQueue::pendingBytes() const {
    return buffer_.size() - offset_;
}

bool OutboundQueue::wantsWrite() const {
    return offset_ < buffer_.size();
}
```

### 구현 후 자가 검증

- [ ] 여러 번의 partial send를 합친 결과가 원래 byte 순서와 정확히 같은가?
- [ ] `WouldBlock` 전후 pending byte와 offset이 같은가?
- [ ] `Interrupted`가 데이터 중복 전송을 만들지 않는가?
- [ ] exact limit enqueue는 성공하고 limit+1은 기존 상태를 바꾸지 않는가?
- [ ] CRLF 두 byte가 limit 계산에 포함되는가?
- [ ] `pendingBytes()`에서 unsigned underflow가 불가능한가?
- [ ] 0-byte `Written`이 무한 루프를 만들지 않는가?
- [ ] drain 뒤 `wantsWrite()`가 false인가?
- [ ] fatal 결과 뒤 close 이유가 외부에 전달되는가?
- [ ] enqueue와 flush의 시간 복잡도 및 잠재적 buffer compaction 비용을 설명할 수 있는가?

### 구현 후 설명할 것

- buffer+offset 모델을 선택한 이유와 메모리 compaction 시점
  - 답변: 부분 송신마다 이미 보낸 prefix를 이동하지 않고 offset만 늘려 전송 경로를 amortized O(N)으로 유지합니다. 이번 축소 구현은 다음 성공 enqueue 때 compact하고, 원본은 미전송 상태에서 offset이 16 KiB를 넘으면 compact합니다.
- enqueue 실패 시 무변경 보장이 호출자에게 주는 장점
  - 답변: 호출자가 실패한 새 메시지와 이미 수락된 출력의 운명을 분리해 결정할 수 있습니다. limit 초과 때문에 이전 protocol 응답까지 손실되거나 byte 순서가 바뀌지 않습니다.
- write readiness와 `wantsWrite()`를 연결하는 방식
  - 답변: `wantsWrite()`가 참일 때만 backend write 관심을 추가하고, flush가 drain되면 제거합니다. `EAGAIN`이나 0-byte 진행에서는 queue를 보존하므로 write 관심도 유지합니다.
- 연결별 queue limit이 slow receiver를 격리하지만 완전한 공정성을 보장하지는 않는 이유
  - 답변: slow receiver가 점유할 메모리와 blocking 시간을 제한하지만 한 sender가 read·parse·fan-out CPU를 오래 쓰는 것은 막지 못합니다. CPU 공정성은 이벤트당 작업량 제한의 별도 책임입니다.
- `send` 오류·0 반환·`SIGPIPE`에 대한 선택한 정책
  - 답변: `EINTR`에 해당하는 결과는 재시도하고, would-block과 0은 진행 없음으로 반환하며, closed/fatal과 반환 byte 계약 위반은 close를 요청합니다. 실제 socket 계층은 원본처럼 `MSG_NOSIGNAL` 또는 동등한 설정으로 process-wide `SIGPIPE`를 막아야 합니다.

### 원본 확인 위치

- Thread 02
- 커밋: `feat(connection): 부분 송신 대기열 처리`, `feat(buffer): 송신 대기열 크기 제한`
- 파일: `include/Connection.hpp`, `src/Connection.cpp`, `src/ConnectionLimits.hpp`, `tests/connection_test.cpp`
- 함수: `Connection::queueRaw`, `Connection::queueLine`, `Connection::flushPending`, `Connection::pendingBytes`, `Connection::wantsWrite`
- 관련 Thread: 03, 10, 13

---

## [Thread 04 / `feat(parser): IRC 메시지 값과 직렬화 정의` · `feat(parser): IRC 한 줄 구문 해석 구현`] P06. IRC 한 줄 문법 파싱과 대칭 직렬화

### 면접 질문

IRC line은 선택적 prefix, command, 공백으로 구분된 middle parameter, `:` 뒤의 trailing parameter로 구성됩니다. `IrcMessage::parseLine`이 단순 `split(' ')`로 구현되면 어떤 입력을 잘못 처리하며, parser와 serializer가 공유해야 할 계약은 무엇입니까?

꼬리 질문:

- trailing parameter가 빈 문자열인 경우 어떻게 표현합니까?
  - 답변: wire에서는 마지막 구분 공백 뒤 `:`만 써서 `COMMAND :`로 표현하고, parser는 빈 문자열 하나를 params에 넣습니다. 단순 split은 이 값을 잃기 쉽습니다.
- 마지막 parameter에 공백이 없더라도 `:`로 시작한다면 serializer는 어떻게 해야 합니까?
  - 답변: `:`를 trailing marker와 데이터로 구분하려면 marker를 하나 더 붙여 `::literal`로 출력해야 합니다. parser는 첫 colon을 소비하고 값 `:literal`을 복원합니다.
- command를 대문자로 정규화하는 장점과 원문 보존의 trade-off는 무엇입니까?
  - 답변: dispatcher가 대소문자별 분기를 만들지 않고 한 key로 비교할 수 있습니다. 대신 원래 표기를 잃으므로 원본 프로젝트처럼 진단이나 감사에 필요하면 `raw` line을 별도 보존하는 것이 좋습니다.
- prefix 표시는 있는데 command가 없는 line을 어떻게 처리합니까?
  - 답변: 원본처럼 parse 오류로 거절합니다. `:` 뒤 prefix가 비었거나 prefix token 뒤 command가 없으면 의미 있는 메시지 구조를 만들 수 없습니다.
- 최대 line 길이는 CRLF 포함인지 제외인지 어디에서 정의해야 합니까?
  - 답변: framing과 parser의 경계 계약으로 한 곳에 명시해야 합니다. 원본 parser는 CRLF를 제거한 뒤 510 octet을 검사하고, wire 전체 IRC line의 512-byte 상한과 맞춥니다.
- parser가 연속 공백을 허용할지 엄격히 거절할지 어떤 기준으로 정합성을 유지합니까?
  - 답변: 원본은 token 사이 연속 공백을 건너뛰는 관대한 정책을 사용합니다. parser·serializer·테스트가 같은 정책을 공유하고, 허용한 입력이 parameter 의미를 바꾸지 않는지가 기준입니다.

### 30초 모범 답변

일반 split은 trailing의 공백을 여러 parameter로 쪼개고 빈 trailing을 잃기 때문에 cursor로 문법을 순서대로 읽어야 합니다. 선택적 prefix 뒤에는 command가 반드시 있어야 하고, command는 비교를 위해 정규화하되 필요하면 raw line을 별도로 보존합니다. `:`가 parameter 시작 위치에 나오면 그 뒤 전체를 마지막 parameter 하나로 소비합니다. serializer는 마지막 값이 비었거나 공백을 포함하거나 `:`로 시작하면 trailing 형식으로 출력해야 parse와 serialize의 의미가 유지됩니다. 길이와 CRLF 책임은 framing 계층과 명확히 합의해야 합니다.

### 답변 핵심 키워드

cursor parser, optional prefix, command 정규화, middle parameter, trailing parameter, 빈 trailing, colon escaping, CRLF, parse/serialize semantic round-trip, raw 보존

### 백지 구현

**구현 목표**

완전한 한 줄 문자열을 IRC 메시지 구조로 파싱하고, 같은 구조를 CRLF 종료 line으로 직렬화한다. TCP buffer 처리는 하지 않는다.

**면접용 축소 인터페이스**

```cpp
#include <string>
#include <string_view>
#include <vector>

struct IrcMessageView {
    std::string prefix;
    std::string command;
    std::vector<std::string> params;
};

struct ParseResult {
    bool ok;
    IrcMessageView message;
    std::string error;
};

ParseResult parseIrcLine(
    std::string_view line,
    std::size_t maxBytesWithoutCrlf);

std::string serializeIrcLine(const IrcMessageView& message);
```

**입력과 출력**

- parser 입력: CRLF가 제거된 완전한 한 줄
- parser 출력: prefix, 정규화된 command, parameter 목록 또는 오류
- serializer 입력: 메시지 값
- serializer 출력: CRLF로 끝나는 wire line

**반드시 만족해야 할 조건**

- 빈 line과 command 없는 line을 거절한다.
- prefix가 있으면 선두 `:` 다음 token으로만 소비한다.
- command 비교에 사용할 정규화 규칙을 일관되게 적용한다.
- middle parameter는 공백으로 구분하되 trailing 시작 `:`를 구별한다.
- trailing은 공백과 빈 문자열을 포함할 수 있다.
- serializer는 마지막 parameter의 의미가 바뀌지 않도록 trailing 표기를 결정한다.
- 출력은 CRLF 하나로 끝난다.
- 최대 길이 위반을 명확한 오류로 반환한다.

**경계 조건**

- `PING`
- `PING :token`
- `:prefix COMMAND`
- `COMMAND a b :c d`
- `COMMAND :`
- `COMMAND a :b`
- 마지막 parameter가 `:literal`인 구조
- leading/trailing 공백이 있는 입력
- prefix만 있는 입력
- command 뒤 연속 공백
- 정확한 최대 길이와 초과 길이

**실패 조건**

- 비어 있는 command
- prefix 뒤 command 누락
- 길이 초과
- serializer가 표현할 수 없다고 판단한 비정상 구조

**제약**

- 20~30분 안에 parser와 serializer를 함께 구현한다.
- 정규식 하나로 전체 문법을 처리하지 않는다.
- framing용 누적 buffer를 만들지 않는다.
- 원본 프로젝트의 numeric 목록이나 command dispatcher는 구현하지 않는다.

```cpp
ParseResult parseIrcLine(
    std::string_view line,
    std::size_t maxBytesWithoutCrlf) {
    ParseResult result{};
    if (line.empty()) {
        result.error = "empty IRC frame";
        return result;
    }
    if (line.size() > maxBytesWithoutCrlf) {
        result.error = "IRC frame exceeds maximum length";
        return result;
    }

    std::size_t pos = 0;
    if (line[pos] == ':') {
        const std::size_t end = line.find(' ');
        if (end == std::string_view::npos || end == 1) {
            result.error = "message prefix is missing a command";
            return result;
        }
        result.message.prefix.assign(line.substr(1, end - 1));
        pos = end + 1;
    }

    while (pos < line.size() && line[pos] == ' ') {
        ++pos;
    }
    const std::size_t commandStart = pos;
    while (pos < line.size() && line[pos] != ' ') {
        ++pos;
    }
    if (commandStart == pos) {
        result.error = "IRC command is missing";
        return result;
    }
    result.message.command.assign(line.substr(commandStart, pos - commandStart));
    for (char& ch : result.message.command) {
        if (ch >= 'a' && ch <= 'z') {
            ch = static_cast<char>(ch - 'a' + 'A');
        }
    }

    while (pos < line.size()) {
        while (pos < line.size() && line[pos] == ' ') {
            ++pos;
        }
        if (pos == line.size()) {
            break;
        }
        if (line[pos] == ':') {
            result.message.params.emplace_back(line.substr(pos + 1));
            break;
        }

        const std::size_t start = pos;
        while (pos < line.size() && line[pos] != ' ') {
            ++pos;
        }
        result.message.params.emplace_back(line.substr(start, pos - start));
    }

    result.ok = true;
    return result;
}

std::string serializeIrcLine(const IrcMessageView& message) {
    std::string line;
    if (!message.prefix.empty()) {
        line += ':';
        line += message.prefix;
        line += ' ';
    }
    line += message.command;
    for (std::size_t i = 0; i < message.params.size(); ++i) {
        line += ' ';
        const std::string& value = message.params[i];
        const bool last = i + 1 == message.params.size();
        if (last &&
            (value.empty() || value.find(' ') != std::string::npos || value[0] == ':')) {
            line += ':';
        }
        line += value;
    }
    line += "\r\n";
    return line;
}
```

### 구현 후 자가 검증

- [ ] trailing의 내부 공백이 하나의 parameter로 남는가?
- [ ] 빈 trailing이 사라지지 않는가?
- [ ] 마지막 parameter가 `:`로 시작할 때 parse 결과가 달라지지 않는가?
- [ ] prefix 없는 line과 있는 line을 모두 처리하는가?
- [ ] command 정규화가 parameter 내용에는 적용되지 않는가?
- [ ] CRLF가 정확히 한 번만 붙는가?
- [ ] `parse(serialize(message))`가 의미상 같은 구조를 만드는가?
- [ ] 허용하지 않는 공백 정책을 테스트로 명확히 했는가?
- [ ] 길이 기준에 CRLF가 포함되는지 설명과 테스트가 일치하는가?
- [ ] parser가 입력 길이에 대해 O(N)으로 동작하는가?

### 구현 후 설명할 것

- 단순 split 대신 cursor 기반으로 처리한 이유
  - 답변: 공백 구분 middle parameter와 `:` 뒤 나머지 전체인 trailing의 규칙이 다르고 빈 trailing도 보존해야 하기 때문입니다. cursor는 입력을 한 번 순회하면서 각 문법 위치의 의미를 명시합니다.
- raw line 보존 여부와 command 정규화의 trade-off
  - 답변: 축소 구조는 비교 편의를 위해 command만 대문자로 만들고 raw는 저장하지 않습니다. 원본은 `raw`도 보존해 진단 정보를 유지하므로 메모리를 조금 더 쓰는 대신 원문 추적이 가능합니다.
- parser와 serializer의 round-trip을 "문자열 동일성"이 아닌 "의미 동일성"으로 보는 이유
  - 답변: 연속 공백, command 대소문자, middle로도 표현 가능한 마지막 parameter처럼 같은 의미의 wire 표현이 여러 개일 수 있기 때문입니다. 직렬화 후 prefix·정규화 command·params가 같으면 계약을 만족합니다.
- framing 계층과 parser 사이의 최대 길이·CRLF 계약
  - 답변: parser 입력에는 CRLF가 없고 parser 상한은 그 전의 byte 수입니다. 원본 기준으로 parser는 최대 510 octet을 받고, framing·queue 계층이 CRLF 두 byte를 소비·부착합니다.
- 엄격한 문법 거절과 관대한 상호운용성 사이에서 선택한 기준
  - 답변: 빈 line, command 누락, 잘못된 prefix와 길이 초과는 구조를 만들 수 없어 거절하고, token 사이 연속 공백은 의미가 명확해 원본처럼 허용했습니다. 관대함이 parameter 경계를 모호하게 만들지 않는 범위로 제한했습니다.

### 원본 확인 위치

- Thread 04
- 커밋: `feat(parser): IRC 메시지 값과 직렬화 정의`, `feat(parser): IRC 한 줄 구문 해석 구현`, `feat(parser): 버퍼에서 IRC 프레임 소비`
- 파일: `include/IrcMessage.hpp`, `src/IrcMessage.cpp`, `src/Replies.cpp`
- 함수·컴포넌트: `IrcMessage::parseLine`, `IrcMessage::toLine`, `IrcMessage::consumeBuffer`, `Replies::formatMessage`, `Replies::numeric`, `Replies::error`, `Replies::hostmask`
- 관련 Thread: 02, 09, 14
