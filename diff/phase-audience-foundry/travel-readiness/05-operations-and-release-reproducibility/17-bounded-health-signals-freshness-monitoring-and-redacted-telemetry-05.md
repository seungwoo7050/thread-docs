## `feat(operations): bind proxy telemetry to release`

diff --git a/operations/tests/test_local_tls_static_proxy.py b/operations/tests/test_local_tls_static_proxy.py
index 38cfafa..3f1ccff 100644
--- a/operations/tests/test_local_tls_static_proxy.py
+++ b/operations/tests/test_local_tls_static_proxy.py
@@ -6,6 +6,7 @@ from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
 import json
 import os
 from pathlib import Path
+import re
 import select
 import socket
 import ssl
@@ -460,7 +461,19 @@ class LocalTlsStaticProxyTests(unittest.TestCase):
         self.assertEqual(record["outcome"], "ready")
         self.assertEqual(
             set(record),
-            {"event", "method", "outcome", "release_sha", "route", "status"},
+            {
+                "event",
+                "timestamp",
+                "method",
+                "outcome",
+                "release_sha",
+                "route",
+                "status",
+            },
+        )
+        self.assertRegex(
+            record["timestamp"],
+            r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
         )
         self.processes.append((process, initial_log))
         return process, front_port, public_host
@@ -710,7 +723,19 @@ class LocalTlsStaticProxyTests(unittest.TestCase):
             record = json.loads(line)
             self.assertEqual(
                 set(record),
-                {"event", "method", "outcome", "release_sha", "route", "status"},
+                {
+                    "event",
+                    "timestamp",
+                    "method",
+                    "outcome",
+                    "release_sha",
+                    "route",
+                    "status",
+                },
+            )
+            self.assertRegex(
+                record["timestamp"],
+                r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
             )
 
     def test_backend_certificate_authentication_failure_is_a_fixed_502(self):
diff --git a/scripts/local-tls-static-proxy b/scripts/local-tls-static-proxy
index 034de5d..da32429 100755
--- a/scripts/local-tls-static-proxy
+++ b/scripts/local-tls-static-proxy
@@ -11,6 +11,7 @@ from __future__ import annotations
 
 import argparse
 from dataclasses import dataclass
+from datetime import UTC, datetime
 import hashlib
 import http.client
 from http import HTTPStatus
