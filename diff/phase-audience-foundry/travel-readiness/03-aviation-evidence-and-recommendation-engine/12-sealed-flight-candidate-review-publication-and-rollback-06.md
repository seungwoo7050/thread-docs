## `fix(db): close flight rollback and seal races`

diff --git a/travel_windows/migrations/0007_flight_review_lifecycle.py b/travel_windows/migrations/0007_flight_review_lifecycle.py
index 0de1255..068c559 100644
--- a/travel_windows/migrations/0007_flight_review_lifecycle.py
+++ b/travel_windows/migrations/0007_flight_review_lifecycle.py
@@ -9,13 +9,16 @@ from django.db import migrations, models
 
 BACKFILL_SOURCE_CHECKED_AT_SQL = """
 UPDATE travel_windows_flightschedulerevision AS revision
-   SET source_checked_at = COALESCE(
-       (
-           SELECT MIN(publication.source_checked_at)
-             FROM travel_windows_flightpublication AS publication
-            WHERE publication.schedule_revision_id = revision.id
-       ),
-       GREATEST(revision.first_observed_at, revision.created_at)
+   SET source_checked_at = GREATEST(
+       revision.first_observed_at,
+       COALESCE(
+           (
+               SELECT MIN(publication.source_checked_at)
+                 FROM travel_windows_flightpublication AS publication
+                WHERE publication.schedule_revision_id = revision.id
+           ),
+           revision.created_at
+       )
    );
 """
 
@@ -156,25 +159,31 @@ DECLARE
     review_actor_principal text;
     review_schedule_revision_id uuid;
     review_reason text;
+    review_decided_at timestamptz;
     publication_review_id uuid;
     publication_schedule_revision_id uuid;
+    publication_published_at timestamptz;
 BEGIN
     SELECT review.actor_id,
            review.actor_principal,
            review.schedule_revision_id,
-           review.reason_code
+           review.reason_code,
+           review.decided_at
       INTO review_actor_id,
            review_actor_principal,
            review_schedule_revision_id,
-           review_reason
+           review_reason,
+           review_decided_at
       FROM travel_windows_flightreviewdecision AS review
      WHERE review.id = NEW.review_decision_id;
 
     IF NEW.publication_id IS NOT NULL THEN
         SELECT publication.review_decision_id,
-               publication.schedule_revision_id
+               publication.schedule_revision_id,
+               publication.published_at
           INTO publication_review_id,
-               publication_schedule_revision_id
+               publication_schedule_revision_id,
+               publication_published_at
           FROM travel_windows_flightpublication AS publication
          WHERE publication.id = NEW.publication_id;
     END IF;
@@ -182,6 +191,7 @@ BEGIN
     IF review_actor_id IS DISTINCT FROM NEW.actor_id
        OR review_actor_principal IS DISTINCT FROM NEW.actor_principal
        OR review_schedule_revision_id IS DISTINCT FROM NEW.schedule_revision_id
+       OR NEW.occurred_at < review_decided_at
        OR (NEW.action = 'REVIEW_REJECT' AND review_reason <> 'REJECT_FLIGHT_CANDIDATE')
        OR (NEW.action = 'PUBLISH' AND review_reason <> 'APPROVE_FLIGHT_CANDIDATE')
        OR (NEW.action = 'ROLLBACK' AND review_reason <> 'ROLLBACK_FLIGHT_PUBLICATION')
@@ -190,6 +200,7 @@ BEGIN
            AND (
                publication_review_id IS DISTINCT FROM NEW.review_decision_id
                OR publication_schedule_revision_id IS DISTINCT FROM NEW.schedule_revision_id
+               OR NEW.occurred_at < publication_published_at
            )
        ) THEN
         RAISE EXCEPTION 'flight audit does not match its reviewed action'
@@ -254,6 +265,7 @@ DECLARE
     review_rollback_target_id uuid;
     review_exists boolean := false;
     rollback_is_strict_ancestor boolean := false;
+    rollback_target_matches boolean := false;
 BEGIN
     SELECT review.schedule_revision_id,
            review.decision,
@@ -286,12 +298,27 @@ BEGIN
             SELECT 1 FROM strict_ancestors
              WHERE id = NEW.rollback_target_id
         ) INTO rollback_is_strict_ancestor;
