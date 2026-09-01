# 개발자 기술면접 워크북 05 — 외부 통신 보안과 자원 수명주기

이 문서는 사용자가 지정한 URL로 서버가 HTTP 요청을 보내는 기능을 안전하게 만드는 문제를 다룬다. 문자열 검증만으로 끝나지 않고 DNS, 실제 연결 주소, redirect, TLS 호스트명, 전체 시간 예산과 socket 정리까지 하나의 경계로 본다.

---

<!-- coverage: SA-17 -->
<a id="sa-17"></a>
## [Thread 12 / `feat(e12): adopt preserved outbound destination safeguards`] URL 정규화, DNS pinning, redirect 재검증으로 SSRF 막기

### 면접 질문

사용자가 입력한 URL을 저장할 때 private IP 문자열만 거부하면 SSRF 방어가 충분하지 않은 이유는 무엇입니까? hostname을 해석한 실제 주소를 검증하고 그 주소로 직접 연결하되, HTTPS 검증에는 원래 hostname을 유지한 이유를 설명해 주세요.

꼬리 질문:

- `localhost`, `127.0.0.1`, IPv6 loopback, link-local, RFC1918, IPv4-mapped IPv6를 어떻게 다뤄야 합니까?
  - 모범답변: production 기본 정책에서는 이름과 literal 모두 연결 전에 거부합니다. 원본은 ordinary global unicast만 허용하고 mapped/compatible IPv6와 transition·documentation 범위도 fail-closed 합니다.
- DNS 검증 후 HTTP 라이브러리가 hostname을 다시 해석하면 어떤 DNS rebinding 창이 생깁니까?
  - 모범답변: 검증 때 public이던 이름이 connector의 두 번째 조회에서는 private로 바뀔 수 있습니다. 검증한 `InetAddress`로 socket을 열어 이 재해석을 없애야 합니다.
- DNS 응답에 public 주소와 private 주소가 섞여 있으면 하나를 골라 연결해도 됩니까?
  - 모범답변: 안 됩니다. 주소 선택·fallback을 완전히 통제하지 못하므로 하나라도 unsafe면 전체 answer를 거부해 모든 가능한 연결이 안전하다는 invariant를 유지합니다.
- 302가 public URL에서 private URL로 이동하면 어느 시점에 차단해야 합니까?
  - 모범답변: `Location`을 현재 URI에 resolve한 직후 canonicalization하고, 다음 socket을 열기 전에 새 host의 DNS 전체를 다시 검증해야 합니다.
- canonicalization 단계에서 DNS나 외부 I/O를 하지 않은 이유는 무엇입니까?
  - 모범답변: 저장 요청을 DNS 가용성과 시점 변화에 결합하지 않고 안정된 syntax policy만 적용하기 위해서입니다. 주소 안전성은 실제 실행 직전의 현재 DNS로 판단합니다.

### 30초 모범 답변

SSRF는 URL 문자열보다 실제 연결 목적지가 핵심입니다. hostname은 실행 시 public과 private 주소로 바뀔 수 있고 redirect도 새 목적지를 만들 수 있습니다. 그래서 먼저 scheme·userinfo·fragment·port·host를 정규화해 저장하되 DNS는 하지 않고, 실행 시 모든 DNS 응답이 허용된 public 주소인지 검사했습니다. 검증한 `InetAddress`로 직접 socket을 열어 두 번째 DNS 해석을 막고, HTTPS의 SNI와 hostname verification에는 원래 hostname을 사용했습니다. redirect마다 URL 정규화와 DNS 검사를 다시 수행하며, private 또는 혼합 응답은 연결 전에 거부했습니다.

### 답변 핵심 키워드

SSRF, canonical URL, actual destination, DNS rebinding, address pinning, mixed DNS answer, redirect revalidation, SNI, hostname verification, fail closed

### 백지 구현

#### 구현 목표

HTTP/HTTPS URL을 정규화하고, 실행 시 DNS 결과를 검증한 뒤 검증된 주소로만 연결할 수 있는 정책 계층을 작성한다. 실제 socket 코드는 인터페이스 뒤에 둔다.

#### 인터페이스 또는 함수 시그니처

