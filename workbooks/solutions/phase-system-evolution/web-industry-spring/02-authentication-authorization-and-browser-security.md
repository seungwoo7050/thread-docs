# 개발자 기술면접 워크북 02 — 인증, 인가, 브라우저 변경 보안

이 문서는 세션 인증의 수명주기, 객체 단위 인가, CSRF와 Origin 검사를 하나의 보안 경계로 묶는다. 인증 여부와 리소스 소유권, 브라우저 요청 출처는 서로 다른 문제이므로 각각 독립된 invariant로 연습한다.

---

<!-- coverage: SA-05 -->
<a id="sa-05"></a>
## [Thread 04 / `Spring Security 세션 인증과 1시간 수명주기 적용`] 세션 고정 방지와 절대 만료 경계

### 면접 질문

로그인 성공 시 기존 익명 세션 ID를 그대로 사용하지 않고 회전시킨 이유는 무엇입니까? 1시간 idle timeout만 두지 않고 별도의 절대 만료 시각도 기록한 이유와, 만료 직전 1밀리초와 정확한 만료 시점의 처리 차이를 설명해 주세요.

꼬리 질문:

- CSRF 토큰을 얻기 위해 만들어진 익명 세션은 왜 인증된 세션이 아닙니까?
  - 모범답변: 세션 객체와 CSRF 값이 존재해도 Spring SecurityContext에 인증 주체가 저장된 것은 아닙니다. 원본은 context key 유무를 기준으로 절대 만료 적용 대상도 구분합니다.
- 세션 만료 검사를 SecurityContext를 읽기 전에 수행하면 어떤 이점이 있습니까?
  - 모범답변: 만료된 context가 request에 복원되어 잠시라도 인증된 주체로 사용되는 창을 닫습니다. 원본은 `SecurityContextHolderFilter` 앞에서 세션을 무효화합니다.
- 로그아웃 시 서버 세션 무효화와 쿠키 삭제를 모두 해야 하는 이유는 무엇입니까?
  - 모범답변: 서버 무효화는 탈취된 복사본까지 권한을 폐기하고 cookie 삭제는 브라우저가 낡은 ID를 계속 보내지 않게 합니다. 어느 한쪽만으로는 두 저장 위치가 모두 정리되지 않습니다.
- 인증 주체 객체에서 비밀번호나 해시를 제거해야 하는 이유는 무엇입니까?
  - 모범답변: SecurityContext가 세션에 직렬화되므로 credential이 남아 있으면 세션 저장·로그·디버깅 경로로 비밀이 확산됩니다. 원본 `ProviderManager`가 인증 후 credential을 지웁니다.

### 30초 모범 답변

공격자가 미리 알던 익명 세션 ID가 로그인 뒤에도 유지되면 세션 고정 공격이 가능하므로 인증 성공 시 ID를 회전시켜야 합니다. idle timeout은 요청이 계속 오면 연장되기 때문에 세션의 최대 생존 시간을 제한하지 못합니다. 그래서 로그인 시 절대 만료 시각을 저장하고, 요청 시 `now < expiresAt`일 때만 유효하게 했습니다. 정확한 만료 시점부터는 401로 처리하고 세션을 무효화합니다. CSRF용 익명 세션은 인증 주체가 없으므로 로그아웃 성공으로 간주하지 않으며, 실제 로그아웃은 서버 세션 폐기와 쿠키 제거를 함께 수행합니다.

### 답변 핵심 키워드

session fixation, ID rotation, idle timeout, absolute expiry, exact boundary, anonymous session, revocation, cookie deletion, credential erasure

### 백지 구현

#### 구현 목표

로그인 전·후 세션 상태와 절대 만료를 관리하는 최소 수명주기 컴포넌트를 작성한다. 웹 프레임워크 종속 코드는 생략하고 세션 저장소 인터페이스로 표현한다.

#### 인터페이스 또는 함수 시그니처

