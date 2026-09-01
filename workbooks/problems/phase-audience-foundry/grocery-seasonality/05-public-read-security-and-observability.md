# 공개 읽기·URL·HTTP 보안·관측성

active-only read, canonical GET, query privacy, redacted event logging을 다룬다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P24. [Thread 06 / `test(public): enforce bounded publication reads`] active-only 공개 읽기와 membership 무결성·query bound

**우선순위:** A

### 면접 질문

- 공개 화면이 최신 parse나 가장 최근 생성 row를 직접 읽지 않고 active revision만 읽어야 하는 이유는 무엇인가요?
- 선택한 series id가 publication member인지 확인하지 않고 snapshot table을 조회하면 어떤 누출이 가능한가요?
- N개 항목을 읽을 때 query 수가 row 수에 비례하지 않게 만들려면 ORM에서 어떤 전략을 쓰나요?
- 꼬리 질문: 중복 publication member나 reference fact 누락을 빈 값으로 보여 주지 않고 실패시키는 이유는 무엇인가요?

### 30초 모범 답변

공개 가능 여부는 수집 시각이 아니라 review·seal·activation으로 결정되므로 current channel이 가리키는 immutable revision만 읽습니다. 요청 series는 그 revision의 membership을 통해서만 조회하고, 중복·누락·불완전 comparison은 데이터 무결성 오류로 처리합니다. `select_related`와 bounded `Prefetch`로 필요한 관계를 고정 query 수에 읽고, 공개 result와 page 상한으로 메모리 사용을 제한합니다.

### 답변 핵심 키워드

`active revision`, `published read model`, `membership boundary`, `select_related`, `prefetch_related`, `bounded result`, `integrity error`

### 백지 구현

**구현 목표**

active revision과 요청 series id 목록에서 publication member만 bounded하게 반환하는 순수 함수를 구현하고, ORM query plan을 별도로 설명한다.

**인터페이스 또는 함수 시그니처**

