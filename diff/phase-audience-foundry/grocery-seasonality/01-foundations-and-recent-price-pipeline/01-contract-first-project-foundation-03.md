## `docs: record approved source gate plan`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
new file mode 100644
index 0000000..bb7c2d2
--- /dev/null
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -0,0 +1,200 @@
+# 첫 구현 계획
+
+## 결정
+
+선택 path는 **A — 최근 비교 MVP**다. 공개 표현은 `KAMIS 소매 조사 평균`과
+source가 같은 row에 제공한 `1주 전·1개월 전·1년 전 제공값`의 결정적 차이로
+제한한다. `1일 전` 및 kg 환산값은 저장·공개하지 않는다.
+
+정확한 reference date는 source에 없으므로 모든 reference에
+`SOURCE_REFERENCE_DATE_UNAVAILABLE`을 보존한다. 조사일에서 날짜를 역산하지
+않는다. `제철`, `평년`, `저렴`, `비싸`, `추천`, `최저가`, `실시간 매장가격`,
+예측을 만들지 않는다.
+
+## source gate 증거
+
+검증일은 2026-08-30(KST)이며 raw response body는 파일이나 Git에 저장하지 않았다.
+
+### 공식성·권리
+
+- landing: `https://www.data.go.kr/data/15156063/openapi.do`
+- 제공기관: 한국농수산식품유통공사(aT), 관리부서: 디지털AI운영부
+- interface: Swagger `1.0.0`, `GET https://apis.data.go.kr/B552845/recent/price`
+- 공식 첨부 명세·코드 ZIP SHA-256:
+  `07417ea9eb882a33615721256ff8be3b131cdb10bbc9c7b40472bf049a7e0f88`
+- 비용 무료, 개발·운영 자동승인, 개발계정 트래픽 10,000. 기간 단위와 초당
+  수치는 문서에 없으므로 호출 예산은 수집 실행당 최대 12회로 더 좁게 제한한다.
+- dataset 표시는 `이용허락범위 제한 없음`이고 포털 이용정책은 자유이용에
+  상업·비상업 이용과 변형·2차 저작물 작성을 허용한다. 제품은 그보다 좁게
+  정규화 사실만 공개하고 aT·공공데이터포털·dataset ID·조사일·검토일을 표시한다.
+- raw payload 보존을 별도로 열거한 문구는 확인하지 못했다. 따라서 artifact
+  retention은 `HASH_ONLY`이고 원문 재배포도 하지 않는다.
+
+### 인증·HTTP·오류
+
+- 공식 Swagger 가이드는 params client에 일반 인증키(Decoding)를 넣으라고 한다.
+  env 값이 percent-encoded 형태이면 process memory에서 한 번만 decode한 뒤 client가
+  한 번 encode한다. key와 전체 query는 어느 receipt에도 남기지 않는다.
+- 최소 live 요청은 HTTPS, redirect 없음, HTTP 200, provider `resultCode=0`,
+  `application/json; charset=utf-8`로 성공했다.
+- XML canary도 HTTP 200, provider code `0`, `application/xml`, UTF-8과 같은 23개
+  item field를 반환했다.
+- key 누락 canary는 HTTP 401과 JSON
+  `OpenAPI_ServiceResponse.cmmMsgHeader` envelope, reason code `20`,
+  `SERVICE_KEY_IS_NULL`을 반환했다.
+- portal 문서는 timeout `05`, 일 허용량 `22`, 초당 허용량 `23`, provider 명세는
+  내부 오류 `-1`, 미등록 `-3`, 서버 오류 `-5`, 트래픽 초과 `-10`을 정의한다.
+  실제 429를 만들기 위한 quota 소진은 하지 않았다. client는 HTTP 429,
+  timeout·일시적 5xx와 provider `-1|-5|-10|22|23`만 bounded retry하고 auth,
+  invalid parameter, schema·identity 오류는 retry하지 않는다.
+
+### pagination·schema·멱등성
+
+- 실제 `totalCount=452`. `numOfRows=100`으로 5 page를 각각
+  `100,100,100,100,52`행 대사했다.
+- 독립된 두 FetchAttempt의 page body hash sequence와 row sequence가 같았다.
+  ordered manifest SHA-256은 두 번 모두
+  `dd893ef82f1f1597a2b65ca6024f31fb7b62ae3f10b13c6d6185365eca2798ba`였다.
+- page별 declared page·total, media type, charset, byte length, body hash만 receipt로
+  남길 수 있다. artifact identity는 ordered page body-hash manifest다.
+- 모든 452 row가 같은 23-field schema였고 semantic series 중복은 0이었다.
+  identity field 13개는 모두 non-empty string이었다.
+- current 가격 452개는 모두 0이 아닌 scale-0 Decimal string이었다. reference
+  missing은 JSON `null`이며 week 27, month 61, year 91개였다. 0, 음수, dash 또는
+  malformed sentinel은 이번 generation에서 없었다. 이후 등장하면 격리한다.
+- 실제 조사일은 row별 최신값이며 하나의 dataset-wide 날짜가 아니다. 오래된
+  series도 존재하므로 row의 `exmn_ymd`를 그대로 보존하고 최근성 cutoff를
+  추측하지 않는다.
+
+### code·series·coverage
+
+- 공식 code와 live response가 `01=소매`, `200=채소류`, `400=과일류`로 일치했다.
+- 공식 code workbook의 item 136, variety 332, grade 693 row에는 동일 semantic code의
+  name conflict가 없었다.
+- exact identity는
+  `(01, category, item, variety, grade, raw unit, raw unit size,
+  KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1)`이다.
+- API에는 region·market field가 없다. 공식 KAMIS 조사요령은 일반 소매 농·수산물과
+  가공식품을 22개 도시에서 매일(휴일 제외) 조사하고, 공개 화면은 지역 `전체`를
+  제공한다. 이 evidence revision을 aggregate coverage identity에 고정하고 도시 목록,
+  조사방법 또는 API field가 바뀌면 publication을 차단한다.
+- source row 안에 identity와 current/week/month/year field가 함께 있고 exact filter가
+  동일 code tuple 한 row만 반환했다. reference date field는 없으므로 날짜 상태는
+  unavailable이다. KAMIS 조사방법상 일시 품절 값이 일정 기간 전일값으로 입력될 수
+  있으므로 공개 문구는 source의 `제공값` 이상으로 확대하지 않는다.
+
+### bounded 5+5 canary
+
+공식 code workbook의 item·variety·grade가 모두 exact match했고 live current/week/month/year가
+유효한 다음 series를 확인했다. unit과 unit size는 live row의 raw identity다.
+
+- 채소: `200/212/00/04 포기×1`, `200/213/00/04 g×100`,
+  `200/214/01/04 g×100`, `200/214/02/04 g×100`,
+  `200/215/00/04 kg×1`
+- 과일: `400/414/12/24 kg×2`, `400/430/00/04 개×1`,
+  `400/411/06/04 개×10`, `400/420/02/04 개×1`,
+  `400/419/02/04 개×10`
+
+필터 canary는 소매 채소 58, 소매 과일 37 total을 선언했고 요청한 첫 5 row가 모두
+요청 code와 일치했다. exact 과일 series filter는 total 1, received 1이었다.
+
+## 가장 작은 첫 폐쇄 루프
+
+1. hash-only 합성 canary 한 generation을 `FetchAttempt → PageReceipt → SourceArtifact →
+   ParseRun`으로 기록한다.
+2. exact 소매 채소·과일 typed snapshot과 세 reference를 만들고 전체 row를 대사한다.
+3. reviewer가 generation을 승인한다.
+4. 승인 decision으로 불변 `PublicationRevision`을 만들고 publisher가 한 transaction에서
+   `ACTIVATE` 사건과 `RECENT_RETAIL` current pointer를 전환한다.
+5. server-rendered 목록에서 category와 공식 item name을 검색하고 exact series 상세에서
+   source 조사일, 22개 도시 aggregate coverage, raw unit, current와 reference 제공값,
+   결정적 차이, publication 검토일을 읽는다.
+6. 같은 artifact replay는 snapshot과 publication을 중복하지 않고 새 FetchAttempt만 남긴다.
+
+첫 local demo는 secret이나 raw body 없이 고정된 최소 합성 fixture로 lifecycle을
+증명한다. fixture는 live 접근·권리 증거로 주장하지 않으며 위 live receipt hash와
+분리한다. 실제 local ingestion command는 `.env.local`을 process에서만 읽고 hash-only
+artifact를 생성한다.
+
+## typed schema와 migration
+
+하나의 Django app `grocery`가 다음 닫힌 model을 소유한다.
+
+- `SourceConfiguration`: dataset/interface/endpoint allowlist, mode, coverage revision,
+  rights evidence hash, `HASH_ONLY`, logical secret name, timeout·retry·page budget
+- `FetchAttempt`, `PageReceipt`: 상태, attempt ordinal, redacted request shape, ordered page,
+  HTTP/provider code, counts, byte length, body SHA-256, failure class
+- `SourceArtifact`: source identity + ordered manifest SHA-256 unique, total bytes,
+  media type·encoding, first seen, `HASH_ONLY`
+- `ParseRun`: artifact/parser/config unique, deterministic result hash, reconciliation counts
+- `PriceSeriesKey`: exact code/name dimensions, raw unit·size, coverage identity; semantic tuple unique
+- `RetailPriceSnapshot`: parse run + series + source effective date unique, current Decimal KRW
+- `ReferencePrice`: snapshot + `WEEK|MONTH|YEAR` unique, optional Decimal, unavailable reason,
+  nullable source reference date
+- `PriceChangeFact`: reference unique, direction, nullable difference·percentage,
+  calculation version `decimal-half-up-v1`
+- `ReviewDecision`: append-only `APPROVE|REJECT`, reviewer actor, report hash, reason
+- `PublicationRevision`: immutable approved generation, `RECENT_RETAIL`, fact-set hash,
+  copy revision
+- `PublicationEntry`: revision과 snapshot의 ordered immutable membership
+- `PublicationActivation`: append-only `ACTIVATE|ROLLBACK|WITHDRAW`, previous/target revision
+- `PublicationChannel`: channel별 nullable current revision pointer
+
+DB check·unique constraint로 enum, non-negative count·price, complete identity와 승인된
+revision activation을 보강한다. activation event insert와 pointer update는
+`transaction.atomic()`과 row lock으로 한 transaction에서 처리한다.
+
+첫 migration은 additive create-only다. rollback은 application rollback 후 새 빈 local DB에
+역방향 migration을 실행하며 production contract 단계는 없다.
+
+## 구현·검증 범위
+
+### positive
+
+- actual live shape와 같은 합성 JSON의 deterministic parse, 5+5 exact series
+- null reference의 `UNAVAILABLE`, valid Decimal의 signed difference/direction/half-up percent
+- full reconciliation, artifact·parse·snapshot replay idempotency
+- approve 후 atomic activation, 목록·검색·상세와 provenance 표시
+- 이전 승인 revision으로 append-only rollback
+
+### negative
+
+- timeout, 401, 429, 5xx, provider error, response-size·page-budget 초과
+- field/type drift, unknown code, code/name conflict, malformed·zero·negative price,
+  missing current, duplicate semantic identity, unit·coverage drift
+- partial page, total mismatch, nondeterministic replay, unapproved activation,
+  transaction failure와 concurrent pointer change
+- query secret·검색어·raw body가 log/receipt/public response에 없음
+- 서로 다른 series comparison 금지와 source date 역산 금지
+
+### 보안·개인정보·license·운영
+
+- 공개 account/session/analytics 없이 GET-only SSR. 검색어는 DB·session·log에 저장하지 않는다.
+- `.env.local`은 ignore·owner-only이고 key는 URL/log/error/fixture/receipt에 넣지 않는다.
+- endpoint allowlist, HTTPS-only, redirect 거부, 10초 timeout, 4 MB/page와 12 call/attempt
+  상한, bounded retry를 적용한다.
+- raw artifact는 저장하지 않고 SHA-256·byte length·redacted receipt만 보존한다.
+- 공개 화면과 Admin evidence에 aT, dataset 15156063, landing URL, 조사일, 확인·검토일,
+  coverage revision을 표시한다.
+- structured audit log는 lifecycle ID·상태만 기록한다. health와 DB/publication readiness,
+  last-known-good 나이는 분리한다.
+- PostgreSQL logical dump를 암호화 production backup으로 주장하지 않는다. local gate는
+  disposable DB의 `pg_dump`/`pg_restore`로 row count·hash·current pointer를 검증하고,
+  production에는 managed encryption/PITR·RPO 24h/RTO 4h가 여전히 필요함을 기록한다.
+
+## 작은 가역적 commit 순서
+
+1. `docs: record approved source gate plan` — 이 문서만; rollback은 문서 commit revert.
+2. `build: pin django runtime` — Python/Django/PostgreSQL/uv 호환성과 lock·license 고지.
+3. `feat(audit): add acquisition schema` — source/fetch/artifact/parse models + migration + tests.
+4. `feat(price): add typed retail facts` — series/snapshot/reference/change + constraints/tests.
+5. `feat(publication): add approval lifecycle` — review/revision/entry + tests.
+6. `feat(publication): activate revisions atomically` — event/current pointer/rollback + tests.
+7. `feat(source): parse kamis recent rows` — deterministic parser/reconciliation + fixtures/tests.
+8. `feat(source): fetch kamis receipts safely` — management command, redaction, bounds/tests.
+9. `feat(public): render published prices` — Forms/routes/templates/CSS + accessibility/security tests.
+10. `ops: verify database recovery` — local PostgreSQL backup/restore and health gates.
+11. `docs: record local completion evidence` — exact commands, SHA, limits와 남은 production gates.
+
+각 commit은 구현과 가장 가까운 test를 함께 둔다. 100줄 또는 3개 파일을 넘으면 migration,
+검토 또는 rollback 경계를 더 분리하며 generated migration과 lockfile은 크기 예외를 기록한다.