```java
enum SessionDecision { ALLOW, EXPIRED, ANONYMOUS }

interface SessionView {
    boolean authenticated();
    Instant absoluteExpiresAt();
    void setAbsoluteExpiresAt(Instant expiresAt);
    void rotateId();
    void invalidate();
}

final class SessionLifecycle {
    void onAuthenticationSuccess(SessionView session, Instant now, Duration absoluteTtl) {
        session.rotateId(); // 공격자가 알던 익명 ID를 인증된 ID로 승격하지 않는다.
        session.setAbsoluteExpiresAt(now.plus(absoluteTtl));
    }

    SessionDecision decide(SessionView session, Instant now) {
        if (!session.authenticated()) return SessionDecision.ANONYMOUS;
        Instant expiresAt = session.absoluteExpiresAt();
        // 인증된 legacy session에 deadline이 없는 경우도 fail-closed 한다.
        if (expiresAt == null || !now.isBefore(expiresAt)) {
            session.invalidate();
            return SessionDecision.EXPIRED;
        }
        return SessionDecision.ALLOW;
    }
}
```

#### 입력과 출력

- 입력: 현재 세션 상태, 신뢰 가능한 시계, 절대 TTL
- 출력: 요청 허용·만료·익명 결정
- 부수 효과: 로그인 시 ID 회전과 만료 시각 설정, 만료 시 무효화

#### 반드시 만족해야 할 조건

- 로그인 성공 시 세션 ID가 회전한다.
- 절대 만료 시각은 로그인 성공 시 한 번 정해지며 일반 요청마다 연장되지 않는다.
- `now`가 만료 시각과 같거나 늦으면 만료다.
- 인증 주체가 없는 세션은 익명으로 판정한다.
- 만료된 세션은 이후 요청에서 다시 인증 상태로 복구되지 않는다.

#### 경계 조건

- 만료 1밀리초 전
- 만료 시각과 정확히 같은 순간
- 만료 1밀리초 후
- 절대 만료 속성이 없는 오래된 세션
- CSRF 토큰만 가진 익명 세션
- 이미 무효화된 세션

#### 실패 조건

- 만료 세션의 SecurityContext를 정상 인증으로 사용하지 않는다.
- 로그인 성공 뒤 이전 세션 ID를 계속 유효하게 두지 않는다.
- 절대 만료가 요청 활동에 의해 연장되지 않는다.
- 로그아웃 또는 만료 후 쿠키만 지우고 서버 상태를 남기지 않는다.

#### 필요한 제약

- `sleep` 없이 주입된 시계로 테스트 가능해야 한다.
- 구현 시간은 15~20분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 로그인 성공 시 ID 회전이 정확히 한 번 발생한다.
- [ ] 이전 ID로 보호 자원에 접근할 수 없다.
- [ ] 만료 직전은 허용되고 정확한 만료 시점은 거부된다.
- [ ] 익명 CSRF 세션은 `ANONYMOUS`다.
- [ ] 만료 후 세션 상태와 쿠키를 함께 폐기할 수 있다.
- [ ] 시계가 고정되어 테스트가 시간 지연에 의존하지 않는다.
- [ ] 동시 요청이 만료 상태를 되돌리지 못한다.

### 구현 후 설명할 것

1. 세션 ID 회전이 막는 공격 경로
   - 모범답변: 공격자가 로그인 전에 피해자에게 고정시킨 익명 ID를 알고 있어도 로그인 성공 때 새 ID로 바뀌므로 그 ID로 인증 세션을 재사용할 수 없습니다.
2. idle timeout과 absolute timeout의 차이
   - 모범답변: idle timeout은 활동할 때마다 연장될 수 있고 absolute timeout은 로그인 시점부터 정한 최대 수명으로 요청 활동과 무관합니다. 원본은 둘 다 1시간으로 둡니다.
3. 정확한 만료 비교 연산을 선택한 이유
   - 모범답변: 유효 조건을 `now.isBefore(expiry)`로 정의해 만료 1ms 전만 허용하고 정확히 같은 순간부터 거부합니다. 경계가 한쪽으로 명확합니다.
4. 익명 세션과 인증 세션을 구분한 기준
   - 모범답변: session 존재나 CSRF attribute가 아니라 저장된 Spring SecurityContext의 인증 주체가 기준입니다. 익명 bootstrap 세션에는 absolute auth deadline을 요구하지 않습니다.
5. 서버 세션 무효화와 브라우저 쿠키 삭제를 함께 하는 이유
   - 모범답변: 서버 권한과 client bearer 저장소를 동시에 정리해 현재 브라우저와 token 복사본 양쪽에서 재사용을 막기 위해서입니다.

### 원본 확인 위치

