# 프로토콜 경계와 리소스 수명

이 문서는 transport 입력·출력에서 면접 가치가 높은 세 지점을 다룬다. 각 백지 구현의 타입은 면접용으로 축소한 인터페이스이며 원본 구현의 복사본이 아니다.

---

## [G02 / feat: add bounded incremental TCP framing] 점진적 TCP 프레임 파서와 버퍼 수명

### 면접 질문

TCP 수신 한 번이 애플리케이션 메시지 하나와 일치한다고 가정하면 어떤 문제가 생깁니까? 이 프로젝트의 length-prefixed parser가 fragmentation과 coalescing을 동시에 처리하면서도 메모리 사용을 16,388바이트 이내로 제한하려면 어떤 상태를 가져야 합니까?

꼬리 질문:

- 길이 0이나 16,384바이트 초과는 왜 terminal framing error이고, 완성된 frame 안의 잘못된 JSON·schema는 왜 연결을 유지할 수 있는 message error입니까?
  - 모범답변: invalid length 뒤에는 다음 frame 경계를 신뢰할 수 없지만, 길이가 유효한 완성 frame은 끝 위치가 확정돼 내용만 버릴 수 있다. 원본은 전자에서 auto-read를 끄고 terminal close하며 후자는 오류 응답 뒤 같은 channel의 다음 frame을 처리한다.
- EOF 시 "버퍼가 비어 있는 정상 종료"와 "frame 일부가 남은 partial EOF"를 어떻게 구분합니까?
  - 모범답변: parser-owned buffer의 writer index가 0이면 clean EOF, 헤더나 payload 일부가 있으면 partial EOF다. `channelInactive`에서 최초 end reason만 기록하고 buffer를 공통 release 경로로 보낸다.
- Netty가 inbound `ByteBuf`를 자동 해제하더라도 parser가 별도로 보유한 cumulation buffer를 직접 해제해야 하는 이유는 무엇입니까?
  - 모범답변: `SimpleChannelInboundHandler`가 해제하는 것은 각 inbound message의 reference이고, handler가 새로 만든 `accumulated`는 별도 ownership이다. inactive·exception·handler removal에서 직접 한 번 해제하지 않으면 direct memory가 남는다.