```java
record ResolvedTarget(URI canonicalUrl, String originalHost, InetAddress address) {}

interface Resolver {
    InetAddress[] resolve(String host) throws IOException;
}

final class PolicyException extends IllegalArgumentException {
    final String reason;

    PolicyException(String reason) {
        super(reason);
        this.reason = reason;
    }
}

final class OutboundPolicy {
    URI canonical(String raw) {
        try {
            URI input = URI.create(raw);
            String scheme = input.getScheme() == null ? ""
                    : input.getScheme().toLowerCase(Locale.ROOT);
            String host = input.getHost() == null ? ""
                    : input.getHost().toLowerCase(Locale.ROOT);
            if (host.startsWith("[") && host.endsWith("]")) {
                host = host.substring(1, host.length() - 1);
            }
            if (host.endsWith(".")) host = host.substring(0, host.length() - 1);
            if (!(scheme.equals("http") || scheme.equals("https")) || host.isEmpty()
                    || input.getRawUserInfo() != null || input.getRawFragment() != null
                    || host.indexOf('%') >= 0 || input.getPort() == 0
                    || input.getPort() > 65_535 || input.getRawAuthority().endsWith(":")) {
                throw new PolicyException("INVALID_URL");
            }
            int port = input.getPort();
            if ((scheme.equals("http") && port == 80)
                    || (scheme.equals("https") && port == 443)) port = -1;
            String authority = host.indexOf(':') >= 0 ? "[" + host + "]" : host;
            if (port >= 0) authority += ":" + port;
            String path = input.getRawPath().isEmpty() ? "/" : input.getRawPath();
            URI result = URI.create(scheme + "://" + authority + path
                    + (input.getRawQuery() == null ? "" : "?" + input.getRawQuery())).normalize();
            InetAddress literal = literal(host);
            if (host.equals("localhost") || host.endsWith(".localhost")
                    || (literal != null && !isPublic(literal))) {
                throw new PolicyException("UNSAFE_ADDRESS");
            }
            return URI.create(result.toASCIIString());
        } catch (IllegalArgumentException | NullPointerException error) {
            if (error instanceof PolicyException blocked) throw blocked;
            throw new PolicyException("INVALID_URL");
        }
    }

    List<ResolvedTarget> resolveAllowed(URI canonical, Resolver resolver) throws IOException {
        String host = canonical.getHost();
        if (host.startsWith("[") && host.endsWith("]")) {
            host = host.substring(1, host.length() - 1);
        }
        InetAddress literal = literal(host);
        InetAddress[] addresses = literal == null
                ? resolver.resolve(host) : new InetAddress[] { literal };
        if (addresses == null || addresses.length == 0) {
            throw new PolicyException("EMPTY_DNS_ANSWER");
        }
        for (InetAddress address : addresses) {
            if (address == null || !isPublic(address)) {
                // connector를 호출하기 전에 혼합 answer 전체를 거부한다.
                throw new PolicyException("UNSAFE_DNS_ANSWER");
            }
        }
        String originalHost = host;
        return Arrays.stream(addresses)
                .map(address -> new ResolvedTarget(canonical, originalHost, address))
                .toList();
    }

    URI redirect(URI current, String location) {
        try {
            return canonical(current.resolve(location).toASCIIString());
        } catch (PolicyException blocked) {
            throw blocked;
        } catch (IllegalArgumentException | NullPointerException error) {
            throw new PolicyException("INVALID_REDIRECT");
        }
    }

    private static InetAddress literal(String host) {
        try {
            if (host.indexOf(':') >= 0) return InetAddress.getByName(host);
            if (!host.matches("[0-9]+(?:\\.[0-9]+){3}")) return null;
            String[] parts = host.split("\\.", -1);
            byte[] bytes = new byte[4];
            for (int index = 0; index < 4; index++) {
                if (!parts[index].matches("0|[1-9][0-9]{0,2}")) {
                    throw new PolicyException("AMBIGUOUS_ADDRESS");
                }
                int value = Integer.parseInt(parts[index]);
                if (value > 255) throw new PolicyException("AMBIGUOUS_ADDRESS");
                bytes[index] = (byte) value;
            }
            return InetAddress.getByAddress(bytes);
        } catch (UnknownHostException error) {
            throw new IllegalArgumentException("INVALID_URL");
        }
    }

    private static boolean isPublic(InetAddress address) {
        byte[] bytes = address.getAddress();
        int a = Byte.toUnsignedInt(bytes[0]);
        int b = Byte.toUnsignedInt(bytes[1]);
        int c = Byte.toUnsignedInt(bytes[2]);
        if (bytes.length == 4) {
            return !(a == 0 || a == 10 || a == 127 || a >= 224
                    || (a == 100 && b >= 64 && b <= 127)
                    || (a == 169 && b == 254) || (a == 172 && b >= 16 && b <= 31)
                    || (a == 192 && (b == 168 || b == 0 && (c == 0 || c == 2)
                        || b == 88 && c == 99))
                    || (a == 198 && (b == 18 || b == 19 || b == 51 && c == 100))
                    || (a == 203 && b == 0 && c == 113));
        }
        // mapped/compatible IPv4, ULA, link-local, multicast와 transition 범위도 제외한다.
        return bytes.length == 16 && a >= 0x20 && a <= 0x3f
                && !(a == 0x20 && b == 0x01 && (c & 0xfe) == 0)
                && !(a == 0x20 && b == 0x01 && c == 0x0d
                    && Byte.toUnsignedInt(bytes[3]) == 0xb8)
                && !(a == 0x20 && b == 0x02)
                && !(a == 0x3f && b == 0xff && (c & 0xf0) == 0);
    }
}
```