## `docs: define phase zero release gate`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index bb7c2d2..8dac302 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -11,6 +11,35 @@ source가 같은 row에 제공한 `1주 전·1개월 전·1년 전 제공값`의
 않는다. `제철`, `평년`, `저렴`, `비싸`, `추천`, `최저가`, `실시간 매장가격`,
 예측을 만들지 않는다.
 
+## Phase 0 배포 직전 완료 정의
+
+이 세션의 종료점은 실제 배포가 아니라 **Phase 0 배포 직전 production candidate**다.
+승인된 A source path의 실제 수집·검수·publication과 핵심 읽기 흐름을 하나의 Django
+server-rendered responsive web으로 완성한다. 별도 SPA, 네이티브 앱과 앱스토어 배포는
+만들지 않는다.
+
+candidate는 다음을 모두 실제 증거로 통과해야 한다.
+
+- PostgreSQL에 실제 live generation을 수집하고 사람이 승인한 revision을 activate한다.
+- desktop·mobile이 같은 SSR route와 semantic HTML을 사용하며 `360x800`, `390x844`,
+  `768x1024`, `1440x900`에서 실제 브라우저 screenshot과 end-to-end flow를 남긴다.
+- 각 viewport에서 가로 overflow, typography·정보 계층, touch target, 긴 한국어
+  identity·단위·출처·freshness, form 입력·제출·오류 수정을 검수한다.
+- loading, empty, unavailable, stale, validation, server error를 결정적으로 재현하고 상태를
+  색상 외의 text·icon·semantic markup으로도 전달한다.
+- keyboard-only 순서, visible focus, label·error association와 screen-reader accessible name을
+  자동 검사와 수동 browser 검수로 확인한다.
+- production-like `DEBUG=False` check, 보안·dependency·license scan, 고정 부하 profile,
+  구조화 log, liveness/readiness와 freshness alert 판단을 검증한다.
+- disposable PostgreSQL에서 backup/restore로 audit chain·row count·hash·current pointer를
+  대사하고 이전 승인 publication rollback을 훈련한다.
+- clean Git의 exact release SHA, migration·deploy·application rollback·publication rollback
+  절차와 production platform·database·secret·domain의 사람 전용 잔여 작업을 기록한다.
+
+production platform·database·credential과 domain·DNS 선택, 실제 secret 주입 및 실제 배포는
+사람 checkpoint다. 따라서 위 local gate가 끝나도 `Phase 0 완료`나 `배포 완료`라고 표현하지
+않고 `Phase 0 배포 직전 완료`라고만 보고한다.
+
 ## source gate 증거
 
 검증일은 2026-08-30(KST)이며 raw response body는 파일이나 Git에 저장하지 않았다.
