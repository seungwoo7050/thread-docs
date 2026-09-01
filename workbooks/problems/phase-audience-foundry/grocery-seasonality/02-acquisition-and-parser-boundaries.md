# 수집·요청·파서 경계

외부 I/O를 bounded evidence로 바꾸고 strict parser와 재현 가능한 parse run으로 연결한다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P07. [Thread 03 / `feat(source): persist reconciled fetch receipts`] 원자적인 fetch 상태 전이와 영수증 대사

**우선순위:** S

### 면접 질문

- STARTED fetch를 SUCCEEDED로 바꿀 때 어떤 값들을 한 트랜잭션에서 대사해야 하나요?
- 페이지 영수증의 request ordinal, page number, declared total, 실제 row 수가 각각 왜 필요합니까?
- 성공 완료 명령을 재실행했을 때 언제 idempotent replay이고 언제 충돌인가요?
- 꼬리 질문: artifact 연결과 attempt 완료가 분리 커밋이면 어떤 중간 상태가 생길 수 있나요?

### 30초 모범 답변

성공 전이는 attempt가 STARTED인지 잠근 뒤, 페이지 번호가 1부터 연속인지, provider total이 모든 페이지에서 일치하는지, 영수증 row 합과 결과 row 수가 같은지, ordered manifest와 budget이 맞는지를 대사합니다. 완료 상태 replay는 저장된 receipts·artifact·결과 hash가 모두 같을 때만 기존 결과를 반환하고, 하나라도 다르면 충돌로 실패합니다. attempt 완료와 artifact 연결은 같은 트랜잭션으로 묶어 성공인데 증거가 없는 상태를 막습니다.

### 답변 핵심 키워드

`state machine`, `transaction.atomic`, `select_for_update`, `receipt reconciliation`, `ordered manifest`, `idempotent replay`

### 백지 구현

**구현 목표**

메모리 모델로 축소한 fetch attempt를 STARTED에서 SUCCEEDED로 원자적으로 완료한다. 동일 replay는 허용하고 충돌 replay는 기존 상태를 보존한 채 거부한다.

**인터페이스 또는 함수 시그니처**

```python
def complete_fetch(
    attempt: FetchAttemptState,
    result: FetchResult,
) -> CompletedFetch:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: STARTED attempt와 page receipts/rows/manifest를 가진 결과
- 출력: SUCCEEDED attempt, 검증된 receipts, hash-only artifact를 묶은 `CompletedFetch`

**반드시 만족해야 할 조건**

- 요청 ordinal과 page number가 1부터 빈틈없이 이어져야 한다.
- 모든 페이지의 declared total이 동일하고 전체 row 수와 일치해야 한다.
- 페이지 row·byte 합이 attempt 집계와 일치해야 한다.
- ordered manifest는 page 순서와 body hash/길이를 canonical하게 반영해야 한다.
- 첫 완료는 한 번만 상태를 전이한다.
- 완료 replay는 모든 의미 필드가 동일할 때만 기존 결과를 반환한다.

**경계 조건**

- 한 페이지 결과
- 0 row 성공 응답을 계약상 허용하거나 거부하는 경계
- 영수증 입력 순서가 뒤섞인 경우
- 완료 직후 동일 replay

**실패 조건**

- page gap 또는 중복
- provider total drift
- row/byte/hash 불일치
- FAILED 또는 SUCCEEDED attempt에 다른 결과로 완료 시도
- budget 초과

**제약**

- 원문 body를 저장하지 않고 hash/길이/메타데이터만 저장한다.
- 검증 실패 시 상태·receipt·artifact 중 일부만 변경되어서는 안 된다.
- 25~30분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 정상 다중 페이지가 정확히 한 번 완료된다.
- [ ] 동일 replay가 새 artifact나 receipt를 만들지 않는다.
- [ ] 충돌 replay 후 기존 완료 결과가 변하지 않는다.
- [ ] 페이지 gap과 total drift가 검출된다.
- [ ] manifest가 page 순서 변화에 민감하다.
- [ ] 실패 시 부분 상태 변화가 없다.

### 구현 후 설명할 것

- 상태 머신과 DB constraint의 역할 분담
- idempotent replay를 단순 key dedupe보다 엄격하게 정의한 이유
- hash-only retention의 감사 가능성과 원문 미보존 trade-off
- select-for-update가 보호하는 race와 unique constraint가 보호하는 race

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): persist reconciled fetch receipts`
- 파일 `grocery/source/persistence.py`
- 구성 요소 `CompletedKamisFetch`, `start_kamis_fetch`, `complete_kamis_fetch`, `fail_kamis_fetch`
- 연관 Thread 04, 09, 10

