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
  - **모범답변:** **프로젝트 특수사항:** active revision의 membership과 각 snapshot의 reference/change fact가 공개 계약이므로 중복이나 필수 fact 누락은 무결성 오류로 실패시킵니다. **일반 원칙:** 권한·감사 경계의 불변식 위반을 빈 값으로 숨기면 손상을 정상적인 “데이터 없음”으로 오인하므로 fail-closed가 안전합니다.

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
    if not revision.is_active or not revision.is_sealed:
        raise PublicReadError("revision_not_active")
    if type(limit) is not int or limit < 1 or limit > 5:
        raise PublicReadError("selection_limit_invalid")
    if len(requested_ids) > limit or len(set(requested_ids)) != len(requested_ids):
        raise PublicReadError("selection_invalid")
    if not requested_ids:
        return []

    requested = set(requested_ids)
    # 호출자는 원본 ORM처럼 requested_ids로 먼저 제한한 entry만 이 순수 함수에 넘긴다.
    entries = tuple(revision.entries)
    members: dict[UUID, PublicationEntry] = {}
    for entry in entries:
        series_id = entry.series_id
        if series_id not in requested:
            raise PublicReadError("publication_read_model_not_bounded")
        if series_id in members:
            raise PublicReadError("duplicate_publication_member")

        try:
            reference = entry.reference
            change = reference.change_fact
        except (AttributeError, ObjectDoesNotExist):
            raise PublicReadError("comparison_incomplete") from None
        if reference.value_status == "AVAILABLE":
            if (
                reference.value is None
                or reference.unavailable_reason is not None
                or change.signed_difference is None
                or change.signed_percentage is None
                or change.direction not in {"LOWER", "EQUAL", "HIGHER"}
            ):
                raise PublicReadError("comparison_incomplete")
            if (
                change.direction == "LOWER"
                and not (change.signed_difference < 0 and change.signed_percentage < 0)
                or change.direction == "EQUAL"
                and not (change.signed_difference == 0 and change.signed_percentage == 0)
                or change.direction == "HIGHER"
                and not (change.signed_difference > 0 and change.signed_percentage > 0)
            ):
                raise PublicReadError("comparison_inconsistent")
        elif reference.value_status == "UNAVAILABLE":
            if (
                reference.value is not None
                or reference.unavailable_reason != "SOURCE_VALUE_MISSING"
                or change.direction != "UNAVAILABLE"
                or change.signed_difference is not None
                or change.signed_percentage is not None
            ):
                raise PublicReadError("comparison_incomplete")
        else:
            raise PublicReadError("comparison_state_invalid")
        members[series_id] = entry

    # 원본 public read와 같이 publication 밖 id는 누출 없이 생략한다.
    return [members[series_id] for series_id in requested_ids if series_id in members]
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
  - **모범답변:** 이 프로젝트에서 공개 승인은 수집·parse 완료가 아니라 review, seal, activation을 거쳐야 생깁니다. 따라서 channel의 current pointer가 가리키는 sealed revision만 공개 가능하고, 최신 candidate를 직접 읽으면 검수 전 데이터가 노출될 수 있습니다.
- read model과 ingestion model 분리
  - **모범답변:** ingestion model은 재시도·부분 실패·candidate 같은 운영 상태를 보존하지만, read model은 활성 revision의 immutable membership과 검증된 fact만 제공합니다. 이 분리로 공개 요청이 source 상태나 credential, 최신 수집 성공 여부에 결합되지 않습니다.
- prefetch로 query 수를 고정하는 방식과 메모리 trade-off
  - **모범답변:** 원본은 먼저 `snapshot__series_id__in=requested_ids`로 최대 다섯 membership만 제한하고 entry·series를 `select_related`로, 선택 period의 reference·change를 제한된 `Prefetch`로 읽습니다. 그래서 1개와 5개 선택 모두 query 수가 같지만, prefetch 결과는 메모리에 materialize되므로 요청 수 상한을 둡니다. 위 순수 함수는 이렇게 bounded하게 만든 read model만 받습니다.
