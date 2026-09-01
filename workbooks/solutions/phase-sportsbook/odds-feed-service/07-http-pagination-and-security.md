# HTTP pagination·내부 보안 경계 워크북

이 문서는 public current-read API와 internal operator API가 서로 다른 신뢰 경계를 갖는다는 점을 다룬다. 하나는 정렬이 변해도 안정적인 page 이동이 핵심이고, 다른 하나는 인증·권한·기본 거부가 핵심이다.

<a id="p23"></a>
## [Thread 11 / `feat(api): encode stable event cursors`] `(kickoff, eventId)` 복합 cursor pagination

### 면접 질문

event 목록을 kickoff 시각으로 정렬할 때 cursor에 kickoff만 넣지 않고 event ID를 함께 넣은 이유는 무엇입니까? offset pagination과 비교해 설명해 주세요.

꼬리 질문:

- 같은 kickoff를 가진 event가 여러 개면 단일 시각 cursor에 어떤 누락/중복이 생깁니까?
- cursor가 가리키던 event가 다음 요청 전에 삭제돼도 page를 계속 찾을 수 있습니까?
- 새 event가 cursor 앞이나 뒤에 삽입되면 각각 어떻게 보입니까?
- 잘못된 Base64, timestamp, UUID는 어떤 HTTP 응답이어야 합니까?
- page size 0, 음수, 100 초과를 어떻게 처리합니까?

### 30초 모범 답변

offset은 앞쪽 삽입·삭제가 생기면 같은 item을 다시 보거나 건너뛸 수 있습니다. 이 목록은 `(scheduledStartAt, eventId)`를 완전한 정렬 키로 사용하고 cursor에도 두 값을 넣습니다. 다음 page는 cursor와 정확히 같은 item을 찾는 대신 복합 키보다 큰 첫 item부터 시작하므로 cursor item이 삭제돼도 진행할 수 있습니다. event ID가 동률을 깨서 순서가 deterministic합니다. cursor는 URL-safe 인코딩하고 decode·timestamp·UUID 검증 실패는 400으로 처리하며 size는 기본값과 최대값으로 제한합니다.

### 답변 핵심 키워드

keyset pagination, composite sort key, tie-breaker, stable cursor, insertion/deletion, URL-safe Base64, validation, page-size clamp

### 백지 구현

**구현 목표**

event 목록을 복합 키로 정렬하고 opaque cursor를 encode/decode해 다음 page를 반환한다.

**인터페이스**

