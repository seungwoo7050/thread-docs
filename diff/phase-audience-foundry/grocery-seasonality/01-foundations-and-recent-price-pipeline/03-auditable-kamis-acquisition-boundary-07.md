## `feat(source): fingerprint historical request scopes`

diff --git a/grocery/source/client.py b/grocery/source/client.py
index f7fefe8..bd6fbe4 100644
--- a/grocery/source/client.py
+++ b/grocery/source/client.py
@@ -262,6 +262,7 @@ class KamisFetchResult:
     page_receipts: tuple[PageReceipt, ...]
     ordered_manifest_sha256: str
     call_count: int
+    request_scope_sha256: str | None = None
 
 
 @dataclass(frozen=True, slots=True)
@@ -331,6 +332,7 @@ class KamisHttpClient:
                 page_size=size,
             ),
             request_shape=REDACTED_REQUEST_SHAPE,
+            request_scope_sha256=None,
         )
 
     def fetch_historical_prices(
@@ -360,6 +362,7 @@ class KamisHttpClient:
             page_size=page_size,
             request_builder=request_builder,
             request_shape=prepared.request_shape,
+            request_scope_sha256=prepared.scope_sha256,
         )
 
     def _fetch_prices(
@@ -369,6 +372,7 @@ class KamisHttpClient:
         page_size: int,
         request_builder: RequestBuilder,
         request_shape: str,
+        request_scope_sha256: str | None,
     ) -> KamisFetchResult:
         _validate_page_size(page_size)
         budget = _CallBudget()
@@ -434,6 +438,7 @@ class KamisHttpClient:
             page_receipts=frozen_receipts,
             ordered_manifest_sha256=_ordered_manifest_sha256(frozen_receipts),
             call_count=budget.count,