- 무결성 오류와 정상 empty 상태의 구분
  - **모범답변:** 요청 id가 현재 publication member가 아니면 정책상 정상적인 부분 결과로 생략할 수 있습니다. 반면 같은 series의 membership 중복, reference 개수 이상, available fact의 값 누락은 봉인된 read model 자체의 손상이므로 empty로 숨기지 않고 고정 무결성 오류로 처리합니다.

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
  - **모범답변:** **프로젝트 특수사항:** field별 고정 오류 문구와 code로 `page`, `date`, `region`처럼 수정할 항목만 알리고 입력값은 넣지 않습니다. **일반 원칙:** parser가 raw value와 사용자 메시지를 분리하고 로그에도 고정 code만 남기면 디버깅 가능성을 유지하면서 반사·로그 유출을 막을 수 있습니다.

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
    allowed = {"page", "date", "series"}
    if set(raw.keys()) - allowed:
        raise PublicQueryError("unknown_parameter")

    for name in ("page", "date"):
        if len(raw.getlist(name)) > 1:
            raise PublicQueryError("duplicate_parameter")

    page_text = raw.getlist("page")[0] if raw.getlist("page") else "1"
    if re.fullmatch(r"(?:[1-9]|[1-9][0-9]|100)", page_text) is None:
        raise PublicQueryError("page_invalid")
    page = int(page_text)

    date_text = raw.getlist("date")[0] if raw.getlist("date") else ""
    parsed_date = None
    if date_text:
        if re.fullmatch(r"[0-9]{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12][0-9]|3[01])", date_text) is None:
            raise PublicQueryError("date_invalid")
        try:
            parsed_date = date.fromisoformat(date_text)
        except ValueError:
            raise PublicQueryError("date_invalid") from None
        if parsed_date.isoformat() != date_text:
            raise PublicQueryError("date_invalid")

    raw_series = raw.getlist("series")
    if len(raw_series) > 5:
        raise PublicQueryError("selection_limit")
    series_ids: list[UUID] = []
    seen: set[UUID] = set()
    for value_text in raw_series:
        if re.fullmatch(
            r"[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}",
            value_text,
        ) is None:
            raise PublicQueryError("series_invalid")
        value = UUID(value_text)
        if str(value) != value_text:
            raise PublicQueryError("series_invalid")
        if value in seen:
            raise PublicQueryError("duplicate_series")
        seen.add(value)
        series_ids.append(value)

    # selection route와 date/page route의 상태를 한 URL에 섞지 않는다.
    if series_ids and (date_text or raw.getlist("page")):
        raise PublicQueryError("field_combination_invalid")

    pairs: list[tuple[str, str]] = []
    if date_text:
        pairs.append(("date", date_text))
    if raw.getlist("page"):
        pairs.append(("page", str(page)))
    pairs.extend(("series", str(value)) for value in series_ids)
    canonical_query = urlencode(pairs)
    return PublicQueryState(
        page=page,
        date=parsed_date,
        series_ids=tuple(series_ids),
        canonical_query=canonical_query,
    )
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
  - **모범답변:** 프로젝트는 `page=01`, 대문자 UUID, 비정규 날짜를 자동 보정하지 않고 거부해 한 상태에 한 canonical URL만 남깁니다. normalize redirect는 사용성은 좋지만 원 입력을 `Location`에 반사하거나 cache key를 늘릴 수 있어, 쓴다면 검증된 typed 값으로만 URL을 재구성해야 합니다.
- stateless GET 상태가 접근성·공유성·개인정보에 미치는 영향
  - **모범답변:** 최대 다섯 UUID를 GET에 두면 세션과 JavaScript 없이 링크 공유, 뒤로 가기, 재현이 가능합니다. 반면 URL은 history·referrer·로그에 남을 수 있으므로 선택 수를 제한하고 `no-store`, `no-referrer`, raw-value 비반사를 함께 적용했습니다.
- MultiDict duplicate 처리의 중요성
  - **모범답변:** 일반 dict로 바꾸면 마지막 값이 앞 값을 덮어 validation 의미가 달라집니다. 프로젝트의 단일값 field는 `getlist` 길이로 중복을 거부하고, 반복 가능한 selection은 원본에서 입력 개수 상한을 먼저 검사한 뒤 첫 등장 순서로 deduplicate합니다. 이 축약 문제는 명시된 조건에 따라 UUID 중복 자체를 거부합니다.
