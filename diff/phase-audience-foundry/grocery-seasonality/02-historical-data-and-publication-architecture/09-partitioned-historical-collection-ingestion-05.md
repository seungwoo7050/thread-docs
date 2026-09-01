## `feat(history): persist regional collection parts`

diff --git a/grocery/historical_daily_generation.py b/grocery/historical_daily_generation.py
new file mode 100644
index 0000000..861c3e1
--- /dev/null
+++ b/grocery/historical_daily_generation.py
@@ -0,0 +1,106 @@
+"""Typed persistence for regional and market daily historical source parts."""
+
+from __future__ import annotations
+
+import uuid
+from datetime import datetime
+from typing import Final
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_daily_models import DailyRegionalRetailPrice
+from grocery.historical_generation_common import (
+    HistoricalPartState,
+    resolve_historical_region,
+    resolve_historical_series,
+    start_historical_part,
+)
+from grocery.source.historical_client import PreparedHistoricalRequest
+from grocery.source.historical_contract import HistoricalDataset
+from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.regional_history import ParsedRegionalPriceRow
+
+REGIONAL_PARSER_REVISION: Final = "kamis-15156062-v1"
+REGIONAL_SOURCE_CONTRACT_REVISION: Final = "data-go-15156062-regional-v1"
+
+
+def _validate_regional_scope(
+    prepared_request: PreparedHistoricalRequest,
+    rows: tuple[ParsedRegionalPriceRow, ...],
+) -> tuple[str, str]:
+    conditions = prepared_request.query.conditions
+    date_min = conditions["cond[exmn_ymd::GTE]"]
+    date_max = conditions["cond[exmn_ymd::LTE]"]
+    filters = {
+        "cond[se_cd::EQ]": "product_class_code",
+        "cond[ctgry_cd::EQ]": "category_code",
+        "cond[item_cd::EQ]": "item_code",
+        "cond[vrty_cd::EQ]": "variety_code",
+        "cond[grd_cd::EQ]": "grade_code",
+    }
+    for row in rows:
+        source_date = row.source_effective_date.strftime("%Y%m%d")
+        if not date_min <= source_date <= date_max:
+            raise ValidationError("Historical regional row is outside its prepared request.")
+        if any(
+            name in conditions and getattr(row.identity, attribute) != conditions[name]
+            for name, attribute in filters.items()
+        ):
+            raise ValidationError("Historical regional identity is outside its prepared request.")
+        if row.region.code != conditions["cond[sgg_cd::EQ]"]:
+            raise ValidationError("Historical regional region is outside its prepared request.")
+    return date_min, date_max
+
+
+@transaction.atomic
+def persist_regional_part(
+    *,
+    collection_id: uuid.UUID,
+    ordinal: int,
+    artifact_id: uuid.UUID,
+    prepared_request: PreparedHistoricalRequest,
+    parsed: ParsedHistoricalResult[ParsedRegionalPriceRow],
+    code_manifest_sha256: str,
+) -> HistoricalPartState:
+    if prepared_request.query.dataset != HistoricalDataset.REGIONAL:
+        raise ValidationError("Regional persistence requires the regional source contract.")
+    date_min, date_max = _validate_regional_scope(prepared_request, parsed.rows)
+    collection = HistoricalSourceCollection.objects.select_for_update().get(pk=collection_id)
+    if (
+        collection.date_min != datetime.strptime(date_min, "%Y%m%d").date()
+        or collection.date_max != datetime.strptime(date_max, "%Y%m%d").date()
+    ):
+        raise ValidationError("Regional part does not match its planned collection window.")
+    state = start_historical_part(
+        collection_id=collection.id,
+        ordinal=ordinal,
+        artifact_id=artifact_id,
+        prepared_request=prepared_request,
+        dataset=HistoricalDataset.REGIONAL,
+        collection_kind=HistoricalSourceCollection.Kind.REGIONAL_DAILY,
+        parser_revision=REGIONAL_PARSER_REVISION,
+        code_manifest_sha256=code_manifest_sha256,
+        parsed_result_sha256=parsed.result_hash,
+        input_row_count=parsed.input_row_count,
+        accepted_row_count=len(parsed.rows),
+    )
+    if state.replayed:
+        return state
+    for row in parsed.rows:
+        DailyRegionalRetailPrice.objects.create(
+            collection=collection,
+            collection_part=state.part,
+            series=resolve_historical_series(
+                row.identity, code_manifest_sha256=code_manifest_sha256
+            ),
+            region=resolve_historical_region(row.region),
+            survey_date=row.source_effective_date,
+            provider_mean=row.raw_average_price,
+            provider_low=row.raw_min_price,
+            provider_high=row.raw_max_price,
+            source_row_sha256=row.source_row_hash,
+            source_contract_revision=REGIONAL_SOURCE_CONTRACT_REVISION,
+        )
+    return state
diff --git a/grocery/tests/test_historical_regional_generation.py b/grocery/tests/test_historical_regional_generation.py
new file mode 100644
index 0000000..ec09a8c
--- /dev/null
+++ b/grocery/tests/test_historical_regional_generation.py
@@ -0,0 +1,77 @@
+import uuid
+from datetime import date
+from decimal import Decimal
+
+from grocery.historical_collection_plans import plan_historical_collection
+from grocery.historical_collections import complete_historical_collection
+from grocery.historical_daily_generation import persist_regional_part
+from grocery.historical_daily_models import DailyRegionalRetailPrice
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.source.historical_dimensions import RegionObservation
+from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.kamis import IdentityObservation
+from grocery.source.regional_history import ParsedRegionalPriceRow
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+from grocery.tests.historical_ingestion_factory import create_historical_artifact
+
+
+def test_regional_parser_result_persists_one_planned_part(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    recent = bundle.series.recent_series
+    query = HistoricalPriceQuery(
+        start="20251231",
+        end="20251231",
+        category_code="200",
+        item_code="212",
+        region_code=bundle.region.region_code,
+    )
+    source, prepared, artifact = create_historical_artifact(
+        HistoricalDataset.REGIONAL, query, row_count=1
+    )
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
+    row = ParsedRegionalPriceRow(
+        identity=identity,
+        region=RegionObservation(bundle.region.region_code, bundle.region.region_name),
+        source_effective_date=date(2025, 12, 31),
+        raw_min_price=Decimal("900"),
+        raw_average_price=Decimal("1000"),
+        raw_max_price=Decimal("1100"),
+        converted_min_price=Decimal("900"),
+        converted_average_price=Decimal("1000"),
+        converted_max_price=Decimal("1100"),
+        source_row_hash="5" * 64,
+    )
+    parsed = ParsedHistoricalResult(rows=(row,), input_row_count=1, result_hash="6" * 64)
+    collection = plan_historical_collection(
+        collection_id=uuid.uuid4(),
+        source_configuration_id=source.id,
+        prepared_requests=(prepared,),
+        code_manifest_sha256="a" * 64,
+    )
+
+    persist_regional_part(
+        collection_id=collection.id,
+        ordinal=1,
+        artifact_id=artifact.id,
+        prepared_request=prepared,
+        parsed=parsed,
+        code_manifest_sha256="a" * 64,
+    )
+    completed = complete_historical_collection(collection.id)
+
+    fact = DailyRegionalRetailPrice.objects.get(collection=completed)
+    assert (completed.state, fact.provider_mean) == ("VALIDATED", Decimal("1000"))


## `feat(history): persist market collection parts`

diff --git a/grocery/historical_daily_generation.py b/grocery/historical_daily_generation.py
index 861c3e1..b7f6844 100644
--- a/grocery/historical_daily_generation.py
+++ b/grocery/historical_daily_generation.py
@@ -10,9 +10,10 @@ from django.core.exceptions import ValidationError
 from django.db import transaction
 
 from grocery.historical_collection_models import HistoricalSourceCollection
-from grocery.historical_daily_models import DailyRegionalRetailPrice
+from grocery.historical_daily_models import DailyMarketRetailPrice, DailyRegionalRetailPrice
 from grocery.historical_generation_common import (
     HistoricalPartState,
+    resolve_historical_market,
     resolve_historical_region,
     resolve_historical_series,
     start_historical_part,
@@ -20,10 +21,13 @@ from grocery.historical_generation_common import (
 from grocery.source.historical_client import PreparedHistoricalRequest
 from grocery.source.historical_contract import HistoricalDataset
 from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.market_history import ParsedMarketPriceRow
 from grocery.source.regional_history import ParsedRegionalPriceRow
 
 REGIONAL_PARSER_REVISION: Final = "kamis-15156062-v1"
 REGIONAL_SOURCE_CONTRACT_REVISION: Final = "data-go-15156062-regional-v1"
+MARKET_PARSER_REVISION: Final = "kamis-15156065-v1"
+MARKET_SOURCE_CONTRACT_REVISION: Final = "data-go-15156065-market-v1"
 
 
 def _validate_regional_scope(
@@ -104,3 +108,88 @@ def persist_regional_part(
             source_contract_revision=REGIONAL_SOURCE_CONTRACT_REVISION,
         )
     return state
+
+
+def _validate_market_scope(
+    prepared_request: PreparedHistoricalRequest,
+    rows: tuple[ParsedMarketPriceRow, ...],
+) -> tuple[str, str]:
+    conditions = prepared_request.query.conditions
+    date_min = conditions["cond[exmn_ymd::GTE]"]
+    date_max = conditions["cond[exmn_ymd::LTE]"]
+    filters = {
+        "cond[ctgry_cd::EQ]": "category_code",
+        "cond[item_cd::EQ]": "item_code",
+        "cond[vrty_cd::EQ]": "variety_code",
+        "cond[grd_cd::EQ]": "grade_code",
+    }
+    for row in rows:
+        source_date = row.source_effective_date.strftime("%Y%m%d")
+        if not date_min <= source_date <= date_max:
+            raise ValidationError("Historical market row is outside its prepared request.")
+        if any(
+            name in conditions and getattr(row.identity, attribute) != conditions[name]
+            for name, attribute in filters.items()
+        ):
+            raise ValidationError("Historical market identity is outside its prepared request.")
+        if "cond[sgg_cd::EQ]" in conditions and (
+            row.region.code != conditions["cond[sgg_cd::EQ]"]
+        ):
+            raise ValidationError("Historical market region is outside its prepared request.")
+        if "cond[mrkt_cd::EQ]" in conditions and (
+            row.market.code != conditions["cond[mrkt_cd::EQ]"]
+        ):
+            raise ValidationError("Historical market is outside its prepared request.")
+    return date_min, date_max
+
+
+@transaction.atomic
+def persist_market_part(
+    *,
+    collection_id: uuid.UUID,
+    ordinal: int,
+    artifact_id: uuid.UUID,
+    prepared_request: PreparedHistoricalRequest,
+    parsed: ParsedHistoricalResult[ParsedMarketPriceRow],
+    code_manifest_sha256: str,
+) -> HistoricalPartState:
+    if prepared_request.query.dataset != HistoricalDataset.MARKET:
+        raise ValidationError("Market persistence requires the market source contract.")
+    date_min, date_max = _validate_market_scope(prepared_request, parsed.rows)
+    collection = HistoricalSourceCollection.objects.select_for_update().get(pk=collection_id)
+    if (
+        collection.date_min != datetime.strptime(date_min, "%Y%m%d").date()
+        or collection.date_max != datetime.strptime(date_max, "%Y%m%d").date()
+    ):
+        raise ValidationError("Market part does not match its planned collection window.")
+    state = start_historical_part(
+        collection_id=collection.id,
+        ordinal=ordinal,
+        artifact_id=artifact_id,
+        prepared_request=prepared_request,
+        dataset=HistoricalDataset.MARKET,
+        collection_kind=HistoricalSourceCollection.Kind.MARKET_DAILY,
+        parser_revision=MARKET_PARSER_REVISION,
+        code_manifest_sha256=code_manifest_sha256,
+        parsed_result_sha256=parsed.result_hash,
+        input_row_count=parsed.input_row_count,
+        accepted_row_count=len(parsed.rows),
+    )
+    if state.replayed:
+        return state
+    for row in parsed.rows:
+        region = resolve_historical_region(row.region)
+        DailyMarketRetailPrice.objects.create(
+            collection=collection,
+            collection_part=state.part,
+            series=resolve_historical_series(
+                row.identity, code_manifest_sha256=code_manifest_sha256
+            ),
+            region=region,
+            market=resolve_historical_market(region, row.market),
+            survey_date=row.source_effective_date,
+            provider_price=row.raw_observed_price,
+            source_row_sha256=row.source_row_hash,
+            source_contract_revision=MARKET_SOURCE_CONTRACT_REVISION,
+        )
+    return state
diff --git a/grocery/historical_generation_common.py b/grocery/historical_generation_common.py
index fc76b9d..fa4e0f5 100644
--- a/grocery/historical_generation_common.py
+++ b/grocery/historical_generation_common.py
@@ -15,11 +15,15 @@ from grocery.historical_collection_models import (
     HistoricalSourceCollection,
     HistoricalSourceCollectionPart,
 )
-from grocery.historical_identity_models import HistoricalRetailSeriesKey, RetailRegionKey
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailMarketKey,
+    RetailRegionKey,
+)
 from grocery.models import FetchAttempt, ParseRun, PriceSeriesKey, SourceArtifact
 from grocery.source.historical_client import PreparedHistoricalRequest
 from grocery.source.historical_contract import HistoricalDataset
-from grocery.source.historical_dimensions import RegionObservation
+from grocery.source.historical_dimensions import MarketObservation, RegionObservation
 from grocery.source.kamis import IdentityObservation
 
 
@@ -164,3 +168,13 @@ def resolve_historical_region(observation: RegionObservation) -> RetailRegionKey
     if region.region_name != observation.name:
         raise ValidationError("Historical row region name drifted from reviewed evidence.")
     return region
+
+
+def resolve_historical_market(
+    region: RetailRegionKey,
+    observation: MarketObservation,
+) -> RetailMarketKey:
+    market = RetailMarketKey.objects.get(region=region, market_code=observation.code)
+    if market.market_name != observation.name:
+        raise ValidationError("Historical row market name drifted from reviewed evidence.")
+    return market
diff --git a/grocery/tests/test_historical_market_generation.py b/grocery/tests/test_historical_market_generation.py
new file mode 100644
index 0000000..6b7a628
--- /dev/null
+++ b/grocery/tests/test_historical_market_generation.py
@@ -0,0 +1,74 @@
+import uuid
+from datetime import date
+from decimal import Decimal
+
+from grocery.historical_collection_plans import plan_historical_collection
+from grocery.historical_collections import complete_historical_collection
+from grocery.historical_daily_generation import persist_market_part
+from grocery.historical_daily_models import DailyMarketRetailPrice
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.source.historical_dimensions import MarketObservation, RegionObservation
+from grocery.source.historical_parser import ParsedHistoricalResult
+from grocery.source.kamis import IdentityObservation
+from grocery.source.market_history import ParsedMarketPriceRow
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+from grocery.tests.historical_ingestion_factory import create_historical_artifact
+
+
+def test_market_parser_result_persists_one_planned_part(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    recent = bundle.series.recent_series
+    query = HistoricalPriceQuery(
+        start="20251231",
+        end="20251231",
+        category_code="200",
+        item_code="212",
+        region_code=bundle.region.region_code,
+    )
+    source, prepared, artifact = create_historical_artifact(
+        HistoricalDataset.MARKET, query, row_count=1
+    )
+    row = ParsedMarketPriceRow(
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
+        market=MarketObservation(bundle.market.market_code, bundle.market.market_name),
+        source_effective_date=date(2025, 12, 31),
+        raw_observed_price=Decimal("1000"),
+        converted_observed_price=Decimal("1000"),
+        source_recorded_at_raw="20251231",
+        source_row_hash="7" * 64,
+    )
+    parsed = ParsedHistoricalResult(rows=(row,), input_row_count=1, result_hash="8" * 64)
+    collection = plan_historical_collection(
+        collection_id=uuid.uuid4(),
+        source_configuration_id=source.id,
+        prepared_requests=(prepared,),
+        code_manifest_sha256="a" * 64,
+    )
+
+    persist_market_part(
+        collection_id=collection.id,
+        ordinal=1,
+        artifact_id=artifact.id,
+        prepared_request=prepared,
+        parsed=parsed,
+        code_manifest_sha256="a" * 64,
+    )
+    completed = complete_historical_collection(collection.id)
+
+    fact = DailyMarketRetailPrice.objects.get(collection=completed)
+    assert (completed.state, fact.provider_price) == ("VALIDATED", Decimal("1000"))


