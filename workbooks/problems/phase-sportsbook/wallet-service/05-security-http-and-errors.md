# 내부 보안, HTTP identity, 오류 계약

이 문서는 요청이 업무 코드에 도달하기 전에 닫아야 할 경계를 다룬다. 자격 증명 검증, route capability, 업무 의미 권한, 요청 identity, 오류 응답은 각각 다른 공격·오작동 경로를 막는다.

<a id="w12"></a>
## W12 — [Thread 15 / `feat(security): compare caller credential digests`] 내부 API key 인증

### 면접 질문

내부 서비스 인증에서 `X-Internal-Service`와 `X-Internal-Api-Key`를 받는다고 할 때, 헤더 없음·일부만 존재·중복 값·알 수 없는 caller·잘못된 key를 어떻게 구분해야 합니까? 이 프로젝트가 unknown caller에도 digest 비교를 수행하는 이유를 설명해 보세요.

꼬리 질문:

- `request.getHeader()` 하나만 읽으면 중복 헤더에서 어떤 모호성이 생깁니까?
- API key 원문 대신 SHA-256 digest를 프로세스에서 비교하는 목적과 한계는 무엇입니까?
- unknown caller를 바로 반환하지 않고 dummy digest와 비교하면 어떤 timing 차이를 줄일 수 있습니까?
- `MessageDigest.isEqual`과 일반 문자열 `equals`의 차이는 무엇입니까?
- 인증 성공 뒤 SecurityContext에 API key를 credential로 보관하면 어떤 위험이 있습니까?
- 시작 시 key 길이와 caller 간 중복을 검증하는 이유는 무엇입니까?

### 30초 모범 답변

두 헤더가 모두 없으면 아직 인증하지 않은 요청으로 다음 보안 단계에 넘기고, 하나만 있거나 값이 여러 개면 모호한 자격 증명으로 즉시 거부합니다. 정확히 하나씩 있을 때만 case-sensitive caller wire name을 찾고, 저장한 SHA-256 digest와 제시 key digest를 `MessageDigest.isEqual`로 비교합니다. unknown caller도 dummy digest를 사용해 비교 경로를 맞추고, 성공 결과에는 caller identity만 넣어 key 원문을 SecurityContext·로그·오류에 남기지 않습니다. 시작 시 다섯 key가 충분히 길고 서로 다른지도 검증해 잘못된 배포를 fail fast합니다.

### 답변 핵심 키워드

`exactly one header pair` · `partial/duplicate reject` · `case-sensitive wire name` · `digest only` · `constant-time compare` · `dummy digest` · `no secret retention` · `startup validation`

### 백지 구현

#### 구현 목표

헤더 목록과 caller별 설정 key를 받아 `Anonymous`, `Authenticated`, `Rejected` 중 하나를 반환한다. 결과나 오류에 API key 원문을 포함하지 않는다.

#### 면접용 축소 인터페이스

