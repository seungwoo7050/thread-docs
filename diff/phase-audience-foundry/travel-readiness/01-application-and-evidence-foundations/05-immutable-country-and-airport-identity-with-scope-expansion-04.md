## `fix(warnings): preserve Hongkong source identity`

diff --git a/travel_warnings/migrations/0005_hongkong_source_identity.py b/travel_warnings/migrations/0005_hongkong_source_identity.py
new file mode 100644
index 0000000..c297bff
--- /dev/null
+++ b/travel_warnings/migrations/0005_hongkong_source_identity.py
@@ -0,0 +1,116 @@
+from django.db import migrations
+
+
+FUNCTION_REPLACEMENTS = {
+    "travel_warnings_guard_revision_change": (
+        """        ',"country_name_en":' || to_jsonb((
+            SELECT name_en FROM countries_country
+             WHERE id = NEW.country_id
+        ))::text ||""",
+        """        ',"country_name_en":' || to_jsonb((
+            SELECT CASE
+                WHEN iso_alpha2 = 'HK' THEN 'Hongkong'
+                ELSE name_en
+            END
+              FROM countries_country
+             WHERE id = NEW.country_id
+        ))::text ||""",
+    ),
+    "travel_warnings_guard_fact_change": (
+        """           source.code, country.iso_alpha2, country.name_en, country.name_ko""",
+        """           source.code, country.iso_alpha2,
+           CASE WHEN country.iso_alpha2 = 'HK'
+                THEN 'Hongkong' ELSE country.name_en END,
+           country.name_ko""",
+    ),
+    "travel_warnings_validate_snapshot_closure": (
+        """           country.iso_alpha2, country.name_en, country.name_ko""",
+        """           country.iso_alpha2,
+           CASE WHEN country.iso_alpha2 = 'HK'
+                THEN 'Hongkong' ELSE country.name_en END,
+           country.name_ko""",
+    ),
+}
+
+
+def _rewrite_functions(schema_editor, *, reverse=False):
+    with schema_editor.connection.cursor() as cursor:
+        if reverse:
+            cursor.execute(
+                """
+                SELECT EXISTS (
+                    SELECT 1
+                      FROM travel_warnings_travelwarningrevision AS revision
+                      JOIN countries_country AS country
+                        ON country.id = revision.country_id
+                     WHERE country.iso_alpha2 = 'HK'
+                )
+                """
+            )
+            if cursor.fetchone()[0]:
+                raise RuntimeError(
+                    "Hongkong source identity rollback requires no HK warning "
+                    "revisions"
+                )
+
+        for function_name, (forward_old, forward_new) in (
+            FUNCTION_REPLACEMENTS.items()
+        ):
+            cursor.execute(
+                """
+                SELECT procedure.prosrc
+                  FROM pg_proc AS procedure
+                  JOIN pg_namespace AS namespace
+                    ON namespace.oid = procedure.pronamespace
+                 WHERE namespace.nspname = current_schema()
+                   AND procedure.proname = %s
+                   AND procedure.pronargs = 0
+                """,
+                [function_name],
+            )
+            rows = cursor.fetchall()
+            if len(rows) != 1:
+                raise RuntimeError(
+                    f"expected one trigger function named {function_name}"
+                )
+            body = rows[0][0]
+            old, new = (
+                (forward_new, forward_old)
+                if reverse
+                else (forward_old, forward_new)
+            )
+            if body.count(old) != 1:
+                raise RuntimeError(
+                    f"unexpected {function_name} trigger definition"
+                )
+            body = body.replace(old, new)
+            quoted_name = schema_editor.quote_name(function_name)
+            cursor.execute(
+                f"""
+                CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+                LANGUAGE plpgsql AS $warning_hongkong_identity$
+                {body}
+                $warning_hongkong_identity$
+                """
+            )
+
+
+def apply_hongkong_source_identity(apps, schema_editor):
+    _rewrite_functions(schema_editor)
+
+
+def restore_country_canonical_identity(apps, schema_editor):
+    _rewrite_functions(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("travel_warnings", "0004_warning_snapshot_facts"),
+    ]
+
+    operations = [
+        migrations.RunPython(
+            apply_hongkong_source_identity,
+            restore_country_canonical_identity,
+        ),
+    ]
diff --git a/travel_warnings/tests/test_snapshot_models.py b/travel_warnings/tests/test_snapshot_models.py
index 76874c5..b302d4d 100644
--- a/travel_warnings/tests/test_snapshot_models.py
+++ b/travel_warnings/tests/test_snapshot_models.py
@@ -30,11 +30,15 @@ def _fingerprint(value):
     return hashlib.sha256(canonical).hexdigest()
 
 
+def _source_name_en(country):
+    return "Hongkong" if country.iso_alpha2 == "HK" else country.name_en
+
+
 def _fact_fingerprint(country, values):
     return _fingerprint(
         {
             "country_iso2": country.iso_alpha2,
-            "country_name_en": country.name_en,
+            "country_name_en": _source_name_en(country),
             "country_name_ko": country.name_ko,
             "source_alarm_level_code": values["source_alarm_level_code"],
             "source_scope_text": values["source_scope_text"],
@@ -52,7 +56,7 @@ def _snapshot_fingerprint(country, facts):
     return _fingerprint(
         {
             "country_iso2": country.iso_alpha2,
-            "country_name_en": country.name_en,
+            "country_name_en": _source_name_en(country),
             "country_name_ko": country.name_ko,
             "facts": [
                 {
@@ -185,6 +189,49 @@ class TravelWarningSnapshotFixtureMixin:
             **values,
         )
 
+    def test_hongkong_provider_identity_is_preserved_in_snapshot_hashes(self):
+        hong_kong = Country.objects.get(iso_alpha2="HK")
+        source = self.run.artifact.source
+        rights = source.rights_decisions.get()
+        started_at = self.run.completed_at + timedelta(seconds=1)
+        observation = self._make_success_attempt(
+            source=source,
+            rights=rights,
+            country_iso2="HK",
+            body_sha256=self.run.artifact.body_sha256,
+            started_at=started_at,
+            completed_at=started_at + timedelta(seconds=1),
+        )
+        fact_values = {
+            "source_alarm_level_code": "1",
+            "source_scope_type": "전역",
+            "source_scope_text": "홍콩 공식 제공 항목",
+            "source_written_date": None,
+        }
+
+        with transaction.atomic():
+            root = TravelWarningRevision.objects.create(
+                **self._root_values(
+                    (fact_values,),
+                    country=hong_kong,
+                    attempt=observation,
+                )
+            )
+            TravelWarningFact.objects.create(
+                revision=root,
+                source_position=1,
+                typed_fingerprint_sha256=_fact_fingerprint(
+                    hong_kong, fact_values
+                ),
+                **fact_values,
+            )
+
+        self.assertEqual(root.source_item_count, 1)
+        self.assertEqual(
+            root.facts.get().source_scope_text,
+            fact_values["source_scope_text"],
+        )
+
 
 class TravelWarningSnapshotPersistenceTests(
     TravelWarningSnapshotFixtureMixin,