#### 입력과 출력

- canonical 입력: 사용자 URL 문자열
- canonical 출력: ASCII 정규형 URI
- resolve 입력: 정규형 URI와 DNS resolver
- resolve 출력: 모두 정책 검증을 통과한 실제 주소 목록
- redirect 입력: 현재 URI와 `Location` 값
- redirect 출력: 다시 정규화해야 하는 절대 URI

#### 반드시 만족해야 할 조건

- scheme은 `http` 또는 `https`만 허용한다.
- host가 반드시 존재해야 한다.
- userinfo와 fragment를 거부한다.
- port 0, 65,535 초과, 비정상 authority를 거부한다.
- 기본 port는 정규형에서 일관되게 처리한다.
- hostname 대소문자와 마지막 점을 정규화한다.
- loopback, unspecified, private, link-local, multicast와 그에 준하는 비공개 주소를 거부한다.
- literal IP도 DNS hostname과 같은 주소 정책을 적용한다.
- DNS 응답이 비어 있거나 하나라도 unsafe면 전체를 거부한다.
- 실제 connector는 검증된 주소를 인자로 받아야 하며 hostname을 다시 해석하지 않는다.
- HTTPS에서는 원래 hostname을 SNI와 endpoint identification에 사용한다.
- 모든 redirect 대상에 같은 전체 정책을 다시 적용한다.

#### 경계 조건

- `HTTP://EXAMPLE.COM:80/a/../b`
- trailing-dot hostname
- IPv4와 bracketed IPv6 literal
- `localhost`, `*.localhost`
- `127.0.0.1`, `::1`, `10.0.0.1`, `169.254.169.254`, `fe80::1`, IPv4-mapped loopback
- DNS 응답 0개, public 1개, public+private 혼합
- 상대 redirect, scheme-relative redirect, userinfo가 있는 redirect
- redirect가 다시 원래 URL로 순환
- 테스트 전용 loopback fixture 예외의 명시적 on/off

#### 실패 조건

- 문자열 prefix나 정규식만으로 private 목적지를 판정하지 않는다.
- DNS 검증 후 일반 URL connector에 hostname을 넘겨 재해석시키지 않는다.
- mixed DNS answer에서 public 주소 하나만 선택해 진행하지 않는다.
- 첫 URL만 검사하고 redirect를 자동 추적하지 않는다.
- canonicalization 중 DNS를 수행해 저장 요청을 네트워크 상태에 결합하지 않는다.
- 정책 거부를 일반 endpoint의 HTTP 실패로 꾸미지 않는다.

#### 필요한 제약

