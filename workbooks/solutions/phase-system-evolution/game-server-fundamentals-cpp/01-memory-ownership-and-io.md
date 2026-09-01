# 메모리·소유권·I/O 면접 워크북

이 문서는 G01~G03에서 확인되는 C++ 자원 수명, TCP 스트림 파싱, 오류 격리, 단일 소유자 상태 변경을 묶는다. G01과 G02는 기록에서 확인한 Conventional Commit subject를 사용했고, G03은 exact subject가 확인되지 않아 Thread 기록 제목으로 명시했다.

---

<a id="w01-g01"></a>

## [G01 / `feat: establish kqueue authoritative arena baseline`] 이동 전용 OS 자원과 종료 수명주기

### 면접 질문

`Fd` 같은 OS 핸들 래퍼를 복사 불가능하고 이동 가능한 RAII 타입으로 만든 이유는 무엇입니까? `Server::shutdown`이 여러 경로에서 호출될 수 있을 때 어떤 invariant를 지켜야 합니까?

꼬리 질문:

- 이동 대입 대상이 이미 유효한 파일 디스크립터를 갖고 있다면 어떤 순서로 처리해야 합니까?
  - 모범답변: 대상이 가진 기존 디스크립터를 먼저 정확히 한 번 닫고, 그다음 원본의 값과 해제 정책을 넘겨받은 뒤 원본을 빈 상태로 만든다. 이 프로젝트의 `Fd::operator=`도 자기 대입을 막은 뒤 `reset()`과 `std::exchange` 순서로 처리한다.
- 시그널 처리기에서 직접 소켓과 컨테이너를 정리하면 왜 위험합니까?
  - 모범답변: `close` 일부를 제외한 컨테이너 조작과 대부분의 C++ 런타임 작업은 async-signal-safe하지 않으며, owner가 상태를 변경하는 도중 재진입하면 invariant도 깨질 수 있다. 일반적으로 처리기는 원자 플래그나 self-pipe만 건드리고 실제 `shutdown`은 owner 흐름에서 수행한다.
- 비차단 전송이 남아 있는 상태에서 종료할 때 "무한 대기하지 않으면서 가능한 만큼 전달"하려면 경계를 어디에 둡니까?
  - 모범답변: 새 입력과 새 연결을 먼저 막고, monotonic wall deadline과 짧은 nonblocking poll로 이미 큐에 든 전송만 제한적으로 flush한다. 이 프로젝트는 500ms 뒤 `SHUTDOWN_FLUSH_TIMEOUT`을 기록하고 연결을 정리하며 종료 중에는 simulation tick을 실행하지 않는다.
- 단순히 프로세스가 끝나면 커널이 파일 디스크립터를 회수한다는 이유로 명시적 정리를 생략하면 어떤 테스트와 운영 문제가 생깁니까?
  - 모범답변: 같은 프로세스에서 서버를 반복 생성하는 테스트는 FD·버퍼 누수를 드러내고, 정상 종료 지표나 flush 결과도 검증해야 한다. 커널의 프로세스 종료 회수에 기대면 장기 실행 중 재시작·부분 teardown에서 누수가 누적되고 `Fd::live`, `cleanup`, descriptor-close 검증도 신뢰할 수 없게 된다.

### 30초 모범 답변

파일 디스크립터는 단일 소유권을 갖는 자원이므로 복사를 막고 이동만 허용해야 이중 `close`와 누수를 구조적으로 방지할 수 있습니다. `reset`, 이동 대입, 소멸자는 기존 자원을 정확히 한 번 정리해야 하고, `release`만 소유권을 외부로 넘깁니다. 서버 종료는 재진입 가능하게 만들고, 새 연결 차단 → owner 경로의 남은 상태 처리 → 제한된 전송 시도 → 연결·reactor 해제 순서를 지켜야 합니다. 시그널 처리기는 플래그만 세우고 실제 정리는 일반 실행 흐름에서 해야 비동기 시그널 안전성과 컨테이너 invariant를 보존할 수 있습니다.

### 답변 핵심 키워드

단일 소유권, RAII, 이동 의미론, 정확히 한 번 해제, 멱등 종료, 비동기 시그널 안전, bounded flush, owner loop

### 백지 구현

#### 구현 목표