## P08. [Thread 03 / `feat(source): retain partial failure receipts`] 부분 네트워크 실패의 증거 보존과 실패 폐쇄

**우선순위:** S

### 면접 질문

- 3페이지 중 2페이지까지 받은 뒤 timeout이 나면 무엇을 남기고 무엇을 남기지 않아야 하나요?
- 예외 객체에 partial receipts를 담는 설계의 장점과 위험은 무엇인가요?
- 실패 코드에 provider 응답·URL·credential 일부를 그대로 넣으면 왜 안 되나요?
- 꼬리 질문: 실패 처리 자체가 budget 검증에서 실패하면 최종 상태를 어떻게 보장하나요?

### 30초 모범 답변

부분 실패도 감사 대상이므로 이미 완료된 페이지의 순서·상태·길이·hash는 보존하되 원문과 민감한 요청 값은 남기지 않습니다. transport 예외는 고정된 안전 코드와 검증 가능한 partial receipt만 전달하고, persistence 계층이 attempt를 잠근 뒤 receipt budget과 연속성을 다시 검증해 FAILED로 한 번만 전이합니다. 실패 저장 중 검증이 깨지면 전체 트랜잭션을 롤백해 반쯤 실패한 상태를 만들지 않습니다.

### 답변 핵심 키워드

`partial receipts`, `safe error code`, `redaction`, `atomic failure finalization`, `no double finalize`, `rollback`

### 백지 구현

**구현 목표**

부분 영수증을 가진 transport 실패를 받아 attempt를 FAILED로 원자적으로 마감하는 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def fail_fetch(
    attempt: FetchAttemptState,
    error: TransportFailure,
) -> FailedFetch:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: STARTED attempt, 안전한 failure code, 완료된 페이지들의 partial receipt
- 출력: FAILED attempt와 저장된 partial receipts

**반드시 만족해야 할 조건**

- 부분 receipt는 ordinal이 1부터 연속이고 각 body 상태가 자기 필드와 일치해야 한다.
- 실패 코드만 저장하며 원본 예외 메시지·URL·query·credential을 저장하지 않는다.
- row/byte/page budget을 초과하면 어떤 상태도 변경하지 않는다.
- STARTED attempt만 최초 실패 전이를 허용한다.
- 동일 실패 replay는 저장된 증거가 같을 때만 허용한다.
- SUCCEEDED attempt를 실패로 덮어쓰지 않는다.

**경계 조건**

- 첫 요청 전 timeout으로 receipt가 하나도 없는 경우
- 한 페이지 body를 받기 전 연결 실패
- 마지막 페이지 직후 persistence 전에 실패한 경우
- 동일 실패 명령 재시도

**실패 조건**

- partial receipt gap 또는 중복
- body 미수신 상태인데 body hash/bytes가 있는 모순
- 안전 코드 allowlist 밖의 문자열
- 이미 다른 증거로 finalized 된 attempt

**제약**

- 원본 예외의 `str()`을 외부 출력이나 저장 필드로 사용하지 않는다.
- 성공 완료 함수와 동일한 receipt validator를 재사용할 수 있게 경계를 설계한다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 0개·1개·여러 개 partial receipt가 각각 처리된다.
- [ ] 실패 코드 외의 비밀 marker가 결과에 포함되지 않는다.
- [ ] 모순된 receipt가 attempt를 FAILED로 바꾸지 않는다.
- [ ] 동일 replay와 충돌 replay가 구분된다.
- [ ] 성공 attempt는 실패 처리로 변경되지 않는다.
- [ ] 예외 경로에서도 mutable 입력을 변경하지 않는다.