@@ -126,6 +127,9 @@ def _fixed_log(
 ) -> None:
     record = {
         "event": "local_rehearsal_proxy",
+        "timestamp": datetime.now(UTC).isoformat(timespec="milliseconds").replace(
+            "+00:00", "Z"
+        ),
         "method": method,
         "outcome": outcome,
         "release_sha": release_sha,


## `test(ops): align three-module freshness contract`

diff --git a/operations/tests/test_observability.py b/operations/tests/test_observability.py
index 1bbfd81..0fcbbb0 100644
--- a/operations/tests/test_observability.py
+++ b/operations/tests/test_observability.py
@@ -142,6 +142,10 @@ class TelemetryTimestampTests(SimpleTestCase):
 class FreshnessIsolationTests(SimpleTestCase):
     def test_one_loader_failure_preserves_the_other_module_state(self):
         with (
+            patch(
+                "operations.views.flight_publication_state",
+                return_value="ready",
+            ),
             patch(
                 "public_web.results._load_entry_card",
                 side_effect=RuntimeError("private-entry-detail"),
@@ -155,7 +159,11 @@ class FreshnessIsolationTests(SimpleTestCase):
 
         self.assertEqual(
             states,
-            {"entry": "server-error", "warning": "ready"},
+            {
+                "flight": "ready",
+                "entry": "server-error",
+                "warning": "ready",
+            },
         )
 
 
@@ -165,12 +173,16 @@ class FreshnessCommandBoundaryTests(SimpleTestCase):
         with patch(
             "operations.management.commands.check_freshness."
             "publication_freshness_snapshot",
-            return_value={"entry": "ready", "warning": "ready"},
+            return_value={
+                "flight": "ready",
+                "entry": "ready",
+                "warning": "ready",
+            },
         ):
             call_command("check_freshness", stdout=output)
         self.assertEqual(
             output.getvalue(),
-            "freshness=OK entry=ready warning=ready\n",
+            "freshness=OK flight=ready entry=ready warning=ready\n",
         )
 
     def test_alert_output_is_fixed_and_nonzero(self):
@@ -178,12 +190,16 @@ class FreshnessCommandBoundaryTests(SimpleTestCase):
         with patch(
             "operations.management.commands.check_freshness."
             "publication_freshness_snapshot",
-            return_value={"entry": "stale", "warning": "unavailable"},
+            return_value={
+                "flight": "ready",
+                "entry": "stale",
+                "warning": "unavailable",
+            },
         ):
             with self.assertRaises(CommandError) as raised:
                 call_command("check_freshness")
         self.assertEqual(
             str(raised.exception),
-            "freshness=ALERT entry=stale warning=unavailable",
+            "freshness=ALERT flight=ready entry=stale warning=unavailable",
         )
         self.assertNotIn(marker, str(raised.exception))


## `fix(ops): measure transient search contract`

diff --git a/operations/tests/test_performance_smoke.py b/operations/tests/test_performance_smoke.py
index 9af889a..d7e4234 100644
--- a/operations/tests/test_performance_smoke.py
+++ b/operations/tests/test_performance_smoke.py
@@ -1,4 +1,5 @@
 from contextlib import redirect_stderr, redirect_stdout
+from datetime import datetime
 from importlib.machinery import SourceFileLoader
 import importlib.util
 from io import StringIO
@@ -7,6 +8,8 @@ import ssl
 import sys
 import tempfile
 from unittest.mock import patch
+from urllib.parse import parse_qs
+from zoneinfo import ZoneInfo
 
 from django.test import SimpleTestCase
 
@@ -35,10 +38,42 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         self.assertEqual(first, second)
         self.assertEqual(len(first), 200)
         self.assertEqual(first.count("root_get"), 40)
-        self.assertEqual(first.count("results_get"), 120)
+        self.assertEqual(first.count("results_redirect"), 120)
         self.assertEqual(first.count("valid_post"), 30)
         self.assertEqual(first.count("invalid_post"), 10)
 
+    def test_post_body_uses_only_the_current_transient_search_contract(self):
+        current = datetime(2026, 8, 31, 9, 30, tzinfo=ZoneInfo("Asia/Seoul"))
+
+        valid = parse_qs(
+            runner._post_body(
+                "valid_post",
+                "token",
+                current=current,
+            ).decode("ascii")
+        )
+        invalid = parse_qs(
+            runner._post_body(
+                "invalid_post",
+                "token",
+                current=current,
+            ).decode("ascii")
+        )
+
+        self.assertEqual(
+            valid,
+            {
+                "departure_at": ["2026-09-01T09:30"],
+                "return_by": ["2026-09-04T09:30"],
+                "csrfmiddlewaretoken": ["token"],
+            },
+        )
+        self.assertEqual(invalid["departure_at"], ["2026-09-01T09:30"])
+        self.assertEqual(invalid["return_by"], ["2026-09-01T08:30"])
+        self.assertNotIn("destination", valid)
+        self.assertNotIn("departure_date", valid)
+        self.assertNotIn("return_date", valid)
+
     def test_target_is_loopback_https_with_an_explicit_port(self):
         accepted = (
             "https://127.0.0.1:8017",
@@ -186,7 +221,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         tls_context = object()
         correct_body = (
             b'<section data-error-summary>\n'
-            + "귀국일은 출국일과 같거나 그 이후여야 합니다.".encode()
+            + "인천 도착 마감 시각은 출발 가능한 시각보다 늦어야 합니다.".encode()
         )
         cases = (
             (200, correct_body, True),
@@ -215,7 +250,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         with patch.object(
             runner,
             "_request",
-            return_value=(303, {"location": "/results/"}, b"", 1.0),
+            return_value=(200, {}, b'<section id="recommendations">', 1.0),
         ) as request:
             accepted, _ = runner._perform(
                 target,
@@ -260,6 +295,12 @@ class PerformanceSmokeContractTests(SimpleTestCase):
 
     def test_listener_and_wsgi_pid_have_separate_semantics(self):
         target = runner.LocalTarget("127.0.0.1", 8017)
+
+        def warmup_request(*args, **kwargs):
+            if kwargs["path"] == "/results/":
+                return 303, {"location": "/"}, b"", 1.0
+            return 200, {}, b"", 1.0
+
         with (
             patch.object(
                 runner,
@@ -274,7 +315,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
             patch.object(
                 runner,
                 "_request",
-                return_value=(200, {}, b"", 1.0),
+                side_effect=warmup_request,
             ),
             patch.object(
                 runner,
@@ -389,8 +430,8 @@ class PerformanceSmokeContractTests(SimpleTestCase):
             "9876",
             "8443",
             "cookie-secret",
-            "JP",
-            "2030-01-10",
+            "departure_at",
+            "return_by",
         )
         with tempfile.TemporaryDirectory() as directory:
             ca_file = Path(directory).resolve() / "ca.pem"
@@ -466,7 +507,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         self.assertEqual(
             stdout.getvalue(),
             "performance=OK requests=200 concurrency=8 root_get=40 "
-            "results_get=120 valid_post=30 invalid_post=10 unexpected=0 "
+            "results_redirect=120 valid_post=30 invalid_post=10 unexpected=0 "
             "p95_ms=12.50 max_rss_mib=42.25\n",
         )
         self.assertEqual(rehearsal.call_args.args[2:], (4321, 9876, 8443))
diff --git a/scripts/performance-smoke b/scripts/performance-smoke
index 8c79777..5e235b0 100755
--- a/scripts/performance-smoke
+++ b/scripts/performance-smoke
@@ -7,6 +7,7 @@ import argparse
 from collections import Counter
 from concurrent.futures import ThreadPoolExecutor
 from dataclasses import dataclass
+from datetime import datetime, timedelta
 import http.client
 import math
 from pathlib import Path
@@ -18,6 +19,7 @@ import subprocess
 import threading
 import time
 from urllib.parse import urlencode, urlsplit
+from zoneinfo import ZoneInfo
 
 
 REQUEST_COUNT = 200
@@ -33,10 +35,11 @@ _CSRF_RE = re.compile(
 _COOKIE_RE = re.compile(r"(?:^|;\s*)csrftoken=([A-Za-z0-9]+)")
 _MIX = (
     ("root_get", 40),
-    ("results_get", 120),
+    ("results_redirect", 120),
     ("valid_post", 30),
     ("invalid_post", 10),
 )
+_SEOUL = ZoneInfo("Asia/Seoul")
 
 
 class RehearsalFailure(Exception):
@@ -190,13 +193,25 @@ def _csrf_material(
     return token_match.group(1).decode("ascii"), cookie_match.group(1)
 
 
-def _post_body(kind: str, csrf_token: str) -> bytes:
+def _post_body(
+    kind: str,
+    csrf_token: str,
+    *,
+    current: datetime | None = None,
+) -> bytes:
+    now = current or datetime.now(_SEOUL)
+    if now.tzinfo is None:
+        raise RehearsalFailure
+    now = now.astimezone(_SEOUL).replace(second=0, microsecond=0)
+    departure_at = now + timedelta(days=1)
+    return_by = departure_at + (
+        timedelta(days=3)
+        if kind == "valid_post"
+        else -timedelta(hours=1)
+    )
     values = {
-        "destination": "JP",
-        "departure_date": "2030-01-10",
-        "return_date": (
-            "2030-01-12" if kind == "valid_post" else "2030-01-09"
-        ),
+        "departure_at": departure_at.strftime("%Y-%m-%dT%H:%M"),
+        "return_by": return_by.strftime("%Y-%m-%dT%H:%M"),
         "csrfmiddlewaretoken": csrf_token,
     }
     return urlencode(values).encode("ascii")
@@ -217,14 +232,18 @@ def _perform(
             path="/",
         )
         return status == 200, elapsed
-    if kind == "results_get":
-        status, _, _, elapsed = _request(
+    if kind == "results_redirect":
+        status, headers, response_body, elapsed = _request(
             target,
             tls_context,
             method="GET",
             path="/results/",
         )
-        return status == 200, elapsed
+        return (
+            status == 303
+            and headers.get("location") == "/"
+            and not response_body
+        ), elapsed
     body = _post_body(kind, csrf_token)
     status, headers, response_body, elapsed = _request(
         target,
@@ -240,14 +259,14 @@ def _perform(
     )
     if kind == "valid_post":
         return (
-            status == 303
-            and headers.get("location") == "/results/"
-            and not response_body
+            status == 200
+            and "location" not in headers
+            and b'id="recommendations"' in response_body
         ), elapsed
     return (
         status == 200
         and b"data-error-summary" in response_body
-        and "귀국일은 출국일과 같거나 그 이후여야 합니다.".encode()
+        and "인천 도착 마감 시각은 출발 가능한 시각보다 늦어야 합니다.".encode()
         in response_body
     ), elapsed
 
@@ -331,13 +350,20 @@ def run_rehearsal(
         raise RehearsalFailure
     csrf_token, csrf_cookie = _csrf_material(target, tls_context)
     for warmup_path in ("/", "/results/") * 8:
-        status, _, _, _ = _request(
+        status, headers, response_body, _ = _request(
             target,
             tls_context,
             method="GET",
             path=warmup_path,
         )
-        if status != 200:
+        expected = (
+            status == 200
+            if warmup_path == "/"
+            else status == 303
+            and headers.get("location") == "/"
+            and not response_body
+        )
+        if not expected:
             raise RehearsalFailure
 
     samples = [_rss_kib(wsgi_pid)]
@@ -441,7 +467,7 @@ def main(argv: list[str] | None = None) -> int:
     print(
         f"performance={result} requests={REQUEST_COUNT} "
         f"concurrency={CONCURRENCY} root_get={counts['root_get']} "
-        f"results_get={counts['results_get']} "
+        f"results_redirect={counts['results_redirect']} "
         f"valid_post={counts['valid_post']} "
         f"invalid_post={counts['invalid_post']} "
         f"unexpected={unexpected} p95_ms={p95_ms:.2f} "
