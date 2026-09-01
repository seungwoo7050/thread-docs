## `test(operations): measure verified TLS traffic`

diff --git a/operations/tests/test_performance_smoke.py b/operations/tests/test_performance_smoke.py
index 0fb63a8..9af889a 100644
--- a/operations/tests/test_performance_smoke.py
+++ b/operations/tests/test_performance_smoke.py
@@ -1,7 +1,11 @@
+from contextlib import redirect_stderr, redirect_stdout
 from importlib.machinery import SourceFileLoader
 import importlib.util
+from io import StringIO
 from pathlib import Path
+import ssl
 import sys
+import tempfile
 from unittest.mock import patch
 
 from django.test import SimpleTestCase
@@ -35,18 +39,20 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         self.assertEqual(first.count("valid_post"), 30)
         self.assertEqual(first.count("invalid_post"), 10)
 
-    def test_target_is_loopback_http_with_an_explicit_port(self):
+    def test_target_is_loopback_https_with_an_explicit_port(self):
         accepted = (
-            "http://127.0.0.1:8017",
-            "http://localhost:8017/",
+            "https://127.0.0.1:8017",
+            "https://127.0.0.1:8017/",
         )
         rejected = (
-            "https://127.0.0.1:8017",
-            "http://example.invalid:8017",
-            "http://127.0.0.1",
-            "http://user:secret@127.0.0.1:8017",
-            "http://127.0.0.1:8017/results/",
-            "http://127.0.0.1:8017/?destination=JP",
+            "http://127.0.0.1:8017",
+            "https://localhost:8017",
+            "https://example.invalid:8017",
+            "https://127.0.0.1",
+            "https://127.0.0.1:0",
+            "https://user:secret@127.0.0.1:8017",
+            "https://127.0.0.1:8017/results/",
+            "https://127.0.0.1:8017/?destination=JP",
         )
 
         for value in accepted:
@@ -60,6 +66,116 @@ class PerformanceSmokeContractTests(SimpleTestCase):
                 with self.assertRaises(runner.RehearsalFailure):
                     runner.validate_target(value)
 
+    def test_ca_file_must_be_absolute_regular_and_not_a_symlink(self):
+        with tempfile.TemporaryDirectory() as directory:
+            root = Path(directory).resolve()
+            ca_file = root / "ca.pem"
+            ca_file.write_text("test-ca", encoding="ascii")
+            ca_symlink = root / "ca-link.pem"
+            ca_symlink.symlink_to(ca_file)
+            real_parent = root / "real-parent"
+            real_parent.mkdir()
+            nested_ca = real_parent / "nested-ca.pem"
+            nested_ca.write_text("test-ca", encoding="ascii")
+            parent_symlink = root / "parent-link"
+            parent_symlink.symlink_to(real_parent, target_is_directory=True)
+
+            self.assertEqual(
+                runner.validate_ca_file(str(ca_file)),
+                ca_file,
+            )
+            rejected = (
+                "relative-ca.pem",
+                str(root / "missing.pem"),
+                str(root),
+                str(ca_symlink),
+                str(parent_symlink / "nested-ca.pem"),
+            )
+            for value in rejected:
+                with self.subTest(value=Path(value).name):
+                    with self.assertRaises(runner.RehearsalFailure):
+                        runner.validate_ca_file(value)
+
+    def test_tls_context_uses_the_ca_with_hostname_verification(self):
+        verified_context = type(
+            "VerifiedContext",
+            (),
+            {
+                "check_hostname": True,
+                "verify_mode": ssl.CERT_REQUIRED,
+            },
+        )()
+        ca_file = Path("/private/tmp/local-ca.pem")
+
+        with patch.object(
+            runner.ssl,
+            "create_default_context",
+            return_value=verified_context,
+        ) as create_context:
+            result = runner._tls_context(ca_file)
+
+        self.assertIs(result, verified_context)
+        create_context.assert_called_once_with(cafile=str(ca_file))
+
+    def test_tls_context_rejects_disabled_hostname_verification(self):
+        unverified_context = type(
+            "UnverifiedContext",
+            (),
+            {
+                "check_hostname": False,
+                "verify_mode": ssl.CERT_REQUIRED,
+            },
+        )()
+        with patch.object(
+            runner.ssl,
+            "create_default_context",
+            return_value=unverified_context,
+        ):
+            with self.assertRaises(runner.RehearsalFailure):
+                runner._tls_context(Path("/private/tmp/local-ca.pem"))
+
+    def test_request_builds_a_verified_https_connection(self):
+        target = runner.LocalTarget("127.0.0.1", 8017)
+        tls_context = object()
+        response = type(
+            "Response",
+            (),
+            {
+                "status": 200,
+                "getheaders": lambda self: [("Content-Type", "text/plain")],
+                "read": lambda self, amount: b"bounded",
+            },
+        )()
+        connection = type(
+            "Connection",
+            (),
+            {
+                "request": lambda self, *args, **kwargs: None,
+                "getresponse": lambda self: response,
+                "close": lambda self: None,
+            },
+        )()
+
+        with patch.object(
+            runner.http.client,
+            "HTTPSConnection",
+            return_value=connection,
+        ) as https_connection:
+            status, _, body, _ = runner._request(
+                target,
+                tls_context,
+                method="GET",
+                path="/",
+            )
+
+        self.assertEqual((status, body), (200, b"bounded"))
+        https_connection.assert_called_once_with(
+            "127.0.0.1",
+            8017,
+            timeout=5,
+            context=tls_context,
+        )
+
     def test_p95_uses_the_nearest_rank(self):
         values = [float(value) for value in range(1, 201)]
 
