# 시간과 자원 보호

이 문서는 외부 입력을 안전하게 수치로 바꾸는 방법, 단조 시계를 사용하는 연결 수명 관리, 명령 호출 제한, 연결별·서버 전체 자원 상한을 다룬다.

## [Thread 11 / `fix(config): 서버 크기 옵션을 오버플로 없이 해석` · `test(config): 크기 옵션 경계와 오류 입력 검증`] P11. 오버플로 안전 십진수 파서

### 면접 질문

`--max-pending-bytes`나 port를 문자열에서 정수로 바꿀 때 `strtoul`이나 `stoull`을 호출한 뒤 범위만 검사하면 충분합니까? 이 프로젝트가 부호·공백·빈 문자열과 목적 타입의 최댓값을 명시적으로 검사한 이유를 설명해 보세요.

꼬리 질문:

- `-1`을 unsigned로 변환하면 왜 위험합니까?
- `errno` 기반 함수는 어떤 호출 전후 규칙을 지켜야 합니까?
- 값이 목적 타입을 이미 넘은 뒤에 검사하면 어떤 문제가 생깁니까?
- port와 byte limit이 같은 parser를 공유하면서 서로 다른 허용 범위를 어떻게 표현합니까?
- decimal digit 하나씩 누적할 때 overflow가 일어나기 전에 어떻게 판정할 수 있습니까?

### 30초 모범 답변

외부 설정은 문자열 문법과 수치 범위를 함께 검증해야 합니다. signed나 더 큰 임시 타입에 변환한 뒤 cast하면 음수 wrap이나 중간 타입 overflow, 플랫폼별 폭 차이를 놓칠 수 있습니다. 안전한 방식은 빈 문자열·부호·공백·비숫자를 먼저 거절하고, 각 digit을 누적하기 전에 다음 곱셈과 덧셈이 지정된 maximum을 넘는지 확인하는 것입니다. 같은 함수에 목적 상한을 인자로 넘기면 port의 65535 같은 도메인 범위와 `size_t` 최대 범위를 같은 원리로 처리할 수 있습니다.

### 답변 핵심 키워드

입력 문법 검증, unsigned wrap, 목적 타입 상한, 변환 전 overflow 검사, digit accumulation, 플랫폼 폭, 부호·공백 거절, 도메인 범위

### 백지 구현

**구현 목표**

ASCII 십진수 문자열을 지정된 unsigned 타입과 최대값 안에서 파싱한다. 실패 이유를 구분해 반환한다.

**면접용 축소 인터페이스**

```cpp
#include <string_view>

enum class ParseUnsignedError {
    None,
    Empty,
    NonDigit,
    Overflow,
    OutOfDomain,
};

template <typename Unsigned>
struct ParseUnsignedResult {
    Unsigned value{};
    ParseUnsignedError error = ParseUnsignedError::None;

    bool ok() const { return error == ParseUnsignedError::None; }
};

template <typename Unsigned>
ParseUnsignedResult<Unsigned> parseUnsignedDecimal(
    std::string_view text,
    Unsigned maximum,
    bool allowZero);
```

**입력과 출력**

- 입력: 원문 문자열, 허용 최댓값, 0 허용 여부
- 출력: 성공 값 또는 실패 종류

**반드시 만족해야 할 조건**

- 빈 문자열을 거절한다.
- `+`, `-`, leading/trailing 공백을 숫자로 인정하지 않는다.
- 모든 문자가 ASCII `0`~`9`인지 검사한다.
- 목적 타입 연산에서 overflow가 실제 발생하기 전에 거절한다.
- `maximum`을 넘는 값과 0 금지 정책을 구분한다.
- 성공 시 문자열 전체를 소비한다.
- `Unsigned`가 실제 unsigned 타입이라는 제약을 둔다.

**경계 조건**

