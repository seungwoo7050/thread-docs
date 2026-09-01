# 시간 기반 검색·소비 계약·개인정보 경계 워크북

이 문서는 게시된 증거를 실제 여행 기회로 계산하는 날짜·시간 알고리즘, 소비 시점의 fail-closed publication 계약, 그리고 사용자가 입력한 여행 시간창을 저장·로그·URL에 남기지 않는 요청 경계를 다룬다.

<a id="w14"></a>

---

# [travel_windows/search.py::_schedule_events, _calculate_candidates] W14 — 시간대 기반 운항 확장·codeshare dedup·결정적 ranking (S)

## 면접 질문

`_schedule_events`와 `_calculate_candidates`는 계절 운항표를 요청 날짜별 event로 확장하고, master flight 기준 codeshare를 제거한 뒤 outbound와 inbound를 duration으로 연결해 목적지 체류시간을 계산합니다. 이때 모든 계산의 기준 시각과 timezone 변환 순서를 설명해 보세요.

꼬리 질문:

1. 운항 event dedup key에 direction, master flight, airport, ICN event datetime을 모두 넣는 이유는 무엇인가요?
2. codeshare 대표를 실제 master flight 우선, flight number, carrier 순으로 고르면 어떤 결정성이 생기나요?
3. 체류시간 40시간을 `>= 2400분`으로 inclusive하게 검사하고 초를 내림하는 선택의 경계 효과는 무엇인가요?
4. NRT와 HND를 같은 city로 묶되 한 city의 alternative itinerary가 서로 다른 airport를 섞지 않게 한 이유는 무엇인가요?

## 30초 모범 답변

운항표의 `icn_event_time`은 KST 날짜·요일·유효기간으로 concrete aware datetime을 만들고 요청 window 안의 event만 남깁니다. outbound ICN 출발에 outbound duration을 더해 목적지 도착으로 변환하고, inbound ICN 도착에서 inbound duration을 빼 목적지 출발을 구합니다. 같은 airport의 쌍만 연결해 40시간 이상을 후보로 만들고, 체류시간·귀환 여유·늦은 출발·stable identity 순으로 정렬합니다. codeshare는 master flight key로 하나만 남기고 city당 같은 airport의 최대 2개, 최대 6개 city로 제한합니다.

## 답변 핵심 키워드

- aware datetime
- schedule expansion
- master-flight dedup
- duration inversion
- inclusive 40-hour invariant
- deterministic tie-break
- city/airport grouping

## 꼬리 질문과 핵심 답변

### outbound와 inbound 모두 ICN event time을 저장하는 모델의 장점은 무엇인가요?

운항표의 기준 endpoint와 시간대를 하나로 고정해 date/weekday 확장을 단순화합니다. 목적지 event는 검증된 방향별 duration을 이용해 계산하므로, 서로 다른 목적지 timezone과 날짜 경계를 일관되게 처리할 수 있습니다.

### `datetime + timedelta` 후 `astimezone(destination)` 순서가 중요한가요?

aware datetime에서 elapsed duration을 더한 뒤 목적지 zone으로 표현하면 동일 instant를 올바른 local time으로 보여줍니다. naive local time끼리 더하거나 timezone label만 붙이면 DST와 날짜 경계에서 잘못된 instant가 됩니다.

### inbound 목적지 출발을 왜 `ICN 도착 - inbound duration`으로 구하나요?

모델이 가진 concrete event가 ICN 도착 시각이기 때문입니다. 목적지 출발 instant를 역산한 뒤 목적지 timezone으로 표시해야 outbound 도착과 같은 기준의 체류 구간을 얻습니다.

### 후보 정렬에서 business tie-break 뒤 stable identity가 필요한 이유는 무엇인가요?

같은 체류시간과 여유를 가진 후보의 DB 반환 순서는 보장되지 않습니다. city/airport/master flight 같은 stable key가 없으면 같은 입력이 요청마다 다른 순위를 내고 테스트·사용자 경험·캐시 가능성이 흔들립니다.

