# 요청 제한과 HTTP 실패 복구 면접 워크북

네트워크 배치, 외부 저장소 장애, 프록시 실패가 요청 경계에 미치는 영향을 다룬다. 핵심은 주소 신뢰 규칙, 분산 상태, bounded I/O, 실패 소유권, 관측 가능한 fail-open이다.

## RES-01 · [Thread 6 / `feat(ratelimit): resolve user and trusted peer buckets`] 신뢰 프록시 체인의 실제 클라이언트 주소 판별

**우선순위:** S

### 면접 질문

- 왜 direct socket peer가 신뢰 프록시일 때만 `X-Forwarded-For`를 해석해야 합니까?
- 여러 hop이 있는 체인을 오른쪽에서 왼쪽으로 걸으며 첫 번째 untrusted 주소를 고르는 이유를 설명해 보세요.
- 잘못된 hop 하나 또는 trusted hop만 있는 체인에서 socket peer로 fallback한 이유는 무엇입니까?
- 꼬리 질문: IPv4·IPv6 표현을 문자열 그대로 비교하지 않고 정규화해야 하는 이유는 무엇입니까?

### 30초 모범 답변

`X-Forwarded-For`는 caller가 직접 보낼 수 있으므로 socket peer가 운영자가 통제하는 trusted proxy일 때만 신뢰합니다. proxy는 오른쪽에 가까울수록 gateway와 가까운 hop이므로 오른쪽부터 trusted 구간을 벗어나는 첫 주소가 실제 클라이언트 후보입니다. hop 하나라도 파싱할 수 없거나 체인 전체가 trusted라서 클라이언트를 특정할 수 없으면 공격자가 선택한 값보다 socket peer를 사용하는 보수적 fallback이 안전합니다. 인증 사용자는 주소 대신 canonical subject bucket을 사용해 NAT와 proxy 변화에 영향을 받지 않게 합니다.

### 답변 핵심 키워드

`direct peer trust`, `X-Forwarded-For`, `right-to-left`, `first untrusted hop`, `IP normalization`, `conservative fallback`, `user/IP namespaces`

### 백지 구현

**구현 목표**

remote address, X-Forwarded-For 값, trusted CIDR 목록을 바탕으로 anonymous rate-limit key에 사용할 주소를 결정한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
interface CidrMatcher {
  boolean matches(String normalizedAddress);
}

record ClientAddressInput(
    String remoteAddress,
    String forwardedFor,
    List<CidrMatcher> trustedProxies) {}

