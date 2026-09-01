# 외부 I/O·파싱·압축파일 안전 워크북

이 문서는 신뢰할 수 없는 외부 입력이 처음 애플리케이션 경계를 통과하는 세 지점을 다룬다. 세 문제 모두 “정상 입력을 읽는 법”보다 상한, 폐쇄형 실패, 완전성, 자원 정리가 핵심이다.

<a id="w01"></a>

---

# [sources/transport.py::_read_once, _contains_secret_reflection, _execute_aviation_attempt] W01 — 제한된 단일 HTTPS 시도와 비밀값 비노출 (S)

## 면접 질문

`_read_once`는 왜 재시도나 redirect를 직접 처리하지 않고 단 한 번의 HTTPS 요청, connect/read timeout, 응답 크기 상한, connection cleanup만 책임지도록 나뉘어 있나요? 이 경계를 그대로 두었을 때 상위 ingestion이 얻는 이점과 남는 위험을 설명해 보세요.

꼬리 질문:

1. `Content-Length`만 검사하면 왜 충분하지 않으며, 실제 body를 `limit + 1`바이트까지 읽는 이유는 무엇인가요?
2. `_execute_aviation_attempt`가 callback의 호출 횟수뿐 아니라 반환 객체 identity까지 확인하는 이유는 무엇인가요?
3. secret reflection을 원문, URL 인코딩, JSON escape, 일정 길이 fragment까지 검사하는 방식의 보안상 이점과 false positive trade-off는 무엇인가요?
4. connection의 `close()`가 실패했을 때 원래 요청 결과를 어떻게 다뤄야 하나요?

## 30초 모범 답변

이 transport는 네트워크 한 번의 사실만 결정적으로 관찰하고, 재시도·영속화는 상위 상태 기계에 맡깁니다. connect와 read timeout을 분리하고 선언 길이와 실제 읽기 모두에 상한을 두며, 어떤 경로에서도 connection을 닫습니다. 오류 상세와 secret은 밖으로 내보내지 않고 allowlist된 실패 코드로 축약합니다. executor가 실제 호출을 생략하거나 다른 성공값으로 바꾸지 못하게 호출 횟수와 객체 identity도 확인해 durable receipt가 실제 wire call과 일치하도록 합니다.

## 답변 핵심 키워드

- single-attempt boundary
- bounded read
- deterministic failure taxonomy
- `finally` cleanup
- secret non-reflection
- callback identity

## 꼬리 질문과 핵심 답변

### connect timeout과 read timeout을 분리해야 하는 이유는 무엇인가요?

TCP/TLS 연결 지연과 연결 후 upstream 응답 지연은 원인과 운영 대응이 다릅니다. 하나의 timeout만 쓰면 연결은 빨랐지만 body가 멈춘 경우를 충분히 제한하지 못하거나, 반대로 정상적인 연결 수립 시간을 지나치게 줄일 수 있습니다.

### `Content-Length`가 상한 이하인데도 oversize가 될 수 있나요?

헤더가 없거나 거짓일 수 있고, chunked transfer처럼 총 크기를 사전에 알 수 없는 경우도 있습니다. 따라서 헤더는 조기 거부 최적화이고, 실제 stream에서 한 바이트 초과를 관찰하는 검사가 최종 안전장치입니다.

### redirect를 자동으로 따라가지 않는 이유는 무엇인가요?

승인된 정확한 host/path에서 다른 origin이나 locator로 이동하면 source 계약과 credential 전송 범위가 달라집니다. redirect가 필요하다면 새 locator를 명시적으로 승인하고 별도 fingerprint를 만들어야 합니다.

### secret fragment 검사에서 8글자 창을 쓰면 어떤 문제가 생길 수 있나요?

짧은 공통 문자열 때문에 정상 응답을 차단할 가능성이 있습니다. 이 구현은 데이터 가용성보다 credential 비노출을 우선한 fail-closed 선택입니다. 임계값을 바꿀 때는 실제 secret 길이, 응답 언어, false positive 관측치를 함께 검토해야 합니다.

### 모든 `Exception`을 transport 실패로 바꾸면 디버깅이 어려워지지 않나요?

공개 결과와 영속 receipt에는 폐쇄형 코드만 남기되, 내부 진단은 민감정보를 제거한 별도 관측 경계에서 할 수 있습니다. 원 예외 문자열을 그대로 반환하거나 기록하지 않는 것이 이 경계의 보안 invariant입니다.

## 백지 구현

### 구현 목표

승인된 HTTPS 요청을 정확히 한 번 실행하고, 응답 크기와 시간을 제한하며, 어떤 종료 경로에서도 connection을 정리하는 transport 함수를 작성한다. 함수 자체는 재시도와 redirect를 하지 않는다.

### 인터페이스

`fetch_once(request: SafeRequest, *, connect_timeout_s: float, read_timeout_s: float, max_bytes: int, connection_factory: Callable) -> AttemptResult`

`SafeRequest`에는 검증이 끝난 host, request target, `GET` 또는 `POST`, 헤더, 선택적 bytes body만 들어 있다고 가정한다. `AttemptResult`는 성공 시 status·body·body hash를, 실패 시 allowlist된 code와 선택적 status만 가진다.