정수형 파일 디스크립터를 소유하는 최소 이동 전용 RAII 타입을 구현한다. 실제 `close` 호출은 주어진 콜백으로 대체해 단위 테스트가 가능하도록 한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class UniqueHandle {
public:
    using CloseFn = void (*)(int) noexcept;

    explicit UniqueHandle(int value = -1, CloseFn close_fn = nullptr) noexcept
        : value_(value), close_fn_(close_fn) {}
    ~UniqueHandle() noexcept { reset(); }

    UniqueHandle(UniqueHandle&& other) noexcept
        : value_(std::exchange(other.value_, -1)),
          close_fn_(std::exchange(other.close_fn_, nullptr)) {}

    UniqueHandle& operator=(UniqueHandle&& other) noexcept {
        if (this == &other) return *this;
        reset();
        value_ = std::exchange(other.value_, -1);
        close_fn_ = std::exchange(other.close_fn_, nullptr);
        return *this;
    }

    UniqueHandle(const UniqueHandle&) = delete;
    UniqueHandle& operator=(const UniqueHandle&) = delete;

    int get() const noexcept { return value_; }
    explicit operator bool() const noexcept { return value_ != -1; }

    int release() noexcept {
        return std::exchange(value_, -1);
    }

    void reset(int next = -1) noexcept {
        // 같은 정수값을 닫은 뒤 다시 소유한 척하지 않는다.
        if (next == value_) return;
        if (value_ != -1 && close_fn_ != nullptr) close_fn_(value_);
        value_ = next;
    }

private:
    int value_ = -1;
    // nullptr은 테스트에서 명시적으로 선택한 no-op 해제 정책이다.
    CloseFn close_fn_ = nullptr;
};
```

#### 입력과 출력

- 입력: 소유할 정수 핸들, 해제 콜백, 이동·`reset`·`release` 연산
- 출력: 현재 핸들 조회값과 해제 콜백의 호출 결과 관찰

#### 반드시 만족해야 할 조건

- 유효하지 않은 값 `-1`은 해제하지 않는다.
- 하나의 유효한 핸들은 소유 기간 전체에서 정확히 한 번만 해제된다.
- 이동 후 원본은 빈 상태가 된다.
- 이동 대입은 대상이 기존에 소유하던 핸들을 먼저 잃지 않도록 안전하게 정리한다.
- `release`는 해제하지 않고 소유권만 반환한다.
- 소멸자와 이동 연산은 예외를 던지지 않는다.

#### 경계 조건

- 빈 객체에서 `reset`, `release`, 이동
- 유효한 객체에 동일한 정수값으로 `reset`
- 자기 이동 대입 표현이 들어온 경우
- 유효한 대상에 빈 원본을 이동 대입하는 경우
- 여러 번 이어지는 이동

#### 실패 조건

- 해제 콜백이 없는데 유효한 핸들을 소유하도록 허용할지 명시해야 한다.
- 콜백이 예외를 던지는 형식은 허용하지 않는다.
- 같은 핸들을 두 객체가 동시에 소유하는 사용은 인터페이스 밖의 계약 위반으로 본다.

#### 필요한 제약

- 10~15분 범위로 제한한다.
- 실제 시스템 호출, 스레드 안전성, 참조 카운팅은 구현 범위에서 제외한다.

### 구현 후 자가 검증

- 정상 경로: 생성 → 조회 → 소멸 시 해제 1회
- 상태 변화: 이동 생성과 이동 대입 후 원본이 빈 상태인지 확인
- 경계값: 빈 객체의 `reset`·`release`가 안전한지 확인
- invariant: `생성된 유효 소유권 수 = 해제 수 + 현재 살아 있는 소유권 수`
- resource cleanup: 대상이 기존 핸들을 가진 이동 대입에서 기존 핸들이 누락되지 않는지 확인
- 중복 처리: 같은 객체를 여러 번 `reset()`해도 이중 해제가 없는지 확인
- 요구사항 충족: 복사 생성·복사 대입이 컴파일되지 않는지 확인

### 구현 후 설명할 것

1. 복사 금지와 이동 허용이 소유권 오류를 어떻게 컴파일 시점에 줄이는지
   - 모범답변: 복사를 삭제하면 동일 핸들을 소유하는 래퍼 두 개를 평범한 값 복사로 만들 수 없어 이중 `close` 경로를 차단한다. 이동만 허용하면 ownership transfer가 타입 연산으로 드러나고 이동 원본은 `-1`이 된다.
2. 이동 대입에서 기존 자원 정리와 원본 무효화의 순서
   - 모범답변: 자기 이동을 먼저 검사하고 대상의 기존 자원을 해제한 뒤 `std::exchange`로 원본 값을 가져온다. 그러면 대상 자원 누수와 원본·대상의 동시 소유를 모두 피할 수 있다.
3. `release`와 `reset`을 분리한 이유
   - 모범답변: `release`는 호출자에게 raw handle 소유권을 넘기므로 닫지 않고, `reset`은 현재 소유 자원을 닫고 새 값 또는 빈 상태로 바꾼다. 서로 다른 ownership 변화를 이름과 동작으로 분리한 것이다.
4. 소멸자를 `noexcept`로 유지해야 하는 이유
   - 모범답변: 스택 unwinding 중 소멸자가 예외를 던지면 `std::terminate`가 발생한다. OS 자원 해제 실패는 보통 반환할 복구 경로가 없으므로 해제 콜백 자체를 `noexcept` 계약으로 제한하고 별도 관측 수단을 둔다.
5. 이 타입이 `Server::shutdown`의 멱등성과 어떤 관계가 있는지
   - 모범답변: `shutdown`이 `stopping_`으로 재진입을 막고 각 `Fd::reset`이 소유 값을 빈 상태로 바꾸므로, 이후 소멸자가 다시 실행돼도 같은 FD를 닫지 않는다. 일반 원칙은 종료 단계와 RAII 소멸이 겹쳐도 해제가 정확히 한 번이어야 한다는 것이다.

### 원본 확인 위치

- Thread: G01
- 커밋 메시지: `feat: establish kqueue authoritative arena baseline`
- 파일: `src/transport.hpp`, `src/transport.cpp`, `src/game.cpp`, `src/game.hpp`
- 함수·클래스·컴포넌트: `Fd`, `Server::shutdown`, `Server::owned_descriptors`, `Server::cleanup`
- 관련 Thread: G03의 owner 정리 순서, G12의 이동 전용 outbound buffer 수명

---

<a id="w01-g02-parser"></a>

## [G02 / `feat: decode bounded incremental TCP frames`] 길이 접두사 TCP 스트림 파서

### 면접 질문

TCP에서 한 번의 `recv`가 한 메시지와 일치하지 않는 상황을 어떻게 처리했습니까? 부분 헤더, 부분 본문, 여러 프레임이 합쳐진 입력을 유한한 메모리로 처리하려면 파서 상태를 어떻게 잡아야 합니까?

꼬리 질문:

- 길이 필드를 읽은 직후 무엇을 검증해야 하며, 왜 본문 크기만큼 동적 할당하기 전에 해야 합니까?
  - 모범답변: unsigned big-endian 길이가 1..16,384인지 즉시 확인한다. peer가 선언한 거대 길이를 믿고 먼저 할당하면 단일 연결이 메모리를 소진할 수 있으므로, 이 프로젝트는 16,388바이트 고정 저장소 안에서 길이를 선검증한다.
- `consume`이 한 번에 한 결과와 소비 바이트 수를 반환하는 설계의 장점은 무엇입니까?
  - 모범답변: 파서는 완성된 한 프레임까지만 책임지고 호출자가 `consumed`만큼 suffix를 전진시킬 수 있다. 따라서 coalesced 입력도 추가 저장 없이 순서대로 처리하고, 오류·mailbox admission 같은 프레임별 정책을 사이에 적용할 수 있다.
- 정상 프레임 뒤에 다음 프레임 일부가 붙어 있다면 누가 suffix를 보존합니까?
  - 모범답변: 이번 호출에서 소비하지 않은 suffix는 호출자의 수신 버퍼에 남고, 호출자가 같은 read loop에서 다시 `consume`에 전달한다. 파서는 자신이 실제로 흡수한 부분 헤더나 부분 본문만 내부 저장소에 보존한다.
- EOF가 완전한 프레임 경계에서 온 경우와 헤더·본문 중간에 온 경우를 왜 구분합니까?
  - 모범답변: 내부 사용량이 0인 EOF는 정상 transport 종료지만, 일부 바이트가 남은 EOF는 완성될 수 없는 framing 위반이다. 이 프로젝트는 이를 `TRANSPORT_EOF`와 `TRANSPORT_EOF_IN_FRAME` 및 `partial_frame`으로 구분해 관측한다.
- 파서가 terminal 상태에 들어간 뒤 추가 바이트가 들어오면 어떻게 해야 합니까?
  - 모범답변: 추가 바이트를 소비하거나 새 메시지를 만들지 않고 동일 terminal 결과를 반환해야 한다. 길이 오류 이후에는 경계를 신뢰할 수 없어 재동기화를 시도하지 않는 것이 프로젝트의 정책이다.

### 30초 모범 답변

TCP는 바이트 스트림이라 수신 단위와 애플리케이션 프레임 경계가 일치하지 않습니다. 그래서 파서는 먼저 고정 크기 헤더를 채우고 길이를 즉시 검증한 뒤, 제한된 고정 저장소에 본문을 누적합니다. 완성된 프레임 하나와 실제 소비량을 반환하면 호출자가 같은 수신 버퍼의 나머지 바이트를 다시 넣어 coalesced 프레임을 순서대로 처리할 수 있습니다. 잘못된 길이는 메모리 할당 전에 terminal 오류로 만들고, 정상 경계 EOF와 부분 프레임 EOF를 분리해 프로토콜 오류와 정상 종료를 구분합니다.

### 답변 핵심 키워드

TCP는 스트림, fragmentation, coalescing, 길이 선검증, 고정 상한, 소비 바이트 수, suffix 재처리, terminal state, partial EOF

### 백지 구현

#### 구현 목표

4바이트 big-endian 길이 접두사와 최대 16,384바이트 payload를 사용하는 증분 파서를 구현한다. 한 호출은 최대 한 프레임 결과를 반환한다.

#### 인터페이스 또는 함수 시그니처

```cpp
enum class ParseState {
    NeedMore,
    Message,
    TerminalFrameError,
    IoEnd
};