- `"0"`, `"00"`, `"1"`
- 빈 문자열
- `"+1"`, `"-1"`, `" 1"`, `"1 "`
- 중간에 문자가 섞인 입력
- 정확히 maximum
- maximum+1을 나타내는 문자열
- 목적 타입의 최대 digit 수보다 훨씬 긴 문자열
- allowZero false에서 0

**실패 조건**

- 문법 오류
- 산술 overflow 가능성
- 도메인 범위 위반

**제약**

- 15~20분 안에 구현한다.
- 예외를 사용하지 않는다.
- `strto*`, `std::stoi`, `std::from_chars`를 사용하지 않는다.
- 누적 중 더 넓은 정수 타입이 항상 존재한다고 가정하지 않는다.

```cpp
template <typename Unsigned>
ParseUnsignedResult<Unsigned> parseUnsignedDecimal(
    std::string_view text,
    Unsigned maximum,
    bool allowZero) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] maximum과 maximum+1이 정확히 구분되는가?
- [ ] 목적 타입 최대보다 훨씬 긴 입력에서도 wrap하지 않는가?
- [ ] 음수와 `+`가 거절되는가?
- [ ] 공백을 암묵적으로 건너뛰지 않는가?
- [ ] non-digit이 앞·중간·끝에 있을 때 모두 실패하는가?
- [ ] allowZero 정책이 overflow·문법 오류와 독립적인가?
- [ ] `unsigned short`, `std::size_t` 등 서로 다른 폭에서 같은 계약을 만족하는가?
- [ ] 시간 복잡도 O(N), 공간 복잡도 O(1)인가?
- [ ] 실패 시 부분 누적값을 성공처럼 사용하지 못하게 했는가?

### 구현 후 설명할 것

- 라이브러리 변환 함수 대신 직접 누적하는 이유
- overflow 판정을 실제 산술 전에 수행하는 원리
- 숫자 문법과 도메인 범위를 분리한 이유
- port, timeout, byte limit에 같은 parser를 재사용하는 방식
- 오류 enum과 예외 중 선택한 trade-off

### 원본 확인 위치

- Thread 11
- 커밋: `feat(config): 기본 실행 인자 해석 모듈 구성`, `fix(config): 서버 크기 옵션을 오버플로 없이 해석`, `test(config): 크기 옵션 경계와 오류 입력 검증`
- 파일: `src/RuntimeConfig.hpp`, `src/RuntimeConfig.cpp`, `tests/irc_contract.py`
- 함수: `RuntimeConfig::parsePort`, `RuntimeConfig::parseOptions`, `RuntimeConfig::parseSize`, `RuntimeConfig::parsePositiveInt`, `parseUnsignedDecimal`
- 관련 Thread: 13, 14

---

## [Thread 12 / `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` · `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`] P12. 단조 시계 기반 timeout·heartbeat 상태 기계

### 면접 질문

유휴 연결에 PING을 보내고 일정 시간 안에 PONG이 없으면 끊는 기능에서 왜 `system_clock`보다 `steady_clock`이 적절합니까? 또 PONG을 받았다는 사실만으로 대기 상태를 해제하지 않고 pending token을 비교해야 하는 이유는 무엇입니까?

꼬리 질문:

- 서버가 PING을 queue하려는 순간 queue 실패로 연결이 제거되면 heartbeat 상태를 언제 변경해야 합니까?
- 일반 명령 수신을 PONG과 같은 생존 증거로 볼지 어떻게 정합니까?
- registration timeout과 idle timeout이 동시에 만족되면 어떤 규칙이 우선합니까?
- 이전 heartbeat의 늦은 PONG이 새 heartbeat를 해제하면 어떤 문제가 생깁니까?
- tick 주기가 timeout보다 거칠면 실제 종료 시각은 어떻게 달라집니까?

### 30초 모범 답변

timeout은 경과 시간을 재므로 wall clock 조정의 영향을 받지 않는 `steady_clock`을 사용해야 합니다. 상태는 connected 시각, 마지막 활동, PING 전송 시각, 현재 대기 token을 가집니다. 등록 전 연결은 registration timeout을 먼저 적용하고, 등록 후 유휴 시간이 지나면 고유 token의 PING을 한 번만 보냅니다. PONG은 현재 pending token과 일치할 때만 대기를 해제해야 늦거나 임의의 응답이 timeout을 우회하지 못합니다. tick 기반 구현의 종료 시각에는 최대 tick 간격 정도의 지연이 있을 수 있음을 보장 범위에 포함해야 합니다.

### 답변 핵심 키워드

`steady_clock`, elapsed time, registration timeout, idle threshold, ping deadline, pending token, stale PONG, one outstanding probe, tick granularity, send failure boundary

### 백지 구현

**구현 목표**

등록 대기와 유휴 heartbeat를 관리하는 연결 하나의 상태 기계를 구현한다. network 송신 대신 action을 반환한다.

**면접용 축소 인터페이스**

```cpp
#include <chrono>
#include <optional>
#include <string>