```python
import hashlib
import socket


FAILURES = {
    "TIMEOUT", "TRANSPORT", "RESPONSE_TOO_LARGE", "AUTHENTICATION",
    "RATE_LIMITED", "UPSTREAM_5XX", "HTTP_CLIENT",
}


def _failure(code: str, status: int | None = None) -> AttemptResult:
    assert code in FAILURES
    return AttemptResult(ok=False, failure_code=code, status=status)


def _http_failure(status: int) -> AttemptResult | None:
    if status in (401, 403):
        return _failure("AUTHENTICATION", status)
    if status == 429:
        return _failure("RATE_LIMITED", status)
    if 500 <= status <= 599:
        return _failure("UPSTREAM_5XX", status)
    if status != 200:
        return _failure("HTTP_CLIENT", status)
    return None


def _safe_close(connection) -> None:
    if connection is not None:
        try:
            connection.close()
        except Exception:
            pass


def fetch_once(request, *, connect_timeout_s, read_timeout_s,
               max_bytes, connection_factory):
    if request.method not in {"GET", "POST"}:
        return _failure("TRANSPORT")
    if (type(connect_timeout_s) not in (int, float)
            or type(read_timeout_s) not in (int, float)
            or not (1 <= connect_timeout_s <= 60 and 1 <= read_timeout_s <= 60)):
        return _failure("TRANSPORT")
    if type(max_bytes) is not int or max_bytes <= 0:
        return _failure("TRANSPORT")

    connection = None
    status = None
    try:
        connection = connection_factory(
            request.host, 443, float(connect_timeout_s)
        )
        connection.connect()
        if connection.sock is None:
            return _failure("TRANSPORT")
        connection.sock.settimeout(float(read_timeout_s))
        connection.request(
            request.method,
            request.target,
            body=request.body,
            headers=dict(request.headers),
        )
        response = connection.getresponse()
        if type(response.status) is not int or not 100 <= response.status <= 599:
            return _failure("TRANSPORT")
        status = response.status

        declared = response.getheader("Content-Length")
        if declared is not None:
            try:
                declared_size = int(declared, 10)
            except (TypeError, ValueError):
                return _failure("TRANSPORT", status)
            if declared_size < 0:
                return _failure("TRANSPORT", status)
            if declared_size > max_bytes:
                return _failure("RESPONSE_TOO_LARGE", status)

        body = response.read(max_bytes + 1)
        if type(body) is not bytes:
            return _failure("TRANSPORT", status)
        if len(body) > max_bytes:
            return _failure("RESPONSE_TOO_LARGE", status)
        if declared is not None and len(body) != declared_size:
            return _failure("TRANSPORT", status)

        classified = _http_failure(status)
        if classified is not None:
            return classified
        return AttemptResult(
            ok=True,
            status=status,
            body=body,
            byte_count=len(body),
            body_sha256=hashlib.sha256(body).hexdigest(),
        )
    except (TimeoutError, socket.timeout):
        return _failure("TIMEOUT", status)
    except Exception:
        return _failure("TRANSPORT", status)
    finally:
        _safe_close(connection)


def execute_exactly_once(executor, call_once):
    """외부 executor가 실제 결과를 한 번만 전달하는지도 검증한다."""
    calls = 0
    actual = None

    def guarded():
        nonlocal calls, actual
        calls += 1
        if calls != 1:
            return _failure("TRANSPORT")
        actual = call_once()
        return actual

    try:
        returned = executor(guarded)
    except Exception:
        return _failure("TRANSPORT")
    return returned if calls == 1 and returned is actual else _failure("TRANSPORT")
```

### 입력

- HTTPS host와 request target
- `GET` 또는 `POST`
- connect/read timeout: 각각 1~60초
- 양의 응답 byte 상한
- 테스트에서 대체할 수 있는 connection factory

### 출력

- 성공: 정확히 읽은 bytes, byte count, SHA-256, HTTP status
- 실패: `TIMEOUT`, `TRANSPORT`, `RESPONSE_TOO_LARGE`, `AUTHENTICATION`, `RATE_LIMITED`, `UPSTREAM_5XX`, `HTTP_CLIENT` 중 하나
- 실패 결과에는 body, socket 오류 문자열, request header, credential을 넣지 않는다.

### 반드시 만족해야 할 조건

- factory로 만든 connection은 성공·조기 반환·예외 모두에서 한 번 정리 시도를 한다.
- connect 뒤 socket read timeout을 별도로 설정한다.
- 선언된 `Content-Length`가 상한을 넘으면 body를 읽지 않고 거부한다.
- 실제 body는 상한 초과 여부를 판별할 수 있을 만큼만 읽는다.
- 자동 redirect와 내부 retry는 하지 않는다.
- 허용되지 않은 method는 wire request 전에 실패한다.
- status 및 body 타입이 계약과 다르면 transport 실패로 닫는다.

### 경계 조건

- body 크기가 0, 정확히 `max_bytes`, `max_bytes + 1`
- `Content-Length` 누락, 음수, 숫자가 아닌 값, 실제 길이와 불일치
- connection 생성 전 예외, connect 중 timeout, response read 중 timeout
- status가 정수가 아니거나 HTTP 범위를 벗어남
- `close()` 자체가 예외를 발생시킴

### 실패 조건

- 실패에 원 예외 메시지나 응답 body를 포함하면 안 된다.
- oversize body 전체를 메모리에 올리면 안 된다.
- close 실패가 원래 성공/실패 분류를 뒤집으면 안 된다.
- 요청을 두 번 호출하거나 외부 callback이 실제 결과를 다른 결과로 바꿀 수 있으면 안 된다.

### 필요한 제약사항

- 표준 HTTPS client 또는 제공된 fake connection만 사용한다.
- 시간 복잡도는 읽은 허용 body 크기에 선형, 추가 메모리는 최대 `O(max_bytes)`로 제한한다.
- 로그 작성과 DB 저장은 이 문제 범위에서 제외한다.

## 구현 후 자가 검증

