# 프로토콜 경계와 리소스 수명

이 문서는 transport 입력·출력에서 면접 가치가 높은 세 지점을 다룬다. 각 백지 구현의 타입은 면접용으로 축소한 인터페이스이며 원본 구현의 복사본이 아니다.

---

## [G02 / feat: add bounded incremental TCP framing] 점진적 TCP 프레임 파서와 버퍼 수명

### 면접 질문

TCP 수신 한 번이 애플리케이션 메시지 하나와 일치한다고 가정하면 어떤 문제가 생깁니까? 이 프로젝트의 length-prefixed parser가 fragmentation과 coalescing을 동시에 처리하면서도 메모리 사용을 16,388바이트 이내로 제한하려면 어떤 상태를 가져야 합니까?

꼬리 질문:

- 길이 0이나 16,384바이트 초과는 왜 terminal framing error이고, 완성된 frame 안의 잘못된 JSON·schema는 왜 연결을 유지할 수 있는 message error입니까?
- EOF 시 "버퍼가 비어 있는 정상 종료"와 "frame 일부가 남은 partial EOF"를 어떻게 구분합니까?
- Netty가 inbound `ByteBuf`를 자동 해제하더라도 parser가 별도로 보유한 cumulation buffer를 직접 해제해야 하는 이유는 무엇입니까?
- 여러 frame이 한 read에 합쳐져 들어왔을 때 첫 frame의 오류가 다음 frame 처리에 미치는 범위를 어떻게 정합니까?

### 30초 모범 답변

TCP는 바이트 스트림이라 read 경계가 메시지 경계가 아닙니다. 그래서 4바이트 길이 헤더와 현재 payload 길이, 누적 바이트를 channel별 상태로 유지하고, 필요한 바이트만 복사해 fragmented frame을 이어 붙이며 한 read 안의 여러 frame은 순서대로 반복 처리합니다. 길이 자체가 잘못되면 이후 경계를 신뢰할 수 없어 연결을 닫고, 경계가 확정된 frame의 내용 오류는 그 frame만 소비한 뒤 다음 frame을 받을 수 있습니다. 종료·예외·handler 제거 모든 경로에서 parser-owned buffer를 한 번만 해제하는 것이 핵심 invariant입니다.

### 답변 핵심 키워드

TCP byte stream, fragmentation, coalescing, length prefix, per-channel state, bounded cumulation, recoverable message error, terminal framing error, partial EOF, ownership, exactly-once release

### 백지 구현

#### 구현 목표

4바이트 big-endian unsigned length prefix를 사용하는 점진 decoder를 구현한다. 여러 번 나뉘어 들어온 frame과 한 번에 합쳐진 여러 frame을 모두 처리하고, framing 오류와 완성된 메시지의 검증 오류를 구분한다.

#### 면접용 인터페이스

```java
import java.nio.ByteBuffer;
import java.util.List;

public interface PayloadValidator {
    Validation validate(ByteBuffer readOnlyPayload);
}

public enum Validation {
    VALID,
    INVALID_MESSAGE
}

public sealed interface DecodeEvent {
    record Payload(byte[] bytes) implements DecodeEvent {}
    record MessageError() implements DecodeEvent {}
    record TerminalFrameError(long declaredLength) implements DecodeEvent {}
    record NeedMoreBytes() implements DecodeEvent {}
    record CleanEof() implements DecodeEvent {}
    record PartialEof(int bufferedBytes) implements DecodeEvent {}
}

public final class LengthPrefixedDecoder implements AutoCloseable {
    public static final int MAX_PAYLOAD_BYTES = 16_384;
    public static final int MAX_BUFFERED_BYTES = 16_388;

    public LengthPrefixedDecoder(PayloadValidator validator) {
        // 직접 구현
    }

    public List<DecodeEvent> feed(ByteBuffer source) {
        // 직접 구현
        return null;
    }

    public DecodeEvent onEof() {
        // 직접 구현
        return null;
    }

    public int bufferedBytes() {
        // 직접 구현
        return 0;
    }

    public boolean isTerminal() {
        // 직접 구현
        return false;
    }

    @Override
    public void close() {
        // 직접 구현
    }
}
```