@@ -67,6 +183,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
 
     def test_invalid_post_requires_the_validation_state(self):
         target = runner.LocalTarget("127.0.0.1", 8017)
+        tls_context = object()
         correct_body = (
             b'<section data-error-summary>\n'
             + "귀국일은 출국일과 같거나 그 이후여야 합니다.".encode()
@@ -86,13 +203,35 @@ class PerformanceSmokeContractTests(SimpleTestCase):
                 ):
                     accepted, _ = runner._perform(
                         target,
+                        tls_context,
                         "invalid_post",
                         "token",
                         "cookie",
                     )
                 self.assertEqual(accepted, expected)
 
-    def test_listener_pid_must_be_the_measured_process(self):
+    def test_post_sends_only_the_same_origin_needed_for_secure_csrf(self):
+        target = runner.LocalTarget("127.0.0.1", 8017)
+        with patch.object(
+            runner,
+            "_request",
+            return_value=(303, {"location": "/results/"}, b"", 1.0),
+        ) as request:
+            accepted, _ = runner._perform(
+                target,
+                object(),
+                "valid_post",
+                "token",
+                "cookie",
+            )
+
+        self.assertTrue(accepted)
+        self.assertEqual(
+            request.call_args.kwargs["headers"]["Origin"],
+            "https://127.0.0.1:8017",
+        )
+
+    def test_listener_pid_must_own_the_front_https_listener(self):
         target = runner.LocalTarget("127.0.0.1", 8017)
         completed = type(
             "Completed",
@@ -109,7 +248,7 @@ class PerformanceSmokeContractTests(SimpleTestCase):
                 "-a",
                 "-p",
                 "1234",
-                "-iTCP:8017",
+                "-iTCP@127.0.0.1:8017",
                 "-sTCP:LISTEN",
                 "-t",
             ],
@@ -118,3 +257,219 @@ class PerformanceSmokeContractTests(SimpleTestCase):
         completed.stdout = "9999\n"
         with patch.object(runner.subprocess, "run", return_value=completed):
             self.assertFalse(runner._pid_owns_listener(1234, target))
