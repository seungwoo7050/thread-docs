## `feat(warnings): accept complete official snapshots`

diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index 030be75..a2d0fef 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -4,6 +4,13 @@ from django.core.management.base import BaseCommand, CommandError
 from django.db import DatabaseError, connection, transaction
 
 from sources.models import SourceConfiguration, SourceRightsDecision
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_DECISION_BASIS,
+    WARNING_SNAPSHOT_FIELD_SCOPE,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
+)
 from travel_windows.contracts import (
     AVIATION_PRIOR_SOURCE_REVISION,
     CITY_ATTRIBUTION,
@@ -228,6 +235,24 @@ APPROVED_SOURCE_SPECS = (
         decision_basis_code="USER_TRAVEL6_SCOPE_20260831",
         attribution_text="외교부|공공데이터포털",
     ),
+    ApprovedSourceSpec(
+        code=WARNING_SNAPSHOT_SOURCE_CODE,
+        revision=WARNING_SNAPSHOT_SOURCE_REVISION,
+        module=SourceConfiguration.Module.TRAVEL_WARNING,
+        owner="대한민국 외교부",
+        official_locator=(
+            "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+            "getTravelAlarmList2"
+        ),
+        secret_reference_name=DATA_GO_SECRET_REFERENCE,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code=WARNING_SNAPSHOT_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        ),
+        decision_basis_code=WARNING_SNAPSHOT_DECISION_BASIS,
+        attribution_text="외교부|공공데이터포털",
+    ),
     _with_prior_aviation_contract(
         ApprovedSourceSpec(
             code=SCHEDULE_SOURCE_CODE,
diff --git a/sources/migrations/0017_warning_snapshot_parser_contract.py b/sources/migrations/0017_warning_snapshot_parser_contract.py
new file mode 100644
index 0000000..1ba1f44
--- /dev/null
+++ b/sources/migrations/0017_warning_snapshot_parser_contract.py
@@ -0,0 +1,95 @@
+from django.db import migrations
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+OLD_WARNING_CLAUSE = """            OR (source_module = 'TRAVEL_WARNING'
+                AND NEW.parser_name = 'MOFA_TRAVEL_ALARM_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 IN (
+                    'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6',
+                    '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+                )
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b')"""
+NEW_WARNING_CLAUSE = """            OR (source_module = 'TRAVEL_WARNING'
+                AND NEW.parser_name = 'MOFA_TRAVEL_ALARM_JSON'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d')"""
+
+
+def _function_body(schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [FUNCTION_NAME],
+        )
+        rows = cursor.fetchall()
+    if len(rows) != 1:
+        raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+    return rows[0][0]
+
+
+def _replace_clause(schema_editor, old, new):
+    body = _function_body(schema_editor)
+    if body.count(old) != 1:
+        raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+    body = body.replace(old, new)
+    quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $warning_snapshot_parser_contract$
+            {body}
+            $warning_snapshot_parser_contract$
+            """
+        )
+
+
+def allow_warning_snapshot_parser(apps, schema_editor):
+    _replace_clause(
+        schema_editor,
+        OLD_WARNING_CLAUSE,
+        OLD_WARNING_CLAUSE + "\n" + NEW_WARNING_CLAUSE,
+    )
+
+
+def restore_warning_v1_parsers(apps, schema_editor):
+    ParseRun = apps.get_model("sources", "ParseRun")
+    if ParseRun.objects.filter(
+        parser_name="MOFA_TRAVEL_ALARM_JSON",
+        parser_version="V2",
+        parser_contract_fingerprint_sha256=(
+            "23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4"
+        ),
+    ).exists():
+        raise RuntimeError(
+            "warning snapshot parser rollback requires no v2 parse runs"
+        )
+    _replace_clause(
+        schema_editor,
+        OLD_WARNING_CLAUSE + "\n" + NEW_WARNING_CLAUSE,
+        OLD_WARNING_CLAUSE,
+    )
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0016_route_duration_reference_contract")]
+
+    operations = [
+        migrations.RunPython(
+            allow_warning_snapshot_parser,
+            restore_warning_v1_parsers,
+        )
+    ]
diff --git a/sources/tests/test_parse_run.py b/sources/tests/test_parse_run.py
index ff0ded6..8c7174d 100644
--- a/sources/tests/test_parse_run.py
+++ b/sources/tests/test_parse_run.py
@@ -20,6 +20,10 @@ from sources.models import (
     SourceConfiguration,
     SourceRightsDecision,
 )
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+)
 
 
 class ParseRunFixtureMixin:
@@ -174,6 +178,38 @@ class ParseRunTests(ParseRunFixtureMixin, TestCase):
         with self.assertRaises(ProtectedError):
             self.artifact.delete()
 
+    def test_exact_warning_snapshot_v2_parser_contract_is_registered(self):
+        run = ParseRun.objects.create(
+            **self.run_values(
+                self.artifact,
+                parser_version=ParseRun.ParserVersion.V2,
+                parser_contract_fingerprint_sha256=(
+                    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+                ),
+                expected_schema_fingerprint_sha256=(
+                    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+            )
+        )
+
+        self.assertEqual(run.parser_version, ParseRun.ParserVersion.V2)
+        other = self.make_artifact(
+            suffix="WARNING_V2_WRONG",
+            body_sha256="c" * 64,
+        )
+        self.assert_integrity_error(
+            lambda: ParseRun.objects.create(
+                **self.run_values(
+                    other,
+                    parser_version=ParseRun.ParserVersion.V2,
+                    parser_contract_fingerprint_sha256="f" * 64,
+                    expected_schema_fingerprint_sha256=(
+                        WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                    ),
+                )
+            )
+        )
+
     def test_direct_sql_cannot_insert_a_terminal_run(self):
         artifact = self.make_artifact("DIRECT", "2" * 64)
         with self.assertRaises(IntegrityError), transaction.atomic(), connection.cursor() as cursor:
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index c6a8754..7990380 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -245,6 +245,9 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
         legacy_warning = specs["MOFA_TRAVEL_ALARM_API_JP"]
         multi_entry = specs["MOFA_ENTRY_CSV_TRAVEL6"]
         multi_warning = specs["MOFA_TRAVEL_ALARM_API_TRAVEL6"]
+        snapshot_warning = specs[
+            "MOFA_TRAVEL_ALARM_API_TRAVEL6_SET"
+        ]
 
         self.assertEqual(legacy_entry.revision, "rights-v1")
         self.assertEqual(legacy_entry.field_scope_code, "JP_ORDINARY_TEXT_V1")
@@ -268,11 +271,28 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             multi_warning.secret_reference_name,
             CANONICAL_SECRET_REFERENCE_NAME,
         )
+        self.assertEqual(snapshot_warning.revision, "travel6-set-v2")
+        self.assertEqual(
+            snapshot_warning.field_scope_code,
+            "SUPPORTED_6_WARNING_SET_V2",
+        )
+        self.assertEqual(
+            snapshot_warning.secret_reference_name,
+            CANONICAL_SECRET_REFERENCE_NAME,
+        )
+        self.assertEqual(
+            snapshot_warning.contract_fingerprint_sha256,
+            "23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4",
+        )
         self.assertEqual(legacy_entry.official_locator, multi_entry.official_locator)
         self.assertEqual(
             legacy_warning.official_locator,
             multi_warning.official_locator,
         )
+        self.assertEqual(
+            multi_warning.official_locator,
+            snapshot_warning.official_locator,
+        )
 
         legacy_entry_row = self.make_exact_source(legacy_entry)
         legacy_entry_rights = self.make_exact_approval(
@@ -322,6 +342,13 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
                 enabled=True,
             ).exists()
         )
+        self.assertTrue(
+            SourceConfiguration.objects.filter(
+                code="MOFA_TRAVEL_ALARM_API_TRAVEL6_SET",
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+            ).exists()
+        )
 
     def test_unregistered_same_locator_source_still_fails_closed(self):
         entry_spec = registration.APPROVED_SOURCE_SPECS[0]
diff --git a/sources/tests/test_transport.py b/sources/tests/test_transport.py
index c6a0d43..1f5b8a4 100644
--- a/sources/tests/test_transport.py
+++ b/sources/tests/test_transport.py
@@ -25,6 +25,7 @@ from sources.transport import (
     PROVIDER_SUCCESS_0,
     WARNING_MAX_RESPONSE_BYTES,
     WARNING_REQUEST_FINGERPRINT,
+    WARNING_SNAPSHOT_REQUEST_FINGERPRINT,
     WARNING_SECRET_REFERENCE,
     WARNING_SOURCE_LOCATOR,
     ROLE_DESTINATION_CITY,
@@ -44,9 +45,15 @@ from sources.transport import (
     fetch_entry_attachment,
     fetch_route_duration_reference,
     fetch_travel_alarm_jp,
+    fetch_travel_alarm_snapshot,
     legacy_arrival_request_fingerprint,
     schedule_page_request_fingerprint,
     warning_request_fingerprint,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+    WARNING_SNAPSHOT_PAGE_SIZE,
 )
 
 
@@ -238,6 +245,25 @@ class TransportTestCase(unittest.TestCase):
             connection_factory=factory or FakeFactory(connection),
         )
 
+    def warning_snapshot_fetch(
+        self,
+        connection,
+        *,
+        secret=None,
+        country_iso2="JP",
+    ):
+        if secret is None:
+            secret = "%2B".join(("Q7vR4mP2", "zN8kL6xT")) + "%3D"
+        return fetch_travel_alarm_snapshot(
+            country_iso2=country_iso2,
+            official_locator=WARNING_SOURCE_LOCATOR,
+            secret_reference_name=WARNING_SECRET_REFERENCE,
+            secret_value=secret,
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=FakeFactory(connection),
+        )
+
     def assert_sensitive_absent(self, result, *values):
         rendered = repr(result)
         if any(value in rendered for value in values):
@@ -254,6 +280,64 @@ class TransportTestCase(unittest.TestCase):
             WARNING_REQUEST_FINGERPRINT.normalized_request_sha256,
             "e9ed083968ebdcd1341df0fbb0da3014ce62f61ba8114a558446b30cfe21f913",
         )
+        self.assertEqual(
+            WARNING_SNAPSHOT_REQUEST_FINGERPRINT.revision,
+            "MOFA_WARNING_V2",
+        )
+        self.assertEqual(
+            WARNING_SNAPSHOT_REQUEST_FINGERPRINT.normalized_request_sha256,
+            "556d714c8151dd5b4ef2b51573a456e0695bc420b27d957347f3b9e84cc6c8d3",
+        )
+
+    def test_warning_snapshot_requests_one_complete_bounded_country_page(self):
+        response_body = warning_body()
+        connection = FakeConnection(FakeResponse(200, response_body))
+
+        result = self.warning_snapshot_fetch(
+            connection,
+            country_iso2="TH",
+        )
+
+        self.assertEqual(result.attempt_result, ATTEMPT_SUCCEEDED)
+        self.assertEqual(
+            result.request_fingerprint,
+            warning_snapshot_request_fingerprint("TH"),
+        )
+        parameters = parse_qs(
+            urlsplit(connection.requests[0][1]).query,
+            keep_blank_values=True,
+        )
+        self.assertEqual(
+            parameters,
+            {
+                "ServiceKey": ["+".join(("Q7vR4mP2", "zN8kL6xT")) + "="],
+                "returnType": ["JSON"],
+                "numOfRows": [str(WARNING_SNAPSHOT_PAGE_SIZE)],
+                "pageNo": ["1"],
+                "cond[country_iso_alp2::EQ]": ["TH"],
+            },
+        )
+        self.assertEqual(
+            connection.response.read_amounts,
+            [WARNING_SNAPSHOT_MAX_RESPONSE_BYTES + 1],
+        )
+
+        oversized = self.warning_snapshot_fetch(
+            FakeConnection(
+                FakeResponse(
+                    200,
+                    b"",
+                    content_length=str(
+                        WARNING_SNAPSHOT_MAX_RESPONSE_BYTES + 1
+                    ),
+                )
+            ),
+            country_iso2="TH",
+        )
+        self.assertEqual(
+            (oversized.attempt_result, oversized.failure_code),
+            (ATTEMPT_TERMINAL_FAILED, FAILURE_RESPONSE_TOO_LARGE),
+        )
 
     def test_duration_reference_discovers_captcha_checks_and_fetches_archive(self):
         archive = b"PK synthetic official archive"
diff --git a/sources/transport.py b/sources/transport.py
index 3db5262..b89421d 100644
--- a/sources/transport.py
+++ b/sources/transport.py
@@ -20,6 +20,10 @@ from typing import Callable, Protocol
 from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsplit
 from xml.etree import ElementTree
 
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+    WARNING_SNAPSHOT_PAGE_SIZE,
+)
 from travel_windows.contracts import (
     CITY_SOURCE_CODE,
     CITY_SOURCE_LOCATOR,
@@ -280,6 +284,17 @@ def _warning_fixed_query(country_iso2: str) -> tuple[tuple[str, str], ...]:
     )
 
 
+def _warning_snapshot_fixed_query(
+    country_iso2: str,
+) -> tuple[tuple[str, str], ...]:
+    return (
+        ("returnType", "JSON"),
+        ("numOfRows", str(WARNING_SNAPSHOT_PAGE_SIZE)),
+        ("pageNo", "1"),
+        ("cond[country_iso_alp2::EQ]", country_iso2),
+    )
+
+
 def warning_request_fingerprint(country_iso2: str) -> RequestFingerprint:
     if (
         type(country_iso2) is not str
@@ -301,6 +316,32 @@ def warning_request_fingerprint(country_iso2: str) -> RequestFingerprint:
 
 WARNING_REQUEST_FINGERPRINT = warning_request_fingerprint("JP")
 
+
+def warning_snapshot_request_fingerprint(
+    country_iso2: str,
+) -> RequestFingerprint:
+    if (
+        type(country_iso2) is not str
+        or country_iso2 not in SUPPORTED_WARNING_COUNTRY_ISO2
+    ):
+        raise ValueError("unsupported warning country identity")
+    return RequestFingerprint(
+        revision="MOFA_WARNING_V2",
+        normalized_request_sha256=_request_hash(
+            _warning_parts.hostname or "",
+            _warning_parts.path,
+            [
+                ("ServiceKey", "<redacted>"),
+                *_warning_snapshot_fixed_query(country_iso2),
+            ],
+        ),
+    )
+
+
+WARNING_SNAPSHOT_REQUEST_FINGERPRINT = warning_snapshot_request_fingerprint(
+    "JP"
+)
+
 _COMMON_REQUEST_HEADERS = {
     "Connection": "close",
     "User-Agent": "travel-readiness-source-fetch/1",
@@ -1294,6 +1335,119 @@ def fetch_travel_alarm_jp(
     )
 
 
+def fetch_travel_alarm_snapshot(
+    *,
+    country_iso2: str,
+    official_locator: str,
+    secret_reference_name: str,
+    secret_value: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory = _default_connection_factory,
+) -> SingleAttemptResult:
+    """Fetch one bounded, complete country warning snapshot candidate."""
+
+    try:
+        fingerprint = warning_snapshot_request_fingerprint(country_iso2)
+    except (TypeError, ValueError):
+        return _failed(
+            WARNING_SNAPSHOT_REQUEST_FINGERPRINT,
+            FAILURE_HTTP_CLIENT,
+        )
+    if (
+        official_locator != WARNING_SOURCE_LOCATOR
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+    ):
+        return _failed(fingerprint, FAILURE_HTTP_CLIENT)
+    if (
+        secret_reference_name != WARNING_SECRET_REFERENCE
+        or not isinstance(secret_value, str)
+        or not secret_value
+        or len(secret_value) > _MAX_SECRET_CHARACTERS
+    ):
+        return _failed(fingerprint, FAILURE_AUTHENTICATION)
+
+    try:
+        decoded_secret = unquote(secret_value, encoding="utf-8", errors="strict")
+        query = urlencode(
+            [
+                ("ServiceKey", decoded_secret),
+                *_warning_snapshot_fixed_query(country_iso2),
+            ],
+            doseq=False,
+        )
+    except (UnicodeError, TypeError, ValueError):
+        return _failed(fingerprint, FAILURE_AUTHENTICATION)
+
+    parts = urlsplit(WARNING_SOURCE_LOCATOR)
+    wire_result = _read_once(
+        host=parts.hostname or "",
+        request_target=f"{parts.path}?{query}",
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        max_response_bytes=WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+        request_headers=_WARNING_REQUEST_HEADERS,
+        connection_factory=connection_factory,
+    )
+    if isinstance(wire_result, _WireFailure):
+        if wire_result.http_status is not None:
+            http_failure = _classify_http_failure(
+                fingerprint,
+                wire_result.http_status,
+            )
+            if http_failure is not None:
+                return http_failure
+        return _failed(
+            fingerprint,
+            wire_result.failure_code,
+            http_status=(
+                wire_result.http_status
+                if wire_result.failure_code == FAILURE_RESPONSE_TOO_LARGE
+                else None
+            ),
+        )
+
+    if _contains_secret_reflection(
+        wire_result.body,
+        secret_value,
+        decoded_secret,
+    ):
+        return _failed(
+            fingerprint,
+            FAILURE_SECRET_REFLECTION,
+            http_status=wire_result.status,
+        )
+
+    provider_result_code = _provider_result_code(wire_result.body)
+    provider_code = _closed_provider_code(provider_result_code)
+    http_provider_code = (
+        provider_code if provider_result_code not in (None, "0") else ""
+    )
+    http_failure = _classify_http_failure(
+        fingerprint,
+        wire_result.status,
+        provider_code=http_provider_code,
+    )
+    if http_failure is not None:
+        return http_failure
+    if provider_result_code is not None and provider_result_code != "0":
+        return _failed(
+            fingerprint,
+            FAILURE_PROVIDER_ERROR,
+            http_status=wire_result.status,
+            provider_code=provider_code,
+        )
+    return _succeeded(
+        fingerprint,
+        http_status=wire_result.status,
+        body=wire_result.body,
+        provider_code=(
+            PROVIDER_SUCCESS_0 if provider_result_code == "0" else ""
+        ),
+    )
+
+
 def _schedule_page_count(body: bytes) -> int | None:
     try:
         document = json.loads(body.decode("utf-8"))
diff --git a/travel_warnings/contracts.py b/travel_warnings/contracts.py
new file mode 100644
index 0000000..286042f
--- /dev/null
+++ b/travel_warnings/contracts.py
@@ -0,0 +1,28 @@
+"""Append-only contracts for complete country travel-warning snapshots."""
+
+from __future__ import annotations
+
+import hashlib
+
+
+WARNING_SNAPSHOT_SOURCE_CODE = "MOFA_TRAVEL_ALARM_API_TRAVEL6_SET"
+WARNING_SNAPSHOT_SOURCE_REVISION = "travel6-set-v2"
+WARNING_SNAPSHOT_FIELD_SCOPE = "SUPPORTED_6_WARNING_SET_V2"
+WARNING_SNAPSHOT_DECISION_BASIS = "LIVE_WARNING_SET_SHAPES_20260831"
+
+WARNING_SNAPSHOT_PAGE_SIZE = 100
+WARNING_SNAPSHOT_MAX_FACTS = 100
+WARNING_SNAPSHOT_MAX_RESPONSE_BYTES = 262_144
+
+WARNING_SNAPSHOT_PARSER_CONTRACT_TEXT = (
+    "MOFA TravelAlarm JSON V2|complete country snapshot|page 1 size 100|"
+    "supported countries JP,TW,HK,MO,VN,TH|zero to 100 ordered regional facts|"
+    "exact typed alarm level, scope, written date|request-bound country identity|"
+    "raw body retention zero"
+)
+WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256 = hashlib.sha256(
+    WARNING_SNAPSHOT_PARSER_CONTRACT_TEXT.encode("utf-8")
+).hexdigest()
+WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256 = (
+    "2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d"
+)
diff --git a/travel_warnings/parser.py b/travel_warnings/parser.py
index c9c9b7c..4cf1cdb 100644
--- a/travel_warnings/parser.py
+++ b/travel_warnings/parser.py
@@ -9,6 +9,13 @@ from typing import Any
 
 from countries.models import SUPPORTED_COUNTRY_ROWS
 from sources.models import ParseRun
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_MAX_FACTS,
+    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+    WARNING_SNAPSHOT_PAGE_SIZE,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+)
 from travel_warnings.models import (
     SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
     SOURCE_SCOPE_TEXT_MAX_LENGTH,
@@ -61,6 +68,7 @@ _WRITTEN_DATE_PATH = (
     "[]",
     "written_dt",
 )
+_ITEM_LIST_PATH = ("response", "body", "items", "item")
 _NULLABLE_LINK_PATHS = frozenset(
     {
         (
@@ -120,6 +128,26 @@ class TravelWarningParseFailure:
 TravelWarningParseResult = TravelWarningParseSuccess | TravelWarningParseFailure
 
 
+@dataclass(frozen=True, slots=True)
+class ParsedTravelWarningSnapshot:
+    country_iso2: str
+    country_name_ko: str
+    country_name_en: str
+    facts: tuple[ParsedTravelWarning, ...]
+    typed_fingerprint_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class TravelWarningSnapshotParseSuccess:
+    snapshot: ParsedTravelWarningSnapshot
+    observed_schema_fingerprint_sha256: str
+
+
+TravelWarningSnapshotParseResult = (
+    TravelWarningSnapshotParseSuccess | TravelWarningParseFailure
+)
+
+
 def _object_without_duplicate_keys(pairs: list[tuple[str, Any]]) -> dict[str, Any]:
     result: dict[str, Any] = {}
     for key, value in pairs:
@@ -164,6 +192,29 @@ def _schema_shape(value: Any, path: tuple[str, ...] = ()) -> Any:
     return "unknown"
 
 
+def _snapshot_envelope_shape(value: Any, path: tuple[str, ...] = ()) -> Any:
+    if path == _ITEM_LIST_PATH and isinstance(value, list):
+        return "array"
+    if isinstance(value, dict):
+        return {
+            key: _snapshot_envelope_shape(child, (*path, key))
+            for key, child in sorted(value.items())
+        }
+    if value is None:
+        return "null"
+    if isinstance(value, bool):
+        return "boolean"
+    if isinstance(value, int):
+        return "integer"
+    if isinstance(value, float):
+        return "number"
+    if isinstance(value, str):
+        return "string"
+    if isinstance(value, list):
+        return "array"
+    return "unknown"
+
+
 def _fingerprint(value: Any) -> str:
     canonical = json.dumps(
         value,
@@ -211,6 +262,37 @@ def _typed_fingerprint(item: dict[str, Any], written_date: date | None) -> str:
     )
 
 
+def warning_snapshot_typed_fingerprint(
+    *,
+    country_iso2: str,
+    country_name_ko: str,
+    country_name_en: str,
+    facts: tuple[ParsedTravelWarning, ...],
+) -> str:
+    """Hash a complete ordered typed snapshot without deriving a judgment."""
+
+    return _fingerprint(
+        {
+            "country_iso2": country_iso2,
+            "country_name_en": country_name_en,
+            "country_name_ko": country_name_ko,
+            "facts": [
+                {
+                    "source_alarm_level_code": fact.source_alarm_level_code,
+                    "source_scope_text": fact.source_scope_text,
+                    "source_scope_type": fact.source_scope_type,
+                    "source_written_date": (
+                        fact.source_written_date.isoformat()
+                        if fact.source_written_date is not None
+                        else None
+                    ),
+                }
+                for fact in facts
+            ],
+        }
+    )
+
+
 def parse_travel_alarm_jp(
     payload: bytes,
     *,
@@ -300,3 +382,156 @@ def parse_travel_alarm_jp(
         warning=warning,
         observed_schema_fingerprint_sha256=observed_schema,
     )
+
+
+def parse_travel_alarm_snapshot(
+    payload: bytes,
+    *,
+    country_iso2: str,
+) -> TravelWarningSnapshotParseResult:
+    """Parse an exact, complete 0..100-row country warning snapshot."""
+
+    if type(payload) is not bytes or type(country_iso2) is not str:
+        return _failure(ParseRun.FailureCode.INTERNAL_PARSER_ERROR)
+    expected_identity = _SUPPORTED_COUNTRY_IDENTITIES.get(country_iso2)
+    if expected_identity is None:
+        return _failure(ParseRun.FailureCode.IDENTITY_MISMATCH)
+    if len(payload) > WARNING_SNAPSHOT_MAX_RESPONSE_BYTES:
+        return _failure(ParseRun.FailureCode.SYNTAX_ERROR)
+    try:
+        document = json.loads(
+            payload.decode("utf-8", errors="strict"),
+            object_pairs_hook=_object_without_duplicate_keys,
+            parse_constant=_reject_non_json_constant,
+        )
+    except UnicodeDecodeError:
+        return _failure(ParseRun.FailureCode.ENCODING_ERROR)
+    except _DuplicateKey:
+        return _failure(ParseRun.FailureCode.SYNTAX_ERROR)
+    except (json.JSONDecodeError, ValueError, TypeError, RecursionError):
+        return _failure(ParseRun.FailureCode.SYNTAX_ERROR)
+
+    try:
+        observed_schema = _fingerprint(
+            _snapshot_envelope_shape(document)
+        )
+    except (RecursionError, UnicodeEncodeError):
+        return _failure(ParseRun.FailureCode.SYNTAX_ERROR)
+    if (
+        observed_schema
+        != WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+    ):
+        return _failure(ParseRun.FailureCode.SCHEMA_MISMATCH, observed_schema)
+
+    response = document["response"]
+    header = response["header"]
+    body = response["body"]
+    items = body["items"]["item"]
+    total_count = body["totalCount"]
+    if header["resultCode"] != "0":
+        return _failure(ParseRun.FailureCode.INVALID_VALUE, observed_schema)
+    if (
+        body["pageNo"] != 1
+        or body["numOfRows"] != WARNING_SNAPSHOT_PAGE_SIZE
+        or total_count < 0
+        or total_count > WARNING_SNAPSHOT_MAX_FACTS
+        or len(items) != total_count
+    ):
+        return _failure(ParseRun.FailureCode.INVALID_VALUE, observed_schema)
+
+    expected_name_ko, expected_name_en = expected_identity
+    facts: list[ParsedTravelWarning] = []
+    observed_fact_fingerprints: set[str] = set()
+    nullable_fields = frozenset(
+        ("flag_download_url", "map_download_url", "written_dt")
+    )
+    for item in items:
+        if type(item) is not dict or set(item) != ITEM_KEYS:
+            return _failure(
+                ParseRun.FailureCode.SCHEMA_MISMATCH,
+                observed_schema,
+            )
+        if any(
+            type(item[field]) is not str
+            for field in ITEM_KEYS - nullable_fields
+        ) or any(
+            item[field] is not None and type(item[field]) is not str
+            for field in nullable_fields
+        ):
+            return _failure(
+                ParseRun.FailureCode.SCHEMA_MISMATCH,
+                observed_schema,
+            )
+        if (
+            item["country_iso_alp2"] != country_iso2
+            or item["country_nm"] != expected_name_ko
+            or item["country_eng_nm"] != expected_name_en
+        ):
+            return _failure(
+                ParseRun.FailureCode.IDENTITY_MISMATCH,
+                observed_schema,
+            )
+        if any(
+            not item[field].strip()
+            for field in ("alarm_lvl", "region_ty", "remark")
+        ):
+            return _failure(
+                ParseRun.FailureCode.REQUIRED_VALUE_MISSING,
+                observed_schema,
+            )
+        if (
+            len(item["alarm_lvl"]) > SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+            or len(item["region_ty"]) > SOURCE_SCOPE_TYPE_MAX_LENGTH
+            or len(item["remark"]) > SOURCE_SCOPE_TEXT_MAX_LENGTH
+        ):
+            return _failure(
+                ParseRun.FailureCode.INVALID_VALUE,
+                observed_schema,
+            )
+        try:
+            written_date = _parse_written_date(item["written_dt"])
+            fact_fingerprint = _typed_fingerprint(item, written_date)
+        except (ValueError, UnicodeEncodeError):
+            return _failure(
+                ParseRun.FailureCode.INVALID_VALUE,
+                observed_schema,
+            )
+        if fact_fingerprint in observed_fact_fingerprints:
+            return _failure(
+                ParseRun.FailureCode.DUPLICATE_RECORD,
+                observed_schema,
+            )
+        observed_fact_fingerprints.add(fact_fingerprint)
+        facts.append(
+            ParsedTravelWarning(
+                country_iso2=item["country_iso_alp2"],
+                country_name_ko=item["country_nm"],
+                country_name_en=item["country_eng_nm"],
+                source_alarm_level_code=item["alarm_lvl"],
+                source_scope_type=item["region_ty"],
+                source_scope_text=item["remark"],
+                source_written_date=written_date,
+                typed_fingerprint_sha256=fact_fingerprint,
+            )
+        )
+
+    facts_tuple = tuple(facts)
+    try:
+        typed_fingerprint = warning_snapshot_typed_fingerprint(
+            country_iso2=country_iso2,
+            country_name_ko=expected_name_ko,
+            country_name_en=expected_name_en,
+            facts=facts_tuple,
+        )
+    except UnicodeEncodeError:
+        return _failure(ParseRun.FailureCode.INVALID_VALUE, observed_schema)
+    return TravelWarningSnapshotParseSuccess(
+        snapshot=ParsedTravelWarningSnapshot(
+            country_iso2=country_iso2,
+            country_name_ko=expected_name_ko,
+            country_name_en=expected_name_en,
+            facts=facts_tuple,
+            typed_fingerprint_sha256=typed_fingerprint,
+        ),
+        observed_schema_fingerprint_sha256=observed_schema,
+    )
diff --git a/travel_warnings/tests/test_parser.py b/travel_warnings/tests/test_parser.py
index 042ccdf..f6847e4 100644
--- a/travel_warnings/tests/test_parser.py
+++ b/travel_warnings/tests/test_parser.py
@@ -16,9 +16,17 @@ from travel_warnings.parser import (
     ITEM_KEYS,
     MAX_RESPONSE_BYTES,
     ParsedTravelWarning,
+    ParsedTravelWarningSnapshot,
     TravelWarningParseFailure,
     TravelWarningParseSuccess,
+    TravelWarningSnapshotParseSuccess,
     parse_travel_alarm_jp,
+    parse_travel_alarm_snapshot,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_MAX_FACTS,
+    WARNING_SNAPSHOT_PAGE_SIZE,
 )
 
 
@@ -68,6 +76,22 @@ class TravelAlarmParserTests(SimpleTestCase):
             **options,
         ).encode("utf-8")
 
+    def snapshot_document(self, *, items, **body_overrides):
+        body = {
+            "dataType": "JSON",
+            "items": {"item": items},
+            "numOfRows": WARNING_SNAPSHOT_PAGE_SIZE,
+            "pageNo": 1,
+            "totalCount": len(items),
+        }
+        body.update(body_overrides)
+        return {
+            "response": {
+                "header": {"resultCode": "0", "resultMsg": "SYNTHETIC"},
+                "body": body,
+            }
+        }
+
     def assert_failure(self, payload, code, *, observed=None):
         result = parse_travel_alarm_jp(payload)
         self.assertIsInstance(result, TravelWarningParseFailure)
@@ -96,6 +120,97 @@ class TravelAlarmParserTests(SimpleTestCase):
             EXPECTED_SCHEMA_FINGERPRINT_SHA256,
         )
 
+    def test_v2_accepts_exact_empty_and_complete_regional_snapshots(self):
+        empty = parse_travel_alarm_snapshot(
+            self.encode(self.snapshot_document(items=[])),
+            country_iso2="TW",
+        )
+        thai_items = [
+            self.item(
+                country_iso_alp2="TH",
+                country_nm="태국",
+                country_eng_nm="Thailand",
+                alarm_lvl=str(level),
+                region_ty=f"지역 {position}",
+                remark=f"공식 범위 {position}",
+                written_dt="2026-08-31" if position == 2 else None,
+            )
+            for position, level in enumerate((1, 2, 3), start=1)
+        ]
+        regional = parse_travel_alarm_snapshot(
+            self.encode(self.snapshot_document(items=thai_items)),
+            country_iso2="TH",
+        )
+
+        self.assertIsInstance(empty, TravelWarningSnapshotParseSuccess)
+        self.assertIsInstance(empty.snapshot, ParsedTravelWarningSnapshot)
+        self.assertEqual(empty.snapshot.country_iso2, "TW")
+        self.assertEqual(empty.snapshot.facts, ())
+        self.assertEqual(
+            empty.observed_schema_fingerprint_sha256,
+            WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+        )
+        empty_canonical = json.dumps(
+            {
+                "country_iso2": "TW",
+                "country_name_en": "Taiwan",
+                "country_name_ko": "대만",
+                "facts": [],
+            },
+            ensure_ascii=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        ).encode("utf-8")
+        self.assertEqual(
+            empty.snapshot.typed_fingerprint_sha256,
+            hashlib.sha256(empty_canonical).hexdigest(),
+        )
+        self.assertIsInstance(regional, TravelWarningSnapshotParseSuccess)
+        self.assertEqual(
+            [fact.source_scope_text for fact in regional.snapshot.facts],
+            ["공식 범위 1", "공식 범위 2", "공식 범위 3"],
+        )
+        self.assertEqual(
+            [fact.source_alarm_level_code for fact in regional.snapshot.facts],
+            ["1", "2", "3"],
+        )
+        self.assertNotEqual(
+            empty.snapshot.typed_fingerprint_sha256,
+            regional.snapshot.typed_fingerprint_sha256,
+        )
+        self.assertRegex(
+            regional.snapshot.typed_fingerprint_sha256,
+            r"^[0-9a-f]{64}$",
+        )
+
+    def test_v2_fails_closed_when_complete_snapshot_cannot_be_proved(self):
+        item = self.item()
+        cases = (
+            self.snapshot_document(items=[item], totalCount=2),
+            self.snapshot_document(
+                items=[item] * (WARNING_SNAPSHOT_MAX_FACTS + 1),
+            ),
+            self.snapshot_document(items=[item, dict(item)]),
+            self.snapshot_document(
+                items=[self.item(country_iso_alp2="KR")]
+            ),
+        )
+
+        expected_codes = (
+            ParseRun.FailureCode.INVALID_VALUE,
+            ParseRun.FailureCode.INVALID_VALUE,
+            ParseRun.FailureCode.DUPLICATE_RECORD,
+            ParseRun.FailureCode.IDENTITY_MISMATCH,
+        )
+        for document, expected_code in zip(cases, expected_codes, strict=True):
+            with self.subTest(expected_code=expected_code):
+                result = parse_travel_alarm_snapshot(
+                    self.encode(document),
+                    country_iso2="JP",
+                )
+                self.assertIsInstance(result, TravelWarningParseFailure)
+                self.assertEqual(result.failure_code, expected_code)
+
     def test_optional_provider_links_accept_null_without_schema_drift(self):
         for field in ("flag_download_url", "map_download_url"):
             with self.subTest(field=field):