```java
record EventSummary(
    UUID eventId,
    Instant scheduledStartAt,
    String name) {}

record EventCursor(
    Instant scheduledStartAt,
    UUID eventId) {}

record CursorPage<T>(
    List<T> items,
    String nextCursor) {} // 다음 page가 없으면 null

final class EventPagination {
  static String encode(EventCursor cursor) {
    // 직접 구현
  }

  static EventCursor decode(String encodedCursor) {
    // 직접 구현
  }

  static CursorPage<EventSummary> page(
      Collection<EventSummary> events,
      String encodedCursor,
      int requestedSize) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: unordered current event collection, 선택적 cursor, 요청 size
- 출력: 정렬된 page와 선택적 next cursor

**반드시 만족해야 할 조건**

- 정렬 키: `scheduledStartAt`, 그다음 `eventId`
- cursor 이후의 첫 복합 키부터 반환
- cursor item 자체가 collection에 없어도 key 비교로 진행
- URL-safe opaque encoding
- malformed encoding/timestamp/UUID는 명확한 invalid cursor 예외
- size <= 0은 기본 20
- size > 100은 100
- page items와 원본 collection 사이 mutable alias 없음
- 다음 item이 있을 때만 next cursor 생성
- 같은 snapshot과 cursor는 같은 page

**경계 조건**

- 빈 목록
- cursor 없음
- 모든 event kickoff 동일
- cursor가 첫 item보다 작음
- cursor가 마지막 item과 같거나 큼
- cursor item 삭제
- cursor 앞/뒤 새 item 삽입
- size 1
- invalid Base64 padding/문자

**실패 조건**

- kickoff만 tie-breaker 없이 사용
- offset을 cursor로 위장
- cursor item을 반드시 exact lookup해야 진행
- invalid cursor를 첫 page로 조용히 처리
- raw delimiter 문자열을 URL escaping 없이 노출
- max size 미적용

**필요한 제약**

- 20~30분
- HTTP controller는 구현하지 않음
- exception을 400에 매핑한다는 계약만 설명
- 정렬 후 linear scan 또는 binary search 선택과 복잡도 설명

### 구현 후 자가 검증

- [ ] 입력 iteration 순서와 무관한 정렬
- [ ] 동일 kickoff에서 UUID 순서가 안정적
- [ ] 첫 page size/default/max 처리
- [ ] next cursor decode 후 다음 page에 중복 없음
- [ ] cursor item 삭제 후에도 그 키 뒤에서 시작
- [ ] cursor 앞 삽입 item이 다음 page에 끼어들지 않음
- [ ] cursor 뒤 삽입 item은 정렬 위치에 따라 보임
- [ ] 마지막 page의 next cursor 없음
- [ ] malformed cursor 예외
- [ ] 시간 복잡도와 메모리 비용 설명 가능

### 구현 후 설명할 것

1. keyset pagination과 offset pagination 차이
2. 완전한 정렬을 만드는 tie-breaker
3. cursor에 filter/sort version을 포함해야 하는 확장 상황
4. snapshot consistency가 필요한 경우의 대안
5. encode가 암호화나 서명은 아니라는 점

### 원본 확인 위치

- Thread 11
- 커밋: `feat(api): encode stable event cursors`
- 커밋: `feat(api): paginate event summaries`
- 파일/컴포넌트: `EventReadController`, `EventCatalog`, `CursorPage`
- 함수: `encodeCursor`, `decodeCursor`, `indexAfter`, `orderedByKickoff`, `clampSize`
- 정렬 키: kickoff, event UUID
- 관련 Thread: 07

---

<a id="p24"></a>
## [Thread 12 / `feat(security): authenticate internal callers`] constant-time 인증, 401/403 분리, deny-by-default

### 면접 질문

내부 API key 인증에서 supplied 문자열을 바로 비교하지 않고 SHA-256 digest를 만든 뒤 `MessageDigest.isEqual`로 비교한 이유는 무엇입니까? 올바른 key지만 service header가 `admin-api`가 아닌 caller는 왜 401이 아니라 403이 됩니까?

꼬리 질문:

- API key가 누락·blank·오류일 때 `SecurityContext`를 왜 명시적으로 비웁니까?
- `X-Internal-Service`를 credential과 동일하게 신뢰하면 어떤 문제가 생깁니까?
- public endpoint allowlist 뒤 `anyRequest().denyAll()`을 둔 이유는 무엇입니까?
- health detail과 prometheus를 모두 anonymous로 공개해도 됩니까?
- API key 인증에서 replay, rotation, audit 한계는 무엇입니까?

### 30초 모범 답변

비밀 문자열의 일반 equality는 첫 불일치 위치에 따라 시간이 달라질 수 있어 digest를 같은 길이로 만든 뒤 constant-time 비교를 사용했습니다. key가 없거나 틀리면 신원을 확립할 수 없으므로 context를 비우고 401로 끝냅니다. key가 맞으면 caller는 인증되지만, service가 `admin-api`일 때만 내부 admin authority를 부여합니다. 다른 service가 admin POST를 호출하면 인증은 됐지만 권한이 없어 403입니다. public GET과 필요한 내부 POST만 allowlist하고 나머지는 기본 거부해 새 route가 실수로 공개되지 않게 했습니다.

### 답변 핵심 키워드

constant-time comparison, digest, authentication vs authorization, 401 vs 403, SecurityContext cleanup, authority, allowlist, deny-by-default

### 백지 구현

**구현 목표**

내부 요청 header를 검증해 authentication을 만들고, method/path별 authorization 결정을 내리는 작은 보안 경계를 구현한다. Spring Security API를 몰라도 풀 수 있게 축소한다.

**인터페이스**

```java
record HttpRequest(
    String method,
    String path,
    Map<String, String> headers) {}