- [ ] 정확히 상한인 body는 성공하고 한 바이트 초과 body는 실패하는가?
- [ ] `Content-Length`가 거짓이거나 없어도 실제 read 상한이 작동하는가?
- [ ] connect timeout과 read timeout이 각각 적용되는가?
- [ ] 모든 정상·실패·예외 경로에서 cleanup이 호출되는가?
- [ ] HTTP 401/403, 429, 5xx, 기타 non-200이 서로 다른 폐쇄형 코드로 분류되는가?
- [ ] 실패 객체와 예외에 body·header·secret·원 오류 문자열이 남지 않는가?
- [ ] fake executor가 `call_once`를 0번 또는 2번 호출하거나 결과를 바꾸면 실패하는가?

## 구현 후 설명할 것

- 네트워크 한 번의 책임과 retry/receipt 책임을 나눈 이유
  - 모범답변: transport는 한 wire attempt의 timeout·크기·HTTP 분류·cleanup만 보장해야 결과가 명확합니다. retry와 durable receipt는 호출 간 상태와 backoff를 다루므로 상위 orchestration으로 분리해야 중복 호출과 열린 transaction을 통제할 수 있습니다.
- 헤더 기반 조기 거부와 실제 bounded read를 함께 둔 이유
  - 모범답변: `Content-Length`는 큰 응답을 읽기 전에 거부하는 최적화일 뿐 신뢰 경계가 아닙니다. 헤더가 없거나 거짓일 수 있으므로 실제로는 `max_bytes + 1`까지만 읽어 한 바이트 초과까지 검출해야 합니다.
- cleanup 실패를 삼키는 선택과 관측성 보완 방법
  - 모범답변: `close()` 실패가 이미 결정된 성공·실패 분류를 덮으면 호출자가 잘못된 원인을 보게 됩니다. cleanup 예외는 반환 계약에서는 삼키되, 비밀값 없는 고정 metric이나 별도 운영 로그로 관측합니다.
- secret reflection 검사의 false positive를 감수한 위협 모델
  - 모범답변: upstream이 credential의 원문·URL 인코딩·JSON escape·일부 조각을 반사할 수 있다는 위협을 가정합니다. 정상 body를 일부 차단하는 비용보다 credential을 parser나 로그로 넘기는 피해가 크므로 fail-closed false positive를 택합니다.
- 성공 body를 메모리에서만 잠시 보유하는 것과 영속 보관의 차이
  - 모범답변: bounded body의 일시 보유는 현재 parse를 위한 제한된 lifetime입니다. DB·파일·로그에 저장하면 retention, 접근통제, 삭제와 재노출 의무가 생기므로 rights가 raw storage를 금지한 시스템에서는 hash와 typed 결과만 영속해야 합니다.

## 원본 확인 위치

- 파일: `sources/transport.py`
- 함수: `_read_once`, `_safe_close`, `_classify_http_failure`, `_contains_secret_reflection`, `_execute_aviation_attempt`
- Commit: `4341556 feat(sources): add bounded approved transports`
- Commit: `753690c feat(source): expose per-call aviation receipts`
- 관련 테스트: `sources/tests/test_transport.py::TransportTestCase`, 특히 timeout/oversize/redirect/secret reflection 테스트
- 연관 구현: `entry_requirements/ingestion.py::_verify_transport_result`, `travel_warnings/ingestion.py::_verify_transport_result`

<a id="w02"></a>

---

# [travel_warnings/parser.py::parse_travel_alarm_snapshot] W02 — 외부 스키마의 폐쇄형 파싱과 완전한 snapshot (S)

## 면접 질문

`parse_travel_alarm_snapshot`은 JSON decode 성공만으로 자료를 받아들이지 않고 duplicate key, schema shape, page metadata, 국가 정체성, fact 순서와 개수까지 검증합니다. 각각이 막는 실패 모드와, schema fingerprint와 typed fingerprint를 따로 두는 이유를 설명해 보세요.

꼬리 질문:

1. `totalCount == 0`인 응답을 유효한 empty snapshot으로 저장하면서도 “해당 국가가 안전하다”고 해석하지 않는 이유는 무엇인가요?
2. JSON object의 중복 key는 일반적인 parser에서 마지막 값으로 덮일 수 있는데 왜 별도로 거부해야 하나요?
3. list 내용 전체 대신 구조 shape를 fingerprint할 때 얻는 장점과 놓칠 수 있는 것은 무엇인가요?
4. 입국 CSV parser의 “동일 행 중복”과 “서로 충돌하는 중복”을 구분하는 방식은 이 문제와 어떻게 연결되나요?

## 30초 모범 답변

외부 API의 유효성은 문법, 구조, 의미, 완전성의 네 층으로 확인해야 합니다. 이 parser는 중복 key와 비표준 JSON을 먼저 거부하고, 예상 schema shape와 페이지 계약을 확인한 뒤 정확한 국가와 각 필드 범위를 검증합니다. 마지막으로 제공자 순서를 포함한 typed snapshot hash를 만듭니다. 빈 snapshot은 “완전하게 관찰된 0개”라는 증거일 뿐 안전 판정이 아니므로 그대로 표현하고 별도 추론을 하지 않습니다.

## 답변 핵심 키워드

- syntax/schema/semantic/completeness
- duplicate-key rejection
- country identity
- ordered snapshot
- typed fingerprint
- no safety inference

## 꼬리 질문과 핵심 답변

### schema fingerprint와 typed fingerprint의 역할 차이는 무엇인가요?

schema fingerprint는 key와 자료형의 구조 계약이 바뀌었는지 감지합니다. typed fingerprint는 검증 후 선택한 도메인 값과 그 순서가 같은지를 증명합니다. 구조가 같아도 값이 달라질 수 있고, 값이 같아도 provider가 예상 밖 필드를 추가할 수 있으므로 둘은 대체 관계가 아닙니다.