+            request_scope_sha256=request_scope_sha256,
         )
 
     def _fetch_page(
diff --git a/grocery/source/historical_client.py b/grocery/source/historical_client.py
index 1618ec3..3142852 100644
--- a/grocery/source/historical_client.py
+++ b/grocery/source/historical_client.py
@@ -6,7 +6,9 @@ retry, pagination, byte limits, exact envelope decoding, and raw-free receipts.
 
 from __future__ import annotations
 
-from dataclasses import dataclass
+import hashlib
+import json
+from dataclasses import dataclass, field
 from urllib.parse import urlencode
 from urllib.request import Request
 
@@ -27,8 +29,9 @@ _COMMON_REDACTED_NAMES = frozenset({"numOfRows", "pageNo", "returnType"})
 class PreparedHistoricalRequest:
     """A validated condition set plus a value-free operational request shape."""
 
-    query: ValidatedHistoricalQuery
+    query: ValidatedHistoricalQuery = field(repr=False)
     request_shape: str
+    scope_sha256: str
 
     def build(self, normalized_key: str, page_number: int, page_size: int) -> Request:
         contract = HISTORICAL_ENDPOINT_CONTRACTS[self.query.dataset]
@@ -65,7 +68,21 @@ def prepare_historical_request(
     names = sorted(_COMMON_REDACTED_NAMES | condition_names)
     names.append("serviceKey:<redacted>")
     request_shape = f"GET {contract.path} parameters=[{','.join(names)}]"
-    return PreparedHistoricalRequest(query=validated, request_shape=request_shape)
+    scope_sha256 = hashlib.sha256(
+        json.dumps(
+            {
+                "conditions": sorted(validated.conditions.items()),
+                "dataset": validated.dataset.value,
+            },
+            ensure_ascii=True,
+            separators=(",", ":"),
+        ).encode("ascii")
+    ).hexdigest()
+    return PreparedHistoricalRequest(
+        query=validated,
+        request_shape=request_shape,
+        scope_sha256=scope_sha256,
+    )
 
 
 def is_safe_historical_request_shape(value: str) -> bool:
diff --git a/grocery/source/historical_contract.py b/grocery/source/historical_contract.py
index 98251bc..3c7d450 100644
--- a/grocery/source/historical_contract.py
+++ b/grocery/source/historical_contract.py
@@ -22,7 +22,7 @@ class HistoricalDataset(StrEnum):
     MARKET = "15156065"
 
 
-@dataclass(frozen=True, slots=True)
+@dataclass(frozen=True, slots=True, repr=False)
 class HistoricalPriceQuery:
     """A bounded historical source slice; values are never retained in errors."""
 
@@ -106,7 +106,7 @@ class HistoricalContractError(ValueError):
         super().__init__(code)
 
 
-@dataclass(frozen=True, slots=True)
+@dataclass(frozen=True, slots=True, repr=False)
 class ValidatedHistoricalQuery:
     """Exact condition names and values after contract validation."""
 
@@ -116,6 +116,10 @@ class ValidatedHistoricalQuery:
     def __post_init__(self) -> None:
         object.__setattr__(self, "conditions", MappingProxyType(dict(self.conditions)))
 
+    def __repr__(self) -> str:
+        names = ",".join(sorted(self.conditions))
+        return f"ValidatedHistoricalQuery(dataset={self.dataset.value}, condition_names=[{names}])"
+
 
 def validate_historical_query(
     dataset: HistoricalDataset,
diff --git a/grocery/tests/test_historical_client.py b/grocery/tests/test_historical_client.py
index a5aba3d..83f94a3 100644
--- a/grocery/tests/test_historical_client.py
+++ b/grocery/tests/test_historical_client.py
@@ -10,6 +10,7 @@ from urllib.request import Request
 import pytest
 
 from grocery.source.client import JsonObject, KamisHttpClient, KamisTransportError
+from grocery.source.historical_client import prepare_historical_request
 from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
 
 
@@ -110,6 +111,9 @@ def test_historical_fetch_reuses_bounded_ordered_pagination() -> None:
     assert [row["synthetic"] for row in result.rows] == ["first", "second"]
     assert [receipt.requested_page_number for receipt in result.page_receipts] == [1, 2]
     assert result.call_count == 2
+    assert result.request_scope_sha256 == prepare_historical_request(
+        HistoricalDataset.REGIONAL, _regional_query()
+    ).scope_sha256
 
 
 def test_invalid_query_is_translated_before_any_request() -> None:
diff --git a/grocery/tests/test_historical_request.py b/grocery/tests/test_historical_request.py
index 3b68d83..2198e4f 100644
--- a/grocery/tests/test_historical_request.py
+++ b/grocery/tests/test_historical_request.py
@@ -99,6 +99,24 @@ def test_approved_endpoint_and_query_allowlist_are_exact(
     }
     assert "selectable" not in actual_endpoint.query
     assert is_safe_historical_request_shape(prepared.request_shape)
+    assert len(prepared.scope_sha256) == 64
     assert "synthetic" not in prepared.request_shape
     assert query.start not in prepared.request_shape
+    assert query.start not in repr(prepared)
+    assert query.region_code is None or query.region_code not in repr(prepared)
 
+
+def test_scope_hash_is_stable_and_changes_with_validated_scope() -> None:
+    base = HistoricalPriceQuery(
+        start="20260801", end="20260831", category_code="200", region_code="11000"
+    )
+    changed = HistoricalPriceQuery(
+        start="20260802", end="20260831", category_code="200", region_code="11000"
+    )
+
+    first = prepare_historical_request(HistoricalDataset.REGIONAL, base)
+    replay = prepare_historical_request(HistoricalDataset.REGIONAL, base)
+    different = prepare_historical_request(HistoricalDataset.REGIONAL, changed)
+
+    assert first.scope_sha256 == replay.scope_sha256
+    assert first.scope_sha256 != different.scope_sha256


## `feat(source): persist historical request scopes`

diff --git a/grocery/source/historical_persistence.py b/grocery/source/historical_persistence.py
new file mode 100644
index 0000000..2fae34e
--- /dev/null
+++ b/grocery/source/historical_persistence.py
@@ -0,0 +1,43 @@
+"""Persist the redacted identity of one explicitly bounded historical fetch."""
+
+from __future__ import annotations
+
+import uuid
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+
+from grocery.models import FetchAttempt, SourceConfiguration
+from grocery.source.historical_client import PreparedHistoricalRequest
+from grocery.source.historical_contract import HistoricalDataset
+
+_DATASET_MODES = {
+    HistoricalDataset.MONTHLY: SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    HistoricalDataset.REGIONAL: SourceConfiguration.PublicationMode.HISTORICAL_REGIONAL,
+    HistoricalDataset.MARKET: SourceConfiguration.PublicationMode.HISTORICAL_MARKET,
+}
+
+
+@transaction.atomic
+def start_historical_fetch(
+    source_configuration_id: uuid.UUID,
+    *,
+    prepared_request: PreparedHistoricalRequest,
+    acquisition_run_id: uuid.UUID | None = None,
+    attempt_ordinal: int = 1,
+) -> FetchAttempt:
+    source = SourceConfiguration.objects.select_for_update().get(pk=source_configuration_id)
+    if source.state != SourceConfiguration.State.ACTIVE:
+        raise ValidationError("Historical fetches require an active source configuration.")
+    expected_mode = _DATASET_MODES[prepared_request.query.dataset]
+    if source.dataset_id != prepared_request.query.dataset.value:
+        raise ValidationError("Historical source dataset and request do not match.")
+    if source.publication_mode != expected_mode:
+        raise ValidationError("Historical source mode and request do not match.")
+    return FetchAttempt.objects.create(
+        source_configuration=source,
+        acquisition_run_id=acquisition_run_id or uuid.uuid4(),
+        attempt_ordinal=attempt_ordinal,
+        redacted_request_shape=prepared_request.request_shape,
+        request_scope_sha256=prepared_request.scope_sha256,
+    )
diff --git a/grocery/source/persistence.py b/grocery/source/persistence.py
index 757a6c5..bc0d04d 100644
--- a/grocery/source/persistence.py
+++ b/grocery/source/persistence.py
@@ -132,8 +132,7 @@ def complete_kamis_fetch(
     if attempt.state != FetchAttempt.State.STARTED:
         raise ValidationError("Only a started fetch attempt can be completed.")
 
-    source = attempt.source_configuration
-    _validate_result_budget(source, result)
+    _validate_result_budget(attempt, result)
     receipts = _receipt_candidates(attempt, result)
     if ordered_page_manifest_sha256(receipts) != result.ordered_manifest_sha256:
         raise ValidationError("The client and persistence ordered manifests differ.")
@@ -220,10 +219,13 @@ def _classify_transport_failure(
     return FetchAttempt.State.TERMINAL_FAILED, failure_class
 
 
-def _validate_result_budget(
-    source: SourceConfiguration,
-    result: KamisFetchResult,
-) -> None:
+def _validate_result_budget(attempt: FetchAttempt, result: KamisFetchResult) -> None:
+    source = attempt.source_configuration
+    if attempt.request_scope_sha256:
+        if result.request_scope_sha256 != attempt.request_scope_sha256:
+            raise ValidationError("The fetch result does not match its historical request scope.")
+    elif result.request_scope_sha256 is not None:
+        raise ValidationError("A recent fetch result cannot carry a historical request scope.")
     if not result.page_receipts:
         raise ValidationError("A successful fetch requires at least one page receipt.")
     if result.call_count < len(result.page_receipts):
@@ -385,8 +387,7 @@ def _validate_completed_replay(
     attempt: FetchAttempt,
     result: KamisFetchResult,
 ) -> CompletedKamisFetch:
-    source = attempt.source_configuration
-    _validate_result_budget(source, result)
+    _validate_result_budget(attempt, result)
     persisted_receipts = list(attempt.page_receipts.order_by("request_ordinal"))
     candidate_receipts = _receipt_candidates(attempt, result)
     persisted_shape = [
diff --git a/grocery/tests/test_historical_source_persistence.py b/grocery/tests/test_historical_source_persistence.py
new file mode 100644
index 0000000..b403782
--- /dev/null
+++ b/grocery/tests/test_historical_source_persistence.py
@@ -0,0 +1,53 @@
+import hashlib
+import json
+
+from grocery.models import FetchAttempt, SourceConfiguration
+from grocery.source.client import KamisFetchResult, PageReceipt
+from grocery.source.historical_client import prepare_historical_request
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.source.historical_persistence import start_historical_fetch
+from grocery.source.persistence import complete_kamis_fetch
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+
+def test_historical_fetch_persists_only_redacted_scope_and_receipts(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    prepared = prepare_historical_request(
+        HistoricalDataset.MONTHLY,
+        HistoricalPriceQuery(start="202301", end="202512", category_code="200"),
+    )
+    attempt = start_historical_fetch(source.id, prepared_request=prepared)
+    body_sha256 = "b" * 64
+    manifest_sha256 = hashlib.sha256(
+        json.dumps([body_sha256], separators=(",", ":")).encode("ascii")
+    ).hexdigest()
+    result = KamisFetchResult(
+        rows=(),
+        page_receipts=(
+            PageReceipt(
+                ordinal=1,
+                requested_page_number=1,
+                declared_page_number=1,
+                declared_page_size=100,
+                declared_total_count=0,
+                row_count=0,
+                http_status=200,
+                provider_result_code="0",
+                byte_length=10,
+                body_sha256=body_sha256,
+            ),
+        ),
+        ordered_manifest_sha256=manifest_sha256,
+        call_count=1,
+        request_scope_sha256=prepared.scope_sha256,
+    )
+
+    completed = complete_kamis_fetch(attempt.id, result)
+
+    assert completed.attempt.state == FetchAttempt.State.SUCCEEDED
+    assert completed.attempt.request_scope_sha256 == prepared.scope_sha256
+    assert "2023" not in completed.attempt.redacted_request_shape
+    assert completed.artifact.source_identity.endswith(prepared.scope_sha256)
