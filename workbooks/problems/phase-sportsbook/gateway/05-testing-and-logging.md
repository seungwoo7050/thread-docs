# 결정적 통합 테스트와 안전한 로깅 면접 워크북

비동기 시스템에서 시간 추측을 상태 신호로 바꾸는 방법, 동시 요청·구독자의 교차 오염을 검증하는 방법, 운영 로그가 비밀정보 경계를 넘지 않게 하는 마지막 방어선을 다룬다.

## TEST-01 · [Thread 15 / `test(websocket): observe broker subscription registration`, `test(websocket): await broker subscription registration`] sleep을 제거한 STOMP 구독 등록 완료 신호

**우선순위:** A

### 면접 질문

- STOMP client의 `subscribe()` 호출이 반환됐다는 사실만으로 broker가 해당 구독을 실제 등록했다고 볼 수 없는 이유는 무엇입니까?
- 고정 sleep이나 주기적 registry 조회 대신 broker가 `SUBSCRIBE`를 성공적으로 처리한 순간을 관찰하면 테스트가 왜 더 결정적입니까?
- subscription ID별 `CompletableFuture` 기대값과 session별 subscription 집합을 함께 관리한 이유를 설명해 보세요.
- 꼬리 질문: 구독 처리 전에 세션이 끊기면 대기 중인 테스트를 timeout까지 방치하지 않고 어떻게 실패시켜야 합니까?
- 꼬리 질문: production 경로를 테스트 편의를 위해 바꾸지 않고 관찰 지점을 추가하는 방법의 장단점은 무엇입니까?

### 30초 모범 답변

`subscribe()`는 client frame 전송을 시작했다는 뜻이지, 비동기 inbound channel과 simple broker가 등록을 끝냈다는 뜻은 아닙니다. 그래서 테스트는 subscription ID로 기대 future를 먼저 만들고, broker가 해당 `SUBSCRIBE`를 예외 없이 처리하고 유효한 broker destination임을 확인한 시점에 future를 완료합니다. disconnect가 먼저 오면 그 세션의 미완료 기대를 즉시 실패시키고, 호출자는 `finally`에서 기대값을 해제합니다. 시간 추측 대신 실제 상태 전이를 기다리므로 느린 CI에서도 race가 줄어듭니다.

### 답변 핵심 키워드

`asynchronous readiness`, `observable state transition`, `subscription ID`, `CompletableFuture`, `bounded wait`, `disconnect cleanup`, `no fixed sleep`, `test-only probe`

### 백지 구현

**구현 목표**

비동기 broker가 구독 등록을 끝낸 시점을 기다리는 테스트용 barrier를 구현한다. 성공 처리, 중복 기대, disconnect, timeout 뒤 정리가 명확해야 한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
enum BrokerEventType {
  SUBSCRIBE, DISCONNECT, OTHER
}

record BrokerEvent(
    BrokerEventType type,
    String sessionId,
    String subscriptionId,
    String destination,
    boolean brokerRunning,
    boolean handledSuccessfully,
    boolean brokerDestination) {}

final class SubscriptionRegistrationBarrier {
  CompletableFuture<Void> expect(String subscriptionId) {
    throw new UnsupportedOperationException("직접 구현");
  }

  void afterBrokerHandled(BrokerEvent event) {
    // 직접 구현
  }

  void release(String subscriptionId) {
    // 직접 구현
  }