record Principal(
    String service,
    Set<String> authorities) {}

enum SecurityOutcome {
  CONTINUE_PUBLIC,
  CONTINUE_AUTHENTICATED,
  UNAUTHORIZED,
  FORBIDDEN,
  NOT_FOUND_OR_DENIED
}

record SecurityDecision(
    SecurityOutcome outcome,
    Principal principal) {} // 없으면 null

final class InternalSecurityBoundary {
  static final String SERVICE_HEADER = "X-Internal-Service";
  static final String API_KEY_HEADER = "X-Internal-Api-Key";
  static final String ADMIN_SERVICE = "admin-api";
  static final String ADMIN_AUTHORITY = "ODDS_INTERNAL_ADMIN";

  InternalSecurityBoundary(String expectedApiKey) {
    // 직접 구현
  }

  SecurityDecision decide(HttpRequest request) {
    // 직접 구현
  }

  static boolean constantTimeSecretMatch(
      byte[] expectedDigest,
      String supplied) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: method, path, headers, 생성 시 expected API key
- 출력: public 통과, 인증 통과, 401, 403, 기본 거부 중 하나와 선택적 principal

**반드시 만족해야 할 조건**

- `/internal/`이 아닌 명시적 public GET allowlist만 anonymous 통과
- internal request는 API key 누락/blank/오류 시 401
- secret은 UTF-8 SHA-256 digest 후 constant-time 비교
- valid key이면 service 이름으로 authenticated principal 생성
- service가 `admin-api`일 때만 admin authority
- admin POST route는 해당 authority 필요
- valid key + 다른 service의 admin POST는 403
- 미등록 route는 기본 거부
- secret 원문을 principal, log, exception에 포함하지 않음
- invalid credential 경로에서 이전 principal이 남지 않는 모델

**경계 조건**

- service header 누락
- API key header 누락/blank
- API key 정확, service 다름
- method만 다른 같은 path
- `/internality/...`처럼 prefix가 비슷한 path
- URL normalization/중복 slash
- public health/prometheus detail 공개 범위
- expected key가 너무 짧거나 비어 있음

**실패 조건**

- 일반 문자열 equality
- service header만으로 authority 부여
- invalid key인데 기존 context 재사용
- `.anyRequest().permitAll()`에 준하는 fallback
- 401과 403 혼동
- header 값을 로그에 그대로 기록

**필요한 제약**

- 20~30분
- allowlist route는 소수의 상수로 제공해도 됨
- HTTP 응답 body, CSRF, session 구현은 범위 밖
- expected digest는 생성 시 한 번 계산

### 구현 후 자가 검증

- [ ] public allowlist GET anonymous 통과
- [ ] 같은 path의 POST는 기본 거부
- [ ] internal key 누락/blank/오류 모두 401
- [ ] valid key + `admin-api`에 authority 존재
- [ ] valid key + 다른 service는 authenticated지만 authority 없음
- [ ] non-admin service의 admin route가 403
- [ ] unknown route 거부
- [ ] expected digest가 요청마다 재계산되지 않음
- [ ] supplied secret이 결과/오류에 노출되지 않음
- [ ] 이전 요청 principal이 invalid 요청으로 새지 않음

### 구현 후 설명할 것

1. 인증과 권한 부여를 분리한 이유
2. digest 후 constant-time 비교의 위협 모델과 한계
3. API key 방식의 rotation·audit·replay 한계
4. deny-by-default가 신규 controller 추가 사고를 줄이는 방식
5. stateless API에서 CSRF 비활성화가 가능한 조건

### 원본 확인 위치

- Thread 12
- 커밋: `feat(security): authenticate internal callers`
- 커밋: `test(security): verify internal authentication`
- 커밋: `test(security): verify route authorization`
- 파일/컴포넌트: `InternalApiKeyAuthenticationFilter`, `SecurityConfig`, `InternalSecurityProperties`
- 상수/경계: `X-Internal-Service`, `X-Internal-Api-Key`, `admin-api`, `ODDS_INTERNAL_ADMIN`
- 함수: `shouldNotFilter`, `doFilterInternal`, `matches`, `securityFilterChain`
- 관련 Thread: 11, 13
