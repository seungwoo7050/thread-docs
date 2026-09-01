## `docs: align vnext operational boundaries`

diff --git a/.env.example b/.env.example
index f947ed8..a569613 100644
--- a/.env.example
+++ b/.env.example
@@ -13,6 +13,8 @@ DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
 DATABASE_CONN_MAX_AGE=60
 KAMIS_API_KEY=
 KAMIS_CONFIRMATION_MAX_AGE_HOURS=36
+KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS=192
+KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS=36
 QA_STATE_PREVIEWS_ENABLED=0
 CONTROL_PLANE_OPERATIONS_ENABLED=0
 DEPLOY_VERSION=0000000
diff --git a/README.md b/README.md
index a2ebf87..2f97301 100644
--- a/README.md
+++ b/README.md
@@ -1,8 +1,9 @@
 # Audience Foundry Grocery Seasonality
 
-한국 소비자가 한국농수산식품유통공사(KAMIS)의 채소·과일 소매 조사 평균을
-동일 품목·품종·등급·판매 단위 안에서 source row가 제공한 조사일 값과 1주·1개월·1년
-제공값을 중립적으로 비교하는 Django server-rendered responsive web 서비스입니다.
+한국 소비자가 한국농수산식품유통공사(KAMIS)의 채소·과일 소매 조사값을
+동일 품목·품종·등급·판매 단위 안에서 살피는 Django server-rendered responsive web
+서비스입니다. 최근 조사 평균과 1주·1개월·1년 제공값뿐 아니라, 별도로 승인된 historical
+publication이 있을 때 선택 지역의 월별 기록, 지역별 조사 범위와 시장별 관측을 제공합니다.
 
 `seasonality`는 저장소 코드명입니다. 첫 MVP는 한 해 전 값 하나를 계절성·평년·제철의
 증거로 바꾸지 않습니다. 공개 화면은 공식 조사값과 결정적인 차이만 표시하며
@@ -14,12 +15,18 @@
 - 저장소: `audience-foundry-grocery-seasonality`
 - 기본 브랜치: `main`
 - 정책 기준선: `0cc95e7`
-- 원격 저장소: 없음
-- 공개 상태: 로컬·미공개
-- 구현 상태: **Phase 0 배포 직전 완료**; production 미배포
-- 활성 source path: **A — 최근 비교 MVP**
+- 원격 저장소: `origin`
+  (`https://github.com/seungwoo7050/audience-foundry-grocery-seasonality.git`)
+- 공개 상태: GitHub 공개 저장소·production service 미배포
+- 구현 상태: **vNext local implementation candidate**; production 미배포·미활성
+- 시작 기준선: `bb0b28038243c539db2eafcfebc05144d9d59d66`
+- source path: 최근 비교 `15156063`; historical `15156060`, `15156062`, `15156065`
 
