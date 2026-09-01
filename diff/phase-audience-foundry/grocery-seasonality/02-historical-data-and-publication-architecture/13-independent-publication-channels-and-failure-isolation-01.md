# 독립 게시 채널과 실패 격리

## `docs: approve historical consumer vnext`

diff --git a/docs/PRODUCT-DECISIONS.md b/docs/PRODUCT-DECISIONS.md
index 23294e6..7f33e4d 100644
--- a/docs/PRODUCT-DECISIONS.md
+++ b/docs/PRODUCT-DECISIONS.md
@@ -177,3 +177,12 @@ CAPTCHA·quota 우회와 다른 가격 source의 자동 보충은 pivot이 아
 고정 결정 변경에는 source evidence, 사용자 문구, data compatibility, privacy·license,
 schema·migration, acceptance와 rollback 영향을 기록합니다. 두 실제 vertical에서
 안정된 중복이 확인되기 전에는 공용 ingestion package를 추출하지 않습니다.
+
+## vNext 승인 결정
+
+제품 소유자는 2026-08-31(KST)에 공공데이터포털 `15156060`, `15156062`, `15156065`
+활용신청 완료와 개발계정 live gate를 승인했습니다. 지역 비교, 월별 과거 패턴, 시장별
+근거, 검증된 전체 series coverage, URL 기반 최대 5개 선택 목록과 안전한 GET state는
+[vNext 제품·source 계약](VNEXT-PRODUCT-CONTRACT.md)이 이 문서의 첫 MVP 비목표를 명시적으로
+대체하는 범위에서만 허용합니다. source gate 실패 시 이 승인은 dependent 구현 근거가 되지
+않으며 기존 `RECENT_RETAIL` 계약은 그대로 유지합니다.
diff --git a/docs/VNEXT-PRODUCT-CONTRACT.md b/docs/VNEXT-PRODUCT-CONTRACT.md
new file mode 100644
index 0000000..89931da
--- /dev/null
+++ b/docs/VNEXT-PRODUCT-CONTRACT.md
@@ -0,0 +1,135 @@
+# vNext 제품·source 계약
+
+승인일은 2026-08-31(KST)다. 이 문서는 제품 소유자가 승인한 첫 MVP 이후의
+소비자 확장 계약이며, 실제 source gate가 통과하기 전에는 source-dependent schema나
+adapter 구현을 허용하지 않는다.
+
+## 제품 목표
+
+초록장부는 비로그인 한국어 사용자가 같은 품목·품종·등급·판매 단위 안에서 다음을
+확인하는 소비자용 조사 장부로 확장한다.
+
+- 최근 KAMIS 소매 조사 평균과 1주·1개월·1년 제공값의 차이
+- 선택한 지역의 월별 과거 가격 패턴
+- 같은 조사일의 지역별 소매 조사 평균·최저·최고
+- 선택 지역·조사일의 KAMIS 시장별 조사값
+- 계정 없이 최대 5개 품목을 모아 보는 URL 기반 선택 목록
+
+기능 수를 강조하는 dashboard가 아니라 품목을 중심으로 최근값, 월별 기록, 지역별 값과
+시장 근거를 순서대로 읽는 것이 핵심 사용자 흐름이다.
+
+## 승인 source와 역할
+
+| dataset | 승인 역할 | 금지된 결합 |
+|---|---|---|
+| `15156063` 최근일자 도·소매가격정보 | 최근 조사 평균과 1주·1개월·1년 제공값 | 역사·지역·시장 행을 채우는 fallback으로 사용하지 않음 |
+| `15156060` 연월별 도·소매가격정보 | provider 월 평균·최저·최고의 유일한 정본 | 일별·시장 행에서 월 값을 재계산하거나 보충하지 않음 |
+| `15156062` 지역별 품목별 도·소매가격정보 | provider 지역별 일 평균·최저·최고 | 시장 관측값을 합쳐 지역 평균을 재구성하지 않음 |
+| `15156065` 기간별 소매가격정보 | 지역·시장별 조사일 관측값 | 시장명 문자열로 시장유형을 추정하지 않음 |
+
+모든 source는 공공데이터포털과 한국농수산식품유통공사가 제공한 공식 계약만 사용한다.
+KAMIS HTML, 검색 cache, 비공식 mirror, 쇼핑몰·마트 crawling과 다른 가격 source는 사용하지
+않는다.
+
+## 공개 coverage
+
+다음 조건을 모두 만족하는 모든 exact series를 공개 후보로 삼는다.
+
+1. source가 소매, 채소류 또는 과일류로 식별한다.
+2. 네 API의 품목·품종·등급·원문 단위·단위크기가 공식 code와 이름까지 일치한다.
+3. 적어도 한 검증 지역에서 완전한 최근 36개월 월별 행을 제공한다.
+4. 지역·시장 code와 소속, 조사일 identity가 공식 문서와 반복 canary에서 안정적이다.
+5. duplicate, 단위 drift, 의미를 확인하지 못한 결측·sentinel과 coverage 충돌이 없다.
+
+이 조건을 통과한 series 수를 임의의 목표 숫자에 맞추지 않는다. 채소·과일이 각각 하나도
+없거나 네 source의 cross-source identity가 증명되지 않으면 vNext source path를 중단한다.
+이름 유사도, 자동 단위환산 또는 사람이 추정한 지역 mapping으로 coverage를 늘리지 않는다.
+
+## 기간과 수집 범위
+
+- 월별 공개 기본값은 최근 36개월이다.
+- 12개월은 36개월 gate를 통과한 series·지역에서 제공한다.
+- 60개월은 해당 series·지역에 완전한 60개월이 있을 때만 제공한다.
+- 지역·시장 일별 자료는 최근 31 calendar day 안의 실제 조사일만 공개한다.
+- 지역·시장 기본일은 두 source가 같은 series·region에서 공유하는 최신 조사일이다.
+- provider가 공식 aggregate `전체` 지역을 보장할 때만 월별 기본 지역으로 사용한다.
+  그렇지 않으면 사용자가 지역을 선택하기 전까지 월별 chart를 만들지 않는다.
+
+월별 source는 주 1회, 지역·시장 source는 24시간마다 platform singleton으로 확인한다.
+source gate가 계산한 최악 호출량이 개발계정 일일 quota의 50%를 넘으면 해당 schedule과
+구현을 승인하지 않는다.
+
+## 공개 사실과 표현
+
+허용하는 첫 표현은 다음과 같다.
+
+- `월별 과거 가격 패턴`
+- `지역별 소매 조사값`
+- `시장별 소매 조사값`
+- `2026년 7월 KAMIS 소매 조사 평균`
+- `조사일 평균이 1주 전 제공값보다 52원 높음 (+3.4%)`
+- `KAMIS가 이 기간의 값을 제공하지 않았습니다.`
+
+다음 표현과 기능은 계속 금지한다.
+
+- `제철`, `평년`, 품질·신선도·맛·영양 판단
+- `저렴하다`, `비싸다`, `최저가`, `시장 최저`, 구매 추천·구매 시점·절약액
+- 실시간 매장가격, 가격 전망과 미래 예측
+- 서로 다른 품종·등급·단위·지역을 합산하거나 직접 우열 비교
+- 시장명에 포함된 문자열로 대형마트·SSM·전통시장 같은 유형을 추정
+- 결측을 0원, 변화 없음, 품절 또는 비제철로 표현
+
+## 사용자 상태와 개인정보
+
+- 사용자 계정, 위치·GPS, 개인화, 알림, 최근 본 품목, server-side 즐겨찾기, analytics와
+  광고 audience를 만들지 않는다.
+- 검색·부류·비교기간·방향·정렬·날짜·지역·선택 품목은 allowlist된 GET state로만 유지한다.
+- 정상화된 유효값은 form과 canonical URL에 다시 표시할 수 있다.
+- query state는 cookie, session, database, cache, analytics, application·proxy log와 audit에
+  저장하지 않는다.
+- public response는 `Cache-Control: no-store`, `Referrer-Policy: no-referrer`와
+  `script-src 'none'`을 사용하고 `Set-Cookie`를 만들지 않는다.
+- invalid raw input, 내부 source code, credential과 전체 query를 response·error·log에
+  반사하지 않는다.
+
+선택 목록은 internal series UUID를 URL 순서대로 최대 5개까지 받는다. 중복은 첫 항목만
+유지하고 malformed 또는 5개 초과는 고정 문구의 400이다. active publication에서 사라진
+UUID는 원문을 노출하지 않고 제외된 수만 알리는 200 partial state로 처리한다. 목록은 각
+품목의 자체 변화만 보여주며 합계·절약액·품목 간 가격순을 만들지 않는다.
+
+## 기술·publication 경계
+
+- Django SSR modular monolith, PostgreSQL, no-JavaScript 공개 화면을 유지한다.
+- 공개 request는 외부 source, candidate, raw artifact와 운영 control plane을 호출하지 않는다.
+- 기존 `RECENT_RETAIL`은 보존하고 세 역사 source의 승인 collection을 하나의 별도
+  `HISTORICAL_RETAIL` bundle로 봉인한다.
+- 세 source 중 하나가 실패·검토 대기이면 새 bundle을 만들지 않고 historical
+  last-known-good를 유지한다.
+- recent와 historical freshness·fact-set hash·rollback은 서로 합성하지 않고 독립적으로
+  표시·운영한다.
+- 월별, 지역별, 시장별 fact는 별도 typed model을 사용하며 범용 EAV나 범용 ingestion
+  framework를 만들지 않는다.
+- raw source body는 process memory 밖에 보존하지 않고 redacted receipt와 content hash만
+  감사 경계에 남긴다.
+
+## 고정 비목표와 사람 checkpoint
+
+CSV export, 지도, 시장유형 taxonomy, 공개 JSON API, native app, SPA, Redis, Celery, queue,
+search engine과 새 외부 source는 이번 버전의 비목표다.
+
+API key 발급·입력, source 권리 판정, code manifest 승인, 첫 historical review·seal·activation,
+production database migration, deployment와 traffic switch는 사람 전용 checkpoint다. 이
+구현 요청은 개발계정 live source gate를 승인하지만 production 활성화나 배포를 승인하지
+않는다.
+
+## frontend 기준
+
+외부 기준 문서
+`/Users/woopinbell/Desktop/content-foundry-worktree/production-grade-frontend-design-rule.md`
+전체를 적용하며 승인 시 SHA-256은
+`ce467f732623722f657155275c40c0667f9819b8d0ad088b27a768bb784fd69b`이다.
+
+초록장부의 따뜻한 종이·장부 visual language를 발전시키되 generic SaaS card grid,
+gradient, glassmorphism, 장식용 blob, 무의미한 badge와 과도한 rounded rectangle을 사용하지
+않는다. Product, Copy, Brand, UI, Frontend Engineering 관점의 첫 렌더 리뷰를 통과한 뒤에만
+browser acceptance를 고정한다.