struct ParseResult {
    ParseState state;
    std::size_t consumed;
    std::span<const std::byte> payload;
    std::string_view error_code;
    bool partial_frame;
};

class FrameParser {
public:
    static constexpr std::size_t kHeaderBytes = 4;
    static constexpr std::size_t kMaxPayloadBytes = 16'384;

    ParseResult consume(std::span<const std::byte> input) {
        if (terminal_) {
            auto result = *terminal_;
            result.consumed = 0;
            return result;
        }

        std::size_t consumed = 0;
        while (!input.empty()) {
            const auto count = std::min(expected_bytes_ - used_, input.size());
            std::copy_n(input.begin(), count, storage_.begin() + used_);
            used_ += count;
            consumed += count;
            input = input.subspan(count);

            if (used_ < expected_bytes_) break;
            if (expected_bytes_ == kHeaderBytes) {
                const auto length =
                    (std::to_integer<std::uint32_t>(storage_[0]) << 24U) |
                    (std::to_integer<std::uint32_t>(storage_[1]) << 16U) |
                    (std::to_integer<std::uint32_t>(storage_[2]) << 8U) |
                    std::to_integer<std::uint32_t>(storage_[3]);
                if (length == 0 || length > kMaxPayloadBytes) {
                    used_ = 0;
                    expected_bytes_ = kHeaderBytes;
                    terminal_ = ParseResult{ParseState::TerminalFrameError,
                                            consumed, {}, "FRAME_SIZE_INVALID", false};
                    return *terminal_;
                }
                expected_bytes_ = kHeaderBytes + static_cast<std::size_t>(length);
                continue;
            }

            const auto payload_size = expected_bytes_ - kHeaderBytes;
            const auto payload = std::span<const std::byte>(storage_).subspan(
                kHeaderBytes, payload_size);
            used_ = 0;
            expected_bytes_ = kHeaderBytes;
            // payload view는 다음 parser 변경 전까지만 유효하다.
            return {ParseState::Message, consumed, payload, {}, false};
        }
        return {ParseState::NeedMore, consumed, {}, {}, false};
    }

