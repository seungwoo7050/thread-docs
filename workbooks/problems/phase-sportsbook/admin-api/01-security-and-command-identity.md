# 보안 경계와 명령 식별자

이 문서는 마스터 인덱스의 W01~W04를 다룬다. 백지 구현은 실제 레포지토리 코드를 복원하는 문제가 아니라, 확인된 요구사항과 invariant를 10~30분 안에 다시 설계하도록 축소한 문제다. 요구사항을 만족한다면 원본과 다른 구현도 유효하다.

<a id="W01"></a>
## [Thread 03 / `feat(security): parse CIDR address ranges`; `feat(security): resolve trusted client addresses`] CIDR와 신뢰 프록시 해석

### 면접 질문

관리자 API 앞에 여러 프록시가 있을 때 실제 client IP를 어떻게 결정했습니까? 직접 접속 peer가 trusted proxy가 아닐 때 `X-Forwarded-For`를 무시해야 하는 이유와, trusted peer일 때 chain을 오른쪽부터 검사해야 하는 이유를 설명해 보세요.

꼬리 질문:

- IPv4와 IPv6 CIDR의 포함 여부를 byte 배열로 어떻게 판정합니까?
- prefix가 0, 8의 배수, 주소 전체 길이일 때 각각 무엇을 조심해야 합니까?
- chain 중 한 hop이라도 malformed라면 왜 일부만 사용하지 않고 fail closed 해야 합니까?
- direct peer와 forwarded hop이 모두 trusted proxy라면 어떤 값을 client로 선택해야 합니까?
- `InetAddress.getByName`을 그대로 사용하면 hostname 또는 비표준 표기가 끼어들 가능성을 어떻게 차단합니까?

### 30초 모범 답변

직접 연결한 peer만 서버가 확실히 관찰한 값이므로, 그 peer가 trusted proxy일 때만 전달 헤더를 신뢰합니다. trusted peer라면 chain의 오른쪽, 즉 서버에 가까운 hop부터 거슬러 올라가 첫 번째 untrusted 주소를 실제 client로 봅니다. 주소 하나라도 해석되지 않거나 chain이 모호하면 허용하지 않고, CIDR 비교는 같은 주소 family의 byte 배열에서 완전한 byte와 남은 bit mask를 비교합니다. 이 방식은 spoofed `X-Forwarded-For`가 allowlist를 우회하는 것을 막는 대신, 프록시 구성이 틀리면 요청을 거부하는 fail-closed 선택입니다.

### 답변 핵심 키워드

`direct peer` · `trust boundary` · `X-Forwarded-For` · `right-to-left` · `first untrusted hop` · `literal IP only` · `IPv4/IPv6 family match` · `prefix mask` · `fail closed`

### 백지 구현

#### 구현 목표

CIDR 포함 여부와 trusted proxy chain 해석을 구현한다. 프레임워크 request 객체는 사용하지 않고 값만 받는다.

#### 인터페이스 또는 함수 시그니처