### 구현 후 설명할 것

- 부분 실패 증거가 운영 디버깅과 재시도 판단에 주는 가치
- 예외를 data carrier로 쓸 때의 검증 경계
- 실패 코드 정규화가 보안 경계인 이유
- 실패 persistence까지 원자적이어야 하는 이유

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): retain partial failure receipts`
- 파일 `grocery/source/client.py`
- 파일 `grocery/source/persistence.py`
- 구성 요소 `KamisTransportError`, `partial_page_receipts`, `fail_kamis_fetch`

## P09. [Thread 03 / `feat(source): persist historical request scopes`] 요청 allowlist·redaction·semantic scope hash

**우선순위:** A

### 면접 질문

- HTTP 요청 전체 URL을 감사 로그나 DB에 저장하지 않고도 동일 요청 범위를 식별하려면 어떻게 해야 하나요?
- scope hash에 secret을 넣지 않으면서도 dataset·mode·기간·region 변경에 민감하게 만드는 방법은 무엇인가요?
- 승인된 endpoint 계약을 애플리케이션 검증과 DB constraint에 함께 고정한 이유는 무엇인가요?
- 꼬리 질문: `repr()`이나 예외 메시지에서 query parameter가 새는 것을 어떻게 테스트하겠습니까?

### 30초 모범 답변

요청은 승인된 dataset·mode·host·path와 bounded query field만 허용하고 credential은 별도 전달합니다. 감사용 scope hash는 secret을 제외한 semantic 요청 필드와 계약 버전을 canonical하게 직렬화해 계산하므로 같은 범위는 안정적으로 식별하면서 기간·region·dataset 변경은 감지합니다. 객체 표현과 오류는 고정 코드와 안전 필드만 노출하고, endpoint 계약은 DB에도 고정해 우회 저장을 막습니다.

### 답변 핵심 키워드

`allowlist`, `semantic scope hash`, `credential separation`, `redacted repr`, `endpoint pinning`, `canonical request`

### 백지 구현

**구현 목표**

승인된 역사 데이터 요청을 검증하고, 비밀을 포함하지 않는 canonical scope hash와 redacted 표현을 만든다.

**인터페이스 또는 함수 시그니처**

```python
def prepare_historical_request(
    contract: EndpointContract,
    query: HistoricalQuery,
) -> PreparedHistoricalRequest:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: 승인된 endpoint contract와 기간/region 등의 typed query
- 출력: 전송용 안전 필드와 64자리 `scope_sha256`을 가진 준비 객체

**반드시 만족해야 할 조건**

- dataset와 publication mode, host/path, auth mode의 승인 조합만 허용한다.
- 기간·페이지·행 수 등 각 query field를 canonical 형식과 상한으로 검증한다.
- secret 값은 scope hash, repr, equality debug output에 포함하지 않는다.
- scope hash는 semantic field와 contract version 변화에 민감하다.
- mapping key 순서에 의존하지 않는다.

**경계 조건**

- 동일 query를 다른 dict 순서로 구성한 경우
- 날짜 범위의 양 끝 경계
- 선행 0이 있는 지역 코드
- 월 단위 query와 일 단위 query의 구분

**실패 조건**

- 승인되지 않은 host/path/dataset 조합
- unknown query parameter 또는 duplicate parameter
- 상한을 넘는 기간/페이지 크기
- repr 또는 예외에 secret marker가 포함됨

**제약**

- 실제 HTTP 호출은 구현하지 않는다.
- secret을 dummy 값으로 scope에 넣는 방식도 금지한다.
- 20분 이내 순수 검증/준비 함수로 작성한다.

### 구현 후 자가 검증

- [ ] 동일 semantic query의 scope hash가 안정적이다.
- [ ] 기간·region·dataset 하나가 바뀌면 hash가 바뀐다.
- [ ] secret 값이 어디에도 직렬화되지 않는다.
- [ ] unknown field가 조용히 무시되지 않는다.
- [ ] 월/일 query 계약이 서로 섞이지 않는다.

