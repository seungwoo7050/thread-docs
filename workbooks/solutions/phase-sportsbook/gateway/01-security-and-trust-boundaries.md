# 보안과 신뢰 경계 면접 워크북

HTTP와 STOMP 진입점에서 외부 입력을 내부 신뢰로 바꾸는 지점을 다룬다. 프레임워크 설정 암기보다 정규형, 허용목록, strip-then-rebuild, fail-fast 같은 보안 불변조건을 설명하고 다시 구현하는 데 초점을 둔다.

## SEC-01 · [Thread 3 / `feat(security): verify RS256 user tokens`] RSA 공개키 로딩과 JWT 검증 정책

**우선순위:** S

### 면접 질문

- 왜 RSA 공개키를 사용한다고 해서 자동으로 RS256만 허용되는 것은 아니며, 검증 알고리즘을 명시적으로 고정해야 합니까?
- X.509 `PUBLIC KEY` PEM, 2048비트 이상 RSA 키, 필수 `exp`, 0초 clock skew를 각각 어느 경계에서 검증하는 것이 좋습니까?
- 꼬리 질문: 잘못된 키 설정을 요청 시점이 아니라 시작 시점에 실패시키는 이유는 무엇입니까?
- 꼬리 질문: 키 원문이나 토큰을 예외 메시지에 포함하지 않아야 하는 이유는 무엇입니까?

### 30초 모범 답변

공개키 형식과 키 강도는 애플리케이션 시작 시 검증하고, 토큰 검증기는 허용 알고리즘을 RS256으로 고정해야 합니다. 서명만 맞아도 수명 계약이 없으면 재사용 위험이 생기므로 `exp`를 필수로 두고 현재 시각보다 미래인지 확인합니다. 이 프로젝트는 clock skew를 두지 않아 경계가 단순하지만, 노드 시계가 정확히 동기화되어야 한다는 운영 부담을 선택했습니다. 설정 오류는 fail-fast로 막고, 오류에는 키나 토큰 원문을 남기지 않습니다.

### 답변 핵심 키워드

`algorithm allowlist`, `X.509 SubjectPublicKeyInfo`, `RSA ≥ 2048`, `required exp`, `zero clock skew`, `fail-fast`, `secret-safe error`

### 백지 구현

**구현 목표**

환경 문자열에서 RSA 공개키를 안전하게 읽고, 토큰 메타데이터가 프로젝트의 최소 검증 계약을 만족하는지 판정하는 작은 보안 컴포넌트를 작성한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
final class RsaPublicKeyLoader {
  RSAPublicKey load(String configuredPem) {
    throw new UnsupportedOperationException("직접 구현");
  }
}

record TokenMetadata(String algorithm, Instant expiresAt) {}

