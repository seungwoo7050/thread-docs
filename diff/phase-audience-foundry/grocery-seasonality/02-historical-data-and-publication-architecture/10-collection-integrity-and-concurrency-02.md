## `fix(history): lock collection fact membership`

diff --git a/grocery/historical_collection_models.py b/grocery/historical_collection_models.py
index e404074..e4a9433 100644
--- a/grocery/historical_collection_models.py
+++ b/grocery/historical_collection_models.py
@@ -14,6 +14,7 @@ from django.utils import timezone
 from grocery.historical_identity_models import YEAR_MONTH_PATTERN
 from grocery.models import (
     SHA256_PATTERN,
+    FetchAttempt,
     ParseRun,
     SourceConfiguration,
     sha256_validator,
@@ -224,3 +225,10 @@ class HistoricalSourceCollectionPart(models.Model):
             raise ValidationError("Parts can only be attached to a started collection.")
         if self.parse_run_id and self.parse_run.status != ParseRun.Status.VALIDATED:
             raise ValidationError("Collection parts require a validated parse run.")
+        if self.collection_id and self.parse_run_id and not FetchAttempt.objects.filter(
+            source_configuration_id=self.collection.source_configuration_id,
+            artifact_id=self.parse_run.artifact_id,
+            state=FetchAttempt.State.SUCCEEDED,
+            request_scope_sha256=self.partition_scope_sha256,
+        ).exists():
+            raise ValidationError("Collection part does not match its successful request scope.")
diff --git a/grocery/historical_collections.py b/grocery/historical_collections.py
index 4392b83..321c96f 100644
--- a/grocery/historical_collections.py
+++ b/grocery/historical_collections.py
@@ -64,9 +64,10 @@ def complete_historical_collection(collection_id: uuid.UUID) -> HistoricalSource
             source_configuration=collection.source_configuration,
             artifact=parse_run.artifact,
             state=FetchAttempt.State.SUCCEEDED,
+            request_scope_sha256=part.partition_scope_sha256,
         ).exists():
             raise ValidationError(
-                "Historical collection parse source does not match the collection."
+                "Historical collection parse source does not match the collection scope."
             )
         actual_count = selected_model.objects.filter(
             collection=collection,
diff --git a/grocery/historical_daily_models.py b/grocery/historical_daily_models.py
index 4439972..4226386 100644
--- a/grocery/historical_daily_models.py
+++ b/grocery/historical_daily_models.py
@@ -62,6 +62,10 @@ class DailyRegionalRetailPrice(HistoricalPriceFact):
             and self.collection.kind != HistoricalSourceCollection.Kind.REGIONAL_DAILY
         ):
             raise ValidationError("Regional facts require a regional-daily collection.")
+        if self.collection_id and not (
+            self.collection.date_min <= self.survey_date <= self.collection.date_max
+        ):
+            raise ValidationError("Regional fact is outside its collection window.")
 
 
 class DailyMarketRetailPrice(HistoricalPriceFact):
@@ -103,3 +107,7 @@ class DailyMarketRetailPrice(HistoricalPriceFact):
             raise ValidationError("Market facts require a market-daily collection.")
         if self.market_id and self.region_id and self.market.region_id != self.region_id:
             raise ValidationError("Market and fact region do not match.")
+        if self.collection_id and not (
+            self.collection.date_min <= self.survey_date <= self.collection.date_max
+        ):
+            raise ValidationError("Market fact is outside its collection window.")
diff --git a/grocery/historical_fact_base.py b/grocery/historical_fact_base.py
index a2cee47..8890283 100644
--- a/grocery/historical_fact_base.py
+++ b/grocery/historical_fact_base.py
@@ -46,3 +46,8 @@ class HistoricalPriceFact(models.Model):
         super().clean()
         if self.collection_part_id and self.collection_part.collection_id != self.collection_id:
             raise ValidationError("Historical fact collection and part do not match.")