final class TrustedProxyAddressResolver {
  String resolve(ClientAddressInput input) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: socket `remoteAddress`, 단일 결합 `X-Forwarded-For` 문자열, trusted CIDR matcher 목록
- 출력: 정규화된 클라이언트 IP 문자열
- 파싱 불가 또는 불명확한 경우: 정규화된 socket peer

**반드시 만족해야 할 조건**

- socket peer가 trusted가 아니면 forwarded header를 전혀 사용하지 않는다.
- trusted peer일 때 comma-separated 모든 hop을 파싱하고 정규화한다.
- 오른쪽에서 왼쪽으로 진행해 첫 untrusted hop을 반환한다.
- 빈 hop·잘못된 IP 하나라도 있으면 socket peer로 fallback한다.
- 체인에 untrusted hop이 없으면 socket peer로 fallback한다.

**경계 조건**

- trusted CIDR 목록이 비어 있는 기본 설정
- 앞뒤 공백, 빈 중간 hop, 마지막 comma
- IPv4, IPv6, IPv4-mapped IPv6 표현
- remote peer 자체가 파싱 불가한 비정상 테스트 입력
- forwarded header가 없거나 blank인 경우

**실패 조건**

- 공격자 제공 문자열을 예외 메시지나 bucket key로 그대로 사용하지 않는다.
- invalid chain을 부분적으로 받아들이지 않는다.
- CIDR 설정이 잘못되면 요청 시 fallback하지 말고 시작 시 설정 오류로 거부한다.

**필요한 제약**

- DNS 조회를 하지 않는다.
- host name은 IP 주소로 간주하지 않는다.
- 문자열 prefix로 CIDR 포함 여부를 판정하지 않는다.

### 구현 후 자가 검증

- [ ] untrusted socket peer에서는 spoofed XFF가 무시된다.
- [ ] trusted proxy 한 개 뒤의 client 주소를 올바르게 고른다.
- [ ] 여러 trusted proxy 뒤에서 오른쪽부터 첫 untrusted hop을 고른다.
- [ ] 잘못된 hop, 빈 hop, trusted-only chain이 socket peer로 fallback한다.
- [ ] 동일 IPv6 주소의 다른 텍스트 표현이 같은 정규 주소로 처리된다.
- [ ] trusted CIDR가 비어 있으면 항상 socket peer를 쓴다.
- [ ] 시간 복잡도가 hop 수 × trusted CIDR 수의 명시적 상한 안에 있다.

### 구현 후 설명할 것

- 헤더 신뢰가 네트워크 배치와 직접 peer 통제에 의존하는 이유
- right-to-left 알고리즘이 proxy append/overwrite 규칙과 맞는 방식
- invalid chain을 전부 버리는 보수적 정책의 장단점
- 인증 사용자 key와 anonymous IP key를 namespace로 분리한 이유

### 원본 확인 위치

- Thread 6
- 커밋: `feat(ratelimit): resolve user and trusted peer buckets`
- 커밋: `test(ratelimit): verify rate limit key isolation`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolver.java`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimitProperties.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolverTest.java`
- 관련 Thread: 3, 5, 16

## RES-02 · [Thread 6 / `feat(ratelimit): consume distributed token buckets`; `feat(ratelimit): enforce request rate limits`] 분산 토큰 버킷 결과와 HTTP 제한 계약

**우선순위:** S

### 면접 질문

- 여러 gateway 인스턴스가 하나의 Redis-backed bucket 용량을 공유해야 하는 이유는 무엇입니까?
- 토큰 소비 결과에서 allowed, remaining, retry-after, fail-open을 서로 독립된 상태로 표현한 이유를 설명해 보세요.
- `Retry-After`를 최소 1초의 정수 초로 올림해야 하는 이유는 무엇입니까?
- Actuator와 error dispatch가 토큰을 소비하지 않게 한 이유는 무엇입니까?

### 30초 모범 답변

프로세스 로컬 bucket은 인스턴스 수만큼 허용량이 늘어나므로 Redis에 key별 상태를 두어 전체 gateway가 같은 용량을 공유해야 합니다. 소비 결과는 정상 허용, 정상 거부, 저장소 실패로 인한 fail-open을 구분해야 HTTP 헤더와 운영 지표를 정확히 만들 수 있습니다. 거부 시 남은 토큰은 0, `Retry-After`는 클라이언트가 즉시 재시도 루프에 빠지지 않도록 나노초 대기를 최소 1초 정수로 올림합니다. 상태 확인과 오류 재처리는 자기 자신을 제한해 장애를 증폭하지 않도록 제외합니다.

### 답변 핵심 키워드

`distributed token bucket`, `shared capacity`, `result algebra`, `Retry-After ceil`, `429 problem`, `filter ordering`, `operational endpoints exclusion`

### 백지 구현

**구현 목표**

저장소가 반환한 token consumption probe를 gateway의 명시적 결과와 HTTP 응답으로 변환한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record ConsumptionProbe(
    boolean consumed,
    long remainingTokens,
    long nanosToWaitForRefill) {}

record LimitResult(
    boolean allowed,
    long remainingTokens,
    long retryAfterSeconds,
    boolean failOpen) {}

final class RateLimitDecisionMapper {
  LimitResult map(ConsumptionProbe probe) {
    throw new UnsupportedOperationException("직접 구현");
  }