### 현재 이중 loop의 복잡도와 개선 방법은 무엇인가요?

outbound 수를 `O`, inbound 수를 `I`라 하면 최악 `O(O×I)`입니다. airport별 inbound를 그룹화하고 시간순으로 정렬한 뒤 40시간 조건과 return window를 이용해 탐색 범위를 줄일 수 있습니다. 현재는 최대 날짜창과 route 수가 작고 상한이 명시되어 단순성이 우선입니다.

## 백지 구현

### 구현 목표

계절 운항 규칙과 공항별 양방향 duration, 사용자의 KST 시간창으로부터 최소 40시간 체류 가능한 왕복 후보를 결정적으로 계산한다.

### 인터페이스

`find_itineraries(schedules: Sequence[ScheduleRule], durations: Mapping[AirportId, Duration], eligible_countries: Set[CountryId], departure_at: datetime, return_by: datetime) -> Sequence[CityResult]`

`CityResult`는 대표 itinerary와 최대 한 개의 alternative를 가지며 최대 6개다.

### 입력

- aware `departure_at`, `return_by`
- schedule rule: direction, master/marketing flight, carrier, airport/city/country, ICN local time, valid date range, 7자리 weekday mask
- airport별 outbound/inbound duration과 IANA timezone
- entry·warning이 모두 eligible한 country ID 집합

### 출력

- 최대 6개의 ranked city result
- city마다 동일 airport의 itinerary 최대 2개
- 각 itinerary에는 목적지 local 도착/출발, stay minutes, outbound/inbound event identity가 있다.

### 반드시 만족해야 할 조건

- 요청 경계와 모든 event datetime은 aware여야 한다.
- KST 기준 inclusive 날짜 범위를 확장하고 유효기간·weekday mask·요청 window를 모두 적용한다.
- codeshare key는 direction/master flight/airport/event instant이며 stable 대표 하나만 남긴다.
- duration이 있고 eligible country에 속한 airport만 계산한다.
- outbound와 inbound는 같은 airport끼리만 연결한다.
- stay는 목적지 출발 instant - 목적지 도착 instant의 내림 정수 minute이며 2400 이상이어야 한다.
- 정렬은 stay desc, return slack desc, outbound departure desc 뒤 stable textual/flight keys다.
- 같은 city라도 첫 대표 airport와 다른 airport의 후보를 alternative로 섞지 않는다.

### 경계 조건

- 정확히 40시간, 39시간 59분
- 요청 window 시작·끝과 정확히 같은 event
- schedule valid_from/valid_until 경계일
- 일요일/월요일을 포함한 weekday index
- master와 codeshare가 동시에 존재, codeshare만 존재
- 목적지 timezone 날짜가 KST 날짜와 다름
- 같은 city의 두 공항이 서로 다른 순위 후보를 가짐
- 6개와 7개 city, city당 2개와 3개 itinerary

### 실패 조건

- naive datetime을 조용히 KST로 가정함
- 목적지 local wall time끼리 직접 빼 DST instant를 틀림
- marketing flight를 별개 운항으로 중복 계산함
- DB/입력 순서에 따라 tie 결과가 달라짐
- alternative에 다른 airport를 섞어 사용자가 공항 변경을 놓치게 함
- 함수가 검색 입력을 DB·cache·log에 저장함

### 필요한 제약사항

- 입력 collection을 변경하지 않는다.
- date expansion은 최대 7일 window라는 상위 계약을 활용한다.
- 단순 구현은 이중 loop를 허용하되 시간복잡도와 airport grouping 개선안을 설명한다.
- 출력은 도메인 객체/값이며 HTML 문자열을 만들지 않는다.

## 구현 후 자가 검증