    ParseResult finish(bool io_error) {
        if (terminal_ && !io_error) {
            auto result = *terminal_;
            result.consumed = 0;
            return result;
        }
        const bool partial = used_ != 0;
        used_ = 0;
        expected_bytes_ = kHeaderBytes;
        terminal_ = ParseResult{
            ParseState::IoEnd,
            0,
            {},
            io_error ? "TRANSPORT_IO_ERROR"
                     : partial ? "TRANSPORT_EOF_IN_FRAME" : "TRANSPORT_EOF",
            partial};
        return *terminal_;
    }

    std::size_t buffered_bytes() const noexcept { return used_; }

private:
    std::array<std::byte, kHeaderBytes + kMaxPayloadBytes> storage_{};
    std::size_t used_ = 0;
    std::size_t expected_bytes_ = kHeaderBytes;
    std::optional<ParseResult> terminal_;
};
```

#### 입력과 출력

- 입력: 임의 크기로 잘린 TCP 바이트 조각
- 출력: 더 필요함, 완성된 payload, terminal framing 오류, 또는 I/O 종료
- `consumed`는 이번 호출에서 실제로 흡수한 바이트 수다.

#### 반드시 만족해야 할 조건

- 4바이트 헤더가 완성되기 전에는 길이를 해석하지 않는다.
- 길이 0 또는 최대값 초과는 즉시 terminal 오류다.
- 길이 검증 전 payload 크기 기반 동적 할당을 하지 않는다.
- 내부 저장 공간은 헤더 포함 최대 16,388바이트를 넘지 않는다.
- 입력에 두 프레임이 붙어 있어도 첫 프레임까지만 소비하고 정확한 `consumed`를 반환한다.
- 부분 헤더와 부분 본문을 다음 호출까지 보존한다.
- terminal 오류 뒤의 `consume`은 상태를 되돌리지 않는다.
- `finish(false)`는 빈 경계 EOF와 부분 프레임 EOF를 구분한다.
- `finish(true)`는 I/O 오류를 우선 분류한다.

#### 경계 조건

- 헤더가 1·2·3·4바이트씩 나뉘어 도착
- payload가 한 바이트씩 도착
- 헤더와 payload, 다음 헤더가 한 번에 도착
- 정확히 16,384바이트 payload
- 길이 0, 16,385, `uint32_t` 최대값
- 완성 직후 EOF, 헤더 1~3바이트 뒤 EOF, 본문 한 바이트 부족한 EOF
- 빈 입력 반복 호출

#### 실패 조건

- 잘못된 길이 이후 프레임 재동기화를 시도하지 않는다.
- I/O 종료 후 새로운 메시지를 산출하지 않는다.
- payload 검증이나 JSON 파싱은 이 문제 범위에서 제외하고, 완성된 바이트만 반환한다.

#### 필요한 제약

- 20~30분 범위다.
- `std::vector`를 매 프레임마다 늘리는 방식은 허용하지 않는다.
- 반환 payload의 유효 기간은 다음 파서 변경 전까지라고 명시한다.

### 구현 후 자가 검증

- 정상 경로: 단일 완전 프레임이 정확한 payload를 반환하는지 확인
- 경계값: 최대 payload와 최대+1을 구분하는지 확인
- 상태 변화: 헤더 단계 → 본문 단계 → 초기 단계 전환이 정확한지 확인
- 중복·누락 처리: coalesced 두 프레임에서 첫 호출 소비량과 두 번째 호출 입력 시작점 확인
- 실패 경로: 길이 오류 후 추가 입력이 메시지로 복구되지 않는지 확인
- invariant: `0 <= buffered_bytes <= 16,388`, 기대 길이보다 사용량이 커지지 않는지 확인
- 시간·공간 복잡도: 각 입력 바이트가 상수 횟수만 복사되고 공간이 상한 내인지 확인
- 요구사항 충족: 부분 EOF와 정상 EOF의 오류 코드가 구분되는지 확인

### 구현 후 설명할 것

1. 왜 TCP 수신 횟수와 프레임 수를 연결하면 안 되는지
   - 모범답변: TCP는 메시지가 아니라 순서 있는 바이트 스트림을 제공하므로 한 frame이 여러 `recv`로 쪼개지거나 여러 frame이 한 `recv`에 합쳐질 수 있다. 따라서 애플리케이션 길이 prefix로 경계를 복원해야 한다.
2. 한 결과만 반환하고 소비량을 노출한 이유
   - 모범답변: 호출자가 suffix를 보존한 채 프레임마다 mailbox·오류 정책을 적용할 수 있고, 파서가 coalesced 결과 vector를 별도로 보유하지 않아 메모리 상한도 단순해진다.
3. 잘못된 길이에서 stream 재동기화를 포기한 이유
   - 모범답변: 신뢰할 수 없는 길이 뒤의 어느 바이트가 다음 헤더인지 알 수 없으므로 임의 검색은 정상 payload를 헤더로 오인할 수 있다. 이 프로젝트는 해당 peer만 terminal로 닫아 오류를 격리한다.
4. 고정 저장소와 동적 저장소의 trade-off
   - 모범답변: 고정 저장소는 연결당 16,388바이트를 예약하지만 allocation churn과 peer 선언 길이에 따른 폭증을 막고 상한이 명확하다. 동적 저장소는 작은 frame의 평균 메모리는 줄일 수 있으나 성장·실패·회수 정책이 더 복잡하다.
5. payload 파싱 오류와 framing 오류를 같은 상태로 만들지 않은 이유
   - 모범답변: 완성 frame의 JSON·schema 오류 뒤에는 다음 길이 경계가 여전히 확정돼 복구할 수 있지만, 길이 오류는 이후 경계를 잃는다. 전자는 메시지만 거부하고 후자는 연결을 닫는 정책 차이를 상태에 보존해야 한다.

### 원본 확인 위치

- Thread: G02
- 커밋 메시지: `feat: decode bounded incremental TCP frames`
- 파일: `src/transport.cpp`, `src/transport.hpp`
- 함수·클래스·컴포넌트: `FrameParser`, `FrameParser::consume`, `FrameParser::finish`, `ParseResult`, `ParseState`, `decode_complete_frame`
- 관련 Thread: G01의 bounded transport, G03의 mailbox 전달, G09의 UDP datagram 경계

---

<a id="w01-g02-errors"></a>

## [G02 / `feat: decode bounded incremental TCP frames`] 복구 가능한 메시지 오류와 terminal 전송 오류 분리

### 면접 질문

잘못된 JSON 또는 지원하지 않는 메시지와 잘못된 프레임 길이를 왜 다르게 처리했습니까? 한 클라이언트의 오류가 다른 연결이나 이후 정상 프레임에 영향을 주지 않게 하려면 경계를 어떻게 나눠야 합니까?

꼬리 질문:

- 중복 JSON key를 DOM 생성 후 확인하면 왜 늦을 수 있습니까?
  - 모범답변: 많은 DOM parser는 같은 key의 이전 값을 덮어써 원래 중복 사실을 잃는다. 이 프로젝트는 parse callback에서 object별 key 집합을 관리해 DOM이 완성되기 전에 중복을 거부한다.
- raw NUL, invalid UTF-8, NaN/Infinity, trailing object를 어디에서 거부해야 합니까?
  - 모범답변: 완성된 frame payload를 strict JSON decoder에 넘기는 메시지 경계에서 거부한다. 프로젝트는 raw NUL을 lexer 전에 검사하고 JSON parser가 invalid UTF-8·비표준 숫자·trailing input을 거부하게 하므로 framing 동기화는 유지된다.
- 오류 메시지에 파서 라이브러리의 원문 예외 문자열을 그대로 넣으면 어떤 문제가 생깁니까?
  - 모범답변: 예외에는 잘못된 UTF-8, 입력 조각, 라이브러리 버전별 세부정보가 들어가 정보 노출과 wire 비결정성을 만들 수 있다. 프로젝트는 외부에는 `MESSAGE_INVALID` 같은 안정된 allow-list 코드만 보내고 상세 진단은 내부 관측으로 제한한다.
- framing 오류를 알리기 위한 전송 자체가 `EAGAIN` 또는 실패를 만났을 때 무엇을 보장해야 합니까?
  - 모범답변: 오류 frame을 한 번 bounded nonblocking 방식으로 시도하되 완료를 무한 대기하지 않고 해당 peer를 닫아야 한다. 오류 전달 성공보다 서버 메모리 상한과 다른 연결의 진행 보장이 우선이다.
- 이미 framing 실패를 기록한 뒤 소켓 I/O 오류까지 발생하면 관측값을 어떻게 중복 분류하지 않습니까?
  - 모범답변: connection에 최초 terminal reason 또는 응답 시도 상태를 latch하고 후속 종료는 같은 원인에 대한 새 protocol error를 만들지 않는다. 이 프로젝트의 parser도 terminal 상태를 보존하고 종료 경로가 peer 단위로 한 번 정리되게 한다.

### 30초 모범 답변

JSON 문법·스키마 오류는 프레임 경계가 살아 있으므로 해당 메시지만 거부하고 같은 연결의 다음 프레임을 계속 처리할 수 있습니다. 반면 길이 오류나 부분 프레임 종료는 이후 바이트의 경계를 신뢰할 수 없으므로 terminal로 만들고 그 peer만 닫아야 합니다. 중복 key와 raw NUL은 DOM이 값을 덮어쓰거나 lexer가 조기 종료하기 전에 검사하고, 외부 입력이 섞인 예외 원문은 wire에 재전송하지 않습니다. 오류 flush도 횟수와 버퍼를 제한하며, 실패해도 다른 연결과 owner 상태는 보존합니다.

### 답변 핵심 키워드

메시지 오류와 framing 오류, stream synchronization, peer isolation, strict decoding, duplicate key, raw NUL, 안전한 오류 코드, bounded error flush

### 백지 구현

#### 구현 목표

파서 결과를 받아 owner mailbox 전달, 복구 가능한 오류 응답, terminal peer 종료 중 하나로 변환하는 작은 정책 계층을 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
enum class ConnectionActionKind {
    EnqueueMessage,
    EnqueueRecoverableError,
    AttemptTerminalErrorAndClose,
    CloseWithoutProtocolReply,
    NoAction
};

struct ConnectionAction {
    ConnectionActionKind kind;
    std::string_view stable_error_code;
    std::size_t consumed;
};

ConnectionAction decide_connection_action(
    const ParseResult& result,
    bool protocol_error_reply_already_attempted);
```