## `docs: fix the vnext public read contract`

diff --git a/docs/VNEXT-PUBLIC-READ-CONTRACT.md b/docs/VNEXT-PUBLIC-READ-CONTRACT.md
new file mode 100644
index 0000000..90e96e0
--- /dev/null
+++ b/docs/VNEXT-PUBLIC-READ-CONTRACT.md
@@ -0,0 +1,131 @@
+# vNext public-read 계약
+
+이 문서는 source·review·publication과 공개 Django SSR 사이의 유일한 vNext read 계약이다.
+공개 view와 template은 이 계약을 벗어나 ORM 산술, candidate 조회 또는 source 호출을 하지
+않는다.
+
+## 공개 경로
+
+| 경로 | 목적 | 읽는 publication |
+|---|---|---|
+| `/` | 품목 탐색과 최근 비교 필터 | `RECENT_RETAIL` |
+| `/series/<uuid>/` | 최근 조사값과 1주·1개월·1년 비교 | `RECENT_RETAIL`; historical 가용성만 별도 확인 |
+| `/series/<uuid>/history/` | 선택 지역의 월별 기록 | `HISTORICAL_RETAIL` |
+| `/series/<uuid>/regions/` | 한 조사일의 지역별 범위 | `HISTORICAL_RETAIL` |
+| `/series/<uuid>/regions/<uuid>/markets/` | 선택 지역·조사일의 시장별 관측 | `HISTORICAL_RETAIL` |
+| `/selection/` | URL 순서의 최대 다섯 품목 최근 비교 | `RECENT_RETAIL` |
+
+모든 경로는 GET과 HEAD만 허용한다. 공개 request는 source client, raw artifact, candidate
+collection, review row와 운영 command를 호출하지 않는다. UUID가 current recent publication에
+없으면 404이며 candidate 존재 여부를 드러내지 않는다.
+
+## GET state
+
+catalog는 다음 값만 허용한다.
+
+- `q`: Unicode control·line break가 없는 공식 품목명 부분 문자열, trim 후 최대 80자
+- `category`: 빈 값, `vegetable`, `fruit`
+- `period`: `week`, `month`, `year`; 기본 `week`
+- `direction`: `all`, `lower`, `equal`, `higher`, `unavailable`; 기본 `all`
+- `sort`: `name`, `change_asc`, `change_desc`; 기본 `name`
+- `page`: canonical decimal integer 1–100; 기본 1
+
+한 page는 30개다. change sort는 available signed percentage를 정렬하고 unavailable을 항상
+뒤에 둔다. 동률은 category, 공식 품목명과 exact series identity 순서로 고정한다. period,
+direction과 sort는 sealed recent reference·change fact만 사용하며 view나 template에서 다시
+계산하지 않는다.
+
+history는 `range=12|36|60`과 `region=<uuid>`만 받는다. 기본 range는 36이다. 12개월은 36개월
+completeness를 통과한 series·region에서만, 60개월은 완전한 60개월이 있을 때만 선택지로
+노출한다. 공식 aggregate 지역이 없는 series는 region 선택 전 안내 상태를 표시하고 chart를
+만들지 않는다.
+
+regions는 `date=YYYY-MM-DD`만 받는다. markets는 같은 `date`와 `page=1..100`을 받는다. date를
+생략하면 active bundle 안에서 해당 series의 regional·market source가 공유하는 최신 조사일을
+사용한다. 선택 가능한 날짜는 bundle 확인 시각 기준 최근 31 calendar day 안의 실제 공통
+조사일뿐이다. 시장 목록은 공식 이름 순서, 30개 page와 stable market identity tie-break를
+사용하며 가격순·시장유형 filter를 제공하지 않는다.
+
+selection은 반복 `series=<uuid>`만 받는다. URL 순서를 보존하고 중복은 첫 항목만 유지한다.
+malformed UUID 또는 중복 제거 전후 어느 쪽이든 5개 초과면 고정 문구 400이다. current recent
+publication에서 사라진 valid UUID는 값 자체를 반사하지 않고 제외 수만 알리는 200 partial
+state다. 합계, 절약액, 서로 다른 단위의 정렬과 품목 간 우열은 계산하지 않는다.
+
+알 수 없는 parameter, 중복이 허용되지 않은 parameter, 비canonical page·date·UUID와 범위 밖
+값은 400이다. 정상화된 유효값만 form과 link에 다시 표시한다. invalid raw value와 전체 query는
+response, log, metric, audit와 artifact에 남기지 않는다.
+
+## active publication 결합
+
+`RECENT_RETAIL`과 `HISTORICAL_RETAIL`은 각각 sealed current pointer와 independent
+last-known-good를 갖는다. 두 revision을 합쳐 새 freshness나 hash를 만들지 않는다.
+
+historical fact는 `series_identity_sha256`로 recent exact series와 연결한다. 품목·품종·등급,
+원문 단위·단위크기와 retail category code가 모두 일치해야 한다. 이름 유사도, 단위환산과
+fallback join은 금지한다. historical bundle 안에 대응 series가 없으면 recent detail은 그대로
+작동하고 확장 link를 숨긴다. 직접 요청한 확장 화면은 공개 recent series라면 200 unavailable,
+공개 recent series가 아니면 404다.
+
+catalog·detail·selection은 기존 `X-Publication-Fact-Set-SHA256`를 유지한다. historical 화면은
+`X-Historical-Fact-Set-SHA256`를 제공한다. 두 publication을 실제로 읽은 response만 두 header를
+모두 제공한다. header 값은 sealed revision의 검증된 lowercase SHA-256 literal이다.
+
+## presentation context
+
+public-read layer는 format과 validation을 끝낸 다음 template-safe primitive만 반환한다.
+
+- catalog item: exact identity, current price, source date, 선택 period comparison, detail URL
+- detail: 기존 recent series·comparisons·provenance와 `historical_links`
+- history: series identity, selected region/range, available ranges, chronological monthly points,
+  provider mean·min·max와 gap flag
+- regions: series identity, selected date, selectable dates와 공식 이름 순 regional mean·min·max
+- markets: series·region identity, selected date, selectable dates, paginated market name·price
+- selection: URL 순서의 recent item facts, excluded count와 add/remove canonical URLs
+
+가격은 검증된 Decimal에서 원화 표시 문자열과 `<data>`용 finite decimal string으로 만든다.
+날짜는 ISO machine value와 한국어 display를 함께 만든다. chart geometry는 server-side Decimal
+계산 결과만 숫자 SVG attribute로 전달하며 inline style, data-driven CSS와 template 산술은
+금지한다.
+
+월별 chart는 provider 평균 line과 최저–최고 범위만 표현한다. 결측 구간을 선으로 잇거나 0으로
+채우지 않는다. 지역 화면은 공식 이름순 range/dot ledger다. 시장 화면은 ruled list다. 어떤
+표면도 추세, 제철, 예측, 저렴함, 추천 또는 시장유형을 추론하지 않는다.
+
+## 상태와 HTTP 계약
+
+| 상태 | HTTP | 계약 |
+|---|---:|---|
+| ready | 200 | current sealed fact만 표시 |
+| empty | 200 | 유효 filter 결과 없음; controls와 전체 보기 제공 |
+| unavailable | 200 | current publication 또는 exact historical facts 없음; 작동하지 않는 controls 제거 |
+| stale | 200 | last-known-good 사실과 publication별 경고·확인 시각 표시 |
+| validation | 400 | 고정 오류 요약과 유효 복구 link; raw input 비반사 |
+| not found | 404 | current recent series가 아님; candidate 존재 비공개 |
+| server error | 503 | DB·shape·integrity 실패; 고정 문구와 안전한 retry link |
+
+loading은 no-JavaScript SSR runtime 상태가 아니다. DEBUG에서만 제공되는 deterministic browser
+acceptance fixture가 loading copy·layout을 검증하며 production 공개 경로는 준비된 response를
+한 번에 반환한다. malformed active fact, incomplete range, duplicate identity와 hash 불일치는
+부분 표시하지 않고 해당 surface 전체를 503으로 실패 폐쇄한다.
+
+recent stale과 historical stale은 각각 자신의 상태 문구와 확인 시각을 사용한다. 한쪽 stale을
+다른 쪽에 전파하지 않는다. 새 candidate 수집 실패는 active pointer를 바꾸지 않으므로 기존
+last-known-good를 계속 표시한다.
+
+## 보안·접근성 계약
+
+모든 public response는 `Cache-Control: no-store`, `Referrer-Policy: no-referrer`,
+`X-Content-Type-Options: nosniff`와 `script-src 'none'` CSP를 유지하고 `Set-Cookie`를 만들지
+않는다. 외부 font, image, script, analytics와 browser source request는 없다. outbound KAMIS
+attribution link는 `rel="external noreferrer"`를 사용한다.
+
+페이지에는 `main`과 h1이 각각 하나이며 skip link, semantic heading/list/table 또는 definition
+list, visible keyboard focus와 44px target을 유지한다. server SVG는 정확한 HTML 값의 보조
+표현이고 `aria-hidden`이다. 360px에서 horizontal overflow와 긴 한글 절단이 없어야 한다.
+
+## copy revision과 운영 checkpoint
+
+vNext 공개 문구는 `ko-v4`로만 새 historical revision에 봉인한다. `ko-v1`–`ko-v3` row는
+수정하지 않는다. disposable local DB에서만 fixture bundle과 browser evidence를 만들며,
+production migration, first collection, code manifest 승인, review, seal, activation, traffic
+switch와 rollback은 사람 checkpoint다.


