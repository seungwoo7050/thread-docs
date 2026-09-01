# 인증, 인가와 브라우저 변이 보호 면접 워크북

Thread 04·05의 인증 설계는 credential 생성·저장·폐기, 객체 수준 권한, 브라우저 요청 출처를 서로 다른 경계로 다룬다. 한 방어가 다른 방어를 대체한다고 설명하지 않고, 각 invariant와 실행 순서를 분리해서 준비한다.

## 문서 내 면접 포인트

- [P12 느린 password KDF, timing-safe 비교와 존재하지 않는 사용자 처리](#p12)
- [P13 고엔트로피 세션 토큰, strict cookie parsing과 저장 시 hash](#p13)
- [P14 세션 교체·폐기의 트랜잭션과 다중 인스턴스 lifecycle](#p14)
- [P15 권한 조건을 SQL predicate에 포함하고 존재 여부를 숨기는 경계](#p15)
- [P16 session-bound CSRF token, Origin 검사와 인증 선행 순서](#p16)

---

<a id="p12"></a>
## [Thread 04 (E04) / 제목 미노출 — 기록상 password hashing과 로그인 검증 구현] 느린 password KDF, timing-safe 비교와 존재하지 않는 사용자 처리

### 면접 질문

- 일반 SHA-256 대신 scrypt 같은 password KDF를 사용해야 하는 이유는 무엇입니까?
  - 꼬리 질문: 존재하지 않는 username에서도 dummy password hash를 검증한 이유는 무엇입니까?
  - 꼬리 질문: 저장된 hash 문자열이 손상됐거나 길이가 다를 때 `timingSafeEqual`을 안전하게 사용하려면 어떤 경계를 처리해야 합니까?

### 30초 모범 답변

비밀번호는 낮은 엔트로피라 빠른 hash를 쓰면 유출 뒤 대입 공격이 매우 싸집니다. scrypt는 salt와 메모리 비용을 사용해 각 추측 비용을 높이고, 이 프로젝트는 파라미터와 salt·key 길이를 저장 형식에 고정했습니다. username이 없을 때 즉시 반환하면 응답 시간으로 계정 존재를 추정할 수 있으므로 dummy hash에도 같은 KDF 작업을 수행합니다. 비교 전 저장 형식을 엄격히 파싱하고 길이를 맞춘 뒤 timing-safe 비교를 하며, 손상된 형식은 인증 실패로 처리하고 내부 예외를 노출하지 않습니다.

### 답변 핵심 키워드

- password KDF
- scrypt
- per-user salt
- memory-hard
- timingSafeEqual
- dummy hash
- username enumeration
- encoded format validation

### 백지 구현

#### 구현 목표

salt가 포함된 password hash 문자열을 만들고 검증하며, 사용자 존재 여부와 무관하게 비슷한 비용의 인증 경로를 제공한다.
#### 인터페이스 또는 함수 시그니처

`hashPassword(password, randomBytes): Promise<string>`, `verifyPassword(password, encoded): Promise<boolean>`, `authenticate(username, password, lookup): Promise<User | null>`
#### 입력과 출력

- 입력: password 문자열, 저장된 encoded hash 또는 사용자 조회 함수
- 출력: 새 encoded hash, 검증 boolean, 또는 인증된 공개 사용자
- 실패: 잘못된 저장 형식·KDF 오류는 credential 세부를 숨긴 인증 실패 또는 내부 오류
#### 반드시 만족해야 할 조건

- 각 새 hash에 독립적인 random salt를 사용한다.
- 알고리즘 파라미터와 salt·derived key를 명시적으로 인코딩한다.
- 검증 전 형식과 길이를 엄격히 확인한다.
- derived key 비교는 timing-safe 방식으로 수행한다.
- 사용자가 없을 때도 고정된 dummy encoded hash로 KDF를 한 번 수행한다.
- 로그나 오류에 password, salt 전체, derived key를 기록하지 않는다.
#### 경계 조건

- 빈 password와 매우 긴 password에 대한 명시적 입력 정책
- 손상된 base64url, 잘못된 필드 수, 잘못된 길이
- 같은 password를 두 번 hash했을 때 서로 다른 encoded 결과
- 없는 사용자와 틀린 password의 외부 오류가 동일한 경우
#### 실패 조건

- 빠른 일반 hash를 password 저장에 사용한다.
- 사용자 없음 경로만 KDF를 건너뛴다.
- 길이가 다른 Buffer를 바로 timing-safe 비교해 예외를 낸다.
- 서버에 과도한 KDF 동시 요청이 들어올 때 자원 한계를 전혀 고려하지 않는다.
#### 필요한 제약

- 프로젝트 기록의 scrypt 파라미터 계약을 변경하지 않고 인터페이스만 구현한다.
- 성능 측정으로 보안을 증명한다고 주장하지 않고, 파라미터 변경은 별도 migration 전략이 필요함을 설명한다.

```ts
export async function hashPassword(
  password: string,
  randomBytes: (length: number) => Uint8Array,
): Promise<string> {
  // 직접 구현
}

export async function verifyPassword(
  password: string,
  encoded: string,
): Promise<boolean> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 같은 password의 두 hash가 서로 다르지만 둘 다 검증된다.
- 틀린 password, 손상된 형식, 잘못된 길이는 false가 된다.
- 없는 사용자와 존재하는 사용자의 틀린 password가 같은 외부 오류를 반환한다.
- 두 경로 모두 KDF 호출 횟수가 같다.
- 비밀 값이 로그·예외·반환 객체에 포함되지 않는다.
- 동시 검증 시 event loop와 메모리 비용에 대한 운영 제한을 설명할 수 있다.

### 구현 후 설명할 것

- 일반 hash와 password KDF의 위협 모델 차이
- salt와 pepper의 차이 및 현재 구현 범위
- dummy hash가 막는 정보와 완전히 동일한 timing을 보장하지는 못하는 이유
- KDF 파라미터 업그레이드와 재해시 전략

### 원본 확인 위치

- Thread 04
- `server/password.ts` — `SCRYPT_OPTIONS`, `hashPassword`, `verifyPassword`, `DUMMY_PASSWORD_HASH`
- `server/contracts.ts` — 로그인 입력 검증
- `test/unit.test.ts`
- 관련 Thread: 14(E24) 비밀 값 로그 금지
---

<a id="p13"></a>
## [Thread 04 (E04) / 제목 미노출 — 기록상 session token과 cookie lifecycle 구현] 고엔트로피 세션 토큰, strict cookie parsing과 저장 시 hash

### 면접 질문

- 세션 token 원문을 DB에 저장하지 않고 SHA-256 hash만 저장한 이유는 무엇입니까?
  - 꼬리 질문: 같은 이름의 session cookie가 두 개 있을 때 첫 값이나 마지막 값을 선택하지 않고 요청을 거부한 이유는 무엇입니까?
  - 꼬리 질문: HttpOnly, SameSite=Lax, Path=/ 속성과 loopback 개발 환경에서 Secure를 생략한 판단을 설명해 보세요.

### 30초 모범 답변

세션 token은 충분한 엔트로피를 가진 bearer credential이므로 DB 유출 시 원문이 있으면 즉시 계정 탈취로 이어집니다. 브라우저에는 raw token을 보내되 DB에는 deterministic hash만 저장해 요청마다 hash로 lookup합니다. 중복 cookie는 프록시·프레임워크마다 선택 규칙이 달라 해석 모호성이 생기므로 정확히 하나의 엄격한 base64url token만 허용합니다. HttpOnly는 JavaScript 노출을 줄이고 SameSite=Lax는 기본 cross-site 전송을 제한합니다. Secure는 HTTPS가 없는 loopback 개발 환경 때문에 생략했지만 production 정책으로 일반화하면 안 됩니다.

### 답변 핵심 키워드

- bearer token
- 32 random bytes
- store hash not raw
- strict base64url
- duplicate cookie rejection
- HttpOnly
- SameSite
- expiry

### 백지 구현

#### 구현 목표

Cookie 헤더에서 유효한 session token 하나만 추출하고, 저장·조회용 hash를 계산한다.
#### 인터페이스 또는 함수 시그니처

`sessionTokenFromCookie(header): string | null`, `sessionTokenHash(raw): string`
#### 입력과 출력

- 입력: 없거나 임의 형식인 Cookie 헤더
- 출력: 엄격히 검증된 raw token 하나 또는 `null`, 그리고 고정 길이 hex hash
- 실패: 중복·비정규 token은 인증되지 않은 상태로 처리
#### 반드시 만족해야 할 조건

- 대상 cookie가 정확히 하나일 때만 token을 반환한다.
- token은 예상 길이의 canonical base64url이어야 한다.
- 빈 값, padding, 잘못된 문자는 거부한다.
- DB lookup에는 raw token이 아니라 hash만 사용한다.
- 만료 시간은 DB query에서 현재 시각보다 미래인 session만 허용한다.
- 공개 사용자에는 id와 username만 포함한다.
#### 경계 조건

- Cookie 헤더 없음
- 대상 cookie 없음
- 대상 cookie 두 개와 서로 다른 값
- 다른 cookie 사이의 공백과 빈 segment
- 올바른 길이지만 noncanonical encoding
- 정확히 만료 시각과 같은 session
#### 실패 조건

- 중복 cookie 중 임의 하나를 선택한다.
- raw token을 로그·DB·클라이언트 prop에 복제한다.
- 만료 검사를 애플리케이션 메모리 cache에만 의존한다.
- 쿠키 속성을 환경과 무관하게 안전하다고 단정한다.
#### 필요한 제약

- parser는 일반 cookie 라이브러리 전체 기능이 아니라 현재 session cookie 계약만 다룬다.
- hash는 token entropy를 대체하지 않으며 random token 생성과 별도 책임이다.

```ts
export function sessionTokenFromCookie(
  header: string | undefined,
): string | null {
  // 직접 구현
}

export function sessionTokenHash(rawToken: string): string {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 정확히 하나의 정상 token만 반환한다.
- 없음·중복·padding·잘못된 문자·길이를 모두 거부한다.
- 같은 token은 같은 hash, 다른 token은 다른 hash가 된다.
- raw token이 저장 row·로그·오류에 남지 않는다.
- 만료 전은 인증되고 정확한 만료 시각 이후는 인증되지 않는다.
- 다른 애플리케이션 인스턴스에서도 같은 DB session이 유효하다.

### 구현 후 설명할 것

- random token과 password의 저장 전략이 다른 이유
- 세션 token hash에 느린 KDF가 필수는 아닌 이유
- 중복 cookie를 모호한 입력으로 본 이유
- 개발용 cookie 속성과 production HTTPS 속성의 차이

### 원본 확인 위치

- Thread 04
- `server/auth.ts` — `SESSION_COOKIE_NAME`, `SESSION_TTL_MS`, `sessionTokenFromCookie`, `sessionTokenHash`, `registerAuthentication`
- `server/migrations/003_sessions.sql`
- `server/model.ts` — 공개 User
- 관련 Thread: 05(E05) session-bound CSRF, 08(E08) raw Cookie forwarding
---

<a id="p14"></a>
## [Thread 04 (E04) / 제목 미노출 — 기록상 로그인 session rotation과 logout 구현] 세션 교체·폐기의 트랜잭션과 다중 인스턴스 lifecycle

### 면접 질문

- 로그인 시 기존 session 삭제와 새 session 삽입을 한 transaction으로 묶은 이유는 무엇입니까?
  - 꼬리 질문: 새 session INSERT가 실패했는데 기존 session 삭제가 commit되면 사용자 경험과 보안에 어떤 문제가 생깁니까?
  - 꼬리 질문: logout을 메모리 상태 초기화만으로 처리하지 않고 DB hash 삭제와 만료 cookie를 함께 수행한 이유는 무엇입니까?

### 30초 모범 답변

세션 회전은 기존 credential을 폐기하고 새 credential을 발급하는 하나의 상태 전환입니다. 두 작업을 별도 autocommit으로 처리하면 새 session 생성 실패 시 사용자의 유효 session만 사라지거나, 반대로 둘 다 살아 rotation 목적을 잃을 수 있습니다. 한 explicit connection의 transaction에서 삭제와 삽입을 묶고 실패 시 rollback합니다. logout은 DB row를 삭제해 모든 인스턴스에서 즉시 무효화하고, 브라우저에는 같은 이름·Path의 만료 cookie를 보내 bearer token도 제거합니다.

### 답변 핵심 키워드

- session rotation
- atomic replace
- explicit transaction
- rollback
- server-side revocation
- cookie clearing
- multi-instance
- logout idempotency

### 백지 구현

#### 구현 목표

기존 session을 선택적으로 폐기하고 새 session을 원자적으로 발급하며, logout 시 저장소와 cookie를 함께 정리하는 service를 작성한다.
#### 인터페이스 또는 함수 시그니처

`rotateSession(db, input): Promise<NewSession>`, `logoutSession(db, tokenHash): Promise<void>`
#### 입력과 출력

- 입력: user ID, 이전 token hash 또는 없음, 새 token hash, 생성·만료 시각
- 출력: 브라우저에 설정할 새 raw token과 만료 정보는 호출자가 별도 조합
- logout 출력: row 존재 여부와 무관한 성공
#### 반드시 만족해야 할 조건

- rotation의 기존 삭제와 새 INSERT가 같은 transaction·connection에 있다.
- 새 INSERT 실패 시 기존 session 상태가 복원된다.
- 새 raw token은 transaction 성공 전 외부 성공 응답으로 확정하지 않는다.
- logout은 token hash row를 삭제하고 cookie clear 응답을 항상 만들 수 있다.
- 모든 DB client가 성공·실패에서 release된다.
- 세션 상태는 프로세스 메모리가 아니라 PostgreSQL이 권위 원천이다.
#### 경계 조건

- 이전 session이 이미 없는 로그인
- 새 token hash 충돌
- DELETE 성공 후 INSERT 실패
- logout token이 이미 만료·삭제된 경우
- 응답 전달이 유실됐지만 transaction은 commit된 경우
#### 실패 조건

- 회전 중간 상태를 commit한다.
- raw token을 DB에 저장한다.
- logout이 현재 API 인스턴스의 cache만 지운다.
- cookie clear의 Path가 발급 cookie와 달라 브라우저에 값이 남는다.
#### 필요한 제약

- token 생성과 DB transaction을 분리하되, 외부 성공 의미는 commit 이후로 둔다.
- 분산 transaction은 사용하지 않고 단일 PostgreSQL 내 원자성을 범위로 한다.

```ts
export async function rotateSession(
  db: SessionDatabase,
  input: RotationInput,
): Promise<RotationResult> {
  // 직접 구현
}

export async function logoutSession(
  db: SessionDatabase,
  tokenHash: string,
): Promise<void> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 정상 rotation 후 이전 hash는 없고 새 hash 하나만 남는다.
- INSERT 실패를 주입하면 이전 hash가 그대로 남는다.
- client release가 성공·rollback 경로 모두 한 번이다.
- logout을 두 번 호출해도 외부 결과가 안정적이다.
- 다른 API 인스턴스에서 이전 token이 즉시 인증되지 않는다.
- cookie clear 속성이 발급 cookie의 이름과 Path를 정확히 덮는다.

### 구현 후 설명할 것

- 세션 회전을 원자적 replace로 본 이유
- transaction commit과 HTTP 응답 전달 사이의 ack-loss 문제
- logout의 server-side revocation과 client-side deletion을 둘 다 하는 이유
- 동시에 여러 로그인 session을 허용할지 한 session만 허용할지의 정책 차이

### 원본 확인 위치

- Thread 04
- `server/auth.ts` — 로그인·logout과 `registerAuthentication`
- `server/migrations/003_sessions.sql`
- `server/database.ts` — explicit PostgreSQL connection
- 관련 Thread: 10(E10) commit 뒤 응답 유실과 멱등성
---

<a id="p15"></a>
## [Thread 05 (E05) / 제목 미노출 — 기록상 Monitor ownership과 IDOR 방어 구현] 권한 조건을 SQL predicate에 포함하고 존재 여부를 숨기는 경계

### 면접 질문

- `SELECT monitor WHERE id=?` 뒤 애플리케이션에서 owner를 비교하는 방식보다 `WHERE id=? AND owner_user_id=?`가 나은 이유는 무엇입니까?
  - 꼬리 질문: 다른 사용자의 Monitor와 존재하지 않는 Monitor를 모두 404로 반환한 이유는 무엇입니까?
  - 꼬리 질문: 직접 CheckRun 조회와 Monitor 하위 history 조회 모두에서 owner predicate가 필요한 이유를 설명해 보세요.

### 30초 모범 답변

권한 검사를 SQL predicate에 포함하면 허용되지 않은 row가 애플리케이션 메모리로 나오지 않고, 조회와 변경을 한 원자적 조건으로 묶을 수 있습니다. 먼저 존재를 확인한 뒤 권한을 검사하면 race가 생기고, 403과 404 차이로 다른 사용자의 자원 존재를 노출할 수 있습니다. 그래서 owner 조건을 만족한 UPDATE·DELETE·SELECT의 row가 없으면 foreign과 absent를 같은 404로 처리합니다. CheckRun ID를 직접 아는 경로도 부모 Monitor owner와 join해 검증해야 nested route 우회가 없습니다.

### 답변 핵심 키워드

- IDOR
- object-level authorization
- owner predicate
- single-statement authorization
- 404 indistinguishability
- nested/direct access
- no side effect
- TOCTOU 회피

### 백지 구현

#### 구현 목표

owner 범위를 벗어난 row를 읽거나 수정하지 않는 repository 메서드와 HTTP 오류 경계를 작성한다.
#### 인터페이스 또는 함수 시그니처

`updateMonitorForOwner(db, ownerId, monitorId, patch): Promise<Monitor | null>`, `getCheckForOwner(db, ownerId, checkId): Promise<CheckRun | null>`
#### 입력과 출력

- 입력: 인증된 owner ID, 자원 ID, 변경 값
- 출력: owner가 가진 자원 또는 `null`
- HTTP 계층: `null`을 foreign/absent 구분 없이 404로 변환
#### 반드시 만족해야 할 조건

- SELECT·UPDATE·DELETE에 owner 조건을 같은 statement에 포함한다.
- 직접 CheckRun 조회는 부모 Monitor와 owner를 연결한다.
- foreign 자원 요청에서 write row 수와 outbound 호출 수가 0이다.
- collection은 현재 owner row만 반환한다.
- 응답과 로그에 다른 사용자의 자원 존재 단서를 남기지 않는다.
#### 경계 조건

- 형식이 잘못된 UUID와 형식은 맞지만 없는 UUID
- 다른 사용자의 Monitor ID
- 다른 사용자의 CheckRun ID를 직접 조회
- owner가 맞지만 부모가 삭제된 직후 요청
- 동시에 ownership 또는 row가 변경되는 상황
#### 실패 조건

- 존재 확인 SELECT와 권한 확인을 별도 statement로 나눈다.
- foreign에 403, absent에 404를 반환해 존재를 구분한다.
- 하위 route만 보호하고 direct route를 놓친다.
- 권한 실패 뒤 외부 Check를 실행한다.
#### 필요한 제약

- SQL 값은 모두 parameter binding을 사용한다.
- 관리자·공유 기능은 현재 범위 밖이며, 추가 시 정책을 별도 모델링한다.

```ts
export async function updateMonitorForOwner(
  db: Database,
  ownerId: string,
  monitorId: string,
  patch: MonitorPatch,
): Promise<Monitor | null> {
  // 직접 구현
}

export async function getCheckForOwner(
  db: Database,
  ownerId: string,
  checkId: string,
): Promise<CheckRun | null> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- owner는 collection/detail/update/delete/history/direct CheckRun을 정상 사용한다.
- foreign과 absent는 동일한 status와 공개 오류 형태다.
- foreign update/delete 뒤 row와 history가 변하지 않는다.
- foreign manual check에서 queue row와 endpoint 요청이 생기지 않는다.
- malformed ID는 존재 조회 전에 입력 오류로 분리된다.
- 동시 삭제 상황에서도 권한 없는 row가 반환되지 않는다.

### 구현 후 설명할 것

- SQL predicate 권한 검사와 애플리케이션 후검사의 차이
- 404로 존재를 숨기는 정책의 장단점
- nested route와 direct route를 함께 검증해야 하는 이유
- owner 컬럼 backfill과 NOT NULL 전환 시 필요한 migration 판단

### 원본 확인 위치

- Thread 05
- `server/app.ts` — owner-scoped Monitor·CheckRun SQL
- `server/migrations/004_monitor_ownership.sql`
- `server/schema.ts` — ownership schema 계약
- `test/contracts.test.ts`, `test/browser/lifecycle.spec.ts`
- 관련 Thread: 07(E07) cursor continuation owner 검사, 08(E08) SSR owner privacy
---

<a id="p16"></a>
## [Thread 05 (E05) / `Require session-bound CSRF evidence for browser mutations`] session-bound CSRF token, Origin 검사와 인증 선행 순서

### 면접 질문

- SameSite cookie만 믿지 않고 session-bound CSRF token과 exact Origin을 함께 검사한 이유는 무엇입니까?
  - 꼬리 질문: CSRF token을 React state나 localStorage에 보관하지 않고 mutation 직전에 읽은 이유는 무엇입니까?
  - 꼬리 질문: 익명 요청은 401, 인증됐지만 CSRF가 틀린 요청은 403이 되도록 인증을 먼저 수행한 이유는 무엇입니까?

### 30초 모범 답변

SameSite는 중요한 방어지만 브라우저 정책·navigation 유형·배포 구성이 바뀔 수 있어 단독 경계로 두지 않았습니다. raw session identifier를 서버 비밀로 HMAC한 token을 요청마다 제시하게 하고, unsafe method는 exact Origin도 확인해 cookie만 자동 전송된 cross-site 요청을 막습니다. token은 mutation 직전에 받아 request-local로 사용해 session 교체 뒤 stale token과 장기 저장 노출을 줄입니다. 인증을 먼저 해 익명은 401로 일관되게 처리하고, 그 다음 Origin·CSRF를 검사한 뒤 body parsing·SQL·outbound를 시작합니다.

### 답변 핵심 키워드

- CSRF
- SameSite defense in depth
- HMAC
- session binding
- exact Origin
- request-local token
- 401 before 403
- pre-body guard
- CORS deny

### 백지 구현

#### 구현 목표

인증된 session에 결박된 CSRF token을 생성·검증하고, unsafe 요청의 실행 순서를 강제하는 guard를 작성한다.
#### 인터페이스 또는 함수 시그니처

`csrfTokenForSession(rawSession, secret): string`, `authorizeMutation(request, session, secret, browserOrigin): void`
#### 입력과 출력

- 입력: raw session identifier, 서버 secret, Origin, CSRF header, HTTP method
- 출력: 허용 시 없음, 익명·정책 위반 시 안정된 401 또는 403
- 후속 handler는 guard 통과 뒤에만 body parsing·DB·외부 I/O를 수행
#### 반드시 만족해야 할 조건

- token은 session identifier와 versioned context에 결박된 HMAC이어야 한다.
- 비교 전 길이·encoding을 검사하고 timing-safe 비교를 사용한다.
- unsafe method와 logout은 exact Origin과 CSRF header를 요구한다.
- 다른 session의 token, 누락·중복·과도한 길이 token을 거부한다.
- 인증 실패가 Origin·CSRF 실패보다 먼저 결정된다.
- 거부 경로에서 body parser, SQL, outbound 호출이 0이다.
- CORS 응답으로 cross-origin credential 접근을 허용하지 않는다.
#### 경계 조건

- Origin 없음, `null`, 대소문자·port가 다른 Origin
- 현재 session과 이전 session의 token
- 정상 prefix를 가진 oversized token
- GET과 unsafe POST의 차이
- preflight OPTIONS 요청
#### 실패 조건

- CSRF token을 session cookie와 동일한 값으로 사용한다.
- 비밀 token을 localStorage·URL·로그에 저장한다.
- body parsing 뒤에 guard를 실행한다.
- wildcard CORS와 credential을 함께 허용한다.
#### 필요한 제약

- 현재 브라우저 origin 하나의 exact-match 정책을 구현한다.
- XSS를 CSRF token으로 막는다고 주장하지 않는다.

```ts
export function csrfTokenForSession(
  rawSession: string,
  secret: Uint8Array,
): string {
  // 직접 구현
}

export function authorizeMutation(
  request: MutationRequest,
  session: AuthenticatedSession | null,
  secret: Uint8Array,
  browserOrigin: string,
): void {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 같은 session은 검증되고 다른 session token은 거부된다.
- 누락·malformed·oversized token과 잘못된 Origin이 403이다.
- 익명 요청은 다른 header 상태와 무관하게 401이다.
- 거부 요청의 body parser·SQL·outbound counters가 0이다.
- 로그인과 logout의 Origin 정책을 각각 확인한다.
- 응답에 credentialed CORS 허용 header가 없다.
- token이 React state, URL, browser storage, 서버 로그에 남지 않는다.

### 구현 후 설명할 것

- SameSite, Origin, CSRF token을 겹쳐 둔 이유
- HMAC token과 random stored token 방식의 trade-off
- 인증→Origin/CSRF→parse→SQL 순서를 택한 이유
- CSRF와 XSS의 위협 모델 차이

### 원본 확인 위치

- Thread 05
- `server/auth.ts` — `csrfTokenForSession`, `validCsrfToken`, `registerAuthentication`의 인증·Origin·unsafe method·OPTIONS 경계
- `app/monitors/api.ts` — `mutationFetch`
- `test/unit.test.ts`, `test/contracts.test.ts`, `test/browser/lifecycle.spec.ts`
- 관련 Thread: 04(E04) session token, 06(E06) mutation admission
