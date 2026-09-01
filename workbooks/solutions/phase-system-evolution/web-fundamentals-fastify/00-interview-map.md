# DevThread 개발자 기술면접 마스터 인덱스

이 문서는 현재 프로젝트에 축적된 01~14 Thread 작업 기록만을 기준으로 만든 전체 작업 지도다. 상세 워크북 작성과 완전성 검증은 아래 P01~P28 식별자를 기준으로 수행했다. 원격 저장소나 프로젝트 밖의 구현은 전제로 삼지 않았다.

## 판정 기준과 표기 규칙

- **S**: 질문과 10~30분 백지 구현 모두 가치가 높아 반드시 준비한다.
- **A**: 준비 가치가 높으며 질문 또는 핵심 축소 구현 가능성이 높다.
- **B**: 별도 구현보다 설계 동기·trade-off·테스트 경계 설명에 적합하다.
- **C**: 회귀 증거, 설정, 반복 소비자처럼 독립 면접 항목으로 만들 필요가 낮다.
- backtick으로 표시한 커밋 메시지는 프로젝트 기록에서 실제 제목을 확인한 경우다. **"제목 미노출 — 기록상 …"은 현재 메모리에서 제목 문자열을 확인하지 못해 작업 내용을 설명한 표기이며 실제 커밋 제목이라고 주장하지 않는다.**
- 관련 위치는 현재 기록에서 이름을 확인한 파일·함수·컴포넌트만 적었다. 위치가 확인되지 않은 세부 구현명은 만들지 않았다.