- 여러 frame이 한 read에 합쳐져 들어왔을 때 첫 frame의 오류가 다음 frame 처리에 미치는 범위를 어떻게 정합니까?
  - 모범답변: message error면 현재 frame을 정확히 소비하고 buffer state를 초기화해 같은 read의 suffix를 계속 처리한다. terminal framing error면 loop와 auto-read를 멈춰 suffix를 해석하지 않는다.

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
        this.validator = java.util.Objects.requireNonNull(validator);
        // peer가 선언한 크기로 grow하지 않는 고정 상한 저장소다.
        this.accumulated = ByteBuffer.allocate(MAX_BUFFERED_BYTES);
    }

    public List<DecodeEvent> feed(ByteBuffer source) {
        java.util.Objects.requireNonNull(source);
        if (terminal || closed) return List.of();

        List<DecodeEvent> events = new java.util.ArrayList<>();
        while (source.hasRemaining() && !terminal) {
            if (payloadLength < 0) {
                copy(source, 4 - accumulated.position());
                if (accumulated.position() < 4) break;
                long declared = Integer.toUnsignedLong(accumulated.getInt(0));
                if (declared == 0 || declared > MAX_PAYLOAD_BYTES) {
                    terminal = true;
                    terminalEvent = new DecodeEvent.TerminalFrameError(declared);
                    events.add(terminalEvent);
                    releaseBuffer();
                    break;
                }
                payloadLength = (int) declared;
            }

            copy(source, 4 + payloadLength - accumulated.position());
            if (accumulated.position() < 4 + payloadLength) break;

            byte[] payload = new byte[payloadLength];
            ByteBuffer view = accumulated.asReadOnlyBuffer();
            view.position(4).limit(4 + payloadLength);
            view.get(payload);
            Validation validation = validator.validate(
                    ByteBuffer.wrap(payload).asReadOnlyBuffer());
            events.add(validation == Validation.VALID
                    ? new DecodeEvent.Payload(payload)
                    : new DecodeEvent.MessageError());
            // 완성 frame만 비우므로 같은 source의 suffix는 다음 loop에서 처리된다.
            accumulated.clear();
            payloadLength = -1;
        }
        if (!terminal && (events.isEmpty() || accumulated.position() != 0))
            events.add(new DecodeEvent.NeedMoreBytes());
        return List.copyOf(events);
    }

    public DecodeEvent onEof() {
        if (eofEvent != null) return eofEvent;
        if (terminalEvent != null) return terminalEvent;
        int buffered = bufferedBytes();
        eofEvent = buffered == 0
                ? new DecodeEvent.CleanEof()
                : new DecodeEvent.PartialEof(buffered);
        terminal = true;
        releaseBuffer();
        return eofEvent;
    }

    public int bufferedBytes() {
        return accumulated == null ? 0 : accumulated.position();
    }

    public boolean isTerminal() {
        return terminal || closed;
    }

    @Override
    public void close() {
        if (closed) return;
        closed = true;
        terminal = true;
        releaseBuffer();
    }

    private final PayloadValidator validator;
    private ByteBuffer accumulated;
    private int payloadLength = -1;
    private boolean terminal;
    private boolean closed;
    private DecodeEvent eofEvent;
    private DecodeEvent terminalEvent;

    private void copy(ByteBuffer source, int needed) {
        int count = Math.min(needed, source.remaining());
        ByteBuffer slice = source.slice();
        slice.limit(count);
        accumulated.put(slice);
        source.position(source.position() + count);
    }

    private void releaseBuffer() {
        if (accumulated == null) return;
        accumulated = null;
        payloadLength = -1;
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
   - 모범답변: TCP stream 전체는 끝이 없으므로 보관하면 peer 속도에 따라 메모리가 무한해진다. 현재 header와 검증된 payload 길이까지만 누적하면 connection당 공간이 16,388바이트로 제한된다.
2. framing error와 message error의 복구 가능성을 다르게 본 기준.
   - 모범답변: 다음 byte의 frame 위치를 확실히 아는지가 기준이다. 완성 frame의 JSON/schema 오류는 경계가 살아 있지만 invalid length나 partial EOF는 동기화를 잃는다.
3. payload를 복사할지 read-only view를 빌려줄지의 수명·성능 trade-off.
   - 모범답변: view는 복사를 줄이지만 다음 buffer clear·release 전까지만 유효해 소비자가 보관하면 위험하다. 축소 구현은 명확한 ownership을 위해 byte copy를 반환하고, 원본은 동기 검증에 borrowed NIO view를 사용한다.
4. decoder를 channel별 non-sharable 객체로 두어 동기화를 제거한 판단.
   - 모범답변: payload length와 cumulation은 channel stream별 순서 상태이므로 공유할 이유가 없다. Netty event-loop confinement을 사용하면 lock 없이 상태 전이를 직렬화하고 channel 간 오류도 격리한다.
5. 모든 종료 경로를 하나의 release 경로로 모아 중복 해제를 막은 방법.
   - 모범답변: `releaseBuffer`가 null을 확인한 뒤 reference와 length state를 비우는 멱등 함수이고 inactive·exception·removed가 모두 이를 호출한다. 원본은 예상 밖 추가 owner가 있으면 ref count invariant도 실패시킨다.

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
  - 모범답변: TCP와 UDP는 독립 namespace라 같은 숫자가 사용 가능하다는 보장이 없고 다른 UDP socket이 이미 점유했을 수 있다. 원본은 UDP도 별도로 port 0에 bind한 뒤 실제 할당 port를 WELCOME으로 광고한다.
- 1,201바이트 datagram을 JSON parse나 owner mailbox dispatch 전에 버리는 이유는 무엇입니까?
  - 모범답변: protocol 상한을 넘은 입력에 parsing·allocation·owner queue 비용을 쓰지 않고 공격 표면을 transport 경계에서 제한하기 위해서다. `UdpData`는 readable length를 먼저 보고 oversize metric만 남긴다.
- 이미 관찰된 endpoint와 packet이 주장하는 `session_id`가 충돌할 때 어느 쪽을 routing authority로 삼아야 합니까?
  - 모범답변: bind 과정에서 인증된 observed endpoint mapping이 authority다. payload claim이 다르면 dispatch하지 않고 `UDP_BIND_INVALID`로 처리해야 다른 session 사칭을 막을 수 있다.
- UDP packet의 `ByteBuf` ownership을 handler가 직접 끝내는 설계에서 double release를 어떻게 막습니까?
  - 모범답변: 원본은 `super(false)`로 자동 release를 끄고 `finally` 한 곳에서 packet을 정확히 한 번 release한다. 이후 ref count가 0이 아니면 예상 밖 owner가 있다고 즉시 실패시킨다.
- token, resume credential을 로그·evidence에 남기지 않는 정책은 기능 테스트와 어떻게 양립합니까?
  - 모범답변: 테스트는 token 원문 대신 발급 수·소비 여부·endpoint mapping·재사용 거절과 같은 상태 전이를 검증한다. 실제 credential은 wire 요청에만 사용하고 metrics와 오류에는 개수나 안정된 코드만 남긴다.

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
        this.tokenSource = java.util.Objects.requireNonNull(tokenSource);
    }

    public String issueBindToken(String sessionId, int ownerEpoch) {
        java.util.Objects.requireNonNull(sessionId);
        String previous = issuedBySession.get(sessionId);
        String token = java.util.Objects.requireNonNull(tokenSource.get());
        if (token.isEmpty() || token.equals(previous) || credentials.containsKey(token))
            throw new IllegalStateException("token source returned an unusable token");
        if (previous != null) credentials.remove(previous);
        credentials.put(token, new Credential(sessionId, ownerEpoch));
        issuedBySession.put(sessionId, token);
        return token;
    }

    public BindResult bind(SocketAddress sender, BindRequest request) {
        java.util.Objects.requireNonNull(sender);
        java.util.Objects.requireNonNull(request);
        Credential credential = credentials.get(request.bindToken());
        Binding existing = bindings.get(request.sessionId());
        if (credential == null
                || !credential.sessionId().equals(request.sessionId())
                || credential.ownerEpoch() != request.ownerEpoch()
                || existing != null
                || endpointOwners.containsKey(sender))
            return new BindResult.Rejected("UDP_BIND_INVALID");

        // 모든 검증 뒤 token 소비와 양방향 endpoint mapping을 함께 commit한다.
        credentials.remove(request.bindToken());
        issuedBySession.remove(request.sessionId(), request.bindToken());
        bindings.put(request.sessionId(),
                new Binding(sender, request.ownerEpoch()));
        endpointOwners.put(sender, request.sessionId());
        return new BindResult.Bound(request.sessionId(), sender);
    }

    public RouteResult route(SocketAddress sender, RealtimeEnvelope envelope) {
        java.util.Objects.requireNonNull(sender);
        java.util.Objects.requireNonNull(envelope);
        if (envelope.encodedBytes() > MAX_DATAGRAM_BYTES)
            return new RouteResult.Drop("DATAGRAM_TOO_LARGE");
        if (envelope.encodedBytes() <= 0)
            return new RouteResult.Drop("DATAGRAM_SIZE_INVALID");
        if (!REALTIME_TYPES.contains(envelope.type()))
            return new RouteResult.Reject("WRONG_TRANSPORT");

        String actualSession = endpointOwners.get(sender);
        if (actualSession == null) return new RouteResult.Drop("UDP_UNBOUND");
        Binding binding = bindings.get(actualSession);
        if (!actualSession.equals(envelope.sessionId())
                || binding == null
                || binding.ownerEpoch() != envelope.ownerEpoch())
            return new RouteResult.Reject("UDP_BIND_INVALID");
        return new RouteResult.Dispatch(actualSession, envelope);
    }

    public void invalidateSession(String sessionId) {
        String token = issuedBySession.remove(sessionId);
        if (token != null) credentials.remove(token);
        Binding binding = bindings.remove(sessionId);
        if (binding != null) endpointOwners.remove(binding.endpoint(), sessionId);
    }

    public Optional<SocketAddress> endpointOf(String sessionId) {
        Binding binding = bindings.get(sessionId);
        return binding == null ? Optional.empty() : Optional.of(binding.endpoint());
    }

    private static final java.util.Set<String> REALTIME_TYPES = java.util.Set.of(
            "INPUT", "SNAPSHOT", "SNAPSHOT_ACK", "PING", "PONG", "UDP_BIND");
    private record Credential(String sessionId, int ownerEpoch) {}
    private record Binding(SocketAddress endpoint, int ownerEpoch) {}

    private final Supplier<String> tokenSource;
    private final java.util.Map<String, Credential> credentials = new java.util.HashMap<>();
    private final java.util.Map<String, String> issuedBySession = new java.util.HashMap<>();
    private final java.util.Map<String, Binding> bindings = new java.util.HashMap<>();
    private final java.util.Map<SocketAddress, String> endpointOwners = new java.util.HashMap<>();
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
   - 모범답변: payload 값은 누구나 주장할 수 있지만 observed endpoint는 성공한 one-time bind에서 session과 결합된 transport 사실이다. endpoint로 실제 session을 찾은 뒤 claim과 epoch를 대조해야 한다.