- Thread 04
- 커밋: `Spring Security 세션 인증과 1시간 수명주기 적용`
- `backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java`
- `backend/src/main/java/dev/evolution/monitor/SessionController.java`
- `backend/src/main/java/dev/evolution/monitor/SessionExpiryFilter.java`
- `backend/src/main/java/dev/evolution/monitor/UserAccounts.java`
- `backend/src/test/java/dev/evolution/monitor/SessionAuthenticationTest.java`
- 관련 Thread: Thread 05의 브라우저 변경 보호, Thread 08의 요청별 SSR 사용자 격리

---

<!-- coverage: SA-06 -->
<a id="sa-06"></a>
## [Thread 05 / `test(authz): freeze the two-user IDOR counterexample`] 소유자 조건을 데이터 접근 경계에 포함하는 객체 단위 인가

### 면접 질문

Alice와 Bob이 각각 Monitor를 가진 상황에서, 컨트롤러에서 로그인 여부만 확인하고 `monitorId`로 조회하면 왜 IDOR가 발생합니까? 모든 읽기·수정·삭제·하위 CheckRun 조회에 `ownerUserId`를 함께 넣어야 하는 이유를 설명해 주세요.

꼬리 질문:

- 다른 사용자의 실제 ID와 존재하지 않는 ID를 모두 404로 응답한 이유는 무엇입니까?
  - 모범답변: 응답 차이로 다른 사용자의 객체 존재를 열거하지 못하게 합니다. owner-scoped query가 0행이라는 하나의 공개 의미만 제공합니다.
- 수정 전에 소유권을 확인한 뒤 ID만으로 `UPDATE`하면 충분합니까?
  - 모범답변: 아닙니다. 확인과 변경 사이 row·소유권이 바뀌는 TOCTOU가 있고 권한 없는 row를 두 번째 statement가 바꿀 수 있습니다. UPDATE 자체에 `id AND ownerUserId`가 있어야 합니다.
- 수동 Check의 외부 HTTP 요청 전과 결과 저장 전 중 어디에서 소유권을 확인해야 합니까?
  - 모범답변: 둘 다입니다. 실행 전 확인으로 foreign URL에 요청하지 않고, 저장 전 재확인 또는 FK/owner 조건으로 조회 뒤 삭제·변경 경합이 권한 없는 history를 만들지 않게 합니다.
- 하위 리소스인 CheckRun도 부모 Monitor의 소유권으로 묶어야 하는 이유는 무엇입니까?
  - 모범답변: CheckRun ID 자체에 사용자 권한 정보가 없으므로 직접 ID route가 우회 경로가 될 수 있습니다. 항상 parent Monitor와 owner를 join해 같은 객체 권한을 상속해야 합니다.

### 30초 모범 답변

인증은 사용자가 누구인지 확인할 뿐, 그 사용자가 요청한 객체를 소유하는지는 보장하지 않습니다. 따라서 ID만으로 조회한 뒤 애플리케이션에서 비교하기보다, 모든 SQL의 선택·수정·삭제 조건에 객체 ID와 소유자 ID를 함께 넣어야 경합 중에도 권한 경계가 유지됩니다. 하위 CheckRun도 소유자 조건이 있는 Monitor와 연결해 조회합니다. 외부 사용자가 객체 존재 여부를 추론하지 못하도록 다른 사용자의 ID와 없는 ID는 같은 404로 처리하고, 거부된 요청은 행·이력·외부 호출을 하나도 바꾸지 않아야 합니다.

### 답변 핵심 키워드

authentication vs authorization, IDOR, owner-scoped query, row-level predicate, nested resource, non-disclosure 404, no side effect, TOCTOU

### 백지 구현

#### 구현 목표

사용자 소유 리소스의 조회·수정·삭제와 하위 작업 생성을 구현한다. 인가 검사는 컨트롤러의 사전 확인이 아니라 저장소 연산 자체의 조건에 포함해야 한다.

#### 인터페이스 또는 함수 시그니처

```java
record Monitor(UUID id, UUID ownerId, String name, String url) {}
record UpdateMonitor(String name, String url) {}

interface OwnedMonitorStore {
    Optional<Monitor> find(UUID ownerId, UUID monitorId);
    boolean update(UUID ownerId, UUID monitorId, UpdateMonitor input);
    boolean delete(UUID ownerId, UUID monitorId);
    UUID createCheckIntent(UUID ownerId, UUID monitorId);
}
```

#### 입력과 출력