+        SELECT EXISTS (
+            SELECT 1
+              FROM travel_windows_flightpublication AS target
+             WHERE target.id = NEW.rollback_target_id
+               AND target.schedule_revision_id = NEW.schedule_revision_id
+               AND target.source_revision = NEW.source_revision
+               AND target.source_locator = NEW.source_locator
+               AND target.source_attribution = NEW.source_attribution
+               AND target.source_checked_at = NEW.source_checked_at
+        ) INTO rollback_target_matches;
     END IF;
 
     IF NOT review_exists
        OR review_schedule_revision_id IS DISTINCT FROM NEW.schedule_revision_id
        OR review_decision <> 'APPROVED'
        OR review_actor_principal IS DISTINCT FROM NEW.published_by
+       OR NEW.published_at < (
+           SELECT review.decided_at
+             FROM travel_windows_flightreviewdecision AS review
+            WHERE review.id = NEW.review_decision_id
+       )
        OR (
            NEW.rollback_target_id IS NULL
            AND (
@@ -305,6 +332,7 @@ BEGIN
                review_reason <> 'ROLLBACK_FLIGHT_PUBLICATION'
                OR review_rollback_target_id IS DISTINCT FROM NEW.rollback_target_id
                OR NOT rollback_is_strict_ancestor
+               OR NOT rollback_target_matches
            )
        ) THEN
         RAISE EXCEPTION 'flight publication requires an authorized review'
@@ -357,6 +385,8 @@ DECLARE
     candidate_duration_count bigint;
     duration_closure_matches boolean;
     rollback_is_strict_ancestor boolean := false;
+    rollback_target_matches boolean := false;
+    rollback_duration_matches boolean := false;
 BEGIN
     IF TG_OP = 'DELETE' THEN
         RAISE EXCEPTION 'flight current pointer cannot be deleted'
@@ -438,6 +468,44 @@ BEGIN
             SELECT 1 FROM strict_ancestors
              WHERE id = candidate_rollback_target_id
         ) INTO rollback_is_strict_ancestor;
+        SELECT EXISTS (
+            SELECT 1
+              FROM travel_windows_flightpublication AS candidate
+              JOIN travel_windows_flightpublication AS target
+                ON target.id = candidate.rollback_target_id
+             WHERE candidate.id = NEW.current_publication_id
+               AND target.schedule_revision_id = candidate.schedule_revision_id
+               AND target.source_revision = candidate.source_revision
+               AND target.source_locator = candidate.source_locator
+               AND target.source_attribution = candidate.source_attribution
+               AND target.source_checked_at = candidate.source_checked_at
+        ) INTO rollback_target_matches;
+
+        SELECT NOT EXISTS (
+                   SELECT 1
+                     FROM travel_windows_flightpublicationduration AS target_link
+                     LEFT JOIN travel_windows_flightpublicationduration AS new_link
+                       ON new_link.publication_id = NEW.current_publication_id
+                      AND new_link.duration_revision_id = target_link.duration_revision_id
+                      AND new_link.destination_airport_id = target_link.destination_airport_id
+                    WHERE target_link.publication_id = candidate_rollback_target_id
+                      AND new_link.id IS NULL
+               )
+               AND NOT EXISTS (
+                   SELECT 1
+                     FROM travel_windows_flightpublicationduration AS new_link
+                     LEFT JOIN travel_windows_flightpublicationduration AS target_link
+                       ON target_link.publication_id = candidate_rollback_target_id
+                      AND target_link.duration_revision_id = new_link.duration_revision_id
+                      AND target_link.destination_airport_id = new_link.destination_airport_id
+                    WHERE new_link.publication_id = NEW.current_publication_id
+                      AND target_link.id IS NULL
+               )
+               AND EXISTS (
+                   SELECT 1 FROM travel_windows_flightpublicationduration
+                    WHERE publication_id = candidate_rollback_target_id
+               )
+          INTO rollback_duration_matches;
     END IF;
 
     SELECT COUNT(*)
@@ -472,7 +540,11 @@ BEGIN
        OR candidate_review_id IS NULL
        OR (
            candidate_rollback_target_id IS NOT NULL
-           AND NOT rollback_is_strict_ancestor
+           AND (
+               NOT rollback_is_strict_ancestor
+               OR NOT rollback_target_matches
+               OR NOT rollback_duration_matches
+           )
        )
        OR NOT review_matches
        OR NOT audit_matches
