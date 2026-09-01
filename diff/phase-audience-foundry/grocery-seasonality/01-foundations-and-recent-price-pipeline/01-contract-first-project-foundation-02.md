## `docs(domain): fix grocery seasonality contract`

diff --git a/README.md b/README.md
index dc2e121..49a3a53 100644
--- a/README.md
+++ b/README.md
@@ -1,18 +1,60 @@
-# Audience Foundry 프로젝트 기준선
+# Audience Foundry Grocery Seasonality
 
-이 저장소는 기본적으로 로컬·비공개인 Audience Foundry 제품 기준선입니다. 이
-commit에는 정책만 있으며 runtime, dependency graph, account, credential,
-database, deployment, 외부 연동 또는 production 준비 완료 주장이 없습니다.
+한국 소비자가 한국농수산식품유통공사(KAMIS)의 채소·과일 소매 조사 평균을
+동일 품목·품종·등급·판매 단위 안에서 조사일, 1주 전, 1개월 전, 1년 전과
+중립적으로 비교하는 한국어 서비스의 문서 기준선입니다.
 
-제품 계약은 `docs/` 아래 여섯 결정 문서를 완성해 별도 문서 checkpoint로
-commit할 때 고정됩니다. 구현은 그 checkpoint와 clean working tree를 확인하고,
-위험한 외부 interface마다 문서에 정한 실제 viability gate를 파일 변경 전에
-통과한 뒤에만 시작합니다.
+`seasonality`는 저장소 코드명입니다. 첫 MVP는 한 해 전 값 하나를 계절성·평년·제철의
+증거로 바꾸지 않습니다. 공개 화면은 공식 조사값과 결정적인 차이만 표시하며
+`제철`, `저렴하다`, `비싸다`, `지금 사기 좋다`, `추천`, `최저가`, `가격 예측`을
+주장하지 않습니다.
 
-저장소 기본값은 다음과 같습니다.
+## 저장소 상태
 
-- branch: `main`
-- remote: 없음
+- 저장소: `audience-foundry-grocery-seasonality`
+- 기본 브랜치: `main`
+- 정책 기준선: `0cc95e7`
+- 원격 저장소: 없음
 - 공개 상태: 로컬·미공개