#### 입력과 출력

- 입력: 임의 크기로 분할되거나 여러 frame이 합쳐진 `ByteBuffer` 조각.
- 출력: 완성된 payload, 완성된 frame의 message error, terminal frame error, 추가 바이트 필요, EOF 상태 중 해당 이벤트.
- validator에는 현재 frame의 payload만 read-only view 또는 복사본으로 전달한다.

#### 반드시 만족해야 할 조건

- payload 길이는 1~16,384바이트만 허용한다.
- parser가 소유하는 누적 바이트는 헤더 포함 16,388바이트를 넘지 않는다.
- 하나의 `feed`에서 여러 frame을 순서대로 내보낼 수 있다.
- validator가 `INVALID_MESSAGE`를 반환한 frame은 정확히 소비하고 다음 frame을 계속 처리할 수 있다.
- 잘못된 길이를 읽은 뒤에는 terminal 상태가 되며 추가 입력을 처리하지 않는다.
- `close`, clean EOF, partial EOF를 여러 번 호출해도 내부 자원이 중복 해제되지 않는다.

#### 경계 조건

- 헤더가 1·2·3바이트씩 나뉘어 도착한다.
- 헤더 직후 payload가 0바이트인 상태에서 다음 read를 기다린다.
- 최대 길이 frame이 1바이트 단위로 들어온다.
- 하나의 입력 조각에 `frame A + frame B + frame C 일부`가 들어온다.
- 완성된 invalid message 뒤에 valid frame이 같은 입력 조각에 들어온다.
- 아무 바이트도 남지 않은 EOF와 헤더·payload 일부가 남은 EOF가 구분된다.

#### 실패 조건

- 길이 0, 16,385 이상, terminal 이후 추가 입력.
- 내부 상태와 실제 buffered byte 수가 불일치한다.
- 검증을 위해 payload 경계를 넘어 다음 frame 바이트를 함께 전달한다.
- EOF·예외·close 경로에서 내부 buffer가 남거나 두 번 해제된다.

#### 필요한 제약

- thread-safe로 만들 필요는 없다. decoder 인스턴스 하나는 한 connection/channel owner만 사용한다고 가정한다.
- JSON parser 자체는 구현하지 않는다. payload 경계와 validator 호출 계약이 과제의 범위다.

### 구현 후 자가 검증

- [ ] 4바이트보다 짧은 헤더 입력에서 message를 만들지 않는다.
- [ ] 정확히 16,384바이트 payload를 받고, 16,385바이트 선언은 terminal 처리한다.
- [ ] 한 frame을 모든 가능한 두 조각 경계로 나눠도 결과가 같다.
- [ ] 세 frame을 한 번에 넣어도 원래 순서로 세 이벤트가 나온다.
- [ ] invalid message 하나가 다음 valid frame을 삼키지 않는다.
- [ ] terminal error 이후 validator가 다시 호출되지 않는다.
- [ ] clean EOF와 partial EOF의 buffered byte 수가 맞다.
- [ ] close를 반복해도 상태와 자원 수가 음수가 되지 않는다.
- [ ] 시간 복잡도는 입력 바이트 수에 선형이고, 공간은 frame 최대 크기로 제한된다.

### 구현 후 설명할 것

1. 전체 스트림이 아니라 현재 frame에 필요한 바이트만 보관하도록 한 이유.
2. framing error와 message error의 복구 가능성을 다르게 본 기준.
3. payload를 복사할지 read-only view를 빌려줄지의 수명·성능 trade-off.
4. decoder를 channel별 non-sharable 객체로 두어 동기화를 제거한 판단.
5. 모든 종료 경로를 하나의 release 경로로 모아 중복 해제를 막은 방법.

### 원본 확인 위치

- Thread: G02
- 커밋: `feat: add bounded incremental TCP framing`
- 파일·컴포넌트: `src/main/java/arena/CompleteFrame.java`, `CompleteFrame.Metrics`
- 함수: `channelRead0`, `copy`, `needMore`, `reject`, `channelInactive`, `exceptionCaught`, `handlerRemoved`, `releaseBuffer`, `protocolError`
- 테스트·시나리오: `src/test/java/arena/CompleteFrameTest.java`, `FramingScenario.java`, `ServerIntegrationTest.java`
- 관련 Thread: G01, G09, G12