+        if (
+            self.collection_id
+            and self.collection.state != HistoricalSourceCollection.State.STARTED
+        ):
+            raise ValidationError("Historical facts can only be attached to a started collection.")
diff --git a/grocery/historical_monthly_models.py b/grocery/historical_monthly_models.py
index 1afdbc5..f577430 100644
--- a/grocery/historical_monthly_models.py
+++ b/grocery/historical_monthly_models.py
@@ -64,3 +64,7 @@ class MonthlyRegionalRetailPrice(HistoricalPriceFact):
         super().clean()
         if self.collection_id and self.collection.kind != HistoricalSourceCollection.Kind.MONTHLY:
             raise ValidationError("Monthly facts require a monthly collection.")
+        if self.collection_id and not (
+            self.collection.month_min <= self.year_month <= self.collection.month_max
+        ):
+            raise ValidationError("Monthly fact is outside its collection window.")
diff --git a/grocery/migrations/0022_guard_historical_collection_membership.py b/grocery/migrations/0022_guard_historical_collection_membership.py
new file mode 100644
index 0000000..33beea9
--- /dev/null
+++ b/grocery/migrations/0022_guard_historical_collection_membership.py
@@ -0,0 +1,171 @@
+from django.db import migrations
+
+CREATE_GUARDS = r"""
+CREATE OR REPLACE FUNCTION grocery_guard_historical_collection_part()
+RETURNS trigger AS $$
+DECLARE
+    collection_source uuid;
+    collection_state varchar;
+    parse_artifact uuid;
+    parse_status varchar;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical collection parts are append-only';
+    END IF;
+    SELECT source_configuration_id, state
+      INTO collection_source, collection_state
+      FROM grocery_historicalsourcecollection
+     WHERE id = NEW.collection_id
+     FOR KEY SHARE;
+    SELECT artifact_id, status
+      INTO parse_artifact, parse_status
+      FROM grocery_parserun
+     WHERE id = NEW.parse_run_id
+     FOR KEY SHARE;
+    IF collection_state IS DISTINCT FROM 'STARTED'
+       OR parse_status IS DISTINCT FROM 'VALIDATED'
+       OR NOT EXISTS (
+           SELECT 1
+             FROM grocery_fetchattempt
+            WHERE source_configuration_id = collection_source
+              AND artifact_id = parse_artifact
+              AND state = 'SUCCEEDED'
+              AND request_scope_sha256 = NEW.partition_scope_sha256
+       ) THEN
+        RAISE EXCEPTION 'historical collection part does not match a started audited scope';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_monthly_fact()
+RETURNS trigger AS $$
+DECLARE
+    collection_state varchar;
+    collection_kind varchar;
+    window_min varchar;
+    window_max varchar;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical monthly facts are append-only';
+    END IF;
+    SELECT state, kind, month_min, month_max
+      INTO collection_state, collection_kind, window_min, window_max
+      FROM grocery_historicalsourcecollection
+     WHERE id = NEW.collection_id
+     FOR KEY SHARE;
+    IF collection_state IS DISTINCT FROM 'STARTED'
+       OR collection_kind IS DISTINCT FROM 'MONTHLY'
+       OR NEW.year_month NOT BETWEEN window_min AND window_max
+       OR NOT EXISTS (
+        SELECT 1
+          FROM grocery_historicalsourcecollectionpart p
+         WHERE p.collection_id = NEW.collection_id
+           AND p.id = NEW.collection_part_id
+    ) THEN
+        RAISE EXCEPTION 'historical monthly fact is outside its started collection';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_regional_fact()
+RETURNS trigger AS $$
+DECLARE
+    collection_state varchar;
+    collection_kind varchar;
+    window_min date;
+    window_max date;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical regional facts are append-only';
+    END IF;
+    SELECT state, kind, date_min, date_max
+      INTO collection_state, collection_kind, window_min, window_max
+      FROM grocery_historicalsourcecollection
+     WHERE id = NEW.collection_id
+     FOR KEY SHARE;
+    IF collection_state IS DISTINCT FROM 'STARTED'
+       OR collection_kind IS DISTINCT FROM 'REGIONAL_DAILY'
+       OR NEW.survey_date NOT BETWEEN window_min AND window_max
+       OR NOT EXISTS (
+        SELECT 1
+          FROM grocery_historicalsourcecollectionpart p
+         WHERE p.collection_id = NEW.collection_id
+           AND p.id = NEW.collection_part_id
+    ) THEN
+        RAISE EXCEPTION 'historical regional fact is outside its started collection';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_market_fact()
+RETURNS trigger AS $$
+DECLARE
+    collection_state varchar;
+    collection_kind varchar;
+    window_min date;
+    window_max date;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical market facts are append-only';
+    END IF;
+    SELECT state, kind, date_min, date_max
+      INTO collection_state, collection_kind, window_min, window_max
+      FROM grocery_historicalsourcecollection
+     WHERE id = NEW.collection_id
+     FOR KEY SHARE;
+    IF collection_state IS DISTINCT FROM 'STARTED'
+       OR collection_kind IS DISTINCT FROM 'MARKET_DAILY'
+       OR NEW.survey_date NOT BETWEEN window_min AND window_max
+       OR NOT EXISTS (
+        SELECT 1
+          FROM grocery_historicalsourcecollectionpart p
+          JOIN grocery_retailmarketkey m
+            ON m.id = NEW.market_id
+         WHERE p.collection_id = NEW.collection_id
+           AND p.id = NEW.collection_part_id
+           AND m.region_id = NEW.region_id
+    ) THEN
+        RAISE EXCEPTION 'historical market fact is outside its started collection';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE TRIGGER grocery_history_part_append_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_historicalsourcecollectionpart
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_collection_part();
+
+CREATE TRIGGER grocery_monthly_fact_append_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_monthlyregionalretailprice
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_monthly_fact();
+
+CREATE TRIGGER grocery_regional_fact_append_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_dailyregionalretailprice
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_regional_fact();
+
+CREATE TRIGGER grocery_market_fact_append_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_dailymarketretailprice
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_market_fact();
+"""
+
+
+DROP_GUARDS = r"""
+DROP TRIGGER IF EXISTS grocery_market_fact_append_only ON grocery_dailymarketretailprice;
+DROP TRIGGER IF EXISTS grocery_regional_fact_append_only ON grocery_dailyregionalretailprice;
+DROP TRIGGER IF EXISTS grocery_monthly_fact_append_only ON grocery_monthlyregionalretailprice;
+DROP TRIGGER IF EXISTS grocery_history_part_append_only
+    ON grocery_historicalsourcecollectionpart;
+DROP FUNCTION IF EXISTS grocery_guard_historical_market_fact();
+DROP FUNCTION IF EXISTS grocery_guard_historical_regional_fact();
+DROP FUNCTION IF EXISTS grocery_guard_historical_monthly_fact();
+DROP FUNCTION IF EXISTS grocery_guard_historical_collection_part();
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0021_bind_historical_source_endpoints")]
+
+    operations = [migrations.RunSQL(CREATE_GUARDS, DROP_GUARDS)]
diff --git a/grocery/tests/historical_bundle_factory.py b/grocery/tests/historical_bundle_factory.py
index b683d75..54d7fdb 100644
--- a/grocery/tests/historical_bundle_factory.py
+++ b/grocery/tests/historical_bundle_factory.py
@@ -19,7 +19,8 @@ from grocery.historical_identity_models import (
 )
 from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
 from grocery.historical_review_models import HistoricalCollectionReviewDecision
-from grocery.models import ParseRun, SourceArtifact, SourceConfiguration
+from grocery.models import ParseRun, SourceConfiguration
+from grocery.tests.historical_test_support import create_scoped_artifact
 from grocery.tests.test_acquisition_models import create_source_configuration
 from grocery.tests.test_price_series_key_models import create_series
 
@@ -57,13 +58,7 @@ def _collection_part(
         publication_mode=publication_mode,
     )
     digest = hashlib.sha256(parser_revision.encode("ascii")).hexdigest()
-    artifact = SourceArtifact.objects.create(
-        source_identity=f"fixture:{dataset_id}:{parser_revision}",
-        ordered_manifest_sha256=digest,
-        page_count=1,
-        total_bytes=1,
-        first_seen_at=now,
-    )
+    artifact = create_scoped_artifact(source, digest, row_count=fact_count)
     parse_run = ParseRun.objects.create(
         artifact=artifact,
         parser_revision=parser_revision,
diff --git a/grocery/tests/historical_test_support.py b/grocery/tests/historical_test_support.py
new file mode 100644
index 0000000..0c40657
--- /dev/null
+++ b/grocery/tests/historical_test_support.py
@@ -0,0 +1,28 @@
+from django.utils import timezone
+
+from grocery.models import FetchAttempt, SourceArtifact, SourceConfiguration, build_source_artifact
+from grocery.tests.test_acquisition_models import create_fetch_attempt, create_page_receipt
+
+
+def create_scoped_artifact(
+    source: SourceConfiguration,
+    scope_sha256: str,
+    *,
+    row_count: int = 1,
+) -> SourceArtifact:
+    attempt = create_fetch_attempt(source, request_scope_sha256=scope_sha256)
+    create_page_receipt(
+        attempt,
+        declared_total_count=row_count,
+        received_row_count=row_count,
+        body_byte_length=10,
+        body_sha256=scope_sha256,
+    )
+    attempt.state = FetchAttempt.State.SUCCEEDED
+    attempt.completed_at = timezone.now()
+    attempt.received_page_count = 1
+    attempt.received_row_count = row_count
+    attempt.received_byte_count = 10
+    attempt.save()
+    artifact, _created = build_source_artifact(attempt.id)
+    return artifact
diff --git a/grocery/tests/test_historical_collections.py b/grocery/tests/test_historical_collections.py
index 85c4a57..42a9c69 100644
--- a/grocery/tests/test_historical_collections.py
+++ b/grocery/tests/test_historical_collections.py
@@ -7,14 +7,17 @@ from grocery.historical_collection_models import (
     HistoricalSourceCollectionPart,
 )
 from grocery.models import ParseRun, SourceConfiguration
+from grocery.tests.historical_test_support import create_scoped_artifact
 from grocery.tests.test_acquisition_models import create_source_configuration
-from grocery.tests.test_artifact_parse_models import create_artifact
 
 
-def _validated_parse_run() -> ParseRun:
+def _validated_parse_run(
+    source: SourceConfiguration,
+    scope_sha256: str,
+) -> ParseRun:
     completed_at = timezone.now()
     return ParseRun.objects.create(
-        artifact=create_artifact(),
+        artifact=create_scoped_artifact(source, scope_sha256),
         parser_revision="historical-monthly-v1",
         configuration_hash="c" * 64,
         result_hash="d" * 64,
@@ -40,11 +43,12 @@ def test_collection_part_is_complete_then_terminally_immutable(db: None) -> None
         month_min="202301",
         month_max="202512",
     )
+    scope = "e" * 64
     part = HistoricalSourceCollectionPart.objects.create(
         collection=collection,
         ordinal=1,
-        partition_scope_sha256="e" * 64,
-        parse_run=_validated_parse_run(),
+        partition_scope_sha256=scope,
+        parse_run=_validated_parse_run(source, scope),
         fact_count=1,
     )
 
diff --git a/grocery/tests/test_historical_daily_facts.py b/grocery/tests/test_historical_daily_facts.py
index 59ce7af..b405227 100644
--- a/grocery/tests/test_historical_daily_facts.py
+++ b/grocery/tests/test_historical_daily_facts.py
@@ -18,24 +18,13 @@ from grocery.historical_identity_models import (
     price_series_identity_sha256,
 )
 from grocery.models import ParseRun, SourceConfiguration
+from grocery.tests.historical_test_support import create_scoped_artifact
 from grocery.tests.test_acquisition_models import create_source_configuration
-from grocery.tests.test_artifact_parse_models import create_artifact
 from grocery.tests.test_price_series_key_models import create_series
 
 
 def _part(kind: str, parser_revision: str) -> HistoricalSourceCollectionPart:
     completed_at = timezone.now()
-    parse_run = ParseRun.objects.create(
-        artifact=create_artifact(),
-        parser_revision=parser_revision,
-        configuration_hash=hashlib.sha256(f"config:{parser_revision}".encode()).hexdigest(),
-        result_hash=hashlib.sha256(f"result:{parser_revision}".encode()).hexdigest(),
-        status=ParseRun.Status.VALIDATED,
-        started_at=completed_at,
-        completed_at=completed_at,
-        total_row_count=1,
-        accepted_row_count=1,
-    )
     source = create_source_configuration(
         dataset_id=(
             "15156062" if kind == HistoricalSourceCollection.Kind.REGIONAL_DAILY else "15156065"
@@ -46,6 +35,18 @@ def _part(kind: str, parser_revision: str) -> HistoricalSourceCollectionPart:
             else SourceConfiguration.PublicationMode.HISTORICAL_MARKET
         ),
     )
+    scope = hashlib.sha256(parser_revision.encode()).hexdigest()
+    parse_run = ParseRun.objects.create(
+        artifact=create_scoped_artifact(source, scope),
+        parser_revision=parser_revision,
+        configuration_hash=hashlib.sha256(f"config:{parser_revision}".encode()).hexdigest(),
+        result_hash=hashlib.sha256(f"result:{parser_revision}".encode()).hexdigest(),
+        status=ParseRun.Status.VALIDATED,
+        started_at=completed_at,
+        completed_at=completed_at,
+        total_row_count=1,
+        accepted_row_count=1,
+    )
     collection = HistoricalSourceCollection.objects.create(
         kind=kind,
         source_configuration=source,
@@ -58,7 +59,7 @@ def _part(kind: str, parser_revision: str) -> HistoricalSourceCollectionPart:
     return HistoricalSourceCollectionPart.objects.create(
         collection=collection,
         ordinal=1,
-        partition_scope_sha256=hashlib.sha256(parser_revision.encode()).hexdigest(),
+        partition_scope_sha256=scope,
         parse_run=parse_run,
         fact_count=1,
     )
diff --git a/grocery/tests/test_historical_integrity_guards.py b/grocery/tests/test_historical_integrity_guards.py
new file mode 100644
index 0000000..9dce502
--- /dev/null
+++ b/grocery/tests/test_historical_integrity_guards.py
@@ -0,0 +1,142 @@
+from decimal import Decimal
+
+import pytest
+from django.db import DatabaseError, transaction
+from django.utils import timezone
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailRegionKey,
+    price_series_identity_sha256,
+)
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.models import ParseRun, SourceConfiguration
+from grocery.tests.historical_test_support import create_scoped_artifact
+from grocery.tests.test_acquisition_models import create_source_configuration
+from grocery.tests.test_price_series_key_models import create_series
+
+
+def _monthly_collection(source: SourceConfiguration, scope: str) -> HistoricalSourceCollection:
+    return HistoricalSourceCollection.objects.create(
+        kind=HistoricalSourceCollection.Kind.MONTHLY,
+        source_configuration=source,
+        code_manifest_sha256="a" * 64,
+        partition_manifest_sha256=scope,
+        expected_part_count=1,
+        month_min="202501",
+        month_max="202512",
+    )
+
+
+def _parse(source: SourceConfiguration, scope: str, suffix: str) -> ParseRun:
+    now = timezone.now()
+    return ParseRun.objects.create(
+        artifact=create_scoped_artifact(source, scope),
+        parser_revision=f"monthly-{suffix}",
+        configuration_hash=suffix * 64,
+        result_hash=scope,
+        status=ParseRun.Status.VALIDATED,
+        started_at=now,
+        completed_at=now,
+        total_row_count=1,
+        accepted_row_count=1,
+    )
+
+
+def test_database_binds_part_scope_and_fact_membership_then_freezes_rows(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    first_scope = "b" * 64
+    first_collection = _monthly_collection(source, first_scope)
+    first_parse = _parse(source, first_scope, "c")
+
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalSourceCollectionPart.objects.bulk_create(
+            [
+                HistoricalSourceCollectionPart(
+                    collection=first_collection,
+                    ordinal=1,
+                    partition_scope_sha256="d" * 64,
+                    parse_run=first_parse,
+                    fact_count=1,
+                )
+            ]
+        )
+
+    first_part = HistoricalSourceCollectionPart.objects.create(
+        collection=first_collection,
+        ordinal=1,
+        partition_scope_sha256=first_scope,
+        parse_run=first_parse,
+        fact_count=1,
+    )
+    second_scope = "e" * 64
+    second_collection = _monthly_collection(source, second_scope)
+    second_part = HistoricalSourceCollectionPart.objects.create(
+        collection=second_collection,
+        ordinal=1,
+        partition_scope_sha256=second_scope,
+        parse_run=_parse(source, second_scope, "f"),
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
+        region_code="1101",
+        region_name="서울",
+        identity_evidence_revision="codes-v1",
+    )
+
+    with pytest.raises(DatabaseError), transaction.atomic():
+        MonthlyRegionalRetailPrice.objects.bulk_create(
+            [
+                MonthlyRegionalRetailPrice(
+                    collection=first_collection,
+                    collection_part=second_part,
+                    series=series,
+                    region=region,
+                    year_month="202512",
+                    provider_mean=Decimal("1200"),
+                    provider_low=Decimal("1000"),
+                    provider_high=Decimal("1500"),
+                    source_row_sha256="1" * 64,
+                    source_contract_revision="15156060-v1",
+                )
+            ]
+        )
+
+    fact = MonthlyRegionalRetailPrice.objects.create(
+        collection=first_collection,
+        collection_part=first_part,
+        series=series,
+        region=region,
+        year_month="202512",
+        provider_mean=Decimal("1200"),
+        provider_low=Decimal("1000"),
+        provider_high=Decimal("1500"),
+        source_row_sha256="2" * 64,
+        source_contract_revision="15156060-v1",
+    )
+    first_collection.state = HistoricalSourceCollection.State.VALIDATED
+    first_collection.accepted_row_count = 1
+    first_collection.result_sha256 = "3" * 64
+    first_collection.completed_at = timezone.now()
+    first_collection.save()
+
+    with pytest.raises(DatabaseError), transaction.atomic():
+        MonthlyRegionalRetailPrice.objects.filter(pk=fact.pk).update(
+            provider_mean=Decimal("1300")
+        )
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalSourceCollectionPart.objects.filter(pk=first_part.pk).delete()
diff --git a/grocery/tests/test_historical_monthly_facts.py b/grocery/tests/test_historical_monthly_facts.py
index d5747da..7d25d38 100644
--- a/grocery/tests/test_historical_monthly_facts.py
+++ b/grocery/tests/test_historical_monthly_facts.py
@@ -33,11 +33,12 @@ def test_monthly_fact_preserves_provider_range_and_is_immutable(db: None) -> Non
         month_min="202512",
         month_max="202512",
     )
+    scope = "c" * 64
     part = HistoricalSourceCollectionPart.objects.create(
         collection=collection,
         ordinal=1,
-        partition_scope_sha256="c" * 64,
-        parse_run=_validated_parse_run(),
+        partition_scope_sha256=scope,
+        parse_run=_validated_parse_run(source, scope),
         fact_count=1,
     )
     recent = create_series()


