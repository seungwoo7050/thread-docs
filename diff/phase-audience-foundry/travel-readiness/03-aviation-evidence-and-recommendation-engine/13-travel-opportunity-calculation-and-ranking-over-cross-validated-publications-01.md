# 교차 검증된 게시 증거 기반 여행 기회 계산과 순위화

## `docs: define holiday flight recommendation contract`

diff --git a/README.md b/README.md
index 0feef32..fe51b3c 100644
--- a/README.md
+++ b/README.md
@@ -1,8 +1,8 @@
-# Audience Foundry Travel Readiness
+# 어디 갈까 ??
 
-한국 일반여권 여행자가 목적지와 여행일을 입력하면 검증된 공식 입국요건 사실과
-여행경보를 출처·확인시각과 함께 보여주고, 별도 검증을 통과한 경우에만 인천발
-직항 왕복 가능 시간 정보를 더하는 서비스의 구현 기준선입니다.
+한국 일반여권 여행자가 출발 가능한 시각과 인천 도착 마감 시각을 입력하면,
+검수·게시된 직항 운항 예정편을 바탕으로 현지 공항에서 40시간 이상 머물 수 있는
+여행지와 독립된 입국요건·여행경보를 보여주는 서비스입니다.
 
 ## 저장소 상태
 
@@ -43,10 +43,11 @@ microservice는 첫 구현에 포함하지 않습니다.
 - `docs/MVP-ACCEPTANCE.md`
 - `docs/IMPLEMENTATION-PLAN.md`
 - `docs/FRONTEND-HANDOFF.md`
+- `docs/TRAVEL-OPPORTUNITY-CONTRACT.md`
 - `docs/OPERATIONS-RUNBOOK.md`
 - `docs/COMPLETION-REPORT.md` (최종 local candidate handoff template)
 
-입국요건 사실과 여행경보는 독립적으로 게시합니다. 항공편 모듈은 실제 source가
-출발·도착 현지시각, timezone/day offset, 유효일과 방향을 모두 증명하기 전에는
-비활성 상태입니다. 이 저장소는 법률자문이나 입국 승인, 실제 운항·좌석·예약을
+입국요건, 여행경보와 항공편은 독립적으로 게시합니다. 항공편 source의 ICN event
+time과 검수된 노선별 비행시간으로 현지 event time을 예상하며 공식 사실과 계산값을
+구분합니다. 이 저장소는 법률자문이나 입국 승인, 실제 운항·좌석·가격·예약을
 보장하지 않습니다.
diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
index b6280b0..d5fd281 100644
--- a/docs/COMPLETION-REPORT.md
+++ b/docs/COMPLETION-REPORT.md
@@ -1,5 +1,8 @@
 # Phase 0 배포 직전 local candidate 보고서
 
+> 이 보고서는 일본 브리핑 기준선의 기록이며 새 제품 완료 증거가 아닙니다.
+> `어디 갈까 ??` 통합 뒤 새 evidence로 갱신합니다.
+
 이 보고서는 production 배포 완료 보고서가 아닙니다. 이 파일을 포함하는 clean `main` commit을
 release candidate로 삼고, 그 commit에서 모든 gate를 다시 실행한 exact SHA와 고정 receipt는
 최종 handoff에 기록합니다. tracked 보고서가 자기 자신의 SHA를 포함할 수 없으므로 이 파일 안에
diff --git a/docs/DATA-AND-AUDIT-MODEL.md b/docs/DATA-AND-AUDIT-MODEL.md
index 97d254d..5c00882 100644
--- a/docs/DATA-AND-AUDIT-MODEL.md
+++ b/docs/DATA-AND-AUDIT-MODEL.md
@@ -44,18 +44,19 @@ ParseRun과 typed snapshot은 중복되지 않습니다. timeout attempt는 cont
 EntryFactRevision의 status는 source statement를 표현하며 `ALLOWED`/`DENIED` enum을 갖지
 않습니다. 숫자 체류기간은 source가 단위와 포함일 계산을 명확히 제공할 때만 저장합니다.
 
-## 조건부 항공 entity
+## 항공 entity
 