### 구현 후 설명할 것

- full URL 저장과 semantic receipt 저장의 보안·감사 trade-off
- application allowlist와 DB constraint를 함께 둔 이유
- secret을 equality/hash/repr에서 분리하는 객체 설계
- hash versioning이 필요한 이유

### 원본 확인 위치

- Thread 03, Thread 07, Thread 09
- 커밋 `feat(source): persist historical request scopes`
- 파일 `grocery/source/historical_persistence.py`
- 파일 `grocery/source/historical_client.py`
- 구성 요소 `PreparedHistoricalRequest`, `prepare_historical_request`, `start_historical_fetch`, `HISTORICAL_ENDPOINT_CONTRACTS`

## P10. [Thread 04 / `feat(source): parse kamis recent rows`] 정확한 row 계약, 중복 의미 키, 결정적 결과 hash

**우선순위:** S

### 면접 질문

- 외부 JSON row에서 알 수 없는 field를 무시하지 않고 실패시킨 이유는 무엇인가요?
- 입력 row 순서나 object key 순서가 달라도 같은 parse 결과 hash를 만들려면 무엇을 고정해야 하나요?
- 동일 semantic key에 값만 다른 두 row가 있으면 dedupe할지 실패할지 어떻게 판단했나요?
- 꼬리 질문: 오류 메시지에 잘못된 원문 값을 넣지 않고도 디버깅 가능하게 만드는 방법은 무엇인가요?

### 30초 모범 답변

source schema drift를 조기에 감지하려고 row field 집합과 문자열 타입을 정확히 검사하고 unknown field도 실패시킵니다. 각 row는 승인된 코드-이름-단위 registry와 수치·날짜 invariant를 통과한 뒤 semantic key로 정렬하고 canonical data만 hash합니다. 같은 semantic key가 두 번 나오면 어느 값을 선택할 근거가 없으므로 중복으로 실패하며, 오류는 row index·field·고정 코드만 남겨 원문 노출을 막습니다.

### 답변 핵심 키워드

`exact schema`, `fail-closed parser`, `semantic key`, `stable sort`, `canonical hash`, `redacted error`

### 백지 구현

**구현 목표**

정확한 field 집합을 가진 외부 row 목록을 typed row로 파싱하고, 입력 순서와 무관한 결정적 결과 hash를 생성한다.

**인터페이스 또는 함수 시그니처**