- [ ] KST date expansion의 양 끝이 inclusive인가?
- [ ] weekday mask와 schedule 유효기간 경계가 정확한가?
- [ ] master flight dedup이 실제 master를 우선하고 결과가 결정적인가?
- [ ] outbound 더하기/inbound 빼기 duration의 방향이 맞는가?
- [ ] 정확히 2400분 후보를 포함하고 그 미만을 제외하는가?
- [ ] same-airport pairing과 city grouping을 혼동하지 않는가?
- [ ] 정렬 tie-break가 입력·DB 순서와 무관한가?
- [ ] city 6개·itinerary 2개 상한과 같은-airport alternative 조건이 지켜지는가?
- [ ] 검색 함수가 어떤 persistence나 외부 I/O도 하지 않는가?

## 구현 후 설명할 것

- ICN 기준 schedule 모델과 목적지 timezone 계산 순서
- codeshare dedup identity와 대표 선택 기준
- 정렬 우선순위가 사용자 가치와 결정성을 함께 반영하는 방식
- city와 airport를 서로 다른 grouping level로 본 이유
- 현재 `O(O×I)` 구현을 유지한 범위 상한과 확장 시 최적화 방향

## 원본 확인 위치

- 파일: `travel_windows/search.py`
- 함수: `_date_range`, `_schedule_events`, `_event_identity`, `_candidate_sort_key`, `_calculate_candidates`, `_itinerary_context`
- 상수: `MINIMUM_LOCAL_STAY_MINUTES`, `MAXIMUM_CITY_RESULTS`, `MAXIMUM_ITINERARIES_PER_CITY`
- Commit: `20cd99e feat(search): calculate eligible travel opportunities`
- 관련 테스트: `travel_windows/tests/test_search.py`
- 대표 테스트: `test_40_hours_is_inclusive_and_codeshare_uses_the_master_flight`, `test_nrt_and_hnd_group_as_one_city_with_two_total_itineraries`, city limit/sort/same-airport alternative 테스트
- 웹 경계 테스트: `public_web/tests/test_travel_opportunity_web.py`

<a id="w15"></a>

---

# [travel_windows/search.py::_load_eligible_country_ids, _warning_snapshot_contract_allowed] W15 — 검색 시점의 exact publication contract와 fail-closed eligibility (A)

## 면접 질문

검색은 current publication pointer가 존재한다는 사실만으로 국가나 항공편을 사용하지 않고 source/parser/schema/observation lineage를 다시 검사합니다. 이미 ingestion·publication·DB trigger가 검증한 데이터를 소비 시점에서 재검증하는 이유와, 이 방어가 막는 실제 drift 사례를 설명해 보세요.

꼬리 질문:

1. entry와 warning의 eligible country ID를 union이 아니라 intersection으로 만드는 이유는 무엇인가요?
2. V2 warning snapshot의 `source_item_count=0`일 때 fact position tuple이 빈 tuple이어야 하는 검사가 왜 중요한가요?
3. warning publication이 가리키는 observation attempt의 request fingerprint와 body hash까지 확인하는 이유는 무엇인가요?
4. flight evidence가 일부만 잘못됐을 때 유효한 airport만 골라 계속하지 않고 module 전체를 unavailable로 닫는 이유는 무엇인가요?

## 30초 모범 답변

DB에 current 포인터가 있어도 과거 migration, direct SQL, legacy contract, 잘못된 join으로 소비 계약과 다른 row가 연결될 수 있습니다. 검색 경계는 entry와 warning 각각의 exact source/parser/schema/typed lineage를 확인해 교집합 국가만 허용하고, warning fact 순서와 실제 수집 attempt fingerprint·body hash까지 연결합니다. flight는 schedule과 duration이 하나의 sealed aggregate이므로 일부만 사용하지 않고 불완전하면 전체 unavailable로 닫아 잘못된 추천을 피합니다.

## 답변 핵심 키워드

- consumer-side validation
- exact contract allowlist
- eligibility intersection
- observation provenance
- ordered empty snapshot
- fail-closed aggregate

