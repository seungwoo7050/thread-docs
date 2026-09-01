# 개발자 기술면접 워크북 마스터 인덱스

이 문서는 현재 GPT 프로젝트에 축적된 DevThread 01–22 기록에서 실제로 확인되는 내용만 사용한 선별 기준표다. 확인되지 않는 커밋 제목·파일·함수는 이름을 만들지 않고 “현재 기록에서 확인되지 않음”으로 표시했다.

상세 문서를 작성한 뒤 이 인덱스의 S/A 행과 다시 대조했다. 같은 역량이 여러 Thread에 반복된 경우 표의 `[Pxx]`가 대표 상세 항목이며, 관련 행은 그 항목에 명시적으로 통합된다.

## 우선순위 기준

- **S:** 질문과 10–30분 직접 구현 모두 가치가 매우 높고 반드시 준비할 항목
- **A:** 질문 가치가 높고 핵심 구현 또는 설계 구현으로 출제될 가능성이 높은 항목
- **B:** 별도 백지 구현보다 설계·개념·trade-off 설명 준비가 적절한 항목
- **C:** boilerplate·표현 조정·프로젝트 특수성이 커서 별도 면접 준비 항목으로 만들 필요가 낮은 항목

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| B | 01 | `docs: define phase zero release gate` | `docs/IMPLEMENTATION-PLAN.md`, `docs/MVP-ACCEPTANCE.md` | 계약 우선 범위 설정과 acceptance·rollback gate | 코드보다 범위·증거·사람 checkpoint를 먼저 고정한 설계 판단은 중요하지만 독립 구현 문제보다는 설명형에 가깝다. | 높음 | 낮음 | 21, 22 · P32 |
| B | 01 | `build: pin django runtime` | `pyproject.toml`, `uv.lock`, `THIRD_PARTY_NOTICES.md` | runtime·dependency·container provenance 고정 | 재현 가능한 release와 supply-chain evidence를 설명할 가치는 높지만 version 나열 자체는 면접 구현 가치가 낮다. | 중간 | 낮음 | 22 · P32 |
| S | 02 | `feat(price): define decimal comparisons` | `grocery/pricing.py`; `ReferencePrice`, `PriceSnapshot`, `PriceComparison`, `compare_snapshot` | [P01] Decimal 산술, HALF_UP 반올림, available/unavailable 불변식 | 금액 계산·상태 모델·경계값을 한 번에 검증하며 20분 내 직접 구현으로 기본기를 확인하기 좋다. | 높음 | 높음 | 04, 06, 08 |
| A | 02 | `feat(price): identify exact retail series` | `grocery/models.py`, `grocery/migrations/0003_price_series_key.py`; `PriceSeriesKey` | [P02] 문자열 코드 기반 composite identity와 변경 불가성 | leading zero 보존, semantic equality, hash identity는 다른 데이터 통합 시스템에도 그대로 일반화된다. | 높음 | 중간 | 04, 07, 08 |
| A | 02 | `feat(price): store current retail snapshots` | `grocery/models.py`, `grocery/migrations/0004_retail_price_snapshot.py`; `RetailPriceSnapshot.get_or_validate` | [P03] immutable snapshot의 idempotent create-or-validate | 재시도와 충돌을 구분하는 전형적인 데이터 정합성 문제이며 ORM·DB 양쪽 invariant를 설명할 수 있다. | 높음 | 높음 | 04, 05 |
| S | 02 | `feat(price): derive typed reference changes` | `grocery/models.py`; `ReferencePrice`, `PriceChangeFact`, `persist_reference_price_facts` | [P01] XOR 상태, 부호·방향 일치, 원자적 비교 사실 생성 | 불가능 상태를 타입과 DB 제약으로 배제하고 결정적 산술을 보장하는 핵심 구현이다. | 높음 | 높음 | 04, 06, 08 |
| A | 03 | `feat(audit): record source fetch attempts` | `grocery/models.py`; `SourceConfiguration`, `FetchAttempt`, `PageReceipt` | [P07] 수집 attempt/page receipt 상태 머신과 budget invariant | 외부 I/O를 감사 가능한 상태 전이로 바꾸는 기초 모델이며 실패 증거 보존의 출발점이다. | 높음 | 중간 | 04, 09 |
| S | 03 | `feat(source): persist reconciled fetch receipts` | `grocery/source/persistence.py`; `start_kamis_fetch`, `complete_kamis_fetch`, `fail_kamis_fetch` | [P07] 성공 완료의 row·page·byte·manifest 원자 대사 | 네트워크 결과와 영속 상태를 정확히 맞추는 트랜잭션·idempotency 문제로 직접 구현 가치가 높다. | 높음 | 높음 | 04, 09, 10 |
| S | 03 | `feat(source): retain partial failure receipts` | `grocery/source/client.py`, `grocery/source/persistence.py`; `KamisTransportError`, `partial_page_receipts` | [P08] 부분 실패 증거 보존, double-finalize 방지, 오류 redaction | 성공보다 까다로운 실패 경로에서 원자성·감사성·비밀 보호를 동시에 다룬다. | 높음 | 높음 | 19, 21 |
| A | 03 | `feat(source): share bounded transport for history` | `grocery/source/client.py`; `KamisHttpClient.fetch_historical_prices`, `_fetch_prices` | [P12] bounded pagination, retry 상한, 공통 transport 추상화 | 무한 pagination·retry 폭주·메모리 증가를 막는 네트워크 기본기와 추상화 trade-off를 묻기 좋다. | 높음 | 높음 | 07, 20 |
| A | 03 | `feat(source): persist historical request scopes` | `grocery/source/historical_persistence.py`; `start_historical_fetch`, `PreparedHistoricalRequest.scope_sha256` | [P09] secret-free canonical request와 semantic scope hash | 민감 입력을 저장하지 않으면서 동일 요청을 식별하는 보안·idempotency 경계다. | 높음 | 중간 | 07, 09 |
| S | 04 | `feat(source): parse kamis recent rows` | 최근 가격 parser; `parse_recent_price_rows`, `ExactIdentityRegistry`, `KamisParseError` | [P10] exact schema, semantic duplicate, 결정적 정렬·hash | strict parser는 자료구조·검증 순서·복잡도·오류 경계를 모두 확인할 수 있는 대표 백지 구현이다. | 높음 | 높음 | 07 |
| A | 04 | `feat(source): seal reviewed series allowlist` | `grocery/source/registry.py`; `INITIAL_RETAIL_IDENTITY_REGISTRY`, `OFFICIAL_DOCS_ZIP_SHA256` | [P11] 검토된 code-name-unit registry와 source drift 거부 | 외부 문자열을 추론하지 않고 evidence-bound allowlist로 검증하는 데이터 계약 지식이 일반화된다. | 높음 | 중간 | 07, 08 |
| A | 04 | `feat(audit): reconcile deterministic parses` | `grocery/models.py`; `SourceArtifact`, `ParseRun`, `build_source_artifact` | [P13] hash-only artifact, parse 상태 머신, count reconciliation | 원문 미보관 환경에서도 재현성과 nondeterminism을 검출하는 감사 모델로 설계 질문 가치가 높다. | 높음 | 높음 | 03, 10 |
| A | 05 | `feat(review): append generation decisions` | `grocery/models.py`; `ReviewDecision` | [P18] append-only review chain과 현재 tail | 승인 결과를 mutable flag가 아니라 감사 가능한 chain으로 표현하는 설계가 일반화된다. | 높음 | 중간 | 11, 12 |
| A | 05 | `feat(publication): seal immutable revisions` | `grocery/models.py`; `PublicationRevision`, publication entry membership | [P19] canonical membership과 immutable fact-set hash 봉인 | 검수 결과와 공개 단위를 분리하고 동일 사실 집합을 결정적으로 식별하는 핵심 경계다. | 높음 | 높음 | 11 |
| S | 05 | `feat(publication): activate revisions atomically` | `grocery/models.py`; `PublicationChannel`, `PublicationActivation`, `transition_recent_publication` | [P20] expected-version/current CAS, idempotent operation, event-pointer 원자성 | 동시성·트랜잭션·복구를 한 문제로 묶을 수 있어 질문과 직접 구현 모두 최상위다. | 높음 | 높음 | 11, 12, 13 |
| A | 06 | `test(public): enforce bounded publication reads` | `grocery/public_read.py`; `load_active_publication`, `publication_entries`, `publication_entries_for_series` | [P24] active-only membership read와 row-independent query count | 공개 authorization 경계와 ORM N+1 방지를 함께 확인할 수 있다. | 높음 | 중간 | 13, 16, 17 |
| A | 06 | `feat(public): connect recent catalog extensions` | `grocery/views.py`, `grocery/public_read.py`; `catalog`, `detail`, selection helpers | [P24] bounded filter·selection을 published read model에 연결 | 사용자 입력을 publication membership과 결합하는 서비스 경계가 중요하며 P24에 통합한다. | 높음 | 중간 | 13, 15 |
| B | 06 | `feat(public): render published prices` | `grocery/views.py`, `grocery/templates/grocery/` | SSR 상태 표현과 provenance 노출 | published data만 렌더링하는 원칙은 중요하지만 template wiring 자체는 별도 구현 문제 가치가 낮다. | 중간 | 낮음 | 18 |
| A | 07 | `feat(source): build redacted historical requests` | `grocery/source/historical_client.py`; `PreparedHistoricalRequest`, `prepare_historical_request` | [P09] endpoint allowlist와 secret 없는 request 표현 | request 생성 단계에서 host/path/auth/query 계약을 고정하는 SSRF·credential 경계로 P09에 통합한다. | 높음 | 중간 | 03, 09 |
| S | 07 | `feat(source): validate historical row primitives` | `grocery/source/historical_parser.py`; `HistoricalRowValidator`, `ParsedHistoricalResult` | [P10] shared strict validator와 redacted parse failure | 최근 parser와 같은 핵심 역량을 더 다양한 date·Decimal·dimension 경계에서 검증하므로 P10에 통합한다. | 높음 | 높음 | 04 |
| A | 07 | historical dimension·month typing 커밋(제목은 현재 기록에서 확인되지 않음) | historical dimension registry; `HistoricalDimensionRegistry`, `YearMonth`, `RegionObservation`, `MarketObservation` | [P11] immutable reviewed dimensions와 “월에 임의의 일자를 만들지 않기” | 도메인 타입이 정보 손실과 추론을 막는 사례로 registry 문제 P11에 통합한다. | 높음 | 중간 | 04, 08 |
| S | 07 | `feat(source): parse market daily history` 및 월별·지역별 parser | `parse_monthly_price_rows`, `parse_regional_price_rows`, `parse_market_price_rows` | [P10] source별 의미 키·가격 범위·결정적 결과 hash | 세 parser가 같은 strict 원리를 공유하므로 가장 대표적인 parser 문제 P10 하나로 통합한다. | 높음 | 높음 | 04, 17 |
| A | 07 | `fix(source): bind historical datasets to modes`; `fix(source): pin historical endpoints` | `grocery/models.py`; `SourceConfiguration` DB constraints, `HISTORICAL_ENDPOINT_CONTRACTS` | [P09] dataset-mode-endpoint-auth 조합의 fail-closed 계약 | 설정 drift를 application validation뿐 아니라 DB에서 막는 경계로 request 문제 P09에 통합한다. | 높음 | 중간 | 03, 09 |
| A | 08 | `feat(history): identify exact retail sources` | `grocery/historical_identity_models.py`; `HistoricalRetailSeriesKey`, `RetailRegionKey`, `RetailMarketKey` | [P02] recent-historical 교차 소스 identity와 leading zero 보존 | 다른 coverage를 제외하면서도 동일 상품을 연결하는 의미 식별자 설계로 P02에 통합한다. | 높음 | 중간 | 02, 07 |
| A | 08 | historical typed fact 도입 커밋(제목은 현재 기록에서 확인되지 않음) | `MonthlyRegionalRetailPrice`, `DailyRegionalRetailPrice`, `DailyMarketRetailPrice` | [P04] source 역할별 typed fact와 low-mean-high invariant | 월별 범위·지역별 범위·시장 관측을 섞지 않는 정합성 설계가 직접 구현과 설명에 모두 적합하다. | 높음 | 높음 | 07, 16, 17 |
| A | 08 | historical price precision 보강 커밋(제목은 현재 기록에서 확인되지 않음) | typed fact Decimal fields와 precision tests | [P04] provider 소수 정밀도 보존 | parser와 DB precision의 불일치를 막는 경계값 문제로 P04에 통합한다. | 높음 | 중간 | 07 |
| A | 09 | `feat(source): scope historical acquisitions` | `FetchAttempt.request_scope_sha256`; historical publication modes | [P09] partition request scope를 attempt identity에 포함 | 같은 endpoint의 다른 기간·지역 요청을 혼동하지 않는 idempotency 경계다. | 높음 | 중간 | 03, 07 |
| A | 09 | `feat(history): plan ordered source partitions` | `grocery/historical_collection_plans.py`; `plan_historical_collection` | [P14] deterministic partition 순서와 manifest | 대규모 수집을 bounded part로 나누면서 누락·중복을 식별하는 알고리즘 문제다. | 높음 | 높음 | 10 |
| A | 09 | `feat(history): persist regional collection parts` | `grocery/historical_daily_generation.py`; `persist_regional_part`, `_validate_regional_scope` | [P15] part provenance·scope·fact count의 idempotent 저장 | 부분 성공을 전역 완료와 분리하고 replay conflict를 검출하는 전형적인 ingestion 문제다. | 높음 | 높음 | 10 |
| A | 09 | `feat(history): persist market collection parts` | `grocery/historical_daily_generation.py`; `persist_market_part`, `resolve_historical_market` | [P15] region-market 관계를 포함한 part membership 검증 | regional part와 같은 역량에 market-region invariant가 추가되므로 P15에 통합한다. | 높음 | 높음 | 08, 10, 17 |
| S | 10 | `feat(history): reconcile collection parts` | `grocery/historical_collections.py`; `partition_manifest_sha256`, `complete_historical_collection` | [P16] planned part·parse·typed fact 전역 대사와 deterministic result hash | 분할 작업의 완전성, 누락·중복, 트랜잭션 완료를 직접 구현하는 최상위 문제다. | 높음 | 높음 | 09, 11 |
| S | 10 | DB membership guard 보강 커밋(제목은 현재 기록에서 확인되지 않음) | `grocery/migrations/0022_guard_historical_collection_membership.py`; collection/fact guard triggers | [P17] STARTED-only insert, append-only fact, kind·window·market-region guard | ORM 우회 write와 insert-versus-finalize 경쟁을 DB에서 막는 동시성 핵심 지점이다. | 높음 | 높음 | 09, 11 |
| S | 10 | collection write 직렬화 보강 커밋(제목은 현재 기록에서 확인되지 않음) | collection completion과 part/fact insert concurrency tests | [P17] completion-versus-insert serialization과 lock ordering | 애플리케이션 lock만으로 부족한 phantom·race를 설명하고 구현하도록 요구할 가치가 높다. | 높음 | 높음 | 09, 11 |
| A | 11 | `feat(history): bind reviews to collections` | historical review models; `HistoricalCollectionReviewDecision` | [P18] collection hash에 고정된 append-only 승인 chain | 검수 대상의 exact result/partition manifest를 승인에 묶는 감사 모델이다. | 높음 | 중간 | 05, 12 |
| A | 11 | `fix(history): authorize review decisions` | `grocery/historical_reviews.py`; `record_historical_review_decision`; DB write guard | [P18] service-only write, current-tail supersession, idempotent decision UUID | 권한·트랜잭션·append-only chain을 결합하므로 P18에 통합한다. | 높음 | 높음 | 05, 12 |
| A | 11 | `feat(history): define publication bundles`; `fix(history): seal only exact reviewed bundles` | `grocery/historical_publication_models.py`; `HistoricalRetailPublicationRevision`, `seal_historical_publication` | [P19] 서로 다른 세 current APPROVE review의 exact bundle 봉인 | review와 publication 사이의 compatibility·membership invariant를 검증하는 핵심 문제다. | 높음 | 높음 | 05 |
| S | 11 | historical activation CAS 보강 커밋(제목은 현재 기록에서 확인되지 않음) | `transition_historical_publication`, `grocery/migrations/0026_guard_historical_activation_cas.py` | [P20] historical channel의 event-pointer CAS와 DB guard | recent와 동일한 핵심 역량이므로 대표 CAS 문제 P20으로 통합한다. | 높음 | 높음 | 05, 13 |
| A | 11 | `fix(history): preserve last-known-good rollback` | historical transition eligibility and activation history | [P21] ACTIVATE current-approval 규칙과 ROLLBACK previously-current 규칙 분리 | 안전한 복구에서 현재 정책과 과거 정상본 자격이 다르다는 trade-off를 잘 드러낸다. | 높음 | 중간 | 05, 22 |
| A | 12 | production control plane 도입 커밋(제목은 현재 기록에서 확인되지 않음) | `grocery/management/control_plane.py`; `resolve_operation_actor`, `bootstrap_control_plane_actors` | [P22] reviewer/publisher 분리, exact permission, release SHA lock | 기능 flag를 인증으로 오해하지 않고 외부 MFA/IAM·DB 권한을 별도 경계로 둔 설계가 중요하다. | 높음 | 중간 | 05, 11, 22 |
| A | 12 | `feat(history): expose collection approval command` 및 historical control commands | `grocery/management/commands/approve_historical_collection.py` 등 | [P22] bounded command input·redacted receipt·role-specific operation | 명령 경계를 안전한 orchestration으로 만드는 패턴이 동일하므로 P22에 통합한다. | 높음 | 중간 | 11 |
| B | 12 | local Phase 0 actor 보강 커밋(제목은 현재 기록에서 확인되지 않음) | `grocery/management/local_phase0.py`; local operator bootstrap | local rehearsal용 non-login 최소 권한 actor | production control plane과 대비해 설명할 가치는 있으나 별도 구현 문제는 P22와 중복된다. | 중간 | 낮음 | 22 · P22 |
| A | 13 | `feat(history): guard publication activation state` | historical activation models, triggers, transition service | [P23] recent와 historical의 독립 current pointer·event log | 한 채널의 장애나 withdraw가 다른 채널 공개를 막지 않게 하는 failure isolation 설계다. | 높음 | 중간 | 05, 11, 16 |
| A | 13 | `feat(public): connect recent catalog extensions` | `grocery/historical_public_read.py`, `grocery/views.py`; `load_active_historical_publication`, `historical_series_for_recent` | [P23] historical read 실패 시 recent detail을 유지하는 부분 기능 저하 | 독립 source/read 모델을 한 화면에서 결합할 때의 예외 격리를 구현·설명하기 좋다. | 높음 | 높음 | 06, 16, 17 |
| B | 13 | vNext product/source 역할 계약 커밋(제목은 현재 기록에서 확인되지 않음) | 제품 계약 문서와 source 역할 표 | source별 정본과 금지된 fallback·재계산 | 시스템 경계를 설명하는 데 중요하지만 계약 문서 자체를 별도 백지 구현으로 만들 필요는 낮다. | 높음 | 낮음 | 07, 08 |
| A | 14 | `fix(public): keep searches private and uncached` | `grocery/security.py`, `grocery/views.py`; `Cache-Control: no-store`, `@require_safe` | [P26] query가 있는 공개 GET의 cache·method privacy | 오류·성공 응답을 포함한 HTTP 보안 경계와 raw value 비반사를 확인할 수 있다. | 높음 | 중간 | 15, 19 |
| A | 14 | `fix(security): suppress public query referrers` | `grocery/security.py`; `Referrer-Policy: no-referrer` | [P26] 외부 navigation으로 query 유출 차단 | URL 상태 설계가 browser referrer와 만나는 실제 보안 trade-off로 P26에 통합한다. | 높음 | 낮음 | 15, 21 |
| A | 14 | `fix(security): fail closed on production settings` | `config/settings.py`; `validate_production_environment` | [P32] production setting allowlist와 supplied value 비반사 | 배포 전제의 누락을 startup에서 거부하는 release gate이므로 P32에 통합한다. | 높음 | 중간 | 19, 22 |
| A | 15 | `test(public): lock URL and geometry contracts` | `grocery/tests/test_vnext_public_read_contract.py`; `CatalogForm`, `HistoryForm`, `MarketsForm`, `RegionsForm`, `parse_selection_query` | [P25] unknown·duplicate·noncanonical query 거부와 최대 5개 no-JS selection | MultiDict·canonical serialization·privacy·stateless state를 직접 구현할 수 있는 좋은 문제다. | 높음 | 높음 | 14, 18 |
| B | 15 | `test(public): lock URL and geometry contracts`의 chart/range 부분 | `grocery/vnext_presentation.py`; chart/range geometry contracts | 표현 geometry의 deterministic contract | 결측 구간 문제는 P05에 포함되며 픽셀·표현 세부는 별도 면접 항목 가치가 낮다. | 중간 | 낮음 | 16 · P05 |
| A | 16 | `feat(public): validate active historical bundle` | `grocery/historical_public_read.py`; `load_active_historical_publication`, `ActiveHistoricalPublication` | [P24] active sealed historical bundle의 review·hash·freshness 무결성 | active-only published read 원칙을 historical bundle에 적용하므로 P24에 통합한다. | 높음 | 중간 | 06, 13 |
| A | 16 | `feat(public): build complete monthly history context` | `grocery/historical_history_read.py`; `history_context`, `_has_complete_months`, `_select_complete_months` | [P05] 12·36·60개월 완전 구간과 gap-aware chart | 정렬·중복·연속성 알고리즘과 제품 의미를 함께 확인할 수 있다. | 높음 | 높음 | 08, 15 |
| B | 16 | `feat(frontend): add monthly price history` | `grocery/templates/grocery/`, frontend presentation | history UI와 summary·year grouping | 표현과 접근성은 설명할 가치가 있지만 핵심 알고리즘은 P05에서 이미 다룬다. | 중간 | 낮음 | 18 · P05 |
| A | 17 | `feat(public): build regional and market ledgers` | `grocery/historical_daily_read.py`; `regions_context`, `markets_context`, `HISTORICAL_PAGE_SIZE` | [P06] 동일 조사일·지역·시장 경계와 stable pagination | 필터 정확성, 정렬 tie-breaker, 페이지 안정성을 20분 문제로 만들 수 있다. | 높음 | 높음 | 08, 16 |
| B | 17 | `feat(public): expose regional and market routes` | `grocery/daily_views.py` | typed form state를 ledger context에 연결하는 SSR route | route wiring보다 underlying ledger invariant P06이 면접 가치가 높다. | 중간 | 낮음 | 15, 18 · P06 |
| B | 18 | `test(web): verify responsive browser flows` | template tests, browser flows, `grocery/templates/grocery/` | semantic HTML, keyboard flow, long Korean identity, no overflow | 접근성·상태 전달·responsive 검증은 설명형으로 중요하지만 별도 알고리즘 구현은 적합하지 않다. | 높음 | 낮음 | 15, 21 |
| B | 18 | error-state·form association 보강 커밋(제목은 현재 기록에서 확인되지 않음) | form error templates and tests; `aria-invalid`, `aria-describedby`, status/alert roles | validation·empty·stale·server error의 의미 구분 | UI 상태 모델의 경계 조건을 질문할 수 있으나 P25·P26과 중복되는 구현은 만들지 않는다. | 중간 | 낮음 | 14, 15 |
| C | 18 | visual ledger·font·spacing 보강 커밋들 | CSS, font notice, visual-only template refinements | 시각 표현과 third-party font 고지 | 프로젝트 품질에는 필요하지만 개발 기본기를 독립적으로 판별하는 면접 항목으로는 특수성이 높다. | 낮음 | 낮음 | 01, 22 |
| A | 19 | `feat(ops): add redacted structured events` | `grocery/observability.py`; `log_event`, `RequestIdMiddleware`, allowlist formatter | [P27] record.msg를 신뢰하지 않는 allowlist JSON logging과 ContextVar reset | 관측성과 개인정보 보호, request lifecycle cleanup을 한 번에 검증할 수 있다. | 높음 | 높음 | 14, 22 |
| A | 19 | `feat(ops): wire safe request logging` | `config/settings.py`; logging configuration and propagation rules | [P27] framework default request log 우회 차단과 단일 redacted pipeline | formatter가 안전해도 다른 logger가 원문을 내보낼 수 있다는 시스템 경계를 P27에 통합한다. | 높음 | 중간 | 14 |
| B | 19 | publication freshness·deploy version 보강 커밋(제목은 현재 기록에서 확인되지 않음) | freshness checks, `DEPLOY_VERSION`, production settings tests | stale 판단과 exact release 식별 | 운영 설명 가치는 높지만 핵심 release gate는 P32에 통합한다. | 높음 | 낮음 | 13, 22 · P32 |
| A | 20 | `fix(perf): measure paced schedule jitter`; `fix(perf): recover paced schedule without bursts` | `scripts/http_load_profile.py`; `run_profile`, `RunMeasurements`, `LoadReport`, `_ActiveRequestCounter` | [P28] monotonic paced scheduler와 catch-up burst 방지 | 시간·동시성·측정 왜곡을 직접 구현 가능한 작은 문제로 축소할 수 있다. | 높음 | 높음 | 22 |
| B | 20 | `fix(source): scope schedule interval by publication mode` | scheduler interval validation by recent/monthly/daily mode | source별 freshness와 singleton schedule 계약 | 운영 설정 판단은 중요하지만 별도 구현보다 P28·P32 설명에 포함하는 편이 낫다. | 중간 | 낮음 | 19, 22 |
| A | 21 | `test(history): build vnext browser fixture` | `grocery/tests/vnext_browser_fixture.py`; `build_vnext_browser_fixture` | [P29] deterministic fixture가 실제 review·seal·activation을 통과하도록 구성 | mock-only 테스트와 실제 lifecycle integration 사이의 경계를 설명할 수 있다. | 높음 | 중간 | 18 |
| A | 21 | `fix(qa): require disposable browser database` | fixture environment and database occupancy guards | [P29] DEBUG+QA·loopback·전용 empty DB allowlist | 파괴적 fixture의 환경 오작동을 fail-closed로 막는 안전 경계로 P29에 통합한다. | 높음 | 높음 | 22 |
| A | 21 | `test(source): add guarded live source-to-SSR smoke` | `scripts/live_api_e2e_smoke.py`; `validate_disposable_environment`, `CachedLiveClient`, `LiveSmokeReceipt` | [P30] bounded live fetch, cached pipeline, SSR source-call 금지, rollback 검증 | 외부 I/O와 deterministic application 검증을 분리하는 E2E 설계가 매우 일반화된다. | 높음 | 높음 | 03, 05, 11, 13 |
| A | 22 | `feat(ops): rehearse postgres recovery` | `scripts/postgres_backup_restore.py`; `create_backup`, `restore_backup` | [P32] custom dump·manifest·inventory·격리 restore 대사 | backup 생성보다 실제 복구 가능성과 publication 계약을 증명하는 절차가 중요하다. | 높음 | 중간 | 01, 05, 11 |
| S | 22 | `fix(ops): harden postgres recovery boundaries` | `scripts/postgres_backup_restore.py`; descriptor validation helpers | [P31] symlink·permission·TOCTOU를 막는 same-FD 검증과 cleanup | 언어·OS resource lifecycle·보안·실패 처리를 모두 확인하는 최상위 직접 구현 문제다. | 높음 | 높음 | 21 |
| A | 22 | release secret 검사 보강 커밋(제목은 현재 기록에서 확인되지 않음) | `scripts/secret_check.py`, `Makefile`; `production-check` | [P32] current tree와 Git object DB의 bounded secret scan | 배포 artifact에 credential이 없음을 재현 가능하게 검증하는 release gate로 P32에 통합한다. | 높음 | 중간 | 03, 14, 19 |
| B | 22 | `feat(web): serve immutable static assets` | `config/settings.py`; `staticfiles_storage_backend`, WhiteNoise middleware | hashed static artifact와 middleware order | 배포 품질에는 필요하지만 framework 설정 비중이 높아 별도 상세 문제로 만들지 않는다. | 중간 | 낮음 | 01 |
| A | 22 | `fix(test): isolate production gate runtime`; `docs: record predeploy completion evidence` | `Makefile`, release checklist, production checks | [P32] exact SHA·clean tree·forward migration·세 rollback domain의 STOP gate | 코드·데이터·publication 복구를 구분하고 모든 evidence를 묶는 시스템 설계 질문 가치가 높다. | 높음 | 중간 | 01, 12, 14, 19 |

