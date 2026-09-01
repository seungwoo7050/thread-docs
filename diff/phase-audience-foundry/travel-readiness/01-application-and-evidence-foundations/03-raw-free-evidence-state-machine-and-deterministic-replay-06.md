## `fix(sources): preserve durable replay evidence`

diff --git a/operations/management/commands/check_live_parser_replay.py b/operations/management/commands/check_live_parser_replay.py
index bb80b93..f54206a 100644
--- a/operations/management/commands/check_live_parser_replay.py
+++ b/operations/management/commands/check_live_parser_replay.py
@@ -1,104 +1,79 @@
+import os
+
 from django.core.management.base import BaseCommand, CommandError
 
 from operations.source_replay import (
     ALLOWED_FAILURE_CODES,
-    ALLOWED_PROVIDER_CODES,
     ALLOWED_SOURCES,
     FAILED,
-    SOURCE_ENTRY,
-    SOURCE_INTERNAL,
-    SOURCE_WARNING,
     VERIFIED,
     LiveParserReplayReceipt,
     verify_live_parser_replay,
 )
-from sources.transport import (
-    ENTRY_REQUEST_FINGERPRINT,
-    WARNING_REQUEST_FINGERPRINT,
+
+
+_SUCCESS_RECEIPT = LiveParserReplayReceipt(
+    result=VERIFIED,
+    source="",
+    failure_code="",
+    attempts=2,
+    artifacts=2,
+    parse_runs=2,
+    parser_invocations=4,
+    typed_revisions=2,
 )
 
+_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+
 
 def _invoke_without_exception_context():
+    secret_value = os.environ.pop(_SECRET_REFERENCE, None)
+
+    def one_secret(reference_name: str):
+        if reference_name != _SECRET_REFERENCE:
+            return None
+        return secret_value
+
     try:
-        return verify_live_parser_replay()
+        return verify_live_parser_replay(secret_loader=one_secret)
     except BaseException:
         return None
+    finally:
+        secret_value = None
 
 
 def _closed_failed_receipt(receipt) -> bool:
-    if not isinstance(receipt, LiveParserReplayReceipt):
-        return False
-    if (
-        receipt.result != FAILED
-        or receipt.source not in ALLOWED_SOURCES
-        or receipt.failure_code not in ALLOWED_FAILURE_CODES
-        or receipt.provider_code not in ALLOWED_PROVIDER_CODES
-    ):
-        return False
-    if receipt.source == SOURCE_INTERNAL:
-        return (
-            receipt.failure_code == "INTERNAL"
-            and receipt.provider_code == ""
-            and receipt.request_revision == ""
-            and receipt.request_sha256 == ""
-        )
-    if receipt.source == SOURCE_ENTRY and (
-        receipt.failure_code == "SECRET_UNAVAILABLE"
-        or receipt.provider_code != ""
-    ):
-        return False
-    if (
-        receipt.provider_code != ""
-        and receipt.failure_code not in {"FETCH_FAILED", "RESPONSE_INTEGRITY"}
-    ):
-        return False
-    expected = {
-        SOURCE_ENTRY: ENTRY_REQUEST_FINGERPRINT,
-        SOURCE_WARNING: WARNING_REQUEST_FINGERPRINT,
-    }[receipt.source]
     return (
-        receipt.request_revision == expected.revision
-        and receipt.request_sha256 == expected.normalized_request_sha256
-    )
-
-
-def _closed_success_receipt(receipt) -> bool:
-    return isinstance(receipt, LiveParserReplayReceipt) and receipt == (
-        LiveParserReplayReceipt(
-            result=VERIFIED,
-            source="",
-            failure_code="",
-            provider_code="",
-            request_revision="",
-            request_sha256="",
-        )
+        isinstance(receipt, LiveParserReplayReceipt)
+        and receipt.result == FAILED
+        and receipt.source in ALLOWED_SOURCES
+        and receipt.failure_code in ALLOWED_FAILURE_CODES
+        and receipt.attempts == 0
+        and receipt.artifacts == 0
+        and receipt.parse_runs == 0
+        and receipt.parser_invocations == 0
+        and receipt.typed_revisions == 0
     )
 
 
 class Command(BaseCommand):
-    help = "Ephemerally replay each approved live source through its parser."
+    help = "Replay approved live sources through durable production ingestion."
 
     def handle(self, *args, **options):
         receipt = _invoke_without_exception_context()
-        if not _closed_success_receipt(receipt) and not _closed_failed_receipt(
-            receipt
-        ):
-            raise CommandError(
-                "live_parser_replay_result=FAILED source=INTERNAL "
-                "failure=INTERNAL"
-            ) from None
-        if _closed_success_receipt(receipt):
+        if receipt == _SUCCESS_RECEIPT:
             self.stdout.write(
-                "live_parser_replay_result=VERIFIED "
-                "sources=2 requests=2 parses=4"
+                "live_parser_replay_result=VERIFIED sources=2 requests=2 "
+                "parser_invocations=4 durable_attempts=2 artifacts=2 "
+                "parse_runs=2 typed=2"
             )
             return
-
-        provider = receipt.provider_code or "NONE"
-        revision = receipt.request_revision or "NONE"
-        request_hash = receipt.request_sha256 or "NONE"
+        if _closed_failed_receipt(receipt):
+            raise CommandError(
+                f"live_parser_replay_result=FAILED source={receipt.source} "
+                f"failure={receipt.failure_code}"
+            ) from None
         raise CommandError(
-            f"live_parser_replay_result=FAILED source={receipt.source} "
-            f"failure={receipt.failure_code} provider_code={provider} "
-            f"request_revision={revision} request_sha256={request_hash}"
+            "live_parser_replay_result=FAILED source=INTERNAL "
+            "failure=INTERNAL"
         ) from None
diff --git a/operations/source_replay.py b/operations/source_replay.py
index 370c9ac..0950c44 100644
--- a/operations/source_replay.py
+++ b/operations/source_replay.py
@@ -1,36 +1,50 @@
-"""Ephemeral live replay of the two approved source parsers.
+"""Durable live replay through the production ingestion boundary.
 
-The verifier makes one bounded request per source, keeps response bytes in
-process memory only, and runs the production parser twice.  Its result is a
-closed receipt that cannot carry response bytes, source facts, locators, or a
-secret value.
+This verifier may run only in an explicitly named disposable PostgreSQL
+database. Production ingestion owns every external request and creates the
+normal rights-bound ``FetchAttempt`` before any response reaches a parser. The
+parser wrappers compare two in-memory parses and return only the first typed
+result to ingestion; response bytes never enter this receipt.
 """
 
 from __future__ import annotations
 
-import hashlib
 import os
+import re
 from dataclasses import dataclass
 from typing import Callable
 
-from entry_requirements.parser import parse_entry_csv
-from sources.models import ParseRun
-from sources.transport import (
-    ENTRY_REQUEST_FINGERPRINT,
-    ENTRY_SOURCE_LOCATOR,
-    PROVIDER_GATEWAY_20,
-    PROVIDER_OTHER_ERROR,
-    PROVIDER_SUCCESS_0,
-    WARNING_REQUEST_FINGERPRINT,
-    WARNING_SECRET_REFERENCE,
-    WARNING_SOURCE_LOCATOR,
-    RequestFingerprint,
-    SingleAttemptResult,
-    fetch_entry_attachment,
-    fetch_travel_alarm_jp,
+from django.db import connection
+
+from entry_requirements.ingestion import (
+    EntryIngestionCode,
+    EntryIngestionOutcome,
+    ingest_entry_snapshot,
+)
+from entry_requirements.models import EntryFactRevision
+from entry_requirements.parser import EntryParseResult, parse_entry_csv
+from reviews.models import (
+    AuditEvent,
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+    ReviewDecision,
+)
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from travel_warnings.ingestion import (
+    TravelWarningIngestionCode,
+    TravelWarningIngestionOutcome,
+    ingest_travel_warning,
 )
+from travel_warnings.models import TravelWarningRevision
 from travel_warnings.parser import (
-    TravelWarningParseSuccess,
+    TravelWarningParseResult,
     parse_travel_alarm_jp,
 )
 
@@ -38,38 +52,43 @@ from travel_warnings.parser import (
 VERIFIED = "VERIFIED"
 FAILED = "FAILED"
 
+SOURCE_DATABASE = "DATABASE"
 SOURCE_ENTRY = "ENTRY"
 SOURCE_WARNING = "WARNING"
 SOURCE_INTERNAL = "INTERNAL"
 
-FAILURE_SECRET_UNAVAILABLE = "SECRET_UNAVAILABLE"
-FAILURE_FETCH = "FETCH_FAILED"
-FAILURE_RESPONSE_INTEGRITY = "RESPONSE_INTEGRITY"
-FAILURE_PARSE = "PARSE_FAILED"
-FAILURE_NONDETERMINISTIC = "NONDETERMINISTIC"
+FAILURE_DATABASE_GATE = "DATABASE_GATE"
+FAILURE_BASELINE = "BASELINE"
+FAILURE_INGESTION = "INGESTION"
+FAILURE_REPLAY = "REPLAY"
+FAILURE_EVIDENCE = "EVIDENCE"
 FAILURE_INTERNAL = "INTERNAL"
 
-_ALLOWED_PROVIDER_CODES = frozenset(
-    {"", PROVIDER_SUCCESS_0, PROVIDER_GATEWAY_20, PROVIDER_OTHER_ERROR}
+ALLOWED_SOURCES = frozenset(
+    {SOURCE_DATABASE, SOURCE_ENTRY, SOURCE_WARNING, SOURCE_INTERNAL}
 )
-ALLOWED_PROVIDER_CODES = _ALLOWED_PROVIDER_CODES
-ALLOWED_SOURCES = frozenset({SOURCE_ENTRY, SOURCE_WARNING, SOURCE_INTERNAL})
 ALLOWED_FAILURE_CODES = frozenset(
     {
-        FAILURE_SECRET_UNAVAILABLE,
-        FAILURE_FETCH,
-        FAILURE_RESPONSE_INTEGRITY,
-        FAILURE_PARSE,
-        FAILURE_NONDETERMINISTIC,
+        FAILURE_DATABASE_GATE,
+        FAILURE_BASELINE,
+        FAILURE_INGESTION,
+        FAILURE_REPLAY,
+        FAILURE_EVIDENCE,
         FAILURE_INTERNAL,
     }
 )
-_CONNECT_TIMEOUT_SECONDS = 5
-_READ_TIMEOUT_SECONDS = 10
 
-EntryTransport = Callable[..., SingleAttemptResult]
-WarningTransport = Callable[..., SingleAttemptResult]
-Parser = Callable[[bytes], object]
+_DATABASE_PATTERN = re.compile(
+    r"^travel_readiness_replaycheck_[a-z0-9]{8,24}$",
+    re.ASCII,
+)
+_SAFETY_TOKEN_ENV = "TRAVEL_READINESS_REPLAY_SAFETY_TOKEN"
+_SAFETY_TOKEN_PREFIX = "LIVE_PARSER_REPLAY_DISPOSABLE:"
+
+EntryIngestor = Callable[..., EntryIngestionOutcome]
+WarningIngestor = Callable[..., TravelWarningIngestionOutcome]
+EntryParser = Callable[[bytes], EntryParseResult]
+WarningParser = Callable[[bytes], TravelWarningParseResult]
 SecretLoader = Callable[[str], object]
 
 
@@ -78,32 +97,23 @@ class LiveParserReplayReceipt:
     result: str
     source: str
     failure_code: str
-    provider_code: str
-    request_revision: str
-    request_sha256: str
-
+    attempts: int
+    artifacts: int
+    parse_runs: int
+    parser_invocations: int
+    typed_revisions: int
 
-def _closed_provider_code(value: object) -> str:
-    return value if type(value) is str and value in _ALLOWED_PROVIDER_CODES else ""
 
-
-def _failure(
-    source: str,
-    failure_code: str,
-    fingerprint: RequestFingerprint | None = None,
-    provider_code: object = "",
-) -> LiveParserReplayReceipt:
+def _failure(source: str, failure_code: str) -> LiveParserReplayReceipt:
     return LiveParserReplayReceipt(
         result=FAILED,
         source=source,
         failure_code=failure_code,
-        provider_code=_closed_provider_code(provider_code),
-        request_revision=fingerprint.revision if fingerprint is not None else "",
-        request_sha256=(
-            fingerprint.normalized_request_sha256
-            if fingerprint is not None
-            else ""
-        ),
+        attempts=0,
+        artifacts=0,
+        parse_runs=0,
+        parser_invocations=0,
+        typed_revisions=0,
     )
 
 
@@ -112,162 +122,210 @@ def _success() -> LiveParserReplayReceipt:
         result=VERIFIED,
         source="",
         failure_code="",
-        provider_code="",
-        request_revision="",
-        request_sha256="",
+        attempts=2,
+        artifacts=2,
+        parse_runs=2,
+        parser_invocations=4,
+        typed_revisions=2,
     )
 
 
-def _response_is_intact(
-    result: SingleAttemptResult,
-    expected_fingerprint: RequestFingerprint,
-) -> bool:
-    return (
-        result.request_fingerprint == expected_fingerprint
-        and result.succeeded
-        and result.http_status == 200
-        and type(result.body) is bytes
-        and bool(result.body)
-        and result.byte_count == len(result.body)
-        and result.response_sha256 == hashlib.sha256(result.body).hexdigest()
+def _storage_boundary_is_clean() -> bool:
+    with connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT count(*)
+              FROM information_schema.columns
+             WHERE table_schema = 'public'
+               AND lower(column_name) ~
+                   '(raw|secret_value|api_?key|service_?key|credential|'
+                   'destination|departure|return_date|travel_date|trip_date|'
+                   'travel_purpose)'
+               AND NOT (
+                   table_name = 'sources_sourceconfiguration'
+                   AND column_name = 'secret_reference_name'
+               )
+               AND NOT (
+                   table_name = 'sources_sourcerightsdecision'
+                   AND column_name IN (
+                       'raw_body_storage_allowed',
+                       'raw_retention_seconds'
+                   )
+               )
+            """
+        )
+        forbidden_columns = cursor.fetchone()
+        cursor.execute("SELECT count(*) FROM pg_catalog.pg_largeobject_metadata")
+        large_objects = cursor.fetchone()
+    return forbidden_columns == (0,) and large_objects == (0,)
+
+
+def _disposable_database_is_approved() -> bool:
+    database_name = connection.settings_dict.get("NAME")
+    if (
+        connection.vendor != "postgresql"
+        or connection.get_autocommit() is not True
+        or connection.in_atomic_block
+        or type(database_name) is not str
+        or _DATABASE_PATTERN.fullmatch(database_name) is None
+        or os.environ.get(_SAFETY_TOKEN_ENV)
+        != f"{_SAFETY_TOKEN_PREFIX}{database_name}"
+    ):
+        return False
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT current_database(), "
+            "current_setting('server_version_num')::integer"
+        )
+        row = cursor.fetchone()
+    return row == (database_name, 180006) and _storage_boundary_is_clean()
+
+
+def _no_evidence_rows() -> bool:
+    return all(
+        count == 0
+        for count in (
+            FetchAttempt.objects.count(),
+            SourceArtifact.objects.count(),
+            ParseRun.objects.count(),
+            EntryFactRevision.objects.count(),
+            TravelWarningRevision.objects.count(),
+            ReviewDecision.objects.count(),
+            PublicationRevision.objects.count(),
+            PublishedEntryFacts.objects.count(),
+            PublishedTravelWarning.objects.count(),
+            AuditEvent.objects.count(),
+        )
     )
 
 