- public address 판정은 표준 `InetAddress` 속성과 명시적 보완 규칙으로 구성한다.
- 국제화 도메인은 ASCII 정규형으로 다룬다.
- 구현 시간은 25~30분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 정상 HTTP/HTTPS URL이 하나의 정규형으로 수렴한다.
- [ ] userinfo·fragment·잘못된 port가 거부된다.
- [ ] 모든 unsafe literal이 DNS와 connector 전에 거부된다.
- [ ] public DNS 응답만 connector에 전달된다.
- [ ] public+private 혼합 응답에서 connector 호출이 0회다.
- [ ] 검증된 주소와 실제 연결 주소가 동일하다.
- [ ] HTTPS SNI와 hostname verification 값은 원래 host다.
- [ ] public → private redirect가 두 번째 연결 전에 거부된다.
- [ ] redirect 예산과 순환 정책을 별도로 적용할 수 있다.
- [ ] 테스트 fixture 예외가 production 기본값에서 꺼져 있다.

### 구현 후 설명할 것

1. URL 문법 검증과 실제 목적지 검증을 나눈 이유
   - 모범답변: syntax canonicalization은 저장 시 결정적이지만 DNS 주소는 실행 시 바뀝니다. 저장 모델은 네트워크 가용성과 분리하고 실제 연결 직전에 현재 목적지를 검증합니다.
2. DNS pinning이 막는 재해석·rebinding 경로
   - 모범답변: 검증한 answer의 숫자 주소로 직접 connect해 라이브러리의 두 번째 lookup이 private address를 선택하는 창을 닫습니다.
3. 혼합 DNS 응답을 전체 거부한 trade-off
   - 모범답변: 일부 public 주소로 서비스 가능해도 하나의 unsafe 주소 때문에 가용성이 낮아집니다. 대신 connector 선택과 fallback에 무관하게 안전하다는 invariant를 얻습니다.
4. IP로 연결하면서 TLS hostname을 보존한 방법
   - 모범답변: TCP는 pinned InetAddress로 열고, SSLSocket 생성과 SNI·`HTTPS` endpoint identification에는 원래 hostname을 전달합니다.
5. redirect마다 보안 경계를 다시 통과시키는 이유
   - 모범답변: 각 Location은 공격자가 제어할 수 있는 새 목적지이므로 scheme·authority·DNS·주소 pinning을 처음 URL과 동일하게 적용해야 합니다.

### 원본 확인 위치