#### 입력과 출력

- 입력: 증분 파서의 한 결과와 terminal 오류 응답 시도 여부
- 출력: 연결 계층이 수행할 단일 명시적 action

#### 반드시 만족해야 할 조건

- 완성된 메시지는 bounded owner mailbox로 전달하는 action이 된다.
- 메시지 수준 오류는 연결을 닫지 않고 안정된 오류 코드만 반환한다.
- framing 오류는 이후 메시지 처리를 금지하고 해당 peer만 종료 대상으로 만든다.
- I/O 오류는 프로토콜 응답 성공 여부와 무관하게 종료한다.
- 같은 terminal 원인에 대해 오류 응답을 무한히 재시도하지 않는다.
- 외부 입력 또는 라이브러리 예외 원문을 오류 코드로 반환하지 않는다.

#### 경계 조건

- recoverable 오류 다음에 정상 메시지가 오는 경우
- terminal framing 오류 직후 I/O 오류가 겹치는 경우
- 오류 응답을 이미 한 번 시도한 경우
- 빈 EOF와 부분 프레임 EOF
- mailbox가 가득 찬 경우는 호출자 정책으로 명시적으로 반환하도록 확장 가능해야 한다.

#### 실패 조건

- terminal 상태에서 `EnqueueMessage`를 반환하면 실패다.
- recoverable 오류 하나 때문에 서버 전체 또는 다른 peer를 종료하면 실패다.
- 동일 입력에서 두 개 이상의 상충 action을 만들면 실패다.