---

## [G09 / feat(udp): add bounded realtime data and independent endpoint binding] TCP control과 UDP data plane의 인증 경계

### 면접 질문

왜 HELLO·room lifecycle 같은 control은 TCP에 남기고 INPUT·SNAPSHOT 같은 realtime 메시지는 UDP로 옮겼습니까? UDP packet에 `session_id`가 들어 있다는 사실만으로 sender를 그 session에 귀속시키면 왜 위험하며, one-time bind token과 실제 source endpoint를 어떤 순서로 검증해야 합니까?

꼬리 질문:

- TCP listener가 port 0으로 bind됐다고 해서 UDP도 같은 숫자를 사용할 수 있다고 가정하면 어떤 startup failure가 생깁니까?
- 1,201바이트 datagram을 JSON parse나 owner mailbox dispatch 전에 버리는 이유는 무엇입니까?
- 이미 관찰된 endpoint와 packet이 주장하는 `session_id`가 충돌할 때 어느 쪽을 routing authority로 삼아야 합니까?
- UDP packet의 `ByteBuf` ownership을 handler가 직접 끝내는 설계에서 double release를 어떻게 막습니까?
- token, resume credential을 로그·evidence에 남기지 않는 정책은 기능 테스트와 어떻게 양립합니까?

### 30초 모범 답변

UDP는 연결이 없으므로 payload의 session claim만 믿으면 다른 endpoint가 세션을 사칭할 수 있습니다. TCP로 발급한 일회용 bind token을 실제 UDP source endpoint와 함께 owner에서 소비해 session에 귀속시키고, 이후 realtime packet은 그 endpoint·session·owner epoch가 모두 맞아야 처리합니다. TCP와 UDP는 서로 다른 port namespace이므로 UDP는 독립적으로 port 0에 bind하고 실제 port를 WELCOME으로 알립니다. 크기 초과나 파싱 불가 datagram은 owner queue에 넣기 전에 버려 비용과 공격 표면을 제한합니다.

### 답변 핵심 키워드

control plane, data plane, connectionless transport, source endpoint, one-time bind token, session claim, owner epoch, independent bind, datagram ceiling, pre-dispatch drop, credential redaction

### 백지 구현

#### 구현 목표

TCP에서 발급한 one-time token으로 UDP sender endpoint를 session에 한 번만 bind하고, 이후 realtime envelope를 안전하게 routing하는 registry를 구현한다. 실제 socket 코드는 제외하고 주소·세션·token 상태 머신만 다룬다.

#### 면접용 인터페이스

```java
import java.net.SocketAddress;
import java.util.Optional;
import java.util.function.Supplier;

public record BindRequest(String sessionId, String bindToken, int ownerEpoch) {}
public record RealtimeEnvelope(
        String type,
        String sessionId,
        String roomId,
        String playerId,
        int ownerEpoch,
        int encodedBytes) {}

public sealed interface BindResult {
    record Bound(String sessionId, SocketAddress endpoint) implements BindResult {}
    record Rejected(String code) implements BindResult {}
}

public sealed interface RouteResult {
    record Dispatch(String sessionId, RealtimeEnvelope envelope) implements RouteResult {}
    record Drop(String reason) implements RouteResult {}
    record Reject(String code) implements RouteResult {}
}

public final class UdpEndpointRegistry {
    public static final int MAX_DATAGRAM_BYTES = 1_200;

    public UdpEndpointRegistry(Supplier<String> tokenSource) {
        // 직접 구현
    }

    public String issueBindToken(String sessionId, int ownerEpoch) {
        // 직접 구현
        return null;
    }

    public BindResult bind(SocketAddress sender, BindRequest request) {
        // 직접 구현
        return null;
    }

    public RouteResult route(SocketAddress sender, RealtimeEnvelope envelope) {
        // 직접 구현
        return null;
    }

    public void invalidateSession(String sessionId) {
        // 직접 구현
    }

    public Optional<SocketAddress> endpointOf(String sessionId) {
        // 직접 구현
        return Optional.empty();
    }
}
```