## `docs: activate the historical system boundary`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index ce4cfd1..06afbc2 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -1,5 +1,9 @@
 # 첫 구현 계획
 
+> 이 문서는 Phase 0 A path의 구현·증거 기록이다. 2026-08-31 이후 historical consumer
+> 확장은 `VNEXT-PRODUCT-CONTRACT.md`, `VNEXT-SOURCE-GATE.md`와
+> `VNEXT-PUBLIC-READ-CONTRACT.md`를 따른다.
+
 ## 결정
 
 선택 path는 **A — 최근 비교 MVP**다. 공개 표현은 `KAMIS 소매 조사 평균`과
diff --git a/docs/MVP-ACCEPTANCE.md b/docs/MVP-ACCEPTANCE.md
index 3d01a95..566f1df 100644
--- a/docs/MVP-ACCEPTANCE.md
+++ b/docs/MVP-ACCEPTANCE.md
@@ -181,10 +181,14 @@ snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표
 CLI 검증과 분리된 **production 사람 checkpoint**다. platform packaging 방식이 선택된 뒤 실제
 artifact를 검사하기 전에는 이 항목을 통과로 기록하지 않는다.
 
-## 월별 과거 패턴 module gate — 해당 없음(N/A)
+## 월별 과거 패턴 module gate — Phase 0 역사 기록
 