- Thread 12
- 커밋: `feat(e12): adopt preserved outbound destination safeguards`
- `backend/src/main/java/dev/evolution/monitor/OutboundUrl.java`
- `OutboundUrl.canonical`, `OutboundUrl.requireAddresses`, `OutboundUrl.isPublic`
- `backend/src/main/java/dev/evolution/monitor/CheckRunner.java`
- `CheckRunner.canonicalUrl`, `CheckRunner.run`
- `CheckRunner.Resolver`, `CheckRunner.Connector`
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java`
- 관련 Thread: Thread 01의 fixture-only HTTP, Thread 09의 worker I/O, Thread 11의 미확정 결과 의미론

---

<!-- coverage: SA-18 -->
<a id="sa-18"></a>
## [Thread 12 / `feat(e12): adopt preserved outbound destination safeguards`] 전체 deadline, bounded DNS 실행기, socket 정리

### 면접 질문

connect timeout과 read timeout을 각각 설정했는데도 전체 실행 시간이 무한히 늘어날 수 있는 이유는 무엇입니까? 하나의 total deadline을 모든 DNS·연결·redirect·header 읽기에 공유하고, 중단되지 않는 DNS 작업을 최대 한 개·대기열 0개로 제한한 이유를 설명해 주세요.

꼬리 질문:

- DNS 호출이 interruption을 무시하면 `Future.cancel(true)`만으로 충분합니까?
  - 모범답변: 아니요. resolver thread가 계속 남을 수 있으므로 원본은 최대 1 thread와 `SynchronousQueue` 0 backlog로 추가 작업 축적을 막고 deadline 뒤 결과의 연결 권한도 검사합니다.
- redirect마다 read timeout을 새로 500ms 주면 전체 1.5초 보장이 깨질 수 있는 이유는 무엇입니까?
  - 모범답변: 각 hop의 DNS·connect·read 시간이 redirect 수만큼 합산될 수 있습니다. 모든 단계 timeout을 단일 deadline의 남은 시간과 local 상한 중 작은 값으로 제한해야 합니다.
- 응답 body를 읽지 않고 headers까지만 관찰한 이유와 socket 종료 시점은 무엇입니까?
  - 모범답변: Monitor 결과는 최종 status만 필요하므로 header 종료 즉시 status를 publish하고 body는 0바이트 소비한 채 `finally`에서 socket을 닫습니다.
- header가 끝없이 오거나 아주 큰 경우 어떤 메모리 제한이 필요합니까?
  - 모범답변: 하나의 고정 65,536-byte buffer 안에서 status와 headers 종료를 찾아야 하며 다음 바이트가 필요하면 정책 실패로 중단합니다. body 크기와는 별도 상한입니다.
- 종료 시 active socket과 executor를 어떤 순서로 정리해야 합니까?
  - 모범답변: active attempt를 cancel해 등록 socket을 먼저 닫아 blocking I/O를 깨우고, 그다음 `shutdownNow()`로 executor intake와 thread를 종료합니다.

### 30초 모범 답변

개별 connect/read timeout은 각 단계에만 적용되므로 DNS가 멈추거나 redirect가 반복되면 전체 시간이 합산돼 상한을 넘을 수 있습니다. 실행 시작 시 단조 시계로 하나의 deadline을 만들고 각 단계의 timeout을 남은 시간과 로컬 상한 중 작은 값으로 계산했습니다. DNS는 interruption을 무시할 수 있어 스레드를 계속 늘리지 않도록 최대 1개 작업, `SynchronousQueue`로 대기열 0개인 executor를 사용했습니다. deadline이 지나면 active socket을 닫아 실제 I/O를 깨우고, 늦게 끝난 resolver는 더 이상 연결하지 못하게 검사합니다. headers는 64KiB까지만 읽고 body는 소비하지 않으며 모든 성공·실패·취소·종료 경로에서 socket을 닫습니다.

### 답변 핵심 키워드

total deadline, monotonic clock, remaining budget, uninterruptible DNS, bounded executor, zero queue, active socket cancellation, header cap, no body read, cleanup

### 백지 구현

#### 구현 목표

외부 HTTP 관측 한 건에 전체 시간·redirect·header·동시 실행 자원 예산을 적용하는 실행기를 설계한다. 실제 HTTP 파서는 status line과 headers 종료까지만 지원한다.

#### 인터페이스 또는 함수 시그니처

```java
record Limits(
    Duration connectTimeout,
    Duration readTimeout,
    Duration totalTimeout,
    int maxRedirects,
    int maxHeaderBytes
) {}

final class BoundedChecker implements AutoCloseable {
    private final OutboundPolicy policy;
    private final DestinationResolver resolver;
    private final PinnedConnector connector;
    private final AtomicReference<Attempt> active = new AtomicReference<>();
    private final ThreadPoolExecutor io = new ThreadPoolExecutor(
            0, 1, 0, TimeUnit.MILLISECONDS, new SynchronousQueue<>(), runnable -> {
                Thread thread = new Thread(runnable, "bounded-check-outbound");
                thread.setDaemon(true);
                return thread;
            });

    BoundedChecker(
            OutboundPolicy policy,
            DestinationResolver resolver,
            PinnedConnector connector) {
        this.policy = policy;
        this.resolver = resolver;
        this.connector = connector;
    }