## 대표 면접 포인트와 상세 워크북

S/A 대표 포인트는 32개이며, 각 포인트는 아래 상세 문서 중 정확히 한 곳에 독립 항목으로 작성했다. 선별표에서 같은 `[Pxx]`를 가리키는 다른 Thread 행은 해당 대표 문제에 통합된 연관 위치다.

| 포인트 | 우선순위 | 대표 주제 | 통합된 Thread | 상태 | 상세 파일 |
|---|---:|---|---|---|---|
| P01 | S | 금액 비교의 수치 정확도와 불가능 상태 배제 | 02, 04, 06, 08 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P02 | A | 문자열 코드 기반 의미 식별자와 교차 소스 동일성 | 02, 04, 07, 08 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P03 | A | 불변 스냅샷의 idempotent 생성과 충돌 검출 | 02, 04, 05 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P04 | A | provider 범위 사실과 소수 정밀도의 보존 | 07, 08, 16, 17 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P05 | A | 완전한 월 구간 선택과 결측을 잇지 않는 차트 | 15, 16, 18 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P06 | A | 일별 ledger의 동일 날짜·지역 경계와 안정적 페이지네이션 | 16, 17, 18 | 독립 상세 항목 작성됨 | [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) |
| P07 | S | 원자적인 fetch 상태 전이와 영수증 대사 | 03, 04, 09, 10 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P08 | S | 부분 네트워크 실패의 증거 보존과 실패 폐쇄 | 03, 19, 21 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P09 | A | 요청 allowlist·redaction·semantic scope hash | 03, 07, 09 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P10 | S | 정확한 row 계약, 중복 의미 키, 결정적 결과 hash | 04, 07 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P11 | A | 검토된 identity/dimension registry와 drift 차단 | 04, 07, 08 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P12 | A | bounded pagination과 재시도 폭주 방지 | 03, 07, 20 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P13 | A | artifact·parse run 상태 머신과 count reconciliation | 03, 04, 10 | 독립 상세 항목 작성됨 | [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) |
| P14 | A | 결정적 partition 계획과 manifest | 09, 10 | 독립 상세 항목 작성됨 | [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md) |
| P15 | A | 계획된 part의 provenance 검증과 idempotent 저장 | 09, 10 | 독립 상세 항목 작성됨 | [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md) |
| P16 | S | collection 완료의 전역 대사와 결정적 결과 hash | 09, 10, 11 | 독립 상세 항목 작성됨 | [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md) |
| P17 | S | DB append-only guard와 completion-versus-insert 동시성 | 10, 11 | 독립 상세 항목 작성됨 | [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md) |
| P18 | A | append-only review chain과 current-tail 승인 | 05, 11, 12 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P19 | A | 불변 publication 봉인과 canonical fact-set hash | 05, 11 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P20 | S | CAS publication 전환과 event-pointer 원자성 | 05, 11, 12, 13 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P21 | A | 현재 승인과 마지막 정상본 rollback의 다른 자격 규칙 | 05, 11, 22 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P22 | A | 역할 분리, release lock, 외부 인증 경계 | 05, 11, 12, 22 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P23 | A | recent·historical publication 채널 독립성과 장애 격리 | 06, 13, 16, 17, 21 | 독립 상세 항목 작성됨 | [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) |
| P24 | A | active-only 공개 읽기와 membership 무결성·query bound | 06, 13, 16, 17 | 독립 상세 항목 작성됨 | [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md) |
| P25 | A | canonical GET 상태와 no-JS 선택 목록 | 14, 15, 18 | 독립 상세 항목 작성됨 | [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md) |
| P26 | A | 공개 query privacy와 HTTP fail-closed 헤더 | 14, 15, 19, 21 | 독립 상세 항목 작성됨 | [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md) |
| P27 | A | allowlist structured logging과 request context 정리 | 14, 19, 22 | 독립 상세 항목 작성됨 | [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md) |
| P28 | A | 지연을 따라잡되 burst를 만들지 않는 paced scheduler | 20, 22 | 독립 상세 항목 작성됨 | [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) |
| P29 | A | 브라우저 fixture를 disposable 환경에만 허용하는 fail-closed gate | 18, 21 | 독립 상세 항목 작성됨 | [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) |
| P30 | A | live source를 한 번만 통과시키고 SSR 경계는 격리하는 E2E 설계 | 03, 05, 11, 13, 21 | 독립 상세 항목 작성됨 | [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) |
| P31 | S | path 검증을 file descriptor identity로 고정하는 TOCTOU 방어 | 22 | 독립 상세 항목 작성됨 | [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) |
| P32 | A | application·publication·database rollback을 분리한 릴리스 관문 | 01, 12, 14, 19, 22 | 독립 상세 항목 작성됨 | [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) |