#### 입력과 출력

- 입력: TCP 측에서 생성한 session과 token, UDP source 주소, bind 또는 realtime envelope.
- 출력: bind 성공·거절 또는 owner dispatch·drop·protocol rejection 결정.
- `encodedBytes`는 JSON parse 전에 transport 계층이 관찰한 datagram 크기라고 가정한다.

#### 반드시 만족해야 할 조건

- token은 발급된 session과 owner epoch에만 유효하고 성공한 bind에서 한 번 소비된다.
- 다른 session의 token, 이미 소비된 token, 잘못된 epoch는 bind하지 않는다.
- bind 전 INPUT과 다른 endpoint에서 온 INPUT은 dispatch하지 않는다.
- 이미 bind된 endpoint와 envelope의 session claim이 일치해야 한다.
- 1,200바이트를 초과한 datagram은 session lookup이나 owner dispatch 전에 drop한다.
- `invalidateSession` 후 기존 endpoint와 token은 모두 무효다.
- token 원문은 결과 객체·예외 메시지·조회 API로 반환하지 않는다.

#### 경계 조건

- 동일 token으로 두 endpoint가 거의 동시에 bind를 시도한다.
- session A가 session B의 token을 제출한다.
- 올바른 endpoint가 잘못된 `sessionId`를 주장한다.
- bind 성공 직후 같은 token을 다시 제출한다.
- 정확히 1,200바이트와 1,201바이트를 구분한다.
- session invalidate 뒤 늦게 도착한 datagram이 있다.

#### 실패 조건

- 거절된 bind가 기존 endpoint를 덮어쓴다.
- token 검증 전에 claimed session을 endpoint에 연결한다.
- oversize packet이 dispatch 결과를 만든다.
- invalidation 뒤 token이나 endpoint mapping이 남는다.
- token 값이 오류 문자열에 포함된다.

#### 필요한 제약

- token의 암호학적 생성 자체는 `Supplier<String>`의 책임으로 둔다.
- 한 registry 인스턴스의 mutation은 owner thread에서만 일어난다고 가정하거나, 선택한 동기화 정책을 명시한다.
- room·player 권한의 상세 규칙은 외부 command handler가 처리한다. 이 과제는 UDP source/session 귀속까지만 담당한다.

### 구현 후 자가 검증

- [ ] 정상 token은 정확히 한 번만 bind에 성공한다.
- [ ] 다른 session token, 잘못된 epoch, 재사용 token이 mapping을 바꾸지 않는다.
- [ ] bind 전·다른 endpoint·invalidated endpoint packet이 dispatch되지 않는다.
- [ ] 1,200바이트는 다음 검증 단계로 가고 1,201바이트는 즉시 drop된다.
- [ ] 거절·drop 경로에서 session 상태가 변하지 않는다.
- [ ] 조회·로그용 결과에 token이 노출되지 않는다.
- [ ] invalidate 후 endpoint와 미사용 token 수가 0이 된다.
- [ ] lookup은 평균 O(1), 저장 공간은 active session 수에 선형이다.

### 구현 후 설명할 것

1. payload의 `session_id`보다 실제 source endpoint를 먼저 고려해야 하는 이유.
2. bind token을 일회용으로 만든 이유와 재전송 신뢰성 trade-off.
3. TCP와 UDP port를 독립 bind하고 실제 UDP port를 광고한 이유.
4. oversize·malformed datagram을 owner handoff 전에 버리는 방어선.
5. credential을 기능 상태와 관측 데이터에서 분리한 방법.

### 원본 확인 위치

- Thread: G09
- 커밋: `feat(udp): add bounded realtime data and independent endpoint binding`
- 파일·컴포넌트: `src/main/java/arena/UdpData.java`, `ArenaServer.java`, `TcpClient.java`
- 함수: `UdpData.channelRead0`, `UdpData.bytes`, `ArenaServer.handleUdp`, `ArenaServer.handleCommand`, `TcpClient.sendTcp`, `TcpClient.sendUdp`
- 테스트·시나리오: `src/test/java/arena/UdpBoundaryTest.java`, `UdpScenario.java`, `UdpFaultProxy.java`
- 관련 Thread: G02, G05, G11, G12