## 꼬리 질문과 핵심 답변

### defense-in-depth가 중복 query와 유지보수 부담을 만들지 않나요?

그렇습니다. 대신 게시와 검색 사이의 schema evolution·migration·ORM 우회 위험을 줄입니다. 계약 상수를 중앙화하고 테스트로 producer/consumer를 함께 갱신해야 하며, 성능은 singleton/current row와 제한된 국가 수로 bounded합니다.

### legacy contract를 모두 거부하지 않고 일본에만 제한해 허용하는 이유는 무엇인가요?

과거 v1 자료는 일본 단일 scope로 수집·승인되었습니다. 현재 multi-country 소비에 일반화하면 권리와 identity 범위를 확장하는 셈이므로, 원래 승인 범위인 일본에서만 명시적으로 호환합니다.

### empty snapshot에 child가 하나라도 있거나 count가 1인데 child가 없으면 어떻게 해야 하나요?

둘 다 closure 불일치입니다. 0이면 정확히 빈 position sequence여야 하고, N이면 정확히 1..N이어야 합니다. “빈 데이터”와 “누락된 데이터”를 구분해야 합니다.

### stale publication은 eligibility에서 제외해야 하나요?

현재 flight 검색은 structurally valid한 stale 자료로 결과를 계산하되 상태를 `stale`로 명확히 표시합니다. freshness와 integrity는 별도 축입니다. 반면 lineage가 불완전하면 계산 자체를 하지 않습니다.

### 일부 airport만 유효할 때 계속하는 것이 사용자에게 더 유용하지 않나요?

후보 seal과 publication이 하나의 complete evidence set으로 검토되었습니다. 소비자가 임의 subset을 만들면 검토된 closure와 다른 자료를 게시하는 셈입니다. 부분 허용 정책이 필요하면 별도 검토·봉인 단위를 설계해야 합니다.

## 백지 구현

### 구현 목표

current entry/warning publication metadata로 검색 가능한 국가 교집합을 계산하고, current flight publication을 완전한 evidence aggregate로 로드하는 fail-closed validator를 구현한다.

### 인터페이스

`eligible_countries(entry_rows: Sequence[PublicationView], warning_rows: Sequence[PublicationView], contracts: ContractRegistry) -> FrozenSet[UUID]`

`load_flight_evidence(pointer: FlightPointerView, contracts: ContractRegistry) -> FlightEvidence | EvidenceState`

### 입력

- country-scoped current publication projection
- source identity/revision/owner/locator/attribution
- publication의 parser name/version/contract/schema snapshot
- typed revision → parse run → artifact → observation attempt lineage
- warning snapshot count와 child source positions
- flight publication → schedule revision/schedules/duration snapshot

### 출력

- entry와 warning이 모두 exact contract를 만족하는 country ID의 immutable 교집합
- flight evidence 또는 `EMPTY`, `UNAVAILABLE`, `SERVER_ERROR`
- freshness는 구조적 eligibility와 분리된 별도 상태로 계산한다.

### 반드시 만족해야 할 조건

- contract registry에 없는 source/parser/schema 조합은 거부한다.
- publication scope country와 typed revision country가 같아야 한다.
- multi-country contract와 허용된 country identity mapping이 정확히 맞아야 한다.
- legacy contract는 명시된 legacy scope에서만 허용한다.
- warning V2 child positions는 정확히 `1..source_item_count`; 0이면 empty sequence다.
- observation attempt는 같은 source/revision, 승인 rights, 정확한 request identity, 성공 status/provider code, artifact body hash를 가져야 한다.
- flight schedule은 complete/validated이고 destination timezone이 유효하며 duration은 airport당 정확히 하나여야 한다.
- aggregate 일부 오류는 전체 flight evidence unavailable로 만든다.

### 경계 조건