2. bind token을 일회용으로 만든 이유와 재전송 신뢰성 trade-off.
   - 모범답변: 성공 후 재사용을 막아 다른 endpoint로 binding을 옮기는 replay를 차단한다. 성공 응답이 손실되면 같은 token 재시도가 실패할 수 있으므로 client는 TCP control 상태를 확인하거나 새 token을 받아야 한다.
3. TCP와 UDP port를 독립 bind하고 실제 UDP port를 광고한 이유.
   - 모범답변: 서로 다른 protocol namespace의 가용성을 독립적으로 확인해 startup 충돌을 피하고, OS가 고른 실제 UDP endpoint를 client가 정확히 알게 한다.
4. oversize·malformed datagram을 owner handoff 전에 버리는 방어선.
   - 모범답변: transport handler가 크기와 JSON 형식을 먼저 검사하면 owner mailbox 용량과 game validation quota를 네트워크 잡음이 소비하지 않는다. 이는 authoritative command 검증보다 바깥쪽의 저비용 경계다.
5. credential을 기능 상태와 관측 데이터에서 분리한 방법.
   - 모범답변: token은 credential registry 내부와 필요한 wire 응답에만 있고 lookup·metric에는 binding 수와 오류 코드만 노출한다. invalidation은 endpoint와 미사용 token을 함께 지워 수명도 일치시킨다.

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
  - 모범답변: 아직 queued slot을 소유한 `OutboundQueue`가 old DELTA를 제거하면서 정확히 한 번 release한다. 원본은 queued·retained byte metric을 함께 감소시키고 ref count 0을 확인한다.