@@ -127,10 +156,15 @@ artifact를 생성한다.
 - `SourceArtifact`: source identity + ordered manifest SHA-256 unique, total bytes,
   media type·encoding, first seen, `HASH_ONLY`
 - `ParseRun`: artifact/parser/config unique, deterministic result hash, reconciliation counts
-- `PriceSeriesKey`: exact code/name dimensions, raw unit·size, coverage identity; semantic tuple unique
-- `RetailPriceSnapshot`: parse run + series + source effective date unique, current Decimal KRW
-- `ReferencePrice`: snapshot + `WEEK|MONTH|YEAR` unique, optional Decimal, unavailable reason,
-  nullable source reference date
+- `PriceSeriesKey`: product class·category/item/variety/grade **code**, raw unit·size와 coverage
+  identity의 semantic tuple unique. name은 display·drift 검수 field이며 unique identity에
+  포함하지 않는다. 같은 code의 name drift는 generation을 차단한다.
+- `RetailPriceSnapshot`: parse run + series + source effective date unique, 0보다 큰 current
+  Decimal KRW
+- `ReferencePrice`: snapshot + `WEEK|MONTH|YEAR` unique,
+  `value_status=AVAILABLE|UNAVAILABLE`, nullable Decimal·unavailable reason,
+  `reference_date_status=PROVIDED|SOURCE_REFERENCE_DATE_UNAVAILABLE`, nullable source
+  reference date를 서로 독립적으로 보존한다.
 - `PriceChangeFact`: reference unique, direction, nullable difference·percentage,
   calculation version `decimal-half-up-v1`
 - `ReviewDecision`: append-only `APPROVE|REJECT`, reviewer actor, report hash, reason