-이번 구현은 A path이므로 이 module은 별도 제품결정 미승인·비활성이다. 아래 항목은 이번
-acceptance에서 평가하지 않는다.
+아래 N/A 판정은 첫 A path Phase 0 acceptance 당시의 역사 기록이다. 2026-08-31 승인된
+vNext의 월별·지역별·시장별 확장은 `VNEXT-PRODUCT-CONTRACT.md`, 실제 interface gate는
+`VNEXT-SOURCE-GATE.md`, 공개 상태·route 인수 기준은 `VNEXT-PUBLIC-READ-CONTRACT.md`가
+대체해 소유한다. 전체 code manifest와 production activation은 아직 통과로 표시하지 않는다.
+
+Phase 0 당시에는 이 module을 평가하지 않았다.
 
 내부 repository 이름만으로 이 module을 활성화하지 않는다. 별도 제품 결정을 승인하기 전에
 다음을 모두 증명한다.
diff --git a/docs/SYSTEM-BOUNDARIES.md b/docs/SYSTEM-BOUNDARIES.md
index 13d53bf..a72f490 100644
--- a/docs/SYSTEM-BOUNDARIES.md
+++ b/docs/SYSTEM-BOUNDARIES.md
@@ -42,11 +42,18 @@ gate로 검증합니다. file path가 통과해도 이를 현재가격, 실시
 채우는 자료로 사용하지 않습니다. 별도 typed monthly model, publication channel, route와
 rollback을 사용합니다.
 