- `writeAndFlush`로 넘긴 in-flight buffer를 queue가 close 시 해제하면 왜 위험합니까?
  - 모범답변: detach 시 ownership이 Netty transport로 넘어갔으므로 queue가 release하면 completion path와 double release하거나 전송 중 메모리를 해제한다. close는 queued만 정리하고 in-flight는 실제 promise callback이 retire한다.
- FULL이 대기 중일 때 이전 FULL과 DELTA를 함께 버릴 수 있는 이유와, ordered control을 coalesce하면 안 되는 이유는 무엇입니까?
  - 모범답변: 새 FULL은 완전한 materialization이라 이전 queued FULL과 그 base에 의존한 DELTA를 모두 대체한다. ordered control은 각 event·응답이 독립 의미와 순서를 가지므로 최신 하나로 대체할 수 없다.
- 같은 종류 snapshot이 아직 in-flight일 때 새 snapshot을 억제하는 정책의 장단점은 무엇입니까?
  - 모범답변: transport-owned buffer와 같은 종류의 추가 slot이 겹치지 않아 메모리 상한과 base 순서가 단순해진다. 대신 느린 전송 중 생성된 더 최신 상태를 바로 보관하지 못하므로 다음 publication 또는 FULL 복구에 의존한다.
- terminal backpressure 메시지를 별도 무한 queue에 넣지 않고 기존 bound 안에서 처리해야 하는 이유는 무엇입니까?
  - 모범답변: 포화 알림을 새 무한 queue에 넣으면 backpressure를 보고하는 동안 더 큰 backpressure를 만든다. 원본은 마지막 bounded control slot을 terminal 오류로 사용하거나 즉시 close해 새 work를 차단한다.

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
        java.util.Objects.requireNonNull(message);
        if (closed || terminal) {
            message.release();
            return closed ? OfferResult.CLOSED : OfferResult.BACKPRESSURE_TERMINAL;
        }
        if (message.kind() == MessageKind.ORDERED) {
            if (ordered.size() == ORDERED_LIMIT) {
                terminal = true;
                message.release();
                return OfferResult.BACKPRESSURE_TERMINAL;
            }
            ordered.addLast(message);
            queuedBytes += message.bytes();
            if (message.terminal()) terminal = true;
            return OfferResult.ACCEPTED;
        }

        if (inFlight != null && inFlight.kind() == message.kind()) {
            // 원본처럼 transport-owned 동종 snapshot이 끝나기 전에는 새 slot을 만들지 않는다.
            message.release();
            return OfferResult.COALESCED;
        }

        boolean coalesced = false;
        if (message.kind() == MessageKind.FULL) {
            coalesced |= releaseQueued(full);
            coalesced |= releaseQueued(delta);
            full = message;
            delta = null;
        } else {
            coalesced = releaseQueued(delta);
            delta = message;
        }
        queuedBytes += message.bytes();
        return coalesced ? OfferResult.COALESCED : OfferResult.ACCEPTED;
    }

    public Optional<ReleasableMessage> detachNext(boolean tcpWritable, boolean udpWritable) {
        if (closed || inFlight != null) return Optional.empty();
        ReleasableMessage next = !ordered.isEmpty() ? ordered.peekFirst()
                : full != null ? full : delta;
        if (next == null) return Optional.empty();
        boolean writable = next.kind() == MessageKind.ORDERED
                ? tcpWritable : udpWritable;
        if (!writable) return Optional.empty();

        if (next.kind() == MessageKind.ORDERED) ordered.removeFirst();
        else if (next.kind() == MessageKind.FULL) full = null;
        else delta = null;
        queuedBytes -= next.bytes();
        inFlight = next; // 이 시점부터 buffer release 책임은 transport에 있다.
        return Optional.of(next);
    }

    public void complete(ReleasableMessage inFlight, boolean success) {
        if (this.inFlight != inFlight)
            throw new IllegalArgumentException("completion does not match in-flight message");
        this.inFlight = null;
        // transport가 completion과 함께 ownership을 끝낸다. queue는 release하지 않는다.
        if (!success || inFlight.terminal()) close();
    }

    public QueueView view() {
        long retained = queuedBytes + (inFlight == null ? 0L : inFlight.bytes());
        return new QueueView(ordered.size(), full == null ? 0 : 1,
                delta == null ? 0 : 1, inFlight == null ? 0 : 1,
                retained, terminal, closed);
    }

    @Override
    public void close() {
        if (closed) return;
        closed = true;
        while (!ordered.isEmpty()) releaseQueued(ordered.removeFirst());
        releaseQueued(full);
        releaseQueued(delta);
        full = null;
        delta = null;
        // 이미 detach된 객체는 실제 completion 전까지 transport 소유다.
    }

    private final java.util.ArrayDeque<ReleasableMessage> ordered =
            new java.util.ArrayDeque<>(ORDERED_LIMIT);
    private ReleasableMessage full;
    private ReleasableMessage delta;
    private ReleasableMessage inFlight;
    private long queuedBytes;
    private boolean terminal;
    private boolean closed;

    private boolean releaseQueued(ReleasableMessage message) {
        if (message == null) return false;
        queuedBytes -= message.bytes();
        message.release();
        return true;
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
   - 모범답변: control은 사건 순서와 개별 delivery가 중요하지만 snapshot은 더 최신 상태가 이전 queued 상태를 대체할 수 있다. 같은 FIFO면 slow client에서 매 tick snapshot이 control 앞뒤로 무한 누적된다.
2. FULL·DELTA supersession 규칙이 replication 의미와 맞는지 판단한 기준.
   - 모범답변: 새 FULL은 자체 base가 없어 queued FULL·DELTA를 모두 대체하고, 새 DELTA는 같은 acknowledged base 계약 아래 이전 queued DELTA만 대체한다. transport in-flight는 이미 ownership이 넘어가므로 건드리지 않는다.
3. queue-owned와 transport-owned buffer의 ownership transfer 시점.
   - 모범답변: `detachNext` 또는 원본의 실제 `writeAndFlush` 직전에 queued metric을 빼고 in-flight로 표시하는 순간이다. 그 뒤 release는 completion callback 계약에 속한다.
4. close와 asynchronous completion이 겹칠 때 exactly-once cleanup을 보장한 방법.
   - 모범답변: close는 멱등하게 queued 객체만 release하고 in-flight reference를 유지한다. completion은 identity가 현재 in-flight와 같은지 확인해 한 번만 retire하며, 늦은 중복 callback은 무시한다.
5. 새 snapshot 억제와 최신 snapshot coalescing 사이의 trade-off.
   - 모범답변: queued snapshot은 아직 queue 소유라 최신본으로 안전하게 교체할 수 있지만 in-flight는 수정할 수 없다. in-flight 동종 억제는 메모리와 ownership을 단순화하는 대신 중간 최신본을 놓치며 periodic/resync FULL이 이를 보완한다.

### 원본 확인 위치

- Thread: G12
- 커밋: `feat: bound outbound snapshot ownership`
- 파일·컴포넌트: `src/main/java/arena/OutboundQueue.java`, `ArenaServer.Peer.outbound`, `OutboundQueue.Metrics`
- 함수: `OutboundQueue.admit`, `flushWhenReady`, `close`, `view`
- 테스트·시나리오: `src/test/java/arena/BackpressureScenario.java`, `OutboundQueueTest.java`, `src/test/resources/G12.json`
- 관련 Thread: G01, G02, G08, G09, G10, G14