@@ -140,9 +174,14 @@ artifact를 생성한다.
 - `PublicationActivation`: append-only `ACTIVATE|ROLLBACK|WITHDRAW`, previous/target revision
 - `PublicationChannel`: channel별 nullable current revision pointer
 
-DB check·unique constraint로 enum, non-negative count·price, complete identity와 승인된
-revision activation을 보강한다. activation event insert와 pointer update는
-`transaction.atomic()`과 row lock으로 한 transaction에서 처리한다.
+각 enum에는 명시적 DB `CheckConstraint`를 두고 count·byte length는 0 이상, available
+price는 0보다 큼, identity는 complete임을 보강한다. 상태 간·row 간 불변식은 service
+transaction과 row lock으로 검사하고 PostgreSQL constraint trigger로 fail-closed한다.
+모든 audit foreign key는 `PROTECT`/`RESTRICT`다. `ReviewDecision`, `PublicationRevision`,
+`PublicationEntry`, `PublicationActivation`은 DB trigger로 update/delete를 막고
+`PublicationChannel.current_revision`만 activation transaction에서 변경할 수 있다.
+activation event insert와 pointer update는 `transaction.atomic()`과 row lock으로 한
+transaction에서 처리한다.
 
 첫 migration은 additive create-only다. rollback은 application rollback 후 새 빈 local DB에
 역방향 migration을 실행하며 production contract 단계는 없다.
