## `feat(history): persist monthly collection parts`

diff --git a/grocery/historical_generation.py b/grocery/historical_generation.py
new file mode 100644
index 0000000..7e85892
--- /dev/null
+++ b/grocery/historical_generation.py
@@ -0,0 +1,172 @@
+"""Atomic typed persistence for bounded historical parser results."""
+
+from __future__ import annotations
+
+import uuid
+from dataclasses import dataclass
+from typing import Final
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+from django.utils import timezone
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_generation_common import (
+    historical_configuration_sha256,
+    resolve_historical_region,
+    resolve_historical_series,
+)
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+)
+from grocery.source.historical_client import PreparedHistoricalRequest
+from grocery.source.historical_contract import HistoricalDataset
+from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.monthly_history import ParsedMonthlyPriceRow
+
+MONTHLY_PARSER_REVISION: Final = "kamis-15156060-v1"
+MONTHLY_SOURCE_CONTRACT_REVISION: Final = "data-go-15156060-monthly-v1"
+
+
+@dataclass(frozen=True, slots=True)
+class CompletedHistoricalPart:
+    parse_run: ParseRun
+    collection: HistoricalSourceCollection
+    part: HistoricalSourceCollectionPart
+    replayed: bool
+
+
+def _validate_monthly_scope(
+    prepared_request: PreparedHistoricalRequest,
+    rows: tuple[ParsedMonthlyPriceRow, ...],
+) -> tuple[str, str]:
+    conditions = prepared_request.query.conditions
+    month_min = conditions["cond[exmn_ym::GTE]"]
+    month_max = conditions["cond[exmn_ym::LTE]"]
+    filter_fields = {
+        "cond[se_cd::EQ]": "product_class_code",
+        "cond[ctgry_cd::EQ]": "category_code",
+        "cond[item_cd::EQ]": "item_code",
+        "cond[vrty_cd::EQ]": "variety_code",
+        "cond[grd_cd::EQ]": "grade_code",
+    }
+    for row in rows:
+        month = row.source_effective_month.source_text()
+        if not month_min <= month <= month_max:
+            raise ValidationError("Historical monthly row is outside its prepared request.")
+        if any(
+            name in conditions and getattr(row.identity, attribute) != conditions[name]
+            for name, attribute in filter_fields.items()
+        ):
+            raise ValidationError("Historical monthly identity is outside its prepared request.")
+        if (
+            "cond[sgg_cd::EQ]" in conditions
+            and row.region.code != conditions["cond[sgg_cd::EQ]"]
+        ):
+            raise ValidationError("Historical monthly region is outside its prepared request.")
+    return month_min, month_max
+
+
+@transaction.atomic
+def persist_monthly_part(
+    *,
+    collection_id: uuid.UUID,
+    ordinal: int,
+    artifact_id: uuid.UUID,
+    prepared_request: PreparedHistoricalRequest,
+    parsed: ParsedHistoricalResult[ParsedMonthlyPriceRow],
+    code_manifest_sha256: str,
+) -> CompletedHistoricalPart:
+    if prepared_request.query.dataset != HistoricalDataset.MONTHLY:
+        raise ValidationError("Monthly persistence requires the monthly source contract.")
+    if parsed.input_row_count != len(parsed.rows):
+        raise ValidationError("Historical parse rows do not reconcile with the input count.")
+    month_min, month_max = _validate_monthly_scope(prepared_request, parsed.rows)
+    collection = (
+        HistoricalSourceCollection.objects.select_for_update()
+        .select_related("source_configuration")
+        .get(pk=collection_id)
+    )
+    source = collection.source_configuration
+    if collection.state != HistoricalSourceCollection.State.STARTED:
+        raise ValidationError("Historical parts require a started collection.")
+    if collection.kind != HistoricalSourceCollection.Kind.MONTHLY:
+        raise ValidationError("Monthly persistence requires a monthly collection.")
+    if ordinal < 1 or ordinal > collection.expected_part_count:
+        raise ValidationError("Historical part ordinal is outside its collection plan.")
+    artifact = SourceArtifact.objects.select_for_update().get(pk=artifact_id)
+    if not FetchAttempt.objects.select_for_update().filter(
+        source_configuration=source,
+        artifact=artifact,
+        state=FetchAttempt.State.SUCCEEDED,
+        request_scope_sha256=prepared_request.scope_sha256,
+    ).exists():
+        raise ValidationError("Historical artifact does not belong to the prepared source scope.")
+
+    configuration_hash = historical_configuration_sha256(
+        dataset=HistoricalDataset.MONTHLY,
+        parser_revision=MONTHLY_PARSER_REVISION,
+        code_manifest_sha256=code_manifest_sha256,
+    )
+    parse_run, created = ParseRun.objects.get_or_create(
+        artifact=artifact,
+        parser_revision=MONTHLY_PARSER_REVISION,
+        configuration_hash=configuration_hash,
+    )
+    if not created:
+        if (
+            parse_run.status != ParseRun.Status.VALIDATED
+            or parse_run.result_hash != parsed.result_hash
+        ):
+            raise ValidationError("Historical parse replay conflicts with its stored generation.")
+        try:
+            part = parse_run.historical_collection_part
+        except HistoricalSourceCollectionPart.DoesNotExist:
+            raise ValidationError(
+                "Historical parse replay is missing its collection part."
+            ) from None
+        if part.collection_id != collection.id or part.ordinal != ordinal:
+            raise ValidationError("Historical parse replay belongs to another collection part.")
+        return CompletedHistoricalPart(parse_run, collection, part, True)
+
+    if (
+        collection.code_manifest_sha256 != code_manifest_sha256
+        or collection.month_min != month_min
+        or collection.month_max != month_max
+    ):
+        raise ValidationError("Monthly part does not match its planned collection.")
+    parse_run.status = ParseRun.Status.VALIDATED
+    parse_run.completed_at = timezone.now()
+    parse_run.result_hash = parsed.result_hash
+    parse_run.total_row_count = parsed.input_row_count
+    parse_run.accepted_row_count = len(parsed.rows)
+    parse_run.save()
+    part = HistoricalSourceCollectionPart.objects.create(
+        collection=collection,
+        ordinal=1,
+        partition_scope_sha256=prepared_request.scope_sha256,
+        parse_run=parse_run,
+        fact_count=len(parsed.rows),
+    )
+    for row in parsed.rows:
+        MonthlyRegionalRetailPrice.objects.create(
+            collection=collection,
+            collection_part=part,
+            series=resolve_historical_series(
+                row.identity, code_manifest_sha256=code_manifest_sha256
+            ),
+            region=resolve_historical_region(row.region),
+            year_month=row.source_effective_month.source_text(),
+            provider_mean=row.pmm_avgprc,
+            provider_low=row.pmm_lwprc,
+            provider_high=row.pmm_hgprc,
+            source_row_sha256=row.source_row_hash,
+            source_contract_revision=MONTHLY_SOURCE_CONTRACT_REVISION,
+        )
+    return CompletedHistoricalPart(parse_run, collection, part, False)
diff --git a/grocery/tests/test_historical_monthly_generation.py b/grocery/tests/test_historical_monthly_generation.py
new file mode 100644
index 0000000..dbc02cd
--- /dev/null
+++ b/grocery/tests/test_historical_monthly_generation.py
@@ -0,0 +1,82 @@
+import uuid
+from decimal import Decimal
+
+from grocery.historical_collection_plans import plan_historical_collection
+from grocery.historical_collections import complete_historical_collection
+from grocery.historical_generation import persist_monthly_part
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.source.historical_dimensions import RegionObservation, YearMonth
+from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.kamis import IdentityObservation
+from grocery.source.monthly_history import ParsedMonthlyPriceRow
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+from grocery.tests.historical_ingestion_factory import create_historical_artifact
+
+
+def test_monthly_parser_result_persists_candidate_without_publication(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    recent = bundle.series.recent_series
+    source, prepared, artifact = create_historical_artifact(
+        HistoricalDataset.MONTHLY,
+        HistoricalPriceQuery(
+            start="202512", end="202512", category_code="200", item_code="212"
+        ),
+        row_count=1,
+    )
+    row = ParsedMonthlyPriceRow(
+        identity=IdentityObservation(
+            product_class_code=recent.product_class_code,
+            product_class_name=recent.product_class_name,
+            category_code=recent.category_code,
+            category_name=recent.category_name,
+            item_code=recent.item_code,
+            item_name=recent.item_name,
+            variety_code=recent.variety_code,
+            variety_name=recent.variety_name,
+            grade_code=recent.grade_code,
+            grade_name=recent.grade_name,
+            raw_unit=recent.raw_unit,
+            raw_unit_size=recent.raw_unit_size,
+            coverage_identity=recent.coverage_identity,
+        ),
+        region=RegionObservation(bundle.region.region_code, bundle.region.region_name),
+        source_effective_month=YearMonth(2025, 12),
+        pmm_avgprc=Decimal("1000"),
+        pmm_hgprc=Decimal("1100"),
+        pmm_lwprc=Decimal("900"),
+        pmm_stddvtn=Decimal("0"),
+        pmm_cfcntvrtn=Decimal("0"),
+        pmm_cfcntrng=Decimal("0"),
+        pyy_avgprc=Decimal("1000"),
+        pyy_hgprc=Decimal("1100"),
+        pyy_lwprc=Decimal("900"),
+        pyy_stddvtn=Decimal("0"),
+        pyy_cfcntvrtn=Decimal("0"),
+        pyy_cfcntrng=Decimal("0"),
+        source_recorded_at_raw="20251231",
+        source_row_hash="3" * 64,
+    )
+    parsed = ParsedHistoricalResult(rows=(row,), input_row_count=1, result_hash="4" * 64)
+    review_count = HistoricalCollectionReviewDecision.objects.count()
+    collection = plan_historical_collection(
+        collection_id=uuid.uuid4(),
+        source_configuration_id=source.id,
+        prepared_requests=(prepared,),
+        code_manifest_sha256="a" * 64,
+    )
+
+    completed = persist_monthly_part(
+        collection_id=collection.id,
+        ordinal=1,
+        artifact_id=artifact.id,
+        prepared_request=prepared,
+        parsed=parsed,
+        code_manifest_sha256="a" * 64,
+    )
+    validated = complete_historical_collection(collection.id)
+
+    fact = MonthlyRegionalRetailPrice.objects.get(collection=completed.collection)
+    assert (validated.state, fact.provider_mean) == ("VALIDATED", Decimal("1000"))
+    assert HistoricalCollectionReviewDecision.objects.count() == review_count


## `refactor(history): share part transaction boundary`

diff --git a/grocery/historical_generation.py b/grocery/historical_generation.py
index 7e85892..85249a1 100644
--- a/grocery/historical_generation.py
+++ b/grocery/historical_generation.py
@@ -3,28 +3,19 @@
 from __future__ import annotations
 
 import uuid
-from dataclasses import dataclass
 from typing import Final
 
 from django.core.exceptions import ValidationError
 from django.db import transaction
-from django.utils import timezone
 
-from grocery.historical_collection_models import (
-    HistoricalSourceCollection,
-    HistoricalSourceCollectionPart,
-)
+from grocery.historical_collection_models import HistoricalSourceCollection
 from grocery.historical_generation_common import (
-    historical_configuration_sha256,
+    HistoricalPartState,
     resolve_historical_region,
     resolve_historical_series,
+    start_historical_part,
 )
 from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
-from grocery.models import (
-    FetchAttempt,
-    ParseRun,
-    SourceArtifact,
-)
 from grocery.source.historical_client import PreparedHistoricalRequest
 from grocery.source.historical_contract import HistoricalDataset
 from grocery.source.historical_parser import ParsedHistoricalResult
@@ -34,14 +25,6 @@ MONTHLY_PARSER_REVISION: Final = "kamis-15156060-v1"
 MONTHLY_SOURCE_CONTRACT_REVISION: Final = "data-go-15156060-monthly-v1"
 
 
-@dataclass(frozen=True, slots=True)
-class CompletedHistoricalPart:
-    parse_run: ParseRun
-    collection: HistoricalSourceCollection
-    part: HistoricalSourceCollectionPart
-    replayed: bool
-
-
 def _validate_monthly_scope(
     prepared_request: PreparedHistoricalRequest,
     rows: tuple[ParsedMonthlyPriceRow, ...],
@@ -82,7 +65,7 @@ def persist_monthly_part(
     prepared_request: PreparedHistoricalRequest,
     parsed: ParsedHistoricalResult[ParsedMonthlyPriceRow],
     code_manifest_sha256: str,
-) -> CompletedHistoricalPart:
+) -> HistoricalPartState:
     if prepared_request.query.dataset != HistoricalDataset.MONTHLY:
         raise ValidationError("Monthly persistence requires the monthly source contract.")
     if parsed.input_row_count != len(parsed.rows):
@@ -93,71 +76,29 @@ def persist_monthly_part(
         .select_related("source_configuration")
         .get(pk=collection_id)
     )
-    source = collection.source_configuration
-    if collection.state != HistoricalSourceCollection.State.STARTED:
-        raise ValidationError("Historical parts require a started collection.")
     if collection.kind != HistoricalSourceCollection.Kind.MONTHLY:
         raise ValidationError("Monthly persistence requires a monthly collection.")
-    if ordinal < 1 or ordinal > collection.expected_part_count:
-        raise ValidationError("Historical part ordinal is outside its collection plan.")
-    artifact = SourceArtifact.objects.select_for_update().get(pk=artifact_id)
-    if not FetchAttempt.objects.select_for_update().filter(
-        source_configuration=source,
-        artifact=artifact,
-        state=FetchAttempt.State.SUCCEEDED,
-        request_scope_sha256=prepared_request.scope_sha256,
-    ).exists():
-        raise ValidationError("Historical artifact does not belong to the prepared source scope.")
-
-    configuration_hash = historical_configuration_sha256(
+    state = start_historical_part(
+        collection_id=collection.id,
+        ordinal=ordinal,
+        artifact_id=artifact_id,
+        prepared_request=prepared_request,
         dataset=HistoricalDataset.MONTHLY,
+        collection_kind=HistoricalSourceCollection.Kind.MONTHLY,
         parser_revision=MONTHLY_PARSER_REVISION,
         code_manifest_sha256=code_manifest_sha256,
+        parsed_result_sha256=parsed.result_hash,
+        input_row_count=parsed.input_row_count,
+        accepted_row_count=len(parsed.rows),
     )
-    parse_run, created = ParseRun.objects.get_or_create(
-        artifact=artifact,
-        parser_revision=MONTHLY_PARSER_REVISION,
-        configuration_hash=configuration_hash,
-    )
-    if not created:
-        if (
-            parse_run.status != ParseRun.Status.VALIDATED
-            or parse_run.result_hash != parsed.result_hash
-        ):
-            raise ValidationError("Historical parse replay conflicts with its stored generation.")
-        try:
-            part = parse_run.historical_collection_part
-        except HistoricalSourceCollectionPart.DoesNotExist:
-            raise ValidationError(
-                "Historical parse replay is missing its collection part."
-            ) from None
-        if part.collection_id != collection.id or part.ordinal != ordinal:
-            raise ValidationError("Historical parse replay belongs to another collection part.")
-        return CompletedHistoricalPart(parse_run, collection, part, True)
-
-    if (
-        collection.code_manifest_sha256 != code_manifest_sha256
-        or collection.month_min != month_min
-        or collection.month_max != month_max
-    ):
+    if collection.month_min != month_min or collection.month_max != month_max:
         raise ValidationError("Monthly part does not match its planned collection.")
-    parse_run.status = ParseRun.Status.VALIDATED
-    parse_run.completed_at = timezone.now()
-    parse_run.result_hash = parsed.result_hash
-    parse_run.total_row_count = parsed.input_row_count
-    parse_run.accepted_row_count = len(parsed.rows)
-    parse_run.save()
-    part = HistoricalSourceCollectionPart.objects.create(
-        collection=collection,
-        ordinal=1,
-        partition_scope_sha256=prepared_request.scope_sha256,
-        parse_run=parse_run,
-        fact_count=len(parsed.rows),
-    )
+    if state.replayed:
+        return state
     for row in parsed.rows:
         MonthlyRegionalRetailPrice.objects.create(
             collection=collection,
-            collection_part=part,
+            collection_part=state.part,
             series=resolve_historical_series(
                 row.identity, code_manifest_sha256=code_manifest_sha256
             ),
@@ -169,4 +110,4 @@ def persist_monthly_part(
             source_row_sha256=row.source_row_hash,
             source_contract_revision=MONTHLY_SOURCE_CONTRACT_REVISION,
         )
-    return CompletedHistoricalPart(parse_run, collection, part, False)
+    return state
diff --git a/grocery/historical_generation_common.py b/grocery/historical_generation_common.py
index f004e4c..fc76b9d 100644
--- a/grocery/historical_generation_common.py
+++ b/grocery/historical_generation_common.py
@@ -4,16 +4,114 @@ from __future__ import annotations
 
 import hashlib
 import json
+import uuid
+from dataclasses import dataclass
 
 from django.core.exceptions import ValidationError
+from django.db import transaction
+from django.utils import timezone
 
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
 from grocery.historical_identity_models import HistoricalRetailSeriesKey, RetailRegionKey
-from grocery.models import PriceSeriesKey
+from grocery.models import FetchAttempt, ParseRun, PriceSeriesKey, SourceArtifact
+from grocery.source.historical_client import PreparedHistoricalRequest
 from grocery.source.historical_contract import HistoricalDataset
 from grocery.source.historical_dimensions import RegionObservation
 from grocery.source.kamis import IdentityObservation
 
 
+@dataclass(frozen=True, slots=True)
+class HistoricalPartState:
+    parse_run: ParseRun
+    collection: HistoricalSourceCollection
+    part: HistoricalSourceCollectionPart
+    replayed: bool
+
+
+@transaction.atomic
+def start_historical_part(
+    *,
+    collection_id: uuid.UUID,
+    ordinal: int,
+    artifact_id: uuid.UUID,
+    prepared_request: PreparedHistoricalRequest,
+    dataset: HistoricalDataset,
+    collection_kind: str,
+    parser_revision: str,
+    code_manifest_sha256: str,
+    parsed_result_sha256: str,
+    input_row_count: int,
+    accepted_row_count: int,
+) -> HistoricalPartState:
+    if input_row_count != accepted_row_count:
+        raise ValidationError("Historical parser rows do not reconcile with the input count.")
+    if prepared_request.query.dataset != dataset:
+        raise ValidationError("Historical parser dataset does not match the prepared request.")
+    collection = (
+        HistoricalSourceCollection.objects.select_for_update()
+        .select_related("source_configuration")
+        .get(pk=collection_id)
+    )
+    if (
+        collection.kind != collection_kind
+        or collection.code_manifest_sha256 != code_manifest_sha256
+    ):
+        raise ValidationError("Historical part does not match its planned collection.")
+    if ordinal < 1 or ordinal > collection.expected_part_count:
+        raise ValidationError("Historical part ordinal is outside its collection plan.")
+    artifact = SourceArtifact.objects.select_for_update().get(pk=artifact_id)
+    if not FetchAttempt.objects.select_for_update().filter(
+        source_configuration=collection.source_configuration,
+        artifact=artifact,
+        state=FetchAttempt.State.SUCCEEDED,
+        request_scope_sha256=prepared_request.scope_sha256,
+    ).exists():
+        raise ValidationError("Historical artifact does not belong to the prepared source scope.")
+    parse_run, created = ParseRun.objects.get_or_create(
+        artifact=artifact,
+        parser_revision=parser_revision,
+        configuration_hash=historical_configuration_sha256(
+            dataset=dataset,
+            parser_revision=parser_revision,
+            code_manifest_sha256=code_manifest_sha256,
+        ),
+    )
+    if not created:
+        if (
+            parse_run.status != ParseRun.Status.VALIDATED
+            or parse_run.result_hash != parsed_result_sha256
+        ):
+            raise ValidationError("Historical parse replay conflicts with its stored generation.")
+        try:
+            part = parse_run.historical_collection_part
+        except HistoricalSourceCollectionPart.DoesNotExist:
+            raise ValidationError(
+                "Historical parse replay is missing its collection part."
+            ) from None
+        if part.collection_id != collection.id or part.ordinal != ordinal:
+            raise ValidationError("Historical parse replay belongs to another collection part.")
+        return HistoricalPartState(parse_run, collection, part, True)
+    if collection.state != HistoricalSourceCollection.State.STARTED:
+        raise ValidationError("New historical parts require a started collection.")
+    parse_run.status = ParseRun.Status.VALIDATED
+    parse_run.completed_at = timezone.now()
+    parse_run.result_hash = parsed_result_sha256
+    parse_run.total_row_count = input_row_count
+    parse_run.accepted_row_count = accepted_row_count
+    parse_run.save()
+    part = HistoricalSourceCollectionPart.objects.create(
+        collection=collection,
+        ordinal=ordinal,
+        partition_scope_sha256=prepared_request.scope_sha256,
+        parse_run=parse_run,
+        fact_count=accepted_row_count,
+    )
+    return HistoricalPartState(parse_run, collection, part, False)
+
+
 def historical_configuration_sha256(
     *,
     dataset: HistoricalDataset,
diff --git a/grocery/tests/test_historical_monthly_generation.py b/grocery/tests/test_historical_monthly_generation.py
index dbc02cd..f1c869e 100644
--- a/grocery/tests/test_historical_monthly_generation.py
+++ b/grocery/tests/test_historical_monthly_generation.py
@@ -76,7 +76,16 @@ def test_monthly_parser_result_persists_candidate_without_publication(db: None)
         code_manifest_sha256="a" * 64,
     )
     validated = complete_historical_collection(collection.id)
+    replay = persist_monthly_part(
+        collection_id=collection.id,
+        ordinal=1,
+        artifact_id=artifact.id,
+        prepared_request=prepared,
+        parsed=parsed,
+        code_manifest_sha256="a" * 64,
+    )
 
     fact = MonthlyRegionalRetailPrice.objects.get(collection=completed.collection)
     assert (validated.state, fact.provider_mean) == ("VALIDATED", Decimal("1000"))
+    assert replay.replayed is True
     assert HistoricalCollectionReviewDecision.objects.count() == review_count