```python
def select_public_members(
    revision: ActiveRevision,
    requested_ids: Sequence[UUID],
    *,
    limit: int,
) -> list[PublicationEntry]:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: active revision membership과 사용자 요청 id 순서
- 출력: 요청 순서를 보존한 공개 entry 목록

**반드시 만족해야 할 조건**

- active/sealed revision이 아니면 읽지 않는다.
- 요청 수와 limit을 검증한다.
- revision member만 반환한다.
- 같은 series의 중복 membership을 무결성 오류로 처리한다.
- publication 밖 id 처리 정책을 명시하고 일관되게 적용한다.
- available comparison의 필수 value/change fields가 완전한지 검증한다.

**경계 조건**

- 빈 요청
- 최대 5개 선택
- 요청 순서가 publication 정렬과 다른 경우
- 일부 id가 현재 publication에 없는 경우

**실패 조건**

- inactive revision read
- 중복 membership
- 불완전 reference/change fact
- limit 초과
- publication 밖 candidate 직접 조회

**제약**

- 외부 API나 candidate/parse table을 조회하지 않는다.
- 순수 함수는 O(n+m)으로 작성하고 ORM query 설계는 설명으로 제시한다.
- 20분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 요청 순서가 보존된다.
- [ ] publication 밖 id가 노출되지 않는다.
- [ ] 중복 member가 검출된다.
- [ ] max 선택 수 경계가 맞다.
- [ ] ORM 설명에서 N+1 query가 생기지 않는다.

### 구현 후 설명할 것

- active pointer를 공개 authorization 경계로 보는 이유
- read model과 ingestion model 분리
- prefetch로 query 수를 고정하는 방식과 메모리 trade-off
- 무결성 오류와 정상 empty 상태의 구분

### 원본 확인 위치

- Thread 06, Thread 16
- 커밋 `test(public): enforce bounded publication reads`
- 파일 `grocery/public_read.py`
- 구성 요소 `load_active_publication`, `publication_entries`, `publication_entries_for_series`, `publication_candidate_entries`
- 구성 요소 `load_active_historical_publication`

## P25. [Thread 15 / `test(public): lock URL and geometry contracts`] canonical GET 상태와 no-JS 선택 목록

**우선순위:** A

### 면접 질문

- unknown query parameter와 duplicate parameter를 무시하지 않고 400으로 거부한 이유는 무엇인가요?
- `page=01`, 대문자 UUID, 비정규 날짜처럼 의미는 같아 보이는 입력을 canonical하지 않다고 거부하는 이유는 무엇인가요?
- 선택 목록을 session이 아니라 최대 5개 UUID의 GET URL로 만든 trade-off는 무엇인가요?
- 꼬리 질문: validation error에 raw query value를 반사하지 않고 사용자에게 수정 지점을 알려 주려면 어떻게 하나요?

### 30초 모범 답변

공개 URL을 상태의 정본으로 쓰므로 같은 의미가 여러 표현을 갖지 않게 canonical 문법을 강제합니다. unknown이나 duplicate field는 cache·로그·링크 공유에서 모호성을 만들고 private value 반사 위험도 있어 고정 오류로 거부합니다. 선택 목록은 최대 5개 canonical UUID를 GET으로 표현해 세션 없이 공유·재현 가능하게 했고, 누락 publication member는 명시적 partial 상태로 처리합니다.

### 답변 핵심 키워드

`canonical URL`, `strict query parser`, `duplicate key`, `no reflection`, `stateless GET`, `bounded selection`, `no-JS`

### 백지 구현

**구현 목표**

MultiDict 형태의 공개 query를 strict하게 파싱하고 canonical 상태와 재직렬화된 query string을 만든다.

**인터페이스 또는 함수 시그니처**

```python
def parse_public_query(
    raw: MultiDict[str, str],
) -> PublicQueryState:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: 중복 key를 표현할 수 있는 raw query mapping
- 출력: typed query state와 canonical query serialization

**반드시 만족해야 할 조건**

- route별 allowlist 밖 key를 거부한다.
- 단일값 field의 duplicate를 거부한다.
- page는 선행 0 없는 양의 canonical decimal이어야 한다.
- UUID는 canonical 소문자 하이픈 형식이어야 한다.
- 날짜는 실제 달력에 존재하는 `YYYY-MM-DD`여야 한다.
- selection은 최대 5개이며 중복 UUID를 거부한다.
- 오류에는 raw value를 포함하지 않는다.

**경계 조건**

- 빈 query
- page=1의 생략 또는 표현 정책
- selection 0개와 5개
- 윤년 날짜
- 같은 UUID의 중복

**실패 조건**

- unknown/duplicate key
- 비정규 UUID·날짜·page
- 금지된 field 조합
- selection limit 초과
- private marker 반사

**제약**

- Django Form을 사용하지 않고도 핵심 parser를 구현할 수 있게 표준 타입만 쓴다.
- 20~25분 구현 크기다.
- 자동 normalize 후 redirect하지 않고 이 문제에서는 strict reject한다.

### 구현 후 자가 검증

- [ ] canonical 입력을 parse→serialize하면 동일 문자열이 된다.
- [ ] 대문자 또는 하이픈 없는 UUID가 거부된다.
- [ ] page=01이 거부된다.
- [ ] duplicate key가 마지막 값으로 덮이지 않는다.
- [ ] 오류에 raw marker가 없다.
- [ ] selection 순서가 보존된다.

### 구현 후 설명할 것

- strict reject와 normalize redirect의 trade-off
- stateless GET 상태가 접근성·공유성·개인정보에 미치는 영향
- MultiDict duplicate 처리의 중요성
- validation message와 raw value reflection을 분리하는 방법

### 원본 확인 위치