## 상세 문서 구성

- [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md) — 금액 산술, 의미 식별자, 불변 snapshot·typed fact, 월별 gap, daily ledger (6개 포인트)
- [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md) — 수집 상태 전이, 부분 실패, request scope, strict parser, registry, pagination, parse audit (7개 포인트)
- [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md) — partition 계획·part 저장·collection 완료·DB 동시성 guard (4개 포인트)
- [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md) — review chain, seal, CAS activation, rollback, role separation, channel isolation (6개 포인트)
- [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md) — active-only read, canonical URL, query privacy, structured logging (4개 포인트)
- [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md) — paced load, disposable fixture, live smoke, descriptor 안전성, release/recovery gate (5개 포인트)

## 완전성 검증

- 선별표의 S/A 행: **58개**. 모든 행에 기존 상세 포인트 `[P01]`–`[P32]` 중 하나가 명시되어 있다.
- 고유 S/A 대표 포인트: **32개**. 상세 문서의 독립 항목: **32개**.
- 다른 대표 항목에만 통합되고 상세 위치가 없는 S/A 포인트: **0개**.
- 상세 문서에는 있으나 인덱스에 없는 고아 포인트: **0개**.
- 한 상세 문서의 포인트 수: 4–7개. 최대 7개를 넘지 않으며 자연스러운 주제 경계를 유지했다.
- B/C 행은 별도 상세 문제를 만들지 않았으며, 관련 S/A 포인트가 있으면 선별표의 연관 Thread 또는 아래 통합 목록에만 연결했다.

