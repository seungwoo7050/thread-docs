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


## `feat(public): build regional and market ledgers`

diff --git a/grocery/historical_daily_read.py b/grocery/historical_daily_read.py
new file mode 100644
index 0000000..8a596d0
--- /dev/null
+++ b/grocery/historical_daily_read.py
@@ -0,0 +1,170 @@
+"""Regional range and market observation contexts from one active bundle."""
+
+from __future__ import annotations
+
+import uuid
+from datetime import date, timedelta
+from decimal import Decimal
+from typing import Final
+
+from grocery.historical_daily_models import DailyMarketRetailPrice, DailyRegionalRetailPrice
+from grocery.historical_identity_models import HistoricalRetailSeriesKey
+from grocery.historical_public_read import (
+    ActiveHistoricalPublication,
+    PublicParameterError,
+    PublicReadIntegrityError,
+)
+from grocery.presentation import format_korean_date
+from grocery.vnext_presentation import decimal_machine, format_provider_krw, range_meter
+
+HISTORICAL_PAGE_SIZE: Final = 30
+
+
+def regions_context(
+    active: ActiveHistoricalPublication,
+    series: HistoricalRetailSeriesKey,
+    *,
+    selected_date: date | None,
+) -> dict[str, object]:
+    lower_date = active.revision.date_max - timedelta(days=30)
+    regional = list(
+        DailyRegionalRetailPrice.objects.filter(
+            collection=active.regional_collection,
+            series=series,
+            survey_date__range=(lower_date, active.revision.date_max),
+        )
+        .select_related("region")
+        .order_by("survey_date", "region__region_name", "region__region_code", "id")
+    )
+    market_keys = set(
+        DailyMarketRetailPrice.objects.filter(
+            collection=active.market_collection,
+            series=series,
+            survey_date__range=(lower_date, active.revision.date_max),
+        ).values_list("region_id", "survey_date")
+    )
+    shared_dates = sorted(
+        {row.survey_date for row in regional if (row.region_id, row.survey_date) in market_keys},
+        reverse=True,
+    )
+    if not shared_dates:
+        raise PublicReadIntegrityError("The published series has no shared daily survey date.")
+    effective_date = selected_date or shared_dates[0]
+    if effective_date not in shared_dates:
+        raise PublicParameterError("The selected survey date is not available.")
+    rows = [row for row in regional if row.survey_date == effective_date]
+    if not rows:
+        raise PublicReadIntegrityError("The selected published date has no regional facts.")
+    for row in rows:
+        _validate_range(row.provider_low, row.provider_mean, row.provider_high)
+    scale_minimum = min(row.provider_low for row in rows)
+    scale_maximum = max(row.provider_high for row in rows)
+    return {
+        "date_options": [_date_option(value, value == effective_date) for value in shared_dates],
+        "selected_date": _date_value(effective_date),
+        "regional_rows": [
+            {
+                "region_id": row.region_id,
+                "region_label": row.region.region_name,
+                "mean_machine": decimal_machine(row.provider_mean),
+                "mean_label": format_provider_krw(row.provider_mean),
+                "minimum_machine": decimal_machine(row.provider_low),
+                "minimum_label": format_provider_krw(row.provider_low),
+                "maximum_machine": decimal_machine(row.provider_high),
+                "maximum_label": format_provider_krw(row.provider_high),
+                "meter": range_meter(
+                    minimum=row.provider_low,
+                    mean=row.provider_mean,
+                    maximum=row.provider_high,
+                    scale_minimum=scale_minimum,
+                    scale_maximum=scale_maximum,
+                ),
+                "market_available": (row.region_id, effective_date) in market_keys,
+            }
+            for row in rows
+        ],
+    }
+
+
+def markets_context(
+    active: ActiveHistoricalPublication,
+    series: HistoricalRetailSeriesKey,
+    *,
+    region_id: uuid.UUID,
+    selected_date: date | None,
+    page: int,
+) -> dict[str, object]:
+    lower_date = active.revision.date_max - timedelta(days=30)
+    regional_dates = set(
+        DailyRegionalRetailPrice.objects.filter(
+            collection=active.regional_collection,
+            series=series,
+            region_id=region_id,
+            survey_date__range=(lower_date, active.revision.date_max),
+        ).values_list("survey_date", flat=True)
+    )
+    market_facts = list(
+        DailyMarketRetailPrice.objects.filter(
+            collection=active.market_collection,
+            series=series,
+            region_id=region_id,
+            survey_date__range=(lower_date, active.revision.date_max),
+        )
+        .select_related("region", "market")
+        .order_by("survey_date", "market__market_name", "market__market_code", "market_id")
+    )
+    shared_dates = sorted(regional_dates & {row.survey_date for row in market_facts}, reverse=True)
+    if not shared_dates:
+        raise PublicParameterError("The selected region is not available for this series.")
+    effective_date = selected_date or shared_dates[0]
+    if effective_date not in shared_dates:
+        raise PublicParameterError("The selected survey date is not available.")
+    rows = [row for row in market_facts if row.survey_date == effective_date]
+    for row in rows:
+        if (
+            not row.provider_price.is_finite()
+            or row.provider_price <= 0
+            or row.market.region_id != region_id
+        ):
+            raise PublicReadIntegrityError("Published market prices are malformed.")
+    total = len(rows)
+    total_pages = max(1, (total + HISTORICAL_PAGE_SIZE - 1) // HISTORICAL_PAGE_SIZE)
+    if page > total_pages:
+        raise PublicParameterError("The selected page is not available.")
+    start = (page - 1) * HISTORICAL_PAGE_SIZE
+    selected_rows = rows[start : start + HISTORICAL_PAGE_SIZE]
+    region = selected_rows[0].region if selected_rows else market_facts[0].region
+    return {
+        "selected_region": {"value": str(region_id), "label": region.region_name},
+        "date_options": [_date_option(value, value == effective_date) for value in shared_dates],
+        "selected_date": _date_value(effective_date),
+        "market_rows": [
+            {
+                "market_name": row.market.market_name,
+                "price_machine": decimal_machine(row.provider_price),
+                "price_label": format_provider_krw(row.provider_price),
+                "survey_date_iso": row.survey_date.isoformat(),
+                "survey_date_label": format_korean_date(row.survey_date),
+            }
+            for row in selected_rows
+        ],
+        "total_count": total,
+        "page": page,
+        "total_pages": total_pages,
+        "selected_date_value": effective_date,
+    }
+
+
+def _validate_range(minimum: Decimal, mean: Decimal, maximum: Decimal) -> None:
+    if any(not value.is_finite() or value <= 0 for value in (minimum, mean, maximum)):
+        raise PublicReadIntegrityError("Published provider prices are malformed.")
+    if not minimum <= mean <= maximum:
+        raise PublicReadIntegrityError("Published provider ranges are malformed.")
+
+
+def _date_option(value: date, selected: bool) -> dict[str, object]:
+    return {"value": value.isoformat(), "label": format_korean_date(value), "selected": selected}
+
+
+def _date_value(value: date) -> dict[str, str]:
+    return {"iso": value.isoformat(), "label": format_korean_date(value)}