using Clock = std::chrono::steady_clock;
using TimePoint = Clock::time_point;

struct HeartbeatConfig {
    std::chrono::seconds registrationTimeout;
    std::chrono::seconds idleTimeout;
    std::chrono::seconds pongTimeout;
};

struct HeartbeatState {
    bool registered = false;
    TimePoint connectedAt{};
    TimePoint lastActivityAt{};
    std::optional<TimePoint> pingSentAt;
    std::optional<std::string> pendingToken;
};

enum class HeartbeatActionKind {
    None,
    SendPing,
    CloseRegistrationTimeout,
    ClosePingTimeout,
};

struct HeartbeatAction {
    HeartbeatActionKind kind;
    std::string token;
};

HeartbeatAction onHeartbeatTick(
    HeartbeatState& state,
    const HeartbeatConfig& config,
    TimePoint now,
    const std::string& nextToken);

bool onPong(
    HeartbeatState& state,
    const std::string& token,
    TimePoint now);

void recordActivity(HeartbeatState& state, TimePoint now);
```

**입력과 출력**

- 입력: 연결 상태, timeout 설정, 단조 시각, 새 probe token 또는 PONG token
- 출력: 아무 일 없음, PING 요청, registration timeout close, PONG timeout close
- 상태 변화: 활동 시각과 outstanding probe

**반드시 만족해야 할 조건**

- 등록 전 timeout과 등록 후 heartbeat 정책을 구분한다.
- outstanding PING은 동시에 하나만 존재한다.
- idle threshold 전에는 PING을 만들지 않는다.
- pending token과 일치하는 PONG만 대기를 해제한다.
- stale·임의 token은 상태를 변경하지 않는다.
- PONG timeout은 PING을 보낸 시각을 기준으로 한다.
- 시각 역행을 가정하지 않되, 입력이 non-monotonic일 때의 방어 정책을 설명한다.
- action 반환 후 실제 send 실패가 발생할 수 있으므로 commit 시점에 대한 호출자 계약을 적는다.

**경계 조건**

- registration timeout 바로 전·정확한 경계·바로 후
- idle timeout 정확한 경계
- PING 직후 여러 tick
- pong timeout 정확한 경계
- pending token 없음 상태의 PONG
- 잘못된 token 뒤 올바른 token
- 이전 token의 늦은 PONG
- idle timeout이 0 또는 비활성 정책인 경우
- recordActivity가 outstanding probe 중 호출되는 경우

**실패 조건**

- registration timeout
- PONG timeout
- PING 송신이 외부에서 실패해 연결이 제거되는 경우

**제약**

- 20~25분 안에 구현한다.
- wall-clock API를 사용하지 않는다.
- token 생성기는 구현하지 않고 입력으로 받는다.
- 실제 socket·event loop·로그는 구현하지 않는다.

```cpp
HeartbeatAction onHeartbeatTick(
    HeartbeatState& state,
    const HeartbeatConfig& config,
    TimePoint now,
    const std::string& nextToken) {
    // 직접 구현
}

