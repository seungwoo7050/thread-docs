## `docs: record predeploy completion evidence`

diff --git a/README.md b/README.md
index 49a3a53..a2ebf87 100644
--- a/README.md
+++ b/README.md
@@ -1,8 +1,8 @@
 # Audience Foundry Grocery Seasonality
 
 한국 소비자가 한국농수산식품유통공사(KAMIS)의 채소·과일 소매 조사 평균을
-동일 품목·품종·등급·판매 단위 안에서 조사일, 1주 전, 1개월 전, 1년 전과
-중립적으로 비교하는 한국어 서비스의 문서 기준선입니다.
+동일 품목·품종·등급·판매 단위 안에서 source row가 제공한 조사일 값과 1주·1개월·1년
+제공값을 중립적으로 비교하는 Django server-rendered responsive web 서비스입니다.
 
 `seasonality`는 저장소 코드명입니다. 첫 MVP는 한 해 전 값 하나를 계절성·평년·제철의
 증거로 바꾸지 않습니다. 공개 화면은 공식 조사값과 결정적인 차이만 표시하며
@@ -16,36 +16,63 @@
 - 정책 기준선: `0cc95e7`
 - 원격 저장소: 없음
 - 공개 상태: 로컬·미공개
-- 구현 상태: runtime, dependency, database, credential, 외부 연동과 배포 없음
+- 구현 상태: **Phase 0 배포 직전 완료**; production 미배포
+- 활성 source path: **A — 최근 비교 MVP**
 
 ## 고정된 첫 범위
 
 - 첫 source는 공공데이터포털의 KAMIS
   [최근일자 도·소매가격정보 API `15156063`](https://www.data.go.kr/data/15156063/openapi.do)입니다.
 - 공개 대상은 source가 `소매`로 식별한 채소류·과일류입니다.
+- 첫 publication은 실제 452행을 대사해 승인한 exact 채소 5개·과일 5개 series입니다.
 - 상세 화면은 정확히 한 품목·품종·등급·판매 단위·검증된 조사범위의 profile입니다.
-- 비교 기준은 source가 제공한 조사일, 1주 전, 1개월 전, 1년 전 가격입니다.
+- 비교 기준은 source가 같은 row에 제공한 조사일 값과 1주·1개월·1년 제공값입니다.
 - 1일 비교, 도매, 수산·축산·곡물, 지역 간 비교, 순위와 알림은 첫 범위가 아닙니다.
 - 계정, 검색 이력, 장바구니, 위치, GPS와 개인화는 만들지 않습니다.
 - 공개 request는 외부 source를 호출하지 않고 승인된 PostgreSQL publication만 읽습니다.
 
-## 구현 전 필수 source gate
+## 완료된 source gate
 
-공공데이터포털 메타데이터는 API가 무료이고 이용허락 제한 없음이며 JSON/XML,
-개발·운영 자동승인이라고 안내하지만 live contract를 증명하지는 않습니다. 파일을
-바꾸기 전에 사람이 key 발급·입력 단계에서 멈추고 실제 최소 요청으로 HTTPS 접근,
-권리, quota, pagination, 오류 envelope, 코드 identity, 결측, 평균·조사범위,
-단위와 세 비교기간의 의미를 검증해야 합니다. 실패한 live evidence를 fixture로
-대체하지 않습니다.
+공식 HTTPS API의 실제 요청으로 인증, JSON/XML, 452행 ordered pagination, provider error,
+소매·채소·과일 code, exact identity·단위·22개 도시 aggregate coverage와 같은 row의
+current/week/month/year 계약을 검증했습니다. 정확한 reference date는 source가 제공하지 않아
+`SOURCE_REFERENCE_DATE_UNAVAILABLE`을 보존합니다. raw 보존 권리는 명시적이지 않아
+`HASH_ONLY`를 사용하며 정규화 사실과 출처만 공개합니다. 세부 증거는
+[구현 계획](docs/IMPLEMENTATION-PLAN.md)에 있습니다.
 
-비교기간 의미만 실패하면 현재 공식 소매 조사 평균 조회로 축소합니다. API가
+비교기간 의미만 실패할 때의 B current-only와 API 운영만 실패할 때의 C monthly file은
+이번 gate에서 선택되지 않은 **해당 없음(N/A)** fallback입니다. API가
 운영에 부적합하지만 공식
 [월별 소매가격 파일 `15087482`](https://www.data.go.kr/data/15087482/fileData.do)의
 권리·identity·단위가 통과하면 파일 공표본과 각 row의 기준 연월을 명시한 별도 정적
 월별 탐색기로 축소할 수 있습니다. KAMIS HTML scraping, 비공식 미러와 quota 우회는
 허용하지 않습니다.
 
-## 계약 문서
+## local 실행
+
+```sh
+make sync
+docker compose up -d db
+env DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py migrate --noinput
+env DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  make serve
+```
+
+public request는 PostgreSQL의 승인 publication만 읽고 KAMIS를 호출하지 않습니다. 실제 ingestion은
+owner-only·Git ignored `.env.local`을 해당 worker process에서만 읽습니다. key를 command argument,
+URL, log, fixture 또는 receipt에 넣지 않습니다. production-like 검증과 실제 배포 전 checkpoint는
+[운영 런북](docs/OPERATIONS-RUNBOOK.md)을 따릅니다.
+local 명령은 ambient `DATABASE_URL`을 상속하지 않도록 위의 고정 loopback Compose database를
+각 process에 명시한다. 다른 database를 대상으로 lifecycle rehearsal을 실행하지 않는다.
+
+local candidate 재현은 clean exact release SHA에서 잠금 설치, 새 빈 DB의 forward migration,
+collectstatic, production-like process와 health·public smoke를 같은 순서로 수행합니다. application
+rollback은 reverse migration 없이 최신 schema를 유지하고 검증된 이전 code·static으로
+되돌립니다. 실제 artifact packaging, bundled notice, vendor deploy·traffic·rollback CLI는
+platform을 선택한 뒤 사람이 별도로 승인하는 production checkpoint입니다.
+
+## 계약·증거 문서
 
 - [도메인 개요](docs/DOMAIN-BRIEF.md)
 - [제품 결정](docs/PRODUCT-DECISIONS.md)
@@ -53,8 +80,11 @@
 - [데이터·감사 모델](docs/DATA-AND-AUDIT-MODEL.md)
 - [기술 결정](docs/TECHNOLOGY-DECISIONS.md)
 - [MVP 인수 기준](docs/MVP-ACCEPTANCE.md)
+- [첫 구현 계획과 source gate 증거](docs/IMPLEMENTATION-PLAN.md)
+- [Phase 0 배포 직전 운영 런북](docs/OPERATIONS-RUNBOOK.md)
+- [Phase 0 배포 직전 완료 보고서](docs/COMPLETION-REPORT.md) — local gate 결과와 production
+  사람 checkpoint를 고정
 
-구현자는 여섯 문서와 root 정책을 처음부터 끝까지 읽고 Git 기준선을 확인한 뒤,
-저장소 변경 없이 source gate를 수행합니다. 안전한 공개 path가 통과한 경우에만
-`docs/IMPLEMENTATION-PLAN.md`를 만들고 작은 검증 가능한 commit으로 첫 루프를
-구현합니다.
+production platform·PostgreSQL·role credential·secret store·domain·DNS 선택, 실제 배포와 traffic
+전환은 사람 전용 작업입니다. 이 저장소는 네이티브 앱, 앱스토어 배포나 별도 SPA를 포함하지
+않습니다.
diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
new file mode 100644
index 0000000..4ee6121
--- /dev/null
+++ b/docs/COMPLETION-REPORT.md
@@ -0,0 +1,232 @@
+# Phase 0 배포 직전 완료 보고서
+
+검증일은 2026-08-30(KST)다. 이 보고서는 local production candidate의 증거이며 실제
+production 배포, traffic 공개나 `Phase 0 완료`를 주장하지 않는다.
+
+현재 상태는 **Phase 0 배포 직전 완료**다. 아래 결과는 local production candidate에만 적용되며
+production platform 선택, 실제 배포와 traffic 공개는 포함하지 않는다.
+
+## 1. release SHA와 Git 상태
+
+- 900초 성능 profile 실행 대상 application SHA:
+  `02f1e5c14e84757d8929da710a41e844bd94bac3`.
+- 최종 release SHA: 이 tracked 보고서를 포함하는 마지막 clean commit이므로 문서 안에서
+  자기 SHA를 참조할 수 없다. 세션 완료 응답이 `git rev-parse HEAD` exact 값을 고정한다.
+- final gate: branch `main`, remote 없음, `git status --porcelain` empty, `git fsck --full` 통과.
+- `.env.local`은 untracked·ignored 상태를 유지한다.
+
+## 2. 구현된 사용자 흐름과 비목표
+
+실제 Path A generation을 검수·승인·seal·activate한 뒤, public request가 active
+`RECENT_RETAIL` revision만 읽는 한국어 Django SSR 폐쇄 루프를 구현했다. 사용자는 채소·과일
+목록에서 공식 품목명을 검색·필터링하고 exact 품목·품종·등급·raw 단위·검증된 aggregate
+coverage 상세로 이동한다. 상세는 source 조사일의 KAMIS 소매 조사 평균과 같은 row의 1주·1개월·
+1년 제공값, 결정적 차이·퍼센트·방향, reference date unavailable, 검토일과 출처를 표시한다.
+
+desktop과 mobile은 별도 SPA 없이 같은 server-rendered responsive route를 쓴다. 네이티브 앱,
+앱스토어 배포, 계정·위치·개인화·analytics, 도매·지역 비교·1일 비교·쇼핑몰 정보·알림과 월별
+과거 패턴 module은 비목표다. 공개 문구는 구매·품질·자연적 시기·미래값 판단으로 확대하지
+않는다.
+
+## 3. 선택 source path와 권리 증거
+
+- 선택: **A — 최근 비교 MVP**.
+- owner: 한국농수산식품유통공사(aT), dataset
+  [15156063](https://www.data.go.kr/data/15156063/openapi.do).
+- endpoint: `GET https://apis.data.go.kr/B552845/recent/price`, Swagger `1.0.0`.
+- 실제 HTTPS·인증·JSON/XML UTF-8·provider success/error envelope와 5-page ordered pagination을
+  검증했다. 452행 schema를 대사했고 소매 채소 58·과일 37 중 exact 5+5를 승인 범위로 삼았다.
+- 두 획득의 ordered manifest는
+  `dd893ef82f1f1597a2b65ca6024f31fb7b62ae3f10b13c6d6185365eca2798ba`로 같았다.
+- 실제 request audit 시각(UTC)은 첫 attempt
+  `2026-08-30T04:00:36.497949Z`~`04:00:37.338969Z`, 두 번째 attempt
+  `2026-08-30T04:00:47.994140Z`~`04:00:48.696744Z`다. body·query 없는 redacted receipt와
+  hash는 [구현 계획](IMPLEMENTATION-PLAN.md#source-gate-증거)에 연결한다.
+- 기존 source configuration의 `state_changed_at`·`rights_confirmed_at`은
+  `2026-08-29T15:00:00Z`(KST 자정)라는 date-precision 값으로 남아 있어 실제 gate 관측 시각으로
+  해석할 수 없다. 정확한 live 관측 시각은 보존되지 않았으며 원본 두 값은 수정·삭제하지 않았다.
+  migration `grocery 0010`은 correction
+  `49143c27-d2dd-5fbd-b1dc-4aa3cc002fab`을 append-only로 추가한다. effective 값
+  `2026-08-30T02:23:44Z`는 증거 commit
+  `d23e5707e1fc3bf6e032d459b149b946b0451e00`의 기록 시각을 사용한 **durable gate-decision
+  recorded-at upper bound**이지 정확한 관측 시각이 아니다. DB trigger가 correction의 base·chronology를
+  검사하고 update/delete를 거부하며 bootstrap·review·inspection은 검증된 effective helper만 쓴다.
+  새 DB는 정확한 effective 값으로 생성되어 correction row가 필요 없다.
+- portal의 `이용허락범위 제한 없음`과 자유이용·상업/비상업 이용·변형 허용을 확인했다.
+  제품은 더 좁게 정규화 사실과 출처만 공개한다. raw payload 보존·재배포 문구는 명시적이지
+  않아 `HASH_ONLY`이고 raw body를 파일·Git·publication에 저장하지 않았다.
+- source configuration의 권리 locator는 공식 dataset landing이다. 저장된
+  `rights_evidence_sha256`은 동적 landing HTML이 아니라 그 페이지의 공식 첨부 명세·코드 ZIP
+  `07417ea9eb882a33615721256ff8be3b131cdb10bbc9c7b40472bf049a7e0f88`이다. landing의 권리 표시,
+  [포털 이용정책](https://www.data.go.kr/ugs/selectPortalPolicyView.do), 2026-08-30(KST) 확인 시각과
+  보수적 공개 판정을 함께 검수했고 ReviewDecision의 별도 private evidence
+  digest/commitment는
+  `2e6dcf9df27077396b8aedf8abaaf69d10bbca2f3a036d6c0127ccb1f434cca6`이다.
+- identity는 소매 `01`, 채소 `200`, 과일 `400`과 item·variety·grade·raw unit·unit size,
+  coverage `KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1`이다. coverage는 공식
+  [KAMIS 조사요령](https://www.kamis.or.kr/customer/price/knowhow/knowhow.do)의 22개 도시 조사와
+  API의 region·market field 부재를 함께 고정했다. 정확한 reference date는 제공되지 않아
+  `SOURCE_REFERENCE_DATE_UNAVAILABLE`이다.
+- quota 기간·초당 한도는 문서에 없고 실제 429 유발을 위한 소진은 하지 않았다. 수집당 최대
+  12회, timeout·size·page·bounded retry로 더 좁게 운영한다.
+
+## 4. test·migration·parser replay
+
+- final locked verification: Ruff format `111 files`, Ruff lint, mypy `97 source files`, migration
+  drift, Django system/deploy check가 모두 통과했고 pytest는 `619 passed`다. production 환경이
+  route test에 HTTPS redirect를 누출한 첫 orchestration run은 `585 passed, 12 failed`로 실패
+  처리한 뒤 test runtime을 deterministic local settings로 격리하고 전체 gate를 다시 통과했다.
+- runtime-only locked sync에서 Python `3.14.7`, Django `5.2.17`, Gunicorn `23.0.0`, psycopg
+  `3.3.4`, WhiteNoise `6.12.0`, uv `0.12.6`을 확인했다. collectstatic은 `129`개 asset과
+  `387`개 post-processed 결과를 재현했다.
+- 실제 FetchAttempt:
+  `6c4bcbeb-d47c-4648-b6ce-01988316b7dc`,
+  `4207e628-3ab9-4a12-a7ae-0cdbec67d744`.
+- SourceArtifact `70955f24-b61d-4b43-a7de-8e603f6ae459`, ParseRun
+  `0c7fad64-e49b-4c8c-9929-aece2782354d`; 452 received, 10 accepted, 442 out-of-scope.
+  두 번째 parse는 replay였고 result hash는
+  `512c65031cdfe2b734af4245d974390073999a32ad494cd9c94b33c2f165261e`다.
+- version 묶음은 parser `kamis-recent-price-v1`, schema migration `grocery 0010`, application
+  `0.1.0`, 성능 검증 application SHA `02f1e5c14e84757d8929da710a41e844bd94bac3`이다.
+- 새 빈 PostgreSQL에 Django·grocery migration `0001`~`0010`을 적용했고 drift가 없었다. 빈
+  상태는 live 200, ready/freshness 503, catalog 200으로 의도대로 실패 폐쇄했다.
+- 실제 publication backup 복원 DB는 migration inventory 28개·public table 25개와 row count,
+  audit/publication contract를 모두 대사했고 live/ready/freshness/catalog가 모두 200이었다.
+
+## 5. desktop·mobile screenshot 검수
+
+`scripts/browser_acceptance.js`가 Chromium 152에서 `360x800`, `390x844`, `768x1024`,
+`1440x900`의 실제 catalog/detail과 상태 matrix를 통과했다. 18개 full-page PNG와 hash는
+[browser evidence](../output/playwright/phase0/README.md)에 있다.
+
+모든 viewport에서 가로 overflow 없음, 최소 44px target, 읽을 수 있는 계층, 긴 한국어
+identity·단위·출처·freshness, mobile 입력·제출·validation 수정, loading·empty·unavailable·
+stale·server error, keyboard 순서·visible focus, landmark·heading·label·accessible name과
+색상 외 text 상태를 확인했다. 발견된 mobile 중복 breadcrumb와 query reflection/cache 결함은
+수정 후 재촬영했다.
+
+## 6. 접근성·성능·보안·license
+
+- axe-core 4.13.0의 실제 catalog/detail WCAG A/AA violation은 각각 0이다. decorative/gradient
+  contrast 한 incomplete 항목은 모든 실제 foreground/background와 양쪽 gradient endpoint가
+  4.5:1 이상인 별도 palette test로 보강했다. axe receipt hash는
+  `a994c5a00a9f5f75213381c5be2ef49624eb796e202d3becd0daef4818d076a6`다.
+- corrected 900초 성능 profile은 exact `9,000/9,000` 성공, catalog·list·search `6,300`, detail
+  `2,700`, error·5xx `0`, p50 `26.858 ms`, p95 `40.656 ms`, max `551.135 ms`, elapsed
+  `900.017초`, throughput `10.0 rps`, revision 단일값으로 통과했다. 고정 logical user는 구성·참여
+  모두 `20`이고 전원 round-robin이었다. 이와 별개로 실제 in-flight peak는 `5`, 상한은 `20`이었다.
+  nominal cadence `100 ms`, bounded recovery floor `90 ms`, effective deadline jitter p95
+  `5.35 ms`·max `76.784 ms`, 실제 최소 submit interval `90.028 ms`, burst `0`, `passed=true`였다.
+  `551.135 ms` max는 관측값이고 통과 gate는 end-to-end p95 `500 ms` 이하다.
+- 두 실패 run도 보존한다. 첫 진단 run은 응답 `9,000`개와 elapsed `900.056초`를 완료했지만 단일
+  scheduler stall을 뒤 요청 전체의 lag로 잘못 누적해 `passed=false`였다. 그 오판을 고친 두 번째
+  run은 응답 `9,000/9,000`, error·5xx `0`, p95 `41.716 ms`였으나 strict `100 ms` floor가 정상
+  overhead를 누적해 elapsed `947.317초`, `9.501 rps`로 실패했다. 최종 runner는 정상 `100 ms`
+  cadence를 유지하면서 stall만 요청당 최대 `10 ms`씩 회복하고 `90 ms` 미만을 burst로 거부한다.
+- security: production setting validation, secure headers/CSP, request ID, no-store HTML,
+  immutable hashed static, GET-only public SSR, default-off Admin·QA·control-plane, exact release
+  lock과 fixed non-login reviewer/publisher permission 경계를 검증했다.
+- secret gate는 `present=yes`, `ignored=yes`, `permissions=ok`, `current_match=no`,
+  `history_match=no`였고
+  key 값·길이·일부·encoding을 출력하지 않았다. `pip-audit`는 알려진 취약점 `0`, locked package
+  license inventory는 해결되지 않은 차단 항목 `0`이었다. Browser assurance 도구까지
+  `THIRD_PARTY_NOTICES.md`에 고정했다. production artifact의 bundled notices는 실제 platform
+  packaging checkpoint다.
+- Make는 ambient `KAMIS_API_KEY`를 모든 recipe child에서 unexport한다. synthetic marker를 parent
+  environment에 둔 negative test에서도 assurance child 경계는 `source_secret_environment=absent`였고
+  marker가 stdout·stderr에 반사되지 않았다.
+
+## 7. backup restore와 publication rollback
+
+- hardened local PostgreSQL 18 custom backup ID `4e74a867-fb92-42be-9e2f-4718a5a276d0`.
+- dump SHA-256 `bcf282944defefc995e7f309fb10e2b5a81fb0c8bd40b08416c86b2780ddb0a5`,
+  manifest SHA-256 `23644fec396e3310c1bd807a3b8321fec62261452323574a5a64d48d15922cf2`.
+- 남긴 local evidence 경계는 directory `0700`, dump·manifest `0600`이며 다른 rehearsal backup과
+  disposable restore DB는 제거했다.
+- 새 격리 DB restore에서 rows·28 migrations·active revision·fact-set·activation chain이 모두
+  일치했다. 이 local dump는 production 암호화 backup/PITR가 아니다.
+- hardened restore는 receipt에서 out-of-band로 보존한 위 manifest SHA-256을
+  `--expected-manifest-sha256 "$BACKUP_MANIFEST_SHA256"`로 반드시 전달하고, 고정 local Docker
+  socket에서 발견·identity-pin한 exact Compose DB container만 사용하는 것이다. target 생성 뒤
+  실패하면 같은 invocation이 만든 exact disposable target만 자동 삭제하고 부재를 확인한다.
+  `25`개 public table·`28`개 migration, row counts, publication metadata와 actual ordered payload를
+  재계산한 canonical fact-set이 모두 일치했고 restored live/ready/freshness/catalog/detail은 모두
+  `200`이었다. 잘못된 manifest receipt는 Docker preflight와 target 생성 전에 fixed code로
+  거부했고 target 부재를 확인했다.
+- publication은 v1 activate → v2 activate → v1 `ROLLBACK`을 append-only로 훈련했다. 현재
+  channel version `3`, v1 `dc6f5c83-92cc-48e7-8103-76f3fd1a668b`, 10 entries, fact-set
+  `6de8e26c22dcee4a7ce4a6e1a0640999399d126d62124cc1b1d7aefcf9aa66a9`다.
+- 승인 ReviewDecision `330cad14-2102-4dcf-a023-93a7368c7efb`는
+  `2026-08-30T04:19:49.128258Z`, 최신 ROLLBACK activation
+  `cd1f3064-2920-4395-9469-7f4b3e0b969d`는 `2026-08-30T04:20:22.343137Z`에 기록됐다.
+- application rollback rehearsal target SHA `d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`도 최신 schema에서
+  live/ready/freshness/catalog/detail와 hashed static 200, 같은 publication hash를 확인했다.
+
+## 8. 구조화 log·health·freshness alert
+
+`grocery.audit`는 allowlist된 single-line JSON만 stdout에 내고 arbitrary message, exception,
+query, search term, body, credential·사용자 정보를 받지 않는다. valid production
+`DEPLOY_VERSION`은 event에 자동 포함되며 request correlation UUID가 응답과 log에 연결된다.
+
+`/health/live`, `/health/ready`, `/health/freshness`는 bounded no-store JSON이다. 현재 실제
+candidate는 모두 200이고 freshness는 `CURRENT`다. active artifact의 마지막 source 확인은
+`2026-08-30T04:00:48.696744Z`이며 36시간 경계는
+`2026-08-31T16:00:48.696744Z`(`2026-09-01 01:00:48 KST`)다. 이 시각 전에도 newer
+content·실패 attempt가 있으면 즉시 stale로 바뀐다. stale·unavailable, DB/migration/publication
+오류, fetch·parse failure는 fixed message code와 non-zero exit로 구분한다. production
+notification route, retention, on-call 담당자, ingress access-log privacy와 backup failure alert는
+platform 선택 뒤 실제 검증해야 한다.
+
+## 9. 실제 배포에 필요한 것
+
+- Python 3.14.7·Django 5.2.17·uv 0.12.6와 PostgreSQL 18.6 호환 platform
+- private managed PostgreSQL, TLS hostname/CA 검증, application·migration·ingestion·reviewer·
+  publisher·backup 역할별 credential/grant
+- managed `DJANGO_SECRET_KEY`, ingestion worker 전용 `KAMIS_API_KEY`, rotation·revocation 절차
+- outbound HTTPS allowlist를 가진 singleton ingestion scheduler, 24시간 cadence, overlap 방지와
+  fixed failure/freshness alert route
+- 승인 domain·DNS·certificate, exact host/CSRF, HSTS subdomain/preload 판단과 trusted proxy 계약
+- external MFA/IAM private operation job, actor provisioning과 첫 production publication 승인
+- encrypted scheduled backup, PITR, retention, restore rehearsal, RPO 24h/RTO 4h evidence
+- health probe, alert route/on-call, log 수집·보존과 query/IP/User-Agent 제거
+
+## 10. deploy·rollback 절차와 사람 작업
+
+아래 순서는 clean `RELEASE_SHA`에서 locked dependency, forward migration, static과 process를
+platform과 무관하게 재현하는 요약이다. authoritative 순서와 environment wrapper는
+[운영 런북](OPERATIONS-RUNBOOK.md)에 있으며 local candidate에서는 synthetic/local assurance
+설정으로 이를 검증한다. production에서는 그 런북의 승인된 environment와 역할별 credential을
+managed injection한 process에서 실행한다.
+
+```sh
+make runtime-sync
+.venv/bin/python manage.py makemigrations --check --dry-run
+.venv/bin/python manage.py showmigrations --plan
+.venv/bin/python manage.py migrate --plan
+.venv/bin/python manage.py check
+.venv/bin/python manage.py collectstatic --noinput
+.venv/bin/python manage.py migrate --noinput
+.venv/bin/python manage.py migrate --check
+.venv/bin/python manage.py check --deploy --fail-level WARNING
+exec .venv/bin/gunicorn config.wsgi:application \
+  --bind "$GUNICORN_BIND" --workers "$GUNICORN_WORKERS" --threads "$GUNICORN_THREADS"
+```
+
+새 release를 traffic 없이 시작해 live→ready→freshness→catalog/detail→hashed static을 검사한 뒤
+사람이 atomic traffic switch를 실행한다. application rollback은 DB·publication을 그대로 두고
+local rehearsal에서 검증한 `PREVIOUS_RELEASE_SHA`
+`d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`와 그 static으로 code를 되돌린다. reverse migration은
+하지 않는다. 이 SHA는 local 호환성 evidence이며 vendor traffic rollback 승인이 아니다.
+publication rollback은 먼저 `inspect_recent_publication`을 실행한 뒤 external-MFA
+publisher job에서 `transition_recent_publication --operation ROLLBACK`과 exact expected state·
+release SHA를 사용한다. DB 복구는 in-place overwrite가 아니라 managed PITR의 새 instance를
+검증한 뒤 connection을 전환한다.
+
+production platform 선택 뒤 artifact 포맷·bundled notice, upload/release, atomic traffic switch와
+application rollback의 exact vendor CLI·account·application scope를 별도로 승인하기 전에는
+배포하지 않는다.
+
+남은 사람 전용 작업은 platform·database·secret store·role/IAM·domain/DNS 선택, production
+backup/PITR·alert 검증, vendor deploy/traffic/rollback 명령 확정과 실제 배포다.
+추가 API key·로그인·약관·결제, 고정 제품 결정 변경과 destructive migration이 필요해져도
+자동 진행하지 않고 별도 사람 승인에서 멈춘다.
diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index 8dac302..ce4cfd1 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -33,8 +33,9 @@ candidate는 다음을 모두 실제 증거로 통과해야 한다.
   구조화 log, liveness/readiness와 freshness alert 판단을 검증한다.
 - disposable PostgreSQL에서 backup/restore로 audit chain·row count·hash·current pointer를
   대사하고 이전 승인 publication rollback을 훈련한다.
-- clean Git의 exact release SHA, migration·deploy·application rollback·publication rollback
-  절차와 production platform·database·secret·domain의 사람 전용 잔여 작업을 기록한다.
+- clean Git의 exact release SHA에서 locked dependency·forward migration·static·process를 다시
+  만들 수 있는 platform-independent deploy 순서와 application·publication rollback 절차를
+  기록한다. vendor CLI와 production packaging은 platform 선택 뒤 사람 checkpoint로 분리한다.
 
 production platform·database·credential과 domain·DNS 선택, 실제 secret 주입 및 실제 배포는
 사람 checkpoint다. 따라서 위 local gate가 끝나도 `Phase 0 완료`나 `배포 완료`라고 표현하지
@@ -51,6 +52,12 @@ production platform·database·credential과 domain·DNS 선택, 실제 secret 
 - interface: Swagger `1.0.0`, `GET https://apis.data.go.kr/B552845/recent/price`
 - 공식 첨부 명세·코드 ZIP SHA-256:
   `07417ea9eb882a33615721256ff8be3b131cdb10bbc9c7b40472bf049a7e0f88`
+- `SourceConfiguration.rights_evidence_locator`는 위 공식 dataset landing이고
+  `rights_evidence_sha256`은 그 landing에서 받은 **첨부 명세·코드 ZIP**의 위 hash다. 동적으로
+  바뀌는 landing HTML 자체의 hash라고 해석하지 않는다. 권리 표시는 landing의
+  `이용허락범위 제한 없음`, 포털
+  [이용정책](https://www.data.go.kr/ugs/selectPortalPolicyView.do), 확인 시각과 아래의 보수적
+  공개 판정을 함께 검수한다.
 - 비용 무료, 개발·운영 자동승인, 개발계정 트래픽 10,000. 기간 단위와 초당
   수치는 문서에 없으므로 호출 예산은 수집 실행당 최대 12회로 더 좁게 제한한다.
 - dataset 표시는 `이용허락범위 제한 없음`이고 포털 이용정책은 자유이용에
@@ -58,6 +65,11 @@ production platform·database·credential과 domain·DNS 선택, 실제 secret 
   정규화 사실만 공개하고 aT·공공데이터포털·dataset ID·조사일·검토일을 표시한다.
 - raw payload 보존을 별도로 열거한 문구는 확인하지 못했다. 따라서 artifact
   retention은 `HASH_ONLY`이고 원문 재배포도 하지 않는다.
+- 실제 generation 승인에 연결된 private gate evidence는 ReviewDecision
+  `330cad14-2102-4dcf-a023-93a7368c7efb`의 `acceptance_evidence_sha256`
+  `2e6dcf9df27077396b8aedf8abaaf69d10bbca2f3a036d6c0127ccb1f434cca6`으로 고정했다.
+  이 hash는 첨부 ZIP hash나 landing HTML hash와 다른 private evidence digest/commitment이며
+  원문 locator가 아니다.
 
 ### 인증·HTTP·오류
 
@@ -88,8 +100,9 @@ production platform·database·credential과 domain·DNS 선택, 실제 secret 
   남길 수 있다. artifact identity는 ordered page body-hash manifest다.
 - 모든 452 row가 같은 23-field schema였고 semantic series 중복은 0이었다.
   identity field 13개는 모두 non-empty string이었다.
-- current 가격 452개는 모두 0이 아닌 scale-0 Decimal string이었다. reference
-  missing은 JSON `null`이며 week 27, month 61, year 91개였다. 0, 음수, dash 또는
+- current 가격 452개와 available week·month·year reference는 모두 0이 아닌 scale-0 Decimal
+  string이었다. reference missing은 JSON `null`이며 week 27, month 61, year 91개였다. 0, 음수,
+  dash 또는
   malformed sentinel은 이번 generation에서 없었다. 이후 등장하면 격리한다.
 - 실제 조사일은 row별 최신값이며 하나의 dataset-wide 날짜가 아니다. 오래된
   series도 존재하므로 row의 `exmn_ymd`를 그대로 보존하고 최근성 cutoff를
@@ -104,7 +117,8 @@ production platform·database·credential과 domain·DNS 선택, 실제 secret 
   `(01, category, item, variety, grade, raw unit, raw unit size,
   KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1)`이다.
 - API에는 region·market field가 없다. 공식 KAMIS 조사요령은 일반 소매 농·수산물과
-  가공식품을 22개 도시에서 매일(휴일 제외) 조사하고, 공개 화면은 지역 `전체`를
+  가공식품을 [22개 도시에서 매일(휴일 제외) 조사](https://www.kamis.or.kr/customer/price/knowhow/knowhow.do)하고,
+  공개 화면은 지역 `전체`를
   제공한다. 이 evidence revision을 aggregate coverage identity에 고정하고 도시 목록,
   조사방법 또는 API field가 바뀌면 publication을 차단한다.
 - source row 안에 identity와 current/week/month/year field가 함께 있고 exact filter가
@@ -140,17 +154,46 @@ production platform·database·credential과 domain·DNS 선택, 실제 secret 
    결정적 차이, publication 검토일을 읽는다.
 6. 같은 artifact replay는 snapshot과 publication을 중복하지 않고 새 FetchAttempt만 남긴다.
 
-첫 local demo는 secret이나 raw body 없이 고정된 최소 합성 fixture로 lifecycle을
-증명한다. fixture는 live 접근·권리 증거로 주장하지 않으며 위 live receipt hash와
-분리한다. 실제 local ingestion command는 `.env.local`을 process에서만 읽고 hash-only
-artifact를 생성한다.
+합성 fixture는 secret이나 raw body 없이 failure·lifecycle contract를 먼저 검증하는 데만
+사용했고 live 접근·권리 증거로 주장하지 않았다. 그 뒤 실제 local ingestion command가
+`.env.local`을 process 안에서만 읽어 다음 live 폐쇄 루프를 완료했다.
+
+### 실제 첫 폐쇄 루프 결과
+
+- FetchAttempt `6c4bcbeb-d47c-4648-b6ce-01988316b7dc`와
+  `4207e628-3ab9-4a12-a7ae-0cdbec67d744`는 각각 5 page·452 row를 완성했고, 같은 ordered
+  manifest `dd893ef82f1f1597a2b65ca6024f31fb7b62ae3f10b13c6d6185365eca2798ba`를 만들었다.
+- 두 attempt는 하나의 hash-only SourceArtifact
+  `70955f24-b61d-4b43-a7de-8e603f6ae459`와 ParseRun
+  `0c7fad64-e49b-4c8c-9929-aece2782354d`로 수렴했다. 첫 parse는 10 accepted·442
+  out-of-scope, 두 번째는 replay였고 result hash는
+  `512c65031cdfe2b734af4245d974390073999a32ad494cd9c94b33c2f165261e`다.
+- ReviewDecision `330cad14-2102-4dcf-a023-93a7368c7efb`가 generation을 승인했다.
+  불변 revision v1 `dc6f5c83-92cc-48e7-8103-76f3fd1a668b`과 v2
+  `2e2b1468-0d41-4635-92a8-c868a80ece1e`를 seal한 뒤 v1 activate, v2 activate, v1 rollback을
+  append-only activation으로 수행했다.
+- 현재 channel version은 `3`, current는 v1이고 entry count는 10이다. canonical active fact-set
+  hash는 `6de8e26c22dcee4a7ce4a6e1a0640999399d126d62124cc1b1d7aefcf9aa66a9`다.
+- public request는 이 current pointer만 읽으며 source API를 호출하지 않는다. actual raw body는
+  저장·fixture·Git·publication에 남기지 않았다.
+- 기존 source row의 `state_changed_at`·`rights_confirmed_at`은 실제 관측 시각이 아니라
+  `2026-08-29T15:00:00Z`(KST 자정) date-precision 값이었고 정확한 live 관측 시각은 보존되지
+  않았다. 원본은 수정·삭제하지 않았다. migration `0010`은 correction
+  `49143c27-d2dd-5fbd-b1dc-4aa3cc002fab`을 append-only로 추가한다. 증거 commit
+  `d23e5707e1fc3bf6e032d459b149b946b0451e00`의 `2026-08-30T02:23:44Z`는 정확한 관측값이 아니라
+  durable source-gate-decision recorded-at upper bound다. helper·DB trigger·review·inspection은
+  base 일치와 chronology를 검사한 이 effective 값만 사용하고 correction update/delete를 거부한다.
+  새 DB bootstrap은 처음부터 exact effective 값을 쓰므로 correction row를 만들지 않는다.
 
 ## typed schema와 migration
 
 하나의 Django app `grocery`가 다음 닫힌 model을 소유한다.
 
 - `SourceConfiguration`: dataset/interface/endpoint allowlist, mode, coverage revision,
-  rights evidence hash, `HASH_ONLY`, logical secret name, timeout·retry·page budget
+  rights evidence hash, `HASH_ONLY`, logical secret name, timeout·retry·page budget,
+  `PLATFORM_SINGLETON` schedule mode와 24시간 cadence
+- `SourceConfigurationGateTimestampCorrection`: 고정 legacy source row의 date-precision 원본을
+  보존하면서 검증된 gate-decision recorded-at upper bound를 append-only로 연결
 - `FetchAttempt`, `PageReceipt`: 상태, attempt ordinal, redacted request shape, ordered page,
   HTTP/provider code, counts, byte length, body SHA-256, failure class
 - `SourceArtifact`: source identity + ordered manifest SHA-256 unique, total bytes,
@@ -183,8 +226,10 @@ transaction과 row lock으로 검사하고 PostgreSQL constraint trigger로 fail
 activation event insert와 pointer update는 `transaction.atomic()`과 row lock으로 한
 transaction에서 처리한다.
 
-첫 migration은 additive create-only다. rollback은 application rollback 후 새 빈 local DB에
-역방향 migration을 실행하며 production contract 단계는 없다.
+현재 migration은 create/add 중심의 forward migration이다. local schema 검증은 새 빈 PostgreSQL에
+`0001`부터 최신까지 처음부터 적용하는 방식이며 역방향 migration을 실행하지 않는다.
+application rollback은 최신 forward schema를 그대로 둔 채, 그 schema를 읽을 수 있다고 검증한
+이전 application SHA와 static으로 되돌린다.
 
 ## 구현·검증 범위
 
@@ -214,14 +259,48 @@ transaction에서 처리한다.
 - endpoint allowlist, HTTPS-only, redirect 거부, 10초 timeout, 4 MB/page와 12 call/attempt
   상한, bounded retry를 적용한다.
 - raw artifact는 저장하지 않고 SHA-256·byte length·redacted receipt만 보존한다.
-- 공개 화면과 Admin evidence에 aT, dataset 15156063, landing URL, 조사일, 확인·검토일,
+- 공개 화면과 운영자 inspection evidence에 aT, dataset 15156063, landing URL, 조사일, 확인·검토일,
   coverage revision을 표시한다.
 - structured audit log는 lifecycle ID·상태만 기록한다. health와 DB/publication readiness,
   last-known-good 나이는 분리한다.
 - PostgreSQL logical dump를 암호화 production backup으로 주장하지 않는다. local gate는
-  disposable DB의 `pg_dump`/`pg_restore`로 row count·hash·current pointer를 검증하고,
+  고정 local Docker socket에서 발견·identity-pin한 exact Compose DB container만 사용한다.
+  backup receipt의 non-secret manifest SHA-256을 out-of-band expected 값으로 반드시 다시 전달하고,
+  disposable DB의 `pg_dump`/`pg_restore`로 row count·canonical publication·current pointer를
+  검증한다. restore 실패 시 같은 invocation이 만든 exact target만 자동 정리하고 부재를 확인하며,
   production에는 managed encryption/PITR·RPO 24h/RTO 4h가 여전히 필요함을 기록한다.
 
+## Phase 0 local candidate 검증 결과
+
+- 실제 browser E2E는 Chromium 152에서 `360x800`, `390x844`, `768x1024`, `1440x900`을
+  통과했고 18개 full-page screenshot을 추적했다. mobile 중복 breadcrumb와 query reflection
+  결함을 수정한 뒤 overflow, touch target, 긴 한국어, 상태 matrix, keyboard·focus와 semantic
+  label을 다시 검증했다.
+- axe-core actual catalog/detail의 WCAG A/AA violation은 0이다. 별도 palette test는 gradient
+  양 끝을 포함해 4.5:1 이상을 보장한다.
+- WhiteNoise compressed manifest storage가 hashed CSS를 만들었고 production-like Gunicorn에서
+  HTML의 hashed reference, CSS `200`, media type과 immutable cache를 확인했다.
+- PostgreSQL custom backup을 새 격리 DB에 restore해 25개 public table, 28개 migration,
+  row count·audit chain·active revision·fact-set hash를 대사했다. 별도 빈 DB에는 migration
+  `0001`~`0010`을 처음부터 적용하고 empty fail-closed health를 확인했다.
+- application rollback rehearsal target `d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`와 현재
+  application이 같은 최신 schema·publication에서
+  live/ready/freshness/catalog/detail·static을 읽는 application rollback을 훈련했다. publication은
+  v2에서 이전 sealed v1으로 append-only rollback했다.
+- production publication command는 external MFA/IAM private job을 전제로 default-off flag,
+  `DEBUG`·Admin·QA off, exact release SHA, fixed reviewer/publisher와 exact Django permission을
+  요구한다. 실제 IAM·role DB grant·actor provisioning은 production 사람 checkpoint다.
+- structured JSON allowlist, request correlation·deploy SHA, liveness/readiness/freshness 판정과
+  fixed scheduler failure code를 검증했다. 실제 alert delivery·retention은 platform checkpoint다.
+- active artifact의 마지막 source 확인 시각은 `2026-08-30T04:00:48.696744Z`이고 36시간 다음
+  확인 경계는 `2026-08-31T16:00:48.696744Z`(`2026-09-01 01:00:48 KST`)다.
+- final locked full suite, clean Git evidence와 고정 900초·9,000요청 profile 결과를
+  `docs/COMPLETION-REPORT.md`에 고정했다. 성능 profile은 nominal `100 ms` cadence, bounded
+  recovery floor `90 ms`, effective paced deadline jitter p95 `100 ms 이하`, catch-up burst `0`,
+  20개 고정 logical virtual-user session의 전원 참여·round-robin과 elapsed `900~903초`를 함께
+  요구했고 최종 corrected run이 모두 통과했다. 실제 in-flight peak는 논리 사용자 수와 분리해
+  기록했다.
+
 ## 작은 가역적 commit 순서
 
 1. `docs: record approved source gate plan` — 이 문서만; rollback은 문서 commit revert.
@@ -244,5 +323,8 @@ transaction에서 처리한다.
 18. `docs: record predeploy completion evidence` — exact release SHA, 배포·rollback 절차와
     production 사람 checkpoint.
 
-각 commit은 구현과 가장 가까운 test를 함께 둔다. 100줄 또는 3개 파일을 넘으면 migration,
-검토 또는 rollback 경계를 더 분리하며 generated migration과 lockfile은 크기 예외를 기록한다.
+각 commit은 한 가지 가역적 의도와 가장 가까운 test를 함께 둔다. generated migration·lockfile·
+browser evidence와 한 불변식을 함께 바꾸는 model/service/test 묶음은 줄 수·파일 수가 커질 수
+있지만, source·schema·parser·review·publication·UI·operations의 rollback 경계는 섞지 않는다.
+실제 history는 이 단일 의도 경계를 기준으로 검증하며 임의의 100줄·3파일 상한을 통과했다고
+주장하지 않는다.
diff --git a/docs/MVP-ACCEPTANCE.md b/docs/MVP-ACCEPTANCE.md
index c7e5007..3d01a95 100644
--- a/docs/MVP-ACCEPTANCE.md
+++ b/docs/MVP-ACCEPTANCE.md
@@ -9,13 +9,17 @@ source, 승인된 권리 판단, 고정 artifact와 운영과 같은 build에서
 
 ## 게이트 0: Git 기준선
 
+아래 결과는 구현 파일을 처음 변경하기 전 기준선 검증 기록이다. 현재 구현 저장소의 파일 수와
+runtime 상태를 뜻하지 않는다.
+
 첫 파일 변경 전에 구현 세션은 다음을 검증한다.
 
-- [ ] 현재 경로가 `audience-foundry-grocery-seasonality`이고 branch는 `main`이다.
-- [ ] 이 문서 계약 commit의 부모가 공통 정책 기준선 `0cc95e70824e02a78207fe983f076e38a59c764f`이다.
-- [ ] 추적 파일은 README와 공통 정책 4개, 제품 문서 6개로 정확히 11개다.
-- [ ] remote가 없고 working tree가 깨끗하며 `git fsck`가 통과한다.
-- [ ] 런타임 코드, dependency, lockfile, credential, 수집 데이터와 배포 설정이 아직 없다.
+- [x] 현재 경로가 `audience-foundry-grocery-seasonality`이고 branch는 `main`이었다.
+- [x] 이 문서 계약 commit의 부모가 공통 정책 기준선 `0cc95e70824e02a78207fe983f076e38a59c764f`였다.
+- [x] 추적 파일은 README와 공통 정책 4개, 제품 문서 6개로 정확히 11개였다.
+- [x] remote가 없고 working tree가 깨끗하며 `git fsck`가 통과했다.
+- [x] 첫 변경 전에는 runtime 코드, dependency, lockfile, credential, 수집 데이터와 배포 설정이
+  없었다.
 
 기준이 다르면 구현을 시작하지 않고 차이를 사람에게 보고한다.
 
@@ -23,31 +27,32 @@ source, 승인된 권리 판단, 고정 artifact와 운영과 같은 build에서
 
 source-dependent schema나 adapter를 만들기 전에 저장소 소유자가 다음 실제 증거를 승인한다.
 
-- [ ] 공식 소유자와 공공데이터포털의
+- [x] 공식 소유자와 공공데이터포털의
   [최근일자 도·소매가격정보 API `15156063`](https://www.data.go.kr/data/15156063/openapi.do)
   landing URL, API 문서 버전과 운영 host가 기록되어 있다.
-- [ ] 사람이 발급받아 secret으로 주입한 key로 공식 HTTPS endpoint의 실제 응답을 재현한다.
-- [ ] 개발·운영 quota, 호출 단위, pagination, timeout, `429`, provider error와 이용 가능 시간이
-  실제 요청과 공식 문서로 확인된다.
-- [ ] 원시 응답 보존, 내부 변환, 파생 가격 사실 공개, cache와 출처 표시의 허용 범위를 각각
+- [x] 사람이 발급받아 secret으로 주입한 key로 공식 HTTPS endpoint의 실제 응답을 재현했다.
+- [x] 개발계정 quota 10,000, pagination, timeout·provider error 계약을 공식 문서와 실제 요청으로
+  확인했다. quota 기간·초당 한도는 문서에 없어 호출당 12회로 더 좁혔고, 실제 429를 만들기 위한
+  quota 소진은 하지 않았다.
+- [x] 원시 응답 보존, 내부 변환, 파생 가격 사실 공개, cache와 출처 표시의 허용 범위를 각각
   판정한다.
-- [ ] JSON 또는 XML의 실제 field, type, encoding, missing·sentinel, 중복과 error envelope를
+- [x] JSON 또는 XML의 실제 field, type, encoding, missing·sentinel, 중복과 error envelope를
   기록한다.
-- [ ] 소매·채소·과일 범위에서 category, item, variety, grade, unit와 unit size code를
+- [x] 소매·채소·과일 범위에서 category, item, variety, grade, unit와 unit size code를
   안정적으로 식별할 수 있다. region·market field가 있으면 그 code를 검증하고, 없으면
   제공자 문서와 실제 응답으로 aggregate 범위와 안정적인 `coverage_identity`를 증명한다.
-- [ ] 현재 조사 평균과 1주·1개월·1년 전 평균의 비교기간 의미, coverage, 단위와 산출
+- [x] 현재 조사 평균과 1주·1개월·1년 전 평균의 비교기간 의미, coverage, 단위와 산출
   의미가 비교 가능한 같은 series로 확인된다. 정확한 reference date는 source가 제공할 때
   검증하고, 제공하지 않으면 `SOURCE_REFERENCE_DATE_UNAVAILABLE`을 보존한다.
-- [ ] 휴일·비조사일, 늦은 갱신, 부분 응답과 과거 날짜 요청에서 source가 반환하는 기준일
-  의미를 확인한다.
-- [ ] 동일 요청 반복의 stable field와 변동 field, content hash와 idempotency 규칙을
+- [x] API에 과거 날짜 입력과 reference date field가 없음을 확인했고, 휴일·비조사일 설명과 row별
+  `exmn_ymd`를 보존한다. partial page·total mismatch는 실패 폐쇄하며 날짜를 역산하지 않는다.
+- [x] 동일 요청 반복의 stable field와 변동 field, content hash와 idempotency 규칙을
   재현한다.
-- [ ] 최소 채소 5개·과일 5개의 서로 다른 exact series를 포함한 bounded canary matrix로
+- [x] 최소 채소 5개·과일 5개의 서로 다른 exact series를 포함한 bounded canary matrix로
   code, 단위, 결측과 반복 조회를 검증한다. source가 이 수를 제공하지 않거나 검증된 호출
   예산 안에서 재현할 수 없으면 범위를 임의로 채우지 않고 path 선택을 다시 승인한다.
-- [ ] 허용된 호출 예산으로 정기 수집, 대사, retry와 운영자 확인을 지속할 수 있다.
-- [ ] 전체 pagination을 완성하는 한 논리적 획득을 `FetchAttempt` 하나로 기록하고 각 page의
+- [x] 허용된 호출 예산으로 정기 수집, 대사, retry와 운영자 확인을 지속할 수 있다.
+- [x] 전체 pagination을 완성하는 한 논리적 획득을 `FetchAttempt` 하나로 기록하고 각 page의
   순서·row count·body hash를 대사한다. 논리적 재시도는 새 attempt이며 서로 다른 attempt의
   page를 한 artifact로 합치지 않는다.
 
@@ -59,12 +64,13 @@ evidence 실패를 fixture, 비공식 mirror, HTML scraping 또는 quota 우회
 ### A. 최근 비교 path
 
 게이트 1 전체를 통과하면 현재 KAMIS 소매 조사 평균과 같은 series의 1주·1개월·1년 전
-reference를 공개한다.
+reference를 공개한다. 이번 구현의 선택 path다.
 
 ### B. current-only path
 
 현재값의 identity·단위·권리·운영성은 통과했지만 reference 기간의 의미나 같은 series임을
 증명하지 못하면 현재 조사 평균만 공개한다. 차액, 퍼센트와 방향 문구는 렌더링하지 않는다.
+이번 gate에서는 A가 통과했으므로 B는 **해당 없음(N/A)**이며 미통과 항목으로 세지 않는다.
 
 ### C. 정적 월별 file path
 
@@ -77,6 +83,8 @@ SourceArtifact → ParseRun → MonthlyRetailPriceSnapshot → ReviewDecision 
 PublicationRevision → PublicationActivation`을 통과해야 한다. 별도
 `STATIC_MONTHLY` publication channel·route·copy·rollback을 사용하고 recent comparison
 snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표현하지 않는다.
+이번 gate에서는 API 운영성이 통과해 A를 선택했으므로 C는 **해당 없음(N/A)**이며 file gate나
+월별 module을 구현하지 않는다.
 
 ### D. stop
 
@@ -87,98 +95,107 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
 
 ### 수집·감사·공개
 
-- [ ] 한 실제 공식 응답이 `FetchAttempt → SourceArtifact → ParseRun → typed
+- [x] 한 실제 공식 응답이 `FetchAttempt → SourceArtifact → ParseRun → typed
   RetailPriceSnapshot/ReferencePrice/PriceChangeFact → ReviewDecision → PublicationRevision`
   전 단계를 거쳐 사람 승인 후 activation으로 공개된다.
-- [ ] 각 단계에서 source, 이전 단계, 실행자 또는 process, 시각, code·parser version, hash와
+- [x] 각 단계에서 source, 이전 단계, 실행자 또는 process, 시각, code·parser version, hash와
   상태를 역추적할 수 있다.
-- [ ] 동일 ordered page manifest를 다시 획득하면 새 `FetchAttempt`는 생기지만
+- [x] 동일 ordered page manifest를 다시 획득하면 새 `FetchAttempt`는 생기지만
   `SourceArtifact`는 중복되지 않는다.
-- [ ] 같은 content의 재확인은 source의 마지막 성공 확인 상태만 갱신하고 artifact,
+- [x] 같은 content의 재확인은 source의 마지막 성공 확인 상태만 갱신하고 artifact,
   source 조사일과 공개 데이터 freshness를 바꾸지 않는다.
-- [ ] 동일 artifact와 parser version 재실행은 같은 typed row 집합 hash를 만들며 snapshot을
+- [x] 동일 artifact와 parser version 재실행은 같은 typed row 집합 hash를 만들며 snapshot을
   중복하지 않는다.
-- [ ] `fetched_at`만 다른 재수집은 새 content나 새 publication의 근거가 되지 않는다.
-- [ ] parser version이 바뀌면 새 `ParseRun`과 review candidate가 생기고 승인 전에는 공개되지
+- [x] `fetched_at`만 다른 재수집은 새 content나 새 publication의 근거가 되지 않는다.
+- [x] parser version이 바뀌면 새 `ParseRun`과 review candidate가 생기고 승인 전에는 공개되지
   않는다.
-- [ ] 승인 revision의 row count, code별 count, coverage, missing·quarantine count와 집합
+- [x] 승인 revision의 row count, code별 count, coverage, missing·quarantine count와 집합
   hash가 대사 보고서와 일치한다.
-- [ ] 새 source 실패 중에도 last-known-good가 유지되고 사용자에게 그 기준일과 검토일을
+- [x] 새 source 실패 중에도 last-known-good가 유지되고 사용자에게 그 기준일과 검토일을
   표시한다.
 
 ### 사용자 읽기
 
-- [ ] 소매·채소 또는 과일 목록에서 공식 품목명으로 검색하고 한 exact series 상세로 이동할
+- [x] 소매·채소 또는 과일 목록에서 공식 품목명으로 검색하고 한 exact series 상세로 이동할
   수 있다.
-- [ ] 상세 화면은 item·variety·grade·unit·unit size와 `market` 또는 검증된 aggregate
+- [x] 상세 화면은 item·variety·grade·unit·unit size와 `market` 또는 검증된 aggregate
   coverage를 source가 제공한 범위 안에서 명확히 표시한다.
-- [ ] 현재값 8,000원과 1주 기준값 10,000원이면 `2,000원 낮음`, `-20.0%`를 표시한다.
-- [ ] 현재값 12,500원과 1개월 기준값 10,000원이면 `2,500원 높음`, `+25.0%`를 표시한다.
-- [ ] 현재값과 1년 기준값이 같으면 `같음`, `0.0%`를 표시한다.
-- [ ] 기준값이 `0`이거나 없으면 차액·퍼센트를 계산하지 않고 `비교 정보 없음`을 표시한다.
-- [ ] 가격 가까이에 `KAMIS 소매 조사 평균`, source 조사일, coverage, 단위와 publication
+- [x] 현재값 8,000원과 1주 기준값 10,000원이면 `2,000원 낮음`, `-20.0%`를 표시한다.
+- [x] 현재값 12,500원과 1개월 기준값 10,000원이면 `2,500원 높음`, `+25.0%`를 표시한다.
+- [x] 현재값과 1년 기준값이 같으면 `같음`, `0.0%`를 표시한다.
+- [x] 기준값이 `0`이거나 없으면 차액·퍼센트를 계산하지 않고 `비교 정보 없음`을 표시한다.
+- [x] 가격 가까이에 `KAMIS 소매 조사 평균`, source 조사일, coverage, 단위와 publication
   검토일을 표시한다. reference별 실제 날짜가 있으면 그 날짜를 표시하고 없으면
   `source가 비교 기준일을 별도로 제공하지 않음`을 표시한다.
-- [ ] 방향은 중립적 사실로만 표현하며 구매·품질·영양·제철·미래 가격 판단을 덧붙이지
+- [x] 방향은 중립적 사실로만 표현하며 구매·품질·영양·제철·미래 가격 판단을 덧붙이지
   않는다.
 
 ## 필수 음성 시나리오
 
-- [ ] 소매와 도매, 채소와 과일, 서로 다른 item·variety·grade·unit·unit size의 값은 비교하지
+- [x] 소매와 도매, 채소와 과일, 서로 다른 item·variety·grade·unit·unit size의 값은 비교하지
   않는다. source가 region·market을 제공하면 그 code가 다른 값도, 제공하지 않으면 검증된
   aggregate coverage가 다른 값도 비교하지 않는다.
-- [ ] 이름이 비슷하다는 이유로 다른 code를 자동 결합하지 않는다. code가 사라지거나 바뀌면
+- [x] 이름이 비슷하다는 이유로 다른 code를 자동 결합하지 않는다. code가 사라지거나 바뀌면
   새 review 없이는 이전 series에 연결하지 않는다.
-- [ ] 임의 kg 환산, 지역 간 평균 합성, 시장 최저가, 쇼핑몰 가격, 할인과 배송비를 만들지
+- [x] 임의 kg 환산, 지역 간 평균 합성, 시장 최저가, 쇼핑몰 가격, 할인과 배송비를 만들지
   않는다.
-- [ ] `null`, 빈 문자열, sentinel, 음수, 잘못된 decimal, 알 수 없는 단위와 중복 identity는
+- [x] `null`, 빈 문자열, sentinel, 음수, 잘못된 decimal, 알 수 없는 단위와 중복 identity는
   `0원`으로 보정하지 않고 격리한다.
-- [ ] 기준값이 `0`일 때 division을 수행하지 않는다. float rounding을 사용하지 않는다.
-- [ ] source가 주지 않은 effective date나 recorded time을 fetch·publish time으로 대신하지
+- [x] 기준값이 `0`일 때 division을 수행하지 않는다. float rounding을 사용하지 않는다.
+- [x] source가 주지 않은 effective date나 recorded time을 fetch·publish time으로 대신하지
   않는다.
-- [ ] HTTP timeout, `429`, 일시적 `5xx`, TLS 오류, 허용되지 않은 redirect, 응답 크기 초과,
+- [x] HTTP timeout, `429`, 일시적 `5xx`, TLS 오류, 허용되지 않은 redirect, 응답 크기 초과,
   content type 변경과 provider error는 성공으로 기록하지 않는다.
-- [ ] field 제거·추가, type·encoding·unit 변경과 duplicate는 publication을 차단한다. 첫 승인
+- [x] field 제거·추가, type·encoding·unit 변경과 duplicate는 publication을 차단한다. 첫 승인
   generation의 전체 key set·상태별 count와 비교한 후속 key 소실·추가·차원 변경은 사람
   검토 전까지 last-known-good를 유지한다.
-- [ ] 일부 row만 parse되거나 publication transaction이 실패하면 current pointer를 바꾸지
+- [x] 일부 row만 parse되거나 publication transaction이 실패하면 current pointer를 바꾸지
   않고 전부 rollback한다.
-- [ ] 늦게 도착한 과거 기준일 응답은 명시적 review 없이 최신 승인 revision을 대체하지
+- [x] 늦게 도착한 과거 기준일 응답은 명시적 review 없이 최신 승인 revision을 대체하지
   않는다.
-- [ ] current-only path에서 reference 값, 차액, 퍼센트와 방향 문구를 노출하지 않는다.
-- [ ] 정적 월별 file path를 현재값, 실시간, 최저가, 예측 또는 개인 추천으로 표현하지 않는다.
-- [ ] 개인정보가 포함된 unexpected field는 domain snapshot과 public output에 복사하지
+- 해당 없음(N/A) — B current-only path는 선택되지 않아 route·copy가 존재하지 않는다. 향후 B를
+  선택하면 reference 값, 차액, 퍼센트와 방향 문구를 노출하지 않는 음성 test가 필수다.
+- 해당 없음(N/A) — C 정적 월별 file path와 월별 module은 선택되지 않았다. 향후 C를 선택하면
+  현재값, 실시간, 최저가, 예측 또는 개인 추천 표현을 막는 음성 test가 필수다.
+- [x] 개인정보가 포함된 unexpected field는 domain snapshot과 public output에 복사하지
   않는다.
-- [ ] 인증되지 않았거나 권한이 부족한 사용자는 Admin, source 설정, review와 publication에
+- [x] 인증되지 않았거나 권한이 부족한 사용자는 Admin, source 설정, review와 publication에
   접근하지 못한다.
-- [ ] API key, 전체 query string, raw body, 검색어와 운영자 개인정보가 Git, log, error,
+- [x] API key, 전체 query string, raw body, 검색어와 운영자 개인정보가 Git, log, error,
   analytics와 public response에 나타나지 않는다.
 
 ## 개인정보·license 인수 기준
 
-- [ ] public 사용에는 계정, 주소, 위치, 장바구니와 구매 이력이 필요하지 않다.
-- [ ] 검색어는 DB, 공개 방문자 session, cache, analytics와 application log에 저장하지
+- [x] public 사용에는 계정, 주소, 위치, 장바구니와 구매 이력이 필요하지 않다.
+- [x] 검색어는 DB, 공개 방문자 session, cache, analytics와 application log에 저장하지
   않는다. Admin 인증 session은 별도 보안 정책을 따른다.
-- [ ] source·domain·public allowlist에 없는 field는 저장·공개하지 않는다.
-- [ ] raw bytes 보존 권리가 없으면 body 저장은 실패 폐쇄되고 SHA-256, byte length, 최소
+- [x] source·domain·public allowlist에 없는 field는 저장·공개하지 않는다.
+- [x] raw bytes 보존 권리가 없으면 body 저장은 실패 폐쇄되고 SHA-256, byte length, 최소
   receipt와 정규화 사실만 남는다.
-- [ ] source 이름, landing URL, recent 조사일 또는 monthly row 기준 연월, coverage, 단위,
+- [x] source 이름, landing URL, recent 조사일 또는 monthly row 기준 연월, coverage, 단위,
   변환 설명과 검토일을 공개 화면에 표시한다.
-- [ ] dependency와 정적 자산의 license·notice 의무가 `THIRD_PARTY_NOTICES.md`와 실제 배포에
-  반영된다.
+- [x] dependency, 정적 자산과 browser assurance 도구의 license·notice 의무가
+  `THIRD_PARTY_NOTICES.md`에 반영됐다.
+
+실제 production artifact에 license·bundled notice를 포함하는 일은 local source tree와 vendor
+CLI 검증과 분리된 **production 사람 checkpoint**다. platform packaging 방식이 선택된 뒤 실제
+artifact를 검사하기 전에는 이 항목을 통과로 기록하지 않는다.
+
+## 월별 과거 패턴 module gate — 해당 없음(N/A)
 
-## 월별 과거 패턴 module gate
+이번 구현은 A path이므로 이 module은 별도 제품결정 미승인·비활성이다. 아래 항목은 이번
+acceptance에서 평가하지 않는다.
 
 내부 repository 이름만으로 이 module을 활성화하지 않는다. 별도 제품 결정을 승인하기 전에
 다음을 모두 증명한다.
 
-- [ ] 공식 source가 최소 3개 완전 연도의 월별 retail rows를 제공하고 공개·보존 권리가 있다.
-- [ ] 기간 전체에서 item·variety·grade·unit code와 coverage identity가 안정적이고,
+- N/A — 공식 source가 최소 3개 완전 연도의 월별 retail rows를 제공하고 공개·보존 권리가 있다.
+- N/A — 기간 전체에서 item·variety·grade·unit code와 coverage identity가 안정적이고,
   region·market code가 제공되면 그 의미도 안정적이거나 사람이 승인한 명시적 migration
   map이 있다.
-- [ ] 결측 월, 조사 빈도, 시장 구성 변화와 명목가격의 한계를 사용자에게 표시할 수 있다.
-- [ ] 한두 달의 값이나 단순 최저 월을 농산물의 자연적 제철·품질·가용성으로 해석하지 않는다.
-- [ ] 독립된 source configuration, parse version, review와 publication rollback이 있다.
+- N/A — 결측 월, 조사 빈도, 시장 구성 변화와 명목가격의 한계를 사용자에게 표시할 수 있다.
+- N/A — 한두 달의 값이나 단순 최저 월을 농산물의 자연적 제철·품질·가용성으로 해석하지 않는다.
+- N/A — 독립된 source configuration, parse version, review와 publication rollback이 있다.
 
 통과해도 첫 public 명칭은 `월별 과거 가격 패턴`이다. 계절성 추정, 구매 추천과 forecast는
 새 근거와 새 제품 결정 없이는 범위 밖이다.
@@ -189,54 +206,102 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
 뜻하며 production platform·database·credential·domain이 정해졌거나 실제 배포가 끝났다는
 뜻이 아니다. 네이티브 모바일 앱, 앱스토어 배포와 별도 SPA는 범위 밖이다.
 
-- [ ] 문서가 허용한 source path로 실제 generation 하나가 수집·검수·publication되어 핵심
-  사용자 폐쇄 루프가 last-known-good revision만 읽는다.
-- [ ] Django server-rendered responsive web 하나가 desktop과 mobile을 함께 지원한다.
-- [ ] 실제 배포에서 사용할 migration, release SHA, deploy·application rollback·publication
-  rollback 명령이 runbook과 clean Git 상태로 재현된다.
-- [ ] 실제 배포에 필요한 platform, PostgreSQL, secret injection, domain·DNS와 운영자 계정은
-  사람 전용 잔여 작업으로 분리되어 있다.
+### local candidate 필수 항목
 
-- [ ] 깨끗한 잠금 설치에서 Python `3.14.7`, Django `5.2.17`, PostgreSQL `18.6`, uv
+- [x] 문서가 허용한 source path로 실제 generation 하나가 수집·검수·publication되어 핵심
+  사용자 폐쇄 루프가 last-known-good revision만 읽는다.
+- [x] Django server-rendered responsive web 하나가 desktop과 mobile을 함께 지원한다.
+- [x] clean Git의 exact release SHA에서 locked dependency, forward migration, collectstatic,
+  production-like process와 smoke를 다시 만들 수 있고, platform과 무관한 deploy 순서 및
+  application·publication rollback 절차가 runbook에서 재현된다.
+- [x] migration 검증은 새 빈 local DB에 `0001`부터 최신까지 forward 적용한다. application
+  rollback은 최신 forward schema를 유지하고 검증된 이전 SHA로 code·static만 되돌리며 reverse
+  migration을 실행하지 않는다.
+
+- [x] 깨끗한 잠금 설치에서 Python `3.14.7`, Django `5.2.17`, PostgreSQL `18.6`, uv
   `0.12.6`이 실제 실행 버전으로 확인된다.
-- [ ] formatter, lint, type, unit, integration, parser replay, negative route와 concurrent
+- [x] formatter, lint, type, unit, integration, parser replay, negative route와 concurrent
   publication test가 통과한다.
-- [ ] 새 빈 DB와 운영 복제 DB에서 migration, `django check`와 `django check --deploy`가
-  통과한다.
-- [ ] 운영은 HTTPS, `DEBUG=False`, 정확한 host·CSRF 설정, HSTS와 secure cookie를 사용한다.
-- [ ] Admin은 운영자별 최소 권한, MFA 또는 동등한 강한 인증과 로그인 제한을 사용한다.
-- [ ] local production-like 설정에서 env secret injection contract·rotation 절차, structured
+- [x] 새 빈 local DB에 모든 migration을 처음부터 적용해 drift·`django check`를 통과했고,
+  live 200·ready/freshness 503·empty catalog 200의 의도한 fail-closed 상태를 확인했다.
+- [x] 실제 publication을 복원한 격리 local DB에서 migration inventory·`migrate --check`·
+  `django check`와 live/ready/freshness/catalog 200을 확인했다.
+- [x] production settings contract가 `DEBUG=False`, exact host·HTTPS CSRF, secure cookie,
+  SSL redirect와 HSTS warning gate를 fail-closed 검증한다.
+- [x] production publication command가 Admin·QA off, default-off flag, exact release lock과
+  역할별 고정 non-login 최소 Django permission을 요구한다.
+- [x] local production-like 설정에서 env secret injection contract·rotation 절차, structured
   log, liveness·readiness와 source freshness alert 판단이 동작한다. 실제 production secret
   주입은 배포 checkpoint에 남긴다.
-- [ ] 연속 fetch 실패, quarantine 증가, 대사 불일치, publication·backup 실패가 운영자에게
-  경보된다.
-- [ ] disposable PostgreSQL의 실제 `pg_dump`/`pg_restore`가 빈 환경에서 승인 revision,
-  audit chain, row count·hash·current pointer를 복원한다. production의 매일 암호화 backup,
-  point-in-time recovery, `RPO 24시간`·`RTO 4시간`은 platform 선택 뒤 별도 확인한다.
-- [ ] 이전 application과 이전 승인 publication으로 각각 rollback하는 훈련이 성공한다.
-- [ ] secret, dependency vulnerability와 license 검사에 해결되지 않은 차단 항목이 없다.
+- [x] source configuration이 platform-owned singleton 실행 방식과 24시간 cadence를 typed하게
+  기록하며 scheduler가 ingestion 뒤 review·seal·activation을 자동 실행하지 않는다.
+- [x] legacy source gate의 자정 date-precision 원본 두 값은 변경하지 않고 append-only correction으로
+  durable gate-decision recorded-at upper bound를 연결한다. 정확한 live 관측 시각은 보존되지
+  않았음을 명시하고, base·chronology 검증과 update/delete 거부 및 새 DB exact bootstrap을
+  test한다.
+- [x] 연속 fetch 실패, quarantine 증가, 대사 불일치, freshness와 publication 실패를 fixed
+  code·exit·DB audit로 판정하는 local 규칙이 있다.
+- [x] disposable PostgreSQL의 실제 `pg_dump`/`pg_restore`가 승인 revision, audit chain,
+  row count·hash·current pointer를 새 격리 DB에 복원했다.
+- [x] hardened local rehearsal이 mandatory out-of-band manifest SHA-256, exact local Docker
+  socket/container identity pin, source↔restore canonical publication exact parity와 실패 시
+  invocation-owned target 자동 cleanup·부재 확인을 함께 통과한다.
+- [x] application rollback rehearsal target
+  `d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`와 이전 승인 publication으로 각각 rollback하는
+  훈련이 성공했다.
+- [x] secret, dependency vulnerability와 license 검사에 해결되지 않은 차단 항목이 없다.
+
+### production 사람 checkpoint — local candidate 필수 항목이 아님
+
+아래 항목은 실제 platform·계정·domain을 선택하고 사람이 승인한 뒤에만 실행한다. 미완료 상태를
+local candidate 실패로 세거나, 반대로 local 검증으로 통과했다고 표시하지 않는다.
+
+- [x] 필요한 platform, PostgreSQL, secret injection, domain·DNS, 운영자 계정과 역할이 사람 전용
+  잔여 작업으로 분리되어 있다.
+- [ ] 선택된 production PostgreSQL 복제 DB에서 migration plan·lock·시간과 `check --deploy`를
+  검증한다.
+- [ ] 실제 production HTTPS·certificate, domain 전체 HSTS·preload와 trusted proxy를 검증한다.
+- [ ] 실제 운영자 external MFA·IAM, 역할별 DB credential·grant와 login 제한을 검증한다.
+- [ ] application·static·migration·notice allowlist를 적용한 vendor-specific production artifact를
+  만들고 bundled license notice 포함 여부를 검사한다.
+- [ ] fixed failure signal과 production backup 실패가 실제 운영자 alert route로 전달되는지
+  검증한다.
+- [ ] production singleton scheduler의 24시간 cadence, overlap 방지, outbound HTTPS allowlist와
+  ingestion/freshness failure alert 전달을 실제 platform에서 검증한다.
+- [ ] 매일 암호화 backup, point-in-time recovery와 `RPO 24시간`·`RTO 4시간`을 실제 managed
+  platform에서 검증한다.
+- [ ] vendor-specific deploy·traffic switch·application rollback CLI를 exact account/application
+  scope와 함께 승인하고 실제 배포를 수행한다.
 
 ## responsive browser·성능·접근성 인수 기준
 
 실제 browser와 end-to-end test로 `360x800`, `390x844`, `768x1024`, `1440x900`을 각각
 검수하고 viewport별 screenshot을 completion evidence에 연결한다.
 
-- [ ] 어느 viewport에도 document 가로 scroll이 없다.
-- [ ] typography와 heading·metadata 계층이 읽기 쉽고 interactive touch target이 충분하다.
-- [ ] mobile에서 검색 form 입력·제출·validation error 확인·수정이 실제 동작한다.
-- [ ] 긴 한국어 품목·품종·등급·단위·출처·freshness가 잘리거나 겹치지 않는다.
-- [ ] loading, empty, unavailable, stale, validation과 server error 상태가 결정적으로
+- [x] 어느 viewport에도 document 가로 scroll이 없다.
+- [x] typography와 heading·metadata 계층이 읽기 쉽고 interactive touch target이 충분하다.
+- [x] mobile에서 검색 form 입력·제출·validation error 확인·수정이 실제 동작한다.
+- [x] 긴 한국어 품목·품종·등급·단위·출처·freshness가 잘리거나 겹치지 않는다.
+- [x] loading, empty, unavailable, stale, validation과 server error 상태가 결정적으로
   재현되고 screenshot·test로 검증된다.
-- [ ] keyboard-only navigation 순서와 visible focus가 동작한다.
-- [ ] semantic landmark·heading·form label·error association과 screen reader accessible name이
+- [x] keyboard-only navigation 순서와 visible focus가 동작한다.
+- [x] semantic landmark·heading·form label·error association과 screen reader accessible name이
   자동 검사 및 browser 검수에서 유효하다.
-- [ ] success·warning·error·direction을 색상만으로 전달하지 않는다.
+- [x] success·warning·error·direction을 색상만으로 전달하지 않는다.
 
 승인된 catalog 크기로 운영과 같은 환경에서 15분 동안 평균 10 requests/s, 동시 사용자 20명,
 목록·검색 70%와 상세 30%의 read-only 부하를 건다. 응답 p95는 500 ms 이하, `5xx`는 0.5%
 미만이며 DB 연결 고갈과 revision 혼합이 없어야 한다. 이 profile을 넘는 실제 수요가 측정될
 때만 별도 cache를 검토한다.
 
+- [x] 고정 900초 profile이 정확히 9,000요청·70:30 workload를 완료하고 p95 500 ms 이하,
+  5xx 0.5% 미만, error 0과 revision 단일값을 만족한다. 20개의 고정 logical virtual-user
+  session이 모두 참여하고 요청 index를 round-robin으로 나누며, 실제 in-flight 동시 요청 수는
+  별도 측정해 20을 넘지 않는다. effective paced deadline
+  대비 schedule jitter p95는 `100 ms 이하`, nominal cadence는 `100 ms`, bounded recovery 중
+  실제 submit interval 최솟값은 `90 ms 이상`, catch-up burst는 `0`, 전체 elapsed는
+  `900~903초`여야 한다.
+
 핵심 page와 form은 keyboard만으로 사용할 수 있고 visible focus, 한국어 label, 오류 요약,
 logical heading, 충분한 contrast와 screen reader로 읽히는 방향 문구를 제공해야 한다.
 
diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
new file mode 100644
index 0000000..67a4b3c
--- /dev/null
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -0,0 +1,841 @@
+# Phase 0 배포 직전 운영 런북
+
+## 문서 상태와 범위
+
+이 문서는 현재 저장소를 **Phase 0 배포 직전 production candidate**로 검증하고, 사람이
+production platform을 선택한 뒤 따라야 할 안전한 배포·복구 순서를 정의한다. 실제
+production platform, PostgreSQL, credential, domain·DNS는 아직 선택하거나 변경하지 않았고,
+이 문서는 실제 배포 완료를 주장하지 않는다.
+
+명령의 `$NAME`은 운영자가 승인된 CI 변수 또는 managed configuration·secret store에서
+주입해야 하는 placeholder다. 이 문서, shell history, process argument, URL, log, ticket와
+receipt에 실제 secret 값을 적지 않는다. 모든 명령은 별도 설명이 없으면 exact release의
+repository root에서 실행한다.
+
+다음은 사람 전용 checkpoint다.
+
+- production platform·database·domain·DNS 선택과 생성
+- production credential·secret store 주입과 provider key 발급·폐기
+- production 배포·traffic switch·application rollback
+- production reviewer·publisher identity, MFA, 최소권한과 첫 publication
+- 파괴적 migration, database 삭제·교체, in-place restore와 PITR 전환
+- 고정 제품 결정, source 권리 또는 공개 범위 변경
+
+이 checkpoint는 명시적 승인과 change record 없이는 실행하지 않는다.
+
+## 현재 candidate가 제공하는 것과 남은 차단점
+
+| 영역 | 현재 제공 범위 | production 판단 |
+|---|---|---|
+| web | Django WSGI와 고정 Gunicorn dependency | platform process·HTTPS·traffic switch가 필요 |
+| schema | `0001`부터 `0010`까지 create/add 중심 migration과 DB trigger | production 복제 DB에서 plan·lock·시간 검증 필요 |
+| ingestion | `ingest_kamis_recent`의 bounded fetch·parse·audit와 typed 24시간 singleton schedule 계약 | managed `KAMIS_API_KEY`, 실제 singleton scheduler·egress·alert 필요 |
+| review/publication | local rehearsal과 fail-closed production command boundary | external MFA·IAM, 역할별 DB credential과 실제 actor provisioning 필요 |
+| public read | active `RECENT_RETAIL` revision만 읽는 SSR | production DB의 승인 pointer와 smoke test 필요 |
+| health | liveness, readiness, freshness endpoint와 scheduler command | platform probe·alert routing이 필요 |
+| backup | local Compose DB용 custom dump·isolated restore 검증 | production backup, encryption, PITR를 제공하지 않음 |
+| logs | allowlist 기반 JSON event, 고정 message code와 exact deploy version | log 수집·보존·alert 담당자와 platform access-log 제거가 필요 |
+
+`approve_recent_generation`, `seal_recent_publication`과
+`transition_recent_publication`은 local rehearsal과 production 경계를 구분한다. production에서는
+Admin·QA가 꺼지고 별도 control-plane enable, exact release SHA, command별 고정 non-login actor와
+최소 permission, fixed reason code가 모두 맞아야 한다. enable flag는 인증 수단이 아니므로
+external MFA·IAM, 역할별 DB credential·grant와 실제 actor provisioning이 승인되기 전에는 첫
+production publication과 traffic 공개가 차단된다. production에서 `DEBUG`를 켜거나 DB pointer를
+직접 수정해 우회하지 않는다.
+
+## 안전 등급
+
+- **읽기 전용**: Git 확인, migration plan, Django check, health 조회, freshness 조회.
+- **비파괴적 생성**: static build, local backup 파일, 새 격리 restore database 생성.
+- **상태 변경·가역적**: forward migration, ingestion, approve, seal, publication pointer 전환.
+  실행 전에 대상·actor·expected state와 rollback을 기록한다.
+- **파괴적**: reverse migration, table/database 삭제, 기존 database에 restore, volume 삭제,
+  backup 삭제, credential 폐기, DNS·traffic의 되돌릴 수 없는 변경. 단, identity-pin한 고정 local
+  container에서 같은 restore invocation이 새로 만든 exact disposable target을 실패 보상으로
+  삭제하고 부재를 확인하는 동작만 자동 허용한다. 그 밖에는 사람 승인 없이는 금지한다.
+
+특히 다음 명령 또는 동등 작업은 이 런북의 자동 절차에 포함되지 않는다.
+
+- `migrate ... zero`, 과거 migration으로의 reverse migration, `--fake`, `flush`
+- 기존 database를 대상으로 한 `pg_restore --clean` 또는 schema overwrite
+- 일반 `dropdb`, `DROP DATABASE`, `DROP TABLE`, `docker compose down -v` 또는 pre-existing target
+  삭제. 위의 invocation-owned disposable restore target 실패 보상만 예외다.
+- 공유 static root의 `collectstatic --clear`
+- Git history rewrite, 강제 push, broad checkout/reset
+- backup directory의 재귀 삭제
+
+필요해지면 exact target을 read-only로 다시 확인하고 별도 파괴적 change approval을 받는다.
+
+## 환경과 역할 계약
+
+### application configuration 이름
+
+production web process에는 최소한 다음 이름이 필요하다.
+
+- `DJANGO_DEBUG`
+- `ADMIN_ENABLED`
+- `DJANGO_SECRET_KEY`
+- `DJANGO_ALLOWED_HOSTS`
+- `DJANGO_CSRF_TRUSTED_ORIGINS`
+- `DATABASE_URL`
+- `DEPLOY_VERSION`
+- `DJANGO_SECURE_SSL_REDIRECT`
+- `DJANGO_SECURE_HSTS_SECONDS`
+- `DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS`
+- `DJANGO_SECURE_HSTS_PRELOAD`
+- `DJANGO_TRUST_X_FORWARDED_PROTO`
+- `DATABASE_CONN_MAX_AGE`
+- `KAMIS_CONFIRMATION_MAX_AGE_HOURS`
+- `QA_STATE_PREVIEWS_ENABLED`
+- `CONTROL_PLANE_OPERATIONS_ENABLED`
+
+production validation은 debug 비활성화, Admin 비활성화, 충분한 Django secret, wildcard가 아닌
+host, path·query·credential이 없는 HTTPS CSRF origin을 요구한다. `DEPLOY_VERSION`에는 배포할
+exact lowercase full Git SHA를 주입한다. QA preview는 production에서 활성화할 수 없다. HSTS
+subdomain·preload와 forwarded-proto 신뢰는 기본 비활성이고, domain 전체와 trusted proxy hop을
+사람이 확인한 뒤에만 명시적으로 켠다.
+
+ingestion worker에는 application·database configuration 외에 `KAMIS_API_KEY`가 필요하다.
+web process에는 source credential을 주입하지 않는다. loader는 process environment를 먼저
+사용하므로 production에 `.env.local`을 복사하거나 mount하지 않는다.
+
+build와 운영 명령에서는 다음 non-secret placeholder를 사용할 수 있다.
+
+- `RELEASE_SHA`, `PREVIOUS_RELEASE_SHA`, `RELEASE_DIRECTORY`
+- `LOCAL_ASSURANCE_DJANGO_SECRET`, `PUBLIC_SERIES_ID`
+- `GUNICORN_BIND`, `GUNICORN_WORKERS`, `GUNICORN_THREADS`
+- `HEALTH_BASE_URL`, `SMOKE_SERIES_PATH`
+- `BACKUP_OUTPUT_DIRECTORY`, `BACKUP_DIRECTORY`, `RESTORE_DATABASE_NAME`
+- `PARSE_RUN_ID`, `REVIEW_DECISION_ID`, `REVIEW_EVIDENCE_SHA256`
+- `PUBLIC_COPY_REVISION`, `PUBLICATION_REVISION_ID`, `ROLLBACK_TARGET_REVISION_ID`
+- `PUBLICATION_OPERATION_ID`, `PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256`
+- `EXPECTED_PUBLICATION_VERSION`, `EXPECTED_CURRENT_REVISION_ID`
+
+### 최소권한 분리
+
+platform 선택 시 다음 권한을 서로 분리한다.
+
+- web: 승인 publication과 필요한 Django metadata를 읽는 권한
+- ingestion: source configuration, attempt, artifact hash, parse와 typed candidate 생성 권한
+- reviewer: 검수 evidence에 기반한 decision 생성 권한
+- publisher: sealed revision의 atomic pointer 전환 권한
+- migration: schema DDL과 trigger 설치 권한
+- backup/restore: backup 읽기와 새 복구 database 생성 권한
+
+repository의 production command boundary는 actor와 Django permission을 다시 확인하지만
+production DB grant, 외부 identity-aware MFA와 역할별 credential을 대신하지 않는다. 동일
+credential 하나에 모든 권한을 합치거나 enable flag를 인증으로 취급하는 것은 production
+해법이 아니다.
+
+## Release preflight
+
+### 1. 사람 확인
+
+아래 항목이 하나라도 없으면 배포를 시작하지 않는다.
+
+- 승인된 `RELEASE_SHA`와 이전에 검증된 `PREVIOUS_RELEASE_SHA`
+- Python·Django·PostgreSQL·uv 고정 버전을 지원하는 platform
+- 관리형 PostgreSQL, private network와 분리된 application/migration/backup 역할
+- HTTPS endpoint, 승인된 domain·DNS 변경 계획과 인증서
+- managed secret injection과 회전·폐기 담당자
+- query string, IP, User-Agent와 search term을 제거한 log pipeline
+- liveness/readiness/freshness probe와 on-call alert route
+- 암호화 backup, PITR, retention과 restore rehearsal 계획
+- production review·publication control plane과 MFA
+
+### 2. local release gate
+
+아래 명령은 production host가 아니라 owner-only `.env.local`이 있는 통제된 local release
+checkout에서 실행한다. `make production-check`의 `secret-check`는 그 local 파일이 Git에서
+제외되고 credential bytes가 현재·과거 Git blob에 없는지 검사한다. key, 길이 또는 일부를
+출력하지 않는다.
+
+```sh
+set -eu
+make sync
+test "$(git rev-parse HEAD)" = "$RELEASE_SHA"
+test -z "$(git status --porcelain)"
+git fsck --full
+
+DJANGO_DEBUG=0 \
+ADMIN_ENABLED=0 \
+DJANGO_SECRET_KEY="$LOCAL_ASSURANCE_DJANGO_SECRET" \
+DJANGO_ALLOWED_HOSTS=candidate.invalid \
+DJANGO_CSRF_TRUSTED_ORIGINS=https://candidate.invalid \
+DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+DEPLOY_VERSION="$RELEASE_SHA" \
+DJANGO_SECURE_SSL_REDIRECT=1 \
+DJANGO_SECURE_HSTS_SECONDS=31536000 \
+DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=1 \
+DJANGO_SECURE_HSTS_PRELOAD=1 \
+make production-check
+```
+
+`LOCAL_ASSURANCE_DJANGO_SECRET`은 실제 credential이 아닌 50자 이상의 local check 전용 synthetic
+값이다. 위 HSTS include-subdomains·preload 값은 Django warning gate를 통과시키는 synthetic
+configuration일 뿐, 실제 domain의 모든 subdomain 보호나 browser preload 제출 승인이 아니다.
+검사 뒤 `git status --porcelain`이 계속 비어 있는지도 다시 확인한다.
+
+이 local gate와 immutable build는 vendor와 무관한 재현 계약이다. 같은 clean `RELEASE_SHA`에서
+잠금 설치, forward migration plan, collectstatic, production-like process, health·catalog/detail
+smoke와 이전 SHA application rollback 순서를 다시 수행할 수 있어야 한다. 실제 artifact 포맷,
+upload·release·traffic switch CLI와 bundled license notice 검사는 platform을 선택한 뒤 별도 사람
+checkpoint에서 고정한다.
+
+`make production-check`는 ambient `KAMIS_API_KEY`를 모든 recipe child에서 unexport하고 그 부재를
+값 없이 확인한 뒤, `DATABASE_URL`이 repository의 고정 loopback Compose database와 정확히
+일치하는지 검사한다. 따라서 source credential이 assurance tool로 전파되지 않고 production 또는
+다른 local database에 release test를 실행하지 않는다. 이어 format, lint, type, migration drift,
+전체 test, Django system check, `check --deploy --fail-level WARNING`, local secret scan, dependency
+audit와 license inventory를 실행한다. 그래도 운영자는 key를 shell에 export하지 않는다.
+license inventory의 성공 exit만으로 라이선스 정책 승인을 대신하지 않으며
+`THIRD_PARTY_NOTICES.md`와 결과를 사람이 대조한다.
+
+production host나 CI에 `.env.local`을 만들기 위해 `secret-check`를 실행하지 않는다. managed
+production secret은 이 local Git-history 검사의 대상이 아니므로 production secret injection
+검증으로 과장하지 않는다.
+
+### 3. immutable build
+
+release마다 별도 `RELEASE_DIRECTORY`에서 잠금 파일 그대로 runtime dependency와 static
+artifact를 만든다.
+
+```sh
+make runtime-sync
+
+env \
+  DJANGO_DEBUG=0 \
+  ADMIN_ENABLED=0 \
+  QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DJANGO_SECRET_KEY="$LOCAL_ASSURANCE_DJANGO_SECRET" \
+  DJANGO_ALLOWED_HOSTS=candidate.invalid \
+  DJANGO_CSRF_TRUSTED_ORIGINS=https://candidate.invalid \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  DEPLOY_VERSION="$RELEASE_SHA" \
+  DJANGO_SECURE_SSL_REDIRECT=1 \
+  DJANGO_SECURE_HSTS_SECONDS=31536000 \
+  DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=1 \
+  DJANGO_SECURE_HSTS_PRELOAD=1 \
+  .venv/bin/python manage.py collectstatic --noinput
+```
+
+production storage는 WhiteNoise compressed manifest backend로 content-hashed filename을 만든다.
+application과 exact release의 `staticfiles`를 한 immutable 단위로 보존하고, 대표 CSS가 hashed
+URL에서 `200`, 올바른 content type과 immutable cache header로 응답하는지 traffic 전 확인한다.
+`collectstatic --clear`는 공유 자산 삭제 위험 때문에 사용하지 않는다.
+
+build artifact에는 application source·template, migration, lockfile, 생성된 static files, exact
+`RELEASE_SHA`, `THIRD_PARTY_NOTICES.md`와 platform packaging 방식이 요구하는 runtime dependency
+license·notice bundle만 명시적 allowlist로 넣는다. artifact 내부 notice와 실제 locked runtime을
+대사한다. browser evidence는 Git에 추적되지만 deployment artifact allowlist에서는 제외한다.
+`.env.local`, backup, test database와 cache directory도 포함하지 않는다. platform packaging이 이
+allowlist와 bundled notice 검사를 구현·검증하기 전에는 traffic을 열지 않는다.
+
+## Database preflight와 migration
+
+production database 생성·credential 주입과 아래 schema 변경 실행은 사람 checkpoint다.
+배포 직전 managed backup/PITR checkpoint가 성공했고 복구 가능한지 먼저 확인한다. local dump를
+production backup의 대체물로 사용하지 않는다.
+
+production `DATABASE_URL`은 TLS certificate와 hostname을 검증하는 `verify-full` 동등 설정 및
+승인된 CA 경로를 사용해야 한다. 현재 repository는 특정 managed PostgreSQL CA를 제공하거나 이
+platform 계약을 자동 검증하지 않으므로, 실제 connection과 server identity evidence가 없으면
+migration과 traffic 공개를 중단한다.
+
+새 release 환경에서 application 전환 전에 다음 순서로 확인한다.
+
+```sh
+.venv/bin/python manage.py makemigrations --check --dry-run
+.venv/bin/python manage.py showmigrations --plan
+.venv/bin/python manage.py migrate --plan
+.venv/bin/python manage.py check
+```
+
+plan을 exact release의 migration 파일과 대조한다. 현재 `0001`~`0010`은 새 model, constraint와
+trigger를 만드는 방향이다. `0010`은 고정 legacy source row의 date-precision
+`state_changed_at`·`rights_confirmed_at` 원본을 수정·삭제하지 않고, 그 원본에만 적용되는
+append-only correction을 추가한다. 정확한 live 관측 시각은 보존되지 않았다. effective
+`2026-08-30T02:23:44Z`는 commit
+`d23e5707e1fc3bf6e032d459b149b946b0451e00`의 durable gate-decision recorded-at upper bound이지
+정확한 관측 시각이 아니다. correction
+`49143c27-d2dd-5fbd-b1dc-4aa3cc002fab`의 insert는 base·chronology trigger로 검증되고 update/delete는
+DB에서 거부된다. bootstrap·review·inspection은 effective helper를 쓰며 새 DB는 처음부터 exact
+effective 값으로 생성되어 correction row가 없다. 기존 canonical row를 삭제·변환하는 contract
+migration은 없다.
+그래도 production data 규모에서 DDL lock과 실행 시간을 복제 DB로 측정하고, 예상 밖 operation이
+보이면 중단한다.
+
+승인된 maintenance/change window에서만 forward migration을 실행한다.
+
+```sh
+.venv/bin/python manage.py migrate --noinput
+.venv/bin/python manage.py migrate --check
+.venv/bin/python manage.py check --deploy --fail-level WARNING
+```
+
+`migrate --noinput`은 현재 release에서 forward-only이지만 실제 schema를 변경한다. 실패하면
+traffic을 새 application으로 전환하지 않고 이전 application을 유지한다. 부분 적용 상태는
+`showmigrations --plan`과 database audit로 확인하며 임의 `--fake`나 reverse migration으로
+감추지 않는다.
+
+첫 빈 database는 아직 active publication이 없으므로 readiness가 실패하는 것이 정상이다.
+production reviewer/publisher 경계가 준비되지 않은 상태에서 readiness를 통과시키려고 local
+actor를 복사하거나 pointer를 직접 수정하지 않는다.
+
+## WSGI·static·traffic 전환
+
+`manage.py runserver`는 production에서 사용하지 않는다. `make serve`는 고정된 local bind와
+worker 설정을 가진 smoke 명령이므로 platform sizing을 대신하지 않는다. production process는
+승인된 platform 설정으로 Gunicorn을 실행한다.
+
+```sh
+exec .venv/bin/gunicorn config.wsgi:application \
+  --bind "$GUNICORN_BIND" \
+  --workers "$GUNICORN_WORKERS" \
+  --threads "$GUNICORN_THREADS"
+```
+
+WhiteNoise middleware는 Django WSGI process 안에서 manifest가 가리키는 hashed static을 제공한다.
+대표 HTML이 hashed CSS URL을 참조하고 해당 asset이 올바른 media type과 immutable cache header로
+응답하는지 smoke한다. HTML·health의 `Cache-Control: no-store`와 static의 immutable cache는
+서로 다른 의도적 계약이다. CDN을 추가한다면 WhiteNoise 결과 앞의 선택적 delivery layer로만
+두고 새 cache purge·rollback 검증을 거친다.
+
+platform proxy가 HTTPS를 종료할 때 forwarding header 신뢰 범위와 Django HTTPS 인식을 별도
+검증한다. `DJANGO_TRUST_X_FORWARDED_PROTO=1`은 platform이 외부 client가 해당 header를 주입하지
+못하게 제거·재작성하고 단일 trusted proxy contract를 보장할 때만 켠다. 기본값은 꺼짐이며,
+platform 선택 뒤 검증 없이 proxy 구성을 추정하지 않는다.
+
+traffic 전환 순서는 다음과 같다.
+
+1. 새 release를 traffic 없이 시작한다.
+2. 새 release의 liveness를 확인한다.
+3. database migration과 active publication을 포함한 readiness를 확인한다.
+4. freshness를 별도 확인한다. stale은 last-known-good가 제공됨을 뜻하며 readiness와 다르다.
+5. catalog와 승인된 `SMOKE_SERIES_PATH`를 query string 없이 읽어 source date, unit, coverage와
+   publication fact-set header가 한 revision인지 확인한다.
+6. platform의 atomic traffic switch를 사람이 승인·실행한다.
+7. 전환 뒤 같은 검사를 반복하고 이전 release를 즉시 rollback 가능한 상태로 유지한다.
+
+platform이 아직 선택되지 않았으므로 실제 traffic switch와 release rollback CLI는 이 문서에
+꾸며내지 않는다. 선택 후 vendor의 exact command, account, application ID, timeout과 rollback
+동작을 별도 승인된 보충 절차로 기록해야 한다.
+
+### local 고정 부하 profile
+
+candidate의 public read 성능은 production-like `DEBUG=False` Gunicorn을 고정 loopback Compose
+DB에 연결해 측정한다. 이 검사는 신뢰된 local peer만 대상으로 하며 production capacity,
+network TLS·proxy latency 또는 managed PostgreSQL sizing을 대신하지 않는다. local HTTP 측정을
+위해 이 process에서만 SSL redirect를 끄고, 별도 `check --deploy --fail-level WARNING`에서는
+production transport contract를 검증한다.
+
+첫 terminal에서 다음 exact candidate process를 시작한다.
+
+```sh
+env \
+  DJANGO_DEBUG=0 \
+  ADMIN_ENABLED=0 \
+  QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DJANGO_SECRET_KEY="$LOCAL_ASSURANCE_DJANGO_SECRET" \
+  DJANGO_ALLOWED_HOSTS=127.0.0.1,localhost \
+  DJANGO_CSRF_TRUSTED_ORIGINS=https://candidate.invalid \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  DEPLOY_VERSION="$RELEASE_SHA" \
+  DJANGO_SECURE_SSL_REDIRECT=0 \
+  DATABASE_CONN_MAX_AGE=60 \
+  .venv/bin/gunicorn config.wsgi:application \
+    --bind 127.0.0.1:8000 \
+    --workers 2 \
+    --threads 4 \
+    --access-logfile /dev/null
+```
+
+두 번째 terminal에서 `inspect_recent_publication`으로 active hash·version을 고정한 뒤 실행한다.
+
+```sh
+.venv/bin/python scripts/http_load_profile.py \
+  --port 8000 \
+  --detail-id "$PUBLIC_SERIES_ID"
+```
+
+인수를 위한 명령에는 `--profile`이나 `--duration-seconds` override를 주지 않는다. 고정 계약은
+900초, 10 requests/s, 총 9,000개, catalog·list·search 6,300개와 detail 2,700개다. 20개의 고정
+logical virtual-user session이 모두 참여하고 request index를 round-robin으로 나눈다. 실제
+in-flight 요청 peak는 논리 사용자 수와 별도로 측정하며 20을 넘을 수 없다. queue를 포함한
+end-to-end p95가 500 ms 이하, 5xx가 0.5% 미만, 오류 0,
+publication fact-set header 단일값과 elapsed 900~903초를 모두 만족해야 통과한다. effective
+paced deadline 대비 schedule jitter p95가 100 ms 이하이고, nominal 제출 cadence는
+100 ms, bounded recovery 제출 간격 최솟값은 90 ms 이상이며 catch-up burst가 0이어야 한다.
+max jitter는 관측값으로
+receipt에 남기되 이후 요청에 반복 전가하지 않는다. profile 중 lifecycle·migration·backup처럼
+DB state를 바꾸는 작업을 병행하지 않는다.
+종료 직후 `inspect_recent_publication`을 다시 실행해 active revision·version·fact-set hash가 시작
+receipt와 같은지 대사한다.
+
+## Health와 freshness
+
+health URL에는 credential이나 query string을 붙이지 않는다. 아래 조회는 고정된 작은 JSON만
+반환하며 `Cache-Control: no-store`를 사용한다.
+
+```sh
+curl --proto '=https' --tlsv1.2 --fail --silent --show-error \
+  --connect-timeout 3 --max-time 10 "$HEALTH_BASE_URL/health/live"
+curl --proto '=https' --tlsv1.2 --fail --silent --show-error \
+  --connect-timeout 3 --max-time 10 "$HEALTH_BASE_URL/health/ready"
+curl --proto '=https' --tlsv1.2 --fail --silent --show-error \
+  --connect-timeout 3 --max-time 10 "$HEALTH_BASE_URL/health/freshness"
+```
+
+- `/health/live`: Django process가 응답하면 성공한다. DB와 publication을 검사하지 않는다.
+- `/health/ready`: DB 연결, migration currency와 sealed active publication을 검사한다. current와
+  stale publication 모두 last-known-good read가 가능하므로 성공할 수 있다.
+- `/health/freshness`: active publication이 current일 때만 성공한다. stale 또는 unavailable은
+  실패 상태이며 operator alert 대상이다.
+
+platform liveness는 process restart 판단에, readiness는 traffic 편입 판단에 사용한다.
+freshness 실패만으로 application을 재시작하거나 승인 pointer를 자동 변경하지 않는다.
+
+scheduler에서도 동일 freshness 경계를 확인할 수 있다.
+
+```sh
+.venv/bin/python manage.py check_recent_publication_freshness
+```
+
+성공은 fixed `CURRENT` receipt를 출력한다. stale, unavailable 또는 검사 실패는 각각 고정
+`RECENT_PUBLICATION_FRESHNESS_*` code와 non-zero exit를 반환한다. scheduler는 stdout의 raw
+재가공이나 exception text 대신 exit status와 fixed code만 alert에 연결한다.
+
+현재 local evidence에서 active artifact의 마지막 source 확인 시각은
+`2026-08-30T04:00:48.696744Z`다. 36시간 다음 확인 경계는
+`2026-08-31T16:00:48.696744Z`(`2026-09-01 01:00:48 KST`)이며, 이 시각은 배포 시점이나
+production scheduler 성공을 뜻하지 않는다. 경계 전에 새 실패 attempt가 있거나 경계를 넘으면
+freshness contract에 따라 다시 확인하고 alert를 판정한다.
+
+## 수집부터 공개까지
+
+### 역할 분리와 자동화 한계
+
+ingestion 성공은 review, seal 또는 activation 성공이 아니다. scheduler는 ingestion까지만
+실행하고 자동 approve·seal·activate를 연결하지 않는다. source 실패, parse 실패 또는 새
+candidate는 현재 pointer를 바꾸지 않는다.
+
+active A-path `SourceConfiguration`은 `schedule_execution_mode=PLATFORM_SINGLETON`,
+`schedule_interval_hours=24`를 기록한다. production platform scheduler는 성공 여부와 관계없이
+인접한 scheduled start 사이를 24시간보다 길게 두지 않고, 이전 실행이 남아 있으면 새 실행을
+겹치거나 catch-up burst로 만들지 않는다. bounded attempt가 끝난 뒤 fixed exit·audit를 alert에
+연결한다. 이 값은 자동 review·seal·activation 권한이 아니며 36시간 freshness 경계 전 확인과
+운영 대응 시간을 확보하기 위한 최대 cadence 계약이다.
+
+production scheduler는 중첩 실행을 막는 singleton job으로 다음 명령만 실행한다.
+
+```sh
+.venv/bin/python manage.py ingest_kamis_recent
+```
+
+`KAMIS_API_KEY`는 worker process 안에서만 managed secret으로 주입한다. command argument, URL,
+shell trace, log 또는 receipt로 전달하지 않는다. 실패 시 key를 확인하려고 `env`, `printenv`,
+`echo`, shell tracing 또는 exception traceback을 사용하지 않는다.
+
+성공 receipt의 `parse_run_id`, row count와 replay 상태는 민감하지 않은 audit locator다. reviewer는
+그 parse run의 typed identity, coverage, unit, missing reference, row counts, reconciliation hash와
+source rights 상태를 별도 승인된 private review surface에서 확인한다. 이 repository에는
+production MFA review surface가 아직 없다.
+
+### local Phase 0 lifecycle rehearsal
+
+아래 형태는 `DEBUG=True`, Admin·QA preview·production control plane 비활성 환경의 local
+rehearsal이다. 각 명령에 이 안전 전제를 명시하며, `DEBUG` 검사를 우회하지 말고
+disposable/local database에서만 lifecycle과 rollback을 재현한다.
+
+먼저 fixed non-login local actor를 준비한다.
+
+```sh
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py bootstrap_local_phase0_operator
+```
+
+검수 evidence 자체는 private 경계에 두고 그 canonical bytes의 SHA-256만 전달한다. UUID와 hash는
+새 logical action마다 외부의 안전한 operator tooling으로 생성하며 command substitution으로
+secret을 만들거나 출력하지 않는다.
+
+```sh
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py approve_recent_generation \
+  --parse-run-id "$PARSE_RUN_ID" \
+  --decision-id "$REVIEW_DECISION_ID" \
+  --acceptance-evidence-sha256 "$REVIEW_EVIDENCE_SHA256"
+
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py seal_recent_publication \
+  --decision-id "$REVIEW_DECISION_ID" \
+  --public-copy-revision "$PUBLIC_COPY_REVISION"
+
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py transition_recent_publication \
+  --operation ACTIVATE \
+  --operation-id "$PUBLICATION_OPERATION_ID" \
+  --acceptance-evidence-sha256 "$PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256" \
+  --expected-version "$EXPECTED_PUBLICATION_VERSION" \
+  --expected-current-revision "$EXPECTED_CURRENT_REVISION_ID" \
+  --target-revision "$PUBLICATION_REVISION_ID"
+```
+
+approve는 validated generation과 exact reconciliation을, seal은 latest approval과 immutable fact
+set을, activation은 sealed target과 optimistic current state를 다시 검증한다. 같은 logical
+operation의 uncertain retry에만 같은 operation UUID를 재사용한다. 다른 target·evidence·expected
+state로 UUID를 재사용하지 않는다.
+
+### production private operation job boundary
+
+repository는 HTTP Admin이나 로그인 화면을 production control plane으로 제공하지 않는다.
+사람이 승인한 platform의 외부 MFA·IAM private job만 다음 command를 실행할 수 있어야 하고,
+job별 role-specific database credential·grant와 immutable audit를 별도로 구성한다.
+`CONTROL_PLANE_OPERATIONS_ENABLED=1`은 accident-prevention flag일 뿐 인증이 아니며 web,
+ingestion과 scheduler process에서는 항상 `0`이다.
+
+actor provisioning credential은 승인된 change에서 한 번만 다음 명령을 실행한다. 두 actor는
+로그인할 수 없고 PII·staff·superuser·group이 없으며 reviewer와 publisher가 각각 정확히 하나의
+Django permission만 가진다. 기존 actor에 drift가 있으면 둘 중 어느 것도 부분 수정하지 않는다.
+
+```sh
+.venv/bin/python manage.py bootstrap_control_plane_actors \
+  --expected-release-sha "$RELEASE_SHA"
+```
+
+외부 MFA job은 실행 release와 같은 exact SHA를 모든 write command에 전달한다. reviewer job은
+approve만, publisher job은 seal과 transition만 실행한다. production reason code는 local
+rehearsal reason과 분리된다.
+
+```sh
+.venv/bin/python manage.py approve_recent_generation \
+  --parse-run-id "$PARSE_RUN_ID" \
+  --decision-id "$REVIEW_DECISION_ID" \
+  --acceptance-evidence-sha256 "$REVIEW_EVIDENCE_SHA256" \
+  --expected-release-sha "$RELEASE_SHA"
+
+.venv/bin/python manage.py seal_recent_publication \
+  --decision-id "$REVIEW_DECISION_ID" \
+  --public-copy-revision "$PUBLIC_COPY_REVISION" \
+  --expected-release-sha "$RELEASE_SHA"
+
+.venv/bin/python manage.py transition_recent_publication \
+  --operation ACTIVATE \
+  --operation-id "$PUBLICATION_OPERATION_ID" \
+  --acceptance-evidence-sha256 "$PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256" \
+  --expected-version "$EXPECTED_PUBLICATION_VERSION" \
+  --expected-current-revision "$EXPECTED_CURRENT_REVISION_ID" \
+  --target-revision "$PUBLICATION_REVISION_ID" \
+  --expected-release-sha "$RELEASE_SHA"
+```
+
+`PUBLIC_COPY_REVISION`은 현재 `ko-v1` 또는 `ko-v2`만 허용한다. 첫 activation은 authoritative
+inspection의 `version=0`, current literal `NONE`을 사용한다. production receipt는 actor,
+release SHA와 evidence hash를 출력하지 않는다. ReviewDecision·Activation은 actor를 DB audit에
+보존하지만 seal invoker는 revision row에 저장되지 않으므로 외부 MFA job audit와 change record가
+필수다. actor bootstrap, IAM, grant와 첫 production publication은 사람 checkpoint다.
+
+## Publication rollback과 withdraw
+
+publication rollback은 application rollback과 별개다. revision row를 수정하거나 삭제하지
+않고 이전에 current였던 sealed revision을 대상으로 새 append-only `ROLLBACK` activation을
+추가한다.
+withdraw는 권리·identity·공개 안전성이 더 이상 보장되지 않을 때 current pointer를 비우는 새
+activation이다.
+
+실행 직전에 승인된 read-only 운영 화면에서 channel의 exact current revision과 version을 다시
+읽는다. 과거 receipt나 기억한 값을 사용하지 않는다. 두 값이 예상과 다르면 concurrent change로
+간주하고 실패 폐쇄한다.
+
+현재 repository의 authoritative read-only 명령은 다음과 같다.
+
+```sh
+.venv/bin/python manage.py inspect_recent_publication
+```
+
+이 명령은 PostgreSQL `REPEATABLE READ, READ ONLY` snapshot에서 activation history, sealed revision,
+canonical entry 집합과 fact-set hash를 다시 계산한다. `AVAILABLE` receipt의 `version`,
+`current_revision_id`, 마지막 activation을 바로 다음 transition의 expected state로 사용한다.
+첫 빈 channel 또는 withdraw 상태는 current revision을 literal `NONE`으로 출력한다. `ERROR`나
+non-zero exit이면 transition하지 않는다.
+
+local rehearsal rollback 명령은 다음과 같다.
+
+```sh
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py transition_recent_publication \
+  --operation ROLLBACK \
+  --operation-id "$PUBLICATION_OPERATION_ID" \
+  --acceptance-evidence-sha256 "$PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256" \
+  --expected-version "$EXPECTED_PUBLICATION_VERSION" \
+  --expected-current-revision "$EXPECTED_CURRENT_REVISION_ID" \
+  --target-revision "$ROLLBACK_TARGET_REVISION_ID"
+```
+
+local rehearsal withdraw 명령은 target revision을 전달하지 않는다.
+
+```sh
+env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
+  CONTROL_PLANE_OPERATIONS_ENABLED=0 \
+  DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery \
+  .venv/bin/python manage.py transition_recent_publication \
+  --operation WITHDRAW \
+  --operation-id "$PUBLICATION_OPERATION_ID" \
+  --acceptance-evidence-sha256 "$PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256" \
+  --expected-version "$EXPECTED_PUBLICATION_VERSION" \
+  --expected-current-revision "$EXPECTED_CURRENT_REVISION_ID"
+```
+
+withdraw 후 readiness와 freshness가 unavailable이 되고 public catalog에 공개 자료가 없으며
+detail이 숨겨지는지 확인한다. 이는 장애가 아니라 안전한 철회 상태일 수 있으므로 운영 change
+record와 alert suppression window를 함께 남긴다.
+
+production rollback·withdraw는 위 publisher private job에서 같은 명령에
+`--expected-release-sha "$RELEASE_SHA"`를 추가해 실행한다. external MFA·IAM, 역할별 credential,
+default-off enable flag, exact release lock과 authoritative optimistic state 중 하나라도 없으면
+실패해야 한다. `DEBUG`를 켜거나 DB pointer를 직접 update하지 않는다.
+
+## Application rollback
+
+application rollback은 database와 publication을 독립적으로 유지한 채 이전 immutable release와
+그 static directory로 traffic을 되돌리는 작업이다.
+
+1. 새 traffic 편입을 중단하고 `RELEASE_SHA`, request ID, health 상태와 최초 이상 시각만
+   기록한다. query, user input 또는 secret은 기록하지 않는다.
+2. migration이 이미 성공했다면 schema를 그대로 둔다. 현재 migration은 additive 방향이지만,
+   `PREVIOUS_RELEASE_SHA`가 최신 schema를 읽을 수 있다고 release별로 실제 검증한 경우에만
+   rollback한다.
+3. platform의 승인된 atomic release switch로 application과 static을 함께
+   `PREVIOUS_RELEASE_SHA`에 되돌린다.
+4. liveness, readiness, freshness와 catalog/detail smoke를 다시 수행한다.
+5. publication 내용 자체가 문제면 application rollback과 별도로 rollback 또는 withdraw한다.
+
+production에서 migration을 역방향 실행하여 code rollback을 맞추지 않는다. 이전 code가 최신
+schema와 호환되지 않으면 traffic을 전환하지 말고 중단한다. database 복구가 필요하면 기존
+database를 덮어쓰지 않고 managed PITR로 새 instance를 만든 뒤 검증·승인된 connection switch를
+수행한다.
+
+local application rollback rehearsal에서 실제로 검증한 `PREVIOUS_RELEASE_SHA`는
+`d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`다. 이는 local 호환성 evidence의 고정 target이지,
+향후 production vendor rollback 명령이나 실제 traffic 전환을 승인한 값이 아니다.
+
+## Local backup·restore rehearsal
+
+`scripts.postgres_backup_restore`는 `docker --host unix:///var/run/docker.sock`의 exact local
+socket만 사용한다. ambient Docker context·host와 `DATABASE_URL`을 읽거나 전달하지 않는다.
+고정 Compose project·`db` service label로 실행 중인 container를 하나만 발견하고, immutable
+container ID·image·project/service identity를 한 invocation 동안 pin한 뒤 모든 `docker exec`에
+그 ID를 직접 사용한다. container가 교체·중지되거나 identity가 달라지면 실패 폐쇄하므로
+production database나 다른 Compose service를 대상으로 사용할 수 없다. PostgreSQL custom dump,
+manifest·dump hash, migration inventory, 모든 public table row count, active revision, canonical
+publication hash와 activation chain을 대사한다.
+이 도구는 sealed active publication이 있는 candidate DB만 허용하며 빈 DB나 withdraw 상태의
+범용 backup 도구가 아니다.
+
+`BACKUP_OUTPUT_DIRECTORY`는 repository 밖의 기존 absolute owner-controlled directory여야 하고
+symlink가 아니어야 한다. 도구는 directory와 parent의 owner·write 경계를 검사하고 열린 directory
+FD에 identity를 고정한다. 새 generated backup directory는 `0700`, 파일은 `0600`이며 dump는 같은
+열린 FD를 checksum·검사·restore까지 사용한다. backup은 source DB를 변경하지 않지만 새 private
+directory와 파일을 만든다.
+
+```sh
+.venv/bin/python -m scripts.postgres_backup_restore backup \
+  --output-dir "$BACKUP_OUTPUT_DIRECTORY"
+```
+
+성공 receipt가 가리키는 exact generated directory를 `BACKUP_DIRECTORY`로 선택한다. 실제 이름은
+`BACKUP_OUTPUT_DIRECTORY/postgres-backup-<backup UUID>`이며 output root 자체를 restore 입력으로
+사용하지 않는다. 같은 receipt의 `manifest_sha256`은 secret이 아닌 out-of-band integrity 값이다.
+그 값을 변경 없이 `BACKUP_MANIFEST_SHA256`에 전달해야 하며, manifest 파일에서 다시 계산한 값을
+expected 값으로 쓰면 독립 대사가 되지 않는다. 값이 누락되거나 형식·내용이 다르면 target 생성
+전에 실패한다. `RESTORE_DATABASE_NAME`은 source와 다른, 존재하지 않는 disposable name이어야
+한다. `grocery_restore_`로 시작하는 소문자·숫자·underscore의 bounded 식별자만 허용하며,
+restore는 새 target database를 만들고 source database를 변경하지 않는다.
+
+```sh
+.venv/bin/python -m scripts.postgres_backup_restore restore \
+  --backup-dir "$BACKUP_DIRECTORY" \
+  --expected-manifest-sha256 "$BACKUP_MANIFEST_SHA256" \
+  --target-database "$RESTORE_DATABASE_NAME"
+```
+
+성공 receipt에서 row counts, migration inventory와 publication contract가 모두 일치하는지
+확인한다. 그 뒤 별도 process의 `DATABASE_URL`을 격리 target에 managed 방식으로 연결하여 다음을
+실행한다.
+
+```sh
+env DATABASE_URL="postgresql://grocery:local-grocery-only@127.0.0.1:55434/$RESTORE_DATABASE_NAME" \
+  .venv/bin/python manage.py migrate --check
+env DATABASE_URL="postgresql://grocery:local-grocery-only@127.0.0.1:55434/$RESTORE_DATABASE_NAME" \
+  .venv/bin/python manage.py check
+```
+
+복원 target으로 시작한 local application에서 readiness와 대표 catalog/detail도 확인한다.
+원본 Compose DB를 가리키고 있지 않은지 database name을 먼저 read-only로 확인한다.
+
+target 생성 뒤 restore·inventory·canonical publication 검사 중 어느 단계가 실패해도 도구는 같은
+invocation이 만든 exact target만 identity-pin한 container에서 자동 삭제하고 실제 부재를 다시
+확인한다. pre-existing target, source `grocery`, 다른 이름이나 다른 container는 삭제하지 않는다.
+자동 cleanup 자체가 실패하거나 target 부재를 증명하지 못하면 성공 receipt를 내지 않고 별도 fixed
+cleanup failure로 중단해 사람이 조사한다. 성공한 restore target과 실패한 backup directory는
+evidence 검토 전 자동 삭제하지 않는다. `docker compose down -v`는 source volume까지 삭제하므로
+cleanup으로 사용하지 않는다.
+
+local dump는 application-level 암호화를 제공하지 않는다. storage volume의 암호화가 별도로
+증명되지 않았다면 민감 backup으로 취급하고 owner-only 경계와 명시적 retention을 적용한다.
+
+## Production backup, PITR와 복구 gap
+
+production 공개 전 managed PostgreSQL에서 다음을 실제로 증명해야 한다.
+
+- 암호화된 자동 backup과 WAL 기반 point-in-time recovery
+- application, migration, backup·restore 역할의 분리와 감사
+- backup retention, region·account 격리와 실패 alert
+- 기존 database를 덮어쓰지 않는 새 instance restore
+- migration inventory, row counts, audit chain, active revision, fact-set hash와 public read 대사
+- 목표 `RPO 24시간`과 `RTO 4시간` 안의 timed rehearsal
+- 분기별 restore rehearsal과 evidence retention
+
+현재 candidate에는 production platform, encrypted scheduled backup, PITR, production restore,
+backup failure structured event와 실제 RPO/RTO 측정이 없다. local Compose rehearsal을 이 항목의
+통과로 기록하지 않는다. backup/PITR가 구성·복원 검증되지 않으면 production migration과
+traffic 공개를 중단한다.
+
+PITR는 항상 새 database instance로 복원하고 검증 후 connection을 전환한다. 기존 instance
+삭제, in-place overwrite, retention 단축과 old backup 폐기는 파괴적 사람 checkpoint다.
+
+## Structured log와 alert
+
+application의 `grocery.audit` logger는 allowlist된 single-line JSON만 stdout에 보낸다. 허용
+field는 timestamp, severity, message code, request ID, deploy version, command run ID와 lifecycle
+ID·status·event뿐이다. arbitrary message, exception, query, response body, key와 user identity를
+넣지 않는다. Django request/server logger는 application에서 버려지므로 platform proxy의 access
+log도 query string, IP, User-Agent와 search term을 별도로 제거해야 한다.
+
+다음 structured `message_code`를 즉시 또는 짧은 반복 window의 alert에 연결한다.
+
+| code | 의미 | 초기 대응 |
+|---|---|---|
+| `health.readiness.unavailable` | DB, migration 또는 active publication read 불가 | traffic 편입 중단; DB·schema·pointer 분리 확인 |
+| `health.freshness.unavailable` | active publication 또는 freshness 판단 불가 | 자동 publication 금지; 권리·pointer·attempt 확인 |
+| `health.freshness.stale` | last-known-good는 있으나 새 확인 필요 | 공개본 유지; ingestion·review backlog 조사 |
+| `public.catalog.unavailable` | catalog read 실패 | request ID로 DB/read 경계 확인; 반복 시 application rollback 검토 |
+| `public.detail.unavailable` | detail read 실패 | request ID로 active revision membership/read 확인 |
+| `ingest.source.start_failed` | source/audit 시작 실패 | DB와 source configuration 확인; pointer 유지 |
+| `ingest.fetch.failed` | bounded fetch 실패 | HTTP class·quota·credential 상태를 redacted receipt로 확인 |
+| `ingest.fetch.finalization_failed` | 실패 attempt 종료 기록 실패 | audit 불완전 incident로 즉시 escalation |
+| `ingest.parse.start_failed` | parse audit 시작 실패 | artifact·DB 상태 확인; candidate 공개 금지 |
+| `ingest.parse.failed` | schema·identity·unit·reconciliation 실패 또는 quarantine | 자동 retry·승인 금지; reviewer 조사 |
+| `ingest.parse.finalization_failed` | parse 실패 상태 기록도 실패 | audit 불완전 incident로 즉시 escalation |
+
+`ingest.fetch.started`, `ingest.fetch.succeeded`, `ingest.parse.started`,
+`ingest.parse.resumed`, `ingest.parse.replay_started`, `ingest.parse.validated`와
+`ingest.command.succeeded`는 정상 lifecycle evidence이며 단독 alert 대상이 아니다. 성공 없이
+started만 남는 run, 연속 fetch 실패, quarantined lifecycle status 증가와 새 validation 부재를
+platform 집계 규칙으로 경보한다.
+
+freshness command의 non-zero `RECENT_PUBLICATION_FRESHNESS_*` code와 ingest command의 fixed
+`INGEST_*` failure code도 scheduler alert에 연결한다. review·seal·transition은 현재 structured
+log를 내지 않고 DB audit와 fixed CLI receipt만 남긴다. private job은
+`CONTROL_PLANE_*`, `RECENT_PUBLICATION_INSPECTION_FAILED`와 non-zero exit를 인자나 원문 오류 없이
+별도 platform audit·alert로 연결한다. local backup script도 structured logger에 연결되지 않으므로
+production control plane과 backup platform은 non-zero job, DB audit gap과 backup failure alert를
+추가해야 한다. actor/bootstrap·seal job audit 누락이나 ReviewDecision/Activation actor chain
+불일치도 production traffic 차단 신호다.
+
+유효한 `DEPLOY_VERSION`은 application JSON event에 자동 포함된다. production settings는 exact
+40자 lowercase Git SHA가 없으면 시작을 거부한다. platform도 immutable release metadata로 같은
+SHA를 보존해 application event와 ingress·runtime signal을 대조할 수 있어야 한다.
+
+## Secret rotation
+
+공통 금지 사항은 다음과 같다.
+
+- secret을 command argument, URL, Git, fixture, receipt, log, screenshot 또는 ticket에 넣지 않음
+- `env`, `printenv`, `echo`, shell tracing, exception repr 또는 길이·fragment로 값을 확인하지 않음
+- 같은 secret을 web, ingestion, migration, database와 backup 역할 사이에 재사용하지 않음
+- 새 secret 검증 전에 old secret을 폐기하지 않음
+
+### KAMIS key
+
+1. provider login·key 발급·폐기는 사람이 수행한다.
+2. 새 값을 managed secret store의 새 version으로 넣고 ingestion worker에만 연결한다.
+3. 새 worker process를 시작하여 값 자체를 출력하지 않고 한 번의 bounded ingestion을 실행한다.
+4. fixed success receipt, attempt audit와 parse result를 확인한다. 자동 activate하지 않는다.
+5. 성공 후 scheduler를 새 secret version으로 전환하고 old process를 종료한다.
+6. propagation이 확인된 뒤 사람이 old provider credential을 폐기한다.
+
+인증 실패나 provider propagation 지연은 source viability 실패나 fallback 전환으로 자동 판정하지
+않는다. old key가 여전히 유효하면 last-known-good와 old worker를 유지하고 원인을 확인한다.
+
+### Django secret
+
+`DJANGO_SECRET_KEY`를 managed store에서 새 version으로 교체하고 rolling restart한다. 현재
+settings는 이전 key fallback을 구성하지 않으므로 rotation은 기존 signed session·token을
+무효화할 수 있다. Admin은 production에서 비활성 상태이며, 향후 MFA Admin을 열기 전에는 session
+영향과 fallback 제거 시점을 별도 검토한다. 새 release health가 통과하기 전에 old version을
+폐기하지 않는다.
+
+### Database credential
+
+role별 새 database credential을 만들고 managed `DATABASE_URL` reference를 새 version으로
+전환한 뒤 process를 rolling restart한다. readiness, migration check, ingestion audit와 backup
+job을 역할별로 검증하고 old connection이 drain된 뒤 old credential을 폐기한다. database URL
+전체 또는 password를 어느 command에도 출력하지 않는다.
+
+`make secret-check`는 local ignored `.env.local`의 Git 누출 검사이지 managed production secret
+rotation 도구가 아니다.
+
+## Incident별 안전한 대응
+
+| 신호 | 유지할 것 | 금지할 자동 대응 |
+|---|---|---|
+| liveness 실패 | DB·publication을 그대로 두고 process/release 조사 | migration reverse, publication 전환 |
+| readiness 실패 | 이전 release traffic과 last-known-good 유지 | pointer 직접 update, 빈 DB를 ready로 가장 |
+| freshness stale | last-known-good 공개와 stale 표시 유지 | 미검수 candidate activate, 비공식 source 보충 |
+| freshness unavailable | 권리·active pointer·DB를 분리 조사 | 오래된 publication이 안전하다고 추정 |
+| fetch/parse 실패 | 현재 publication 유지, 새 attempt audit 보존 | partial generation 혼합, 무한 retry |
+| public read 반복 실패 | request ID와 release를 대조하고 application rollback 검토 | 오류 원문·query logging |
+| publication 오류 | optimistic state를 다시 읽고 별도 rollback/withdraw 승인 | revision 수정·삭제, 같은 UUID의 다른 action 재사용 |
+| backup 실패 | migration·deploy 중단, 기존 verified backup 보존 | 실패 backup으로 restore, old backup 조기 삭제 |
+| secret 노출 의심 | incident 보존, 새 version 검증 후 revoke | ticket/log에 노출값 복사, 무검증 즉시 폐기 |
+
+source 권리 철회, identity·unit·coverage 불신 또는 잘못된 공개 사실은 단순 freshness 장애가
+아니다. 사람이 public safety를 판단하고 필요하면 production publisher를 통해 `WITHDRAW`해야
+한다.
+
+## 배포 종료와 rollback 준비 확인
+
+traffic 공개 전 change record에는 값 자체가 아니라 다음 evidence locator만 남긴다.
+
+- exact `RELEASE_SHA`, clean pre-build Git 상태와 `git fsck`
+- locked dependency build, `production-check`, migration plan과 forward migration 결과
+- collectstatic artifact와 Gunicorn process revision
+- liveness, readiness, freshness와 catalog/detail smoke 결과
+- 현재 publication revision, pointer version과 마지막 activation ID
+- backup/PITR checkpoint, restore rehearsal ID와 RPO/RTO 측정
+- alert route와 담당자, known gap와 다음 검토 시각
+- `PREVIOUS_RELEASE_SHA`, 이전 static artifact와 platform-specific rollback command locator
+
+다음 항목이 남아 있으면 상태는 계속 **Phase 0 배포 직전**이다.
+
+- production platform·PostgreSQL·secret store·domain·DNS가 미승인
+- production MFA reviewer/publisher control이 없음
+- production backup/PITR·restore와 RPO/RTO가 미검증
+- platform-specific deploy·traffic switch·application rollback command가 미기록
+- production alert delivery와 ingress log privacy가 미검증
+
+실제 deploy, production publication, domain 전환과 rollback rehearsal은 각각 사람 승인 뒤에
+수행하고, 그때 별도의 production 완료 evidence를 남긴다.