- 입력: 인증 컨텍스트에서 얻은 `ownerId`, 요청 경로의 `monitorId`, 변경 데이터
- 출력: 소유한 객체 또는 성공 여부
- 실패: foreign과 nonexistent를 같은 공개 결과로 반환

#### 반드시 만족해야 할 조건

- 모든 리소스 쿼리는 `ownerId`와 객체 ID를 함께 사용한다.
- UPDATE와 DELETE도 조건부 단일 연산으로 수행한다.
- Check intent 생성 전 부모 Monitor 소유권을 데이터베이스 경계에서 확인한다.
- 하위 CheckRun 단건·목록 조회도 부모 소유권을 통과해야 한다.
- 거부된 요청은 Monitor, CheckRun, 외부 요청 수를 바꾸지 않는다.

#### 경계 조건

- 본인 소유 ID
- 다른 사용자 소유 ID
- 존재하지 않는 ID
- 조회와 수정 사이에 소유권 또는 행이 변경되는 경합
- 부모는 삭제됐지만 하위 ID를 알고 있는 요청
- 같은 ID에 대한 두 사용자의 동시 접근

#### 실패 조건

- foreign 요청에 403, nonexistent에 404처럼 존재 여부를 구분해 노출하지 않는다.
- 먼저 ID만 조회한 뒤 메모리에서 owner를 비교하는 방식에 의존하지 않는다.
- 사전 검사 후 owner 조건 없는 UPDATE/DELETE를 실행하지 않는다.
- 권한 거부 후 외부 HTTP를 호출하지 않는다.

#### 필요한 제약

- SQL 또는 repository 메서드 이름으로 조건을 표현해도 된다.
- 실제 ORM 설정은 생략하고 20분 안에 핵심 경계를 구현한다.

### 구현 후 자가 검증

- [ ] 본인 객체의 정상 CRUD가 가능하다.
- [ ] foreign과 nonexistent의 공개 상태·오류 코드·본문 모양이 같다.
- [ ] 거부된 수정·삭제가 0행을 변경한다.
- [ ] 거부된 Check 생성이 행과 외부 요청을 만들지 않는다.
- [ ] 하위 리소스 직접 ID 접근도 부모 소유권을 우회하지 못한다.
- [ ] 동시 변경 중에도 owner 없는 쓰기가 발생하지 않는다.
- [ ] SQL 또는 저장소 호출에 owner 조건이 누락된 경로가 없다.

### 구현 후 설명할 것

1. 인증과 객체 단위 인가를 분리한 이유
   - 모범답변: 인증은 호출자의 신원만 증명하고 특정 Monitor를 읽거나 바꿀 권한은 증명하지 않습니다. 모든 자원 접근마다 별도 owner predicate가 필요합니다.
2. owner 조건을 SQL에 넣어 TOCTOU를 줄인 방식
   - 모범답변: JPQL UPDATE/DELETE와 SELECT가 `id`와 `ownerUserId`를 같은 statement에서 평가해 허용된 row만 반환·변경합니다. 사전 검사와 실행 사이 창을 없앱니다.
3. foreign과 nonexistent를 같은 404로 처리한 정보 노출 trade-off
   - 모범답변: 객체 존재 privacy를 높이는 대신 정당한 호출자에게 권한 부족과 오타를 구분해 주지 못합니다. 현재 개인 소유 모델에서는 비공개를 우선했습니다.
4. 하위 리소스를 부모 권한에 묶는 방법
   - 모범답변: CheckRun query에 Monitor를 join하고 `check.monitorId = monitor.id AND monitor.ownerUserId = :owner`를 포함합니다. 목록과 단건 모두 같은 경계를 씁니다.
5. 거부된 요청의 무부수효과를 검증하는 방법
   - 모범답변: 두 사용자 fixture로 foreign CRUD·Check를 호출하고 row/history snapshot과 endpoint invocation count가 전후 동일하며 foreign/absent 응답도 같은지 확인합니다.

### 원본 확인 위치