### 왜 `len(items) == totalCount`만이 아니라 `pageNo`와 `numOfRows`도 확인하나요?

현재 body가 전체 snapshot인지 일부 페이지인지 증명해야 하기 때문입니다. 우연히 첫 페이지 길이와 `totalCount`가 같더라도 요청·응답 계약이 다르면 이후 provider 변경을 놓칠 수 있습니다. 페이지 번호와 상한까지 고정해 “이 응답 하나가 전체”임을 확인합니다.

### 빈 배열의 shape가 비어 있어 item 구조를 확인할 수 없다는 문제는 어떻게 다루나요?

envelope shape와 item shape를 구분하고, empty snapshot은 정확한 envelope·개수 계약으로 허용합니다. item이 존재할 때는 모든 item이 동일한 정확한 key/type 계약을 만족하도록 별도 검증합니다.

### provider 순서를 fingerprint에 넣는 이유는 무엇인가요?

원본 순서가 표시·감사 의미를 가질 수 있고, DB child row의 `source_position` 연속성을 검증할 수 있습니다. 정렬해서 hash하면 순서 변조나 누락 후 재배치를 구분하지 못합니다.

### 모든 unknown field를 거부하면 호환성이 나빠지지 않나요?

맞습니다. 대신 schema drift를 조용히 수용해 잘못된 의미를 게시하는 위험을 줄입니다. 변경이 확인되면 parser contract와 schema fingerprint를 새 버전으로 명시적으로 승인하는 방식입니다.

## 백지 구현

### 구현 목표

한 국가에 대한 외부 여행 경보 JSON을 완전한 ordered snapshot으로 검증하고, raw JSON과 분리된 타입화 결과 또는 폐쇄형 실패를 반환한다.

### 인터페이스

`parse_warning_snapshot(payload: bytes, expected_country: CountryIdentity, *, max_bytes: int, max_facts: int = 100) -> ParseResult`

`ParseResult`는 성공 시 국가 정체성, provider 순서가 유지된 immutable fact 목록, observed schema hash, typed snapshot hash를 가진다. 실패 시 `SYNTAX_ERROR`, `ENCODING_ERROR`, `SCHEMA_MISMATCH`, `IDENTITY_MISMATCH`, `REQUIRED_VALUE_MISSING`, `INVALID_VALUE`, `DUPLICATE_RECORD` 중 하나와 안전한 경우의 schema hash만 가진다.

```python
import hashlib
import json
from datetime import date


EXPECTED_ENVELOPE_SHA256 = (
    "2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d"
)
ITEM_KEYS = frozenset({
    "alarm_lvl", "continent_cd", "continent_eng_nm", "continent_nm",
    "country_eng_nm", "country_iso_alp2", "country_nm",
    "dang_map_download_url", "flag_download_url", "map_download_url",
    "org_country_idx", "region_ty", "remark", "written_dt",
})
NULLABLE = frozenset({"flag_download_url", "map_download_url", "written_dt"})


class DuplicateKey(ValueError):
    pass


def _closed_object(pairs):
    value = {}
    for key, child in pairs:
        if key in value:
            raise DuplicateKey
        value[key] = child
    return value


def _reject_constant(_):
    raise ValueError


def _canonical_hash(value) -> str:
    encoded = json.dumps(
        value, ensure_ascii=False, sort_keys=True, separators=(",", ":")
    ).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()


def _envelope_shape(value, path=()):
    if path == ("response", "body", "items", "item") and type(value) is list:
        return "array"
    if type(value) is dict:
        return {k: _envelope_shape(v, (*path, k)) for k, v in sorted(value.items())}
    if value is None:
        return "null"
    if type(value) is bool:
        return "boolean"
    if type(value) is int:
        return "integer"
    if type(value) is float:
        return "number"
    if type(value) is str:
        return "string"
    if type(value) is list:
        return "array"
    return "unknown"


def _fail(code, schema=""):
    return ParseResult(ok=False, failure_code=code, observed_schema_hash=schema)


def _written_date(value):
    if value is None:
        return None
    if type(value) is not str or len(value) != 10:
        raise ValueError
    parsed = date.fromisoformat(value)
    if parsed.isoformat() != value:
        raise ValueError
    return parsed


def parse_warning_snapshot(payload, expected_country, *, max_bytes, max_facts=100):
    if type(payload) is not bytes or len(payload) > max_bytes:
        return _fail("SYNTAX_ERROR")
    try:
        text = payload.decode("utf-8", errors="strict")
    except UnicodeDecodeError:
        return _fail("ENCODING_ERROR")
    try:
        document = json.loads(
            text,
            object_pairs_hook=_closed_object,
            parse_constant=_reject_constant,
        )
    except (DuplicateKey, json.JSONDecodeError, ValueError, TypeError, RecursionError):
        return _fail("SYNTAX_ERROR")

    try:
        schema = _canonical_hash(_envelope_shape(document))
    except (RecursionError, UnicodeEncodeError):
        return _fail("SYNTAX_ERROR")
    if schema != EXPECTED_ENVELOPE_SHA256:
        return _fail("SCHEMA_MISMATCH", schema)

    response = document["response"]
    body = response["body"]
    items = body["items"]["item"]
    total = body["totalCount"]
    if response["header"]["resultCode"] != "0":
        return _fail("INVALID_VALUE", schema)
    if (body["pageNo"] != 1 or body["numOfRows"] != 100
            or type(total) is not int or not 0 <= total <= max_facts
            or len(items) != total):
        return _fail("INVALID_VALUE", schema)

    facts, seen = [], set()
    for item in items:
        if type(item) is not dict or set(item) != ITEM_KEYS:
            return _fail("SCHEMA_MISMATCH", schema)
        if any(type(item[k]) is not str for k in ITEM_KEYS - NULLABLE):
            return _fail("SCHEMA_MISMATCH", schema)
        if any(item[k] is not None and type(item[k]) is not str for k in NULLABLE):
            return _fail("SCHEMA_MISMATCH", schema)
        if (item["country_iso_alp2"] != expected_country.iso_alpha2
                or item["country_nm"] != expected_country.name_ko
                or item["country_eng_nm"] != expected_country.name_en):
            return _fail("IDENTITY_MISMATCH", schema)
        if any(not item[k].strip() for k in ("alarm_lvl", "region_ty", "remark")):
            return _fail("REQUIRED_VALUE_MISSING", schema)
        if (len(item["alarm_lvl"]) > 32 or len(item["region_ty"]) > 100
                or len(item["remark"]) > 1000):
            return _fail("INVALID_VALUE", schema)
        try:
            written = _written_date(item["written_dt"])
            typed = {
                "source_alarm_level_code": item["alarm_lvl"],
                "source_scope_type": item["region_ty"],
                "source_scope_text": item["remark"],
                "source_written_date": written.isoformat() if written else None,
            }
            fact_hash = _canonical_hash({
                **typed,
                "country_iso2": expected_country.iso_alpha2,
                "country_name_ko": expected_country.name_ko,
                "country_name_en": expected_country.name_en,
            })
        except (ValueError, UnicodeEncodeError):
            return _fail("INVALID_VALUE", schema)
        if fact_hash in seen:
            return _fail("DUPLICATE_RECORD", schema)
        seen.add(fact_hash)
        facts.append(Fact(**typed, typed_fingerprint_sha256=fact_hash))

    snapshot_hash = _canonical_hash({
        "country_iso2": expected_country.iso_alpha2,
        "country_name_ko": expected_country.name_ko,
        "country_name_en": expected_country.name_en,
        "facts": [fact.canonical_fields() for fact in facts],
    })
    return ParseResult(
        ok=True,
        country=expected_country,
        facts=tuple(facts),
        observed_schema_hash=schema,
        typed_snapshot_hash=snapshot_hash,
    )
```

