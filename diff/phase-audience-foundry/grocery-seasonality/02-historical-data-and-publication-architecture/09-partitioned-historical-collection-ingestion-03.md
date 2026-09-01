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


## `feat(history): load reviewed parse dimensions`

diff --git a/grocery/historical_registry.py b/grocery/historical_registry.py
new file mode 100644
index 0000000..cb8843e
--- /dev/null
+++ b/grocery/historical_registry.py
@@ -0,0 +1,101 @@
+"""Load the pre-reviewed identity dimensions used by historical source parsers."""
+
+from __future__ import annotations
+
+from django.core.exceptions import ValidationError
+
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailMarketKey,
+    RetailRegionKey,
+)
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry
+from grocery.source.kamis import (
+    IdentityObservation,
+    KamisParseError,
+    build_identity_registry_from_reviewed_evidence,
+)
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+
+
+def load_historical_dimension_registry(
+    code_manifest_sha256: str,
+) -> HistoricalDimensionRegistry:
+    series_keys = list(
+        HistoricalRetailSeriesKey.objects.select_related("recent_series").order_by(
+            "series_identity_sha256"
+        )
+    )
+    if not series_keys or any(
+        key.code_manifest_sha256 != code_manifest_sha256 for key in series_keys
+    ):
+        raise ValidationError("Historical series identities do not match the code manifest.")
+    item_names: dict[tuple[str, str], str] = {}
+    variety_names: dict[tuple[str, str, str], str] = {}
+    grade_names: dict[tuple[str, str, str, str], str] = {}
+    units: dict[tuple[str, str, str, str], tuple[str, str]] = {}
+    try:
+        for key in series_keys:
+            series = key.recent_series
+            INITIAL_RETAIL_IDENTITY_REGISTRY.validate(
+                IdentityObservation(
+                    product_class_code=series.product_class_code,
+                    product_class_name=series.product_class_name,
+                    category_code=series.category_code,
+                    category_name=series.category_name,
+                    item_code=series.item_code,
+                    item_name=series.item_name,
+                    variety_code=series.variety_code,
+                    variety_name=series.variety_name,
+                    grade_code=series.grade_code,
+                    grade_name=series.grade_name,
+                    raw_unit=series.raw_unit,
+                    raw_unit_size=series.raw_unit_size,
+                    coverage_identity=series.coverage_identity,
+                ),
+                row_index=0,
+            )
+            item_names[(series.category_code, series.item_code)] = series.item_name
+            variety_names[
+                (series.category_code, series.item_code, series.variety_code)
+            ] = series.variety_name
+            series_code_key = (
+                series.category_code,
+                series.item_code,
+                series.variety_code,
+                series.grade_code,
+            )
+            grade_names[series_code_key] = series.grade_name
+            units[series_code_key] = (series.raw_unit, series.raw_unit_size)
+    except KamisParseError:
+        raise ValidationError(
+            "Historical series identity drifted from reviewed evidence."
+        ) from None
+
+    regions = list(RetailRegionKey.objects.order_by("region_code"))
+    markets = list(RetailMarketKey.objects.select_related("region").order_by("market_code"))
+    if not regions:
+        raise ValidationError("Historical ingestion requires reviewed region identities.")
+    evidence_revisions = {
+        *(region.identity_evidence_revision for region in regions),
+        *(market.identity_evidence_revision for market in markets),
+    }
+    if len(evidence_revisions) != 1:
+        raise ValidationError("Historical region and market evidence revisions must match.")
+    historical_identity_registry = build_identity_registry_from_reviewed_evidence(
+        item_names=item_names,
+        variety_names=variety_names,
+        grade_names=grade_names,
+        units=units,
+        evidence=INITIAL_RETAIL_IDENTITY_REGISTRY.evidence,
+        coverage_identity=INITIAL_RETAIL_IDENTITY_REGISTRY.coverage_identity,
+    )
+    return HistoricalDimensionRegistry(
+        identity_registry=historical_identity_registry,
+        region_names={region.region_code: region.region_name for region in regions},
+        market_names={
+            (market.region.region_code, market.market_code): market.market_name
+            for market in markets
+        },
+        dimension_evidence_revision=evidence_revisions.pop(),
+    )
diff --git a/grocery/tests/historical_bundle_factory.py b/grocery/tests/historical_bundle_factory.py
index 1d556d7..b683d75 100644
--- a/grocery/tests/historical_bundle_factory.py
+++ b/grocery/tests/historical_bundle_factory.py
@@ -123,7 +123,7 @@ def _approve(
 
 
 def create_reviewed_historical_bundle() -> ReviewedHistoricalBundle:
-    recent = create_series()
+    recent = create_series(item_name="양배추", variety_name="양배추")
     series = HistoricalRetailSeriesKey.objects.create(
         recent_series=recent,
         series_identity_sha256=price_series_identity_sha256(recent),
diff --git a/grocery/tests/test_historical_registry.py b/grocery/tests/test_historical_registry.py
new file mode 100644
index 0000000..c4405ef
--- /dev/null
+++ b/grocery/tests/test_historical_registry.py
@@ -0,0 +1,21 @@
+from grocery.historical_registry import load_historical_dimension_registry
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+
+
+def test_registry_loads_only_pre_reviewed_series_region_and_market(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+
+    registry = load_historical_dimension_registry("a" * 64)
+
+    assert registry.region_names == {bundle.region.region_code: bundle.region.region_name}
+    assert registry.market_names == {
+        (bundle.region.region_code, bundle.market.market_code): bundle.market.market_name
+    }
+    assert set(registry.identity_registry.units) == {
+        (
+            bundle.series.recent_series.category_code,
+            bundle.series.recent_series.item_code,
+            bundle.series.recent_series.variety_code,
+            bundle.series.recent_series.grade_code,
+        )
+    }


## `feat(history): plan ordered source partitions`

diff --git a/grocery/historical_collection_plans.py b/grocery/historical_collection_plans.py
new file mode 100644
index 0000000..ee5f72f
--- /dev/null
+++ b/grocery/historical_collection_plans.py
@@ -0,0 +1,87 @@
+"""Idempotent plans for ordered multi-part historical source collections."""
+
+from __future__ import annotations
+
+import uuid
+from datetime import datetime
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_collections import partition_manifest_sha256
+from grocery.models import SourceConfiguration
+from grocery.source.historical_client import PreparedHistoricalRequest
+from grocery.source.historical_contract import HistoricalDataset
+
+
+@transaction.atomic
+def plan_historical_collection(
+    *,
+    collection_id: uuid.UUID,
+    source_configuration_id: uuid.UUID,
+    prepared_requests: tuple[PreparedHistoricalRequest, ...],
+    code_manifest_sha256: str,
+) -> HistoricalSourceCollection:
+    if not prepared_requests or len(prepared_requests) > 100:
+        raise ValidationError("Historical collection plans require 1 to 100 partitions.")
+    datasets = {prepared.query.dataset for prepared in prepared_requests}
+    scopes = [prepared.scope_sha256 for prepared in prepared_requests]
+    if len(datasets) != 1 or len(set(scopes)) != len(scopes):
+        raise ValidationError("Historical collection partitions must be unique for one dataset.")
+    dataset = datasets.pop()
+    source = SourceConfiguration.objects.select_for_update().get(pk=source_configuration_id)
+    expected_mode = {
+        HistoricalDataset.MONTHLY: SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+        HistoricalDataset.REGIONAL: SourceConfiguration.PublicationMode.HISTORICAL_REGIONAL,
+        HistoricalDataset.MARKET: SourceConfiguration.PublicationMode.HISTORICAL_MARKET,
+    }[dataset]
+    if (
+        source.state != SourceConfiguration.State.ACTIVE
+        or source.dataset_id != dataset.value
+        or source.publication_mode != expected_mode
+    ):
+        raise ValidationError("Historical collection plan does not match its active source.")
+    date_field = "exmn_ym" if dataset == HistoricalDataset.MONTHLY else "exmn_ymd"
+    windows = {
+        (
+            prepared.query.conditions[f"cond[{date_field}::GTE]"],
+            prepared.query.conditions[f"cond[{date_field}::LTE]"],
+        )
+        for prepared in prepared_requests
+    }
+    if len(windows) != 1:
+        raise ValidationError("Historical collection partitions require one common window.")
+    window_min, window_max = windows.pop()
+    kind = {
+        HistoricalDataset.MONTHLY: HistoricalSourceCollection.Kind.MONTHLY,
+        HistoricalDataset.REGIONAL: HistoricalSourceCollection.Kind.REGIONAL_DAILY,
+        HistoricalDataset.MARKET: HistoricalSourceCollection.Kind.MARKET_DAILY,
+    }[dataset]
+    fields: dict[str, object] = {
+        "kind": kind,
+        "source_configuration_id": source.id,
+        "code_manifest_sha256": code_manifest_sha256,
+        "partition_manifest_sha256": partition_manifest_sha256(scopes),
+        "expected_part_count": len(scopes),
+        "month_min": window_min if dataset == HistoricalDataset.MONTHLY else "",
+        "month_max": window_max if dataset == HistoricalDataset.MONTHLY else "",
+        "date_min": (
+            None
+            if dataset == HistoricalDataset.MONTHLY
+            else datetime.strptime(window_min, "%Y%m%d").date()
+        ),
+        "date_max": (
+            None
+            if dataset == HistoricalDataset.MONTHLY
+            else datetime.strptime(window_max, "%Y%m%d").date()
+        ),
+    }
+    existing = (
+        HistoricalSourceCollection.objects.select_for_update().filter(pk=collection_id).first()
+    )
+    if existing is not None:
+        if any(getattr(existing, name) != value for name, value in fields.items()):
+            raise ValidationError("Historical collection plan UUID conflicts with stored scope.")
+        return existing
+    return HistoricalSourceCollection.objects.create(id=collection_id, **fields)
diff --git a/grocery/tests/test_historical_collection_plans.py b/grocery/tests/test_historical_collection_plans.py
new file mode 100644
index 0000000..69acf7a
--- /dev/null
+++ b/grocery/tests/test_historical_collection_plans.py
@@ -0,0 +1,31 @@
+import uuid
+
+from grocery.historical_collection_plans import plan_historical_collection
+from grocery.models import SourceConfiguration
+from grocery.source.historical_client import prepare_historical_request
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+
+def test_collection_plan_preserves_ordered_multi_partition_manifest(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    prepared = tuple(
+        prepare_historical_request(
+            HistoricalDataset.MONTHLY,
+            HistoricalPriceQuery(start="202301", end="202512", category_code=category),
+        )
+        for category in ("200", "400")
+    )
+
+    collection = plan_historical_collection(
+        collection_id=uuid.uuid4(),
+        source_configuration_id=source.id,
+        prepared_requests=prepared,
+        code_manifest_sha256="a" * 64,
+    )
+
+    assert collection.expected_part_count == 2
+    assert collection.parts.count() == 0


## `feat(history): resolve reviewed generation identities`

diff --git a/grocery/historical_generation_common.py b/grocery/historical_generation_common.py
new file mode 100644
index 0000000..f004e4c
--- /dev/null
+++ b/grocery/historical_generation_common.py
@@ -0,0 +1,68 @@
+"""Shared exact-identity and configuration checks for historical persistence."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+
+from django.core.exceptions import ValidationError
+
+from grocery.historical_identity_models import HistoricalRetailSeriesKey, RetailRegionKey
+from grocery.models import PriceSeriesKey
+from grocery.source.historical_contract import HistoricalDataset
+from grocery.source.historical_dimensions import RegionObservation
+from grocery.source.kamis import IdentityObservation
+
+
+def historical_configuration_sha256(
+    *,
+    dataset: HistoricalDataset,
+    parser_revision: str,
+    code_manifest_sha256: str,
+) -> str:
+    canonical = json.dumps(
+        {
+            "code_manifest_sha256": code_manifest_sha256,
+            "dataset": dataset.value,
+            "parser_revision": parser_revision,
+        },
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("ascii")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def resolve_historical_series(
+    identity: IdentityObservation,
+    *,
+    code_manifest_sha256: str,
+) -> HistoricalRetailSeriesKey:
+    recent = PriceSeriesKey.objects.get(
+        product_class_code=identity.product_class_code,
+        category_code=identity.category_code,
+        item_code=identity.item_code,
+        variety_code=identity.variety_code,
+        grade_code=identity.grade_code,
+        raw_unit=identity.raw_unit,
+        raw_unit_size=identity.raw_unit_size,
+        coverage_identity=identity.coverage_identity,
+    )
+    if (
+        recent.product_class_name != identity.product_class_name
+        or recent.category_name != identity.category_name
+        or recent.item_name != identity.item_name
+        or recent.variety_name != identity.variety_name
+        or recent.grade_name != identity.grade_name
+    ):
+        raise ValidationError("Historical row display identity drifted from its reviewed series.")
+    return HistoricalRetailSeriesKey.objects.get(
+        recent_series=recent,
+        code_manifest_sha256=code_manifest_sha256,
+    )
+
+
+def resolve_historical_region(observation: RegionObservation) -> RetailRegionKey:
+    region = RetailRegionKey.objects.get(region_code=observation.code)
+    if region.region_name != observation.name:
+        raise ValidationError("Historical row region name drifted from reviewed evidence.")
+    return region
diff --git a/grocery/tests/test_historical_generation_common.py b/grocery/tests/test_historical_generation_common.py
new file mode 100644
index 0000000..ee72f72
--- /dev/null
+++ b/grocery/tests/test_historical_generation_common.py
@@ -0,0 +1,41 @@
+from grocery.historical_generation_common import (
+    historical_configuration_sha256,
+    resolve_historical_region,
+    resolve_historical_series,
+)
+from grocery.source.historical_contract import HistoricalDataset
+from grocery.source.historical_dimensions import RegionObservation
+from grocery.source.kamis import IdentityObservation
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+
+
+def test_generation_resolves_only_exact_reviewed_identity_and_manifest(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    recent = bundle.series.recent_series
+    identity = IdentityObservation(
+        product_class_code=recent.product_class_code,
+        product_class_name=recent.product_class_name,
+        category_code=recent.category_code,
+        category_name=recent.category_name,
+        item_code=recent.item_code,
+        item_name=recent.item_name,
+        variety_code=recent.variety_code,
+        variety_name=recent.variety_name,
+        grade_code=recent.grade_code,
+        grade_name=recent.grade_name,
+        raw_unit=recent.raw_unit,
+        raw_unit_size=recent.raw_unit_size,
+        coverage_identity=recent.coverage_identity,
+    )
+
+    assert resolve_historical_series(identity, code_manifest_sha256="a" * 64) == bundle.series
+    assert resolve_historical_region(
+        RegionObservation(bundle.region.region_code, bundle.region.region_name)
+    ) == bundle.region
+    assert len(
+        historical_configuration_sha256(
+            dataset=HistoricalDataset.MONTHLY,
+            parser_revision="kamis-15156060-v1",
+            code_manifest_sha256="a" * 64,
+        )
+    ) == 64