-항공 gate 승인 전에는 다음 table migration을 적용하지 않습니다.
+- `Airport`: 내부 UUID, IATA 후보키, 국가, 도시, 승인된 IANA timezone과 mapping evidence
+- `FlightScheduleRevision`: source artifact, season, coverage, completeness, state와 parser revision
+- `FlightSchedule`: revision, direction, carrier/flight와 master flight, 목적지 airport,
+  ICN local event time, validity range와 weekday mask
+- `RouteDurationRevision`: 목적지 airport, 양방향 예상 duration, reference locator/date,
+  validation state와 review identity
+- `FlightPublication`: 승인된 schedule revision과 duration set의 불변 generation
+- `PublishedFlightSchedule`: current published flight generation pointer
 
-- `Airport`: 내부 UUID, IATA/ICAO 후보키, 국가, 좌표, 승인된 IANA timezone과 mapping evidence
-- `ScheduleSnapshot`: source artifact, season, coverage, completeness, state와 activation decision
-- `FlightSchedule`: snapshot, direction, carrier/flight, origin/destination, departure local time,
-  arrival local time, arrival day offset, 각 timezone, validity range와 weekday mask
-- `TravelOpportunity`: publication, normalized ICN window, outbound/inbound schedule과 계산 revision의 재생성 가능한 projection
-
-현지 도착·출발 필드가 없는 pivot에서는 `TravelOpportunity` 대신 검증된 ICN 출발과
-귀착 event만 사용하고 `Local Stay Time` field를 만들지 않습니다.
+`TravelOpportunity`는 사용자의 normalized ICN window와 current publication으로 계산하는
+request-only projection이며 table로 저장하지 않습니다.
 
 ## 상태·정합성·동시성
 
@@ -72,7 +73,8 @@ fetch는 DB transaction 밖에서 수행하고 response evidence를 먼저 durab
 
 ## 개인정보·보존·복구
 
-목적지와 날짜는 request scope이며 DB, session, cache, audit, access/error log, APM,
+출발 가능 시각, 인천 도착 마감 시각과 계산 결과는 request scope이며 DB, session,
+cache, audit, access/error log, APM,
 trace, metric label 또는 backup에 원문으로 남기지 않습니다. 보안 운영상 필요한
 access log는 query-less route·status·latency만 기록하고 IP 원문을 저장하지 않으며
 30일 이내 삭제합니다.
diff --git a/docs/DOMAIN-BRIEF.md b/docs/DOMAIN-BRIEF.md
index 85172ac..ed329e3 100644
--- a/docs/DOMAIN-BRIEF.md
+++ b/docs/DOMAIN-BRIEF.md
@@ -2,19 +2,18 @@
 
 ## 제품 정체성
 
-- 작업명: `Travel Readiness KR` (`여행준비`).
-- 한 문장 제품: 한국 일반여권 여행자가 목적지와 여행일을 입력하면 검증된 공식
-  입국요건 사실과 여행경보를 출처·확인시각과 함께 확인하고, 별도 항공 source gate를
-  통과한 경우에만 인천발 직항 왕복 가능 시간 정보를 함께 보는 공개 웹서비스입니다.
+- 제품명: `어디 갈까 ??`.
+- 한 문장 제품: 한국 일반여권 여행자가 출발 가능한 시각과 인천 도착 마감 시각을
+  입력하면 검수·게시된 직항 운항 예정편으로 예상 현지 체류시간 40시간 이상인
+  목적지와 독립된 입국요건·여행경보를 확인하는 공개 웹서비스입니다.
 - 최종 결정권자: 프로젝트 소유자는 source 권리, 공개 범위, production 배포와
   파괴적 변경을 승인합니다.
 - 저장소 기준선: local `main`, remote 없음, baseline SHA `0cc95e7`입니다.
 
 ## 첫 고객과 문제
 
-첫 고객은 대한민국 일반여권으로 단기 관광을 준비하며 목적지의 입국요건과
-여행경보를 확인하려는 한국 거주자입니다. 항공편 모듈이 활성화된 뒤의 첫 고객은
-인천에서 출발해 주말 또는 0~2일 연차로 직항 단기여행을 검토하는 사람입니다.
+첫 고객은 대한민국 일반여권으로 인천에서 출발해 주말 또는 0~2일 연차로 직항
+단기여행을 검토하는 한국 거주자입니다.
 
 현재 사용자는 외교부, 목적국 공식 사이트, 달력과 항공 검색을 오가며 출처의 범위와
 확인시각을 스스로 대조해야 합니다. 이 제품은 입국 가능 여부를 대신 결정하지 않고,
