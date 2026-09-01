## `fix(entry): report unavailable source countries`

diff --git a/entry_requirements/ingestion.py b/entry_requirements/ingestion.py
index 7d2ed24..09e47ab 100644
--- a/entry_requirements/ingestion.py
+++ b/entry_requirements/ingestion.py
@@ -90,6 +90,7 @@ ENTRY_INGESTION_LOCK_KEY = 1_006_001
 class EntryIngestionCode:
     REVIEW_REQUIRED = "REVIEW_REQUIRED"
     REPLAY_OBSERVED = "REPLAY_OBSERVED"
+    SOURCE_RECORD_UNAVAILABLE = "SOURCE_RECORD_UNAVAILABLE"
     PARSE_QUARANTINED = "PARSE_QUARANTINED"
     PARSE_FAILED = "PARSE_FAILED"
     FETCH_TERMINAL_FAILED = "FETCH_TERMINAL_FAILED"
@@ -548,11 +549,38 @@ def _consistent_replay_run(
         == ENTRY_SCHEMA_FINGERPRINT_SHA256
         and run.observed_schema_fingerprint_sha256
         == ENTRY_SCHEMA_FINGERPRINT_SHA256
-        and EntryFactRevision.objects.filter(parse_run=run).exists()
     )
     return run if consistent else None
 
 
