## `feat(ops): expose bounded health signals`

diff --git a/config/urls.py b/config/urls.py
index 052f697..f480012 100644
--- a/config/urls.py
+++ b/config/urls.py
@@ -1,7 +1,10 @@
 from django.contrib import admin
 from django.urls import include, path
 
+from grocery import health
+
 urlpatterns = [
+    path("health/", include((health.urlpatterns, health.app_name), namespace=health.app_name)),
     path("", include("grocery.urls")),
     path("admin/", admin.site.urls),
 ]
diff --git a/grocery/health.py b/grocery/health.py
new file mode 100644
index 0000000..5fcb169
--- /dev/null
+++ b/grocery/health.py
@@ -0,0 +1,131 @@
+"""Safe process, database, and publication health endpoints."""
+
+from __future__ import annotations
+
+import logging
+from typing import Final
+
+from django.db import connection
+from django.db.migrations.executor import MigrationExecutor
+from django.http import HttpRequest, JsonResponse
+from django.urls import path
+from django.views.decorators.http import require_safe
+
+from grocery.observability import log_event
+from grocery.public_read import RECENT_RETAIL_CHANNEL, load_active_publication
+
+_LOGGER: Final = logging.getLogger("grocery.audit")
+_NO_STORE: Final = "no-store"
+
+
+def _response(payload: dict[str, str], *, status: int) -> JsonResponse:
+    response = JsonResponse(payload, status=status)
+    response.headers["Cache-Control"] = _NO_STORE
+    return response
+
+
+def _database_and_migrations_ready() -> bool:
+    with connection.cursor() as cursor:
+        cursor.execute("SELECT 1")
+        if cursor.fetchone() != (1,):
+            return False
+
+    executor = MigrationExecutor(connection)
+    executor.loader.check_consistent_history(connection)
+    targets = executor.loader.graph.leaf_nodes()
+    return not executor.migration_plan(targets)
+
+
+def _readiness_unavailable() -> JsonResponse:
+    log_event(_LOGGER, "WARNING", "health.readiness.unavailable")
+    return _response(
+        {"check": "READINESS", "status": "UNAVAILABLE"},
+        status=503,
+    )
+
+
+def _freshness_unavailable() -> JsonResponse:
+    log_event(_LOGGER, "WARNING", "health.freshness.unavailable")
+    return _response(
+        {
+            "check": "FRESHNESS",
+            "channel": RECENT_RETAIL_CHANNEL,
+            "publication_state": "UNAVAILABLE",
+            "freshness_state": "UNAVAILABLE",
+        },
+        status=503,
+    )
+
+
+@require_safe
+def live(request: HttpRequest) -> JsonResponse:
+    """Confirm only that the Django process can serve a bounded response."""
+
+    del request
+    return _response(
+        {"check": "LIVENESS", "status": "OK"},
+        status=200,
+    )
+
+
+@require_safe
+def ready(request: HttpRequest) -> JsonResponse:
+    """Confirm schema currency and one readable, sealed publication pointer."""
+
+    del request
+    try:
+        if not _database_and_migrations_ready():
+            return _readiness_unavailable()
+        active = load_active_publication()
+        if active is None or active.freshness_state not in {"current", "stale"}:
+            return _readiness_unavailable()
+    except Exception:
+        return _readiness_unavailable()
+    return _response(
+        {"check": "READINESS", "status": "READY"},
+        status=200,
+    )
+
+
+@require_safe
+def freshness(request: HttpRequest) -> JsonResponse:
+    """Report only fixed publication availability and freshness states."""
+
+    del request
+    try:
+        active = load_active_publication()
+    except Exception:
+        return _freshness_unavailable()
+    if active is None:
+        return _freshness_unavailable()
+    if active.freshness_state == "stale":
+        log_event(_LOGGER, "WARNING", "health.freshness.stale")
+        return _response(
+            {
+                "check": "FRESHNESS",
+                "channel": RECENT_RETAIL_CHANNEL,
+                "publication_state": "AVAILABLE",
+                "freshness_state": "STALE",
+            },
+            status=503,
+        )
+    if active.freshness_state != "current":
+        return _freshness_unavailable()
+    return _response(
+        {
+            "check": "FRESHNESS",
+            "channel": RECENT_RETAIL_CHANNEL,
+            "publication_state": "AVAILABLE",
+            "freshness_state": "CURRENT",
+        },
+        status=200,
+    )
+
+
+app_name = "health"
+
+urlpatterns = [
+    path("live", live, name="live"),
+    path("ready", ready, name="ready"),
+    path("freshness", freshness, name="freshness"),
+]
diff --git a/grocery/tests/test_health.py b/grocery/tests/test_health.py
new file mode 100644
index 0000000..2b649ee
--- /dev/null
+++ b/grocery/tests/test_health.py
@@ -0,0 +1,206 @@
+import json
+from types import SimpleNamespace
+from typing import cast
+from unittest.mock import MagicMock, patch
+
+from django.db import DatabaseError
+from django.http import JsonResponse
+from django.test import RequestFactory, SimpleTestCase
+
+from grocery import health
+
+
+def payload(response: JsonResponse) -> dict[str, str]:
+    return cast(dict[str, str], json.loads(response.content))
+
+
+class HealthEndpointTests(SimpleTestCase):
+    def setUp(self) -> None:
+        self.requests = RequestFactory()
+
+    def test_liveness_is_process_only_and_always_returns_bounded_200(self) -> None:
+        request = self.requests.get("/health/live")
+        with (
+            patch.object(
+                health,
+                "_database_and_migrations_ready",
+                side_effect=AssertionError("database used"),
+            ),
+            patch.object(
+                health,
+                "load_active_publication",
+                side_effect=AssertionError("publication used"),
+            ),
+        ):
+            response = health.live(request)
+
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(payload(response), {"check": "LIVENESS", "status": "OK"})
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+
+    def test_readiness_checks_database_migration_plan_and_publication(self) -> None:
+        cursor = MagicMock()
+        cursor.fetchone.return_value = (1,)
+        cursor_context = MagicMock()
+        cursor_context.__enter__.return_value = cursor
+        fake_connection = MagicMock()
+        fake_connection.cursor.return_value = cursor_context
+        executor = MagicMock()
+        executor.loader.graph.leaf_nodes.return_value = [("grocery", "0008")]
+        executor.migration_plan.return_value = []
+        active = SimpleNamespace(freshness_state="stale")
+
+        with (
+            patch.object(health, "connection", fake_connection),
+            patch.object(health, "MigrationExecutor", return_value=executor) as constructor,
+            patch.object(health, "load_active_publication", return_value=active),
+        ):
+            response = health.ready(self.requests.get("/health/ready"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(payload(response), {"check": "READINESS", "status": "READY"})
+        cursor.execute.assert_called_once_with("SELECT 1")
+        executor.loader.check_consistent_history.assert_called_once_with(fake_connection)
+        executor.migration_plan.assert_called_once_with([("grocery", "0008")])
+        constructor.assert_called_once_with(fake_connection)
+
+    def test_readiness_is_unavailable_for_migration_drift_or_missing_publication(self) -> None:
+        unavailable_cases = (
+            (False, SimpleNamespace(freshness_state="current")),
+            (True, None),
+        )
+        for database_ready, active in unavailable_cases:
+            with self.subTest(database_ready=database_ready, active=active):
+                with (
+                    patch.object(
+                        health,
+                        "_database_and_migrations_ready",
+                        return_value=database_ready,
+                    ),
+                    patch.object(health, "load_active_publication", return_value=active),
+                    patch.object(health, "log_event") as safe_log,
+                ):
+                    response = health.ready(self.requests.get("/health/ready"))
+
+                self.assertEqual(response.status_code, 503)
+                self.assertEqual(
+                    payload(response),
+                    {"check": "READINESS", "status": "UNAVAILABLE"},
+                )
+                safe_log.assert_called_once_with(
+                    health._LOGGER,
+                    "WARNING",
+                    "health.readiness.unavailable",
+                )
+
+    def test_current_stale_and_unavailable_freshness_have_fixed_safe_shapes(self) -> None:
+        cases = (
+            (
+                SimpleNamespace(freshness_state="current"),
+                200,
+                "AVAILABLE",
+                "CURRENT",
+            ),
+            (
+                SimpleNamespace(freshness_state="stale"),
+                503,
+                "AVAILABLE",
+                "STALE",
+            ),
+            (None, 503, "UNAVAILABLE", "UNAVAILABLE"),
+        )
+        for active, expected_status, publication_state, freshness_state in cases:
+            with self.subTest(freshness_state=freshness_state):
+                with (
+                    patch.object(health, "load_active_publication", return_value=active),
+                    patch.object(health, "log_event"),
+                ):
+                    response = health.freshness(self.requests.get("/health/freshness"))
+
+                self.assertEqual(response.status_code, expected_status)
+                self.assertEqual(
+                    payload(response),
+                    {
+                        "check": "FRESHNESS",
+                        "channel": "RECENT_RETAIL",
+                        "publication_state": publication_state,
+                        "freshness_state": freshness_state,
+                    },
+                )
+                self.assertEqual(response.headers["Cache-Control"], "no-store")
+
+    def test_database_or_internal_failure_is_redacted_from_response_and_log_call(self) -> None:
+        malicious = "serviceKey=secret-value raw-row actor=42 hash=" + "a" * 64
+        with (
+            patch.object(
+                health,
+                "_database_and_migrations_ready",
+                side_effect=DatabaseError(malicious),
+            ),
+            patch.object(health, "log_event") as safe_log,
+        ):
+            ready_response = health.ready(self.requests.get("/health/ready"))
+
+        with (
+            patch.object(
+                health,
+                "load_active_publication",
+                side_effect=RuntimeError(malicious),
+            ),
+            patch.object(health, "log_event") as freshness_log,
+        ):
+            freshness_response = health.freshness(self.requests.get("/health/freshness"))
+
+        for response in (ready_response, freshness_response):
+            self.assertEqual(response.status_code, 503)
+            body = response.content.decode("utf-8")
+            self.assertNotIn("secret-value", body)
+            self.assertNotIn("raw-row", body)
+            self.assertNotIn("actor", body)
+            self.assertNotIn("a" * 64, body)
+        self.assertNotIn(malicious, repr(safe_log.call_args))
+        self.assertNotIn(malicious, repr(freshness_log.call_args))
+
+    def test_freshness_does_not_call_source_or_serialize_publication_evidence(self) -> None:
+        active = SimpleNamespace(
+            freshness_state="current",
+            revision_id="private-revision-id",
+            actor_id="private-actor-id",
+            typed_fact_set_sha256="b" * 64,
+        )
+        with (
+            patch.object(health, "load_active_publication", return_value=active),
+            patch("grocery.source.client.KamisHttpClient.fetch_recent_prices") as source_fetch,
+        ):
+            response = health.freshness(self.requests.get("/health/freshness"))
+
+        source_fetch.assert_not_called()
+        body = response.content.decode("utf-8")
+        self.assertNotIn("private-revision-id", body)
+        self.assertNotIn("private-actor-id", body)
+        self.assertNotIn("b" * 64, body)
+        self.assertEqual(
+            set(payload(response)),
+            {
+                "check",
+                "channel",
+                "publication_state",
+                "freshness_state",
+            },
+        )
+
+    def test_health_views_reject_unsafe_methods_before_any_probe(self) -> None:
+        for view, route in (
+            (health.live, "/health/live"),
+            (health.ready, "/health/ready"),
+            (health.freshness, "/health/freshness"),
+        ):
+            with self.subTest(route=route):
+                with (
+                    patch.object(health, "_database_and_migrations_ready") as database_probe,
+                    patch.object(health, "load_active_publication") as publication_probe,
+                ):
+                    response = view(self.requests.post(route))
+                self.assertEqual(response.status_code, 405)
+                database_probe.assert_not_called()
+                publication_probe.assert_not_called()


## `feat(ops): alert on publication freshness`

diff --git a/grocery/management/commands/check_recent_publication_freshness.py b/grocery/management/commands/check_recent_publication_freshness.py
new file mode 100644
index 0000000..cda443e
--- /dev/null
+++ b/grocery/management/commands/check_recent_publication_freshness.py
@@ -0,0 +1,72 @@
+"""Emit one scheduler-safe state receipt for the recent-retail publication."""
+
+from __future__ import annotations
+
+import json
+from enum import StrEnum
+from typing import Final
+
+from django.core.management.base import BaseCommand, CommandError
+
+from grocery.public_read import RECENT_RETAIL_CHANNEL, load_active_publication
+
+
+class _FailureCode(StrEnum):
+    STALE = "RECENT_PUBLICATION_FRESHNESS_STALE"
+    UNAVAILABLE = "RECENT_PUBLICATION_FRESHNESS_UNAVAILABLE"
+    FAILED = "RECENT_PUBLICATION_FRESHNESS_FAILED"
+
+
+def _receipt(*, publication_state: str, freshness_state: str) -> str:
+    return json.dumps(
+        {
+            "check": "FRESHNESS",
+            "channel": RECENT_RETAIL_CHANNEL,
+            "publication_state": publication_state,
+            "freshness_state": freshness_state,
+        },
+        ensure_ascii=True,
+        separators=(",", ":"),
+    )
+
+
+_CURRENT_RECEIPT: Final = _receipt(
+    publication_state="AVAILABLE",
+    freshness_state="CURRENT",
+)
+_STALE_RECEIPT: Final = _receipt(
+    publication_state="AVAILABLE",
+    freshness_state="STALE",
+)
+_UNAVAILABLE_RECEIPT: Final = _receipt(
+    publication_state="UNAVAILABLE",
+    freshness_state="UNAVAILABLE",
+)
+
+
+class Command(BaseCommand):
+    help = "Check the database-backed recent publication freshness for scheduler alerting."
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args, options
+        try:
+            active = load_active_publication()
+            freshness_state = None if active is None else active.freshness_state
+        except Exception:
+            self.stdout.write(_UNAVAILABLE_RECEIPT)
+            raise CommandError(f"code={_FailureCode.FAILED.value}") from None
+
+        if active is None:
+            self.stdout.write(_UNAVAILABLE_RECEIPT)
+            raise CommandError(f"code={_FailureCode.UNAVAILABLE.value}") from None
+        if type(freshness_state) is not str:
+            self.stdout.write(_UNAVAILABLE_RECEIPT)
+            raise CommandError(f"code={_FailureCode.FAILED.value}") from None
+        if freshness_state == "stale":
+            self.stdout.write(_STALE_RECEIPT)
+            raise CommandError(f"code={_FailureCode.STALE.value}") from None
+        if freshness_state != "current":
+            self.stdout.write(_UNAVAILABLE_RECEIPT)
+            raise CommandError(f"code={_FailureCode.FAILED.value}") from None
+
+        self.stdout.write(_CURRENT_RECEIPT)
diff --git a/grocery/tests/test_check_recent_publication_freshness_command.py b/grocery/tests/test_check_recent_publication_freshness_command.py
new file mode 100644
index 0000000..ef61ebb
--- /dev/null
+++ b/grocery/tests/test_check_recent_publication_freshness_command.py
@@ -0,0 +1,147 @@
+import io
+import json
+from types import SimpleNamespace
+from unittest.mock import patch
+
+import pytest
+from django.core.management import call_command
+from django.core.management.base import CommandError
+
+_COMMAND = "check_recent_publication_freshness"
+_LOAD = "grocery.management.commands.check_recent_publication_freshness.load_active_publication"
+_SOURCE_FETCH = "grocery.source.client.KamisHttpClient.fetch_recent_prices"
+_EXPECTED_KEYS = {
+    "check",
+    "channel",
+    "publication_state",
+    "freshness_state",
+}
+
+
+def invoke() -> tuple[io.StringIO, object]:
+    output = io.StringIO()
+    result = call_command(_COMMAND, stdout=output)
+    return output, result
+
+
+def parsed_receipt(output: io.StringIO) -> dict[str, str]:
+    lines = output.getvalue().splitlines()
+    assert len(lines) == 1
+    value = json.loads(lines[0])
+    assert isinstance(value, dict)
+    assert set(value) == _EXPECTED_KEYS
+    return value
+
+
+def test_current_publication_emits_one_safe_receipt_and_exits_successfully() -> None:
+    active = SimpleNamespace(
+        freshness_state="current",
+        revision_id="private-revision-id",
+        actor_id="private-actor-id",
+        typed_fact_set_sha256="a" * 64,
+    )
+    with (
+        patch(_LOAD, return_value=active) as load,
+        patch(_SOURCE_FETCH) as source_fetch,
+    ):
+        output, result = invoke()
+
+    assert result is None
+    load.assert_called_once_with()
+    source_fetch.assert_not_called()
+    assert parsed_receipt(output) == {
+        "check": "FRESHNESS",
+        "channel": "RECENT_RETAIL",
+        "publication_state": "AVAILABLE",
+        "freshness_state": "CURRENT",
+    }
+    receipt = output.getvalue()
+    assert "private-revision-id" not in receipt
+    assert "private-actor-id" not in receipt
+    assert "a" * 64 not in receipt
+
+
+@pytest.mark.parametrize(
+    ("active", "expected_code", "publication_state", "freshness_state"),
+    (
+        (
+            SimpleNamespace(freshness_state="stale"),
+            "RECENT_PUBLICATION_FRESHNESS_STALE",
+            "AVAILABLE",
+            "STALE",
+        ),
+        (
+            None,
+            "RECENT_PUBLICATION_FRESHNESS_UNAVAILABLE",
+            "UNAVAILABLE",
+            "UNAVAILABLE",
+        ),
+    ),
+)
+def test_stale_or_unavailable_publication_emits_receipt_and_nonzero_fixed_code(
+    active: object,
+    expected_code: str,
+    publication_state: str,
+    freshness_state: str,
+) -> None:
+    output = io.StringIO()
+    with (
+        patch(_LOAD, return_value=active) as load,
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(_COMMAND, stdout=output)
+
+    load.assert_called_once_with()
+    assert caught.value.returncode != 0
+    assert str(caught.value) == f"code={expected_code}"
+    assert parsed_receipt(output) == {
+        "check": "FRESHNESS",
+        "channel": "RECENT_RETAIL",
+        "publication_state": publication_state,
+        "freshness_state": freshness_state,
+    }
+
+
+@pytest.mark.parametrize(
+    "failure",
+    (
+        RuntimeError("serviceKey=secret-value raw-row actor=42 hash=" + "b" * 64),
+        ValueError("https://provider.example/?serviceKey=secret-value"),
+    ),
+)
+def test_internal_failure_is_redacted_and_uses_only_fixed_failure_state(
+    failure: Exception,
+) -> None:
+    output = io.StringIO()
+    with patch(_LOAD, side_effect=failure), pytest.raises(CommandError) as caught:
+        call_command(_COMMAND, stdout=output)
+
+    assert caught.value.returncode != 0
+    assert str(caught.value) == "code=RECENT_PUBLICATION_FRESHNESS_FAILED"
+    assert parsed_receipt(output) == {
+        "check": "FRESHNESS",
+        "channel": "RECENT_RETAIL",
+        "publication_state": "UNAVAILABLE",
+        "freshness_state": "UNAVAILABLE",
+    }
+    combined = output.getvalue() + str(caught.value) + repr(caught.value)
+    assert "secret-value" not in combined
+    assert "raw-row" not in combined
+    assert "actor=42" not in combined
+    assert "b" * 64 not in combined
+    assert "provider.example" not in combined
+
+
+def test_unknown_freshness_state_is_not_reflected_and_fails_closed() -> None:
+    marker = "private-unknown-state"
+    output = io.StringIO()
+    with (
+        patch(_LOAD, return_value=SimpleNamespace(freshness_state=marker)),
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(_COMMAND, stdout=output)
+
+    assert str(caught.value) == "code=RECENT_PUBLICATION_FRESHNESS_FAILED"
+    assert marker not in output.getvalue()
+    assert marker not in str(caught.value)
+    assert parsed_receipt(output)["freshness_state"] == "UNAVAILABLE"