- Thread 05
- 커밋: `test(authz): freeze the two-user IDOR counterexample`
- `backend/src/main/resources/db/migration/V4__require_monitor_ownership.sql`
- `backend/src/main/java/dev/evolution/monitor/MonitorEntity.java`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java`
- `backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java`
- `tests/browser/ownership.spec.ts`
- 관련 Thread: Thread 03의 트랜잭션 경계, Thread 07의 owner-scoped history, Thread 10의 idempotency 소유자 결합

---

<!-- coverage: SA-07 -->
<a id="sa-07"></a>
## [Thread 05 / `test(authz): preserve E05 security gates and observation history`] CSRF 토큰과 정확한 Origin을 함께 검사하는 변경 경계

### 면접 질문

세션 쿠키 기반 애플리케이션에서 Spring Security의 CSRF 검사를 유지하면서, 변경 요청에 정확히 하나의 신뢰된 `Origin`도 요구한 이유는 무엇입니까? Origin 검사가 성공하면 CSRF 토큰 검사를 생략해도 됩니까?

꼬리 질문:

- `Origin: null`, Origin 헤더 없음, 여러 Origin 값은 어떻게 처리해야 합니까?
  - 모범답변: unsafe method에서는 모두 거부합니다. 원본은 header enumeration에 정확히 한 값이 있고 고정 origin과 동일할 때만 신뢰합니다.
- `Referer`, `Host`, `X-Forwarded-Host`, 문자열 suffix 비교를 신뢰하지 않은 이유는 무엇입니까?
  - 모범답변: Referer는 생략될 수 있고 Host/forwarded 값은 proxy 신뢰 설정에 좌우되며 suffix 비교는 `trusted.evil`을 통과시킬 수 있습니다. 배포의 정확한 scheme·host·port 문자열 하나와 비교합니다.
- CORS와 CSRF/Origin 검사는 어떤 문제를 각각 해결합니까?
  - 모범답변: CORS는 다른 origin JavaScript의 응답 읽기 권한이고, CSRF/Origin은 cookie가 자동 첨부된 변경 요청 위조를 막습니다. CORS를 끈 것만으로 request 전송을 막는다고 볼 수 없습니다.
- 인증되지 않은 변경 요청과 인증됐지만 Origin이 잘못된 요청의 공개 상태를 달리한 이유는 무엇입니까?
  - 모범답변: 전자는 유효한 주체가 없어 401이고 후자는 신원은 있으나 변경 요청 증거가 정책을 위반해 403입니다. Spring access denied handler도 같은 taxonomy를 유지합니다.

### 30초 모범 답변

세션 쿠키는 브라우저가 자동으로 보내므로 변경 요청은 CSRF 방어가 필요합니다. CSRF 토큰은 세션에 결합된 비밀 증거이고, 정확한 Origin은 요청이 허용된 브라우저 프런트엔드에서 왔다는 추가 증거입니다. 둘은 대체 관계가 아니므로 Origin이 맞아도 기본 CsrfFilter를 계속 통과해야 합니다. 신뢰 Origin은 하나의 고정 문자열과 정확히 일치해야 하고, 누락·`null`·복수 값·외부 Origin은 거부합니다. CORS는 다른 Origin의 응답 읽기 권한을 다루는 별도 정책이라 credentialed cross-origin 허용을 만들지 않았습니다.

### 답변 핵심 키워드

cookie authentication, CSRF token, exact Origin, defense in depth, no bypass, multiple headers, CORS separation, fail closed

### 백지 구현

#### 구현 목표

브라우저 변경 요청의 인증 상태와 Origin 형식을 먼저 검사하고, 이후 기존 CSRF 검사가 반드시 실행될 수 있도록 하는 필터 판단 로직을 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
enum OriginDecision { CONTINUE_TO_CSRF, UNAUTHENTICATED, FORBIDDEN }

static OriginDecision inspectMutation(
    String method,
    List<String> originValues,
    boolean authenticated,
    boolean corsPreflight,
    String trustedOrigin
) {
    if (corsPreflight) return OriginDecision.FORBIDDEN;
    boolean safe = method.equals("GET") || method.equals("HEAD")
            || method.equals("TRACE") || method.equals("OPTIONS");
    if (safe) return OriginDecision.CONTINUE_TO_CSRF;
    if (!authenticated) return OriginDecision.UNAUTHENTICATED;
    if (originValues == null || originValues.size() != 1
            || !trustedOrigin.equals(originValues.getFirst())) {
        return OriginDecision.FORBIDDEN;
    }
    // Origin은 추가 증거일 뿐, 호출자가 후속 CsrfFilter를 반드시 실행한다.
    return OriginDecision.CONTINUE_TO_CSRF;
}
```

#### 입력과 출력