@@ -30,16 +29,17 @@
    사실만 typed revision 후보로 만들고, 모호한 내용은 검수 대상으로 보냅니다.
 4. 운영자가 후보를 승인하면 입국요건 사실과 여행경보가 서로 독립된 publication
    pointer와 audit를 통해 게시됩니다.
-5. 사용자는 목적지와 입출국일을 입력합니다. 입력은 요청 동안만 사용하며 개인
-   여행기록으로 저장하지 않습니다.
-6. 서버 렌더링 페이지는 source 사실, 관측·마지막 성공 확인시각, 적용 범위와
-   `확인 필요`를 표시합니다. `ALLOWED`, 법적 판단 또는 입국 보장 표현은 사용하지 않습니다.
+5. 사용자는 출발 가능한 시각과 인천 도착 마감 시각을 입력합니다. 입력은 요청
+   동안만 사용하며 개인 여행기록이나 계산 결과로 저장하지 않습니다.
+6. 서버 렌더링 페이지는 추천 운항 예정편, 예상 현지 체류시간, source 사실,
+   마지막 성공 확인시각, 적용 범위와 `확인 필요`를 표시합니다. `ALLOWED`, 법적 판단,
+   실제 운항·좌석 또는 입국 보장 표현은 사용하지 않습니다.
 7. 동일 content를 다시 확인하면 새 FetchAttempt로 freshness는 갱신되지만 같은
    SourceArtifact와 domain revision은 중복 생성되지 않습니다.
 
-항공편 모듈은 이 루프의 선행조건이 아닙니다. 실제 schedule source가 origin과
-destination의 local time, timezone, 날짜 offset, 방향, 유효 시작·종료일과 운항요일을
-모두 제공하는 것이 증명된 뒤 별도 migration과 publication으로 추가합니다.
+항공편, 입국요건과 여행경보는 각각 독립된 publication입니다. 항공편은 검수된 ICN
+event time과 노선별 duration을 결합해 예상 현지 event를 계산하며 상세 gate는
+`docs/TRAVEL-OPPORTUNITY-CONTRACT.md`를 따릅니다.
 
 ## 성공과 허용할 수 없는 실패
 
@@ -55,13 +55,14 @@ destination의 local time, timezone, 날짜 offset, 방향, 유효 시작·종
 - 여행경보를 입국허가 판단으로 사용하는 일
 - 새 fetch 실패가 기존 publication을 지우는 일
 - 서로 다른 모듈 revision을 하나의 원자적 사실인 것처럼 묶는 일
-- 항공 source gate 전에 현지 체류시간이나 실제 여행 가능성을 공개하는 일
-- 목적지·날짜, IP 또는 cookie를 개인 profile로 영구 저장하는 일
+- 검수되지 않은 duration을 쓰거나 예상값을 source의 실제 현지 event로 표현하는 일
+- 검색 시각, 계산 결과, IP 또는 cookie를 개인 profile로 영구 저장하는 일
 
 ## 범위와 비목표
 
-첫 production 범위는 대한민국 일반여권·단기 관광, 목적국과 날짜 입력, 검증된
-입국요건 사실, 여행경보, source link/revision/freshness, 운영자 검수와 변경 이력입니다.
+첫 production 범위는 대한민국 일반여권·단기 관광, 최대 7일의 ICN 여행 가능 시간,
+12개 후보 도시의 직항 운항 예정편, 예상 현지 체류시간, 검증된 입국요건 사실,
+여행경보, source link/revision/freshness, 운영자 검수와 변경 이력입니다.
 
 비목표는 비자·ETA 신청, 법률자문, 항공사 탑승·입국 승인 보장, 다른 국적·여권,
 장기체류·취업·유학, 사용자 계정·일정 저장·알림, 결제·예약·가격·재고, 경유편,
@@ -76,4 +77,5 @@ destination의 local time, timezone, 날짜 offset, 방향, 유효 시작·종
 - `SourceArtifact`: stable source ID와 body SHA-256으로 식별되는 불변 content입니다.
 - `Domain Revision`: SourceArtifact를 고정 parser revision으로 해석한 typed 후보입니다.
 - `Publication`: 한 모듈에서 공개가 승인된 domain revision pointer입니다.