- validation message와 raw value reflection을 분리하는 방법
  - **모범답변:** parser는 `date_invalid`, `duplicate_parameter`처럼 고정 code를 만들고 UI는 field와 code에 대응하는 고정 문구를 표시합니다. 예외 문자열, 응답 본문, 로그 어디에도 supplied value를 연결하지 않는 것이 핵심입니다.

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
  - **모범답변:** **프로젝트 특수사항:** 보안 middleware가 downstream status와 무관하게 고정 header를 덮어써 400·404·500에도 성공 응답과 같은 privacy 경계를 적용합니다. **일반 원칙:** 오류 응답도 query를 담거나 cache되고 외부 navigation의 출발점이 될 수 있으므로 보안 정책은 status와 무관해야 합니다.

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
    fixed_headers = {
        "Cache-Control": "no-store",
        "Content-Security-Policy": (
            "default-src 'self'; script-src 'none'; style-src 'self'; "
            "img-src 'self' data:; font-src 'self'; connect-src 'self'; "
            "frame-src 'none'; frame-ancestors 'none'; object-src 'none'; "
            "base-uri 'none'; form-action 'self'"
        ),
        "Permissions-Policy": "camera=(), geolocation=(), microphone=(), payment=()",
        "Cross-Origin-Opener-Policy": "same-origin",
        "Cross-Origin-Resource-Policy": "same-origin",
        "Referrer-Policy": "no-referrer",
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
    }
    # 입력과 status에 무관한 고정값으로 덮어써 기존 cache 지시의 모순도 제거한다.
    for name, value in fixed_headers.items():
        response.headers[name] = value
    if request.method == "HEAD":
        response.body = b""
    return response


def require_secure_public_safe(view):
    from functools import wraps

    @wraps(view)
    def wrapped(request: Request, *args: object, **kwargs: object) -> Response:
        # 원본의 @require_safe처럼 unsafe method는 view 호출 전에 차단한다.
        if request.method not in {"GET", "HEAD"}:
            response = Response(status=405, body=b"", headers={"Allow": "GET, HEAD"})
        else:
            response = view(request, *args, **kwargs)
        return secure_public_response(request, response)

    return wrapped
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
  - **모범답변:** 인증이 없어도 검색어와 선택 UUID는 browser history, intermediary cache, access log에 남을 수 있는 사용자 상태입니다. 그래서 공개라는 사실과 cache·referrer에 남겨도 된다는 판단을 분리해야 합니다.
- cache control과 referrer policy가 해결하는 서로 다른 경로
  - **모범답변:** `Cache-Control: no-store`는 browser와 intermediary가 응답을 저장하는 경로를 막고, `Referrer-Policy: no-referrer`는 그 페이지에서 외부로 이동할 때 URL query가 Referer로 전송되는 경로를 막습니다. 하나가 다른 하나를 대체하지 않습니다.
- require-safe와 CSRF의 역할 차이
  - **모범답변:** `require_safe`는 route 자체를 GET/HEAD로 제한해 unsafe method가 view에 도달하지 않게 합니다. CSRF는 cookie 기반 권한으로 상태를 바꾸는 허용된 unsafe 요청이 다른 origin에서 위조되는 것을 막으므로, method allowlist와 보호 대상이 다릅니다.