final class JwtVerificationPolicy {
  ValidationResult validate(TokenMetadata token, RSAPublicKey key, Instant now) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- `configuredPem`: 실제 개행 또는 이스케이프된 `\n`을 포함할 수 있는 PEM 문자열
- `TokenMetadata`: 헤더 알고리즘과 만료 시각
- 출력: 유효한 `RSAPublicKey` 또는 성공/실패를 나타내는 `ValidationResult`

**반드시 만족해야 할 조건**

- `-----BEGIN PUBLIC KEY-----` / `-----END PUBLIC KEY-----` 형식만 허용한다.
- RSA 공개키가 아니거나 modulus가 2048비트보다 작으면 거부한다.
- 토큰 알고리즘은 정확히 RS256이어야 한다.
- `exp`가 없거나 `exp <= now`이면 거부한다.
- 실패 메시지에 PEM 본문이나 토큰 값이 노출되지 않아야 한다.

**경계 조건**

- 앞뒤 공백, 실제 개행, 문자열 안의 `\n` 표현
- 빈 값, 헤더만 있는 PEM, 손상된 Base64, EC 공개키
- 만료 시각이 현재 시각과 정확히 같은 경우
- 4096비트 RSA처럼 최소값보다 큰 정상 키

**실패 조건**

- 설정 키가 없거나 형식·알고리즘·강도가 잘못되면 시작 불가 수준의 예외
- 토큰 메타데이터가 계약을 위반하면 인증 실패 결과
- 암호화 라이브러리 내부 예외를 원문 비밀정보 없이 경계 예외로 변환

**필요한 제약**

- 네트워크에서 키를 조회하거나 캐시를 구현하지 않는다.
- issuer·audience·폐기 목록은 이 축소 문제의 범위가 아니다.
- 정규식만으로 DER 키 종류와 비트 길이를 판정하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 RS256 메타데이터와 2048비트 이상 X.509 RSA 공개키가 통과한다.
- [ ] 빈 PEM, 손상된 Base64, PKCS#1 헤더, 비-RSA 키, 2048비트 미만 키가 거부된다.
- [ ] unsigned·HS256·다른 RSA 알고리즘 표기가 거부된다.
- [ ] `exp` 누락, 과거, 현재와 동일한 시각이 거부된다.
- [ ] 예외 문자열에 입력 PEM과 토큰 값이 포함되지 않는다.
- [ ] 키 파싱은 요청마다 반복되지 않고 시작 경계에서 수행할 수 있는 형태다.

### 구현 후 설명할 것

- 알고리즘 고정이 알고리즘 혼동 공격과 잘못된 설정을 어떻게 줄이는지
- 키 형식·강도 검증과 토큰 수명 검증을 서로 다른 경계에 둔 이유
- 0초 clock skew가 주는 단순성과 운영상 시계 동기화 부담
- fail-fast가 가용성보다 안전한 설정을 우선하는 선택인 이유

### 원본 확인 위치

- Thread 3
- 커밋: `feat(security): verify RS256 user tokens`
- 커밋: `test(security): verify token key and lifetime checks`
- 파일: `src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java`
- 파일: `src/main/java/com/sportsbook/gateway/security/JwtSecurityProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/security/RsaPublicKeyLoader.java`
- 테스트: `src/test/java/com/sportsbook/gateway/security/JwtVerificationTest.java`
- 관련 Thread: 8, 9

## SEC-02 · [Thread 3 / `feat(security): require canonical user claims`] 정규형 사용자 신원과 bounded roles 계약

**우선순위:** S

### 면접 질문

- `UUID.fromString`이 성공하는 것만으로 사용자 ID의 정규형을 보장할 수 없는 이유는 무엇입니까?
- `roles`를 문자열 하나가 아니라 크기가 제한된 중복 없는 문자열 배열로 제한한 이유를 설명해 보세요.
- 인증 객체의 이름과 권한을 어떤 검증된 claim에서 만들며, 검증과 변환 순서를 어떻게 보장해야 합니까?
- 꼬리 질문: 대소문자·중복·과도한 role 개수를 허용하면 어떤 경계 문제가 생깁니까?

### 30초 모범 답변

사용자 ID는 파싱 성공뿐 아니라 `UUID.fromString(value).toString().equals(value)`까지 확인해 하나의 정규 표현만 허용합니다. `roles`는 선택 사항이지만 존재하면 최대 16개, 중복 없음, 제한된 대문자 패턴을 만족해야 하므로 권한 문자열의 모호성과 비정상적인 메모리·헤더 팽창을 막습니다. 검증이 끝난 claim만 인증 이름과 `ROLE_` 권한으로 변환해야 하며, 원본 요청 값이나 검증되지 않은 claim을 신원으로 사용하지 않습니다.

### 답변 핵심 키워드

`canonical UUID`, `parse then round-trip`, `bounded collection`, `unique roles`, `role grammar`, `validated principal`, `authority mapping`

### 백지 구현

**구현 목표**

JWT claim 맵에서 프로젝트가 신뢰할 수 있는 사용자 신원과 권한 집합을 만들되, 비정규형·중복·과도한 입력을 모두 거부한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record CanonicalIdentity(String subject, List<String> roles) {}

final class IdentityClaimsParser {
  CanonicalIdentity parse(Map<String, Object> claims) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: `sub`, 선택적 `roles`를 포함하는 claim 맵
- 출력: 정규형 subject와 불변 roles 목록을 가진 `CanonicalIdentity`
- 계약 위반 시: 신원 파싱 예외 또는 실패 결과

**반드시 만족해야 할 조건**

- `sub`는 null·blank가 아니며 소문자 정규형 UUID 문자열이어야 한다.
- `roles`가 없으면 빈 목록으로 처리한다.
- `roles`가 있으면 배열/목록이며 최대 16개, 중복 없음이어야 한다.
- 각 role은 `[A-Z][A-Z0-9_]{0,31}`을 만족해야 한다.
- 반환 목록은 호출자가 변경해 내부 계약을 깨뜨릴 수 없어야 한다.

**경계 조건**

- 대문자 UUID, 짧은 UUID처럼 보이는 문자열, 앞뒤 공백
- `roles`가 문자열 하나이거나 숫자 원소를 포함하는 경우
- 정확히 16개와 17개 role
- 동일 role 중복, 빈 문자열, 32자와 33자 role

**실패 조건**

- 유효하지 않은 subject와 roles를 구분 가능한 계약 오류로 거부한다.
- 일부 role만 정상인 목록도 전체를 거부한다.
- 검증 실패 후 부분적으로 만들어진 인증 객체를 반환하지 않는다.

**필요한 제약**

- role의 비즈니스 의미나 실제 권한 계층은 판단하지 않는다.
- issuer·audience 검증은 이 문제의 범위가 아니다.
- 입력 순서는 보존하되 중복 제거로 조용히 보정하지 않는다.

### 구현 후 자가 검증

- [ ] 정규형 UUID와 roles 미지정 입력이 통과한다.
- [ ] 정규형 UUID와 정상 role 목록의 순서가 보존된다.
- [ ] 대문자·공백 포함·비UUID subject가 모두 거부된다.
- [ ] 문자열형 roles, 비문자 원소, 중복, 17개, 잘못된 패턴이 거부된다.
- [ ] 반환된 roles 목록을 외부에서 변경할 수 없다.
- [ ] 실패 입력이 인증 이름이나 권한 객체로 변환되지 않는다.
- [ ] 검증 복잡도가 role 수와 문자열 길이에 선형으로 제한된다.

### 구현 후 설명할 것

- 정규형을 강제하면 캐시 키·로그·사용자 목적지에서 동일 신원의 다중 표현을 막는다는 점
- 중복을 조용히 제거하지 않고 토큰 자체를 거부한 이유
- 개수와 길이 제한이 보안뿐 아니라 자원 사용 상한을 만드는 방식
- 검증과 인증 객체 변환을 분리하거나 결합할 때의 장단점

### 원본 확인 위치

- Thread 3
- 커밋: `feat(security): require canonical user claims`
- 커밋: `test(security): reject incomplete user identities`
- 파일: `src/main/java/com/sportsbook/gateway/security/GatewayClaimsValidator.java`
- 파일: `src/main/java/com/sportsbook/gateway/security/GatewayJwtAuthenticationConverter.java`
- 테스트: `src/test/java/com/sportsbook/gateway/security/GatewayClaimsValidatorTest.java`
- 관련 Thread: 5, 6, 8, 9, 14

## SEC-03 · [Thread 4 / `feat(security): protect HTTP access boundaries`] HTTP 메서드·경로 허용목록과 deny-by-default

**우선순위:** S

### 면접 질문

- 게이트웨이에서 URL prefix 하나를 통째로 허용하지 않고 메서드와 경로 모양을 함께 허용목록으로 관리한 이유는 무엇입니까?
- 인증 실패 401과 인증은 되었지만 허용되지 않은 요청 403을 어떻게 구분합니까?
- 오류 dispatch를 다시 인증하지 않도록 허용한 이유와, 이것이 일반 요청 허용목록을 우회하지 않게 하는 조건은 무엇입니까?
- 꼬리 질문: `/api/v1/bets/*` 같은 패턴에서 중첩 경로가 실수로 열리지 않았음을 어떻게 테스트합니까?

### 30초 모범 답변

게이트웨이는 외부 공격 표면이므로 공개 prefix가 아니라 정확한 메서드와 경로 모양만 열고 마지막 규칙은 deny-all로 둡니다. 인증이 필요한 정상 경로에 자격 증명이 없으면 401, 인증된 요청이 허용목록 밖이면 403으로 구분합니다. 오류 dispatch는 이미 발생한 공개 경로의 프록시 실패가 재인증 오류로 덮이지 않도록 통과시키되, 최초 요청의 라우팅 허용 여부와는 별도 dispatch 타입에서만 적용합니다. 경로 깊이와 잘못된 메서드는 다운스트림 호출이 전혀 없다는 것까지 검증해야 합니다.

### 답변 핵심 키워드

`method+path allowlist`, `deny-by-default`, `401 vs 403`, `stateless`, `ERROR dispatch`, `path-shape boundary`, `no downstream call`

### 백지 구현

**구현 목표**

인증 상태, HTTP 메서드, 경로, dispatch 타입을 받아 PUBLIC·AUTHENTICATED·DENY 중 하나를 결정하는 순수 접근 정책을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
enum AccessRequirement { PUBLIC, AUTHENTICATED, DENY }

record AccessRequest(
    String method,
    String rawPath,
    boolean authenticated,
    String dispatcherType) {}

final class GatewayAccessPolicy {
  AccessRequirement classify(AccessRequest request) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 메서드, raw path, 인증 여부, dispatch 타입
- 출력: 경로의 접근 요구 수준
- 호출 측은 `AUTHENTICATED`인데 미인증이면 401, `DENY`이면 403으로 변환

**반드시 만족해야 할 조건**

- 프로젝트에서 선언된 공개 GET, 비공개 betting/wallet 요청, WebSocket handshake, 운영 endpoint만 허용한다.
- 경로 segment 수가 추가되거나 부족하면 기존 허용 경로와 접두사가 같아도 거부한다.
- 허용되지 않은 메서드는 동일 경로에서도 거부한다.
- 일반 요청의 기본값은 DENY다.
- ERROR dispatch 예외는 일반 REQUEST dispatch 규칙과 혼합하지 않는다.

**경계 조건**

- trailing slash, 중복 slash, URL 인코딩 segment, query string이 붙은 요청
- `/api/v1/bets/internal/hidden`처럼 허용 prefix 아래의 중첩 경로
- 공개 경로에 인증이 함께 들어오는 경우
- 운영 endpoint의 하위 경로와 비슷한 애플리케이션 경로

**실패 조건**

- 정책이 모르는 메서드·경로는 예외가 아니라 명시적 DENY로 처리한다.
- 미인증과 인가 거부의 응답 계약이 뒤바뀌지 않아야 한다.
- 거부된 요청에서는 라우터나 다운스트림 클라이언트가 실행되지 않아야 한다.

**필요한 제약**

- 프레임워크 matcher API를 외우는 문제로 만들지 말고 순수 정책부터 구현한다.
- 경로를 무조건 normalize해 원래 경계가 넓어지지 않게 한다.
- 비즈니스 리소스 ID의 형식 검증은 담당 다운스트림과 계약에 따라 분리한다.

### 구현 후 자가 검증

- [ ] 각 공개 경로는 허용된 GET에서만 PUBLIC로 분류된다.
- [ ] betting·wallet 경로는 정확한 메서드·segment 수에서만 AUTHENTICATED로 분류된다.
- [ ] 미인증 AUTHENTICATED 요청은 401, 인증된 DENY 요청은 403으로 매핑된다.
- [ ] 중첩·부족·trailing slash·잘못된 메서드가 모두 DENY다.
- [ ] ERROR dispatch의 예외가 일반 요청으로 재사용되지 않는다.
- [ ] 거부 경로가 다운스트림 호출을 만들지 않는 테스트가 있다.
- [ ] 규칙 추가 시 기본 DENY가 유지된다.

### 구현 후 설명할 것

- prefix 허용보다 메서드·경로 모양 허용이 공격 표면을 줄이는 이유
- 보안 필터와 실제 라우터 양쪽에 경계를 두었을 때의 중복과 방어 심층성
- ERROR dispatch 예외가 필요한 실제 실패 응답 흐름
- 정규화·인코딩을 어느 계층에서 다룰지에 대한 판단

### 원본 확인 위치

- Thread 4
- 커밋: `feat(security): protect HTTP access boundaries`
- 커밋: `test(security): verify public and private paths`
- 파일: `src/main/java/com/sportsbook/gateway/security/SecurityConfig.java`
- 테스트: `src/test/java/com/sportsbook/gateway/security/HttpAccessBoundaryTest.java`
- 관련 Thread: 2, 5, 7, 15

## SEC-04 · [Thread 4, 5 / `feat(routing): route betting requests`; `feat(routing): route authenticated wallet reads`] 정확한 프록시 재작성과 subject 강제 주입

**우선순위:** S

### 면접 질문

- `GET /api/v1/bets`의 caller-supplied `userId`를 보존하지 않고 검증된 JWT subject로 덮어쓴 이유는 무엇입니까?
- collection 경로, 개별 bet 경로, wallet balance 경로를 각각 어떻게 재작성해야 과도한 라우팅을 피할 수 있습니까?
- 다운스트림 base URI에 user-info, query, fragment, non-root path를 허용하지 않은 이유를 설명해 보세요.
- 꼬리 질문: path segment를 문자열 치환할 때 인코딩·prefix 충돌을 어떻게 방지합니까?

### 30초 모범 답변

게이트웨이가 소유권 경계를 담당하므로 사용자 범위 조회의 `userId`는 클라이언트 입력이 아니라 검증된 subject에서 재구성해야 합니다. 라우팅은 허용된 경로별로 명시적인 forward plan을 만들고, collection 조회에서만 `userId`를 set해 기존 값을 덮어씁니다. base URI는 HTTP(S) absolute host와 root path만 허용해 설정 값이 숨은 경로·자격 증명·query를 주입하지 못하게 합니다. 일반 문자열 prefix 라우팅보다 조금 반복되지만, 외부 API와 내부 API의 대응 관계가 감사 가능해집니다.

### 답변 핵심 키워드

`subject overwrite`, `exact rewrite`, `forward plan`, `safe base URI`, `opaque resource ID`, `query preservation`, `boundary ownership`

### 백지 구현

**구현 목표**

허용된 외부 요청 하나를 내부 다운스트림 요청 계획으로 변환한다. 사용자가 제어한 소유권 query는 반드시 검증된 subject로 대체한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record ProxyRequest(
    String method,
    URI uri,
    Map<String, List<String>> query,
    String verifiedSubject) {}

record ForwardPlan(
    URI downstreamUri,
    Map<String, List<String>> query) {}

final class ExactRouteRewriter {
  ForwardPlan rewrite(URI baseUri, ProxyRequest request) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 검증된 base URI, 외부 메서드·URI·query, 선택적 verified subject
- 출력: 내부 path와 최종 query를 가진 `ForwardPlan`
- 허용되지 않은 요청은 계획을 만들지 않고 거부

**반드시 만족해야 할 조건**

- base URI는 `http` 또는 `https`, absolute, host 존재, root path, user-info/query/fragment 없음이어야 한다.
- bet collection GET은 내부 collection path로 바꾸고 `userId`를 subject 하나로 강제한다.
- 개별 bet GET과 bet POST는 정의된 내부 prefix로만 재작성한다.
- wallet balance는 subject를 내부 계정 path에 사용하고 클라이언트 path 값을 사용하지 않는다.
- 다른 query는 계약상 허용된 범위에서 보존한다.

**경계 조건**

- caller가 `userId`를 여러 번 보낸 경우
- subject가 없는데 비공개 경로 재작성을 요청한 경우
- base URI의 trailing `/`, non-root path, query, fragment, user-info
- 퍼센트 인코딩된 path segment와 단순 prefix 유사 경로

**실패 조건**

- 허용되지 않은 메서드·경로에는 forward plan을 반환하지 않는다.
- subject가 필요한 경로에서 subject가 없으면 보안 실패로 거부한다.
- 안전하지 않은 base URI는 시작 시점 설정 오류로 거부할 수 있어야 한다.

**필요한 제약**

- 네트워크 호출은 구현하지 않고 순수 재작성만 작성한다.
- `betId` 내용 자체는 opaque segment로 전달하며 형식 검증을 새로 추측하지 않는다.
- 경로 문자열을 decode 후 재인코딩해 의미를 바꾸지 않는다.

### 구현 후 자가 검증

- [ ] bet POST, collection GET, 개별 GET, wallet GET의 정확한 재작성 결과가 맞다.
- [ ] 기존 `userId`가 하나 또는 여러 개여도 subject 하나로 대체된다.
- [ ] 다른 허용 query는 누락되지 않는다.
- [ ] subject가 필요한 요청에 subject가 없으면 실패한다.
- [ ] 비슷하지만 허용되지 않은 중첩 path는 재작성되지 않는다.
- [ ] ftp, relative URI, user-info, non-root path, query, fragment가 있는 base URI가 거부된다.
- [ ] 입력 query 컬렉션을 부수적으로 변경하지 않는다.

### 구현 후 설명할 것

- 소유권 필터를 게이트웨이가 강제하고 downstream도 재검증해야 하는 이유
- 명시적 route table과 범용 prefix rewrite의 trade-off
- base URI 검증이 SSRF·설정 오염 위험을 줄이는 방식
- opaque ID 검증 책임을 downstream에 남긴 이유

### 원본 확인 위치

- Thread 4
- 커밋: `feat(routing): route betting requests`
- 커밋: `test(routing): verify betting route contracts`
- 커밋: `feat(routing): expose public event and odds reads`
- 커밋: `test(routing): verify public read route boundaries`
- Thread 5
- 커밋: `feat(routing): route authenticated wallet reads`
- 커밋: `test(routing): verify wallet authentication boundary`
- 파일: `src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/WalletDownstreamProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/WalletRequestAuthentication.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/WalletRoutes.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/OddsFeedDownstreamProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/PublicReadRoutes.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java`
- 관련 Thread: 3, 5, 7, 15

## SEC-05 · [Thread 5 / `feat(routing): establish downstream identity boundary`; `feat(routing): require distinct downstream credentials`] 신뢰 헤더 제거 후 검증된 신원·자격 증명 재구성

**우선순위:** S

### 면접 질문

- 클라이언트가 보낸 `X-User-Id` 같은 헤더를 인증 전에 숨기고, 프록시 직전에도 다시 제거한 이유는 무엇입니까?
- HTTP 헤더 이름의 대소문자 비구분과 중복 값까지 고려하면 단순 `getHeader` 검사만으로 부족한 이유는 무엇입니까?
- 외부 bearer token을 downstream에 전달하지 않고 `X-User-Id`, `X-User-Roles`, 내부 서비스명과 서비스별 API key를 재구성한 설계 의도를 설명해 보세요.
- 꼬리 질문: betting과 wallet API key가 같으면 시작을 실패시키는 이유는 무엇입니까?

### 30초 모범 답변

신뢰 헤더는 caller가 쓰는 순간 내부 신원 경계가 무너지므로 가장 바깥 servlet 경계에서 모든 대소문자와 중복 값을 숨깁니다. 라우팅 직전에도 `Authorization`과 신뢰 헤더를 다시 제거한 뒤, 검증된 JWT와 해당 downstream 전용 자격 증명으로 필요한 값만 재구성합니다. 이중 제거는 다른 필터나 라우트 변경으로 헤더가 다시 섞이는 것을 막는 방어 심층성입니다. betting과 wallet 키를 분리하고 동일 값이면 시작을 거부해 한 서비스의 자격 증명 유출이 다른 서비스 권한으로 확대되는 것을 제한합니다.

### 답변 핵심 키워드

`strip then rebuild`, `case-insensitive headers`, `duplicate values`, `defense in depth`, `credential scoping`, `bearer termination`, `fail-fast secret separation`

### 백지 구현

**구현 목표**

인바운드 헤더 목록에서 caller-controlled trust 헤더를 완전히 제거하고, 검증된 신원 및 route별 내부 자격 증명만으로 outbound 헤더를 새로 만든다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record Header(String name, String value) {}

record VerifiedIdentity(String subject, List<String> roles) {}

enum Downstream { BETTING, WALLET, ODDS }

final class TrustBoundaryHeaders {
  List<Header> sanitizeInbound(List<Header> incoming) {
    throw new UnsupportedOperationException("직접 구현");
  }

