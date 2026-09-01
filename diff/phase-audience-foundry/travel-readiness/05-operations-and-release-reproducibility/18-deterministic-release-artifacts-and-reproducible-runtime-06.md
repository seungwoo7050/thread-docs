## `feat(operations): add local TLS static proxy`

diff --git a/operations/tests/test_local_tls_static_proxy.py b/operations/tests/test_local_tls_static_proxy.py
new file mode 100644
index 0000000..38cfafa
--- /dev/null
+++ b/operations/tests/test_local_tls_static_proxy.py
@@ -0,0 +1,940 @@
+from __future__ import annotations
+
+import hashlib
+import http.client
+from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
+import json
+import os
+from pathlib import Path
+import select
+import socket
+import ssl
+import subprocess
+import sys
+import tempfile
+import threading
+import time
+import unittest
+
+
+REPOSITORY_ROOT = Path(__file__).resolve().parents[2]
+PROXY_SCRIPT = REPOSITORY_ROOT / "scripts" / "local-tls-static-proxy"
+RELEASE_SHA = "1" * 40
+SENSITIVE_MARKER = "private-query-body-ip-header-marker"
+
+
+def canonical_json(value: object) -> bytes:
+    return (
+        json.dumps(
+            value,
+            ensure_ascii=False,
+            allow_nan=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        )
+        + "\n"
+    ).encode("utf-8")
+
+
+def unused_port() -> int:
+    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as probe:
+        probe.bind(("127.0.0.1", 0))
+        return probe.getsockname()[1]
+
+
+class LoopbackHTTPSConnection(http.client.HTTPSConnection):
+    def connect(self) -> None:
+        raw = socket.create_connection(
+            ("127.0.0.1", self.port), timeout=self.timeout
+        )
+        self.sock = self._context.wrap_socket(raw, server_hostname=self.host)
+
+
+class LocalTlsStaticProxyTests(unittest.TestCase):
+    def setUp(self) -> None:
+        self.temporary = tempfile.TemporaryDirectory(
+            prefix="travel-readiness-proxy-test-"
+        )
+        self.root = Path(self.temporary.name).resolve()
+        self.static_root = self.root / "static"
+        (self.static_root / "public_web").mkdir(parents=True)
+        self.css = b"body { color: #132f2b; }\n"
+        self.javascript = b"document.documentElement.dataset.ready = 'true';\n"
+        (self.static_root / "public_web" / "site.css").write_bytes(self.css)
+        (self.static_root / "public_web" / "site.js").write_bytes(self.javascript)
+        self.cert = self.root / "candidate-cert.pem"
+        self.key = self.root / "candidate-key.pem"
+        self.wrong_cert = self.root / "wrong-cert.pem"
+        self.wrong_key = self.root / "wrong-key.pem"
+        self.dns_only_cert = self.root / "dns-only-cert.pem"
+        self.dns_only_key = self.root / "dns-only-key.pem"
+        self._make_certificate(self.cert, self.key, common_name="candidate.invalid")
+        self._make_certificate(self.wrong_cert, self.wrong_key, common_name="wrong.invalid")
+        self._make_certificate(
+            self.dns_only_cert,
+            self.dns_only_key,
+            common_name="candidate.invalid",
+            include_ip_san=False,
+        )
+        self.manifest = self.root / "release-manifest.json"
+        self._write_manifest(self.manifest)
+        self.processes: list[tuple[subprocess.Popen[str], str]] = []
+        self.backend_servers: list[tuple[ThreadingHTTPServer, threading.Thread]] = []
+        self.backend_requests: list[dict[str, object]] = []
+        self.backend_stall_started = threading.Event()
+        self.backend_stall_release = threading.Event()
+
+    def tearDown(self) -> None:
+        for process, initial_log in reversed(self.processes):
+            self._stop_proxy(process, initial_log)
+        for server, thread in reversed(self.backend_servers):
+            server.shutdown()
+            server.server_close()
+            thread.join(timeout=3)
+        self.temporary.cleanup()
+
+    def _make_certificate(
+        self,
+        cert: Path,
+        key: Path,
+        *,
+        common_name: str,
+        include_ip_san: bool = True,
+    ) -> None:
+        configuration = cert.with_suffix(".cnf")
+        configuration.write_text(
+            "[req]\n"
+            "distinguished_name=dn\n"
+            "x509_extensions=v3\n"
+            "prompt=no\n"
+            "[dn]\n"
+            f"CN={common_name}\n"
+            "[v3]\n"
+            f"subjectAltName=DNS:candidate.invalid{',IP:127.0.0.1' if include_ip_san else ''}\n"
+            "basicConstraints=critical,CA:TRUE\n"
+            "keyUsage=critical,digitalSignature,keyEncipherment,keyCertSign\n"
+            "extendedKeyUsage=serverAuth\n",
+            encoding="ascii",
+        )
+        generated = subprocess.run(
+            [
+                "openssl",
+                "req",
+                "-x509",
+                "-newkey",
+                "rsa:2048",
+                "-nodes",
+                "-days",
+                "1",
+                "-config",
+                str(configuration),
+                "-keyout",
+                str(key),
+                "-out",
+                str(cert),
+            ],
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=15,
+        )
+        self.assertEqual(generated.returncode, 0)
+        key.chmod(0o600)
+
+    def _manifest_value(self) -> dict[str, object]:
+        collected = [
+            {
+                "path": "public_web/site.css",
+                "sha256": hashlib.sha256(self.css).hexdigest(),
+            },
+            {
+                "path": "public_web/site.js",
+                "sha256": hashlib.sha256(self.javascript).hexdigest(),
+            },
+        ]
+        packages = [
+            {"name": "django", "version": "5.2.17"},
+            {"name": "gunicorn", "version": "26.2.0"},
+            {"name": "psycopg", "version": "3.3.4"},
+            {"name": "psycopg-binary", "version": "3.3.4"},
+        ]
+        runtime_versions = {
+            "django": "5.2.17",
+            "gunicorn": "26.2.0",
+            "postgresql": "18.6",
+            "psycopg": "3.3.4",
+            "psycopg_distribution": "binary-wheel",
+            "python": "3.14.7",
+            "uv": "0.12.6",
+        }
+        migration_digest = hashlib.sha256(b"synthetic migration\n").hexdigest()
+        runtime_digest = hashlib.sha256(b"synthetic runtime\n").hexdigest()
+        lock_digest = hashlib.sha256(b"synthetic lock\n").hexdigest()
+        proxy_digest = hashlib.sha256(PROXY_SCRIPT.read_bytes()).hexdigest()
+        source_tracked = [
+            {
+                "git_mode": "100644",
+                "path": "countries/migrations/0001_initial.py",
+                "sha256": migration_digest,
+            },
+            {
+                "git_mode": "100644",
+                "path": "public_web/static/public_web/site.css",
+                "sha256": hashlib.sha256(self.css).hexdigest(),
+            },
+            {
+                "git_mode": "100644",
+                "path": "public_web/static/public_web/site.js",
+                "sha256": hashlib.sha256(self.javascript).hexdigest(),
+            },
+            {
+                "git_mode": "100644",
+                "path": "runtime/versions.toml",
+                "sha256": runtime_digest,
+            },
+            {
+                "git_mode": "100755",
+                "path": "scripts/local-tls-static-proxy",
+                "sha256": proxy_digest,
+            },
+            {"git_mode": "100644", "path": "uv.lock", "sha256": lock_digest},
+        ]
+        leaves = [
+            {
+                "app": "countries",
+                "module": "countries.migrations.0001_initial",
+                "name": "0001_initial",
+                "origin": "source",
+                "path": "countries/migrations/0001_initial.py",
+                "sha256": migration_digest,
+            }
+        ]
+        origins = [
+            {
+                "collected_path": "public_web/site.css",
+                "origin": "source",
+                "path": "public_web/static/public_web/site.css",
+                "sha256": hashlib.sha256(self.css).hexdigest(),
+            },
+            {
+                "collected_path": "public_web/site.js",
+                "origin": "source",
+                "path": "public_web/static/public_web/site.js",
+                "sha256": hashlib.sha256(self.javascript).hexdigest(),
+            },
+        ]
+        tracked_static = [
+            {
+                "collected_path": item["collected_path"],
+                "path": item["path"],
+                "sha256": item["sha256"],
+            }
+            for item in origins
+        ]
+        return {
+            "archive": {
+                "compression": "none",
+                "format": "ustar",
+                "gid": 0,
+                "group": "root",
+                "mtime": 0,
+                "regular_modes": ["0644", "0755"],
+                "uid": 0,
+                "user": "root",
+            },
+            "dependencies": {
+                "lock_path": "uv.lock",
+                "lock_sha256": lock_digest,
+                "package_set_sha256": hashlib.sha256(
+                    canonical_json(packages)
+                ).hexdigest(),
+                "packages": packages,
+            },
+            "git_object_format": "sha1",
+            "migrations": {
+                "leaf_set_sha256": hashlib.sha256(canonical_json(leaves)).hexdigest(),
+                "leaves": leaves,
+            },
+            "release_sha": RELEASE_SHA,
+            "runtime": {
+                "path": "runtime/versions.toml",
+                "sha256": runtime_digest,
+                "versions": runtime_versions,
+            },
+            "schema_version": 1,
+            "source": {
+                "tracked": source_tracked,
+                "tracked_set_sha256": hashlib.sha256(
+                    canonical_json(source_tracked)
+                ).hexdigest(),
+            },
+            "static": {
+                "collected": collected,
+                "collected_set_sha256": hashlib.sha256(
+                    canonical_json(collected)
+                ).hexdigest(),
+                "origins": origins,
+                "tracked": tracked_static,
+                "tracked_set_sha256": hashlib.sha256(
+                    canonical_json(tracked_static)
+                ).hexdigest(),
+            },
+        }
+
+    def _write_manifest(
+        self, path: Path, value: dict[str, object] | None = None
+    ) -> None:
+        path.write_bytes(canonical_json(value or self._manifest_value()))
+
+    def _start_backend(
+        self, *, cert: Path | None = None, key: Path | None = None
+    ) -> int:
+        requests = self.backend_requests
+        stall_started = self.backend_stall_started
+        stall_release = self.backend_stall_release
+
+        class BackendHandler(BaseHTTPRequestHandler):
+            protocol_version = "HTTP/1.1"
+
+            def log_message(self, format: str, *args: object) -> None:
+                return
+
+            def _serve(self, *, include_body: bool = True) -> None:
+                length = int(self.headers.get("Content-Length", "0"))
+                body = self.rfile.read(length) if length else b""
+                requests.append(
+                    {
+                        "method": self.command,
+                        "path": self.path,
+                        "headers": {
+                            name.lower(): value for name, value in self.headers.items()
+                        },
+                        "body": body,
+                    }
+                )
+                if self.path == "/unsafe-location":
+                    self.send_response(302)
+                    self.send_header("Location", "/\\outside.invalid")
+                    self.send_header("Content-Length", "0")
+                    self.end_headers()
+                    return
+                if self.path == "/oversized-response":
+                    response = b"x" * (2 * 1_024 * 1_024 + 1)
+                    self.send_response(200)
+                    self.send_header("Content-Length", str(len(response)))
+                    self.end_headers()
+                    try:
+                        self.wfile.write(response)
+                    except OSError:
+                        pass
+                    return
+                if self.path == "/oversized-header":
+                    self.send_response(200)
+                    self.send_header("X-Oversized", "x" * 17_000)
+                    self.send_header("Content-Length", "0")
+                    self.end_headers()
+                    return
+                if self.path == "/connection-response":
+                    response = b"connection response\n"
+                    self.send_response(200)
+                    self.send_header("Connection", "Set-Cookie")
+                    self.send_header("Set-Cookie", f"private={SENSITIVE_MARKER}")
+                    self.send_header("Content-Length", str(len(response)))
+                    self.end_headers()
+                    self.wfile.write(response)
+                    return
+                if self.path == "/stall":
+                    stall_started.set()
+                    stall_release.wait(timeout=3)
+                response = b"backend response\n"
+                self.send_response(200)
+                self.send_header("Content-Type", "text/plain; charset=utf-8")
+                self.send_header("Cache-Control", "no-store")
+                self.send_header("Set-Cookie", "csrftoken=synthetic; Secure; SameSite=Strict")
+                self.send_header("Content-Length", str(len(response)))
+                self.end_headers()
+                if include_body:
+                    self.wfile.write(response)
+
+            def do_GET(self) -> None:
+                self._serve()
+
+            def do_HEAD(self) -> None:
+                self._serve(include_body=False)
+
+            def do_POST(self) -> None:
+                self._serve()
+
+        class QuietBackendServer(ThreadingHTTPServer):
+            def handle_error(self, request, client_address) -> None:
+                return
+
+        server = QuietBackendServer(("127.0.0.1", 0), BackendHandler)
+        context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+        context.minimum_version = ssl.TLSVersion.TLSv1_2
+        context.load_cert_chain(cert or self.cert, key or self.key)
+        server.socket = context.wrap_socket(server.socket, server_side=True)
+        thread = threading.Thread(target=server.serve_forever, daemon=True)
+        thread.start()
+        self.backend_servers.append((server, thread))
+        return server.server_address[1]
+
+    @staticmethod
+    def _proxy_environment() -> dict[str, str]:
+        return {
+            "LANG": "C",
+            "LC_ALL": "C",
+            "PATH": os.environ.get("PATH", "/usr/bin:/bin"),
+            "PYTHONDONTWRITEBYTECODE": "1",
+        }
+
+    def _proxy_arguments(
+        self,
+        *,
+        front_port: int,
+        backend_port: int,
+        backend_ca: Path | None = None,
+        cert: Path | None = None,
+        key: Path | None = None,
+        static_root: Path | None = None,
+        manifest: Path | None = None,
+    ) -> list[str]:
+        return [
+            sys.executable,
+            str(PROXY_SCRIPT),
+            "--listen-port",
+            str(front_port),
+            "--public-host",
+            f"candidate.invalid:{front_port}",
+            "--backend-port",
+            str(backend_port),
+            "--backend-ca-file",
+            str(backend_ca or self.cert),
+            "--tls-cert-file",
+            str(cert or self.cert),
+            "--tls-key-file",
+            str(key or self.key),
+            "--static-root",
+            str(static_root or self.static_root),
+            "--release-manifest",
+            str(manifest or self.manifest),
+        ]
+
+    def _start_proxy(
+        self,
+        *,
+        backend_port: int | None = None,
+        backend_ca: Path | None = None,
+        manifest: Path | None = None,
+    ) -> tuple[subprocess.Popen[str], int, str]:
+        front_port = unused_port()
+        if backend_port is None:
+            backend_port = self._start_backend()
+        public_host = f"candidate.invalid:{front_port}"
+        process = subprocess.Popen(
+            self._proxy_arguments(
+                front_port=front_port,
+                backend_port=backend_port,
+                backend_ca=backend_ca,
+                manifest=manifest,
+            ),
+            cwd="/private/tmp",
+            env=self._proxy_environment(),
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            text=True,
+        )
+        self.assertIsNotNone(process.stderr)
+        ready, _, _ = select.select([process.stderr], [], [], 5)
+        if not ready:
+            process.terminate()
+            process.wait(timeout=3)
+            self.fail("local proxy did not emit its fixed readiness record")
+        initial_log = process.stderr.readline()
+        try:
+            record = json.loads(initial_log)
+        except json.JSONDecodeError:
+            process.terminate()
+            process.wait(timeout=3)
+            self.fail("local proxy readiness record was not JSON")
+        self.assertEqual(record["outcome"], "ready")
+        self.assertEqual(
+            set(record),
+            {"event", "method", "outcome", "release_sha", "route", "status"},
+        )
+        self.processes.append((process, initial_log))
+        return process, front_port, public_host
+
+    def _stop_proxy(self, process: subprocess.Popen[str], initial_log: str) -> str:
+        if process.poll() is None:
+            process.terminate()
+        try:
+            stdout, stderr = process.communicate(timeout=5)
+        except subprocess.TimeoutExpired:
+            process.kill()
+            stdout, stderr = process.communicate(timeout=3)
+        self.assertEqual(stdout, "")
+        for index, (candidate, _) in enumerate(self.processes):
+            if candidate is process:
+                self.processes.pop(index)
+                break
+        return initial_log + stderr
+
+    def _request(
+        self,
+        port: int,
+        method: str,
+        target: str,
+        *,
+        body: bytes | None = None,
+        headers: dict[str, str] | None = None,
+        ca: Path | None = None,
+    ) -> tuple[int, list[tuple[str, str]], bytes]:
+        context = ssl.create_default_context(cafile=str(ca or self.cert))
+        context.minimum_version = ssl.TLSVersion.TLSv1_2
+        connection = LoopbackHTTPSConnection(
+            "candidate.invalid", port, context=context, timeout=5
+        )
+        request_headers = {"Host": f"candidate.invalid:{port}"}
+        request_headers.update(headers or {})
+        try:
+            connection.request(method, target, body=body, headers=request_headers)
+            response = connection.getresponse()
+            response_body = response.read()
+            return response.status, response.getheaders(), response_body
+        finally:
+            connection.close()
+
+    @staticmethod
+    def _header(headers: list[tuple[str, str]], name: str) -> str | None:
+        values = [value for candidate, value in headers if candidate.lower() == name.lower()]
+        if not values:
+            return None
+        return values[-1]
+
+    def test_verified_front_tls_serves_manifest_bound_static_get_head_and_etag(self):
+        process, port, _ = self._start_proxy()
+
+        status, headers, body = self._request(port, "GET", "/static/public_web/site.css")
+        self.assertEqual(status, 200)
+        self.assertEqual(body, self.css)
+        digest = hashlib.sha256(self.css).hexdigest()
+        etag = f'"sha256-{digest}"'
+        self.assertEqual(self._header(headers, "ETag"), etag)
+        self.assertEqual(self._header(headers, "Cache-Control"), "no-cache")
+        self.assertEqual(self._header(headers, "X-Content-Type-Options"), "nosniff")
+        self.assertEqual(
+            self._header(headers, "Strict-Transport-Security"),
+            "max-age=31536000; includeSubDomains; preload",
+        )
+
+        status, headers, body = self._request(port, "HEAD", "/static/public_web/site.css")
+        self.assertEqual(status, 200)
+        self.assertEqual(body, b"")
+        self.assertEqual(self._header(headers, "Content-Length"), str(len(self.css)))
+
+        status, headers, body = self._request(
+            port,
+            "GET",
+            "/static/public_web/site.css",
+            headers={"If-None-Match": etag},
+        )
+        self.assertEqual(status, 304)
+        self.assertEqual(body, b"")
+        self.assertEqual(self._header(headers, "ETag"), etag)
+
+        wrong_context = ssl.create_default_context(cafile=str(self.wrong_cert))
+        connection = LoopbackHTTPSConnection(
+            "candidate.invalid", port, context=wrong_context, timeout=5
+        )
+        with self.assertRaises(ssl.SSLCertVerificationError):
+            connection.request(
+                "GET",
+                "/static/public_web/site.css",
+                headers={"Host": f"candidate.invalid:{port}"},
+            )
+        connection.close()
+        logs = self._stop_proxy(process, self.processes[-1][1])
+        self.assertNotIn(str(self.root), logs)
+
+    def test_static_rejects_traversal_symlink_and_runtime_tamper(self):
+        _, port, _ = self._start_proxy()
+
+        status, _, body = self._request(port, "GET", "/static/../private.txt")
+        self.assertEqual(status, 400)
+        self.assertEqual(body, b"bad request\n")
+        status, _, body = self._request(port, "GET", "/static/%2e%2e/private.txt")
+        self.assertEqual(status, 400)
+        self.assertEqual(body, b"bad request\n")
+        status, _, body = self._request(port, "GET", "/static/public_web/./site.css")
+        self.assertEqual(status, 400)
+        self.assertEqual(body, b"bad request\n")
+
+        (self.static_root / "public_web" / "site.css").write_bytes(b"tampered\n")
+        status, _, body = self._request(port, "GET", "/static/public_web/site.css")
+        self.assertEqual(status, 503)
+        self.assertEqual(body, b"service unavailable\n")
+
+        outside = self.root / "outside.txt"
+        outside.write_text(SENSITIVE_MARKER, encoding="utf-8")
+        javascript = self.static_root / "public_web" / "site.js"
+        javascript.unlink()
+        javascript.symlink_to(outside)
+        status, _, body = self._request(port, "GET", "/static/public_web/site.js")
+        self.assertEqual(status, 503)
+        self.assertEqual(body, b"service unavailable\n")
+        self.assertNotIn(SENSITIVE_MARKER.encode(), body)
+
+    def test_host_method_transfer_encoding_and_body_bounds_fail_closed(self):
+        _, port, _ = self._start_proxy()
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            "/",
+            headers={"Host": f"wrong.invalid:{port}"},
+        )
+        self.assertEqual((status, body), (421, b"misdirected request\n"))
+
+        status, _, body = self._request(port, "PUT", "/")
+        self.assertEqual((status, body), (405, b"method not allowed\n"))
+
+        status, _, body = self._request(
+            port,
+            "POST",
+            "/",
+            body=b"x" * (16 * 1_024 + 1),
+            headers={"Content-Type": "application/x-www-form-urlencoded"},
+        )
+        self.assertEqual((status, body), (413, b"request too large\n"))
+
+        status, _, body = self._request(
+            port,
+            "POST",
+            "/",
+            body=b"x=1",
+            headers={
+                "Content-Type": "application/x-www-form-urlencoded",
+                "Transfer-Encoding": "chunked",
+            },
+        )
+        self.assertEqual((status, body), (400, b"bad request\n"))
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            "/",
+            headers={"X-Oversized": "x" * 9_000},
+        )
+        self.assertEqual((status, body), (431, b"request headers too large\n"))
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            "/",
+            headers={"X-Invalid": "prefix\x85suffix"},
+        )
+        self.assertEqual((status, body), (400, b"bad request\n"))
+
+    def test_dynamic_proxy_uses_verified_backend_tls_and_drops_forwarding_headers(self):
+        backend_port = self._start_backend()
+        process, port, public_host = self._start_proxy(backend_port=backend_port)
+        marker_target = f"/inspect?value={SENSITIVE_MARKER}"
+        status, headers, body = self._request(
+            port,
+            "GET",
+            marker_target,
+            headers={
+                "Accept": "text/html",
+                "Forwarded": f"for={SENSITIVE_MARKER}",
+                "X-Forwarded-For": SENSITIVE_MARKER,
+                "X-Forwarded-Host": SENSITIVE_MARKER,
+                "X-Forwarded-Proto": SENSITIVE_MARKER,
+            },
+        )
+        self.assertEqual((status, body), (200, b"backend response\n"))
+        self.assertEqual(self._header(headers, "Cache-Control"), "no-store")
+        self.assertIn("csrftoken=synthetic", self._header(headers, "Set-Cookie") or "")
+        observed = self.backend_requests[-1]
+        observed_headers = observed["headers"]
+        self.assertEqual(observed["path"], marker_target)
+        self.assertEqual(observed_headers["host"], public_host)
+        self.assertEqual(observed_headers["accept"], "text/html")
+        self.assertNotIn("forwarded", observed_headers)
+        self.assertFalse(
+            any(name.startswith("x-forwarded-") for name in observed_headers)
+        )
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            "/connection-nominated",
+            headers={
+                "Connection": "keep-alive, Cookie",
+                "Cookie": f"private={SENSITIVE_MARKER}",
+            },
+        )
+        self.assertEqual((status, body), (200, b"backend response\n"))
+        nominated_headers = self.backend_requests[-1]["headers"]
+        self.assertNotIn("connection", nominated_headers)
+        self.assertNotIn("cookie", nominated_headers)
+
+        form = f"destination={SENSITIVE_MARKER}".encode()
+        status, _, body = self._request(
+            port,
+            "POST",
+            "/",
+            body=form,
+            headers={
+                "Content-Type": "application/x-www-form-urlencoded",
+                "Origin": f"https://{public_host}",
+                "Referer": f"https://{public_host}/",
+            },
+        )
+        self.assertEqual((status, body), (200, b"backend response\n"))
+        self.assertEqual(self.backend_requests[-1]["body"], form)
+
+        status, _, body = self._request(port, "GET", "/unsafe-location")
+        self.assertEqual((status, body), (502, b"bad gateway\n"))
+        status, _, body = self._request(port, "GET", "/oversized-response")
+        self.assertEqual((status, body), (502, b"bad gateway\n"))
+        status, _, body = self._request(port, "GET", "/oversized-header")
+        self.assertEqual((status, body), (502, b"bad gateway\n"))
+        status, headers, body = self._request(port, "GET", "/connection-response")
+        self.assertEqual((status, body), (200, b"connection response\n"))
+        self.assertIsNone(self._header(headers, "Set-Cookie"))
+
+        logs = self._stop_proxy(process, self.processes[-1][1])
+        self.assertNotIn(SENSITIVE_MARKER, logs)
+        for line in logs.splitlines():
+            record = json.loads(line)
+            self.assertEqual(
+                set(record),
+                {"event", "method", "outcome", "release_sha", "route", "status"},
+            )
+
+    def test_backend_certificate_authentication_failure_is_a_fixed_502(self):
+        backend_port = self._start_backend()
+        process, port, _ = self._start_proxy(
+            backend_port=backend_port,
+            backend_ca=self.wrong_cert,
+        )
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            f"/failure?marker={SENSITIVE_MARKER}",
+            headers={"X-Forwarded-For": SENSITIVE_MARKER},
+        )
+        self.assertEqual((status, body), (502, b"bad gateway\n"))
+        logs = self._stop_proxy(process, self.processes[-1][1])
+        self.assertIn('"outcome":"backend_tls_auth_failed"', logs)
+        self.assertNotIn(SENSITIVE_MARKER, logs)
+        self.assertNotIn(str(self.root), logs)
+
+        dns_backend_port = self._start_backend(
+            cert=self.dns_only_cert,
+            key=self.dns_only_key,
+        )
+        process, port, _ = self._start_proxy(
+            backend_port=dns_backend_port,
+            backend_ca=self.dns_only_cert,
+        )
+        status, _, body = self._request(port, "GET", "/hostname-mismatch")
+        self.assertEqual((status, body), (502, b"bad gateway\n"))
+        logs = self._stop_proxy(process, self.processes[-1][1])
+        self.assertIn('"outcome":"backend_tls_auth_failed"', logs)
+
+    def test_backend_outage_is_a_fixed_503_without_sensitive_material(self):
+        process, port, _ = self._start_proxy(backend_port=unused_port())
+
+        status, _, body = self._request(
+            port,
+            "GET",
+            f"/outage?marker={SENSITIVE_MARKER}",
+            headers={"Forwarded": SENSITIVE_MARKER},
+        )
+        self.assertEqual((status, body), (503, b"service unavailable\n"))
+        self.assertNotIn(SENSITIVE_MARKER.encode(), body)
+        logs = self._stop_proxy(process, self.processes[-1][1])
+        self.assertIn('"outcome":"backend_unavailable"', logs)
+        self.assertNotIn(SENSITIVE_MARKER, logs)
+        self.assertNotIn(str(self.root), logs)
+
+    def test_threaded_front_keeps_static_and_dynamic_available_during_a_stall(self):
+        backend_port = self._start_backend()
+        _, port, _ = self._start_proxy(backend_port=backend_port)
+        stalled_result: dict[str, tuple[int, list[tuple[str, str]], bytes]] = {}
+        stalled_errors: list[BaseException] = []
+
+        def request_stalled_route() -> None:
+            try:
+                stalled_result["response"] = self._request(port, "GET", "/stall")
+            except BaseException as error:
+                stalled_errors.append(error)
+
+        stalled_thread = threading.Thread(target=request_stalled_route, daemon=True)
+        stalled_thread.start()
+        self.assertTrue(self.backend_stall_started.wait(timeout=2))
+        started = time.monotonic()
+        try:
+            static_status, _, static_body = self._request(
+                port, "GET", "/static/public_web/site.css"
+            )
+            dynamic_status, _, dynamic_body = self._request(port, "GET", "/fast")
+        finally:
+            self.backend_stall_release.set()
+        elapsed = time.monotonic() - started
+        stalled_thread.join(timeout=3)
+
+        self.assertLess(elapsed, 2)
+        self.assertEqual((static_status, static_body), (200, self.css))
+        self.assertEqual((dynamic_status, dynamic_body), (200, b"backend response\n"))
+        self.assertEqual(stalled_errors, [])
+        self.assertEqual(stalled_result["response"][0], 200)
+
+    def test_startup_rejects_unsafe_permissions_symlinks_and_relative_paths(self):
+        front_port = unused_port()
+        backend_port = unused_port()
+
+        def assert_rejected(**overrides: Path) -> None:
+            result = subprocess.run(
+                self._proxy_arguments(
+                    front_port=front_port,
+                    backend_port=backend_port,
+                    **overrides,
+                ),
+                cwd="/private/tmp",
+                env=self._proxy_environment(),
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=5,
+            )
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(json.loads(result.stderr)["outcome"], "configuration_rejected")
+            self.assertNotIn(str(self.root), result.stderr)
+
+        original_key_mode = self.key.stat().st_mode & 0o777
+        self.key.chmod(0o640)
+        try:
+            assert_rejected()
+        finally:
+            self.key.chmod(original_key_mode)
+
+        original_cert_mode = self.cert.stat().st_mode & 0o777
+        self.cert.chmod(0o664)
+        try:
+            assert_rejected()
+        finally:
+            self.cert.chmod(original_cert_mode)
+
+        original_ca_mode = self.wrong_cert.stat().st_mode & 0o777
+        self.wrong_cert.chmod(0o664)
+        try:
+            assert_rejected(backend_ca=self.wrong_cert)
+        finally:
+            self.wrong_cert.chmod(original_ca_mode)
+
+        original_manifest_mode = self.manifest.stat().st_mode & 0o777
+        self.manifest.chmod(0o664)
+        try:
+            assert_rejected()
+        finally:
+            self.manifest.chmod(original_manifest_mode)
+
+        original_root_mode = self.static_root.stat().st_mode & 0o777
+        self.static_root.chmod(0o775)
+        try:
+            assert_rejected()
+        finally:
+            self.static_root.chmod(original_root_mode)
+
+        key_link = self.root / "key-link.pem"
+        key_link.symlink_to(self.key)
+        assert_rejected(key=key_link)
+        manifest_link = self.root / "manifest-link.json"
+        manifest_link.symlink_to(self.manifest)
+        assert_rejected(manifest=manifest_link)
+        assert_rejected(manifest=Path("relative-release-manifest.json"))
+
+    def test_manifest_identity_digest_and_exact_static_tree_fail_closed(self):
+        cases: list[tuple[str, dict[str, object] | None, bool]] = []
+        invalid_sha = self._manifest_value()
+        invalid_sha["release_sha"] = SENSITIVE_MARKER
+        cases.append(("invalid-sha", invalid_sha, False))
+        invalid_digest = self._manifest_value()
+        invalid_digest["static"]["collected_set_sha256"] = "0" * 64
+        cases.append(("invalid-digest", invalid_digest, False))
+        reduced_schema = {
+            "git_object_format": "sha1",
+            "release_sha": RELEASE_SHA,
+            "schema_version": 1,
+            "static": self._manifest_value()["static"],
+        }
+        cases.append(("reduced-schema", reduced_schema, False))
+        script_mismatch = self._manifest_value()
+        tracked = script_mismatch["source"]["tracked"]
+        for entry in tracked:
+            if entry["path"] == "scripts/local-tls-static-proxy":
+                entry["sha256"] = "0" * 64
+        script_mismatch["source"]["tracked_set_sha256"] = hashlib.sha256(
+            canonical_json(tracked)
+        ).hexdigest()
+        cases.append(("script-mismatch", script_mismatch, False))
+        cases.append(("extra-static", None, True))
+
+        for name, value, add_extra in cases:
+            with self.subTest(name=name):
+                manifest = self.root / f"{name}-{SENSITIVE_MARKER}.json"
+                self._write_manifest(manifest, value)
+                extra = self.static_root / "unexpected.txt"
+                if add_extra:
+                    extra.write_text(SENSITIVE_MARKER, encoding="utf-8")
+                front_port = unused_port()
+                backend_port = unused_port()
+                result = subprocess.run(
+                    self._proxy_arguments(
+                        front_port=front_port,
+                        backend_port=backend_port,
+                        manifest=manifest,
+                    ),
+                    cwd="/private/tmp",
+                    env=self._proxy_environment(),
+                    capture_output=True,
+                    text=True,
+                    check=False,
+                    timeout=5,
+                )
+                if add_extra:
+                    extra.unlink()
+                self.assertEqual(result.returncode, 64)
+                self.assertEqual(result.stdout, "")
+                record = json.loads(result.stderr)
+                self.assertEqual(record["outcome"], "configuration_rejected")
+                self.assertEqual(record["release_sha"], "unavailable")
+                self.assertNotIn(SENSITIVE_MARKER, result.stderr)
+                self.assertNotIn(str(self.root), result.stderr)
+
+    def test_proxy_script_has_local_only_boundary_and_no_tls_bypass(self):
+        script = PROXY_SCRIPT.read_text(encoding="utf-8")
+        self.assertTrue(PROXY_SCRIPT.stat().st_mode & 0o111)
+        self.assertIn("not a production proxy selection", script)
+        self.assertIn('(\"127.0.0.1\", configuration.listen_port)', script)
+        self.assertIn('http.client.HTTPSConnection(\n            "127.0.0.1"', script)
+        self.assertIn("context.check_hostname = True", script)
+        self.assertIn("context.verify_mode = ssl.CERT_REQUIRED", script)
+        self.assertIn("TLSVersion.TLSv1_2", script)
+        for forbidden in (
+            "CERT_NONE",
+            "check_hostname = False",
+            "0.0.0.0",
+            "X-Forwarded-Proto\", \"https",
+            "traceback",
+        ):
+            self.assertNotIn(forbidden, script)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/local-tls-static-proxy b/scripts/local-tls-static-proxy
new file mode 100755
index 0000000..034de5d
--- /dev/null
+++ b/scripts/local-tls-static-proxy
@@ -0,0 +1,1162 @@
+#!/usr/bin/env python3
+"""Local-only TLS/static reverse proxy for a release rehearsal.
+
+This deliberately small stdlib process is not a production proxy selection.  It
+exists so the immutable release files and the loopback-only HTTPS application
+can be exercised through one browser origin before a deployment platform is
+chosen by a person.
+"""
+
+from __future__ import annotations
+
+import argparse
+from dataclasses import dataclass
+import hashlib
+import http.client
+from http import HTTPStatus
+from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
+import json
+import mimetypes
+import os
+from pathlib import Path, PurePosixPath
+import re
+import signal
+import socket
+import ssl
+import stat
+import sys
+import threading
+from urllib.parse import urlsplit
+
+
+SCHEMA_VERSION = 1
+MAX_REQUEST_TARGET = 2_048
+MAX_HEADER_BYTES = 16_384
+MAX_HEADER_LINE = 8_192
+MAX_HEADER_COUNT = 50
+MAX_REQUEST_BODY = 16 * 1_024
+MAX_RESPONSE_BODY = 2 * 1_024 * 1_024
+MAX_STATIC_FILE = 4 * 1_024 * 1_024
+MAX_STATIC_TOTAL = 16 * 1_024 * 1_024
+SOCKET_TIMEOUT_SECONDS = 5
+HSTS_VALUE = "max-age=31536000; includeSubDomains; preload"
+HEX_DIGEST = re.compile(r"[0-9a-f]{64}", re.ASCII)
+STATIC_PATH = re.compile(r"[A-Za-z0-9._~/-]+", re.ASCII)
+DNS_NAME = re.compile(
+    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?"
+    r"(?:\.[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?)*",
+    re.ASCII,
+)
+HEADER_NAME = re.compile(r"[!#$%&'*+\-.^_`|~0-9A-Za-z]+", re.ASCII)
+FORWARDED_REQUEST_HEADERS = {
+    "accept": "Accept",
+    "accept-language": "Accept-Language",
+    "content-type": "Content-Type",
+    "cookie": "Cookie",
+    "origin": "Origin",
+    "referer": "Referer",
+}
+SAFE_RESPONSE_HEADERS = {
+    "cache-control": "Cache-Control",
+    "content-language": "Content-Language",
+    "content-security-policy": "Content-Security-Policy",
+    "content-type": "Content-Type",
+    "cross-origin-opener-policy": "Cross-Origin-Opener-Policy",
+    "etag": "ETag",
+    "expires": "Expires",
+    "location": "Location",
+    "set-cookie": "Set-Cookie",
+    "vary": "Vary",
+    "x-frame-options": "X-Frame-Options",
+}
+FIXED_BODIES = {
+    400: b"bad request\n",
+    404: b"not found\n",
+    405: b"method not allowed\n",
+    413: b"request too large\n",
+    414: b"request target too long\n",
+    415: b"unsupported media type\n",
+    421: b"misdirected request\n",
+    431: b"request headers too large\n",
+    500: b"internal server error\n",
+    502: b"bad gateway\n",
+    503: b"service unavailable\n",
+    505: b"http version not supported\n",
+}
+
+
+class ConfigurationError(RuntimeError):
+    """A fixed, non-sensitive startup failure."""
+
+
+class ProxyFailure(RuntimeError):
+    def __init__(self, status: int, outcome: str):
+        super().__init__(outcome)
+        self.status = status
+        self.outcome = outcome
+
+
+@dataclass(frozen=True)
+class Configuration:
+    listen_port: int
+    public_host: str
+    backend_port: int
+    static_root: Path
+    release_sha: str
+    static_hashes: dict[str, str]
+    server_context: ssl.SSLContext
+    backend_context: ssl.SSLContext
+
+
+def _canonical_json(value: object) -> bytes:
+    return (
+        json.dumps(
+            value,
+            ensure_ascii=False,
+            allow_nan=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        )
+        + "\n"
+    ).encode("utf-8")
+
+
+def _fixed_log(
+    *, release_sha: str, route: str, method: str, status: int, outcome: str
+) -> None:
+    record = {
+        "event": "local_rehearsal_proxy",
+        "method": method,
+        "outcome": outcome,
+        "release_sha": release_sha,
+        "route": route,
+        "status": status,
+    }
+    print(
+        json.dumps(record, sort_keys=True, separators=(",", ":")),
+        file=sys.stderr,
+        flush=True,
+    )
+
+
+def _absolute_owned_path(raw: str, *, kind: str, private: bool = False) -> Path:
+    path = Path(raw)
+    if not path.is_absolute():
+        raise ConfigurationError(kind)
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except (OSError, RuntimeError):
+        raise ConfigurationError(kind)
+    expected_type = stat.S_ISDIR if kind == "static_root" else stat.S_ISREG
+    if (
+        stat.S_ISLNK(status.st_mode)
+        or not expected_type(status.st_mode)
+        or status.st_uid != os.geteuid()
+        or resolved != path
+        or (not private and status.st_mode & 0o022)
+        or (private and stat.S_IMODE(status.st_mode) != 0o600)
+    ):
+        raise ConfigurationError(kind)
+    return path
+
+
+def _port(value: str, *, kind: str) -> int:
+    if not value.isascii() or not value.isdecimal() or len(value) > 5:
+        raise ConfigurationError(kind)
+    number = int(value)
+    if not 1 <= number <= 65_535:
+        raise ConfigurationError(kind)
+    return number
+
+
+def _public_host(value: str, listen_port: int) -> str:
+    hostname, separator, port_text = value.rpartition(":")
+    if (
+        separator != ":"
+        or hostname != hostname.lower()
+        or DNS_NAME.fullmatch(hostname) is None
+        or _port(port_text, kind="public_host") != listen_port
+    ):
+        raise ConfigurationError("public_host")
+    return value
+
+
+def _safe_static_path(value: object) -> str:
+    if (
+        not isinstance(value, str)
+        or not value
+        or not value.isascii()
+        or STATIC_PATH.fullmatch(value) is None
+    ):
+        raise ConfigurationError("manifest")
+    if any(ord(character) < 0x21 or ord(character) > 0x7E for character in value):
+        raise ConfigurationError("manifest")
+    pure = PurePosixPath(value)
+    if (
+        pure.is_absolute()
+        or value.startswith("/")
+        or value.endswith("/")
+        or "//" in value
+        or "\\" in value
+        or "%" in value
+        or any(part in {"", ".", ".."} for part in value.split("/"))
+    ):
+        raise ConfigurationError("manifest")
+    return value
+
+
+def _read_static_file(root: Path, relative: str) -> bytes:
+    parts = PurePosixPath(relative).parts
+    flags = os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0)
+    directory_flags = flags | getattr(os, "O_DIRECTORY", 0)
+    descriptors: list[int] = []
+    try:
+        descriptor = os.open(root, directory_flags)
+        descriptors.append(descriptor)
+        for part in parts[:-1]:
+            descriptor = os.open(part, directory_flags, dir_fd=descriptor)
+            descriptors.append(descriptor)
+        file_descriptor = os.open(parts[-1], flags, dir_fd=descriptor)
+        descriptors.append(file_descriptor)
+        file_status = os.fstat(file_descriptor)
+        if not stat.S_ISREG(file_status.st_mode) or file_status.st_size > MAX_STATIC_FILE:
+            raise OSError("unsafe static file")
+        chunks: list[bytes] = []
+        remaining = file_status.st_size + 1
+        while remaining:
+            chunk = os.read(file_descriptor, min(65_536, remaining))
+            if not chunk:
+                break
+            chunks.append(chunk)
+            remaining -= len(chunk)
+        data = b"".join(chunks)
+        if len(data) != file_status.st_size or len(data) > MAX_STATIC_FILE:
+            raise OSError("changed static file")
+        return data
+    finally:
+        for descriptor in reversed(descriptors):
+            try:
+                os.close(descriptor)
+            except OSError:
+                pass
+
+
+def _validate_static_tree(root: Path, hashes: dict[str, str]) -> None:
+    observed: set[str] = set()
+    total = 0
+    pending = [(root, "")]
+    while pending:
+        directory, prefix = pending.pop()
+        try:
+            entries = list(os.scandir(directory))
+        except OSError:
+            raise ConfigurationError("static_root")
+        for entry in entries:
+            relative = f"{prefix}/{entry.name}" if prefix else entry.name
+            try:
+                entry_status = entry.stat(follow_symlinks=False)
+            except OSError:
+                raise ConfigurationError("static_root")
+            if stat.S_ISLNK(entry_status.st_mode):
+                raise ConfigurationError("static_root")
+            if stat.S_ISDIR(entry_status.st_mode):
+                pending.append((Path(entry.path), relative))
+            elif stat.S_ISREG(entry_status.st_mode):
+                observed.add(relative)
+            else:
+                raise ConfigurationError("static_root")
+    if observed != set(hashes):
+        raise ConfigurationError("static_root")
+    for relative, expected in hashes.items():
+        try:
+            data = _read_static_file(root, relative)
+        except OSError:
+            raise ConfigurationError("static_root")
+        total += len(data)
+        if total > MAX_STATIC_TOTAL or hashlib.sha256(data).hexdigest() != expected:
+            raise ConfigurationError("static_root")
+
+
+def _sha256_value(value: object) -> str:
+    if not isinstance(value, str) or HEX_DIGEST.fullmatch(value) is None:
+        raise ConfigurationError("manifest")
+    return value
+
+
+def _exact_mapping(value: object, keys: set[str]) -> dict[str, object]:
+    if not isinstance(value, dict) or set(value) != keys:
+        raise ConfigurationError("manifest")
+    return value
+
+
+def _text(value: object) -> str:
+    if (
+        not isinstance(value, str)
+        or not value
+        or not value.isascii()
+        or any(ord(character) < 0x21 or ord(character) > 0x7E for character in value)
+    ):
+        raise ConfigurationError("manifest")
+    return value
+
+
+def _release_path(value: object) -> str:
+    return _safe_static_path(value)
+
+
+def _validate_archive(value: object) -> None:
+    archive = _exact_mapping(
+        value,
+        {
+            "compression",
+            "format",
+            "gid",
+            "group",
+            "mtime",
+            "regular_modes",
+            "uid",
+            "user",
+        },
+    )
+    if (
+        any(type(archive[name]) is not int for name in ("gid", "mtime", "uid"))
+        or archive
+        != {
+            "compression": "none",
+            "format": "ustar",
+            "gid": 0,
+            "group": "root",
+            "mtime": 0,
+            "regular_modes": ["0644", "0755"],
+            "uid": 0,
+            "user": "root",
+        }
+    ):
+        raise ConfigurationError("manifest")
+
+
+def _validate_dependencies(value: object) -> dict[str, str]:
+    dependencies = _exact_mapping(
+        value,
+        {"lock_path", "lock_sha256", "package_set_sha256", "packages"},
+    )
+    if dependencies["lock_path"] != "uv.lock":
+        raise ConfigurationError("manifest")
+    lock_digest = _sha256_value(dependencies["lock_sha256"])
+    package_digest = _sha256_value(dependencies["package_set_sha256"])
+    packages = dependencies["packages"]
+    if not isinstance(packages, list) or not packages:
+        raise ConfigurationError("manifest")
+    inventory: list[tuple[str, str]] = []
+    versions: dict[str, str] = {}
+    for item in packages:
+        package = _exact_mapping(item, {"name", "version"})
+        name = _text(package["name"])
+        version = _text(package["version"])
+        normalized = name.lower().replace("_", "-")
+        if normalized in versions:
+            raise ConfigurationError("manifest")
+        versions[normalized] = version
+        inventory.append((name, version))
+    if inventory != sorted(inventory) or hashlib.sha256(
+        _canonical_json(packages)
+    ).hexdigest() != package_digest:
+        raise ConfigurationError("manifest")
+    versions["__lock_sha256__"] = lock_digest
+    return versions
+
+
+def _validate_runtime(value: object) -> tuple[str, dict[str, str]]:
+    runtime = _exact_mapping(value, {"path", "sha256", "versions"})
+    if runtime["path"] != "runtime/versions.toml":
+        raise ConfigurationError("manifest")
+    digest = _sha256_value(runtime["sha256"])
+    versions = _exact_mapping(
+        runtime["versions"],
+        {
+            "django",
+            "gunicorn",
+            "postgresql",
+            "psycopg",
+            "psycopg_distribution",
+            "python",
+            "uv",
+        },
+    )
+    validated = {name: _text(version) for name, version in versions.items()}
+    if validated["psycopg_distribution"] != "binary-wheel":
+        raise ConfigurationError("manifest")
+    return digest, validated
+
+
+def _validate_source(value: object) -> dict[str, str]:
+    source = _exact_mapping(value, {"tracked", "tracked_set_sha256"})
+    tracked_digest = _sha256_value(source["tracked_set_sha256"])
+    tracked = source["tracked"]
+    if not isinstance(tracked, list) or not tracked:
+        raise ConfigurationError("manifest")
+    result: dict[str, str] = {}
+    observed_paths: list[str] = []
+    proxy_mode: str | None = None
+    for item in tracked:
+        entry = _exact_mapping(item, {"git_mode", "path", "sha256"})
+        path = _release_path(entry["path"])
+        mode = entry["git_mode"]
+        digest = _sha256_value(entry["sha256"])
+        if mode not in {"100644", "100755"} or path in result:
+            raise ConfigurationError("manifest")
+        result[path] = digest
+        observed_paths.append(path)
+        if path == "scripts/local-tls-static-proxy":
+            proxy_mode = mode
+    if (
+        observed_paths != sorted(observed_paths)
+        or hashlib.sha256(_canonical_json(tracked)).hexdigest() != tracked_digest
+        or proxy_mode != "100755"
+    ):
+        raise ConfigurationError("manifest")
+    try:
+        running_digest = hashlib.sha256(Path(__file__).resolve(strict=True).read_bytes()).hexdigest()
+    except OSError:
+        raise ConfigurationError("manifest")
+    if result.get("scripts/local-tls-static-proxy") != running_digest:
+        raise ConfigurationError("manifest")
+    return result
+
+
+def _validate_migrations(value: object, source_hashes: dict[str, str]) -> None:
+    migrations = _exact_mapping(value, {"leaf_set_sha256", "leaves"})
+    leaf_digest = _sha256_value(migrations["leaf_set_sha256"])
+    leaves = migrations["leaves"]
+    if not isinstance(leaves, list) or not leaves:
+        raise ConfigurationError("manifest")
+    identities: list[tuple[str, str]] = []
+    for item in leaves:
+        leaf = _exact_mapping(
+            item, {"app", "module", "name", "origin", "path", "sha256"}
+        )
+        app = _text(leaf["app"])
+        _text(leaf["module"])
+        name = _text(leaf["name"])
+        origin = leaf["origin"]
+        path = _release_path(leaf["path"])
+        digest = _sha256_value(leaf["sha256"])
+        if origin not in {"source", "django"}:
+            raise ConfigurationError("manifest")
+        if origin == "source" and source_hashes.get(path) != digest:
+            raise ConfigurationError("manifest")
+        if origin == "django" and not path.startswith("django/"):
+            raise ConfigurationError("manifest")
+        identities.append((app, name))
+    if identities != sorted(identities) or len(set(identities)) != len(identities):
+        raise ConfigurationError("manifest")
+    if hashlib.sha256(_canonical_json(leaves)).hexdigest() != leaf_digest:
+        raise ConfigurationError("manifest")
+
+
+def _manifest(path: Path, static_root: Path) -> tuple[str, dict[str, str]]:
+    try:
+        raw = path.read_bytes()
+        if len(raw) > 2 * 1_024 * 1_024:
+            raise ConfigurationError("manifest")
+        data = json.loads(raw.decode("utf-8"))
+        if raw != _canonical_json(data):
+            raise ConfigurationError("manifest")
+    except (OSError, UnicodeDecodeError, ValueError, json.JSONDecodeError):
+        raise ConfigurationError("manifest")
+    if (
+        not isinstance(data, dict)
+        or set(data)
+        != {
+            "archive",
+            "dependencies",
+            "git_object_format",
+            "migrations",
+            "release_sha",
+            "runtime",
+            "schema_version",
+            "source",
+            "static",
+        }
+        or type(data.get("schema_version")) is not int
+        or data["schema_version"] != SCHEMA_VERSION
+    ):
+        raise ConfigurationError("manifest")
+    object_format = data.get("git_object_format")
+    release_sha = data.get("release_sha")
+    expected_length = {"sha1": 40, "sha256": 64}.get(object_format)
+    if (
+        not isinstance(release_sha, str)
+        or expected_length is None
+        or len(release_sha) != expected_length
+        or any(character not in "0123456789abcdef" for character in release_sha)
+    ):
+        raise ConfigurationError("manifest")
+    _validate_archive(data["archive"])
+    package_versions = _validate_dependencies(data["dependencies"])
+    runtime_digest, runtime_versions = _validate_runtime(data["runtime"])
+    source_hashes = _validate_source(data["source"])
+    if (
+        source_hashes.get("uv.lock") != package_versions.pop("__lock_sha256__")
+        or source_hashes.get("runtime/versions.toml") != runtime_digest
+        or ".".join(str(part) for part in sys.version_info[:3])
+        != runtime_versions["python"]
+        or package_versions.get("django") != runtime_versions["django"]
+        or package_versions.get("gunicorn") != runtime_versions["gunicorn"]
+        or package_versions.get("psycopg") != runtime_versions["psycopg"]
+        or package_versions.get("psycopg-binary") != runtime_versions["psycopg"]
+    ):
+        raise ConfigurationError("manifest")
+    _validate_migrations(data["migrations"], source_hashes)
+    static = _exact_mapping(
+        data["static"],
+        {
+            "collected",
+            "collected_set_sha256",
+            "origins",
+            "tracked",
+            "tracked_set_sha256",
+        },
+    )
+    collected = static.get("collected")
+    collected_digest = static.get("collected_set_sha256")
+    if (
+        not isinstance(collected, list)
+        or not collected
+        or not isinstance(collected_digest, str)
+        or HEX_DIGEST.fullmatch(collected_digest) is None
+        or hashlib.sha256(_canonical_json(collected)).hexdigest() != collected_digest
+    ):
+        raise ConfigurationError("manifest")
+    hashes: dict[str, str] = {}
+    observed_paths: list[str] = []
+    for item in collected:
+        if not isinstance(item, dict) or set(item) != {"path", "sha256"}:
+            raise ConfigurationError("manifest")
+        relative = _safe_static_path(item["path"])
+        digest = item["sha256"]
+        if (
+            not isinstance(digest, str)
+            or HEX_DIGEST.fullmatch(digest) is None
+            or relative in hashes
+        ):
+            raise ConfigurationError("manifest")
+        hashes[relative] = digest
+        observed_paths.append(relative)
+    if observed_paths != sorted(observed_paths):
+        raise ConfigurationError("manifest")
+    origins = static["origins"]
+    if not isinstance(origins, list) or len(origins) != len(hashes):
+        raise ConfigurationError("manifest")
+    source_origins: list[dict[str, str]] = []
+    origin_paths: list[str] = []
+    for item in origins:
+        origin = _exact_mapping(
+            item, {"collected_path", "origin", "path", "sha256"}
+        )
+        collected_path = _safe_static_path(origin["collected_path"])
+        source_kind = origin["origin"]
+        source_path = _release_path(origin["path"])
+        digest = _sha256_value(origin["sha256"])
+        if (
+            collected_path in origin_paths
+            or source_kind not in {"source", "django"}
+            or hashes.get(collected_path) != digest
+            or (source_kind == "source" and source_hashes.get(source_path) != digest)
+            or (source_kind == "django" and not source_path.startswith("django/"))
+        ):
+            raise ConfigurationError("manifest")
+        origin_paths.append(collected_path)
+        if source_kind == "source":
+            source_origins.append(
+                {
+                    "collected_path": collected_path,
+                    "path": source_path,
+                    "sha256": digest,
+                }
+            )
+    if origin_paths != sorted(hashes):
+        raise ConfigurationError("manifest")
+    tracked = static["tracked"]
+    tracked_digest = _sha256_value(static["tracked_set_sha256"])
+    if (
+        not isinstance(tracked, list)
+        or tracked != source_origins
+        or hashlib.sha256(_canonical_json(tracked)).hexdigest() != tracked_digest
+    ):
+        raise ConfigurationError("manifest")
+    _validate_static_tree(static_root, hashes)
+    return release_sha, hashes
+
+
+def _tls_server_context(cert: Path, key: Path) -> ssl.SSLContext:
+    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    try:
+        context.load_cert_chain(certfile=cert, keyfile=key)
+    except (OSError, ssl.SSLError):
+        raise ConfigurationError("tls")
+    return context
+
+
+def _tls_backend_context(ca: Path) -> ssl.SSLContext:
+    try:
+        context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH, cafile=ca)
+    except (OSError, ssl.SSLError):
+        raise ConfigurationError("backend_ca")
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    context.check_hostname = True
+    context.verify_mode = ssl.CERT_REQUIRED
+    return context
+
+
+def _configuration(arguments: argparse.Namespace) -> Configuration:
+    listen_port = _port(arguments.listen_port, kind="listen_port")
+    backend_port = _port(arguments.backend_port, kind="backend_port")
+    if listen_port == backend_port:
+        raise ConfigurationError("ports")
+    public_host = _public_host(arguments.public_host, listen_port)
+    cert = _absolute_owned_path(arguments.tls_cert_file, kind="tls_cert")
+    key = _absolute_owned_path(arguments.tls_key_file, kind="tls_key", private=True)
+    ca = _absolute_owned_path(arguments.backend_ca_file, kind="backend_ca")
+    static_root = _absolute_owned_path(arguments.static_root, kind="static_root")
+    manifest_path = _absolute_owned_path(arguments.release_manifest, kind="manifest")
+    release_sha, static_hashes = _manifest(manifest_path, static_root)
+    return Configuration(
+        listen_port=listen_port,
+        public_host=public_host,
+        backend_port=backend_port,
+        static_root=static_root,
+        release_sha=release_sha,
+        static_hashes=static_hashes,
+        server_context=_tls_server_context(cert, key),
+        backend_context=_tls_backend_context(ca),
+    )
+
+
+class _BoundedHeaderReader:
+    def __init__(self, raw):
+        self.raw = raw
+        self.total = 0
+
+    def readline(self, size: int = -1) -> bytes:
+        remaining = MAX_HEADER_BYTES - self.total
+        if remaining <= 0:
+            raise http.client.LineTooLong("header block")
+        limit = min(MAX_HEADER_LINE + 1, remaining + 1)
+        if size >= 0:
+            limit = min(limit, size)
+        line = self.raw.readline(limit)
+        self.total += len(line)
+        if len(line) > MAX_HEADER_LINE or self.total > MAX_HEADER_BYTES:
+            raise http.client.LineTooLong("header block")
+        return line
+
+
+class RehearsalServer(ThreadingHTTPServer):
+    daemon_threads = True
+    allow_reuse_address = False
+    request_queue_size = 32
+
+    def __init__(self, configuration: Configuration):
+        self.configuration = configuration
+        self.log_lock = threading.Lock()
+        super().__init__(("127.0.0.1", configuration.listen_port), RehearsalHandler)
+
+    def get_request(self):
+        connection, address = super().get_request()
+        connection.settimeout(SOCKET_TIMEOUT_SECONDS)
+        try:
+            tls_connection = self.configuration.server_context.wrap_socket(
+                connection,
+                server_side=True,
+                do_handshake_on_connect=False,
+            )
+        except (OSError, ssl.SSLError):
+            connection.close()
+            raise
+        return tls_connection, address
+
+    def emit(
+        self, *, route: str, method: str, status: int, outcome: str
+    ) -> None:
+        with self.log_lock:
+            _fixed_log(
+                release_sha=self.configuration.release_sha,
+                route=route,
+                method=method,
+                status=status,
+                outcome=outcome,
+            )
+
+    def handle_error(self, request, client_address) -> None:
+        self.emit(route="internal", method="OTHER", status=500, outcome="handler_error")
+
+
+class RehearsalHandler(BaseHTTPRequestHandler):
+    protocol_version = "HTTP/1.1"
+    server_version = ""
+    sys_version = ""
+
+    @property
+    def configuration(self) -> Configuration:
+        return self.server.configuration  # type: ignore[attr-defined]
+
+    def setup(self) -> None:
+        super().setup()
+        self.connection.settimeout(SOCKET_TIMEOUT_SECONDS)
+
+    def address_string(self) -> str:
+        return "redacted"
+
+    def version_string(self) -> str:
+        return "local-rehearsal"
+
+    def log_message(self, format: str, *args: object) -> None:
+        return
+
+    def log_request(self, code: int | str = "-", size: int | str = "-") -> None:
+        return
+
+    def handle_one_request(self) -> None:
+        try:
+            self.raw_requestline = self.rfile.readline(MAX_REQUEST_TARGET + 32)
+            if len(self.raw_requestline) > MAX_REQUEST_TARGET + 16:
+                self.requestline = ""
+                self.request_version = "HTTP/1.1"
+                self.command = None
+                self._fixed_response(414, "target_too_long", route="invalid")
+                return
+            if not self.raw_requestline:
+                self.close_connection = True
+                return
+            if not self.parse_request():
+                return
+            method_name = "do_" + self.command
+            if not hasattr(self, method_name):
+                self._fixed_response(405, "method_rejected", route="invalid")
+                return
+            getattr(self, method_name)()
+            self.wfile.flush()
+        except TimeoutError:
+            self.close_connection = True
+        except (BrokenPipeError, ConnectionError, OSError, ssl.SSLError):
+            self.close_connection = True
+        except Exception:
+            try:
+                self._fixed_response(500, "handler_error", route="internal")
+            except Exception:
+                self.close_connection = True
+
+    def parse_request(self) -> bool:
+        original = self.rfile
+        bounded = _BoundedHeaderReader(original)
+        self.rfile = bounded
+        try:
+            parsed = super().parse_request()
+        finally:
+            self.rfile = original
+        if not parsed:
+            return False
+        if len(self.headers) > MAX_HEADER_COUNT:
+            self._fixed_response(431, "headers_too_large", route="invalid")
+            return False
+        for name, value in self.headers.items():
+            if (
+                HEADER_NAME.fullmatch(name) is None
+                or len(value) > MAX_HEADER_LINE
+                or any(
+                    character != "\t" and not 0x20 <= ord(character) <= 0x7E
+                    for character in value
+                )
+                or "\r" in value
+                or "\n" in value
+            ):
+                self._fixed_response(400, "header_rejected", route="invalid")
+                return False
+        return True
+
+    def send_error(
+        self,
+        code: int,
+        message: str | None = None,
+        explain: str | None = None,
+    ) -> None:
+        if code == 501:
+            code = 405
+        if code not in FIXED_BODIES:
+            code = 400 if code < 500 else 500
+        self._fixed_response(code, "protocol_rejected", route="invalid")
+
+    def _security_headers(self) -> None:
+        self.send_header("Strict-Transport-Security", HSTS_VALUE)
+        self.send_header("X-Content-Type-Options", "nosniff")
+        self.send_header("Referrer-Policy", "no-referrer")
+        self.send_header("Connection", "close")
+
+    def _emit(self, route: str, status: int, outcome: str) -> None:
+        method = self.command if self.command in {"GET", "HEAD", "POST"} else "OTHER"
+        self.server.emit(  # type: ignore[attr-defined]
+            route=route, method=method, status=status, outcome=outcome
+        )
+
+    def _fixed_response(self, status: int, outcome: str, *, route: str) -> None:
+        body = FIXED_BODIES.get(status, FIXED_BODIES[500])
+        self.close_connection = True
+        self.send_response_only(status)
+        self.send_header("Content-Type", "text/plain; charset=utf-8")
+        self.send_header("Cache-Control", "no-store")
+        self.send_header("Content-Length", str(len(body)))
+        self._security_headers()
+        self.end_headers()
+        if getattr(self, "command", None) != "HEAD":
+            self.wfile.write(body)
+        self._emit(route, status, outcome)
+
+    def _request_target(self) -> tuple[str, str]:
+        try:
+            fields = self.requestline.split()
+            raw_target = fields[1]
+        except (AttributeError, IndexError):
+            raise ProxyFailure(400, "target_rejected")
+        if (
+            len(raw_target) > MAX_REQUEST_TARGET
+            or not raw_target.isascii()
+            or any(ord(character) < 0x21 or ord(character) > 0x7E for character in raw_target)
+            or "\\" in raw_target
+        ):
+            raise ProxyFailure(400, "target_rejected")
+        split = urlsplit(raw_target)
+        if split.scheme or split.netloc or split.fragment or not split.path.startswith("/"):
+            raise ProxyFailure(400, "target_rejected")
+        index = 0
+        while True:
+            index = raw_target.find("%", index)
+            if index < 0:
+                break
+            if (
+                index + 2 >= len(raw_target)
+                or any(character not in "0123456789abcdefABCDEF" for character in raw_target[index + 1 : index + 3])
+            ):
+                raise ProxyFailure(400, "target_rejected")
+            index += 3
+        if "%" in split.path or "//" in split.path:
+            raise ProxyFailure(400, "target_rejected")
+        if any(part in {".", ".."} for part in split.path.split("/")):
+            raise ProxyFailure(400, "target_rejected")
+        return raw_target, split.path
+
+    def _validated_body(self, method: str) -> bytes:
+        if self.headers.get_all("Transfer-Encoding"):
+            raise ProxyFailure(400, "transfer_encoding_rejected")
+        lengths = self.headers.get_all("Content-Length", [])
+        if len(lengths) > 1:
+            raise ProxyFailure(400, "content_length_rejected")
+        length_text = lengths[0] if lengths else "0"
+        if (
+            not length_text.isascii()
+            or not length_text.isdecimal()
+            or len(length_text) > 5
+        ):
+            raise ProxyFailure(400, "content_length_rejected")
+        length = int(length_text)
+        if length > MAX_REQUEST_BODY:
+            raise ProxyFailure(413, "body_too_large")
+        if method != "POST" and length:
+            raise ProxyFailure(400, "unexpected_body")
+        if method == "POST":
+            content_types = self.headers.get_all("Content-Type", [])
+            if content_types != ["application/x-www-form-urlencoded"]:
+                raise ProxyFailure(415, "media_type_rejected")
+        if not length:
+            return b""
+        body = self.rfile.read(length)
+        if len(body) != length:
+            raise ProxyFailure(400, "body_incomplete")
+        return body
+
+    def _host_is_exact(self) -> bool:
+        return self.headers.get_all("Host", []) == [self.configuration.public_host]
+
+    def _connection_nominated_headers(self) -> set[str]:
+        nominated: set[str] = set()
+        for value in self.headers.get_all("Connection", []):
+            for token in value.split(","):
+                token = token.strip().lower()
+                if not token or HEADER_NAME.fullmatch(token) is None:
+                    raise ProxyFailure(400, "connection_header_rejected")
+                nominated.add(token)
+        if nominated & {"content-length", "host", "transfer-encoding"}:
+            raise ProxyFailure(400, "connection_header_rejected")
+        return nominated
+
+    def _static(self, method: str, path: str) -> None:
+        if method not in {"GET", "HEAD"}:
+            raise ProxyFailure(405, "method_rejected")
+        if "?" in self.requestline.split()[1]:
+            raise ProxyFailure(400, "static_query_rejected")
+        relative = path.removeprefix("/static/")
+        if path == "/static/" or _safe_static_path(relative) != relative:
+            raise ProxyFailure(404, "static_not_found")
+        expected = self.configuration.static_hashes.get(relative)
+        if expected is None:
+            raise ProxyFailure(404, "static_not_found")
+        try:
+            data = _read_static_file(self.configuration.static_root, relative)
+        except OSError:
+            raise ProxyFailure(503, "static_unavailable")
+        if hashlib.sha256(data).hexdigest() != expected:
+            raise ProxyFailure(503, "static_integrity_failed")
+        etag = f'"sha256-{expected}"'
+        if_none_match = self.headers.get_all("If-None-Match", [])
+        status = 200
+        response_body = data
+        if if_none_match == [etag]:
+            status = 304
+            response_body = b""
+        elif len(if_none_match) > 1:
+            raise ProxyFailure(400, "header_rejected")
+        content_type, _ = mimetypes.guess_type(relative)
+        self.close_connection = True
+        self.send_response_only(status)
+        self.send_header("Content-Type", content_type or "application/octet-stream")
+        self.send_header("Cache-Control", "no-cache")
+        self.send_header("ETag", etag)
+        self.send_header("Content-Length", str(len(data) if status == 200 else 0))
+        self._security_headers()
+        self.end_headers()
+        if method == "GET" and status == 200:
+            self.wfile.write(response_body)
+        self._emit("static", status, "static_ok" if status == 200 else "not_modified")
+
+    def _forward_headers(self, body: bytes) -> list[tuple[str, str]]:
+        result = [("Host", self.configuration.public_host)]
+        nominated = self._connection_nominated_headers()
+        for lower_name, forwarded_name in FORWARDED_REQUEST_HEADERS.items():
+            if lower_name in nominated:
+                continue
+            values = self.headers.get_all(forwarded_name, [])
+            if len(values) > 1:
+                raise ProxyFailure(400, "header_rejected")
+            if values:
+                result.append((forwarded_name, values[0]))
+        if body or self.command == "POST":
+            result.append(("Content-Length", str(len(body))))
+        return result
+
+    @staticmethod
+    def _safe_header_value(value: str) -> bool:
+        return (
+            len(value) <= MAX_HEADER_LINE
+            and "\r" not in value
+            and "\n" not in value
+            and all(
+                character == "\t" or 0x20 <= ord(character) <= 0x7E
+                for character in value
+            )
+        )
+
+    def _backend(self, method: str, target: str, body: bytes) -> None:
+        connection = http.client.HTTPSConnection(
+            "127.0.0.1",
+            self.configuration.backend_port,
+            timeout=SOCKET_TIMEOUT_SECONDS,
+            context=self.configuration.backend_context,
+        )
+        try:
+            headers = self._forward_headers(body)
+            connection.putrequest(
+                method, target, skip_host=True, skip_accept_encoding=True
+            )
+            for name, value in headers:
+                connection.putheader(name, value)
+            connection.endheaders(body if body else None)
+            response = connection.getresponse()
+            if not 200 <= response.status <= 599:
+                raise ProxyFailure(502, "backend_protocol_failed")
+            response_headers = response.getheaders()
+            if (
+                len(response_headers) > MAX_HEADER_COUNT
+                or sum(len(name) + len(value) + 4 for name, value in response_headers)
+                > MAX_HEADER_BYTES
+            ):
+                raise ProxyFailure(502, "backend_protocol_failed")
+            nominated_response_headers: set[str] = set()
+            for name, value in response_headers:
+                if name.lower() != "connection":
+                    continue
+                if not self._safe_header_value(value):
+                    raise ProxyFailure(502, "backend_protocol_failed")
+                for token in value.split(","):
+                    token = token.strip().lower()
+                    if not token or HEADER_NAME.fullmatch(token) is None:
+                        raise ProxyFailure(502, "backend_protocol_failed")
+                    nominated_response_headers.add(token)
+            if nominated_response_headers & {"content-length", "transfer-encoding"}:
+                raise ProxyFailure(502, "backend_protocol_failed")
+            safe_headers: list[tuple[str, str]] = []
+            seen_response_headers: set[str] = set()
+            upstream_length: int | None = None
+            for name, value in response_headers:
+                lower_name = name.lower()
+                if not self._safe_header_value(value):
+                    raise ProxyFailure(502, "backend_protocol_failed")
+                if lower_name in nominated_response_headers:
+                    continue
+                if lower_name == "content-length":
+                    if upstream_length is not None or not value.isdecimal():
+                        raise ProxyFailure(502, "backend_protocol_failed")
+                    upstream_length = int(value)
+                    if upstream_length > MAX_RESPONSE_BODY:
+                        raise ProxyFailure(502, "backend_response_too_large")
+                    continue
+                forwarded_name = SAFE_RESPONSE_HEADERS.get(lower_name)
+                if forwarded_name is None:
+                    continue
+                if lower_name != "set-cookie" and lower_name in seen_response_headers:
+                    raise ProxyFailure(502, "backend_protocol_failed")
+                seen_response_headers.add(lower_name)
+                if lower_name == "location" and (
+                    not value.startswith("/")
+                    or value.startswith("//")
+                    or not value.isascii()
+                    or "\\" in value
+                    or "%" in value
+                ):
+                    raise ProxyFailure(502, "backend_protocol_failed")
+                safe_headers.append((forwarded_name, value))
+            if method == "HEAD":
+                data = b""
+                content_length = upstream_length or 0
+            else:
+                data = response.read(MAX_RESPONSE_BODY + 1)
+                if len(data) > MAX_RESPONSE_BODY:
+                    raise ProxyFailure(502, "backend_response_too_large")
+                if upstream_length is not None and upstream_length != len(data):
+                    raise ProxyFailure(502, "backend_protocol_failed")
+                content_length = len(data)
+        except ssl.SSLCertVerificationError:
+            raise ProxyFailure(502, "backend_tls_auth_failed")
+        except ssl.SSLError:
+            raise ProxyFailure(502, "backend_tls_failed")
+        except ProxyFailure:
+            raise
+        except http.client.HTTPException:
+            raise ProxyFailure(502, "backend_protocol_failed")
+        except (ConnectionError, TimeoutError, OSError, socket.timeout):
+            raise ProxyFailure(503, "backend_unavailable")
+        finally:
+            connection.close()
+        self.close_connection = True
+        self.send_response_only(response.status)
+        for name, value in safe_headers:
+            self.send_header(name, value)
+        self.send_header("Content-Length", str(content_length))
+        self._security_headers()
+        self.end_headers()
+        if method != "HEAD":
+            self.wfile.write(data)
+        self._emit("dynamic", response.status, "backend_ok")
+
+    def _dispatch(self) -> None:
+        try:
+            if not self._host_is_exact():
+                raise ProxyFailure(421, "host_rejected")
+            target, path = self._request_target()
+            body = self._validated_body(self.command)
+            if path.startswith("/static/"):
+                self._static(self.command, path)
+            else:
+                self._backend(self.command, target, body)
+        except ConfigurationError:
+            self._fixed_response(404, "static_not_found", route="static")
+        except ProxyFailure as failure:
+            route = "static" if "/static/" in getattr(self, "path", "") else "dynamic"
+            self._fixed_response(failure.status, failure.outcome, route=route)
+
+    def do_GET(self) -> None:
+        self._dispatch()
+
+    def do_HEAD(self) -> None:
+        self._dispatch()
+
+    def do_POST(self) -> None:
+        self._dispatch()
+
+    def do_CONNECT(self) -> None:
+        self._fixed_response(405, "method_rejected", route="invalid")
+
+    do_DELETE = do_CONNECT
+    do_OPTIONS = do_CONNECT
+    do_PATCH = do_CONNECT
+    do_PUT = do_CONNECT
+    do_TRACE = do_CONNECT
+
+
+class FixedArgumentParser(argparse.ArgumentParser):
+    def error(self, message: str) -> None:
+        raise ConfigurationError("arguments")
+
+
+def _parse_arguments() -> argparse.Namespace:
+    parser = FixedArgumentParser(
+        description=(
+            "Run the local-only release rehearsal TLS/static proxy; this is not "
+            "a production proxy selection."
+        )
+    )
+    parser.add_argument("--listen-port", required=True)
+    parser.add_argument("--public-host", required=True)
+    parser.add_argument("--backend-port", required=True)
+    parser.add_argument("--backend-ca-file", required=True)
+    parser.add_argument("--tls-cert-file", required=True)
+    parser.add_argument("--tls-key-file", required=True)
+    parser.add_argument("--static-root", required=True)
+    parser.add_argument("--release-manifest", required=True)
+    return parser.parse_args()
+
+
+def main() -> int:
+    try:
+        configuration = _configuration(_parse_arguments())
+        server = RehearsalServer(configuration)
+    except ConfigurationError:
+        _fixed_log(
+            release_sha="unavailable",
+            route="lifecycle",
+            method="NONE",
+            status=64,
+            outcome="configuration_rejected",
+        )
+        return 64
+    except OSError:
+        _fixed_log(
+            release_sha="unavailable",
+            route="lifecycle",
+            method="NONE",
+            status=69,
+            outcome="bind_failed",
+        )
+        return 69
+
+    stop = threading.Event()
+
+    def request_stop(signum, frame) -> None:
+        stop.set()
+
+    signal.signal(signal.SIGINT, request_stop)
+    signal.signal(signal.SIGTERM, request_stop)
+    server.timeout = 0.2
+    server.emit(route="lifecycle", method="NONE", status=0, outcome="ready")
+    try:
+        while not stop.is_set():
+            server.handle_request()
+    finally:
+        server.server_close()
+        server.emit(route="lifecycle", method="NONE", status=0, outcome="stopped")
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