-### 3. 후속 API `15156060`과 `15156065`
+### 3. vNext historical API `15156060`, `15156062`, `15156065`
 
-연월별·기간별 소매가격은 다년 월별 패턴을 위한 비활성 source 후보입니다. 첫
-MVP schema, ingestion schedule, 공개 UI와 acceptance에 포함하지 않습니다. 별도
-gate와 제품 결정 없이 이 데이터를 가져오거나 기존 profile에 섞지 않습니다.
+2026-08-31 승인된 vNext는 연월별, 지역별, 기간별 소매가격을 각각 월별 provider 범위,
+지역별 일 범위, 시장별 관측의 독립 source로 사용합니다. 최소 live interface gate와 실제
+JSON wrapper 증거는 `VNEXT-SOURCE-GATE.md`, 역할·결합 금지는
+`VNEXT-PRODUCT-CONTRACT.md`가 소유합니다.
+
+세 source는 별도 collection, typed fact, review와 `HISTORICAL_RETAIL` publication을
+사용합니다. daily·market 행으로 monthly 값을 만들거나 market 행으로 regional 평균을
+재구성하지 않습니다. 공개 request는 이 API를 호출하지 않고 active sealed historical
+publication만 읽습니다. 전체 code manifest 승인과 첫 production collection·seal·activation은
+계속 사람 checkpoint입니다.
 
 ### 제외된 외부 경계
 