-def _entry_parse_is_valid(result: object) -> bool:
+def _source_baseline_is_exact() -> bool:
+    expected_codes = {"MOFA_ENTRY_CSV", "MOFA_TRAVEL_ALARM_API_JP"}
+    active_codes = set(
+        SourceConfiguration.objects.filter(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        ).values_list("code", flat=True)
+    )
+    approved_rights = SourceRightsDecision.objects.filter(
+        decision=SourceRightsDecision.Decision.APPROVED,
+        access_allowed=True,
+        automated_collection_allowed=True,
+        typed_field_storage_allowed=True,
+        raw_body_storage_allowed=False,
+        typed_republication_allowed=True,
+        raw_retention_seconds=0,
+        typed_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+        evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+    )
     return (
-        getattr(result, "outcome", None) == ParseRun.Outcome.VALIDATED
-        and getattr(result, "projection", None) is not None
+        SourceConfiguration.objects.count() == 2
+        and active_codes == expected_codes
+        and SourceRightsDecision.objects.count() == 2
+        and set(approved_rights.values_list("source__code", flat=True))
+        == expected_codes
+        and _no_evidence_rows()
     )
 
 
-def _warning_parse_is_valid(result: object) -> bool:
-    return isinstance(result, TravelWarningParseSuccess)
-
-
-def _replay_one(
-    *,
-    source: str,
-    response: SingleAttemptResult,
-    fingerprint: RequestFingerprint,
-    parser: Parser,
-    validator: Callable[[object], bool],
-) -> LiveParserReplayReceipt | None:
-    if not _response_is_intact(response, fingerprint):
-        failure_code = (
-            FAILURE_FETCH
-            if not response.succeeded
-            else FAILURE_RESPONSE_INTEGRITY
-        )
-        return _failure(
-            source,
-            failure_code,
-            fingerprint,
-            response.provider_code,
+def _evidence_is_exact() -> bool:
+    expected_sources = {"MOFA_ENTRY_CSV", "MOFA_TRAVEL_ALARM_API_JP"}
+    attempts = FetchAttempt.objects.all()
+    artifacts = SourceArtifact.objects.all()
+    parse_runs = ParseRun.objects.all()
+    return (
+        attempts.count() == 2
+        and set(attempts.values_list("source__code", flat=True))
+        == expected_sources
+        and not attempts.exclude(result=FetchAttempt.Result.SUCCEEDED).exists()
+        and all(
+            value == 1
+            for value in attempts.values_list("attempt_number", flat=True)
         )
-
-    try:
-        first = parser(response.body)
-        second = parser(response.body)
-    except BaseException:
-        return _failure(source, FAILURE_PARSE, fingerprint)
-
-    if not validator(first) or not validator(second):
-        return _failure(source, FAILURE_PARSE, fingerprint)
-    if first != second:
-        return _failure(source, FAILURE_NONDETERMINISTIC, fingerprint)
-    return None
+        and not attempts.filter(result=FetchAttempt.Result.STARTED).exists()
+        and artifacts.count() == 2
+        and set(artifacts.values_list("source__code", flat=True))
+        == expected_sources
+        and not artifacts.exclude(
+            state=SourceArtifact.State.REVIEW_REQUIRED
+        ).exists()
+        and parse_runs.count() == 2
+        and not parse_runs.exclude(outcome=ParseRun.Outcome.VALIDATED).exists()
+        and EntryFactRevision.objects.count() == 1
+        and TravelWarningRevision.objects.count() == 1
+        and ReviewDecision.objects.count() == 0
+        and PublicationRevision.objects.count() == 0
+        and PublishedEntryFacts.objects.count() == 0
+        and PublishedTravelWarning.objects.count() == 0
+        and AuditEvent.objects.count() == 0
+        and _storage_boundary_is_clean()
+    )
 
 
 def verify_live_parser_replay(
     *,
-    entry_transport: EntryTransport = fetch_entry_attachment,
-    warning_transport: WarningTransport = fetch_travel_alarm_jp,
-    entry_parser: Parser = parse_entry_csv,
-    warning_parser: Parser = parse_travel_alarm_jp,
+    entry_ingestor: EntryIngestor = ingest_entry_snapshot,
+    warning_ingestor: WarningIngestor = ingest_travel_warning,
+    entry_parser: EntryParser = parse_entry_csv,
+    warning_parser: WarningParser = parse_travel_alarm_jp,
     secret_loader: SecretLoader = os.environ.get,
+    database_guard: Callable[[], bool] = _disposable_database_is_approved,
+    baseline_guard: Callable[[], bool] = _source_baseline_is_exact,
+    evidence_guard: Callable[[], bool] = _evidence_is_exact,
 ) -> LiveParserReplayReceipt:
-    """Verify two live bodies without persistence, retries, or detailed output."""
+    """Run production ingestion once per source in a disposable database."""
+
+    parser_calls = {SOURCE_ENTRY: 0, SOURCE_WARNING: 0}
+
+    def replay_entry(payload: bytes) -> EntryParseResult:
+        first = entry_parser(payload)
+        parser_calls[SOURCE_ENTRY] += 1
+        second = entry_parser(payload)
+        parser_calls[SOURCE_ENTRY] += 1
+        if first != second:
+            raise RuntimeError from None
+        return first
+
+    def replay_warning(payload: bytes) -> TravelWarningParseResult:
+        first = warning_parser(payload)
+        parser_calls[SOURCE_WARNING] += 1
+        second = warning_parser(payload)
+        parser_calls[SOURCE_WARNING] += 1
+        if first != second:
+            raise RuntimeError from None
+        return first
 
-    secret_value: object = None
-    entry_response: SingleAttemptResult | None = None
-    warning_response: SingleAttemptResult | None = None
     try:
-        try:
-            entry_response = entry_transport(
-                official_locator=ENTRY_SOURCE_LOCATOR,
-                connect_timeout_seconds=_CONNECT_TIMEOUT_SECONDS,
-                read_timeout_seconds=_READ_TIMEOUT_SECONDS,
-            )
-        except BaseException:
-            return _failure(
-                SOURCE_ENTRY,
-                FAILURE_FETCH,
-                ENTRY_REQUEST_FINGERPRINT,
-            )
-        if not isinstance(entry_response, SingleAttemptResult):
-            return _failure(
-                SOURCE_ENTRY,
-                FAILURE_INTERNAL,
-                ENTRY_REQUEST_FINGERPRINT,
-            )
-        entry_failure = _replay_one(
-            source=SOURCE_ENTRY,
-            response=entry_response,
-            fingerprint=ENTRY_REQUEST_FINGERPRINT,
-            parser=entry_parser,
-            validator=_entry_parse_is_valid,
-        )
-        if entry_failure is not None:
-            return entry_failure
-
-        try:
-            secret_value = secret_loader(WARNING_SECRET_REFERENCE)
-        except BaseException:
-            return _failure(
-                SOURCE_WARNING,
-                FAILURE_SECRET_UNAVAILABLE,
-                WARNING_REQUEST_FINGERPRINT,
-            )
-        if type(secret_value) is not str or not secret_value:
-            return _failure(
-                SOURCE_WARNING,
-                FAILURE_SECRET_UNAVAILABLE,
-                WARNING_REQUEST_FINGERPRINT,
-            )
-
-        try:
-            warning_response = warning_transport(
-                official_locator=WARNING_SOURCE_LOCATOR,
-                secret_reference_name=WARNING_SECRET_REFERENCE,
-                secret_value=secret_value,
-                connect_timeout_seconds=_CONNECT_TIMEOUT_SECONDS,
-                read_timeout_seconds=_READ_TIMEOUT_SECONDS,
-            )
-        except BaseException:
-            return _failure(
-                SOURCE_WARNING,
-                FAILURE_FETCH,
-                WARNING_REQUEST_FINGERPRINT,
-            )
-        if not isinstance(warning_response, SingleAttemptResult):
-            return _failure(
-                SOURCE_WARNING,
-                FAILURE_INTERNAL,
-                WARNING_REQUEST_FINGERPRINT,
-            )
-        warning_failure = _replay_one(
-            source=SOURCE_WARNING,
-            response=warning_response,
-            fingerprint=WARNING_REQUEST_FINGERPRINT,
-            parser=warning_parser,
-            validator=_warning_parse_is_valid,
+        if not database_guard():
+            return _failure(SOURCE_DATABASE, FAILURE_DATABASE_GATE)
+        if not baseline_guard():
+            return _failure(SOURCE_DATABASE, FAILURE_BASELINE)
+
+        entry_outcome = entry_ingestor(parser=replay_entry)
+        if (
+            not isinstance(entry_outcome, EntryIngestionOutcome)
+            or entry_outcome.code != EntryIngestionCode.REVIEW_REQUIRED
+            or entry_outcome.attempt_count != 1
+        ):
+            return _failure(SOURCE_ENTRY, FAILURE_INGESTION)
+        if parser_calls[SOURCE_ENTRY] != 2:
+            return _failure(SOURCE_ENTRY, FAILURE_REPLAY)
+
+        warning_outcome = warning_ingestor(
+            parser=replay_warning,
+            secret_loader=secret_loader,
         )
-        if warning_failure is not None:
-            return warning_failure
+        if (
+            not isinstance(warning_outcome, TravelWarningIngestionOutcome)
+            or warning_outcome.code != TravelWarningIngestionCode.REVIEW_REQUIRED
+            or warning_outcome.attempt_count != 1
+        ):
+            return _failure(SOURCE_WARNING, FAILURE_INGESTION)
+        if parser_calls[SOURCE_WARNING] != 2:
+            return _failure(SOURCE_WARNING, FAILURE_REPLAY)
+
+        if not evidence_guard():
+            return _failure(SOURCE_DATABASE, FAILURE_EVIDENCE)
         return _success()
     except BaseException:
         return _failure(SOURCE_INTERNAL, FAILURE_INTERNAL)
-    finally:
-        secret_value = None
-        entry_response = None
-        warning_response = None
diff --git a/operations/tests/test_live_parser_replay.py b/operations/tests/test_live_parser_replay.py
index 598a3bc..fbedfcd 100644
--- a/operations/tests/test_live_parser_replay.py
+++ b/operations/tests/test_live_parser_replay.py
@@ -1,33 +1,37 @@
-import hashlib
 import io
+import os
 from dataclasses import replace
 from datetime import date
-from unittest.mock import patch
+from pathlib import Path
+from unittest.mock import MagicMock, patch
 
 from django.core.management import call_command
 from django.core.management.base import CommandError
 from django.test import SimpleTestCase
 
+from entry_requirements.ingestion import (
+    EntryIngestionCode,
+    EntryIngestionOutcome,
+)
 from entry_requirements.parser import EntryParseResult, EntryProjection
+from operations.management.commands import check_live_parser_replay
 from operations.source_replay import (
     FAILED,
-    FAILURE_FETCH,
-    FAILURE_NONDETERMINISTIC,
-    FAILURE_PARSE,
-    FAILURE_SECRET_UNAVAILABLE,
+    FAILURE_BASELINE,
+    FAILURE_DATABASE_GATE,
+    FAILURE_EVIDENCE,
+    FAILURE_INGESTION,
+    SOURCE_DATABASE,
     SOURCE_ENTRY,
-    SOURCE_WARNING,
     VERIFIED,
+    LiveParserReplayReceipt,
+    _disposable_database_is_approved,
     verify_live_parser_replay,
 )
 from sources.models import ParseRun
-from sources.transport import (
-    ATTEMPT_SUCCEEDED,
-    ENTRY_REQUEST_FINGERPRINT,
-    FAILURE_AUTHENTICATION,
-    PROVIDER_GATEWAY_20,
-    WARNING_REQUEST_FINGERPRINT,
-    SingleAttemptResult,
+from travel_warnings.ingestion import (
+    TravelWarningIngestionCode,
+    TravelWarningIngestionOutcome,
 )
 from travel_warnings.parser import (
     ParsedTravelWarning,
@@ -36,20 +40,7 @@ from travel_warnings.parser import (
 
 
 SECRET_MARKER = "private-live-replay-secret-marker"
-ENTRY_BODY = b"entry-live-body"
-WARNING_BODY = b"warning-live-body"
-
-
-def _response(fingerprint, body, provider_code=""):
-    return SingleAttemptResult(
-        request_fingerprint=fingerprint,
-        attempt_result=ATTEMPT_SUCCEEDED,
-        http_status=200,
-        provider_code=provider_code,
-        response_sha256=hashlib.sha256(body).hexdigest(),
-        byte_count=len(body),
-        body=body,
-    )
+RAW_MARKER = b"private-live-response-marker"
 
 
 def _entry_success(period="90일"):
@@ -59,11 +50,11 @@ def _entry_success(period="90일"):
         observed_schema_fingerprint_sha256="a" * 64,
         projection=EntryProjection(
             country_iso2="JP",
-            passport_policy_code="KR_ORDINARY_SHORT_TOURISM",
+            passport_policy_code="KOR_ORDINARY_SHORT_TOURISM",
             passport_policy_revision="V1",
             ordinary_passport_period_text=period,
             basis_text="일반여권 소지자: unit-test fact",
-            snapshot_date=date(2026, 8, 1),
+            snapshot_date=date(2025, 1, 20),
             typed_fingerprint_sha256="b" * 64,
         ),
     )
@@ -87,186 +78,244 @@ def _warning_success(level="1"):
 
 class LiveParserReplayTests(SimpleTestCase):
     def _verify(self, **overrides):
+        calls = {"entry_ingest": 0, "warning_ingest": 0}
+
+        def entry_ingestor(*, parser):
+            calls["entry_ingest"] += 1
+            self.assertEqual(parser(RAW_MARKER), _entry_success())
+            return EntryIngestionOutcome(
+                EntryIngestionCode.REVIEW_REQUIRED,
+                1,
+            )
+
+        def warning_ingestor(*, parser, secret_loader):
+            calls["warning_ingest"] += 1
+            self.assertEqual(
+                secret_loader("MOFA_TRAVEL_ALARM_SERVICE_KEY"),
+                SECRET_MARKER,
+            )
+            self.assertEqual(parser(RAW_MARKER), _warning_success())
+            return TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.REVIEW_REQUIRED,
+                1,
+            )
+
         values = {
-            "entry_transport": lambda **kwargs: _response(
-                ENTRY_REQUEST_FINGERPRINT, ENTRY_BODY
-            ),
-            "warning_transport": lambda **kwargs: _response(
-                WARNING_REQUEST_FINGERPRINT, WARNING_BODY
-            ),
+            "entry_ingestor": entry_ingestor,
+            "warning_ingestor": warning_ingestor,
             "entry_parser": lambda body: _entry_success(),
             "warning_parser": lambda body: _warning_success(),
             "secret_loader": lambda name: SECRET_MARKER,
+            "database_guard": lambda: True,
+            "baseline_guard": lambda: True,
+            "evidence_guard": lambda: True,
         }
         values.update(overrides)
-        return verify_live_parser_replay(**values)
-
-    def test_one_request_and_two_identical_parses_per_source(self):
-        calls = {
-            "entry_fetch": 0,
-            "warning_fetch": 0,
-            "entry_parse": 0,
-            "warning_parse": 0,
-        }
-
-        def entry_transport(**kwargs):
-            calls["entry_fetch"] += 1
-            return _response(ENTRY_REQUEST_FINGERPRINT, ENTRY_BODY)
+        return verify_live_parser_replay(**values), calls
 
-        def warning_transport(**kwargs):
-            calls["warning_fetch"] += 1
-            self.assertEqual(kwargs["secret_value"], SECRET_MARKER)
-            calls["warning_parse"] += 0
-            return _response(WARNING_REQUEST_FINGERPRINT, WARNING_BODY)
+    def test_production_ingestion_owns_each_request_and_parser_runs_twice(self):
+        parser_calls = {"entry": 0, "warning": 0}
 
         def entry_parser(body):
-            self.assertEqual(body, ENTRY_BODY)
-            calls["entry_parse"] += 1
+            self.assertEqual(body, RAW_MARKER)
+            parser_calls["entry"] += 1
             return _entry_success()
 
         def warning_parser(body):
-            self.assertEqual(body, WARNING_BODY)
-            calls["warning_parse"] += 1
+            self.assertEqual(body, RAW_MARKER)
+            parser_calls["warning"] += 1
             return _warning_success()
 
-        receipt = self._verify(
-            entry_transport=entry_transport,
-            warning_transport=warning_transport,
+        receipt, ingestion_calls = self._verify(
             entry_parser=entry_parser,
             warning_parser=warning_parser,
         )
 
         self.assertEqual(receipt.result, VERIFIED)
-        self.assertEqual(
-            calls,
-            {
-                "entry_fetch": 1,
-                "warning_fetch": 1,
-                "entry_parse": 2,
-                "warning_parse": 2,
-            },
-        )
+        self.assertEqual(ingestion_calls, {"entry_ingest": 1, "warning_ingest": 1})
+        self.assertEqual(parser_calls, {"entry": 2, "warning": 2})
         self.assertNotIn(SECRET_MARKER, repr(receipt))
-        self.assertNotIn("live-body", repr(receipt))
-
-    def test_secret_is_loaded_only_after_entry_validation(self):
-        secret_calls = []
-        receipt = self._verify(
-            entry_parser=lambda body: replace(
-                _entry_success(),
-                outcome=ParseRun.Outcome.QUARANTINED,
-                projection=None,
-            ),
-            secret_loader=lambda name: secret_calls.append(name) or SECRET_MARKER,
-        )
+        self.assertNotIn(RAW_MARKER.decode(), repr(receipt))
+
+    def test_database_gate_runs_before_any_ingestion(self):
+        receipt, calls = self._verify(database_guard=lambda: False)
         self.assertEqual(receipt.result, FAILED)
+        self.assertEqual(receipt.source, SOURCE_DATABASE)
+        self.assertEqual(receipt.failure_code, FAILURE_DATABASE_GATE)
+        self.assertEqual(calls, {"entry_ingest": 0, "warning_ingest": 0})
+
+    def test_empty_baseline_runs_before_any_ingestion(self):
+        receipt, calls = self._verify(baseline_guard=lambda: False)
+        self.assertEqual(receipt.failure_code, FAILURE_BASELINE)
+        self.assertEqual(calls, {"entry_ingest": 0, "warning_ingest": 0})
+
+    def test_entry_failure_stops_before_warning(self):
+        receipt, calls = self._verify(
+            entry_ingestor=lambda **kwargs: EntryIngestionOutcome(
+                EntryIngestionCode.PARSE_FAILED,
+                1,
+            )
+        )
         self.assertEqual(receipt.source, SOURCE_ENTRY)
-        self.assertEqual(receipt.failure_code, FAILURE_PARSE)
-        self.assertEqual(secret_calls, [])
-
-    def test_missing_secret_fails_without_calling_warning_transport(self):
-        warning_calls = []
-        receipt = self._verify(
-            secret_loader=lambda name: None,
-            warning_transport=lambda **kwargs: warning_calls.append(kwargs),
+        self.assertEqual(receipt.failure_code, FAILURE_INGESTION)
+        self.assertEqual(calls["warning_ingest"], 0)
+
+    def test_nondeterministic_parse_fails_inside_production_ingestion(self):
+        parses = iter((_entry_success("90일"), _entry_success("30일")))
+
+        def ingestor(*, parser):
+            try:
+                parser(RAW_MARKER)
+            except RuntimeError:
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.PARSE_FAILED,
+                    1,
+                )
+            self.fail("nondeterministic parser was accepted")
+
+        receipt, _ = self._verify(
+            entry_ingestor=ingestor,
+            entry_parser=lambda body: next(parses),
         )
-        self.assertEqual(receipt.result, FAILED)
-        self.assertEqual(receipt.source, SOURCE_WARNING)
-        self.assertEqual(receipt.failure_code, FAILURE_SECRET_UNAVAILABLE)
-        self.assertEqual(warning_calls, [])
+        self.assertEqual(receipt.source, SOURCE_ENTRY)
+        self.assertEqual(receipt.failure_code, FAILURE_INGESTION)
 
-    def test_parse_exception_detail_and_body_never_enter_receipt(self):
-        marker = "private-parser-exception-marker"
+    def test_durable_evidence_counts_are_a_success_gate(self):
+        receipt, _ = self._verify(evidence_guard=lambda: False)
+        self.assertEqual(receipt.source, SOURCE_DATABASE)
+        self.assertEqual(receipt.failure_code, FAILURE_EVIDENCE)
 
-        def failed_parser(body):
-            raise RuntimeError(f"{marker}:{body!r}:{SECRET_MARKER}")
+    def test_guard_exception_is_sanitized(self):
+        marker = "private-database-exception-marker"
 
-        receipt = self._verify(entry_parser=failed_parser)
+        def failed_guard():
+            raise RuntimeError(f"{marker}:{SECRET_MARKER}")
 
-        self.assertEqual(receipt.failure_code, FAILURE_PARSE)
+        receipt, calls = self._verify(database_guard=failed_guard)
+        self.assertEqual(receipt.source, "INTERNAL")
         self.assertNotIn(marker, repr(receipt))
         self.assertNotIn(SECRET_MARKER, repr(receipt))
-        self.assertNotIn("live-body", repr(receipt))
-
-    def test_parser_drift_is_fail_closed(self):
-        results = iter((_entry_success("90일"), _entry_success("30일")))
-        receipt = self._verify(entry_parser=lambda body: next(results))
-        self.assertEqual(receipt.failure_code, FAILURE_NONDETERMINISTIC)
-
-    def test_warning_provider_failure_is_allowlisted_and_redacted(self):
-        failed = SingleAttemptResult(
-            request_fingerprint=WARNING_REQUEST_FINGERPRINT,
-            attempt_result="TERMINAL_FAILED",
-            http_status=403,
-            provider_code=PROVIDER_GATEWAY_20,
-            failure_code=FAILURE_AUTHENTICATION,
-        )
-        receipt = self._verify(warning_transport=lambda **kwargs: failed)
-        self.assertEqual(receipt.failure_code, FAILURE_FETCH)
-        self.assertEqual(receipt.provider_code, PROVIDER_GATEWAY_20)
-        rendered = repr(receipt)
-        self.assertIn(WARNING_REQUEST_FINGERPRINT.revision, rendered)
-        self.assertIn(WARNING_REQUEST_FINGERPRINT.normalized_request_sha256, rendered)
-        self.assertNotIn(SECRET_MARKER, rendered)
+        self.assertEqual(calls, {"entry_ingest": 0, "warning_ingest": 0})
 
-    def test_invalid_transport_object_fails_with_fixed_receipt(self):
-        marker = "private-invalid-transport-marker"
-        receipt = self._verify(entry_transport=lambda **kwargs: marker)
-        self.assertEqual(receipt.result, FAILED)
-        self.assertEqual(receipt.source, SOURCE_ENTRY)
-        self.assertNotIn(marker, repr(receipt))
+    def test_default_path_does_not_import_or_call_transports_directly(self):
+        source = Path(
+            "operations/source_replay.py"
+        ).read_text(encoding="utf-8")
+        self.assertNotIn("fetch_entry_attachment", source)
+        self.assertNotIn("fetch_travel_alarm_jp", source)
+        self.assertNotIn("sources.transport", source)
+        self.assertIn("ingest_entry_snapshot", source)
+        self.assertIn("ingest_travel_warning", source)
+
+    def test_disposable_database_requires_exact_name_token_and_postgresql_18_6(self):
+        database_name = "travel_readiness_replaycheck_abc12345"
+        fake_connection = MagicMock()
+        fake_connection.vendor = "postgresql"
+        fake_connection.get_autocommit.return_value = True
+        fake_connection.in_atomic_block = False
+        fake_connection.settings_dict = {"NAME": database_name}
+        cursor = fake_connection.cursor.return_value.__enter__.return_value
+        cursor.fetchone.return_value = (database_name, 180006)
+        token = f"LIVE_PARSER_REPLAY_DISPOSABLE:{database_name}"
+        with (
+            patch("operations.source_replay.connection", fake_connection),
+            patch.dict(
+                os.environ,
+                {"TRAVEL_READINESS_REPLAY_SAFETY_TOKEN": token},
+                clear=False,
+            ),
+            patch(
+                "operations.source_replay._storage_boundary_is_clean",
+                return_value=True,
+            ),
+        ):
+            self.assertTrue(_disposable_database_is_approved())
+
+        for bad_name, bad_token, vendor, version in (
+            ("travel_readiness", token, "postgresql", 180006),
+            (database_name, "wrong", "postgresql", 180006),
+            (database_name, token, "sqlite", 180006),
+            (database_name, token, "postgresql", 180005),
+        ):
+            fake_connection.vendor = vendor
+            fake_connection.settings_dict = {"NAME": bad_name}
+            cursor.fetchone.return_value = (bad_name, version)
+            with (
+                patch("operations.source_replay.connection", fake_connection),
+                patch.dict(
+                    os.environ,
+                    {"TRAVEL_READINESS_REPLAY_SAFETY_TOKEN": bad_token},
+                    clear=False,
+                ),
+                patch(
+                    "operations.source_replay._storage_boundary_is_clean",
+                    return_value=True,
+                ),
+            ):
+                self.assertFalse(_disposable_database_is_approved())
 
     def test_management_command_success_is_a_fixed_receipt(self):
+        receipt, _ = self._verify()
         stdout = io.StringIO()
         with patch(
             "operations.management.commands.check_live_parser_replay."
             "verify_live_parser_replay",
-            return_value=self._verify(),
+            return_value=receipt,
         ):
             call_command("check_live_parser_replay", stdout=stdout)
         self.assertEqual(
             stdout.getvalue().strip(),
-            "live_parser_replay_result=VERIFIED sources=2 requests=2 parses=4",
+            "live_parser_replay_result=VERIFIED sources=2 requests=2 "
+            "parser_invocations=4 durable_attempts=2 artifacts=2 "
+            "parse_runs=2 typed=2",
         )
 
-    def test_management_command_failure_has_only_redacted_shape(self):
-        receipt = self._verify(
-            warning_transport=lambda **kwargs: SingleAttemptResult(
-                request_fingerprint=WARNING_REQUEST_FINGERPRINT,
-                attempt_result="TERMINAL_FAILED",
-                http_status=403,
-                provider_code=PROVIDER_GATEWAY_20,
-                failure_code=FAILURE_AUTHENTICATION,
-            )
-        )
-        stderr = io.StringIO()
-        with patch(
-            "operations.management.commands.check_live_parser_replay."
-            "verify_live_parser_replay",
-            return_value=receipt,
+    def test_management_command_pops_secret_before_invoking_verifier(self):
+        secret_name = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+        receipt, _ = self._verify()
+
+        def verifier(*, secret_loader):
+            self.assertNotIn(secret_name, os.environ)
+            self.assertIsNone(secret_loader("UNAPPROVED_REFERENCE"))
+            self.assertEqual(secret_loader(secret_name), SECRET_MARKER)
+            return receipt
+
+        with (
+            patch.dict(os.environ, {secret_name: SECRET_MARKER}, clear=False),
+            patch(
+                "operations.management.commands.check_live_parser_replay."
+                "verify_live_parser_replay",
+                side_effect=verifier,
+            ),
         ):
-            with self.assertRaises(CommandError) as raised:
-                call_command("check_live_parser_replay", stderr=stderr)
-        rendered = str(raised.exception)
-        self.assertIn("provider_code=MOFA_GATEWAY_20", rendered)
-        self.assertIn("request_revision=MOFA_WARNING_V1", rendered)
-        self.assertNotIn(SECRET_MARKER, rendered)
-        self.assertNotIn("https://", rendered)
-        self.assertEqual(stderr.getvalue(), "")
+            observed = check_live_parser_replay._invoke_without_exception_context()
+            self.assertNotIn(secret_name, os.environ)
+            self.assertNotIn(SECRET_MARKER, repr(observed))
+        self.assertEqual(observed, receipt)
 
-    def test_management_command_rejects_an_unclosed_receipt(self):
-        injected = replace(
-            self._verify(
-                warning_transport=lambda **kwargs: SingleAttemptResult(
-                    request_fingerprint=WARNING_REQUEST_FINGERPRINT,
-                    attempt_result="TERMINAL_FAILED",
-                    failure_code=FAILURE_AUTHENTICATION,
-                )
+    def test_management_command_sanitizes_exception_after_secret_pop(self):
+        secret_name = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+
+        def failed_verifier(*, secret_loader):
+            self.assertNotIn(secret_name, os.environ)
+            self.assertEqual(secret_loader(secret_name), SECRET_MARKER)
+            raise RuntimeError(SECRET_MARKER)
+
+        with (
+            patch.dict(os.environ, {secret_name: SECRET_MARKER}, clear=False),
+            patch(
+                "operations.management.commands.check_live_parser_replay."
+                "verify_live_parser_replay",
+                side_effect=failed_verifier,
             ),
-            failure_code=SECRET_MARKER,
-            request_revision=SECRET_MARKER,
-            request_sha256=SECRET_MARKER,
-        )
+        ):
+            observed = check_live_parser_replay._invoke_without_exception_context()
+            self.assertNotIn(secret_name, os.environ)
+        self.assertIsNone(observed)
+
+    def test_management_command_rejects_an_unclosed_receipt(self):
+        receipt, _ = self._verify(database_guard=lambda: False)
+        injected = replace(receipt, failure_code=SECRET_MARKER)
         with patch(
             "operations.management.commands.check_live_parser_replay."
             "verify_live_parser_replay",
@@ -281,3 +330,27 @@ class LiveParserReplayTests(SimpleTestCase):
             "failure=INTERNAL",
         )
         self.assertNotIn(SECRET_MARKER, rendered)
+
+    def test_management_command_failure_is_closed(self):
+        receipt = LiveParserReplayReceipt(
+            result=FAILED,
+            source=SOURCE_DATABASE,
+            failure_code=FAILURE_DATABASE_GATE,
+            attempts=0,
+            artifacts=0,
+            parse_runs=0,
+            parser_invocations=0,
+            typed_revisions=0,
+        )
+        with patch(
+            "operations.management.commands.check_live_parser_replay."
+            "verify_live_parser_replay",
+            return_value=receipt,
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("check_live_parser_replay")
+        self.assertEqual(
+            str(raised.exception),
+            "live_parser_replay_result=FAILED source=DATABASE "
+            "failure=DATABASE_GATE",
+        )
diff --git a/operations/tests/test_live_parser_replay_runner.py b/operations/tests/test_live_parser_replay_runner.py
new file mode 100644
index 0000000..3af9314
--- /dev/null
+++ b/operations/tests/test_live_parser_replay_runner.py
@@ -0,0 +1,471 @@
+from __future__ import annotations
+
+import os
+from pathlib import Path
+import shutil
+import stat
+import subprocess
+import tempfile
+import textwrap
+import unittest
+
+
+class LiveParserReplayRunnerTests(unittest.TestCase):
+    @classmethod
+    def setUpClass(cls):
+        super().setUpClass()
+        cls.root = Path(__file__).resolve().parents[2]
+        cls.script = cls.root / "scripts" / "check-live-parser-replay"
+        cls.database_name = "travel_readiness_replaycheck_unit12345"
+        cls.release_sha = "a" * 40
+
+    def arguments(self, **overrides):
+        values = {
+            "host": "127.0.0.1",
+            "port": "5432",
+            "admin_role": "postgres",
+            "admin_password_env": "TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD",
+            "database_name": self.database_name,
+            "safety_token": (
+                f"LIVE_PARSER_REPLAY_DISPOSABLE:{self.database_name}"
+            ),
+            "release_sha": self.release_sha,
+        }
+        values.update(overrides)
+        return [
+            "--host",
+            values["host"],
+            "--port",
+            values["port"],
+            "--admin-role",
+            values["admin_role"],
+            "--admin-password-env",
+            values["admin_password_env"],
+            "--database-name",
+            values["database_name"],
+            "--safety-token",
+            values["safety_token"],
+            "--release-sha",
+            values["release_sha"],
+        ]
+
+    def run_script(self, script=None, *arguments, env=None, cwd=None):
+        return subprocess.run(
+            ["/bin/sh", str(script or self.script), *arguments],
+            cwd=cwd or self.root,
+            env=env,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=20,
+        )
+
+    def test_script_is_executable_and_help_is_fixed(self):
+        self.assertEqual(stat.S_IMODE(self.script.stat().st_mode), 0o755)
+        result = self.run_script(None, "--help")
+        self.assertEqual(result.returncode, 0)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(
+            result.stdout,
+            "usage: check-live-parser-replay --host LOOPBACK --port PORT "
+            "--admin-role ROLE --admin-password-env "
+            "TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD --database-name "
+            "travel_readiness_replaycheck_NAME --safety-token "
+            "LIVE_PARSER_REPLAY_DISPOSABLE:"
+            "travel_readiness_replaycheck_NAME --release-sha SHA\n",
+        )
+
+    def test_arguments_and_target_are_fail_closed_before_secret_access(self):
+        cases = (
+            ((), 64, "live_parser_replay_check=INVALID_ARGUMENTS\n"),
+            (
+                ("--unknown",),
+                64,
+                "live_parser_replay_check=INVALID_ARGUMENTS\n",
+            ),
+            (
+                ("--host", "localhost", "--host", "127.0.0.1"),
+                64,
+                "live_parser_replay_check=INVALID_ARGUMENTS\n",
+            ),
+            (
+                tuple(self.arguments(host="remote.invalid")),
+                65,
+                "live_parser_replay_check=NON_LOOPBACK_REFUSED\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_name="travel_readiness",
+                        safety_token=(
+                            "LIVE_PARSER_REPLAY_DISPOSABLE:travel_readiness"
+                        ),
+                    )
+                ),
+                65,
+                "live_parser_replay_check=UNSAFE_DATABASE\n",
+            ),
+            (
+                tuple(self.arguments(safety_token="wrong")),
+                65,
+                "live_parser_replay_check=SAFETY_TOKEN_MISMATCH\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_name=(
+                            "travel_readiness_replaycheck_bad_name"
+                        ),
+                        safety_token=(
+                            "LIVE_PARSER_REPLAY_DISPOSABLE:"
+                            "travel_readiness_replaycheck_bad_name"
+                        ),
+                    )
+                ),
+                65,
+                "live_parser_replay_check=UNSAFE_DATABASE\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_name=(
+                            "travel_readiness_replaycheck_prod1234"
+                        ),
+                        safety_token=(
+                            "LIVE_PARSER_REPLAY_DISPOSABLE:"
+                            "travel_readiness_replaycheck_prod1234"
+                        ),
+                    )
+                ),
+                65,
+                (
+                    "live_parser_replay_check="
+                    "PRODUCTION_LIKE_DATABASE_REFUSED\n"
+                ),
+            ),
+            (
+                tuple(self.arguments(release_sha="A" * 40)),
+                65,
+                "live_parser_replay_check=INVALID_RELEASE_SHA\n",
+            ),
+        )
+        for arguments, code, stderr in cases:
+            with self.subTest(stderr=stderr):
+                result = self.run_script(None, *arguments, env={})
+                self.assertEqual(result.returncode, code)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, stderr)
+
+    def test_both_secret_references_are_required_without_disclosure(self):
+        environment = os.environ.copy()
+        environment.pop("TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD", None)
+        environment.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)
+        missing_admin = self.run_script(
+            None,
+            *self.arguments(),
+            env=environment,
+        )
+        self.assertEqual(missing_admin.returncode, 66)
+        self.assertEqual(
+            missing_admin.stderr,
+            "live_parser_replay_check=ADMIN_PASSWORD_MISSING\n",
+        )
+
+        admin_marker = "private-admin-password-marker"
+        environment["TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD"] = admin_marker
+        missing_warning = self.run_script(
+            None,
+            *self.arguments(),
+            env=environment,
+        )
+        self.assertEqual(missing_warning.returncode, 66)
+        self.assertEqual(
+            missing_warning.stderr,
+            "live_parser_replay_check=WARNING_SECRET_MISSING\n",
+        )
+        self.assertNotIn(
+            admin_marker,
+            missing_warning.stdout + missing_warning.stderr,
+        )
+
+    def _fake_project(self, temporary: Path):
+        project = temporary / "project"
+        scripts = project / "scripts"
+        python_dir = project / ".venv" / "bin"
+        fake_bin = temporary / "bin"
+        scripts.mkdir(parents=True)
+        python_dir.mkdir(parents=True)
+        fake_bin.mkdir()
+        copied_script = scripts / "check-live-parser-replay"
+        shutil.copy2(self.script, copied_script)
+        (project / "manage.py").write_text("# test placeholder\n", encoding="utf-8")
+
+        fake_python = python_dir / "python"
+        fake_python.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                if [ "${1:-}" = "--version" ]; then
+                    printf '%s\\n' 'Python 3.14.7'
+                    exit 0
+                fi
+                case " $* " in
+                    *" check_live_parser_replay "*)
+                        [ "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" = "${FAKE_WARNING_SECRET}" ] || exit 91
+                        [ "${TRAVEL_READINESS_REPLAY_SAFETY_TOKEN:-}" = "${FAKE_SAFETY_TOKEN}" ] || exit 92
+                        printf '%s\\n' live >> "$FAKE_LOG"
+                        [ "${FAKE_LIVE_FAIL:-0}" = 0 ] || exit 93
+                        printf '%s\\n' 'live_parser_replay_result=VERIFIED sources=2 requests=2 parser_invocations=4 durable_attempts=2 artifacts=2 parse_runs=2 typed=2'
+                        ;;
+                    *)
+                        [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 94
+                        printf '%s\\n' manage >> "$FAKE_LOG"
+                        [ "${FAKE_MANAGE_FAIL:-0}" = 0 ] || exit 97
+                        ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_python.chmod(0o755)
+
+        fake_psql = fake_bin / "psql"
+        fake_psql.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                if [ "${1:-}" = "--version" ]; then
+                    printf '%s\\n' 'psql (PostgreSQL) 18.6'
+                    exit 0
+                fi
+                [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 95
+                input=$(cat)
+                combined="$* $input"
+                case "$combined" in
+                    *"current_setting('server_version_num')"*) printf '%s\\n' 180006 ;;
+                    *"SELECT rolsuper"*) printf '%s\\n' t ;;
+                    *"CREATE DATABASE"*)
+                        : > "$FAKE_PG_STATE"
+                        printf '%s\\n' create >> "$FAKE_LOG"
+                        ;;
+                    *"DROP DATABASE"*)
+                        rm -f "$FAKE_PG_STATE"
+                        printf '%s\\n' drop >> "$FAKE_LOG"
+                        ;;
+                    *"SELECT count(*) FROM pg_catalog.pg_database"*)
+                        if [ -e "$FAKE_PG_STATE" ]; then printf '%s\\n' 1; else printf '%s\\n' 0; fi
+                        ;;
+                    *) : ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_psql.chmod(0o755)
+
+        fake_git = fake_bin / "git"
+        fake_git.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                case "$*" in
+                    *"rev-parse --verify HEAD"*) printf '%s\\n' "$FAKE_RELEASE_SHA" ;;
+                    *"status --porcelain"*)
+                        [ "${FAKE_DIRTY:-0}" = 0 ] || printf '%s\\n' ' M marker'
+                        ;;
+                    *) exit 96 ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_git.chmod(0o755)
+
+        fake_openssl = fake_bin / "openssl"
+        fake_openssl.write_text(
+            "#!/bin/sh\nprintf '%064d\\n' 0\n",
+            encoding="utf-8",
+        )
+        fake_openssl.chmod(0o755)
+        return project, copied_script, fake_bin
+
+    def _fake_environment(self, temporary: Path, fake_bin: Path):
+        environment = os.environ.copy()
+        environment.update(
+            {
+                "PATH": f"{fake_bin}:/usr/bin:/bin",
+                "TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD": "admin-marker",
+                "MOFA_TRAVEL_ALARM_SERVICE_KEY": "warning-marker",
+                "FAKE_WARNING_SECRET": "warning-marker",
+                "FAKE_SAFETY_TOKEN": (
+                    f"LIVE_PARSER_REPLAY_DISPOSABLE:{self.database_name}"
+                ),
+                "FAKE_RELEASE_SHA": self.release_sha,
+                "FAKE_PG_STATE": str(temporary / "database-created"),
+                "FAKE_LOG": str(temporary / "events.log"),
+            }
+        )
+        return environment
+
+    def test_full_fake_rehearsal_is_ordered_and_cleans_the_database(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+            events = Path(environment["FAKE_LOG"]).read_text(
+                encoding="utf-8"
+            ).splitlines()
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(
+            result.stdout,
+            "live_parser_replay_result=VERIFIED sources=2 requests=2 "
+            "parser_invocations=4 durable_attempts=2 artifacts=2 "
+            "parse_runs=2 typed=2 cleanup=match\n",
+        )
+        self.assertFalse(state_exists)
+        self.assertEqual(events, ["create", "manage", "manage", "manage", "manage", "live", "drop"])
+        self.assertNotIn("admin-marker", result.stdout + result.stderr)
+        self.assertNotIn("warning-marker", result.stdout + result.stderr)
+
+    def test_live_failure_still_cleans_the_database(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            environment["FAKE_LIVE_FAIL"] = "1"
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+            events = Path(environment["FAKE_LOG"]).read_text(
+                encoding="utf-8"
+            ).splitlines()
+
+        self.assertEqual(result.returncode, 75)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "live_parser_replay_check=LIVE_REPLAY_FAILED\n",
+        )
+        self.assertFalse(state_exists)
+        self.assertEqual(events[-2:], ["live", "drop"])
+
+    def test_migration_failure_still_cleans_the_database(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            environment["FAKE_MANAGE_FAIL"] = "1"
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+            events = Path(environment["FAKE_LOG"]).read_text(
+                encoding="utf-8"
+            ).splitlines()
+
+        self.assertEqual(result.returncode, 73)
+        self.assertEqual(
+            result.stderr,
+            "live_parser_replay_check=MIGRATION_PLAN_FAILED\n",
+        )
+        self.assertFalse(state_exists)
+        self.assertEqual(events, ["create", "manage", "drop"])
+
+    def test_dirty_tree_is_rejected_before_database_creation(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            environment["FAKE_DIRTY"] = "1"
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+
+        self.assertEqual(result.returncode, 70)
+        self.assertEqual(
+            result.stderr,
+            "live_parser_replay_check=WORKTREE_NOT_CLEAN\n",
+        )
+
+    def test_release_mismatch_is_rejected_before_database_creation(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            environment["FAKE_RELEASE_SHA"] = "b" * 40
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+
+        self.assertEqual(result.returncode, 70)
+        self.assertEqual(
+            result.stderr,
+            "live_parser_replay_check=RELEASE_IDENTITY_MISMATCH\n",
+        )
+        self.assertFalse(state_exists)
+
+    def test_preexisting_target_is_never_mutated_or_cleaned(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            state = Path(environment["FAKE_PG_STATE"])
+            state.touch(mode=0o600)
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = state.exists()
+            log_exists = Path(environment["FAKE_LOG"]).exists()
+
+        self.assertEqual(result.returncode, 71)
+        self.assertEqual(
+            result.stderr,
+            "live_parser_replay_check=TARGET_ALREADY_EXISTS\n",
+        )
+        self.assertTrue(state_exists)
+        self.assertFalse(log_exists)
+
+    def test_script_never_reads_dotenv_or_accepts_source_secret_arguments(self):
+        source = self.script.read_text(encoding="utf-8")
+        lower = source.lower()
+        self.assertNotIn(".env", lower)
+        self.assertNotIn("servicekey=", lower)
+        self.assertNotIn("curl", lower)
+        self.assertNotIn("set -x", lower)
+        self.assertIn("unset", lower)
+        self.assertIn("cleanup_on_exit", source)
+        self.assertIn("register_approved_sources --apply", source)
+        self.assertIn("check_live_parser_replay", source)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/check-live-parser-replay b/scripts/check-live-parser-replay
new file mode 100755
index 0000000..554097e
--- /dev/null
+++ b/scripts/check-live-parser-replay
@@ -0,0 +1,346 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+
+POSTGRESQL_REQUIRED_VERSION="18.6"
+POSTGRESQL_REQUIRED_VERSION_NUM="180006"
+INNER_RECEIPT="live_parser_replay_result=VERIFIED sources=2 requests=2 parser_invocations=4 durable_attempts=2 artifacts=2 parse_runs=2 typed=2"
+
+usage() {
+    printf '%s\n' 'usage: check-live-parser-replay --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD --database-name travel_readiness_replaycheck_NAME --safety-token LIVE_PARSER_REPLAY_DISPOSABLE:travel_readiness_replaycheck_NAME --release-sha SHA'
+}
+
+fail() {
+    printf '%s\n' "$1" >&2
+    exit "$2"
+}
+
+is_identifier() {
+    value=$1
+    [ -n "$value" ] || return 1
+    [ "${#value}" -le 63 ] || return 1
+    case "$value" in
+        [a-z_]* ) ;;
+        * ) return 1 ;;
+    esac
+    case "$value" in
+        *[!a-z0-9_]* ) return 1 ;;
+    esac
+}
+
+is_port() {
+    value=$1
+    case "$value" in
+        ''|*[!0-9]* ) return 1 ;;
+    esac
+    [ "${#value}" -le 5 ] || return 1
+    [ "$value" -ge 1 ] 2>/dev/null || return 1
+    [ "$value" -le 65535 ] 2>/dev/null || return 1
+}
+
+is_release_sha() {
+    value=$1
+    [ "${#value}" -eq 40 ] || return 1
+    case "$value" in
+        *[!0-9a-f]* ) return 1 ;;
+    esac
+}
+
+host=''
+port=''
+admin_role=''
+admin_password_env=''
+database_name=''
+safety_token=''
+release_sha=''
+
+while [ "$#" -gt 0 ]; do
+    case "$1" in
+        --help)
+            [ "$#" -eq 1 ] \
+                || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --host|--port|--admin-role|--admin-password-env|--database-name|--safety-token|--release-sha)
+            [ "$#" -ge 2 ] \
+                || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+            option=$1
+            value=$2
+            shift 2
+            case "$option" in
+                --host)
+                    [ -z "$host" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    host=$value
+                    ;;
+                --port)
+                    [ -z "$port" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    port=$value
+                    ;;
+                --admin-role)
+                    [ -z "$admin_role" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    admin_role=$value
+                    ;;
+                --admin-password-env)
+                    [ -z "$admin_password_env" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    admin_password_env=$value
+                    ;;
+                --database-name)
+                    [ -z "$database_name" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    database_name=$value
+                    ;;
+                --safety-token)
+                    [ -z "$safety_token" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    safety_token=$value
+                    ;;
+                --release-sha)
+                    [ -z "$release_sha" ] \
+                        || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+                    release_sha=$value
+                    ;;
+            esac
+            ;;
+        *) fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+[ -n "$host" ] \
+    && [ -n "$port" ] \
+    && [ -n "$admin_role" ] \
+    && [ -n "$admin_password_env" ] \
+    && [ -n "$database_name" ] \
+    && [ -n "$safety_token" ] \
+    && [ -n "$release_sha" ] \
+    || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+case "$host" in
+    127.0.0.1|localhost|::1) ;;
+    *) fail 'live_parser_replay_check=NON_LOOPBACK_REFUSED' 65 ;;
+esac
+is_port "$port" || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+is_identifier "$admin_role" \
+    || fail 'live_parser_replay_check=INVALID_ARGUMENTS' 64
+is_identifier "$database_name" \
+    || fail 'live_parser_replay_check=UNSAFE_DATABASE' 65
+case "$database_name" in
+    travel_readiness_replaycheck_[a-z0-9]*) ;;
+    *) fail 'live_parser_replay_check=UNSAFE_DATABASE' 65 ;;
+esac
+database_suffix=${database_name#travel_readiness_replaycheck_}
+[ "${#database_suffix}" -ge 8 ] && [ "${#database_suffix}" -le 24 ] \
+    || fail 'live_parser_replay_check=UNSAFE_DATABASE' 65
+case "$database_suffix" in
+    *[!a-z0-9]*) fail 'live_parser_replay_check=UNSAFE_DATABASE' 65 ;;
+esac
+case "$database_suffix" in
+    *prod*|*live*|*stag*|*main*|*master*|*release*)
+        fail 'live_parser_replay_check=PRODUCTION_LIKE_DATABASE_REFUSED' 65
+        ;;
+esac
+[ "$safety_token" = "LIVE_PARSER_REPLAY_DISPOSABLE:$database_name" ] \
+    || fail 'live_parser_replay_check=SAFETY_TOKEN_MISMATCH' 65
+[ "$admin_password_env" = 'TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD' ] \
+    || fail 'live_parser_replay_check=UNSAFE_PASSWORD_REFERENCE' 65
+is_release_sha "$release_sha" \
+    || fail 'live_parser_replay_check=INVALID_RELEASE_SHA' 65
+
+command -v printenv >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=REQUIRED_TOOL_MISSING' 69
+admin_password=$(printenv "$admin_password_env" 2>/dev/null) \
+    || fail 'live_parser_replay_check=ADMIN_PASSWORD_MISSING' 66
+warning_secret=$(printenv MOFA_TRAVEL_ALARM_SERVICE_KEY 2>/dev/null) \
+    || fail 'live_parser_replay_check=WARNING_SECRET_MISSING' 66
+unset TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD
+unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_NAME
+unset TRAVEL_READINESS_DB_USER TRAVEL_READINESS_DB_PASSWORD
+unset TRAVEL_READINESS_DB_HOST TRAVEL_READINESS_DB_PORT
+unset TRAVEL_READINESS_REPLAY_SAFETY_TOKEN
+
+[ -n "$admin_password" ] && [ "${#admin_password}" -le 1024 ] \
+    || fail 'live_parser_replay_check=ADMIN_PASSWORD_INVALID' 66
+[ -n "$warning_secret" ] && [ "${#warning_secret}" -le 512 ] \
+    || fail 'live_parser_replay_check=WARNING_SECRET_INVALID' 66
+case "$admin_password$warning_secret" in
+    *'
+'*) fail 'live_parser_replay_check=SECRET_INVALID' 66 ;;
+esac
+
+script_dir=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+project_dir=$(CDPATH='' cd "$script_dir/.." && pwd -P)
+python_bin="$project_dir/.venv/bin/python"
+
+command -v psql >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=POSTGRESQL_18_6_REQUIRED' 69
+reported_version=$(psql --version 2>/dev/null) \
+    || fail 'live_parser_replay_check=POSTGRESQL_18_6_REQUIRED' 69
+case "$reported_version" in
+    "psql (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION"|"psql (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION "*) ;;
+    *) fail 'live_parser_replay_check=POSTGRESQL_18_6_REQUIRED' 69 ;;
+esac
+command -v openssl >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=REQUIRED_TOOL_MISSING' 69
+command -v git >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=REQUIRED_TOOL_MISSING' 69
+[ -x "$python_bin" ] \
+    || fail 'live_parser_replay_check=PINNED_PYTHON_REQUIRED' 69
+python_version=$($python_bin --version 2>/dev/null) \
+    || fail 'live_parser_replay_check=PINNED_PYTHON_REQUIRED' 69
+[ "$python_version" = 'Python 3.14.7' ] \
+    || fail 'live_parser_replay_check=PINNED_PYTHON_REQUIRED' 69
+
+resolved_sha=$(git -C "$project_dir" rev-parse --verify HEAD 2>/dev/null) \
+    || fail 'live_parser_replay_check=RELEASE_IDENTITY_FAILED' 70
+[ "$resolved_sha" = "$release_sha" ] \
+    || fail 'live_parser_replay_check=RELEASE_IDENTITY_MISMATCH' 70
+worktree_state=$(git -C "$project_dir" status --porcelain \
+    --untracked-files=normal 2>/dev/null) \
+    || fail 'live_parser_replay_check=RELEASE_IDENTITY_FAILED' 70
+[ -z "$worktree_state" ] \
+    || fail 'live_parser_replay_check=WORKTREE_NOT_CLEAN' 70
+
+admin_psql() {
+    connection_database=$1
+    shift
+    PGPASSWORD="$admin_password" \
+    PGAPPNAME=travel-readiness-live-replay-admin \
+    PGCONNECT_TIMEOUT=5 \
+        psql \
+        --no-password \
+        --host="$host" \
+        --port="$port" \
+        --dbname="$connection_database" \
+        --username="$admin_role" \
+        --no-psqlrc \
+        --set=ON_ERROR_STOP=1 \
+        "$@"
+}
+
+admin_scalar() {
+    query=$1
+    admin_psql postgres --quiet --tuples-only --no-align \
+        --command="$query" 2>/dev/null
+}
+
+run_manage() {
+    TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+    TRAVEL_READINESS_DB_NAME="$database_name" \
+    TRAVEL_READINESS_DB_USER="$admin_role" \
+    TRAVEL_READINESS_DB_PASSWORD="$admin_password" \
+    TRAVEL_READINESS_DB_HOST="$host" \
+    TRAVEL_READINESS_DB_PORT="$port" \
+    TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+    TRAVEL_READINESS_RELEASE_SHA="$release_sha" \
+    TRAVEL_READINESS_BUILD=0 \
+    TRAVEL_READINESS_DEBUG=0 \
+    TRAVEL_READINESS_HTTPS=0 \
+    DJANGO_SETTINGS_MODULE=travel_readiness.settings \
+    PYTHONPATH="$project_dir" \
+        "$python_bin" -s "$project_dir/manage.py" "$@"
+}
+
+run_live_replay() {
+    MOFA_TRAVEL_ALARM_SERVICE_KEY="$warning_secret" \
+    TRAVEL_READINESS_REPLAY_SAFETY_TOKEN="$safety_token" \
+        run_manage check_live_parser_replay --verbosity 0
+}
+
+server_version=$(admin_scalar \
+    "SELECT current_setting('server_version_num')") \
+    || fail 'live_parser_replay_check=ADMIN_CONNECTION_FAILED' 70
+[ "$server_version" = "$POSTGRESQL_REQUIRED_VERSION_NUM" ] \
+    || fail 'live_parser_replay_check=DATABASE_VERSION_MISMATCH' 70
+admin_is_superuser=$(admin_scalar \
+    "SELECT rolsuper FROM pg_catalog.pg_roles WHERE rolname = current_user") \
+    || fail 'live_parser_replay_check=ADMIN_CONNECTION_FAILED' 70
+[ "$admin_is_superuser" = 't' ] \
+    || fail 'live_parser_replay_check=ADMIN_CAPABILITY_REQUIRED' 70
+existing_target=$(admin_scalar \
+    "SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$database_name'") \
+    || fail 'live_parser_replay_check=TARGET_PREFLIGHT_FAILED' 70
+[ "$existing_target" = '0' ] \
+    || fail 'live_parser_replay_check=TARGET_ALREADY_EXISTS' 71
+
+django_secret=$(openssl rand -hex 32 2>/dev/null) \
+    || fail 'live_parser_replay_check=SECRET_GENERATION_FAILED' 69
+[ "${#django_secret}" -eq 64 ] \
+    || fail 'live_parser_replay_check=SECRET_GENERATION_FAILED' 69
+case "$django_secret" in
+    *[!0-9a-f]*) fail 'live_parser_replay_check=SECRET_GENERATION_FAILED' 69 ;;
+esac
+
+database_created=0
+
+cleanup() {
+    cleanup_result=0
+    if [ "$database_created" = '1' ]; then
+        admin_psql postgres --quiet \
+            --command="REVOKE CONNECT ON DATABASE \"$database_name\" FROM PUBLIC" \
+            >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet \
+            --command="SELECT pg_catalog.pg_terminate_backend(pid) FROM pg_catalog.pg_stat_activity WHERE datname = '$database_name' AND pid <> pg_catalog.pg_backend_pid()" \
+            >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet \
+            --command="DROP DATABASE \"$database_name\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+        if [ "$cleanup_result" = '0' ]; then
+            database_created=0
+        fi
+    fi
+    remaining=$(admin_scalar \
+        "SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$database_name'" \
+        2>/dev/null) || cleanup_result=1
+    [ "${remaining:-1}" = '0' ] || cleanup_result=1
+    return "$cleanup_result"
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    if ! cleanup; then
+        printf '%s\n' 'live_parser_replay_check_cleanup=FAILED' >&2
+        exit 77
+    fi
+    exit "$original_status"
+}
+
+trap cleanup_on_exit EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+printf '%s\n' \
+    "CREATE DATABASE \"$database_name\" OWNER \"$admin_role\" TEMPLATE template0" \
+    | admin_psql postgres --quiet >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=DATABASE_CREATE_FAILED' 72
+database_created=1
+
+run_manage migrate --plan --noinput --verbosity 0 >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=MIGRATION_PLAN_FAILED' 73
+run_manage migrate --noinput --verbosity 0 >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=MIGRATION_FAILED' 73
+run_manage makemigrations --check --dry-run --verbosity 0 >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=MIGRATION_DRIFT' 73
+run_manage register_approved_sources --apply --verbosity 0 >/dev/null 2>&1 \
+    || fail 'live_parser_replay_check=SOURCE_REGISTRATION_FAILED' 74
+
+live_receipt=$(run_live_replay 2>/dev/null) \
+    || fail 'live_parser_replay_check=LIVE_REPLAY_FAILED' 75
+[ "$live_receipt" = "$INNER_RECEIPT" ] \
+    || fail 'live_parser_replay_check=LIVE_RECEIPT_MISMATCH' 75
+
+cleanup || fail 'live_parser_replay_check_cleanup=FAILED' 77
+trap - EXIT HUP INT TERM
+warning_secret=''
+admin_password=''
+django_secret=''
+printf '%s\n' "$INNER_RECEIPT cleanup=match"