### 입력

- UTF-8 JSON bytes
- 기대 국가의 ISO alpha-2, 한국어명, 영문명
- 전체 응답 byte 상한과 fact 상한

### 출력

- 성공: `0..max_facts`개의 순서 있는 typed facts와 두 fingerprint
- 실패: allowlist code, 선택적 observed schema fingerprint
- 원문, provider 오류 메시지, 임의 key/value를 실패 결과에 넣지 않는다.

### 반드시 만족해야 할 조건

- duplicate object key와 `NaN`/`Infinity` 같은 비표준 상수를 거부한다.
- envelope key와 타입은 exact match여야 한다.
- 성공 provider code, 첫 페이지, 고정 page size 계약, `totalCount == len(items)`를 검증한다.
- 각 item의 국가 정체성이 기대 국가와 정확히 일치해야 한다.
- 필수 문자열의 공백-only 값과 길이 초과를 거부한다.
- 날짜는 정확한 ISO 형식만 허용한다.
- duplicate typed fact를 거부하고 provider 순서를 보존한다.
- 0건은 성공 가능한 empty snapshot이지만 안전 의미를 파생하지 않는다.

### 경계 조건

- 0건, 1건, 정확히 100건, 101건
- item field의 `null` 허용 위치와 금지 위치
- 동일 key 중복, 동일 fact 중복, 순서만 다른 snapshot
- `totalCount`와 item 수 불일치
- 국가 코드만 맞고 이름이 다르거나 그 반대인 경우
- 유효해 보이지만 canonical form이 아닌 날짜

### 실패 조건

- 일부 page를 완전 snapshot으로 받아들이면 안 된다.
- 빈 결과를 안전 상태로 치환하면 안 된다.
- schema가 달라졌는데 알려진 field만 골라 조용히 성공하면 안 된다.
- 실패 결과나 예외에 raw item을 복사하면 안 된다.

### 필요한 제약사항

- JSON decode 단계에서 duplicate key를 관찰할 수 있는 hook을 사용한다.
- fingerprint는 명시적 canonical serialization과 SHA-256으로 계산한다.
- 전체 시간·공간 복잡도는 payload와 fact 수에 선형이어야 한다.

## 구현 후 자가 검증

- [ ] duplicate key와 비표준 JSON 상수가 문법 실패로 닫히는가?
- [ ] extra/missing key, 잘못된 type, page drift가 schema 또는 value 실패가 되는가?
- [ ] 0개 snapshot이 성공하되 어떠한 안전 판정도 생성하지 않는가?
- [ ] 100개 fact의 원래 순서와 `source_position`이 보존되는가?
- [ ] 같은 값이라도 순서가 달라지면 typed snapshot hash가 달라지는가?
- [ ] 국가 코드·이름 중 하나라도 다르면 identity mismatch인가?
- [ ] 실패 객체에 원문과 임의 provider 문자열이 없는가?

## 구현 후 설명할 것

- 문법·구조·의미·완전성 검증을 분리한 이유
  - 모범답변: 문법은 JSON 자체, 구조는 exact key/type, 의미는 국가·값·날짜, 완전성은 paging과 count 계약을 검증합니다. 층을 나누면 실패 코드가 안정적이고, 일부만 맞는 payload를 정상 snapshot으로 오인하지 않습니다.
- schema hash와 typed hash가 각각 증명하는 것
  - 모범답변: schema hash는 관찰한 envelope의 key와 타입이 승인된 wire shape인지 증명합니다. typed hash는 검증 후 선택한 도메인 값과 그 순서를 증명하므로, 같은 스키마의 값 변경이나 재정렬도 감지합니다.