- 입력: HTTP 메서드, 모든 Origin 헤더 값, 인증 여부, preflight 여부, 정확한 신뢰 Origin
- 출력: CSRF 검사로 계속 진행, 401, 403 중 하나

#### 반드시 만족해야 할 조건

- 안전한 읽기 메서드는 Origin 요구 없이 계속 진행할 수 있다.
- 변경 메서드는 Origin 값이 정확히 하나이고 신뢰 Origin과 완전히 같아야 한다.
- preflight 요청은 cross-origin 권한을 부여하지 않고 거부한다.
- Origin이 맞더라도 결과는 `CONTINUE_TO_CSRF`이며 성공이 아니다.
- 인증되지 않은 변경 요청은 정상 인증 실패 의미를 유지한다.

#### 경계 조건

- Origin 헤더 없음
- 빈 Origin, `null`
- 같은 값 두 개, 서로 다른 값 두 개
- `http://127.0.0.1:4323.evil.example`
- 포트·스킴이 다른 유사 Origin
- GET, HEAD, POST, PUT, DELETE, OPTIONS
- 인증됨/익명 조합

#### 실패 조건

- `contains`, `endsWith` 같은 문자열 부분 일치로 신뢰하지 않는다.
- Host나 forwarded 헤더로 신뢰 Origin을 즉석에서 만들지 않는다.
- Origin 검사 성공을 CSRF 성공으로 취급하지 않는다.
- CORS 응답 헤더를 추가해 cross-origin credential 요청을 허용하지 않는다.

#### 필요한 제약

- URI 정규화보다 고정된 배포 Origin의 정확한 비교에 집중한다.
- 구현 시간은 15분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 정상 동일 Origin 변경 요청이 CSRF 단계로 넘어간다.
- [ ] 정상 Origin이지만 CSRF 토큰이 없으면 최종 성공할 수 없다.
- [ ] 누락·`null`·복수·외부 Origin이 거부된다.
- [ ] suffix 공격 문자열이 거부된다.
- [ ] 익명 요청과 인증된 위조 요청의 상태 정책이 일관된다.
- [ ] 거부된 요청이 컨트롤러와 저장소에 도달하지 않는다.
- [ ] CORS 허용 헤더가 추가되지 않는다.

### 구현 후 설명할 것

1. CSRF와 Origin 검사를 중복이 아닌 서로 다른 증거로 본 이유
   - 모범답변: CSRF token은 현재 session에 결합된 비밀이고 Origin은 browser가 보고한 요청 출처입니다. 서로 다른 설정·공격 경로를 보완하므로 하나의 성공으로 다른 검사를 생략하지 않습니다.
2. 정확한 단일 Origin 비교가 필요한 이유
   - 모범답변: 복수 header 해석 차이와 suffix·scheme·port 혼동을 없애고, 고정 배포 origin 한 개만 허용하는 fail-closed 계약을 유지합니다.
3. Origin 필터를 CsrfFilter 앞에 둔 의도와 후속 검사를 유지한 방법
   - 모범답변: 명백한 foreign mutation을 먼저 거부해 controller에 닿지 않게 하되, 일치하면 `chain.doFilter`로 넘겨 기본 session-backed CsrfFilter가 계속 token을 검증합니다.
4. CORS와 변경 요청 위조 방어의 차이
   - 모범답변: CORS grant를 만들지 않아 cross-origin 응답 읽기를 막고, 별도의 Origin/CSRF 검사는 응답을 못 읽더라도 보낼 수 있는 위조 mutation 자체를 거부합니다.
5. 거부된 요청의 무변경성을 브라우저와 서버에서 검증하는 방법
   - 모범답변: 실제 browser origin 조합과 raw HTTP missing/multiple header fixture를 보내 401/403을 확인하고, controller·repository 호출 count와 DB snapshot·외부 invocation이 0인지 검증합니다.

### 원본 확인 위치

- Thread 05
- 커밋: `test(authz): preserve E05 security gates and observation history`
- `backend/src/main/java/dev/evolution/monitor/BrowserOriginFilter.java`
- `BrowserOriginFilter.TRUSTED_ORIGIN`, `BrowserOriginFilter.doFilterInternal`
- `backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java`
- `tests/browser/ownership.spec.ts`
- 관련 Thread: Thread 04의 세션·CSRF 수명주기, Thread 08의 동일 Origin 서버 렌더링 경계