```python
def parse_rows(
    items: object,
    *,
    expected_fields: frozenset[str],
    registry: IdentityRegistry,
) -> ParsedResult:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: JSON에서 얻은 임의 객체와 승인된 exact field set/identity registry
- 출력: semantic key 순서의 immutable row tuple, input count, result hash

**반드시 만족해야 할 조건**

- items는 sequence이고 각 row는 string key mapping이어야 한다.
- field 집합은 expected set과 정확히 같아야 한다.
- 각 field 값은 승인된 문자열 타입과 bounded 형식을 만족해야 한다.
- identity code/name/unit는 registry와 정확히 일치해야 한다.
- semantic key 중복을 거부한다.
- canonical row를 semantic key로 정렬한 뒤 contract version과 함께 hash한다.
- 오류에는 원문 값이 아니라 code, row index, field만 포함한다.

**경계 조건**

- 빈 목록의 허용 여부를 계약으로 명시하는 경우
- 입력 row 순서와 mapping key 순서가 반대인 경우
- 한글·선행 0 코드·두 자리 소수
- 같은 key와 완전히 같은 row가 중복된 경우도 실패하는 정책

**실패 조건**

- missing/unknown field
- 비문자 field name 또는 value type drift
- 코드-이름/단위 drift
- 중복 semantic identity
- 날짜·Decimal·범위 invariant 위반

**제약**

- unknown field를 자동 보존하거나 무시하지 않는다.
- 원문 row 전체를 예외 문자열로 출력하지 않는다.
- 25~30분 이내 구현할 수 있도록 field별 세부 parser는 제공된다고 가정한다.

### 구현 후 자가 검증

- [ ] row 순서를 바꿔도 결과와 hash가 같다.
- [ ] object key 순서를 바꿔도 결과와 hash가 같다.
- [ ] 한 field 값이 바뀌면 hash가 바뀐다.
- [ ] 중복 semantic key가 검출된다.
- [ ] 오류 문자열에 private marker가 없다.
- [ ] 입력 list와 dict를 수정하지 않는다.

### 구현 후 설명할 것

- 관대한 parser보다 strict parser를 택한 이유
- canonicalization 단계와 validation 단계를 분리한 이유
- 중복을 임의 선택하지 않고 failure로 보내는 이유
- 결정적 hash가 재현·승인·publication에 연결되는 방식

### 원본 확인 위치

- Thread 04, Thread 07
- 커밋 `feat(source): parse kamis recent rows`
- 커밋 `feat(source): validate historical row primitives`
- 파일 `grocery/source/historical_parser.py`
- 구성 요소 `parse_recent_price_rows`, `HistoricalRowValidator`, `parse_monthly_price_rows`, `parse_regional_price_rows`, `parse_market_price_rows`

## P11. [Thread 04 / `feat(source): seal reviewed series allowlist`] 검토된 identity/dimension registry와 drift 차단

**우선순위:** A

### 면접 질문

- 외부 API가 새로운 품목 코드나 이름을 보내면 자동 등록하지 않고 실패시킨 이유는 무엇인가요?
- registry를 immutable하게 만들고 evidence revision을 함께 둔 이유는 무엇인가요?
- 시장 코드는 region과 독립 키가 아니라 `(region, market)` 쌍으로 검증해야 하는 이유는 무엇인가요?
- 꼬리 질문: 운영 중 registry 업데이트가 필요할 때 데이터 수집과 publication을 어떻게 분리하겠습니까?

### 30초 모범 답변

새 코드나 이름은 단순 데이터가 아니라 의미 계약 변경이므로 자동 수용하지 않습니다. 승인된 registry는 복사 후 immutable하게 고정하고 evidence revision을 남겨 어떤 codebook을 근거로 파싱했는지 추적합니다. region·market 관계와 품목 코드-이름-단위 조합을 정확히 검증하며, drift는 별도 사람 검토와 새 revision 후에만 수용합니다.

### 답변 핵심 키워드

`reviewed registry`, `immutability`, `evidence revision`, `code-name drift`, `region-market pair`, `human gate`

### 백지 구현

**구현 목표**

승인된 품목·지역·시장 registry에 대해 관측 identity가 정확히 일치하는지 검증한다.

**인터페이스 또는 함수 시그니처**

```python
def validate_observation(
    registry: ReviewedRegistry,
    observation: SourceObservation,
) -> ValidatedIdentity:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: immutable reviewed registry와 source observation
- 출력: 검증된 identity 값 객체

**반드시 만족해야 할 조건**

- 품목 계열의 코드-이름-등급-원문 단위가 exact contract와 일치해야 한다.
- region code-name이 exact mapping과 일치해야 한다.
- market은 `(region_code, market_code)` mapping으로 검증한다.
- unknown code를 registry에 자동 추가하지 않는다.
- registry 생성 후 원본 dict 변경이 내부 상태에 영향을 주지 않아야 한다.

**경계 조건**

- 같은 시장 코드가 다른 region에 존재할 수 있는 경우
- 선행 0 코드
- Unicode 제어 문자가 포함된 이름
- display name만 drift한 경우

**실패 조건**

- unknown code
- code-name mismatch
- region-market 관계 불일치
- 빈 evidence revision
- registry 외부 mutation 가능

**제약**

- fuzzy matching이나 문자열 유사도로 대체하지 않는다.
- registry 업데이트 기능은 구현하지 않는다.
- 15~20분 구현 크기다.

### 구현 후 자가 검증