```java
public final class NetworkBoundary {
  public static boolean contains(byte[] network, int prefixLength, byte[] candidate) {
    // 직접 구현
  }

  public static Optional<String> resolveClient(
      String remoteAddress,
      List<String> forwardedForHeaderValues,
      Predicate<String> trustedProxy) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- `contains`
  - 입력: 정규화된 network 주소 bytes, prefix 길이, candidate 주소 bytes
  - 출력: candidate가 해당 CIDR에 포함되면 `true`
- `resolveClient`
  - 입력: direct peer 문자열, 여러 개일 수 있는 `X-Forwarded-For` header 값, trusted proxy 판정 함수
  - 출력: 확정 가능한 client literal IP. 확정할 수 없으면 `Optional.empty()`

#### 반드시 만족해야 할 조건

- IPv4와 IPv6처럼 byte 길이가 다르면 포함되지 않는다.
- `prefixLength`는 `0..addressBits` 범위여야 한다.
- 완전한 byte 이후 남은 bit만 mask로 비교한다.
- `remoteAddress`가 literal IP가 아니면 client를 확정하지 않는다.
- direct peer가 trusted가 아니면 forwarded header를 전부 무시하고 peer를 반환한다.
- direct peer가 trusted라면 모든 forwarded hop이 유효한 literal IP여야 한다.
- 여러 header 값과 comma-separated 값을 원래 순서대로 하나의 chain으로 취급한다.
- trusted peer의 chain은 오른쪽부터 검사해 첫 untrusted hop을 반환한다.
- chain이 비어 있거나 모든 hop이 trusted이면 확정 실패로 처리한다.
- DNS 조회로 hostname을 IP로 바꾸지 않는다.

#### 경계 조건

- `/0`, `/32`, `/128`
- prefix가 8의 배수인 경우
- IPv4 network와 IPv6 candidate
- 한 header에 여러 hop, 여러 header에 분산된 hop
- 공백이 포함된 정상 literal
- 빈 token, 연속 comma, 잘못된 IPv6

#### 실패 조건

- prefix 범위 초과
- direct peer 또는 forwarded hop 파싱 실패
- trusted direct peer인데 usable chain이 없음
- chain 전체가 trusted proxy로만 구성됨

#### 필요한 제약

- 시간 복잡도는 CIDR 비교가 주소 byte 길이에 선형, chain 해석이 hop 수에 선형이어야 한다.
- 입력을 수정하지 않는다.
- parsing 실패 시 부분 결과를 사용하지 않는다.

### 구현 후 자가 검증

- [ ] IPv4 network 시작 주소와 마지막 주소가 포함된다.
- [ ] 바로 다음 network 주소는 제외된다.
- [ ] IPv6 prefix의 남은 bit mask가 정확하다.
- [ ] `/0`은 같은 address family의 모든 주소를 포함한다.
- [ ] untrusted peer가 보낸 위조 forwarded header는 무시된다.
- [ ] trusted proxy 2개 뒤의 client가 오른쪽부터 올바르게 선택된다.
- [ ] malformed hop 하나가 섞이면 전체 해석이 실패한다.
- [ ] 모든 hop이 trusted일 때 임의 주소를 client로 선택하지 않는다.
- [ ] hostname 문자열이 DNS를 통해 허용되지 않는다.
- [ ] 시간 복잡도가 `O(addressBytes + hopCount)` 범위다.

### 구현 후 설명할 것

1. forwarded chain을 왼쪽이 아니라 오른쪽부터 검사한 이유
2. malformed chain을 부분 수용하지 않은 이유
3. 문자열 비교가 아니라 byte/prefix 비교를 선택한 이유
4. 프록시 설정 오류 시 가용성보다 보안을 우선한 trade-off
5. literal IP parsing과 DNS resolution을 분리한 이유

### 원본 확인 위치

- Thread 03
- `feat(security): parse CIDR address ranges`
- `feat(security): resolve trusted client addresses`
- `feat(security): enforce the admin IP allowlist`
- `CidrBlock`
- `TrustedProxyResolver`
- `IpAllowlistFilter`
- `AdminNetworkProperties`
- 관련 Thread: 04

---

<a id="W02"></a>
## [Thread 04 / `feat(security): require bounded JWT lifetime`; `feat(security): decode verified RS256 tokens`] JWT 운영자 신뢰 경계

### 면접 질문

JWT의 RSA 서명이 검증되었다면 왜 `exp`, `nbf`, `sub`, `role`, issuer를 다시 강하게 검증해야 합니까? 이 프로젝트에서 RS256과 2048-bit 이상 SPKI public key를 고정하고 clock skew를 0으로 둔 이유를 설명해 보세요.

꼬리 질문:

- `exp`가 없거나 현재 시각과 같으면 어떻게 처리해야 합니까?
- subject 길이를 Java `String.length()` 대신 code point 기준으로 볼 이유가 무엇입니까?
- `" ADMIN "`, 소문자 role, 배열 형태 role을 허용하지 않는 이유는 무엇입니까?
- issuer가 설정된 환경과 설정되지 않은 환경의 정책은 어떻게 달라집니까?
- 테스트에서 `Clock`을 주입하는 것이 왜 중요한가요?
- public key 파싱 오류 메시지에 원문 PEM을 넣으면 어떤 위험이 있습니까?

### 30초 모범 답변

서명 검증은 토큰이 특정 key로 서명되었다는 것만 보장하고, 지금 이 서비스의 운영자 권한으로 사용 가능한지는 claim 정책이 결정합니다. 그래서 RS256만 허용하고 충분한 RSA key 길이를 요구하며, `exp` 필수·zero skew·`nbf` 검증으로 시간 경계를 명확히 합니다. `sub`는 공백·control character·길이를 제한하고 `role`은 네 값 중 정확한 문자열만 허용하며, issuer가 설정되면 exact match를 요구합니다. 실패 메시지에는 token이나 key를 넣지 않고 모든 누락·모호한 경우를 거부합니다.

### 답변 핵심 키워드

`signature authenticity` · `authorization semantics` · `algorithm pinning` · `RS256` · `SPKI` · `2048-bit` · `exp required` · `zero skew` · `nbf` · `exact issuer` · `code points` · `exact role` · `no secret echo`

### 백지 구현

#### 구현 목표

서명 자체는 신뢰할 수 있는 JWT 라이브러리가 완료했다고 가정하고, 운영자 claim policy를 순수 함수로 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public record OperatorClaims(
    String algorithm,
    Instant expiresAt,
    Instant notBefore,
    String subject,
    String role,
    String issuer) {}

public final class OperatorTokenPolicy {
  public static void validate(
      OperatorClaims claims,
      Instant now,
      Optional<String> expectedIssuer) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 서명 검증된 token의 핵심 claim, 현재 시각, 선택적 expected issuer
- 출력: 유효하면 정상 반환, 무효하면 하나의 정책 예외 발생

#### 반드시 만족해야 할 조건

- algorithm은 정확히 `RS256`이어야 한다.
- `expiresAt`은 필수이며 `now`보다 뒤여야 한다. clock skew는 없다.
- `notBefore`가 있으면 `now`보다 뒤일 수 없다.
- subject는 null/blank가 아니어야 한다.
- subject는 앞뒤 공백이 없어야 한다.
- subject는 Unicode code point 기준 128개 이하여야 한다.
- subject에 ISO control character가 없어야 한다.
- role은 `ADMIN`, `TRADER`, `CS`, `READONLY` 중 정확히 하나여야 한다.
- expected issuer가 있으면 token issuer가 null이 아니고 exact match여야 한다.
- expected issuer가 없을 때는 임의 issuer를 신뢰 근거로 사용하지 않는다.
- 예외 메시지에 token 원문, key, 전체 claim dump를 넣지 않는다.

#### 경계 조건

- `expiresAt == now`
- `notBefore == now`
- 128 code point와 129 code point subject
- supplementary Unicode character가 포함된 subject
- leading/trailing Unicode whitespace
- control character가 중간에 포함된 subject
- role의 대소문자·공백 변형
- issuer 설정 유무

#### 실패 조건

- 필수 claim 누락
- 만료 또는 아직 유효하지 않은 token
- 허용하지 않은 algorithm/role
- subject 형식 위반
- issuer mismatch

#### 필요한 제약

- 검증은 입력 길이에 선형이어야 한다.
- 시스템 clock을 함수 내부에서 직접 읽지 않는다.
- 여러 실패가 있더라도 민감값을 포함하지 않는 안정된 오류 유형을 사용한다.

### 구현 후 자가 검증

- [ ] 정상 RS256 claim set이 통과한다.
- [ ] `exp` 누락과 `exp == now`가 거부된다.
- [ ] future `nbf`가 거부되고 `nbf == now`는 통과한다.
- [ ] 128 code point는 통과하고 129는 거부된다.
- [ ] emoji 등 surrogate pair가 문자 두 개로 잘못 계산되지 않는다.
- [ ] 공백·control character subject가 거부된다.
- [ ] 정확한 네 role 외에는 모두 거부된다.
- [ ] configured issuer가 exact match가 아니면 거부된다.
- [ ] 실패 메시지에 subject 전체나 token/key가 노출되지 않는다.
- [ ] 고정 `Instant`로 경계 테스트를 재현할 수 있다.

### 구현 후 설명할 것

1. 서명 검증과 서비스별 claim policy를 분리한 이유
2. zero clock skew가 운영 안정성에 주는 장단점
3. role을 느슨하게 정규화하지 않고 exact match한 이유
4. code point 기준 길이를 선택한 이유
5. time source를 주입해 테스트 가능하게 만든 이유

### 원본 확인 위치

- Thread 04
- `feat(security): require bounded JWT lifetime`
- `feat(security): validate operator subjects`
- `feat(security): validate operator role claims`
- `feat(security): decode verified RS256 tokens`
- `AdminJwtTimestampValidator`
- `AdminJwtSubjectValidator`
- `AdminJwtRoleValidator`
- `JwtDecoderConfiguration`
- `RsaPublicKeyParser`
- `AdminRole`
- 관련 Thread: 03, 05

---

<a id="W03"></a>
## [Thread 05 / `feat(context): create mutation identities early`; `feat(api): require one idempotency header`] action ID와 idempotency key lifecycle

### 면접 질문

`X-Admin-Action-Id`와 `Idempotency-Key`는 둘 다 요청 식별자처럼 보입니다. 둘의 의미와 수명은 어떻게 다르며, 왜 action ID는 인증 이후이지만 권한 검사·body 검증·하위 호출 이전에 생성해야 합니까?

꼬리 질문:

- 같은 논리 명령을 재시도할 때 두 값 중 무엇을 유지하고 무엇을 새로 생성합니까?
- 하나의 HTTP request에서 resolver와 filter가 여러 번 접근해도 action ID를 한 번만 만드는 방법은 무엇입니까?
- 같은 이름의 `Idempotency-Key` header가 두 개 오면 첫 번째만 쓰면 안 되는 이유는 무엇입니까?
- 일부 settlement endpoint가 UUID key만 받는 반면 다른 endpoint는 raw protocol key를 보존하는 이유는 무엇입니까?
- 인증 실패 요청에도 action ID를 붙여야 합니까? 프로젝트의 경계는 어디입니까?
- 응답 실패 시에도 action ID header를 남기는 이유는 무엇입니까?

### 30초 모범 답변

idempotency key는 여러 HTTP 시도를 하나의 논리 명령으로 묶어 하위 서비스의 중복 적용을 막는 값이고, action ID는 각 물리 시도를 감사·추적하는 새 UUIDv7입니다. 따라서 재시도는 같은 idempotency key와 다른 action ID를 가집니다. action ID는 인증된 `/admin/v1/` mutation에서 한 번 생성해 request attribute에 저장하고, 이후 권한 거절이나 validation 실패도 같은 ID로 응답·감사합니다. idempotency header는 정확히 하나만 허용하고 raw 값 보존 또는 UUID 전용 계약을 endpoint별로 명확히 합니다.

### 답변 핵심 키워드

`logical command` · `physical attempt` · `same idempotency key` · `new action ID` · `request attribute cache` · `after authentication` · `before authorization/validation` · `exactly one header` · `raw preservation` · `UUID-only contract` · `correlation`

### 백지 구현

#### 구현 목표

요청 메타데이터를 받아 mutation action identity를 한 번 생성하고, endpoint 정책에 따라 idempotency key를 검증한다.

#### 인터페이스 또는 함수 시그니처

```java
public record RequestMeta(
    boolean authenticated,
    String method,
    String path,
    List<String> idempotencyHeaderValues,
    boolean idempotencyRequired,
    boolean uuidIdempotencyRequired) {}