bool onPong(
    HeartbeatState& state,
    const std::string& token,
    TimePoint now) {
    // 직접 구현
}

void recordActivity(HeartbeatState& state, TimePoint now) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 등록 전 연결에는 idle PING보다 registration timeout이 먼저 적용되는가?
- [ ] idle 경계에서 PING이 정확히 한 번 생성되는가?
- [ ] outstanding PING이 있는 동안 새 token으로 덮어쓰지 않는가?
- [ ] 잘못된 PONG이 deadline을 연장하지 않는가?
- [ ] 올바른 PONG 뒤 pending token과 ping 시각이 함께 정리되는가?
- [ ] 늦은 이전 PONG이 새 pending token을 해제하지 않는가?
- [ ] timeout 경계에서 `<`와 `<=` 정책이 테스트와 일치하는가?
- [ ] tick 간격 때문에 생기는 최대 지연을 설명할 수 있는가?
- [ ] 실제 send 성공 전후 state commit 시점을 호출자와 합의했는가?

### 구현 후 설명할 것

- `steady_clock`을 선택한 이유
- token 검증이 replay·stale response를 막는 방식
- registration, idle, pong timeout의 우선순위
- action 생성과 실제 송신 성공 사이의 상태 commit 방법
- tick polling과 per-connection timer queue의 trade-off

### 원본 확인 위치

- Thread 12
- 커밋: `feat(registration): 등록 대기 시간 제한`, `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`, `test(client): 유휴 연결의 PING·PONG 흐름 검증`, `refactor(command): 명령 처리 시각 기록 통합`, `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`
- 파일: `src/ClientRegistry.hpp`, `src/ClientRegistry.cpp`, `src/IrcApplication.cpp`, `src/IrcApplication.hpp`, `src/RegistrationCommands.cpp`, `tools/irc_smoke_client.py`
- 함수·컴포넌트: `MonotonicClock`, `MonotonicTime`, `ClientState::connectedAt`, `lastActivityAt`, `lastPingAt`, `pendingPongToken`, `IrcApplication::onTick`, `maintainClient`, `handlePong`
- 관련 Thread: 05, 11, 14

---

## [Thread 11 / `feat(throttle): 클라이언트별 명령 호출 횟수 제한`] P13. Sliding-window 명령 호출 제한

### 면접 질문

연결별로 "최근 S초 동안 C개 명령"을 허용하려면 어떤 상태를 저장하고, 경계 시각의 오래된 항목을 언제 제거해야 합니까? 고정 창 카운터와 sliding window의 차이도 설명해 보세요.

꼬리 질문:

- 정확히 `now - window`인 timestamp는 만료로 볼지 포함할지 어떻게 정합니까?
- 제한된 명령 자체의 timestamp를 window에 넣습니까?
- PONG이나 QUIT 같은 명령도 같은 제한을 적용합니까?
- 연결마다 timestamp deque를 저장하면 메모리 상한은 어떻게 계산합니까?
- token bucket으로 바꾸면 burst 허용 특성이 어떻게 달라집니까?

### 30초 모범 답변

정확한 sliding window는 연결마다 허용된 최근 명령 시각을 오름차순 deque로 보관하고, 새 명령 전에 window 밖 timestamp를 앞에서 제거합니다. 남은 개수가 limit보다 작으면 현재 시각을 추가해 허용하고, 아니면 거절합니다. 경계 시각을 포함할지는 `now - oldest >= window`처럼 계약을 하나로 고정해야 합니다. 저장 개수는 허용 limit을 넘지 않으므로 연결당 O(C)로 제한할 수 있습니다. 고정 창은 구현이 단순하지만 창 경계에서 두 배 burst가 가능하고, token bucket은 평균 rate와 burst capacity를 별도로 조정할 수 있습니다.

### 답변 핵심 키워드