-- `Travel Window`: 항공 모듈 승인 후 사용하는 ICN 출발 가능시각부터 귀착 마감시각까지의 구간입니다.
+- `Travel Window`: ICN 출발 가능시각부터 귀착 마감시각까지 최대 7일의 구간입니다.
+- `Travel Opportunity`: current schedule과 검수된 duration으로 계산한 request-only 추천입니다.
diff --git a/docs/FRONTEND-HANDOFF.md b/docs/FRONTEND-HANDOFF.md
index 13ee1b1..d3f5668 100644
--- a/docs/FRONTEND-HANDOFF.md
+++ b/docs/FRONTEND-HANDOFF.md
@@ -1,5 +1,9 @@
 # Backend → frontend handoff
 
+> 이 문서는 `58eb91930e0d0b0683ca00d48e37a7d89124cb19`까지의 일본 브리핑 frontend
+> handoff 기록입니다. 새 제품 구현은 `docs/TRAVEL-OPPORTUNITY-CONTRACT.md`를 우선하며
+> 이 문서의 항공 비활성·303 계약을 계승하지 않습니다.
+
 ## 이 경계의 상태
 
 이 문서를 포함하는 clean `main` commit은 출시 완료나 production candidate가 아니라
diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index c87c37a..e3d882e 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -1,5 +1,8 @@
 # 구현 계획
 
+> 이 문서의 기존 본문은 일본 브리핑 기준선의 구현 기록입니다. 2026-08-31 이후
+> `어디 갈까 ??` 구현은 `docs/TRAVEL-OPPORTUNITY-CONTRACT.md`와 후속 commit을 따릅니다.
+
 ## checkpoint와 범위
 
 - 시작 checkpoint: local `main`의 `ea6def259272298770fa3158d5306bb30b0a89c1`
diff --git a/docs/MVP-ACCEPTANCE.md b/docs/MVP-ACCEPTANCE.md
index 973aedd..ceff98f 100644
--- a/docs/MVP-ACCEPTANCE.md
+++ b/docs/MVP-ACCEPTANCE.md
@@ -1,5 +1,9 @@
 # Phase 0 배포 직전 production candidate 인수 기준
 
+> 2026-08-31 제품 전환 이후 항공 추천과 공개 요청 계약은
+> `docs/TRAVEL-OPPORTUNITY-CONTRACT.md`가 이 문서의 기존 일본 브리핑 전용 조건보다
+> 우선합니다. 이 문서는 통합 커밋에서 새 제품의 절제된 acceptance로 갱신합니다.
+
 이 문서는 실제 production 배포를 제외한 로컬 release candidate의 완료 조건입니다. 통과 상태는
 `Phase 0 배포 직전 완료`이며 `Phase 0 완료` 또는 production 운영 중이라고 표현하지 않습니다.
 `docs/PRODUCT-DECISIONS.md`의 실제 배포 `Stage 0`은 이 candidate 뒤의 별도 deployment gate입니다.
diff --git a/docs/PRODUCT-DECISIONS.md b/docs/PRODUCT-DECISIONS.md
index 8dc8739..b21befe 100644
--- a/docs/PRODUCT-DECISIONS.md
+++ b/docs/PRODUCT-DECISIONS.md
@@ -17,7 +17,8 @@ rollback과 재검증 결과가 있어야 합니다.
 2. 공개 결과는 `ALLOWED`, `DENIED`, 법적 자문 또는 입국 보장으로 표현하지 않습니다.
 3. source가 직접 뒷받침하는 typed Entry Fact만 게시하며 불충분하면 `확인 필요`입니다.
 4. 입국요건과 여행경보는 모델, 검수, publication과 장애 경계가 독립적입니다.
-5. 항공편 모듈은 별도 source contract gate 전까지 disabled이며 다른 모듈의 완료를 막지 않습니다.
+5. 항공편 모듈은 `docs/TRAVEL-OPPORTUNITY-CONTRACT.md`의 source gate를 통과한
+   publication만 공개하며 다른 모듈의 완료나 pointer를 막지 않습니다.
 6. 공개 사실에는 source owner, locator, contract revision 또는 fingerprint, content hash,
    마지막 성공 확인시각과 publication identity를 추적할 수 있어야 합니다.
 7. 모든 외부 조회는 `FetchAttempt`를 새로 만들지만 content identity에는 `fetched_at`,
@@ -29,7 +30,8 @@ rollback과 재검증 결과가 있어야 합니다.
 10. 외부 실패, schema 변경과 parser 실패는 last-known-good publication을 바꾸지 않습니다.
 11. raw body는 source 권리와 retention 승인이 있을 때만 암호화된 내부 저장소에 보관합니다.
     승인 전에는 hash, redacted receipt와 필요한 최소 typed field만 보관합니다.
