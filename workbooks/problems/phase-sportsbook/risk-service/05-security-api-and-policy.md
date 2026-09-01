# 내부 보안·HTTP 계약·정책 override

이 문서는 내부 호출자 인증과 경로 소유권, 비즈니스 결과와 프로토콜 오류의 분리, 통화 차원을 타입으로 강제하는 사용자별 한도 override를 다룬다. 프레임워크 설정 암기보다 "어떤 값을 신뢰 경계에 남길 것인가"와 "잘못된 저장 상태에서 기본값으로 우회할 것인가"를 중심으로 구성했다.

---

## P17. [Thread 03 / `feat(auth): bind internal caller credentials`] 호출자별 secret과 경로 소유권을 분리한 내부 인증

### 면접 질문

세 내부 서비스가 같은 API key를 공유하지 않고 호출자별 secret을 사용한 이유와, 인증 성공 뒤 SecurityContext에 원본 secret을 남기지 않은 이유를 설명해 주세요.

꼬리 질문:

- secret을 애플리케이션 시작 시 digest로 바꿔 보관하면 어떤 장점이 있습니까?
- 문자열 `equals` 대신 digest와 `MessageDigest.isEqual`을 사용한 이유는 무엇입니까?
- secret이 짧거나 서로 같을 때 애플리케이션 시작을 실패시킨 이유는 무엇입니까?
- 인증된 caller와 endpoint 소유권 검사는 왜 별도 단계입니까?
- health와 Prometheus endpoint를 익명 허용하면서 나머지는 `denyAll`로 끝낸 이유는 무엇입니까?

### 30초 모범 답변

caller 이름과 caller별 API key를 함께 검증해 누가 호출했는지를 principal로 만들고, 역할별 route ownership을 별도로 검사했습니다. secret은 최소 길이와 상호 중복을 시작 시 검증한 뒤 SHA-256 digest만 보관하고 일정 시간 비교를 사용하며, 인증 객체에는 credentials를 남기지 않습니다. 세션·폼 로그인·기본 인증은 끄고 stateless filter로 처리하며, 명시한 health·metrics와 소유 route 외 요청은 기본 거부해 새 endpoint가 우연히 공개되지 않게 했습니다.

### 답변 핵심 키워드

caller-specific secret · startup validation · digest at rest in process · constant-time comparison · credential erasure · stateless · least privilege · deny by default

### 백지 구현

#### 구현 목표

프레임워크 없이 내부 caller 인증과 단순 route ownership을 구현한다. 구성 시 secret 계약을 검증하고, 인증 결과에는 원본 secret이 남지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```java
enum Caller {
  BETTING_SERVICE("betting-service"),
  ADMIN_API("admin-api"),
  PLATFORM("platform");

  private final String wireName;
}

record Principal(String name, Set<String> roles) {}

public final class InternalCredentials {
  public InternalCredentials(Map<Caller, String> secrets) {
    // 직접 구현
  }

  public Optional<Principal> authenticate(
      String callerWireName,
      String candidateSecret) {
    // 직접 구현
  }
}

public static boolean authorized(
    Principal principal,
    String method,
    String path) {
  // 직접 구현
}
```

#### 입력과 출력

- 구성 입력: 모든 caller의 secret
- 인증 입력: wire caller 이름과 후보 secret
- 인증 출력: secret을 포함하지 않는 principal 또는 빈 결과
- 인가 입력: principal, HTTP method, path
- 인가 출력: 소유 route면 true, 그 외 false

#### 반드시 만족해야 할 조건

- 모든 caller secret은 존재하고 공백이 아니며 최소 32자다.
- 서로 다른 caller가 같은 secret을 사용할 수 없다.
- 생성 뒤에는 원본 secret 문자열을 필드에 보관하지 않는다.
- 인증 비교는 caller별 SHA-256 digest를 대상으로 일정 시간 비교 함수를 사용한다.
- caller wire 이름은 canonical 값과 정확히 일치해야 한다.
- principal의 credentials나 `toString`에 secret이 들어가지 않는다.
- 베팅 서비스는 reservation create/commit/release, admin은 limit route, platform은 diagnostic과 보호 actuator route만 소유한다.
- 익명 허용 route는 health 계열과 Prometheus로 제한한다.
- 매칭되지 않은 route는 기본 거부한다.

#### 경계 조건

- secret 길이 31, 32
- 두 caller의 중복 secret
- null/unknown/case가 다른 caller 이름
- 올바른 caller와 다른 caller의 secret 조합
- path가 비슷하지만 소유하지 않은 endpoint
- 새 method 또는 새 route

#### 실패 조건

- 구성 누락·짧은 secret·중복 secret
- 해시 알고리즘 초기화 실패
- 인증 header 누락 또는 mismatch
- principal 없음
- route matrix에 없는 요청