#### 필요한 제약

- 10분 내외의 정책 함수로 제한한다.
- 실제 JSON 파서, 소켓 write, mailbox 구현은 제공된다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: 메시지 결과가 owner 전달 action인지 확인
- 실패 경로: recoverable과 terminal의 연결 유지 여부가 반대가 되지 않는지 확인
- 상태 변화: terminal 응답 1회 시도 후 재호출이 다시 버퍼를 만들지 않는지 확인
- 중복 처리: framing 오류와 후속 I/O 오류가 이중 메시지 전송을 만들지 않는지 확인
- 보안: 출력 오류 코드가 정해진 allow-list 밖의 문자열을 포함하지 않는지 확인
- 요구사항 충족: 한 peer의 오류가 전역 종료 action으로 변환되지 않는지 확인

### 구현 후 설명할 것

1. 오류 분류 기준을 "복구 가능한 stream 경계인가"로 잡은 이유
   - 모범답변: 다음 frame 시작 위치를 확실히 알면 현재 메시지만 버리고 계속할 수 있지만, 알 수 없으면 이후 데이터의 의미를 안전하게 해석할 수 없다. 그래서 message error와 terminal framing error의 연결 유지 정책이 달라진다.
2. 상세 진단은 로그에 남기고 wire에는 안정된 코드만 주는 이유
   - 모범답변: wire code는 클라이언트 계약이므로 버전과 입력 내용에 흔들리지 않아야 하고 민감하거나 비정상인 parser 원문을 노출해서도 안 된다. 내부 로그는 접근 통제와 escaping을 적용해 별도로 진단할 수 있다.
3. terminal 오류 응답을 best-effort·bounded로 제한한 이유
   - 모범답변: 이미 비정상인 peer가 writable하지 않을 수 있어 응답 완료를 기다리면 DoS 경로가 된다. 프로젝트는 한 번의 제한된 flush 뒤 연결을 닫아 자원 회수를 보장한다.
4. parser, connection, owner mailbox 사이 책임 분리
   - 모범답변: parser는 바이트 경계와 메시지 유효성 결과를 만들고, connection은 응답·종료와 I/O ownership을 처리하며, owner mailbox 이후에서만 Room 상태를 변경한다. 이 분리로 한 peer의 transport 오류가 authoritative state나 다른 peer로 번지지 않는다.

### 원본 확인 위치

- Thread: G02
- 커밋 메시지: `feat: decode bounded incremental TCP frames`
- 파일: `src/transport.cpp`, `src/transport.hpp`
- 함수·클래스·컴포넌트: `parse_object`, `request_error`, `FrameParser::consume`, `FrameParser::finish`, `Server::read_ready`, `Server::end_transport`
- 관련 Thread: G03의 owner mailbox, G12의 bounded outbound와 종료 정책

---

<a id="w01-g03"></a>

## [G03 / 기록 제목: Connection, session, player and room ownership] 단일 owner mailbox와 식별자·상태 invariant

### 면접 질문