  HttpOutcome toHttp(LimitResult result) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 토큰 소비 성공 여부, 남은 토큰, refill까지 나노초
- 출력: 허용/거부 상태와 remaining·retry-after를 가진 `LimitResult`
- 출력: 통과 또는 429 Problem 응답에 필요한 `HttpOutcome`

**반드시 만족해야 할 조건**

- 소비 성공은 allowed=true, 실제 remaining, retry-after=0이다.
- 소비 실패는 allowed=false, remaining=0, retry-after는 최소 1초의 ceiling이다.
- 정상 결정에만 `X-RateLimit-Remaining` 의미가 있다.
- 거부 응답은 429, `Retry-After`, remaining 0, 공통 Problem 계약을 사용한다.
- disabled·Actuator·ERROR dispatch는 limiter 호출 전에 제외할 수 있어야 한다.

**경계 조건**

- 대기 나노초 0, 1, 정확히 1초, 1초+1나노초
- 매우 큰 나노초 값의 overflow
- capacity가 1인 bucket
- 인증 사용자와 anonymous 주소가 동일 문자열을 가져도 namespace가 다른 경우

**실패 조건**

- 음수 remaining이나 retry-after 같은 불가능 상태를 조용히 내보내지 않는다.
- 저장소 예외는 이 mapper에서 429로 위장하지 않고 별도 fail-open 경로로 전달한다.
- 거부 후 downstream filter chain을 실행하지 않는다.

**필요한 제약**

- 실제 Redis나 Bucket4j 연결은 구현하지 않는다.
- 부동소수점 연산 없이 정수 나노초를 정수 초로 안전하게 변환한다.
- 정책 capacity와 refill period는 미리 양수로 검증되었다고 가정한다.

### 구현 후 자가 검증

- [ ] 성공 probe가 정확한 remaining을 보존한다.
- [ ] 실패 probe의 1나노초와 1초가 모두 1초로, 1초+1나노초가 2초로 변환된다.
- [ ] 거부 결과가 429와 필수 헤더·Problem code를 만든다.
- [ ] 허용 결과에서 fail-open과 정상 허용을 구분할 수 있다.
- [ ] disabled·Actuator·ERROR 경로에서는 store 호출이 없다.
- [ ] user와 IP key prefix가 충돌하지 않는다.
- [ ] overflow 입력에 대해 음수 retry-after가 나오지 않는다.

### 구현 후 설명할 것

- 분산 bucket과 프로세스 로컬 bucket의 수평 확장 차이
- 도메인 결과를 HTTP 표현과 분리한 이유
- ceiling 계산과 최소 1초 보장의 의미
- rate-limit filter를 인증 뒤에 둬 canonical subject를 쓸 수 있게 한 trade-off

### 원본 확인 위치

- Thread 6
- 커밋: `feat(ratelimit): consume distributed token buckets`
- 커밋: `test(ratelimit): verify shared token consumption`
- 커밋: `feat(ratelimit): enforce request rate limits`
- 커밋: `test(ratelimit): verify HTTP limit responses`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimitFilter.java`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimitHttpConfiguration.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java`
- 관련 Thread: 2, 3, 17

## RES-03 · [Thread 6 / `feat(ratelimit): configure bounded Redis access`] Redis 연결 single-flight, bounded I/O, fail-open 복구

**우선순위:** S

### 면접 질문

- Redis 연결 300ms, command 500ms 같은 상한을 두지 않으면 gateway의 요청 latency에 어떤 일이 생깁니까?
- 여러 요청이 첫 연결을 동시에 만들 때 하나만 초기화하고 나머지는 오래 기다리지 않게 한 이유를 설명해 보세요.
- Redis 예외에서 현재 요청을 허용하는 fail-open을 선택한 이유와 반드시 필요한 운영 보완책은 무엇입니까?
- 연결 실패 후 부분 생성된 connection·manager를 어떻게 정리하고 다음 요청이 재시도할 수 있게 해야 합니까?

### 30초 모범 답변

rate limiter가 보호 기능이라도 Redis I/O가 무한정 막히면 gateway 자체가 장애가 되므로 연결과 command 시간을 짧게 제한합니다. 초기화는 한 요청만 수행하고 경쟁 요청은 기다리며 스레드를 묶기보다 빠르게 fail-open하도록 해 tail latency를 제한합니다. 실패한 현재 요청은 허용하지만 remaining 헤더를 내지 않고 fail-open counter를 올려 보호가 사라진 구간을 관측합니다. 초기화가 실패하면 부분 connection과 manager 참조를 모두 원상복구하고, 이후 요청이 다시 연결을 시도할 수 있어야 합니다.

### 답변 핵심 키워드

`bounded latency`, `lazy connection`, `single-flight`, `tryLock`, `volatile publication`, `fail-open`, `cleanup and retry`, `operational counter`

### 백지 구현

**구현 목표**

지연 생성되는 외부 저장소 핸들을 동시 요청 사이에서 안전하게 공유하고, 초기화·명령 실패 시 빠른 fail-open 결과와 재시도 가능한 상태를 만든다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
interface StoreConnection extends AutoCloseable {}
interface StoreManager {}

interface StoreFactory {
  StoreConnection connect();
  StoreManager createManager(StoreConnection connection);
}

final class LazyRateLimitStore implements AutoCloseable {
  LimitResult tryConsume(String key, Limit policy) {
    throw new UnsupportedOperationException("직접 구현");
  }