## 백지 구현 우선순위

1. **P20 · CAS publication 전환과 event-pointer 원자성** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
2. **P31 · path 검증을 file descriptor identity로 고정하는 TOCTOU 방어** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
3. **P16 · collection 완료의 전역 대사와 결정적 결과 hash** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
4. **P10 · 정확한 row 계약, 중복 의미 키, 결정적 결과 hash** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
5. **P07 · 원자적인 fetch 상태 전이와 영수증 대사** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
6. **P17 · DB append-only guard와 completion-versus-insert 동시성** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
7. **P01 · 금액 비교의 수치 정확도와 불가능 상태 배제** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
8. **P08 · 부분 네트워크 실패의 증거 보존과 실패 폐쇄** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
9. **P25 · canonical GET 상태와 no-JS 선택 목록** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
10. **P28 · 지연을 따라잡되 burst를 만들지 않는 paced scheduler** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
11. **P15 · 계획된 part의 provenance 검증과 idempotent 저장** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
12. **P19 · 불변 publication 봉인과 canonical fact-set hash** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
13. **P24 · active-only 공개 읽기와 membership 무결성·query bound** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
14. **P13 · artifact·parse run 상태 머신과 count reconciliation** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
15. **P30 · live source를 한 번만 통과시키고 SSR 경계는 격리하는 E2E 설계** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
16. **P09 · 요청 allowlist·redaction·semantic scope hash** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
17. **P14 · 결정적 partition 계획과 manifest** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
18. **P03 · 불변 스냅샷의 idempotent 생성과 충돌 검출** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
19. **P18 · append-only review chain과 current-tail 승인** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
20. **P32 · application·publication·database rollback을 분리한 릴리스 관문** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
21. **P05 · 완전한 월 구간 선택과 결측을 잇지 않는 차트** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
22. **P06 · 일별 ledger의 동일 날짜·지역 경계와 안정적 페이지네이션** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
23. **P12 · bounded pagination과 재시도 폭주 방지** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
24. **P27 · allowlist structured logging과 request context 정리** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
25. **P23 · recent·historical publication 채널 독립성과 장애 격리** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
26. **P22 · 역할 분리, release lock, 외부 인증 경계** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
27. **P02 · 문자열 코드 기반 의미 식별자와 교차 소스 동일성** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
28. **P04 · provider 범위 사실과 소수 정밀도의 보존** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
29. **P11 · 검토된 identity/dimension registry와 drift 차단** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
30. **P21 · 현재 승인과 마지막 정상본 rollback의 다른 자격 규칙** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
31. **P29 · 브라우저 fixture를 disposable 환경에만 허용하는 fail-closed gate** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
32. **P26 · 공개 query privacy와 HTTP fail-closed 헤더** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)