    CheckResult execute(URI initial, Limits limits) {
        if (limits.connectTimeout().isNegative() || limits.connectTimeout().isZero()
                || limits.readTimeout().isNegative() || limits.readTimeout().isZero()
                || limits.totalTimeout().isNegative() || limits.totalTimeout().isZero()
                || limits.maxRedirects() < 0 || limits.maxHeaderBytes() < 4) {
            throw new IllegalArgumentException("Outbound limits must be positive");
        }
        long started = System.nanoTime();
        Attempt attempt = new Attempt(started + limits.totalTimeout().toNanos());
        Future<Integer> task;
        try {
            task = io.submit(() -> {
                active.set(attempt);
                try {
                    if (io.isShutdown()) {
                        throw new IllegalStateException("Outbound observation stopped");
                    }
                    return observe(initial, limits, attempt); // redirect마다 resolveAllowed를 다시 호출한다.
                } finally {
                    attempt.closeSocket();
                    active.compareAndSet(attempt, null);
                }
            });
        } catch (RejectedExecutionException busy) {
            throw new IllegalStateException("Outbound observation capacity unavailable", busy);
        }
        try {
            int status = task.get(Math.max(0, attempt.remainingNanos()), TimeUnit.NANOSECONDS);
            long latency = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started);
            return status >= 200 && status < 300
                    ? CheckResult.succeeded(status, latency)
                    : CheckResult.failed(status, latency, "HTTP_STATUS");
        } catch (TimeoutException timeout) {
            attempt.cancel(); // Future interruption과 별도로 실제 socket을 닫는다.
            task.cancel(true);
            Integer finalStatus = attempt.finalStatus();
            if (finalStatus != null) {
                long latency = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started);
                return finalStatus >= 200 && finalStatus < 300
                        ? CheckResult.succeeded(finalStatus, latency)
                        : CheckResult.failed(finalStatus, latency, "HTTP_STATUS");
            }
            return CheckResult.failed(null,
                    TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started), "TIMEOUT");
        } catch (InterruptedException interrupted) {
            attempt.cancel();
            task.cancel(true);
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Outbound observation interrupted", interrupted);
        } catch (ExecutionException failed) {
            if (failed.getCause() instanceof PolicyException) return CheckResult.aborted();
            if (failed.getCause() instanceof SocketTimeoutException) {
                return CheckResult.failed(null,
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started), "TIMEOUT");
            }
            if (failed.getCause() instanceof IOException) {
                return CheckResult.failed(null,
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started), "CONNECTION_FAILURE");
            }
            throw new IllegalStateException("Outbound observation failed", failed.getCause());
        }
    }

    private int observe(URI initial, Limits limits, Attempt attempt) throws IOException {
        URI url = policy.canonical(initial.toASCIIString());
        for (int redirects = 0; ; redirects++) {
            attempt.check();
            List<ResolvedTarget> targets = policy.resolveAllowed(url, resolver::resolve);
            attempt.check(); // interruption을 무시한 DNS가 deadline 뒤 연결 권한을 얻지 못하게 한다.
            ResolvedTarget target = targets.getFirst();
            Socket socket = connector.connect(
                    url, target.address(), attempt.timeout(limits.connectTimeout()));
            try {
                attempt.register(socket);
                String path = url.getRawPath()
                        + (url.getRawQuery() == null ? "" : "?" + url.getRawQuery());
                String request = "GET " + path + " HTTP/1.1\r\nHost: "
                        + url.getRawAuthority() + "\r\nConnection: close\r\n\r\n";
                socket.getOutputStream().write(request.getBytes(StandardCharsets.US_ASCII));
                socket.setSoTimeout(attempt.timeout(limits.readTimeout()));
                Headers headers = readHeaders(
                        socket.getInputStream(), limits.maxHeaderBytes(), attempt);
                if (isRedirect(headers.status()) && headers.location() != null) {
                    if (redirects == limits.maxRedirects()) {
                        throw new PolicyException("REDIRECT_LIMIT");
                    }
                    url = policy.redirect(url, headers.location());
                } else {
                    // 최종 headers를 publish한 뒤에는 body를 한 바이트도 읽지 않는다.
                    attempt.publishFinal(headers.status());
                    return headers.status();
                }
            } finally {
                attempt.closeSocket();
                close(socket);
            }
        }
    }

    private static Headers readHeaders(
            InputStream input,
            int maximumBytes,
            Attempt attempt) throws IOException {
        byte[] bytes = new byte[maximumBytes];
        int total = 0;
        while (true) {
            int start = total;
            while (true) {
                attempt.check();
                if (total == bytes.length) throw new PolicyException("HEADER_LIMIT");
                int next = input.read();
                if (next < 0) throw new EOFException("Incomplete HTTP headers");
                bytes[total++] = (byte) next;
                if (total - start >= 4
                        && bytes[total - 4] == '\r' && bytes[total - 3] == '\n'
                        && bytes[total - 2] == '\r' && bytes[total - 1] == '\n') break;
            }

            String[] lines = new String(
                    bytes, start, total - start - 4, StandardCharsets.ISO_8859_1)
                    .split("\r\n");
            if (!lines[0].matches("HTTP/1\\.[01] [1-5][0-9]{2}( .*)?")) {
                throw new IOException("Invalid HTTP status line");
            }
            int status = Integer.parseInt(lines[0].substring(9, 12));
            String location = null;
            for (int index = 1; index < lines.length; index++) {
                int colon = lines[index].indexOf(':');
                if (colon < 1
                        || !lines[index].substring(0, colon)
                        .matches("[!#$%&'*+.^_`|~0-9A-Za-z-]+")) {
                    throw new IOException("Invalid HTTP header");
                }
                String value = lines[index].substring(colon + 1).strip();
                for (int offset = 0; offset < value.length(); offset++) {
                    char character = value.charAt(offset);
                    if ((character < 32 && character != '\t') || character == 127) {
                        throw new IOException("Invalid HTTP header value");
                    }
                }
                if (lines[index].substring(0, colon)
                        .equalsIgnoreCase("location")) {
                    if (location != null) throw new PolicyException("INVALID_REDIRECT");
                    location = value;
                }
            }
            if (status == 101) throw new IOException("HTTP upgrade is not supported");
            if (status >= 200) return new Headers(status, location);
            // 1xx headers도 같은 hop의 byte/time 예산을 계속 소비한다.
        }
    }

    private static boolean isRedirect(int status) {
        return status == 301 || status == 302 || status == 303
                || status == 307 || status == 308;
    }

    private static void close(Socket socket) {
        try {
            socket.close();
        } catch (IOException ignored) {
            // headers가 확정된 뒤 close 실패가 endpoint 결과를 바꾸지 않는다.
        }
    }

    @Override
    public void close() {
        Attempt attempt = active.get();
        if (attempt != null) attempt.cancel();
        io.shutdownNow();
    }

    private record Headers(int status, String location) {}

    private static final class Attempt {
        private final long deadline;
        private final AtomicReference<Socket> socket = new AtomicReference<>();
        private boolean cancelled;
        private Integer finalStatus;

        Attempt(long deadline) {
            this.deadline = deadline;
        }

        long remainingNanos() {
            return deadline - System.nanoTime();
        }

        synchronized void check() throws SocketTimeoutException {
            if (cancelled || remainingNanos() <= 0) {
                throw new SocketTimeoutException("Outbound deadline exceeded");
            }
        }

        int timeout(Duration maximum) throws SocketTimeoutException {
            check();
            long remainingMs = TimeUnit.NANOSECONDS.toMillis(remainingNanos()) + 1;
            long bounded = Math.min(maximum.toMillis(), remainingMs);
            return (int) Math.max(1L, Math.min(Integer.MAX_VALUE, bounded));
        }

        void register(Socket value) throws IOException {
            socket.set(value);
            try {
                check();
            } catch (IOException expired) {
                closeSocket();
                throw expired;
            }
        }

        synchronized void publishFinal(int status) throws SocketTimeoutException {
            check();
            finalStatus = status;
        }

        synchronized Integer finalStatus() {
            return finalStatus;
        }

        void cancel() {
            synchronized (this) {
                cancelled = true;
            }
            closeSocket();
        }

        void closeSocket() {
            Socket value = socket.getAndSet(null);
            if (value != null) close(value);
        }
    }
}
```

필요한 협력 인터페이스는 직접 정의한다.

```java
interface DestinationResolver { InetAddress[] resolve(String host) throws IOException; }
interface PinnedConnector { Socket connect(URI url, InetAddress address, int timeoutMs) throws IOException; }
```

#### 입력과 출력

- 입력: 이미 문법 정규화된 URL과 제한값
- 출력: 최종 HTTP 상태 기반 결과, timeout/connection failure, 정책 거부 또는 서비스 불확실성
- 부수 효과: 네트워크 socket 한정. 응답 body는 저장하지 않는다.

#### 반드시 만족해야 할 조건

- 전체 deadline은 `System.nanoTime()`과 같은 단조 시계 기준이다.
- DNS, connect, write, header read, redirect 모두 동일한 남은 예산을 소비한다.
- 각 socket timeout은 로컬 제한과 남은 total budget 중 작은 값이다.
- redirect 횟수는 최대 3이다.
- headers는 최대 65,536바이트까지만 읽는다.
- 최종 headers 이후 body를 읽지 않는다.
- 1xx 응답은 최종 endpoint 결과가 아니며 같은 hop 예산 안에서 다음 status를 읽는다.
- HTTP 101 upgrade는 지원하지 않는다.
- DNS 작업은 최대 1개가 실행되고 대기열은 0개다.
- executor가 포화되면 새 endpoint 결과를 꾸미지 않고 즉시 실패한다.
- timeout·취소·예외·정상 종료에서 등록된 socket이 모두 닫힌다.
- `close()`는 active socket을 취소하고 executor를 종료한다.

#### 경계 조건

- DNS가 deadline 이후까지 interruption을 무시
- 첫 DNS가 멈춘 동안 두 번째 실행 요청
- connect 직전 deadline 만료
- header 한 바이트씩 늦게 전송
- 정확히 65,536바이트와 65,537바이트 header
- body 65KiB 이상을 제시하지만 headers는 정상
- 100 Continue 뒤 200
- 4번째 redirect
- close와 execute의 동시 호출

#### 실패 조건

- 매 redirect마다 total timeout을 새로 시작하지 않는다.
- `currentTimeMillis()`로 경과 시간을 재지 않는다.
- timeout 시 Future만 취소하고 socket을 열어 두지 않는다.
- DNS 지연마다 새 스레드를 계속 생성하지 않는다.
- 응답 body를 메모리에 누적하지 않는다.
- close 예외가 이미 관측한 최종 HTTP 상태를 뒤집지 않는다.

#### 필요한 제약

- HTTP/1.0과 HTTP/1.1 status line 및 headers까지만 구현한다.
- 자동 재시도는 넣지 않는다.
- 구현 시간은 30분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 정상 200 headers가 total budget 안에 반환된다.
- [ ] connect·read·total timeout이 각각 의도한 실패로 분류된다.
- [ ] 느린 redirect 체인의 전체 시간이 total timeout을 넘지 않는다.
- [ ] 중단되지 않는 DNS 하나가 있어도 스레드와 대기 작업 수가 증가하지 않는다.
- [ ] 늦게 끝난 DNS가 deadline 이후 connector를 호출하지 않는다.
- [ ] header 상한 초과가 bounded memory로 실패한다.
- [ ] 큰 body에서 body bytes를 읽지 않는다.
- [ ] 모든 생성 socket이 정상·예외·timeout·close 경로에서 닫힌다.
- [ ] 1xx 뒤 최종 status를 읽되 101은 거부한다.
- [ ] 시간·공간 사용량의 상한을 숫자로 설명할 수 있다.

### 구현 후 설명할 것

1. 단계별 timeout과 total deadline의 차이
   - 모범답변: connect/read timeout은 한 단계의 정체를 제한하고 total deadline은 DNS·모든 redirect·write·header read를 합친 실행 수명을 제한합니다.
2. 단조 시계를 사용한 이유
   - 모범답변: `System.nanoTime()` 경과 시간은 NTP·관리자 wall clock 조정에 영향받지 않아 deadline의 남은 예산이 뒤로 늘어나지 않습니다.
3. interruption을 신뢰할 수 없는 DNS에 bounded executor를 둔 이유
   - 모범답변: 취소를 무시한 resolver마다 새 thread나 대기 작업을 만들면 자원 고갈이 됩니다. 최대 실행 1, queue 0으로 손상 범위를 고정합니다.
4. active socket 종료가 cooperative cancellation보다 강한 지점
   - 모범답변: interrupt flag를 확인하지 않는 blocking connect/read도 실제 socket close로 깨울 수 있습니다. Attempt가 connect 전에 socket을 등록하는 이유입니다.
5. headers-only 관측과 header cap의 자원 절약 trade-off
   - 모범답변: status monitoring에 불필요한 body download·저장을 없애고 header 메모리를 64KiB로 제한합니다. 대신 body 기반 health 의미나 oversized header endpoint는 지원하지 않습니다.

### 원본 확인 위치

- Thread 12
- 커밋: `feat(e12): adopt preserved outbound destination safeguards`
- `backend/src/main/java/dev/evolution/monitor/CheckRunner.java`
- `CheckRunner.CONNECT_MS`, `READ_MS`, `TOTAL_MS`, `MAX_REDIRECTS`, `MAX_HEADER_BYTES`
- `CheckRunner.run`, `CheckRunner.close`
- `CheckRunner.Attempt`, `CheckRunner.Resolver`, `CheckRunner.Connector`
- `backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java`
- 관련 Thread: Thread 01의 connect/read timeout과 disconnect, Thread 11의 graceful shutdown, Thread 14의 worker readiness·active metric