@@ -156,6 +195,7 @@ revision activation을 보강한다. activation event insert와 pointer update
 - full reconciliation, artifact·parse·snapshot replay idempotency
 - approve 후 atomic activation, 목록·검색·상세와 provenance 표시
 - 이전 승인 revision으로 append-only rollback
+- 실제 live generation의 검수·activation과 네 viewport browser E2E/screenshot
 
 ### negative
 
@@ -185,16 +225,24 @@ revision activation을 보강한다. activation event insert와 pointer update
 ## 작은 가역적 commit 순서
 
 1. `docs: record approved source gate plan` — 이 문서만; rollback은 문서 commit revert.
-2. `build: pin django runtime` — Python/Django/PostgreSQL/uv 호환성과 lock·license 고지.
-3. `feat(audit): add acquisition schema` — source/fetch/artifact/parse models + migration + tests.
-4. `feat(price): add typed retail facts` — series/snapshot/reference/change + constraints/tests.
-5. `feat(publication): add approval lifecycle` — review/revision/entry + tests.
-6. `feat(publication): activate revisions atomically` — event/current pointer/rollback + tests.
-7. `feat(source): parse kamis recent rows` — deterministic parser/reconciliation + fixtures/tests.
-8. `feat(source): fetch kamis receipts safely` — management command, redaction, bounds/tests.
-9. `feat(public): render published prices` — Forms/routes/templates/CSS + accessibility/security tests.
-10. `ops: verify database recovery` — local PostgreSQL backup/restore and health gates.
-11. `docs: record local completion evidence` — exact commands, SHA, limits와 남은 production gates.
+2. `docs: define phase zero release gate` — responsive·browser·release acceptance 보강.
+3. `build: pin django runtime` — Python/Django/PostgreSQL/uv 호환성과 lock·license 고지.
+4. `feat(audit): record source fetch attempts` — configuration/fetch/page + state constraints.
+5. `feat(audit): deduplicate source artifacts` — hash manifest + two-attempt replay.
+6. `feat(audit): reconcile deterministic parses` — parse counts·hash·nondeterminism gate.
+7. `feat(price): identify exact retail series` — code identity·name drift·coverage constraints.
+8. `feat(price): store current retail snapshots` — positive price·date·idempotency.
+9. `feat(price): derive typed reference changes` — value/date 상태 XOR·half-up·sign constraints.
+10. `feat(review): append generation decisions` — approve/reject chain과 DB immutability.
+11. `feat(publication): seal immutable revisions` — canonical membership·fact-set hash.
+12. `feat(publication): activate revisions atomically` — operation idempotency·pointer·rollback.
+13. `feat(source): parse kamis recent rows` — deterministic parser/reconciliation + fixtures/tests.
+14. `feat(source): fetch kamis receipts safely` — management command, redaction, bounds/tests.
+15. `feat(public): render published prices` — Forms/routes/templates/CSS + 상태·접근성 tests.
+16. `test(web): verify responsive browser flows` — 네 viewport E2E·screenshot과 결함 수정.
+17. `ops: verify release operations` — 관측성, 부하, PostgreSQL backup/restore와 rollback gates.
+18. `docs: record predeploy completion evidence` — exact release SHA, 배포·rollback 절차와
+    production 사람 checkpoint.
 
 각 commit은 구현과 가장 가까운 test를 함께 둔다. 100줄 또는 3개 파일을 넘으면 migration,
 검토 또는 rollback 경계를 더 분리하며 generated migration과 lockfile은 크기 예외를 기록한다.
