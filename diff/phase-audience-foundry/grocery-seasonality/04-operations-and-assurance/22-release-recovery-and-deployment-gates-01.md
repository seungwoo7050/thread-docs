# 릴리스·복구·배포 관문

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