- entry만 있고 warning이 없음, 또는 반대
- 두 module에 서로 다른 country 집합
- valid legacy 일본 자료와 legacy 비일본 자료
- warning count 0/1/100과 position 누락·중복
- publication snapshot hash는 맞지만 observation attempt request identity가 다름
- artifact body hash와 attempt response hash가 다름
- flight duration 중복·누락, invalid IANA timezone
- source checked time이 미래 또는 stale threshold 경계

### 실패 조건

- pointer 존재만으로 eligible 처리함
- entry/warning union을 사용함
- child count만 비교하고 연속 position을 확인하지 않음
- stale와 structurally unavailable을 같은 의미로 처리함
- flight 일부 row만 골라 검토되지 않은 subset을 만듦
- ORM/DB 예외 상세를 public state에 노출함

### 필요한 제약사항

- predicate는 순수 함수에 가깝게 분리해 contract fixture로 테스트한다.
- source/parser contract 값은 allowlist registry에서만 온다.
- 국가 수와 current pointer scope가 작다는 전제에서 bounded query 수를 유지한다.

## 구현 후 자가 검증

- [ ] entry와 warning의 exact 교집합만 반환하는가?
- [ ] legacy와 current contract의 국가 scope를 구분하는가?
- [ ] warning 0개 snapshot과 누락된 child를 구분하는가?
- [ ] request fingerprint·attempt status·artifact hash까지 lineage를 확인하는가?
- [ ] parser expected/observed schema와 typed fingerprint를 모두 확인하는가?
- [ ] flight의 schedule/duration/timezone 오류 하나가 전체 unavailable을 만드는가?
- [ ] stale structurally valid 자료는 결과와 경고 상태를 함께 제공하는가?
- [ ] 예외가 고정 public state로 축약되는가?

## 구현 후 설명할 것

- producer 검증 뒤 consumer 검증을 반복하는 위협 모델
- integrity와 freshness를 별도 상태 축으로 둔 이유
- entry·warning 교집합이 제품 eligibility를 정의하는 방식
- legacy contract를 scope-limited allowlist로 다룬 방식
- aggregate partial salvage를 거부한 설계 trade-off

## 원본 확인 위치

- 파일: `travel_windows/search.py`
- 함수: `_load_eligible_country_ids`, `_publication_contract_allowed`, `_warning_snapshot_contract_allowed`, `_warning_fact_positions`, `_load_current_flight_evidence`, `_current_publication_state`
- Commit: `95f7468 fix(search): require usable travel publications`
- Commit: `e951404 fix(search): enforce country publication contracts`
- Commit: `5edb437 feat(search): accept verified warning snapshots`
- 관련 테스트: `travel_windows/tests/test_search.py`의 exact country contract, zero/ordered warning fact, stale publication, no-persistence 테스트
- 연관 소비 경계: `public_web/results.py::_freshness`, `_matching_success_is_current`, `_snapshot_warning_facts`
- 웹 테스트: `public_web/tests/test_travel_opportunity_web.py`의 module isolation·snapshot count·legacy scope 테스트

<a id="w16"></a>

---

# [public_web/views.py::index, public_web/middleware.py::PublicPrivacyMiddleware] W16 — 여행 시간 입력의 no-retention/privacy boundary (A)

## 면접 질문

이 애플리케이션은 여행 출발·귀환 시각을 개인정보성 transient input으로 보고 GET query, session, cache, DB, 일반 request log에 남기지 않습니다. POST 결과를 redirect하지 않고 같은 요청의 200 응답으로 렌더링하고, query가 오면 보안 redirect보다 먼저 queryless 303으로 바꾸는 이유를 설명해 보세요.

꼬리 질문:

1. 일반적인 POST/Redirect/GET 패턴을 쓰지 않은 이유와 새로 생기는 UX trade-off는 무엇인가요?
2. `datetime-local` 값을 KST aware datetime으로 바꾸고 미래·순서·7일 상한을 함께 검사하는 이유는 무엇인가요?
3. `sensitive_post_parameters`만으로 privacy가 완성되지 않는 이유는 무엇인가요?
4. telemetry가 raw duration 대신 bucket, resolver route allowlist, method allowlist만 기록하는 이유는 무엇인가요?

## 30초 모범 답변

PRG를 쓰면 결과를 재현하려고 query, session, cache, 서버 저장소 중 하나에 입력을 보관하기 쉽습니다. 이 구현은 POST body를 현재 요청에서만 검증·계산하고 바로 200을 렌더링해 아무 복구 토큰도 남기지 않습니다. query는 HTTPS나 slash redirect가 원 URL을 복사하기 전에 queryless 303으로 제거합니다. 모든 응답은 no-store이고 500 body는 고정 메시지로 교체하며, telemetry는 route/method/status/시간 bucket/상관 ID 같은 allowlisted 값만 남깁니다.

## 답변 핵심 키워드

- transient POST
- no PRG state
- query stripping before redirect
- KST-aware validation
- no-store/fixed 500
- allowlisted telemetry
- sensitive-absence verification

## 꼬리 질문과 핵심 답변

### same-request POST 200의 UX 단점은 무엇인가요?

새로고침 시 브라우저가 form 재전송 경고를 낼 수 있고 결과 URL을 공유·북마크할 수 없습니다. 이 프로젝트는 결과 재현성과 공유보다 입력 비보존을 우선합니다.

### GET query를 view에서만 제거하면 왜 늦을 수 있나요?

`SECURE_SSL_REDIRECT`나 `APPEND_SLASH` 같은 middleware가 view 전에 원 query를 포함한 Location을 만들 수 있습니다. privacy middleware가 canonical path를 먼저 인식해 queryless redirect를 반환해야 합니다.

### `sensitive_post_parameters`가 보호하는 범위는 무엇인가요?

Django debug/error report에서 지정 POST 값을 가리는 보조 장치입니다. access log, proxy URL, session/cache/DB, custom telemetry, template의 500 렌더링까지 자동으로 막지는 않습니다.

### invalid form에서 입력값을 다시 보여주는 것은 retention 아닌가요?

같은 요청 메모리와 response body 안에서 사용자가 수정하도록 반사하는 것은 영속 보관과 다릅니다. 다만 response는 no-store이고 500/debug 경로에서는 bound value를 고정 body로 교체해 누출을 막습니다.

### correlation ID도 사용자를 추적할 수 있지 않나요?

요청마다 임의 UUID를 새로 만들고 cookie/session/user/IP와 결합하지 않으면 단일 요청 진단용입니다. 장기 보존 기간과 downstream log 결합 정책도 별도로 제한해야 합니다.

## 백지 구현

### 구현 목표

두 개의 `datetime-local` 입력을 현재 요청에서만 검증해 계산 함수를 호출하고, query·session·cache·DB·일반 telemetry에 값을 남기지 않는 HTTP 경계를 구현한다.

### 인터페이스

- `parse_trip_window(form_data: Mapping[str, str], now: datetime) -> ValidWindow | FormErrors`
- `privacy_middleware(request, next_handler) -> response`
- `search_view(request) -> response`
- `telemetry_event(request, response, elapsed_ns) -> SafeEvent`

### 입력

- GET 또는 POST request
- `departure_at`, `return_by`의 정확한 minute 형식 문자열
- KST aware 현재 시각

### 출력

- query가 있는 canonical route: query 없는 303
- valid POST: same-request 200 결과
- invalid POST: same-request 200 bound error form
- public 500: 입력·예외 문자열 없는 고정 body
- 모든 public response: no-store 보안 헤더

### 반드시 만족해야 할 조건