```java
enum Caller { PLATFORM, GATEWAY, BETTING, SETTLEMENT, ADMIN }

sealed interface AuthResult {
  record Anonymous() implements AuthResult {}
  record Authenticated(Caller caller) implements AuthResult {}
  record Rejected() implements AuthResult {}
}

record CredentialConfig(Map<Caller, String> apiKeys) {}

final class InternalAuthenticator {
  InternalAuthenticator(CredentialConfig config) {
    // 직접 구현
  }

  AuthResult authenticate(
      List<String> serviceHeaderValues,
      List<String> apiKeyHeaderValues) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 같은 헤더 이름에 들어온 모든 service 값과 모든 key 값
- 출력: anonymous, caller만 담은 authenticated, 정보 없는 rejected

#### 반드시 만족해야 할 조건

설정 시:

- 모든 정의된 caller에 key가 있어야 한다.
- 각 key는 null·blank가 아니고 최소 32자다.
- caller별 key는 서로 달라야 한다.
- 비교용으로는 caller별 SHA-256 digest만 보관한다.
- unknown caller 비교용 고정 dummy digest를 준비한다.

요청 시:

- 두 목록이 모두 비어 있으면 `Anonymous`다.
- 어느 한쪽만 비어 있거나 어느 목록이든 크기가 1이 아니면 `Rejected`다.
- caller wire name은 trim·소문자화 없이 정확히 비교한다.
- 알려진 caller와 unknown caller 모두 presented key를 digest하고 비교 함수를 호출한다.
- 알려진 caller의 expected digest와 presented digest가 일치할 때만 `Authenticated`다.
- 반환 객체·예외·로그용 문자열에 key 원문이나 digest를 넣지 않는다.

#### 경계 조건

- 두 헤더 모두 없음
- service만 있음, key만 있음
- 같은 값이 두 번 들어온 중복 헤더
- 서로 다른 값이 여러 개 들어온 헤더
- 올바른 caller와 다른 caller의 key 조합
- 대소문자 또는 앞뒤 공백이 다른 caller wire name
- unknown caller와 임의 key
- 빈 key, null에 해당하는 입력
- caller 두 개가 같은 설정 key를 가진 시작 구성
- 정확히 32자인 key

#### 실패 조건

- 불완전·중복 자격 증명
- 알 수 없는 caller
- digest 불일치
- 잘못된 시작 구성
- SHA-256 알고리즘을 사용할 수 없는 런타임 오류

#### 필요한 제약

- 일반 문자열 비교로 secret을 검증하지 않는다.
- `Rejected`는 caller 존재 여부나 어떤 값이 틀렸는지 구분해 주지 않는다.
- 요청마다 설정 key 원문을 다시 보관하거나 인증 객체에 복사하지 않는다.
- API key 발급·배포·회전 자체는 이 축소 코드가 해결하는 범위가 아니다.

### 구현 후 자가 검증

- [ ] 두 헤더 모두 없을 때만 anonymous다.
- [ ] 부분 헤더와 중복 헤더를 모두 거부한다.
- [ ] 모든 올바른 caller/key 조합이 자신의 caller로 인증된다.
- [ ] cross-caller key 조합이 모두 거부된다.
- [ ] unknown caller도 digest 비교 경로를 거친다.
- [ ] caller wire name을 trim하거나 case-fold하지 않는다.
- [ ] 결과·예외 메시지·`toString()`에 key가 없다.
- [ ] 짧은 key와 중복 key 구성에서 객체 생성이 실패한다.
- [ ] 비교에 `MessageDigest.isEqual`과 같은 상수시간 목적 API를 사용한다.

### 구현 후 설명할 것

1. 헤더가 모두 없는 경우를 filter에서 곧바로 401로 만들지 않고 authorization 단계에 넘긴 이유.
2. digest 비교가 secret manager·TLS·key rotation을 대체하지 않는다는 한계.
3. unknown caller dummy digest가 줄이는 정보 차이와 완전한 timing 방어가 아닌 이유.
4. 인증 principal에는 caller만 넣고 credential을 null로 둔 이유.
5. key 길이·고유성을 시작 시 검증해 runtime ambiguity를 줄인 이유.

### 원본 확인 위치

- Thread: **15 — 내부 서비스 인증과 권한 경계**
- 커밋: `feat(security): define caller wire names`
- 커밋: `feat(security): validate caller API keys`
- 커밋: `feat(security): compare caller credential digests`
- 커밋: `feat(security): authenticate internal API keys`
- 커밋: `test(security): reject cross-caller credentials`
- 커밋: `test(security): reject ambiguous service credentials`
- 파일: `src/main/java/com/sportsbook/wallet/domain/WalletCaller.java`
- 파일: `src/main/java/com/sportsbook/wallet/security/WalletSecurityProperties.java`
- 파일: `src/main/java/com/sportsbook/wallet/security/WalletCredentials.java`
- 파일: `src/main/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilter.java`
- 클래스·메서드: `WalletCaller.fromWireName()`, `WalletCredentials.authenticate()`, `InternalApiKeyAuthenticationFilter.doFilterInternal()`
- 관련 Thread: 9, 16, 17

---

<a id="w13"></a>
## W13 — [Thread 15 / `feat(security): authorize wallet route capabilities`] 닫힌 route와 업무 의미 권한

### 면접 질문

내부 caller 인증에 성공했다면 모든 `/internal/v1/wallet/**` 요청을 허용해도 됩니까? 이 프로젝트가 method+path 기반 route capability와 credit source/reason 기반 업무 권한을 둘 다 둔 이유를 설명해 보세요.

꼬리 질문:

- unauthenticated 요청과 authenticated-but-forbidden 요청은 각각 어떤 상태여야 합니까?
- `anyRequest().denyAll()` 같은 닫힌 기본값이 새 endpoint 추가에 주는 효과는 무엇입니까?
- `BETTING`이 credit endpoint에 접근할 수 있어도 `HOUSE_POOL + PAYOUT`을 실행하면 안 되는 이유는 무엇입니까?
- actuator health와 metrics를 모두 anonymous로 열지 않은 이유는 무엇입니까?
- path 문자열에 `startsWith`를 쓰는 방식보다 framework route matcher를 쓰는 이유는 무엇입니까?

### 30초 모범 답변

인증은 호출자의 신원만 증명하고 그 caller가 어떤 돈 흐름을 실행할 수 있는지는 별도 권한입니다. 그래서 HTTP method와 path를 caller allowlist로 닫고 나머지는 모두 deny합니다. credit처럼 한 endpoint가 여러 업무를 받는 경우에는 서비스 계층에서 caller, source, reason 조합까지 다시 검증합니다. 예를 들어 BETTING은 사용자 locked stake의 refund만 가능하고, SETTLEMENT만 house-funded payout을 할 수 있습니다. 자격 증명이 없으면 401, 신원은 맞지만 capability가 없으면 403으로 구분합니다.

### 답변 핵심 키워드

`authentication ≠ authorization` · `method+path capability` · `deny by default` · `401 vs 403` · `semantic authorization` · `source/reason matrix` · `defense in depth`

### 백지 구현

#### 구현 목표

실제 프로젝트 규칙을 축소한 route allowlist와 credit 의미 권한 함수를 작성한다. 등록되지 않은 method/path/조합은 항상 거부한다.

#### 면접용 축소 인터페이스

```java
enum Caller { PLATFORM, GATEWAY, BETTING, SETTLEMENT, ADMIN }
enum HttpMethod { GET, POST }
enum CreditSource { USER_LOCKED, HOUSE_POOL }
enum CreditReason { PAYOUT, VOID, REFUND }

final class WalletAuthorization {
  static boolean routeAllowed(Caller caller, HttpMethod method, String path) {
    // 직접 구현
  }

  static boolean creditAllowed(
      Caller caller,
      CreditSource source,
      CreditReason reason) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 인증된 caller, HTTP method/path 또는 credit source/reason
- 출력: 허용 여부
- 인증되지 않은 요청의 401 처리는 호출자 밖의 보안 계층이 담당한다고 가정한다.

#### 반드시 만족해야 할 조건

route capability:

- `PLATFORM`
  - POST `/internal/v1/wallet/accounts`
  - GET `/internal/v1/wallet/accounts/{id}/balance`
  - POST deposit·withdraw
  - 비공개 actuator route
- `GATEWAY`
  - GET balance
- `BETTING`
  - POST debit
  - GET debit outcome
  - POST credit route
- `SETTLEMENT`
  - POST credit·forfeit·adjustment
  - GET adjustment proof
- `ADMIN`
  - POST credit route
- anonymous 공개 범위는 GET health 계열과 prometheus로 제한된다고 설명할 수 있어야 한다.
- 위에 명시되지 않은 method/path는 false다.
- path parameter는 정확한 route pattern으로 매칭하며 임의 prefix 허용을 만들지 않는다.

credit 의미 권한:

- `BETTING`: `USER_LOCKED + REFUND`만 허용
- `SETTLEMENT`: `USER_LOCKED + VOID`, `USER_LOCKED + REFUND`, `HOUSE_POOL + PAYOUT` 허용
- `ADMIN`: `HOUSE_POOL + REFUND`만 허용
- 그 밖의 모든 caller/source/reason 조합은 false

#### 경계 조건

- 올바른 path지만 method가 다른 요청
- 비슷한 prefix를 가진 미등록 path
- route에는 접근 가능하지만 의미 조합이 금지된 credit
- BETTING의 house-funded payout 시도
- SETTLEMENT의 locked-source payout 시도
- ADMIN의 사용자 locked refund 시도
- PLATFORM의 credit route 시도
- actuator health GET과 POST 차이

#### 실패 조건

- null caller·method·path
- 알 수 없는 path
- 금지된 source/reason 조합
- path normalization을 우회하는 잘못된 입력

#### 필요한 제약

- 기본값은 허용이 아니라 거부다.
- route 권한과 의미 권한을 하나의 거대한 문자열 조건으로 섞지 않는다.
- caller enum 이름과 외부 wire name을 혼동하지 않는다.
- 권한 거부 오류에 secret이나 내부 정책 상세를 불필요하게 노출하지 않는다.

### 구현 후 자가 검증

- [ ] 각 caller의 허용 route를 하나씩 확인했다.
- [ ] 허용 route의 method를 바꾸면 거부된다.
- [ ] 미등록 path는 모든 caller에게 거부된다.
- [ ] anonymous 공개 actuator 범위와 platform 전용 범위를 구분한다.
- [ ] credit 허용 조합을 정확히 열거했다.
- [ ] 허용 조합 수보다 훨씬 많은 금지 조합을 기본 false로 처리한다.
- [ ] route 허용만으로 semantic credit이 자동 허용되지 않는다.
- [ ] 인증 실패 401과 권한 부족 403의 책임 계층을 설명할 수 있다.

### 구현 후 설명할 것

1. route-level allowlist와 service-level semantic authorization을 분리한 이유.
2. `denyAll` 기본값이 endpoint 추가 시 의도적인 권한 변경을 강제하는 방식.
3. 권한 매트릭스를 코드·테스트에서 열거하는 장점과 변경 비용.
4. 내부 네트워크에 있다는 사실만으로 monetary API를 신뢰하지 않은 이유.
5. caller가 늘어날 때 role hierarchy보다 explicit capability를 유지할지에 대한 trade-off.

### 원본 확인 위치

- Thread: **15 — 내부 서비스 인증과 권한 경계**
- 커밋: `feat(security): authorize wallet route capabilities`
- 커밋: `test(security): lock caller route capabilities`
- 파일: `src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java`
- 클래스·메서드: `WalletSecurityConfig`, caller별 `AuthorizationManager`
- 테스트: `WalletSecurityConfigTest`
- 관련 Thread: **9 — 환불·지급·몰수 이체**
- 커밋: `feat(service): prepare semantic credit transfers`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java`
- 클래스·메서드: `WalletTransferExecutor.requireAllowedCredit()`
- 테스트: `WalletCreditAuthorizationTest`
- 관련 Thread: 16, 17

---

<a id="w14"></a>
## W14 — [Thread 16 / `feat(api): parse wallet request identities`] 단일값 헤더와 canonical UUID

### 면접 질문

`Idempotency-Key`를 `request.getHeader()`로 하나만 읽고, debit path의 UUID를 `UUID.fromString()`으로 파싱하는 것만으로 request identity가 하나로 고정됩니까? 이 프로젝트가 추가로 검사한 두 가지 경계를 설명해 보세요.

꼬리 질문:

- 같은 `Idempotency-Key` 값이 헤더에 두 번 들어오면 왜 허용하지 않습니까?
- 서로 다른 duplicate 값에서 첫 값만 읽는 것은 어떤 보안·멱등성 문제를 만듭니까?
- Java의 UUID parser가 받아들이는 문자열과 canonical lowercase UUID 문자열은 항상 같습니까?
- parser 결과를 다시 문자열로 만들어 원본과 비교하는 검사가 어떤 표현을 거부합니까?
- malformed identity가 durable operation을 만들기 전에 거부되어야 하는 이유는 무엇입니까?

### 30초 모범 답변

요청 identity는 값뿐 아니라 표현도 하나여야 합니다. 그래서 header enumeration으로 정확히 한 개의 `Idempotency-Key`만 허용하고, 같은 값을 두 번 보내도 모호한 요청으로 거부합니다. UUID는 파싱 성공만 보지 않고 `parsed.toString().equals(raw)`를 확인해 표준 lowercase 8-4-4-4-12 표현만 받습니다. trim이나 case normalization을 하지 않으므로 프록시·클라이언트마다 다른 표현이 같은 durable key로 합쳐지지 않습니다. 이런 형식 오류는 업무 operation 생성 전에 끝나야 key를 소비하지 않습니다.

### 답변 핵심 키워드

`all header values` · `exactly one` · `duplicate reject` · `no normalization` · `parse + round-trip equality` · `canonical UUID` · `fail before durable write`

### 백지 구현

#### 구현 목표

다중값 HTTP 헤더 맵에서 정확히 한 값을 추출하고, 문자열 UUID가 canonical form인지 검증한다.

#### 면접용 축소 인터페이스

```java
final class RequestIdentities {
  static String requireSingleHeader(
      Map<String, List<String>> headers,
      String headerName) {
    // 직접 구현
  }

  static UUID requireCanonicalUuid(String raw) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 헤더 이름별 모든 값, 요구 헤더 이름, raw UUID 문자열
- 출력: 원본 single header 값, canonical UUID 객체

#### 반드시 만족해야 할 조건

single header:

- 헤더가 없거나 값 목록이 비면 실패한다.
- 값은 정확히 하나여야 한다.
- 동일 문자열 두 개도 중복이므로 실패한다.
- 값을 trim·case-fold·구분자 결합하지 않고 원본 그대로 반환한다.
- 헤더 이름의 case-insensitive lookup은 HTTP adapter 책임으로 분리하거나 명시적으로 처리한다.

canonical UUID:

- null은 실패한다.
- 표준 UUID parser로 파싱되어야 한다.
- 파싱 결과의 표준 문자열이 raw와 정확히 같아야 한다.
- uppercase, 앞뒤 공백, braces, 축약 표현, 추가 문자를 허용하지 않는다.

#### 경계 조건

- 헤더 없음
- 빈 값 목록
- 한 값
- 같은 값 두 개
- 다른 값 두 개
- blank 한 값
- 표준 lowercase UUID
- uppercase UUID
- 앞뒤 공백 UUID
- 축약되어 parser가 받아들일 수 있는 표현
- braces가 붙은 UUID
- 잘못된 hex 또는 dash 위치

#### 실패 조건

- missing·duplicate header
- null·malformed·noncanonical UUID
- 헤더 adapter가 값을 잃어버린 경우

#### 필요한 제약

- 첫 번째 값만 취하고 나머지를 무시하지 않는다.
- canonicalization해서 새 identity로 바꾸지 않고 비정규 입력을 거부한다.
- 두 함수 모두 입력 길이에 대해 O(n), 추가 공간 O(1) 수준이어야 한다.

### 구현 후 자가 검증

- [ ] exactly-one 조건을 missing·one·duplicate 세 경우로 나눠 검증했다.
- [ ] duplicate가 같은 값이어도 거부한다.
- [ ] 반환 header 값을 trim하지 않는다.
- [ ] lowercase canonical UUID만 통과한다.
- [ ] uppercase·공백·braces·축약 표현을 거부한다.
- [ ] parser 예외를 안정적인 invalid-request 경계로 바꿀 수 있다.
- [ ] malformed identity 뒤 durable store를 호출하지 않는 테스트 시나리오를 설명할 수 있다.

### 구현 후 설명할 것

1. 관대한 normalization보다 strict rejection을 택한 이유.
2. duplicate header가 request smuggling·프록시 해석 차이와 만나는 위험.
3. UUID round-trip equality가 parser의 관대한 입력 범위를 좁히는 방식.
4. 형식 검증과 semantic fingerprint 검증의 책임 차이.
5. 이 검사를 controller마다 복사하지 않고 공통 boundary helper로 둔 이유.

### 원본 확인 위치

- Thread: **16 — 내부 지갑 HTTP 계약**
- 커밋: `feat(api): parse wallet request identities`
- 커밋: `test(api): verify request identity parsing`
- 파일: `src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java`
- 클래스·메서드: `WalletRequestHeaders.requireIdempotencyKey()`, `requireCanonicalDebitKey()`, `requireCanonicalDebitId()`
- 테스트: `WalletRequestHeadersTest`
- 관련 Thread: 6, 15, 17

---

<a id="w15"></a>
## W15 — [Thread 17 / `feat(errors): map retryable database outages`] 안정적인 오류 응답과 DB 실패 분류

### 면접 질문

같은 idempotent 요청의 첫 실행과 재생이 서로 다른 시점의 계정 상태를 보더라도 동일한 업무 오류 body를 반환하려면 무엇을 저장해야 합니까? 동시에 예외 메시지·SQL 진단·idempotency key를 응답에 노출하지 않으려면 오류 변환 계층을 어떻게 설계하겠습니까?

꼬리 질문:

- 잔액 부족 replay에서 현재 잔액을 다시 읽어 detail을 만들면 어떤 계약 위반이 생깁니까?
- malformed request, idempotency conflict, retryable DB outage, permanent DB error, unknown exception을 어떻게 구분합니까?
- deadlock이나 serialization failure를 500이 아니라 503으로 표현하는 이유는 무엇입니까?
- retryable 인프라 실패가 outcome을 커밋하지 않았다면 같은 key 재시도는 가능해야 합니까?
- `instance`에 query string을 넣지 않는 이유는 무엇입니까?
- catch-all handler가 exception message를 그대로 detail로 쓰면 어떤 정보가 샐 수 있습니까?

### 30초 모범 답변

확정 업무 거절은 status, code, title, detail과 필요한 balance·expectedCurrency facts를 operation outcome에 저장하고 replay 때 그대로 Problem Details로 변환해야 합니다. malformed 요청과 conflict도 고정된 안전 문구를 사용하고 key나 원래 예외 메시지를 반사하지 않습니다. 연결 장애, lock timeout, deadlock, serialization처럼 재시도 가능한 DB 실패는 503과 `Retry-After`로, 영구 DB 오류와 예상하지 못한 예외는 opaque 500으로 매핑합니다. 인프라 실패에 outcome이 없다면 같은 key를 다시 쓸 수 있어야 하고, `instance`는 query가 빠진 request path만 담습니다.

### 답변 핵심 키워드

`persist failure facts` · `immutable replay` · `Problem Details` · `fixed safe detail` · `no secret reflection` · `retryable 503` · `Retry-After` · `opaque 500`

### 백지 구현

#### 구현 목표

구조화된 failure 입력을 안정적인 `application/problem+json` 응답 모델로 변환한다. 원래 exception message·SQL·credential·operation key는 어떤 출력 필드에도 들어가지 않는다.

#### 면접용 축소 인터페이스

```java
enum DbFailureClass { RETRYABLE, PERMANENT }

record DurableFailureSnapshot(
    int httpStatus,
    String errorCode,
    String title,
    String detail,
    Long balance,
    Currency expectedCurrency) {}

sealed interface Failure {
  record DurableRejected(DurableFailureSnapshot snapshot) implements Failure {}
  record InvalidRequest() implements Failure {}
  record IdempotencyConflict() implements Failure {}
  record NotFound(String kind) implements Failure {}
  record Database(DbFailureClass classification) implements Failure {}
  record Unexpected() implements Failure {}
}

record ProblemResponse(
    URI type,
    String title,
    int status,
    String detail,
    URI instance,
    String errorCode,
    Map<String, Object> extensions,
    Map<String, String> headers) {}

final class WalletProblemMapper {
  static ProblemResponse map(Failure failure, String requestPath) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 이미 분류된 failure와 query 없는 request path
- 출력: 안정적인 type/title/status/detail/instance/errorCode, 선택 확장 필드, 응답 헤더

#### 반드시 만족해야 할 조건

- 모든 응답은 stable error code와 type URI를 가진다.
- `instance`는 request path만 포함하고 query·fragment를 포함하지 않는다.
- `DurableRejected`는 저장 snapshot의 status/title/detail/code를 그대로 사용한다.
- 저장 snapshot의 `balance`, `expectedCurrency`는 존재할 때만 확장 필드로 추가한다.
- `IdempotencyConflict`는 key와 내부 예외 메시지가 없는 고정 409 body다.
- malformed 입력은 고정 400 body다.
- retryable DB 실패는 고정 503 body와 `Retry-After: 1`을 가진다.
- permanent DB 실패와 unexpected 실패는 고정된 opaque 500 body다.
- 어떤 응답도 exception message, SQL state 원문, stack trace, API key, idempotency key, 요청 query를 반사하지 않는다.
- baseline problem shape는 type/title/status/detail/instance/errorCode로 안정적으로 유지한다.

#### 경계 조건

- 저장 balance만 있는 거절
- expectedCurrency만 있는 거절
- 선택 확장 필드가 둘 다 없는 거절
- conflict 예외 내부 메시지에 secret key가 포함된 상황
- invalid request 내부 메시지에 header/query 값이 포함된 상황
- retryable DB 분류
- permanent DB 분류
- 알 수 없는 runtime failure
- request path가 빈 값이거나 query를 포함해 잘못 전달된 경우

#### 실패 조건

- 저장 snapshot 필수 필드 누락
- 지원하지 않는 HTTP status
- 오류 vocabulary에 없는 code
- 안전한 path를 만들 수 없는 입력
- DB failure가 분류되지 않은 상태

#### 필요한 제약

- mapper가 live account를 조회해 durable failure detail을 재계산하지 않는다.
- catch-all 경로는 diagnostic 정보를 로깅 계층에 맡기고 client body에는 넣지 않는다.
- error vocabulary는 테스트로 고정할 수 있어야 한다.
- 매핑 시간·공간 복잡도는 O(1)이다.

### 구현 후 자가 검증

- [ ] 저장된 거절을 두 번 매핑하면 현재 외부 상태와 무관하게 body facts가 같다.
- [ ] conflict·invalid·unexpected 내부 메시지에 `secret`을 넣어도 응답에 나타나지 않는다.
- [ ] retryable DB만 503과 `Retry-After: 1`을 가진다.
- [ ] permanent DB와 unexpected가 안전한 500으로 수렴한다.
- [ ] instance에 query string이 없다.
- [ ] baseline 필드 수와 이름이 안정적이다.
- [ ] optional balance·expectedCurrency가 조건부로만 존재한다.
- [ ] mapper가 저장소나 현재 계정 상태를 조회하지 않는다.

### 구현 후 설명할 것

1. 업무 거절 facts를 저장하고 replay 때 재사용한 이유.
2. 재시도 가능 DB 오류를 분류하는 기준과 오분류 trade-off.
3. 503과 500이 caller의 재시도 정책에 주는 차이.
4. client-safe error와 내부 observability log를 분리한 이유.
5. RFC Problem Details 기본 필드에 stable `errorCode`를 추가한 이유.

### 원본 확인 위치

- Thread: **17 — 안정적인 오류 응답 계약**
- 커밋: `feat(errors): shape wallet problem details`
- 커밋: `feat(errors): map durable and retryable failures`
- 커밋: `feat(errors): map direct wallet failures`
- 커밋: `feat(errors): map malformed and unexpected requests`
- 커밋: `feat(errors): map retryable database outages`
- 커밋: `test(errors): verify wallet problem shapes`
- 커밋: `test(errors): map currency and access failures`
- 커밋: `test(errors): contain unexpected wallet failures`
- 커밋: `test(errors): return stable database outage responses`
- 파일: `src/main/java/com/sportsbook/wallet/web/WalletError.java`
- 파일: `src/main/java/com/sportsbook/wallet/web/WalletProblems.java`
- 파일: `src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/PostgresFailureTranslator.java`
- 클래스·메서드: `WalletProblems.from()`, `WalletExceptionHandler`, `PostgresFailureTranslator`
- 테스트: `WalletErrorTest`, `WalletProblemsTest`, `WalletExceptionHandlerReplayTest`, `WalletExceptionHandlerAccessTest`, `WalletExceptionHandlerMalformedMvcTest`, `WalletExceptionHandlerTerminalTest`
- 관련 Thread: 6, 15, 16