-## 고정된 첫 범위
+vNext 변경은 시작 기준선 이후 local에서만 만들었고 이 문서 갱신 시점에는 `origin`으로
+push하거나 production에 배포하지 않았습니다. exact 최종 local SHA와 검증 상태는 완료
+보고서에서 고정합니다.
+
+## 역사적 첫 범위
 
 - 첫 source는 공공데이터포털의 KAMIS
   [최근일자 도·소매가격정보 API `15156063`](https://www.data.go.kr/data/15156063/openapi.do)입니다.
@@ -31,6 +38,20 @@
 - 계정, 검색 이력, 장바구니, 위치, GPS와 개인화는 만들지 않습니다.
 - 공개 request는 외부 source를 호출하지 않고 승인된 PostgreSQL publication만 읽습니다.
 
+## 승인된 vNext 소비자 확장
+
+- 월별 화면은 선택한 지역의 최근 36개월 KAMIS 제공 평균·최저·최고와 결측 구간을 그대로
+  표시합니다. 12개월과 60개월 선택지는 completeness 계약을 통과한 경우에만 노출합니다.
+- 지역별 화면은 동일 series·조사일의 공식 지역명과 제공 평균·최저·최고를 표시합니다.
+- 시장별 화면은 선택 지역·조사일의 공식 시장명과 KAMIS 제공값을 가격순이나 시장유형 추정
+  없이 표시합니다.
+- 선택 목록은 계정·cookie·session 없이 URL에 최대 5개 exact series를 담아 각 품목 자체의
+  최근 변화만 나란히 봅니다. 합계·절약액·서로 다른 단위의 우열은 만들지 않습니다.
+- `RECENT_RETAIL`과 `HISTORICAL_RETAIL`은 독립적으로 검수·봉인·활성화·rollback하며 freshness와
+  fact-set hash를 합성하지 않습니다.
+- historical publication이 없거나 exact mapping이 없으면 최근 상세는 계속 작동하고 확장
+  링크만 숨깁니다. 공개 request는 어느 화면에서도 source API를 호출하지 않습니다.
+
 ## 완료된 source gate
 
 공식 HTTPS API의 실제 요청으로 인증, JSON/XML, 452행 ordered pagination, provider error,
@@ -48,6 +69,13 @@ current/week/month/year 계약을 검증했습니다. 정확한 reference date
 월별 탐색기로 축소할 수 있습니다. KAMIS HTML scraping, 비공식 미러와 quota 우회는
 허용하지 않습니다.
 
+vNext 개발계정의 최소 live gate는 공식 API `15156060`, `15156062`, `15156065`의 HTTPS 접근,
+non-empty 소매 응답과 exact wrapper·field/type schema를 확인했습니다. 이는 전체 pagination,
+모든 series의 cross-source identity, 36·60개월 completeness, production schedule·권리 판정 또는
+첫 publication 승인이 아닙니다. 현재 구현의 browser evidence는 disposable PostgreSQL의 합성
+fixture를 사용하며 live historical coverage로 해석하지 않습니다. 자세한 경계는
+[vNext source gate](docs/VNEXT-SOURCE-GATE.md)에 있습니다.
+
 ## local 실행
 
 ```sh
@@ -81,10 +109,15 @@ platform을 선택한 뒤 사람이 별도로 승인하는 production checkpoint
 - [기술 결정](docs/TECHNOLOGY-DECISIONS.md)
 - [MVP 인수 기준](docs/MVP-ACCEPTANCE.md)
 - [첫 구현 계획과 source gate 증거](docs/IMPLEMENTATION-PLAN.md)
-- [Phase 0 배포 직전 운영 런북](docs/OPERATIONS-RUNBOOK.md)
+- [vNext 제품·source 계약](docs/VNEXT-PRODUCT-CONTRACT.md)
+- [vNext source gate](docs/VNEXT-SOURCE-GATE.md)
+- [vNext public-read 계약](docs/VNEXT-PUBLIC-READ-CONTRACT.md)
+- [Phase 0 역사적 기준을 포함한 현재 운영 런북](docs/OPERATIONS-RUNBOOK.md)
 - [Phase 0 배포 직전 완료 보고서](docs/COMPLETION-REPORT.md) — local gate 결과와 production
-  사람 checkpoint를 고정
+  사람 checkpoint를 보존하며 vNext 결과는 별도 부록으로 기록
 
-production platform·PostgreSQL·role credential·secret store·domain·DNS 선택, 실제 배포와 traffic
-전환은 사람 전용 작업입니다. 이 저장소는 네이티브 앱, 앱스토어 배포나 별도 SPA를 포함하지
-않습니다.
+production platform·PostgreSQL·role credential·secret store·domain·DNS 선택, historical code
+manifest·mapping 승인, 첫 production collection·review·seal·activation, 실제 배포와 traffic
+전환은 사람 전용 작업입니다. historical 전용 health·authoritative inspection·backup canonical
+검증과 이전 application의 migration `0028` 호환성도 production 전 별도 증명이 필요합니다.
+이 저장소는 네이티브 앱, 앱스토어 배포나 별도 SPA를 포함하지 않습니다.
diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
index cc10861..1aa2098 100644
--- a/docs/OPERATIONS-RUNBOOK.md
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -1,11 +1,12 @@
-# Phase 0 배포 직전 운영 런북
+# Phase 0 역사적 기준을 포함한 vNext 운영 런북
 
 ## 문서 상태와 범위
 
-이 문서는 현재 저장소를 **Phase 0 배포 직전 production candidate**로 검증하고, 사람이
-production platform을 선택한 뒤 따라야 할 안전한 배포·복구 순서를 정의한다. 실제
-production platform, PostgreSQL, credential, domain·DNS는 아직 선택하거나 변경하지 않았고,
-이 문서는 실제 배포 완료를 주장하지 않는다.
+이 문서는 Phase 0 배포 직전 기준을 역사적 증거로 보존하면서 현재 저장소의 **vNext local
+implementation candidate**를 사람이 production에 옮기기 전에 따라야 할 안전한 배포·복구
+순서를 정의한다. 실제 production platform, PostgreSQL, credential, domain·DNS는 아직
+선택하거나 변경하지 않았고, vNext production migration·publication activation·traffic 전환도
+수행하지 않았다. 이 문서는 실제 배포 완료나 production readiness를 주장하지 않는다.
 
 명령의 `$NAME`은 운영자가 승인된 CI 변수 또는 managed configuration·secret store에서
 주입해야 하는 placeholder다. 이 문서, shell history, process argument, URL, log, ticket와
@@ -28,12 +29,12 @@ repository root에서 실행한다.
 | 영역 | 현재 제공 범위 | production 판단 |
 |---|---|---|
 | web | Django WSGI와 고정 Gunicorn dependency | platform process·HTTPS·traffic switch가 필요 |
-| schema | `0001`부터 `0010`까지 create/add 중심 migration과 DB trigger | production 복제 DB에서 plan·lock·시간 검증 필요 |
-| ingestion | `ingest_kamis_recent`의 bounded fetch·parse·audit와 typed 24시간 singleton schedule 계약 | managed `KAMIS_API_KEY`, 실제 singleton scheduler·egress·alert 필요 |
-| review/publication | local rehearsal과 fail-closed production command boundary | external MFA·IAM, 역할별 DB credential과 실제 actor provisioning 필요 |
-| public read | active `RECENT_RETAIL` revision만 읽는 SSR | production DB의 승인 pointer와 smoke test 필요 |
-| health | liveness, readiness, freshness endpoint와 scheduler command | platform probe·alert routing이 필요 |
-| backup | local Compose DB용 custom dump·isolated restore 검증 | production backup, encryption, PITR를 제공하지 않음 |
+| schema | `0001`부터 `0028`까지의 recent·historical typed model, constraint와 DB trigger | production 복제 DB에서 plan·lock·시간·이전 release 호환성 검증 필요 |
+| ingestion | recent 24시간, monthly 최대 168시간, regional·market 최대 24시간의 bounded singleton command | managed `KAMIS_API_KEY`, 실제 singleton scheduler·egress·alert 필요 |
+| review/publication | recent와 historical의 독립 review·seal·CAS transition command boundary | external MFA·IAM, 역할별 DB credential과 실제 actor provisioning 필요 |
+| public read | 독립 active `RECENT_RETAIL`·`HISTORICAL_RETAIL`을 읽는 no-JS SSR | production의 두 승인 pointer와 전체 route smoke가 필요 |
+| health | liveness와 recent-only readiness·freshness endpoint·scheduler command | historical 전용 monitor가 없으므로 승인된 별도 probe·alert가 필요 |
+| backup | Phase 0 recent publication용 local custom dump·isolated restore 검증 | historical canonical publication 검증, production encryption·PITR가 필요 |
 | logs | allowlist 기반 JSON event, 고정 message code와 exact deploy version | log 수집·보존·alert 담당자와 platform access-log 제거가 필요 |
 
 `approve_recent_generation`, `seal_recent_publication`과
@@ -87,6 +88,8 @@ production web process에는 최소한 다음 이름이 필요하다.
 - `DJANGO_TRUST_X_FORWARDED_PROTO`
 - `DATABASE_CONN_MAX_AGE`
 - `KAMIS_CONFIRMATION_MAX_AGE_HOURS`
+- `KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS`
+- `KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS`
 - `QA_STATE_PREVIEWS_ENABLED`
 - `CONTROL_PLANE_OPERATIONS_ENABLED`
 
@@ -111,6 +114,14 @@ build와 운영 명령에서는 다음 non-secret placeholder를 사용할 수 
 - `PUBLIC_COPY_REVISION`, `PUBLICATION_REVISION_ID`, `ROLLBACK_TARGET_REVISION_ID`
 - `PUBLICATION_OPERATION_ID`, `PUBLICATION_ACCEPTANCE_EVIDENCE_SHA256`
 - `EXPECTED_PUBLICATION_VERSION`, `EXPECTED_CURRENT_REVISION_ID`
+- `HISTORICAL_COLLECTION_ID`, `HISTORICAL_SOURCE_CONFIGURATION_ID`
+- `HISTORICAL_CODE_MANIFEST_SHA256`, `HISTORICAL_START`, `HISTORICAL_END`
+- `HISTORICAL_CATEGORY_CODE`, `HISTORICAL_REGION_CODE`, `HISTORICAL_REVIEW_ID`
+- `RECONCILIATION_REPORT_SHA256`, `MONTHLY_REVIEW_ID`, `REGIONAL_REVIEW_ID`, `MARKET_REVIEW_ID`
+- `COMPATIBILITY_REPORT_SHA256`, `HISTORICAL_PUBLICATION_REVISION_ID`
+- `HISTORICAL_PUBLICATION_OPERATION_ID`, `HISTORICAL_PUBLICATION_EVIDENCE_SHA256`
+- `EXPECTED_HISTORICAL_VERSION`, `EXPECTED_HISTORICAL_REVISION_OR_NONE`
+- `DISPOSABLE_VNEXT_DATABASE_URL`
 
 ### 최소권한 분리
 
@@ -140,7 +151,7 @@ credential 하나에 모든 권한을 합치거나 enable flag를 인증으로 
 - HTTPS endpoint, 승인된 domain·DNS 변경 계획과 인증서
 - managed secret injection과 회전·폐기 담당자
 - query string, IP, User-Agent와 search term을 제거한 log pipeline
-- liveness/readiness/freshness probe와 on-call alert route
+- liveness와 recent readiness·freshness probe, 별도 historical monitor와 on-call alert route
 - 암호화 backup, PITR, retention과 restore rehearsal 계획
 - production review·publication control plane과 MFA
 
@@ -253,8 +264,14 @@ migration과 traffic 공개를 중단한다.
 .venv/bin/python manage.py check
 ```
 
-plan을 exact release의 migration 파일과 대조한다. 현재 `0001`~`0010`은 새 model, constraint와
-trigger를 만드는 방향이다. `0010`은 고정 legacy source row의 date-precision
+plan을 exact release의 migration 파일과 대조한다. 현재 schema는 `0001`~`0028`이다.
+`0011`~`0028`은 historical source scope, typed monthly·regional·market fact, independent
+review·publication·activation, append-only·CAS guard, last-known-good rollback과 source별
+schedule interval constraint를 추가한다. create/add만 있는 migration 묶음이 아니므로 production
+복제 DB에서 constraint 교체와 trigger DDL의 lock·실행 시간, 기존 row 적합성과 이전 application
+호환성을 따로 측정한다.
+
+역사적 Phase 0 migration `0010`은 고정 legacy source row의 date-precision
 `state_changed_at`·`rights_confirmed_at` 원본을 수정·삭제하지 않고, 그 원본에만 적용되는
 append-only correction을 추가한다. 정확한 live 관측 시각은 보존되지 않았다. effective
 `2026-08-30T02:23:44Z`는 commit
@@ -263,9 +280,8 @@ append-only correction을 추가한다. 정확한 live 관측 시각은 보존
 `49143c27-d2dd-5fbd-b1dc-4aa3cc002fab`의 insert는 base·chronology trigger로 검증되고 update/delete는
 DB에서 거부된다. bootstrap·review·inspection은 effective helper를 쓰며 새 DB는 처음부터 exact
 effective 값으로 생성되어 correction row가 없다. 기존 canonical row를 삭제·변환하는 contract
-migration은 없다.
-그래도 production data 규모에서 DDL lock과 실행 시간을 복제 DB로 측정하고, 예상 밖 operation이
-보이면 중단한다.
+migration은 없다. 예상 밖 operation이나 constraint 위반 row가 보이면 중단하며 production
+데이터를 즉석에서 수정해 migration을 통과시키지 않는다.
 
 승인된 maintenance/change window에서만 forward migration을 실행한다.
 
@@ -280,7 +296,8 @@ traffic을 새 application으로 전환하지 않고 이전 application을 유
 `showmigrations --plan`과 database audit로 확인하며 임의 `--fake`나 reverse migration으로
 감추지 않는다.
 
-첫 빈 database는 아직 active publication이 없으므로 readiness가 실패하는 것이 정상이다.
+첫 빈 database는 아직 active recent publication이 없으므로 readiness가 실패하는 것이 정상이다.
+historical pointer 유무는 현재 repository readiness 결과를 바꾸지 않는다.
 production reviewer/publisher 경계가 준비되지 않은 상태에서 readiness를 통과시키려고 local
 actor를 복사하거나 pointer를 직접 수정하지 않는다.
 
@@ -312,12 +329,17 @@ traffic 전환 순서는 다음과 같다.
 
 1. 새 release를 traffic 없이 시작한다.
 2. 새 release의 liveness를 확인한다.
-3. database migration과 active publication을 포함한 readiness를 확인한다.
-4. freshness를 별도 확인한다. stale은 last-known-good가 제공됨을 뜻하며 readiness와 다르다.
-5. catalog와 승인된 `SMOKE_SERIES_PATH`를 query string 없이 읽어 source date, unit, coverage와
-   publication fact-set header가 한 revision인지 확인한다.
-6. platform의 atomic traffic switch를 사람이 승인·실행한다.
-7. 전환 뒤 같은 검사를 반복하고 이전 release를 즉시 rollback 가능한 상태로 유지한다.
+3. database migration과 active recent publication을 포함한 repository readiness를 확인한다.
+4. recent freshness를 별도 확인한다. stale은 last-known-good가 제공됨을 뜻하며 readiness와
+   다르다. 이 endpoint가 historical freshness를 검사한다고 해석하지 않는다.
+5. catalog와 승인된 `SMOKE_SERIES_PATH`의 detail을 query string 없이 읽어 source date, unit,
+   coverage와 recent fact-set header가 한 revision인지 확인한다.
+6. production historical bundle이 승인·활성화된 release라면 같은 series의 history, regions,
+   markets와 selection SSR을 순회한다. 실제로 읽은 publication에만 recent·historical fact-set
+   header가 붙고 두 hash가 각자 고정되며, 모든 response가 no-store·no-referrer·no-script인지
+   확인한다.
+7. platform의 atomic traffic switch를 사람이 승인·실행한다.
+8. 전환 뒤 같은 검사를 반복하고 이전 release를 즉시 rollback 가능한 상태로 유지한다.
 
 platform이 아직 선택되지 않았으므로 실제 traffic switch와 release rollback CLI는 이 문서에
 꾸며내지 않는다. 선택 후 vendor의 exact command, account, application ID, timeout과 rollback
@@ -375,6 +397,11 @@ DB state를 바꾸는 작업을 병행하지 않는다.
 종료 직후 `inspect_recent_publication`을 다시 실행해 active revision·version·fact-set hash가 시작
 receipt와 같은지 대사한다.
 
+이 고정 profile은 Phase 0 recent catalog·list·search·detail만 부하 대상으로 삼는다. vNext
+history·regions·markets·selection의 성능, historical fact-set 안정성 또는 production capacity를
+검증하지 않는다. 해당 route의 production 성능 인수는 실제 historical bundle과 platform을
+확정한 뒤 별도 승인된 profile로 측정해야 하며 기존 recent 결과로 대신하지 않는다.
+
 ## Health와 freshness
 
 health URL에는 credential이나 query string을 붙이지 않는다. 아래 조회는 고정된 작은 JSON만
@@ -390,15 +417,17 @@ curl --proto '=https' --tlsv1.2 --fail --silent --show-error \
 ```
 
 - `/health/live`: Django process가 응답하면 성공한다. DB와 publication을 검사하지 않는다.
-- `/health/ready`: DB 연결, migration currency와 sealed active publication을 검사한다. current와
-  stale publication 모두 last-known-good read가 가능하므로 성공할 수 있다.
-- `/health/freshness`: active publication이 current일 때만 성공한다. stale 또는 unavailable은
-  실패 상태이며 operator alert 대상이다.
+- `/health/ready`: DB 연결, migration currency와 sealed active `RECENT_RETAIL` publication을
+  검사한다. recent current와 stale publication은 모두 last-known-good read가 가능하므로 성공할
+  수 있다. `HISTORICAL_RETAIL` pointer·shape·freshness는 검사하지 않는다.
+- `/health/freshness`: active `RECENT_RETAIL` publication이 current일 때만 성공한다. recent stale
+  또는 unavailable은 실패 상태이며 operator alert 대상이다. historical freshness endpoint가
+  아니다.
 
 platform liveness는 process restart 판단에, readiness는 traffic 편입 판단에 사용한다.
 freshness 실패만으로 application을 재시작하거나 승인 pointer를 자동 변경하지 않는다.
 
-scheduler에서도 동일 freshness 경계를 확인할 수 있다.
+scheduler에서도 recent와 동일한 freshness 경계만 확인할 수 있다.
 
 ```sh
 .venv/bin/python manage.py check_recent_publication_freshness
@@ -408,6 +437,12 @@ scheduler에서도 동일 freshness 경계를 확인할 수 있다.
 `RECENT_PUBLICATION_FRESHNESS_*` code와 non-zero exit를 반환한다. scheduler는 stdout의 raw
 재가공이나 exception text 대신 exit status와 fixed code만 alert에 연결한다.
 
+현재 repository에는 `check_historical_publication_freshness`나 동등한 historical health command가
+없다. production historical 공개 전 세 collection 완료 시각, independent pointer, sealed fact-set과
+monthly·daily freshness를 검증하는 승인된 read-only probe·alert를 별도로 마련해야 한다. 이 gap을
+recent readiness 성공이나 public 화면의 stale 문구로 대체하지 않으며, monitor evidence가 없으면
+historical traffic 공개를 중단한다.
+
 현재 local evidence에서 active artifact의 마지막 source 확인 시각은
 `2026-08-30T04:00:48.696744Z`다. 36시간 다음 확인 경계는
 `2026-08-31T16:00:48.696744Z`(`2026-09-01 01:00:48 KST`)이며, 이 시각은 배포 시점이나
@@ -422,19 +457,48 @@ ingestion 성공은 review, seal 또는 activation 성공이 아니다. schedule
 실행하고 자동 approve·seal·activate를 연결하지 않는다. source 실패, parse 실패 또는 새
 candidate는 현재 pointer를 바꾸지 않는다.
 
-active A-path `SourceConfiguration`은 `schedule_execution_mode=PLATFORM_SINGLETON`,
-`schedule_interval_hours=24`를 기록한다. production platform scheduler는 성공 여부와 관계없이
-인접한 scheduled start 사이를 24시간보다 길게 두지 않고, 이전 실행이 남아 있으면 새 실행을
-겹치거나 catch-up burst로 만들지 않는다. bounded attempt가 끝난 뒤 fixed exit·audit를 alert에
-연결한다. 이 값은 자동 review·seal·activation 권한이 아니며 36시간 freshness 경계 전 확인과
-운영 대응 시간을 확보하기 위한 최대 cadence 계약이다.
+각 `SourceConfiguration`은 `schedule_execution_mode=PLATFORM_SINGLETON`과 source별 interval을
+기록한다. recent·regional·market은 최대 24시간, monthly는 최대 168시간이다. production platform
+scheduler는 source별로 이전 실행이 남아 있으면 같은 source 실행을 겹치거나 catch-up burst로
+만들지 않는다. bounded attempt가 끝난 뒤 fixed exit·audit를 alert에 연결한다. 이 값은 자동
+review·seal·activation 권한이 아니며 public freshness보다 먼저 운영 대응 시간을 확보하기 위한
+최대 cadence 계약이다.
 
-production scheduler는 중첩 실행을 막는 singleton job으로 다음 명령만 실행한다.
+production scheduler는 서로 독립된 singleton job으로 ingestion command만 실행한다. 여러
+command를 하나의 성공 chain으로 묶거나 하나의 실패를 다른 source data로 보충하지 않는다.
 
 ```sh
 .venv/bin/python manage.py ingest_kamis_recent
+
+.venv/bin/python manage.py ingest_kamis_monthly \
+  --collection-id "$HISTORICAL_COLLECTION_ID" \
+  --source-configuration-id "$HISTORICAL_SOURCE_CONFIGURATION_ID" \
+  --code-manifest-sha256 "$HISTORICAL_CODE_MANIFEST_SHA256" \
+  --start "$HISTORICAL_START" --end "$HISTORICAL_END" \
+  --category-code "$HISTORICAL_CATEGORY_CODE"
+
+.venv/bin/python manage.py ingest_kamis_regional_daily \
+  --collection-id "$HISTORICAL_COLLECTION_ID" \
+  --source-configuration-id "$HISTORICAL_SOURCE_CONFIGURATION_ID" \
+  --code-manifest-sha256 "$HISTORICAL_CODE_MANIFEST_SHA256" \
+  --start "$HISTORICAL_START" --end "$HISTORICAL_END" \
+  --category-code "$HISTORICAL_CATEGORY_CODE" \
+  --region-code "$HISTORICAL_REGION_CODE"
+
+.venv/bin/python manage.py ingest_kamis_market_daily \
+  --collection-id "$HISTORICAL_COLLECTION_ID" \
+  --source-configuration-id "$HISTORICAL_SOURCE_CONFIGURATION_ID" \
+  --code-manifest-sha256 "$HISTORICAL_CODE_MANIFEST_SHA256" \
+  --start "$HISTORICAL_START" --end "$HISTORICAL_END" \
+  --category-code "$HISTORICAL_CATEGORY_CODE"
 ```
 
+세 historical invocation은 같은 placeholder 이름을 예시로 쓸 뿐 실제 값이나 collection UUID를
+공유한다는 뜻이 아니다. 각 job은 자기 dataset의 승인 source configuration, 새 logical collection
+UUID, exact 기간·부류와 검토된 partition을 managed configuration으로 받는다. 더 좁은 품목·품종·
+등급 또는 여러 지역 partition은 승인 manifest에 있는 typed argument만 반복 전달하며 임의 shell
+문자열을 조립하지 않는다.
+
 `KAMIS_API_KEY`는 worker process 안에서만 managed secret으로 주입한다. command argument, URL,
 shell trace, log 또는 receipt로 전달하지 않는다. 실패 시 key를 확인하려고 `env`, `printenv`,
 `echo`, shell tracing 또는 exception traceback을 사용하지 않는다.
@@ -589,6 +653,12 @@ decision·operation UUID를 재사용한다. `ACTIVATE`는 current review tail
 대상으로 한다. production actor provisioning, 첫 review·seal·activation, traffic 전환과 rollback은
 모두 사람 전용 checkpoint다.
 
+repository에는 historical channel의 current revision·version·activation chain·canonical fact-set을
+한 read-only snapshot에서 재계산하는 authoritative inspection command가 아직 없다. 최초 상태를
+추정하거나 seal receipt만으로 CAS 값을 정하지 않는다. production historical activation,
+rollback·withdraw 전에 외부 MFA control plane의 승인된 repeatable-read inspection과 immutable
+audit를 마련하고 실제 결과를 대사해야 하며, 그 전에는 해당 operation을 실행하지 않는다.
+
 browser evidence fixture는 production command가 아니다. `DEBUG=1`, Admin 비활성, QA preview
 활성, loopback PostgreSQL, `grocery_vnext_`로 시작하는 비어 있는 DB를 모두 확인한 뒤에만 실행된다.
 핵심 source·domain·publication row가 하나라도 있으면 실패 폐쇄한다.
@@ -602,8 +672,9 @@ DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=1 \
 `PUBLIC_COPY_REVISION`은 현재 `ko-v1`, `ko-v2`, `ko-v3` 또는 `ko-v4`만 허용한다. `ko-v3`는
 초록장부 frontend redesign copy이고 `ko-v4`는 historical consumer 확장 copy다. 기존 revision
 row를 수정하지 않고 새 sealed revision으로만 만든다. production의 `ko-v4` seal·activation,
-traffic 전환과 rollback 결정은 사람 checkpoint다. 첫 activation은 authoritative
-inspection의 `version=0`, current literal `NONE`을 사용한다. production receipt는 actor,
+traffic 전환과 rollback 결정은 사람 checkpoint다. 첫 activation은 위에서 요구한 승인된
+read-only inspection이 빈 channel을 확인했을 때만 `version=0`, current literal `NONE`을
+사용한다. production receipt는 actor,
 release SHA와 evidence hash를 출력하지 않는다. ReviewDecision·Activation은 actor를 DB audit에
 보존하지만 seal invoker는 revision row에 저장되지 않으므로 외부 MFA job audit와 change record가
 필수다. actor bootstrap, IAM, grant와 첫 production publication은 사람 checkpoint다.
@@ -620,7 +691,7 @@ activation이다.
 읽는다. 과거 receipt나 기억한 값을 사용하지 않는다. 두 값이 예상과 다르면 concurrent change로
 간주하고 실패 폐쇄한다.
 
-현재 repository의 authoritative read-only 명령은 다음과 같다.
+현재 repository의 authoritative read-only 명령은 recent channel에만 다음과 같다.
 
 ```sh
 .venv/bin/python manage.py inspect_recent_publication
@@ -632,6 +703,11 @@ canonical entry 집합과 fact-set hash를 다시 계산한다. `AVAILABLE` rece
 첫 빈 channel 또는 withdraw 상태는 current revision을 literal `NONE`으로 출력한다. `ERROR`나
 non-zero exit이면 transition하지 않는다.
 
+`inspect_recent_publication`은 `HISTORICAL_RETAIL`의 expected version·current revision이나
+canonical fact-set을 제공하지 않는다. historical rollback·withdraw는 위 historical inspection
+gap을 해결하고 approved last-known-good membership과 activation history를 별도 대사하기 전에는
+production에서 실행하지 않는다.
+
 local rehearsal rollback 명령은 다음과 같다.
 
 ```sh
@@ -661,9 +737,9 @@ env DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=0 \
   --expected-current-revision "$EXPECTED_CURRENT_REVISION_ID"
 ```
 
-withdraw 후 readiness와 freshness가 unavailable이 되고 public catalog에 공개 자료가 없으며
-detail이 숨겨지는지 확인한다. 이는 장애가 아니라 안전한 철회 상태일 수 있으므로 운영 change
-record와 alert suppression window를 함께 남긴다.
+recent withdraw 후 readiness와 freshness가 unavailable이 되고 public catalog에 공개 자료가
+없으며 detail이 숨겨지는지 확인한다. 이는 장애가 아니라 안전한 철회 상태일 수 있으므로 운영
+change record와 alert suppression window를 함께 남긴다.
 
 production rollback·withdraw는 위 publisher private job에서 같은 명령에
 `--expected-release-sha "$RELEASE_SHA"`를 추가해 실행한다. external MFA·IAM, 역할별 credential,
@@ -677,12 +753,13 @@ application rollback은 database와 publication을 독립적으로 유지한 채
 
 1. 새 traffic 편입을 중단하고 `RELEASE_SHA`, request ID, health 상태와 최초 이상 시각만
    기록한다. query, user input 또는 secret은 기록하지 않는다.
-2. migration이 이미 성공했다면 schema를 그대로 둔다. 현재 migration은 additive 방향이지만,
-   `PREVIOUS_RELEASE_SHA`가 최신 schema를 읽을 수 있다고 release별로 실제 검증한 경우에만
-   rollback한다.
+2. migration이 이미 성공했다면 schema를 그대로 둔다. 현재 migration에는 constraint·trigger
+   교체가 있으므로 `PREVIOUS_RELEASE_SHA`가 최신 schema를 읽을 수 있다고 release별로 실제
+   검증한 경우에만 rollback한다.
 3. platform의 승인된 atomic release switch로 application과 static을 함께
    `PREVIOUS_RELEASE_SHA`에 되돌린다.
-4. liveness, readiness, freshness와 catalog/detail smoke를 다시 수행한다.
+4. liveness, recent readiness·freshness, catalog/detail과 활성화된 vNext historical route smoke를
+   다시 수행한다.
 5. publication 내용 자체가 문제면 application rollback과 별도로 rollback 또는 withdraw한다.
 
 production에서 migration을 역방향 실행하여 code rollback을 맞추지 않는다. 이전 code가 최신
@@ -692,7 +769,11 @@ database를 덮어쓰지 않고 managed PITR로 새 instance를 만든 뒤 검
 
 local application rollback rehearsal에서 실제로 검증한 `PREVIOUS_RELEASE_SHA`는
 `d6d7d08c9de9a78eb597fec6e232b0e2d24a1ec1`다. 이는 local 호환성 evidence의 고정 target이지,
-향후 production vendor rollback 명령이나 실제 traffic 전환을 승인한 값이 아니다.
+향후 production vendor rollback 명령이나 실제 traffic 전환을 승인한 값이 아니다. 이 rehearsal은
+Phase 0 schema `0010`까지의 역사적 evidence이며 migration `0011`~`0028`과 historical route가
+추가된 현재 schema에서 이전 application의 호환성을 증명하지 않는다. vNext production rollback
+target은 exact 이전 release를 `0028` schema와 별도 검증하기 전까지 미정이며, 검증되지 않은 위
+SHA로 traffic을 되돌리지 않는다.
 
 ## Local backup·restore rehearsal
 
@@ -702,10 +783,12 @@ socket만 사용한다. ambient Docker context·host와 `DATABASE_URL`을 읽거
 container ID·image·project/service identity를 한 invocation 동안 pin한 뒤 모든 `docker exec`에
 그 ID를 직접 사용한다. container가 교체·중지되거나 identity가 달라지면 실패 폐쇄하므로
 production database나 다른 Compose service를 대상으로 사용할 수 없다. PostgreSQL custom dump,
-manifest·dump hash, migration inventory, 모든 public table row count, active revision, canonical
-publication hash와 activation chain을 대사한다.
-이 도구는 sealed active publication이 있는 candidate DB만 허용하며 빈 DB나 withdraw 상태의
-범용 backup 도구가 아니다.
+manifest·dump hash, migration inventory, 모든 public table row count와 Phase 0 `RECENT_RETAIL`
+active revision·canonical publication hash·activation chain을 대사한다.
+이 도구는 sealed active recent publication이 있는 candidate DB만 허용하며 빈 DB나 withdraw
+상태의 범용 backup 도구가 아니다. historical table row는 dump·row-count inventory에 포함될 수
+있지만 `HISTORICAL_RETAIL` review membership, three-collection compatibility, canonical fact-set과
+activation chain을 재계산하지 않는다. 이를 vNext historical backup 검증으로 보고하지 않는다.
 
 `BACKUP_OUTPUT_DIRECTORY`는 repository 밖의 기존 absolute owner-controlled directory여야 하고
 symlink가 아니어야 한다. 도구는 directory와 parent의 owner·write 경계를 검사하고 열린 directory
@@ -734,9 +817,9 @@ restore는 새 target database를 만들고 source database를 변경하지 않
   --target-database "$RESTORE_DATABASE_NAME"
 ```
 
-성공 receipt에서 row counts, migration inventory와 publication contract가 모두 일치하는지
-확인한다. 그 뒤 별도 process의 `DATABASE_URL`을 격리 target에 managed 방식으로 연결하여 다음을
-실행한다.
+성공 receipt에서 row counts, migration inventory와 recent publication contract가 모두
+일치하는지 확인한다. 그 뒤 별도 process의 `DATABASE_URL`을 격리 target에 managed 방식으로
+연결하여 다음을 실행한다.
 
 ```sh
 env DATABASE_URL="postgresql://grocery:local-grocery-only@127.0.0.1:55434/$RESTORE_DATABASE_NAME" \
@@ -745,9 +828,14 @@ env DATABASE_URL="postgresql://grocery:local-grocery-only@127.0.0.1:55434/$RESTO
   .venv/bin/python manage.py check
 ```
 
-복원 target으로 시작한 local application에서 readiness와 대표 catalog/detail도 확인한다.
+복원 target으로 시작한 local application에서 recent readiness와 대표 catalog/detail도 확인한다.
 원본 Compose DB를 가리키고 있지 않은지 database name을 먼저 read-only로 확인한다.
 
+vNext production 전에는 별도 rehearsal에서 historical review·collection membership, sealed
+fact-set, activation chain과 history·regions·markets read를 새 restore target에서 다시 계산하고
+두 publication header가 원본과 일치함을 증명해야 한다. 현재 local Phase 0 restore evidence는 이
+historical 검사를 통과한 것으로 재사용하지 않는다.
+
 target 생성 뒤 restore·inventory·canonical publication 검사 중 어느 단계가 실패해도 도구는 같은
 invocation이 만든 exact target만 identity-pin한 container에서 자동 삭제하고 실제 부재를 다시
 확인한다. pre-existing target, source `grocery`, 다른 이름이나 다른 container는 삭제하지 않는다.
@@ -767,14 +855,15 @@ production 공개 전 managed PostgreSQL에서 다음을 실제로 증명해야
 - application, migration, backup·restore 역할의 분리와 감사
 - backup retention, region·account 격리와 실패 alert
 - 기존 database를 덮어쓰지 않는 새 instance restore
-- migration inventory, row counts, audit chain, active revision, fact-set hash와 public read 대사
+- migration inventory, row counts, recent·historical audit chain, 두 active revision과 독립
+  fact-set hash·public read 대사
 - 목표 `RPO 24시간`과 `RTO 4시간` 안의 timed rehearsal
 - 분기별 restore rehearsal과 evidence retention
 
-현재 candidate에는 production platform, encrypted scheduled backup, PITR, production restore,
-backup failure structured event와 실제 RPO/RTO 측정이 없다. local Compose rehearsal을 이 항목의
-통과로 기록하지 않는다. backup/PITR가 구성·복원 검증되지 않으면 production migration과
-traffic 공개를 중단한다.
+현재 candidate에는 historical canonical restore rehearsal뿐 아니라 production platform,
+encrypted scheduled backup, PITR, production restore, backup failure structured event와 실제
+RPO/RTO 측정이 없다. local Phase 0 Compose rehearsal을 이 항목의 통과로 기록하지 않는다.
+backup/PITR가 구성·복원 검증되지 않으면 production migration과 traffic 공개를 중단한다.
 
 PITR는 항상 새 database instance로 복원하고 검증 후 connection을 전환한다. 기존 instance
 삭제, in-place overwrite, retention 단축과 old backup 폐기는 파괴적 사람 checkpoint다.
@@ -791,11 +880,16 @@ log도 query string, IP, User-Agent와 search term을 별도로 제거해야 한
 
 | code | 의미 | 초기 대응 |
 |---|---|---|
-| `health.readiness.unavailable` | DB, migration 또는 active publication read 불가 | traffic 편입 중단; DB·schema·pointer 분리 확인 |
-| `health.freshness.unavailable` | active publication 또는 freshness 판단 불가 | 자동 publication 금지; 권리·pointer·attempt 확인 |
-| `health.freshness.stale` | last-known-good는 있으나 새 확인 필요 | 공개본 유지; ingestion·review backlog 조사 |
+| `health.readiness.unavailable` | DB, migration 또는 active recent publication read 불가 | traffic 편입 중단; DB·schema·recent pointer 분리 확인 |
+| `health.freshness.unavailable` | recent publication 또는 freshness 판단 불가 | 자동 publication 금지; 권리·recent pointer·attempt 확인 |
+| `health.freshness.stale` | recent last-known-good는 있으나 새 확인 필요 | 공개본 유지; recent ingestion·review backlog 조사 |
 | `public.catalog.unavailable` | catalog read 실패 | request ID로 DB/read 경계 확인; 반복 시 application rollback 검토 |
 | `public.detail.unavailable` | detail read 실패 | request ID로 active revision membership/read 확인 |
+| `public.detail.history_hidden` | historical link 판정 실패; recent detail은 계속 제공 | historical pointer·mapping·bundle 무결성 확인; recent pointer는 유지 |
+| `public.history.unavailable` | 월별 public read 실패 | historical monthly collection·series membership·fact-set 확인 |
+| `public.regions.unavailable` | 지역별 public read 실패 | historical regional collection·공통 조사일·fact-set 확인 |
+| `public.markets.unavailable` | 시장별 public read 실패 | historical market collection·region membership·pagination 확인 |
+| `public.selection.unavailable` | recent 선택 목록 read 실패 | recent membership·canonical GET state 확인; query 원문은 기록하지 않음 |
 | `ingest.source.start_failed` | source/audit 시작 실패 | DB와 source configuration 확인; pointer 유지 |
 | `ingest.fetch.failed` | bounded fetch 실패 | HTTP class·quota·credential 상태를 redacted receipt로 확인 |
 | `ingest.fetch.finalization_failed` | 실패 attempt 종료 기록 실패 | audit 불완전 incident로 즉시 escalation |
@@ -810,14 +904,20 @@ started만 남는 run, 연속 fetch 실패, quarantined lifecycle status 증가
 platform 집계 규칙으로 경보한다.
 
 freshness command의 non-zero `RECENT_PUBLICATION_FRESHNESS_*` code와 ingest command의 fixed
-`INGEST_*` failure code도 scheduler alert에 연결한다. review·seal·transition은 현재 structured
-log를 내지 않고 DB audit와 fixed CLI receipt만 남긴다. private job은
+`INGEST_*` failure code, historical command의 `HISTORICAL_INGEST_*` non-zero code도 scheduler
+alert에 연결한다. review·seal·transition은 현재 structured log를 내지 않고 DB audit와 fixed CLI
+receipt만 남긴다. private job은
 `CONTROL_PLANE_*`, `RECENT_PUBLICATION_INSPECTION_FAILED`와 non-zero exit를 인자나 원문 오류 없이
 별도 platform audit·alert로 연결한다. local backup script도 structured logger에 연결되지 않으므로
 production control plane과 backup platform은 non-zero job, DB audit gap과 backup failure alert를
 추가해야 한다. actor/bootstrap·seal job audit 누락이나 ReviewDecision/Activation actor chain
 불일치도 production traffic 차단 신호다.
 
+historical 전용 freshness command가 없으므로 위 public error event가 historical health를
+대신한다고 간주하지 않는다. 세 source scheduler 실패, collection age, pointer·fact-set inspection을
+묶는 별도 bounded monitor와 alert route가 승인되기 전에는 historical production traffic을 열지
+않는다.
+
 유효한 `DEPLOY_VERSION`은 application JSON event에 자동 포함된다. production settings는 exact
 40자 lowercase Git SHA가 없으면 시작을 거부한다. platform도 immutable release metadata로 같은
 SHA를 보존해 application event와 ingress·runtime signal을 대조할 수 있어야 한다.
@@ -866,9 +966,10 @@ rotation 도구가 아니다.
 | 신호 | 유지할 것 | 금지할 자동 대응 |
 |---|---|---|
 | liveness 실패 | DB·publication을 그대로 두고 process/release 조사 | migration reverse, publication 전환 |
-| readiness 실패 | 이전 release traffic과 last-known-good 유지 | pointer 직접 update, 빈 DB를 ready로 가장 |
-| freshness stale | last-known-good 공개와 stale 표시 유지 | 미검수 candidate activate, 비공식 source 보충 |
-| freshness unavailable | 권리·active pointer·DB를 분리 조사 | 오래된 publication이 안전하다고 추정 |
+| recent readiness 실패 | 이전 release traffic과 last-known-good 유지 | pointer 직접 update, 빈 DB를 ready로 가장 |
+| recent freshness stale | recent last-known-good 공개와 stale 표시 유지 | 미검수 candidate activate, 비공식 source 보충 |
+| recent freshness unavailable | 권리·recent pointer·DB를 분리 조사 | 오래된 publication이 안전하다고 추정 |
+| historical monitor 불가 | recent 공개본은 유지하고 historical traffic·operation 중단 | recent health 성공으로 historical 상태 추정 |
 | fetch/parse 실패 | 현재 publication 유지, 새 attempt audit 보존 | partial generation 혼합, 무한 retry |
 | public read 반복 실패 | request ID와 release를 대조하고 application rollback 검토 | 오류 원문·query logging |
 | publication 오류 | optimistic state를 다시 읽고 별도 rollback/withdraw 승인 | revision 수정·삭제, 같은 UUID의 다른 action 재사용 |
@@ -886,16 +987,22 @@ traffic 공개 전 change record에는 값 자체가 아니라 다음 evidence l
 - exact `RELEASE_SHA`, clean pre-build Git 상태와 `git fsck`
 - locked dependency build, `production-check`, migration plan과 forward migration 결과
 - collectstatic artifact와 Gunicorn process revision
-- liveness, readiness, freshness와 catalog/detail smoke 결과
-- 현재 publication revision, pointer version과 마지막 activation ID
+- liveness, recent readiness·freshness, catalog/detail과 활성화된 vNext route smoke 결과
+- recent·historical publication별 revision, pointer version, fact-set과 마지막 activation ID
+- historical 전용 freshness monitor·authoritative inspection evidence와 known gap
 - backup/PITR checkpoint, restore rehearsal ID와 RPO/RTO 측정
 - alert route와 담당자, known gap와 다음 검토 시각
 - `PREVIOUS_RELEASE_SHA`, 이전 static artifact와 platform-specific rollback command locator
 
-다음 항목이 남아 있으면 상태는 계속 **Phase 0 배포 직전**이다.
+현재 상태는 **vNext local implementation candidate**이며 Phase 0 완료 보고서의 과거 evidence를
+production evidence로 확대하지 않는다. 다음 항목이 남아 있으면 production ready 또는 배포
+완료로 상태를 바꾸지 않는다.
 
 - production platform·PostgreSQL·secret store·domain·DNS가 미승인
 - production MFA reviewer/publisher control이 없음
+- production historical code manifest·mapping, 첫 collection·review·seal·activation이 미승인
+- historical freshness monitor·authoritative inspection과 canonical backup restore가 미검증
+- exact 이전 release의 migration `0028` 호환성과 vNext route rollback smoke가 미검증
 - production backup/PITR·restore와 RPO/RTO가 미검증
 - platform-specific deploy·traffic switch·application rollback command가 미기록
 - production alert delivery와 ingress log privacy가 미검증
diff --git a/docs/PRODUCT-DECISIONS.md b/docs/PRODUCT-DECISIONS.md
index 7f33e4d..9bed99a 100644
--- a/docs/PRODUCT-DECISIONS.md
+++ b/docs/PRODUCT-DECISIONS.md
@@ -13,16 +13,19 @@
 | 기본 브랜치 | `main` |
 | remote | `origin` (`https://github.com/seungwoo7050/audience-foundry-grocery-seasonality.git`) |
 | 공개 상태 | GitHub 공개 저장소·production service 미배포 |
-| 현재 구현 기준선 | `bb0b28038243c539db2eafcfebc05144d9d59d66` |
-| 구현 상태 | Phase 0와 production-grade SSR redesign 완료; vNext source gate 승인 전 |
+| vNext 시작 기준선 | `bb0b28038243c539db2eafcfebc05144d9d59d66` |
+| 구현 상태 | vNext local implementation candidate; production migration·publication·deploy 미실행 |
 
 legacy 코드, 경쟁 서비스 이력, 비공식 가격 dump, KAMIS HTML과 다른 Audience Foundry
 제품의 구현을 가져오지 않습니다. 재사용은 exact source·license·revision·검증·rollback을
 별도 결정으로 승인한 뒤에만 가능합니다.
 
-위 상태는 2026-08-31(KST)에 제품 소유자가 원격 연결과 공개 저장소 반영을 승인한 뒤
-`main == origin/main`, clean working tree와 GitHub repository visibility를 재검증한 결과입니다.
-production 배포·database·publication activation·traffic 공개를 뜻하지 않습니다.
+위 remote 공개 상태와 시작 기준선은 2026-08-31(KST)에 제품 소유자가 원격 연결과 공개
+저장소 반영을 승인한 뒤 `main == origin/main`, clean working tree와 GitHub repository
+visibility를 재검증한 당시 사실입니다. 이후 vNext candidate는 local에서만 만들었고
+`origin`을 변경하지 않았습니다. 현재 exact local SHA·branch·clean 상태는 완료 보고서가
+고정하며, 어떤 상태도 production 배포·database migration·publication activation·traffic
+공개를 뜻하지 않습니다.
 
 ## 변경할 수 없는 MVP 결정
 