Connection, session, player, room 식별자를 분리한 이유는 무엇입니까? 네트워크 콜백이 `Room`을 직접 수정하지 않고 bounded mailbox를 거쳐 owner가 처리하도록 한 설계는 어떤 문제를 해결합니까?

꼬리 질문:

- 연결이 끊어져도 player와 session을 같은 객체로 취급하면 어떤 lifecycle 버그가 생깁니까?
  - 모범답변: transport가 끊기는 순간 논리 player까지 제거돼 reconnect grace 동안 보존해야 할 위치·점수·sequence를 잃거나, 반대로 새 connection이 old transport 자원을 계속 참조할 수 있다. 프로젝트는 connection, stable session, player 상태를 별도 수명으로 관리한다.
- 전체 mailbox 512개와 연결별 64개 제한을 동시에 둔 이유는 무엇입니까?
  - 모범답변: 전역 한도는 인스턴스 총 메모리와 owner 작업량을 제한하고, 연결별 한도는 한 peer가 전체 용량을 독점하지 못하게 한다. 프로젝트 상수에서는 전체 mailbox 512와 connection별 pending 64가 이 두 실패 형태를 각각 막는다.
- 큐가 가득 찬 요청은 per-source count와 FIFO 순서에 어떤 영향을 주어야 합니까?
  - 모범답변: admission 실패는 commit 전 상태이므로 count와 queue 모두 그대로여야 한다. 성공한 항목의 FIFO만 보존되고, 거부 항목이 빈 자리나 순번을 예약해서는 안 된다.
- `poll_io`와 `drain_mailbox`를 분리하면 테스트에서 어떤 경계를 검증할 수 있습니까?
  - 모범답변: I/O를 poll한 직후에는 envelope만 대기하고 Room snapshot은 바뀌지 않았으며, owner drain 뒤에만 상태가 변하는지를 검증할 수 있다. 이는 network callback과 authoritative mutation 사이의 명시적 경계다.
- owner thread가 아닌 곳에서 `Room` 변경을 시도했을 때 조용히 직렬화하는 대신 거부한 이유는 무엇입니까?
  - 모범답변: 잘못된 호출을 내부에서 몰래 enqueue하면 호출 시점·예외·순서 계약이 달라져 race가 숨는다. 프로젝트의 `Room::assert_owner`는 즉시 `logic_error`를 내 테스트에서 ownership 위반을 원인 가까이 드러낸다.
- ABSENT, LOBBY, RUNNING, FINISHED, CLOSED에서 CREATE/JOIN/LEAVE의 허용 범위를 어떻게 설명하겠습니까?
  - 모범답변: 프로젝트에서는 ABSENT에서만 CREATE로 LOBBY가 되고 JOIN은 LOBBY에서 정원 안일 때만 가능하다. LEAVE는 참가자의 연결·pending을 정리하는 owner 연산이며, RUNNING/FINISHED에서도 기존 참가자 정리에 쓰일 수 있지만 CLOSED에서는 새 생성·참가·게임 mutation을 허용하지 않는다.

### 30초 모범 답변

Connection은 전송 수명, session은 인증된 논리 접속, player는 게임 참가자, room은 상태 소유 경계라서 서로 교체 가능한 식별자가 아닙니다. I/O 콜백은 검증된 envelope만 bounded mailbox에 넣고, owner가 FIFO로 drain할 때만 `Room`을 바꾸면 경쟁 조건과 중간 상태 노출을 줄일 수 있습니다. 전역 상한은 서버 메모리를, 연결별 상한은 한 peer의 독점을 막습니다. 거부된 항목은 큐나 카운터를 바꾸지 않아야 하고, lifecycle별 허용 연산과 소유 관계 검증이 상태 변경 전에 끝나야 합니다.

### 답변 핵심 키워드

식별자 수명 분리, single writer, owner thread, bounded MPSC 경계, FIFO, global/per-source cap, state machine, mutation-before-validation 금지

### 백지 구현

#### 구현 목표

전역 용량과 source별 용량을 함께 지키는 FIFO mailbox를 구현한다. 생산자는 source ID와 payload를 넣고, 단일 owner가 꺼낸다.

#### 인터페이스 또는 함수 시그니처

```cpp
template <typename SourceId, typename T>
class BoundedMailbox {
public:
    BoundedMailbox(std::size_t global_capacity,
                   std::size_t per_source_capacity)
        : global_capacity_(global_capacity),
          per_source_capacity_(per_source_capacity) {}

    bool try_push(const SourceId& source, T value) {
        const auto found = counts_.find(source);
        const auto source_size = found == counts_.end() ? 0 : found->second;
        // 모든 한도를 확인한 뒤에만 queue와 보조 count를 함께 commit한다.
        if (queue_.size() >= global_capacity_ ||
            source_size >= per_source_capacity_) return false;
        queue_.emplace_back(source, std::move(value));
        ++counts_[source];
        return true;
    }

    std::optional<std::pair<SourceId, T>> pop() {
        if (queue_.empty()) return std::nullopt;
        auto item = std::move(queue_.front());
        queue_.pop_front();
        auto found = counts_.find(item.first);
        if (--found->second == 0) counts_.erase(found);
        return item;
    }

    std::size_t size() const noexcept { return queue_.size(); }

    std::size_t size_for(const SourceId& source) const {
        const auto found = counts_.find(source);
        return found == counts_.end() ? 0 : found->second;
    }

    bool empty() const noexcept { return queue_.empty(); }

private:
    std::size_t global_capacity_;
    std::size_t per_source_capacity_;
    std::deque<std::pair<SourceId, T>> queue_;
    std::unordered_map<SourceId, std::size_t> counts_;
};
```