## 설명 연습 우선순위

1. **P20 · CAS publication 전환과 event-pointer 원자성** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
2. **P31 · path 검증을 file descriptor identity로 고정하는 TOCTOU 방어** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
3. **P17 · DB append-only guard와 completion-versus-insert 동시성** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
4. **P22 · 역할 분리, release lock, 외부 인증 경계** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
5. **P32 · application·publication·database rollback을 분리한 릴리스 관문** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
6. **P23 · recent·historical publication 채널 독립성과 장애 격리** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
7. **P18 · append-only review chain과 current-tail 승인** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
8. **P19 · 불변 publication 봉인과 canonical fact-set hash** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
9. **P01 · 금액 비교의 수치 정확도와 불가능 상태 배제** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
10. **P07 · 원자적인 fetch 상태 전이와 영수증 대사** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
11. **P08 · 부분 네트워크 실패의 증거 보존과 실패 폐쇄** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
12. **P10 · 정확한 row 계약, 중복 의미 키, 결정적 결과 hash** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
13. **P16 · collection 완료의 전역 대사와 결정적 결과 hash** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
14. **P24 · active-only 공개 읽기와 membership 무결성·query bound** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
15. **P26 · 공개 query privacy와 HTTP fail-closed 헤더** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
16. **P27 · allowlist structured logging과 request context 정리** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
17. **P28 · 지연을 따라잡되 burst를 만들지 않는 paced scheduler** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
18. **P30 · live source를 한 번만 통과시키고 SSR 경계는 격리하는 E2E 설계** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
19. **P09 · 요청 allowlist·redaction·semantic scope hash** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
20. **P13 · artifact·parse run 상태 머신과 count reconciliation** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
21. **P21 · 현재 승인과 마지막 정상본 rollback의 다른 자격 규칙** — [04-review-publication-and-control-plane.md](04-review-publication-and-control-plane.md)
22. **P05 · 완전한 월 구간 선택과 결측을 잇지 않는 차트** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
23. **P02 · 문자열 코드 기반 의미 식별자와 교차 소스 동일성** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
24. **P04 · provider 범위 사실과 소수 정밀도의 보존** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
25. **P12 · bounded pagination과 재시도 폭주 방지** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)
26. **P14 · 결정적 partition 계획과 manifest** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
27. **P15 · 계획된 part의 provenance 검증과 idempotent 저장** — [03-partitioned-ingestion-and-concurrency.md](03-partitioned-ingestion-and-concurrency.md)
28. **P03 · 불변 스냅샷의 idempotent 생성과 충돌 검출** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
29. **P25 · canonical GET 상태와 no-JS 선택 목록** — [05-public-read-security-and-observability.md](05-public-read-security-and-observability.md)
30. **P29 · 브라우저 fixture를 disposable 환경에만 허용하는 fail-closed gate** — [06-performance-testing-and-recovery.md](06-performance-testing-and-recovery.md)
31. **P06 · 일별 ledger의 동일 날짜·지역 경계와 안정적 페이지네이션** — [01-domain-invariants-and-identity.md](01-domain-invariants-and-identity.md)
32. **P11 · 검토된 identity/dimension registry와 drift 차단** — [02-acquisition-and-parser-boundaries.md](02-acquisition-and-parser-boundaries.md)

