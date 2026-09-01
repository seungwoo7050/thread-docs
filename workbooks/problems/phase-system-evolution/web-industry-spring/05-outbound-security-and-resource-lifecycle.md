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
- DNS 검증 후 HTTP 라이브러리가 hostname을 다시 해석하면 어떤 DNS rebinding 창이 생깁니까?
- DNS 응답에 public 주소와 private 주소가 섞여 있으면 하나를 골라 연결해도 됩니까?
- 302가 public URL에서 private URL로 이동하면 어느 시점에 차단해야 합니까?
- canonicalization 단계에서 DNS나 외부 I/O를 하지 않은 이유는 무엇입니까?

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

final class OutboundPolicy {
    URI canonical(String raw) {
        // 직접 구현
    }

    List<ResolvedTarget> resolveAllowed(URI canonical, Resolver resolver) throws IOException {
        // 직접 구현
    }

    URI redirect(URI current, String location) {
        // 직접 구현
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
2. DNS pinning이 막는 재해석·rebinding 경로
3. 혼합 DNS 응답을 전체 거부한 trade-off
4. IP로 연결하면서 TLS hostname을 보존한 방법
5. redirect마다 보안 경계를 다시 통과시키는 이유

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
- redirect마다 read timeout을 새로 500ms 주면 전체 1.5초 보장이 깨질 수 있는 이유는 무엇입니까?
- 응답 body를 읽지 않고 headers까지만 관찰한 이유와 socket 종료 시점은 무엇입니까?
- header가 끝없이 오거나 아주 큰 경우 어떤 메모리 제한이 필요합니까?
- 종료 시 active socket과 executor를 어떤 순서로 정리해야 합니까?

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
    CheckResult execute(URI initial, Limits limits) {
        // 직접 구현
    }

    @Override
    public void close() {
        // 직접 구현
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
2. 단조 시계를 사용한 이유
3. interruption을 신뢰할 수 없는 DNS에 bounded executor를 둔 이유
4. active socket 종료가 cooperative cancellation보다 강한 지점
5. headers-only 관측과 header cap의 자원 절약 trade-off

### 원본 확인 위치

- Thread 12
- 커밋: `feat(e12): adopt preserved outbound destination safeguards`
- `backend/src/main/java/dev/evolution/monitor/CheckRunner.java`
- `CheckRunner.CONNECT_MS`, `READ_MS`, `TOTAL_MS`, `MAX_REDIRECTS`, `MAX_HEADER_BYTES`
- `CheckRunner.run`, `CheckRunner.close`
- `CheckRunner.Attempt`, `CheckRunner.Resolver`, `CheckRunner.Connector`
- `backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java`
- 관련 Thread: Thread 01의 connect/read timeout과 disconnect, Thread 11의 graceful shutdown, Thread 14의 worker readiness·active metric