#### 입력과 출력

- 입력: source ID와 이동 가능한 메시지
- 출력: admission 성공 여부, owner가 받는 FIFO 항목

#### 반드시 만족해야 할 조건

- 전체 항목 수는 전역 capacity를 넘지 않는다.
- source별 항목 수는 per-source capacity를 넘지 않는다.
- 성공한 항목의 전역 FIFO 순서는 보존된다.
- 거부된 `try_push`는 메시지 순서, 전체 크기, source count를 바꾸지 않는다.
- `pop`은 해당 source count를 정확히 감소시킨다.
- 마지막 항목이 빠진 source의 보조 상태를 정리한다.
- payload는 불필요하게 복사하지 않는다.

#### 경계 조건

- capacity 0
- 한 source가 per-source 한도까지 채운 뒤 다른 source가 넣는 경우
- 전역 한도는 찼지만 특정 source 한도는 남은 경우
- 같은 source의 거부 후 다시 `pop`하고 재시도하는 경우
- source ID가 처음 등장하거나 마지막 항목이 제거되는 경우
- 512번째 성공과 513번째 거부에 해당하는 일반화된 경계

#### 실패 조건

- 거부된 항목 때문에 source count가 증가하면 실패다.
- `pop` 순서가 source별 round-robin으로 바뀌면 요구사항 위반이다.
- 비어 있는 mailbox에서 임의 값을 반환하면 실패다.

#### 필요한 제약

- 실제 동시성 primitive는 구현하지 않는다. 생산·소비 호출이 외부에서 직렬화된다고 가정한다.
- 15~20분 범위로 제한한다.
- owner thread 검사는 별도의 래퍼가 수행한다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: 여러 source가 섞인 성공 입력을 원래 순서대로 꺼내는지 확인
- 경계값: 전역·source별 정확한 마지막 허용 항목과 다음 거부 확인
- 실패 경로: 거부 전후 모든 size가 동일한지 확인
- 상태 변화: `pop` 후 해당 source가 다시 admission 가능한지 확인
- invariant: 전체 size가 source count 합과 항상 같은지 확인
- resource cleanup: 모두 pop한 뒤 큐와 source count 보조 저장소가 비는지 확인
- 동시성·비동기 문제: 구현 자체보다 single-owner 전제가 어디에서 강제되는지 설명 가능한지 확인
- 시간·공간 복잡도: `try_push`, `pop`, `size_for`의 목표 복잡도를 설명

### 구현 후 설명할 것

1. 전역 한도와 source별 한도가 막는 실패 형태의 차이
   - 모범답변: 전역 한도는 여러 source를 합친 메모리·owner work 폭증을 막고, source별 한도는 한 source가 그 전역 예산을 독점하는 것을 막는다. 두 검사를 모두 통과한 뒤에만 admission을 commit해야 한다.
2. FIFO를 유지하면서 source별 count를 관리한 자료구조 선택
   - 모범답변: 성공 항목은 하나의 `deque`에 넣어 전역 FIFO를 유지하고, `unordered_map<SourceId, size_t>`는 source별 admission count만 보조한다. 평균적으로 push/pop/count 조회가 O(1)이며 마지막 항목을 pop하면 map entry도 지운다.
3. 네트워크 콜백과 도메인 mutation을 분리한 이유
   - 모범답변: callback은 bounded envelope 생성까지만 맡고 Room mutation은 owner drain에 모으면 명령 순서와 tick 경계를 직렬화할 수 있다. 프로젝트의 `poll_io`와 `drain_mailbox` 분리가 바로 이 경계다.
4. 식별자별 수명과 소유 관계를 상태 변경 전에 검증하는 순서
   - 모범답변: 실제 connection을 찾고 그 connection에 귀속된 session, room, player를 요청 claim과 차례로 대조한 뒤 lifecycle 허용 여부를 확인하고 마지막에 mutation한다. 실패 전에 state나 quota를 바꾸지 않는 것이 핵심이다.
5. owner thread 위반을 예외 또는 assertion으로 드러내는 trade-off
   - 모범답변: 예외는 테스트와 운영 진단에 명시적으로 남지만 처리 비용과 복구 정책이 필요하고, assertion은 개발 중 빠르게 실패하나 release build에서 사라질 수 있다. 프로젝트는 런타임 `logic_error`로 위반을 계속 드러낸다.

### 원본 확인 위치

- Thread: G03
- 기록 제목: Connection, session, player and room ownership (Conventional Commit subject는 현재 프로젝트 기록에서 확인되지 않음)
- 파일: `src/transport.hpp`, `src/transport.cpp`, `src/game.hpp`, `src/game.cpp`, `src/lifecycle_scenario.cpp`
- 함수·클래스·컴포넌트: `Server::Mailbox`, `Server::poll_io`, `Server::drain_mailbox`, `Room`, `LifecycleFixture`, `Connection`
- 관련 Thread: G01의 자원 종료, G04의 owner scheduler, G11의 reconnect 시 식별자 승계, G13의 Room별 소유 문맥