diff --git a/docs/MVP-ACCEPTANCE.md b/docs/MVP-ACCEPTANCE.md
index e9eebb5..c7e5007 100644
--- a/docs/MVP-ACCEPTANCE.md
+++ b/docs/MVP-ACCEPTANCE.md
@@ -183,7 +183,19 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
 통과해도 첫 public 명칭은 `월별 과거 가격 패턴`이다. 계절성 추정, 구매 추천과 forecast는
 새 근거와 새 제품 결정 없이는 범위 밖이다.
 
-## 생산 준비 인수 기준
+## Phase 0 배포 직전 production candidate 인수 기준
+
+이 gate는 실제 배포 전 local candidate의 완료 조건이다. 통과는 `Phase 0 배포 직전 완료`를
+뜻하며 production platform·database·credential·domain이 정해졌거나 실제 배포가 끝났다는
+뜻이 아니다. 네이티브 모바일 앱, 앱스토어 배포와 별도 SPA는 범위 밖이다.
+
+- [ ] 문서가 허용한 source path로 실제 generation 하나가 수집·검수·publication되어 핵심
+  사용자 폐쇄 루프가 last-known-good revision만 읽는다.
+- [ ] Django server-rendered responsive web 하나가 desktop과 mobile을 함께 지원한다.
+- [ ] 실제 배포에서 사용할 migration, release SHA, deploy·application rollback·publication
+  rollback 명령이 runbook과 clean Git 상태로 재현된다.
+- [ ] 실제 배포에 필요한 platform, PostgreSQL, secret injection, domain·DNS와 운영자 계정은
+  사람 전용 잔여 작업으로 분리되어 있다.
 
 - [ ] 깨끗한 잠금 설치에서 Python `3.14.7`, Django `5.2.17`, PostgreSQL `18.6`, uv
   `0.12.6`이 실제 실행 버전으로 확인된다.
@@ -193,16 +205,32 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
   통과한다.
 - [ ] 운영은 HTTPS, `DEBUG=False`, 정확한 host·CSRF 설정, HSTS와 secure cookie를 사용한다.
 - [ ] Admin은 운영자별 최소 권한, MFA 또는 동등한 강한 인증과 로그인 제한을 사용한다.
