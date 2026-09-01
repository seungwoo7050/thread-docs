# 히스토리 자료 계약과 파서

## `docs: record historical source gate`

diff --git a/docs/VNEXT-PRODUCT-CONTRACT.md b/docs/VNEXT-PRODUCT-CONTRACT.md
index 89931da..9802413 100644
--- a/docs/VNEXT-PRODUCT-CONTRACT.md
+++ b/docs/VNEXT-PRODUCT-CONTRACT.md
@@ -1,8 +1,8 @@
 # vNext 제품·source 계약
 
 승인일은 2026-08-31(KST)다. 이 문서는 제품 소유자가 승인한 첫 MVP 이후의
-소비자 확장 계약이며, 실제 source gate가 통과하기 전에는 source-dependent schema나
-adapter 구현을 허용하지 않는다.
+소비자 확장 계약이다. 세 신규 API의 최소 live interface gate는 같은 날 통과했으며
+비민감 증거와 아직 열려 있는 stop 조건은 `docs/VNEXT-SOURCE-GATE.md`에 고정한다.
 
 ## 제품 목표
 
diff --git a/docs/VNEXT-SOURCE-GATE.md b/docs/VNEXT-SOURCE-GATE.md
new file mode 100644
index 0000000..3f7c71f
--- /dev/null
+++ b/docs/VNEXT-SOURCE-GATE.md
@@ -0,0 +1,47 @@
+# vNext source gate
+
+이 문서는 2026-08-31(KST)에 승인된 개발계정으로 수행한 최소 live viability check의
+비민감 증거다. production 활성화, 전체 수집, coverage 승인 또는 운영 권리 판정을 대신하지
+않는다.
+
+## 실행 경계
+
+- 대상은 공공데이터포털 public gateway의 `15156060`, `15156062`, `15156065`다.
+- 각 진단 round는 dataset별 공식 예시 기간의 첫 page·최대 1행만 GET했다.
+- redirect와 retry를 허용하지 않았고, 응답 크기를 512 KiB로 제한했다.
+- 기존 프로젝트 credential은 저장소의 owner-only loader로 읽고 HTTP 요청을 만드는 즉시
+  release했다. 값, 길이, 일부, encoding과 완성 URL은 검사하거나 출력하지 않았다.
+- response body, 가격, 이름, raw row와 전체 query는 파일, log, fixture 또는 report에 남기지
+  않았다. schema key·container type과 검증 결과만 process output으로 확인했다.
+- wrapper 차이를 안전하게 진단하고 수정하기까지 dataset별 네 번 이하의 단일 호출을
+  수행했다. 자동 재시도와 추가 page 조회는 없었다.
+
+## 결과
+
+| dataset | endpoint path | HTTP | retail·non-empty | field count | field/type schema SHA-256 |
+|---|---|---:|---|---:|---|
+| `15156060` | `/B552845/perYearMonth/price` | 200 | pass | 28 | `97c0ec5188c96b982880d67724816cae04f10745a4d08be4dbbc27abb342af6a` |
+| `15156062` | `/B552845/perRegion/price` | 200 | pass | 21 | `d8bcd211de68e26e1fdc83da50a702a8316e05a586137d0056ca8dc21e83b6e1` |
+| `15156065` | `/B552845/periodRetail/price` | 200 | pass | 20 | `15b731c8fced378d3741f5ec061ef48e359d370a5fcaabd781f7345e44968406` |
+
+세 응답은 모두 JSON이며 live envelope는 공식 Swagger 설명과 달리 최상위
+`response.header`와 `response.body` wrapper를 사용했다. `body.items.item`은 배열이고,
+page metadata는 integer, item property는 이번 non-empty canary에서 모두 string이었다.
+adapter는 wrapper 없는 JSON을 허용하거나 자동 추측하지 않고 이 live shape에 실패 폐쇄한다.
+
+## 증명한 것과 증명하지 않은 것
+
+이 gate는 승인 credential이 세 public endpoint에 접근할 수 있고, non-empty 소매 응답의
+envelope와 필드 집합이 typed adapter를 만들 만큼 안정적임을 증명한다.
+
+다음은 아직 증명하지 않았다.
+
+- zero-result serialization, nullable·blank·sentinel과 가격 문자열의 모든 변형
+- 전체 기간·지역·시장 pagination과 quota 최악치
+- 네 source 사이 모든 채소·과일 exact series의 identity와 36·60개월 completeness
+- production scheduler, egress, credential rotation, review, seal과 activation
+
+parser는 값 변형과 결측을 fixture로 추정하지 않는다. 전체 code manifest 생성은 bounded
+collection을 거쳐 별도 사람이 승인하며, cross-source identity 또는 completeness가 하나라도
+맞지 않는 series는 공개 후보에서 제외한다. production first collection·review·seal·activation은
+계속 사람 checkpoint다.


