# WebSocket 수명 주기와 실시간 전달 면접 워크북

장기 연결의 인증 수명, raw transport resource 관리, 공개·사용자 범위 fan-out, process-local broker가 만드는 배치 불변조건을 다룬다. 메시지 전달 횟수를 과장하지 않고 중복·누락·재연결 경계를 설명하는 데 초점을 둔다.

## WS-01 · [Thread 9 / `feat(websocket): track raw WebSocket sessions`, `test(websocket): verify session registry lifecycle`] raw WebSocket 세션 레지스트리와 예외 안전한 수명 주기

**우선순위:** S

### 면접 질문

- STOMP 인증·구독 정보만으로는 토큰 만료 시 실제 소켓을 닫기 어려운 이유와, raw `WebSocketSession`을 별도로 추적한 이유를 설명해 보세요.
- 연결 수립 콜백 전에 세션을 등록한 뒤 delegate가 실패하면 어떤 상태가 남을 수 있으며, 이를 어떻게 되돌려야 합니까?
- 연결 종료 콜백 자체가 예외를 던져도 레지스트리 정리가 보장되어야 하는 이유는 무엇입니까?
- 꼬리 질문: `remove(sessionId)`보다 `remove(sessionId, session)`이 안전한 경우는 언제입니까?

### 30초 모범 답변

인증 만료 정책은 STOMP 논리 세션이 아니라 실제 전송 연결을 닫아야 하므로 raw WebSocket 세션의 참조가 필요합니다. 연결 수립 전 등록하면 다른 수명 주기 컴포넌트가 세션을 찾을 수 있지만, delegate가 실패하면 같은 항목을 조건부 제거해 롤백해야 합니다. 종료 시에는 delegate 결과와 무관하게 `finally`에서 제거해 누수를 막고, 세션 ID 재사용이나 교체 가능성을 고려해 현재 참조와 일치할 때만 제거합니다.

### 답변 핵심 키워드

`transport vs protocol lifecycle`, `ConcurrentMap`, `register-before-delegate`, `rollback`, `finally cleanup`, `conditional remove`, `resource leak`

### 백지 구현

**구현 목표**

동시에 여러 연결이 열리고 닫히는 환경에서 raw 세션을 추적하는 decorator를 작성한다. delegate 수명 주기 콜백이 실패해도 레지스트리 불변조건이 깨지지 않아야 한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
interface RawSession {
  String id();
  boolean isOpen();
  void close(int code) throws IOException;
}

interface SessionHandler {
  void afterEstablished(RawSession session) throws Exception;
  void afterClosed(RawSession session) throws Exception;
}

final class TrackingSessionHandler implements SessionHandler {
  TrackingSessionHandler(SessionHandler delegate) {
    // 직접 구현
  }

  @Override
  public void afterEstablished(RawSession session) throws Exception {
    // 직접 구현
  }

  @Override
  public void afterClosed(RawSession session) throws Exception {
    // 직접 구현
  }

  RawSession find(String sessionId) {
    throw new UnsupportedOperationException("직접 구현");
  }