- strict schema가 호환성보다 안전을 우선하는 trade-off
  - 모범답변: provider가 무해한 필드를 추가해도 수집이 중단될 수 있습니다. 대신 의미가 바뀐 응답을 오래된 parser가 조용히 부분 해석해 잘못 게시하는 위험을 막고, 변경은 새 contract 검토를 거치게 합니다.
- empty snapshot과 “안전”의 의미를 분리한 이유
  - 모범답변: 0건은 계약상 완전한 관측 결과일 수 있지만 “위험 없음”이라는 도메인 판단은 아닙니다. 결측·정책 의미를 parser가 추론하지 않고 ordered empty evidence로만 보존해야 fail-open을 피합니다.
- provider 순서를 도메인 증거로 보존한 이유
  - 모범답변: source position은 원 snapshot의 재현성과 child closure 검증에 쓰입니다. 정렬이나 set 변환을 하면 같은 fact 집합이더라도 provider가 제공한 증거 순서와 snapshot identity를 잃습니다.

## 원본 확인 위치

- 대표 파일: `travel_warnings/parser.py`
- 대표 함수: `parse_travel_alarm_snapshot`, `_object_without_duplicate_keys`, `_snapshot_envelope_shape`, `warning_snapshot_typed_fingerprint`
- Commit: `e0cc6f0 feat(warnings): accept complete official snapshots`
- 관련 테스트: `travel_warnings/tests/test_parser.py`, 특히 exact empty/complete snapshot, duplicate key/item, page/schema drift 테스트
- DB 완전성: `travel_warnings/migrations/0004_warning_snapshot_facts.py::travel_warnings_validate_snapshot_closure`
- 연관 parser: `entry_requirements/parser.py::parse_entry_csv`
- 연관 Commit: `c982f0b feat(entry): validate typed JP snapshot facts`

<a id="w03"></a>

---

# [travel_windows/duration_reference.py::_open_outer_archive] W03 — 중첩 ZIP의 안전한 제한 추출 (S)

## 면접 질문

`_open_outer_archive`는 archive를 디스크에 풀지 않고, outer ZIP의 정확한 두 member와 inner ZIP의 정확한 세 CSV만 읽습니다. `_checked_members`와 `_read_member`가 각각 어떤 공격과 손상을 막으며, 왜 메타데이터 검사와 실제 stream 읽기를 둘 다 해야 하나요?

꼬리 질문:

1. filename에 `/`, `\\`, `:`, 절대경로, `..`를 모두 금지하는 것이 일반적인 path normalization보다 강한 이유는 무엇인가요?
2. `file_size / compress_size` 비율과 전체 uncompressed 크기 상한을 동시에 확인해야 하는 이유는 무엇인가요?
3. symlink, encrypted member, 허용되지 않은 compression method를 왜 폐쇄적으로 거부하나요?
4. ZIP을 메모리에서만 처리해도 zip bomb 위험이 사라지지 않는 이유는 무엇인가요?

## 30초 모범 답변

ZIP metadata는 공격자가 조작할 수 있으므로 구조와 자원 상한을 먼저 검증하고 실제 stream에서도 byte 상한과 최종 길이를 다시 확인해야 합니다. 이 구현은 flat filename만 허용해 traversal·symlink를 제거하고, member 수·개별 크기·총 크기·압축비·암호화·compression type을 allowlist합니다. 디스크 추출을 피하면 filesystem 공격 면적은 줄지만 메모리·CPU 고갈은 남으므로 bounded read와 총량 제한이 여전히 필요합니다.

## 답변 핵심 키워드

- zip bomb
- path traversal
- metadata distrust
- per-member/aggregate limits
- bounded streaming
- allowlist compression

## 꼬리 질문과 핵심 답변

### `ZipInfo.file_size`를 확인했는데 실제 stream 길이도 확인하는 이유는 무엇인가요?

메타데이터가 손상되거나 library가 예상과 다른 stream을 반환할 가능성을 방어하기 위해서입니다. 실제 read는 `byte_limit + 1`을 넘지 않게 하고, 끝난 뒤 선언 길이와 관찰 길이를 비교해 truncation도 감지합니다.

### 압축비 상한만 두면 충분한가요?

아닙니다. 압축비가 낮아도 매우 큰 파일 하나가 메모리를 고갈시킬 수 있고, 작은 member가 많이 모여 총량을 넘길 수도 있습니다. member 개수, 개별 크기, 총 uncompressed 크기, archive 자체 크기가 모두 필요합니다.

### 왜 안전한 경로로 sanitize해서 추출하지 않고 flat name만 허용하나요?

공식 archive 계약이 flat한 정확한 파일 집합이므로 유연한 sanitize가 필요 없습니다. 이름을 고쳐 받아들이면 서로 다른 악성 이름이 같은 경로로 충돌하거나 계약 drift를 숨길 수 있습니다.

### CRC 오류는 어디에서 드러나나요?

member stream을 끝까지 읽는 과정에서 ZIP library가 CRC 불일치를 예외로 보고할 수 있습니다. 사용하지 않는 codebook도 실제로 읽는 이유 중 하나가 구조뿐 아니라 archive 무결성을 확인하기 위해서입니다.

### inner ZIP bytes 자체에도 상한이 필요한가요?

필요합니다. outer member가 nested archive라는 이유로 무제한 읽으면 inner 검증 전에 메모리를 소모합니다. outer 단계에서 nested archive compressed payload의 최대 bytes를 제한한 뒤 inner uncompressed 총량을 다시 제한해야 합니다.