---

## [G12 / feat: bound outbound snapshot ownership] Slow consumer와 bounded outbound ownership

### 면접 질문

클라이언트 하나의 socket이 계속 not-writable인 동안 매 tick snapshot을 생성하면 어떤 종류의 메모리 문제가 생깁니까? 이 프로젝트가 connection마다 ordered message 최대 64개, queued FULL 1개, queued DELTA 1개, transport in-flight buffer 1개로 나눈 이유를 설명하십시오.

꼬리 질문:

- queued DELTA가 새 DELTA에 supersede될 때 누가 기존 `ByteBuf`를 해제해야 합니까?
- `writeAndFlush`로 넘긴 in-flight buffer를 queue가 close 시 해제하면 왜 위험합니까?
- FULL이 대기 중일 때 이전 FULL과 DELTA를 함께 버릴 수 있는 이유와, ordered control을 coalesce하면 안 되는 이유는 무엇입니까?
- 같은 종류 snapshot이 아직 in-flight일 때 새 snapshot을 억제하는 정책의 장단점은 무엇입니까?
- terminal backpressure 메시지를 별도 무한 queue에 넣지 않고 기존 bound 안에서 처리해야 하는 이유는 무엇입니까?

### 30초 모범 답변

slow consumer를 transport에 그대로 맡기면 애플리케이션이 snapshot buffer를 무한 생성하거나 transport queue에 떠넘길 수 있습니다. 그래서 순서를 보존해야 하는 control은 bounded FIFO로 두고, 대체 가능한 FULL·DELTA는 종류별 최신 하나만 보관합니다. 아직 queue가 소유한 superseded buffer는 queue가 정확히 한 번 release하지만, write로 넘긴 in-flight buffer는 Netty 소유이므로 실제 completion callback만 retire합니다. close는 queued 것만 정리하고 in-flight는 callback에 맡겨 ownership 경계를 지키는 것이 핵심입니다.

### 답변 핵심 키워드

slow consumer, backpressure, bounded FIFO, coalescing, supersession, queued vs in-flight, ownership transfer, completion callback, exactly-once release, writability gate

### 백지 구현

#### 구현 목표

순서 보존 메시지와 대체 가능한 FULL·DELTA snapshot을 제한된 슬롯에 저장하고, transport가 writable일 때 한 개씩 detach하는 queue를 구현한다. 메시지는 `release()`가 정확히 한 번 호출돼야 하는 객체로 모델링한다.

#### 면접용 인터페이스

```java
import java.util.Optional;

public interface ReleasableMessage {
    MessageKind kind();
    int bytes();
    boolean terminal();
    void release();
}

public enum MessageKind {
    ORDERED,
    FULL,
    DELTA
}

public enum OfferResult {
    ACCEPTED,
    COALESCED,
    BACKPRESSURE_TERMINAL,
    CLOSED
}

public final class BoundedOutboundQueue implements AutoCloseable {
    public static final int ORDERED_LIMIT = 64;

    public OfferResult offer(ReleasableMessage message) {
        // 직접 구현
        return null;
    }

    public Optional<ReleasableMessage> detachNext(boolean tcpWritable, boolean udpWritable) {
        // 직접 구현
        return Optional.empty();
    }

    public void complete(ReleasableMessage inFlight, boolean success) {
        // 직접 구현
    }

    public QueueView view() {
        // 직접 구현
        return null;
    }

    @Override
    public void close() {
        // 직접 구현
    }
}

public record QueueView(
        int ordered,
        int queuedFull,
        int queuedDelta,
        int inFlight,
        long retainedBytes,
        boolean terminal,
        boolean closed) {}
```

#### 입력과 출력

- 입력: `ORDERED`, `FULL`, `DELTA` 메시지, TCP/UDP writability, transport completion.
- 출력: admission 결과, transport로 넘길 다음 메시지, 현재 bound 관측값.
- `detachNext`가 반환한 객체는 그 순간 transport 소유로 넘어간다고 가정한다.

#### 반드시 만족해야 할 조건