@@ -89,12 +96,14 @@ gate와 제품 결정 없이 이 데이터를 가져오거나 기존 profile에
 
 ### public read 경계
 
-- Django form은 부류와 공식 품목명 검색만 받습니다.
+- Django form은 공식 품목명, 부류, 비교기간·방향·정렬·page와 승인된 history range,
+  region, date, 최대 다섯 internal series UUID만 bounded GET state로 받습니다.
 - 검색 길이, 문자와 result 수를 제한하며 검색어를 공개 방문자 session, cache, analytics,
   log와 audit에 남기지 않습니다. Django Admin의 보안 인증 session은 별도 운영 정책을
   따릅니다.
-- 목록·상세는 published read model만 조회하고 외부 source, 운영 candidate와 raw
-  artifact에 접근하지 않습니다.
+- 목록·상세·선택 목록은 active recent publication, 월별·지역별·시장별 화면은 active
+  historical publication만 조회하고 외부 source, 운영 candidate와 raw artifact에 접근하지
+  않습니다.
 - 공개 URL에는 stable internal slug만 사용하며 source key나 secret query를 넣지 않습니다.
 
 ## 데이터 흐름
@@ -106,11 +115,11 @@ platform cron
   -> FetchAttempt + redacted receipt
   -> content-addressed SourceArtifact
   -> versioned ParseRun + reconciliation
