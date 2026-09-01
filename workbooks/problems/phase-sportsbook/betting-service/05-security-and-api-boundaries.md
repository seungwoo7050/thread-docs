# 게이트웨이 신뢰 경계와 안정적인 HTTP 계약

Thread 13과 14는 같은 보안 역량을 서로 다른 층에서 반복한다. 이 문서는 게이트웨이에서 온 요청만 허용하는 default-deny 경계와, 그 경계가 검증한 actor를 요청 본문과 분리해 도메인 command로 전달하는 HTTP 계약을 하나의 면접 문제로 통합한다.

<a id="t13-t14-trust-boundary"></a>
## [Thread 13·14 / 대표 커밋 `feat(api): enforce exact gateway ingress`] 인증된 전달자와 신뢰된 actor를 분리하기

### 면접 질문

내부 betting API가 `X-Internal-Service`, 내부 API key, `X-User-Id`를 받을 때, 왜 단순히 API key가 맞는지만 검사하지 않고 헤더 개수, caller identity, method·path allowlist, canonical actor ID까지 정확히 검증했습니까? 또 placement body에서 `userId`를 받지 않고 게이트웨이가 전달한 actor만 command에 넣은 이유는 무엇입니까?

꼬리 질문:

- missing credential과 올바른 credential을 가진 금지 route를 각각 401과 403으로 구분한 이유는 무엇입니까?
- 동일 이름의 헤더가 두 번 온 경우 첫 값만 읽으면 어떤 header smuggling·parser disagreement 위험이 있습니까?
- `/internal/`과 `/api/` 경로를 넓게 filter하고 허용 route만 열어 둔 이유는 무엇입니까?
- item route의 UUID를 lowercase canonical 형식으로 제한한 이유는 무엇입니까?
- API key 비교에 일반 문자열 equality 대신 constant-time 비교를 사용한 이유는 무엇입니까?
- gateway key와 risk/wallet outbound key가 같으면 어떤 lateral movement 위험이 생깁니까?
- body에 userId와 header actor가 둘 다 있으면 어느 쪽을 신뢰해야 합니까?
- collection POST 응답에서 201과 202를 상태에 따라 나눈 이유는 무엇입니까?
- public `Location`과 internal route를 분리한 이유는 무엇입니까?

### 30초 모범 답변

서비스 credential은 "누가 전달했는가"만 인증하고, actor header는 "누구를 대신하는가"를 나타냅니다. 두 값을 분리하고 각 헤더가 정확히 하나인지 검사해 중복 해석을 막습니다. 인증 실패는 401, 인증된 caller의 금지 route는 403으로 처리하며, 모든 business path를 기본 거부하고 명시된 method·path만 허용합니다. 사용자 ID는 body에서 받지 않고 신뢰 경계가 검증한 canonical header에서만 command에 주입해 actor spoofing을 막습니다. ingress key는 outbound key와 분리하고 secret은 로그나 `toString`에 노출하지 않습니다.

### 답변 핵심 키워드

`trust boundary`, `caller vs actor`, `default deny`, `exactly one header`, `header smuggling`, `route allowlist`, `canonical identity`, `constant-time comparison`, `credential separation`, `DTO-to-command mapping`

### 백지 구현

**구현 목표**

순수 함수 형태의 gateway ingress 판정기와, 신뢰된 actor를 placement command로 매핑하는 함수를 구현한다. 실제 servlet 응답 작성은 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record RequestView(
    String method,
    String path,
    List<String> serviceHeaders,
    List<String> apiKeyHeaders,
    List<String> actorHeaders) {}

enum BoundaryResult { ALLOW, UNAUTHORIZED, FORBIDDEN, BAD_REQUEST }

record PlaceBetBody(String slipType, Integer minWins, Integer totalSelections, List<SelectionBody> selections, long stake) {}
record PlaceBetCommand(UUID actorId, String idempotencyKey, String slipType, Integer minWins, Integer totalSelections, List<SelectionBody> selections, long stake) {}

final class GatewayBoundary {
  BoundaryResult authorize(RequestView request, byte[] expectedKey) {
    // 직접 구현
    return null;
  }