- [ ] 원본 mapping을 생성 후 변경해도 registry가 변하지 않는다.
- [ ] 선행 0 코드가 유지된다.
- [ ] market이 잘못된 region 아래에서 통과하지 않는다.
- [ ] 이름 drift가 unknown이 아닌 명확한 mismatch로 구분된다.
- [ ] 오류에 원문 전체 row가 노출되지 않는다.

### 구현 후 설명할 것

- 자동 schema evolution을 거부한 이유
- immutable registry와 evidence revision의 감사 가치
- identity drift와 단순 display 변경을 구분하는 기준
- 사람 검토 checkpoint가 ingestion throughput에 주는 trade-off

### 원본 확인 위치

- Thread 04, Thread 07, Thread 08
- 커밋 `feat(source): seal reviewed series allowlist`
- 파일 `grocery/source/registry.py`
- 구성 요소 `INITIAL_RETAIL_IDENTITY_REGISTRY`, `OFFICIAL_DOCS_ZIP_SHA256`, `HistoricalDimensionRegistry`

## P12. [Thread 03 / `feat(source): share bounded transport for history`] bounded pagination과 재시도 폭주 방지

**우선순위:** A

### 면접 질문

- 외부 API pagination에서 page 수, row 수, byte 수를 모두 제한해야 하는 이유는 무엇인가요?
- provider의 total count가 페이지마다 달라질 때 마지막 값을 믿지 않고 실패시키는 이유는 무엇인가요?
- retry를 transport 내부에서 무제한 수행하지 않은 이유는 무엇인가요?
- 꼬리 질문: 부분 receipt를 유지하면서도 중복 row를 막는 semantic boundary는 어디인가요?

### 30초 모범 답변

외부 total이나 body 크기는 신뢰 경계 밖이므로 page·row·byte·timeout을 각각 제한해야 메모리와 실행 시간을 예측할 수 있습니다. 페이지 번호와 declared total이 일관되고 누적 row가 정확히 맞을 때만 성공하며, drift는 임의 보정하지 않습니다. transport는 bounded 시도와 partial receipt까지만 책임지고, 재시도 정책은 scheduler나 호출자가 결정해 중첩 retry와 폭주를 막습니다.

### 답변 핵심 키워드

`bounded I/O`, `pagination reconciliation`, `timeout`, `retry ownership`, `partial receipt`, `resource budget`

### 백지 구현

**구현 목표**

주어진 `fetch_page` 콜백으로 bounded pagination을 수행하고, 성공 또는 안전한 partial failure 정보를 반환한다.

**인터페이스 또는 함수 시그니처**

```python
def fetch_all_pages(
    fetch_page: FetchPage,
    *,
    max_pages: int,
    max_rows: int,
    max_bytes: int,
) -> FetchResult:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: page number를 받아 page result를 반환하는 콜백과 자원 상한
- 출력: 연속 page receipts와 rows, 또는 partial receipts를 가진 bounded failure

**반드시 만족해야 할 조건**

- page 1부터 순서대로 요청한다.
- 각 응답의 declared page/total과 HTTP/provider status를 검증한다.
- page·row·byte 상한을 넘기기 전에 중단한다.
- 누적 row 수가 declared total에 도달하면 종료하고 초과도 실패한다.
- 실패 시 완료된 page receipt를 순서대로 보존한다.
- 내부에서 무제한 retry하지 않는다.

**경계 조건**

- total=0 정책
- 마지막 페이지가 비어 있는 provider 동작
- 정확히 상한에 도달하는 응답
- 첫 페이지 후 total이 바뀌는 응답

**실패 조건**

- page number 불일치
- HTTP/provider 실패
- total drift
- 예산 초과
- timeout/network exception

**제약**

- 동시에 여러 page를 speculative fetch하지 않는다.
- 응답 원문을 오류 문자열에 포함하지 않는다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 한 페이지와 여러 페이지가 모두 정상 종료된다.
- [ ] 정확한 상한과 상한+1이 구분된다.
- [ ] partial failure에 완료된 receipt만 들어 있다.
- [ ] total drift를 마지막 값으로 덮지 않는다.
- [ ] 호출 횟수가 max_pages를 넘지 않는다.

### 구현 후 설명할 것

- 다중 자원 budget을 둔 이유
- retry 책임을 scheduler로 올린 이유
- 순차 pagination의 단순성 대 병렬 처리 성능 trade-off
- provider total을 consistency signal로 사용하는 방법

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): share bounded transport for history`
- 파일 `grocery/source/client.py`
- 구성 요소 `KamisHttpClient.fetch_historical_prices`, `_fetch_prices`
- 연관 Thread 20, 21