public record MutationIdentity(UUID actionId, Optional<String> idempotencyKey) {}

public interface ActionIdGenerator {
  UUID next();
}

public final class MutationIdentityResolver {
  public static Optional<MutationIdentity> resolve(
      RequestMeta request,
      Map<String, Object> requestAttributes,
      ActionIdGenerator generator) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 인증 여부, method/path, header values, endpoint key 정책, request-local attribute map, ID generator
- 출력: 대상 mutation이면 identity, 대상이 아니면 empty

#### 반드시 만족해야 할 조건

- 인증된 `POST`, `PATCH`, `DELETE` 중 `/admin/v1/` 경계에만 action ID를 만든다.
- 동일 request에서 이미 생성한 action ID가 있으면 그대로 재사용한다.
- 새 action ID는 UUIDv7이어야 하며, 생성기는 동일 request에서 최대 한 번 호출된다.
- idempotency가 필요하지 않은 endpoint는 key 없이도 identity를 만든다.
- idempotency가 필요한 endpoint는 header 값이 정확히 하나여야 한다.
- 일반 key endpoint는 raw 문자열을 프로젝트의 protocol key 검증에 그대로 넘기고, 성공한 값도 정규화하지 않는다.
- UUID 전용 endpoint는 raw 문자열 전체가 UUID로 파싱되어야 하며 앞뒤 공백을 임의로 제거하지 않는다.
- duplicate header를 첫 값으로 축약하지 않는다.
- logical retry는 같은 key를 사용할 수 있지만 resolver가 action ID를 이전 request와 공유하지 않는다.
- 실패가 발생해도 생성된 action ID를 응답에 사용할 수 있도록 request-local state에 남긴다.

#### 경계 조건

- `GET`, `POST`, `PATCH`, `DELETE`
- `/admin/v1/x`, `/administrator`, `/actuator/health`
- 인증 전/후
- header 없음, 하나, 둘 이상
- 빈 문자열, 공백 문자열, 앞뒤 공백과 내부 공백을 포함한 general key
- UUID 문자열 앞뒤에 공백이 있는 경우와 정상 UUID 표기
- resolver 중복 호출

#### 실패 조건

- key 필수인데 누락
- duplicate key
- general key가 protocol validator에서 거부됨
- UUID 전용 key의 parsing 실패
- action ID generator가 null을 반환하는 비정상 상황

#### 필요한 제약

- request-local map 외의 global mutable state를 사용하지 않는다.
- key 값을 로그나 예외 메시지에 그대로 노출하지 않는다.
- 해결 과정은 header 수에 선형이어야 한다.

### 구현 후 자가 검증

- [ ] 인증되지 않은 요청에는 action ID가 생성되지 않는다.
- [ ] 인증된 mutation에는 action ID가 생성된다.
- [ ] 새 action ID의 UUID version이 7이다.
- [ ] 같은 request를 두 번 resolve해도 같은 ID이고 generator 호출은 한 번이다.
- [ ] 다른 request의 동일 idempotency key는 서로 다른 action ID를 가진다.
- [ ] duplicate header가 거부된다.
- [ ] protocol contract가 허용한 general key의 앞뒤 공백까지 downstream 전달용 raw 값으로 보존된다.
- [ ] UUID 전용 정책에서 일반 문자열이나 공백이 덧붙은 UUID가 거부된다.
- [ ] key가 필요 없는 risk mutation도 action identity는 갖는다.
- [ ] path prefix가 비슷한 `/administrator`에는 적용되지 않는다.
- [ ] 오류 응답에서도 request-local action ID를 회수할 수 있다.

### 구현 후 설명할 것

1. 논리 명령과 물리 시도를 별도 ID로 모델링한 이유
2. action ID 생성 시점을 인증 뒤·권한 검사 앞에 둔 이유
3. duplicate header를 모호한 입력으로 거부한 이유
4. global cache 대신 request attribute를 사용한 이유
5. 모든 endpoint에 동일 key 형식을 강제하지 않은 trade-off

### 원본 확인 위치

- Thread 05
- `feat(context): create mutation identities early`
- `feat(api): require one idempotency header`
- `AdminMutationContextFilter`
- `AdminContextArgumentResolver`
- `AdminContext`
- `AdminRequestHeaders`
- `AdminRequestException`
- 관련 Thread: 09, 11, 16, 18, 19, 20
- Thread 18: `feat(odds): delegate all market actions`, `test(odds): preserve four market headers`, `MarketClient`

---

<a id="W04"></a>
## [Thread 06 / `feat(client): validate isolated downstream credentials`; `test(client): prevent cross-client secret leakage`] 서비스별 credential과 RestClient 격리

### 면접 질문

네 하위 서비스가 모두 HTTP API인데 왜 하나의 공용 RestClient와 동적 interceptor를 쓰지 않고 별도 client bean으로 분리했습니까? 구성 시점 검증과 cross-client secret leakage 테스트가 각각 막는 실패를 설명해 보세요.

꼬리 질문:

- base URL을 "HTTP origin"으로 검증할 때 어떤 URI 구성 요소를 거부해야 합니까?
- 네 key가 각각 충분히 길어도 서로 같은 값이면 왜 시작 실패시켜야 합니까?
- builder를 clone하지 않고 순차적으로 default header를 붙이면 어떤 문제가 생길 수 있습니까?
- Wallet/Risk/Odds와 Settlement가 서로 다른 header family를 사용할 때 어떻게 오염을 방지합니까?
- credential 객체의 `toString()`과 exception message는 어떻게 설계해야 합니까?
- 별도 client 수가 늘어나는 비용과 보안 이득의 trade-off는 무엇입니까?

### 30초 모범 답변

자격 증명은 대상 서비스와 결합된 권한이므로 client도 그 경계를 구조적으로 분리했습니다. 각 base URI는 absolute HTTP/HTTPS, host 존재, user-info·query·fragment 없음으로 검증하고 timeout은 양수, 네 key는 필수·최소 길이·상호 distinct로 시작 시점에 확인합니다. 각 RestClient는 clone된 builder와 해당 서비스 key, 해당 header family만 갖게 해 조건문 실수로 다른 secret이 전송되는 것을 막습니다. 대신 bean 수와 설정 코드는 늘지만, 누출 blast radius와 리뷰 복잡도가 줄어듭니다.

### 답변 핵심 키워드

`credential-to-service binding` · `least privilege` · `fail fast` · `absolute HTTP origin` · `positive timeout` · `pairwise distinct keys` · `builder clone` · `header family` · `no secret in toString` · `cross-client integration test`

### 백지 구현

#### 구현 목표

네 서비스의 URI·timeout·credential을 검증하고, 각 서비스에 필요한 default header만 가진 불변 client specification을 만든다. 실제 네트워크 client 생성은 하지 않는다.

#### 인터페이스 또는 함수 시그니처

```java
public enum Service { WALLET, RISK, ODDS, SETTLEMENT }

public record ClientSpec(
    URI baseUri,
    Duration connectTimeout,
    Duration readTimeout,
    Map<String, String> defaultHeaders) {}

public final class DownstreamClientSpecs {
  public static Map<Service, ClientSpec> create(
      Map<Service, URI> origins,
      Map<Service, String> credentials,
      Duration connectTimeout,
      Duration readTimeout) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 서비스별 origin, 서비스별 credential, 공통 connect/read timeout
- 출력: 네 서비스 모두를 포함하는 불변 `Map<Service, ClientSpec>`

#### 반드시 만족해야 할 조건

- 네 서비스 origin과 credential이 모두 있어야 한다.
- URI는 absolute이고 scheme은 정확히 `http` 또는 `https`다.
- host가 있어야 하고 user-info, query, fragment는 없어야 한다.
- connect/read timeout은 0보다 커야 한다.
- credential은 null/blank가 아니고 최소 32문자다.
- 네 credential은 pairwise distinct여야 한다.
- WALLET/RISK/ODDS에는 `X-Internal-Service: admin-api`와 해당 `X-Internal-Api-Key`만 둔다.
- SETTLEMENT에는 `X-Service-Name: admin-api`와 해당 `X-API-Key`만 둔다.
- 다른 서비스 key나 다른 header family가 섞이면 안 된다.
- 반환 map과 header map은 외부에서 수정할 수 없어야 한다.
- exception이나 객체 문자열 표현에 credential 원문을 포함하지 않는다.

#### 경계 조건

- `ftp`, 상대 URI, host 없는 URI
- user-info, query, fragment가 있는 URI
- timeout 0, 음수
- 길이 31/32 credential
- 두 서비스만 같은 key
- settlement와 internal header name 혼동
- 입력 map이 이후 변경되는 경우

#### 실패 조건

- 구성 일부 누락
- origin 또는 timeout 위반
- key 길이·중복 위반
- unknown service 또는 네 서비스가 모두 구성되지 않음

#### 필요한 제약

- 생성 결과는 deep immutable이어야 한다.
- credential 비교는 값 비교로 수행하되, 오류 메시지에 값을 넣지 않는다.
- 생성 시간은 서비스 수와 credential 길이에 선형이면 충분하다.

### 구현 후 자가 검증

- [ ] 네 정상 spec이 모두 생성된다.
- [ ] 각 spec에는 자기 key만 존재한다.
- [ ] Settlement spec에 internal header가 없다.
- [ ] 나머지 spec에 settlement header가 없다.
- [ ] 한 key를 재사용하면 전체 구성이 실패한다.
- [ ] 길이 31은 실패하고 32는 통과한다.
- [ ] 상대 URI와 `ftp` URI가 거부된다.
- [ ] user-info/query/fragment가 있는 URI가 거부된다.
- [ ] timeout 0과 음수가 거부된다.
- [ ] 반환 map과 nested header map을 수정할 수 없다.
- [ ] 예외와 `toString()` 경로에서 secret이 노출되지 않는다.

### 구현 후 설명할 것

1. 공용 interceptor보다 별도 client를 선택한 이유
2. configuration validation을 첫 요청 시점이 아니라 startup에 둔 이유
3. key를 distinct로 강제한 이유와 운영상의 제약
4. builder clone 또는 immutable spec이 필요한 이유
5. integration test에서 "예상 key 존재"뿐 아니라 "다른 key 부재"를 확인해야 하는 이유

### 원본 확인 위치

- Thread 06
- `feat(client): validate isolated downstream credentials`
- `feat(client): validate downstream HTTP origins`
- `feat(client): isolate the Wallet RestClient`
- `feat(client): isolate the Risk RestClient`
- `feat(client): isolate the Odds RestClient`
- `feat(client): isolate the Settlement RestClient`
- `test(client): prevent cross-client secret leakage`
- `DownstreamCredentials`
- `DownstreamProperties`
- `DownstreamClientConfiguration`
- `DownstreamHeaders`
- `CrossClientSecretIsolationTest`
- 관련 Thread: 15, 16, 17, 18, 19, 20