sliding window, timestamp deque, monotonic time, 만료 경계, 연결별 격리, O(C) memory, fixed-window boundary burst, token bucket trade-off

### 백지 구현

**구현 목표**

연결 하나에 대한 정확한 sliding-window limiter를 구현한다.

**면접용 축소 인터페이스**

```cpp
#include <chrono>
#include <cstddef>
#include <deque>

class SlidingWindowLimiter {
public:
    using Clock = std::chrono::steady_clock;
    using TimePoint = Clock::time_point;

    SlidingWindowLimiter(
        std::size_t maxCommands,
        std::chrono::milliseconds window);

    bool allow(TimePoint now);
    std::size_t activeCount(TimePoint now);

private:
    std::size_t maxCommands_;
    std::chrono::milliseconds window_;
    std::deque<TimePoint> accepted_;
};
```

**입력과 출력**

- 입력: 단조 시각
- 출력: 현재 명령 허용 여부, 현재 window 안의 허용 명령 수

**반드시 만족해야 할 조건**

- 만료 timestamp를 판정 전에 제거한다.
- 허용된 명령만 deque에 추가한다.
- deque 순서는 단조 증가한다는 계약을 명시한다.
- 저장 항목 수는 maxCommands를 넘지 않는다.
- maxCommands 0과 window 0의 정책을 명시한다.
- 경계 시각의 포함·제외 규칙을 `allow`와 `activeCount`에 동일하게 적용한다.

**경계 조건**

- 첫 명령
- 정확히 limit개 연속 명령
- limit+1번째 명령
- 가장 오래된 timestamp가 정확히 window 경계
- 여러 항목이 한 번에 만료
- 거절 후 시간이 지나 다시 허용
- 같은 `now`로 여러 호출
- 입력 시각이 이전 호출보다 과거인 경우

**실패 조건**

- 설정 자체가 유효하지 않은 경우
- non-monotonic 입력 시각 정책 위반

**제약**

- 15~20분 안에 구현한다.
- system clock을 사용하지 않는다.
- thread safety는 요구하지 않는다.
- 거절된 시도를 저장하지 않는다.