+
+    def test_listener_and_wsgi_pid_have_separate_semantics(self):
+        target = runner.LocalTarget("127.0.0.1", 8017)
+        with (
+            patch.object(
+                runner,
+                "_pid_owns_listener",
+                return_value=True,
+            ) as owns_listener,
+            patch.object(
+                runner,
+                "_csrf_material",
+                return_value=("token", "cookie"),
+            ),
+            patch.object(
+                runner,
+                "_request",
+                return_value=(200, {}, b"", 1.0),
+            ),
+            patch.object(
+                runner,
+                "_perform",
+                return_value=(True, 1.0),
+            ),
+            patch.object(runner, "_rss_kib", return_value=1024) as rss,
+        ):
+            result = runner.run_rehearsal(
+                target,
+                object(),
+                listener_pid=1234,
+                wsgi_pid=5678,
+                backend_port=8443,
+            )
+
+        self.assertEqual(result, (1.0, 1.0, 0))
+        self.assertEqual(
+            {
+                (call.args[0], call.args[1].port)
+                for call in owns_listener.call_args_list
+            },
+            {(1234, 8017), (5678, 8443)},
+        )
+        self.assertTrue(rss.call_args_list)
+        self.assertTrue(
+            all(call.args[0] == 5678 for call in rss.call_args_list)
+        )
+
+    def test_front_and_wsgi_must_own_their_exact_distinct_listeners(self):
+        target = runner.LocalTarget("127.0.0.1", 8017)
+        cases = (
+            ({(1234, 8017)}, 8443),
+            ({(5678, 8443)}, 8443),
+            ({(1234, 8017), (5678, 8017)}, 8017),
+        )
+        for accepted, backend_port in cases:
+            with self.subTest(accepted=accepted, backend_port=backend_port):
+                def owns(pid, listener):
+                    return (pid, listener.port) in accepted
+
+                with patch.object(runner, "_pid_owns_listener", side_effect=owns):
+                    with self.assertRaises(runner.RehearsalFailure):
+                        runner.run_rehearsal(
+                            target,
+                            object(),
+                            listener_pid=1234,
+                            wsgi_pid=5678,
+                            backend_port=backend_port,
+                        )
+
+    def test_response_body_and_headers_are_bounded(self):
+        target = runner.LocalTarget("127.0.0.1", 8017)
+        cases = (
+            ([("X-Test", "value")] * 101, b"ok", None),
+            ([("X-Test", "value")], b"oversized", 3),
+        )
+        for headers, body, body_limit in cases:
+            with self.subTest(body_limit=body_limit):
+                response = type(
+                    "Response",
+                    (),
+                    {
+                        "status": 200,
+                        "getheaders": lambda self, value=headers: value,
+                        "read": lambda self, amount, value=body: value,
+                    },
+                )()
+                connection = type(
+                    "Connection",
+                    (),
+                    {
+                        "request": lambda self, *args, **kwargs: None,
+                        "getresponse": lambda self: response,
+                        "close": lambda self: None,
+                    },
+                )()
+                patches = [
+                    patch.object(
+                        runner.http.client,
+                        "HTTPSConnection",
+                        return_value=connection,
+                    )
+                ]
+                if body_limit is not None:
+                    patches.append(
+                        patch.object(runner, "MAX_RESPONSE_BYTES", body_limit)
+                    )
+                with patches[0]:
+                    if len(patches) == 2:
+                        with patches[1]:
+                            with self.assertRaises(runner.RehearsalFailure):
+                                runner._request(
+                                    target,
+                                    object(),
+                                    method="GET",
+                                    path="/",
+                                )
+                    else:
+                        with self.assertRaises(runner.RehearsalFailure):
+                            runner._request(
+                                target,
+                                object(),
+                                method="GET",
+                                path="/",
+                            )
+
+    def test_main_emits_only_the_fixed_redacted_failure_receipt(self):
+        sensitive_values = (
+            "https://127.0.0.1:8017",
+            "4321",
+            "9876",
+            "8443",
+            "cookie-secret",
+            "JP",
+            "2030-01-10",
+        )
+        with tempfile.TemporaryDirectory() as directory:
+            ca_file = Path(directory).resolve() / "ca.pem"
+            ca_file.write_text("test-ca", encoding="ascii")
+            stdout = StringIO()
+            stderr = StringIO()
+            with (
+                patch.object(runner, "_tls_context", return_value=object()),
+                patch.object(
+                    runner,
+                    "run_rehearsal",
+                    side_effect=RuntimeError(" ".join(sensitive_values)),
+                ),
+                redirect_stdout(stdout),
+                redirect_stderr(stderr),
+            ):
+                status = runner.main(
+                    [
+                        "--base-url",
+                        sensitive_values[0],
+                        "--ca-file",
+                        str(ca_file),
+                        "--listener-pid",
+                        sensitive_values[1],
+                        "--wsgi-pid",
+                        sensitive_values[2],
+                        "--backend-port",
+                        sensitive_values[3],
+                    ]
+                )
+
+        self.assertEqual(status, 1)
+        self.assertEqual(
+            stdout.getvalue(),
+            "performance=FAIL reason=bounded-rehearsal-failure\n",
+        )
+        self.assertEqual(stderr.getvalue(), "")
+        combined = stdout.getvalue() + stderr.getvalue()
+        for value in sensitive_values:
+            with self.subTest(value=value):
+                self.assertNotIn(value, combined)
+
+    def test_main_keeps_the_success_receipt_fixed_and_redacted(self):
+        with tempfile.TemporaryDirectory() as directory:
+            ca_file = Path(directory).resolve() / "ca.pem"
+            ca_file.write_text("test-ca", encoding="ascii")
+            stdout = StringIO()
+            with (
+                patch.object(runner, "_tls_context", return_value=object()),
+                patch.object(
+                    runner,
+                    "run_rehearsal",
+                    return_value=(12.5, 42.25, 0),
+                ) as rehearsal,
+                redirect_stdout(stdout),
+            ):
+                status = runner.main(
+                    [
+                        "--base-url",
+                        "https://127.0.0.1:8017",
+                        "--ca-file",
+                        str(ca_file),
+                        "--listener-pid",
+                        "4321",
+                        "--wsgi-pid",
+                        "9876",
+                        "--backend-port",
+                        "8443",
+                    ]
+                )
+
+        self.assertEqual(status, 0)
+        self.assertEqual(
+            stdout.getvalue(),
+            "performance=OK requests=200 concurrency=8 root_get=40 "
+            "results_get=120 valid_post=30 invalid_post=10 unexpected=0 "
+            "p95_ms=12.50 max_rss_mib=42.25\n",
+        )
+        self.assertEqual(rehearsal.call_args.args[2:], (4321, 9876, 8443))
+        for value in ("127.0.0.1", str(ca_file), "4321", "9876", "8443"):
+            with self.subTest(value=value):
+                self.assertNotIn(value, stdout.getvalue())
diff --git a/scripts/performance-smoke b/scripts/performance-smoke
index b117288..8c79777 100755
--- a/scripts/performance-smoke
+++ b/scripts/performance-smoke
@@ -9,8 +9,11 @@ from concurrent.futures import ThreadPoolExecutor
 from dataclasses import dataclass
 import http.client
 import math