  List<Header> buildOutbound(
      List<Header> sanitized,
      VerifiedIdentity identity,
      Downstream downstream,
      Map<Downstream, String> apiKeys) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 중복과 서로 다른 대소문자를 허용하는 raw header 목록
- 입력: 검증된 identity, 대상 downstream, downstream별 API key
- 출력: caller trust 값과 bearer token이 없는 최종 outbound header 목록

**반드시 만족해야 할 조건**

- `X-User-Id`, `X-User-Roles`, `X-Internal-Service`, `X-Internal-Api-Key`는 대소문자와 개수에 관계없이 제거한다.
- 인바운드 단계에서는 인증에 필요한 `Authorization`은 보존할 수 있지만 outbound 단계에서는 제거한다.
- 비공개 betting/wallet 요청에만 검증된 subject와 route 전용 내부 credential을 추가한다.
- public odds 요청에는 사용자 신원과 내부 credential을 추가하지 않는다.
- roles가 비어 있으면 빈 roles 헤더를 억지로 만들지 않는다.

**경계 조건**

- 같은 헤더의 여러 값과 혼합 대소문자
- 신뢰 헤더 이름과 비슷하지만 다른 일반 헤더
- identity가 없는 public 요청과 잘못된 private 요청
- betting·wallet 키 누락, 짧은 키, 두 키가 동일한 설정

**실패 조건**

- private route에서 검증된 identity나 해당 credential이 없으면 요청을 만들지 않는다.
- 설정된 키의 원문을 예외에 포함하지 않는다.
- sanitize 후에도 forbidden header가 하나라도 남으면 계약 실패로 취급한다.

**필요한 제약**

- 원본 header 목록을 직접 변경하지 않는다.
- 일반 헤더의 상대적 순서와 중복은 필요 이상으로 손상하지 않는다.
- API key 값의 생성·회전은 구현 범위가 아니며 유효성·분리만 확인한다.

### 구현 후 자가 검증

- [ ] 신뢰 헤더가 대소문자·중복 형태와 무관하게 완전히 사라진다.
- [ ] 인바운드 sanitize에서 bearer는 유지되고 outbound build에서는 사라진다.
- [ ] betting과 wallet에 서로 다른 키가 들어가며 서비스명이 `gateway`로 재구성된다.
- [ ] public odds 요청에는 identity·internal credential이 없다.
- [ ] caller가 넣은 값보다 검증된 subject·roles가 우선한다.
- [ ] 키 누락·너무 짧음·동일 값이 안전한 오류로 거부된다.
- [ ] 입력 목록과 identity 객체가 변경되지 않는다.

### 구현 후 설명할 것

- 인증 전 제거와 프록시 전 제거를 모두 둔 방어 심층성
- bearer token을 gateway에서 종결하고 내부 계약으로 바꾼 이유
- 서비스별 credential 분리가 blast radius를 줄이는 방식
- 헤더 기반 내부 신원 계약의 장점과 mTLS·서명 기반 대안

### 원본 확인 위치

- Thread 5
- 커밋: `test(security): reject spoofed trust headers`
- 커밋: `feat(routing): establish downstream identity boundary`
- 커밋: `test(routing): verify proxied credential isolation`
- 커밋: `feat(routing): validate wallet service credentials`
- 커밋: `feat(routing): route authenticated wallet reads`
- 커밋: `test(routing): verify wallet authentication boundary`
- 커밋: `feat(routing): require betting credentials`
- 커밋: `test(routing): reject invalid betting credentials`
- 커밋: `feat(routing): authenticate betting requests`
- 커밋: `test(routing): isolate betting credentials`
- 커밋: `feat(routing): require distinct downstream credentials`
- 커밋: `test(routing): reject reused downstream credentials`
- 파일: `src/main/java/com/sportsbook/gateway/security/GatewayHeaders.java`
- 파일: `src/main/java/com/sportsbook/gateway/security/TrustedHeaderFilter.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/DownstreamRequestSanitizer.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/IdentityForwarding.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/WalletDownstreamProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/WalletRequestAuthentication.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/BettingRequestAuthentication.java`
- 파일: `src/main/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicy.java`
- 테스트: `src/test/java/com/sportsbook/gateway/security/TrustedHeaderFilterTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/DownstreamIdentityBoundaryTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/WalletRoutesStartupTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/BettingDownstreamPropertiesTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicyTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java`
- 관련 Thread: 3, 4, 15, 18

## SEC-06 · [Thread 8 / `feat(websocket): authenticate STOMP sessions`; `feat(websocket): restrict client STOMP commands`] STOMP CONNECT 인증과 명령·구독 허용목록

**우선순위:** S

### 면접 질문

- WebSocket HTTP handshake가 열려 있어도 STOMP `CONNECT` 프레임에서 별도 인증이 필요한 이유는 무엇입니까?
- Authorization native header가 없을 때 anonymous session을 허용하면서도, 중복·빈 bearer·잘못된 scheme은 왜 거부해야 합니까?
- 왜 `SEND`, 트랜잭션, ACK/NACK 등을 모두 막고 `SUBSCRIBE`, `UNSUBSCRIBE`, `DISCONNECT`만 필요한 범위에서 허용했습니까?
- `/topic/odds/{eventId}`와 `/user/queue/bets`의 인가 규칙 차이를 설명해 보세요.

### 30초 모범 답변

handshake 허용은 transport 연결만 열 뿐 사용자 principal을 확정하지 않으므로 STOMP `CONNECT`에서 HTTP와 같은 JWT decoder를 사용해 인증합니다. 헤더가 없으면 공개 odds용 anonymous 연결이지만, 둘 이상이거나 형식이 잘못되면 해석 모호성을 없애기 위해 거부합니다. 이 gateway는 클라이언트 발행 기능이 없으므로 명령도 최소 허용목록으로 제한하고, 공개 odds 목적지는 정규형 event UUID 한 segment만, bet queue는 검증된 JWT principal만 허용합니다. 연결 시 기존 principal을 그대로 신뢰하지 않고 인증 결과로 다시 설정하는 것도 중요합니다.

### 답변 핵심 키워드

`protocol-layer auth`, `native header cardinality`, `anonymous connect`, `shared JwtDecoder`, `command allowlist`, `canonical destination`, `user destination`

### 백지 구현

**구현 목표**

STOMP frame 하나를 받아 CONNECT 인증과 이후 명령·목적지 인가를 수행하는 순수 정책을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record StompFrame(
    String command,
    List<String> authorizationHeaders,
    String destination,
    Principal principal) {}

interface TokenDecoder {
  VerifiedIdentity decode(String token);
}

record StompDecision(boolean allowed, Principal resultingPrincipal) {}

final class StompBoundaryPolicy {
  StompDecision evaluate(StompFrame frame, TokenDecoder decoder) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: STOMP command, native Authorization 값 목록, destination, 현재 principal
- 입력: 검증된 신원을 반환하거나 실패하는 `TokenDecoder`
- 출력: 허용 여부와 CONNECT 후 설정할 principal

**반드시 만족해야 할 조건**

- `CONNECT`와 `STOMP`는 같은 연결 인증 명령으로 처리한다.
- Authorization이 없으면 anonymous principal로 초기화한다.
- Authorization이 정확히 하나이고 `Bearer ` 뒤에 비어 있지 않은 토큰이 있을 때만 decoder를 호출한다.
- `SUBSCRIBE`는 canonical `/topic/odds/{uuid}` 또는 JWT 인증 principal의 `/user/queue/bets`만 허용한다.
- `UNSUBSCRIBE`, `DISCONNECT`는 허용하고 그 밖의 client command는 거부한다.

**경계 조건**

- CONNECT frame에 이미 principal이 들어 있는 경우
- Authorization 0개, 1개, 2개, `Bearer `만 있는 값, 다른 scheme
- 목적지 없음, 대문자 UUID, 추가 path segment, wildcard
- 일반 `Principal`과 JWT 기반 principal의 구분

**실패 조건**

- decoder 오류·만료 토큰은 연결 거부로 변환한다.
- 지원하지 않는 command나 목적지는 message delivery 실패로 거부한다.
- 실패 시 기존 principal을 인증된 상태로 남기지 않는다.

**필요한 제약**

- 토큰의 서명·claim 검증은 decoder에 위임하고 중복 구현하지 않는다.
- 정규형 UUID 검사는 parse 후 round-trip 방식으로 한다.
- 프레임 payload 처리나 실제 WebSocket close는 이 축소 문제 범위가 아니다.

### 구현 후 자가 검증

- [ ] anonymous CONNECT는 decoder 호출 없이 허용되고 principal이 비인증 상태다.
- [ ] 정상 bearer CONNECT는 decoder 결과를 principal로 설정한다.
- [ ] 중복·빈 bearer·다른 scheme·decoder 실패가 거부된다.
- [ ] 공개 odds destination의 정확한 UUID 한 segment만 허용된다.
- [ ] bet user queue는 JWT 인증 principal에서만 허용된다.
- [ ] 모든 SEND·ACK/NACK·transaction 계열 명령이 거부된다.
- [ ] UNSUBSCRIBE와 DISCONNECT는 불필요한 인증 재검사 없이 처리된다.

### 구현 후 설명할 것

- HTTP handshake 보안과 STOMP protocol 보안이 서로 다른 경계인 이유
- 헤더 cardinality를 엄격히 제한한 이유
- deny-by-default command 정책이 공격 표면과 서버 기능을 맞추는 방식
- 단순 broker의 user destination이 principal 이름에 의존하는 점

### 원본 확인 위치

- Thread 8
- 커밋: `feat(websocket): authenticate STOMP sessions`
- 커밋: `test(websocket): verify CONNECT authentication`
- 커밋: `feat(websocket): restrict client STOMP commands`
- 커밋: `test(websocket): verify destination permissions`
- 파일: `src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java`
- 파일: `src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java`
- 테스트: `src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java`
- 관련 Thread: 3, 9, 13, 14, 16