## 한 문제로 통합한 Thread 묶음

1. **Thread 02 + 08 → P01–P04** — Decimal·semantic identity·immutable typed fact를 “도메인 사실을 불가능 상태 없이 저장한다”는 한 축으로 통합
2. **Thread 03 + 07 + 09 → P07–P12** — request 계약, bounded transport, receipt, scope hash를 외부 I/O 경계 문제로 통합
3. **Thread 04 + 07 → P10–P11** — 최근·월별·지역별·시장별 parser를 exact schema·reviewed registry 문제로 통합
4. **Thread 03 + 04 + 10 → P13** — artifact·parse run·collection completion의 deterministic reconciliation을 하나의 감사 pipeline으로 통합
5. **Thread 09 + 10 → P14–P17** — partition plan→part persist→complete와 insert/finalize race를 한 ingestion 묶음으로 통합
6. **Thread 05 + 11 → P18–P20** — recent·historical review/seal/activation을 공통 publication lifecycle 문제로 통합
7. **Thread 05 + 11 + 22 → P21, P32** — last-known-good publication rollback과 application/database recovery의 차이를 한 복구 축으로 연결
8. **Thread 12 + 05 + 11 → P22** — local/production actor, reviewer/publisher 역할, command receipt를 control-plane 경계 하나로 통합
9. **Thread 06 + 13 + 16 + 17 → P23–P24** — 독립 publication 채널과 active-only public read를 하나의 공개 authorization·격리 축으로 통합
10. **Thread 14 + 15 + 19 → P25–P27** — canonical URL, no-reflection HTTP 정책, allowlist logging을 공개 query privacy 축으로 통합
11. **Thread 18 + 21 → P29 및 B 항목** — 접근성 browser flow와 deterministic disposable fixture를 UI assurance 축으로 연결
12. **Thread 01 + 19 + 22 → P32** — contract-first acceptance, release identity, backup/restore evidence를 predeploy gate 하나로 통합