-12. 사용자의 목적지·입출국일은 request scope에서만 처리하며 개인별 history로 저장하지 않습니다.
+12. 사용자의 출발 가능 시각과 인천 도착 마감 시각은 request scope에서만 처리하며
+    개인별 history나 계산 결과로 저장하지 않습니다.
 13. 공개 request path에서 외부 source를 동기 호출하지 않습니다.
 14. publication pointer 변경, 승인 결정과 성공 audit는 한 PostgreSQL transaction입니다.
 15. module A의 실패·미게시 상태가 module B의 pointer를 변경하거나 결과를 숨기지 않습니다.
@@ -98,11 +100,13 @@ receiver·담당자 provision은 production deployment의 사람 전용 checkpoi
 9. source outage, 권한, migration, backup/restore와 production security gate를 검증합니다.
 10. production 배포는 사람 승인을 기다리고, 항공은 별도 live gate 통과 후에만 결정·migration부터 추가합니다.
 
-## 항공편 stop/pivot
+## 항공편 활성화 결정
 
-실제 source가 origin/destination local time, 각 timezone, day offset, 방향, 유효일과
-요일을 모두 제공하지 않으면 `Local Stay Time` 개발을 중단합니다. 축소된
-`ICN departure-to-return window`도 outbound의 ICN 현지 출발 날짜·시각, inbound의
-ICN 현지 귀착 날짜·시각, 각 방향, 유효기간·요일, page completeness와 저장·재공개
-권리를 모두 실제 source로 증명한 경우에만 허용합니다. 하나라도 부족하면 항공
-모듈을 비활성으로 유지합니다.
+프로젝트 소유자는 2026-08-31에 정기운항편의 ICN event time과 별도로 검수된 노선별
+비행시간을 결합한 `예상 현지 체류시간`을 승인했습니다. source가 직접 제공한 현지
+event time으로 표현하지 않고 모든 계산값에 `예상` label과 계산 근거를 표시합니다.
+
+운항 방향, ICN event time, 시즌, 유효일, 요일, page completeness, 저장·재공개 권리와
+노선별 duration review 중 하나라도 부족하면 해당 도시를 추천하지 않습니다. 실제 운항,
+판매·좌석과 예약 가능성을 추론하지 않습니다. 상세 계약은
+`docs/TRAVEL-OPPORTUNITY-CONTRACT.md`가 우선합니다.
diff --git a/docs/SYSTEM-BOUNDARIES.md b/docs/SYSTEM-BOUNDARIES.md
index a598d65..cffb978 100644
--- a/docs/SYSTEM-BOUNDARIES.md
+++ b/docs/SYSTEM-BOUNDARIES.md
@@ -52,29 +52,23 @@ bounded context는 다음과 같습니다.
 
 ### 항공 정기편 source
 
-초기 상태는 `SourceConfiguration.state=DRAFT`, `enabled=false`입니다. 다음을 한
-실제 왕복 목적지에서 모두 증명해야 `RIGHTS_APPROVED`, `enabled=true`로 전환할
-수 있습니다.
-
-- ICN 출발편과 ICN 도착편의 명시적 방향
-- origin departure와 destination arrival의 현지시각 및 날짜 offset
-- destination departure와 ICN arrival의 현지시각 및 날짜 offset
-- 각 공항의 검증된 IANA timezone ID
-- 시즌, 유효 시작·종료일, 운항요일과 page completeness
-- 저장·재공개 권리와 정기편이지 실제 운항·판매 보장이 아니라는 표시
-
-필드가 부족하면 destination Local Stay Time을 만들지 않습니다. 축소된 ICN
-departure-to-return window도 outbound ICN 출발과 inbound ICN 귀착의 현지
-날짜·시각, 명시적 방향, 유효기간·요일, page completeness와 저장·재공개 권리를
-모두 실제 source로 검증해야 합니다. 하나라도 불완전하면 모듈을 disable합니다.
+항공 source는 ICN 출발편과 도착편의 방향, ICN event time, 시즌, 유효 시작·종료일,
+운항요일, 코드셰어와 page completeness를 제공합니다. 공항의 IANA timezone과 노선별
+양방향 비행시간은 별도 typed revision으로 검수·게시합니다.
+
+public web은 이 둘을 결합해 예상 현지 도착, 예상 귀국편 출발과 예상 현지 체류시간을
+계산합니다. 계산값은 source 사실과 분리해 표시하고 공항 이동·수속·시내 이동을 포함하지
+않음을 알립니다. source gate 하나라도 실패하면 해당 도시만 제외하며 entry와 warning
+publication은 유지합니다. 실제 운항, 판매·좌석 또는 예약 가능성을 추론하지 않습니다.
 
 ## 공개 HTML 경계
 
 공개 인터페이스는 Django `GET`과 form `POST`가 반환하는 server-rendered HTML입니다.