-- [ ] secret injection·rotation, structured log, liveness·readiness와 source freshness alert가
-  실제 환경에서 동작한다.
+- [ ] local production-like 설정에서 env secret injection contract·rotation 절차, structured
+  log, liveness·readiness와 source freshness alert 판단이 동작한다. 실제 production secret
+  주입은 배포 checkpoint에 남긴다.
 - [ ] 연속 fetch 실패, quarantine 증가, 대사 불일치, publication·backup 실패가 운영자에게
   경보된다.
-- [ ] 매일 암호화 backup과 point-in-time recovery가 작동하고 빈 환경 복원으로 `RPO 24시간`,
-  `RTO 4시간`, 승인 revision과 audit chain을 검증한다.
+- [ ] disposable PostgreSQL의 실제 `pg_dump`/`pg_restore`가 빈 환경에서 승인 revision,
+  audit chain, row count·hash·current pointer를 복원한다. production의 매일 암호화 backup,
+  point-in-time recovery, `RPO 24시간`·`RTO 4시간`은 platform 선택 뒤 별도 확인한다.
 - [ ] 이전 application과 이전 승인 publication으로 각각 rollback하는 훈련이 성공한다.
 - [ ] secret, dependency vulnerability와 license 검사에 해결되지 않은 차단 항목이 없다.
 
-## 성능과 접근성 인수 기준
+## responsive browser·성능·접근성 인수 기준
+
+실제 browser와 end-to-end test로 `360x800`, `390x844`, `768x1024`, `1440x900`을 각각
+검수하고 viewport별 screenshot을 completion evidence에 연결한다.
+
+- [ ] 어느 viewport에도 document 가로 scroll이 없다.
+- [ ] typography와 heading·metadata 계층이 읽기 쉽고 interactive touch target이 충분하다.
+- [ ] mobile에서 검색 form 입력·제출·validation error 확인·수정이 실제 동작한다.
+- [ ] 긴 한국어 품목·품종·등급·단위·출처·freshness가 잘리거나 겹치지 않는다.
+- [ ] loading, empty, unavailable, stale, validation과 server error 상태가 결정적으로
+  재현되고 screenshot·test로 검증된다.
+- [ ] keyboard-only navigation 순서와 visible focus가 동작한다.
+- [ ] semantic landmark·heading·form label·error association과 screen reader accessible name이
+  자동 검사 및 browser 검수에서 유효하다.
+- [ ] success·warning·error·direction을 색상만으로 전달하지 않는다.
 
 승인된 catalog 크기로 운영과 같은 환경에서 15분 동안 평균 10 requests/s, 동시 사용자 20명,
 목록·검색 70%와 상세 30%의 read-only 부하를 건다. 응답 p95는 500 ms 이하, `5xx`는 0.5%
@@ -210,8 +238,7 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
 때만 별도 cache를 검토한다.
 
 핵심 page와 form은 keyboard만으로 사용할 수 있고 visible focus, 한국어 label, 오류 요약,
-logical heading, 충분한 contrast와 screen reader로 읽히는 방향 문구를 제공해야 한다. 색만으로
-상태를 전달하지 않는다.
+logical heading, 충분한 contrast와 screen reader로 읽히는 방향 문구를 제공해야 한다.
 
 ## 사람 승인과 완료 증거
 
@@ -225,5 +252,10 @@ rollback을 각각 승인한다. 구현 이후 만드는 `docs/COMPLETION-REPORT
 - 보안·license 검사, 접근성·성능 결과, backup restore와 rollback 결과
 - 현재 승인 `PublicationRevision`, 최근 `PublicationActivation`, 알려진 비목표와 다음
   review 날짜
+- 네 viewport의 실제 browser screenshot·E2E 결과와 발견·수정한 UI/UX 결함
+- exact release SHA, clean status·`git fsck`, deploy·rollback 명령과 production
+  platform·database·secret·domain의 사람 전용 잔여 작업
 
-모든 필수 항목과 사람 승인이 실제 증거에 연결될 때만 MVP 완료로 판정한다.
+local candidate의 모든 필수 항목이 실제 증거에 연결되면 `Phase 0 배포 직전 완료`로 판정한다.
+실제 배포와 production 전용 항목은 해당 사람 checkpoint 이후에만 Phase 0 완료 여부를 별도로
+판정한다.