#### 필요한 제약

- Spring Security API는 사용하지 않는다.
- plain secret을 map에 보관하지 않는다.
- raw path prefix만 과도하게 넓게 허용하지 않는다.
- 20~30분 안에 구성 검증, 인증, 대표 route 인가 테스트를 작성한다.

### 구현 후 자가 검증

- [ ] 세 caller의 정상 secret으로 객체가 생성된다.
- [ ] 누락·31자·중복 secret에서 생성이 실패한다.
- [ ] 올바른 caller/secret만 principal을 반환한다.
- [ ] unknown caller와 caller-secret 교차 조합이 실패한다.
- [ ] principal과 구성 객체 문자열 표현에 secret이 없다.
- [ ] 인증 결과의 roles가 caller와 일치한다.
- [ ] 각 소유 route의 정상 method만 허용된다.
- [ ] health/Prometheus 외 익명 route는 허용되지 않는다.
- [ ] 알 수 없는 route가 기본 거부된다.
- [ ] 인증 테스트 뒤 thread-local security state를 쓰는 구현이라면 cleanup이 보장된다.

### 구현 후 설명할 것

1. 공유 key보다 caller별 key가 감사·폐기·최소 권한에 유리한 이유
2. SHA-256 digest 보관이 완전한 secret 저장소 보호는 아니라는 한계
3. 일정 시간 비교를 적용해도 header 길이·네트워크 등 다른 timing 신호가 남는다는 점
4. authentication과 authorization을 분리한 이유
5. allowlist 뒤 `denyAll`을 둬 새 endpoint의 기본 상태를 안전하게 만든 이유

### 원본 확인 위치

- Thread 03
- 커밋: `feat(auth): bind internal caller credentials`
- 파일: `src/main/java/com/sportsbook/risk/auth/InternalAuthProperties.java`, `src/main/java/com/sportsbook/risk/auth/InternalAuthenticationFilter.java`, `src/main/java/com/sportsbook/risk/auth/InternalSecurityConfiguration.java`
- 클래스: `InternalAuthProperties`, `InternalAuthenticationFilter`, `InternalSecurityConfiguration`
- 테스트: `InternalAuthPropertiesTest`, `InternalAuthenticationFilterTest`, `InternalSecurityIntegrationTest`
- 관련 Thread: 04, 16, 17

---

## P18. [Thread 04 / `feat(api): render stable problem details`] 비즈니스 거절과 HTTP 오류를 구분하는 안정적 계약

### 면접 질문

위험 한도 초과는 왜 HTTP 4xx가 아니라 HTTP 200의 비즈니스 결과로 반환하고, malformed request나 lifecycle conflict는 안정적인 `ProblemDetail`로 분리했나요?

꼬리 질문:

- 같은 "승인되지 않음"이라도 policy rejection과 fingerprint conflict가 다른 protocol 의미를 갖는 이유는 무엇입니까?
- 예상하지 못한 예외 메시지를 그대로 응답에 노출하면 어떤 문제가 생깁니까?
- 오류 code와 HTTP status 중 클라이언트가 어느 값을 장기 계약으로 사용해야 합니까?
- 진단 endpoint의 거절과 reservation endpoint의 거절을 같은 권위로 보면 왜 안 됩니까?
- 테스트 가능한 controller에서 `Clock`을 주입한 이유는 무엇입니까?

### 30초 모범 답변

한도 초과는 요청 형식이나 인증 실패가 아니라 정상적으로 계산된 업무 결과라 200 응답의 `approved=false`로 표현했습니다. 반면 잘못된 입력, 없는 lifecycle, 허용되지 않은 transition, identity conflict는 protocol 오류이므로 상태 코드와 안정적인 error code를 가진 problem shape로 보냅니다. 예상하지 못한 예외는 서버에 원인을 로그로 남기되 외부에는 공통 `INTERNAL_ERROR`만 노출합니다. 진단 결과는 advisory이고 실제 용량을 예약하지 않으므로 권위 승인은 reservation 응답만 담당합니다.

### 답변 핵심 키워드

application result vs protocol error · stable error code · opaque 500 · `application/problem+json` · diagnostic vs authoritative · injected clock

### 백지 구현

#### 구현 목표

도메인 결과와 예외를 HTTP 응답 모델로 변환하는 순수 mapper를 작성한다. 정상적인 업무 거절과 protocol 실패를 명확히 분리한다.

#### 인터페이스 또는 함수 시그니처