선별 결과는 S 15개, A 13개, B 15개, C 14개 행이다. S/A의 독립 면접 포인트는 총 28개다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S | 01 (E01) | `메모리 Monitor에서 동기 fixture Check를 실행` | `server/check.ts` — `checkMonitor`, `fixtureUrl`; `server/main.ts` | [P01 비동기 HTTP Check의 단일 종료와 자원 정리](01-http-contracts-and-outbound-io.md#p01) | 응답·timeout·error 경쟁에서 결과를 한 번만 확정하고 request·response·timer를 정리하는 일반적인 비동기 I/O 문제다. | 높음 | 높음 | 12 |
| B | 01 (E01) | 제목 미노출 — 기록상 메모리 Monitor CRUD와 최초 Next UI 구현 | `server/app.ts` — `buildApp`, process-local `Map`; `app/monitors/page.tsx` | process memory 상태와 최초 UI lifecycle | 후속 Thread 03·06·08에서 더 강한 형태로 대체됐다. 상태 수명 설명에는 유용하지만 별도 구현 문제 가치는 낮다. | 중간 | 낮음 | 03·06·08 |
| C | 01 (E01) | `E01 기본 검증을 CI에 연결` | `.github/workflows/check.yml` | 기본 type·unit·functional·browser gate | 중요한 품질 기반이지만 설정 연결 자체는 면접 기본기 판별력이 낮다. | 낮음 | 없음 | 전 Thread |
| A | 02 (E02) | 제목 미노출 — 기록상 런타임 입력·오류 계약 구현 | `server/contracts.ts` — `ApiError`, `monitorInput`, `monitorId`, `ERROR_STATUS`; `server/app.ts` | [P02 `unknown` 입력의 런타임 검증과 안정적인 오류 분류](01-http-contracts-and-outbound-io.md#p02) | TypeScript 타입 밖의 HTTP 입력을 도메인 값으로 좁히고 parser·media type·body size 오류를 같은 계약으로 수렴시키는 경계 설계다. | 높음 | 높음 | 01·03·04·05·10 |
| A | 02 (E02) | 제목 미노출 — 기록상 브라우저 응답 decoder 구현 | `app/monitors/api.ts` — `RequestFailure`, `responseData`, `failureCode` | [P03 신뢰하지 않는 API 응답의 방어적 디코딩](01-http-contracts-and-outbound-io.md#p03) | HTTP status와 envelope·오류 code를 함께 검증하고 malformed 응답을 안전한 실패로 접는 클라이언트 경계 문제다. | 높음 | 높음 | 04·05·06 |
| S | 03 (E03) | `Introduce checked PostgreSQL schema migrations` | `server/migrate.ts` — `migrationFiles`, `checkMigrationHistory`, `migrate`; `server/database.ts` | [P07 체크섬을 포함한 순차 마이그레이션과 단일 트랜잭션](02-database-consistency-and-pagination.md#p07) | append-only 이력, 정확한 prefix, checksum, 단일 connection·transaction으로 반쪽 적용을 막는 데이터 정합성 핵심이다. | 높음 | 높음 | 04·05·07·09·10·11·12·13 |
| A | 03 (E03) | `Reject unexpected PostgreSQL columns before API startup` | `server/schema.ts` — `verifySchema`; `server/migrate.ts` | [P08 애플리케이션 시작 전 실제 DB schema fail-fast 검증](02-database-consistency-and-pagination.md#p08) | migration history와 실제 catalog 상태의 차이를 감지해 잘못된 인스턴스가 listen하기 전에 실패시키는 계약 검증이다. | 높음 | 중간 | 05·09·10·11·12 |
| A | 03 (E03) | `Make PostgreSQL authoritative for Monitor lifecycle` | `server/mapping.ts` — `monitorFromRow`, `checkRunFromRow`, `monitorToValues`, `checkRunToValues`; `server/app.ts` | [P09 명시적 row mapping과 PostgreSQL 단일 진실 원천](02-database-consistency-and-pagination.md#p09) | snake_case·Date·null·0·false를 명시적으로 변환하고 process memory가 아닌 DB를 CRUD와 history의 권위로 둔다. | 높음 | 높음 | 01·04·05·08 |
| B | 03 (E03) | `Classify unpersistable NUL Monitor names as invalid input` | `server/contracts.ts`; `server/app.ts` | DB가 저장할 수 없는 문자열을 외부 입력 오류로 선분류 | 좋은 경계 사례지만 NUL 한 종류에 특수하다. P02의 런타임 검증과 P09의 DB mapping 설명에 통합한다. | 중간 | 낮음 | 02 |
| C | 03 (E03) | 제목 미노출 — 기록상 PostgreSQL lifecycle 브라우저 회귀 검증 | `test/browser/lifecycle.spec.ts`; `test/persistence.test.ts` | CRUD·restart·cascade 통합 검증 | 검증 대상은 P07~P09에 포함되며 테스트 소비자 자체를 별도 면접 항목으로 만들 필요는 낮다. | 낮음 | 없음 | 01·02 |
| S | 04 (E04) | 제목 미노출 — 기록상 salted scrypt 비밀번호 저장 구현 | `server/password.ts` — `SCRYPT_OPTIONS`, `hashPassword`, `verifyPassword`, `DUMMY_PASSWORD_HASH` | [P12 비밀번호 KDF, 독립 salt, constant-time 비교와 계정 존재 timing 완화](03-authentication-authorization-and-csrf.md#p12) | 암호 저장 표현, 비용 고정, malformed hash, missing-user dummy work를 함께 다루는 보안·런타임 기본기다. | 높음 | 높음 | 05 |
| S | 04 (E04) | 제목 미노출 — 기록상 session cookie·token lifecycle 구현 | `server/auth.ts` — `sessionTokenFromCookie`, `sessionTokenHash`, `registerAuthentication`, cookie 상수; `server/migrations/003_sessions.sql` | [P13 고엔트로피 session token, hash 저장, 엄격한 cookie parser와 만료](03-authentication-authorization-and-csrf.md#p13) | 원문 token 비저장, duplicate cookie 거부, exact expiry, HttpOnly/SameSite 범위를 연결하는 인증 경계다. | 높음 | 높음 | 05·08 |
| S | 04 (E04) | 제목 미노출 — 기록상 session rotation과 logout transaction 구현 | `server/auth.ts` — login rotation·logout; `server/migrations/003_sessions.sql` | [P14 세션 교체·폐기의 트랜잭션과 수명주기 invariant](03-authentication-authorization-and-csrf.md#p14) | 기존 session 폐기와 신규 session 삽입을 한 connection transaction으로 묶어 다중 유효 세션·무세션 반쪽 상태를 제어한다. | 높음 | 높음 | 08 |
| C | 04 (E04) | `Freeze the E04 anonymous session counterexample` | `evidence/E04/baseline-reproducer.mjs`; `evidence/E04/scenario.json` | 인증 도입 전 anonymous baseline 고정 | 재현 절차는 유용하지만 핵심 역량은 이후 session 구현 P12~P14다. | 낮음 | 없음 | 04 |
| B | 04 (E04) | `Connect browser sign-in to the session lifecycle` | `app/login/page.tsx`; `test/browser/session.ts`; `test/browser/lifecycle.spec.ts` | 브라우저 로그인·로그아웃과 document 교체 | account 전환 경계는 중요하지만 상세 판단은 P14와 P20에 통합된다. 폼 연결 자체는 framework 비중이 높다. | 중간 | 낮음 | 08 |
| C | 04 (E04) | `Record E04 session lifecycle verification` | `evidence/E04/verification.json`; 관련 test 파일 | 세션 수명주기 검증 기록 | 검증 증거 고정이며 독립 구현 주제는 P12~P14에 이미 있다. | 낮음 | 없음 | 04 |
| S | 05 (E05) | 제목 미노출 — 기록상 Monitor ownership와 IDOR 차단 구현 | `server/migrations/004_monitor_ownership.sql`; `server/app.ts` — Monitor·CheckRun owner predicate; `server/schema.ts` | [P15 권한 조건을 SQL predicate에 포함하고 foreign·absent를 404로 통합](03-authentication-authorization-and-csrf.md#p15) | 조회 후 애플리케이션 권한 검사보다 TOCTOU와 정보 노출을 줄이고 모든 nested/direct mutation에 같은 authority invariant를 적용한다. | 높음 | 높음 | 03·04·07·10 |
| S | 05 (E05) | `Require session-bound CSRF evidence for browser mutations` | `server/auth.ts` — `registerAuthentication`, CSRF token 생성·검증, exact Origin·unsafe method·CORS 경계; `app/monitors/api.ts` — `mutationFetch` | [P16 session-bound CSRF·Origin·CORS와 검증 순서](03-authentication-authorization-and-csrf.md#p16) | SameSite만 의존하지 않고 token과 exact Origin을 결합하며, 인증→CSRF→body parse→SQL 순서로 부작용 없는 거부를 보장한다. | 높음 | 높음 | 04·06·08 |
| C | 05 (E05) | 제목 미노출 — 기록상 E05 ownership·CSRF counterexample 고정 | `evidence/E05/` | 두 사용자 권한·browser mutation baseline | 테스트 자료이며 핵심 권한·CSRF 설계는 P15·P16에 포함된다. | 낮음 | 없음 | 04 |
| C | 05 (E05) | 제목 미노출 — 기록상 E05 권한·CSRF 검증 증거 기록 | `test/contracts.test.ts`; `test/browser/`; `evidence/E05/` | 권한 matrix와 거부 부작용 검증 | 중요한 회귀 증거지만 별도 면접 문제보다 P15·P16의 자가 검증 조건으로 쓰는 편이 낫다. | 낮음 | 없음 | 04·06 |
| S | 06 (E06) | 제목 미노출 — 기록상 단일 server-state owner와 mutation admission 구현 | `app/monitors/use-monitors.ts` — `useMonitors`, `pending`; `evidence/E06/browser-scenario.ts` | [P17 첫 `await` 전 동기식 중복 mutation admission](04-client-state-and-server-rendering.md#p17) | disabled UI는 programmatic submit을 막지 못한다. 같은 JS turn에서 공유 pending 집합을 먼저 갱신해야 중복 네트워크 요청을 막는다. | 높음 | 높음 | 05·10 |
| A | 06 (E06) | 제목 미노출 — 기록상 server-state cache·mutation invalidation 구현 | `app/monitors/use-monitors.ts` — `lifetime`, `historyVersions`, mutation·history 상태 | [P18 generation·query version으로 늦은 비동기 응답 무효화](04-client-state-and-server-rendering.md#p18) | logout·delete·filter 변경 뒤 도착한 오래된 응답이 권위 있는 새 상태를 되살리지 못하게 하고 mutation ack와 refresh 실패를 분리한다. | 높음 | 높음 | 07·08·09 |
| B | 06 (E06) | `test: freeze repeated-submit response barrier` | `evidence/E06/browser-scenario.ts`; `evidence/E06/scenario.json` | commit 뒤 response를 보류하는 deterministic race test | sleep 없이 경쟁 지점을 고정하는 테스트 기법은 설명 가치가 있지만 구현 역량은 P17·P18에 통합된다. | 중간 | 낮음 | 10 |
| S | 07 (E07) | 제목 미노출 — 기록상 CheckRun cursor history 구현 | `server/history.ts` — `historyQuery`, `historyCursor`; `server/app.ts`; `server/migrations/005_check_history_index.sql` | [P10 조건에 결박된 cursor와 안정적인 seek pagination](02-database-consistency-and-pagination.md#p10) | 동률 정렬키, 삽입 중 continuation, canonical token, look-ahead row, owner scope를 함께 만족하는 자료구조·SQL 문제다. | 높음 | 높음 | 05·13 |
| A | 07 (E07) | 제목 미노출 — 기록상 history URL state와 stale response 방지 구현 | Thread 07 당시 `app/monitors/page.tsx` — `useSearchParams`, history navigation; 현재 `app/monitors/monitor-workspace.tsx`; `app/monitors/initial-state.ts` — `historyLocation`; `app/monitors/use-monitors.ts` — query identity | [P19 서버 query 조건을 URL과 cache identity에 일치시키기](04-client-state-and-server-rendering.md#p19) | monitor·state·limit·cursor를 URL에서 복원하고 filter 변경 시 cursor를 제거하며 같은 query string만 같은 cache data로 취급한다. | 높음 | 중간 | 06·08 |
| B | 07 (E07) | 제목 미노출 — 기록상 tied-row cursor counterexample 고정 | `evidence/E07/scenario.json`; `evidence/E07/fixture.ts` | 동일 timestamp row와 continuation test 설계 | 경계 조건 선택은 좋지만 핵심 자료구조·SQL은 P10에 포함된다. | 중간 | 낮음 | 13 |
| C | 07 (E07) | 제목 미노출 — 기록상 cursor·URL history 검증 기록 | `test/browser/history.spec.ts`; `test/unit.test.ts`; `evidence/E07/` | back/forward·reload·삽입 중 pagination 회귀 | P10·P19의 검증 항목으로 충분하며 테스트 파일 자체는 독립 면접 지점이 아니다. | 낮음 | 없음 | 06·08 |
| A | 08 (E08) | 제목 미노출 — 기록상 authenticated Server Component와 hydration 경계 구현 | `app/monitors/page.tsx`; `app/monitors/server-data.ts` — `readInitialMonitors`; `app/monitors/initial-state.ts`; `app/monitors/monitor-workspace.tsx`; `app/monitors/use-monitors.ts` | [P20 인증된 SSR의 no-store, coherent initial payload와 account cache 경계](04-client-state-and-server-rendering.md#p20) | cookie를 client prop으로 넘기지 않고 요청별 HTML/RSC를 공유 cache에서 제외하며, hydration 중 중복 read와 사용자 전환 시 stale route cache를 막는다. | 높음 | 중간 | 04·05·06·07 |
| B | 08 (E08) | 제목 미노출 — 기록상 production HTML·RSC baseline 고정 | `evidence/E08/run.mjs`; `evidence/E08/browser-scenario.mjs` | JavaScript 없이 인증된 초기 HTML 관찰 | production build에서 SSR 실패를 고정한 테스트 설계는 설명 가치가 있으나 핵심 설계는 P20이다. | 중간 | 낮음 | 06·07 |
| B | 08 (E08) | `test: verify E08 production rendering and browser boundaries` | `test/browser/`; `test/e08-rsc-transport.mjs`; `app/monitors/page.tsx` | SSR·hydration·privacy·keyboard 접근성 종합 검증 | 접근성은 중요한 품질 속성이지만 이 프로젝트의 대표 구현 질문은 authenticated SSR P20이 더 강하다. | 중간 | 낮음 | 06·07 |
| S | 09 (E09) | 제목 미노출 — 기록상 queued CheckRun과 별도 worker 구현 | `server/worker.ts`; `server/app.ts` — manual Check 202 enqueue; `server/migrations/006_check_queue.sql`; `server/model.ts` | [P21 durable CheckRun 상태 머신과 API·worker 분리](05-worker-concurrency-and-recovery.md#p21) | 요청 수명과 외부 I/O 수명을 분리하고 QUEUED→RUNNING→terminal invariant를 DB에 남겨 재시작 후에도 관찰 가능한 실행을 만든다. | 높음 | 높음 | 10·11·12 |
| A | 09 (E09) | 제목 미노출 — 기록상 interval scheduler와 scheduled intent 구현 | `server/worker.ts` — `scheduleDueChecks`; `server/migrations/006_check_queue.sql` | [P22 고정 시간 슬롯과 unique constraint를 이용한 scheduler 멱등성](05-worker-concurrency-and-recovery.md#p22) | 반복 tick·재시작에서도 같은 monitor·slot intent를 한 번만 생성하고 긴 downtime을 어떻게 처리할지 trade-off를 드러낸다. | 높음 | 높음 | 10·11 |
| B | 09 (E09) | 제목 미노출 — 기록상 API 내부 동기 실행 baseline 고정 | `evidence/E09/` | worker 분리 전 요청 수명 결합 counterexample | 경계 분리의 동기를 설명하는 자료이며 독립 구현은 P21·P22에 통합된다. | 중간 | 낮음 | 01 |
| C | 09 (E09) | `test: verify E09 worker lifecycle and async regressions` | `test/execution.test.ts`; `test/browser/lifecycle.spec.ts`; `evidence/E09/` | worker·scheduler·browser polling 회귀 검증 | 핵심 상태 머신과 scheduler는 P21·P22에 이미 있다. | 낮음 | 없음 | 10·11 |
| S | 10 (E10) | `feat: persist manual identities and atomic check ownership` | `server/worker.ts` — claim·`completeCheck`; `server/migrations/007_check_ownership.sql` | [P23 `FOR UPDATE SKIP LOCKED` 기반 원자적 claim과 worker ownership](05-worker-concurrency-and-recovery.md#p23) | 여러 worker가 같은 QUEUED row를 실행하지 못하게 하고 terminal write도 claim owner에게만 허용하는 핵심 동시성 문제다. | 높음 | 높음 | 09·11 |
| S | 10 (E10) | `feat: persist manual identities and atomic check ownership` | `server/contracts.ts` — `idempotencyKey`; `server/app.ts`; `server/migrations/007_check_ownership.sql`; `app/monitors/use-monitors.ts` — `checkIntents` | [P24 응답 유실을 견디는 요청 idempotency와 intent lifecycle](05-worker-concurrency-and-recovery.md#p24) | 동일 사용자·동일 key unique constraint와 클라이언트 key 보존으로 commit 뒤 ack 유실을 중복 실행 없이 재시도한다. | 높음 | 높음 | 06·09·11 |
| B | 10 (E10) | `test: freeze E10 intent and ownership counterexamples` | `evidence/phase-1/E10/` | 중복 claim과 lost-ack replay counterexample | 좋은 동시성 test barrier이지만 대표 구현 문제는 P23·P24다. | 중간 | 낮음 | 09·11 |
| C | 10 (E10) | 제목 미노출 — 기록상 E10 ownership·idempotency 검증 기록 | `test/ownership.test.ts`; `evidence/phase-1/E10/` | 두 worker와 같은-key replay 검증 | 설계 증거이며 독립 항목은 P23·P24에 포함된다. | 낮음 | 없음 | 09·11 |
| S | 11 (E11) | 제목 미노출 — 기록상 lease·crash recovery와 terminal transaction 구현 | `server/worker.ts` — `CHECK_LEASE_MS`, `recoverExpiredChecks`, claim·`completeCheck`; `server/migrations/008_check_lease.sql` | [P25 lease 만료, stale completion 차단과 crash 후 같은 ID의 terminal 복구](05-worker-concurrency-and-recovery.md#p25) | worker crash를 감지할 heartbeat 대신 유한 lease로 모델링하고, row lock 뒤 lease를 재검사해 늦은 worker가 복구 결과를 덮지 못하게 한다. | 높음 | 높음 | 09·10·12 |
| A | 11 (E11) | 제목 미노출 — 기록상 worker graceful shutdown 구현 | `server/worker.ts` — worker loop, `SHUTDOWN_GRACE_MS`; `server/main.ts` | [P26 신규 claim 중단, in-flight drain, DB close와 강제 종료 상한](05-worker-concurrency-and-recovery.md#p26) | signal 이후 intake와 현재 작업을 분리하고 종료 상한 안에 drain·connection 정리를 마치지 못하면 비정상 종료로 관측한다. | 높음 | 중간 | 14 |
| B | 11 (E11) | `test: freeze E11 crash and shutdown checkpoints` | `evidence/phase-1/E11/` | crash checkpoint와 signal 경계 고정 | 복구 구현의 전제와 testability 설명에는 유용하지만 별도 구현보다 P25·P26에 통합한다. | 중간 | 낮음 | 10·14 |
| C | 11 (E11) | 제목 미노출 — 기록상 E11 observer·baseline repair | `evidence/phase-1/E11/` | 복구 증거 관찰기 보정 | 제품 코드가 아니라 검증 harness 보정에 가깝다. | 낮음 | 없음 | 11 |
| C | 11 (E11) | `test: preserve E11 recovery proof and repair provenance` | `evidence/phase-1/E11/` | 복구 증거와 repair provenance 보존 | 재현 신뢰성에는 중요하지만 면접 기본기 항목으로 분리할 필요는 낮다. | 낮음 | 없음 | 11 |
| S | 12 (E12) | `feat: validate outbound destinations and bound check resources` | `server/outbound.ts` — `canonicalUrl`, `validatedDestination`, `publicAddress`, `OutboundPolicyError` | [P04 URL 정규화·DNS 검증·redirect 재검증을 통한 SSRF 차단](01-http-contracts-and-outbound-io.md#p04) | 문자열 URL 검사만으로 막을 수 없는 literal IP, 혼합 DNS, 재바인딩, redirect 우회를 네트워크 연결 경계에서 닫는다. | 높음 | 높음 | 01·05 |
| S | 12 (E12) | `feat: validate outbound destinations and bound check resources` | `server/outbound.ts` — `CONNECT_TIMEOUT_MS`, `READ_TIMEOUT_MS`, `TOTAL_TIMEOUT_MS`, `MAX_REDIRECTS`, connector 경계 | [P05 connect/read/total timeout과 redirect를 포함한 자원 상한](01-http-contracts-and-outbound-io.md#p05) | 각 단계 timeout만으로는 전체 실행 시간이 제한되지 않는다. socket·request·response·timer 정리까지 포함한 I/O lifecycle 문제다. | 높음 | 높음 | 01·11 |
| A | 12 (E12) | `feat: validate outbound destinations and bound check resources` | `server/check.ts`; `server/outbound.ts`; `server/model.ts` — terminal result와 policy failure reason | [P06 관찰 결과와 재시도 가능성의 분리](01-http-contracts-and-outbound-io.md#p06) | HTTP 상태, transport 실패, 정책 중단, crash 불확실성을 하나의 성공/실패 boolean으로 뭉치지 않고 후속 정책과 분리한다. | 높음 | 중간 | 09·11 |
| B | 12 (E12) | `test: freeze E12 outbound safety and resource cases` | `evidence/phase-1/E12/scenario.json`; `evidence/phase-1/E12/fixture.ts`; `evidence/phase-1/E12/baseline.mjs` | SSRF·timeout·redirect·large response counterexample 집합 | 보안 경계 조건 선정은 설명 가치가 높지만 구현 문제는 P04~P06으로 통합한다. | 중간 | 낮음 | 01·11 |
| C | 12 (E12) | `test: record E12 destination and resource evidence` | `evidence/phase-1/E12/`; `test/outbound.test.ts` | 목적지·자원 상한 검증 증거 | 핵심 구현의 증거 고정이며 별도 워크북은 필요 없다. | 낮음 | 없음 | 12 |
| A | 13 (E20) | `perf: index sparse failed history without changing pagination` | `server/app.ts` — history query; `server/migrations/010_failed_history_index.sql` | [P11 실제 query plan으로 부분 인덱스를 선택하는 판단](02-database-consistency-and-pagination.md#p11) | 데이터 분포와 predicate를 고정하고 반환 row·filter 제거·buffer·sort를 비교해 기능 계약을 바꾸지 않는 성능 개선을 설명할 수 있다. | 높음 | 중간 | 07 |
| B | 13 (E20) | `test: freeze E20 history query and skewed dataset` | `evidence/phase-1/E20/`; `server/app.ts` | 고정 분포와 실제 parameter binding으로 query plan 비교 | 성능 실험 설계 설명에 유용하지만 최종 면접 포인트는 부분 인덱스 판단 P11이다. | 중간 | 낮음 | 07 |
| A | 14 (E24) | 제목 미노출 — 기록상 readiness·metrics와 운영 endpoint 구현 | `server/operations.ts` — `Operations.ready`, `Operations.metrics`, `workerHealth`; `server/app.ts`; `server/worker.ts`; `server/main.ts` | [P27 liveness·readiness·metric unknown을 구분하는 운영 계약](06-operations-and-observability.md#p27) | DB 장애에서 process restart와 traffic removal을 분리하고, 읽지 못한 queue 상태를 0으로 조작하지 않는 의미론이 중요하다. 현재 기록의 readiness는 `SELECT 1`을 사용하며 별도 probe timeout은 확인되지 않아 보강점까지 설명할 가치가 있다. | 높음 | 중간 | 03·11 |
| A | 14 (E24) | 제목 미노출 — 기록상 구조화 로그와 metric 경계 구현 | `server/operations.ts` — `Operations`, `PROCESS_ID`, `recordHttp`; `server/app.ts`; `server/worker.ts`; `server/main.ts` | [P28 allowlist 기반 구조화 로그와 bounded cardinality](06-operations-and-observability.md#p28) | raw URL·request·error 직렬화를 피하고 route template과 유한 label로 비밀 노출과 시계열 폭증을 함께 막는다. | 높음 | 중간 | 04·05·11 |
| B | 14 (E24) | `test: freeze E24 operations and container boundaries` | `evidence/phase-1/E24/` | 운영 endpoint·로그·container 경계 baseline | 운영 요구를 고정한 자료이며 핵심 구현은 P27·P28이다. | 중간 | 낮음 | 11 |
| B | 14 (E24) | `fix(e24): report completed frontend SIGTERM cleanup as success` | production Next server patch·checksum 검증 기록; `compose.production.yaml` | upstream process 종료 의미를 배포 계약에 맞추는 patch | 실제 운영 trade-off 설명 가치는 있으나 pinned framework 구현에 매우 특수해 독립 백지 구현에는 부적합하다. | 중간 | 낮음 | 11 |
| B | 14 (E24) | 제목 미노출 — 기록상 production container 경계 구현 | `Dockerfile`; `compose.production.yaml`; `next.config.ts`; `test/container-smoke.mjs` | production image·role process·SIGTERM·container smoke 계약 | 실행 사용자·standalone 산출물·role별 process와 signal 검증은 설명 가치가 높지만 Docker/Next 설정 비중이 커 별도 백지 구현은 적합하지 않다. | 중간 | 낮음 | 11 |
| C | 14 (E24) | 제목 미노출 — 기록상 E24 운영 회귀 검증과 provenance 보존 | `evidence/phase-1/E24/`; `test/container-smoke.mjs` | DB 중단·metric cardinality·secret suppression·container smoke 증거 | P27·P28과 production container 설명의 검증 증거이며 독립 면접 문제는 필요 없다. | 낮음 | 없음 | 03·04·05·11 |
| C | 14 (E24) | `ci(e24): run worker lifecycle checks after browser build` | `.github/workflows/check.yml` | browser build 뒤 worker lifecycle CI 배치 | 검증 순서 조정은 재현성에 중요하지만 CI wiring 자체의 면접 판별력은 낮다. | 낮음 | 없음 | 09·11 |

## 상세 워크북 파일 지도

| 파일 | 면접 포인트 | 역할 |
| --- | --- | --- |
| `01-http-contracts-and-outbound-io.md` | P01~P06 | HTTP runtime 계약, client decoder, outbound I/O, SSRF, 자원 상한, 실패 disposition |
| `02-database-consistency-and-pagination.md` | P07~P11 | migration, schema 계약, mapping, cursor seek pagination, query plan과 partial index |
| `03-authentication-authorization-and-csrf.md` | P12~P16 | password KDF, session token·rotation, owner predicate, CSRF·Origin·CORS |
| `04-client-state-and-server-rendering.md` | P17~P20 | mutation admission, stale response 무효화, URL query identity, authenticated SSR·hydration |
| `05-worker-concurrency-and-recovery.md` | P21~P26 | durable queue, scheduler, atomic claim, idempotency, lease recovery, graceful shutdown |
| `06-operations-and-observability.md` | P27~P28 | liveness·readiness·unknown metric, safe structured logs와 cardinality |

## 대표 Thread와 연관 Thread 관계

| 통합 역량 | 대표 Thread·포인트 | 연관 Thread | 통합 기준 | 상세 문서 |
| --- | --- | --- | --- | --- |
| 외부 HTTP 실행 경계 | 12(E12) P04~P06 | 01(E01) P01, 02(E02) P02 | E01의 단일 종료를 E12의 목적지 검증·전체 예산·결과 의미로 확장했다. | `01-http-contracts-and-outbound-io.md` |
| HTTP 입력·응답 계약 | 02(E02) P02·P03 | 03·04·05·10의 새 오류 code와 입력 | 서버 validation과 브라우저 decoder가 같은 status/code envelope를 공유한다. | `01-http-contracts-and-outbound-io.md` |
| DB 계약과 정합성 | 03(E03) P07~P09 | 05·07·09·10·11·12의 migration 확장 | 후속 schema 변경은 E03의 ordered checksum·startup verification·mapping 원칙을 재사용한다. | `02-database-consistency-and-pagination.md` |
| history 조회와 성능 | 07(E07) P10 | 13(E20) P11 | seek pagination의 정렬·predicate 계약은 유지하고 FAILED 전용 partial index만 추가했다. | `02-database-consistency-and-pagination.md` |
| 인증과 세션 수명 | 04(E04) P12~P14 | 05(E05), 08(E08) | password·token·rotation을 기반으로 CSRF와 authenticated SSR의 account boundary가 성립한다. | `03-authentication-authorization-and-csrf.md` |
| 권한과 browser mutation 보호 | 05(E05) P15·P16 | 04·06·07·08 | owner predicate와 CSRF/Origin은 각각 resource authority와 browser request authority를 담당한다. | `03-authentication-authorization-and-csrf.md` |
| 클라이언트 권위와 navigation | 06(E06) P17·P18 | 07(E07) P19, 08(E08) P20 | 동기 admission과 versioned state owner가 URL query identity·SSR initial payload까지 이어진다. | `04-client-state-and-server-rendering.md` |
| durable execution과 복구 | 10(E10) P23·P24, 11(E11) P25·P26 | 09(E09) P21·P22, 12(E12) | queue 상태 머신 위에 claim ownership, request identity, lease recovery, bounded outbound 실행을 층별로 추가했다. | `05-worker-concurrency-and-recovery.md` |
| 운영 lifecycle과 관측 | 14(E24) P27·P28 | 03(E03), 11(E11) | DB 준비 상태와 process/worker 종료·로그를 운영자가 오해하지 않는 계약으로 노출한다. | `06-operations-and-observability.md` |

## S/A 완전성 검증

아래 표의 28개 행을 상세 문서의 실제 anchor와 대조했다. 모든 S/A 항목은 독립 상세 항목으로 작성됐으며, 중복 Thread의 역량은 마지막 열에 대표 문제로 통합한 범위를 명시했다.

| ID | 우선순위 | 상세 항목 | 상태 | 통합·대표 범위 |
| --- | --- | --- | --- | --- |
| P01 | S | [비동기 HTTP Check의 단일 종료와 자원 정리](01-http-contracts-and-outbound-io.md#p01) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E12 자원 경계의 선행 원리로 연결. |
| P02 | A | [`unknown` 입력의 런타임 검증과 안정적인 오류 분류](01-http-contracts-and-outbound-io.md#p02) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. 후속 Thread의 입력·오류 code 확장은 이 계약에 통합. |
| P03 | A | [신뢰하지 않는 API 응답의 방어적 디코딩](01-http-contracts-and-outbound-io.md#p03) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E04·E05 client 오류 분류를 대표. |
| P04 | S | [URL 정규화·DNS 검증·redirect 재검증을 통한 SSRF 차단](01-http-contracts-and-outbound-io.md#p04) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E01 origin 제한을 일반 SSRF 방어로 통합. |
| P05 | S | [connect/read/total timeout과 redirect를 포함한 자원 상한](01-http-contracts-and-outbound-io.md#p05) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E01 timeout·cleanup을 확장. |
| P06 | A | [관찰 결과와 재시도 가능성의 분리](01-http-contracts-and-outbound-io.md#p06) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E09~E12 terminal 의미를 통합. |
| P07 | S | [체크섬을 포함한 순차 마이그레이션과 단일 트랜잭션](02-database-consistency-and-pagination.md#p07) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. 모든 후속 migration chain을 대표. |
| P08 | A | [애플리케이션 시작 전 실제 DB schema fail-fast 검증](02-database-consistency-and-pagination.md#p08) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. 후속 schema 제약 검증을 대표. |
| P09 | A | [명시적 row mapping과 PostgreSQL 단일 진실 원천](02-database-consistency-and-pagination.md#p09) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. process memory→PostgreSQL 권위 전환과 mapping을 통합. |
| P10 | S | [조건에 결박된 cursor와 안정적인 seek pagination](02-database-consistency-and-pagination.md#p10) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. cursor validation·seek SQL·index를 한 문제로 통합. |
| P11 | A | [실제 query plan으로 부분 인덱스를 선택하는 판단](02-database-consistency-and-pagination.md#p11) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. E07 query 계약을 유지한 E20 성능 판단. |
| P12 | S | [비밀번호 KDF, 독립 salt, constant-time 비교와 계정 존재 timing 완화](03-authentication-authorization-and-csrf.md#p12) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P13 | S | [고엔트로피 session token, hash 저장, 엄격한 cookie parser와 만료](03-authentication-authorization-and-csrf.md#p13) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P14 | S | [세션 교체·폐기의 트랜잭션과 수명주기 invariant](03-authentication-authorization-and-csrf.md#p14) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P15 | S | [권한 조건을 SQL predicate에 포함하고 foreign·absent를 404로 통합](03-authentication-authorization-and-csrf.md#p15) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. nested/direct read와 모든 mutation 권한을 통합. |
| P16 | S | [session-bound CSRF·Origin·CORS와 검증 순서](03-authentication-authorization-and-csrf.md#p16) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. CSRF·Origin·CORS·검증 순서를 통합. |
| P17 | S | [첫 `await` 전 동기식 중복 mutation admission](04-client-state-and-server-rendering.md#p17) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. response barrier counterexample를 구현 문제에 통합. |
| P18 | A | [generation·query version으로 늦은 비동기 응답 무효화](04-client-state-and-server-rendering.md#p18) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. read·mutation·logout 수명과 query version을 통합. |
| P19 | A | [서버 query 조건을 URL과 cache identity에 일치시키기](04-client-state-and-server-rendering.md#p19) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. URL navigation과 cache identity를 통합. |
| P20 | A | [인증된 SSR의 no-store, coherent initial payload와 account cache 경계](04-client-state-and-server-rendering.md#p20) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. SSR·hydration·account route cache를 통합. |
| P21 | S | [durable CheckRun 상태 머신과 API·worker 분리](05-worker-concurrency-and-recovery.md#p21) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. API enqueue와 worker execution 상태를 통합. |
| P22 | A | [고정 시간 슬롯과 unique constraint를 이용한 scheduler 멱등성](05-worker-concurrency-and-recovery.md#p22) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P23 | S | [`FOR UPDATE SKIP LOCKED` 기반 원자적 claim과 worker ownership](05-worker-concurrency-and-recovery.md#p23) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P24 | S | [응답 유실을 견디는 요청 idempotency와 intent lifecycle](05-worker-concurrency-and-recovery.md#p24) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. |
| P25 | S | [lease 만료, stale completion 차단과 crash 후 같은 ID의 terminal 복구](05-worker-concurrency-and-recovery.md#p25) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. lease·복구·stale completion·terminal transaction을 통합. |
| P26 | A | [신규 claim 중단, in-flight drain, DB close와 강제 종료 상한](05-worker-concurrency-and-recovery.md#p26) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. signal·drain·DB close·강제 종료를 통합. |
| P27 | A | [liveness·readiness·metric unknown을 구분하는 운영 계약](06-operations-and-observability.md#p27) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. health와 metric unknown 의미를 통합. |
| P28 | A | [allowlist 기반 구조화 로그와 bounded cardinality](06-operations-and-observability.md#p28) | 독립적인 상세 워크북 항목으로 작성됨 | 독립 상세 항목. request log·process event·metric label 경계를 통합. |

## 백지 구현 우선순위

1. **동시성·복구 1차**: P23 원자적 claim → P25 lease 복구 → P24 request idempotency → P21 durable 상태 머신
2. **보안·외부 경계 1차**: P04 SSRF 방어 → P16 CSRF·Origin → P15 owner predicate → P12 password KDF → P13 session token
3. **정합성·자료구조 1차**: P07 migration → P10 cursor seek pagination → P14 session rotation transaction
4. **비동기·자원 1차**: P01 단일 종료 → P05 전체 자원 예산 → P17 동기 mutation admission
5. **A급 구현 확장**: P22 scheduler → P18 stale response 무효화 → P03 client decoder → P02 runtime validator → P09 row mapping → P27 health 계약 → P28 safe log projection
6. **설계 중심 축소 구현**: P06 disposition, P08 schema verifier, P11 index 후보 판단, P19 URL query identity, P20 SSR 초기 payload, P26 shutdown coordinator

## 설명 연습 우선순위

1. P25 lease와 stale completion, P23 claim ownership, P24 lost-ack idempotency의 차이
2. P04 DNS·redirect SSRF 방어와 P05 connect/read/total timeout의 결합
3. P16 CSRF·Origin·SameSite·CORS의 역할 분리와 검증 순서
4. P07 migration history와 P08 실제 schema 검증이 서로 대체되지 않는 이유
5. P10 OFFSET 대신 seek cursor를 선택한 이유와 P11 partial index의 적용 범위
6. P12 password KDF·dummy work, P13 raw token/hash, P14 rotation transaction의 계층
7. P17 admission과 P18 generation/version이 막는 서로 다른 race
8. P20 authenticated SSR의 no-store·coherent payload·account cache boundary
9. P21 API/worker 분리와 P22 scheduler slot 멱등성
10. P27 liveness/readiness/unknown metric, P28 cardinality·secret 최소화

## 한 문제로 통합한 Thread 묶음

- **Thread 01 + 12** → P01·P04·P05: 단일 종료에서 목적지 안전성·전체 자원 상한까지 이어지는 외부 HTTP 실행 경계
- **Thread 02 + 03의 NUL 경계 + 04·05의 오류 확장** → P02·P03: 서버 runtime validation과 client decoder의 공통 오류 계약
- **Thread 03 + 후속 04·05·07·09·10·11·12·13 migration** → P07·P08: append-only migration과 실제 schema startup gate
- **Thread 03 + 07 + 13(E20)** → P09·P10·P11: DB mapping·history seek pagination·query-plan 기반 index 선택
- **Thread 04 + 05 + 08** → P12~P16·P20: credential 저장, session lifecycle, resource authority, browser request authority, authenticated rendering
- **Thread 06 + 07 + 08 + 09의 polling** → P17~P20: 클라이언트 상태 권위, query identity, hydration과 stale response 차단
- **Thread 09 + 10 + 11 + 12** → P21~P26: durable intent, scheduler, claim ownership, idempotency, lease recovery, bounded endpoint 실행
- **Thread 11 + 14(E24)** → P26~P28: graceful shutdown, health semantics, structured process·request 관측
