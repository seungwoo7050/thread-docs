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


## `feat(public): validate active historical bundle`

diff --git a/config/settings.py b/config/settings.py
index 8da0f19..1f6f023 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -232,7 +232,7 @@ validate_hsts_configuration(
     preload=SECURE_HSTS_PRELOAD,
 )
 SECURE_CONTENT_TYPE_NOSNIFF = True
-SECURE_REFERRER_POLICY = "same-origin"
+SECURE_REFERRER_POLICY = "no-referrer"
 X_FRAME_OPTIONS = "DENY"
 
 KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
@@ -240,6 +240,16 @@ KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
     36,
     maximum=168,
 )
+KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS = env_positive_int(
+    "KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS",
+    192,
+    maximum=744,
+)
+KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS = env_positive_int(
+    "KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS",
+    36,
+    maximum=168,
+)
 QA_STATE_PREVIEWS_ENABLED = DEBUG and env_bool("QA_STATE_PREVIEWS_ENABLED", False)
 
 LOGGING = {
diff --git a/grocery/historical_public_read.py b/grocery/historical_public_read.py
new file mode 100644
index 0000000..b31980a
--- /dev/null
+++ b/grocery/historical_public_read.py
@@ -0,0 +1,197 @@
+"""Active historical publication and exact recent-series membership boundary."""
+
+from __future__ import annotations
+
+import re
+import uuid
+from dataclasses import dataclass
+from datetime import datetime, timedelta
+from typing import Final, cast
+
+from django.conf import settings
+from django.core.exceptions import ValidationError
+from django.utils import timezone
+
+from grocery.historical_activation_models import HistoricalRetailPublicationChannel
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_daily_models import DailyMarketRetailPrice, DailyRegionalRetailPrice
+from grocery.historical_identity_models import HistoricalRetailSeriesKey
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.presentation import format_korean_datetime, format_unit
+
+HISTORICAL_RETAIL_CHANNEL: Final = "HISTORICAL_RETAIL"
+_SHA256_RE: Final = re.compile(r"^[0-9a-f]{64}$")
+_KIND_REVIEW_FIELDS: Final = {
+    HistoricalSourceCollection.Kind.MONTHLY: "monthly_review",
+    HistoricalSourceCollection.Kind.REGIONAL_DAILY: "regional_review",
+    HistoricalSourceCollection.Kind.MARKET_DAILY: "market_review",
+}
+
+
+class PublicReadIntegrityError(ValidationError):
+    """An active publication cannot be presented without inventing facts."""
+
+
+class PublicParameterError(ValidationError):
+    """A syntactically valid public value is not in the active allowlist."""
+
+
+@dataclass(frozen=True, slots=True)
+class ActiveHistoricalPublication:
+    revision: HistoricalRetailPublicationRevision
+    checked_at: datetime
+    freshness_state: str
+    freshness_label: str
+    stale_message: str
+
+    @property
+    def monthly_collection(self) -> HistoricalSourceCollection:
+        return self.revision.monthly_review.collection
+
+    @property
+    def regional_collection(self) -> HistoricalSourceCollection:
+        return self.revision.regional_review.collection
+
+    @property
+    def market_collection(self) -> HistoricalSourceCollection:
+        return self.revision.market_review.collection
+
+
+def load_active_historical_publication(
+    *, observed_at: datetime | None = None
+) -> ActiveHistoricalPublication | None:
+    """Load and validate the historical pointer independently of recent retail."""
+
+    channel = (
+        HistoricalRetailPublicationChannel.objects.select_related(
+            "current_revision__monthly_review__collection__source_configuration",
+            "current_revision__regional_review__collection__source_configuration",
+            "current_revision__market_review__collection__source_configuration",
+        )
+        .filter(pk=HISTORICAL_RETAIL_CHANNEL)
+        .first()
+    )
+    if channel is None or channel.current_revision is None:
+        return None
+    revision = channel.current_revision
+    if (
+        revision.sealed_at is None
+        or revision.public_copy_revision != HistoricalRetailPublicationRevision.COPY_REVISION
+        or not _SHA256_RE.fullmatch(revision.typed_fact_set_sha256)
+    ):
+        raise PublicReadIntegrityError("The historical pointer is not a sealed public revision.")
+
+    collections: list[HistoricalSourceCollection] = []
+    for expected_kind, review_field in _KIND_REVIEW_FIELDS.items():
+        decision = getattr(revision, review_field)
+        collection = decision.collection
+        if (
+            decision.decision != HistoricalCollectionReviewDecision.Decision.APPROVE
+            or collection.kind != expected_kind
+            or collection.state != HistoricalSourceCollection.State.VALIDATED
+            or collection.completed_at is None
+            or collection.code_manifest_sha256 != revision.code_manifest_sha256
+            or decision.approved_result_sha256 != collection.result_sha256
+            or decision.approved_partition_manifest_sha256 != collection.partition_manifest_sha256
+        ):
+            raise PublicReadIntegrityError(
+                "The historical revision has an invalid reviewed source."
+            )
+        collections.append(collection)
+
+    now = observed_at or timezone.now()
+    monthly, regional, market = collections
+    monthly_completed = cast(datetime, monthly.completed_at)
+    regional_completed = cast(datetime, regional.completed_at)
+    market_completed = cast(datetime, market.completed_at)
+    monthly_age = timedelta(hours=settings.KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS)
+    daily_age = timedelta(hours=settings.KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS)
+    stale = bool(
+        now - monthly_completed > monthly_age
+        or now - regional_completed > daily_age
+        or now - market_completed > daily_age
+    )
+    completed_by_source = {
+        collection.source_configuration_id: cast(datetime, collection.completed_at)
+        for collection in collections
+    }
+    newer_outcome_exists = any(
+        completed_at is not None and completed_at > completed_by_source[source_configuration_id]
+        for source_configuration_id, completed_at in HistoricalSourceCollection.objects.filter(
+            source_configuration_id__in=completed_by_source,
+            completed_at__isnull=False,
+        )
+        .exclude(id__in={collection.id for collection in collections})
+        .values_list("source_configuration_id", "completed_at")
+    )
+    stale = stale or newer_outcome_exists
+    checked_at = min(monthly_completed, regional_completed, market_completed)
+    if stale:
+        return ActiveHistoricalPublication(
+            revision=revision,
+            checked_at=checked_at,
+            freshness_state="stale",
+            freshness_label="마지막 공개 자료 · 최근 확인 필요",
+            stale_message=(
+                "최근 자료 확인이 필요합니다. 마지막으로 검토를 마친 조사값을 표시합니다."
+            ),
+        )
+    return ActiveHistoricalPublication(
+        revision=revision,
+        checked_at=checked_at,
+        freshness_state="current",
+        freshness_label="KAMIS 자료 확인 완료",
+        stale_message="",
+    )
+
+
+def historical_publication_context(active: ActiveHistoricalPublication) -> dict[str, str]:
+    return {
+        "checked_at_iso": active.checked_at.isoformat(),
+        "checked_at_display": format_korean_datetime(active.checked_at),
+        "freshness_state": active.freshness_state,
+        "freshness_label": active.freshness_label,
+    }
+
+
+def historical_series_for_recent(
+    active: ActiveHistoricalPublication, recent_series_id: uuid.UUID
+) -> HistoricalRetailSeriesKey | None:
+    series = (
+        HistoricalRetailSeriesKey.objects.select_related("recent_series")
+        .filter(recent_series_id=recent_series_id)
+        .first()
+    )
+    if series is None:
+        return None
+    if series.code_manifest_sha256 != active.revision.code_manifest_sha256:
+        raise PublicReadIntegrityError("Historical series identity uses a different code manifest.")
+    memberships = (
+        MonthlyRegionalRetailPrice.objects.filter(
+            collection=active.monthly_collection, series=series
+        ).exists(),
+        DailyRegionalRetailPrice.objects.filter(
+            collection=active.regional_collection, series=series
+        ).exists(),
+        DailyMarketRetailPrice.objects.filter(
+            collection=active.market_collection, series=series
+        ).exists(),
+    )
+    if len(set(memberships)) != 1:
+        raise PublicReadIntegrityError("Historical series membership is incomplete.")
+    if not all(memberships):
+        return None
+    return series
+
+
+def historical_series_context(series: HistoricalRetailSeriesKey) -> dict[str, str]:
+    recent = series.recent_series
+    return {
+        "category_label": recent.category_name,
+        "item_name": recent.item_name,
+        "variety_name": recent.variety_name,
+        "grade_name": recent.grade_name,
+        "unit_label": format_unit(recent.raw_unit, recent.raw_unit_size),
+    }