diff --git a/travel_windows/migrations/0008_seal_flight_candidates.py b/travel_windows/migrations/0008_seal_flight_candidates.py
index d318cbb..5b7770c 100644
--- a/travel_windows/migrations/0008_seal_flight_candidates.py
+++ b/travel_windows/migrations/0008_seal_flight_candidates.py
@@ -59,6 +59,7 @@ def backfill_candidate_seals(apps, schema_editor):
     CandidateDuration = apps.get_model(
         "travel_windows", "FlightCandidateDuration"
     )
+    Publication = apps.get_model("travel_windows", "FlightPublication")
     Seal = apps.get_model("travel_windows", "FlightCandidateSeal")
 
     for revision in Revision.objects.using(alias).all().iterator():
@@ -76,8 +77,12 @@ def backfill_candidate_seals(apps, schema_editor):
             .filter(schedule_revision_id=revision.pk)
         )
         if not schedules or not links:
+            if not Publication.objects.using(alias).filter(
+                schedule_revision_id=revision.pk
+            ).exists():
+                continue
             raise RuntimeError(
-                "flight candidate sealing requires complete typed membership"
+                "published flight candidate sealing requires complete typed membership"
             )
         schedule_hash = _fingerprint(_schedule_payload(revision, schedules))
         duration_identities = []
@@ -121,6 +126,12 @@ DECLARE
     observed_duration_count bigint;
     revision_created_at timestamptz;
 BEGIN
+    PERFORM pg_advisory_xact_lock(
+        hashtextextended(
+            'FLIGHT_CANDIDATE:' || NEW.schedule_revision_id::text,
+            260831
+        )
+    );
     IF NEW.seal_version <> 'CANONICAL_V1' THEN
         RAISE EXCEPTION 'new flight candidate seals require canonical evidence'
             USING ERRCODE = 'check_violation';
@@ -151,6 +162,12 @@ FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_candidate_seal_insert();
 CREATE FUNCTION travel_windows_guard_sealed_schedule_insert() RETURNS trigger
 LANGUAGE plpgsql AS $$
 BEGIN