## `docs: pin the live provider success code`

diff --git a/docs/VNEXT-SOURCE-GATE.md b/docs/VNEXT-SOURCE-GATE.md
index 3f7c71f..9950eae 100644
--- a/docs/VNEXT-SOURCE-GATE.md
+++ b/docs/VNEXT-SOURCE-GATE.md
@@ -27,7 +27,8 @@
 세 응답은 모두 JSON이며 live envelope는 공식 Swagger 설명과 달리 최상위
 `response.header`와 `response.body` wrapper를 사용했다. `body.items.item`은 배열이고,
 page metadata는 integer, item property는 이번 non-empty canary에서 모두 string이었다.
-adapter는 wrapper 없는 JSON을 허용하거나 자동 추측하지 않고 이 live shape에 실패 폐쇄한다.
+성공 `resultCode` literal은 string `0`이었다. adapter는 wrapper 없는 JSON이나 다른 성공
+literal을 허용하거나 자동 추측하지 않고 이 live shape에 실패 폐쇄한다.
 
 ## 증명한 것과 증명하지 않은 것
 


## `feat(source): validate historical query scope`

diff --git a/grocery/source/historical_contract.py b/grocery/source/historical_contract.py
new file mode 100644
index 0000000..98251bc
--- /dev/null
+++ b/grocery/source/historical_contract.py
@@ -0,0 +1,199 @@
+"""Fixed dataset, filter, and date-range contracts for KAMIS historical APIs."""
+
+from __future__ import annotations
+
+import re
+from collections.abc import Mapping
+from dataclasses import dataclass
+from datetime import date, datetime
+from enum import StrEnum
+from types import MappingProxyType
+
+MAX_HISTORICAL_MONTHS = 60
+MAX_HISTORICAL_DAYS = 31
+_FILTER_CODE = re.compile(r"[0-9]{1,20}\Z")
+
+
+class HistoricalDataset(StrEnum):
+    """Approved public-data API identifiers."""
+
+    MONTHLY = "15156060"
+    REGIONAL = "15156062"
+    MARKET = "15156065"
+
+
+@dataclass(frozen=True, slots=True)
+class HistoricalPriceQuery:
+    """A bounded historical source slice; values are never retained in errors."""
+
+    start: str
+    end: str
+    category_code: str
+    item_code: str | None = None
+    variety_code: str | None = None
+    grade_code: str | None = None
+    region_code: str | None = None
+    market_code: str | None = None
+
+
+@dataclass(frozen=True, slots=True)
+class HistoricalEndpointContract:
+    endpoint: str
+    date_field: str
+    allowed_filter_fields: frozenset[str]
+    required_filter_fields: frozenset[str]
+    monthly: bool
+
+    @property
+    def path(self) -> str:
+        return f"/{self.endpoint.split('/', 3)[3]}"
+
+    @property
+    def allowed_condition_names(self) -> frozenset[str]:
+        return frozenset(
+            {
+                f"cond[{self.date_field}::GTE]",
+                f"cond[{self.date_field}::LTE]",
+                *(f"cond[{field}::EQ]" for field in self.allowed_filter_fields),
+            }
+        )
+
+
+HISTORICAL_ENDPOINT_CONTRACTS: Mapping[
+    HistoricalDataset, HistoricalEndpointContract
+] = MappingProxyType(
+    {
+        HistoricalDataset.MONTHLY: HistoricalEndpointContract(
+            endpoint="https://apis.data.go.kr/B552845/perYearMonth/price",
+            date_field="exmn_ym",
+            allowed_filter_fields=frozenset(
+                {"se_cd", "ctgry_cd", "item_cd", "vrty_cd", "grd_cd", "sgg_cd"}
+            ),
+            required_filter_fields=frozenset({"se_cd", "ctgry_cd"}),
+            monthly=True,
+        ),
+        HistoricalDataset.REGIONAL: HistoricalEndpointContract(
+            endpoint="https://apis.data.go.kr/B552845/perRegion/price",
+            date_field="exmn_ymd",
+            allowed_filter_fields=frozenset(
+                {"se_cd", "ctgry_cd", "item_cd", "vrty_cd", "grd_cd", "sgg_cd"}
+            ),
+            required_filter_fields=frozenset({"se_cd", "ctgry_cd", "sgg_cd"}),
+            monthly=False,
+        ),
+        HistoricalDataset.MARKET: HistoricalEndpointContract(
+            endpoint="https://apis.data.go.kr/B552845/periodRetail/price",
+            date_field="exmn_ymd",
+            allowed_filter_fields=frozenset(
+                {"ctgry_cd", "item_cd", "vrty_cd", "grd_cd", "sgg_cd", "mrkt_cd"}
+            ),
+            required_filter_fields=frozenset({"ctgry_cd"}),
+            monthly=False,
+        ),
+    }
+)
+
+KAMIS_HISTORICAL_ENDPOINTS: Mapping[HistoricalDataset, str] = MappingProxyType(
+    {dataset: contract.endpoint for dataset, contract in HISTORICAL_ENDPOINT_CONTRACTS.items()}
+)
+
+
+class HistoricalContractError(ValueError):
+    """Value-free contract failure translated by the shared HTTP client."""
+
+    def __init__(self, code: str) -> None:
+        self.code = code
+        super().__init__(code)
+
+
+@dataclass(frozen=True, slots=True)
+class ValidatedHistoricalQuery:
+    """Exact condition names and values after contract validation."""
+
+    dataset: HistoricalDataset
+    conditions: Mapping[str, str]
+
+    def __post_init__(self) -> None:
+        object.__setattr__(self, "conditions", MappingProxyType(dict(self.conditions)))
+
+
+def validate_historical_query(
+    dataset: HistoricalDataset,
+    query: HistoricalPriceQuery,
+) -> ValidatedHistoricalQuery:
+    """Validate endpoint scope, hierarchy, codes, and inclusive source range."""
+
+    if not isinstance(dataset, HistoricalDataset):
+        raise HistoricalContractError("invalid_historical_dataset")
+    if not isinstance(query, HistoricalPriceQuery):
+        raise HistoricalContractError("invalid_historical_filter")
+    contract = HISTORICAL_ENDPOINT_CONTRACTS[dataset]
+    if query.category_code not in {"200", "400"}:
+        raise HistoricalContractError("invalid_historical_filter")
+    if query.variety_code is not None and query.item_code is None:
+        raise HistoricalContractError("invalid_historical_filter")
+    if query.grade_code is not None and query.variety_code is None:
+        raise HistoricalContractError("invalid_historical_filter")
+    if query.market_code is not None and query.region_code is None:
+        raise HistoricalContractError("invalid_historical_filter")
+    _validate_range(contract, query.start, query.end)
+
+    logical_filters: dict[str, str] = {"ctgry_cd": query.category_code}
+    if "se_cd" in contract.allowed_filter_fields:
+        logical_filters["se_cd"] = "01"
+    for field, value in (
+        ("item_cd", query.item_code),
+        ("vrty_cd", query.variety_code),
+        ("grd_cd", query.grade_code),
+        ("sgg_cd", query.region_code),
+        ("mrkt_cd", query.market_code),
+    ):
+        if value is not None:
+            if field not in contract.allowed_filter_fields or _FILTER_CODE.fullmatch(value) is None:
+                raise HistoricalContractError("invalid_historical_filter")
+            logical_filters[field] = value
+    if not contract.required_filter_fields <= logical_filters.keys():
+        if "sgg_cd" in contract.required_filter_fields:
+            raise HistoricalContractError("missing_historical_region")
+        raise HistoricalContractError("invalid_historical_filter")
+
+    conditions = {
+        f"cond[{contract.date_field}::GTE]": query.start,
+        f"cond[{contract.date_field}::LTE]": query.end,
+        **{f"cond[{field}::EQ]": value for field, value in logical_filters.items()},
+    }
+    return ValidatedHistoricalQuery(dataset=dataset, conditions=conditions)
+
+
+def _validate_range(contract: HistoricalEndpointContract, start: str, end: str) -> None:
+    if contract.monthly:
+        start_ordinal = _parse_month(start)
+        end_ordinal = _parse_month(end)
+        too_large = end_ordinal - start_ordinal + 1 > MAX_HISTORICAL_MONTHS
+    else:
+        start_day = _parse_day(start)
+        end_day = _parse_day(end)
+        start_ordinal = start_day.toordinal()
+        end_ordinal = end_day.toordinal()
+        too_large = (end_day - start_day).days + 1 > MAX_HISTORICAL_DAYS
+    if start_ordinal > end_ordinal or too_large:
+        raise HistoricalContractError("invalid_historical_range")
+
+
+def _parse_month(value: str) -> int:
+    if not isinstance(value, str) or re.fullmatch(r"[0-9]{6}", value) is None:
+        raise HistoricalContractError("invalid_historical_date")
+    try:
+        parsed = datetime.strptime(value, "%Y%m")
+    except ValueError:
+        raise HistoricalContractError("invalid_historical_date") from None
+    return parsed.year * 12 + parsed.month
+
+
+def _parse_day(value: str) -> date:
+    if not isinstance(value, str) or re.fullmatch(r"[0-9]{8}", value) is None:
+        raise HistoricalContractError("invalid_historical_date")
+    try:
+        return datetime.strptime(value, "%Y%m%d").date()
+    except ValueError:
+        raise HistoricalContractError("invalid_historical_date") from None
diff --git a/grocery/tests/test_historical_contract.py b/grocery/tests/test_historical_contract.py
new file mode 100644
index 0000000..15e7b83
--- /dev/null
+++ b/grocery/tests/test_historical_contract.py
@@ -0,0 +1,91 @@
+"""Focused tests for historical dataset, filter, and inclusive range validation."""
+
+import pytest
+
+from grocery.source.historical_contract import (
+    HISTORICAL_ENDPOINT_CONTRACTS,
+    HistoricalContractError,
+    HistoricalDataset,
+    HistoricalPriceQuery,
+    validate_historical_query,
+)
+
+
+def test_approved_dataset_paths_and_retail_scope_are_fixed() -> None:
+    assert {
+        dataset.value: contract.path
+        for dataset, contract in HISTORICAL_ENDPOINT_CONTRACTS.items()
+    } == {
+        "15156060": "/B552845/perYearMonth/price",
+        "15156062": "/B552845/perRegion/price",
+        "15156065": "/B552845/periodRetail/price",
+    }
+    monthly = validate_historical_query(
+        HistoricalDataset.MONTHLY,
+        HistoricalPriceQuery(start="202401", end="202412", category_code="200"),
+    )
+    assert monthly.conditions == {
+        "cond[exmn_ym::GTE]": "202401",
+        "cond[exmn_ym::LTE]": "202412",
+        "cond[ctgry_cd::EQ]": "200",
+        "cond[se_cd::EQ]": "01",
+    }
+
+
+@pytest.mark.parametrize(
+    ("dataset", "query", "error_code"),
+    [
+        (
+            HistoricalDataset.MONTHLY,
+            HistoricalPriceQuery(start="202613", end="202613", category_code="200"),
+            "invalid_historical_date",
+        ),
+        (
+            HistoricalDataset.MONTHLY,
+            HistoricalPriceQuery(start="202001", end="202501", category_code="200"),
+            "invalid_historical_range",
+        ),
+        (
+            HistoricalDataset.REGIONAL,
+            HistoricalPriceQuery(
+                start="20260731", end="20260831", category_code="200", region_code="11000"
+            ),
+            "invalid_historical_range",
+        ),
+        (
+            HistoricalDataset.REGIONAL,
+            HistoricalPriceQuery(start="20260801", end="20260831", category_code="200"),
+            "missing_historical_region",
+        ),
+        (
+            HistoricalDataset.MARKET,
+            HistoricalPriceQuery(start="20260801", end="20260831", category_code="999"),
+            "invalid_historical_filter",
+        ),
+        (
+            HistoricalDataset.MARKET,
+            HistoricalPriceQuery(
+                start="20260801", end="20260831", category_code="200", variety_code="00"
+            ),
+            "invalid_historical_filter",
+        ),
+        (
+            HistoricalDataset.REGIONAL,
+            HistoricalPriceQuery(
+                start="20260801",
+                end="20260831",
+                category_code="200",
+                region_code="11000",
+                market_code="110001",
+            ),
+            "invalid_historical_filter",
+        ),
+    ],
+)
+def test_invalid_ranges_and_filters_fail_closed(
+    dataset: HistoricalDataset,
+    query: HistoricalPriceQuery,
+    error_code: str,
+) -> None:
+    with pytest.raises(HistoricalContractError, match=error_code):
+        validate_historical_query(dataset, query)