```cpp
SlidingWindowLimiter::SlidingWindowLimiter(
    std::size_t maxCommands,
    std::chrono::milliseconds window)
    : maxCommands_(maxCommands),
      window_(window) {}

bool SlidingWindowLimiter::allow(TimePoint now) {
    // 직접 구현
}

std::size_t SlidingWindowLimiter::activeCount(TimePoint now) {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] limit번째는 허용되고 limit+1번째는 거절되는가?
- [ ] 거절된 시도가 deque 크기를 키우지 않는가?
- [ ] 여러 timestamp 만료가 한 호출에서 모두 정리되는가?
- [ ] 정확한 경계 시각 정책이 테스트로 고정되어 있는가?
- [ ] 같은 시각 burst의 결과가 예측 가능한가?
- [ ] deque 크기가 maxCommands를 넘지 않는가?
- [ ] 입력 시각 역행 정책이 명시되어 있는가?
- [ ] 각 timestamp가 한 번 삽입·제거되어 amortized O(1)인지 설명할 수 있는가?
- [ ] 연결별 limiter를 둘 때 전체 메모리가 O(connectionCount × limit)임을 설명할 수 있는가?

### 구현 후 설명할 것

- 경계 시각 포함 규칙
- 거절된 시도를 저장하지 않은 이유
- fixed window와 비교한 정확성·비용
- token bucket으로 바꿀 때 필요한 상태
- exempt command를 둘 경우 정책 계층을 어디에 배치할지

### 원본 확인 위치

- Thread 11
- 커밋: `feat(throttle): 클라이언트별 명령 호출 횟수 제한`
- 파일: `src/ClientRegistry.hpp`, `src/IrcApplication.cpp`, `src/RuntimeConfig.hpp`, `src/RuntimeConfig.cpp`
- 함수·컴포넌트: `ClientState::commandWindow`, `IrcApplication::recordCommand`, `RuntimeConfig::rateLimitCount`, `RuntimeConfig::rateLimitWindowSeconds`
- 관련 Thread: 12, 13

---

## [Thread 13 / `feat(buffer): 송신 대기열 크기 제한` · `feat(server): 최대 연결 수 제한` · `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`] P14. 계층형 자원 제한과 slow receiver 격리

### 면접 질문

이 서버는 연결별 미전송 byte 상한과 서버 전체 동시 연결 상한을 따로 둡니다. 둘 중 하나만 있어서는 어떤 공격·장애를 막지 못하며, 느린 수신자 하나가 다른 연결을 멈추지 않게 하려면 event loop와 queue 정책이 어떻게 협력해야 합니까?

꼬리 질문:

- max connections가 0이면 무제한으로 해석하는 정책의 장단점은 무엇입니까?
- accept한 뒤 limit 초과를 확인하면 이미 얻은 fd를 어떻게 처리해야 합니까?
- 연결별 queue limit 초과 시 그 메시지만 drop할지 연결을 닫을지 어떤 기준이 있습니까?
- per-connection queue가 있어도 fast sender가 CPU를 독점할 수 있는 이유는 무엇입니까?
- 160개 연결 테스트와 slow receiver 테스트가 "엄격한 공정성"까지 증명하지는 못하는 이유는 무엇입니까?

### 30초 모범 답변

동시 연결 상한은 fd·connection object·per-client state의 총량을 제한하고, 연결별 queue 상한은 읽지 않는 한 client가 메모리를 독점하는 것을 막습니다. accept 뒤 limit 초과라면 새 fd를 즉시 닫고 event backend나 connection map에 반등록 상태를 남기지 않아야 합니다. 송신은 연결별 queue와 write readiness로 분리되어야 slow receiver의 `send`가 다른 연결 처리를 block하지 않습니다. 다만 한 readable 연결에서 무제한으로 명령을 처리하면 CPU 공정성은 여전히 깨질 수 있어 per-event work quantum은 별도 정책입니다.

### 답변 핵심 키워드

layered limits, fd exhaustion, per-client memory, admission control, rollback, slow receiver isolation, nonblocking queue, work quantum, progress vs strict fairness, overload policy

### 백지 구현

**구현 목표**

서버 전체 연결 수와 연결별 예약된 outbound byte를 관리하는 작은 resource budget을 구현한다. 실제 socket·queue 내용은 관리하지 않는다.

**면접용 축소 인터페이스**

```cpp
#include <cstddef>
#include <unordered_map>

enum class OpenResult {
    Opened,
    DuplicateFd,
    ConnectionLimit,
};

enum class ReserveResult {
    Reserved,
    MissingConnection,
    QueueLimit,
    Overflow,
};

class ResourceBudget {
public:
    ResourceBudget(
        std::size_t maxConnections,
        std::size_t maxPendingPerConnection);

    OpenResult tryOpen(int fd);
    void close(int fd) noexcept;

    ReserveResult tryReserveOutbound(int fd, std::size_t bytes);
    bool releaseOutbound(int fd, std::size_t bytes) noexcept;