  int size() {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 세션 ID를 가진 raw 세션과 delegate 수명 주기 콜백
- 출력: 현재 연결된 세션 조회 및 개수
- 콜백 실패 시: 원래 예외를 유지하되 내부 등록 상태는 정리

**반드시 만족해야 할 조건**

- 연결 수립 시 delegate 호출 전에 세션을 레지스트리에 넣는다.
- 연결 수립 delegate가 실패하면 방금 넣은 동일 세션만 제거한다.
- 종료 delegate 성공 여부와 무관하게 동일 세션을 제거한다.
- 동시 접근에 안전해야 하며 외부 동기화 없이 조회할 수 있어야 한다.
- 다른 세션 참조가 같은 ID에 등록된 경우 오래된 종료 콜백이 새 항목을 지우면 안 된다.

**경계 조건**

- null 또는 빈 session ID
- 동일 ID의 중복 연결 시도
- 수립 콜백과 종료 콜백이 매우 가깝게 실행되는 경우
- delegate가 수립·종료 각각에서 예외를 던지는 경우
- 이미 닫힌 세션 조회

**실패 조건**

- 세션 등록 후 수립 실패 시 유령 세션이 남는 오류
- 종료 콜백 예외 때문에 항목이 제거되지 않는 누수
- 오래된 세션 종료가 새 세션 등록을 삭제하는 ABA 성격의 상태 오류

**필요한 제약**

- 세션 메시지 전송이나 인증은 구현하지 않는다.
- 세션 객체의 생명은 외부 전송 계층이 소유하며, 레지스트리는 참조만 관리한다.
- 전역 lock 하나로 모든 콜백을 직렬화하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 연결 수립 뒤 조회와 크기가 갱신되고, 정상 종료 뒤 원상 복구된다.
- [ ] 수립 delegate 실패 뒤 레지스트리가 비어 있고 원래 예외가 전달된다.
- [ ] 종료 delegate 실패 뒤에도 항목이 제거된다.
- [ ] 오래된 세션 종료가 같은 ID의 새 세션 항목을 제거하지 않는다.
- [ ] 동시 등록·조회·종료에서 자료구조 예외나 음수 상태가 발생하지 않는다.
- [ ] 콜백마다 반드시 정리되는 참조와 외부가 계속 소유하는 자원을 구분했다.

### 구현 후 설명할 것

- raw 전송 세션을 STOMP 인증 상태와 별도로 추적한 이유
- 등록 시점과 delegate 호출 순서를 선택한 이유
- 조건부 제거가 ID 재사용·지연 콜백 문제를 줄이는 방식
- 예외 보존과 resource cleanup을 동시에 만족시키는 `try`/`finally` 구조

### 원본 확인 위치

- Thread 9
- 커밋: `feat(websocket): track raw WebSocket sessions`
- 커밋: `test(websocket): verify session registry lifecycle`
- 파일: `src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java`
- 관련 Thread: 8, 16

## WS-02 · [Thread 9 / `feat(websocket): expire authenticated STOMP sessions`, `test(websocket): close expired authenticated sessions`] JWT 만료 예약, 조기 해제, 스케줄 등록 경쟁

**우선순위:** S

### 면접 질문

- HTTP 요청은 매번 JWT를 검증하지만 장시간 열린 WebSocket 연결은 왜 `exp` 이후에도 별도 조치가 없으면 살아 있을 수 있습니까?
- 인증된 CONNECT가 성공했을 때 만료 작업을 예약하고, 조기 disconnect에서는 취소해야 하는 상태 전이를 설명해 보세요.
- 스케줄러가 작업을 반환하기 전후로 disconnect가 발생하면 어떤 orphan task 경쟁이 생기며 어떻게 막습니까?
- 꼬리 질문: 만료 작업이 실행될 때 map에서 자신을 먼저 제거하고 소켓을 닫는 순서의 장점은 무엇입니까?

### 30초 모범 답변

WebSocket은 연결 후 매 프레임마다 HTTP 인증 필터를 거치지 않으므로 CONNECT에서 검증한 JWT의 `exp`에 맞춰 별도 종료 작업이 필요합니다. 세션당 작업은 하나만 등록하고, 조기 disconnect는 map에서 작업을 제거한 뒤 취소합니다. 예약 도중 disconnect가 끼어들 수 있으므로 placeholder와 조건부 교체 또는 동등한 원자적 상태 전이를 사용해, 등록되지 못한 실제 future를 즉시 취소해야 합니다. 만료 실행은 자기 항목을 제거한 경우에만 raw 세션을 1008로 닫아 중복 종료를 피합니다.

### 답변 핵심 키워드

`long-lived authentication`, `JWT exp`, `scheduled close`, `one task per session`, `placeholder registration`, `cancel-on-disconnect`, `orphan task race`, `close code 1008`

### 백지 구현

**구현 목표**

인증된 장기 연결을 토큰 만료 시각에 닫는 작은 스케줄러를 구현한다. 동시 CONNECT·disconnect·task 실행에서도 세션당 작업이 하나이고, 취소되지 않은 orphan 작업이 없어야 한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
interface Cancellable {
  boolean cancel();
}

interface DeadlineScheduler {
  Cancellable schedule(Instant when, Runnable task);
}

interface ExpirableSessionRegistry {
  void closeExpired(String sessionId) throws IOException;
}

final class JwtSessionExpiry {
  JwtSessionExpiry(DeadlineScheduler scheduler, ExpirableSessionRegistry sessions) {
    // 직접 구현
  }

  void onAuthenticatedConnect(String sessionId, Instant expiresAt) {
    // 직접 구현
  }

  void onDisconnect(String sessionId) {
    // 직접 구현
  }

  int pendingTasks() {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 인증된 CONNECT의 session ID와 JWT `expiresAt`, disconnect의 session ID
- 출력: 예약 작업 수와 만료 시 raw 세션 종료 부수효과
- 익명 연결은 이 컴포넌트의 입력 대상이 아니다.

**반드시 만족해야 할 조건**

- session ID나 만료 시각이 없으면 인증 세션 수명 계약 오류로 처리한다.
- 같은 session ID에 두 번째 만료 작업을 조용히 덮어쓰지 않는다.
- 예약 성공 뒤 map에는 취소 가능한 실제 작업 하나만 남아야 한다.
- 예약 실패·null 반환·등록 경쟁 시 placeholder와 실제 작업을 모두 정리한다.
- disconnect는 현재 작업을 map에서 제거하고 취소한다.
- 만료 task는 자신이 아직 현재 작업일 때만 항목을 제거하고 세션을 닫는다.
- 만료 종료는 policy violation에 해당하는 코드 1008을 사용할 수 있도록 경계를 둔다.

**경계 조건**

- 이미 지난 만료 시각
- CONNECT 직후 즉시 disconnect
- 스케줄러 호출 중 disconnect
- task 실행과 disconnect 동시 발생
- 동일 session ID의 중복 CONNECT
- 세션이 이미 닫혔거나 레지스트리에 없는 경우

**실패 조건**

- 스케줄러 예외 또는 예약 결과 없음
- map에 placeholder만 남는 상태
- disconnect 뒤에도 실행되는 orphan task
- 취소와 만료가 모두 소켓을 닫으려는 중복 side effect
- 세션 종료 I/O 실패

**필요한 제약**

- busy-wait나 별도 세션당 thread를 만들지 않는다.
- 토큰 갱신 프로토콜은 구현하지 않고, 만료 후 재연결을 전제로 한다.
- 익명 odds 세션에는 만료 task를 만들지 않는다.

### 구현 후 자가 검증

- [ ] 정상 인증 연결은 정확한 만료 시각으로 하나의 작업을 등록한다.
- [ ] 익명 연결 경로에서는 pending task가 생기지 않는다.
- [ ] 조기 disconnect는 작업을 제거하고 취소하며 이후 실행 부수효과가 없다.
- [ ] 예약 실패 뒤 placeholder·실제 future·세션 map에 잔여 상태가 없다.
- [ ] CONNECT 등록과 disconnect를 여러 순서로 교차시켜도 orphan task가 남지 않는다.
- [ ] 만료와 disconnect가 경합해도 raw 세션 종료는 최대 한 번 일어난다.
- [ ] 세션이 이미 닫혀 있어도 상태 정리는 완료된다.

### 구현 후 설명할 것

- 요청 단위 인증과 장기 연결 인증의 수명 주기 차이
- 세션당 task 하나라는 invariant와 이를 원자적으로 세우는 방법
- placeholder/조건부 교체 방식의 장점과 더 단순한 대안의 조건
- 취소는 best-effort이므로 task 내부에서도 소유권을 재확인해야 하는 이유
- 만료 시 강제 종료와 토큰 갱신 프로토콜 사이의 trade-off

### 원본 확인 위치

- Thread 9
- 커밋: `feat(websocket): expire authenticated STOMP sessions`
- 커밋: `test(websocket): close expired authenticated sessions`
- 파일: `src/main/java/com/sportsbook/gateway/ws/AuthenticatedSessionExpiryInterceptor.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java`
- 관련 Thread: 3, 8, 16

## WS-03 · [Thread 13, 16 / `feat(websocket): publish odds updates`; Thread 16 문서화 작업] 이벤트 범위 공개 팬아웃과 process-local broker의 단일 복제본 불변조건

**우선순위:** A

### 면접 질문

- 검증된 `OddsChanged`를 왜 원본 Avro 객체 그대로 보내지 않고 공개용 `OddsUpdate` projection으로 바꾸었습니까?
- 배당 변경을 하나의 전역 topic이 아니라 `/topic/odds/{eventId}`로 나누는 장점과 비용은 무엇입니까?
- Kafka consumer group과 process-local simple broker를 함께 사용할 때 여러 gateway replica에서 메시지가 누락될 수 있는 시나리오를 설명해 보세요.
- 꼬리 질문: 현재 단일 복제본 계약을 깨고 수평 확장하려면 어떤 종류의 shared fan-out 경계가 필요합니까?
- 꼬리 질문: 이 실시간 경로를 exactly-once 또는 durable replay라고 설명하면 안 되는 이유는 무엇입니까?

### 30초 모범 답변

Kafka 레코드는 strict decode와 key·payload 계약 검증 뒤 클라이언트에 필요한 필드만 공개 projection으로 변환하고, event별 destination으로 팬아웃합니다. 이렇게 하면 스키마 노출과 불필요한 트래픽을 줄이지만 destination 수와 구독 관리 비용이 늘어납니다. 현재 simple broker는 프로세스 로컬인데 Kafka group은 partition을 replica 사이에 나누므로, 구독자가 A에 있고 이벤트를 B가 소비하면 전달되지 않습니다. 그래서 1.0은 정확히 한 replica가 불변조건이며, 확장하려면 shared broker나 cross-instance fan-out이 필요합니다. WebSocket 전달은 비내구적이고 중복·누락 가능성을 전제로 합니다.

### 답변 핵심 키워드

`validated projection`, `event-scoped destination`, `fan-out`, `process-local broker`, `Kafka consumer group`, `single-replica invariant`, `nondurable delivery`, `cross-instance broker`

### 백지 구현

**구현 목표**

검증이 끝난 배당 이벤트를 공개 projection과 정확한 destination으로 변환하는 순수 컴포넌트를 작성하고, 주어진 배치 토폴로지에서 전달 가능 여부를 판정한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record OddsChanged(
    String eventId,
    String marketId,
    String selectionId,
    String previousOdds,
    String newOdds,
    Instant changedAt) {}

record OddsUpdate(
    String eventId,
    String marketId,
    String selectionId,
    String previousOdds,
    String newOdds,
    Instant changedAt) {}

record Publication(String destination, OddsUpdate payload) {}

final class OddsPublicationPlanner {
  Publication plan(OddsChanged validatedEvent) {
    throw new UnsupportedOperationException("직접 구현");
  }
}

final class LocalBrokerTopology {
  boolean canGuaranteeDelivery(
      String consumingReplica,
      Set<String> subscriberReplicas) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 이미 이벤트 계약을 통과한 `OddsChanged`
- 출력: `/topic/odds/{eventId}` destination과 공개 `OddsUpdate`
- 토폴로지 입력: 이벤트를 소비한 replica와 해당 destination의 subscriber가 연결된 replica 집합

**반드시 만족해야 할 조건**

- destination은 정확히 하나의 event ID segment로 구성한다.
- projection은 필요한 공개 필드만 복사하며 원본 객체를 수정하지 않는다.
- 검증되지 않은 이벤트를 이 계층에서 보정하거나 조용히 drop하지 않는다.
- process-local broker에서 소비 replica에 subscriber가 없으면 다른 replica의 subscriber 전달을 보장할 수 없다고 판정한다.
- 복제본 하나일 때만 현재 구조의 모든 로컬 구독자에게 전달 가능한 불변조건을 설명할 수 있어야 한다.

**경계 조건**

- event ID가 null 또는 비정규형인 입력은 상위 계약 오류로 간주
- subscriber가 0명·1명·여러 명인 경우
- 같은 event를 여러 client가 구독한 경우
- 소비 replica와 subscriber replica 집합이 겹치지 않는 경우
- 재연결 중 이벤트가 도착하는 경우

**실패 조건**

- 검증 전에 공개 projection을 생성해 잘못된 identity가 destination에 들어가는 오류
- 전역 destination 사용으로 모든 client가 불필요한 이벤트를 받는 설계
- 여러 replica인데 local broker만 사용하면서 전달 보장을 주장하는 오류
- WebSocket 전송 실패를 Kafka business state 성공과 동일시하는 오류

**필요한 제약**

- 메시지 broker 자체나 네트워크 전송을 구현하지 않는다.
- durable queue, replay store, exactly-once는 범위 밖이다.
- 현재 프로젝트의 단일 replica 계약을 기본 가정으로 하되 확장 대안을 설명한다.

### 구현 후 자가 검증

- [ ] 정상 이벤트가 정확한 event destination과 동일 값의 projection으로 변환된다.
- [ ] projection 변경이 원본 이벤트를 변경하지 않는다.
- [ ] 서로 다른 event ID가 서로 다른 destination을 만든다.
- [ ] 같은 event의 여러 구독자는 동일 payload를 받는 모델로 표현된다.
- [ ] 소비 replica와 구독 replica가 다르면 현재 local broker로 전달을 보장할 수 없다고 판정한다.
- [ ] 재연결·중복 소비·일시적 전송 실패에서 durable 또는 exactly-once를 잘못 주장하지 않는다.
- [ ] 수평 확장 대안이 shared broker 또는 명시적 cross-instance fan-out 경계를 포함한다.

### 구현 후 설명할 것

- 원본 이벤트와 외부 projection을 분리한 계약·보안상 이유
- event-scoped destination이 대역폭과 구독 관리에 주는 trade-off
- Kafka partition ownership과 WebSocket connection ownership이 다른 문제라는 점
- 단일 replica가 단순한 운영 제약이 아니라 현재 전달 정확성의 invariant인 이유
- 비내구 실시간 알림과 HTTP authoritative refresh를 함께 두는 설계

### 원본 확인 위치

- Thread 13
- Thread 16
- 커밋: `feat(websocket): publish odds updates`
- Thread 16 문서화 작업: 현재 프로젝트 메모리에서 커밋 제목은 확인되지 않음
- 파일: `src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java`
- 문서: `docs/realtime-contract.md`
- 문서: `docs/operations.md`
- 관련 Thread: 10, 11, 15

## WS-04 · [Thread 14, 16 / `feat(websocket): project terminal bet updates`, `feat(websocket): publish resolution revisions`; Thread 16 문서화 작업] 소유자 범위 베팅 상태 projection과 정정 순서 계약

**우선순위:** S

### 면접 질문

- 베팅 종료·무효·정정 이벤트를 하나의 `BetStatusUpdate` 계약으로 투영할 때 각 상태에서 반드시 달라져야 하는 필드를 설명해 보세요.
- 사용자별 `/user/queue/bets` 전달에서 이벤트의 `userId`, 인증 principal, Spring user destination이 어떻게 연결되어야 합니까?
- 초기 settled update의 revision number를 0으로 두고, voided에는 revision number가 없으며, revised에는 명시적 양수 revision을 전달하는 이유는 무엇입니까?
- 꼬리 질문: 중복·순서 역전·재연결 누락을 클라이언트가 어떻게 다뤄야 하며, 게이트웨이가 authoritative 상태 저장소가 아닌 이유는 무엇입니까?
- 꼬리 질문: 여러 replica에서 user-scoped delivery가 현재 왜 안전하지 않습니까?

### 30초 모범 답변

게이트웨이는 검증된 도메인 이벤트를 클라이언트용 최신 상태 snapshot으로 투영합니다. settled는 결과·지급액과 revision 0, voided는 환급·사유와 revision 없음, revised는 새 결과·새 지급액·revision ID와 양수 번호를 담습니다. 전달 대상은 검증된 이벤트 `userId`이며 CONNECT에서 만든 canonical principal의 user destination과 일치해야 다른 사용자가 받지 않습니다. Kafka와 WebSocket에는 중복·순서 역전·재연결 누락 가능성이 있으므로 client는 revision을 비교하고 불확실하면 HTTP로 authoritative 상태를 다시 읽어야 합니다. 현재 local broker 구조는 단일 replica를 전제로 합니다.

### 답변 핵심 키워드

`owner-scoped delivery`, `canonical principal`, `state projection`, `settled revision 0`, `voided terminal rule`, `monotonic revision`, `duplicate tolerance`, `HTTP reconciliation`, `single replica`

### 백지 구현

**구현 목표**

세 종류의 검증된 베팅 이벤트를 하나의 client projection으로 변환하고, 해당 projection의 정확한 사용자 목적지를 계획하는 순수 로직을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
sealed interface BetEvent permits BetSettled, BetVoided, BetResolutionRevised {
  String betId();
  String userId();
  String eventId();
}

record MoneyView(long amount, String currency) {}

record BetStatusUpdate(
    String betId,
    String userId,
    String eventId,
    String status,
    String result,
    MoneyView amount,
    String reason,
    String revisionId,
    Long revisionNumber,
    Instant updatedAt) {}

record UserPublication(
    String userId,
    String destination,
    BetStatusUpdate payload) {}

final class BetStatusProjector {
  BetStatusUpdate project(BetEvent validatedEvent) {
    throw new UnsupportedOperationException("직접 구현");
  }

  UserPublication publication(BetEvent validatedEvent) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 이벤트 계약 검증을 끝낸 `BetSettled`, `BetVoided`, `BetResolutionRevised` 중 하나
- 출력: `BetStatusUpdate`와 소유자 `userId`, `/queue/bets` 목적지
- 실제 STOMP client는 `/user/queue/bets`를 구독한다.

**반드시 만족해야 할 조건**

- settled는 `status=SETTLED`, 결과, payout, revision number 0, settled 시각을 사용한다.
- voided는 `status=VOIDED`, refund, reason, null result·revision ID·revision number, voided 시각을 사용한다.
- revised는 `status=SETTLED`, 새 결과·새 payout, revision ID·revision number, revised 시각을 사용한다.
- publication user ID는 payload와 동일한 검증된 이벤트 user ID여야 한다.
- 다른 사용자의 ID나 현재 thread-local 인증을 이벤트 소유자 대신 사용하지 않는다.
- 원본 이벤트와 금액 객체를 외부 projection에서 수정할 수 없게 한다.

**경계 조건**

- 0 금액 payout 또는 refund
- LOST→WON, WON→LOST 정정
- 동일 revision 중복 도착
- 더 작은 revision이 늦게 도착
- voided 뒤 revised처럼 도메인 계약상 허용되지 않을 수 있는 순서
- 구독자가 일시적으로 연결되지 않은 경우

**실패 조건**

- event user ID와 전달 대상 user ID 불일치로 인한 정보 유출
- settled·voided·revised에서 null 가능 필드를 혼동하는 projection 오류
- 도착 순서를 business state로 간주해 오래된 revision이 최신 상태를 덮는 오류
- 재연결 뒤 누락된 메시지를 gateway가 자동 replay한다고 가정하는 오류
- 여러 replica에서 subscriber가 없는 process가 이벤트를 소비하는 전달 누락

**필요한 제약**

- 이 문제에서는 이벤트 의미 검증 자체를 다시 구현하지 않는다.
- WebSocket 전달을 durable store로 만들지 않는다.
- 클라이언트 상태 병합 알고리즘 전체 대신 projection과 전달 계획까지만 구현한다.

### 구현 후 자가 검증

- [ ] settled·voided·revised 각각의 status, amount, reason, revision, timestamp가 계약대로다.
- [ ] payload의 user ID와 publication 대상 user ID가 항상 같다.
- [ ] 다른 사용자 publication을 생성할 수 있는 외부 파라미터가 없다.
- [ ] 동일 이벤트를 두 번 투영해도 값이 결정적이다.
- [ ] 클라이언트 관점에서 더 큰 revision만 적용하고 중복을 무시할 수 있는 정보가 존재한다.
- [ ] voided의 revision 없음과 settled의 revision 0을 구분했다.
- [ ] 누락·중복·순서 역전 시 HTTP 재조회가 필요한 비내구 경계를 설명할 수 있다.
- [ ] 현재 여러 replica에서 user-scoped 전달을 보장하지 못하는 이유를 확인했다.

### 구현 후 설명할 것

- 서로 다른 도메인 이벤트를 하나의 client snapshot 계약으로 합친 이유
- 검증된 event user ID를 유일한 routing key로 사용하는 보안 판단
- revision 0·null·양수의 의미와 client 병합 규칙
- 서버가 중복 제거 상태를 보유하지 않고 client reconciliation을 요구한 trade-off
- 단일 replica 제한을 제거하려면 user destination을 공유할 broker가 필요한 이유

### 원본 확인 위치

- Thread 14
- Thread 16
- 커밋: `feat(websocket): project terminal bet updates`
- 커밋: `feat(websocket): publish terminal bet updates`
- 커밋: `feat(websocket): publish resolution revisions`
- 커밋: `test(websocket): verify revision projections`
- Thread 16 문서화 작업: 현재 프로젝트 메모리에서 커밋 제목은 확인되지 않음
- 파일: `src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/BetStatusUpdateTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java`
- 문서: `docs/realtime-contract.md`
- 관련 Thread: 3, 8, 10, 11, 15