-입력은 목적국, 입국일과 출국일이며 CSRF-protected form validation을 거칩니다. 결과는
-각 module status, source, revision, last successful check, observation time, limitations와
-`확인 필요`를 표시합니다. 잘못된 날짜는 400 수준 form error, 미게시 국가는 명시적
-unavailable, 내부 장애는 일반화된 5xx로 처리합니다. 공개 JSON contract는 제공하지 않습니다.
+입력은 출발 가능한 시각과 인천 도착 마감 시각이며 CSRF-protected form validation을
+거칩니다. valid POST는 같은 200 response에서 transient result를 표시하며 검색 입력은
+URL이나 서버 저장소에 남기지 않습니다. 결과는 flight, entry와 warning의 독립 status,
+source, revision, last successful check, limitations와 `확인 필요`를 표시합니다. 공개 JSON
+contract는 제공하지 않습니다.
 
 ## 계정·운영·legacy 경계
 
diff --git a/docs/TECHNOLOGY-DECISIONS.md b/docs/TECHNOLOGY-DECISIONS.md
index 1caf6cc..28cf0d6 100644
--- a/docs/TECHNOLOGY-DECISIONS.md
+++ b/docs/TECHNOLOGY-DECISIONS.md
@@ -32,7 +32,7 @@ Kubernetes와 microservice를 추가하지 않습니다. 사용자 규모만으
 - `countries`: 승인된 내부 Country identity allowlist만 소유하며 source나 publication 상태는 소유하지 않음
 - `entry_requirements`: PassportPolicy, EntryFactRevision과 publication
 - `travel_warnings`: warning revision과 publication
-- `travel_windows`: source gate 통과 후 별도 migration으로 활성화
+- `travel_windows`: airport, schedule/duration revision, publication과 request-only opportunity 계산
 - `reviews`: Admin workflow, decisions와 audit
 - `public_web`: forms, views와 templates
 - `operations`: health, backup/restore check와 management commands
@@ -45,8 +45,9 @@ Country를 entry app 안에 두어 warning이 entry publication schema에 의존
 ## UI와 background work
 
 Django templates와 forms가 공개 UI의 source입니다. 필요한 progressive interaction은 작은
-server-rendered fragment 또는 vanilla JavaScript로 제한합니다. 모든 POST는 CSRF를 검증하고
-PRG 패턴을 사용합니다. 목적지·날짜 form은 저장 모델을 만들지 않습니다.
+server-rendered fragment 또는 vanilla JavaScript로 제한합니다. 모든 POST는 CSRF를 검증합니다.
+여행 가능 시간 form의 valid POST는 transient result를 같은 200 response에 렌더링하며 입력이나
+계산 결과의 저장 모델을 만들지 않습니다.
 
 수집은 module별 management command와 cron entry로 분리합니다. 한 module command 실패가
 다른 command의 exit status나 publication을 바꾸지 않습니다. network call은 finite connect/read