+from pathlib import Path
 import random
 import re
+import ssl
+import stat
 import subprocess
 import threading
 import time
@@ -22,6 +25,8 @@ CONCURRENCY = 8
 MAX_P95_MS = 500.0
 MAX_RSS_MIB = 256.0
 MAX_RESPONSE_BYTES = 1_048_576
+MAX_RESPONSE_HEADERS = 100
+MAX_RESPONSE_HEADER_BYTES = 65_536
 _CSRF_RE = re.compile(
     rb'name="csrfmiddlewaretoken" value="([A-Za-z0-9]+)"'
 )
@@ -43,17 +48,27 @@ class LocalTarget:
     host: str
     port: int
 
+    @property
+    def origin(self) -> str:
+        return f"https://{self.host}:{self.port}"
+
+
+class QuietArgumentParser(argparse.ArgumentParser):
+    def error(self, message: str) -> None:
+        raise RehearsalFailure from None
+
 
 def validate_target(value: str) -> LocalTarget:
     try:
         parsed = urlsplit(value)
         port = parsed.port
-    except ValueError as exc:
+    except ValueError:
         raise RehearsalFailure from None
     if (
-        parsed.scheme != "http"
-        or parsed.hostname not in {"127.0.0.1", "localhost"}
+        parsed.scheme != "https"
+        or parsed.hostname != "127.0.0.1"
         or port is None
+        or not 1 <= port <= 65_535
         or parsed.username is not None
         or parsed.password is not None
         or parsed.path not in {"", "/"}
@@ -64,6 +79,34 @@ def validate_target(value: str) -> LocalTarget:
     return LocalTarget(parsed.hostname, port)
 
 
+def validate_ca_file(value: str) -> Path:
+    path = Path(value)
+    if not path.is_absolute():
+        raise RehearsalFailure
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except (OSError, RuntimeError, ValueError):
+        raise RehearsalFailure from None
+    if (
+        resolved != path
+        or stat.S_ISLNK(status.st_mode)
+        or not stat.S_ISREG(status.st_mode)
+    ):
+        raise RehearsalFailure
+    return path
+
+
+def _tls_context(ca_file: Path) -> ssl.SSLContext:
+    try:
+        context = ssl.create_default_context(cafile=str(ca_file))
+    except (OSError, ssl.SSLError, ValueError):
+        raise RehearsalFailure from None
+    if not context.check_hostname or context.verify_mode != ssl.CERT_REQUIRED:
+        raise RehearsalFailure
+    return context
+
+
 def fixed_mix() -> list[str]:
     tasks = [name for name, count in _MIX for _ in range(count)]
     random.Random(20260830).shuffle(tasks)
@@ -79,6 +122,7 @@ def percentile_95(values: list[float]) -> float:
 
 def _request(
     target: LocalTarget,
+    tls_context: ssl.SSLContext,
     *,
     method: str,
     path: str,
@@ -86,7 +130,12 @@ def _request(
     headers: dict[str, str] | None = None,
 ) -> tuple[int, dict[str, str], bytes, float]:
     started = time.perf_counter_ns()
-    connection = http.client.HTTPConnection(target.host, target.port, timeout=5)
+    connection = http.client.HTTPSConnection(
+        target.host,
+        target.port,
+        timeout=5,
+        context=tls_context,
+    )
     try:
         connection.request(
             method,
@@ -99,11 +148,21 @@ def _request(
             },
         )
         response = connection.getresponse()
+        raw_headers = response.getheaders()
+        if (
+            len(raw_headers) > MAX_RESPONSE_HEADERS
+            or sum(
+                len(name) + len(value) + 4
+                for name, value in raw_headers
+            )
+            > MAX_RESPONSE_HEADER_BYTES
+        ):
+            raise RehearsalFailure
         response_body = response.read(MAX_RESPONSE_BYTES + 1)
         if len(response_body) > MAX_RESPONSE_BYTES:
             raise RehearsalFailure
         response_headers = {
-            name.lower(): value for name, value in response.getheaders()
+            name.lower(): value for name, value in raw_headers
         }
         status = response.status
     except (OSError, http.client.HTTPException):
@@ -114,8 +173,16 @@ def _request(
     return status, response_headers, response_body, elapsed_ms
 
 
-def _csrf_material(target: LocalTarget) -> tuple[str, str]:
-    status, headers, body, _ = _request(target, method="GET", path="/")
+def _csrf_material(
+    target: LocalTarget,
+    tls_context: ssl.SSLContext,
+) -> tuple[str, str]:
+    status, headers, body, _ = _request(
+        target,
+        tls_context,
+        method="GET",
+        path="/",
+    )
     token_match = _CSRF_RE.search(body)
     cookie_match = _COOKIE_RE.search(headers.get("set-cookie", ""))
     if status != 200 or token_match is None or cookie_match is None:
@@ -137,16 +204,23 @@ def _post_body(kind: str, csrf_token: str) -> bytes:
 
 def _perform(
     target: LocalTarget,
+    tls_context: ssl.SSLContext,
     kind: str,
     csrf_token: str,
     csrf_cookie: str,
 ) -> tuple[bool, float]:
     if kind == "root_get":
-        status, _, _, elapsed = _request(target, method="GET", path="/")
+        status, _, _, elapsed = _request(
+            target,
+            tls_context,
+            method="GET",
+            path="/",
+        )
         return status == 200, elapsed
     if kind == "results_get":
         status, _, _, elapsed = _request(
             target,
+            tls_context,
             method="GET",
             path="/results/",
         )
@@ -154,12 +228,14 @@ def _perform(
     body = _post_body(kind, csrf_token)
     status, headers, response_body, elapsed = _request(
         target,
+        tls_context,
         method="POST",
         path="/",
         body=body,
         headers={
             "Content-Type": "application/x-www-form-urlencoded",
             "Cookie": f"csrftoken={csrf_cookie}",
+            "Origin": target.origin,
         },
     )
     if kind == "valid_post":
@@ -185,7 +261,7 @@ def _pid_owns_listener(root_pid: int, target: LocalTarget) -> bool:
                 "-a",
                 "-p",
                 str(root_pid),
-                f"-iTCP:{target.port}",
+                f"-iTCP@{target.host}:{target.port}",
                 "-sTCP:LISTEN",
                 "-t",
             ],
@@ -238,27 +314,40 @@ def _rss_kib(root_pid: int) -> int:
     return sum(rows[pid][1] for pid in included)
 
 
-def run_rehearsal(target: LocalTarget, root_pid: int) -> tuple[float, float, int]:
-    if not _pid_owns_listener(root_pid, target):
+def run_rehearsal(
+    target: LocalTarget,
+    tls_context: ssl.SSLContext,
+    listener_pid: int,
+    wsgi_pid: int,
+    backend_port: int,
+) -> tuple[float, float, int]:
+    backend_target = LocalTarget("127.0.0.1", backend_port)
+    if (
+        listener_pid == wsgi_pid
+        or target.port == backend_port
+        or not _pid_owns_listener(listener_pid, target)
+        or not _pid_owns_listener(wsgi_pid, backend_target)
+    ):
         raise RehearsalFailure
-    csrf_token, csrf_cookie = _csrf_material(target)
+    csrf_token, csrf_cookie = _csrf_material(target, tls_context)
     for warmup_path in ("/", "/results/") * 8:
         status, _, _, _ = _request(
             target,
+            tls_context,
             method="GET",
             path=warmup_path,
         )
         if status != 200:
             raise RehearsalFailure
 
-    samples: list[int] = []
+    samples = [_rss_kib(wsgi_pid)]
     stop_sampling = threading.Event()
     sampling_failed = threading.Event()
 
     def sample_rss() -> None:
         while not stop_sampling.is_set():
             try:
-                samples.append(_rss_kib(root_pid))
+                samples.append(_rss_kib(wsgi_pid))
             except RehearsalFailure:
                 sampling_failed.set()
                 stop_sampling.set()
@@ -273,6 +362,7 @@ def run_rehearsal(target: LocalTarget, root_pid: int) -> tuple[float, float, int
                 executor.map(
                     lambda kind: _perform(
                         target,
+                        tls_context,
                         kind,
                         csrf_token,
                         csrf_cookie,
@@ -287,7 +377,8 @@ def run_rehearsal(target: LocalTarget, root_pid: int) -> tuple[float, float, int
         not samples
         or sampling_failed.is_set()
         or sampler.is_alive()
-        or not _pid_owns_listener(root_pid, target)
+        or not _pid_owns_listener(listener_pid, target)
+        or not _pid_owns_listener(wsgi_pid, backend_target)
     ):
         raise RehearsalFailure
     unexpected = sum(not accepted for accepted, _ in outcomes)
@@ -295,18 +386,47 @@ def run_rehearsal(target: LocalTarget, root_pid: int) -> tuple[float, float, int
     return percentile_95(latencies), max(samples) / 1024, unexpected
 
 
-def main() -> int:
-    parser = argparse.ArgumentParser(add_help=True)
+def _parse_pid(value: str) -> int:
+    if re.fullmatch(r"[1-9][0-9]*", value) is None:
+        raise RehearsalFailure
+    pid = int(value)
+    if pid <= 1:
+        raise RehearsalFailure
+    return pid
+
+
+def _parse_port(value: str) -> int:
+    if re.fullmatch(r"[1-9][0-9]*", value) is None:
+        raise RehearsalFailure
+    port = int(value)
+    if not 1 <= port <= 65_535:
+        raise RehearsalFailure
+    return port
+
+
+def main(argv: list[str] | None = None) -> int:
+    parser = QuietArgumentParser(add_help=True, allow_abbrev=False)
     parser.add_argument("--base-url", required=True)
-    parser.add_argument("--pid", required=True, type=int)
-    arguments = parser.parse_args()
+    parser.add_argument("--ca-file", required=True)
+    parser.add_argument("--listener-pid", required=True)
+    parser.add_argument("--wsgi-pid", required=True)
+    parser.add_argument("--backend-port", required=True)
     try:
-        if arguments.pid <= 1:
+        arguments = parser.parse_args(argv)
+        listener_pid = _parse_pid(arguments.listener_pid)
+        wsgi_pid = _parse_pid(arguments.wsgi_pid)
+        backend_port = _parse_port(arguments.backend_port)
+        if listener_pid == wsgi_pid:
             raise RehearsalFailure
         target = validate_target(arguments.base_url)
+        ca_file = validate_ca_file(arguments.ca_file)
+        tls_context = _tls_context(ca_file)
         p95_ms, max_rss_mib, unexpected = run_rehearsal(
             target,
-            arguments.pid,
+            tls_context,
+            listener_pid,
+            wsgi_pid,
+            backend_port,
         )
     except Exception:
         print("performance=FAIL reason=bounded-rehearsal-failure")