## 백지 구현

### 구현 목표

메모리의 outer ZIP에서 정확한 codebook CSV 하나와 nested ZIP 하나를 검증하고, nested ZIP의 정확한 CSV 세 개를 제한된 bytes로 반환한다. 어떤 파일도 filesystem에 추출하지 않는다.

### 인터페이스

`read_reference_archive(payload: bytes, policy: ArchivePolicy) -> ArchiveResult`

`ArchivePolicy`는 outer/inner member 수, archive·member·총 uncompressed byte 상한, 최대 압축비, 허용 compression method를 가진다. `ArchiveResult`는 성공 시 정렬된 세 CSV bytes와 실패 시 폐쇄형 code만 가진다.

```python
import csv
import io
import stat
import zipfile
import zlib
from pathlib import PurePosixPath


class ArchiveProblem(Exception):
    def __init__(self, code: str):
        self.code = code


def _safe_name(name: str) -> bool:
    return bool(name) and not any(c in name for c in ("\x00", "/", "\\", ":")) \
        and name not in {".", ".."} and PurePosixPath(name).name == name


def _checked_members(archive, *, expected_count, total_limit,
                     member_limit, policy):
    members = tuple(archive.infolist())
    if len(members) != expected_count or len({m.filename for m in members}) != len(members):
        raise ArchiveProblem("SCHEMA_MISMATCH")
    total = 0
    for member in members:
        original = getattr(member, "orig_filename", member.filename)
        mode = (member.external_attr >> 16) & 0xFFFF
        if (not _safe_name(original) or member.is_dir() or stat.S_ISLNK(mode)
                or member.flag_bits & 0x41
                or member.compress_type not in policy.allowed_compression
                or member.file_size <= 0 or member.compress_size <= 0
                or member.file_size > member_limit
                or member.file_size > member.compress_size * policy.max_ratio):
            raise ArchiveProblem("INVALID_VALUE")
        total += member.file_size
        if total > total_limit:
            raise ArchiveProblem("INVALID_VALUE")
    return members


def _read_member(archive, member, limit):
    if member.file_size > limit:
        raise ArchiveProblem("INVALID_VALUE")
    chunks, total = [], 0
    with archive.open(member, "r") as stream:
        while True:
            chunk = stream.read(min(64 * 1024, limit + 1 - total))
            if not chunk:
                break
            total += len(chunk)
            if total > limit:
                raise ArchiveProblem("INVALID_VALUE")
            chunks.append(chunk)
    if total != member.file_size:
        raise ArchiveProblem("SYNTAX_ERROR")
    return b"".join(chunks)


def _validate_codebook(payload, expected_header):
    if not payload.startswith(b"\xef\xbb\xbf"):
        raise ArchiveProblem("ENCODING_ERROR")
    try:
        text = payload.decode("utf-8-sig", errors="strict")
        if "\x00" in text:
            raise ArchiveProblem("SYNTAX_ERROR")
        rows = csv.reader(io.StringIO(text, newline=""), strict=True)
        if tuple(next(rows)) != expected_header:
            raise ArchiveProblem("SCHEMA_MISMATCH")
        row_count = 0
        for row in rows:
            row_count += 1
            if len(row) != len(expected_header):
                raise ArchiveProblem("SCHEMA_MISMATCH")
        if row_count == 0:
            raise ArchiveProblem("REQUIRED_VALUE_MISSING")
    except UnicodeDecodeError:
        raise ArchiveProblem("ENCODING_ERROR")
    except (StopIteration, csv.Error):
        raise ArchiveProblem("SYNTAX_ERROR")


def read_reference_archive(payload, policy):
    if type(payload) is not bytes or len(payload) > policy.outer_archive_limit:
        return ArchiveResult.failure("INVALID_VALUE")
    if not payload.startswith(b"PK\x03\x04"):
        return ArchiveResult.failure("SYNTAX_ERROR")
    try:
        with zipfile.ZipFile(io.BytesIO(payload), "r") as outer:
            members = _checked_members(
                outer,
                expected_count=policy.outer_member_count,
                total_limit=policy.outer_total_limit,
                member_limit=max(policy.codebook_limit, policy.nested_archive_limit),
                policy=policy,
            )
            codebooks = [m for m in members if m.filename.lower().endswith(".csv")]
            nested = [m for m in members if m.filename.lower().endswith(".zip")]
            if len(codebooks) != 1 or len(nested) != 1:
                raise ArchiveProblem("SCHEMA_MISMATCH")
            _validate_codebook(
                _read_member(outer, codebooks[0], policy.codebook_limit),
                policy.codebook_header,
            )
            nested_payload = _read_member(
                outer, nested[0], policy.nested_archive_limit
            )

        if not nested_payload.startswith(b"PK\x03\x04"):
            raise ArchiveProblem("SYNTAX_ERROR")
        with zipfile.ZipFile(io.BytesIO(nested_payload), "r") as inner:
            members = _checked_members(
                inner,
                expected_count=policy.inner_member_count,
                total_limit=policy.inner_total_limit,
                member_limit=policy.inner_member_limit,
                policy=policy,
            )
            if any(not m.filename.lower().endswith(".csv") for m in members):
                raise ArchiveProblem("SCHEMA_MISMATCH")
            bodies = tuple(
                _read_member(inner, member, policy.inner_member_limit)
                for member in sorted(members, key=lambda m: m.filename)
            )
            for body in bodies:
                body.decode("cp949", errors="strict")
        return ArchiveResult.success(bodies)
    except ArchiveProblem as problem:
        return ArchiveResult.failure(problem.code)
    except UnicodeDecodeError:
        return ArchiveResult.failure("ENCODING_ERROR")
    except (zipfile.BadZipFile, zipfile.LargeZipFile, EOFError,
            NotImplementedError, OSError, RuntimeError, ValueError, zlib.error):
        return ArchiveResult.failure("SYNTAX_ERROR")
```