  PlaceBetCommand toCommand(UUID trustedActor, String idempotencyKey, PlaceBetBody body) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: method, context 제거가 끝난 business path, 모든 동일명 헤더 값, 예상 credential, 신뢰된 actor, body
- 출력: 경계 판정 또는 actor가 고정된 placement command

**반드시 만족해야 할 조건**

- 보호 경로의 service, API key, actor 헤더는 각각 정확히 하나다.
- API key가 누락·중복·불일치하면 UNAUTHORIZED다.
- caller는 정확히 허용된 gateway identity여야 한다.
- 허용 route는 collection GET/POST와 canonical UUID item GET로 제한한다.
- 올바른 credential이지만 caller·method·path가 금지되면 FORBIDDEN이다.
- actor UUID는 canonical lowercase 문자열만 허용한다.
- body에는 actor identity가 없어야 하며 command actor는 함수 인자에서만 온다.
- slip shape와 필수 body 값이 잘못되면 BAD_REQUEST 또는 validation exception으로 처리한다.
- secret 비교 결과 외에는 secret 값을 문자열화·로그·오류 메시지에 넣지 않는다.

**경계 조건**

- 각 헤더 0개, 1개, 2개
- 같은 값이 두 번 온 경우
- context path가 있는 실제 URI를 business path로 정규화한 경우
- collection의 PUT/DELETE
- item의 POST
- 대문자 UUID, trailing slash, extra path segment
- actor header와 body의 조작된 userId 필드 시도
- short/blank secret
- 동일한 ingress·outbound secret 구성

**실패 조건**

- credential 구조가 잘못되면 route 존재 여부를 자세히 드러내기 전에 거부한다.
- 인증된 caller라도 allowlist 밖이면 통과시키지 않는다.
- body actor 또는 비정규 actor를 도메인 command로 전달하지 않는다.
- secret 설정이 약하거나 다른 방향의 credential과 같으면 startup 실패로 처리할 수 있다.

**제약**

정규식 또는 명시적 path parser를 사용할 수 있다. framework route metadata 자동 탐색은 제외한다. constant-time 비교는 표준 라이브러리 primitive를 사용한다.

### 구현 후 자가 검증

- [ ] 정상 gateway request만 ALLOW다.
- [ ] missing·duplicate·wrong key가 모두 UNAUTHORIZED다.
- [ ] 같은 값의 duplicate header도 거부된다.
- [ ] wrong caller와 금지 method/path는 FORBIDDEN이다.
- [ ] `/internal/` 또는 `/api/`의 새 route가 자동으로 열리지 않는다.
- [ ] canonical item UUID만 허용된다.
- [ ] actor는 body가 아니라 trusted argument에서 command로 들어간다.
- [ ] 다른 actor의 body 조작이 결과에 영향을 주지 않는다.
- [ ] secret이 예외·디버그 문자열에 나타나지 않는다.
- [ ] ingress credential과 outbound credential 분리 invariant를 검증한다.
- [ ] 정상 response status·Location 정책을 설명할 수 있다.

### 구현 후 설명할 것

1. caller 인증과 end-user actor 신뢰를 분리한 이유
2. exact-one header 검사가 parser disagreement를 줄이는 방식
3. default-deny allowlist의 유지보수 비용과 안전성 trade-off
4. actor를 body에서 제거한 것이 API 사용성보다 중요한 이유
5. credential 방향별 분리, constant-time comparison, secret redaction의 역할

### 원본 확인 위치

- 대표 Thread: 13, 게이트웨이 신뢰 경계와 기본 거부 인가
- 통합 Thread: 14, 신뢰된 actor 매핑과 안정적인 betting HTTP 계약
- Thread 13 커밋: `feat(api): authenticate the gateway boundary`, `feat(api): enforce exact gateway ingress`
- Thread 14 커밋: `feat(api): define placement wire models`, `feat(api): map trusted placement commands`, `feat(api): expose trusted placement and history routes`
- 파일: `GatewayAuthFilter.java`, `GatewayAuthProperties.java`, `PlaceBetRequest.java`, `BetResponse.java`, `BetController.java`
- 함수·컴포넌트: `GatewayAuthFilter.shouldNotFilter(...)`, `GatewayAuthFilter.doFilterInternal(...)`, `GatewayAuthProperties.matches(...)`, `PlaceBetRequest.toCommand(...)`, placement/history controller methods
- 관련 Thread: 04, 05, 19