-- legacy 구현 또는 이력 재사용: 이후의 고정 결정이 범위, provenance, 검증과
-  rollback을 명시적으로 승인하기 전에는 금지
+- 구현 상태: runtime, dependency, database, credential, 외부 연동과 배포 없음
+
+## 고정된 첫 범위
+
+- 첫 source는 공공데이터포털의 KAMIS
+  [최근일자 도·소매가격정보 API `15156063`](https://www.data.go.kr/data/15156063/openapi.do)입니다.
+- 공개 대상은 source가 `소매`로 식별한 채소류·과일류입니다.
+- 상세 화면은 정확히 한 품목·품종·등급·판매 단위·검증된 조사범위의 profile입니다.
+- 비교 기준은 source가 제공한 조사일, 1주 전, 1개월 전, 1년 전 가격입니다.
+- 1일 비교, 도매, 수산·축산·곡물, 지역 간 비교, 순위와 알림은 첫 범위가 아닙니다.
+- 계정, 검색 이력, 장바구니, 위치, GPS와 개인화는 만들지 않습니다.
+- 공개 request는 외부 source를 호출하지 않고 승인된 PostgreSQL publication만 읽습니다.
+
+## 구현 전 필수 source gate
+
+공공데이터포털 메타데이터는 API가 무료이고 이용허락 제한 없음이며 JSON/XML,
+개발·운영 자동승인이라고 안내하지만 live contract를 증명하지는 않습니다. 파일을
+바꾸기 전에 사람이 key 발급·입력 단계에서 멈추고 실제 최소 요청으로 HTTPS 접근,
+권리, quota, pagination, 오류 envelope, 코드 identity, 결측, 평균·조사범위,
+단위와 세 비교기간의 의미를 검증해야 합니다. 실패한 live evidence를 fixture로
+대체하지 않습니다.
+
+비교기간 의미만 실패하면 현재 공식 소매 조사 평균 조회로 축소합니다. API가
+운영에 부적합하지만 공식
+[월별 소매가격 파일 `15087482`](https://www.data.go.kr/data/15087482/fileData.do)의
+권리·identity·단위가 통과하면 파일 공표본과 각 row의 기준 연월을 명시한 별도 정적
+월별 탐색기로 축소할 수 있습니다. KAMIS HTML scraping, 비공식 미러와 quota 우회는
+허용하지 않습니다.
+
+## 계약 문서
+
+- [도메인 개요](docs/DOMAIN-BRIEF.md)
+- [제품 결정](docs/PRODUCT-DECISIONS.md)
+- [시스템 경계](docs/SYSTEM-BOUNDARIES.md)
+- [데이터·감사 모델](docs/DATA-AND-AUDIT-MODEL.md)
+- [기술 결정](docs/TECHNOLOGY-DECISIONS.md)
+- [MVP 인수 기준](docs/MVP-ACCEPTANCE.md)
+
+구현자는 여섯 문서와 root 정책을 처음부터 끝까지 읽고 Git 기준선을 확인한 뒤,
+저장소 변경 없이 source gate를 수행합니다. 안전한 공개 path가 통과한 경우에만
+`docs/IMPLEMENTATION-PLAN.md`를 만들고 작은 검증 가능한 commit으로 첫 루프를
+구현합니다.
diff --git a/docs/DATA-AND-AUDIT-MODEL.md b/docs/DATA-AND-AUDIT-MODEL.md
new file mode 100644
index 0000000..2a17966
--- /dev/null
+++ b/docs/DATA-AND-AUDIT-MODEL.md
@@ -0,0 +1,293 @@
+# 데이터·감사 모델
+
+## 설계 원칙
+
+- 범용 EAV나 원문 JSON을 domain model로 사용하지 않습니다.
+- source receipt, bytes, parsing, domain fact, 사람 결정과 publication을 분리합니다.
+- 공식 코드가 identity이고 이름은 표시·검색 보조입니다.
+- 돈은 float가 아닌 검증된 자릿수의 Decimal로 저장합니다.
+- 조사일, source 기록시각, fetch 시각과 공개시각을 서로 바꾸어 쓰지 않습니다.
+- 불변 publication, append-only activation과 원자적인 current pointer로 rollback을
+  수행합니다.
+
+## 상태 수명주기
+
+```text
+FetchAttempt
+  -> SourceArtifact
+  -> ParseRun
+  -> RetailPriceSnapshot + ReferencePrice + PriceChangeFact
+     | MonthlyRetailPriceSnapshot
+  -> ReviewDecision
+  -> PublicationRevision
+  -> PublicationActivation + channel current pointer
+```
+
+### SourceConfiguration
+
+`DRAFT → RIGHTS_APPROVED → ACTIVE | PAUSED | REJECTED`
+
+- source owner, dataset ID, interface revision과 endpoint allowlist
+- authentication mode, quota, timeout, retry policy와 schedule
+- rights evidence locator·hash·확인시각, raw retention 결정
+- 활성 공개 mode: `RECENT_COMPARISON`, `CURRENT_ONLY`, `STATIC_MONTHLY_FILE`
+
+credential 값은 저장하지 않고 managed secret의 논리 이름만 참조합니다.
+
+### FetchAttempt
+
+`STARTED → SUCCEEDED | RETRYABLE_FAILED | TERMINAL_FAILED`
+
+- 전체 pagination을 완성하려는 하나의 논리적 획득 시도와 `attempt_ordinal`
+- source configuration revision, 시작·종료시각과 redacted normalized request
+- 순서가 있는 `PageReceipt`: request ordinal, page identity, HTTP status, provider result code,
+  declared total, received row count와 body hash
+- 전체 received rows·pages, page 대사 결과와 failure class
+- response body hash 또는 body를 받지 못한 이유
+
+논리적 획득의 재시도는 새 attempt입니다. 한 attempt의 page를 다른 attempt와 섞지 않으며
+모든 page·row 대사가 끝나기 전에는 `SourceArtifact`를 만들지 않습니다. key, raw query
+string, response body와 개인정보를 receipt에 넣지 않습니다.
+
+### SourceArtifact
+
+`RECEIVED → VALIDATED | REVIEW_REQUIRED | REJECTED`
+
+- source identity와 ordered page body hash manifest의 SHA-256 content hash
+- 전체 byte length, page media type·encoding과 acquisition method
+- first observed timestamp, rights·retention decision
+- private object locator 또는 `HASH_ONLY`
+
+같은 source identity와 ordered page manifest hash만 중복 제거합니다. page가 하나인 source도
+같은 manifest 규칙을 사용합니다. body가 같아도 새 attempt가 기존 artifact를 참조하며
+artifact 자체는 바꾸지 않습니다. `last_checked_at`은 source별 성공 attempt에서 계산하고
+`fetched_at`은 artifact identity가 아닙니다. 같은 bytes 확인은 source 조사일이나 공개 데이터
+freshness를 갱신하지 않습니다.
+
+### ParseRun
+
+`STARTED → VALIDATED | QUARANTINED | FAILED`
+
+- artifact, parser revision, source schema/interface revision
+- deterministic configuration hash와 result hash
+- total, accepted, missing-reference, out-of-scope, quarantined row count
+- duplicate series, code/name conflict, unit drift와 error summary
+
+같은 artifact·parser·configuration의 replay는 같은 result hash와 candidate를 만들어야
+하며 중복 snapshot이나 review task를 만들지 않습니다.
+
+## Typed domain model
+
+### ProductClass
+
+첫 공개 값은 `RETAIL`뿐입니다. source의 도매 행은 `out_of_scope`로 대사합니다.
+
+### GroceryCategory
+
+- `VEGETABLE`
+- `FRUIT`
+
+공식 source code와 표시명을 함께 보존합니다. code mapping이 실제 응답과 code 문서에서
+증명되지 않으면 category를 추측하지 않습니다.
+
+### CommodityKey
+
+- 공식 부류 code
+- 공식 품목 code
+- 공식 품종 code
+- 각 공식 원문 표시명
+
+이름 수정은 identity 변경이 아닐 수 있으므로 code/name 충돌을 검토 대상으로 둡니다.
+공식 code가 재사용되거나 범위가 바뀌면 자동 병합하지 않습니다.
+
+### GradeKey
+
+- 공식 등급 code
+- 공식 원문 등급명
+
+등급 없음과 등급 미제공을 같은 값으로 만들지 않습니다.
+
+### SaleUnit
+
+- 원문 단위
+- 원문 단위크기
+- 의미를 증명한 source contract revision
+
+`개`, `포기`, `단`, `봉`, `kg` 사이를 자체 변환하지 않습니다. 단위 표현을 정규화해
+검색할 수 있어도 원문과 semantic identity를 보존합니다.
+
+### MarketCoverage
+
+- source가 제공하거나 gate에서 검증한 coverage code
+- 공개 가능한 정확한 설명
+- 집계 수준과 평균 의미의 evidence reference
+
+source가 지역 차원을 제공하지 않으면 `전국`을 만들어내지 않습니다. 검증된 source
+aggregate를 별도 typed 값으로 둡니다. coverage를 설명하지 못한 candidate는 공개하지
+않습니다.
+
+### PriceSeriesKey
+
+다음 tuple의 immutable semantic identity입니다.
+
+`(RETAIL, category_code, item_code, variety_code, grade_code, raw_unit, raw_unit_size, coverage_identity)`
+
+공식 code뿐 아니라 단위·coverage가 모두 같아야 비교할 수 있습니다. 명칭 유사도와
+관측시각은 identity에 포함하지 않습니다.
+
+### RetailPriceSnapshot
+
+- `PriceSeriesKey`
+- source 조사일
+- source 원본등록일시가 실제 제공될 때 그 값
+- 현재 조사 평균 Decimal과 통화 `KRW`
+- source row identity 또는 deterministic semantic hash
+- artifact·parse run·source contract revision
+
+`0`이 실제 가격인지 sentinel인지 gate에서 확인하기 전에는 유효 Decimal로 받지 않습니다.
+
+### ReferencePrice
+
+- snapshot identity
+- period: `WEEK | MONTH | YEAR`
+- source 제공 Decimal 또는 `UNAVAILABLE`
+- source가 실제 기준일을 제공할 때만 reference date
+- 결측·invalid·unsupported reason
+
+source가 `1주 전` 값만 제공하면 조사일에서 7일을 빼서 날짜를 만들지 않습니다.
+
+### PriceChangeFact
+
+- reference price identity
+- direction: `LOWER | EQUAL | HIGHER | UNAVAILABLE`
+- signed KRW difference
+- optional signed percentage
+- calculation version과 rounding mode
+
+비율은 기준값이 0보다 크고 두 값이 같은 series·currency·unit일 때만
+`(current - reference) / reference × 100`으로 계산해 소수점 첫째 자리 half-up으로
+반올림합니다. direction은 산술 결과일 뿐 `저렴`, `비싸`, `추천`의 의미가 없습니다.
+
+### MonthlyRetailPriceSnapshot
+
+정적 월별 file path 전용 typed fact이며 recent comparison model에 넣지 않습니다.
+
+- 공식 file publication identity와 artifact·parse run
+- source row의 `year_month`
+- `PriceSeriesKey`
+- source gate가 증명한 Decimal scale의 monthly mean과 통화 `KRW`
+- source row identity 또는 deterministic semantic hash
+
+이 snapshot은 현재 조사 평균, 1주·1개월·1년 reference와 `PriceChangeFact`를 만들지 않습니다.
+별도 `STATIC_MONTHLY` publication channel과 route에서 file 공표본·row 기준 연월을 표시합니다.
+recent source의 결측 기간을 채우거나 두 model을 한 graph에 섞지 않습니다.
+
+## ReviewDecision
+
+append-only decision type은 `APPROVE | REJECT`입니다.
+
+- reviewer actor, decision timestamp와 reason code
+- source configuration, artifact와 parse run
+- reconciliation report hash와 acceptance evidence
+- 승인된 mode와 coverage
+- 이전 결정을 교체할 때의 optional `supersedes_decision`
+
+reviewer는 source fact를 편집해 맞추지 않습니다. 수정은 새 source artifact 또는 새
+parser run으로 표현합니다. 이전 decision의 상태를 바꾸지 않고 새 decision으로 교체합니다.
+
+## PublicationRevision
+
+승인된 공개 내용의 불변 revision입니다.
+
+- immutable generation identity, channel과 mode
+- 승인 decision, typed fact set hash와 parser revision
+- revision 생성시각과 public copy revision
+- source 조사일 범위 또는 file 공표본·row 기준 연월 범위
+
+revision은 활성화·대체·철회 때 상태를 바꾸지 않습니다. 하나의 revision에 이전
+generation의 빠진 행을 채워 넣지 않습니다. channel은 `RECENT_RETAIL`과
+`STATIC_MONTHLY`를 분리합니다.
+
+## PublicationActivation
+
+`ACTIVATE | ROLLBACK | WITHDRAW`
+
+- publication channel과 대상 revision; `WITHDRAW`에는 대상이 없을 수 있음
+- actor, reason code, 승인 evidence와 append-only event timestamp
+- 직전 current revision과 전환 결과
+
+activation event 추가, channel current pointer와 성공 audit는 한 PostgreSQL transaction에서
+전환합니다. rollback은 이전 revision을 수정하거나 상태를 되돌리지 않고 그 revision을
+가리키는 새 `ROLLBACK` event를 추가합니다.
+
+## 멱등성과 중복 방지
+
+- request retry: 별도 FetchAttempt
+- artifact: `(source identity, ordered page manifest SHA-256)`
+- parse: `(artifact, parser revision, configuration hash)`
+- snapshot: `(parse run, PriceSeriesKey, source survey date)`
+- reference: `(snapshot, period)`
+- monthly snapshot: `(parse run, PriceSeriesKey, source year_month)`
+- publication: `(channel, approved generation set hash, public copy revision)`
+- activation: append-only event identity; 재활성화도 새 event
+
+`fetched_at`, `observed_at`, 실행 ID, attempt ordinal과 database sequence는 semantic key에
+넣지 않습니다. 동일 bytes 재확인은 마지막 성공 확인 상태만 갱신하고 source 조사일,
+publication revision과 domain fact를 바꾸거나 복제하지 않습니다.
+
+## 시간 모델
+
+- `source_effective_date: LocalDate`: recent source의 조사일
+- `source_effective_month: YearMonth`: monthly row의 기준 연월; 임의의 첫날 날짜로 바꾸지 않음
+- `source_recorded_at`: source가 실제 제공하는 원본등록일시
+- `fetch_started_at`·`fetch_completed_at`: 실제 외부 호출 시각
+- `artifact_first_seen_at`: artifact를 처음 만든 성공 attempt 시각
+- `last_checked_at`: source configuration의 최근 성공 attempt에서 계산한 확인시각
+- `reviewed_at`: 사람 결정 시각
+- `revision_created_at`: 불변 publication revision 생성시각
+- `activated_at`: 현재 pointer 전환 event 시각
+
+공공데이터포털의 `실시간` 갱신 표시는 매장 실시간 가격이 아닙니다. 어느 시각도 다른
+시각의 fallback으로 사용하지 않습니다.
+
+## 전체 대사와 publication gate
+
+각 source 행은 정확히 다음 중 하나여야 합니다.
+
+- `PUBLISHABLE`
+- `PUBLISHABLE_WITH_MISSING_REFERENCE`
+- `OUT_OF_SCOPE`
+- `QUARANTINED`
+
+page total과 네 상태의 합이 일치해야 합니다. 현재값 누락은 항상 `QUARANTINED`이며
+reference 누락만 period별 `UNAVAILABLE`을 가진 공개 가능 상태입니다. pagination 누락,
+중복 series, code/name conflict, unknown category·grade, unit drift, malformed Decimal,
+coverage 부재와 비결정적 replay는 generation 전체 publication을 차단합니다.
+
+첫 승인 generation은 전체 `PriceSeriesKey` 집합과 상태별 count의 대사 기준선입니다.
+후속 generation에서 key 소실·추가, identity 차원 변화 또는 상태 이동이 생기면 자동으로
+안전하다고 보지 않고 사람 검토를 요구합니다.
+
+비교값이 일부 없는 유효 snapshot은 현재값을 공개할 수 있지만 해당 period를
+`비교값 없음`으로 표시합니다. 누락을 0원·변화 없음·비제철로 해석하지 않습니다.
+
+## 개인정보와 retention
+
+- 공개 profile에는 사용자 데이터가 없습니다.
+- 검색어, IP, User-Agent, 클릭·관심 이력, 공개 방문자 session과 analytics identifier를
+  domain·audit에 저장하지 않습니다. Django Admin 인증 session은 별도 보안 retention과
+  최소권한 정책을 따릅니다.
+- source key, 전체 query와 gateway trace는 어느 table에도 저장하지 않습니다.
+- raw bytes는 권리와 운영 필요가 승인된 경우에만 private storage와 정해진 retention으로
+  보존합니다. 아니면 hash·최소 receipt·정규화 사실만 남깁니다.
+- audit·publication은 정책상 필요한 기간 append-only로 보존하고, 삭제·축약 정책도
+  별도 승인과 증거를 요구합니다.
+
+## 복구와 rollback
+
+- 잘못된 publication은 history를 삭제하거나 revision을 수정하지 않고 `WITHDRAW` 또는 이전
+  승인 revision을 대상으로 한 새 `ROLLBACK` activation으로 처리합니다.
+- parser rollback과 publication rollback을 분리합니다.
+- schema 변경은 expand → compatible write/read → migrate → contract 단계로 나눕니다.
+- backup은 암호화하고 PITR 정책을 가지며 production 공개 전 실제 restore를 연습합니다.
+- source 권리가 철회되면 raw artifact retention과 공개 publication을 각각 재평가합니다.
diff --git a/docs/DOMAIN-BRIEF.md b/docs/DOMAIN-BRIEF.md
new file mode 100644
index 0000000..935de50
--- /dev/null
+++ b/docs/DOMAIN-BRIEF.md
@@ -0,0 +1,97 @@
+# 도메인 개요
+
+## 제품 정체성
+
+Audience Foundry Grocery Seasonality는 한국 소비자가 KAMIS의 채소·과일 소매
+조사 평균과 source가 제공한 과거 비교값을 정확한 품목·품종·등급·판매 단위 안에서
+확인하는 한국어 B2C 서비스입니다. 쇼핑몰 가격비교, 구매 추천, 제철 식품 사전,
+가격 예측 또는 영양 서비스가 아닙니다.
+
+`seasonality`는 저장소 코드명입니다. 첫 MVP는 `1년 전` 값 하나를 계절성이나 평년의
+증거로 해석하지 않습니다. 계절성이라는 공개 주장은 동일 series의 다년 월별 이력,
+코드 연속성과 coverage를 별도 gate와 제품 결정으로 승인하기 전에는 비활성입니다.
+
+## 사용자가 겪는 문제
+
+공식 농수산물 가격 자료는 품목·품종·등급·단위가 섞이면 비교 결과가 달라지지만,
+일반 소비자가 그 차원을 한눈에 확인하기 어렵습니다. 뉴스나 검색 결과의 `가격 상승`,
+`제철`, `저렴` 같은 표현은 조사범위, 비교기준과 단위를 생략할 수 있습니다. 제품은
+해석을 확대하지 않고 한 profile의 공식 조사값, 차이, 기준과 freshness를 함께
+보여줍니다.
+
+## 첫 사용자와 첫 질문
+
+- 한국어로 채소·과일 가격 변화를 확인하려는 비로그인 소비자
+- 특정 공식 품목·품종·등급·판매 단위의 조사일 평균이 source가 제공한 1주 전,
+  1개월 전, 1년 전 값과 어떻게 다른지 알고 싶은 사용자
+- 데이터가 없거나 비교할 수 없을 때도 0원·변동 없음·비제철이라는 오답 대신
+  정직한 상태를 원하는 사용자
+
+제품은 실제 매장 구매가, 최저가, 사용자의 생활권 가격 또는 미래 가격을 답하지
+않습니다.
+
+## 첫 폐쇄 루프
+
+1. 스케줄 작업이 승인된 KAMIS source를 한 번 조회합니다.
+2. 전체 pagination을 완성하려는 논리적 획득 시도를 `FetchAttempt`로 기록하고 각 HTTP
+   page의 redacted receipt를 순서대로 남깁니다.
+3. versioned parser가 소매 채소·과일 행을 typed candidate로 만들고 전체 행을
+   대사합니다.
+4. 코드·단위·조사범위·결측·중복·행 수 gate를 통과한 generation만 검토합니다.
+5. 승인된 불변 `PublicationRevision`을 append-only activation 사건으로 현재 pointer에
+   원자적으로 연결합니다.
+6. 사용자는 외부 호출 없이 목록에서 profile을 고르고 조사일·비교값·차이·출처를
+   확인합니다.
+7. 실패 시 마지막 정상 publication을 유지하고 공개 조사일과 마지막 확인 상태를 구분해
+   제공합니다.
+
+## 첫 공개 범위
+
+- source 구분: 소매
+- 부류: 채소류, 과일류
+- 화면: 부류별 목록, 공식 품목명 검색, 정확히 한 series의 상세
+- 비교: 1주, 1개월, 1년
+- 결과: 현재 조사 평균, 과거 제공값, 원화 차이, `높음·같음·낮음·비교불가`
+- provenance: source, 조사일, 실제 확인시각, 검증된 조사범위, 단위·크기, freshness
+
+공식 코드로 안전하게 식별되는 공개 profile이 채소와 과일에 각각 하나 이상 있어야
+첫 루프가 성립합니다. 명칭만 같거나 필수 차원이 누락된 행은 공개하지 않습니다.
+
+## 성공 기준
+
+- 사용자가 한 화면에서 무엇과 무엇을 비교했는지 다시 설명할 수 있습니다.
+- 모든 공개 가격과 차이를 source artifact, parser revision, snapshot, 검토 결정과
+  publication으로 역추적할 수 있습니다.
+- 같은 bytes의 replay는 같은 typed 결과를 만들고 중복 publication을 만들지 않습니다.
+- source 장애·parser 실패·불완전 generation이 마지막 정상 화면을 손상하지 않습니다.
+- 공개 문구가 조사 평균을 매장가·전국 평균·최저가·제철·추천·예측으로 확대하지
+  않습니다.
+
+## 명시적 실패
+
+- 평균의 조사 대상·공간범위를 설명하지 못하면서 가격을 공개합니다.
+- 품종·등급·단위·단위크기·coverage가 다른 값을 비교합니다.
+- 결측, `0`, `-` 또는 sentinel을 0원이나 변화 없음으로 바꿉니다.
+- 행 소실을 품절·판매 종료·비제철로 표현합니다.
+- 1년 전 값 하나로 평년·계절성·제철을 주장합니다.
+- 사용자의 검색어·관심 품목을 공개 방문자 session, log, analytics 또는 광고 audience에
+  남깁니다.
+
+## 비목표
+
+- 마트·쇼핑몰별 실구매가, 최저가와 구매 링크
+- 가격 전망, 구매 시점, 절약액, 장바구니 최적화와 추천 순위
+- 제철·신선도·품질·맛·영양·건강·레시피 판단
+- 도매, 수산·축산·곡물, 지역 간 비교와 서로 다른 source 결합
+- 사용자 계정, 즐겨찾기, 알림, 위치, GPS, 개인화와 광고 추적
+- 모바일 앱, 공개 JSON API, 별도 SPA와 범용 ingestion framework
+
+## 핵심 용어
+
+- **조사일 가격**: KAMIS가 해당 record에 제공한 소매 조사 평균입니다.
+- **비교값**: source가 1주·1개월·1년 전으로 제공한 동일 series의 후보 값입니다.
+- **Price series**: 소매 구분, 공식 품목·품종·등급, 원문 판매 단위·크기와 검증된
+  조사범위가 모두 같은 가격 흐름입니다.
+- **조사범위**: source gate가 실제로 증명한 시장·지역·집계 의미입니다.
+- **freshness**: 공개 중인 source 조사일과 마지막 성공 확인 상태를 서로 바꾸지 않고
+  함께 나타낸 정보입니다. 같은 bytes를 다시 확인해도 조사일이 새로워지지는 않습니다.
diff --git a/docs/MVP-ACCEPTANCE.md b/docs/MVP-ACCEPTANCE.md
new file mode 100644
index 0000000..e9eebb5
--- /dev/null
+++ b/docs/MVP-ACCEPTANCE.md
@@ -0,0 +1,229 @@
+# MVP 인수 기준
+
+## 판정 원칙
+
+이 저장소의 두 번째 커밋은 제품 계약 완료 지점이지 구현 완료가 아니다. MVP는 실제 공식
+source, 승인된 권리 판단, 고정 artifact와 운영과 같은 build에서 아래 필수 항목을 증명해야
+완료된다. 필수 항목 하나라도 실패하면 해당 path를 공개하지 않고 문서의 current-only,
+정적 월별 file 또는 stop 조건으로 이동한다.
+
+## 게이트 0: Git 기준선
+
+첫 파일 변경 전에 구현 세션은 다음을 검증한다.
+
+- [ ] 현재 경로가 `audience-foundry-grocery-seasonality`이고 branch는 `main`이다.
+- [ ] 이 문서 계약 commit의 부모가 공통 정책 기준선 `0cc95e70824e02a78207fe983f076e38a59c764f`이다.
+- [ ] 추적 파일은 README와 공통 정책 4개, 제품 문서 6개로 정확히 11개다.
+- [ ] remote가 없고 working tree가 깨끗하며 `git fsck`가 통과한다.
+- [ ] 런타임 코드, dependency, lockfile, credential, 수집 데이터와 배포 설정이 아직 없다.
+
+기준이 다르면 구현을 시작하지 않고 차이를 사람에게 보고한다.
+
+## 게이트 1: live source·권리 생존성
+
+source-dependent schema나 adapter를 만들기 전에 저장소 소유자가 다음 실제 증거를 승인한다.
+
+- [ ] 공식 소유자와 공공데이터포털의
+  [최근일자 도·소매가격정보 API `15156063`](https://www.data.go.kr/data/15156063/openapi.do)
+  landing URL, API 문서 버전과 운영 host가 기록되어 있다.
+- [ ] 사람이 발급받아 secret으로 주입한 key로 공식 HTTPS endpoint의 실제 응답을 재현한다.
+- [ ] 개발·운영 quota, 호출 단위, pagination, timeout, `429`, provider error와 이용 가능 시간이
+  실제 요청과 공식 문서로 확인된다.
+- [ ] 원시 응답 보존, 내부 변환, 파생 가격 사실 공개, cache와 출처 표시의 허용 범위를 각각
+  판정한다.
+- [ ] JSON 또는 XML의 실제 field, type, encoding, missing·sentinel, 중복과 error envelope를
+  기록한다.
+- [ ] 소매·채소·과일 범위에서 category, item, variety, grade, unit와 unit size code를
+  안정적으로 식별할 수 있다. region·market field가 있으면 그 code를 검증하고, 없으면
+  제공자 문서와 실제 응답으로 aggregate 범위와 안정적인 `coverage_identity`를 증명한다.
+- [ ] 현재 조사 평균과 1주·1개월·1년 전 평균의 비교기간 의미, coverage, 단위와 산출
+  의미가 비교 가능한 같은 series로 확인된다. 정확한 reference date는 source가 제공할 때
+  검증하고, 제공하지 않으면 `SOURCE_REFERENCE_DATE_UNAVAILABLE`을 보존한다.
+- [ ] 휴일·비조사일, 늦은 갱신, 부분 응답과 과거 날짜 요청에서 source가 반환하는 기준일
+  의미를 확인한다.
+- [ ] 동일 요청 반복의 stable field와 변동 field, content hash와 idempotency 규칙을
+  재현한다.
+- [ ] 최소 채소 5개·과일 5개의 서로 다른 exact series를 포함한 bounded canary matrix로
+  code, 단위, 결측과 반복 조회를 검증한다. source가 이 수를 제공하지 않거나 검증된 호출
+  예산 안에서 재현할 수 없으면 범위를 임의로 채우지 않고 path 선택을 다시 승인한다.
+- [ ] 허용된 호출 예산으로 정기 수집, 대사, retry와 운영자 확인을 지속할 수 있다.
+- [ ] 전체 pagination을 완성하는 한 논리적 획득을 `FetchAttempt` 하나로 기록하고 각 page의
+  순서·row count·body hash를 대사한다. 논리적 재시도는 새 attempt이며 서로 다른 attempt의
+  page를 한 artifact로 합치지 않는다.
+
+key 발급·로그인·약관 동의·유료 전환이 필요하면 자동화하지 않고 사람에게 멈춘다. live
+evidence 실패를 fixture, 비공식 mirror, HTML scraping 또는 quota 우회로 대체하지 않는다.
+
+## 활성 path 판정
+
+### A. 최근 비교 path
+
+게이트 1 전체를 통과하면 현재 KAMIS 소매 조사 평균과 같은 series의 1주·1개월·1년 전
+reference를 공개한다.
+
+### B. current-only path
+
+현재값의 identity·단위·권리·운영성은 통과했지만 reference 기간의 의미나 같은 series임을
+증명하지 못하면 현재 조사 평균만 공개한다. 차액, 퍼센트와 방향 문구는 렌더링하지 않는다.
+
+### C. 정적 월별 file path
+
+API 운영이 불가능할 때만
+[월별 소매가격 파일 `15087482`](https://www.data.go.kr/data/15087482/fileData.do)를 별도 gate로
+검증한다. 공식성, 재배포 권리, file 공표본 identity, row별 기준 연월, code identity,
+unit와 coverage가 통과하면 공표본과 각 row의 기준 연월을 명시한 정적 탐색 profile만
+공개한다. 실제 공식 file이 `FetchAttempt →
+SourceArtifact → ParseRun → MonthlyRetailPriceSnapshot → ReviewDecision →
+PublicationRevision → PublicationActivation`을 통과해야 한다. 별도
+`STATIC_MONTHLY` publication channel·route·copy·rollback을 사용하고 recent comparison
+snapshot과 섞지 않는다. 이를 현재 가격이나 실시간 정보로 표현하지 않는다.
+
+### D. stop
+
+어느 path도 source 공식성, 권리, identity, 단위, coverage와 반복 가능한 운영을 증명하지
+못하면 공개 출시와 source-dependent 구현을 중단한다. 실패 이유와 증거만 남긴다.
+
+## 필수 양성 시나리오
+
+### 수집·감사·공개
+
+- [ ] 한 실제 공식 응답이 `FetchAttempt → SourceArtifact → ParseRun → typed
+  RetailPriceSnapshot/ReferencePrice/PriceChangeFact → ReviewDecision → PublicationRevision`
+  전 단계를 거쳐 사람 승인 후 activation으로 공개된다.
+- [ ] 각 단계에서 source, 이전 단계, 실행자 또는 process, 시각, code·parser version, hash와
+  상태를 역추적할 수 있다.
+- [ ] 동일 ordered page manifest를 다시 획득하면 새 `FetchAttempt`는 생기지만
+  `SourceArtifact`는 중복되지 않는다.
+- [ ] 같은 content의 재확인은 source의 마지막 성공 확인 상태만 갱신하고 artifact,
+  source 조사일과 공개 데이터 freshness를 바꾸지 않는다.
+- [ ] 동일 artifact와 parser version 재실행은 같은 typed row 집합 hash를 만들며 snapshot을
+  중복하지 않는다.
+- [ ] `fetched_at`만 다른 재수집은 새 content나 새 publication의 근거가 되지 않는다.
+- [ ] parser version이 바뀌면 새 `ParseRun`과 review candidate가 생기고 승인 전에는 공개되지
+  않는다.
+- [ ] 승인 revision의 row count, code별 count, coverage, missing·quarantine count와 집합
+  hash가 대사 보고서와 일치한다.
+- [ ] 새 source 실패 중에도 last-known-good가 유지되고 사용자에게 그 기준일과 검토일을
+  표시한다.
+
+### 사용자 읽기
+
+- [ ] 소매·채소 또는 과일 목록에서 공식 품목명으로 검색하고 한 exact series 상세로 이동할
+  수 있다.
+- [ ] 상세 화면은 item·variety·grade·unit·unit size와 `market` 또는 검증된 aggregate
+  coverage를 source가 제공한 범위 안에서 명확히 표시한다.
+- [ ] 현재값 8,000원과 1주 기준값 10,000원이면 `2,000원 낮음`, `-20.0%`를 표시한다.
+- [ ] 현재값 12,500원과 1개월 기준값 10,000원이면 `2,500원 높음`, `+25.0%`를 표시한다.
+- [ ] 현재값과 1년 기준값이 같으면 `같음`, `0.0%`를 표시한다.
+- [ ] 기준값이 `0`이거나 없으면 차액·퍼센트를 계산하지 않고 `비교 정보 없음`을 표시한다.
+- [ ] 가격 가까이에 `KAMIS 소매 조사 평균`, source 조사일, coverage, 단위와 publication
+  검토일을 표시한다. reference별 실제 날짜가 있으면 그 날짜를 표시하고 없으면
+  `source가 비교 기준일을 별도로 제공하지 않음`을 표시한다.
+- [ ] 방향은 중립적 사실로만 표현하며 구매·품질·영양·제철·미래 가격 판단을 덧붙이지
+  않는다.
+
+## 필수 음성 시나리오
+
+- [ ] 소매와 도매, 채소와 과일, 서로 다른 item·variety·grade·unit·unit size의 값은 비교하지
+  않는다. source가 region·market을 제공하면 그 code가 다른 값도, 제공하지 않으면 검증된
+  aggregate coverage가 다른 값도 비교하지 않는다.
+- [ ] 이름이 비슷하다는 이유로 다른 code를 자동 결합하지 않는다. code가 사라지거나 바뀌면
+  새 review 없이는 이전 series에 연결하지 않는다.
+- [ ] 임의 kg 환산, 지역 간 평균 합성, 시장 최저가, 쇼핑몰 가격, 할인과 배송비를 만들지
+  않는다.
+- [ ] `null`, 빈 문자열, sentinel, 음수, 잘못된 decimal, 알 수 없는 단위와 중복 identity는
+  `0원`으로 보정하지 않고 격리한다.
+- [ ] 기준값이 `0`일 때 division을 수행하지 않는다. float rounding을 사용하지 않는다.
+- [ ] source가 주지 않은 effective date나 recorded time을 fetch·publish time으로 대신하지
+  않는다.
+- [ ] HTTP timeout, `429`, 일시적 `5xx`, TLS 오류, 허용되지 않은 redirect, 응답 크기 초과,
+  content type 변경과 provider error는 성공으로 기록하지 않는다.
+- [ ] field 제거·추가, type·encoding·unit 변경과 duplicate는 publication을 차단한다. 첫 승인
+  generation의 전체 key set·상태별 count와 비교한 후속 key 소실·추가·차원 변경은 사람
+  검토 전까지 last-known-good를 유지한다.
+- [ ] 일부 row만 parse되거나 publication transaction이 실패하면 current pointer를 바꾸지
+  않고 전부 rollback한다.
+- [ ] 늦게 도착한 과거 기준일 응답은 명시적 review 없이 최신 승인 revision을 대체하지
+  않는다.
+- [ ] current-only path에서 reference 값, 차액, 퍼센트와 방향 문구를 노출하지 않는다.
+- [ ] 정적 월별 file path를 현재값, 실시간, 최저가, 예측 또는 개인 추천으로 표현하지 않는다.
+- [ ] 개인정보가 포함된 unexpected field는 domain snapshot과 public output에 복사하지
+  않는다.
+- [ ] 인증되지 않았거나 권한이 부족한 사용자는 Admin, source 설정, review와 publication에
+  접근하지 못한다.
+- [ ] API key, 전체 query string, raw body, 검색어와 운영자 개인정보가 Git, log, error,
+  analytics와 public response에 나타나지 않는다.
+
+## 개인정보·license 인수 기준
+
+- [ ] public 사용에는 계정, 주소, 위치, 장바구니와 구매 이력이 필요하지 않다.
+- [ ] 검색어는 DB, 공개 방문자 session, cache, analytics와 application log에 저장하지
+  않는다. Admin 인증 session은 별도 보안 정책을 따른다.
+- [ ] source·domain·public allowlist에 없는 field는 저장·공개하지 않는다.
+- [ ] raw bytes 보존 권리가 없으면 body 저장은 실패 폐쇄되고 SHA-256, byte length, 최소
+  receipt와 정규화 사실만 남는다.
+- [ ] source 이름, landing URL, recent 조사일 또는 monthly row 기준 연월, coverage, 단위,
+  변환 설명과 검토일을 공개 화면에 표시한다.
+- [ ] dependency와 정적 자산의 license·notice 의무가 `THIRD_PARTY_NOTICES.md`와 실제 배포에
+  반영된다.
+
+## 월별 과거 패턴 module gate
+
+내부 repository 이름만으로 이 module을 활성화하지 않는다. 별도 제품 결정을 승인하기 전에
+다음을 모두 증명한다.
+
+- [ ] 공식 source가 최소 3개 완전 연도의 월별 retail rows를 제공하고 공개·보존 권리가 있다.
+- [ ] 기간 전체에서 item·variety·grade·unit code와 coverage identity가 안정적이고,
+  region·market code가 제공되면 그 의미도 안정적이거나 사람이 승인한 명시적 migration
+  map이 있다.
+- [ ] 결측 월, 조사 빈도, 시장 구성 변화와 명목가격의 한계를 사용자에게 표시할 수 있다.
+- [ ] 한두 달의 값이나 단순 최저 월을 농산물의 자연적 제철·품질·가용성으로 해석하지 않는다.
+- [ ] 독립된 source configuration, parse version, review와 publication rollback이 있다.
+
+통과해도 첫 public 명칭은 `월별 과거 가격 패턴`이다. 계절성 추정, 구매 추천과 forecast는
+새 근거와 새 제품 결정 없이는 범위 밖이다.
+
+## 생산 준비 인수 기준
+
+- [ ] 깨끗한 잠금 설치에서 Python `3.14.7`, Django `5.2.17`, PostgreSQL `18.6`, uv
+  `0.12.6`이 실제 실행 버전으로 확인된다.
+- [ ] formatter, lint, type, unit, integration, parser replay, negative route와 concurrent
+  publication test가 통과한다.
+- [ ] 새 빈 DB와 운영 복제 DB에서 migration, `django check`와 `django check --deploy`가
+  통과한다.
+- [ ] 운영은 HTTPS, `DEBUG=False`, 정확한 host·CSRF 설정, HSTS와 secure cookie를 사용한다.
+- [ ] Admin은 운영자별 최소 권한, MFA 또는 동등한 강한 인증과 로그인 제한을 사용한다.
+- [ ] secret injection·rotation, structured log, liveness·readiness와 source freshness alert가
+  실제 환경에서 동작한다.
+- [ ] 연속 fetch 실패, quarantine 증가, 대사 불일치, publication·backup 실패가 운영자에게
+  경보된다.
+- [ ] 매일 암호화 backup과 point-in-time recovery가 작동하고 빈 환경 복원으로 `RPO 24시간`,
+  `RTO 4시간`, 승인 revision과 audit chain을 검증한다.
+- [ ] 이전 application과 이전 승인 publication으로 각각 rollback하는 훈련이 성공한다.
+- [ ] secret, dependency vulnerability와 license 검사에 해결되지 않은 차단 항목이 없다.
+
+## 성능과 접근성 인수 기준
+
+승인된 catalog 크기로 운영과 같은 환경에서 15분 동안 평균 10 requests/s, 동시 사용자 20명,
+목록·검색 70%와 상세 30%의 read-only 부하를 건다. 응답 p95는 500 ms 이하, `5xx`는 0.5%
+미만이며 DB 연결 고갈과 revision 혼합이 없어야 한다. 이 profile을 넘는 실제 수요가 측정될
+때만 별도 cache를 검토한다.
+
+핵심 page와 form은 keyboard만으로 사용할 수 있고 visible focus, 한국어 label, 오류 요약,
+logical heading, 충분한 contrast와 screen reader로 읽히는 방향 문구를 제공해야 한다. 색만으로
+상태를 전달하지 않는다.
+
+## 사람 승인과 완료 증거
+
+저장소 소유자는 key 발급·이용조건, live source gate, path 선택, 첫 publication, 배포와
+rollback을 각각 승인한다. 구현 이후 만드는 `docs/COMPLETION-REPORT.md`에는 다음 증거를
+연결한다.
+
+- source 문서·요청 시각, redacted receipt, 권리 판정과 code·unit·기간 의미
+- artifact hash, parser·schema·application version, 대사와 test 결과
+- 선택한 A·B·C path 또는 D stop 사유와 비활성 module 목록
+- 보안·license 검사, 접근성·성능 결과, backup restore와 rollback 결과
+- 현재 승인 `PublicationRevision`, 최근 `PublicationActivation`, 알려진 비목표와 다음
+  review 날짜
+
+모든 필수 항목과 사람 승인이 실제 증거에 연결될 때만 MVP 완료로 판정한다.
diff --git a/docs/PRODUCT-DECISIONS.md b/docs/PRODUCT-DECISIONS.md
new file mode 100644
index 0000000..443832c
--- /dev/null
+++ b/docs/PRODUCT-DECISIONS.md
@@ -0,0 +1,174 @@
+# 고정 제품 결정
+
+이 문서는 첫 구현 세션의 변경 승인 없이는 바뀌지 않는 계약입니다. 변경은 제품
+소유자의 명시적 승인과 이유·evidence·영향·migration·rollback을 담은 작은 전용
+문서 commit으로만 수행합니다.
+
+## 저장소와 공개 상태
+
+| 항목 | 결정 |
+|---|---|
+| 저장소 | `audience-foundry-grocery-seasonality` |
+| 정책 기준선 | `0cc95e7` |
+| 기본 브랜치 | `main` |
+| remote | 없음 |
+| 공개 상태 | 로컬·미공개 |
+| 구현 상태 | source gate와 구현계획 승인 전에는 구현하지 않음 |
+
+legacy 코드, 경쟁 서비스 이력, 비공식 가격 dump, KAMIS HTML과 다른 Audience Foundry
+제품의 구현을 가져오지 않습니다. 재사용은 exact source·license·revision·검증·rollback을
+별도 결정으로 승인한 뒤에만 가능합니다.
+
+## 변경할 수 없는 MVP 결정
+
+1. 첫 live comparison source는 공공데이터포털의 KAMIS 최근일자 도·소매가격정보 API
+   `15156063` 하나입니다. 정적 월별 file은 A·B path를 운영할 수 없을 때만 별도
+   source configuration과 제품 profile로 평가하는 fallback입니다.
+2. 실제 HTTPS 응답의 접근·권리·schema·identity·평균·조사범위·비교기간·단위·결측을
+   증명하기 전에 source 종속 schema나 adapter를 구현하지 않습니다.
+3. 공개 대상은 source가 `소매`로 식별한 채소류·과일류뿐입니다.
+4. 공개 profile은 공식 품목·품종·등급·판매 단위·단위크기·검증된 조사범위가 모두
+   같은 하나의 typed series입니다.
+5. 공개 비교 기준은 source가 제공한 1주, 1개월, 1년입니다. 1일 값은 저장·공개하지
+   않습니다.
+6. 화면은 조사일 평균, 비교값, 원화 차이, 변화 방향과 조건부 비율만 표시합니다.
+7. 비율은 같은 series의 유효한 기준값이 0보다 클 때만 Decimal로 계산하고 소수점
+   첫째 자리에서 half-up 반올림합니다.
+8. source가 정확한 과거 기준일을 제공하지 않으면 조사일에서 날짜를 역산하지 않고
+   `source reference date unavailable` 상태를 보존합니다.
+9. 공식 kg 환산값은 의미와 변환 계약을 별도 승인하기 전에는 domain snapshot과
+   공개 화면에 넣지 않습니다. 개·포기·단·봉과 kg를 자체 변환하지 않습니다.
+10. 결측, 빈 문자열, `0`, `-`, sentinel과 malformed 값은 의미를 실제로 검증하기
+    전에는 가격이나 변동 없음으로 바꾸지 않습니다.
+11. 공식 이름은 표시·검색 보조이며 identity가 아닙니다. 이름 유사도로 코드·등급·단위가
+    다른 행을 병합하지 않습니다.
+12. `seasonality`는 저장소 코드명입니다. 다년 월별 source를 별도 gate와 결정으로
+    승인하기 전에는 제철·평년·계절성 공개 문구와 module을 활성화하지 않습니다.
+13. 공개 request는 외부 source를 호출하지 않고 PostgreSQL의 승인된 publication만
+    읽습니다.
+14. source 실패나 불완전 generation은 마지막 정상 publication을 바꾸지 않습니다.
+15. 사용자 계정·검색 이력·위치·개인화·장바구니·알림·광고 audience를 만들지 않습니다.
+16. 범용 EAV, 범용 상품 모델과 공용 자체 ingestion framework를 만들지 않습니다.
+17. measured bottleneck과 별도 결정 없이 Node, SPA, Redis, Celery, queue, search engine,
+    analytics store, spatial extension 또는 microservice를 도입하지 않습니다.
+
+## 공개 표현 계약
+
+허용되는 표현은 다음 형태입니다.
+
+- `KAMIS 소매 조사 평균`
+- `1주 전 제공값보다 2,000원 낮음 (-20.0%)`
+- `비교값 없음`
+- `조사범위 확인 필요`인 candidate는 공개하지 않음
+- source가 보장한 정확한 조사일·단위·품종·등급·coverage
+
+다음 표현은 첫 MVP에서 금지합니다.
+
+- `제철`, `평년보다`, `저렴하다`, `비싸다`, `가성비`, `사세요`, `추천`
+- `실시간 가격`, `전국 평균`, `마트 가격`, `최저가`, `시장 최저`
+- `곧 오른다`, `곧 내린다`, 미래가격·절약액·구매 시점
+- 행 소실에 대한 `품절`, `판매 종료`, `철 종료`, `비제철`
+
+내부 enum `LOWER`와 `HIGHER`는 동일 제공값 간 산술 방향일 뿐 가치 판단이 아닙니다.
+
+## 행위자와 권한
+
+- **방문자**: 공개된 HTML 목록·상세를 읽습니다. canonical state를 변경하지 않습니다.
+- **ingestion worker**: 승인된 read-only source를 호출해 attempt·artifact·candidate를
+  만들 수 있지만 검토와 publication을 수행하지 못합니다.
+- **reviewer**: reconciliation, 결측·단위·identity·coverage report를 보고 generation을
+  승인하거나 거부합니다.
+- **publisher**: 승인된 generation만 publication으로 전환하거나 이전 revision으로
+  rollback합니다.
+- **제품 소유자**: source 권리, 첫 live contract, pivot, production 배포와 고정 결정
+  변경을 승인합니다.
+- **aT·공공데이터포털**: source, 인증, quota, schema와 이용조건을 소유하는 외부
+  경계입니다.
+
+production reviewer·publisher·administrator는 최소권한과 MFA를 사용합니다. key 발급·입력,
+약관 판단, raw 보존 승인, 첫 source 활성화, 파괴적 migration, production 배포·rollback은
+사람 전용 checkpoint입니다.
+
+## 공통 수집·공개 lifecycle
+
+```text
+FetchAttempt → content-addressed SourceArtifact → versioned ParseRun
+  → (RetailPriceSnapshot + ReferencePrice + PriceChangeFact
+     | MonthlyRetailPriceSnapshot)
+  → ReviewDecision → PublicationRevision → PublicationActivation
+```
+
+- 전체 pagination을 완성하려는 논리적 획득과 그 재시도마다 별도 `FetchAttempt`를
+  남기고, 각 HTTP page는 순서가 있는 redacted receipt로 기록합니다.
+- `SourceArtifact`만 `(source identity, ordered page manifest SHA-256)`로 중복 제거합니다.
+- `fetched_at`·`observed_at`·실행 UUID는 semantic identity에 넣지 않습니다.
+- 한 publication은 하나의 완전하고 승인된 generation만 가리킵니다.
+- 승인 decision은 불변 revision을 만들고 append-only activation 사건과 공개 pointer 전환은
+  한 PostgreSQL transaction입니다.
+- 같은 bytes의 재획득은 마지막 성공 확인 상태만 갱신하고 artifact·snapshot을 중복하지
+  않습니다. source 조사일이나 공개 데이터 freshness를 새 값으로 바꾸지 않습니다.
+- 조사일, source 원본등록일시가 있을 때의 그 시각, 실제 fetch 시각과 공개시각을
+  분리합니다.
+
+정적 월별 fallback은 `MonthlyRetailPriceSnapshot`을 사용하며 최근 비교 snapshot에 섞지
+않습니다. source 수명주기는 같지만 별도 publication channel·route·copy와 rollback을
+가집니다.
+
+## 필수 live source gate
+
+key가 필요한 단계에서 사람에게 멈추고, 파일을 변경하지 않은 상태에서 최소 실제
+요청으로 다음을 증명합니다.
+
+1. aT 소유권, 공공데이터포털 랜딩과 HTTPS 배포 URL, interface revision
+2. 무료 여부, 저장·변환·파생값 공개·재배포 권리와 출처표시 의무
+3. key 전달 방식, 개발·운영 quota, timeout, pagination과 오류 envelope
+4. JSON content type·encoding, page total, 중복, 빈값, sentinel과 숫자 형식
+5. 도·소매, 부류, 품목, 품종, 등급, 단위와 단위크기의 공식 코드·의미
+6. 조사일 평균의 표본, 시장·공간범위, 집계 의미와 휴일 처리
+7. 1주·1개월·1년 전 값이 현재와 같은 series라는 제공자 계약과 기간 의미
+8. 반복 요청에서 안정적인 series identity, 행 수와 code/name 일치
+9. 채소·과일에 공개 가능한 유효 profile이 각각 하나 이상 존재함
+10. HTTP 200 내부 오류, 429, timeout, TLS·redirect·schema drift의 실패 동작
+
+메타데이터, 합성 fixture, 비공식 예시와 실패한 live evidence는 이 gate를 통과시키지
+못합니다. key와 실제 query string은 prompt, command history 출력, URL, log, receipt,
+fixture와 문서에 남기지 않습니다.
+
+## source 활성화와 pivot 순서
+
+### A. 최근 비교 path
+
+`15156063`의 live gate가 모두 통과하면 1주·1개월·1년 비교를 활성화합니다.
+
+### B. 현재값-only path
+
+권리·identity·현재 평균은 통과하지만 비교기간 의미 또는 동일 series 보장이 실패하면
+비교값·차이·방향을 모두 제거하고 현재 공식 소매 조사 평균 조회로 축소합니다.
+
+### C. 정적 월별 file path
+
+API 자동 운영이 불가능하고 공식 월별 소매가격 파일 `15087482`의 접근·권리·schema·
+공표본 identity·row별 기준 연월·series identity·단위가 통과하면 운영자 승인형 정적 월별
+탐색기로 축소할 수 있습니다. 이 path는 별도 `MonthlyRetailPriceSnapshot`, publication
+channel과 route를 사용합니다. 화면은 파일 공표본, 각 row의 기준 연월과 조사범위를
+명시하고 현재가·freshness·알림을 주장하지 않습니다.
+
+### D. stop
+
+조사범위, 평균 의미, identity, 등급·단위, 재공개 권리와 안전한 acquisition 중 하나라도
+증명하지 못하면 가격 공개와 dependent code를 중단합니다. HTML scraping, 비공식 미러,
+CAPTCHA·quota 우회와 다른 가격 source의 자동 보충은 pivot이 아닙니다.
+
+## 후속 월별 패턴 module
+
+연월별 API `15156060`과 기간별 소매가격 API `15156065`는 첫 MVP source가 아닙니다.
+동일 series의 최소 3개 완전한 연도, 코드·등급·단위·coverage 연속성, 정정·결측 의미,
+권리와 재현 가능한 월별 집계를 별도 gate로 증명한 뒤 전용 제품 결정 commit에서만
+활성화합니다. 통과해도 첫 표현은 `월별 과거 가격 패턴`이며 제철·품질·추천·예측이
+아닙니다.
+
+## 변경 승인
+
+고정 결정 변경에는 source evidence, 사용자 문구, data compatibility, privacy·license,
+schema·migration, acceptance와 rollback 영향을 기록합니다. 두 실제 vertical에서
+안정된 중복이 확인되기 전에는 공용 ingestion package를 추출하지 않습니다.
diff --git a/docs/SYSTEM-BOUNDARIES.md b/docs/SYSTEM-BOUNDARIES.md
new file mode 100644
index 0000000..13d53bf
--- /dev/null
+++ b/docs/SYSTEM-BOUNDARIES.md
@@ -0,0 +1,158 @@
+# 시스템 경계
+
+## 소유권과 신뢰 경계
+
+- aT와 공공데이터포털은 가격 source, code list, 조사·집계 의미, 인증, quota와
+  이용조건을 소유합니다.
+- 이 서비스는 source 후보를 수집·검증·공개할 책임을 지지만 원천 사실을 수정해
+  꾸며내거나 source authority를 대신하지 않습니다.
+- PostgreSQL의 append-only `PublicationActivation`이 현재로 가리키는 승인된
+  `PublicationRevision`만 공개 화면의 단일 진실 원천입니다.
+- 사용자 browser는 비신뢰 입력 경계이며 공개 결과를 변경할 권한이 없습니다.
+- Django Admin은 비공개 운영 경계이고 production에서 일반 사용자 경로와 분리하며
+  MFA와 최소권한을 요구합니다.
+
+## 외부 source 경계
+
+### 1. 최근일자 도·소매가격정보 API `15156063`
+
+첫 구현의 유일한 live 가격 후보입니다. 공공데이터포털 메타데이터는 REST JSON/XML,
+무료, 이용허락 제한 없음, 개발·운영 자동승인과 개발계정 10,000회를 안내합니다.
+그 설명은 조사일과 1일·1주·1개월·1년 전 평균 가격 field를 열거합니다. 이 문서
+checkpoint에서는 live key, 응답, production quota와 재배포 동작을 확보했다고
+주장하지 않습니다.
+
+gate는 다음 경계를 독립적으로 검증합니다.
+
+- key 발급·입력과 약관 판단: 사람 전용
+- HTTPS endpoint, method, query allowlist, timeout, redirect와 TLS
+- 실제 JSON media type·encoding·error envelope와 HTTP 200 내부 오류
+- pagination, total count, duplicate, quota·429와 재시도 조건
+- 소매·채소·과일 code, 품목·품종·등급·단위 identity와 조사범위
+- 현재·과거 값의 동일 series, 결측·0·sentinel·휴일·정정 의미
+- raw body 저장, 정규화·파생값 공개와 출처표시 권리
+
+공개 request path에서는 이 API를 호출하지 않습니다.
+
+### 2. 월별 소매가격 파일 `15087482`
+
+live API가 운영에 부적합할 때만 평가하는 공식 정적 fallback입니다. CSV의 공표본,
+row별 기준 연월, 도시·시장, 품목·품종·등급, 단위, 행 identity, 갱신주기와 권리를 별도
+gate로 검증합니다. file path가 통과해도 이를 현재가격, 실시간 또는 live API의 빈 구간을
+채우는 자료로 사용하지 않습니다. 별도 typed monthly model, publication channel, route와
+rollback을 사용합니다.
+
+### 3. 후속 API `15156060`과 `15156065`
+
+연월별·기간별 소매가격은 다년 월별 패턴을 위한 비활성 source 후보입니다. 첫
+MVP schema, ingestion schedule, 공개 UI와 acceptance에 포함하지 않습니다. 별도
+gate와 제품 결정 없이 이 데이터를 가져오거나 기존 profile에 섞지 않습니다.
+
+### 제외된 외부 경계
+
+- KAMIS HTML, 게시물·첨부 이미지와 로고 scraping
+- 검색엔진 cache, 비공식 mirror, 뉴스·블로그 가격
+- 쇼핑몰·마트·전통시장 크롤링과 제휴 feed
+- nutrition, recipe, weather, 생산량, 도매시장과 다른 price source
+- CAPTCHA, key, quota, robots 또는 이용조건 우회
+
+## 내부 경계
+
+첫 시스템은 하나의 Django 배포 단위와 하나의 PostgreSQL database인 modular monolith입니다.
+
+### ingestion 경계
+
+- platform cron이 제한된 management command를 실행합니다.
+- command는 source allowlist, timeout, pagination·row 상한과 제한 재시도를 사용합니다.
+- 전체 pagination을 완성하려는 논리적 획득마다 `FetchAttempt`를 만들고, 순서가 있는
+  page receipt에서 key·query secret을 제거합니다. 논리적 재시도는 새 attempt입니다.
+- raw bytes는 승인된 권리와 retention이 있을 때만 private artifact storage에 둡니다.
+- 모든 page와 row의 대사가 끝나기 전에는 `SourceArtifact`를 만들지 않습니다.
+
+### parsing·reconciliation 경계
+
+- parser는 exact source/interface revision에 연결됩니다.
+- 입력 artifact를 수정하지 않고 typed candidate와 deterministic result hash를 만듭니다.
+- 모든 source 행은 공개 가능, reference가 누락된 공개 가능, 범위 밖 또는 quarantine으로
+  대사합니다. 현재값 누락은 공개 가능 상태가 아닙니다.
+- 코드·이름 충돌, malformed Decimal, 중복 series, 단위 drift와 pagination 누락은
+  publication을 차단합니다. 첫 승인 generation은 전체 key set과 상태별 count의 대사
+  기준선이 되며 후속 key 소실·추가·차원 변경은 사람 검토를 요구합니다.
+
+### review·publication 경계
+
+- worker는 candidate를 만들 수 있지만 승인·공개하지 못합니다.
+- reviewer는 evidence와 reconciliation report로 generation 전체를 승인·거부합니다.
+- publisher만 승인 generation의 immutable publication을 대상으로 append-only
+  `ACTIVATE | ROLLBACK | WITHDRAW` 사건과 current pointer를 한 transaction으로 전환합니다.
+- source·parser rollback과 publication activation은 각각 별도 사건과 audit를 가집니다.
+
+### public read 경계
+
+- Django form은 부류와 공식 품목명 검색만 받습니다.
+- 검색 길이, 문자와 result 수를 제한하며 검색어를 공개 방문자 session, cache, analytics,
+  log와 audit에 남기지 않습니다. Django Admin의 보안 인증 session은 별도 운영 정책을
+  따릅니다.
+- 목록·상세는 published read model만 조회하고 외부 source, 운영 candidate와 raw
+  artifact에 접근하지 않습니다.
+- 공개 URL에는 stable internal slug만 사용하며 source key나 secret query를 넣지 않습니다.
+
+## 데이터 흐름
+
+```text
+platform cron
+  -> Django management command
+  -> KAMIS HTTPS source
+  -> FetchAttempt + redacted receipt
+  -> content-addressed SourceArtifact
+  -> versioned ParseRun + reconciliation
+  -> typed recent retail facts or typed monthly retail snapshots
+  -> ReviewDecision
+  -> immutable PublicationRevision
+  -> append-only PublicationActivation + atomic channel pointer
+  -> Django server-rendered list/detail
+```
+
+한 단계의 성공을 다음 단계의 성공으로 간주하지 않습니다. HTTP 200은 artifact 승인,
+parse 성공, reviewer 승인 또는 publication 성공을 뜻하지 않습니다.
+
+## 실패 경계
+
+- timeout, 429와 일시적 5xx만 bounded retry 대상입니다.
+- auth·rights 실패, unsupported schema, identity·unit·coverage 충돌은 terminal 또는
+  review-required 상태입니다.
+- 부분 page, 오래된 page와 새 page, 이전 generation의 결측 행을 섞지 않습니다.
+- 새 generation 실패는 activation을 만들거나 current pointer를 이동하지 않고
+  last-known-good와 stale 사유를 유지합니다.
+- source 신뢰 또는 공개 권리가 철회되면 오래된 publication을 안전하다고 가정하지
+  않고 `no current publication`으로 철회할 수 있습니다.
+
+## 개인정보 allowlist
+
+공개 가능 field는 공식 부류·품목·품종·등급 code와 표시명, 원문 단위·크기,
+검증된 조사범위, 조사일, 현재·1주·1개월·1년 제공값, 결정적 차이와 방향, source URL,
+publication, source 조사일과 마지막 확인 상태입니다.
+
+다음은 저장·로그·공개하지 않습니다.
+
+- API key, secret, 전체 query string과 gateway trace
+- 방문자의 IP, User-Agent, 검색어, 클릭 이력과 관심 profile
+- 운영자 email·이름 등 공개에 필요 없는 identity
+- source response의 allowlist 밖 field
+- 광고·remarketing identifier와 정밀 위치
+
+보안 운영에 필요한 request log도 URL query를 제거하고 짧은 정책 retention을 사용합니다.
+
+## 운영 경계
+
+production 노출 전 HTTPS, secure settings, 관리자 MFA, managed secret injection,
+구조화 로그, health·readiness·source freshness 경보, 자동 PostgreSQL backup과 실제 restore
+rehearsal을 통과해야 합니다. source credential, database credential, artifact storage와
+publisher 권한은 서로 분리하고 최소권한을 사용합니다.
+
+## Legacy와 portability
+
+기존 repository, 경쟁 서비스 DB, 수집 이력과 다른 Audience Foundry artifact를 이
+서비스의 관측 이력으로 가져오지 않습니다. 검증된 오픈소스 dependency는 exact version,
+license, integrity와 purpose를 승인한 격리 commit에서만 도입합니다. 범용 플랫폼을
+만들기보다 이 도메인의 typed model과 Django 기본 기능을 우선합니다.
diff --git a/docs/TECHNOLOGY-DECISIONS.md b/docs/TECHNOLOGY-DECISIONS.md
new file mode 100644
index 0000000..79e7761
--- /dev/null
+++ b/docs/TECHNOLOGY-DECISIONS.md
@@ -0,0 +1,163 @@
+# 기술 결정
+
+## 문서 상태
+
+이 문서는 첫 구현이 따라야 할 기술 기준선이다. 현재 저장소에는 런타임 코드, 의존성,
+잠금 파일, 수집 데이터와 배포 구성이 없다. 아래 버전과 구조는 source gate를 통과한 뒤
+구현 계획에서 실제 호환성을 다시 증명하고 도입한다.
+
+## 고정 기준 스택
+
+| 구성 요소 | 기준 버전 | 용도 |
+|---|---:|---|
+| Python | `3.14.7` | 애플리케이션, 수집, 파싱 런타임 |
+| Django | `5.2.17` | Templates, Forms, Admin, Auth, ORM, 관리 명령 |
+| PostgreSQL | `18.6` | 출처, 감사, typed 가격 사실, 승인 공개 리비전 |
+| uv | `0.12.6` | Python 버전, 의존성, 잠금, 명령 실행 |
+
+구현 시작 시 공식 배포물, 배포 플랫폼 지원, 보안 공지와 패키지 호환성을 확인한다.
+하나라도 재현할 수 없으면 조용히 다른 버전을 선택하지 않고 근거와 migration·rollback
+영향을 기술 결정으로 승인받는다. 도입한 직접·전이 의존성은 해시가 포함된 잠금 파일로
+고정한다.
+
+## 개발 원칙 적용
+
+- **Open Source and Standards First**: Python, Django, PostgreSQL, uv와 HTTP·TLS·JSON·
+  SHA-256 표준을 우선 사용하고 제품 고유 코드에는 source 계약, typed 변환, 대사와 공개
+  표현 규칙만 남긴다.
+- **Less Code, Less Complexity**: 서버 렌더링 Django 애플리케이션 하나, PostgreSQL 하나,
+  관리 명령과 platform cron 하나로 시작한다.
+- **Production Quality from Day One**: 출처 감사, DB 제약, 실패 폐쇄, 테스트, 최소 권한,
+  관측성, 백업·복원과 publication rollback을 첫 공개 조건으로 둔다.
+- **Small, Reversible Commits**: live source 증거, schema, adapter, parser, review, publication,
+  UI와 운영 변경을 각각 독립적으로 검증하고 되돌릴 수 있게 나눈다.
+- **Prove Risky Assumptions Before Build**: 실제 응답의 권리, 인증, quota, schema, 코드
+  identity, 단위, 기준일과 비교 기간 의미가 승인되기 전에는 그 결과에 의존하는 schema나
+  adapter를 만들지 않는다.
+
+전체 모토는 `Build less. Reuse more. Ship solid. Change safely.`이다.
+
+## 애플리케이션 구조
+
+하나의 Django modular monolith 안에서 다음 책임만 분리한다. 실제 package 이름은 구현
+계획에서 이 경계를 가장 적은 코드로 표현하도록 정한다.
+
+- **sources**: source configuration, 요청 영수증, content-addressed artifact와 권리 결정
+- **prices**: 결정적 파싱, typed identity, recent·monthly snapshot, reference price와 대사
+- **publication**: review decision, 불변 revision, append-only activation과 channel별
+  last-known-good pointer
+- **public**: 품목 목록·검색, exact series 상세와 중립적 가격 변화 표현
+- **operations**: Admin 권한, 관리 명령, source freshness, health와 감사 조회
+
+Django ORM과 migration이 schema를 소유한다. public read는 현재 activation이 가리키는
+승인된 `PublicationRevision`만 PostgreSQL에서 읽는다. 웹 요청 안에서 외부 API를 호출하거나
+긴 수집·파싱을 실행하지 않는다. 수집은 중복 실행에 안전한 관리 명령으로 수행하고 platform
+cron이 호출한다.
+
+## 서버 렌더링 UI
+
+사용자 화면은 Django Templates와 Forms로 구현한다. 품목 목록, 공식 품목명 검색, 한
+`PriceSeriesKey`의 상세 화면만 첫 navigation에 둔다. 핵심 읽기와 검색은 JavaScript 없이도
+동작해야 하며 다음을 만족한다.
+
+- semantic HTML, 명시적 label, 논리적인 heading과 keyboard focus를 사용한다.
+- 색만으로 상승·하락·동일·비교 불가를 구분하지 않고 텍스트와 기호를 함께 제공한다.
+- recent source 조사일 또는 monthly row 기준 연월, coverage, 판매 단위, grade와 검토일을
+  가격 가까이에 표시한다.
+- 값이 없거나 비교할 수 없으면 빈 숫자나 `0원` 대신 `비교 정보 없음`을 표시한다.
+- 광고나 analytics SDK는 첫 MVP에 넣지 않는다.
+
+CSS와 필수 정적 자산은 Django static files로 제공한다. 외부 CDN JavaScript와 외부 font가
+핵심 기능, 개인정보 경계 또는 가용성의 단일 실패점이 되지 않게 한다.
+
+## 데이터베이스와 타입 규칙
+
+- 범용 EAV와 schema-less domain JSON 대신 닫힌 enum, 외래 키, `DecimalField`, 날짜 필드와
+  명시적 DB 제약을 사용한다.
+- `PriceSeriesKey`의 모든 차원을 unique constraint에 포함하고 표시 이름을 identity로 쓰지
+  않는다.
+- 가격은 source gate가 증명한 Decimal scale로 원문 정밀도를 보존하고 float를 사용하지
+  않는다. 화면 반올림·원화 표시 규칙은 실제 scale 확인 후 승인한다. 퍼센트는 저장된
+  float가 아니라 현재값과 기준값으로 계산하고 `Decimal`의 `ROUND_HALF_UP`으로 소수 첫째
+  자리까지 낸다.
+- recent row의 `source_effective_date: LocalDate`와 monthly row의
+  `source_effective_month: YearMonth`를 분리한다. `source_recorded_at`, `fetched_at`,
+  `revision_created_at`, `activated_at`도 서로 다른 필드로 유지한다. 월을 임의의 첫날로
+  바꾸거나 source가 주지 않은 날짜·시각을 다른 값으로 채우지 않는다.
+- 현재 channel pointer 전환과 append-only activation 사건은 한 transaction에서 이루어진다.
+  이전 승인 revision은 상태를 바꾸지 않는 불변 내용으로 남긴다.
+- source·module별 current pointer를 분리해 한 source 실패가 다른 승인 공개본을 바꾸지 않게
+  한다.
+
+## HTTP 수집과 파싱 규칙
+
+- source gate가 승인한 공식 HTTPS host, path, method와 query parameter만 허용한다.
+- API key는 배포 secret store에서 주입하며 URL, 로그, 오류, artifact와 Git에 기록하지
+  않는다.
+- 연결·읽기 timeout, 최대 response 크기, redirect, content type, encoding, pagination과
+  호출 예산을 명시한다. `429`와 일시적 `5xx`만 제한된 backoff 대상으로 분류한다.
+- HTTP `200`이어도 provider error code·message가 있으면 성공으로 처리하지 않는다.
+- raw bytes는 권리가 명시적으로 허용할 때만 격리 저장한다. 그렇지 않으면 body hash,
+  byte length, 최소 response receipt와 정규화 사실만 남긴다.
+- parser는 network, 현재 시각과 mutable global state를 참조하지 않는 결정적 함수로 만든다.
+- 알 수 없는 코드, 누락 필드, sentinel, 잘못된 숫자, 단위 변경, 중복 identity와 coverage
+  급변은 추정 보정하지 않고 격리한다.
+- 같은 artifact와 parser version을 다시 처리하면 typed row 집합 hash가 같아야 한다.
+  `fetched_at`은 idempotency key에서 제외한다.
+
+표준 라이브러리와 Django 내장 기능을 먼저 사용한다. 외부 HTTP·retry·parser library는 실제
+source 증거가 내장 기능으로 안전하게 충족되지 않음을 보인 뒤 최소 하나만 선택하고 버전과
+license를 고정한다.
+
+## 테스트와 품질 게이트
+
+모든 구현 변경은 영향 범위에 맞춰 다음 검사를 자동화한다.
+
+- 잠금 상태의 깨끗한 설치와 Python·Django 실제 버전 확인
+- formatter, linter, static type checker와 migration 누락 검사
+- enum·identity·단위·날짜·`Decimal` 계산의 단위 테스트
+- 승인된 최소 artifact에 대한 parser golden test와 동일 artifact replay test
+- fetch 실패, provider error, schema drift, missing value, duplicate row와 승인 key set 변경 테스트
+- review 전 비공개, transaction 실패, 동시 publication과 last-known-good 통합 테스트
+- Forms·route·template의 양성·음성 경로, HTML 유효성, keyboard와 screen reader 점검
+- `django check`와 production 설정의 `django check --deploy`
+- secret scan, dependency vulnerability·license 검사와 clean Git tree 확인
+
+live source gate가 실패한 것을 합성 fixture의 성공으로 대체하지 않는다. 원시 데이터 보존
+권리가 없으면 승인된 schema와 경계 사례를 재현한 최소 합성 fixture만 Git에 둘 수 있으며,
+그 fixture는 source 접근·권리·운영 가능성의 증거가 아니다.
+
+## 의도적으로 제외한 기술
+
+첫 구현에는 Node.js, Astro, SPA, 별도 public JSON API, GraphQL, Redis, Celery, Kafka,
+OpenSearch, PostGIS, data warehouse, Kubernetes와 microservice를 도입하지 않는다. 사용자
+위치와 지도 기능이 없으므로 GPS·geospatial schema도 없다. 측정된 운영 병목과 승인된 새
+요구가 생기기 전에는 cache server나 queue를 추가하지 않는다.
+
+## 보안과 운영 게이트
+
+- 운영은 `DEBUG=False`, 정확한 `ALLOWED_HOSTS`·CSRF origin, HTTPS, HSTS, secure cookie와
+  보안 header를 사용한다.
+- Admin은 public route와 분리하고 운영자별 최소 권한, MFA 또는 동등한 강한 인증, 로그인
+  rate limit을 적용한다.
+- application, migration, backup DB 역할을 분리하고 secret은 환경별 secret store에서
+  주입·회전한다.
+- 구조화 로그에는 request ID, deploy version, command run ID와 lifecycle 내부 ID·상태만
+  남긴다. API key, query string, response body, 검색어와 사용자 식별자는 기록하지 않는다.
+- liveness는 process 응답, readiness는 DB와 승인 publication read를 검사한다. source
+  freshness와 last-known-good 나이는 readiness와 분리해 경보한다.
+- 연속 fetch 실패, parse quarantine 증가, schema·coverage 대사 실패, publication transaction
+  실패, backup 실패에 운영자 alert를 연결한다.
+
+공개 전 PostgreSQL 암호화 backup과 point-in-time recovery를 구성한다. 목표는 `RPO 24시간`,
+`RTO 4시간`이며 빈 환경 복원으로 승인 revision, audit chain, row count·hash와 Admin/public
+read를 검증한다. 분기마다 복원 훈련을 반복한다.
+
+## 배포와 rollback
+
+개발 server가 아닌 고정 WSGI process와 관리형 HTTPS 경계를 사용한다. 배포는 backup 확인,
+호환 migration, static assets, application 전환, health와 핵심 공개 읽기 순으로 한다.
+application version과 승인 `PublicationRevision`은 서로 독립적으로 이전 상태로 되돌릴 수
+있어야 한다. publication rollback은 이전 revision을 수정하지 않고 새 `ROLLBACK` activation을
+추가한다. 파괴적 migration은 expand·migrate·contract 단계로 나누며 복원 검증 전에 contract
+단계를 실행하지 않는다. key 발급·약관 동의·결제·배포·첫 publication은 사람 승인 지점이다.