- queued ordered는 최대 64개이며 FIFO 순서를 보존한다.
- queued FULL과 DELTA는 각각 최대 1개다.
- 새 FULL은 기존 queued FULL·DELTA를 대체할 수 있다.
- 새 DELTA는 기존 queued DELTA만 대체할 수 있다.
- superseded 또는 close로 취소된 queued 객체는 정확히 한 번 release한다.
- detach된 in-flight 객체는 `complete` 전까지 queue가 release·수정하지 않는다.
- in-flight는 최대 1개다.
- ordered 메시지가 준비돼 있으면 snapshot보다 먼저 보낸다.
- terminal 상태가 결정된 뒤 새 일반 메시지를 받지 않는다.
- `close` 후 queued count와 queued bytes는 0이 되지만, 이미 transport 소유인 객체는 completion 전까지 임의 해제하지 않는다.

#### 경계 조건

- ordered 63개 상태에서 한 개를 더 넣는 경우와 그다음 입력.
- DELTA → DELTA → FULL → DELTA 순서로 빠르게 들어온다.
- FULL이 in-flight인 동안 새 FULL이 생성된다.
- not-writable 상태에서 snapshot 수백 개가 생성된다.
- close와 completion이 인접하게 발생한다.
- transport write가 실패한다.

#### 실패 조건

- superseded 객체가 해제되지 않거나 두 번 해제된다.
- in-flight 객체를 queued 객체처럼 release한다.
- close 이후 새로운 buffer를 보관한다.
- coalescing 때문에 ordered control 순서가 바뀐다.
- retained buffer 수가 `64 ordered + 1 FULL + 1 DELTA + 1 in-flight`를 넘는다.

#### 필요한 제약

- `ReleasableMessage.release()`는 테스트에서 중복 호출을 감지하도록 구현돼 있다고 가정한다.
- 실제 Netty API 호출은 구현하지 않는다. ownership transfer 시점만 `detachNext`/`complete`로 모델링한다.
- thread-safe로 구현할 경우 모든 상태 전이의 단일 선형화 지점을 설명해야 한다. owner-confined로 구현할 경우 그 전제를 문서화한다.

### 구현 후 자가 검증

- [ ] ordered 64개가 정확한 FIFO로 나가며 bound를 넘지 않는다.
- [ ] DELTA가 여러 번 들어와도 queued DELTA는 하나이고 이전 객체가 한 번씩 해제된다.
- [ ] FULL admission 후 불필요해진 queued FULL·DELTA가 모두 정리된다.
- [ ] in-flight 객체의 release count는 completion 전 0이고 completion 처리와 transport 계약에 맞다.
- [ ] not-writable 동안 snapshot 생성 횟수와 무관하게 retained slot 수가 일정하다.
- [ ] close가 queued 객체를 모두 정리하고, 반복 close가 중복 release를 만들지 않는다.
- [ ] terminal 이후 새 메시지가 들어와도 retained work가 늘지 않는다.
- [ ] `view()`의 retained bytes가 실제 queued+in-flight 합과 일치한다.
- [ ] offer·detach·complete는 queue 길이에 대해 O(1), ordered dequeue는 O(1)이다.

### 구현 후 설명할 것

1. ordered와 snapshot을 같은 FIFO에 넣지 않은 이유.
2. FULL·DELTA supersession 규칙이 replication 의미와 맞는지 판단한 기준.
3. queue-owned와 transport-owned buffer의 ownership transfer 시점.
4. close와 asynchronous completion이 겹칠 때 exactly-once cleanup을 보장한 방법.
5. 새 snapshot 억제와 최신 snapshot coalescing 사이의 trade-off.

### 원본 확인 위치

- Thread: G12
- 커밋: `feat: bound outbound snapshot ownership`
- 파일·컴포넌트: `src/main/java/arena/OutboundQueue.java`, `ArenaServer.Peer.outbound`, `OutboundQueue.Metrics`
- 함수: `OutboundQueue.admit`, `flushWhenReady`, `close`, `view`
- 테스트·시나리오: `src/test/java/arena/BackpressureScenario.java`, `OutboundQueueTest.java`, `src/test/resources/G12.json`
- 관련 Thread: G01, G02, G08, G09, G10, G14