```java
sealed interface DomainResult
    permits Approved, PolicyRejected {}

record Approved(String token, boolean replayed)
    implements DomainResult {}

record PolicyRejected(String reason, boolean replayed)
    implements DomainResult {}

enum ApiFailure {
  VALIDATION_FAILED,
  RESERVATION_NOT_FOUND,
  RESERVATION_COMMITTED,
  RESERVATION_CONFLICT,
  INTERNAL_ERROR
}

record HttpResponse(
    int status,
    String contentType,
    Map<String, Object> body) {}

public static HttpResponse success(DomainResult result) {
  // 직접 구현
}

public static HttpResponse failure(
    ApiFailure failure,
    String safeDetail) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 정상 도메인 결과 또는 분류된 API failure
- 출력: status, content type, 안정적인 body를 가진 응답 모델

#### 반드시 만족해야 할 조건

- 승인과 policy rejection은 모두 HTTP 200의 정상 application result다.
- policy rejection body에는 안정적인 reason이 있고 token은 없다.
- validation, not found, committed conflict, identity conflict는 각 계약 status와 stable error code를 가진다.
- 오류 응답 content type은 problem JSON 계열이다.
- 예상하지 못한 exception의 class명, stack trace, 내부 message는 body에 들어가지 않는다.
- 동일 failure는 호출할 때마다 같은 error code와 구조를 유지한다.
- safe detail과 내부 log detail의 책임을 구분한다.
- replay 여부는 정상 결과에서만 의미 있게 표현한다.

#### 경계 조건

- 최초 승인과 replay 승인
- 최초 거절과 replay 거절
- 빈 safe detail
- 알려진 reservation conflict
- 알 수 없는 내부 예외
- body field의 null 포함 정책

#### 실패 조건

- null result/failure
- 지원하지 않는 결과 subtype
- status와 error code의 잘못된 매핑
- 내부 예외 message가 응답에 노출
- policy rejection을 4xx로 잘못 분류

#### 필요한 제약

- Spring MVC 없이 순수 mapper로 구현한다.
- stack trace나 exception 객체를 body에 넣지 않는다.
- 전체 RFC 구현보다 프로젝트에 필요한 안정적 필드만 만든다.
- 15~20분 안에 대표 매핑과 테스트를 끝낸다.

### 구현 후 자가 검증

- [ ] 승인과 policy rejection이 모두 200이다.
- [ ] 거절 응답에 token이 없다.
- [ ] not found·committed·identity conflict가 서로 다른 stable error code를 갖는다.
- [ ] validation 오류가 내부 parser message를 그대로 노출하지 않는다.
- [ ] 내부 예외 응답에 private detail이 없다.
- [ ] 모든 problem 응답의 content type과 공통 shape가 일관된다.
- [ ] 동일 입력의 직렬화 field가 결정적이다.
- [ ] controller가 주입된 시각으로 command를 만든다는 별도 테스트를 설명할 수 있다.
- [ ] 진단 응답을 reservation token처럼 사용할 수 있는 field가 없다.

### 구현 후 설명할 것

1. 업무 거절을 200으로 표현한 장점과 모니터링 시 주의점
2. HTTP status와 stable error code의 역할 분담
3. opaque 500 응답과 서버-side logging의 경계
4. diagnostic endpoint와 authoritative reservation endpoint를 분리한 이유
5. 오류 계약을 버전 없이 바꿀 때 클라이언트가 받는 영향

### 원본 확인 위치

- Thread 04
- 커밋: `feat(api): render stable problem details`
- 파일: `src/main/java/com/sportsbook/risk/api/RestExceptionHandler.java`
- 클래스: `RestExceptionHandler`, `RiskApiException`, `RiskCheckController`, `RiskReservationController`
- 테스트: `RestExceptionHandlerTest`, `RiskCheckControllerTest`, `RiskReservationAdmissionTest`
- 관련 Thread: 03, 09, 11, 16

---

## P19. [Thread 05 / `feat(limits): encode user override fields`] 통화 차원과 기본값 우선순위를 타입으로 강제하기

### 면접 질문

금액 한도는 통화별이고 선택 수 한도는 통화 중립인데, 이를 단순 문자열 key 조합이 아니라 `LimitOverrideField` 타입으로 만든 이유는 무엇입니까?

꼬리 질문:

- 금액 한도에 currency가 없거나 선택 한도에 currency가 있으면 어디서 막습니까?
- override가 없을 때와 override 값이 0일 때를 어떻게 구분합니까?
- Redis에 `"corrupt"`나 `-1`이 저장되어 있으면 기본 정책으로 fallback해도 됩니까?
- 기본 정책과 사용자 override의 우선순위를 어디에 집중시켰습니까?
- key 생성 계약을 store 밖의 공통 컴포넌트로 뺀 이유는 무엇입니까?

### 30초 모범 답변

한도 종류마다 차원이 다르므로 필드 타입 생성 시 금액 한도에는 currency를 필수로, 선택 한도에는 currency를 금지했습니다. 그래서 `"STAKE_DAILY:KRW"`와 `"SELECTIONS_PER_MINUTE"` 같은 Redis field가 유효 객체에서만 생성됩니다. resolver는 override 존재 시 그 값을 사용하고 없을 때만 배포 기본값으로 내려갑니다. 저장값이 비정수·음수·정확 범위 밖이면 "override 없음"으로 취급하지 않고 실패시켜 손상된 관리 정책이 기본값으로 조용히 우회되지 않게 합니다.

### 답변 핵심 키워드

dimension-aware type · currency-scoped · currency-neutral · `OptionalLong` · override precedence · corrupt value fail-closed · canonical key

### 백지 구현

#### 구현 목표

한도 종류의 차원 invariant를 표현하는 field 타입과, 사용자 override를 기본 정책보다 우선해 해석하는 resolver를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
enum LimitType {
  STAKE_DAILY(true),
  STAKE_WEEKLY(true),
  STAKE_MONTHLY(true),
  SELECTIONS_PER_MINUTE(false);

  private final boolean currencyScoped;
}

record OverrideField(
    LimitType type,
    String currency) {

  public OverrideField {
    // 직접 구현
  }

  public String redisField() {
    // 직접 구현
  }
}

interface OverrideStore {
  OptionalLong find(String userId, OverrideField field);
}

interface Defaults {
  long value(LimitType type, String currency);
}

public static long resolve(
    OverrideStore store,
    Defaults defaults,
    String userId,
    LimitType type,
    String currency) {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 사용자, 한도 종류, 필요한 경우 통화
- 출력: override가 있으면 override, 없으면 기본 정책 값
- 잘못된 차원 또는 손상 store 결과: 예외

#### 반드시 만족해야 할 조건

- currency-scoped type은 null/빈 currency를 허용하지 않는다.
- currency-neutral type은 currency를 허용하지 않는다.
- Redis field 문자열은 동일한 유효 객체에서 항상 동일하게 나온다.
- override `OptionalLong.of(0)`과 `OptionalLong.empty()`를 구분한다.
- override가 있으면 defaults를 조회하지 않아도 된다.
- override가 없을 때만 defaults로 fallback한다.
- 반환 값은 정확 정수 non-negative 도메인을 만족한다.
- store가 손상 값을 예외로 알리면 resolver가 이를 "없음"으로 바꾸지 않는다.

#### 경계 조건

- KRW·USD 금액 field
- 통화 중립 선택 field
- override 없음
- override 0
- override가 정확 정수 상한
- currency-scoped type에 currency 누락
- neutral type에 currency 제공

#### 실패 조건

- null user/type
- 차원 불일치
- store의 비정수·음수·상한 초과 값
- defaults가 잘못된 값을 반환
- 알 수 없는 currency

#### 필요한 제약

- raw field 문자열을 호출자가 직접 조립하지 않는다.
- store exception을 삼키지 않는다.
- Redis 연결 코드는 구현하지 않고 field와 resolver에 집중한다.
- 15~25분 안에 테스트를 포함해 끝낸다.

### 구현 후 자가 검증

- [ ] 각 금액 type과 currency가 canonical field를 만든다.
- [ ] 선택 한도 field에는 currency suffix가 없다.
- [ ] 잘못된 차원 조합이 생성 시 실패한다.
- [ ] override 존재 시 그 값이 반환되고 defaults는 사용되지 않는다.
- [ ] override 없음에서만 defaults가 반환된다.
- [ ] override 0이 "없음"으로 처리되지 않는다.
- [ ] store 손상 예외가 그대로 실패로 전달된다.
- [ ] 반환 값의 정확 정수 범위가 검증된다.
- [ ] field equality와 hash semantics가 차원을 정확히 반영한다.

### 구현 후 설명할 것

1. 문자열 조합보다 값 타입이 안전한 이유
2. 0 override와 override 없음의 의미 차이
3. 손상된 override에서 기본값 fallback을 금지한 이유
4. resolver에 precedence 규칙을 집중시킨 이유
5. 새 한도 차원을 추가할 때 enum boolean만으로 충분한지, 더 강한 타입 계층이 필요한지

### 원본 확인 위치

- Thread 05
- 커밋: `feat(limits): encode user override fields`
- 파일: `src/main/java/com/sportsbook/risk/limit/LimitOverrideField.java`
- 클래스: `LimitOverrideField`, `LimitResolver`, `LimitOverrideStore`, `RedisLimitOverrideStore`, `LimitOverrideKeys`
- 테스트: `LimitOverrideFieldTest`, `LimitResolverTest`, `RedisLimitOverrideStoreTest`
- 관련 Thread: 02, 07, 08, 10, 12