- canonical GET query는 어떤 하위 middleware/view보다 먼저 제거한다.
- parser는 형식을 엄격히 확인하고 KST aware datetime을 만든다.
- departure는 현재보다 미래, return은 departure보다 뒤, 차이는 7일 이하여야 한다.
- valid POST만 DB-only 순수 검색 함수를 호출한다.
- session 수정, cache write, search row insert, query redirect를 하지 않는다.
- debug 여부와 무관하게 public 500 body에서 입력과 exception을 제거한다.
- telemetry에는 고정 event, UTC timestamp, bounded release, allowlisted route/method, valid status, duration bucket, 새 correlation ID, redacted flag만 허용한다.
- telemetry logging 실패는 사용자 response를 바꾸지 않는다.

### 경계 조건

- query가 있는 HTTP 요청에 HTTPS redirect도 필요한 경우
- `/results`의 slash 보정과 query가 동시에 있는 경우
- departure가 현재와 정확히 같음
- return이 departure와 같음
- 정확히 7일과 7일+1분
- DST를 쓰지 않는 KST 입력과 다른 timezone의 aware datetime 객체 입력
- invalid POST, CSRF 실패, view/template 예외
- unknown route/method/status와 logging exception

### 실패 조건

- 입력을 URL 또는 redirect token에 인코딩함
- PRG를 위해 session/cache/DB에 값을 저장함
- request 전체 path/query/body를 telemetry에 기록함
- raw elapsed time을 고카디널리티 값으로 기록함
- debug 500이 bound form이나 exception repr를 렌더링함
- 로그 실패가 정상 response를 실패로 바꿈

### 필요한 제약사항

- canonical route allowlist를 명시한다.
- no-store와 기본 보안 header는 중앙 middleware에서 적용한다.
- 검색 함수에는 validated aware datetime만 전달한다.
- DB schema에 transient field가 없는지는 별도 release gate/test로 검증한다.

## 구현 후 자가 검증

- [ ] query가 HTTPS/slash redirect Location에 복사되기 전에 제거되는가?
- [ ] valid/invalid POST가 session·cache·DB write 없이 같은 요청에서 끝나는가?
- [ ] 현재/순서/7일 경계와 KST aware 변환이 정확한가?
- [ ] CSRF와 same-origin 정책을 유지하면서 privacy를 지키는가?
- [ ] 모든 public response에 no-store가 적용되는가?
- [ ] debug 500에도 입력값·예외 문자열이 나타나지 않는가?
- [ ] telemetry key/value가 allowlist 밖 request 정보를 포함하지 않는가?
- [ ] logging failure가 response 상태·body를 바꾸지 않는가?
- [ ] DB catalog 검사에서 transient input column이 없음을 확인할 수 있는가?

## 구현 후 설명할 것

- 표준 PRG를 포기하고 transient POST 200을 선택한 이유
- query 제거 middleware 순서가 privacy invariant인 이유
- 같은 요청의 form 반사와 영속 retention의 차이
- 관측 가능성과 최소 수집 사이에서 bucket/allowlist를 택한 이유
- 테스트를 넘어 DB catalog·artifact까지 민감값 부재를 검증한 이유

## 원본 확인 위치

- form: `public_web/forms.py::_KSTDateTimeField`, `TripForm.clean`
- view: `public_web/views.py::index`, `search_travel_opportunities`
- privacy middleware: `public_web/middleware.py::PublicPrivacyMiddleware`, `_protect`
- telemetry: `operations/middleware.py::RequestTelemetryMiddleware`, `_duration_bucket`
- Commit: `deb9208 feat(web): add the no-retention trip form`
- release gate Commit: `001ee49 ops: add silent sensitive-absence release gate`
- 브라우저 evidence 보완: `d83788a fix(browser): redact trip inputs from acceptance evidence`
- 관련 테스트: `public_web/tests/test_privacy_form.py`
- 민감정보 부재 테스트: `operations/tests/test_sensitive_absence.py`, `operations/tests/test_observability.py`, `operations/tests/test_live_parser_replay.py`