  int pendingExpectations() {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 기대할 subscription ID와 broker 처리 결과를 표현한 이벤트
- 출력: 해당 구독이 실제 등록됐을 때 완료되는 `CompletableFuture<Void>`
- disconnect 시: 그 session의 미완료 future는 예외 완료

**반드시 만족해야 할 조건**

- 같은 subscription ID의 기대를 두 번 등록하면 즉시 거부한다.
- 성공한 `SUBSCRIBE`이고 broker가 실행 중이며 broker destination일 때만 완료한다.
- 관계없는 handler·message type·subscription ID는 무시한다.
- session과 subscription의 관계를 기록해 disconnect 시 해당 기대만 정리한다.
- 완료 여부와 무관하게 호출자가 `release`하여 expectations map에서 제거할 수 있어야 한다.
- 대기에는 명시적 timeout을 둘 수 있어야 한다.

**경계 조건**

- subscription ID 또는 session ID 누락
- expect 이전에 이벤트가 도착하는 잘못된 호출 순서
- 같은 session의 여러 subscription
- SUBSCRIBE 처리 실패
- broker가 중지된 상태
- 등록 직전 또는 직후 disconnect

**실패 조건**

- 고정 sleep 후 아직 등록되지 않은 상태에서 이벤트를 publish하는 race
- disconnect된 세션의 future가 영원히 남는 누수
- 다른 session의 동일하지 않은 subscription을 잘못 완료하는 교차 신호
- timeout 후 expectation을 제거하지 않아 후속 테스트가 영향을 받는 오류

**필요한 제약**

- 실제 broker 구현을 재작성하지 않는다.
- production 메시지 처리 순서를 변경하지 않는 테스트 전용 관찰자로 설계한다.
- polling loop와 무제한 대기를 사용하지 않는다.

### 구현 후 자가 검증

- [ ] expect 후 정상 SUBSCRIBE 처리 이벤트가 오면 정확한 future 하나만 완료된다.
- [ ] 실패한 처리, 다른 type, 다른 subscription ID에서는 완료되지 않는다.
- [ ] 한 session의 여러 subscription이 독립적으로 완료된다.
- [ ] disconnect는 해당 session의 미완료 기대를 즉시 예외 완료한다.
- [ ] timeout·성공·실패 모든 경로에서 `release` 후 pending count가 원상 복구된다.
- [ ] 고정 sleep 없이도 publish 이전에 registration 완료를 보장하는 호출 순서가 성립한다.
- [ ] probe가 production business logic이나 destination 정책을 우회하지 않는다.

### 구현 후 설명할 것

- client API 반환과 server-side 상태 완료가 다른 비동기 경계라는 점
- 시간 기반 대기보다 상태 기반 신호가 결정적인 이유
- subscription ID map과 session 역인덱스를 함께 둔 이유
- disconnect와 timeout에서 future를 실패·정리하는 수명 주기
- 내부 구현 관찰 테스트가 가지는 결합도와 이를 제한하는 방법

### 원본 확인 위치

- Thread 15
- 커밋: `test(websocket): observe broker subscription registration`
- 커밋: `test(websocket): await broker subscription registration`
- 파일: `src/test/java/com/sportsbook/gateway/ws/SubscriptionRegistrationProbe.java`
- 파일: `src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java`
- 관련 Thread: 13, 14

## TEST-02 · [Thread 15 / `test(routing): verify concurrent request isolation`, `test(realtime): verify concurrent subscriber isolation`] 동시 요청 tuple과 구독자 소유권의 교차 오염 검증

**우선순위:** A

### 면접 질문

- 단일 요청 테스트가 모두 통과해도 보안 principal, idempotency key, trace context가 동시 요청 사이에서 섞일 수 있는 이유는 무엇입니까?
- 여러 요청을 동시에 시작시키고 각 요청의 고유 tuple이 다운스트림에서 정확히 한 번 관찰됐음을 어떻게 검증합니까?
- 사용자별 WebSocket fan-out에서 각 owner가 자기 payload만 받았다는 positive assertion과 다른 payload를 받지 않았다는 negative assertion을 함께 두어야 하는 이유는 무엇입니까?
- 꼬리 질문: 이 테스트를 부하 테스트로 보지 않고 격리 invariant 테스트로 보는 이유는 무엇입니까?
- 꼬리 질문: thread pool과 WebSocket session 정리가 실패 경로에서도 보장되지 않으면 어떤 flakiness가 생깁니까?

### 30초 모범 답변

동시성 격리 테스트의 목적은 처리량 측정이 아니라 요청별 상태가 공유 가변 객체나 잘못된 context 전달 때문에 섞이지 않는지 확인하는 것입니다. 모든 작업을 barrier로 함께 시작하고 각 요청에 고유한 subject·idempotency key·traceparent·body를 넣은 뒤, 다운스트림에서 그 tuple이 정확히 한 번 대응되는지 검증합니다. WebSocket도 owner별 session과 queue를 만들고 자기 이벤트만 수신했으며 추가 메시지가 없음을 확인합니다. pool과 session은 `finally`에서 닫아 후속 테스트 오염을 막습니다.

### 답변 핵심 키워드

`isolation invariant`, `correlated tuple`, `CyclicBarrier`, `request-scoped context`, `negative assertion`, `owner isolation`, `exact correspondence`, `deterministic cleanup`

### 백지 구현

**구현 목표**

서로 다른 요청·구독자 식별자가 동시에 처리될 때 교차 오염이 없는지 검증하는 축소 테스트 harness를 작성한다. 성능 수치가 아니라 입력 tuple과 관찰 tuple의 정확한 대응을 확인한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record RequestCase(
    String subject,
    String idempotencyKey,
    String traceparent,
    String body,
    Map<String, String> spoofedTrustHeaders) {}

record DownstreamObservation(
    String subject,
    String idempotencyKey,
    String traceparent,
    String body,
    Set<String> forwardedHeaders) {}

interface RequestInvoker {
  DownstreamObservation invoke(RequestCase request) throws Exception;
}

final class ConcurrentIsolationHarness {
  List<DownstreamObservation> run(
      List<RequestCase> cases,
      RequestInvoker invoker,
      Duration timeout) throws Exception {
    throw new UnsupportedOperationException("직접 구현");
  }

  void assertOneToOneIsolation(
      List<RequestCase> cases,
      List<DownstreamObservation> observations) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 서로 다른 subject·idempotency key·traceparent·body를 가진 요청 목록
- 출력: 다운스트림에서 관찰한 tuple 목록
- 검증 결과: 입력마다 정확히 하나의 동일 tuple이 존재하고 다른 입력과 필드가 섞이지 않음

**반드시 만족해야 할 조건**

- 모든 worker가 준비된 뒤 가능한 한 동시에 요청을 시작한다.
- 각 작업 결과와 예외를 bounded timeout 안에 수집한다.
- 관찰 결과를 개별 필드 집합이 아니라 하나의 correlation tuple로 비교한다.
- 외부에서 주입한 trust header와 Authorization 같은 제거 대상이 전달되지 않았음을 검증한다.
- 입력 개수와 관찰 개수가 같고 각 tuple이 정확히 한 번 존재해야 한다.
- executor와 연결 자원은 정상·예외·timeout 모든 경로에서 정리한다.

**경계 조건**

- 작업 수가 0개·1개인 경우
- worker 하나가 barrier 전에 실패하는 경우
- 일부 요청만 timeout 또는 예외가 나는 경우
- 두 요청이 일부 필드만 같고 나머지는 다른 경우
- 결과 순서가 입력 순서와 다른 경우
- 한 owner queue에 메시지가 없거나 두 개 이상 들어온 경우

**실패 조건**

- ThreadLocal 또는 공유 builder가 요청 사이에 남아 subject·trace가 섞이는 오류
- 결과를 단순 정렬 위치로 비교해 순서 변화만으로 실패하는 취약한 테스트
- positive assertion만 하고 다른 owner의 메시지 유입을 놓치는 오류
- worker 예외 때문에 barrier가 깨졌는데 무제한 대기하는 deadlock
- pool·session 미정리로 후속 테스트에 listener나 연결이 남는 오류

**필요한 제약**

- 처리량, 평균 지연, percentile은 측정하지 않는다.
- 작업 수는 면접 시간 안에 실행 가능한 작은 고정값으로 둔다.
- 실제 네트워크가 없으면 invoker fake로도 tuple 격리 로직을 검증할 수 있다.

### 구현 후 자가 검증

- [ ] 모든 정상 작업이 barrier 이후 실행되고 bounded 시간 안에 끝난다.
- [ ] 결과 순서와 무관하게 입력 tuple마다 정확히 하나의 관찰 tuple이 대응된다.
- [ ] 한 요청의 subject와 다른 요청의 traceparent가 합쳐진 혼합 tuple을 탐지한다.
- [ ] spoofed trust headers와 원본 Authorization이 관찰 결과에 없다.
- [ ] 중복 관찰·누락 관찰을 모두 실패로 처리한다.
- [ ] owner별 메시지 검증에는 자기 메시지 존재와 추가·타인 메시지 부재가 모두 포함된다.
- [ ] 일부 task 실패와 timeout에서도 executor·session 정리가 완료된다.
- [ ] 이 테스트 결과를 성능 보장이나 exactly-once 보장으로 과장하지 않는다.

### 구현 후 설명할 것

- 개별 필드 검증보다 correlation tuple 검증이 교차 오염을 잘 찾는 이유
- 동시 시작 barrier가 race 재현 가능성을 높이는 방식과 한계
- 요청 격리와 사용자 구독 격리의 공통 invariant
- positive·negative assertion을 함께 둔 이유
- 통합 테스트의 bounded timeout과 resource cleanup 원칙

### 원본 확인 위치

- Thread 15
- 커밋: `test(routing): verify concurrent request isolation`
- 커밋: `test(realtime): verify concurrent subscriber isolation`
- 파일: `src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java`
- 파일: `src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java`
- 관련 Thread: 4, 5, 7, 13, 14

## LOG-01 · [Thread 18 / `feat(logging): emit redacted structured logs`, `test(logging): verify JSON redaction and context`] 메시지·stack trace 비밀정보 정화와 MDC 허용목록

**우선순위:** A

### 면접 질문

- 구조화 JSON 로그가 보안에 자동으로 유리한 것은 아니며, 오히려 어떤 방식으로 비밀정보를 더 쉽게 확산시킬 수 있습니까?
- formatted message뿐 아니라 stack trace도 같은 정화 경계를 통과시켜야 하는 이유는 무엇입니까?
- MDC 전체를 출력하지 않고 `traceId`와 `spanId`만 허용목록으로 내보낸 이유를 설명해 보세요.
- 꼬리 질문: labelled secret과 독립적인 `Bearer ...` 패턴을 모두 처리할 때 false positive·false negative trade-off는 무엇입니까?
- 꼬리 질문: 최종 redaction provider가 있어도 애플리케이션 코드에서 credential을 로그 인자로 넘기면 안 되는 이유는 무엇입니까?

### 30초 모범 답변

JSON 로그는 수집·검색·복제가 쉬워 한 번 들어간 비밀도 더 넓게 퍼집니다. 그래서 최종 encoder 경계에서 formatted message와 stack trace를 모두 정화하고, MDC는 traceId와 spanId만 허용합니다. authorization·API key·password·token처럼 label이 있는 값과 독립 Bearer 토큰을 가리되, 정규식은 모든 비밀 형식을 완전하게 찾지 못하고 정상 텍스트를 가릴 수도 있습니다. 따라서 redaction은 마지막 방어선이고, 애초에 자격 증명을 로그·예외 메시지에 넣지 않는 것이 기본 원칙입니다.

### 답변 핵심 키워드

`structured log amplification`, `last-line defense`, `formatted message`, `stack trace`, `MDC allowlist`, `labelled secret`, `Bearer redaction`, `false positive/negative`

### 백지 구현

**구현 목표**

로그 이벤트에서 허용된 context만 남기고, 메시지와 예외 텍스트의 알려진 credential 패턴을 정화해 직렬화 가능한 안전한 이벤트를 만드는 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record LogEvent(
    String level,
    String loggerName,
    String formattedMessage,
    String stackTrace,
    Map<String, String> mdc) {}

record SafeLogEvent(
    String level,
    String loggerName,
    String message,
    String stackTrace,
    Map<String, String> context,
    String service) {}

final class LogSanitizer {
  SafeLogEvent sanitize(LogEvent event) {
    throw new UnsupportedOperationException("직접 구현");
  }

  String redact(String value) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: formatted message, 선택적 stack trace, MDC map을 가진 로그 이벤트
- 출력: 정화된 message·stack trace, 허용된 context, 고정 service 필드를 가진 `SafeLogEvent`
- null message 또는 stack trace는 안전한 빈 값 또는 필드 생략 정책으로 처리

**반드시 만족해야 할 조건**

- authorization, internal API key, API key, password, token label 뒤 값을 `[REDACTED]`로 바꾼다.
- label 없이 나타난 `Bearer <value>`도 대소문자와 무관하게 정화한다.
- formatted message와 stack trace에 동일한 정화 함수를 적용한다.
- MDC에서는 `traceId`, `spanId`만 복사하며 다른 key는 출력하지 않는다.
- service 필드는 `gateway`로 고정한다.
- 원본 비밀 문자열이 최종 이벤트의 어떤 필드에도 남지 않아야 한다.

**경계 조건**

- null·빈 message
- 따옴표로 감싼 값, 쉼표·세미콜론·개행으로 끝나는 값
- 서로 다른 대소문자의 header label
- 한 문자열에 여러 credential이 있는 경우
- stack trace의 예외 메시지 안에 credential이 있는 경우
- MDC에 authorization·password 같은 key가 있는 경우

**실패 조건**

- message만 가리고 stack trace에서 비밀이 남는 오류
- MDC 전체 복사로 임의 context가 노출되는 오류
- 첫 번째 credential만 가리고 뒤의 값이 남는 오류
- 정규식의 과도한 backtracking으로 긴 로그에서 지연이 커지는 문제
- redaction 결과에 원본 값 일부가 남는 부분 치환 오류

**필요한 제약**

- 모든 가능한 비밀 형식을 완전 탐지한다고 주장하지 않는다.
- 원본 이벤트 객체를 수정하지 않고 새 안전 이벤트를 만든다.
- 로그 수집기나 파일 보존 정책은 구현 범위가 아니다.

### 구현 후 자가 검증

- [ ] 각 labelled secret과 독립 Bearer token이 message에서 가려진다.
- [ ] 동일 패턴이 stack trace에서도 가려진다.
- [ ] 한 줄에 여러 secret이 있어도 모두 제거된다.
- [ ] 최종 직렬화 문자열 전체에 원본 secret이 하나도 없다.
- [ ] MDC의 traceId·spanId는 남고 authorization 등 다른 key는 없다.
- [ ] 예외가 없으면 stack trace 필드 생략 정책이 일관된다.
- [ ] 일반 메시지의 비밀이 아닌 부분과 label은 읽을 수 있게 유지된다.
- [ ] 매우 긴 입력과 반복 패턴에서 실행 시간이 비정상적으로 폭증하지 않는지 확인한다.
- [ ] 테스트가 redaction 존재뿐 아니라 원본 값 부재를 검증한다.

### 구현 후 설명할 것

- 구조화 로그가 운영 편의와 유출 범위를 동시에 키우는 이유
- message·stack trace·MDC를 하나의 출력 경계로 본 설계
- denylist 패턴과 MDC allowlist를 함께 사용한 이유
- 정규식 redaction의 한계와 upstream no-secret logging 원칙
- 필드 안정성이 로그 소비자와 경보 규칙에 주는 장점

### 원본 확인 위치

- Thread 18
- 커밋: `feat(logging): emit redacted structured logs`
- 커밋: `test(logging): verify JSON redaction and context`
- 파일: `src/main/java/com/sportsbook/gateway/logging/RedactedEventJsonProvider.java`
- 파일: `src/main/resources/logback-spring.xml`
- 테스트: `src/test/java/com/sportsbook/gateway/logging/StructuredLoggingTest.java`
- 관련 Thread: 2, 3, 5, 7, 17