    std::size_t connectionCount() const;
    std::size_t pendingBytes(int fd) const;

private:
    std::size_t maxConnections_;
    std::size_t maxPendingPerConnection_;
    std::unordered_map<int, std::size_t> pendingByFd_;
};
```

**입력과 출력**

- 입력: 새 fd, 예약·해제 byte 수
- 출력: admission 또는 예약 결과
- 내부 상태: 열린 fd와 각 fd의 논리 pending byte

**반드시 만족해야 할 조건**

- maxConnections 0을 무제한으로 해석한다.
- 중복 fd open은 기존 항목을 덮어쓰지 않는다.
- connection limit 거절 시 상태가 변하지 않는다.
- per-connection pending 계산에서 unsigned overflow가 발생하지 않는다.
- queue limit 거절 시 기존 pending 값이 변하지 않는다.
- 존재하지 않는 fd reserve/release 정책을 구분한다.
- release가 현재 pending보다 크면 underflow하지 않는다.
- close는 해당 fd의 pending 예약도 함께 제거한다.
- 한 연결의 실패가 다른 연결의 budget을 바꾸지 않는다.

**경계 조건**

- maxConnections 0, 1
- 첫 연결과 limit 정확히 도달
- limit+1번째 연결
- 중복 fd
- 0 byte reserve
- exact queue limit와 limit+1
- 일부 release 뒤 재예약
- pending보다 큰 release
- close 두 번 호출
- 매우 큰 bytes 입력

**실패 조건**

- connection limit
- queue limit
- 산술 overflow
- 존재하지 않는 연결
- release underflow 시도

**제약**

- 20~25분 안에 구현한다.
- 전역 total byte limit은 요구하지 않지만 확장 위치를 설명한다.
- thread safety는 요구하지 않는다.
- 실제 message drop·disconnect 정책은 반환값의 호출자가 결정한다.

```cpp
ResourceBudget::ResourceBudget(
    std::size_t maxConnections,
    std::size_t maxPendingPerConnection)
    : maxConnections_(maxConnections),
      maxPendingPerConnection_(maxPendingPerConnection) {}

OpenResult ResourceBudget::tryOpen(int fd) {
    // 직접 구현
}

void ResourceBudget::close(int fd) noexcept {
    // 직접 구현
}

ReserveResult ResourceBudget::tryReserveOutbound(
    int fd,
    std::size_t bytes) {
    // 직접 구현
}

bool ResourceBudget::releaseOutbound(
    int fd,
    std::size_t bytes) noexcept {
    // 직접 구현
}
```

### 구현 후 자가 검증

- [ ] connection limit 거절 전후 map이 동일한가?
- [ ] maxConnections 0이 실제로 무제한인가?
- [ ] exact queue limit 예약은 성공하고 +1은 무변경 실패인가?
- [ ] 큰 입력에서 pending 합산이 wrap하지 않는가?
- [ ] release underflow가 pending을 큰 값으로 바꾸지 않는가?
- [ ] close 뒤 connection count와 pending 정보가 함께 사라지는가?
- [ ] 한 fd의 reserve 실패가 다른 fd pending에 영향을 주지 않는가?
- [ ] 평균 lookup 복잡도가 O(1)인지 설명할 수 있는가?
- [ ] 이 자료구조만으로 CPU 공정성과 event-loop progress가 보장되지 않음을 설명했는가?
- [ ] slow receiver 통합 테스트에서 어떤 외부 현상을 관찰해야 하는지 제시했는가?

### 구현 후 설명할 것

- 서버 전체 limit과 연결별 limit을 분리한 이유
- accept 후 거절 경로에서 정리해야 할 리소스 순서
- queue limit 초과 시 drop·disconnect·producer throttling의 trade-off
- slow receiver 격리와 strict fairness의 차이
- global pending byte limit이나 per-event work quantum을 추가할 위치

### 원본 확인 위치

- Thread 13
- 커밋: `feat(buffer): 송신 대기열 크기 제한`, `feat(server): 최대 연결 수 제한`, `test(smoke): 서버 보호 옵션으로 실행 검증`, `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`
- 파일: `src/Connection.cpp`, `src/ConnectionLimits.hpp`, `src/Server.cpp`, `src/RuntimeConfig.cpp`, `tests/irc_event_fairness.py`, `tests/irc_smoke.sh`
- 함수·컴포넌트: `Connection::queueRaw`, `Connection::pendingBytes`, `Server::acceptReadyClients`, `Server::Config::maxConnections`, `check_many_connections`, `check_slow_receiver_isolation`
- 관련 Thread: 02, 03, 11, 15