## P13. [Thread 04 / `feat(audit): reconcile deterministic parses`] artifact·parse run 상태 머신과 count reconciliation

**우선순위:** A

### 면접 질문

- 왜 artifact를 원문 payload가 아니라 ordered manifest hash와 크기 메타데이터로 모델링했나요?
- parse run의 idempotency key에 artifact, parser revision, configuration hash가 모두 필요한 이유는 무엇인가요?
- `total = accepted + out_of_scope + quarantined` 같은 count invariant는 어떤 오류를 잡나요?
- 꼬리 질문: 같은 입력에서 result hash가 달라지면 retry할지 quarantine할지 어떻게 판단합니까?

### 30초 모범 답변

artifact는 수집된 페이지들의 순서·hash·길이를 묶은 불변 증거로 두고 원문은 보존하지 않았습니다. parse run은 같은 artifact·parser revision·configuration에서 재실행 가능한 단위이며, 동일 key의 result hash가 달라지면 비결정성으로 취급해야 합니다. 상태와 count 합계, missing reference가 accepted 안에 포함된다는 invariant를 DB까지 내려 부분 집계나 잘못된 완료를 막습니다.

### 답변 핵심 키워드

`hash-only artifact`, `parse idempotency`, `configuration hash`, `count reconciliation`, `nondeterminism`, `quarantine`

### 백지 구현

**구현 목표**

artifact와 parser/config 버전을 키로 parse run을 시작하거나 replay하고, count 및 result hash를 검증해 완료한다.

**인터페이스 또는 함수 시그니처**

```python
def complete_parse_run(
    run: ParseRunState,
    result: ParsedGeneration,
) -> ParseRunState:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: STARTED parse run과 count/result hash를 가진 parse 결과
- 출력: VALIDATED 또는 명시적 실패 상태의 parse run

**반드시 만족해야 할 조건**

- total, accepted, out_of_scope, quarantined는 음수가 아니다.
- `total == accepted + out_of_scope + quarantined`를 만족한다.
- missing reference count는 accepted 이하이다.
- result hash는 canonical lowercase SHA-256이다.
- 완료 replay는 count와 hash가 완전히 같을 때만 허용한다.
- 동일 idempotency key의 다른 결과는 비결정성 충돌로 처리한다.

**경계 조건**

- accepted=0인 검증 결과
- 모든 row가 out of scope인 경우
- 동일 결과 replay
- parser revision만 바뀐 재실행

**실패 조건**

- count 합계 불일치
- invalid status transition
- 잘못된 result hash
- 동일 key의 결과 drift

**제약**

- 원문 payload를 parse run에 저장하지 않는다.
- 상태 변경과 count 저장은 원자적이어야 한다.
- 20분 내 구현한다.

### 구현 후 자가 검증

- [ ] 정상 count 조합이 완료된다.
- [ ] 합계가 1만 달라도 실패한다.
- [ ] 동일 replay가 새 run을 만들지 않는다.
- [ ] parser/config 버전 변화는 다른 run으로 구분된다.
- [ ] result hash drift가 기존 run을 덮어쓰지 않는다.

### 구현 후 설명할 것

- artifact와 parse run을 분리한 이유
- configuration hash가 재현성에 필요한 이유
- count invariant를 DB에 둘 가치
- 비결정성 발견 시 fail-closed 정책의 장단점

### 원본 확인 위치

- Thread 04
- 커밋 `feat(audit): reconcile deterministic parses`
- 파일 `grocery/models.py`
- 구성 요소 `SourceArtifact`, `ParseRun`, `build_source_artifact`, `start_or_get_kamis_parse_run`, `complete_kamis_parse_generation`
