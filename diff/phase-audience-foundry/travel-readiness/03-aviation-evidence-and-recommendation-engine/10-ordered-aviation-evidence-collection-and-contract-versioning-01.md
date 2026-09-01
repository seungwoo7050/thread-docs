# 순서 보존 항공 증거 수집과 계약 버전 관리

## `feat(source): publish scheduled flight evidence`

diff --git a/countries/migrations/0003_supported_country_allowlist.py b/countries/migrations/0003_supported_country_allowlist.py
new file mode 100644
index 0000000..e375701
--- /dev/null
+++ b/countries/migrations/0003_supported_country_allowlist.py
@@ -0,0 +1,47 @@
+import uuid
+
+from django.db import migrations, models
+from django.db.models import Q
+
+
+SUPPORTED_COUNTRIES = (
+    (uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9"), "JP", "일본", "Japan", True),
+    (uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"), "TW", "대만", "Taiwan", True),
+    (uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"), "HK", "홍콩", "Hong Kong", True),
+    (uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"), "MO", "마카오", "Macau", True),
+    (uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"), "VN", "베트남", "Vietnam", True),
+    (uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"), "TH", "태국", "Thailand", True),
+)
+
+
+def _condition():
+    result = Q(
+        id=SUPPORTED_COUNTRIES[0][0],
+        iso_alpha2="JP",
+        name_ko="일본",
+        name_en="Japan",
+        is_public=True,
+    )
+    for country_id, iso_alpha2, name_ko, name_en, is_public in SUPPORTED_COUNTRIES[1:]:
+        result |= Q(
+            id=country_id,
+            iso_alpha2=iso_alpha2,
+            name_ko=name_ko,
+            name_en=name_en,
+            is_public=is_public,
+        )
+    return result
+
+
+class Migration(migrations.Migration):
+    dependencies = [("countries", "0002_generalize_country_identity")]
+
+    operations = [
+        migrations.AddConstraint(
+            model_name="country",
+            constraint=models.CheckConstraint(
+                condition=_condition(),
+                name="country_supported_exact_allowlist",
+            ),
+        ),
+    ]
diff --git a/countries/models.py b/countries/models.py
index 37facac..6ffa2bb 100644
--- a/countries/models.py
+++ b/countries/models.py
@@ -13,6 +13,27 @@ SUPPORTED_COUNTRY_IDS = {
     "VN": uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"),
     "TH": uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"),
 }
+SUPPORTED_COUNTRY_ROWS = (
+    (SUPPORTED_COUNTRY_IDS["JP"], "JP", "일본", "Japan", True),
+    (SUPPORTED_COUNTRY_IDS["TW"], "TW", "대만", "Taiwan", True),
+    (SUPPORTED_COUNTRY_IDS["HK"], "HK", "홍콩", "Hong Kong", True),
+    (SUPPORTED_COUNTRY_IDS["MO"], "MO", "마카오", "Macau", True),
+    (SUPPORTED_COUNTRY_IDS["VN"], "VN", "베트남", "Vietnam", True),
+    (SUPPORTED_COUNTRY_IDS["TH"], "TH", "태국", "Thailand", True),
+)
+
+
+def _supported_country_condition() -> Q:
+    condition = Q(id=SUPPORTED_COUNTRY_ROWS[0][0], iso_alpha2="JP", name_ko="일본", name_en="Japan", is_public=True)
+    for country_id, iso_alpha2, name_ko, name_en, is_public in SUPPORTED_COUNTRY_ROWS[1:]:
+        condition |= Q(
+            id=country_id,
+            iso_alpha2=iso_alpha2,
+            name_ko=name_ko,
+            name_en=name_en,
+            is_public=is_public,
+        )
+    return condition
 
 
 class Country(models.Model):
@@ -30,6 +51,10 @@ class Country(models.Model):
             ),
             models.CheckConstraint(condition=~Q(name_ko=""), name="country_name_ko_present"),
             models.CheckConstraint(condition=~Q(name_en=""), name="country_name_en_present"),
+            models.CheckConstraint(
+                condition=_supported_country_condition(),
+                name="country_supported_exact_allowlist",
+            ),
         ]
 
     def __str__(self) -> str:
diff --git a/operations/tests/migration_helpers.py b/operations/tests/migration_helpers.py
index 7f6fca6..77bf502 100644
--- a/operations/tests/migration_helpers.py
+++ b/operations/tests/migration_helpers.py
@@ -26,6 +26,9 @@ def restore_fixed_seed_state(connection) -> None:
         apps,
         schema_editor,
     )
+    import_module(
+        "countries.migrations.0002_generalize_country_identity"
+    ).seed_supported_countries(apps, schema_editor)
     import_module(
         "entry_requirements.migrations.0001_initial"
     ).seed_passport_policy(
@@ -36,6 +39,12 @@ def restore_fixed_seed_state(connection) -> None:
         apps,
         schema_editor,
     )
+    import_module(
+        "travel_windows.migrations.0002_scheduled_flight_evidence"
+    ).seed_pointer(apps, schema_editor)
+    import_module(
+        "travel_windows.migrations.0003_curated_airports"
+    ).seed_airports(apps, schema_editor)
 
 
 def restore_canonical_migration_graph(connection) -> None:
@@ -60,6 +69,8 @@ def _restore_test_seeds_after_migrate(sender, *, using, **_kwargs) -> None:
         "reviews_publicationrevision",
         "reviews_publishedentryfacts",
         "reviews_publishedtravelwarning",
+        "travel_windows_airport",
+        "travel_windows_publishedflightschedule",
     }
     if not required_tables.issubset(connection.introspection.table_names()):
         # A migration-boundary test can intentionally flush a partial graph.
diff --git a/reviews/migrations/0002_country_scoped_publications.py b/reviews/migrations/0002_country_scoped_publications.py
new file mode 100644
index 0000000..512158e
--- /dev/null
+++ b/reviews/migrations/0002_country_scoped_publications.py
@@ -0,0 +1,269 @@
+import uuid
+
+from django.db import migrations, models
+
+
+PASSPORT_POLICY_ID = uuid.UUID("f461f7a7-18f7-5b0d-9831-bfbd47b695e5")
+
+
+FUNCTION_REPLACEMENTS = {
+    "reviews_guard_publication_revision": (
+        (
+            """         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = typed_country_id
+           AND passport_policy_id = typed_policy_id
+         FOR UPDATE;""",
+        ),
+        (
+            """         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = typed_country_id
+         FOR UPDATE;""",
+        ),
+        (
+            """       OR typed_country_id IS DISTINCT FROM
+          '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'""",
+            """       OR NEW.scope_country_id IS DISTINCT FROM typed_country_id
+       OR NEW.scope_passport_policy_id IS DISTINCT FROM typed_policy_id
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'""",
+        ),
+    ),
+    "reviews_guard_entry_pointer": (
+        (
+            """        IF NEW.id <> '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+           OR NEW.current_publication_id IS NOT NULL OR NEW.version <> 0 THEN
+            RAISE EXCEPTION 'entry pointer insert is not the fixed seed'""",
+            """        IF NEW.current_publication_id IS NOT NULL OR NEW.version <> 0
+           OR NEW.passport_policy_id IS DISTINCT FROM
+              'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid THEN
+            RAISE EXCEPTION 'entry pointer insert has an invalid empty scope'""",
+        ),
+    ),
+    "reviews_guard_warning_pointer": (
+        (
+            """        IF NEW.id <> 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+           OR NEW.current_publication_id IS NOT NULL OR NEW.version <> 0 THEN
+            RAISE EXCEPTION 'warning pointer insert is not the fixed seed'""",
+            """        IF NEW.current_publication_id IS NOT NULL OR NEW.version <> 0 THEN
+            RAISE EXCEPTION 'warning pointer insert has an invalid empty scope'""",
+        ),
+    ),
+    "reviews_prelock_review_insert": (
+        (
+            """         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;""",
+            """         WHERE (country_id, passport_policy_id) = (
+            SELECT country_id, passport_policy_id
+              FROM entry_requirements_entryfactrevision
+             WHERE id = NEW.entry_fact_revision_id
+         )
+         FOR UPDATE;""",
+        ),
+        (
+            """         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = (
+            SELECT country_id FROM travel_warnings_travelwarningrevision
+             WHERE id = NEW.travel_warning_revision_id
+         )
+         FOR UPDATE;""",
+        ),
+    ),
+    "reviews_prelock_publication_insert": (
+        (
+            """         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = NEW.scope_country_id
+           AND passport_policy_id = NEW.scope_passport_policy_id
+         FOR UPDATE;""",
+        ),
+        (
+            """         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = NEW.scope_country_id
+         FOR UPDATE;""",
+        ),
+    ),
+    "reviews_prelock_audit_insert": (
+        (
+            """         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;""",
+            """         WHERE (country_id, passport_policy_id) = (
+            SELECT country_id, passport_policy_id
+              FROM entry_requirements_entryfactrevision
+             WHERE id = NEW.entry_fact_revision_id
+         )
+         FOR UPDATE;""",
+        ),
+        (
+            """         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;""",
+            """         WHERE country_id = (
+            SELECT country_id FROM travel_warnings_travelwarningrevision
+             WHERE id = NEW.travel_warning_revision_id
+         )
+         FOR UPDATE;""",
+        ),
+    ),
+    "reviews_enforce_deferred_closure": (
+        (
+            """             WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;""",
+            """             WHERE country_id = publication_country_id
+               AND passport_policy_id = publication_policy_id;""",
+        ),
+        (
+            """             WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;""",
+            """             WHERE country_id = publication_country_id;""",
+        ),
+        (
+            """          FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;""",
+            """          FROM reviews_publishedentryfacts
+         WHERE id = NEW.id;""",
+        ),
+        (
+            """          FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;""",
+            """          FROM reviews_publishedtravelwarning
+         WHERE id = NEW.id;""",
+        ),
+        (
+            """             WHERE pointer.id =
+                '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;""",
+            """             WHERE (pointer.country_id, pointer.passport_policy_id) = (
+                SELECT country_id, passport_policy_id
+                  FROM entry_requirements_entryfactrevision
+                 WHERE id = review_entry_id
+             );""",
+        ),
+        (
+            """             WHERE pointer.id =
+                'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;""",
+            """             WHERE pointer.country_id = (
+                SELECT country_id FROM travel_warnings_travelwarningrevision
+                 WHERE id = review_warning_id
+             );""",
+        ),
+    ),
+}
+
+
+def _rewrite_functions(schema_editor, *, reverse=False):
+    with schema_editor.connection.cursor() as cursor:
+        for function_name, replacements in FUNCTION_REPLACEMENTS.items():
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
+            for forward_old, forward_new in replacements:
+                old, new = (
+                    (forward_new, forward_old)
+                    if reverse
+                    else (forward_old, forward_new)
+                )
+                if body.count(old) != 1:
+                    raise RuntimeError(
+                        f"unexpected {function_name} trigger definition"
+                    )
+                body = body.replace(old, new)
+            quoted_name = schema_editor.quote_name(function_name)
+            cursor.execute(
+                f"""
+                CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+                LANGUAGE plpgsql AS $reviews_country_scope$
+                {body}
+                $reviews_country_scope$
+                """
+            )
+
+
+def generalize_guard_functions(apps, schema_editor):
+    _rewrite_functions(schema_editor)
+
+
+def restore_guard_functions(apps, schema_editor):
+    _rewrite_functions(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("countries", "0002_generalize_country_identity"),
+        ("entry_requirements", "0001_initial"),
+        ("reviews", "0001_initial"),
+    ]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="publicationrevision",
+            name="publication_scope_country_jp",
+        ),
+        migrations.RemoveConstraint(
+            model_name="publishedentryfacts",
+            name="published_entry_singleton_scope",
+        ),
+        migrations.RemoveConstraint(
+            model_name="publishedtravelwarning",
+            name="published_warning_singleton_scope",
+        ),
+        migrations.AlterField(
+            model_name="publishedentryfacts",
+            name="id",
+            field=models.UUIDField(
+                default=uuid.uuid4,
+                editable=False,
+                primary_key=True,
+                serialize=False,
+            ),
+        ),
+        migrations.AlterField(
+            model_name="publishedtravelwarning",
+            name="id",
+            field=models.UUIDField(
+                default=uuid.uuid4,
+                editable=False,
+                primary_key=True,
+                serialize=False,
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="publishedentryfacts",
+            constraint=models.CheckConstraint(
+                condition=models.Q(passport_policy_id=PASSPORT_POLICY_ID),
+                name="published_entry_policy_exact",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="publishedentryfacts",
+            constraint=models.UniqueConstraint(
+                fields=("country", "passport_policy"),
+                name="published_entry_country_policy_unique",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="publishedtravelwarning",
+            constraint=models.UniqueConstraint(
+                fields=("country",),
+                name="published_warning_country_unique",
+            ),
+        ),
+        migrations.RunPython(
+            generalize_guard_functions,
+            restore_guard_functions,
+        ),
+    ]
diff --git a/reviews/models.py b/reviews/models.py
index 12e564c..c25c8d9 100644
--- a/reviews/models.py
+++ b/reviews/models.py
@@ -5,7 +5,6 @@ from django.db import models
 from django.db.models import F, Q
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID
 from entry_requirements.models import PASSPORT_POLICY_ID
 
 
@@ -232,10 +231,6 @@ class PublicationRevision(models.Model):
 
     class Meta:
         constraints = [
-            models.CheckConstraint(
-                condition=Q(scope_country_id=JP_COUNTRY_ID),
-                name="publication_scope_country_jp",
-            ),
             models.CheckConstraint(
                 condition=(
                     Q(
@@ -358,7 +353,7 @@ class PublicationRevision(models.Model):
 class PublishedEntryFacts(models.Model):
     id = models.UUIDField(
         primary_key=True,
-        default=ENTRY_POINTER_ID,
+        default=uuid.uuid4,
         editable=False,
     )
     country = models.ForeignKey("countries.Country", on_delete=models.PROTECT)
@@ -379,12 +374,12 @@ class PublishedEntryFacts(models.Model):
     class Meta:
         constraints = [
             models.CheckConstraint(
-                condition=Q(
-                    id=ENTRY_POINTER_ID,
-                    country_id=JP_COUNTRY_ID,
-                    passport_policy_id=PASSPORT_POLICY_ID,
-                ),
-                name="published_entry_singleton_scope",
+                condition=Q(passport_policy_id=PASSPORT_POLICY_ID),
+                name="published_entry_policy_exact",
+            ),
+            models.UniqueConstraint(
+                fields=("country", "passport_policy"),
+                name="published_entry_country_policy_unique",
             ),
             models.CheckConstraint(
                 condition=(
@@ -399,7 +394,7 @@ class PublishedEntryFacts(models.Model):
 class PublishedTravelWarning(models.Model):
     id = models.UUIDField(
         primary_key=True,
-        default=WARNING_POINTER_ID,
+        default=uuid.uuid4,
         editable=False,
     )
     country = models.ForeignKey("countries.Country", on_delete=models.PROTECT)
@@ -415,9 +410,9 @@ class PublishedTravelWarning(models.Model):
 
     class Meta:
         constraints = [
-            models.CheckConstraint(
-                condition=Q(id=WARNING_POINTER_ID, country_id=JP_COUNTRY_ID),
-                name="published_warning_singleton_scope",
+            models.UniqueConstraint(
+                fields=("country",),
+                name="published_warning_country_unique",
             ),
             models.CheckConstraint(
                 condition=(
diff --git a/reviews/publication.py b/reviews/publication.py
index bec2786..11508ce 100644
--- a/reviews/publication.py
+++ b/reviews/publication.py
@@ -13,7 +13,6 @@ from django.db import connection, transaction
 from django.db.models import F
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID
 from entry_requirements.ingestion import (
     ENTRY_ATTRIBUTION,
     ENTRY_CONTRACT_FINGERPRINT_SHA256,
@@ -27,10 +26,7 @@ from entry_requirements.ingestion import (
     ENTRY_SOURCE_OWNER,
     ENTRY_SOURCE_REVISION,
 )
-from entry_requirements.models import (
-    PASSPORT_POLICY_ID,
-    EntryFactRevision,
-)
+from entry_requirements.models import EntryFactRevision
 from entry_requirements.parser import ENTRY_SCHEMA_FINGERPRINT_SHA256
 from sources.models import (
     FetchAttempt,
@@ -479,17 +475,79 @@ def _create_review(
     )
 
 
-def _pointer(spec: _ModuleSpec):
+def _subject_pointer_scope(spec: _ModuleSpec, subject) -> dict[str, object]:
+    country_id = getattr(subject, "country_id", None)
+    if country_id is None:
+        raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+    scope: dict[str, object] = {"country_id": country_id}
+    if spec.module == PublicationModule.ENTRY:
+        passport_policy_id = getattr(subject, "passport_policy_id", None)
+        if passport_policy_id is None:
+            raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+        scope["passport_policy_id"] = passport_policy_id
+    return scope
+
+
+def _publication_pointer_scope(
+    spec: _ModuleSpec, publication: PublicationRevision
+) -> dict[str, object]:
+    scope: dict[str, object] = {
+        "country_id": publication.scope_country_id,
+    }
+    if spec.module == PublicationModule.ENTRY:
+        if publication.scope_passport_policy_id is None:
+            raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+        scope["passport_policy_id"] = publication.scope_passport_policy_id
+    return scope
+
+
+def _pointer(
+    spec: _ModuleSpec,
+    *,
+    scope: dict[str, object],
+    create: bool = True,
+):
     try:
-        return (
-            spec.pointer_model.objects.select_for_update(of=("self",))
-            .select_related("current_publication")
-            .get()
-        )
+        queryset = spec.pointer_model.objects.select_for_update(
+            of=("self",)
+        ).select_related("current_publication")
+        pointer = queryset.filter(**scope).first()
+        if pointer is not None or not create:
+            return pointer
+        created = spec.pointer_model.objects.create(**scope)
+        return queryset.get(pk=created.pk)
+    except _ClosedFailure:
+        raise
     except Exception:
         raise _ClosedFailure(PublicationCode.TRANSACTION_ABORTED) from None
 
 
+def _pointer_for_subject(
+    spec: _ModuleSpec,
+    subject,
+    *,
+    create: bool = True,
+):
+    return _pointer(
+        spec,
+        scope=_subject_pointer_scope(spec, subject),
+        create=create,
+    )
+
+
+def _pointer_for_publication(
+    spec: _ModuleSpec,
+    publication: PublicationRevision,
+    *,
+    create: bool = True,
+):
+    return _pointer(
+        spec,
+        scope=_publication_pointer_scope(spec, publication),
+        create=create,
+    )
+
+
 def _pointer_has_subject(pointer, spec: _ModuleSpec, subject) -> bool:
     current = pointer.current_publication
     return bool(
@@ -562,12 +620,12 @@ def _failure_audit(
     try:
         with transaction.atomic(durable=True):
             _lock_module(spec)
-            _pointer(spec)
             subject = None
             if subject_id is not None:
                 try:
                     subject = _load_subject(spec, subject_id)
                     _lock_subject_source(spec, subject)
+                    _pointer_for_subject(spec, subject, create=False)
                 except _ClosedFailure:
                     subject = None
             actor_value = _lock_audit_actor(actor)
@@ -668,10 +726,10 @@ def _publish_candidate_inner(
     try:
         with transaction.atomic(durable=True):
             _lock_module(spec)
-            pointer = _pointer(spec)
+            subject = _load_subject(spec, typed_revision_id)
+            pointer = _pointer_for_subject(spec, subject)
             if pointer.version != expected_pointer_version:
                 raise _ClosedFailure(PublicationCode.STALE_POINTER)
-            subject = _load_subject(spec, typed_revision_id)
             snapshots = _validate_source_chain(spec, subject)
             if _subject_was_published(spec, subject):
                 raise _ClosedFailure(PublicationCode.INVALID_TARGET)
@@ -695,9 +753,9 @@ def _publish_candidate_inner(
             now = timezone.now()
             publication = PublicationRevision.objects.create(
                 module=spec.module,
-                scope_country_id=JP_COUNTRY_ID,
+                scope_country_id=subject.country_id,
                 scope_passport_policy_id=(
-                    PASSPORT_POLICY_ID
+                    subject.passport_policy_id
                     if spec.module == PublicationModule.ENTRY
                     else None
                 ),
@@ -775,8 +833,8 @@ def _reject_candidate_inner(
     try:
         with transaction.atomic(durable=True):
             _lock_module(spec)
-            pointer = _pointer(spec)
             subject = _load_subject(spec, typed_revision_id)
+            pointer = _pointer_for_subject(spec, subject)
             _lock_subject_source(spec, subject)
             if _pointer_has_subject(pointer, spec, subject):
                 raise _ClosedFailure(PublicationCode.INVALID_TARGET)
@@ -848,12 +906,6 @@ def _rollback_publication_inner(
     try:
         with transaction.atomic(durable=True):
             _lock_module(spec)
-            pointer = _pointer(spec)
-            if pointer.version != expected_pointer_version:
-                raise _ClosedFailure(PublicationCode.STALE_POINTER)
-            current = pointer.current_publication
-            if current is None:
-                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
             try:
                 target = PublicationRevision.objects.select_for_update().get(
                     pk=target_publication_id,
@@ -861,6 +913,14 @@ def _rollback_publication_inner(
                 )
             except Exception:
                 raise _ClosedFailure(PublicationCode.INVALID_TARGET) from None
+            pointer = _pointer_for_publication(spec, target, create=False)
+            if pointer is None:
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+            if pointer.version != expected_pointer_version:
+                raise _ClosedFailure(PublicationCode.STALE_POINTER)
+            current = pointer.current_publication
+            if current is None:
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
             if (
                 target.id == current.id
                 or target.generation >= current.generation
@@ -891,9 +951,9 @@ def _rollback_publication_inner(
             now = timezone.now()
             publication = PublicationRevision.objects.create(
                 module=spec.module,
-                scope_country_id=JP_COUNTRY_ID,
+                scope_country_id=subject.country_id,
                 scope_passport_policy_id=(
-                    PASSPORT_POLICY_ID
+                    subject.passport_policy_id
                     if spec.module == PublicationModule.ENTRY
                     else None
                 ),
diff --git a/reviews/tests/test_country_scoped_publications.py b/reviews/tests/test_country_scoped_publications.py
new file mode 100644
index 0000000..cf3849f
--- /dev/null
+++ b/reviews/tests/test_country_scoped_publications.py
@@ -0,0 +1,130 @@
+from types import SimpleNamespace
+
+from django.db import IntegrityError, connection, transaction
+from django.test import TransactionTestCase
+
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS, Country
+from entry_requirements.models import (
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    PassportPolicy,
+)
+from reviews.models import (
+    ENTRY_POINTER_ID,
+    WARNING_POINTER_ID,
+    PublicationModule,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+)
+from reviews.publication import _SPECS, _pointer_for_subject
+
+
+class CountryScopedPublicationPointerTests(TransactionTestCase):
+    def setUp(self):
+        self.jp, _ = Country.objects.get_or_create(
+            id=JP_COUNTRY_ID,
+            defaults={
+                "iso_alpha2": "JP",
+                "name_ko": "일본",
+                "name_en": "Japan",
+                "is_public": True,
+            },
+        )
+        self.policy, _ = PassportPolicy.objects.get_or_create(
+            id=PASSPORT_POLICY_ID,
+            defaults={
+                "code": PASSPORT_POLICY_CODE,
+                "revision": PASSPORT_POLICY_REVISION,
+            },
+        )
+        PublishedEntryFacts.objects.get_or_create(
+            id=ENTRY_POINTER_ID,
+            defaults={
+                "country": self.jp,
+                "passport_policy": self.policy,
+            },
+        )
+        PublishedTravelWarning.objects.get_or_create(
+            id=WARNING_POINTER_ID,
+            defaults={"country": self.jp},
+        )
+        self.tw, _ = Country.objects.get_or_create(
+            id=SUPPORTED_COUNTRY_IDS["TW"],
+            defaults={
+                "iso_alpha2": "TW",
+                "name_ko": "대만",
+                "name_en": "Taiwan",
+                "is_public": True,
+            },
+        )
+
+    def test_japan_seed_identities_are_preserved(self):
+        entry = PublishedEntryFacts.objects.get(country=self.jp)
+        warning = PublishedTravelWarning.objects.get(country=self.jp)
+
+        self.assertEqual(entry.id, ENTRY_POINTER_ID)
+        self.assertEqual(warning.id, WARNING_POINTER_ID)
+        self.assertEqual(entry.passport_policy_id, PASSPORT_POLICY_ID)
+
+    def test_pointer_service_creates_and_resolves_each_country_scope(self):
+        with transaction.atomic():
+            entry = _pointer_for_subject(
+                _SPECS[PublicationModule.ENTRY],
+                SimpleNamespace(
+                    country_id=self.tw.id,
+                    passport_policy_id=self.policy.id,
+                ),
+            )
+            warning = _pointer_for_subject(
+                _SPECS[PublicationModule.TRAVEL_WARNING],
+                SimpleNamespace(country_id=self.tw.id),
+            )
+
+        self.assertEqual(entry.country_id, self.tw.id)
+        self.assertEqual(entry.passport_policy_id, self.policy.id)
+        self.assertNotEqual(entry.id, ENTRY_POINTER_ID)
+        self.assertEqual(warning.country_id, self.tw.id)
+        self.assertNotEqual(warning.id, WARNING_POINTER_ID)
+        self.assertEqual(entry.version, 0)
+        self.assertEqual(warning.version, 0)
+        self.assertIsNone(entry.current_publication_id)
+        self.assertIsNone(warning.current_publication_id)
+
+        with transaction.atomic():
+            resolved = _pointer_for_subject(
+                _SPECS[PublicationModule.TRAVEL_WARNING],
+                SimpleNamespace(country_id=self.tw.id),
+                create=False,
+            )
+        self.assertEqual(resolved.id, warning.id)
+
+    def test_country_scope_is_unique_but_different_countries_can_coexist(self):
+        PublishedEntryFacts.objects.create(
+            country=self.tw,
+            passport_policy=self.policy,
+        )
+        PublishedTravelWarning.objects.create(country=self.tw)
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PublishedEntryFacts.objects.create(
+                country=self.tw,
+                passport_policy=self.policy,
+            )
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PublishedTravelWarning.objects.create(country=self.tw)
+
+        self.assertEqual(PublishedEntryFacts.objects.count(), 2)
+        self.assertEqual(PublishedTravelWarning.objects.count(), 2)
+
+    def test_publication_schema_has_no_japan_scope_constraint(self):
+        with connection.cursor() as cursor:
+            cursor.execute(
+                """
+                SELECT count(*)
+                  FROM pg_constraint
+                 WHERE conrelid = 'reviews_publicationrevision'::regclass
+                   AND conname = 'publication_scope_country_jp'
+                """
+            )
+            self.assertEqual(cursor.fetchone()[0], 0)
diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index 097cead..54a32cc 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -4,6 +4,30 @@ from django.core.management.base import BaseCommand, CommandError
 from django.db import DatabaseError, connection, transaction
 
 from sources.models import SourceConfiguration, SourceRightsDecision
+from travel_windows.contracts import (
+    CITY_ATTRIBUTION,
+    CITY_CONTRACT_FINGERPRINT_SHA256,
+    CITY_FIELD_SCOPE,
+    CITY_SOURCE_CODE,
+    CITY_SOURCE_LOCATOR,
+    CITY_SOURCE_OWNER,
+    CITY_SOURCE_REVISION,
+    DATA_GO_SECRET_REFERENCE,
+    DURATION_ATTRIBUTION,
+    DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_FIELD_SCOPE,
+    DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
+    DURATION_SOURCE_OWNER,
+    DURATION_SOURCE_REVISION,
+    SCHEDULE_ATTRIBUTION,
+    SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_FIELD_SCOPE,
+    SCHEDULE_SOURCE_CODE,
+    SCHEDULE_SOURCE_LOCATOR,
+    SCHEDULE_SOURCE_OWNER,
+    SCHEDULE_SOURCE_REVISION,
+)
 
 
 REGISTRATION_LOCK_NAMESPACE = 1_414_678_614
@@ -22,6 +46,7 @@ class ApprovedSourceSpec:
     field_scope_code: str
     contract_fingerprint_sha256: str
     decision_basis_code: str
+    attribution_text: str
     connect_timeout_seconds: int = 5
     read_timeout_seconds: int = 15
     max_retries: int = 2
@@ -54,7 +79,7 @@ class ApprovedSourceSpec:
             "typed_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
             "evidence_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
             "field_scope_code": self.field_scope_code,
-            "attribution_text": "외교부|공공데이터포털",
+            "attribution_text": self.attribution_text,
             "contract_fingerprint_sha256": self.contract_fingerprint_sha256,
             "decided_by": "PROJECT_OWNER_REQUEST",
             "decision_basis_code": self.decision_basis_code,
@@ -78,6 +103,7 @@ APPROVED_SOURCE_SPECS = (
             "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
         ),
         decision_basis_code="USER_DIRECTIVE_20260830",
+        attribution_text="외교부|공공데이터포털",
     ),
     ApprovedSourceSpec(
         code="MOFA_TRAVEL_ALARM_API_JP",
@@ -95,6 +121,46 @@ APPROVED_SOURCE_SPECS = (
             "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
         ),
         decision_basis_code="USER_FOLLOWUP_20260830",
+        attribution_text="외교부|공공데이터포털",
+    ),
+    ApprovedSourceSpec(
+        code=SCHEDULE_SOURCE_CODE,
+        revision=SCHEDULE_SOURCE_REVISION,
+        module=SourceConfiguration.Module.AVIATION,
+        owner=SCHEDULE_SOURCE_OWNER,
+        official_locator=SCHEDULE_SOURCE_LOCATOR,
+        secret_reference_name=DATA_GO_SECRET_REFERENCE,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code=SCHEDULE_FIELD_SCOPE,
+        contract_fingerprint_sha256=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        attribution_text=SCHEDULE_ATTRIBUTION,
+    ),
+    ApprovedSourceSpec(
+        code=CITY_SOURCE_CODE,
+        revision=CITY_SOURCE_REVISION,
+        module=SourceConfiguration.Module.AVIATION,
+        owner=CITY_SOURCE_OWNER,
+        official_locator=CITY_SOURCE_LOCATOR,
+        secret_reference_name=DATA_GO_SECRET_REFERENCE,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code=CITY_FIELD_SCOPE,
+        contract_fingerprint_sha256=CITY_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        attribution_text=CITY_ATTRIBUTION,
+    ),
+    ApprovedSourceSpec(
+        code=DURATION_SOURCE_CODE,
+        revision=DURATION_SOURCE_REVISION,
+        module=SourceConfiguration.Module.AVIATION,
+        owner=DURATION_SOURCE_OWNER,
+        official_locator=DURATION_SOURCE_LOCATOR,
+        secret_reference_name="",
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        field_scope_code=DURATION_FIELD_SCOPE,
+        contract_fingerprint_sha256=DURATION_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        attribution_text=DURATION_ATTRIBUTION,
     ),
 )
 
@@ -166,13 +232,18 @@ def _acquire_registration_lock() -> None:
 
 
 def _validate_aviation_boundary() -> None:
+    approved_codes = {
+        spec.code for spec in APPROVED_SOURCE_SPECS
+        if spec.module == SourceConfiguration.Module.AVIATION
+    }
     aviation_sources = list(
         SourceConfiguration.objects.select_for_update()
         .filter(module=SourceConfiguration.Module.AVIATION)
-        .only("state", "enabled")
+        .only("code", "state", "enabled")
     )
     if any(
-        source.state != SourceConfiguration.State.DRAFT or source.enabled
+        source.code not in approved_codes
+        and (source.state != SourceConfiguration.State.DRAFT or source.enabled)
         for source in aviation_sources
     ):
         _raise_conflict("AVIATION_BOUNDARY_VIOLATION", "AVIATION")
@@ -367,7 +438,7 @@ def register_approved_sources(*, apply: bool) -> tuple[RegistrationOutcome, ...]
 
 
 class Command(BaseCommand):
-    help = "Check or register the two approved source contracts."
+    help = "Check or register the exact approved source contracts."
     requires_migrations_checks = True
 
     def add_arguments(self, parser):
@@ -380,7 +451,7 @@ class Command(BaseCommand):
         mode.add_argument(
             "--apply",
             action="store_true",
-            help="Register and activate both exact approved source contracts.",
+            help="Register and activate every exact approved source contract.",
         )
 
     def handle(self, *args, **options):
diff --git a/sources/migrations/0010_aviation_approved_sources.py b/sources/migrations/0010_aviation_approved_sources.py
new file mode 100644
index 0000000..0061350
--- /dev/null
+++ b/sources/migrations/0010_aviation_approved_sources.py
@@ -0,0 +1,247 @@
+from django.db import migrations, models
+from django.db.models import Q
+
+
+PARSE_GUARD_SQL = """
+CREATE OR REPLACE FUNCTION sources_guard_parse_run_change() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    artifact_state text;
+    source_module text;
+BEGIN
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.outcome <> 'STARTED' THEN
+            RAISE EXCEPTION 'a parse run must be inserted in STARTED state'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT artifact.state, source.module
+          INTO artifact_state, source_module
+          FROM sources_sourceartifact AS artifact
+          JOIN sources_sourceconfiguration AS source ON source.id = artifact.source_id
+         WHERE artifact.id = NEW.artifact_id
+         FOR UPDATE OF artifact;
+        IF NOT FOUND OR artifact_state <> 'RECEIVED' THEN
+            RAISE EXCEPTION 'a parse run requires a RECEIVED artifact'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF NOT (
+            (source_module = 'ENTRY'
+             AND NEW.parser_name = 'MOFA_ENTRY_CSV'
+             AND NEW.parser_version = 'V1'
+             AND NEW.parser_contract_fingerprint_sha256 =
+                 '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+             AND NEW.expected_schema_fingerprint_sha256 =
+                 '46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b')
+            OR (source_module = 'TRAVEL_WARNING'
+                AND NEW.parser_name = 'MOFA_TRAVEL_ALARM_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b')
+            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_FLIGHT_SCHEDULE_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '3b4295504d24cfb1e0da398399d61c328afa0f8124d7c298f5d4e4f950dfd372'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '3d8d37c4a23731d11ed6c3b3ff2d87324c9dcbde2933c158b5f4ace689b82074')
+            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ROUTE_DURATION_CSV'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '3018365ff3d3549765d5a428a1413ea730071209434def786ab6734f89f8c2ba'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307')
+        ) THEN
+            RAISE EXCEPTION 'parser contract is not registered for the source module'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'parse runs cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF (NEW.id, NEW.artifact_id, NEW.parser_name, NEW.parser_version,
+        NEW.parser_contract_fingerprint_sha256,
+        NEW.expected_schema_fingerprint_sha256, NEW.started_at)
+       IS DISTINCT FROM
+       (OLD.id, OLD.artifact_id, OLD.parser_name, OLD.parser_version,
+        OLD.parser_contract_fingerprint_sha256,
+        OLD.expected_schema_fingerprint_sha256, OLD.started_at) THEN
+        RAISE EXCEPTION 'parse run identity is immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF OLD.outcome IS DISTINCT FROM 'STARTED'
+       OR NEW.outcome NOT IN ('VALIDATED', 'QUARANTINED', 'FAILED') THEN
+        RAISE EXCEPTION 'a parse run can be closed exactly once'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+"""
+
+
+PARSE_GUARD_REVERSE_SQL = """
+CREATE OR REPLACE FUNCTION sources_guard_parse_run_change() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    artifact_state text;
+    source_module text;
+BEGIN
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.outcome <> 'STARTED' THEN
+            RAISE EXCEPTION 'a parse run must be inserted in STARTED state'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT artifact.state, source.module
+          INTO artifact_state, source_module
+          FROM sources_sourceartifact AS artifact
+          JOIN sources_sourceconfiguration AS source ON source.id = artifact.source_id
+         WHERE artifact.id = NEW.artifact_id
+         FOR UPDATE OF artifact;
+        IF NOT FOUND OR artifact_state <> 'RECEIVED' THEN
+            RAISE EXCEPTION 'a parse run requires a RECEIVED artifact'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF NOT (
+            (source_module = 'ENTRY'
+             AND NEW.parser_name = 'MOFA_ENTRY_CSV'
+             AND NEW.parser_version = 'V1'
+             AND NEW.parser_contract_fingerprint_sha256 =
+                 '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+             AND NEW.expected_schema_fingerprint_sha256 =
+                 '46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b')
+            OR (source_module = 'TRAVEL_WARNING'
+                AND NEW.parser_name = 'MOFA_TRAVEL_ALARM_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b')
+        ) THEN
+            RAISE EXCEPTION 'parser contract is not registered for the source module'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'parse runs cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF (NEW.id, NEW.artifact_id, NEW.parser_name, NEW.parser_version,
+        NEW.parser_contract_fingerprint_sha256,
+        NEW.expected_schema_fingerprint_sha256, NEW.started_at)
+       IS DISTINCT FROM
+       (OLD.id, OLD.artifact_id, OLD.parser_name, OLD.parser_version,
+        OLD.parser_contract_fingerprint_sha256,
+        OLD.expected_schema_fingerprint_sha256, OLD.started_at) THEN
+        RAISE EXCEPTION 'parse run identity is immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF OLD.outcome IS DISTINCT FROM 'STARTED'
+       OR NEW.outcome NOT IN ('VALIDATED', 'QUARANTINED', 'FAILED') THEN
+        RAISE EXCEPTION 'a parse run can be closed exactly once'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0009_aviation_draft_gate")]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="sourceconfiguration",
+            name="source_aviation_draft_disabled",
+        ),
+        migrations.AddConstraint(
+            model_name="sourceconfiguration",
+            constraint=models.CheckConstraint(
+                condition=(
+                    ~Q(module="AVIATION")
+                    | Q(state="DRAFT", enabled=False)
+                    | Q(
+                        code__in=["ICN_SCHEDULE_API", "ICN_DESTINATION_CITY_API"],
+                        revision="travel-v1",
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code="PORT_LOGISTICS_ROUTE_DURATION",
+                        revision="travel-v1",
+                        secret_reference_name="",
+                    )
+                ),
+                name="source_aviation_approved_activation",
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="fetchattempt",
+            name="fetch_provider_code_allowlist",
+        ),
+        migrations.RemoveConstraint(model_name="parserun", name="parse_name_allowlist"),
+        migrations.AlterField(
+            model_name="fetchattempt",
+            name="provider_code",
+            field=models.CharField(
+                blank=True,
+                choices=[
+                    ("MOFA_SUCCESS_0", "MOFA success code 0"),
+                    ("MOFA_GATEWAY_20", "MOFA gateway code 20"),
+                    ("MOFA_OTHER_ERROR", "Other MOFA provider error"),
+                    ("DATA_GO_SUCCESS", "Data.go.kr success"),
+                    ("DATA_GO_ERROR", "Data.go.kr provider error"),
+                ],
+                max_length=64,
+            ),
+        ),
+        migrations.AlterField(
+            model_name="parserun",
+            name="parser_name",
+            field=models.CharField(
+                choices=[
+                    ("MOFA_ENTRY_CSV", "MOFA entry CSV"),
+                    ("MOFA_TRAVEL_ALARM_JSON", "MOFA travel alarm JSON"),
+                    ("ICN_FLIGHT_SCHEDULE_JSON", "ICN scheduled flights JSON"),
+                    ("ROUTE_DURATION_CSV", "Route duration CSV"),
+                ],
+                max_length=64,
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=Q(
+                    provider_code__in=[
+                        "",
+                        "MOFA_SUCCESS_0",
+                        "MOFA_GATEWAY_20",
+                        "MOFA_OTHER_ERROR",
+                        "DATA_GO_SUCCESS",
+                        "DATA_GO_ERROR",
+                    ]
+                ),
+                name="fetch_provider_code_allowlist",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="parserun",
+            constraint=models.CheckConstraint(
+                condition=Q(
+                    parser_name__in=[
+                        "MOFA_ENTRY_CSV",
+                        "MOFA_TRAVEL_ALARM_JSON",
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ROUTE_DURATION_CSV",
+                    ]
+                ),
+                name="parse_name_allowlist",
+            ),
+        ),
+        migrations.RunSQL(PARSE_GUARD_SQL, PARSE_GUARD_REVERSE_SQL),
+    ]
diff --git a/sources/models.py b/sources/models.py
index 2763c6f..423631c 100644
--- a/sources/models.py
+++ b/sources/models.py
@@ -38,9 +38,21 @@ class SourceConfiguration(models.Model):
             models.CheckConstraint(condition=Q(revision__regex=r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"), name="source_revision_format"),
             models.CheckConstraint(condition=Q(module__in=["ENTRY", "TRAVEL_WARNING", "AVIATION"]), name="source_module_known"),
             models.CheckConstraint(
-                condition=~Q(module="AVIATION")
-                | Q(state="DRAFT", enabled=False),
-                name="source_aviation_draft_disabled",
+                condition=(
+                    ~Q(module="AVIATION")
+                    | Q(state="DRAFT", enabled=False)
+                    | Q(
+                        code__in=["ICN_SCHEDULE_API", "ICN_DESTINATION_CITY_API"],
+                        revision="travel-v1",
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code="PORT_LOGISTICS_ROUTE_DURATION",
+                        revision="travel-v1",
+                        secret_reference_name="",
+                    )
+                ),
+                name="source_aviation_approved_activation",
             ),
             models.CheckConstraint(condition=Q(state__in=["DRAFT", "RIGHTS_APPROVED", "ACTIVE", "PAUSED", "REJECTED"]), name="source_state_known"),
             models.CheckConstraint(condition=Q(official_locator__startswith="https://"), name="source_locator_https"),
@@ -123,6 +135,8 @@ class FetchAttempt(models.Model):
         MOFA_SUCCESS_0 = "MOFA_SUCCESS_0", "MOFA success code 0"
         MOFA_GATEWAY_20 = "MOFA_GATEWAY_20", "MOFA gateway code 20"
         MOFA_OTHER_ERROR = "MOFA_OTHER_ERROR", "Other MOFA provider error"
+        DATA_GO_SUCCESS = "DATA_GO_SUCCESS", "Data.go.kr success"
+        DATA_GO_ERROR = "DATA_GO_ERROR", "Data.go.kr provider error"
 
     class FailureCode(models.TextChoices):
         TIMEOUT = "TIMEOUT", "Timeout"
@@ -166,7 +180,7 @@ class FetchAttempt(models.Model):
             models.CheckConstraint(condition=Q(normalized_request_sha256__regex=r"^[0-9a-f]{64}$"), name="fetch_request_hash_format"),
             models.CheckConstraint(condition=Q(response_sha256="") | Q(response_sha256__regex=r"^[0-9a-f]{64}$"), name="fetch_response_hash_format"),
             models.CheckConstraint(
-                condition=Q(provider_code__in=["", "MOFA_SUCCESS_0", "MOFA_GATEWAY_20", "MOFA_OTHER_ERROR"]),
+                condition=Q(provider_code__in=["", "MOFA_SUCCESS_0", "MOFA_GATEWAY_20", "MOFA_OTHER_ERROR", "DATA_GO_SUCCESS", "DATA_GO_ERROR"]),
                 name="fetch_provider_code_allowlist",
             ),
             models.CheckConstraint(condition=Q(http_status__isnull=True) | Q(http_status__gte=100, http_status__lte=599), name="fetch_http_status_bounded"),
@@ -220,6 +234,11 @@ class ParseRun(models.Model):
     class ParserName(models.TextChoices):
         MOFA_ENTRY_CSV = "MOFA_ENTRY_CSV", "MOFA entry CSV"
         MOFA_TRAVEL_ALARM_JSON = "MOFA_TRAVEL_ALARM_JSON", "MOFA travel alarm JSON"
+        ICN_FLIGHT_SCHEDULE_JSON = (
+            "ICN_FLIGHT_SCHEDULE_JSON",
+            "ICN scheduled flights JSON",
+        )
+        ROUTE_DURATION_CSV = "ROUTE_DURATION_CSV", "Route duration CSV"
 
     class ParserVersion(models.TextChoices):
         V1 = "V1", "Version 1"
@@ -256,7 +275,17 @@ class ParseRun(models.Model):
     class Meta:
         constraints = [
             models.UniqueConstraint(fields=["artifact", "parser_name", "parser_version"], name="parse_identity_unique"),
-            models.CheckConstraint(condition=Q(parser_name__in=["MOFA_ENTRY_CSV", "MOFA_TRAVEL_ALARM_JSON"]), name="parse_name_allowlist"),
+            models.CheckConstraint(
+                condition=Q(
+                    parser_name__in=[
+                        "MOFA_ENTRY_CSV",
+                        "MOFA_TRAVEL_ALARM_JSON",
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ROUTE_DURATION_CSV",
+                    ]
+                ),
+                name="parse_name_allowlist",
+            ),
             models.CheckConstraint(condition=Q(parser_version="V1"), name="parse_version_allowlist"),
             models.CheckConstraint(condition=Q(parser_contract_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="parse_contract_fingerprint_format"),
             models.CheckConstraint(condition=Q(expected_schema_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="parse_expected_schema_format"),
diff --git a/sources/tests/test_configuration.py b/sources/tests/test_configuration.py
index a56cba5..a111bfd 100644
--- a/sources/tests/test_configuration.py
+++ b/sources/tests/test_configuration.py
@@ -62,7 +62,7 @@ class SourceConfigurationTests(TestCase):
         self.assertEqual(source.module, SourceConfiguration.Module.ENTRY)
         self.assertEqual(source.revision, "rights-v1")
 
-    def test_aviation_cannot_leave_draft_or_be_enabled_before_its_gate(self):
+    def test_unapproved_aviation_cannot_leave_draft_or_be_enabled(self):
         source = self.valid(
             code="AVIATION_DRAFT_ONLY",
             module=SourceConfiguration.Module.AVIATION,
@@ -93,7 +93,7 @@ class SourceConfigurationTests(TestCase):
             )
         self.assertEqual(
             caught.exception.__cause__.diag.constraint_name,
-            "source_aviation_draft_disabled",
+            "source_aviation_approved_activation",
         )
         with self.assertRaises(IntegrityError), transaction.atomic():
             SourceConfiguration.objects.filter(pk=source.pk).update(enabled=True)
diff --git a/sources/tests/test_parse_run.py b/sources/tests/test_parse_run.py
index 97c67c9..459a6bf 100644
--- a/sources/tests/test_parse_run.py
+++ b/sources/tests/test_parse_run.py
@@ -273,7 +273,9 @@ class ParseRunTests(ParseRunFixtureMixin, TestCase):
             registry_guard = cursor.fetchone()[0]
         self.assertIn("source_module = 'ENTRY'", registry_guard)
         self.assertIn("source_module = 'TRAVEL_WARNING'", registry_guard)
-        self.assertNotIn("source_module = 'AVIATION'", registry_guard)
+        self.assertIn("source_module = 'AVIATION'", registry_guard)
+        self.assertIn("ICN_FLIGHT_SCHEDULE_JSON", registry_guard)
+        self.assertIn("ROUTE_DURATION_CSV", registry_guard)
 
     def test_artifact_must_be_received_and_started_run_blocks_transition(self):
         terminal_artifact = self.make_artifact("TERMINAL", "8" * 64)
@@ -482,9 +484,8 @@ class ParseRunTests(ParseRunFixtureMixin, TestCase):
 
 class ParseRunRaceTests(ParseRunFixtureMixin, TransactionTestCase):
     def setUp(self):
-        MigrationExecutor(connection).migrate(
-            [("sources", "0009_aviation_draft_gate")]
-        )
+        self.addCleanup(lambda: restore_canonical_migration_graph(connection))
+        restore_canonical_migration_graph(connection)
         self.artifact = self.make_artifact("RACE")
 
     def run_workers(self, workers):
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index ac167cf..27a54de 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -77,14 +77,23 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
 
         self.assertEqual(SourceConfiguration.objects.count(), 0)
         self.assertEqual(SourceRightsDecision.objects.count(), 0)
-        self.assertEqual(output.count("result=WOULD_CREATE_AND_ACTIVATE"), 2)
+        self.assertEqual(
+            output.count("result=WOULD_CREATE_AND_ACTIVATE"),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
         self.assertIn("mode=check result=ok", output)
 
-    def test_apply_creates_the_two_exact_active_sources_and_rights(self):
+    def test_apply_creates_every_exact_active_source_and_rights(self):
         output = self.call_registration("--apply")
 
-        self.assertEqual(SourceConfiguration.objects.count(), 2)
-        self.assertEqual(SourceRightsDecision.objects.count(), 2)
+        self.assertEqual(
+            SourceConfiguration.objects.count(),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
+        self.assertEqual(
+            SourceRightsDecision.objects.count(),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
         self.assertEqual(
             set(
                 SourceConfiguration.objects.values_list(
@@ -103,33 +112,19 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             ),
             {
                 (
-                    "MOFA_ENTRY_CSV",
-                    "rights-v1",
-                    "ENTRY",
-                    "대한민국 외교부 정보화담당관실",
-                    "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
-                    "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N",
+                    spec.code,
+                    spec.revision,
+                    spec.module,
+                    spec.owner,
+                    spec.official_locator,
                     "ACTIVE",
                     True,
-                    5,
-                    15,
-                    2,
-                    "",
-                ),
-                (
-                    "MOFA_TRAVEL_ALARM_API_JP",
-                    "rights-v1",
-                    "TRAVEL_WARNING",
-                    "대한민국 외교부",
-                    "https://apis.data.go.kr/1262000/TravelAlarmService2/"
-                    "getTravelAlarmList2",
-                    "ACTIVE",
-                    True,
-                    5,
-                    15,
-                    2,
-                    SECRET_REFERENCE_NAME,
-                ),
+                    spec.connect_timeout_seconds,
+                    spec.read_timeout_seconds,
+                    spec.max_retries,
+                    spec.secret_reference_name,
+                )
+                for spec in registration.APPROVED_SOURCE_SPECS
             },
         )
         self.assertEqual(
@@ -157,45 +152,28 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             ),
             {
                 (
-                    "MOFA_ENTRY_CSV",
-                    "rights-v1",
-                    1,
-                    "APPROVED",
-                    "ANONYMOUS_PUBLIC",
-                    True,
-                    True,
-                    True,
-                    False,
-                    True,
-                    0,
-                    "PRODUCT_HISTORY",
-                    "PRODUCT_HISTORY",
-                    "JP_ORDINARY_TEXT_V1",
-                    "외교부|공공데이터포털",
-                    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021",
-                    "PROJECT_OWNER_REQUEST",
-                    "USER_DIRECTIVE_20260830",
-                ),
-                (
-                    "MOFA_TRAVEL_ALARM_API_JP",
-                    "rights-v1",
-                    1,
-                    "APPROVED",
-                    "CREDENTIAL_REFERENCE",
-                    True,
-                    True,
-                    True,
-                    False,
-                    True,
-                    0,
-                    "PRODUCT_HISTORY",
-                    "PRODUCT_HISTORY",
-                    "JP_WARNING_V1",
-                    "외교부|공공데이터포털",
-                    "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6",
-                    "PROJECT_OWNER_REQUEST",
-                    "USER_FOLLOWUP_20260830",
-                ),
+                    spec.code,
+                    *tuple(spec.rights_values()[field] for field in (
+                        "source_revision",
+                        "decision_sequence",
+                        "decision",
+                        "access_mode",
+                        "access_allowed",
+                        "automated_collection_allowed",
+                        "typed_field_storage_allowed",
+                        "raw_body_storage_allowed",
+                        "typed_republication_allowed",
+                        "raw_retention_seconds",
+                        "typed_retention",
+                        "evidence_retention",
+                        "field_scope_code",
+                        "attribution_text",
+                        "contract_fingerprint_sha256",
+                        "decided_by",
+                        "decision_basis_code",
+                    )),
+                )
+                for spec in registration.APPROVED_SOURCE_SPECS
             },
         )
         self.assertTrue(
@@ -206,12 +184,15 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
                 )
             )
         )
-        self.assertEqual(output.count("result=CREATED_AND_ACTIVATED"), 2)
+        self.assertEqual(
+            output.count("result=CREATED_AND_ACTIVATED"),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
         self.assertEqual(
             SourceConfiguration.objects.filter(
                 module=SourceConfiguration.Module.AVIATION
             ).count(),
-            0,
+            3,
         )
 
     def test_apply_rerun_is_a_noop_and_preserves_immutable_times(self):
@@ -245,10 +226,13 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             ),
             rights_identity,
         )
-        self.assertEqual(output.count("result=ALREADY_ACTIVE"), 2)
+        self.assertEqual(
+            output.count("result=ALREADY_ACTIVE"),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
 
     def test_exact_partial_rows_resume_without_replacement(self):
-        entry_spec, warning_spec = registration.APPROVED_SOURCE_SPECS
+        entry_spec, warning_spec = registration.APPROVED_SOURCE_SPECS[:2]
         entry = self.make_exact_source(entry_spec)
         warning = self.make_exact_source(warning_spec)
         warning_approval = self.make_exact_approval(warning, warning_spec)
@@ -428,7 +412,7 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
         self.assertFalse(warning.enabled)
         self.assertEqual(warning.rights_decisions.count(), 2)
 
-    def test_draft_aviation_is_untouched_and_no_aviation_is_created(self):
+    def test_unapproved_draft_aviation_is_untouched(self):
         aviation = self.make_aviation_source()
         original = (aviation.pk, aviation.revision, aviation.created_at)
 
@@ -444,7 +428,7 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             SourceConfiguration.objects.filter(
                 module=SourceConfiguration.Module.AVIATION
             ).count(),
-            1,
+            4,
         )
 
     def test_secret_environment_is_not_read_or_disclosed(self):
@@ -520,8 +504,14 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
         self.assertEqual(len(results), 2)
         for output in results:
             self.assert_safe_output(output)
-        self.assertEqual(SourceConfiguration.objects.count(), 2)
-        self.assertEqual(SourceRightsDecision.objects.count(), 2)
+        self.assertEqual(
+            SourceConfiguration.objects.count(),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
+        self.assertEqual(
+            SourceRightsDecision.objects.count(),
+            len(registration.APPROVED_SOURCE_SPECS),
+        )
         self.assertTrue(
             all(
                 source.state == SourceConfiguration.State.ACTIVE
diff --git a/sources/transport.py b/sources/transport.py
index 643d713..ed7c9c6 100644
--- a/sources/transport.py
+++ b/sources/transport.py
@@ -11,6 +11,7 @@ from __future__ import annotations
 import hashlib
 import http.client
 import json
+import os
 import re
 import socket
 from dataclasses import dataclass, field
@@ -18,6 +19,13 @@ from typing import Callable, Protocol
 from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsplit
 from xml.etree import ElementTree
 
+from travel_windows.contracts import (
+    DATA_GO_SECRET_REFERENCE,
+    SCHEDULE_ARRIVALS_LOCATOR,
+    SCHEDULE_DEPARTURES_LOCATOR,
+    load_data_go_service_key,
+)
+
 
 ENTRY_SOURCE_LOCATOR = (
     "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
@@ -31,6 +39,9 @@ WARNING_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
 
 ENTRY_MAX_RESPONSE_BYTES = 262_144
 WARNING_MAX_RESPONSE_BYTES = 4_096
+SCHEDULE_PAGE_MAX_RESPONSE_BYTES = 1_048_576
+SCHEDULE_PAGE_SIZE = 100
+SCHEDULE_MAX_PAGES_PER_DIRECTION = 100
 
 ATTEMPT_SUCCEEDED = "SUCCEEDED"
 ATTEMPT_RETRYABLE_FAILED = "RETRYABLE_FAILED"
@@ -51,6 +62,24 @@ PROVIDER_GATEWAY_20 = "MOFA_GATEWAY_20"
 PROVIDER_OTHER_ERROR = "MOFA_OTHER_ERROR"
 
 
+def load_aviation_secret_reference(
+    environment: dict[str, str] | None = None,
+) -> str | None:
+    """Resolve canonical DATA_GO key with the documented legacy fallback."""
+
+    return load_data_go_service_key(
+        environment if environment is not None else os.environ
+    )
+
+
+@dataclass(frozen=True, slots=True)
+class SchedulePageFetchResult:
+    succeeded: bool
+    departure_pages: tuple[bytes, ...] = field(default=(), repr=False)
+    arrival_pages: tuple[bytes, ...] = field(default=(), repr=False)
+    failure_code: str = ""
+
+
 @dataclass(frozen=True, slots=True)
 class RequestFingerprint:
     revision: str
@@ -563,3 +592,138 @@ def fetch_travel_alarm_jp(
             PROVIDER_SUCCESS_0 if provider_result_code == "0" else ""
         ),
     )
+
+
+def _schedule_page_count(body: bytes) -> int | None:
+    try:
+        document = json.loads(body.decode("utf-8"))
+        response = document["response"]
+        header = response["header"]
+        page_body = response["body"]
+        if header["resultCode"] not in {"00", "0"}:
+            return None
+        page_number = int(page_body["pageNo"])
+        page_size = int(page_body["numOfRows"])
+        total = int(page_body["totalCount"])
+        if page_number < 1 or page_size < 1 or total < 1:
+            return None
+        return (total + page_size - 1) // page_size
+    except (KeyError, TypeError, ValueError, UnicodeError, json.JSONDecodeError):
+        return None
+
+
+def _fetch_schedule_direction(
+    *,
+    locator: str,
+    decoded_secret: str,
+    season: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory,
+) -> tuple[tuple[bytes, ...], str]:
+    parts = urlsplit(locator)
+    pages: list[bytes] = []
+    expected_pages: int | None = None
+    for page_number in range(1, SCHEDULE_MAX_PAGES_PER_DIRECTION + 1):
+        query = urlencode(
+            (
+                ("serviceKey", decoded_secret),
+                ("pageNo", str(page_number)),
+                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+                ("season", season),
+                ("type", "json"),
+            ),
+            doseq=False,
+        )
+        wire_result = _read_once(
+            host=parts.hostname or "",
+            request_target=f"{parts.path}?{query}",
+            connect_timeout_seconds=connect_timeout_seconds,
+            read_timeout_seconds=read_timeout_seconds,
+            max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
+            request_headers={**_COMMON_REQUEST_HEADERS, "Accept": "application/json"},
+            connection_factory=connection_factory,
+        )
+        if isinstance(wire_result, _WireFailure):
+            return (), wire_result.failure_code
+        if wire_result.status != 200:
+            if wire_result.status in {401, 403}:
+                return (), FAILURE_AUTHENTICATION
+            if wire_result.status == 429:
+                return (), FAILURE_RATE_LIMITED
+            if 500 <= wire_result.status <= 599:
+                return (), FAILURE_UPSTREAM_5XX
+            return (), FAILURE_HTTP_CLIENT
+        if _contains_secret_reflection(
+            wire_result.body,
+            decoded_secret,
+            decoded_secret,
+        ):
+            return (), FAILURE_SECRET_REFLECTION
+        result_code = _provider_result_code(wire_result.body)
+        if result_code not in {"0", "00"}:
+            return (), FAILURE_PROVIDER_ERROR
+        page_count = _schedule_page_count(wire_result.body)
+        if page_count is None or page_count > SCHEDULE_MAX_PAGES_PER_DIRECTION:
+            return (), FAILURE_PROVIDER_ERROR
+        if expected_pages is None:
+            expected_pages = page_count
+        elif expected_pages != page_count:
+            return (), FAILURE_PROVIDER_ERROR
+        pages.append(wire_result.body)
+        if page_number == expected_pages:
+            return tuple(pages), ""
+    return (), FAILURE_RESPONSE_TOO_LARGE
+
+
+def fetch_data_go_schedule_pages(
+    *,
+    secret_reference_name: str,
+    secret_value: str,
+    season: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory = _default_connection_factory,
+) -> SchedulePageFetchResult:
+    """Fetch complete official departure/arrival pages without persistence."""
+
+    if (
+        secret_reference_name != DATA_GO_SECRET_REFERENCE
+        or not isinstance(secret_value, str)
+        or not secret_value
+        or len(secret_value) > _MAX_SECRET_CHARACTERS
+        or not isinstance(season, str)
+        or not re.fullmatch(r"[A-Za-z0-9가-힣_-]{2,32}", season)
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+    ):
+        return SchedulePageFetchResult(False, failure_code=FAILURE_AUTHENTICATION)
+    try:
+        decoded_secret = unquote(secret_value, encoding="utf-8", errors="strict")
+    except (UnicodeError, TypeError, ValueError):
+        return SchedulePageFetchResult(False, failure_code=FAILURE_AUTHENTICATION)
+    departures, failure = _fetch_schedule_direction(
+        locator=SCHEDULE_DEPARTURES_LOCATOR,
+        decoded_secret=decoded_secret,
+        season=season,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+    )
+    if failure:
+        return SchedulePageFetchResult(False, failure_code=failure)
+    arrivals, failure = _fetch_schedule_direction(
+        locator=SCHEDULE_ARRIVALS_LOCATOR,
+        decoded_secret=decoded_secret,
+        season=season,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+    )
+    if failure:
+        return SchedulePageFetchResult(False, failure_code=failure)
+    return SchedulePageFetchResult(
+        True,
+        departure_pages=departures,
+        arrival_pages=arrivals,
+    )
diff --git a/travel_windows/contracts.py b/travel_windows/contracts.py
new file mode 100644
index 0000000..a6ebd88
--- /dev/null
+++ b/travel_windows/contracts.py
@@ -0,0 +1,80 @@
+import hashlib
+
+
+DATA_GO_SECRET_REFERENCE = "DATA_GO_KR_SERVICE_KEY"
+LEGACY_DATA_GO_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+
+SCHEDULE_SOURCE_CODE = "ICN_SCHEDULE_API"
+SCHEDULE_SOURCE_REVISION = "travel-v1"
+SCHEDULE_SOURCE_OWNER = "인천국제공항공사"
+SCHEDULE_SOURCE_LOCATOR = (
+    "https://apis.data.go.kr/B551177/statusOfSPaxFlt4TripPlatform"
+)
+SCHEDULE_DEPARTURES_LOCATOR = (
+    f"{SCHEDULE_SOURCE_LOCATOR}/getSPaxFlt4TripPlatformDepartures"
+)
+SCHEDULE_ARRIVALS_LOCATOR = (
+    f"{SCHEDULE_SOURCE_LOCATOR}/getSPaxFlt4TripPlatformArrivals"
+)
+SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
+SCHEDULE_FIELD_SCOPE = "ICN_SCHEDULE_V1"
+
+CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API"
+CITY_SOURCE_REVISION = "travel-v1"
+CITY_SOURCE_OWNER = "인천국제공항공사"
+CITY_SOURCE_LOCATOR = "https://www.data.go.kr/"
+CITY_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
+CITY_FIELD_SCOPE = "ICN_DESTINATION_CITY_V1"
+
+DURATION_SOURCE_CODE = "PORT_LOGISTICS_ROUTE_DURATION"
+DURATION_SOURCE_REVISION = "travel-v1"
+DURATION_SOURCE_OWNER = "대한민국 해양수산부"
+DURATION_SOURCE_LOCATOR = "https://www.ulip.go.kr/"
+DURATION_ATTRIBUTION = "해양수산부 수출입 물류 플랫폼"
+DURATION_FIELD_SCOPE = "ROUTE_DURATION_V1"
+
+SCHEDULE_CONTRACT_TEXT = (
+    "ICN schedule data.go v1|response(header(resultCode,resultMsg),body("
+    "items(item(codeshare,masterFlightId,flightId,st,season,firstdate,"
+    "lastdate,ynMon,ynTue,ynWed,ynThu,ynFri,ynSat,ynSun,terminalId,airline,"
+    "airlineCode,airport,airportCode,tmp1,tmp2)),pageNo,numOfRows,totalCount))|"
+    "departures+arrivals|complete-pages|codeshare-master-dedupe"
+)
+SCHEDULE_SCHEMA_TEXT = (
+    "documented-data-go-json(response.header+response.body.items.item+"
+    "pageNo+numOfRows+totalCount); normalized-object(source_date:date,"
+    "season:string,coverage_complete:true,flights:list[direction,carrier_code,"
+    "carrier_name,flight_number,master_flight_number,destination_iata,"
+    "icn_event_time,valid_from,valid_until,weekdays])"
+)
+DURATION_CONTRACT_TEXT = (
+    "route duration v1|destination_iata,outbound_minutes,inbound_minutes,"
+    "reference_date,reference_locator"
+)
+DURATION_SCHEMA_TEXT = (
+    "csv(destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+    "reference_locator)"
+)
+
+
+def _sha256(value: str) -> str:
+    return hashlib.sha256(value.encode("utf-8")).hexdigest()
+
+
+SCHEDULE_CONTRACT_FINGERPRINT_SHA256 = _sha256(SCHEDULE_CONTRACT_TEXT)
+SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(SCHEDULE_SCHEMA_TEXT)
+DURATION_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_CONTRACT_TEXT)
+DURATION_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_SCHEMA_TEXT)
+CITY_CONTRACT_FINGERPRINT_SHA256 = _sha256(
+    "ICN destination city v1|iata|country|city|timezone evidence"
+)
+
+
+def load_data_go_service_key(environment: dict[str, str]) -> str | None:
+    """Return the canonical key, falling back without exposing either value."""
+
+    canonical = environment.get(DATA_GO_SECRET_REFERENCE)
+    if canonical:
+        return canonical
+    legacy = environment.get(LEGACY_DATA_GO_SECRET_REFERENCE)
+    return legacy or None
diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
new file mode 100644
index 0000000..d64733a
--- /dev/null
+++ b/travel_windows/ingestion.py
@@ -0,0 +1,275 @@
+"""Offline ingestion bridge from documented source pages to publication.
+
+Raw response bytes are accepted in memory and are never written to a model,
+log, audit row, or exception.  Only bounded receipt hashes and typed revisions
+survive the call.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import uuid
+from dataclasses import dataclass
+from datetime import date, datetime
+
+from django.db import transaction
+from django.utils import timezone
+
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+
+from .contracts import (
+    DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_CODE,
+    SCHEDULE_ARRIVALS_LOCATOR,
+    SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_DEPARTURES_LOCATOR,
+    SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    SCHEDULE_SOURCE_CODE,
+)
+from .parser import (
+    AviationParseFailure,
+    DurationParseSuccess,
+    ScheduleParseSuccess,
+    adapt_data_go_schedule_pages,
+    parse_route_durations,
+)
+from .publication import FlightPublicationCode, publish_flight_evidence
+
+
+class FlightIngestionCode:
+    PUBLISHED = "PUBLISHED"
+    PARSE_QUARANTINED = "PARSE_QUARANTINED"
+    SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
+    PERSISTENCE_FAILED = "PERSISTENCE_FAILED"
+
+
+@dataclass(frozen=True, slots=True)
+class FlightIngestionOutcome:
+    code: str
+    publication_id: str | None = None
+    generation: int | None = None
+
+    @property
+    def succeeded(self) -> bool:
+        return self.code == FlightIngestionCode.PUBLISHED
+
+
+class _IngestionRejected(Exception):
+    def __init__(self, code: str):
+        self.code = code
+        super().__init__(code)
+
+
+def _digest(parts: tuple[bytes, ...]) -> tuple[str, int]:
+    digest = hashlib.sha256()
+    byte_count = 0
+    for part in parts:
+        if not isinstance(part, bytes):
+            raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
+        digest.update(len(part).to_bytes(8, "big"))
+        digest.update(part)
+        byte_count += len(part)
+    return digest.hexdigest(), byte_count
+
+
+def _request_hash(value: str) -> str:
+    return hashlib.sha256(value.encode("utf-8")).hexdigest()
+
+
+def _locked_source(code: str) -> tuple[SourceConfiguration, SourceRightsDecision]:
+    source = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(
+            code=code,
+            module=SourceConfiguration.Module.AVIATION,
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        .first()
+    )
+    if source is None:
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    rights = (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=source.revision)
+        .order_by("-decision_sequence", "-id")
+        .first()
+    )
+    if (
+        rights is None
+        or rights.decision != SourceRightsDecision.Decision.APPROVED
+        or not rights.access_allowed
+        or not rights.automated_collection_allowed
+        or not rights.typed_field_storage_allowed
+        or not rights.typed_republication_allowed
+        or rights.raw_body_storage_allowed
+    ):
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    return source, rights
+
+
+def _record_validated_parse(
+    *,
+    source_code: str,
+    payload_parts: tuple[bytes, ...],
+    request_fingerprint_revision: str,
+    normalized_request_sha256: str,
+    provider_code: str,
+    parser_name: str,
+    contract_fingerprint: str,
+    schema_fingerprint: str,
+    completed_at: datetime,
+) -> ParseRun:
+    body_sha256, byte_count = _digest(payload_parts)
+    with transaction.atomic(durable=True):
+        source, rights = _locked_source(source_code)
+        operation_id = uuid.uuid4()
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=operation_id,
+            attempt_number=1,
+            request_fingerprint_revision=request_fingerprint_revision,
+            normalized_request_sha256=normalized_request_sha256,
+        )
+        closed_at = max(completed_at, attempt.started_at)
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            result=FetchAttempt.Result.SUCCEEDED,
+            completed_at=closed_at,
+            http_status=200,
+            provider_code=provider_code,
+            response_sha256=body_sha256,
+            failure_code="",
+        )
+        attempt.refresh_from_db()
+        artifact = SourceArtifact.objects.filter(
+            source=source,
+            body_sha256=body_sha256,
+        ).first()
+        if artifact is None:
+            artifact = SourceArtifact.objects.create(
+                source=source,
+                body_sha256=body_sha256,
+                byte_count=byte_count,
+                first_successful_attempt=attempt,
+            )
+            parse_run = ParseRun.objects.create(
+                artifact=artifact,
+                parser_name=parser_name,
+                parser_version=ParseRun.ParserVersion.V1,
+                parser_contract_fingerprint_sha256=contract_fingerprint,
+                expected_schema_fingerprint_sha256=schema_fingerprint,
+            )
+            parse_completed_at = max(closed_at, parse_run.started_at)
+            ParseRun.objects.filter(pk=parse_run.pk).update(
+                completed_at=parse_completed_at,
+                outcome=ParseRun.Outcome.VALIDATED,
+                failure_code="",
+                observed_schema_fingerprint_sha256=schema_fingerprint,
+            )
+            SourceArtifact.objects.filter(pk=artifact.pk).update(
+                state=SourceArtifact.State.REVIEW_REQUIRED
+            )
+            parse_run.refresh_from_db()
+            return parse_run
+        if artifact.byte_count != byte_count:
+            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+        try:
+            parse_run = artifact.parse_runs.get(
+                parser_name=parser_name,
+                parser_version=ParseRun.ParserVersion.V1,
+                parser_contract_fingerprint_sha256=contract_fingerprint,
+                expected_schema_fingerprint_sha256=schema_fingerprint,
+                observed_schema_fingerprint_sha256=schema_fingerprint,
+                outcome=ParseRun.Outcome.VALIDATED,
+            )
+        except ParseRun.DoesNotExist:
+            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED) from None
+        return parse_run
+
+
+def ingest_and_publish_flight_evidence(
+    *,
+    departure_pages: tuple[bytes, ...],
+    arrival_pages: tuple[bytes, ...],
+    duration_csv: bytes,
+    source_date: date,
+    published_by: str,
+    source_checked_at: datetime | None = None,
+) -> FlightIngestionOutcome:
+    """Validate complete pages, persist redacted evidence, and publish atomically."""
+
+    checked_at = source_checked_at or timezone.now()
+    schedule = adapt_data_go_schedule_pages(
+        departure_pages=departure_pages,
+        arrival_pages=arrival_pages,
+        source_date=source_date,
+    )
+    durations = parse_route_durations(duration_csv)
+    if isinstance(schedule, AviationParseFailure) or isinstance(
+        durations, AviationParseFailure
+    ):
+        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
+    if not isinstance(schedule, ScheduleParseSuccess) or not isinstance(
+        durations, DurationParseSuccess
+    ):
+        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
+    try:
+        schedule_run = _record_validated_parse(
+            source_code=SCHEDULE_SOURCE_CODE,
+            payload_parts=(*departure_pages, *arrival_pages),
+            request_fingerprint_revision="ICN_SCHEDULE_V1",
+            normalized_request_sha256=_request_hash(
+                f"{SCHEDULE_DEPARTURES_LOCATOR}\n{SCHEDULE_ARRIVALS_LOCATOR}\n"
+                "serviceKey=<redacted>&pageNo=1..N&type=json"
+            ),
+            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+            parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+            contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            completed_at=checked_at,
+        )
+        duration_run = _record_validated_parse(
+            source_code=DURATION_SOURCE_CODE,
+            payload_parts=(duration_csv,),
+            request_fingerprint_revision="ROUTE_DURATION_V1",
+            normalized_request_sha256=_request_hash(
+                "reviewed-route-duration-csv-v1"
+            ),
+            provider_code="",
+            parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
+            contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
+            completed_at=checked_at,
+        )
+        outcome = publish_flight_evidence(
+            schedule_run=schedule_run,
+            schedule=schedule,
+            duration_run=duration_run,
+            durations=durations,
+            published_by=published_by,
+            source_checked_at=checked_at,
+        )
+    except _IngestionRejected as exc:
+        return FlightIngestionOutcome(exc.code)
+    except Exception:
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
+    if outcome.code == FlightPublicationCode.SOURCE_GATE_FAILED:
+        return FlightIngestionOutcome(FlightIngestionCode.SOURCE_GATE_FAILED)
+    if outcome.code == FlightPublicationCode.INVALID_EVIDENCE:
+        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
+    if not outcome.succeeded:
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
+    return FlightIngestionOutcome(
+        FlightIngestionCode.PUBLISHED,
+        outcome.publication_id,
+        outcome.generation,
+    )
diff --git a/travel_windows/management/__init__.py b/travel_windows/management/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/travel_windows/management/__init__.py
@@ -0,0 +1 @@
+
diff --git a/travel_windows/management/commands/__init__.py b/travel_windows/management/commands/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/travel_windows/management/commands/__init__.py
@@ -0,0 +1 @@
+
diff --git a/travel_windows/management/commands/publish_scheduled_flights.py b/travel_windows/management/commands/publish_scheduled_flights.py
new file mode 100644
index 0000000..97346a9
--- /dev/null
+++ b/travel_windows/management/commands/publish_scheduled_flights.py
@@ -0,0 +1,65 @@
+from datetime import date
+from pathlib import Path
+
+from django.core.management.base import BaseCommand, CommandError
+
+from sources.models import SourceConfiguration
+from sources.transport import (
+    fetch_data_go_schedule_pages,
+    load_aviation_secret_reference,
+)
+from travel_windows.contracts import DATA_GO_SECRET_REFERENCE, SCHEDULE_SOURCE_CODE
+from travel_windows.ingestion import ingest_and_publish_flight_evidence
+
+
+class Command(BaseCommand):
+    help = (
+        "Validate complete documented schedule pages and a reviewed duration "
+        "CSV, then advance the flight publication pointer."
+    )
+    requires_migrations_checks = True
+
+    def add_arguments(self, parser):
+        parser.add_argument("--duration-csv", required=True)
+        parser.add_argument("--source-date", required=True)
+        parser.add_argument("--season", required=True)
+        parser.add_argument("--published-by", required=True)
+
+    def handle(self, *args, **options):
+        try:
+            source_date = date.fromisoformat(options["source_date"])
+            duration_csv = Path(options["duration_csv"]).read_bytes()
+            source = SourceConfiguration.objects.get(
+                code=SCHEDULE_SOURCE_CODE,
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+                secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            )
+        except (OSError, TypeError, ValueError):
+            raise CommandError("result=blocked code=INVALID_INPUT") from None
+        except SourceConfiguration.DoesNotExist:
+            raise CommandError("result=blocked code=SOURCE_GATE_FAILED") from None
+
+        secret_value = load_aviation_secret_reference()
+        if secret_value is None:
+            raise CommandError("result=blocked code=SECRET_UNAVAILABLE")
+        fetched = fetch_data_go_schedule_pages(
+            secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            secret_value=secret_value,
+            season=options["season"],
+            connect_timeout_seconds=source.connect_timeout_seconds,
+            read_timeout_seconds=source.read_timeout_seconds,
+        )
+        if not fetched.succeeded:
+            raise CommandError(f"result=blocked code={fetched.failure_code}")
+
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=fetched.departure_pages,
+            arrival_pages=fetched.arrival_pages,
+            duration_csv=duration_csv,
+            source_date=source_date,
+            published_by=options["published_by"],
+        )
+        if not outcome.succeeded:
+            raise CommandError(f"result=blocked code={outcome.code}")
+        self.stdout.write(f"result=published generation={outcome.generation}")
diff --git a/travel_windows/migrations/0002_scheduled_flight_evidence.py b/travel_windows/migrations/0002_scheduled_flight_evidence.py
new file mode 100644
index 0000000..3cf134c
--- /dev/null
+++ b/travel_windows/migrations/0002_scheduled_flight_evidence.py
@@ -0,0 +1,268 @@
+import django.db.models.deletion
+import django.utils.timezone
+import uuid
+from django.db import migrations, models
+from django.db.models import F, Q
+
+
+FLIGHT_POINTER_ID = uuid.UUID("bd18f3e9-7928-5b79-8a68-4bdcfdd329fd")
+
+
+IMMUTABILITY_SQL = """
+CREATE FUNCTION travel_windows_reject_revision_mutation() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    RAISE EXCEPTION 'published flight evidence is immutable'
+        USING ERRCODE = 'check_violation';
+END;
+$$;
+CREATE TRIGGER travel_windows_schedule_revision_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_flightschedulerevision
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+CREATE TRIGGER travel_windows_schedule_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_flightschedule
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+CREATE TRIGGER travel_windows_duration_revision_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_routedurationrevision
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+CREATE TRIGGER travel_windows_publication_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_flightpublication
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+CREATE TRIGGER travel_windows_publication_duration_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_flightpublicationduration
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+
+CREATE FUNCTION travel_windows_guard_flight_pointer() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    candidate_generation bigint;
+    candidate_supersedes uuid;
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'flight current pointer cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.id IS DISTINCT FROM OLD.id
+       OR NEW.version <> OLD.version + 1
+       OR NEW.current_publication_id IS NULL THEN
+        RAISE EXCEPTION 'flight current pointer transition is invalid'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT generation, supersedes_id
+      INTO candidate_generation, candidate_supersedes
+      FROM travel_windows_flightpublication
+     WHERE id = NEW.current_publication_id;
+    IF NOT FOUND
+       OR candidate_generation <> NEW.version
+       OR candidate_supersedes IS DISTINCT FROM OLD.current_publication_id THEN
+        RAISE EXCEPTION 'flight publication does not extend current history'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_windows_flight_pointer_guard
+BEFORE UPDATE OR DELETE ON travel_windows_publishedflightschedule
+FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_flight_pointer();
+"""
+
+
+IMMUTABILITY_REVERSE_SQL = """
+DROP TRIGGER IF EXISTS travel_windows_flight_pointer_guard
+    ON travel_windows_publishedflightschedule;
+DROP FUNCTION IF EXISTS travel_windows_guard_flight_pointer();
+DROP TRIGGER IF EXISTS travel_windows_publication_duration_immutable
+    ON travel_windows_flightpublicationduration;
+DROP TRIGGER IF EXISTS travel_windows_publication_immutable
+    ON travel_windows_flightpublication;
+DROP TRIGGER IF EXISTS travel_windows_duration_revision_immutable
+    ON travel_windows_routedurationrevision;
+DROP TRIGGER IF EXISTS travel_windows_schedule_immutable
+    ON travel_windows_flightschedule;
+DROP TRIGGER IF EXISTS travel_windows_schedule_revision_immutable
+    ON travel_windows_flightschedulerevision;
+DROP FUNCTION IF EXISTS travel_windows_reject_revision_mutation();
+"""
+
+
+def seed_pointer(apps, schema_editor):
+    Pointer = apps.get_model("travel_windows", "PublishedFlightSchedule")
+    pointer, _ = Pointer.objects.using(schema_editor.connection.alias).get_or_create(
+        id=FLIGHT_POINTER_ID,
+        defaults={"version": 0},
+    )
+    if pointer.current_publication_id is not None or pointer.version != 0:
+        raise RuntimeError("flight pointer seed conflicts with publication history")
+
+
+def require_empty_pointer(apps, schema_editor):
+    Pointer = apps.get_model("travel_windows", "PublishedFlightSchedule")
+    pointer = (
+        Pointer.objects.using(schema_editor.connection.alias)
+        .filter(id=FLIGHT_POINTER_ID)
+        .first()
+    )
+    if pointer is None:
+        return
+    if pointer.current_publication_id is not None or pointer.version != 0:
+        raise RuntimeError("flight evidence rollback requires an empty pointer")
+    pointer.delete()
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("sources", "0010_aviation_approved_sources"),
+        ("travel_windows", "0001_initial"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="FlightScheduleRevision",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("source_date", models.DateField()),
+                ("season", models.CharField(max_length=32)),
+                ("coverage_code", models.CharField(max_length=64)),
+                ("completeness", models.CharField(choices=[("COMPLETE", "Complete")], default="COMPLETE", max_length=16)),
+                ("state", models.CharField(choices=[("VALIDATED", "Validated"), ("QUARANTINED", "Quarantined")], max_length=16)),
+                ("first_observed_at", models.DateTimeField()),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                ("created_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("parse_run", models.OneToOneField(on_delete=django.db.models.deletion.PROTECT, related_name="flight_schedule_revision", to="sources.parserun")),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(condition=~Q(season=""), name="flight_revision_season_present"),
+                    models.CheckConstraint(condition=Q(coverage_code__regex=r"^[A-Z][A-Z0-9_]{2,63}$"), name="flight_revision_coverage_format"),
+                    models.CheckConstraint(condition=Q(completeness="COMPLETE"), name="flight_revision_complete_only"),
+                    models.CheckConstraint(condition=Q(state__in=["VALIDATED", "QUARANTINED"]), name="flight_revision_state_known"),
+                    models.CheckConstraint(condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="flight_revision_typed_hash"),
+                    models.CheckConstraint(condition=Q(created_at__gte=F("first_observed_at")), name="flight_revision_created_after_observed"),
+                ]
+            },
+        ),
+        migrations.CreateModel(
+            name="FlightSchedule",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("direction", models.CharField(choices=[("OUTBOUND", "ICN departure"), ("INBOUND", "ICN arrival")], max_length=16)),
+                ("carrier_code", models.CharField(max_length=3)),
+                ("carrier_name", models.CharField(max_length=100)),
+                ("flight_number", models.CharField(max_length=8)),
+                ("master_flight_number", models.CharField(max_length=8)),
+                ("icn_event_time", models.TimeField()),
+                ("valid_from", models.DateField()),
+                ("valid_until", models.DateField()),
+                ("weekday_mask", models.CharField(max_length=7)),
+                ("destination_airport", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="flight_schedules", to="travel_windows.airport")),
+                ("revision", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="schedules", to="travel_windows.flightschedulerevision")),
+            ],
+            options={
+                "ordering": ("destination_airport__city_code", "direction", "icn_event_time", "master_flight_number", "flight_number"),
+                "constraints": [
+                    models.CheckConstraint(condition=Q(direction__in=["OUTBOUND", "INBOUND"]), name="flight_schedule_direction_known"),
+                    models.CheckConstraint(condition=Q(carrier_code__regex=r"^[A-Z0-9]{2,3}$"), name="flight_schedule_carrier_format"),
+                    models.CheckConstraint(condition=~Q(carrier_name=""), name="flight_schedule_carrier_present"),
+                    models.CheckConstraint(condition=Q(flight_number__regex=r"^[A-Z0-9]{2,3}[0-9]{1,4}[A-Z]?$"), name="flight_schedule_number_format"),
+                    models.CheckConstraint(condition=Q(master_flight_number__regex=r"^[A-Z0-9]{2,3}[0-9]{1,4}[A-Z]?$"), name="flight_schedule_master_format"),
+                    models.CheckConstraint(condition=Q(valid_until__gte=F("valid_from")), name="flight_schedule_valid_range"),
+                    models.CheckConstraint(condition=Q(weekday_mask__regex=r"^[01]{7}$") & ~Q(weekday_mask="0000000"), name="flight_schedule_weekday_mask"),
+                    models.UniqueConstraint(fields=("revision", "direction", "master_flight_number", "destination_airport", "icn_event_time", "valid_from", "valid_until", "weekday_mask"), name="flight_schedule_master_identity"),
+                ],
+            },
+        ),
+        migrations.CreateModel(
+            name="FlightPublication",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("generation", models.PositiveBigIntegerField(unique=True)),
+                ("state", models.CharField(choices=[("PUBLISHED", "Published")], default="PUBLISHED", max_length=16)),
+                ("source_revision", models.CharField(max_length=64)),
+                ("source_locator", models.URLField(max_length=500)),
+                ("source_attribution", models.CharField(max_length=300)),
+                ("source_checked_at", models.DateTimeField()),
+                ("published_by", models.CharField(max_length=100)),
+                ("published_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("schedule_revision", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="publications", to="travel_windows.flightschedulerevision")),
+                ("supersedes", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="successor", to="travel_windows.flightpublication")),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(condition=Q(generation__gte=1), name="flight_publication_generation_positive"),
+                    models.CheckConstraint(condition=Q(state="PUBLISHED"), name="flight_publication_state_fixed"),
+                    models.CheckConstraint(condition=Q(source_revision__regex=r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"), name="flight_publication_source_revision"),
+                    models.CheckConstraint(condition=Q(source_locator__startswith="https://"), name="flight_publication_locator_https"),
+                    models.CheckConstraint(condition=~Q(source_attribution=""), name="flight_publication_attribution_present"),
+                    models.CheckConstraint(condition=~Q(published_by=""), name="flight_publication_actor_present"),
+                    models.CheckConstraint(condition=Q(published_at__gte=F("source_checked_at")), name="flight_publication_after_check"),
+                    models.CheckConstraint(condition=~Q(id=F("supersedes_id")), name="flight_publication_not_self_superseding"),
+                ]
+            },
+        ),
+        migrations.CreateModel(
+            name="PublishedFlightSchedule",
+            fields=[
+                ("id", models.UUIDField(default=FLIGHT_POINTER_ID, editable=False, primary_key=True, serialize=False)),
+                ("version", models.PositiveBigIntegerField(default=0)),
+                ("updated_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("current_publication", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="current_pointer", to="travel_windows.flightpublication")),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(condition=Q(id=FLIGHT_POINTER_ID), name="flight_pointer_singleton"),
+                    models.CheckConstraint(condition=Q(current_publication__isnull=True, version=0) | Q(current_publication__isnull=False, version__gte=1), name="flight_pointer_shape"),
+                ]
+            },
+        ),
+        migrations.CreateModel(
+            name="RouteDurationRevision",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("outbound_minutes", models.PositiveSmallIntegerField()),
+                ("inbound_minutes", models.PositiveSmallIntegerField()),
+                ("reference_date", models.DateField()),
+                ("reference_locator", models.URLField(max_length=500)),
+                ("state", models.CharField(choices=[("VALIDATED", "Validated"), ("QUARANTINED", "Quarantined")], max_length=16)),
+                ("reviewed_by", models.CharField(max_length=100)),
+                ("reviewed_at", models.DateTimeField()),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                ("created_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("destination_airport", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="duration_revisions", to="travel_windows.airport")),
+                ("parse_run", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="route_duration_revisions", to="sources.parserun")),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(condition=Q(outbound_minutes__gte=1, outbound_minutes__lte=1440), name="route_duration_outbound_bounded"),
+                    models.CheckConstraint(condition=Q(inbound_minutes__gte=1, inbound_minutes__lte=1440), name="route_duration_inbound_bounded"),
+                    models.CheckConstraint(condition=Q(reference_locator__startswith="https://"), name="route_duration_locator_https"),
+                    models.CheckConstraint(condition=Q(state__in=["VALIDATED", "QUARANTINED"]), name="route_duration_state_known"),
+                    models.CheckConstraint(condition=~Q(reviewed_by=""), name="route_duration_reviewer_present"),
+                    models.CheckConstraint(condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="route_duration_typed_hash"),
+                    models.CheckConstraint(condition=Q(created_at__gte=F("reviewed_at")), name="route_duration_created_after_review"),
+                    models.UniqueConstraint(fields=("parse_run", "destination_airport"), name="route_duration_parse_airport_unique"),
+                ]
+            },
+        ),
+        migrations.CreateModel(
+            name="FlightPublicationDuration",
+            fields=[
+                ("id", models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name="ID")),
+                ("destination_airport", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="travel_windows.airport")),
+                ("duration_revision", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="travel_windows.routedurationrevision")),
+                ("publication", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="travel_windows.flightpublication")),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(fields=("publication", "destination_airport"), name="flight_publication_airport_duration_unique"),
+                    models.UniqueConstraint(fields=("publication", "duration_revision"), name="flight_publication_duration_unique"),
+                ]
+            },
+        ),
+        migrations.AddField(
+            model_name="flightpublication",
+            name="durations",
+            field=models.ManyToManyField(related_name="flight_publications", through="travel_windows.FlightPublicationDuration", to="travel_windows.routedurationrevision"),
+        ),
+        migrations.RunPython(seed_pointer, require_empty_pointer),
+        migrations.RunSQL(IMMUTABILITY_SQL, IMMUTABILITY_REVERSE_SQL),
+    ]
diff --git a/travel_windows/migrations/0003_curated_airports.py b/travel_windows/migrations/0003_curated_airports.py
new file mode 100644
index 0000000..738ac51
--- /dev/null
+++ b/travel_windows/migrations/0003_curated_airports.py
@@ -0,0 +1,96 @@
+import uuid
+
+from django.db import migrations
+
+
+TIMEZONE_EVIDENCE_LOCATOR = "https://www.iana.org/time-zones"
+COUNTRY_IDS = {
+    "JP": uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9"),
+    "TW": uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"),
+    "HK": uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"),
+    "MO": uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"),
+    "VN": uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"),
+    "TH": uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"),
+}
+COUNTRIES = (
+    (COUNTRY_IDS["JP"], "JP", "일본", "Japan"),
+    (COUNTRY_IDS["TW"], "TW", "대만", "Taiwan"),
+    (COUNTRY_IDS["HK"], "HK", "홍콩", "Hong Kong"),
+    (COUNTRY_IDS["MO"], "MO", "마카오", "Macau"),
+    (COUNTRY_IDS["VN"], "VN", "베트남", "Vietnam"),
+    (COUNTRY_IDS["TH"], "TH", "태국", "Thailand"),
+)
+AIRPORTS = (
+    (uuid.UUID("cfb41e65-7299-56c6-95fb-03cb567e4c0d"), "NRT", "JP", "TYO", "도쿄", "나리타 국제공항", "Asia/Tokyo"),
+    (uuid.UUID("6f3f41d2-8dae-5ea0-89e5-a7ccfd4e3979"), "HND", "JP", "TYO", "도쿄", "하네다 공항", "Asia/Tokyo"),
+    (uuid.UUID("e9f3559f-501e-54e0-8b63-52aa6087a1ae"), "KIX", "JP", "OSA", "오사카", "간사이 국제공항", "Asia/Tokyo"),
+    (uuid.UUID("969f0f90-9f1e-5be7-89b1-4e8dde0d2271"), "FUK", "JP", "FUK", "후쿠오카", "후쿠오카 공항", "Asia/Tokyo"),
+    (uuid.UUID("b093de1f-a946-575d-8a05-498bdd72b26a"), "CTS", "JP", "SPK", "삿포로", "신치토세 공항", "Asia/Tokyo"),
+    (uuid.UUID("f58c5657-d3e2-55d9-803c-52d25656f58c"), "OKA", "JP", "OKA", "오키나와", "나하 공항", "Asia/Tokyo"),
+    (uuid.UUID("43824afe-8b3b-543b-8e63-233b566918cb"), "TPE", "TW", "TPE", "타이베이", "타오위안 국제공항", "Asia/Taipei"),
+    (uuid.UUID("3f0622a5-79d2-5588-8ea1-b962ed555efb"), "HKG", "HK", "HKG", "홍콩", "홍콩 국제공항", "Asia/Hong_Kong"),
+    (uuid.UUID("5f1c6261-c16c-5651-996b-a5a00cdb01ab"), "MFM", "MO", "MFM", "마카오", "마카오 국제공항", "Asia/Macau"),
+    (uuid.UUID("26c8da61-735c-509a-aece-f75e02bc13c8"), "HAN", "VN", "HAN", "하노이", "노이바이 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("d20bdbad-f0c6-51d1-97ca-4af264aa6a40"), "DAD", "VN", "DAD", "다낭", "다낭 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("8e6e8c4c-9c21-5a2f-aa75-f0b2c84f724f"), "SGN", "VN", "SGN", "호찌민", "떤선녓 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("bd0c4b4c-1423-5295-8e91-2169fcbaad5a"), "BKK", "TH", "BKK", "방콕", "수완나품 국제공항", "Asia/Bangkok"),
+)
+
+
+def seed_airports(apps, schema_editor):
+    Country = apps.get_model("countries", "Country")
+    Airport = apps.get_model("travel_windows", "Airport")
+    alias = schema_editor.connection.alias
+    for country_id, iso_alpha2, name_ko, name_en in COUNTRIES:
+        country, _ = Country.objects.using(alias).get_or_create(
+            id=country_id,
+            defaults={
+                "iso_alpha2": iso_alpha2,
+                "name_ko": name_ko,
+                "name_en": name_en,
+                "is_public": True,
+            },
+        )
+        if (
+            country.iso_alpha2,
+            country.name_ko,
+            country.name_en,
+            country.is_public,
+        ) != (iso_alpha2, name_ko, name_en, True):
+            raise RuntimeError(f"country identity conflicts with {iso_alpha2}")
+    for airport_id, iata, country, city_code, city_name, name, zone in AIRPORTS:
+        expected = {
+            "iata_code": iata,
+            "country_id": COUNTRY_IDS[country],
+            "city_code": city_code,
+            "city_name_ko": city_name,
+            "name_ko": name,
+            "iana_timezone": zone,
+            "timezone_evidence_locator": TIMEZONE_EVIDENCE_LOCATOR,
+            "is_public": True,
+        }
+        airport, _ = Airport.objects.using(alias).get_or_create(
+            id=airport_id,
+            defaults=expected,
+        )
+        if any(getattr(airport, field) != value for field, value in expected.items()):
+            raise RuntimeError(f"airport identity conflicts with {iata}")
+
+
+def remove_unreferenced_airports(apps, schema_editor):
+    Airport = apps.get_model("travel_windows", "Airport")
+    alias = schema_editor.connection.alias
+    expected_ids = {row[0] for row in AIRPORTS}
+    observed_ids = set(Airport.objects.using(alias).values_list("id", flat=True))
+    if observed_ids and observed_ids != expected_ids:
+        raise RuntimeError("curated airport rollback found unknown rows")
+    Airport.objects.using(alias).filter(id__in=expected_ids).delete()
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("countries", "0003_supported_country_allowlist"),
+        ("travel_windows", "0002_scheduled_flight_evidence"),
+    ]
+
+    operations = [migrations.RunPython(seed_airports, remove_unreferenced_airports)]
diff --git a/travel_windows/migrations/0004_curated_airport_constraint.py b/travel_windows/migrations/0004_curated_airport_constraint.py
new file mode 100644
index 0000000..0b25a9c
--- /dev/null
+++ b/travel_windows/migrations/0004_curated_airport_constraint.py
@@ -0,0 +1,51 @@
+from importlib import import_module
+
+from django.db import migrations, models
+from django.db.models import Q
+
+
+_catalog = import_module("travel_windows.migrations.0003_curated_airports")
+AIRPORTS = _catalog.AIRPORTS
+COUNTRY_IDS = _catalog.COUNTRY_IDS
+TIMEZONE_EVIDENCE_LOCATOR = _catalog.TIMEZONE_EVIDENCE_LOCATOR
+
+
+def _condition():
+    result = Q(pk=AIRPORTS[0][0], iata_code=AIRPORTS[0][1])
+    first = AIRPORTS[0]
+    result &= Q(
+        country_id=COUNTRY_IDS[first[2]],
+        city_code=first[3],
+        city_name_ko=first[4],
+        name_ko=first[5],
+        iana_timezone=first[6],
+        timezone_evidence_locator=TIMEZONE_EVIDENCE_LOCATOR,
+        is_public=True,
+    )
+    for airport_id, iata, country, city_code, city_name, name, zone in AIRPORTS[1:]:
+        result |= Q(
+            pk=airport_id,
+            iata_code=iata,
+            country_id=COUNTRY_IDS[country],
+            city_code=city_code,
+            city_name_ko=city_name,
+            name_ko=name,
+            iana_timezone=zone,
+            timezone_evidence_locator=TIMEZONE_EVIDENCE_LOCATOR,
+            is_public=True,
+        )
+    return result
+
+
+class Migration(migrations.Migration):
+    dependencies = [("travel_windows", "0003_curated_airports")]
+
+    operations = [
+        migrations.AddConstraint(
+            model_name="airport",
+            constraint=models.CheckConstraint(
+                condition=_condition(),
+                name="airport_curated_exact_allowlist",
+            ),
+        ),
+    ]
diff --git a/travel_windows/models.py b/travel_windows/models.py
index 381a4f2..cf82211 100644
--- a/travel_windows/models.py
+++ b/travel_windows/models.py
@@ -1,7 +1,55 @@
 import uuid
 
 from django.db import models
-from django.db.models import Q
+from django.db.models import F, Q
+from django.utils import timezone
+
+from countries.models import SUPPORTED_COUNTRY_IDS
+
+
+FLIGHT_POINTER_ID = uuid.UUID("bd18f3e9-7928-5b79-8a68-4bdcfdd329fd")
+TIMEZONE_EVIDENCE_LOCATOR = "https://www.iana.org/time-zones"
+CURATED_AIRPORT_ROWS = (
+    (uuid.UUID("cfb41e65-7299-56c6-95fb-03cb567e4c0d"), "NRT", "JP", "TYO", "도쿄", "나리타 국제공항", "Asia/Tokyo"),
+    (uuid.UUID("6f3f41d2-8dae-5ea0-89e5-a7ccfd4e3979"), "HND", "JP", "TYO", "도쿄", "하네다 공항", "Asia/Tokyo"),
+    (uuid.UUID("e9f3559f-501e-54e0-8b63-52aa6087a1ae"), "KIX", "JP", "OSA", "오사카", "간사이 국제공항", "Asia/Tokyo"),
+    (uuid.UUID("969f0f90-9f1e-5be7-89b1-4e8dde0d2271"), "FUK", "JP", "FUK", "후쿠오카", "후쿠오카 공항", "Asia/Tokyo"),
+    (uuid.UUID("b093de1f-a946-575d-8a05-498bdd72b26a"), "CTS", "JP", "SPK", "삿포로", "신치토세 공항", "Asia/Tokyo"),
+    (uuid.UUID("f58c5657-d3e2-55d9-803c-52d25656f58c"), "OKA", "JP", "OKA", "오키나와", "나하 공항", "Asia/Tokyo"),
+    (uuid.UUID("43824afe-8b3b-543b-8e63-233b566918cb"), "TPE", "TW", "TPE", "타이베이", "타오위안 국제공항", "Asia/Taipei"),
+    (uuid.UUID("3f0622a5-79d2-5588-8ea1-b962ed555efb"), "HKG", "HK", "HKG", "홍콩", "홍콩 국제공항", "Asia/Hong_Kong"),
+    (uuid.UUID("5f1c6261-c16c-5651-996b-a5a00cdb01ab"), "MFM", "MO", "MFM", "마카오", "마카오 국제공항", "Asia/Macau"),
+    (uuid.UUID("26c8da61-735c-509a-aece-f75e02bc13c8"), "HAN", "VN", "HAN", "하노이", "노이바이 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("d20bdbad-f0c6-51d1-97ca-4af264aa6a40"), "DAD", "VN", "DAD", "다낭", "다낭 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("8e6e8c4c-9c21-5a2f-aa75-f0b2c84f724f"), "SGN", "VN", "SGN", "호찌민", "떤선녓 국제공항", "Asia/Ho_Chi_Minh"),
+    (uuid.UUID("bd0c4b4c-1423-5295-8e91-2169fcbaad5a"), "BKK", "TH", "BKK", "방콕", "수완나품 국제공항", "Asia/Bangkok"),
+)
+
+
+def _curated_airport_condition() -> Q:
+    condition = Q(pk=CURATED_AIRPORT_ROWS[0][0], iata_code="NRT")
+    condition &= Q(
+        country_id=SUPPORTED_COUNTRY_IDS["JP"],
+        city_code="TYO",
+        city_name_ko="도쿄",
+        name_ko="나리타 국제공항",
+        iana_timezone="Asia/Tokyo",
+        timezone_evidence_locator=TIMEZONE_EVIDENCE_LOCATOR,
+        is_public=True,
+    )
+    for airport_id, iata, country, city_code, city_name, name, iana_timezone in CURATED_AIRPORT_ROWS[1:]:
+        condition |= Q(
+            pk=airport_id,
+            iata_code=iata,
+            country_id=SUPPORTED_COUNTRY_IDS[country],
+            city_code=city_code,
+            city_name_ko=city_name,
+            name_ko=name,
+            iana_timezone=iana_timezone,
+            timezone_evidence_locator=TIMEZONE_EVIDENCE_LOCATOR,
+            is_public=True,
+        )
+    return condition
 
 
 class Airport(models.Model):
@@ -40,7 +88,297 @@ class Airport(models.Model):
                 condition=Q(timezone_evidence_locator__startswith="https://"),
                 name="airport_timezone_evidence_https",
             ),
+            models.CheckConstraint(
+                condition=_curated_airport_condition(),
+                name="airport_curated_exact_allowlist",
+            ),
         ]
 
     def __str__(self) -> str:
         return f"{self.iata_code} — {self.city_name_ko}"
+
+
+class FlightScheduleRevision(models.Model):
+    class Completeness(models.TextChoices):
+        COMPLETE = "COMPLETE", "Complete"
+
+    class State(models.TextChoices):
+        VALIDATED = "VALIDATED", "Validated"
+        QUARANTINED = "QUARANTINED", "Quarantined"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    parse_run = models.OneToOneField(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="flight_schedule_revision",
+    )
+    source_date = models.DateField()
+    season = models.CharField(max_length=32)
+    coverage_code = models.CharField(max_length=64)
+    completeness = models.CharField(
+        max_length=16,
+        choices=Completeness.choices,
+        default=Completeness.COMPLETE,
+    )
+    state = models.CharField(max_length=16, choices=State.choices)
+    first_observed_at = models.DateTimeField()
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(condition=~Q(season=""), name="flight_revision_season_present"),
+            models.CheckConstraint(
+                condition=Q(coverage_code__regex=r"^[A-Z][A-Z0-9_]{2,63}$"),
+                name="flight_revision_coverage_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(completeness="COMPLETE"),
+                name="flight_revision_complete_only",
+            ),
+            models.CheckConstraint(
+                condition=Q(state__in=["VALIDATED", "QUARANTINED"]),
+                name="flight_revision_state_known",
+            ),
+            models.CheckConstraint(
+                condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="flight_revision_typed_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(created_at__gte=F("first_observed_at")),
+                name="flight_revision_created_after_observed",
+            ),
+        ]
+
+
+class FlightSchedule(models.Model):
+    class Direction(models.TextChoices):
+        OUTBOUND = "OUTBOUND", "ICN departure"
+        INBOUND = "INBOUND", "ICN arrival"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    revision = models.ForeignKey(
+        FlightScheduleRevision,
+        on_delete=models.PROTECT,
+        related_name="schedules",
+    )
+    direction = models.CharField(max_length=16, choices=Direction.choices)
+    carrier_code = models.CharField(max_length=3)
+    carrier_name = models.CharField(max_length=100)
+    flight_number = models.CharField(max_length=8)
+    master_flight_number = models.CharField(max_length=8)
+    destination_airport = models.ForeignKey(
+        Airport,
+        on_delete=models.PROTECT,
+        related_name="flight_schedules",
+    )
+    icn_event_time = models.TimeField()
+    valid_from = models.DateField()
+    valid_until = models.DateField()
+    weekday_mask = models.CharField(max_length=7)
+
+    class Meta:
+        ordering = (
+            "destination_airport__city_code",
+            "direction",
+            "icn_event_time",
+            "master_flight_number",
+            "flight_number",
+        )
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(direction__in=["OUTBOUND", "INBOUND"]),
+                name="flight_schedule_direction_known",
+            ),
+            models.CheckConstraint(
+                condition=Q(carrier_code__regex=r"^[A-Z0-9]{2,3}$"),
+                name="flight_schedule_carrier_format",
+            ),
+            models.CheckConstraint(condition=~Q(carrier_name=""), name="flight_schedule_carrier_present"),
+            models.CheckConstraint(
+                condition=Q(flight_number__regex=r"^[A-Z0-9]{2,3}[0-9]{1,4}[A-Z]?$"),
+                name="flight_schedule_number_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(master_flight_number__regex=r"^[A-Z0-9]{2,3}[0-9]{1,4}[A-Z]?$"),
+                name="flight_schedule_master_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(valid_until__gte=F("valid_from")),
+                name="flight_schedule_valid_range",
+            ),
+            models.CheckConstraint(
+                condition=Q(weekday_mask__regex=r"^[01]{7}$") & ~Q(weekday_mask="0000000"),
+                name="flight_schedule_weekday_mask",
+            ),
+            models.UniqueConstraint(
+                fields=(
+                    "revision",
+                    "direction",
+                    "master_flight_number",
+                    "destination_airport",
+                    "icn_event_time",
+                    "valid_from",
+                    "valid_until",
+                    "weekday_mask",
+                ),
+                name="flight_schedule_master_identity",
+            ),
+        ]
+
+
+class RouteDurationRevision(models.Model):
+    class State(models.TextChoices):
+        VALIDATED = "VALIDATED", "Validated"
+        QUARANTINED = "QUARANTINED", "Quarantined"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    parse_run = models.ForeignKey(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="route_duration_revisions",
+    )
+    destination_airport = models.ForeignKey(
+        Airport,
+        on_delete=models.PROTECT,
+        related_name="duration_revisions",
+    )
+    outbound_minutes = models.PositiveSmallIntegerField()
+    inbound_minutes = models.PositiveSmallIntegerField()
+    reference_date = models.DateField()
+    reference_locator = models.URLField(max_length=500)
+    state = models.CharField(max_length=16, choices=State.choices)
+    reviewed_by = models.CharField(max_length=100)
+    reviewed_at = models.DateTimeField()
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(outbound_minutes__gte=1, outbound_minutes__lte=1440),
+                name="route_duration_outbound_bounded",
+            ),
+            models.CheckConstraint(
+                condition=Q(inbound_minutes__gte=1, inbound_minutes__lte=1440),
+                name="route_duration_inbound_bounded",
+            ),
+            models.CheckConstraint(
+                condition=Q(reference_locator__startswith="https://"),
+                name="route_duration_locator_https",
+            ),
+            models.CheckConstraint(
+                condition=Q(state__in=["VALIDATED", "QUARANTINED"]),
+                name="route_duration_state_known",
+            ),
+            models.CheckConstraint(condition=~Q(reviewed_by=""), name="route_duration_reviewer_present"),
+            models.CheckConstraint(
+                condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="route_duration_typed_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(created_at__gte=F("reviewed_at")),
+                name="route_duration_created_after_review",
+            ),
+            models.UniqueConstraint(
+                fields=("parse_run", "destination_airport"),
+                name="route_duration_parse_airport_unique",
+            ),
+        ]
+
+
+class FlightPublication(models.Model):
+    class State(models.TextChoices):
+        PUBLISHED = "PUBLISHED", "Published"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    schedule_revision = models.ForeignKey(
+        FlightScheduleRevision,
+        on_delete=models.PROTECT,
+        related_name="publications",
+    )
+    generation = models.PositiveBigIntegerField(unique=True)
+    state = models.CharField(
+        max_length=16,
+        choices=State.choices,
+        default=State.PUBLISHED,
+    )
+    source_revision = models.CharField(max_length=64)
+    source_locator = models.URLField(max_length=500)
+    source_attribution = models.CharField(max_length=300)
+    source_checked_at = models.DateTimeField()
+    published_by = models.CharField(max_length=100)
+    published_at = models.DateTimeField(default=timezone.now)
+    supersedes = models.OneToOneField(
+        "self",
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="successor",
+    )
+    durations = models.ManyToManyField(
+        RouteDurationRevision,
+        through="FlightPublicationDuration",
+        related_name="flight_publications",
+    )
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(condition=Q(generation__gte=1), name="flight_publication_generation_positive"),
+            models.CheckConstraint(condition=Q(state="PUBLISHED"), name="flight_publication_state_fixed"),
+            models.CheckConstraint(
+                condition=Q(source_revision__regex=r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"),
+                name="flight_publication_source_revision",
+            ),
+            models.CheckConstraint(condition=Q(source_locator__startswith="https://"), name="flight_publication_locator_https"),
+            models.CheckConstraint(condition=~Q(source_attribution=""), name="flight_publication_attribution_present"),
+            models.CheckConstraint(condition=~Q(published_by=""), name="flight_publication_actor_present"),
+            models.CheckConstraint(
+                condition=Q(published_at__gte=F("source_checked_at")),
+                name="flight_publication_after_check",
+            ),
+            models.CheckConstraint(condition=~Q(id=F("supersedes_id")), name="flight_publication_not_self_superseding"),
+        ]
+
+
+class FlightPublicationDuration(models.Model):
+    publication = models.ForeignKey(FlightPublication, on_delete=models.PROTECT)
+    duration_revision = models.ForeignKey(RouteDurationRevision, on_delete=models.PROTECT)
+    destination_airport = models.ForeignKey(Airport, on_delete=models.PROTECT)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("publication", "destination_airport"),
+                name="flight_publication_airport_duration_unique",
+            ),
+            models.UniqueConstraint(
+                fields=("publication", "duration_revision"),
+                name="flight_publication_duration_unique",
+            ),
+        ]
+
+
+class PublishedFlightSchedule(models.Model):
+    id = models.UUIDField(primary_key=True, default=FLIGHT_POINTER_ID, editable=False)
+    current_publication = models.OneToOneField(
+        FlightPublication,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="current_pointer",
+    )
+    version = models.PositiveBigIntegerField(default=0)
+    updated_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(condition=Q(id=FLIGHT_POINTER_ID), name="flight_pointer_singleton"),
+            models.CheckConstraint(
+                condition=(
+                    Q(current_publication__isnull=True, version=0)
+                    | Q(current_publication__isnull=False, version__gte=1)
+                ),
+                name="flight_pointer_shape",
+            ),
+        ]
diff --git a/travel_windows/parser.py b/travel_windows/parser.py
new file mode 100644
index 0000000..b52c533
--- /dev/null
+++ b/travel_windows/parser.py
@@ -0,0 +1,472 @@
+from __future__ import annotations
+
+import csv
+import io
+import json
+import re
+from dataclasses import dataclass
+from datetime import date, time
+from typing import Any
+
+from .contracts import (
+    DURATION_SCHEMA_FINGERPRINT_SHA256,
+    SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+)
+
+
+MAX_SCHEDULE_BYTES = 1_048_576
+MAX_DURATION_BYTES = 131_072
+_ROOT_KEYS = {"source_date", "season", "coverage_complete", "flights"}
+_FLIGHT_KEYS = {
+    "direction",
+    "carrier_code",
+    "carrier_name",
+    "flight_number",
+    "master_flight_number",
+    "destination_iata",
+    "icn_event_time",
+    "valid_from",
+    "valid_until",
+    "weekdays",
+}
+_DURATION_HEADERS = (
+    "destination_iata",
+    "outbound_minutes",
+    "inbound_minutes",
+    "reference_date",
+    "reference_locator",
+)
+_IATA = re.compile(r"^[A-Z]{3}$")
+_FLIGHT_NUMBER = re.compile(r"^[A-Z0-9]{2,3}[0-9]{1,4}[A-Z]?$" )
+_OFFICIAL_ROOT_KEYS = {"response"}
+_OFFICIAL_RESPONSE_KEYS = {"header", "body"}
+_OFFICIAL_HEADER_KEYS = {"resultCode", "resultMsg"}
+_OFFICIAL_BODY_KEYS = {"items", "pageNo", "numOfRows", "totalCount"}
+_OFFICIAL_ITEMS_KEYS = {"item"}
+_OFFICIAL_ITEM_KEYS = {
+    "codeshare",
+    "masterFlightId",
+    "flightId",
+    "st",
+    "season",
+    "firstdate",
+    "lastdate",
+    "ynMon",
+    "ynTue",
+    "ynWed",
+    "ynThu",
+    "ynFri",
+    "ynSat",
+    "ynSun",
+    "terminalId",
+    "airline",
+    "airlineCode",
+    "airport",
+    "airportCode",
+    "tmp1",
+    "tmp2",
+}
+_OFFICIAL_REQUIRED_ITEM_KEYS = {
+    "flightId",
+    "st",
+    "season",
+    "firstdate",
+    "lastdate",
+    "ynMon",
+    "ynTue",
+    "ynWed",
+    "ynThu",
+    "ynFri",
+    "ynSat",
+    "ynSun",
+    "airline",
+    "airlineCode",
+    "airport",
+    "airportCode",
+}
+_WEEKDAY_KEYS = ("ynMon", "ynTue", "ynWed", "ynThu", "ynFri", "ynSat", "ynSun")
+
+
+class _DuplicateKey(ValueError):
+    pass
+
+
+def _unique_object(pairs: list[tuple[str, Any]]) -> dict[str, Any]:
+    result: dict[str, Any] = {}
+    for key, value in pairs:
+        if key in result:
+            raise _DuplicateKey(key)
+        result[key] = value
+    return result
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedFlightSchedule:
+    direction: str
+    carrier_code: str
+    carrier_name: str
+    flight_number: str
+    master_flight_number: str
+    destination_iata: str
+    icn_event_time: time
+    valid_from: date
+    valid_until: date
+    weekday_mask: str
+
+
+@dataclass(frozen=True, slots=True)
+class ScheduleParseSuccess:
+    source_date: date
+    season: str
+    flights: tuple[ParsedFlightSchedule, ...]
+    observed_schema_fingerprint_sha256: str = SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedRouteDuration:
+    destination_iata: str
+    outbound_minutes: int
+    inbound_minutes: int
+    reference_date: date
+    reference_locator: str
+
+
+@dataclass(frozen=True, slots=True)
+class DurationParseSuccess:
+    routes: tuple[ParsedRouteDuration, ...]
+    observed_schema_fingerprint_sha256: str = DURATION_SCHEMA_FINGERPRINT_SHA256
+
+
+@dataclass(frozen=True, slots=True)
+class AviationParseFailure:
+    failure_code: str
+    observed_schema_fingerprint_sha256: str = ""
+
+
+def _date(value: object) -> date:
+    if type(value) is not str:
+        raise ValueError
+    return date.fromisoformat(value)
+
+
+def _time(value: object) -> time:
+    if type(value) is not str or not re.fullmatch(r"[0-2][0-9]:[0-5][0-9]", value):
+        raise ValueError
+    parsed = time.fromisoformat(value)
+    if parsed.hour > 23:
+        raise ValueError
+    return parsed
+
+
+def parse_schedule(payload: bytes) -> ScheduleParseSuccess | AviationParseFailure:
+    if not isinstance(payload, bytes) or not 0 < len(payload) <= MAX_SCHEDULE_BYTES:
+        return AviationParseFailure("INVALID_SIZE")
+    try:
+        document = json.loads(payload.decode("utf-8"), object_pairs_hook=_unique_object)
+    except (UnicodeDecodeError, json.JSONDecodeError, _DuplicateKey):
+        return AviationParseFailure("SYNTAX_ERROR")
+    if type(document) is not dict or set(document) != _ROOT_KEYS:
+        return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+    if document["coverage_complete"] is not True or type(document["flights"]) is not list:
+        return AviationParseFailure("INCOMPLETE_COVERAGE", SCHEDULE_SCHEMA_FINGERPRINT_SHA256)
+    try:
+        source_date = _date(document["source_date"])
+        season = document["season"]
+        if type(season) is not str or not season or len(season) > 32:
+            raise ValueError
+        flights: list[ParsedFlightSchedule] = []
+        identities: set[tuple[object, ...]] = set()
+        for row in document["flights"]:
+            if type(row) is not dict or set(row) != _FLIGHT_KEYS:
+                return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+            if row["direction"] not in {"OUTBOUND", "INBOUND"}:
+                raise ValueError
+            weekdays = row["weekdays"]
+            if (
+                type(weekdays) is not list
+                or not weekdays
+                or any(type(day) is not int or day not in range(7) for day in weekdays)
+                or len(set(weekdays)) != len(weekdays)
+            ):
+                raise ValueError
+            if not _IATA.fullmatch(row["destination_iata"]):
+                raise ValueError
+            if not _FLIGHT_NUMBER.fullmatch(row["flight_number"]):
+                raise ValueError
+            if not _FLIGHT_NUMBER.fullmatch(row["master_flight_number"]):
+                raise ValueError
+            valid_from = _date(row["valid_from"])
+            valid_until = _date(row["valid_until"])
+            if valid_until < valid_from:
+                raise ValueError
+            parsed = ParsedFlightSchedule(
+                direction=row["direction"],
+                carrier_code=row["carrier_code"],
+                carrier_name=row["carrier_name"],
+                flight_number=row["flight_number"],
+                master_flight_number=row["master_flight_number"],
+                destination_iata=row["destination_iata"],
+                icn_event_time=_time(row["icn_event_time"]),
+                valid_from=valid_from,
+                valid_until=valid_until,
+                weekday_mask="".join("1" if day in weekdays else "0" for day in range(7)),
+            )
+            if not parsed.carrier_code or not parsed.carrier_name:
+                raise ValueError
+            identity = (
+                parsed.direction,
+                parsed.master_flight_number,
+                parsed.destination_iata,
+                parsed.icn_event_time,
+                parsed.valid_from,
+                parsed.valid_until,
+                parsed.weekday_mask,
+            )
+            if identity in identities:
+                return AviationParseFailure("DUPLICATE_RECORD", SCHEDULE_SCHEMA_FINGERPRINT_SHA256)
+            identities.add(identity)
+            flights.append(parsed)
+    except (TypeError, ValueError):
+        return AviationParseFailure("INVALID_VALUE", SCHEDULE_SCHEMA_FINGERPRINT_SHA256)
+    if not flights:
+        return AviationParseFailure("INCOMPLETE_COVERAGE", SCHEDULE_SCHEMA_FINGERPRINT_SHA256)
+    return ScheduleParseSuccess(source_date=source_date, season=season, flights=tuple(flights))
+
+
+def parse_route_durations(payload: bytes) -> DurationParseSuccess | AviationParseFailure:
+    if not isinstance(payload, bytes) or not 0 < len(payload) <= MAX_DURATION_BYTES:
+        return AviationParseFailure("INVALID_SIZE")
+    try:
+        reader = csv.DictReader(io.StringIO(payload.decode("utf-8", errors="strict"), newline=""))
+        if tuple(reader.fieldnames or ()) != _DURATION_HEADERS:
+            return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+        routes: list[ParsedRouteDuration] = []
+        seen: set[str] = set()
+        for row in reader:
+            if None in row or set(row) != set(_DURATION_HEADERS):
+                return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+            iata = row["destination_iata"]
+            outbound = int(row["outbound_minutes"])
+            inbound = int(row["inbound_minutes"])
+            locator = row["reference_locator"]
+            if (
+                not _IATA.fullmatch(iata)
+                or iata in seen
+                or not 1 <= outbound <= 1_440
+                or not 1 <= inbound <= 1_440
+                or not locator.startswith("https://")
+            ):
+                raise ValueError
+            seen.add(iata)
+            routes.append(
+                ParsedRouteDuration(
+                    destination_iata=iata,
+                    outbound_minutes=outbound,
+                    inbound_minutes=inbound,
+                    reference_date=_date(row["reference_date"]),
+                    reference_locator=locator,
+                )
+            )
+    except (UnicodeDecodeError, csv.Error, TypeError, ValueError):
+        return AviationParseFailure("INVALID_VALUE", DURATION_SCHEMA_FINGERPRINT_SHA256)
+    if not routes:
+        return AviationParseFailure("INCOMPLETE_COVERAGE", DURATION_SCHEMA_FINGERPRINT_SHA256)
+    return DurationParseSuccess(routes=tuple(routes))
+
+
+def _integer(value: object) -> int:
+    if type(value) is int:
+        result = value
+    elif type(value) is str and re.fullmatch(r"[0-9]+", value):
+        result = int(value)
+    else:
+        raise ValueError
+    if result < 0:
+        raise ValueError
+    return result
+
+
+def _official_date(value: object) -> str:
+    if type(value) is not str:
+        raise ValueError
+    compact = value.replace("-", "")
+    if not re.fullmatch(r"[0-9]{8}", compact):
+        raise ValueError
+    parsed = date.fromisoformat(f"{compact[:4]}-{compact[4:6]}-{compact[6:]}")
+    return parsed.isoformat()
+
+
+def _official_time(value: object) -> str:
+    if type(value) is not str:
+        raise ValueError
+    compact = value.replace(":", "")
+    if not re.fullmatch(r"[0-9]{4}", compact):
+        raise ValueError
+    return _time(f"{compact[:2]}:{compact[2:]}").strftime("%H:%M")
+
+
+def _flight_id(value: object) -> str:
+    if type(value) is not str:
+        raise ValueError
+    normalized = re.sub(r"\s+", "", value).upper()
+    if not _FLIGHT_NUMBER.fullmatch(normalized):
+        raise ValueError
+    return normalized
+
+
+def _decode_official_page(payload: bytes) -> tuple[int, int, int, list[dict[str, object]]]:
+    if not isinstance(payload, bytes) or not 0 < len(payload) <= MAX_SCHEDULE_BYTES:
+        raise ValueError
+    document = json.loads(payload.decode("utf-8"), object_pairs_hook=_unique_object)
+    if type(document) is not dict or set(document) != _OFFICIAL_ROOT_KEYS:
+        raise ValueError
+    response = document["response"]
+    if type(response) is not dict or set(response) != _OFFICIAL_RESPONSE_KEYS:
+        raise ValueError
+    header = response["header"]
+    body = response["body"]
+    if (
+        type(header) is not dict
+        or set(header) != _OFFICIAL_HEADER_KEYS
+        or header["resultCode"] not in {"00", "0"}
+        or type(header["resultMsg"]) is not str
+        or type(body) is not dict
+        or set(body) != _OFFICIAL_BODY_KEYS
+    ):
+        raise ValueError
+    items = body["items"]
+    if type(items) is not dict or set(items) != _OFFICIAL_ITEMS_KEYS:
+        raise ValueError
+    raw_items = items["item"]
+    if type(raw_items) is dict:
+        rows = [raw_items]
+    elif type(raw_items) is list:
+        rows = raw_items
+    else:
+        raise ValueError
+    if any(
+        type(row) is not dict
+        or not _OFFICIAL_REQUIRED_ITEM_KEYS.issubset(row)
+        or not set(row).issubset(_OFFICIAL_ITEM_KEYS)
+        for row in rows
+    ):
+        raise ValueError
+    return (
+        _integer(body["pageNo"]),
+        _integer(body["numOfRows"]),
+        _integer(body["totalCount"]),
+        rows,
+    )
+
+
+def _normalized_direction_pages(
+    pages: tuple[bytes, ...],
+    *,
+    direction: str,
+) -> list[dict[str, object]]:
+    if not pages:
+        raise ValueError
+    decoded = [_decode_official_page(payload) for payload in pages]
+    first_page, page_size, total, _ = decoded[0]
+    if first_page != 1 or page_size < 1 or total < 1:
+        raise ValueError
+    expected_pages = (total + page_size - 1) // page_size
+    if len(decoded) != expected_pages:
+        raise ValueError
+    all_rows: list[dict[str, object]] = []
+    for expected_page, (page_number, observed_size, observed_total, rows) in enumerate(decoded, 1):
+        if (
+            page_number != expected_page
+            or observed_size != page_size
+            or observed_total != total
+            or len(rows) > page_size
+        ):
+            raise ValueError
+        all_rows.extend(rows)
+    if len(all_rows) != total:
+        raise ValueError
+
+    normalized_by_master: dict[tuple[object, ...], tuple[bool, dict[str, object]]] = {}
+    for row in all_rows:
+        flight_number = _flight_id(row["flightId"])
+        codeshare = row.get("codeshare", "")
+        if codeshare not in {"Master", "Slave", "MASTER", "SLAVE", ""}:
+            raise ValueError
+        master_raw = row.get("masterFlightId", "")
+        if codeshare in {"Slave", "SLAVE"} and not master_raw:
+            raise ValueError
+        master_number = _flight_id(master_raw or flight_number)
+        weekdays = []
+        for index, key in enumerate(_WEEKDAY_KEYS):
+            if row[key] not in {"Y", "N"}:
+                raise ValueError
+            if row[key] == "Y":
+                weekdays.append(index)
+        normalized = {
+            "direction": direction,
+            "carrier_code": row["airlineCode"],
+            "carrier_name": row["airline"],
+            "flight_number": flight_number,
+            "master_flight_number": master_number,
+            "destination_iata": row["airportCode"],
+            "icn_event_time": _official_time(row["st"]),
+            "valid_from": _official_date(row["firstdate"]),
+            "valid_until": _official_date(row["lastdate"]),
+            "weekdays": weekdays,
+        }
+        identity = (
+            direction,
+            master_number,
+            normalized["destination_iata"],
+            normalized["icn_event_time"],
+            normalized["valid_from"],
+            normalized["valid_until"],
+            tuple(weekdays),
+        )
+        is_master = codeshare in {"Master", "MASTER"} or flight_number == master_number
+        previous = normalized_by_master.get(identity)
+        if previous is None or (is_master and not previous[0]):
+            normalized_by_master[identity] = (is_master, normalized)
+    return [
+        value[1]
+        for _, value in sorted(
+            normalized_by_master.items(),
+            key=lambda item: tuple(str(part) for part in item[0]),
+        )
+    ]
+
+
+def adapt_data_go_schedule_pages(
+    *,
+    departure_pages: tuple[bytes, ...],
+    arrival_pages: tuple[bytes, ...],
+    source_date: date,
+) -> ScheduleParseSuccess | AviationParseFailure:
+    """Normalize complete documented data.go.kr pages into strict typed rows."""
+
+    if not isinstance(source_date, date):
+        return AviationParseFailure("INVALID_VALUE", SCHEDULE_SCHEMA_FINGERPRINT_SHA256)
+    try:
+        flights = [
+            *_normalized_direction_pages(departure_pages, direction="OUTBOUND"),
+            *_normalized_direction_pages(arrival_pages, direction="INBOUND"),
+        ]
+        provider_seasons = set()
+        for payloads in (departure_pages, arrival_pages):
+            for payload in payloads:
+                _, _, _, rows = _decode_official_page(payload)
+                provider_seasons.update(row["season"] for row in rows)
+        if len(provider_seasons) != 1:
+            raise ValueError
+        normalized = {
+            "source_date": source_date.isoformat(),
+            "season": provider_seasons.pop(),
+            "coverage_complete": True,
+            "flights": flights,
+        }
+        result = parse_schedule(
+            json.dumps(normalized, ensure_ascii=False, separators=(",", ":")).encode("utf-8")
+        )
+        return result
+    except (UnicodeDecodeError, json.JSONDecodeError, _DuplicateKey, TypeError, ValueError):
+        return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
new file mode 100644
index 0000000..18465cf
--- /dev/null
+++ b/travel_windows/publication.py
@@ -0,0 +1,295 @@
+"""Atomic publication of validated scheduled-flight evidence.
+
+The public request path never calls this module.  An offline ingestion/review
+job supplies already parsed, synthetic-safe values and this service advances
+the single current pointer only after every referenced row is durable.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import json
+from dataclasses import dataclass
+from datetime import datetime
+
+from django.db import connection, transaction
+from django.utils import timezone
+
+from sources.models import ParseRun, SourceConfiguration, SourceRightsDecision
+
+from .contracts import (
+    DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_CODE,
+    DURATION_SOURCE_REVISION,
+    SCHEDULE_ATTRIBUTION,
+    SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    SCHEDULE_SOURCE_CODE,
+    SCHEDULE_SOURCE_LOCATOR,
+    SCHEDULE_SOURCE_REVISION,
+)
+from .models import (
+    FLIGHT_POINTER_ID,
+    Airport,
+    FlightPublication,
+    FlightPublicationDuration,
+    FlightSchedule,
+    FlightScheduleRevision,
+    PublishedFlightSchedule,
+    RouteDurationRevision,
+)
+from .parser import DurationParseSuccess, ScheduleParseSuccess
+
+
+PUBLICATION_LOCK_NAMESPACE = 1_414_678_614
+PUBLICATION_LOCK_KEY = 20_260_831
+
+
+class FlightPublicationCode:
+    PUBLISHED = "PUBLISHED"
+    INVALID_EVIDENCE = "INVALID_EVIDENCE"
+    SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
+
+
+@dataclass(frozen=True, slots=True)
+class FlightPublicationOutcome:
+    code: str
+    publication_id: str | None = None
+    generation: int | None = None
+
+    @property
+    def succeeded(self) -> bool:
+        return self.code == FlightPublicationCode.PUBLISHED
+
+
+class _PublicationRejected(Exception):
+    def __init__(self, code: str):
+        self.code = code
+        super().__init__(code)
+
+
+def _fingerprint(value: object) -> str:
+    encoded = json.dumps(
+        value,
+        ensure_ascii=False,
+        separators=(",", ":"),
+        sort_keys=True,
+        default=str,
+    ).encode("utf-8")
+    return hashlib.sha256(encoded).hexdigest()
+
+
+def _lock_registry() -> None:
+    if connection.vendor != "postgresql":
+        raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT pg_advisory_xact_lock(%s, %s)",
+            [PUBLICATION_LOCK_NAMESPACE, PUBLICATION_LOCK_KEY],
+        )
+
+
+def _approved_run(
+    run: ParseRun,
+    *,
+    source_code: str,
+    source_revision: str,
+    parser_name: str,
+    contract_fingerprint: str,
+    schema_fingerprint: str,
+) -> ParseRun:
+    try:
+        locked = (
+            ParseRun.objects.select_for_update()
+            .select_related("artifact__source")
+            .get(pk=run.pk)
+        )
+    except (ParseRun.DoesNotExist, AttributeError, TypeError):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE) from None
+    source = locked.artifact.source
+    if (
+        locked.outcome != ParseRun.Outcome.VALIDATED
+        or locked.parser_name != parser_name
+        or locked.parser_version != ParseRun.ParserVersion.V1
+        or locked.parser_contract_fingerprint_sha256 != contract_fingerprint
+        or locked.expected_schema_fingerprint_sha256 != schema_fingerprint
+        or locked.observed_schema_fingerprint_sha256 != schema_fingerprint
+        or source.code != source_code
+        or source.revision != source_revision
+        or source.module != SourceConfiguration.Module.AVIATION
+        or source.state != SourceConfiguration.State.ACTIVE
+        or not source.enabled
+    ):
+        raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+    approved = SourceRightsDecision.objects.select_for_update().filter(
+        source=source,
+        source_revision=source_revision,
+        decision=SourceRightsDecision.Decision.APPROVED,
+        access_allowed=True,
+        typed_field_storage_allowed=True,
+        typed_republication_allowed=True,
+        raw_body_storage_allowed=False,
+        contract_fingerprint_sha256=contract_fingerprint,
+    )
+    if approved.count() != 1:
+        raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+    return locked
+
+
+def _aware(value: datetime) -> bool:
+    return isinstance(value, datetime) and timezone.is_aware(value)
+
+
+def publish_flight_evidence(
+    *,
+    schedule_run: ParseRun,
+    schedule: ScheduleParseSuccess,
+    duration_run: ParseRun,
+    durations: DurationParseSuccess,
+    published_by: str,
+    source_checked_at: datetime,
+) -> FlightPublicationOutcome:
+    """Publish one complete schedule and its reviewed route durations.
+
+    The function is fail-closed and never retains raw response bytes.  Invalid
+    input rolls the transaction back and returns only a closed outcome code.
+    """
+
+    if (
+        not isinstance(schedule, ScheduleParseSuccess)
+        or not isinstance(durations, DurationParseSuccess)
+        or not isinstance(published_by, str)
+        or not published_by.strip()
+        or len(published_by) > 100
+        or not _aware(source_checked_at)
+    ):
+        return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
+    try:
+        with transaction.atomic(durable=True):
+            _lock_registry()
+            locked_schedule_run = _approved_run(
+                schedule_run,
+                source_code=SCHEDULE_SOURCE_CODE,
+                source_revision=SCHEDULE_SOURCE_REVISION,
+                parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+                contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+                schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            )
+            locked_duration_run = _approved_run(
+                duration_run,
+                source_code=DURATION_SOURCE_CODE,
+                source_revision=DURATION_SOURCE_REVISION,
+                parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
+                contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
+                schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
+            )
+            if (
+                schedule.observed_schema_fingerprint_sha256
+                != SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+                or durations.observed_schema_fingerprint_sha256
+                != DURATION_SCHEMA_FINGERPRINT_SHA256
+            ):
+                raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+
+            iata_codes = {row.destination_iata for row in schedule.flights}
+            duration_codes = {row.destination_iata for row in durations.routes}
+            if not iata_codes or iata_codes != duration_codes:
+                raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+            airports = {
+                airport.iata_code: airport
+                for airport in Airport.objects.select_for_update().filter(
+                    iata_code__in=iata_codes,
+                    is_public=True,
+                )
+            }
+            if set(airports) != iata_codes:
+                raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+
+            revision = FlightScheduleRevision.objects.create(
+                parse_run=locked_schedule_run,
+                source_date=schedule.source_date,
+                season=schedule.season,
+                coverage_code="ICN_DIRECT_PUBLIC_V1",
+                completeness=FlightScheduleRevision.Completeness.COMPLETE,
+                state=FlightScheduleRevision.State.VALIDATED,
+                first_observed_at=locked_schedule_run.completed_at,
+                typed_fingerprint_sha256=_fingerprint(schedule),
+            )
+            FlightSchedule.objects.bulk_create(
+                [
+                    FlightSchedule(
+                        revision=revision,
+                        direction=row.direction,
+                        carrier_code=row.carrier_code,
+                        carrier_name=row.carrier_name,
+                        flight_number=row.flight_number,
+                        master_flight_number=row.master_flight_number,
+                        destination_airport=airports[row.destination_iata],
+                        icn_event_time=row.icn_event_time,
+                        valid_from=row.valid_from,
+                        valid_until=row.valid_until,
+                        weekday_mask=row.weekday_mask,
+                    )
+                    for row in schedule.flights
+                ]
+            )
+            duration_revisions = []
+            for row in durations.routes:
+                duration_revisions.append(
+                    RouteDurationRevision.objects.create(
+                        parse_run=locked_duration_run,
+                        destination_airport=airports[row.destination_iata],
+                        outbound_minutes=row.outbound_minutes,
+                        inbound_minutes=row.inbound_minutes,
+                        reference_date=row.reference_date,
+                        reference_locator=row.reference_locator,
+                        state=RouteDurationRevision.State.VALIDATED,
+                        reviewed_by=published_by.strip(),
+                        reviewed_at=source_checked_at,
+                        typed_fingerprint_sha256=_fingerprint(row),
+                    )
+                )
+
+            pointer, _ = (
+                PublishedFlightSchedule.objects.select_for_update().get_or_create(
+                    pk=FLIGHT_POINTER_ID,
+                    defaults={"version": 0},
+                )
+            )
+            generation = pointer.version + 1
+            publication = FlightPublication.objects.create(
+                schedule_revision=revision,
+                generation=generation,
+                source_revision=SCHEDULE_SOURCE_REVISION,
+                source_locator=SCHEDULE_SOURCE_LOCATOR,
+                source_attribution=SCHEDULE_ATTRIBUTION,
+                source_checked_at=source_checked_at,
+                published_by=published_by.strip(),
+                supersedes=pointer.current_publication,
+            )
+            FlightPublicationDuration.objects.bulk_create(
+                [
+                    FlightPublicationDuration(
+                        publication=publication,
+                        duration_revision=duration,
+                        destination_airport=duration.destination_airport,
+                    )
+                    for duration in duration_revisions
+                ]
+            )
+            pointer.current_publication = publication
+            pointer.version = generation
+            pointer.updated_at = timezone.now()
+            pointer.save(
+                update_fields=("current_publication", "version", "updated_at")
+            )
+        return FlightPublicationOutcome(
+            FlightPublicationCode.PUBLISHED,
+            str(publication.pk),
+            generation,
+        )
+    except _PublicationRejected as exc:
+        return FlightPublicationOutcome(exc.code)
+    except Exception:
+        return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
diff --git a/travel_windows/tests/test_domain.py b/travel_windows/tests/test_domain.py
index 7df1441..eae7bef 100644
--- a/travel_windows/tests/test_domain.py
+++ b/travel_windows/tests/test_domain.py
@@ -2,7 +2,7 @@ from django.db import IntegrityError, transaction
 from django.test import TestCase
 
 from countries.models import Country, JP_COUNTRY_ID
-from travel_windows.models import Airport
+from travel_windows.models import Airport, CURATED_AIRPORT_ROWS
 
 
 class TravelDestinationDomainTests(TestCase):
@@ -16,15 +16,7 @@ class TravelDestinationDomainTests(TestCase):
             Country.objects.create(iso_alpha2="tw", name_ko="대만", name_en="Taiwan")
 
     def test_airport_keeps_country_city_timezone_and_evidence(self):
-        airport = Airport.objects.create(
-            iata_code="NRT",
-            country=Country.objects.get(pk=JP_COUNTRY_ID),
-            city_code="TYO",
-            city_name_ko="도쿄",
-            name_ko="나리타 국제공항",
-            iana_timezone="Asia/Tokyo",
-            timezone_evidence_locator="https://example.invalid/timezones/NRT",
-        )
+        airport = Airport.objects.get(iata_code="NRT")
         self.assertEqual(airport.country.iso_alpha2, "JP")
         self.assertEqual(airport.iana_timezone, "Asia/Tokyo")
 
@@ -40,6 +32,25 @@ class TravelDestinationDomainTests(TestCase):
                     country=country,
                     city_name_ko="도쿄",
                     name_ko="나리타 국제공항",
-                    timezone_evidence_locator="https://example.invalid/timezones/NRT",
+                    timezone_evidence_locator="https://www.iana.org/time-zones",
                     **values,
                 )
+
+    def test_country_and_airport_catalogs_reject_unknown_rows(self):
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            Country.objects.create(
+                iso_alpha2="US",
+                name_ko="미국",
+                name_en="United States",
+            )
+        self.assertEqual(Airport.objects.count(), len(CURATED_AIRPORT_ROWS))
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            Airport.objects.create(
+                iata_code="AAA",
+                country=Country.objects.get(pk=JP_COUNTRY_ID),
+                city_code="AAA",
+                city_name_ko="알 수 없음",
+                name_ko="알 수 없는 공항",
+                iana_timezone="Asia/Tokyo",
+                timezone_evidence_locator="https://www.iana.org/time-zones",
+            )
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
new file mode 100644
index 0000000..5bfec28
--- /dev/null
+++ b/travel_windows/tests/test_source_publication.py
@@ -0,0 +1,265 @@
+import json
+from datetime import date
+
+from django.test import SimpleTestCase, TransactionTestCase
+from django.utils import timezone
+
+from countries.models import Country
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from sources.models import FetchAttempt, ParseRun, SourceArtifact
+from sources.tests.test_transport import FakeConnection, FakeResponse
+from sources.transport import (
+    fetch_data_go_schedule_pages,
+    load_aviation_secret_reference,
+)
+from travel_windows.ingestion import (
+    FlightIngestionCode,
+    ingest_and_publish_flight_evidence,
+)
+from travel_windows.models import (
+    Airport,
+    CURATED_AIRPORT_ROWS,
+    FlightSchedule,
+    PublishedFlightSchedule,
+    TIMEZONE_EVIDENCE_LOCATOR,
+)
+from travel_windows.parser import (
+    AviationParseFailure,
+    adapt_data_go_schedule_pages,
+)
+
+
+def official_row(**overrides):
+    row = {
+        "codeshare": "Master",
+        "masterFlightId": "KE701",
+        "flightId": "KE701",
+        "st": "0910",
+        "season": "S26",
+        "firstdate": "20260329",
+        "lastdate": "20261024",
+        "ynMon": "Y",
+        "ynTue": "Y",
+        "ynWed": "Y",
+        "ynThu": "Y",
+        "ynFri": "Y",
+        "ynSat": "Y",
+        "ynSun": "Y",
+        "terminalId": "P02",
+        "airline": "대한항공",
+        "airlineCode": "KE",
+        "airport": "도쿄 나리타",
+        "airportCode": "NRT",
+        "tmp1": "",
+        "tmp2": "",
+    }
+    row.update(overrides)
+    return row
+
+
+def official_page(rows, *, page=1, page_size=None, total=None):
+    page_size = len(rows) if page_size is None else page_size
+    total = len(rows) if total is None else total
+    return json.dumps(
+        {
+            "response": {
+                "header": {"resultCode": "00", "resultMsg": "NORMAL SERVICE."},
+                "body": {
+                    "items": {"item": rows},
+                    "pageNo": str(page),
+                    "numOfRows": str(page_size),
+                    "totalCount": str(total),
+                },
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+class DataGoScheduleAdapterTests(SimpleTestCase):
+    def test_complete_pages_normalize_and_dedupe_codeshare_by_master(self):
+        departure = official_page(
+            [
+                official_row(
+                    codeshare="Slave",
+                    flightId="OZ 9701",
+                    masterFlightId="KE 701",
+                    airline="아시아나항공",
+                    airlineCode="OZ",
+                ),
+                official_row(),
+            ]
+        )
+        arrival = official_page(
+            [official_row(flightId="KE704", masterFlightId="KE704", st="2015")]
+        )
+        result = adapt_data_go_schedule_pages(
+            departure_pages=(departure,),
+            arrival_pages=(arrival,),
+            source_date=date(2026, 8, 31),
+        )
+        self.assertNotIsInstance(result, AviationParseFailure)
+        self.assertEqual(len(result.flights), 2)
+        outbound = next(row for row in result.flights if row.direction == "OUTBOUND")
+        self.assertEqual(outbound.flight_number, "KE701")
+        self.assertEqual(outbound.master_flight_number, "KE701")
+
+    def test_missing_page_and_unknown_schema_are_quarantined(self):
+        first = official_page([official_row()], page_size=1, total=2)
+        arrival = official_page(
+            [official_row(flightId="KE704", masterFlightId="KE704")]
+        )
+        incomplete = adapt_data_go_schedule_pages(
+            departure_pages=(first,),
+            arrival_pages=(arrival,),
+            source_date=date(2026, 8, 31),
+        )
+        self.assertIsInstance(incomplete, AviationParseFailure)
+
+        document = json.loads(arrival)
+        document["response"]["body"]["items"]["item"][0]["unknown"] = "drift"
+        drift = adapt_data_go_schedule_pages(
+            departure_pages=(official_page([official_row()]),),
+            arrival_pages=(json.dumps(document).encode(),),
+            source_date=date(2026, 8, 31),
+        )
+        self.assertIsInstance(drift, AviationParseFailure)
+
+    def test_canonical_secret_name_precedes_legacy_without_exposure(self):
+        environment = {
+            "DATA_GO_KR_SERVICE_KEY": "canonical-value",
+            "MOFA_TRAVEL_ALARM_SERVICE_KEY": "legacy-value",
+        }
+        self.assertEqual(
+            load_aviation_secret_reference(environment),
+            "canonical-value",
+        )
+        self.assertEqual(
+            load_aviation_secret_reference(
+                {"MOFA_TRAVEL_ALARM_SERVICE_KEY": "legacy-value"}
+            ),
+            "legacy-value",
+        )
+
+    def test_transport_fetches_both_official_pages_with_bounded_redaction(self):
+        departures = FakeConnection(FakeResponse(200, official_page([official_row()])))
+        arrivals = FakeConnection(
+            FakeResponse(
+                200,
+                official_page(
+                    [official_row(flightId="KE704", masterFlightId="KE704")]
+                ),
+            )
+        )
+        connections = [departures, arrivals]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        secret = "synthetic-sensitive-key"
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value=secret,
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+        )
+        self.assertTrue(result.succeeded)
+        self.assertEqual(len(result.departure_pages), 1)
+        self.assertEqual(len(result.arrival_pages), 1)
+        self.assertNotIn(secret, repr(result))
+        self.assertTrue(departures.closed)
+        self.assertTrue(arrivals.closed)
+
+
+class FlightEvidencePublicationTests(TransactionTestCase):
+    def setUp(self):
+        register_approved_sources(apply=True)
+        row = CURATED_AIRPORT_ROWS[0]
+        Airport.objects.get_or_create(
+            id=row[0],
+            defaults={
+                "iata_code": row[1],
+                "country": Country.objects.get(iso_alpha2=row[2]),
+                "city_code": row[3],
+                "city_name_ko": row[4],
+                "name_ko": row[5],
+                "iana_timezone": row[6],
+                "timezone_evidence_locator": TIMEZONE_EVIDENCE_LOCATOR,
+                "is_public": True,
+            },
+        )
+
+    def test_receipts_typed_rows_and_current_pointer_publish_without_raw_body(self):
+        departure = official_page([official_row()])
+        arrival = official_page(
+            [official_row(flightId="KE704", masterFlightId="KE704", st="2015")]
+        )
+        duration = (
+            b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+            b"reference_locator\r\nNRT,150,165,2026-08-01,"
+            b"https://www.ulip.go.kr/\r\n"
+        )
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=(departure,),
+            arrival_pages=(arrival,),
+            duration_csv=duration,
+            source_date=date(2026, 8, 31),
+            published_by="synthetic-reviewer",
+            source_checked_at=timezone.now(),
+        )
+        self.assertEqual(outcome.code, FlightIngestionCode.PUBLISHED)
+        pointer = PublishedFlightSchedule.objects.select_related(
+            "current_publication"
+        ).get()
+        self.assertEqual(pointer.version, 1)
+        self.assertEqual(pointer.current_publication.generation, 1)
+        self.assertEqual(FlightSchedule.objects.count(), 2)
+        self.assertEqual(FetchAttempt.objects.count(), 2)
+        self.assertEqual(SourceArtifact.objects.count(), 2)
+        self.assertEqual(ParseRun.objects.count(), 2)
+        model_fields = {
+            field.name
+            for model in (FetchAttempt, SourceArtifact, ParseRun)
+            for field in model._meta.fields
+        }
+        self.assertNotIn("raw_body", model_fields)
+        self.assertNotIn("response_body", model_fields)
+
+    def test_unknown_airport_schedule_is_quarantined_without_pointer_change(self):
+        departure = official_page(
+            [official_row(airportCode="AAA", airport="알 수 없는 공항")]
+        )
+        arrival = official_page(
+            [
+                official_row(
+                    airportCode="AAA",
+                    airport="알 수 없는 공항",
+                    flightId="KE704",
+                    masterFlightId="KE704",
+                )
+            ]
+        )
+        duration = (
+            b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+            b"reference_locator\r\nAAA,150,165,2026-08-01,"
+            b"https://www.ulip.go.kr/\r\n"
+        )
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=(departure,),
+            arrival_pages=(arrival,),
+            duration_csv=duration,
+            source_date=date(2026, 8, 31),
+            published_by="synthetic-reviewer",
+            source_checked_at=timezone.now(),
+        )
+        self.assertEqual(outcome.code, FlightIngestionCode.PARSE_QUARANTINED)
+        self.assertFalse(
+            PublishedFlightSchedule.objects.filter(
+                current_publication__isnull=False
+            ).exists()
+        )


