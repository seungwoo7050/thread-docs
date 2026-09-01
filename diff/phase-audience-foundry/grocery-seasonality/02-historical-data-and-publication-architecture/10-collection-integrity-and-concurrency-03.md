## `fix(history): serialize collection completion`

diff --git a/grocery/migrations/0023_serialize_historical_collection_writes.py b/grocery/migrations/0023_serialize_historical_collection_writes.py
new file mode 100644
index 0000000..7679dec
--- /dev/null
+++ b/grocery/migrations/0023_serialize_historical_collection_writes.py
@@ -0,0 +1,145 @@
+from django.db import migrations
+
+LOCK_GUARDS = r"""
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
+     FOR SHARE;
+    SELECT artifact_id, status
+      INTO parse_artifact, parse_status
+      FROM grocery_parserun
+     WHERE id = NEW.parse_run_id
+     FOR SHARE;
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
+     FOR SHARE;
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
+     FOR SHARE;
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
+     FOR SHARE;
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
+"""
+
+
+REVERSE_LOCK_GUARDS = LOCK_GUARDS.replace("FOR SHARE;", "FOR KEY SHARE;")
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0022_guard_historical_collection_membership")]
+
+    operations = [migrations.RunSQL(LOCK_GUARDS, REVERSE_LOCK_GUARDS)]
diff --git a/grocery/tests/test_historical_integrity_guards.py b/grocery/tests/test_historical_integrity_guards.py
index 9dce502..c878d75 100644
--- a/grocery/tests/test_historical_integrity_guards.py
+++ b/grocery/tests/test_historical_integrity_guards.py
@@ -1,7 +1,9 @@
+import uuid
 from decimal import Decimal
 
+import psycopg
 import pytest
-from django.db import DatabaseError, transaction
+from django.db import DatabaseError, connection, transaction
 from django.utils import timezone
 
 from grocery.historical_collection_models import (
@@ -140,3 +142,35 @@ def test_database_binds_part_scope_and_fact_membership_then_freezes_rows(db: Non
         )
     with pytest.raises(DatabaseError), transaction.atomic():
         HistoricalSourceCollectionPart.objects.filter(pk=first_part.pk).delete()
+
+
+def test_collection_completion_serializes_against_part_insert(transactional_db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    scope = "9" * 64
+    collection = _monthly_collection(source, scope)
+    parse_run = _parse(source, scope, "8")
+    connection_params = connection.get_connection_params()
+
+    with (
+        psycopg.connect(**connection_params) as completing,
+        psycopg.connect(**connection_params) as appending,
+    ):
+        completing.execute(
+            "UPDATE grocery_historicalsourcecollection "
+            "SET state = 'VALIDATED', completed_at = now(), result_sha256 = %s "
+            "WHERE id = %s",
+            ("7" * 64, collection.id),
+        )
+        appending.execute("SET LOCAL lock_timeout = '100ms'")
+        with pytest.raises(psycopg.errors.LockNotAvailable):
+            appending.execute(
+                "INSERT INTO grocery_historicalsourcecollectionpart "
+                "(id, ordinal, partition_scope_sha256, fact_count, collection_id, parse_run_id) "
+                "VALUES (%s, %s, %s, %s, %s, %s)",
+                (uuid.uuid4(), 1, scope, 1, collection.id, parse_run.id),
+            )
+        appending.rollback()
+        completing.rollback()