-  -> typed recent retail facts or typed monthly retail snapshots
+  -> typed recent retail facts or typed historical monthly/regional/market facts
   -> ReviewDecision
-  -> immutable PublicationRevision
-  -> append-only PublicationActivation + atomic channel pointer
-  -> Django server-rendered list/detail
+  -> immutable recent or historical publication revision
+  -> channel별 append-only activation + atomic pointer
+  -> Django server-rendered list/detail/history/region/market/selection
 ```
 
 한 단계의 성공을 다음 단계의 성공으로 간주하지 않습니다. HTTP 200은 artifact 승인,
diff --git a/docs/TECHNOLOGY-DECISIONS.md b/docs/TECHNOLOGY-DECISIONS.md
index 79e7761..55556c6 100644
--- a/docs/TECHNOLOGY-DECISIONS.md
+++ b/docs/TECHNOLOGY-DECISIONS.md
@@ -2,9 +2,10 @@
 
 ## 문서 상태
 
-이 문서는 첫 구현이 따라야 할 기술 기준선이다. 현재 저장소에는 런타임 코드, 의존성,
-잠금 파일, 수집 데이터와 배포 구성이 없다. 아래 버전과 구조는 source gate를 통과한 뒤
-구현 계획에서 실제 호환성을 다시 증명하고 도입한다.
+이 문서는 첫 구현이 따랐던 기술 기준선이며 저장소에는 현재 Django runtime, 잠금 파일,
+PostgreSQL migration과 Phase 0 공개 경계가 구현돼 있다. vNext도 같은 고정 stack과 modular
+monolith를 유지하며 신규 계약은 `VNEXT-PRODUCT-CONTRACT.md`와
+`VNEXT-PUBLIC-READ-CONTRACT.md`가 추가한다.
 
 ## 고정 기준 스택
 


