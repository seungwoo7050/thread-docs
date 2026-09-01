# 컬렉션 무결성과 동시성

## `feat(history): reconcile collection parts`

diff --git a/grocery/historical_collections.py b/grocery/historical_collections.py
new file mode 100644
index 0000000..4392b83
--- /dev/null
+++ b/grocery/historical_collections.py
@@ -0,0 +1,105 @@
+"""Atomic reconciliation for complete historical source collections."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import uuid
+from collections.abc import Sequence
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+from django.utils import timezone
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_daily_models import DailyMarketRetailPrice, DailyRegionalRetailPrice
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.models import FetchAttempt, ParseRun
+
+
+def partition_manifest_sha256(scope_hashes: Sequence[str]) -> str:
+    canonical = json.dumps(list(scope_hashes), separators=(",", ":")).encode("ascii")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+@transaction.atomic
+def complete_historical_collection(collection_id: uuid.UUID) -> HistoricalSourceCollection:
+    collection = HistoricalSourceCollection.objects.select_for_update().get(pk=collection_id)
+    if collection.state == HistoricalSourceCollection.State.VALIDATED:
+        return collection
+    if collection.state != HistoricalSourceCollection.State.STARTED:
+        raise ValidationError("Only a started historical collection can be completed.")
+
+    parts = list(
+        HistoricalSourceCollectionPart.objects.select_for_update()
+        .select_related("parse_run__artifact")
+        .filter(collection=collection)
+        .order_by("ordinal")
+    )
+    expected_ordinals = list(range(1, collection.expected_part_count + 1))
+    if [part.ordinal for part in parts] != expected_ordinals:
+        raise ValidationError("Historical collection parts are incomplete or non-contiguous.")
+    scopes = [part.partition_scope_sha256 for part in parts]
+    if partition_manifest_sha256(scopes) != collection.partition_manifest_sha256:
+        raise ValidationError("Historical collection partition manifest does not match its plan.")
+
+    fact_models = {
+        HistoricalSourceCollection.Kind.MONTHLY: MonthlyRegionalRetailPrice,
+        HistoricalSourceCollection.Kind.REGIONAL_DAILY: DailyRegionalRetailPrice,
+        HistoricalSourceCollection.Kind.MARKET_DAILY: DailyMarketRetailPrice,
+    }
+    selected_model = fact_models[collection.kind]
+    other_models = tuple(model for model in fact_models.values() if model is not selected_model)
+    accepted = 0
+    out_of_scope = 0
+    result_parts: list[dict[str, object]] = []
+    for part in parts:
+        parse_run = part.parse_run
+        if parse_run.status != ParseRun.Status.VALIDATED or not parse_run.result_hash:
+            raise ValidationError("Historical collection parts require validated parse runs.")
+        if not FetchAttempt.objects.filter(
+            source_configuration=collection.source_configuration,
+            artifact=parse_run.artifact,
+            state=FetchAttempt.State.SUCCEEDED,
+        ).exists():
+            raise ValidationError(
+                "Historical collection parse source does not match the collection."
+            )
+        actual_count = selected_model.objects.filter(
+            collection=collection,
+            collection_part=part,
+        ).count()
+        if actual_count != part.fact_count or actual_count != parse_run.accepted_row_count:
+            raise ValidationError("Historical collection fact count does not match its parse part.")
+        if any(
+            model.objects.filter(collection=collection, collection_part=part).exists()
+            for model in other_models
+        ):
+            raise ValidationError("Historical collection contains facts from another source kind.")
+        accepted += actual_count
+        out_of_scope += parse_run.out_of_scope_row_count
+        result_parts.append(
+            {
+                "ordinal": part.ordinal,
+                "partition_scope_sha256": part.partition_scope_sha256,
+                "parse_result_sha256": parse_run.result_hash,
+                "fact_count": actual_count,
+            }
+        )
+
+    result_bytes = json.dumps(
+        {"kind": collection.kind, "parts": result_parts},
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("ascii")
+    collection.state = HistoricalSourceCollection.State.VALIDATED
+    collection.accepted_row_count = accepted
+    collection.out_of_scope_row_count = out_of_scope
+    collection.quarantined_row_count = 0
+    collection.result_sha256 = hashlib.sha256(result_bytes).hexdigest()
+    collection.completed_at = timezone.now()
+    collection.save()
+    return collection
diff --git a/grocery/tests/test_historical_collection_reconciliation.py b/grocery/tests/test_historical_collection_reconciliation.py
new file mode 100644
index 0000000..3a0dcfd
--- /dev/null
+++ b/grocery/tests/test_historical_collection_reconciliation.py
@@ -0,0 +1,102 @@
+from django.utils import timezone
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_collections import (
+    complete_historical_collection,
+    partition_manifest_sha256,
+)
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailRegionKey,
+    price_series_identity_sha256,
+)
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.models import FetchAttempt, ParseRun, SourceConfiguration, build_source_artifact
+from grocery.tests.test_acquisition_models import (
+    create_fetch_attempt,
+    create_page_receipt,
+    create_source_configuration,
+)
+from grocery.tests.test_price_series_key_models import create_series
+
+
+def test_completion_reconciles_planned_partition_parse_and_typed_fact(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    scope = "e" * 64
+    attempt = create_fetch_attempt(source, request_scope_sha256=scope)
+    create_page_receipt(
+        attempt,
+        declared_total_count=1,
+        received_row_count=1,
+        body_byte_length=10,
+        body_sha256="b" * 64,
+    )
+    completed_at = timezone.now()
+    attempt.state = FetchAttempt.State.SUCCEEDED
+    attempt.completed_at = completed_at
+    attempt.received_page_count = 1
+    attempt.received_row_count = 1
+    attempt.received_byte_count = 10
+    attempt.save()
+    artifact, _created = build_source_artifact(attempt.id)
+    parse_run = ParseRun.objects.create(
+        artifact=artifact,
+        parser_revision="historical-monthly-v1",
+        configuration_hash="c" * 64,
+        result_hash="d" * 64,
+        status=ParseRun.Status.VALIDATED,
+        started_at=completed_at,
+        completed_at=completed_at,
+        total_row_count=1,
+        accepted_row_count=1,
+    )
+    collection = HistoricalSourceCollection.objects.create(
+        kind=HistoricalSourceCollection.Kind.MONTHLY,
+        source_configuration=source,
+        code_manifest_sha256="a" * 64,
+        partition_manifest_sha256=partition_manifest_sha256([scope]),
+        expected_part_count=1,
+        month_min="202512",
+        month_max="202512",
+    )
+    part = HistoricalSourceCollectionPart.objects.create(
+        collection=collection,
+        ordinal=1,
+        partition_scope_sha256=scope,
+        parse_run=parse_run,
+        fact_count=1,
+    )
+    recent = create_series()
+    series = HistoricalRetailSeriesKey.objects.create(
+        recent_series=recent,
+        series_identity_sha256=price_series_identity_sha256(recent),
+        cross_source_evidence_revision="cross-v1",
+        code_manifest_sha256="a" * 64,
+    )
+    region = RetailRegionKey.objects.create(
+        region_code="1101", region_name="서울", identity_evidence_revision="codes-v1"
+    )
+    MonthlyRegionalRetailPrice.objects.create(
+        collection=collection,
+        collection_part=part,
+        series=series,
+        region=region,
+        year_month="202512",
+        provider_mean=1200,
+        provider_low=1000,
+        provider_high=1500,
+        source_row_sha256="f" * 64,
+        source_contract_revision="15156060-v1",
+    )
+
+    completed = complete_historical_collection(collection.id)
+
+    assert completed.state == HistoricalSourceCollection.State.VALIDATED
+    assert completed.accepted_row_count == 1
+    assert len(completed.result_sha256) == 64