+def _source_record_is_unavailable(result: EntryParseResult) -> bool:
+    return bool(
+        result.outcome == ParseRun.Outcome.QUARANTINED
+        and result.failure_code == ParseRun.FailureCode.IDENTITY_MISMATCH
+        and result.observed_schema_fingerprint_sha256
+        == ENTRY_SCHEMA_FINGERPRINT_SHA256
+        and result.projection is None
+    )
+
+
+def _validated_artifact_result(
+    *,
+    body: bytes,
+    parser: Callable[[bytes], EntryParseResult],
+    requested_country_iso2: str,
+    requested_result: EntryParseResult,
+) -> EntryParseResult:
+    if not _source_record_is_unavailable(requested_result):
+        return requested_result
+    for country_iso2 in SUPPORTED_COUNTRY_IDS:
+        if country_iso2 == requested_country_iso2:
+            continue
+        candidate = _safe_parse(body, parser, country_iso2)
+        if candidate.outcome == ParseRun.Outcome.VALIDATED:
+            return candidate
+    return requested_result
+
+
 def _create_entry_revision(
     *,
     source: SourceConfiguration,
@@ -643,6 +671,8 @@ def _persist_success(
                     parser,
                     country_iso2,
                 )
+                if _source_record_is_unavailable(parse_result):
+                    return EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE
                 projection = parse_result.projection
                 if (
                     parse_result.outcome != ParseRun.Outcome.VALIDATED
@@ -704,16 +734,22 @@ def _persist_success(
                 parser,
                 country_iso2,
             )
+            artifact_result = _validated_artifact_result(
+                body=verified.body,
+                parser=parser,
+                requested_country_iso2=country_iso2,
+                requested_result=parse_result,
+            )
             completed_at = timezone.now()
             updated = ParseRun.objects.filter(
                 pk=parse_run.id,
                 outcome=ParseRun.Outcome.STARTED,
             ).update(
-                outcome=parse_result.outcome,
+                outcome=artifact_result.outcome,
                 completed_at=completed_at,
-                failure_code=parse_result.failure_code,
+                failure_code=artifact_result.failure_code,
                 observed_schema_fingerprint_sha256=(
-                    parse_result.observed_schema_fingerprint_sha256
+                    artifact_result.observed_schema_fingerprint_sha256
                 ),
             )
             if updated != 1:
@@ -721,7 +757,7 @@ def _persist_success(
                     EntryIngestionCode.PERSISTENCE_FAILED
                 )
 
-            if parse_result.outcome != ParseRun.Outcome.VALIDATED:
+            if artifact_result.outcome != ParseRun.Outcome.VALIDATED:
                 updated = SourceArtifact.objects.filter(
                     pk=artifact.id,
                     state=SourceArtifact.State.RECEIVED,
@@ -730,15 +766,10 @@ def _persist_success(
                     raise _ClosedPersistenceFailure(
                         EntryIngestionCode.PERSISTENCE_FAILED
                     )
-                if parse_result.outcome == ParseRun.Outcome.QUARANTINED:
+                if artifact_result.outcome == ParseRun.Outcome.QUARANTINED:
                     return EntryIngestionCode.PARSE_QUARANTINED
                 return EntryIngestionCode.PARSE_FAILED
 
-            projection = parse_result.projection
-            if projection is None:
-                raise _ClosedPersistenceFailure(
-                    EntryIngestionCode.PERSISTENCE_FAILED
-                )
             updated = SourceArtifact.objects.filter(
                 pk=artifact.id,
                 state=SourceArtifact.State.RECEIVED,
@@ -748,6 +779,15 @@ def _persist_success(
                     EntryIngestionCode.PERSISTENCE_FAILED
                 )
 
+            if _source_record_is_unavailable(parse_result):
+                return EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE
+
+            projection = parse_result.projection
+            if projection is None:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.PERSISTENCE_FAILED
+                )
+
             _create_entry_revision(
                 source=source,
                 rights=rights,
diff --git a/entry_requirements/tests/test_ingestion.py b/entry_requirements/tests/test_ingestion.py
index 92759dd..f54f136 100644
--- a/entry_requirements/tests/test_ingestion.py
+++ b/entry_requirements/tests/test_ingestion.py
@@ -287,6 +287,53 @@ class EntryIngestionTests(TransactionTestCase):
             1,
         )
 
+    def test_missing_country_is_reported_without_invalidating_shared_artifact(self):
+        payload = entry_payload()
+
+        jp = ingest_entry_snapshot(
+            country_iso2="JP",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+        hk = ingest_entry_snapshot(
+            country_iso2="HK",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+
+        self.assertEqual(jp.code, EntryIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(
+            hk.code,
+            EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE,
+        )
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertEqual(EntryFactRevision.objects.count(), 1)
+        self.assertEqual(
+            EntryFactRevision.objects.get().country.iso_alpha2,
+            "JP",
+        )
+
+    def test_missing_country_can_be_checked_before_first_available_country(self):
+        payload = entry_payload()
+
+        hk = ingest_entry_snapshot(
+            country_iso2="HK",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+        jp = ingest_entry_snapshot(
+            country_iso2="JP",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+
+        self.assertEqual(
+            hk.code,
+            EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE,
+        )
+        self.assertEqual(jp.code, EntryIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.get().outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(EntryFactRevision.objects.count(), 1)
+
     def test_same_body_replay_adds_only_fresh_attempt_evidence(self):
         payload = entry_payload()
         first = ingest_entry_snapshot(
@@ -821,6 +868,26 @@ class EntryIngestionCommandTests(SimpleTestCase):
         )
         ingestion.assert_called_once_with(country_iso2="TW")
 
+    def test_command_reports_missing_source_country_with_closed_code(self):
+        with patch(
+            "operations.management.commands.ingest_entry_snapshot."
+            "ingest_entry_snapshot",
+            return_value=EntryIngestionOutcome(
+                EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE,
+                1,
+            ),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_entry_snapshot", "--country", "HK")
+
+        self.assertEqual(
+            str(raised.exception),
+            (
+                "entry_ingestion_result=SOURCE_RECORD_UNAVAILABLE "
+                "attempts=1 country=HK"
+            ),
+        )
+
     def test_command_never_exposes_internal_exception_text(self):
         marker = "sensitive-secret-and-raw-marker"
         with patch(
diff --git a/operations/management/commands/ingest_entry_snapshot.py b/operations/management/commands/ingest_entry_snapshot.py
index 7818469..8a55077 100644
--- a/operations/management/commands/ingest_entry_snapshot.py
+++ b/operations/management/commands/ingest_entry_snapshot.py
@@ -13,6 +13,7 @@ KNOWN_CODES = frozenset(
     {
         EntryIngestionCode.REVIEW_REQUIRED,
         EntryIngestionCode.REPLAY_OBSERVED,
+        EntryIngestionCode.SOURCE_RECORD_UNAVAILABLE,
         EntryIngestionCode.PARSE_QUARANTINED,
         EntryIngestionCode.PARSE_FAILED,
         EntryIngestionCode.FETCH_TERMINAL_FAILED,