- Thread 15
- 커밋 `test(public): lock URL and geometry contracts`
- 파일 `grocery/tests/test_vnext_public_read_contract.py`
- 구성 요소 `CatalogForm`, `HistoryForm`, `MarketsForm`, `RegionsForm`, `parse_selection_query`
- 연관 Thread 14, 18

## P26. [Thread 14 / `fix(public): keep searches private and uncached`] 공개 query privacy와 HTTP fail-closed 헤더

**우선순위:** A

### 면접 질문

- 비로그인 공개 GET 응답에도 `no-store`를 적용한 이유는 무엇인가요?
- `Referrer-Policy: no-referrer`가 검색 query privacy에 어떻게 기여하나요?
- GET/HEAD 외 메서드를 `require_safe`로 막는 것과 CSRF 방어는 어떤 차이가 있나요?
- 꼬리 질문: validation error, 404, 503 같은 실패 응답에도 같은 보안 헤더가 필요한 이유는 무엇인가요?

### 30초 모범 답변

검색어와 선택 상태가 URL에 있으므로 공유 cache나 browser intermediary에 남지 않게 공개 동적 응답을 `no-store`로 처리하고, 외부 링크에 query가 넘어가지 않도록 `no-referrer`를 사용했습니다. route는 GET/HEAD만 허용해 예상하지 않은 state-changing 요청 면적을 줄입니다. 보안 헤더는 성공뿐 아니라 오류 응답에도 일관되게 적용하고, 오류 본문이나 로그에 raw query를 반사하지 않습니다.

### 답변 핵심 키워드

`Cache-Control no-store`, `Referrer-Policy no-referrer`, `safe methods`, `error response headers`, `query privacy`, `no reflection`

### 백지 구현

**구현 목표**

공개 응답에 일관된 보안·privacy header를 적용하고 unsafe method를 거부하는 작은 middleware 또는 decorator 조합을 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def secure_public_response(
    request: Request,
    response: Response,
) -> Response:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: method/path/query를 가진 request와 status/body를 가진 response
- 출력: 보안 header가 적용되거나 method 오류가 된 response

**반드시 만족해야 할 조건**

- 공개 route는 GET/HEAD만 허용한다.
- 동적 공개 응답에 `Cache-Control`의 `no-store`를 포함한다.
- `Referrer-Policy`는 `no-referrer`다.
- 승인된 고정 보안 header를 일관되게 적용한다.
- 400/404/500/503에도 동일한 기본 header를 적용한다.
- query value를 header나 오류 body에 복사하지 않는다.

**경계 조건**

- HEAD 응답
- 이미 일부 Cache-Control directive가 있는 응답
- redirect 응답
- 예외 처리기로 생성된 오류 응답

**실패 조건**

- POST가 view 로직까지 도달
- 성공 응답에만 header 적용
- 기존 header에 private value가 섞임
- query string을 Location 또는 오류 본문에 반사

**제약**

- 인증 session용 Admin 정책과 공개 route 정책을 혼합하지 않는다.
- 15~20분 구현 크기다.
- header 값은 고정 상수로 둔다.

### 구현 후 자가 검증

- [ ] GET/HEAD 정상, POST 거부가 확인된다.
- [ ] 성공과 오류 status 모두 header가 같다.
- [ ] `no-store`가 중복 또는 모순 없이 존재한다.
- [ ] raw query marker가 body/header에 없다.
- [ ] 기존 unrelated header가 손실되지 않는다.

### 구현 후 설명할 것

- public GET에도 privacy 위험이 있는 이유
- cache control과 referrer policy가 해결하는 서로 다른 경로
- require-safe와 CSRF의 역할 차이
- middleware ordering이 fail-closed에 미치는 영향

### 원본 확인 위치

- Thread 14
- 커밋 `fix(public): keep searches private and uncached`
- 커밋 `fix(security): suppress public query referrers`
- 파일 `grocery/security.py`
- 파일 `grocery/views.py`; decorator `require_safe`