diff --git a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
new file mode 100644
index 0000000..d2c4631
--- /dev/null
+++ b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
@@ -0,0 +1,72 @@
+# 어디 갈까 ?? 여행 가능 시간 탐색 계약
+
+## 제품 목적
+
+`어디 갈까 ??`는 대한민국 일반여권 여행자가 짧은 휴일 안에 인천에서 출발해
+돌아올 수 있는 직항 목적지를 찾는 server-rendered web 제품입니다. 사용자가
+`출발 가능한 시각`과 `인천 도착 마감 시각`을 입력하면 검수·게시된 정기운항편과
+노선별 예상 비행시간을 조합해 현지 공항에서 40시간 이상 머물 수 있는 일정을
+추천합니다.
+
+항공권 가격, 좌석, 예약, 실제 운항과 입국 가능 여부는 제공하거나 보장하지 않습니다.
+항공편은 `운항 예정편`, 계산값은 `예상 현지 도착`, `예상 귀국편 출발`,
+`예상 현지 체류시간`으로만 표시합니다.
+
+## 공개 요청 계약
+
+- `GET /`: 빈 검색 form을 반환합니다.
+- invalid `POST /`: bound value와 오류를 같은 200 response에서 반환합니다.
+- valid `POST /`: 같은 200 response에서 검색 조건과 결과를 반환합니다.
+- `GET /results/`: query 없이 `/`로 redirect합니다.
+- `departure_at`과 `return_by`는 `datetime-local`이며 `Asia/Seoul`로 해석합니다.
+- 검색 범위는 미래 시각부터 최대 7일이고 `return_by > departure_at`이어야 합니다.
+- 사용자 입력과 계산 결과는 response scope 밖의 URL, DB, session, cookie, cache,
+  log, metric, trace, audit와 backup에 저장하지 않습니다.
+- 공개 request path는 외부 source를 호출하지 않고 current publication만 읽습니다.
+
+## 추천 계약
+
+- 첫 후보 도시는 도쿄, 오사카, 후쿠오카, 삿포로, 오키나와, 타이베이, 홍콩,
+  마카오, 하노이, 다낭, 호찌민, 방콕입니다.
+- 직항이며 current flight publication, 검수된 양방향 비행시간, entry publication과
+  warning publication이 모두 있는 도시만 추천합니다.
+- 코드셰어는 master flight를 기준으로 중복 제거합니다.
+- 예상 현지 도착은 ICN 출발 instant에 검수된 outbound duration을 더해 계산합니다.
+- 예상 귀국편 출발은 ICN 도착 instant에서 검수된 inbound duration을 빼 계산합니다.
+- 두 instant 차이가 40시간 이상인 조합만 남깁니다. 40시간 정각은 포함합니다.
+- 공항 이동, 입출국 수속과 시내 이동 시간은 차감하지 않습니다.
+- 도시는 최대 6개, 도시별 일정은 최대 2개입니다.
+- 정렬은 예상 체류시간 내림차순, 귀국 마감 여유 내림차순, 출발시각 내림차순,
+  도시 코드 오름차순입니다.
+
+## 표시와 장애 계약
+
+public result context는 `search_window`, `flight_state`, `opportunities`를 제공합니다.
+`flight_state`와 entry/warning state는 각각 `ready`, `empty`, `unavailable`, `stale`,
+`server-error` 중 하나이며 서로의 pointer나 표시 여부를 변경하지 않습니다.
+
+각 opportunity는 destination, estimated local stay, outbound schedule, inbound schedule,
+최대 두 개의 alternative, calculation basis, entry publication과 warning publication을
+가집니다. 공식 source 사실과 제품이 계산한 예상값은 label과 구조로 구분합니다.
+
+## source와 권리
+
+- 인천국제공항공사 정기운항편 source는 시즌·유효일·요일·방향·ICN event time을
+  current schedule 사실로 제공합니다.
+- 인천국제공항공사 취항도시 source는 공항 code와 국가·도시 명칭을 제공합니다.
+- 해양수산부 수출입 물류 플랫폼 항공 스케줄 파일의 비행시간은 검수된 duration
+  reference로만 사용하며 current 운항 사실로 표시하지 않습니다.
+- source rights, parser, typed revision, review, publication, atomic pointer, rollback과
+  freshness는 기존 공통 lifecycle을 통과합니다.
+- `DATA_GO_KR_SERVICE_KEY`를 표준 secret reference로 사용하고 기존
+  `MOFA_TRAVEL_ALARM_SERVICE_KEY`는 한시적 호환 fallback으로만 허용합니다.
+- live schema probe는 management command에서 endpoint별 최소 한 번만 수행하며 raw
+  response를 fixture, file, log 또는 audit에 저장하지 않습니다.
+
+## 첫 frontend 방향
+
+브랜드는 정확히 `어디 갈까 ??`입니다. 메인은 여행 가능 시간 입력에 집중하고,
+결과는 card grid가 아닌 세로형 일정 목록으로 구성합니다. 결과의 읽기 순서는 검색
+시간, 추천 도시와 예상 체류시간, 출국·귀국 운항 예정편, 대체 일정, 계산 근거,
+독립된 입국요건과 여행경보입니다. 외부 font·image·icon, SPA와 공개 JSON API를
+추가하지 않습니다.


