## `fix(security): require explicit transport scope`

diff --git a/.env.example b/.env.example
index edbe21f..5ea1ce6 100644
--- a/.env.example
+++ b/.env.example
@@ -4,7 +4,13 @@ ADMIN_ENABLED=0
 DJANGO_SECRET_KEY=replace-in-secret-store
 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
 DJANGO_CSRF_TRUSTED_ORIGINS=https://replace-with-approved-domain.example
+DJANGO_SECURE_SSL_REDIRECT=1
+DJANGO_SECURE_HSTS_SECONDS=31536000
+DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=0
+DJANGO_SECURE_HSTS_PRELOAD=0
+DJANGO_TRUST_X_FORWARDED_PROTO=0
 DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
+DATABASE_CONN_MAX_AGE=60
 KAMIS_API_KEY=
 KAMIS_CONFIRMATION_MAX_AGE_HOURS=36
 QA_STATE_PREVIEWS_ENABLED=0
diff --git a/config/settings.py b/config/settings.py
index 8f50c8c..eae511c 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -81,7 +81,14 @@ def validate_production_environment(
         return
     if "DJANGO_SECRET_KEY" not in environment or len(secret_key) < 50:
         raise ImproperlyConfigured("production_secret_key_required")
-    if "DJANGO_ALLOWED_HOSTS" not in environment or not allowed_hosts or "*" in allowed_hosts:
+    if (
+        "DJANGO_ALLOWED_HOSTS" not in environment
+        or not allowed_hosts
+        or any(
+            host == "*" or host.startswith(".") or any(character in host for character in "/?#@:")
+            for host in allowed_hosts
+        )
+    ):
         raise ImproperlyConfigured("production_allowed_hosts_required")
     if "DJANGO_CSRF_TRUSTED_ORIGINS" not in environment or not csrf_trusted_origins:
         raise ImproperlyConfigured("production_csrf_origins_required")
@@ -107,6 +114,18 @@ def validate_production_environment(
         raise ImproperlyConfigured("production_admin_strong_auth_not_configured")
 
 
+def validate_hsts_configuration(
+    *,
+    seconds: int,
+    include_subdomains: bool,
+    preload: bool,
+) -> None:
+    """Reject a preload opt-in that cannot meet the browser preload contract."""
+
+    if preload and (not include_subdomains or seconds < 31_536_000):
+        raise ImproperlyConfigured("production_hsts_preload_invalid")
+
+
 validate_production_environment(
     os.environ,
     debug=DEBUG,
@@ -181,8 +200,21 @@ CSRF_COOKIE_SECURE = not DEBUG
 SECURE_HSTS_SECONDS = int(
     os.environ.get("DJANGO_SECURE_HSTS_SECONDS", "0" if DEBUG else "31536000")
 )
-SECURE_HSTS_INCLUDE_SUBDOMAINS = not DEBUG
-SECURE_HSTS_PRELOAD = not DEBUG
+SECURE_HSTS_INCLUDE_SUBDOMAINS = env_bool(
+    "DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS",
+    False,
+)
+SECURE_HSTS_PRELOAD = env_bool("DJANGO_SECURE_HSTS_PRELOAD", False)
+SECURE_PROXY_SSL_HEADER = (
+    ("HTTP_X_FORWARDED_PROTO", "https")
+    if env_bool("DJANGO_TRUST_X_FORWARDED_PROTO", False)
+    else None
+)
+validate_hsts_configuration(
+    seconds=SECURE_HSTS_SECONDS,
+    include_subdomains=SECURE_HSTS_INCLUDE_SUBDOMAINS,
+    preload=SECURE_HSTS_PRELOAD,
+)
 SECURE_CONTENT_TYPE_NOSNIFF = True
 SECURE_REFERRER_POLICY = "same-origin"
 X_FRAME_OPTIONS = "DENY"
diff --git a/grocery/tests/test_production_settings.py b/grocery/tests/test_production_settings.py
index 1135745..4b95e40 100644
--- a/grocery/tests/test_production_settings.py
+++ b/grocery/tests/test_production_settings.py
@@ -3,7 +3,11 @@ from collections.abc import Mapping, Sequence
 import pytest
 from django.core.exceptions import ImproperlyConfigured
 
-from config.settings import env_positive_int, validate_production_environment
+from config.settings import (
+    env_positive_int,
+    validate_hsts_configuration,
+    validate_production_environment,
+)
 
 _SAFE_SECRET = "x" * 50
 _SAFE_HOSTS = ("prices.example",)
@@ -95,6 +99,14 @@ def test_complete_production_environment_is_accepted_with_admin_disabled() -> No
             _SAFE_ORIGINS,
             "production_allowed_hosts_required",
         ),
+        (
+            production_environment(),
+            False,
+            _SAFE_SECRET,
+            (".prices.example",),
+            _SAFE_ORIGINS,
+            "production_allowed_hosts_required",
+        ),
         (
             {
                 "DJANGO_SECRET_KEY": "present",
@@ -184,3 +196,35 @@ def test_positive_integer_environment_bound_is_explicit() -> None:
         with pytest.MonkeyPatch.context() as monkeypatch:
             monkeypatch.setenv("BOUNDED_TEST_VALUE", "0")
             env_positive_int("BOUNDED_TEST_VALUE", 36, maximum=168)
+
+
+def test_hsts_preload_requires_explicit_subdomain_scope_and_one_year() -> None:
+    validate_hsts_configuration(
+        seconds=31_536_000,
+        include_subdomains=True,
+        preload=True,
+    )
+    validate_hsts_configuration(
+        seconds=31_536_000,
+        include_subdomains=False,
+        preload=False,
+    )
+
+    with pytest.raises(
+        ImproperlyConfigured,
+        match="^production_hsts_preload_invalid$",
+    ):
+        validate_hsts_configuration(
+            seconds=31_536_000,
+            include_subdomains=False,
+            preload=True,
+        )
+    with pytest.raises(
+        ImproperlyConfigured,
+        match="^production_hsts_preload_invalid$",
+    ):
+        validate_hsts_configuration(
+            seconds=31_535_999,
+            include_subdomains=True,
+            preload=True,
+        )


## `fix(security): isolate source secret from checks`

diff --git a/Makefile b/Makefile
index aeaa091..21e04d0 100644
--- a/Makefile
+++ b/Makefile
@@ -1,12 +1,17 @@
 UV_RUN := env UV_CACHE_DIR=.cache/uv UV_TOOL_DIR=.cache/uv-tools UV_PYTHON_INSTALL_DIR=.cache/python uvx --from uv==0.12.6 uv
 PYTHON := .venv/bin/python
 
+# The source credential belongs only to the explicit ingestion process. Even if a
+# developer exported it in the parent shell, no Make recipe or assurance tool may
+# inherit it. secret-check reads the ignored owner-only file in its own process.
+unexport KAMIS_API_KEY
+
 # production-check requires explicit DJANGO_DEBUG=0, ADMIN_ENABLED=0,
 # DJANGO_SECRET_KEY, DJANGO_ALLOWED_HOSTS, DJANGO_CSRF_TRUSTED_ORIGINS,
 # DATABASE_URL, and the exact 40-character lowercase release DEPLOY_VERSION.
 # Its secret-check reads the ignored owner-only .env.local in-process; do not export
 # KAMIS_API_KEY into Make, a command argument, or a child-process environment.
-.PHONY: check db-up dependency-audit format-check license-inventory lint local-release-db-check migrate migration-check production-check production-env-check runtime-sync secret-check serve sync test type
+.PHONY: check db-up dependency-audit format-check license-inventory lint local-release-db-check migrate migration-check production-check production-env-check runtime-sync secret-check serve source-secret-env-check sync test type
 
 sync:
 	$(UV_RUN) sync --frozen
@@ -61,10 +66,14 @@ production-env-check:
 	@test "$${#DEPLOY_VERSION}" -eq 40 || { echo "production_check=failed code=release_sha_required"; exit 2; }
 	@case "$${DEPLOY_VERSION}" in *[!0-9a-f]*) echo "production_check=failed code=release_sha_required"; exit 2;; esac
 
+source-secret-env-check:
+	@test -z "$${KAMIS_API_KEY+x}" || { echo "source_secret_environment=failed code=ambient_source_secret_inherited"; exit 2; }
+	@echo "source_secret_environment=absent"
+
 local-release-db-check:
 	$(PYTHON) -m scripts.local_release_database_check
 
-production-check: production-env-check local-release-db-check check secret-check dependency-audit license-inventory
+production-check: source-secret-env-check production-env-check local-release-db-check check secret-check dependency-audit license-inventory
 	$(PYTHON) manage.py check --deploy --fail-level WARNING
 
 serve:
diff --git a/scripts/tests/test_makefile_secret_boundary.py b/scripts/tests/test_makefile_secret_boundary.py
new file mode 100644
index 0000000..849e865
--- /dev/null
+++ b/scripts/tests/test_makefile_secret_boundary.py
@@ -0,0 +1,30 @@
+"""Regression coverage for the Make-level source credential boundary."""
+
+from __future__ import annotations
+
+import os
+import subprocess
+from pathlib import Path
+
+
+def test_make_recipes_do_not_inherit_ambient_source_secret() -> None:
+    repository = Path(__file__).resolve().parents[2]
+    marker = "synthetic-source-secret-must-not-cross-make-boundary"
+    environment = os.environ.copy()
+    environment["KAMIS_API_KEY"] = marker
+
+    completed = subprocess.run(  # noqa: S603 - fixed local make target.
+        ("/usr/bin/make", "source-secret-env-check"),
+        cwd=repository,
+        env=environment,
+        stdin=subprocess.DEVNULL,
+        capture_output=True,
+        text=True,
+        timeout=10,
+        check=False,
+    )
+
+    assert completed.returncode == 0
+    assert completed.stdout.strip() == "source_secret_environment=absent"
+    assert marker not in completed.stdout
+    assert marker not in completed.stderr


## `feat(source): share bounded transport for history`

diff --git a/grocery/source/client.py b/grocery/source/client.py
index 3061400..f7fefe8 100644
--- a/grocery/source/client.py
+++ b/grocery/source/client.py
@@ -1,4 +1,4 @@
-"""Secret-safe HTTPS transport for the KAMIS recent-price endpoint.
+"""Secret-safe HTTPS transport for approved KAMIS price endpoints.
 
 The transport deliberately retains only normalized rows and redacted receipts. Raw
 response bodies and request URLs exist only inside a single call and are never put in
@@ -27,6 +27,16 @@ from urllib.request import (
     build_opener,
 )
 
+from grocery.source.historical_client import (
+    is_safe_historical_request_shape,
+    prepare_historical_request,
+)
+from grocery.source.historical_contract import (
+    HistoricalContractError,
+    HistoricalDataset,
+    HistoricalPriceQuery,
+)
+
 KAMIS_ENDPOINT = "https://apis.data.go.kr/B552845/recent/price"
 REDACTED_REQUEST_SHAPE = (
     "GET /B552845/recent/price parameters=[numOfRows,pageNo,returnType,serviceKey:<redacted>]"
@@ -65,6 +75,11 @@ _SAFE_TRANSPORT_ERROR_CODES = frozenset(
         "invalid_items_envelope",
         "invalid_json",
         "invalid_page_size",
+        "invalid_historical_dataset",
+        "invalid_historical_filter",
+        "invalid_historical_range",
+        "invalid_historical_date",
+        "missing_historical_region",
         "invalid_provider_header",
         "invalid_response_state",
         "invalid_retry_state",
@@ -98,6 +113,7 @@ type JsonValue = None | bool | int | float | str | list[JsonValue] | dict[str, J
 type JsonObject = dict[str, JsonValue]
 type OpenUrl = Callable[[Request, float], ResponseLike]
 type Sleep = Callable[[float], None]
+type RequestBuilder = Callable[[str, int, int], Request]
 
 
 class ResponseLike(Protocol):
@@ -201,6 +217,26 @@ class KamisTransportError(RuntimeError):
 
         self.partial_page_receipts = receipts
 
+    def _use_request_shape(self, request_shape: str) -> None:
+        """Replace the generic shape with a generated value-only-safe shape."""
+
+        if request_shape != REDACTED_REQUEST_SHAPE and not is_safe_historical_request_shape(
+            request_shape
+        ):
+            return
+        self.request_shape = request_shape
+        details = [f"code={self.code}"]
+        if self.page_number is not None:
+            details.append(f"page={self.page_number}")
+        if self.attempt is not None:
+            details.append(f"attempt={self.attempt}")
+        if self.http_status is not None:
+            details.append(f"http_status={self.http_status}")
+        if self.provider_result_code is not None:
+            details.append(f"provider_result_code={self.provider_result_code}")
+        details.append(f"request={request_shape}")
+        self.args = (" ".join(details),)
+
 
 @dataclass(frozen=True, slots=True)
 class PageReceipt:
@@ -286,6 +322,54 @@ class KamisHttpClient:
         """Fetch every ordered page, keeping raw bytes only within this call."""
 
         normalized_key = _normalize_service_key(service_key)
+        return self._fetch_prices(
+            normalized_key,
+            page_size=page_size,
+            request_builder=lambda key, page, size: _build_request(
+                key,
+                page_number=page,
+                page_size=size,
+            ),
+            request_shape=REDACTED_REQUEST_SHAPE,
+        )
+
+    def fetch_historical_prices(
+        self,
+        dataset: HistoricalDataset,
+        service_key: str,
+        *,
+        query: HistoricalPriceQuery,
+        page_size: int = DEFAULT_PAGE_SIZE,
+    ) -> KamisFetchResult:
+        """Fetch one bounded approved historical slice with no query-value retention."""
+
+        try:
+            prepared = prepare_historical_request(dataset, query)
+        except HistoricalContractError as error:
+            raise KamisTransportError(error.code) from None
+
+        def request_builder(key: str, page: int, size: int) -> Request:
+            try:
+                return prepared.build(key, page, size)
+            except HistoricalContractError as error:
+                raise KamisTransportError(error.code) from None
+
+        normalized_key = _normalize_service_key(service_key)
+        return self._fetch_prices(
+            normalized_key,
+            page_size=page_size,
+            request_builder=request_builder,
+            request_shape=prepared.request_shape,
+        )
+
+    def _fetch_prices(
+        self,
+        normalized_key: str,
+        *,
+        page_size: int,
+        request_builder: RequestBuilder,
+        request_shape: str,
+    ) -> KamisFetchResult:
         _validate_page_size(page_size)
         budget = _CallBudget()
         rows: list[JsonObject] = []
@@ -300,6 +384,7 @@ class KamisHttpClient:
                     page_number=page_number,
                     page_size=page_size,
                     budget=budget,
+                    request_builder=request_builder,
                 )
 
                 if expected_total is None:
@@ -340,6 +425,7 @@ class KamisHttpClient:
                     raise KamisTransportError("page_budget_exceeded", page_number=page_number)
         except KamisTransportError as error:
             error._retain_completed_pages(tuple(receipts))
+            error._use_request_shape(request_shape)
             raise
 
         frozen_receipts = tuple(receipts)
@@ -357,6 +443,7 @@ class KamisHttpClient:
         page_number: int,
         page_size: int,
         budget: _CallBudget,
+        request_builder: RequestBuilder,
     ) -> _DecodedPage:
         last_retry: _RetrySignal | None = None
 
@@ -366,6 +453,7 @@ class KamisHttpClient:
                 normalized_key,
                 page_number=page_number,
                 page_size=page_size,
+                request_builder=request_builder,
             )
             if isinstance(outcome, _DecodedPage):
                 return outcome
@@ -390,12 +478,9 @@ class KamisHttpClient:
         *,
         page_number: int,
         page_size: int,
+        request_builder: RequestBuilder,
     ) -> _DecodedPage | _RetrySignal:
-        request = _build_request(
-            normalized_key,
-            page_number=page_number,
-            page_size=page_size,
-        )
+        request = request_builder(normalized_key, page_number, page_size)
         response: ResponseLike | None = None
         safe_error: KamisTransportError | None = None
         retry: _RetrySignal | None = None
diff --git a/grocery/tests/test_historical_client.py b/grocery/tests/test_historical_client.py
new file mode 100644
index 0000000..a5aba3d
--- /dev/null
+++ b/grocery/tests/test_historical_client.py
@@ -0,0 +1,182 @@
+"""Synthetic historical transport tests; these tests never call a source API."""
+
+from __future__ import annotations
+
+import json
+from collections.abc import Mapping
+from urllib.error import URLError
+from urllib.request import Request
+
+import pytest
+
+from grocery.source.client import JsonObject, KamisHttpClient, KamisTransportError
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+
+
+class FakeResponse:
+    def __init__(self, body: bytes) -> None:
+        self.status = 200
+        self.body = body
+        self.headers: Mapping[str, str] = {
+            "Content-Type": "application/json; charset=utf-8",
+            "Content-Length": str(len(body)),
+        }
+        self.closed = False
+
+    def read(self, amount: int | None = None) -> bytes:
+        return self.body if amount is None else self.body[:amount]
+
+    def close(self) -> None:
+        self.closed = True
+
+
+class FakeOpener:
+    def __init__(self, scripted: list[FakeResponse | Exception]) -> None:
+        self.scripted = list(scripted)
+        self.requests: list[Request] = []
+
+    def __call__(self, request: Request, timeout: float) -> FakeResponse:
+        del timeout
+        self.requests.append(request)
+        if not self.scripted:
+            raise AssertionError("unexpected synthetic request")
+        result = self.scripted.pop(0)
+        if isinstance(result, Exception):
+            raise result
+        return result
+
+
+def _page_bytes(
+    *,
+    page_number: int = 1,
+    page_size: int = 100,
+    total_count: int = 1,
+    items: list[JsonObject] | None = None,
+    result_code: object = "0",
+) -> bytes:
+    payload = {
+        "response": {
+            "header": {"resultCode": result_code, "resultMsg": "NORMAL_SERVICE"},
+            "body": {
+                "dataType": "JSON",
+                "items": {"item": items if items is not None else [{"synthetic": "row"}]},
+                "numOfRows": page_size,
+                "pageNo": page_number,
+                "totalCount": total_count,
+            },
+        }
+    }
+    return json.dumps(payload, ensure_ascii=False, separators=(",", ":")).encode()
+
+
+def _regional_query() -> HistoricalPriceQuery:
+    return HistoricalPriceQuery(
+        start="20260801",
+        end="20260831",
+        category_code="200",
+        region_code="11000",
+    )
+
+
+def test_historical_fetch_reuses_bounded_ordered_pagination() -> None:
+    opener = FakeOpener(
+        [
+            FakeResponse(
+                _page_bytes(
+                    page_number=1,
+                    page_size=1,
+                    total_count=2,
+                    items=[{"synthetic": "first"}],
+                )
+            ),
+            FakeResponse(
+                _page_bytes(
+                    page_number=2,
+                    page_size=1,
+                    total_count=2,
+                    items=[{"synthetic": "second"}],
+                )
+            ),
+        ]
+    )
+
+    result = KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_historical_prices(
+        HistoricalDataset.REGIONAL,
+        "synthetic-key",
+        query=_regional_query(),
+        page_size=1,
+    )
+
+    assert [row["synthetic"] for row in result.rows] == ["first", "second"]
+    assert [receipt.requested_page_number for receipt in result.page_receipts] == [1, 2]
+    assert result.call_count == 2
+
+
+def test_invalid_query_is_translated_before_any_request() -> None:
+    opener = FakeOpener([])
+
+    with pytest.raises(KamisTransportError, match="missing_historical_region"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_historical_prices(
+            HistoricalDataset.REGIONAL,
+            "synthetic-key",
+            query=HistoricalPriceQuery(
+                start="20260801", end="20260831", category_code="200"
+            ),
+        )
+
+    assert opener.requests == []
+
+
+@pytest.mark.parametrize("result_code", [0, "00", "-3"])
+def test_success_code_must_be_the_exact_string_zero(result_code: object) -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes(result_code=result_code))])
+
+    with pytest.raises(KamisTransportError):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_historical_prices(
+            HistoricalDataset.REGIONAL,
+            "synthetic-key",
+            query=_regional_query(),
+        )
+
+
+def test_wrapperless_json_is_rejected() -> None:
+    body = json.dumps({"header": {}, "body": {}}, separators=(",", ":")).encode()
+    opener = FakeOpener([FakeResponse(body)])
+
+    with pytest.raises(KamisTransportError, match="unexpected_envelope_keys"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_historical_prices(
+            HistoricalDataset.MARKET,
+            "synthetic-key",
+            query=HistoricalPriceQuery(
+                start="20260801", end="20260831", category_code="200"
+            ),
+        )
+
+
+def test_dependency_error_cannot_leak_url_key_or_query_values() -> None:
+    class LeakyOpener:
+        def __init__(self) -> None:
+            self.requests: list[Request] = []
+
+        def __call__(self, request: Request, timeout: float) -> FakeResponse:
+            del timeout
+            self.requests.append(request)
+            raise URLError(request.full_url)
+
+    opener = LeakyOpener()
+    client = KamisHttpClient(open_url=opener, sleep=lambda _: None)
+
+    with pytest.raises(KamisTransportError, match="retry_exhausted") as raised:
+        client.fetch_historical_prices(
+            HistoricalDataset.REGIONAL,
+            "synthetic-secret-key",
+            query=_regional_query(),
+        )
+
+    visible = f"{raised.value!s} {raised.value!r}"
+    assert "synthetic-secret-key" not in visible
+    assert "20260801" not in visible
+    assert "11000" not in visible
+    assert opener.requests[0].full_url not in visible
+    assert "serviceKey:<redacted>" in visible
+    assert raised.value.__context__ is None