### 입력

- ZIP magic을 가진 bytes
- 명시적인 resource policy
- 예상 outer 구성: CSV 1개 + ZIP 1개
- 예상 inner 구성: CSV 3개

### 출력

- 성공: filename 순으로 결정적으로 정렬된 세 CSV payload
- 실패: `SYNTAX_ERROR`, `SCHEMA_MISMATCH`, `INVALID_VALUE`, `ENCODING_ERROR`, `REQUIRED_VALUE_MISSING` 중 하나

### 반드시 만족해야 할 조건

- 중복 filename과 예상 개수가 아닌 archive를 거부한다.
- member 이름은 flat basename만 허용한다.
- directory, symlink, 암호화 관련 flag, 미허용 compression을 거부한다.
- compressed size와 uncompressed size는 양수이고 개별·총량·비율 상한 이하여야 한다.
- stream은 chunk 단위로 읽고 한 바이트 초과를 관찰하면 즉시 실패한다.
- 관찰한 길이와 metadata 길이가 달라지면 실패한다.
- outer와 inner archive는 context manager로 닫는다.

### 경계 조건

- 정확한 member 수보다 하나 적거나 많음
- 같은 filename이 두 번 등장
- `../x.csv`, `/x.csv`, `a/b.csv`, `a\\b.csv`, `x:stream`
- 압축된 크기가 0이거나 압축비가 경계값을 한 단계 초과
- 개별 제한은 만족하지만 합계 제한을 초과
- inner payload가 ZIP magic이 아니거나 CSV 이외 확장자를 포함
- 선언 길이보다 실제 stream이 짧거나 김

### 실패 조건

- archive entry 이름을 사용해 디스크 경로를 만들면 안 된다.
- `read()` 무제한 호출로 전체 member를 먼저 메모리에 올리면 안 된다.
- 예상 밖 member를 무시하고 필요한 파일만 선택하면 안 된다.
- library 예외 문자열이나 filename을 외부 실패 결과로 노출하면 안 된다.

### 필요한 제약사항

- 임시 디렉터리와 shell unzip을 사용하지 않는다.
- 추가 메모리는 정책상 허용한 payload 총량을 넘지 않아야 한다.
- 결과 순서는 archive 내부 순서가 아니라 명시적 정렬 기준으로 결정한다.

## 구현 후 자가 검증

- [ ] traversal·절대경로·separator·symlink member가 모두 거부되는가?
- [ ] 개별 크기, 합계, 압축비, member 수의 각 경계를 독립적으로 테스트했는가?
- [ ] 정확히 제한값인 정상 member는 허용되는가?
- [ ] 실제 stream이 선언 길이와 다를 때 실패하는가?
- [ ] outer와 inner archive가 예외 경로에서도 닫히는가?
- [ ] unused codebook도 실제 읽기와 encoding/schema 검증을 거치는가?
- [ ] 실패 결과에 raw bytes와 archive 내부 이름이 포함되지 않는가?

## 구현 후 설명할 것

- 디스크 추출을 피하면서도 메모리·CPU 상한이 필요한 이유
  - 모범답변: 디스크 추출을 피하면 path traversal과 잔존 파일 위험은 줄지만, 압축 해제 자체가 메모리와 CPU를 고갈시킬 수 있습니다. 따라서 각 archive layer에 member 수·압축비·개별 크기·총량 상한을 별도로 둡니다.
- metadata 검증과 stream 검증을 중복해 둔 이유
  - 모범답변: central directory metadata는 읽기 전 빠른 거부에 유용하지만 공격자 입력이라 신뢰할 수 없습니다. 실제 stream도 `limit + 1`까지 읽고 관찰 길이와 선언 길이를 비교해야 거짓 크기와 truncation을 잡습니다.
- 공식 archive의 exact shape를 일반 ZIP reader보다 엄격하게 적용한 이유
  - 모범답변: 이 함수는 범용 압축 해제기가 아니라 버전이 고정된 공식 자료 parser입니다. 예상 밖 member를 무시하면 provider drift나 숨겨진 payload를 놓치므로 CSV 1개+ZIP 1개, 내부 CSV 3개를 exact contract로 검사합니다.
- 압축비·개별 크기·총량이 서로 막는 다른 공격
  - 모범답변: 압축비는 작은 compressed payload의 폭발을, 개별 크기는 하나의 거대 member를, 총량은 각각 정상처럼 보이는 다수 member의 누적 고갈을 막습니다. 하나의 상한만으로는 나머지 공격 형태를 닫지 못합니다.
- 허용 format 변경 시 policy version을 올려야 하는 이유
  - 모범답변: member 수·encoding·compression 허용값은 파싱 의미와 자원 예산의 일부입니다. 기존 policy를 조용히 넓히면 과거 seal과 테스트가 다른 입력 집합을 뜻하게 되므로 새 version으로 감사 가능한 계약 변경을 남겨야 합니다.

## 원본 확인 위치

- 파일: `travel_windows/duration_reference.py`
- 함수: `_is_safe_member_name`, `_checked_members`, `_read_member`, `_validate_codebook`, `_open_outer_archive`
- Commit: `f3075b5 feat(source): collect official route duration reference`
- 관련 테스트: `travel_windows/tests/test_duration_reference.py::DurationReferenceDerivationTests`, 특히 `test_rejects_schema_drift_and_unsafe_zip_members`
- 연관 transport: `sources/transport.py::fetch_route_duration_reference`, `_duration_wire_attempt`