- middleware ordering이 fail-closed에 미치는 영향
  - **모범답변:** 보안 header middleware가 예외 처리 뒤의 모든 응답을 감싸야 오류 응답에도 header가 붙습니다. 반대로 middleware 밖에서 예외 응답이 만들어지면 정책을 우회하므로, 프로젝트처럼 입력과 status에 무관한 마지막 응답 경계로 배치하고 통합 테스트로 확인합니다.

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
  - **모범답변:** **프로젝트 특수사항:** 고정 key, 짧은 정규식 값, UUID와 제한된 lifecycle token만 허용해 event 크기를 구조적으로 제한합니다. **일반 원칙:** 큰 event는 수집·저장 비용을 높이고 고유 값을 metric label로 쓰면 cardinality가 폭증하므로, 고유 식별자는 상관관계에만 쓰고 집계 label에서는 제외합니다.

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
    severities = {"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"}
    optional_order = (
        "request_id",
        "deploy_version",
        "command_run_id",
        "lifecycle_id",
        "lifecycle_status",
        "lifecycle_event",
        "count",
    )
    fields = event.fields
    if event.severity not in severities or set(fields) - set(optional_order):
        raise AuditEventError("event_invalid")
    if re.fullmatch(r"[a-z][a-z0-9]*(?:[._-][a-z0-9]+){1,7}", event.name) is None:
        raise AuditEventError("event_invalid")
    if event.timestamp.tzinfo is None:
        raise AuditEventError("timestamp_invalid")

    uuid_fields = {"request_id", "command_run_id", "lifecycle_id"}
    uuid_pattern = re.compile(
        r"[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}"
    )
    token_fields = {"lifecycle_status", "lifecycle_event"}
    normalized: dict[str, str | int] = {}
    for name in optional_order:
        if name not in fields:
            continue
        value = fields[name]
        if name in uuid_fields:
            if type(value) is UUID:
                normalized[name] = str(value)
            else:
                try:
                    parsed = (
                        UUID(value)
                        if type(value) is str and uuid_pattern.fullmatch(value)
                        else None
                    )
                except ValueError:
                    parsed = None
                if parsed is None or str(parsed) != value:
                    raise AuditEventError("field_invalid")
                normalized[name] = value
        elif name == "deploy_version":
            if type(value) is not str or re.fullmatch(r"[0-9a-f]{7,40}", value) is None:
                raise AuditEventError("field_invalid")
            normalized[name] = value
        elif name in token_fields:
            if type(value) is not str or re.fullmatch(r"[A-Z][A-Z0-9_]{0,63}", value) is None:
                raise AuditEventError("field_invalid")
            normalized[name] = value
        elif type(value) is int and not isinstance(value, bool) and 0 <= value <= 1_000_000:
            normalized[name] = value
        else:
            raise AuditEventError("field_invalid")

    payload: dict[str, str | int] = {
        "timestamp": event.timestamp.astimezone(UTC).isoformat(timespec="milliseconds").replace("+00:00", "Z"),
        "severity": event.severity,
        "message_code": event.name,
    }
    for name in optional_order:
        if name in normalized:
            payload[name] = normalized[name]
    line = json.dumps(payload, ensure_ascii=True, separators=(",", ":"))
    if len(line.encode("utf-8")) > 2_048:
        raise AuditEventError("event_too_large")
    return line


@contextmanager
def request_context(request_id: str) -> Iterator[None]:
    uuid_pattern = re.compile(
        r"[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}"
    )
    if type(request_id) is not str or uuid_pattern.fullmatch(request_id) is None:
        raise AuditEventError("request_id_invalid")
    try:
        parsed = UUID(request_id)
    except ValueError:
        raise AuditEventError("request_id_invalid") from None
    if str(parsed) != request_id:
        raise AuditEventError("request_id_invalid")

    token = _request_id_context.set(request_id)
    try:
        yield
    finally:
        # Token reset은 중첩 context에서 단순 None 대입과 달리 바깥 값을 복원한다.
        _request_id_context.reset(token)
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
  - **모범답변:** denylist는 새 field나 예상하지 못한 객체 표현을 놓치지만 allowlist는 승인한 envelope 밖의 값을 애초에 순회·문자열화하지 않습니다. 원본 formatter도 `record.msg`, args, traceback, 임의 extra를 무시하고 정규화된 event만 출력합니다.
- 로그 가용성과 fail-closed redaction의 균형
  - **모범답변:** 호출 시 잘못된 event는 고정 validation 오류로 거부하고, formatter 경계에서는 raw record를 출력하는 대신 `observability.invalid_event` 같은 안전한 fallback을 사용할 수 있습니다. 이렇게 하면 민감 값을 구제 로그에 남기지 않으면서 로깅 계약 위반 자체는 관측할 수 있습니다.
- ContextVar가 thread-local보다 적합한 경우와 token reset의 의미
  - **모범답변:** `ContextVar`는 async task별 context 전파를 지원해 같은 thread에서 여러 요청이 교차하는 환경에 맞습니다. `set`이 반환한 token을 `finally`에서 reset하면 예외와 중첩 호출에서도 바로 이전 값을 정확히 복구합니다.
- 고 cardinality field가 관측 비용과 개인정보에 미치는 영향
  - **모범답변:** 사용자 입력이나 URL을 field로 허용하면 거의 모든 event가 고유해져 인덱스·집계 비용이 커지고 개인 상태도 장기 보존될 수 있습니다. 프로젝트는 고정 message code와 제한된 UUID·lifecycle 상태만 허용하고 원문 payload는 기록하지 않습니다.

### 원본 확인 위치

- Thread 19
- 커밋 `feat(ops): add redacted structured events`
- 커밋 `feat(ops): wire safe request logging`
- 파일 `grocery/observability.py`
- 구성 요소 `ObservabilityValidationError`, `log_event`, `RequestIdMiddleware`