  @Override
  public void close() {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: bucket key와 양수 capacity/refill 정책
- 의존성: bounded timeout을 가진 `StoreFactory`
- 출력: 정상 허용/거부 또는 failOpen=true인 허용 결과

**반드시 만족해야 할 조건**

- 완성된 manager가 있으면 lock 없이 재사용할 수 있어야 한다.
- 동시에 초기화할 때 하나의 호출만 connection과 manager를 게시한다.
- 다른 초기화 경쟁자는 무기한 기다리지 않고 실패 경계로 빠질 수 있어야 한다.
- connect/create/consume 중 저장소 예외는 fail-open 결과로 변환한다.
- 초기화 실패 시 부분 connection을 닫고 공유 참조를 비워 다음 요청이 재시도할 수 있게 한다.

**경계 조건**

- connect 성공 후 manager 생성 실패
- manager 게시 직전에 close가 호출되는 경쟁
- 두 스레드의 첫 요청, 다수 스레드 burst
- 이미 끊어진 공유 connection에서 명령 실패
- close의 중복 호출

**실패 조건**

- lock을 획득한 경로는 모든 종료에서 해제한다.
- 실패한 local resource와 공유 resource를 이중 close하지 않는다.
- fail-open 결과를 정상 store 허용과 구분하고 metric을 한 번만 증가시킨다.

**필요한 제약**

- 스레드를 sleep/poll하며 초기화를 기다리지 않는다.
- 모든 요청마다 새 connection을 만들지 않는다.
- 외부 저장소 예외 외의 프로그래밍 오류를 무조건 fail-open으로 삼지 않는다.

### 구현 후 자가 검증

- [ ] 첫 정상 호출이 connection·manager를 한 번만 만들고 이후 재사용한다.
- [ ] 동시 첫 호출에서 factory 호출 횟수가 상한을 넘지 않는다.
- [ ] 경쟁 호출이 bounded 시간 안에 fail-open 또는 정상 결과로 끝난다.
- [ ] connect 성공 후 manager 생성 실패 시 connection이 닫히고 다음 호출이 재시도한다.
- [ ] consume 실패는 failOpen=true이며 remaining 헤더용 값을 제공하지 않는다.
- [ ] 복구 후 후속 요청이 다시 정상 분산 제한을 사용한다.
- [ ] close가 공유 참조를 비우고 resource를 한 번만 닫는다.

### 구현 후 설명할 것

- lazy 초기화와 startup 연결 검사의 가용성 trade-off
- `volatile`/lock 또는 다른 동시성 도구로 안전한 publication을 만드는 방법
- 경쟁 요청을 기다리지 않고 fail-open한 tail latency 선택
- fail-open counter와 경보가 보안·운영 위험을 보완하는 방식
- fail-closed가 더 적절한 시스템에서는 무엇이 달라지는지

### 원본 확인 위치

- Thread 6
- 커밋: `feat(ratelimit): configure bounded Redis access`
- 커밋: `test(ratelimit): verify Redis connection bounds`
- 커밋: `feat(ratelimit): consume distributed token buckets`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfiguration.java`
- 파일: `src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfigurationTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java`
- 관련 Thread: 17

## RES-04 · [Thread 2, 7 / `feat(errors): define gateway problem responses`; `feat(routing): bound downstream failures`] 다운스트림 실패 정규화와 응답 계약 보존

**우선순위:** A

### 면접 질문

- gateway가 만든 오류와 downstream이 실제로 반환한 HTTP 응답을 왜 구분해야 합니까?
- `ResourceAccessException` cause chain에서 timeout 계열은 504, 그 밖의 연결·I/O 실패는 502로 분류한 근거는 무엇입니까?
- 공개 경로의 downstream 실패가 error dispatch에서 401로 바뀌지 않게 한 흐름을 설명해 보세요.
- correlation ID가 active trace ID가 없을 때 UUID로 대체되는 이유는 무엇입니까?

### 30초 모범 답변

downstream이 응답을 보냈다면 그 상태·본문·헤더는 서비스 계약이므로 gateway가 임의 Problem으로 덮지 않습니다. 반대로 연결·읽기 단계에서 응답을 받지 못한 gateway-owned 실패만 502 또는 timeout 원인이 있는 504로 정규화합니다. 모든 gateway-owned 오류는 공통 `application/problem+json` 모양과 correlation ID를 사용하며, 공개 요청의 오류 dispatch가 보안 필터에서 재인증되지 않게 해 원래 실패 의미를 보존합니다. cause chain 전체를 보되 너무 넓은 예외를 삼키지 않는 것이 핵심입니다.

### 답변 핵심 키워드

`failure ownership`, `response preservation`, `cause-chain classification`, `502 vs 504`, `problem+json`, `correlation ID`, `error dispatch`

### 백지 구현

**구현 목표**

downstream 호출 결과 또는 예외를 받아, 실제 downstream 응답은 그대로 보존하고 gateway-owned transport 실패만 공통 Problem 응답으로 바꾼다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
sealed interface DownstreamOutcome permits ReceivedResponse, TransportFailure {}
record ReceivedResponse(int status, Map<String, List<String>> headers, byte[] body)
    implements DownstreamOutcome {}
record TransportFailure(Throwable cause) implements DownstreamOutcome {}

final class DownstreamFailureNormalizer {
  GatewayResponse normalize(
      DownstreamOutcome outcome,
      String requestPath,
      String activeTraceId) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 실제 응답 또는 transport failure, 외부 request path, 선택적 active trace ID
- 출력: 보존된 downstream 응답 또는 gateway Problem 응답
- Problem에는 type, title, status, errorCode, detail, instance, correlationId가 필요

**반드시 만족해야 할 조건**

- 실제 downstream 응답은 상태·본문을 gateway 오류로 교체하지 않는다.
- transport failure cause chain에 timeout 계열이 있으면 504 `GATEWAY_TIMEOUT`이다.
- 그 밖의 확인된 연결/I/O 접근 실패는 502 `GATEWAY_BAD_GATEWAY`다.
- gateway Problem의 content type은 `application/problem+json`이다.
- active trace ID가 있으면 correlationId로 쓰고 없으면 새 UUID를 사용한다.

**경계 조건**

- timeout이 여러 겹의 cause 아래 있는 경우
- cause chain에 cycle이 있는 비정상 Throwable
- 본문이 빈 downstream 4xx/5xx 응답
- trace provider 또는 current span이 없는 경우
- request path에 query가 있어도 instance에는 확인된 URI path만 쓰는 계약

**실패 조건**

- 분류 대상이 아닌 프로그래밍 예외를 502로 숨기지 않는다.
- Problem serialization 실패를 원래 transport 오류와 혼동하지 않는다.
- 공개 경로 실패가 401/403으로 재분류되지 않아야 한다.

**필요한 제약**

- 원본 구현처럼 gateway-owned transport 경계만 다룬다.
- retry는 이 문제 범위가 아니다.
- 다운스트림 응답 헤더 중 hop-by-hop 처리 정책은 별도 계층으로 둔다.

### 구현 후 자가 검증

- [ ] downstream 400, 404, 500 응답이 상태와 body를 유지한다.
- [ ] 직접 timeout과 중첩 timeout cause가 504로 분류된다.
- [ ] connection refused와 일반 transport I/O 실패가 502로 분류된다.
- [ ] 무관한 런타임 예외는 그대로 전파되거나 별도 오류로 처리된다.
- [ ] trace ID가 있을 때 correlationId가 같고 없을 때 유효한 UUID다.
- [ ] Problem의 모든 필수 필드와 content type이 일관된다.
- [ ] 공개 경로의 실패 테스트가 인증 오류로 바뀌지 않는다.

### 구현 후 설명할 것

- 오류의 소유권을 gateway와 downstream 사이에서 나눈 기준
- cause chain 탐색의 장점과 과도한 분류 위험
- 안정된 오류 schema가 클라이언트와 관측성에 주는 이점
- 502와 504를 운영자가 다르게 대응할 수 있는 이유
- Thread 2의 공통 계약과 Thread 7의 transport 경계를 하나로 통합한 이유

### 원본 확인 위치

- Thread 2
- 커밋: `feat(errors): define gateway problem responses`
- 파일: `src/main/java/com/sportsbook/gateway/error/GatewayErrorCode.java`
- 파일: `src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java`
- 테스트: `src/test/java/com/sportsbook/gateway/error/GatewayProblemWriterTest.java`
- Thread 7
- 커밋: `feat(routing): bound downstream failures`
- 커밋: `test(routing): verify upstream failure responses`
- 커밋: `test(routing): preserve downstream response contracts`
- 파일: `src/main/java/com/sportsbook/gateway/routing/DownstreamFailureBoundary.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/DownstreamFailureBoundaryTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/ClosedPortProxyTest.java`
- 관련 Thread: 4, 6, 15, 17

## RES-05 · [Thread 7 / `feat(routing): propagate trace context`] traceparent 엄격 검증·보존·재구성

**우선순위:** A

### 면접 질문

- 인바운드 `traceparent`가 정확히 하나이고 유효할 때 그대로 보존한 이유는 무엇입니까?
- 대문자 hex, all-zero trace ID/span ID, 중복 헤더를 거부한 이유를 설명해 보세요.
- 잘못된 인바운드 값이 있을 때 active span으로 재구성하고, active span도 없으면 헤더를 제거하는 정책의 장점은 무엇입니까?
- 꼬리 질문: `sampled == null`을 어떻게 다루며 왜 임의로 sampled=true로 만들면 안 됩니까?

### 30초 모범 답변

유효한 단일 `traceparent`는 upstream이 만든 trace 연속성을 보존하므로 바꾸지 않습니다. 하지만 중복이나 규격 밖 값, all-zero ID는 모호하거나 무효이므로 먼저 제거합니다. 그 뒤 현재 서버 span이 있으면 version 00 형식으로 새 값을 만들고, 없으면 공격자 입력을 전달하지 않은 채 헤더 없이 보냅니다. sampling flag는 실제 context가 명시적으로 sampled일 때만 `01`, 그 밖에는 `00`으로 표현해 추적 결정을 부풀리지 않습니다.

### 답변 핵심 키워드

`W3C traceparent`, `single header`, `strict lowercase hex`, `nonzero IDs`, `preserve valid`, `rebuild from active span`, `sampling flag`

### 백지 구현

**구현 목표**

인바운드 traceparent 목록과 선택적 active trace context를 받아 최종 downstream traceparent를 결정한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record ActiveTrace(String traceId, String spanId, Boolean sampled) {}

final class TraceparentPolicy {
  Optional<String> resolve(List<String> inboundValues, ActiveTrace active) {
    throw new UnsupportedOperationException("직접 구현");
  }

  boolean isValid(String value) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 동일 헤더의 모든 값, 선택적 active trace context
- 출력: 보존·재구성할 단일 값 또는 헤더 제거를 뜻하는 empty
- 버전 범위: 프로젝트에서 확인된 version `00` 형식

**반드시 만족해야 할 조건**

- 인바운드 값이 정확히 하나이고 유효하면 그대로 반환한다.
- 형식은 `00-32hex-16hex-(00|01)`이며 소문자 hex만 허용한다.
- trace ID와 span ID는 all-zero가 아니어야 한다.
- 0개·2개 이상·invalid 인바운드는 active context가 유효하면 재구성한다.
- active context가 없거나 invalid이면 empty를 반환한다.

**경계 조건**

- 대문자 hex, 길이 하나 부족/초과, 추가 suffix
- sampled null, false, true
- 유효한 inbound와 다른 active span이 동시에 있는 경우
- all-zero active ID
- 동일한 값이 두 번 들어온 중복 헤더

**실패 조건**

- invalid inbound를 그대로 전달하지 않는다.
- 부분적으로 유효한 ID를 보정하거나 padding하지 않는다.
- active context의 잘못된 길이를 가진 값을 조합하지 않는다.

**필요한 제약**

- baggage·tracestate는 범위 밖이다.
- 새 trace ID를 이 컴포넌트가 생성하지 않는다.
- 정규식 통과 후 all-zero 의미 검사를 별도로 수행한다.

### 구현 후 자가 검증

- [ ] 정상 sampled/unsampled 단일 값이 byte-for-byte 보존된다.
- [ ] 중복·대문자·all-zero·길이 오류가 거부된다.
- [ ] invalid inbound와 정상 active context 조합이 active 값으로 재구성된다.
- [ ] active sampled true만 `01`이고 false/null은 `00`이다.
- [ ] 양쪽 모두 유효하지 않으면 empty다.
- [ ] 입력 목록이 변경되지 않는다.
- [ ] 검증과 생성 결과가 동일한 규칙을 사용한다.

### 구현 후 설명할 것

- 유효 inbound 우선 정책이 distributed trace 연속성을 지키는 방식
- 중복 헤더를 첫 값으로 고르지 않고 거부한 이유
- trace context와 security trust header의 차이와 공통점
- 직접 문자열 생성 대신 tracing propagator를 사용할 때의 장단점

### 원본 확인 위치

- Thread 7
- 커밋: `feat(routing): propagate trace context`
- 커밋: `test(routing): verify trace propagation`
- 파일: `src/main/java/com/sportsbook/gateway/routing/TraceForwarding.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/TraceForwardingTest.java`
- 관련 Thread: 2, 15, 18