## P27. [Thread 19 / `feat(ops): add redacted structured events`] allowlist structured logging과 request context 정리

**우선순위:** A

### 면접 질문

- 일반 `logging` formatter가 `record.msg`, `args`, exception traceback을 그대로 쓰지 않게 한 이유는 무엇인가요?
- 로그 field를 denylist가 아니라 allowlist로 선택한 이유는 무엇인가요?
- request id를 `ContextVar`에 넣을 때 reset을 빠뜨리면 어떤 비동기 또는 worker 재사용 문제가 생기나요?
- 꼬리 질문: JSON 로그 크기와 cardinality를 왜 제한해야 하나요?

### 30초 모범 답변

로그는 외부 입력과 예외 문자열이 새기 쉬운 경계라 고정 event name, severity, request id, deploy version, 승인된 UUID와 상태 field만 allowlist로 직렬화합니다. formatter는 `record.msg`, args, traceback, 임의 extra를 신뢰하지 않고 검증된 event payload만 bounded JSON으로 만듭니다. request middleware는 ContextVar token을 `finally`에서 reset해 worker가 다음 요청에 이전 id를 재사용하지 않게 합니다.

### 답변 핵심 키워드

`allowlist logging`, `redaction`, `bounded JSON`, `ContextVar token`, `finally reset`, `cardinality`, `fixed event code`

### 백지 구현

**구현 목표**

승인된 field만 JSON으로 출력하는 formatter와 request-id context manager를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def format_audit_event(event: AuditEvent) -> str:
    # 직접 구현
    raise NotImplementedError


@contextmanager
def request_context(request_id: str) -> Iterator[None]:
    # 직접 구현
    raise NotImplementedError
```

**입력과 출력**

- 입력: 고정 event name/severity와 선택적 승인 field를 가진 event
- 출력: 한 줄 bounded canonical JSON
- context manager: 요청 범위에서만 request id를 노출하고 종료 시 원상 복구

**반드시 만족해야 할 조건**

- event name과 key 집합을 allowlist로 검증한다.
- 문자열/UUID/count 길이와 범위를 제한한다.
- 원본 exception/message/args/arbitrary extra를 출력하지 않는다.
- JSON key 순서와 separator를 고정한다.
- ContextVar token을 정상·예외 경로 모두 reset한다.
- 중첩 context에서 바깥 request id가 복구되어야 한다.

**경계 조건**

- optional field가 없는 event
- 최대 길이 값
- 중첩 request context
- context 내부에서 예외 발생

**실패 조건**

- unknown event/key
- raw query/URL/credential marker 입력
- 과도한 JSON 크기
- ContextVar 누수
- 비직렬화 타입

**제약**

- generic `str(value)` fallback을 사용하지 않는다.
- 로그 실패가 민감 값을 담은 2차 예외로 이어지지 않게 고정 오류를 사용한다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 승인 event가 한 줄 JSON으로 나온다.
- [ ] unknown key와 private marker가 출력되지 않는다.
- [ ] 예외 경로 후 request id가 초기값으로 돌아온다.
- [ ] 중첩 context가 바깥 값을 복구한다.
- [ ] 출력 크기 상한이 적용된다.
- [ ] formatter가 `record.msg`에 의존하지 않는다.

### 구현 후 설명할 것

- allowlist가 denylist보다 안전한 이유
- 로그 가용성과 fail-closed redaction의 균형
- ContextVar가 thread-local보다 적합한 경우와 token reset의 의미
- 고 cardinality field가 관측 비용과 개인정보에 미치는 영향

### 원본 확인 위치

- Thread 19
- 커밋 `feat(ops): add redacted structured events`
- 커밋 `feat(ops): wire safe request logging`
- 파일 `grocery/observability.py`
- 구성 요소 `ObservabilityValidationError`, `log_event`, `RequestIdMiddleware`