+    PERFORM pg_advisory_xact_lock(
+        hashtextextended(
+            'FLIGHT_CANDIDATE:' || NEW.revision_id::text,
+            260831
+        )
+    );
     IF EXISTS (
         SELECT 1 FROM travel_windows_flightcandidateseal
          WHERE schedule_revision_id = NEW.revision_id
@@ -171,6 +188,12 @@ DECLARE
     pointer_publication_id uuid;
     revision_is_published boolean;
 BEGIN
+    PERFORM pg_advisory_xact_lock(
+        hashtextextended(
+            'FLIGHT_CANDIDATE:' || NEW.schedule_revision_id::text,
+            260831
+        )
+    );
     IF EXISTS (
         SELECT 1 FROM travel_windows_flightcandidateseal
          WHERE schedule_revision_id = NEW.schedule_revision_id


## `fix(flights): preserve v1 publication rollback`

diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index 8116917..b591a0a 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -29,10 +29,13 @@ from sources.models import (
 )
 
 from .contracts import (
+    AVIATION_PRIOR_SOURCE_REVISION,
     CITY_CONTRACT_FINGERPRINT_SHA256,
     CITY_SCHEMA_FINGERPRINT_SHA256,
     CITY_SOURCE_CODE,
     CITY_SOURCE_REVISION,
+    CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+    CITY_V1_SCHEMA_FINGERPRINT_SHA256,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
@@ -41,12 +44,16 @@ from .contracts import (
     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_SOURCE_CODE,
     LEGACY_SCHEDULE_SOURCE_REVISION,
+    LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_ATTRIBUTION,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
     SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_SOURCE_CODE,
     SCHEDULE_SOURCE_REVISION,
+    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
 )
 from .models import (
     FLIGHT_POINTER_ID,
@@ -232,9 +239,11 @@ def _approved_run(
     source_code: str,
     source_revision: str,
     parser_name: str,
+    parser_version: str,
     contract_fingerprint: str,
     schema_fingerprint: str,
     allow_legacy_unordered: bool = False,
+    allow_historical_source_revision: bool = False,
 ) -> ParseRun:
     try:
         locked = (
@@ -248,12 +257,15 @@ def _approved_run(
     if (
         locked.outcome != ParseRun.Outcome.VALIDATED
         or locked.parser_name != parser_name
-        or locked.parser_version != ParseRun.ParserVersion.V1
+        or locked.parser_version != parser_version
         or locked.parser_contract_fingerprint_sha256 != contract_fingerprint
         or locked.expected_schema_fingerprint_sha256 != schema_fingerprint
         or locked.observed_schema_fingerprint_sha256 != schema_fingerprint
         or source.code != source_code
-        or source.revision != source_revision
+        or (
+            not allow_historical_source_revision
+            and source.revision != source_revision
+        )
         or source.module != SourceConfiguration.Module.AVIATION
         or source.state != SourceConfiguration.State.ACTIVE
         or not source.enabled
@@ -510,6 +522,7 @@ def stage_flight_evidence(
                 source_code=SCHEDULE_SOURCE_CODE,
                 source_revision=SCHEDULE_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+                parser_version=ParseRun.ParserVersion.V2,
                 contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
             )
@@ -518,6 +531,7 @@ def stage_flight_evidence(
                 source_code=CITY_SOURCE_CODE,
                 source_revision=CITY_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+                parser_version=ParseRun.ParserVersion.V2,
                 contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
             )
@@ -526,6 +540,7 @@ def stage_flight_evidence(
                 source_code=LEGACY_SCHEDULE_SOURCE_CODE,
                 source_revision=LEGACY_SCHEDULE_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+                parser_version=ParseRun.ParserVersion.V2,
                 contract_fingerprint=(
                     LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
                 ),
@@ -735,33 +750,101 @@ def _load_valid_candidate(
         raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
 
     allow_legacy_lineage = revision.publications.exists()
+    historical_schedule = (
+        allow_legacy_lineage
+        and revision.parse_run.parser_version == ParseRun.ParserVersion.V1
+    )
+    historical_city = (
+        allow_legacy_lineage
+        and revision.city_reference_parse_run.parser_version
+        == ParseRun.ParserVersion.V1
+    )
+    historical_legacy = (
+        allow_legacy_lineage
+        and revision.legacy_arrivals_parse_run.parser_version
+        == ParseRun.ParserVersion.V1
+    )
 
     _approved_run(
         revision.parse_run,
         source_code=SCHEDULE_SOURCE_CODE,
-        source_revision=SCHEDULE_SOURCE_REVISION,
+        source_revision=(
+            AVIATION_PRIOR_SOURCE_REVISION
+            if historical_schedule
+            else SCHEDULE_SOURCE_REVISION
+        ),
         parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
-        contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-        schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        parser_version=(
+            ParseRun.ParserVersion.V1
+            if historical_schedule
+            else ParseRun.ParserVersion.V2
+        ),
+        contract_fingerprint=(
+            SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256
+            if historical_schedule
+            else SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+        ),
+        schema_fingerprint=(
+            SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256
+            if historical_schedule
+            else SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+        ),
         allow_legacy_unordered=allow_legacy_lineage,
+        allow_historical_source_revision=historical_schedule,
     )
     _approved_run(
         revision.city_reference_parse_run,
         source_code=CITY_SOURCE_CODE,
-        source_revision=CITY_SOURCE_REVISION,
+        source_revision=(
+            AVIATION_PRIOR_SOURCE_REVISION
+            if historical_city
+            else CITY_SOURCE_REVISION
+        ),
         parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
-        contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
-        schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+        parser_version=(
+            ParseRun.ParserVersion.V1
+            if historical_city
+            else ParseRun.ParserVersion.V2
+        ),
+        contract_fingerprint=(
+            CITY_V1_CONTRACT_FINGERPRINT_SHA256
+            if historical_city
+            else CITY_CONTRACT_FINGERPRINT_SHA256
+        ),
+        schema_fingerprint=(
+            CITY_V1_SCHEMA_FINGERPRINT_SHA256
+            if historical_city
+            else CITY_SCHEMA_FINGERPRINT_SHA256
+        ),
         allow_legacy_unordered=allow_legacy_lineage,
+        allow_historical_source_revision=historical_city,
     )
     _approved_run(
         revision.legacy_arrivals_parse_run,
         source_code=LEGACY_SCHEDULE_SOURCE_CODE,
-        source_revision=LEGACY_SCHEDULE_SOURCE_REVISION,
+        source_revision=(
+            AVIATION_PRIOR_SOURCE_REVISION
+            if historical_legacy
+            else LEGACY_SCHEDULE_SOURCE_REVISION
+        ),
         parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
-        contract_fingerprint=LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-        schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        parser_version=(
+            ParseRun.ParserVersion.V1
+            if historical_legacy
+            else ParseRun.ParserVersion.V2
+        ),
+        contract_fingerprint=(
+            LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256
+            if historical_legacy
+            else LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+        ),
+        schema_fingerprint=(
+            LEGACY_SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256
+            if historical_legacy
+            else LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+        ),
         allow_legacy_unordered=allow_legacy_lineage,
+        allow_historical_source_revision=historical_legacy,
     )
 
     schedules = list(
@@ -844,6 +927,7 @@ def _load_valid_candidate(
                 source_code=DURATION_SOURCE_CODE,
                 source_revision=DURATION_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
+                parser_version=ParseRun.ParserVersion.V1,
                 contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
             )
diff --git a/travel_windows/tests/test_review_lifecycle_migration.py b/travel_windows/tests/test_review_lifecycle_migration.py
index 06b13cf..44de263 100644
--- a/travel_windows/tests/test_review_lifecycle_migration.py
+++ b/travel_windows/tests/test_review_lifecycle_migration.py
@@ -1,5 +1,6 @@
 from datetime import date, time, timedelta
 import uuid
+from unittest import mock
 
 from django.contrib.auth import get_user_model
 from django.contrib.auth.models import Permission
@@ -11,25 +12,23 @@ from django.utils import timezone
 from operations.tests.migration_helpers import (
     restore_canonical_migration_graph,
 )
-from sources.management.commands.register_approved_sources import (
-    register_approved_sources,
-)
+from sources.management.commands import register_approved_sources as registration
 from travel_windows.contracts import (
-    CITY_CONTRACT_FINGERPRINT_SHA256,
-    CITY_SCHEMA_FINGERPRINT_SHA256,
+    AVIATION_PRIOR_SOURCE_REVISION,
+    CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+    CITY_V1_SCHEMA_FINGERPRINT_SHA256,
     CITY_SOURCE_CODE,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
-    LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-    LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_SOURCE_CODE,
     SCHEDULE_ATTRIBUTION,
-    SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
-    SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_SOURCE_CODE,
-    SCHEDULE_SOURCE_REVISION,
 )
 from travel_windows.models import (
     FlightCandidateSeal,
@@ -61,7 +60,16 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
         executor = MigrationExecutor(connection)
         executor.migrate(self.previous)
         self.old_apps = executor.loader.project_state(self.previous).apps
-        register_approved_sources(apply=True)
+        historical_specs = tuple(
+            spec.prior_contracts[-1] if spec.prior_contracts else spec
+            for spec in registration.APPROVED_SOURCE_SPECS
+        )
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            historical_specs,
+        ):
+            registration.register_approved_sources(apply=True)
 
     def _fixture_teardown(self):
         try:
@@ -169,22 +177,26 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
         schedule_run = self._validated_parse_run(
             source_code=SCHEDULE_SOURCE_CODE,
             parser_name="ICN_FLIGHT_SCHEDULE_JSON",
-            contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            contract_fingerprint=SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
             identity=base + 1,
         )
         city_run = self._validated_parse_run(
             source_code=CITY_SOURCE_CODE,
             parser_name="ICN_DESTINATION_CITY_JSON",
-            contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+            contract_fingerprint=CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=CITY_V1_SCHEMA_FINGERPRINT_SHA256,
             identity=base + 2,
         )
         legacy_run = self._validated_parse_run(
             source_code=LEGACY_SCHEDULE_SOURCE_CODE,
             parser_name="ICN_LEGACY_ARRIVALS_JSON",
-            contract_fingerprint=LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            contract_fingerprint=(
+                LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256
+            ),
+            schema_fingerprint=(
+                LEGACY_SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256
+            ),
             identity=base + 3,
         )
         duration_run = self._validated_parse_run(
@@ -258,7 +270,7 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
         publication = Publication.objects.create(
             schedule_revision=revision,
             generation=generation,
-            source_revision=SCHEDULE_SOURCE_REVISION,
+            source_revision=AVIATION_PRIOR_SOURCE_REVISION,
             source_locator=SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
             source_attribution=SCHEDULE_ATTRIBUTION,
             source_checked_at=source_checked_at,