## `feat(source): build redacted historical requests`

diff --git a/grocery/source/historical_client.py b/grocery/source/historical_client.py
new file mode 100644
index 0000000..1618ec3
--- /dev/null
+++ b/grocery/source/historical_client.py
@@ -0,0 +1,84 @@
+"""Redacted request construction for validated KAMIS historical queries.
+
+This module performs no I/O. The shared bounded transport in ``source.client`` owns
+retry, pagination, byte limits, exact envelope decoding, and raw-free receipts.
+"""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+from urllib.parse import urlencode
+from urllib.request import Request
+
+from grocery.source.historical_contract import (
+    HISTORICAL_ENDPOINT_CONTRACTS,
+    HistoricalContractError,
+    HistoricalDataset,
+    HistoricalPriceQuery,
+    ValidatedHistoricalQuery,
+    validate_historical_query,
+)
+
+_COMMON_PARAMETER_NAMES = frozenset({"serviceKey", "returnType", "pageNo", "numOfRows"})
+_COMMON_REDACTED_NAMES = frozenset({"numOfRows", "pageNo", "returnType"})
+
+
+@dataclass(frozen=True, slots=True)
+class PreparedHistoricalRequest:
+    """A validated condition set plus a value-free operational request shape."""
+
+    query: ValidatedHistoricalQuery
+    request_shape: str
+
+    def build(self, normalized_key: str, page_number: int, page_size: int) -> Request:
+        contract = HISTORICAL_ENDPOINT_CONTRACTS[self.query.dataset]
+        if not self.query.conditions.keys() <= contract.allowed_condition_names:
+            raise HistoricalContractError("request_parameter_allowlist_violation")
+        parameters = {
+            "serviceKey": normalized_key,
+            "returnType": "JSON",
+            "pageNo": str(page_number),
+            "numOfRows": str(page_size),
+            **self.query.conditions,
+        }
+        if frozenset(parameters) != _COMMON_PARAMETER_NAMES | frozenset(self.query.conditions):
+            raise HistoricalContractError("request_parameter_allowlist_violation")
+        query_string = urlencode(parameters, doseq=False, safe="")
+        return Request(  # noqa: S310 - the endpoint is a fixed HTTPS constant.
+            f"{contract.endpoint}?{query_string}",
+            headers={"Accept": "application/json"},
+            method="GET",
+        )
+
+
+def prepare_historical_request(
+    dataset: HistoricalDataset,
+    query: HistoricalPriceQuery,
+) -> PreparedHistoricalRequest:
+    """Validate and prepare one exact, redacted historical request contract."""
+
+    validated = validate_historical_query(dataset, query)
+    contract = HISTORICAL_ENDPOINT_CONTRACTS[validated.dataset]
+    condition_names = frozenset(validated.conditions)
+    if not condition_names <= contract.allowed_condition_names:
+        raise HistoricalContractError("request_parameter_allowlist_violation")
+    names = sorted(_COMMON_REDACTED_NAMES | condition_names)
+    names.append("serviceKey:<redacted>")
+    request_shape = f"GET {contract.path} parameters=[{','.join(names)}]"
+    return PreparedHistoricalRequest(query=validated, request_shape=request_shape)
+
+
+def is_safe_historical_request_shape(value: str) -> bool:
+    """Return whether a shape contains names from one fixed contract and no values."""
+
+    for contract in HISTORICAL_ENDPOINT_CONTRACTS.values():
+        prefix = f"GET {contract.path} parameters=["
+        if not value.startswith(prefix) or not value.endswith("]"):
+            continue
+        names = value[len(prefix) : -1].split(",")
+        if not names or names[-1] != "serviceKey:<redacted>":
+            return False
+        actual_names = frozenset(names[:-1])
+        allowed_names = contract.allowed_condition_names | _COMMON_REDACTED_NAMES
+        return actual_names <= allowed_names and _COMMON_REDACTED_NAMES <= actual_names
+    return False
diff --git a/grocery/tests/test_historical_request.py b/grocery/tests/test_historical_request.py
new file mode 100644
index 0000000..3b68d83
--- /dev/null
+++ b/grocery/tests/test_historical_request.py
@@ -0,0 +1,104 @@
+"""Unit tests for fixed historical endpoint and query contracts."""
+
+from urllib.parse import parse_qs, urlsplit
+
+import pytest
+
+from grocery.source.historical_client import (
+    is_safe_historical_request_shape,
+    prepare_historical_request,
+)
+from grocery.source.historical_contract import (
+    KAMIS_HISTORICAL_ENDPOINTS,
+    HistoricalDataset,
+    HistoricalPriceQuery,
+)
+
+
+@pytest.mark.parametrize(
+    ("dataset", "query", "expected_parameters"),
+    [
+        (
+            HistoricalDataset.MONTHLY,
+            HistoricalPriceQuery(
+                start="202401",
+                end="202412",
+                category_code="200",
+                item_code="212",
+                variety_code="00",
+                grade_code="04",
+                region_code="11000",
+            ),
+            {
+                "cond[exmn_ym::GTE]": ["202401"],
+                "cond[exmn_ym::LTE]": ["202412"],
+                "cond[se_cd::EQ]": ["01"],
+                "cond[ctgry_cd::EQ]": ["200"],
+                "cond[item_cd::EQ]": ["212"],
+                "cond[vrty_cd::EQ]": ["00"],
+                "cond[grd_cd::EQ]": ["04"],
+                "cond[sgg_cd::EQ]": ["11000"],
+            },
+        ),
+        (
+            HistoricalDataset.REGIONAL,
+            HistoricalPriceQuery(
+                start="20260801",
+                end="20260831",
+                category_code="400",
+                region_code="11000",
+            ),
+            {
+                "cond[exmn_ymd::GTE]": ["20260801"],
+                "cond[exmn_ymd::LTE]": ["20260831"],
+                "cond[se_cd::EQ]": ["01"],
+                "cond[ctgry_cd::EQ]": ["400"],
+                "cond[sgg_cd::EQ]": ["11000"],
+            },
+        ),
+        (
+            HistoricalDataset.MARKET,
+            HistoricalPriceQuery(
+                start="20260815",
+                end="20260831",
+                category_code="200",
+                region_code="11000",
+                market_code="110001",
+            ),
+            {
+                "cond[exmn_ymd::GTE]": ["20260815"],
+                "cond[exmn_ymd::LTE]": ["20260831"],
+                "cond[ctgry_cd::EQ]": ["200"],
+                "cond[sgg_cd::EQ]": ["11000"],
+                "cond[mrkt_cd::EQ]": ["110001"],
+            },
+        ),
+    ],
+)
+def test_approved_endpoint_and_query_allowlist_are_exact(
+    dataset: HistoricalDataset,
+    query: HistoricalPriceQuery,
+    expected_parameters: dict[str, list[str]],
+) -> None:
+    prepared = prepare_historical_request(dataset, query)
+    request = prepared.build("synthetic+key/segment=", 1, 100)
+
+    actual_endpoint = urlsplit(request.full_url)
+    approved_endpoint = urlsplit(KAMIS_HISTORICAL_ENDPOINTS[dataset])
+    assert (actual_endpoint.scheme, actual_endpoint.netloc, actual_endpoint.path) == (
+        "https",
+        approved_endpoint.netloc,
+        approved_endpoint.path,
+    )
+    assert parse_qs(actual_endpoint.query, strict_parsing=True) == {
+        "serviceKey": ["synthetic+key/segment="],
+        "returnType": ["JSON"],
+        "pageNo": ["1"],
+        "numOfRows": ["100"],
+        **expected_parameters,
+    }
+    assert "selectable" not in actual_endpoint.query
+    assert is_safe_historical_request_shape(prepared.request_shape)
+    assert "synthetic" not in prepared.request_shape
+    assert query.start not in prepared.request_shape
+


