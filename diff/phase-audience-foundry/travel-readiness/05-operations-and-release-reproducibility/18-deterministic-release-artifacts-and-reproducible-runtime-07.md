## `build: attest reproducible dependencies`

diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index d165d36..590f97c 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -9,6 +9,7 @@ vendoring하지 않습니다.
 | 구성요소 | revision | license | upstream와 목적 |
 | --- | --- | --- | --- |
 | CPython | 3.14.7 | PSF License Version 2 | `https://www.python.org/`; application interpreter |
+| python-build-standalone | tag `20260825`, commit `c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779` | MPL-2.0 | `https://github.com/astral-sh/python-build-standalone`; CPython redistributable provider |
 | uv | 0.12.6, commit `7938ca5d53dbb9c614a4a030df406e41ff101ab9` | MIT OR Apache-2.0 | `https://github.com/astral-sh/uv`; resolver와 frozen installer |
 | PostgreSQL | 18.6 | PostgreSQL License | `https://www.postgresql.org/`; canonical database version decision |
 
@@ -19,6 +20,21 @@ production Linux target은 아직 선택하거나 내려받지 않았습니다.
 3.10+를 지원하는 Gunicorn 26.2.0 sync worker로 고정했으며 실제 platform worker 수와
 resource limit은 배포 checkpoint에 남깁니다.
 
+macOS arm64 CPython은 Astral의 immutable python-build-standalone `20260825` release가
+제공한 exact asset
+`cpython-3.14.7+20260825-aarch64-apple-darwin-install_only_stripped.tar.gz`입니다. 공식
+asset URL은
+`https://github.com/astral-sh/python-build-standalone/releases/download/20260825/cpython-3.14.7%2B20260825-aarch64-apple-darwin-install_only_stripped.tar.gz`,
+크기는 `26488085` bytes, SHA-256은
+`17ecb3d29c49765856370cbb47d948a24ec518be40f607362c1bbb3ebbc5c442`입니다. 공식 GitHub
+release API의 asset name·size·digest와 로컬 SHA-256을 일치시켰고, immutable release가
+가리키는 GitHub-verified signed commit
+`c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779`도 확인했습니다. Gate 자체는 이 archive와
+전체 pristine extraction tree, executable digest를 다시 검사하며 반드시 pinned absolute
+interpreter를 `-I -S -B scripts/check-dependencies ... --python-bin <같은 absolute interpreter>`
+형태로 실행해야 합니다. `scripts/check-dependencies` 자체를 executable로 직접 실행하면
+fail closed합니다.
+
 ## lock에 포함된 Python distribution
 
 | dependency | 관계 | license | upstream | 사용 목적 |
diff --git a/operations/tests/test_dependency_gate.py b/operations/tests/test_dependency_gate.py
new file mode 100644
index 0000000..4903736
--- /dev/null
+++ b/operations/tests/test_dependency_gate.py
@@ -0,0 +1,1719 @@
+from __future__ import annotations
+
+import ast
+import base64
+import email.message
+import hashlib
+import io
+import importlib.machinery
+import importlib.metadata
+import importlib.util
+import json
+import os
+from pathlib import Path
+import shutil
+import stat
+import subprocess
+import sys
+import tempfile
+import unittest
+import urllib.error
+import zipfile
+from unittest import mock
+
+
+SCRIPT = Path(__file__).resolve().parents[2] / "scripts" / "check-dependencies"
+
+
+def load_gate_module():
+    loader = importlib.machinery.SourceFileLoader("dependency_gate_script", str(SCRIPT))
+    specification = importlib.util.spec_from_loader(loader.name, loader)
+    if specification is None:
+        raise AssertionError("dependency gate module specification unavailable")
+    module = importlib.util.module_from_spec(specification)
+    loader.exec_module(module)
+    return module
+
+
+gate = load_gate_module()
+
+
+class FakeResponse:
+    def __init__(
+        self,
+        document: object,
+        *,
+        final_url: str,
+        status: int = 200,
+        headers: dict[str, str] | None = None,
+        raw: bytes | None = None,
+    ):
+        self.status = status
+        self._final_url = final_url
+        self._body = (
+            json.dumps(document, separators=(",", ":")).encode("utf-8")
+            if raw is None
+            else raw
+        )
+        self.headers = email.message.Message()
+        self.headers["Content-Type"] = "application/json"
+        self.headers["Content-Length"] = str(len(self._body))
+        for name, value in (headers or {}).items():
+            if name in self.headers:
+                del self.headers[name]
+            self.headers[name] = value
+        self.closed = False
+
+    def read(self, maximum: int) -> bytes:
+        return self._body[:maximum]
+
+    def geturl(self) -> str:
+        return self._final_url
+
+    def close(self) -> None:
+        self.closed = True
+
+
+class FakeOpener:
+    def __init__(self, response_or_error):
+        self.response_or_error = response_or_error
+        self.calls = []
+
+    def open(self, request, *, timeout):
+        self.calls.append((request, timeout))
+        if isinstance(self.response_or_error, BaseException):
+            raise self.response_or_error
+        return self.response_or_error
+
+
+def synthetic_wheel(files: dict[str, bytes]) -> bytes:
+    rows = []
+    for filename, content in sorted(files.items()):
+        encoded = base64.urlsafe_b64encode(hashlib.sha256(content).digest()).rstrip(
+            b"="
+        ).decode("ascii")
+        rows.append(f"{filename},sha256={encoded},{len(content)}\n")
+    record_name = "fixture-1.0.dist-info/RECORD"
+    rows.append(f"{record_name},,\n")
+    buffer = io.BytesIO()
+    with zipfile.ZipFile(buffer, mode="w", compression=zipfile.ZIP_DEFLATED) as archive:
+        for filename, content in files.items():
+            archive.writestr(filename, content)
+        archive.writestr(record_name, "".join(rows).encode("utf-8"))
+    return buffer.getvalue()
+
+
+class WheelAttestationTests(unittest.TestCase):
+    def artifact(self, body: bytes) -> dict[str, object]:
+        return {
+            "digest": hashlib.sha256(body).hexdigest(),
+            "packagetype": "bdist_wheel",
+            "size": len(body),
+            "url": (
+                "https://files.pythonhosted.org/packages/aa/bb/fixture/"
+                "fixture-1.0-py3-none-any.whl"
+            ),
+        }
+
+    def test_exact_official_wheel_and_record_are_attested(self):
+        files = {
+            "fixture/__init__.py": b"VALUE = 1\n",
+            "fixture-1.0.dist-info/METADATA": b"Name: fixture\nVersion: 1.0\n",
+        }
+        body = synthetic_wheel(files)
+        attestation = gate.attest_exact_wheel(body, self.artifact(body))
+        self.assertEqual(set(attestation), set(files))
+        for filename, content in files.items():
+            self.assertEqual(
+                attestation[filename],
+                {
+                    "digest": hashlib.sha256(content).hexdigest(),
+                    "size": len(content),
+                },
+            )
+
+    def test_wheel_digest_record_and_archive_paths_fail_closed(self):
+        body = synthetic_wheel({"fixture/__init__.py": b"VALUE = 1\n"})
+        wrong_artifact = dict(self.artifact(body))
+        wrong_artifact["digest"] = "0" * 64
+        with self.assertRaises(gate.GateError):
+            gate.attest_exact_wheel(body, wrong_artifact)
+
+        buffer = io.BytesIO()
+        with zipfile.ZipFile(buffer, mode="w") as archive:
+            archive.writestr("../escape.py", b"fixture")
+            archive.writestr(
+                "fixture-1.0.dist-info/RECORD",
+                b"../escape.py,sha256=invalid,7\n"
+                b"fixture-1.0.dist-info/RECORD,,\n",
+            )
+        unsafe = buffer.getvalue()
+        with self.assertRaises(gate.GateError):
+            gate.attest_exact_wheel(unsafe, self.artifact(unsafe))
+
+        buffer = io.BytesIO()
+        with zipfile.ZipFile(buffer, mode="w") as archive:
+            archive.writestr("fixture/__init__.py", b"VALUE = 1\n")
+            archive.writestr(
+                "fixture-1.0.dist-info/RECORD",
+                b"fixture/__init__.py,sha256=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA,10\n"
+                b"fixture-1.0.dist-info/RECORD,,\n",
+            )
+        tampered = buffer.getvalue()
+        with self.assertRaises(gate.GateError):
+            gate.attest_exact_wheel(tampered, self.artifact(tampered))
+
+    def test_fetch_requires_exact_size_digest_url_and_bounded_headers(self):
+        body = synthetic_wheel({"fixture/__init__.py": b"VALUE = 1\n"})
+        artifact = self.artifact(body)
+        response = FakeResponse(
+            {},
+            final_url=artifact["url"],
+            raw=body,
+            headers={"Content-Type": "application/octet-stream"},
+        )
+        download = gate.fetch_exact_wheel(
+            artifact, opener=FakeOpener(response)
+        )
+        self.assertEqual(download["body"], body)
+        self.assertIn("fixture/__init__.py", download["records"])
+        self.assertEqual(download["digest"], artifact["digest"])
+        self.assertEqual(download["size"], len(body))
+        redirected = FakeResponse(
+            {},
+            final_url="https://files.pythonhosted.org/packages/aa/bb/fixture/other.whl",
+            raw=body,
+            headers={"Content-Type": "application/octet-stream"},
+        )
+        with self.assertRaises(gate.GateError):
+            gate.fetch_exact_wheel(artifact, opener=FakeOpener(redirected))
+
+
+class DependencyAdvisoryTests(unittest.TestCase):
+    name = "django"
+    version = "5.2.17"
+
+    @property
+    def url(self) -> str:
+        return f"https://pypi.org/pypi/{self.name}/{self.version}/json"
+
+    @property
+    def expected_artifacts(self):
+        filename = "django-5.2.17-py3-none-any.whl"
+        return {
+            filename: {
+                "digest": "a" * 64,
+                "packagetype": "bdist_wheel",
+                "size": 12345,
+                "url": f"https://files.pythonhosted.org/packages/aa/bb/fixture/{filename}",
+            },
+        }
+
+    def document(self, *, vulnerable=False, yanked=False):
+        filename, expected = next(iter(self.expected_artifacts.items()))
+        return {
+            "last_serial": 123456,
+            "info": {
+                "name": "Django",
+                "version": self.version,
+                "yanked": yanked,
+                "yanked_reason": None,
+            },
+            "urls": [
+                {
+                    "digests": {"sha256": expected["digest"]},
+                    "filename": filename,
+                    "packagetype": expected["packagetype"],
+                    "python_version": "py3",
+                    "requires_python": ">=3.10",
+                    "size": expected["size"],
+                    "upload_time_iso_8601": "2026-08-04T15:03:59.100Z",
+                    "url": expected["url"],
+                    "yanked": yanked,
+                    "yanked_reason": None,
+                }
+            ],
+            "vulnerabilities": ([{"id": "fixture-only"}] if vulnerable else []),
+        }
+
+    def assert_failed(self, response_or_error):
+        opener = FakeOpener(response_or_error)
+        with self.assertRaises(gate.GateError) as raised:
+            gate.check_one_advisory(
+                self.name,
+                self.version,
+                expected_artifacts=self.expected_artifacts,
+                opener=opener,
+            )
+        self.assertEqual(str(raised.exception), gate.FAILURE_RECEIPT)
+
+    def test_positive_metadata_uses_bounded_anonymous_request(self):
+        response = FakeResponse(self.document(), final_url=self.url)
+        opener = FakeOpener(response)
+
+        gate.check_one_advisory(
+            self.name,
+            self.version,
+            expected_artifacts=self.expected_artifacts,
+            opener=opener,
+        )
+
+        self.assertTrue(response.closed)
+        self.assertEqual(len(opener.calls), 1)
+        request, timeout = opener.calls[0]
+        self.assertEqual(timeout, 5)
+        self.assertEqual(request.method, "GET")
+        self.assertEqual(request.full_url, self.url)
+        headers = {name.lower(): value for name, value in request.header_items()}
+        self.assertEqual(
+            headers,
+            {
+                "accept": "application/json",
+                "accept-encoding": "identity",
+                "user-agent": "travel-readiness-dependency-gate/1",
+            },
+        )
+        self.assertNotIn("authorization", headers)
+        self.assertNotIn("cookie", headers)
+
+    def test_vulnerability_fails_closed(self):
+        self.assert_failed(FakeResponse(self.document(vulnerable=True), final_url=self.url))
+
+    def test_yanked_release_or_artifact_fails_closed(self):
+        self.assert_failed(FakeResponse(self.document(yanked=True), final_url=self.url))
+        artifact_yanked = self.document()
+        artifact_yanked["urls"][0]["yanked"] = True
+        self.assert_failed(FakeResponse(artifact_yanked, final_url=self.url))
+
+    def test_outage_fails_closed_without_error_detail(self):
+        self.assert_failed(urllib.error.URLError("synthetic sensitive transport detail"))
+
+    def test_schema_drift_fails_closed(self):
+        for document in (
+            [],
+            {"info": [], "urls": [], "vulnerabilities": []},
+            {
+                "info": {"name": "Django", "version": self.version},
+                "urls": [{"yanked": False}],
+                "vulnerabilities": [],
+                "last_serial": 1,
+            },
+            {
+                "info": {
+                    "name": "Django",
+                    "version": self.version,
+                    "yanked": False,
+                },
+                "urls": [{"yanked": "false"}],
+                "vulnerabilities": [],
+                "last_serial": 1,
+            },
+        ):
+            with self.subTest(document=document):
+                self.assert_failed(FakeResponse(document, final_url=self.url))
+
+    def test_release_files_must_exactly_match_lock_names_sizes_and_digests(self):
+        for field, value in (
+            ("filename", "different.whl"),
+            ("size", 12346),
+            ("packagetype", "sdist"),
+            ("url", "https://files.pythonhosted.org/packages/aa/bb/fixture/different.whl"),
+        ):
+            document = self.document()
+            document["urls"][0][field] = value
+            with self.subTest(field=field):
+                self.assert_failed(FakeResponse(document, final_url=self.url))
+        document = self.document()
+        document["urls"][0]["digests"]["sha256"] = "b" * 64
+        self.assert_failed(FakeResponse(document, final_url=self.url))
+
+    def test_oversize_body_and_declared_length_fail_closed(self):
+        oversized = b"{" + b" " * gate.MAX_METADATA_BYTES + b"}"
+        self.assert_failed(FakeResponse({}, final_url=self.url, raw=oversized))
+        self.assert_failed(
+            FakeResponse(
+                self.document(),
+                final_url=self.url,
+                headers={"Content-Length": str(gate.MAX_METADATA_BYTES + 1)},
+            )
+        )
+        self.assert_failed(
+            FakeResponse(
+                self.document(),
+                final_url=self.url,
+                headers={"Content-Length": ""},
+            )
+        )
+
+    def test_redirect_outside_exact_official_endpoint_fails_closed(self):
+        for redirected in (
+            f"https://example.invalid/pypi/{self.name}/{self.version}/json",
+            f"http://pypi.org/pypi/{self.name}/{self.version}/json",
+            f"https://pypi.org/pypi/{self.name}/{self.version}/json?token=fixture",
+            f"https://pypi.org/pypi/{self.name}/latest/json",
+        ):
+            with self.subTest(redirected=redirected):
+                self.assert_failed(
+                    FakeResponse(self.document(), final_url=redirected)
+                )
+
+    def test_redirect_handler_rejects_an_intermediate_origin_change(self):
+        request = gate.urllib.request.Request(self.url)
+        expected_path = f"/pypi/{self.name}/{self.version}/json"
+        setattr(request, "dependency_gate_expected_path", expected_path)
+        with self.assertRaises(gate.GateError):
+            gate.SameOfficialOriginRedirect().redirect_request(
+                request,
+                None,
+                302,
+                "fixture",
+                email.message.Message(),
+                f"https://example.invalid{expected_path}",
+            )
+
+    def test_shared_network_deadline_fails_closed(self):
+        with mock.patch.object(gate.time, "monotonic", return_value=11.0):
+            with self.assertRaises(gate.GateError):
+                gate.bounded_network_timeout(10.0)
+            with self.assertRaises(gate.GateError):
+                gate.require_network_deadline(10.0)
+            self.assertEqual(gate.bounded_network_timeout(None), 5.0)
+
+    def test_network_alarm_bounds_a_slow_response_body(self):
+        class SlowResponse(FakeResponse):
+            def read(self, maximum: int) -> bytes:
+                gate.time.sleep(0.2)
+                return super().read(maximum)
+
+        response = SlowResponse(self.document(), final_url=self.url)
+        with self.assertRaises(gate.GateError):
+            gate.check_one_advisory(
+                self.name,
+                self.version,
+                expected_artifacts=self.expected_artifacts,
+                opener=FakeOpener(response),
+                deadline=gate.time.monotonic() + 0.02,
+            )
+        self.assertEqual(gate.signal.getsignal(gate.signal.SIGALRM), gate.signal.SIG_DFL)
+        self.assertEqual(gate.signal.getitimer(gate.signal.ITIMER_REAL), (0.0, 0.0))
+
+
+class DependencyContractTests(unittest.TestCase):
+    def test_policy_snapshot_is_double_read_and_fails_on_change(self):
+        source = SCRIPT.parent.parent
+        stable = gate.capture_policy_snapshot(source)
+        self.assertEqual(set(stable), set(gate.POLICY_FILE_LIMITS))
+        call_count = 0
+
+        def changing_read(root, relative, *, maximum):
+            nonlocal call_count
+            call_count += 1
+            data = stable[relative]
+            if call_count > len(gate.POLICY_FILE_LIMITS) and relative == "uv.lock":
+                return data + b"\n"
+            return data
+
+        with mock.patch.object(
+            gate, "read_regular_file", side_effect=changing_read
+        ), self.assertRaises(gate.GateError):
+            gate.capture_policy_snapshot(source)
+
+    def test_final_policy_snapshot_change_fails_closed(self):
+        source = SCRIPT.parent.parent
+        stable = gate.capture_policy_snapshot(source)
+        changed = dict(stable)
+        changed["pyproject.toml"] += b"\n"
+        with mock.patch.object(
+            gate, "capture_policy_snapshot", return_value=changed
+        ), self.assertRaises(gate.GateError):
+            gate.verify_policy_snapshot_unchanged(source, stable)
+
+    def test_notice_content_is_exact_not_a_substring_claim(self):
+        original = (SCRIPT.parent.parent / "THIRD_PARTY_NOTICES.md").read_bytes()
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            notice = root / "THIRD_PARTY_NOTICES.md"
+            notice.write_bytes(original + b"\ncontradictory fixture\n")
+            with self.assertRaises(gate.GateError):
+                gate.verify_notices(root)
+
+    def test_controlled_tls_ignores_parent_ssl_and_proxy_environment(self):
+        ca_file = Path("/private/etc/ssl/cert.pem")
+        gate.verify_ca_provenance(ca_file)
+        with mock.patch.dict(
+            gate.os.environ,
+            {
+                "SSLKEYLOGFILE": "/synthetic/should-not-be-used",
+                "SSL_CERT_FILE": "/synthetic/should-not-be-used",
+                "SSL_CERT_DIR": "/synthetic/should-not-be-used",
+                "HTTPS_PROXY": "http://synthetic.invalid",
+            },
+            clear=False,
+        ), mock.patch.object(
+            gate.urllib.request,
+            "build_opener",
+            wraps=gate.urllib.request.build_opener,
+        ) as build_opener:
+            context = gate.controlled_ssl_context(ca_file)
+            opener = gate.official_opener(ca_file)
+        self.assertTrue(context.check_hostname)
+        self.assertEqual(context.verify_mode, gate.ssl.CERT_REQUIRED)
+        self.assertEqual(context.minimum_version, gate.ssl.TLSVersion.TLSv1_2)
+        self.assertIsNone(context.keylog_filename)
+        proxy_handlers = [
+            handler
+            for handler in build_opener.call_args.args
+            if isinstance(handler, gate.urllib.request.ProxyHandler)
+        ]
+        self.assertEqual(len(proxy_handlers), 1)
+        self.assertEqual(proxy_handlers[0].proxies, {})
+        self.assertTrue(opener.handlers)
+
+    def test_repository_contract_files_match_the_exact_gate(self):
+        root = SCRIPT.parent.parent
+        self.assertEqual(gate.parse_runtime(root), gate.EXPECTED_RUNTIME)
+        gate.parse_project(root)
+        self.assertEqual(gate.parse_lock(root), gate.EXPECTED_LOCK_PACKAGES)
+        gate.verify_notices(root)
+
+    def test_lock_graph_markers_and_extras_fail_closed_when_stale(self):
+        original = (SCRIPT.parent.parent / "uv.lock").read_text(encoding="utf-8")
+        mutations = {
+            "graph": original.replace(
+                '    { name = "asgiref" },\n', "", 1
+            ),
+            "marker": original.replace(
+                "sys_platform == 'win32'",
+                "sys_platform != 'win32'",
+                1,
+            ),
+            "extra": original.replace(
+                'extra = ["binary"]', 'extra = ["source"]', 1
+            ),
+            "metadata": original.replace(
+                'extras = ["binary"]', 'extras = ["source"]', 1
+            ),
+        }
+        for label, mutated in mutations.items():
+            self.assertNotEqual(mutated, original)
+            with self.subTest(label=label), tempfile.TemporaryDirectory() as temporary:
+                root = Path(temporary)
+                (root / "uv.lock").write_text(mutated, encoding="utf-8")
+                with self.assertRaises(gate.GateError):
+                    gate.parse_lock(root)
+
+    def test_all_filesystem_inputs_must_be_explicit_absolute_paths(self):
+        for kind in ("directory", "executable"):
+            with self.subTest(kind=kind), self.assertRaises(gate.GateError):
+                gate.safe_absolute_path("relative/fixture", kind=kind)
+
+    def test_filesystem_inputs_reject_symlinks_and_shared_writable_paths(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary)
+            executable = root / "tool"
+            executable.write_bytes(b"fixture")
+            executable.chmod(0o700)
+            link = root / "tool-link"
+            link.symlink_to(executable)
+            self.assertEqual(
+                gate.safe_absolute_path(str(executable), kind="executable"),
+                executable,
+            )
+            with self.assertRaises(gate.GateError):
+                gate.safe_absolute_path(str(link), kind="executable")
+            writable = root / "writable"
+            writable.mkdir(mode=0o700)
+            writable.chmod(0o722)
+            with self.assertRaises(gate.GateError):
+                gate.safe_absolute_path(str(writable), kind="directory")
+
+    def test_current_platform_policy_binds_exact_executable_and_ca_digests(self):
+        policy = gate.current_platform_policy()
+        for field in (
+            "dependency_environment_sha256",
+            "python_executable_sha256",
+            "python_tree_sha256",
+            "uv_executable_sha256",
+            "ca_sha256",
+        ):
+            self.assertRegex(policy[field], r"\A[0-9a-f]{64}\Z")
+        self.assertEqual(
+            policy["python_relative"],
+            "python/bin/python3.14",
+        )
+        self.assertEqual(policy["python_archive_relative"], "cpython.tar.gz")
+        self.assertEqual(policy["uv_relative"], "uv-aarch64-apple-darwin/uv")
+        self.assertEqual(
+            set(policy["lock_wheel_filenames"]),
+            set(gate.EXPECTED_LOCK_PACKAGES)
+            - {"audience-foundry-travel-readiness"},
+        )
+
+    def test_focused_suite_uses_the_pristine_pinned_python_artifact(self):
+        self.assertEqual(tuple(sys.version_info[:3]), (3, 14, 7))
+        self.assertEqual(gate.platform.machine(), "arm64")
+        executable = Path(sys.executable)
+        self.assertTrue(executable.is_absolute())
+        self.assertEqual(executable.name, "python3.14")
+        self.assertEqual(executable.parent.name, "bin")
+        policy = gate.current_platform_policy()
+        self.assertEqual(
+            gate.sha256_regular_file(executable, maximum=128 * 1024 * 1024),
+            policy["python_executable_sha256"],
+        )
+        artifact_root = executable.parents[1]
+        self.assertEqual(
+            gate.python_artifact_tree_sha256(artifact_root),
+            policy["python_tree_sha256"],
+        )
+        archive = executable.parents[2] / "cpython.tar.gz"
+        self.assertEqual(archive.stat().st_size, gate.EXPECTED_PYTHON_ARCHIVE_SIZE)
+        self.assertEqual(
+            gate.sha256_regular_file(archive, maximum=32 * 1024 * 1024),
+            gate.EXPECTED_PYTHON_ARCHIVE_SHA256,
+        )
+
+    def test_python_tree_rejects_added_or_mutated_bytecode(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary) / "python"
+            cache = root / "lib/python3.14/encodings/__pycache__"
+            cache.mkdir(mode=0o700, parents=True)
+            bytecode = cache / "fixture.cpython-314.pyc"
+            bytecode.write_bytes(b"pinned-bytecode")
+            (root / "runtime.py").write_bytes(b"VALUE = 1\n")
+            relative = bytecode.relative_to(root).as_posix()
+            expected = {relative: hashlib.sha256(bytecode.read_bytes()).hexdigest()}
+            with (
+                mock.patch.object(gate, "EXPECTED_PYTHON_BYTECODE", expected),
+                mock.patch.object(gate, "MIN_PYTHON_TREE_ENTRIES", 1),
+                mock.patch.object(gate, "MIN_PYTHON_TREE_BYTES", 1),
+            ):
+                self.assertRegex(
+                    gate.python_artifact_tree_sha256(root),
+                    r"\A[0-9a-f]{64}\Z",
+                )
+                extra = cache / "added.cpython-314.pyc"
+                extra.write_bytes(b"unapproved")
+                with self.assertRaises(gate.GateError):
+                    gate.python_artifact_tree_sha256(root)
+                extra.unlink()
+                bytecode.write_bytes(b"mutated-bytecode")
+                with self.assertRaises(gate.GateError):
+                    gate.python_artifact_tree_sha256(root)
+
+    def test_runner_requires_exact_interpreter_and_isolated_no_site_flags(self):
+        gate.enforce_gate_runner(sys.executable)
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            ambient = Path(temporary)
+            marker = ambient / "ambient-ran"
+            (ambient / "sitecustomize.py").write_text(
+                "from pathlib import Path\n"
+                f"Path({str(marker)!r}).write_text('ran', encoding='utf-8')\n",
+                encoding="utf-8",
+            )
+            arguments = [
+                "--source-dir",
+                str(SCRIPT.parent.parent),
+                "--python-bin",
+                sys.executable,
+                "--uv-bin",
+                "/private/tmp/nonexistent-uv-fixture",
+                "--cache-dir",
+                "/private/tmp/nonexistent-cache-fixture",
+                "--ca-file",
+                "/private/etc/ssl/cert.pem",
+                "--environment-dir",
+                str(ambient / ".venv"),
+            ]
+            unsafe = subprocess.run(
+                [sys.executable, str(SCRIPT), *arguments],
+                cwd=SCRIPT.parent.parent,
+                env={
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                    "PYTHONPATH": str(ambient),
+                },
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                timeout=10,
+                check=False,
+            )
+            self.assertEqual(unsafe.returncode, 1)
+            self.assertEqual(unsafe.stdout, b"")
+            self.assertEqual(
+                unsafe.stderr, f"{gate.FAILURE_RECEIPT}\n".encode("ascii")
+            )
+            self.assertTrue(marker.exists())
+            marker.unlink()
+            safe = subprocess.run(
+                [sys.executable, "-I", "-S", "-B", str(SCRIPT), *arguments],
+                cwd=SCRIPT.parent.parent,
+                env={"PATH": "/usr/bin:/bin", "PYTHONPATH": str(ambient)},
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                timeout=10,
+                check=False,
+            )
+            self.assertEqual(safe.returncode, 1)
+            self.assertEqual(safe.stdout, b"")
+            self.assertEqual(
+                safe.stderr, f"{gate.FAILURE_RECEIPT}\n".encode("ascii")
+            )
+            self.assertFalse(marker.exists())
+            direct = subprocess.run(
+                [str(SCRIPT), *arguments],
+                cwd=SCRIPT.parent.parent,
+                env={"PATH": "/usr/bin:/bin"},
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                timeout=10,
+                check=False,
+            )
+            self.assertNotEqual(direct.returncode, 0)
+            self.assertEqual(direct.stdout, b"")
+            self.assertEqual(direct.stderr, b"")
+
+    def test_lock_includes_marker_excluded_tzdata_in_advisory_set(self):
+        locked = dict(gate.EXPECTED_LOCK_PACKAGES)
+        artifacts = {
+            name: (
+                {}
+                if name == "audience-foundry-travel-readiness"
+                else {
+                    f"{name}.whl": {
+                        "digest": "a" * 64,
+                        "packagetype": "bdist_wheel",
+                        "size": 1,
+                        "url": "fixture",
+                    }
+                }
+            )
+            for name in locked
+        }
+        calls = []
+
+        def capture(name, version, *, expected_artifacts, opener, deadline=None):
+            calls.append((name, version, expected_artifacts, opener, deadline))
+
+        sentinel = object()
+        with mock.patch.object(gate, "check_one_advisory", side_effect=capture):
+            gate.check_advisories(locked, artifacts, opener=sentinel)
+
+        expected_registry = dict(gate.EXPECTED_LOCK_PACKAGES)
+        expected_registry.pop("audience-foundry-travel-readiness")
+        self.assertEqual(
+            [(name, version) for name, version, _, _, _ in calls],
+            sorted(expected_registry.items()),
+        )
+        self.assertTrue(
+            all(opener is sentinel for _, _, _, opener, _ in calls)
+        )
+        self.assertTrue(
+            any(
+                name == "tzdata"
+                and version == "2026.3"
+                and expected_artifacts is artifacts["tzdata"]
+                and opener is sentinel
+                for name, version, expected_artifacts, opener, _ in calls
+            )
+        )
+
+    def test_only_exact_python_abi_binary_wheels_are_lock_relevant(self):
+        self.assertTrue(
+            gate.artifact_is_relevant_to_exact_lock(
+                "psycopg-binary",
+                "fixture-cp314-cp314-macosx_11_0_arm64.whl",
+                "bdist_wheel",
+            )
+        )
+        self.assertFalse(
+            gate.artifact_is_relevant_to_exact_lock(
+                "psycopg-binary",
+                "fixture-cp313-cp313-macosx_11_0_arm64.whl",
+                "bdist_wheel",
+            )
+        )
+        self.assertTrue(
+            gate.artifact_is_relevant_to_exact_lock(
+                "django", "django-5.2.17.tar.gz", "sdist"
+            )
+        )
+
+    def test_installed_set_excludes_local_project_and_non_windows_tzdata(self):
+        expected = dict(gate.EXPECTED_LOCK_PACKAGES)
+        expected.pop("audience-foundry-travel-readiness")
+        if gate.sys.platform != "win32":
+            expected.pop("tzdata")
+        gate.verify_inspection_report(
+            {
+                "installed": expected,
+                "native_machine": gate.platform.machine(),
+                "native_count": 12,
+                "native_role_count": 5,
+                "site_files_verified": True,
+                "wheel_count": len(expected),
+            }
+        )
+        extra = dict(expected)
+        extra["unapproved-extra"] = "1.0"
+        with self.assertRaises(gate.GateError):
+            gate.verify_inspection_report(
+                {
+                    "installed": extra,
+                    "native_machine": gate.platform.machine(),
+                    "native_count": 12,
+                    "native_role_count": 5,
+                    "site_files_verified": True,
+                    "wheel_count": len(extra),
+                }
+            )
+
+    def test_uv_version_requires_integrity_commit_prefix(self):
+        good = f"uv 0.12.6 ({gate.EXPECTED_UV_COMMIT[:9]} 2026-08-25 aarch64-apple-darwin)\n".encode()
+        bad = b"uv 0.12.6 (000000000 2026-08-25 aarch64-apple-darwin)\n"
+        with mock.patch.object(gate, "run_fixed", return_value=good):
+            gate.exact_uv_version(
+                Path("/fixture/uv"),
+                cwd=Path("/fixture"),
+                environment={},
+            )
+        with mock.patch.object(gate, "run_fixed", return_value=bad):
+            with self.assertRaises(gate.GateError):
+                gate.exact_uv_version(
+                    Path("/fixture/uv"),
+                    cwd=Path("/fixture"),
+                    environment={},
+                )
+
+    def test_subprocess_output_is_bounded_and_never_returned_on_failure(self):
+        environment = {
+            "LANG": "C",
+            "LC_ALL": "C",
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+        }
+        with mock.patch.object(
+            gate.os, "killpg", wraps=gate.os.killpg
+        ) as killpg, self.assertRaises(gate.GateError) as raised:
+            gate.run_fixed(
+                [
+                    sys.executable,
+                    "-I",
+                    "-S",
+                    "-B",
+                    "-c",
+                    f"print('x' * {gate.MAX_SUBPROCESS_OUTPUT_BYTES + 1})",
+                ],
+                cwd=SCRIPT.parent.parent,
+                environment=environment,
+                timeout=10,
+            )
+        self.assertEqual(str(raised.exception), gate.FAILURE_RECEIPT)
+        self.assertTrue(
+            any(call.args[1] == gate.signal.SIGKILL for call in killpg.call_args_list)
+        )
+
+    def test_subprocess_group_is_reaped_on_external_interrupt(self):
+        environment = {
+            "LANG": "C",
+            "LC_ALL": "C",
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+        }
+        with (
+            mock.patch.object(
+                gate.selectors.DefaultSelector,
+                "select",
+                side_effect=KeyboardInterrupt,
+            ),
+            mock.patch.object(
+                gate.os, "killpg", wraps=gate.os.killpg
+            ) as killpg,
+            self.assertRaises(KeyboardInterrupt),
+        ):
+            gate.run_fixed(
+                [
+                    sys.executable,
+                    "-I",
+                    "-S",
+                    "-B",
+                    "-c",
+                    "import time; time.sleep(30)",
+                ],
+                cwd=SCRIPT.parent.parent,
+                environment=environment,
+                timeout=10,
+            )
+        self.assertTrue(
+            any(call.args[1] == gate.signal.SIGKILL for call in killpg.call_args_list)
+        )
+
+    def test_child_environment_is_an_allowlist(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            workspace = root / "workspace"
+            cache = root / "cache"
+            workspace.mkdir(mode=0o700)
+            cache.mkdir(mode=0o700)
+            environment = gate.fixed_child_environment(
+                workspace, cache, Path("/fixture/python")
+            )
+        self.assertEqual(
+            set(environment),
+            {
+                "LANG",
+                "LC_ALL",
+                "PATH",
+                "PYTHONDONTWRITEBYTECODE",
+                "PYTHONHASHSEED",
+                "TZ",
+                "UV_CACHE_DIR",
+                "UV_NO_CACHE",
+                "UV_NO_CONFIG",
+                "UV_NO_BUILD",
+                "UV_NO_PROGRESS",
+                "UV_OFFLINE",
+                "UV_PYTHON",
+                "UV_PYTHON_DOWNLOADS",
+                "XDG_CACHE_HOME",
+                "XDG_CONFIG_HOME",
+                "XDG_DATA_HOME",
+                "XDG_STATE_HOME",
+            },
+        )
+        joined = json.dumps(environment, sort_keys=True)
+        self.assertNotIn("MOFA", joined)
+        self.assertNotIn("DATABASE", joined)
+        self.assertNotIn("SECRET", joined)
+        self.assertNotIn("HOME", environment)
+
+    def test_clean_install_uses_only_the_explicit_offline_inputs(self):
+        calls = []
+        events = []
+        workspaces = []
+        temporary_root = gate.system_temporary_root()
+        python_temporary = tempfile.TemporaryDirectory(dir=temporary_root)
+        uv_temporary = tempfile.TemporaryDirectory(dir=temporary_root)
+        cache_temporary = tempfile.TemporaryDirectory(dir=temporary_root)
+        output_temporary = tempfile.TemporaryDirectory(dir=temporary_root)
+        self.addCleanup(python_temporary.cleanup)
+        self.addCleanup(uv_temporary.cleanup)
+        self.addCleanup(cache_temporary.cleanup)
+        self.addCleanup(output_temporary.cleanup)
+        python_fixture = Path(python_temporary.name) / "python"
+        uv_fixture = Path(uv_temporary.name) / "uv"
+        cache_fixture = Path(cache_temporary.name)
+        environment_fixture = Path(output_temporary.name) / ".venv"
+        python_fixture.write_bytes(b"fixture")
+        uv_fixture.write_bytes(b"fixture")
+        python_fixture.chmod(0o700)
+        uv_fixture.chmod(0o700)
+
+        def fixed_run(arguments, *, cwd, environment, timeout=120):
+            events.append(("command", list(arguments)))
+            calls.append((list(arguments), Path(cwd), dict(environment), timeout))
+            workspaces.append(Path(environment["XDG_CONFIG_HOME"]).parent)
+            if arguments[1:] == ["-I", "-S", "-B", "--version"] and arguments[0] == str(python_fixture):
+                return b"Python 3.14.7\n"
+            if arguments[1:] == ["--version"] and arguments[0] == str(uv_fixture):
+                return (
+                    f"uv 0.12.6 ({gate.EXPECTED_UV_COMMIT[:9]} "
+                    "2026-08-25 aarch64-apple-darwin)\n"
+                ).encode("ascii")
+            if len(arguments) > 1 and arguments[0:2] == [str(uv_fixture), "lock"]:
+                return b""
+            if len(arguments) > 1 and arguments[0:2] == [str(uv_fixture), "venv"]:
+                environment_python = (
+                    Path(environment["UV_PROJECT_ENVIRONMENT"]) / "bin" / "python"
+                )
+                environment_python.parent.mkdir(mode=0o700, parents=True)
+                environment_python.symlink_to(python_fixture)
+                (environment_python.parent / "python3").symlink_to("python")
+                (environment_python.parent / "python3.14").symlink_to("python")
+                environment_lock = environment_python.parents[1] / ".lock"
+                environment_lock.write_bytes(b"")
+                environment_lock.chmod(0o666)
+                return b""
+            if len(arguments) > 2 and arguments[0:3] == [
+                str(uv_fixture),
+                "pip",
+                "install",
+            ]:
+                return b""
+            if gate.INSPECTION_PROGRAM in arguments:
+                expected = dict(gate.EXPECTED_LOCK_PACKAGES)
+                expected.pop("audience-foundry-travel-readiness")
+                if gate.sys.platform != "win32":
+                    expected.pop("tzdata")
+                return json.dumps(
+                    {
+                        "installed": expected,
+                        "native_machine": gate.platform.machine(),
+                        "native_count": 12,
+                        "native_role_count": 5,
+                        "site_files_verified": True,
+                        "wheel_count": len(expected),
+                    },
+                    sort_keys=True,
+                ).encode("utf-8")
+            if gate.IMPORT_PROGRAM in arguments:
+                return json.dumps(
+                    {
+                        "imports_verified": True,
+                        "native_machine": gate.platform.machine(),
+                    },
+                    sort_keys=True,
+                ).encode("utf-8")
+            raise AssertionError("unexpected child command shape")
+
+        snapshot = gate.capture_policy_snapshot(SCRIPT.parent.parent)
+        def preinspect(environment_root, attestations):
+            events.append(("preinspection", Path(environment_root)))
+            self.assertEqual(attestations, wheel_attestations)
+
+        def normalize(environment_root, attestations):
+            events.append(("normalization", Path(environment_root)))
+            self.assertEqual(attestations, wheel_attestations)
+
+        wheel_attestations = {
+            name: {
+                "fixture.py": {"digest": "a" * 64, "size": 1},
+            }
+            for name in gate.current_platform_policy()["wheel_tags"]
+        }
+        downloaded_wheels = {}
+        for name, filename in gate.current_platform_policy()[
+            "lock_wheel_filenames"
+        ].items():
+            body = f"{name}-official-wheel-fixture".encode("ascii")
+            downloaded_wheels[name] = {
+                "body": body,
+                "digest": hashlib.sha256(body).hexdigest(),
+                "filename": filename,
+                "records": wheel_attestations.get(
+                    name,
+                    {
+                        "tzdata-fixture.py": {
+                            "digest": "b" * 64,
+                            "size": 1,
+                        }
+                    },
+                ),
+                "size": len(body),
+            }
+
+        with (
+            mock.patch.object(gate, "run_fixed", side_effect=fixed_run),
+            mock.patch.object(
+                gate, "verify_preinspection_site", side_effect=preinspect
+            ) as preinspection,
+            mock.patch.object(
+                gate, "normalize_uv_install_metadata", side_effect=normalize
+            ) as normalization,
+            mock.patch.object(
+                gate,
+                "read_regular_file",
+                side_effect=AssertionError("source must not be reread during build"),
+            ),
+        ):
+            environment_digest, environment_identity = (
+                gate.build_and_inspect_environment(
+                    SCRIPT.parent.parent,
+                    snapshot,
+                    downloaded_wheels,
+                    python_fixture,
+                    uv_fixture,
+                    cache_fixture,
+                    environment_fixture,
+                )
+            )
+        preinspection.assert_called_once()
+        normalization.assert_called_once()
+        self.assertRegex(environment_digest, r"\A[0-9a-f]{64}\Z")
+        self.assertEqual(
+            environment_identity,
+            (
+                environment_fixture.stat().st_dev,
+                environment_fixture.stat().st_ino,
+            ),
+        )
+        self.assertTrue(environment_fixture.is_dir())
+        self.assertEqual(
+            stat.S_IMODE((environment_fixture / ".lock").stat().st_mode), 0o600
+        )
+
+        self.assertGreaterEqual(len(calls), 7)
+        lock_check = next(call for call in calls if call[0][1:2] == ["lock"])
+        self.assertIn("--check", lock_check[0])
+        self.assertIn("--offline", lock_check[0])
+        self.assertIn("--no-cache", lock_check[0])
+        self.assertIn("--no-python-downloads", lock_check[0])
+        self.assertNotIn("--find-links", lock_check[0])
+        venv = next(call for call in calls if call[0][1:2] == ["venv"])
+        self.assertIn("--allow-existing", venv[0])
+        self.assertIn("--no-project", venv[0])
+        self.assertIn("--offline", venv[0])
+        self.assertIn("--no-cache", venv[0])
+        self.assertEqual(venv[0][venv[0].index("--link-mode") + 1], "copy")
+        install = next(
+            call for call in calls if call[0][1:3] == ["pip", "install"]
+        )
+        self.assertIn("--requirements", install[0])
+        self.assertIn("--offline", install[0])
+        self.assertIn("--no-cache", install[0])
+        self.assertIn("--no-index", install[0])
+        self.assertIn("--find-links", install[0])
+        self.assertIn("--only-binary", install[0])
+        self.assertEqual(
+            install[0][install[0].index("--only-binary") + 1], ":all:"
+        )
+        self.assertEqual(install[2]["UV_NO_BUILD"], "1")
+        self.assertIn("--no-deps", install[0])
+        self.assertIn("--require-hashes", install[0])
+        self.assertIn("--exact", install[0])
+        self.assertIn("--strict", install[0])
+        self.assertEqual(
+            install[0][install[0].index("--link-mode") + 1], "copy"
+        )
+        self.assertIn(str(cache_fixture), install[0])
+        self.assertIn(str(environment_fixture / "bin/python"), install[0])
+        self.assertNotIn(str(SCRIPT.parent.parent), " ".join(install[0]))
+        self.assertFalse(any(call[0][1:2] == ["sync"] for call in calls))
+        self.assertTrue(cache_fixture.is_dir())
+        self.assertEqual(list(cache_fixture.iterdir()), [])
+        self.assertEqual(list(environment_fixture.rglob("*.whl")), [])
+        for installed_file in (
+            path for path in environment_fixture.rglob("*") if path.is_file()
+        ):
+            self.assertEqual(installed_file.stat().st_nlink, 1)
+        inspections = [
+            call for call in calls if gate.INSPECTION_PROGRAM in call[0]
+        ]
+        imports = [call for call in calls if gate.IMPORT_PROGRAM in call[0]]
+        self.assertEqual(len(inspections), 2)
+        self.assertEqual(len(imports), 1)
+        self.assertFalse(
+            any(call[0][1:3] == ["pip", "check"] for call in calls)
+        )
+        preinspection_index = next(
+            index for index, event in enumerate(events) if event[0] == "preinspection"
+        )
+        normalization_index = next(
+            index
+            for index, event in enumerate(events)
+            if event[0] == "normalization"
+        )
+        first_python_inspection_index = next(
+            index
+            for index, event in enumerate(events)
+            if event[0] == "command"
+            and (
+                gate.INSPECTION_PROGRAM in event[1]
+                or gate.IMPORT_PROGRAM in event[1]
+            )
+        )
+        self.assertLess(normalization_index, preinspection_index)
+        self.assertLess(preinspection_index, first_python_inspection_index)
+        for call in inspections + imports:
+            self.assertEqual(call[0][0], str(python_fixture))
+            self.assertIn("-I", call[0])
+            self.assertIn("-S", call[0])
+            self.assertIn("-B", call[0])
+            self.assertNotEqual(call[0][0], str(Path(call[0][-1]) / "bin/python"))
+        self.assertTrue(workspaces)
+        self.assertTrue(all(not workspace.exists() for workspace in workspaces))
+        fixed_temporary = Path("/private/tmp" if gate.sys.platform == "darwin" else "/tmp")
+        self.assertTrue(all(workspace.parent == fixed_temporary for workspace in workspaces))
+        for _, _, environment, _ in calls:
+            self.assertNotIn("MOFA_TRAVEL_ALARM_SERVICE_KEY", environment)
+            self.assertNotIn("DATABASE_URL", environment)
+            self.assertNotIn("HTTP_PROXY", environment)
+
+    def test_inspection_program_is_static_and_uses_pinned_runner(self):
+        self.assertEqual(tuple(sys.version_info[:3]), (3, 14, 7))
+        self.assertEqual(gate.platform.machine(), "arm64")
+        compile(gate.INSPECTION_PROGRAM, "<static-inspection>", "exec")
+        compile(gate.IMPORT_PROGRAM, "<import-check>", "exec")
+        self.assertIn(
+            "importlib.metadata.distributions(path=[str(site_packages)])",
+            gate.INSPECTION_PROGRAM,
+        )
+        self.assertNotIn("import django", gate.INSPECTION_PROGRAM)
+        self.assertNotIn("sys.path.insert", gate.INSPECTION_PROGRAM)
+        self.assertIn("sys.flags.no_site != 1", gate.IMPORT_PROGRAM)
+        self.assertIn("sys.flags.dont_write_bytecode != 1", gate.IMPORT_PROGRAM)
+        self.assertIn("sys.path.insert(0, str(site_packages))", gate.IMPORT_PROGRAM)
+
+    def test_preinspection_rejects_pth_before_it_can_create_a_marker(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            environment_root = Path(temporary) / "environment"
+            site_packages = environment_root / "lib/python3.14/site-packages"
+            site_packages.mkdir(mode=0o700, parents=True)
+            installed = {
+                "fixture/__init__.py": b"VALUE = 1\n",
+                "fixture-1.0.dist-info/WHEEL": (
+                    b"Wheel-Version: 1.0\nRoot-Is-Purelib: true\nTag: py3-none-any\n"
+                ),
+            }
+            attestation = {
+                relative: {
+                    "digest": hashlib.sha256(content).hexdigest(),
+                    "size": len(content),
+                }
+                for relative, content in installed.items()
+            }
+            for relative, content in installed.items():
+                target = site_packages / relative
+                target.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
+                target.write_bytes(content)
+            dist_info = site_packages / "fixture-1.0.dist-info"
+            (dist_info / "INSTALLER").write_bytes(b"uv")
+            (dist_info / "REQUESTED").write_bytes(b"")
+            (dist_info / "RECORD").write_bytes(b"fixture-only")
+            seed_contents = {
+                "_virtualenv.py": b"SYNTHETIC_SEED = True\n",
+                "_virtualenv.pth": b"import _virtualenv\n",
+            }
+            seed_policy = {
+                relative: {
+                    "size": len(content),
+                    "digest": hashlib.sha256(content).hexdigest(),
+                }
+                for relative, content in seed_contents.items()
+            }
+            for relative, record in seed_policy.items():
+                content = seed_contents[relative]
+                (site_packages / relative).write_bytes(content)
+
+            wheel_attestations = {"fixture": attestation}
+            with mock.patch.object(
+                gate, "EXPECTED_UV_SEED_FILES", seed_policy
+            ):
+                gate.verify_preinspection_site(
+                    environment_root, wheel_attestations
+                )
+
+                marker = Path(temporary) / "must-not-exist"
+                malicious = site_packages / "malicious.pth"
+                malicious.write_text(
+                    f"import pathlib; pathlib.Path({str(marker)!r}).write_text('ran')\n",
+                    encoding="utf-8",
+                )
+                with self.assertRaises(gate.GateError):
+                    gate.verify_preinspection_site(
+                        environment_root, wheel_attestations
+                    )
+                self.assertFalse(marker.exists())
+                malicious.unlink()
+                (site_packages / "fixture/__init__.py").write_bytes(
+                    b"VALUE = 2\n"
+                )
+                with self.assertRaises(gate.GateError):
+                    gate.verify_preinspection_site(
+                        environment_root, wheel_attestations
+                    )
+
+    def test_uv_cache_metadata_schema_is_exact_before_normalization(self):
+        valid = {
+            "commit": None,
+            "directories": {},
+            "env": {},
+            "tags": None,
+            "timestamp": {
+                "nanos_since_epoch": 123,
+                "secs_since_epoch": 456,
+            },
+        }
+        gate.parse_uv_cache_metadata(
+            json.dumps(valid, separators=(",", ":")).encode("utf-8")
+        )
+        mutations = []
+        extra = dict(valid)
+        extra["unexpected"] = True
+        mutations.append(extra)
+        bad_env = dict(valid)
+        bad_env["env"] = {"ambient": "value"}
+        mutations.append(bad_env)
+        bad_timestamp = dict(valid)
+        bad_timestamp["timestamp"] = {
+            "nanos_since_epoch": 1_000_000_000,
+            "secs_since_epoch": 456,
+        }
+        mutations.append(bad_timestamp)
+        for mutation in mutations:
+            with self.subTest(mutation=mutation), self.assertRaises(
+                gate.GateError
+            ):
+                gate.parse_uv_cache_metadata(
+                    json.dumps(mutation, separators=(",", ":")).encode(
+                        "utf-8"
+                    )
+                )
+        duplicate = (
+            b'{"commit":null,"commit":null,"directories":{},"env":{},'
+            b'"tags":null,"timestamp":{"nanos_since_epoch":0,'
+            b'"secs_since_epoch":0}}'
+        )
+        with self.assertRaises(gate.GateError):
+            gate.parse_uv_cache_metadata(duplicate)
+
+    def test_environment_inventory_is_canonical_and_content_bound(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            environment_root = Path(temporary) / ".venv"
+            directory = environment_root / "package"
+            directory.mkdir(mode=0o700, parents=True)
+            payload = directory / "module.py"
+            payload.write_bytes(b"VALUE = 1\n")
+            link = environment_root / "python"
+            link.symlink_to("package/module.py")
+            first_inventory = gate.dependency_environment_inventory_bytes(
+                environment_root, python=payload
+            )
+            first_digest = gate.dependency_environment_sha256(
+                environment_root, python=payload
+            )
+            document = json.loads(first_inventory.decode("utf-8"))
+            self.assertEqual(
+                [entry["path"] for entry in document],
+                [".", "package", "package/module.py", "python"],
+            )
+            self.assertTrue(first_inventory.endswith(b"\n"))
+            self.assertRegex(first_digest, r"\A[0-9a-f]{64}\Z")
+            self.assertEqual(
+                first_digest,
+                "474c212e3e4a775a489922fd7f5a891bd88a2c21900a6188fea2d14b39566812",
+            )
+            self.assertEqual(
+                set(document[0]), {"mode", "path", "sha256", "size", "type"}
+            )
+            payload.write_bytes(b"VALUE = 2\n")
+            self.assertNotEqual(
+                gate.dependency_environment_sha256(
+                    environment_root, python=payload
+                ),
+                first_digest,
+            )
+
+    def test_independent_clean_environment_inventories_are_equal(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            digests = []
+            for name in ("first", "second"):
+                environment = Path(temporary) / name / ".venv"
+                package = environment / "package"
+                package.mkdir(mode=0o700, parents=True)
+                environment.chmod(0o700)
+                package.chmod(0o700)
+                (package / "module.py").write_bytes(b"VALUE = 1\n")
+                (environment / "python").symlink_to("package/module.py")
+                digests.append(
+                    gate.dependency_environment_sha256(
+                        environment, python=package / "module.py"
+                    )
+                )
+            self.assertEqual(digests[0], digests[1])
+
+    @unittest.skipUnless(
+        os.environ.get("DEPENDENCY_GATE_LIVE_REPRODUCIBILITY") == "1",
+        "explicit anonymous live dependency reproducibility test",
+    )
+    def test_actual_uv_environments_are_path_independent(self):
+        uv_value = os.environ.get("DEPENDENCY_GATE_LIVE_UV_BIN")
+        self.assertIsNotNone(uv_value)
+        uv = Path(str(uv_value))
+        source_archive = Path(sys.executable).parents[2] / "cpython.tar.gz"
+        self.assertTrue(source_archive.is_file())
+        self.assertTrue(uv.is_file())
+        temporary_roots = [
+            tempfile.TemporaryDirectory(dir=gate.system_temporary_root())
+            for _index in range(8)
+        ]
+        for temporary in temporary_roots:
+            self.addCleanup(temporary.cleanup)
+        digests = []
+        bootstrap_digests = []
+        for index in range(2):
+            python_root = Path(temporary_roots[index * 4].name)
+            cache = Path(temporary_roots[index * 4 + 1].name)
+            output = Path(temporary_roots[index * 4 + 2].name)
+            staging = Path(temporary_roots[index * 4 + 3].name)
+            archive = python_root / "cpython.tar.gz"
+            shutil.copyfile(source_archive, archive)
+            archive.chmod(0o600)
+            extracted = subprocess.run(
+                [
+                    "/usr/bin/tar",
+                    "-xzf",
+                    str(archive),
+                    "-C",
+                    str(python_root),
+                ],
+                cwd=staging,
+                env={"LANG": "C", "LC_ALL": "C", "PATH": "/usr/bin:/bin"},
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                timeout=30,
+                check=False,
+            )
+            self.assertEqual(extracted.returncode, 0)
+            self.assertEqual(extracted.stdout, b"")
+            self.assertEqual(extracted.stderr, b"")
+            python = python_root / "python/bin/python3.14"
+            environment = output / ".venv"
+            result = subprocess.run(
+                [
+                    str(python),
+                    "-I",
+                    "-S",
+                    "-B",
+                    str(SCRIPT),
+                    "--source-dir",
+                    str(SCRIPT.parent.parent),
+                    "--python-bin",
+                    str(python),
+                    "--uv-bin",
+                    str(uv),
+                    "--cache-dir",
+                    str(cache),
+                    "--ca-file",
+                    "/private/etc/ssl/cert.pem",
+                    "--environment-dir",
+                    str(environment),
+                ],
+                cwd=SCRIPT.parent.parent,
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                    "PYTHONHASHSEED": "0",
+                    "TZ": "UTC",
+                },
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                timeout=180,
+                check=False,
+            )
+            self.assertEqual(result.returncode, 0)
+            self.assertEqual(result.stderr, b"")
+            try:
+                receipt = result.stdout.decode("ascii", "strict").strip()
+            except UnicodeDecodeError as error:
+                raise AssertionError("dependency receipt was not ASCII") from error
+            match = gate.re.fullmatch(
+                gate.re.escape(gate.SUCCESS_RECEIPT)
+                + r" dependency_environment_sha256=([0-9a-f]{64})"
+                + r" bootstrap_python_tree_sha256=([0-9a-f]{64})",
+                receipt,
+            )
+            self.assertIsNotNone(match)
+            digests.append(match.group(1))
+            bootstrap_digests.append(match.group(2))
+            self.assertEqual(tuple(cache.iterdir()), ())
+        self.assertEqual(len(set(digests)), 1)
+        self.assertEqual(len(set(bootstrap_digests)), 1)
+        self.assertEqual(
+            digests[0],
+            gate.current_platform_policy()["dependency_environment_sha256"],
+        )
+        self.assertEqual(
+            bootstrap_digests[0],
+            gate.current_platform_policy()["python_tree_sha256"],
+        )
+
+    def test_unreviewed_embedded_environment_paths_fail_closed(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            environment = Path(temporary) / ".venv"
+            environment.mkdir(mode=0o700)
+            python = Path(temporary) / "bootstrap/bin/python3.14"
+            target = environment / "unreviewed.txt"
+            target.write_bytes(os.fsencode(str(environment)) + b"\n")
+            with self.assertRaises(gate.GateError):
+                gate.dependency_environment_sha256(
+                    environment, python=python
+                )
+            target.write_bytes(os.fsencode(str(python.parent)) + b"\n")
+            with self.assertRaises(gate.GateError):
+                gate.dependency_environment_sha256(
+                    environment, python=python
+                )
+            target.unlink()
+            link = environment / "unreviewed-link"
+            link.symlink_to(str(python.parent))
+            with self.assertRaises(gate.GateError):
+                gate.dependency_environment_sha256(
+                    environment, python=python
+                )
+
+    def test_environment_inventory_rejects_hardlinked_files(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            environment = Path(temporary) / ".venv"
+            environment.mkdir(mode=0o700)
+            first = environment / "first"
+            second = environment / "second"
+            first.write_bytes(b"shared")
+            os.link(first, second)
+            self.assertEqual(first.stat().st_nlink, 2)
+            with self.assertRaises(gate.GateError):
+                gate.dependency_environment_sha256(
+                    environment, python=first
+                )
+
+    def test_environment_symlink_topology_is_exact(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary)
+            environment = root / ".venv"
+            binary = environment / "bin"
+            binary.mkdir(mode=0o700, parents=True)
+            python = root / "python3.14"
+            python.write_bytes(b"fixture")
+            python.chmod(0o700)
+            (binary / "python").symlink_to(python)
+            (binary / "python3").symlink_to("python")
+            (binary / "python3.14").symlink_to("python")
+            gate.verify_environment_symlink_topology(environment, python)
+            unexpected = environment / "unexpected"
+            unexpected.symlink_to("bin/python")
+            with self.assertRaises(gate.GateError):
+                gate.verify_environment_symlink_topology(environment, python)
+
+    def test_environment_output_must_be_new_private_and_named_venv(self):
+        source = SCRIPT.parent.parent
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            parent = Path(temporary)
+            candidate = parent / ".venv"
+            self.assertEqual(
+                gate.safe_new_environment_path(str(candidate), source=source),
+                candidate,
+            )
+            with self.assertRaises(gate.GateError):
+                gate.safe_new_environment_path(
+                    str(parent / "different-name"), source=source
+                )
+            candidate.mkdir()
+            with self.assertRaises(gate.GateError):
+                gate.safe_new_environment_path(str(candidate), source=source)
+
+    def test_environment_claim_is_exclusive_and_refuses_replacement_cleanup(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary)
+            environment = root / ".venv"
+            identity = gate.claim_environment_directory(environment)
+            with self.assertRaises(gate.GateError):
+                gate.claim_environment_directory(environment)
+            owned = root / "owned-by-first-invocation"
+            environment.rename(owned)
+            environment.mkdir(mode=0o700)
+            (environment / "second-invocation").write_bytes(b"preserve")
+            with self.assertRaises(gate.GateError):
+                gate.remove_owned_environment(environment, identity)
+            self.assertTrue(environment.is_dir())
+            self.assertTrue((environment / "second-invocation").is_file())
+            self.assertTrue(owned.is_dir())
+
+    def test_final_policy_failure_removes_the_owned_environment(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary)
+            environment = root / ".venv"
+            parsed = {
+                "--source-dir": "/fixture/source",
+                "--python-bin": "/fixture/python",
+                "--uv-bin": "/fixture/uv",
+                "--cache-dir": "/fixture/cache",
+                "--ca-file": "/fixture/ca",
+                "--environment-dir": str(environment),
+            }
+            resolved = {
+                "/fixture/source": Path("/fixture/source"),
+                "/fixture/python": Path("/fixture/python"),
+                "/fixture/uv": Path("/fixture/uv"),
+                "/fixture/cache": Path("/fixture/cache"),
+                "/fixture/ca": Path("/fixture/ca"),
+            }
+
+            def create_environment(*_arguments):
+                environment.mkdir(mode=0o700)
+                (environment / "partial").write_bytes(b"fixture")
+                status = environment.stat()
+                return "a" * 64, (status.st_dev, status.st_ino)
+
+            with (
+                mock.patch.object(gate, "parse_arguments", return_value=parsed),
+                mock.patch.object(gate, "enforce_gate_runner"),
+                mock.patch.object(
+                    gate,
+                    "safe_absolute_path",
+                    side_effect=lambda value, **_kwargs: resolved[value],
+                ),
+                mock.patch.object(
+                    gate, "safe_new_environment_path", return_value=environment
+                ),
+                mock.patch.object(gate, "verify_input_boundaries"),
+                mock.patch.object(gate, "capture_policy_snapshot", return_value={}),
+                mock.patch.object(gate, "parse_runtime"),
+                mock.patch.object(gate, "parse_project"),
+                mock.patch.object(gate, "parse_lock_contract", return_value=({}, {})),
+                mock.patch.object(gate, "verify_notices"),
+                mock.patch.object(gate, "verify_tool_provenance"),
+                mock.patch.object(gate, "verify_ca_provenance"),
+                mock.patch.object(gate, "check_advisories"),
+                mock.patch.object(
+                    gate, "fetch_selected_wheels", return_value={}
+                ),
+                mock.patch.object(
+                    gate,
+                    "build_and_inspect_environment",
+                    side_effect=create_environment,
+                ),
+                mock.patch.object(
+                    gate,
+                    "verify_policy_snapshot_unchanged",
+                    side_effect=gate.GateError(gate.FAILURE_RECEIPT),
+                ),
+                self.assertRaises(gate.GateError),
+            ):
+                gate.run_gate([])
+            self.assertFalse(environment.exists())
+
+    def test_script_never_reads_environment_or_environment_files(self):
+        source = SCRIPT.read_text(encoding="utf-8")
+        forbidden_files = {".env", ".env.local"}
+        for label, program in (
+            ("gate", source),
+            ("inspection", gate.INSPECTION_PROGRAM),
+            ("import", gate.IMPORT_PROGRAM),
+        ):
+            tree = ast.parse(program, filename=label)
+            for node in ast.walk(tree):
+                if isinstance(node, ast.Attribute):
+                    self.assertFalse(
+                        isinstance(node.value, ast.Name)
+                        and node.value.id == "os"
+                        and node.attr == "environ"
+                    )
+                if isinstance(node, ast.Call) and isinstance(
+                    node.func, ast.Attribute
+                ):
+                    self.assertFalse(
+                        isinstance(node.func.value, ast.Name)
+                        and node.func.value.id == "os"
+                        and node.func.attr == "getenv"
+                    )
+                if isinstance(node, ast.Constant) and isinstance(
+                    node.value, str
+                ):
+                    self.assertNotIn(node.value, forbidden_files)
+            lowered = program.lower()
+            self.assertNotIn("mofa_travel_alarm_service_key", lowered)
+            self.assertNotIn("database_url", lowered)
+
+    def test_receipts_are_fixed_and_detail_free(self):
+        self.assertEqual(gate.FAILURE_RECEIPT, "dependency_gate=failed")
+        self.assertEqual(
+            gate.SUCCESS_RECEIPT,
+            "dependency_gate=ok lock=offline-current environment=isolated "
+            "installed=official-wheel-bytes licenses=installed-files "
+            "native=current-platform-import advisories=release-json-clear "
+            "production_linux_native=checkpoint",
+        )
+        combined = gate.SUCCESS_RECEIPT + gate.FAILURE_RECEIPT
+        for forbidden in ("http", "/", "django", "psycopg", "secret", "key"):
+            self.assertNotIn(forbidden, combined.lower())
+
+    def test_main_emits_only_fixed_receipt_and_environment_digest(self):
+        digest = "a" * 64
+        bootstrap_digest = "b" * 64
+        with (
+            mock.patch.object(
+                gate, "run_gate", return_value=(digest, bootstrap_digest)
+            ),
+            mock.patch.object(gate.sys, "argv", ["check-dependencies"]),
+            mock.patch("builtins.print") as printed,
+        ):
+            self.assertEqual(gate.main(), 0)
+        printed.assert_called_once_with(
+            f"{gate.SUCCESS_RECEIPT} dependency_environment_sha256={digest} "
+            f"bootstrap_python_tree_sha256={bootstrap_digest}"
+        )
+
+    def test_main_redacts_unexpected_failure_details(self):
+        with (
+            mock.patch.object(
+                gate,
+                "run_gate",
+                side_effect=RuntimeError("synthetic path, URL, and secret detail"),
+            ),
+            mock.patch.object(gate.sys, "argv", ["check-dependencies"]),
+            mock.patch("builtins.print") as printed,
+        ):
+            self.assertEqual(gate.main(), 1)
+        printed.assert_called_once_with(gate.FAILURE_RECEIPT, file=gate.sys.stderr)
+
+    def test_script_is_executable_and_uses_no_third_party_imports(self):
+        self.assertEqual(stat.S_IMODE(SCRIPT.stat().st_mode), 0o755)
+        source = SCRIPT.read_text(encoding="utf-8")
+        self.assertEqual(source.splitlines()[0], "#!/usr/bin/false")
+        tree = ast.parse(source)
+        allowed_roots = {
+            "base64",
+            "csv",
+            "email",
+            "hashlib",
+            "io",
+            "json",
+            "os",
+            "pathlib",
+            "platform",
+            "re",
+            "selectors",
+            "shutil",
+            "ssl",
+            "stat",
+            "subprocess",
+            "sys",
+            "tempfile",
+            "time",
+            "tomllib",
+            "typing",
+            "urllib",
+            "zipfile",
+            "signal",
+        }
+        for node in ast.walk(tree):
+            if isinstance(node, ast.Import):
+                for alias in node.names:
+                    self.assertIn(alias.name.split(".", 1)[0], allowed_roots)
+            elif isinstance(node, ast.ImportFrom):
+                self.assertIn((node.module or "").split(".", 1)[0], allowed_roots | {"__future__"})
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/runtime/versions.toml b/runtime/versions.toml
index e2bd57e..6350928 100644
--- a/runtime/versions.toml
+++ b/runtime/versions.toml
@@ -8,6 +8,14 @@ psycopg_distribution = "binary-wheel"
 gunicorn = "26.2.0"
 
 [integrity]
+python_provider = "astral-sh/python-build-standalone"
+python_release_tag = "20260825"
+python_release_commit = "c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779"
+python_macos_arm64_asset = "cpython-3.14.7+20260825-aarch64-apple-darwin-install_only_stripped.tar.gz"
+python_macos_arm64_archive_url = "https://github.com/astral-sh/python-build-standalone/releases/download/20260825/cpython-3.14.7%2B20260825-aarch64-apple-darwin-install_only_stripped.tar.gz"
+python_macos_arm64_archive_size = 26488085
+python_macos_arm64_archive_sha256 = "17ecb3d29c49765856370cbb47d948a24ec518be40f607362c1bbb3ebbc5c442"
+python_verification = "official-github-immutable-release-api-digest-local-sha256-and-signed-release-commit"
 uv_release_commit = "7938ca5d53dbb9c614a4a030df406e41ff101ab9"
 uv_macos_arm64_archive_sha256 = "14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d"
 uv_attestation = "verified-with-github-cli-and-public-sigstore-rekor"
diff --git a/scripts/check-dependencies b/scripts/check-dependencies
new file mode 100755
index 0000000..0612876
--- /dev/null
+++ b/scripts/check-dependencies
@@ -0,0 +1,3899 @@
+#!/usr/bin/false
+"""Fail-closed dependency, license, and advisory gate.
+
+The command deliberately accepts every filesystem input explicitly and gives
+child processes a newly constructed environment.  It never loads an
+application environment file and never emits package names, paths, URLs, raw
+metadata, or subprocess output.
+"""
+
+from __future__ import annotations
+
+import base64
+import email.message
+import csv
+import hashlib
+import io
+import json
+import os
+from pathlib import Path, PurePosixPath
+import platform
+import re
+import selectors
+import signal
+import shutil
+import ssl
+import stat
+import subprocess
+import sys
+import tempfile
+import time
+import tomllib
+from typing import Iterable
+import urllib.error
+import urllib.parse
+import urllib.request
+import zipfile
+
+
+EXPECTED_PYTHON = "3.14.7"
+EXPECTED_UV = "0.12.6"
+EXPECTED_UV_COMMIT = "7938ca5d53dbb9c614a4a030df406e41ff101ab9"
+EXPECTED_LOCK_REVISION = 3
+EXPECTED_LOCK_FORMAT = 1
+MAX_METADATA_BYTES = 1024 * 1024
+MAX_WHEEL_BYTES = 16 * 1024 * 1024
+MAX_WHEEL_CONTENT_BYTES = 64 * 1024 * 1024
+MAX_RESPONSE_HEADERS_BYTES = 16 * 1024
+MAX_SUBPROCESS_OUTPUT_BYTES = 1024 * 1024
+MAX_ENVIRONMENT_ENTRIES = 20000
+MAX_ENVIRONMENT_FILE_BYTES = 64 * 1024 * 1024
+MAX_ENVIRONMENT_TOTAL_BYTES = 256 * 1024 * 1024
+MAX_WHEELHOUSE_BYTES = 64 * 1024 * 1024
+MAX_PYTHON_TREE_ENTRIES = 5000
+MAX_PYTHON_TREE_BYTES = 256 * 1024 * 1024
+MIN_PYTHON_TREE_ENTRIES = 1000
+MIN_PYTHON_TREE_BYTES = 32 * 1024 * 1024
+NETWORK_TIMEOUT_SECONDS = 5
+NETWORK_TOTAL_SECONDS = 120
+PYPI_ORIGIN = "https://pypi.org"
+PYPI_REGISTRY = "https://pypi.org/simple"
+SUCCESS_RECEIPT = (
+    "dependency_gate=ok lock=offline-current environment=isolated "
+    "installed=official-wheel-bytes "
+    "licenses=installed-files native=current-platform-import "
+    "advisories=release-json-clear "
+    "production_linux_native=checkpoint"
+)
+FAILURE_RECEIPT = "dependency_gate=failed"
+
+EXPECTED_UV_ARCHIVE_SHA256 = (
+    "14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d"
+)
+EXPECTED_PYTHON_ARCHIVE_SHA256 = (
+    "17ecb3d29c49765856370cbb47d948a24ec518be40f607362c1bbb3ebbc5c442"
+)
+EXPECTED_PYTHON_ARCHIVE_SIZE = 26488085
+EXPECTED_PYTHON_RELEASE_COMMIT = (
+    "c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779"
+)
+EXPECTED_PYTHON_ASSET = (
+    "cpython-3.14.7+20260825-aarch64-apple-darwin-"
+    "install_only_stripped.tar.gz"
+)
+EXPECTED_PYTHON_ASSET_URL = (
+    "https://github.com/astral-sh/python-build-standalone/releases/download/"
+    "20260825/cpython-3.14.7%2B20260825-aarch64-apple-darwin-"
+    "install_only_stripped.tar.gz"
+)
+EXPECTED_PYTHON_BYTECODE = {
+    "lib/python3.14/encodings/__pycache__/__init__.cpython-314.pyc": (
+        "3d41c16ce94d79e6ae8095061375e9c7c0317bfcc3e1e8c599f0bf8483c1866c"
+    ),
+    "lib/python3.14/encodings/__pycache__/aliases.cpython-314.pyc": (
+        "6e9b9a5adf1635d59b62aa5bea54b4874c4461ad413550ebad933bb80d52b1d5"
+    ),
+    "lib/python3.14/encodings/__pycache__/utf_8.cpython-314.pyc": (
+        "a397a28be21861fb1ea37744063c2a4d3c507c1fcb87b76fad9ac17be188256e"
+    ),
+}
+EXPECTED_NOTICE_SHA256 = (
+    "b10cb31a42abed830c86d8b7f8c399852b5a24a4c3dcf90e955217bbcedf1290"
+)
+EXPECTED_UV_SEED_FILES = {
+    "_virtualenv.py": {
+        "digest": "cfb3db86aaa53bb62b5ff764970bec2d71c9228590a0ebec57f6ec926cc0bf1a",
+        "size": 5246,
+    },
+    "_virtualenv.pth": {
+        "digest": "69ac3d8f27e679c81b94ab30b3b56e9cd138219b1ba94a1fa3606d5a76a1433d",
+        "size": 18,
+    },
+}
+EXPECTED_GENERATED_SCRIPT_PATHS = {
+    "django": {"../../../bin/django-admin"},
+    "gunicorn": {"../../../bin/gunicorn", "../../../bin/gunicornc"},
+    "sqlparse": {"../../../bin/sqlformat"},
+}
+CANONICAL_VENV_ROOT = b"@VENV@"
+CANONICAL_BOOTSTRAP_PYTHON = b"@BOOTSTRAP_PYTHON@"
+EXPECTED_ACTIVATION_FILES = {
+    "bin/activate": "a34528c5c97451c3989f660cbe189186f84792daa0cc07f49b1d05cf399a6cb7",
+    "bin/activate.bat": "94c3f10fb540da2576739c5b810d2002f50e5623d2e9d29db61b012f63f5a9cc",
+    "bin/activate.csh": "e75ddfa21f5d2a77a0db4233739dc5f5de9420b606f85b00f265e53ce4a43ad0",
+    "bin/activate.fish": "2024437eaed7a805d6895c548cfcd32c9c6756aeae37fb3fffeabd6f9abb93fd",
+    "bin/activate.nu": "99a48d6797a0506c1cdcad4c5774b1df5809c8b4afe0ac7fa9cff5b5fd4c321a",
+}
+EXPECTED_CONSOLE_SCRIPTS = {
+    "bin/django-admin": {
+        "call": "execute_from_command_line()",
+        "import": "from django.core.management import execute_from_command_line",
+        "record": "lib/python3.14/site-packages/django-5.2.17.dist-info/RECORD",
+        "record_path": "../../../bin/django-admin",
+    },
+    "bin/gunicorn": {
+        "call": "run()",
+        "import": "from gunicorn.app.wsgiapp import run",
+        "record": "lib/python3.14/site-packages/gunicorn-26.2.0.dist-info/RECORD",
+        "record_path": "../../../bin/gunicorn",
+    },
+    "bin/gunicornc": {
+        "call": "main()",
+        "import": "from gunicorn.ctl.cli import main",
+        "record": "lib/python3.14/site-packages/gunicorn-26.2.0.dist-info/RECORD",
+        "record_path": "../../../bin/gunicornc",
+    },
+    "bin/sqlformat": {
+        "call": "main()",
+        "import": "from sqlparse.__main__ import main",
+        "record": "lib/python3.14/site-packages/sqlparse-0.6.0.dist-info/RECORD",
+        "record_path": "../../../bin/sqlformat",
+    },
+}
+EXPECTED_PLATFORM_POLICY = {
+    ("darwin", "arm64"): {
+        "python_executable_sha256": (
+            "1bfa9a829d950ecd4870a3d7a6826eb57edb4aa93f69d07cd3bb21e9fcc6d439"
+        ),
+        "python_tree_sha256": (
+            "28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110"
+        ),
+        "dependency_environment_sha256": (
+            "ade0e58f6e5e3cc08d4c2a2631bea93fae5c1d86b460971afd34e3de3119ee69"
+        ),
+        "python_relative": "python/bin/python3.14",
+        "python_archive_relative": "cpython.tar.gz",
+        "uv_executable_sha256": (
+            "e8929237934c8679686428f5a7736c7ae7a5fe7a33b0504d1b03446cdbc43c94"
+        ),
+        "uv_relative": "uv-aarch64-apple-darwin/uv",
+        "ca_sha256": (
+            "9dae8d76e55cb08991f2b672d58999ea15560d910759c16b544f843bdffbb994"
+        ),
+        "wheel_filenames": {
+            "asgiref": "asgiref-3.12.1-py3-none-any.whl",
+            "django": "django-5.2.17-py3-none-any.whl",
+            "gunicorn": "gunicorn-26.2.0-py3-none-any.whl",
+            "psycopg": "psycopg-3.3.4-py3-none-any.whl",
+            "psycopg-binary": (
+                "psycopg_binary-3.3.4-cp314-cp314-macosx_11_0_arm64.whl"
+            ),
+            "sqlparse": "sqlparse-0.6.0-py3-none-any.whl",
+        },
+        "lock_wheel_filenames": {
+            "asgiref": "asgiref-3.12.1-py3-none-any.whl",
+            "django": "django-5.2.17-py3-none-any.whl",
+            "gunicorn": "gunicorn-26.2.0-py3-none-any.whl",
+            "psycopg": "psycopg-3.3.4-py3-none-any.whl",
+            "psycopg-binary": (
+                "psycopg_binary-3.3.4-cp314-cp314-macosx_11_0_arm64.whl"
+            ),
+            "sqlparse": "sqlparse-0.6.0-py3-none-any.whl",
+            "tzdata": "tzdata-2026.3-py2.py3-none-any.whl",
+        },
+        "wheel_tags": {
+            "asgiref": "py3-none-any",
+            "django": "py3-none-any",
+            "gunicorn": "py3-none-any",
+            "psycopg": "py3-none-any",
+            "psycopg-binary": "cp314-cp314-macosx_11_0_arm64",
+            "sqlparse": "py3-none-any",
+        },
+    },
+}
+
+EXPECTED_RUNTIME = {
+    "python": EXPECTED_PYTHON,
+    "django": "5.2.17",
+    "postgresql": "18.6",
+    "uv": EXPECTED_UV,
+    "psycopg": "3.3.4",
+    "psycopg_distribution": "binary-wheel",
+    "gunicorn": "26.2.0",
+}
+
+EXPECTED_LOCK_PACKAGES = {
+    "asgiref": "3.12.1",
+    "audience-foundry-travel-readiness": "0.1.0",
+    "django": "5.2.17",
+    "gunicorn": "26.2.0",
+    "psycopg": "3.3.4",
+    "psycopg-binary": "3.3.4",
+    "sqlparse": "0.6.0",
+    "tzdata": "2026.3",
+}
+
+EXPECTED_LICENSES = {
+    "asgiref": "BSD-3-Clause",
+    "django": "BSD-3-Clause",
+    "gunicorn": "MIT",
+    "psycopg": "LGPL-3.0-only",
+    "psycopg-binary": "LGPL-3.0-only",
+    "sqlparse": "BSD-3-Clause",
+    "tzdata": "Apache-2.0",
+}
+
+EXPECTED_LICENSE_FILES = {
+    "asgiref": {"LICENSE"},
+    "django": {"AUTHORS", "LICENSE", "LICENSE.python"},
+    "gunicorn": {"LICENSE"},
+    "psycopg": {"LICENSE.txt"},
+    "psycopg-binary": {"LICENSE.txt"},
+    "sqlparse": {"AUTHORS", "LICENSE"},
+    "tzdata": {"LICENSE"},
+}
+
+EXPECTED_DIRECT_DEPENDENCIES = {
+    "Django==5.2.17",
+    "gunicorn==26.2.0",
+    "psycopg[binary]==3.3.4",
+}
+
+EXPECTED_LOCK_DEPENDENCIES = {
+    "asgiref": [],
+    "audience-foundry-travel-readiness": [
+        {"name": "django"},
+        {"name": "gunicorn"},
+        {"name": "psycopg", "extra": ["binary"]},
+    ],
+    "django": [
+        {"name": "asgiref"},
+        {"name": "sqlparse"},
+        {"name": "tzdata", "marker": "sys_platform == 'win32'"},
+    ],
+    "gunicorn": [],
+    "psycopg": [
+        {"name": "tzdata", "marker": "sys_platform == 'win32'"},
+    ],
+    "psycopg-binary": [],
+    "sqlparse": [],
+    "tzdata": [],
+}
+
+EXPECTED_LOCK_OPTIONAL_DEPENDENCIES = {
+    "psycopg": {
+        "binary": [
+            {"name": "psycopg-binary", "marker": "implementation_name != 'pypy'"},
+        ],
+    },
+}
+
+EXPECTED_LOCAL_METADATA = {
+    "requires-dist": [
+        {"name": "django", "specifier": "==5.2.17"},
+        {"name": "gunicorn", "specifier": "==26.2.0"},
+        {"name": "psycopg", "extras": ["binary"], "specifier": "==3.3.4"},
+    ],
+}
+
+NOTICE_REQUIREMENTS = (
+    "| CPython | 3.14.7 | PSF License Version 2 |",
+    "| python-build-standalone | tag `20260825`, commit `c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779` | MPL-2.0 |",
+    "| uv | 0.12.6, commit `7938ca5d53dbb9c614a4a030df406e41ff101ab9` | MIT OR Apache-2.0 |",
+    "| PostgreSQL | 18.6 | PostgreSQL License |",
+    "| Django 5.2.17 | direct | BSD-3-Clause |",
+    "| Gunicorn 26.2.0 | direct | MIT |",
+    "| psycopg 3.3.4 | direct | LGPL-3.0-only |",
+    "| psycopg-binary 3.3.4 | direct extra | LGPL-3.0-only |",
+    "| asgiref 3.12.1 | Django transitive | BSD-3-Clause |",
+    "| sqlparse 0.6.0 | Django transitive | BSD-3-Clause |",
+    "| tzdata 2026.3 | Windows-only transitive | Apache-2.0 |",
+    "production Linux target",
+)
+
+POLICY_FILE_LIMITS = {
+    "runtime/versions.toml": 64 * 1024,
+    "pyproject.toml": 64 * 1024,
+    "uv.lock": 8 * 1024 * 1024,
+    "THIRD_PARTY_NOTICES.md": 512 * 1024,
+}
+
+
+class GateError(RuntimeError):
+    """A deliberately detail-free dependency gate failure."""
+
+
+def fail() -> None:
+    raise GateError(FAILURE_RECEIPT)
+
+
+def canonical_name(value: str) -> str:
+    if not isinstance(value, str) or not value:
+        fail()
+    return re.sub(r"[-_.]+", "-", value).lower()
+
+
+def current_platform_policy() -> dict[str, object]:
+    policy = EXPECTED_PLATFORM_POLICY.get((sys.platform, platform.machine()))
+    if policy is None:
+        fail()
+    return policy
+
+
+def enforce_gate_runner_flags() -> None:
+    if (
+        not isinstance(sys.executable, str)
+        or not sys.executable
+        or not Path(sys.executable).is_absolute()
+        or sys.flags.isolated != 1
+        or sys.flags.no_site != 1
+        or sys.flags.dont_write_bytecode != 1
+        or sys.flags.ignore_environment != 1
+        or sys.flags.safe_path is not True
+    ):
+        fail()
+
+
+def enforce_gate_runner(python_value: str) -> None:
+    enforce_gate_runner_flags()
+    if (
+        not isinstance(python_value, str)
+        or not python_value
+        or not Path(python_value).is_absolute()
+        or sys.executable != python_value
+    ):
+        fail()
+
+
+def system_temporary_root() -> Path:
+    text = "/private/tmp" if sys.platform == "darwin" else "/tmp"
+    candidate = Path(text)
+    try:
+        status = candidate.lstat()
+    except OSError:
+        fail()
+    if (
+        stat.S_ISLNK(status.st_mode)
+        or not stat.S_ISDIR(status.st_mode)
+        or status.st_uid != 0
+        or not status.st_mode & stat.S_ISVTX
+    ):
+        fail()
+    return candidate
+
+
+def absolute_path_chain(candidate: Path) -> list[tuple[Path, os.stat_result]]:
+    if not candidate.is_absolute() or ".." in candidate.parts:
+        fail()
+    chain: list[tuple[Path, os.stat_result]] = []
+    current = Path(candidate.anchor)
+    try:
+        chain.append((current, current.lstat()))
+        for part in candidate.parts[1:]:
+            if part in ("", ".", ".."):
+                fail()
+            current = current / part
+            chain.append((current, current.lstat()))
+    except OSError:
+        fail()
+    for path, status in chain:
+        if stat.S_ISLNK(status.st_mode):
+            fail()
+        if status.st_uid not in (0, os.geteuid()):
+            fail()
+        if status.st_mode & (stat.S_IWGRP | stat.S_IWOTH):
+            if path != system_temporary_root():
+                fail()
+    return chain
+
+
+def safe_absolute_path(value: str, *, kind: str) -> Path:
+    candidate = Path(value)
+    chain = absolute_path_chain(candidate)
+    candidate_status = chain[-1][1]
+    if kind == "directory":
+        if not stat.S_ISDIR(candidate_status.st_mode):
+            fail()
+    elif kind == "executable":
+        if not stat.S_ISREG(candidate_status.st_mode) or not os.access(
+            candidate, os.X_OK
+        ):
+            fail()
+    elif kind == "file":
+        if not stat.S_ISREG(candidate_status.st_mode):
+            fail()
+    else:
+        fail()
+    return candidate
+
+
+def safe_new_environment_path(value: str, *, source: Path) -> Path:
+    candidate = Path(value)
+    if (
+        not candidate.is_absolute()
+        or ".." in candidate.parts
+        or candidate.name != ".venv"
+        or os.path.lexists(candidate)
+    ):
+        fail()
+    parent = candidate.parent
+    chain = absolute_path_chain(parent)
+    parent_status = chain[-1][1]
+    if (
+        not stat.S_ISDIR(parent_status.st_mode)
+        or parent_status.st_uid != os.geteuid()
+        or parent_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+    if paths_overlap(candidate, source) and candidate != source / ".venv":
+        fail()
+    return candidate
+
+
+def open_bounded_regular_file(path: Path, *, maximum: int) -> bytes:
+    flags = os.O_RDONLY
+    if hasattr(os, "O_CLOEXEC"):
+        flags |= os.O_CLOEXEC
+    if hasattr(os, "O_NOFOLLOW"):
+        flags |= os.O_NOFOLLOW
+    descriptor: int | None = None
+    try:
+        descriptor = os.open(path, flags)
+        before = os.fstat(descriptor)
+        if (
+            not stat.S_ISREG(before.st_mode)
+            or before.st_size < 0
+            or before.st_size > maximum
+            or before.st_uid not in (0, os.geteuid())
+            or before.st_nlink != 1
+            or before.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        ):
+            fail()
+        chunks: list[bytes] = []
+        remaining = maximum + 1
+        while remaining:
+            chunk = os.read(descriptor, min(64 * 1024, remaining))
+            if not chunk:
+                break
+            chunks.append(chunk)
+            remaining -= len(chunk)
+        after = os.fstat(descriptor)
+    except OSError:
+        fail()
+    finally:
+        if descriptor is not None:
+            try:
+                os.close(descriptor)
+            except OSError:
+                fail()
+    data = b"".join(chunks)
+    if (
+        len(data) > maximum
+        or len(data) != before.st_size
+        or (
+            before.st_dev,
+            before.st_ino,
+            before.st_size,
+            before.st_mtime_ns,
+            before.st_mode,
+            before.st_uid,
+            before.st_nlink,
+        )
+        != (
+            after.st_dev,
+            after.st_ino,
+            after.st_size,
+            after.st_mtime_ns,
+            after.st_mode,
+            after.st_uid,
+            after.st_nlink,
+        )
+    ):
+        fail()
+    return data
+
+
+def sha256_regular_file(path: Path, *, maximum: int) -> str:
+    return hashlib.sha256(
+        open_bounded_regular_file(path, maximum=maximum)
+    ).hexdigest()
+
+
+def python_artifact_tree_sha256(root: Path) -> str:
+    digest = hashlib.sha256()
+    entry_count = 0
+    byte_count = 0
+    observed_bytecode: dict[str, str] = {}
+    observed_pycache_directories: set[str] = set()
+    try:
+        candidates = sorted(
+            root.rglob("*"),
+            key=lambda path: path.relative_to(root).as_posix().encode("utf-8"),
+        )
+    except (OSError, RuntimeError, UnicodeError):
+        fail()
+    for path in candidates:
+        entry_count += 1
+        if entry_count > MAX_PYTHON_TREE_ENTRIES:
+            fail()
+        try:
+            status = path.lstat()
+            relative_bytes = path.relative_to(root).as_posix().encode("utf-8", "strict")
+        except (OSError, ValueError, UnicodeError):
+            fail()
+        if (
+            status.st_uid != os.geteuid()
+            or status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        ):
+            fail()
+        if stat.S_ISDIR(status.st_mode):
+            kind = b"d"
+            content = b""
+            if path.name == "__pycache__":
+                observed_pycache_directories.add(
+                    path.relative_to(root).as_posix()
+                )
+        elif stat.S_ISREG(status.st_mode):
+            kind = b"f"
+            content = open_bounded_regular_file(path, maximum=64 * 1024 * 1024)
+            byte_count += len(content)
+            if byte_count > MAX_PYTHON_TREE_BYTES:
+                fail()
+            content_digest = hashlib.sha256(content).hexdigest()
+            if path.suffix == ".pyc":
+                observed_bytecode[path.relative_to(root).as_posix()] = (
+                    content_digest
+                )
+            content = bytes.fromhex(content_digest)
+        elif stat.S_ISLNK(status.st_mode):
+            kind = b"l"
+            try:
+                link_text = os.readlink(path)
+                if Path(link_text).is_absolute():
+                    fail()
+                (path.parent / link_text).resolve(strict=True).relative_to(root)
+                content = link_text.encode("utf-8", "strict")
+            except (OSError, RuntimeError, ValueError, UnicodeError):
+                fail()
+        else:
+            fail()
+        digest.update(kind)
+        digest.update(len(relative_bytes).to_bytes(4, "big"))
+        digest.update(relative_bytes)
+        digest.update(stat.S_IMODE(status.st_mode).to_bytes(4, "big"))
+        digest.update(len(content).to_bytes(8, "big"))
+        digest.update(content)
+    expected_pycache_directories = {
+        str(PurePosixPath(relative).parent)
+        for relative in EXPECTED_PYTHON_BYTECODE
+    }
+    if (
+        observed_bytecode != EXPECTED_PYTHON_BYTECODE
+        or observed_pycache_directories != expected_pycache_directories
+    ):
+        fail()
+    if (
+        entry_count < MIN_PYTHON_TREE_ENTRIES
+        or byte_count < MIN_PYTHON_TREE_BYTES
+    ):
+        fail()
+    return digest.hexdigest()
+
+
+def read_regular_file(root: Path, relative: str, *, maximum: int) -> bytes:
+    path = root / relative
+    try:
+        path_status = path.lstat()
+        resolved = path.resolve(strict=True)
+        resolved.relative_to(root)
+    except (OSError, RuntimeError, ValueError):
+        fail()
+    if (
+        stat.S_ISLNK(path_status.st_mode)
+        or not stat.S_ISREG(path_status.st_mode)
+        or path_status.st_uid != os.geteuid()
+        or path_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+    if path_status.st_size < 0 or path_status.st_size > maximum:
+        fail()
+    return open_bounded_regular_file(path, maximum=maximum)
+
+
+def capture_policy_snapshot(source: Path) -> dict[str, bytes]:
+    first = {
+        relative: read_regular_file(source, relative, maximum=maximum)
+        for relative, maximum in POLICY_FILE_LIMITS.items()
+    }
+    second = {
+        relative: read_regular_file(source, relative, maximum=maximum)
+        for relative, maximum in POLICY_FILE_LIMITS.items()
+    }
+    if first != second:
+        fail()
+    return first
+
+
+def snapshot_file(
+    source_or_snapshot: Path | dict[str, bytes], relative: str, *, maximum: int
+) -> bytes:
+    if isinstance(source_or_snapshot, Path):
+        return read_regular_file(source_or_snapshot, relative, maximum=maximum)
+    if not isinstance(source_or_snapshot, dict) or set(source_or_snapshot) != set(
+        POLICY_FILE_LIMITS
+    ):
+        fail()
+    data = source_or_snapshot.get(relative)
+    if not isinstance(data, bytes) or len(data) > maximum:
+        fail()
+    return data
+
+
+def verify_policy_snapshot_unchanged(
+    source: Path, snapshot: dict[str, bytes]
+) -> None:
+    if capture_policy_snapshot(source) != snapshot:
+        fail()
+
+
+def private_root_for(path: Path) -> Path:
+    temporary_root = system_temporary_root()
+    try:
+        relative = path.relative_to(temporary_root)
+    except ValueError:
+        fail()
+    if len(relative.parts) < 1:
+        fail()
+    private_root = temporary_root / relative.parts[0]
+    chain = absolute_path_chain(path)
+    status_by_path = {candidate: status for candidate, status in chain}
+    root_status = status_by_path.get(private_root)
+    if (
+        root_status is None
+        or not stat.S_ISDIR(root_status.st_mode)
+        or root_status.st_uid != os.geteuid()
+        or stat.S_IMODE(root_status.st_mode) != 0o700
+    ):
+        fail()
+    return private_root
+
+
+def paths_overlap(first: Path, second: Path) -> bool:
+    try:
+        first.relative_to(second)
+        return True
+    except ValueError:
+        pass
+    try:
+        second.relative_to(first)
+        return True
+    except ValueError:
+        return False
+
+
+def directory_is_empty(directory: Path) -> bool:
+    try:
+        return next(directory.iterdir(), None) is None
+    except OSError:
+        fail()
+
+
+def verify_input_boundaries(
+    source: Path,
+    python: Path,
+    uv: Path,
+    cache: Path,
+    ca_file: Path,
+    environment: Path,
+) -> None:
+    source_status = source.lstat()
+    cache_status = cache.lstat()
+    ca_status = ca_file.lstat()
+    if (
+        source_status.st_uid != os.geteuid()
+        or cache_status.st_uid != os.geteuid()
+        or stat.S_IMODE(cache_status.st_mode) != 0o700
+        or not directory_is_empty(cache)
+        or ca_status.st_uid != 0
+        or ca_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+    python_root = private_root_for(python)
+    uv_root = private_root_for(uv)
+    cache_root = private_root_for(cache)
+    if cache != cache_root:
+        fail()
+    boundaries = (source, python_root, uv_root, cache_root)
+    for index, first in enumerate(boundaries):
+        for second in boundaries[index + 1 :]:
+            if paths_overlap(first, second):
+                fail()
+    for boundary in (python_root, uv_root, cache_root):
+        if paths_overlap(environment, boundary):
+            fail()
+    if paths_overlap(environment, source) and environment != source / ".venv":
+        fail()
+
+
+def verify_tool_provenance(
+    source_or_snapshot: Path | dict[str, bytes], python: Path, uv: Path
+) -> None:
+    policy = current_platform_policy()
+    python_root = private_root_for(python)
+    uv_root = private_root_for(uv)
+    if str(python.relative_to(python_root)) != policy["python_relative"]:
+        fail()
+    if str(uv.relative_to(uv_root)) != policy["uv_relative"]:
+        fail()
+    if sha256_regular_file(python, maximum=128 * 1024 * 1024) != policy[
+        "python_executable_sha256"
+    ]:
+        fail()
+    python_artifact_root = python_root / Path(
+        str(policy["python_relative"])
+    ).parents[1]
+    if python_artifact_tree_sha256(python_artifact_root) != policy[
+        "python_tree_sha256"
+    ]:
+        fail()
+    if sha256_regular_file(uv, maximum=128 * 1024 * 1024) != policy[
+        "uv_executable_sha256"
+    ]:
+        fail()
+    python_archive = python_root / str(policy["python_archive_relative"])
+    absolute_path_chain(python_archive)
+    try:
+        python_archive_status = python_archive.lstat()
+    except OSError:
+        fail()
+    if (
+        python_archive_status.st_size != EXPECTED_PYTHON_ARCHIVE_SIZE
+        or sha256_regular_file(
+            python_archive, maximum=32 * 1024 * 1024
+        )
+        != EXPECTED_PYTHON_ARCHIVE_SHA256
+    ):
+        fail()
+    uv_archive = uv_root / "uv.tar.gz"
+    absolute_path_chain(uv_archive)
+    if sha256_regular_file(uv_archive, maximum=64 * 1024 * 1024) != (
+        EXPECTED_UV_ARCHIVE_SHA256
+    ):
+        fail()
+    data = snapshot_file(
+        source_or_snapshot, "runtime/versions.toml", maximum=64 * 1024
+    )
+    try:
+        document = tomllib.loads(data.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, tomllib.TOMLDecodeError):
+        fail()
+    integrity = document.get("integrity")
+    if (
+        not isinstance(integrity, dict)
+        or integrity.get("python_provider")
+        != "astral-sh/python-build-standalone"
+        or integrity.get("python_release_tag") != "20260825"
+        or integrity.get("python_release_commit")
+        != EXPECTED_PYTHON_RELEASE_COMMIT
+        or integrity.get("python_macos_arm64_asset")
+        != EXPECTED_PYTHON_ASSET
+        or integrity.get("python_macos_arm64_archive_url")
+        != EXPECTED_PYTHON_ASSET_URL
+        or integrity.get("python_macos_arm64_archive_size")
+        != EXPECTED_PYTHON_ARCHIVE_SIZE
+        or integrity.get("python_macos_arm64_archive_sha256")
+        != EXPECTED_PYTHON_ARCHIVE_SHA256
+        or integrity.get("python_verification")
+        != (
+            "official-github-immutable-release-api-digest-local-sha256-"
+            "and-signed-release-commit"
+        )
+        or integrity.get("uv_macos_arm64_archive_sha256")
+        != EXPECTED_UV_ARCHIVE_SHA256
+    ):
+        fail()
+
+
+def verify_ca_provenance(ca_file: Path) -> None:
+    policy = current_platform_policy()
+    if sha256_regular_file(ca_file, maximum=1024 * 1024) != policy["ca_sha256"]:
+        fail()
+
+
+def parse_runtime(
+    source_or_snapshot: Path | dict[str, bytes]
+) -> dict[str, str]:
+    data = snapshot_file(
+        source_or_snapshot, "runtime/versions.toml", maximum=64 * 1024
+    )
+    try:
+        document = tomllib.loads(data.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, tomllib.TOMLDecodeError):
+        fail()
+    runtime = document.get("runtime")
+    integrity = document.get("integrity")
+    if runtime != EXPECTED_RUNTIME or not isinstance(integrity, dict):
+        fail()
+    commit = integrity.get("uv_release_commit")
+    if commit != EXPECTED_UV_COMMIT or not re.fullmatch(r"[0-9a-f]{40}", commit):
+        fail()
+    if integrity.get("uv_attestation") != "verified-with-github-cli-and-public-sigstore-rekor":
+        fail()
+    expected_python_integrity = {
+        "python_provider": "astral-sh/python-build-standalone",
+        "python_release_tag": "20260825",
+        "python_release_commit": EXPECTED_PYTHON_RELEASE_COMMIT,
+        "python_macos_arm64_asset": EXPECTED_PYTHON_ASSET,
+        "python_macos_arm64_archive_url": EXPECTED_PYTHON_ASSET_URL,
+        "python_macos_arm64_archive_size": EXPECTED_PYTHON_ARCHIVE_SIZE,
+        "python_macos_arm64_archive_sha256": EXPECTED_PYTHON_ARCHIVE_SHA256,
+        "python_verification": (
+            "official-github-immutable-release-api-digest-local-sha256-"
+            "and-signed-release-commit"
+        ),
+    }
+    if any(integrity.get(key) != value for key, value in expected_python_integrity.items()):
+        fail()
+    archive_hash = integrity.get("uv_macos_arm64_archive_sha256")
+    if (
+        archive_hash != EXPECTED_UV_ARCHIVE_SHA256
+        or not isinstance(archive_hash, str)
+        or not re.fullmatch(r"[0-9a-f]{64}", archive_hash)
+    ):
+        fail()
+    return dict(runtime)
+
+
+def parse_project(source_or_snapshot: Path | dict[str, bytes]) -> None:
+    data = snapshot_file(
+        source_or_snapshot, "pyproject.toml", maximum=64 * 1024
+    )
+    try:
+        document = tomllib.loads(data.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, tomllib.TOMLDecodeError):
+        fail()
+    project = document.get("project")
+    uv = document.get("tool", {}).get("uv") if isinstance(document.get("tool"), dict) else None
+    if not isinstance(project, dict) or not isinstance(uv, dict):
+        fail()
+    dependencies = project.get("dependencies")
+    if (
+        project.get("name") != "audience-foundry-travel-readiness"
+        or project.get("version") != "0.1.0"
+        or project.get("requires-python") != f"=={EXPECTED_PYTHON}"
+        or not isinstance(dependencies, list)
+        or any(not isinstance(item, str) for item in dependencies)
+        or set(dependencies) != EXPECTED_DIRECT_DEPENDENCIES
+        or len(dependencies) != len(EXPECTED_DIRECT_DEPENDENCIES)
+        or uv.get("required-version") != f"=={EXPECTED_UV}"
+        or uv.get("package") is not False
+    ):
+        fail()
+
+
+def parse_lock_contract(
+    source_or_snapshot: Path | dict[str, bytes],
+) -> tuple[dict[str, str], dict[str, dict[str, dict[str, object]]]]:
+    data = snapshot_file(
+        source_or_snapshot, "uv.lock", maximum=8 * 1024 * 1024
+    )
+    try:
+        document = tomllib.loads(data.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, tomllib.TOMLDecodeError):
+        fail()
+    packages = document.get("package")
+    if (
+        document.get("version") != EXPECTED_LOCK_FORMAT
+        or document.get("revision") != EXPECTED_LOCK_REVISION
+        or document.get("requires-python") != f"=={EXPECTED_PYTHON}"
+        or not isinstance(packages, list)
+        or set(document) != {"version", "revision", "requires-python", "package"}
+    ):
+        fail()
+
+    locked: dict[str, str] = {}
+    release_artifacts: dict[str, dict[str, dict[str, object]]] = {}
+    for package in packages:
+        if not isinstance(package, dict):
+            fail()
+        if not {"name", "version", "source"}.issubset(package) or not set(
+            package
+        ).issubset(
+            {
+                "name",
+                "version",
+                "source",
+                "dependencies",
+                "optional-dependencies",
+                "metadata",
+                "sdist",
+                "wheels",
+            }
+        ):
+            fail()
+        name = canonical_name(package.get("name"))
+        version = package.get("version")
+        source_table = package.get("source")
+        if (
+            name in locked
+            or not isinstance(version, str)
+            or not isinstance(source_table, dict)
+        ):
+            fail()
+        locked[name] = version
+        if name == "audience-foundry-travel-readiness":
+            if source_table != {"virtual": "."}:
+                fail()
+        elif source_table != {"registry": PYPI_REGISTRY}:
+            fail()
+
+        expected_dependencies = EXPECTED_LOCK_DEPENDENCIES.get(name)
+        if expected_dependencies is None or package.get("dependencies", []) != (
+            expected_dependencies
+        ):
+            fail()
+        expected_optional = EXPECTED_LOCK_OPTIONAL_DEPENDENCIES.get(name)
+        if package.get("optional-dependencies") != expected_optional:
+            if expected_optional is not None or "optional-dependencies" in package:
+                fail()
+        expected_metadata = (
+            EXPECTED_LOCAL_METADATA
+            if name == "audience-foundry-travel-readiness"
+            else None
+        )
+        if package.get("metadata") != expected_metadata:
+            if expected_metadata is not None or "metadata" in package:
+                fail()
+
+        package_artifacts: dict[str, dict[str, object]] = {}
+        for artifact_key in ("sdist", "wheels"):
+            artifact_value = package.get(artifact_key)
+            artifacts: Iterable[object]
+            if artifact_value is None:
+                continue
+            if artifact_key == "sdist":
+                artifacts = (artifact_value,)
+            elif isinstance(artifact_value, list):
+                artifacts = artifact_value
+            else:
+                fail()
+            for artifact in artifacts:
+                if not isinstance(artifact, dict):
+                    fail()
+                if set(artifact) != {"url", "hash", "size", "upload-time"}:
+                    fail()
+                if artifact.get("yanked") not in (None, False):
+                    fail()
+                digest = artifact.get("hash")
+                if not isinstance(digest, str) or not re.fullmatch(
+                    r"sha256:[0-9a-f]{64}", digest
+                ):
+                    fail()
+                url = artifact.get("url")
+                size = artifact.get("size")
+                upload_time = artifact.get("upload-time")
+                if (
+                    not isinstance(url, str)
+                    or not isinstance(size, int)
+                    or isinstance(size, bool)
+                    or size <= 0
+                    or not isinstance(upload_time, str)
+                    or not re.fullmatch(
+                        r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z",
+                        upload_time,
+                    )
+                ):
+                    fail()
+                try:
+                    parsed_url = urllib.parse.urlsplit(url)
+                except ValueError:
+                    fail()
+                path_parts = PurePosixPath(parsed_url.path).parts
+                filename = path_parts[-1] if path_parts else ""
+                if (
+                    parsed_url.scheme != "https"
+                    or parsed_url.hostname != "files.pythonhosted.org"
+                    or parsed_url.username is not None
+                    or parsed_url.password is not None
+                    or parsed_url.port not in (None, 443)
+                    or parsed_url.query
+                    or parsed_url.fragment
+                    or len(path_parts) != 6
+                    or path_parts[1] != "packages"
+                    or urllib.parse.unquote(filename) != filename
+                    or not filename
+                    or filename in package_artifacts
+                ):
+                    fail()
+                packagetype = "sdist" if artifact_key == "sdist" else "bdist_wheel"
+                if packagetype == "bdist_wheel" and not filename.endswith(".whl"):
+                    fail()
+                if packagetype == "sdist" and filename.endswith(".whl"):
+                    fail()
+                package_artifacts[filename] = {
+                    "digest": digest.removeprefix("sha256:"),
+                    "packagetype": packagetype,
+                    "size": size,
+                    "url": url,
+                }
+        if name == "audience-foundry-travel-readiness":
+            if package_artifacts:
+                fail()
+        elif not package_artifacts or not any(
+            artifact["packagetype"] == "bdist_wheel"
+            for artifact in package_artifacts.values()
+        ):
+            fail()
+        release_artifacts[name] = package_artifacts
+    if locked != EXPECTED_LOCK_PACKAGES:
+        fail()
+    policy = current_platform_policy()
+    wheel_filenames = policy.get("wheel_filenames")
+    lock_wheel_filenames = policy.get("lock_wheel_filenames")
+    if not isinstance(wheel_filenames, dict) or not isinstance(
+        lock_wheel_filenames, dict
+    ):
+        fail()
+    for name, expected_filename in lock_wheel_filenames.items():
+        artifact = release_artifacts.get(name, {}).get(str(expected_filename))
+        if not isinstance(artifact, dict) or artifact.get("packagetype") != "bdist_wheel":
+            fail()
+    expected_installed = set(EXPECTED_LOCK_PACKAGES) - {
+        "audience-foundry-travel-readiness",
+        "tzdata",
+    }
+    if set(wheel_filenames) != expected_installed:
+        fail()
+    expected_registry = set(EXPECTED_LOCK_PACKAGES) - {
+        "audience-foundry-travel-readiness"
+    }
+    if (
+        set(lock_wheel_filenames) != expected_registry
+        or any(
+            lock_wheel_filenames.get(name) != filename
+            for name, filename in wheel_filenames.items()
+        )
+    ):
+        fail()
+    return locked, release_artifacts
+
+
+def parse_lock(source_or_snapshot: Path | dict[str, bytes]) -> dict[str, str]:
+    locked, _ = parse_lock_contract(source_or_snapshot)
+    return locked
+
+
+def verify_notices(source_or_snapshot: Path | dict[str, bytes]) -> None:
+    data = snapshot_file(
+        source_or_snapshot, "THIRD_PARTY_NOTICES.md", maximum=512 * 1024
+    )
+    try:
+        notice = data.decode("utf-8", "strict")
+    except UnicodeDecodeError:
+        fail()
+    lines = notice.splitlines()
+    required_rows = NOTICE_REQUIREMENTS[:-1]
+    component_names: list[str] = []
+    for row in required_rows:
+        matching_lines = [line for line in lines if row in line]
+        if len(matching_lines) != 1:
+            fail()
+        fields = [
+            field.strip()
+            for field in matching_lines[0].strip("|").split("|")
+        ]
+        if len(fields) not in (4, 5) or not fields[0]:
+            fail()
+        component_names.append(fields[0])
+    if (
+        hashlib.sha256(data).hexdigest() != EXPECTED_NOTICE_SHA256
+        or len(component_names) != len(set(component_names))
+        or NOTICE_REQUIREMENTS[-1] not in notice
+    ):
+        fail()
+
+
+def fixed_child_environment(workspace: Path, cache: Path, python: Path) -> dict[str, str]:
+    config = workspace / "config"
+    data = workspace / "data"
+    state = workspace / "state"
+    for directory in (config, data, state):
+        directory.mkdir(mode=0o700)
+    return {
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "PYTHONDONTWRITEBYTECODE": "1",
+        "PYTHONHASHSEED": "0",
+        "TZ": "UTC",
+        "UV_CACHE_DIR": str(cache),
+        "UV_NO_CACHE": "1",
+        "UV_NO_CONFIG": "1",
+        "UV_NO_PROGRESS": "1",
+        "UV_OFFLINE": "1",
+        "UV_NO_BUILD": "1",
+        "UV_PYTHON": str(python),
+        "UV_PYTHON_DOWNLOADS": "never",
+        "XDG_CACHE_HOME": str(cache),
+        "XDG_CONFIG_HOME": str(config),
+        "XDG_DATA_HOME": str(data),
+        "XDG_STATE_HOME": str(state),
+    }
+
+
+def kill_and_reap_process_group(process: subprocess.Popen[bytes]) -> None:
+    failed = False
+    try:
+        os.killpg(process.pid, signal.SIGKILL)
+    except ProcessLookupError:
+        pass
+    except OSError:
+        failed = True
+    try:
+        process.wait(timeout=5)
+    except (OSError, subprocess.SubprocessError):
+        failed = True
+    if failed:
+        fail()
+
+
+def require_process_group_gone(process: subprocess.Popen[bytes]) -> None:
+    try:
+        os.killpg(process.pid, 0)
+    except ProcessLookupError:
+        return
+    except OSError:
+        fail()
+    kill_and_reap_process_group(process)
+    fail()
+
+
+def run_fixed(
+    arguments: list[str],
+    *,
+    cwd: Path,
+    environment: dict[str, str],
+    timeout: int = 120,
+) -> bytes:
+    process: subprocess.Popen[bytes] | None = None
+    try:
+        process = subprocess.Popen(
+            arguments,
+            cwd=cwd,
+            env=environment,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            start_new_session=True,
+        )
+    except (OSError, subprocess.SubprocessError):
+        fail()
+    if process.stdout is None or process.stderr is None:
+        kill_and_reap_process_group(process)
+        fail()
+    streams = {
+        process.stdout.fileno(): (process.stdout, True),
+        process.stderr.fileno(): (process.stderr, False),
+    }
+    selector = selectors.DefaultSelector()
+    stdout_chunks: list[bytes] = []
+    output_sizes = {True: 0, False: 0}
+    deadline = time.monotonic() + timeout
+    try:
+        for descriptor, (stream, _) in streams.items():
+            os.set_blocking(descriptor, False)
+            selector.register(stream, selectors.EVENT_READ)
+        while selector.get_map():
+            remaining = deadline - time.monotonic()
+            if remaining <= 0:
+                fail()
+            events = selector.select(min(remaining, 0.25))
+            if not events and process.poll() is not None:
+                events = [
+                    (key, selectors.EVENT_READ)
+                    for key in tuple(selector.get_map().values())
+                ]
+            for key, _ in events:
+                stream = key.fileobj
+                descriptor = stream.fileno()
+                is_stdout = streams[descriptor][1]
+                try:
+                    chunk = os.read(descriptor, 64 * 1024)
+                except BlockingIOError:
+                    continue
+                if not chunk:
+                    selector.unregister(stream)
+                    stream.close()
+                    continue
+                output_sizes[is_stdout] += len(chunk)
+                if output_sizes[is_stdout] > MAX_SUBPROCESS_OUTPUT_BYTES:
+                    fail()
+                if is_stdout:
+                    stdout_chunks.append(chunk)
+        remaining = deadline - time.monotonic()
+        if remaining <= 0:
+            fail()
+        return_code = process.wait(timeout=remaining)
+    except GateError:
+        kill_and_reap_process_group(process)
+        raise
+    except (OSError, subprocess.SubprocessError):
+        kill_and_reap_process_group(process)
+        fail()
+    except BaseException:
+        kill_and_reap_process_group(process)
+        raise
+    finally:
+        selector.close()
+        for stream, _ in streams.values():
+            if not stream.closed:
+                stream.close()
+    require_process_group_gone(process)
+    if return_code != 0:
+        fail()
+    return b"".join(stdout_chunks)
+
+
+def exact_python_version(python: Path, *, cwd: Path, environment: dict[str, str]) -> None:
+    output = run_fixed(
+        [str(python), "-I", "-S", "-B", "--version"],
+        cwd=cwd,
+        environment=environment,
+    )
+    try:
+        version = output.decode("ascii", "strict").strip()
+    except UnicodeDecodeError:
+        fail()
+    if version != f"Python {EXPECTED_PYTHON}":
+        fail()
+
+
+def exact_uv_version(uv: Path, *, cwd: Path, environment: dict[str, str]) -> None:
+    output = run_fixed([str(uv), "--version"], cwd=cwd, environment=environment)
+    try:
+        version = output.decode("ascii", "strict").strip()
+    except UnicodeDecodeError:
+        fail()
+    match = re.fullmatch(
+        rf"uv {re.escape(EXPECTED_UV)} \(([0-9a-f]{{9,40}}) [^)]+\)", version
+    )
+    if match is None or not EXPECTED_UV_COMMIT.startswith(match.group(1)):
+        fail()
+
+
+INSPECTION_PROGRAM = r'''
+import base64
+import csv
+import hashlib
+import importlib.metadata
+import io
+import json
+import os
+from pathlib import Path, PurePosixPath
+import platform
+import re
+import stat
+import sys
+
+expected_licenses = json.loads(sys.argv[1])
+expected_license_files = {
+    key: set(value) for key, value in json.loads(sys.argv[2]).items()
+}
+expected_platform = sys.argv[3]
+expected_machine = sys.argv[4]
+expected_wheel_tags = json.loads(sys.argv[5])
+attestation_path = Path(sys.argv[6])
+environment_root = Path(sys.argv[7])
+license_fingerprints = {
+    "Apache-2.0": (b"Apache License", b"Version 2.0"),
+    "BSD-3-Clause": (b"Redistribution and use in source and binary forms",),
+    "LGPL-3.0-only": (b"GNU LESSER GENERAL PUBLIC LICENSE", b"Version 3"),
+    "MIT": (b"Permission is hereby granted, free of charge",),
+}
+try:
+    environment_status = environment_root.lstat()
+    environment_root = environment_root.resolve(strict=True)
+except (OSError, RuntimeError):
+    raise SystemExit(10)
+if (
+    stat.S_ISLNK(environment_status.st_mode)
+    or not stat.S_ISDIR(environment_status.st_mode)
+    or environment_status.st_uid != os.geteuid()
+    or environment_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+):
+    raise SystemExit(10)
+site_packages = environment_root / "lib/python3.14/site-packages"
+try:
+    site_status = site_packages.lstat()
+    site_packages.resolve(strict=True).relative_to(environment_root)
+except (OSError, RuntimeError, ValueError):
+    raise SystemExit(10)
+if (
+    stat.S_ISLNK(site_status.st_mode)
+    or not stat.S_ISDIR(site_status.st_mode)
+    or site_status.st_uid != os.geteuid()
+    or site_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+):
+    raise SystemExit(10)
+
+try:
+    attestation_status = attestation_path.lstat()
+    if attestation_path.resolve(strict=True) != attestation_path:
+        raise OSError
+except (OSError, RuntimeError):
+    raise SystemExit(10)
+if (
+    stat.S_ISLNK(attestation_status.st_mode)
+    or not stat.S_ISREG(attestation_status.st_mode)
+    or stat.S_IMODE(attestation_status.st_mode) != 0o600
+    or attestation_status.st_uid != os.geteuid()
+    or attestation_status.st_nlink != 1
+    or attestation_status.st_size < 2
+    or attestation_status.st_size > 8 * 1024 * 1024
+):
+    raise SystemExit(10)
+with attestation_path.open("rb") as handle:
+    attestation_bytes = handle.read(8 * 1024 * 1024 + 1)
+if len(attestation_bytes) != attestation_status.st_size:
+    raise SystemExit(10)
+try:
+    expected_wheel_records = json.loads(attestation_bytes.decode("utf-8", "strict"))
+except (UnicodeDecodeError, json.JSONDecodeError):
+    raise SystemExit(10)
+if not isinstance(expected_wheel_records, dict) or set(expected_wheel_records) != set(expected_wheel_tags):
+    raise SystemExit(10)
+
+if (
+    sys.platform != expected_platform
+    or platform.machine() != expected_machine
+    or sys.implementation.name != "cpython"
+    or tuple(sys.version_info[:3]) != (3, 14, 7)
+    or sys.flags.isolated != 1
+    or sys.flags.no_site != 1
+    or sys.flags.dont_write_bytecode != 1
+):
+    raise SystemExit(10)
+
+def canonical(value):
+    import re
+    return re.sub(r"[-_.]+", "-", value).lower()
+
+def declared_license(metadata):
+    declared = set()
+    expression = metadata.get("License-Expression")
+    if expression:
+        declared.add(expression)
+    legacy = metadata.get("License")
+    if legacy:
+        declared.add(legacy)
+    classifier_map = {
+        "License :: OSI Approved :: BSD License": "BSD-3-Clause",
+        "License :: OSI Approved :: MIT License": "MIT",
+        "License :: OSI Approved :: GNU Lesser General Public License v3 (LGPLv3)": "LGPL-3.0-only",
+        "License :: OSI Approved :: Apache Software License": "Apache-2.0",
+    }
+    for classifier in metadata.get_all("Classifier", []):
+        mapped = classifier_map.get(classifier)
+        if mapped:
+            declared.add(mapped)
+    return declared
+
+def locate_regular(distribution, distribution_files, suffix_parts, maximum):
+    matches = []
+    for recorded in distribution_files:
+        recorded_path = PurePosixPath(str(recorded))
+        if (
+            len(recorded_path.parts) >= len(suffix_parts) + 1
+            and recorded_path.parts[-len(suffix_parts):] == tuple(suffix_parts)
+            and recorded_path.parts[-len(suffix_parts) - 1].endswith(".dist-info")
+        ):
+            if recorded_path.is_absolute() or ".." in recorded_path.parts:
+                raise SystemExit(10)
+            matches.append(recorded)
+    if len(matches) != 1:
+        raise SystemExit(10)
+    target = Path(distribution.locate_file(matches[0]))
+    try:
+        target_status = target.lstat()
+        target.resolve(strict=True).relative_to(environment_root)
+    except (OSError, RuntimeError, ValueError):
+        raise SystemExit(10)
+    if (
+        stat.S_ISLNK(target_status.st_mode)
+        or not stat.S_ISREG(target_status.st_mode)
+        or target_status.st_uid != os.geteuid()
+        or target_status.st_nlink != 1
+        or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        raise SystemExit(10)
+    if target_status.st_size < 1 or target_status.st_size > maximum:
+        raise SystemExit(10)
+    with target.open("rb") as handle:
+        content = handle.read(maximum + 1)
+    if len(content) != target_status.st_size or len(content) > maximum:
+        raise SystemExit(10)
+    return content
+
+installed = {}
+native = []
+native_roles = set()
+expected_site_files = set()
+generated_scripts = {
+    "django": {
+        "../../../bin/django-admin": (
+            "from django.core.management import execute_from_command_line",
+            "execute_from_command_line()",
+        ),
+    },
+    "gunicorn": {
+        "../../../bin/gunicorn": (
+            "from gunicorn.app.wsgiapp import run",
+            "run()",
+        ),
+        "../../../bin/gunicornc": (
+            "from gunicorn.ctl.cli import main",
+            "main()",
+        ),
+    },
+    "sqlparse": {
+        "../../../bin/sqlformat": (
+            "from sqlparse.__main__ import main",
+            "main()",
+        ),
+    },
+}
+for distribution in importlib.metadata.distributions(path=[str(site_packages)]):
+    metadata = distribution.metadata
+    name_value = metadata.get("Name")
+    if not name_value:
+        raise SystemExit(10)
+    name = canonical(name_value)
+    if name in installed:
+        raise SystemExit(10)
+    expected_license = expected_licenses.get(name)
+    expected_files = expected_license_files.get(name)
+    if expected_license is None or expected_files is None:
+        raise SystemExit(10)
+    if declared_license(metadata) != {expected_license}:
+        raise SystemExit(10)
+    license_headers = metadata.get_all("License-File", [])
+    if set(license_headers) != expected_files or len(license_headers) != len(expected_files):
+        raise SystemExit(10)
+    distribution_files = tuple(distribution.files or ())
+    record_content = locate_regular(
+        distribution, distribution_files, ("RECORD",), 4 * 1024 * 1024
+    )
+    try:
+        record_rows = list(
+            csv.reader(io.StringIO(record_content.decode("utf-8", "strict"), newline=""))
+        )
+    except (UnicodeDecodeError, csv.Error):
+        raise SystemExit(10)
+    recorded_paths = set()
+    for row in record_rows:
+        if len(row) != 3:
+            raise SystemExit(10)
+        relative_text, encoded_digest, size_text = row
+        relative_path = PurePosixPath(relative_text)
+        if (
+            not relative_text
+            or relative_path.is_absolute()
+            or relative_text in recorded_paths
+        ):
+            raise SystemExit(10)
+        recorded_paths.add(relative_text)
+        target = Path(distribution.locate_file(relative_text))
+        try:
+            target_status = target.lstat()
+            resolved_target = target.resolve(strict=True)
+            resolved_target.relative_to(environment_root)
+        except (OSError, RuntimeError, ValueError):
+            raise SystemExit(10)
+        try:
+            expected_site_files.add(
+                resolved_target.relative_to(site_packages).as_posix()
+            )
+        except ValueError:
+            pass
+        if (
+            stat.S_ISLNK(target_status.st_mode)
+            or not stat.S_ISREG(target_status.st_mode)
+            or target_status.st_uid != os.geteuid()
+            or target_status.st_nlink != 1
+            or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        ):
+            raise SystemExit(10)
+        if relative_path.name == "RECORD" and relative_path.parent.name.endswith(".dist-info"):
+            with target.open("rb") as handle:
+                current_record = handle.read(4 * 1024 * 1024 + 1)
+            if encoded_digest or size_text or current_record != record_content:
+                raise SystemExit(10)
+            continue
+        if target_status.st_size < 0 or target_status.st_size > 64 * 1024 * 1024:
+            raise SystemExit(10)
+        try:
+            expected_size = int(size_text, 10)
+        except ValueError:
+            raise SystemExit(10)
+        if expected_size != target_status.st_size or not encoded_digest.startswith("sha256="):
+            raise SystemExit(10)
+        with target.open("rb") as handle:
+            content = handle.read(64 * 1024 * 1024 + 1)
+        if len(content) != expected_size:
+            raise SystemExit(10)
+        observed_digest = base64.urlsafe_b64encode(
+            hashlib.sha256(content).digest()
+        ).rstrip(b"=").decode("ascii")
+        if encoded_digest.removeprefix("sha256=") != observed_digest:
+            raise SystemExit(10)
+    if recorded_paths != {str(recorded) for recorded in distribution_files}:
+        raise SystemExit(10)
+    official_records = expected_wheel_records.get(name)
+    if not isinstance(official_records, dict) or not official_records:
+        raise SystemExit(10)
+    for relative_text, expected_record in official_records.items():
+        relative_path = PurePosixPath(relative_text)
+        if (
+            not isinstance(relative_text, str)
+            or not relative_text
+            or relative_path.is_absolute()
+            or ".." in relative_path.parts
+            or not isinstance(expected_record, dict)
+            or set(expected_record) != {"digest", "size"}
+            or not isinstance(expected_record.get("digest"), str)
+            or not re.fullmatch(r"[0-9a-f]{64}", expected_record["digest"])
+            or not isinstance(expected_record.get("size"), int)
+            or isinstance(expected_record["size"], bool)
+            or expected_record["size"] < 0
+            or expected_record["size"] > 64 * 1024 * 1024
+        ):
+            raise SystemExit(10)
+        target = Path(distribution.locate_file(relative_text))
+        try:
+            target_status = target.lstat()
+            target.resolve(strict=True).relative_to(environment_root)
+        except (OSError, RuntimeError, ValueError):
+            raise SystemExit(10)
+        if (
+            stat.S_ISLNK(target_status.st_mode)
+            or not stat.S_ISREG(target_status.st_mode)
+            or target_status.st_uid != os.geteuid()
+            or target_status.st_nlink != 1
+            or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            or target_status.st_size != expected_record["size"]
+        ):
+            raise SystemExit(10)
+        with target.open("rb") as handle:
+            content = handle.read(64 * 1024 * 1024 + 1)
+        if (
+            len(content) != expected_record["size"]
+            or hashlib.sha256(content).hexdigest() != expected_record["digest"]
+        ):
+            raise SystemExit(10)
+    record_paths = [
+        path for path in recorded_paths if path.endswith(".dist-info/RECORD")
+    ]
+    installer_paths = [
+        path for path in recorded_paths if path.endswith(".dist-info/INSTALLER")
+    ]
+    requested_paths = [
+        path for path in recorded_paths if path.endswith(".dist-info/REQUESTED")
+    ]
+    if (
+        len(record_paths) != 1
+        or len(installer_paths) != 1
+        or len(requested_paths) != 1
+    ):
+        raise SystemExit(10)
+    expected_installed_paths = (
+        set(official_records)
+        | {record_paths[0], installer_paths[0], requested_paths[0]}
+        | set(generated_scripts.get(name, {}))
+    )
+    if recorded_paths != expected_installed_paths:
+        raise SystemExit(10)
+    requested_target = Path(distribution.locate_file(requested_paths[0]))
+    try:
+        requested_status = requested_target.lstat()
+        requested_target.resolve(strict=True).relative_to(environment_root)
+    except (OSError, RuntimeError, ValueError):
+        raise SystemExit(10)
+    if (
+        stat.S_ISLNK(requested_status.st_mode)
+        or not stat.S_ISREG(requested_status.st_mode)
+        or requested_status.st_uid != os.geteuid()
+        or requested_status.st_nlink != 1
+        or requested_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        or requested_status.st_size != 0
+    ):
+        raise SystemExit(10)
+    for relative_text, (import_line, call_text) in generated_scripts.get(name, {}).items():
+        target = Path(distribution.locate_file(relative_text))
+        expected_script = (
+            f"#!{environment_root}/bin/python\n"
+            "# -*- coding: utf-8 -*-\n"
+            "import sys\n"
+            f"{import_line}\n"
+            'if __name__ == "__main__":\n'
+            '    if sys.argv[0].endswith("-script.pyw"):\n'
+            "        sys.argv[0] = sys.argv[0][:-11]\n"
+            '    elif sys.argv[0].endswith(".exe"):\n'
+            "        sys.argv[0] = sys.argv[0][:-4]\n"
+            f"    sys.exit({call_text})\n"
+        ).encode("utf-8")
+        try:
+            target_status = target.lstat()
+            target.resolve(strict=True).relative_to(environment_root)
+        except (OSError, RuntimeError, ValueError):
+            raise SystemExit(10)
+        if (
+            stat.S_ISLNK(target_status.st_mode)
+            or not stat.S_ISREG(target_status.st_mode)
+            or target_status.st_uid != os.geteuid()
+            or target_status.st_nlink != 1
+            or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            or target_status.st_size != len(expected_script)
+        ):
+            raise SystemExit(10)
+        with target.open("rb") as handle:
+            if handle.read(len(expected_script) + 1) != expected_script:
+                raise SystemExit(10)
+    installer = locate_regular(
+        distribution, distribution_files, ("INSTALLER",), 128
+    )
+    wheel_metadata = locate_regular(
+        distribution, distribution_files, ("WHEEL",), 16 * 1024
+    )
+    if installer != b"uv":
+        raise SystemExit(10)
+    try:
+        wheel_lines = wheel_metadata.decode("utf-8", "strict").splitlines()
+    except UnicodeDecodeError:
+        raise SystemExit(10)
+    wheel_tags = [line.removeprefix("Tag: ") for line in wheel_lines if line.startswith("Tag: ")]
+    expected_wheel_tag = expected_wheel_tags.get(name)
+    if wheel_tags != [expected_wheel_tag] or expected_wheel_tag is None:
+        raise SystemExit(10)
+    pure_values = [
+        line.removeprefix("Root-Is-Purelib: ").lower()
+        for line in wheel_lines
+        if line.startswith("Root-Is-Purelib: ")
+    ]
+    expected_pure = "false" if name == "psycopg-binary" else "true"
+    if pure_values != [expected_pure]:
+        raise SystemExit(10)
+    combined_license_material = bytearray()
+    for relative_text in license_headers:
+        relative = PurePosixPath(relative_text)
+        if relative.is_absolute() or ".." in relative.parts or not relative.parts:
+            raise SystemExit(10)
+        sample = locate_regular(
+            distribution,
+            distribution_files,
+            ("licenses", *relative.parts),
+            2 * 1024 * 1024,
+        )
+        if len(sample) < 32 or not sample.strip():
+            raise SystemExit(10)
+        combined_license_material.extend(sample)
+    fingerprints = license_fingerprints.get(expected_license)
+    if fingerprints is None or any(
+        fingerprint not in combined_license_material for fingerprint in fingerprints
+    ):
+        raise SystemExit(10)
+    installed[name] = distribution.version
+
+    if name == "psycopg-binary":
+        native_seen = set()
+        for relative in distribution_files:
+            relative_text = str(relative)
+            relative_path = PurePosixPath(relative_text)
+            basename = relative_path.name.lower()
+            suffix = relative_path.suffix.lower()
+            if re.search(r"\.so(?:\.\d+)*$", basename):
+                native_kind = ".so"
+            elif suffix in {".dylib", ".dll", ".pyd"}:
+                native_kind = suffix
+            else:
+                continue
+            if relative_path.is_absolute() or ".." in relative_path.parts:
+                raise SystemExit(10)
+            if relative_text in native_seen:
+                raise SystemExit(10)
+            native_seen.add(relative_text)
+            target = Path(distribution.locate_file(relative))
+            try:
+                target_status = target.lstat()
+                target.resolve(strict=True).relative_to(environment_root)
+            except (OSError, RuntimeError, ValueError):
+                raise SystemExit(10)
+            if (
+                stat.S_ISLNK(target_status.st_mode)
+                or not stat.S_ISREG(target_status.st_mode)
+                or target_status.st_uid != os.geteuid()
+                or target_status.st_nlink != 1
+                or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            ):
+                raise SystemExit(10)
+            if target_status.st_size <= 0:
+                raise SystemExit(10)
+            with target.open("rb") as handle:
+                native_header = handle.read(8)
+            if (
+                expected_platform != "darwin"
+                or expected_machine != "arm64"
+                or native_header != bytes.fromhex("cffaedfe0c000001")
+            ):
+                raise SystemExit(10)
+            native.append((relative_text, native_kind, target_status.st_size))
+            if native_kind in {".so", ".pyd"}:
+                native_roles.add("extension")
+            if "libpq" in basename:
+                native_roles.add("libpq")
+            if "libssl" in basename or "libcrypto" in basename:
+                native_roles.add("tls")
+            if "krb5" in basename or "gssapi" in basename or "com_err" in basename:
+                native_roles.add("kerberos")
+            if "ldap" in basename or "lber" in basename:
+                native_roles.add("ldap")
+
+seed_files = {
+    "_virtualenv.py": (
+        5246,
+        "cfb3db86aaa53bb62b5ff764970bec2d71c9228590a0ebec57f6ec926cc0bf1a",
+    ),
+    "_virtualenv.pth": (
+        18,
+        "69ac3d8f27e679c81b94ab30b3b56e9cd138219b1ba94a1fa3606d5a76a1433d",
+    ),
+}
+for relative_text, (expected_size, expected_digest) in seed_files.items():
+    target = site_packages / relative_text
+    try:
+        target_status = target.lstat()
+        target.resolve(strict=True).relative_to(site_packages)
+    except (OSError, RuntimeError, ValueError):
+        raise SystemExit(10)
+    if (
+        stat.S_ISLNK(target_status.st_mode)
+        or not stat.S_ISREG(target_status.st_mode)
+        or target_status.st_uid != os.geteuid()
+        or target_status.st_nlink != 1
+        or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        or target_status.st_size != expected_size
+    ):
+        raise SystemExit(10)
+    with target.open("rb") as handle:
+        content = handle.read(expected_size + 1)
+    if len(content) != expected_size or hashlib.sha256(content).hexdigest() != expected_digest:
+        raise SystemExit(10)
+    expected_site_files.add(relative_text)
+
+observed_site_files = set()
+entry_count = 0
+for target in site_packages.rglob("*"):
+    entry_count += 1
+    if entry_count > 20000:
+        raise SystemExit(10)
+    try:
+        target_status = target.lstat()
+        relative_text = target.relative_to(site_packages).as_posix()
+    except (OSError, ValueError):
+        raise SystemExit(10)
+    if (
+        stat.S_ISLNK(target_status.st_mode)
+        or target_status.st_uid != os.geteuid()
+        or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        raise SystemExit(10)
+    if stat.S_ISDIR(target_status.st_mode):
+        continue
+    if not stat.S_ISREG(target_status.st_mode):
+        raise SystemExit(10)
+    if target_status.st_nlink != 1:
+        raise SystemExit(10)
+    if relative_text.endswith(".pth") and relative_text != "_virtualenv.pth":
+        raise SystemExit(10)
+    observed_site_files.add(relative_text)
+if observed_site_files != expected_site_files:
+    raise SystemExit(10)
+
+if not native:
+    raise SystemExit(10)
+required_native_roles = {"extension", "libpq", "tls"}
+if sys.platform == "darwin":
+    required_native_roles.update({"kerberos", "ldap"})
+if not required_native_roles.issubset(native_roles):
+    raise SystemExit(10)
+
+print(json.dumps({
+    "installed": installed,
+    "native_machine": platform.machine(),
+    "native_count": len(native),
+    "native_role_count": len(native_roles),
+    "site_files_verified": True,
+    "wheel_count": len(installed),
+}, sort_keys=True, separators=(",", ":")))
+'''
+
+
+IMPORT_PROGRAM = r'''
+import json
+import os
+from pathlib import Path
+import platform
+import stat
+import sys
+
+site_packages = Path(sys.argv[1])
+environment_root = Path(sys.argv[2])
+try:
+    site_status = site_packages.lstat()
+    site_packages = site_packages.resolve(strict=True)
+    environment_root = environment_root.resolve(strict=True)
+    site_packages.relative_to(environment_root)
+except (OSError, RuntimeError, ValueError):
+    raise SystemExit(10)
+if (
+    stat.S_ISLNK(site_status.st_mode)
+    or not stat.S_ISDIR(site_status.st_mode)
+    or site_status.st_uid != os.geteuid()
+    or site_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    or sys.flags.isolated != 1
+    or sys.flags.no_site != 1
+    or sys.flags.dont_write_bytecode != 1
+    or sys.implementation.name != "cpython"
+    or tuple(sys.version_info[:3]) != (3, 14, 7)
+    or sys.platform != "darwin"
+    or platform.machine() != "arm64"
+    or str(site_packages) in sys.path
+):
+    raise SystemExit(10)
+sys.path.insert(0, str(site_packages))
+
+import django
+import gunicorn
+import psycopg
+import psycopg.pq
+import psycopg_binary.pq
+
+if (
+    django.get_version() != "5.2.17"
+    or gunicorn.__version__ != "26.2.0"
+    or psycopg.__version__ != "3.3.4"
+    or psycopg.pq.__impl__ != "binary"
+):
+    raise SystemExit(10)
+for module in (django, gunicorn, psycopg, psycopg_binary.pq):
+    module_file = getattr(module, "__file__", None)
+    if not isinstance(module_file, str):
+        raise SystemExit(10)
+    target = Path(module_file)
+    try:
+        target_status = target.lstat()
+        target.resolve(strict=True).relative_to(site_packages)
+    except (OSError, RuntimeError, ValueError):
+        raise SystemExit(10)
+    if (
+        stat.S_ISLNK(target_status.st_mode)
+        or not stat.S_ISREG(target_status.st_mode)
+        or target_status.st_uid != os.geteuid()
+        or target_status.st_nlink != 1
+        or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        raise SystemExit(10)
+if type(psycopg_binary.pq.__loader__).__name__ != "ExtensionFileLoader":
+    raise SystemExit(10)
+
+print(json.dumps({
+    "imports_verified": True,
+    "native_machine": platform.machine(),
+}, sort_keys=True, separators=(",", ":")))
+'''
+
+
+def verify_inspection_report(report: object) -> None:
+    if not isinstance(report, dict) or set(report) != {
+        "installed",
+        "native_machine",
+        "native_count",
+        "native_role_count",
+        "site_files_verified",
+        "wheel_count",
+    }:
+        fail()
+    installed = report.get("installed")
+    expected = dict(EXPECTED_LOCK_PACKAGES)
+    expected.pop("audience-foundry-travel-readiness")
+    if sys.platform != "win32":
+        expected.pop("tzdata")
+    if installed != expected:
+        fail()
+    native_count = report.get("native_count")
+    native_role_count = report.get("native_role_count")
+    if (
+        report.get("site_files_verified") is not True
+        or report.get("native_machine") != platform.machine()
+        or report.get("wheel_count") != len(expected)
+        or not isinstance(native_count, int)
+        or isinstance(native_count, bool)
+        or native_count < 3
+        or not isinstance(native_role_count, int)
+        or isinstance(native_role_count, bool)
+        or native_role_count < 3
+    ):
+        fail()
+
+
+def verify_import_report(report: object) -> None:
+    if report != {
+        "imports_verified": True,
+        "native_machine": platform.machine(),
+    }:
+        fail()
+
+
+def verify_preinspection_site(
+    environment_root: Path,
+    wheel_attestations: dict[str, dict[str, dict[str, object]]],
+) -> None:
+    site_packages = environment_root / "lib/python3.14/site-packages"
+    try:
+        site_status = site_packages.lstat()
+        site_packages.resolve(strict=True).relative_to(environment_root)
+    except (OSError, RuntimeError, ValueError):
+        fail()
+    if (
+        stat.S_ISLNK(site_status.st_mode)
+        or not stat.S_ISDIR(site_status.st_mode)
+        or site_status.st_uid != os.geteuid()
+        or site_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+    expected_records: dict[str, dict[str, object]] = {}
+    for records in wheel_attestations.values():
+        if not isinstance(records, dict) or not records:
+            fail()
+        dist_info_directories = {
+            PurePosixPath(relative).parent.as_posix()
+            for relative in records
+            if relative.endswith(".dist-info/WHEEL")
+        }
+        if len(dist_info_directories) != 1:
+            fail()
+        for relative, record in records.items():
+            path = PurePosixPath(relative)
+            if (
+                not isinstance(relative, str)
+                or not relative
+                or path.is_absolute()
+                or ".." in path.parts
+                or relative in expected_records
+                or not isinstance(record, dict)
+                or set(record) != {"digest", "size"}
+            ):
+                fail()
+            expected_records[relative] = record
+        dist_info = next(iter(dist_info_directories))
+        generated_records = {
+            f"{dist_info}/INSTALLER": {
+                "digest": hashlib.sha256(b"uv").hexdigest(),
+                "size": 2,
+            },
+            f"{dist_info}/REQUESTED": {
+                "digest": hashlib.sha256(b"").hexdigest(),
+                "size": 0,
+            },
+            f"{dist_info}/RECORD": {"generated": True},
+        }
+        for generated, generated_record in generated_records.items():
+            if generated in expected_records:
+                fail()
+            expected_records[generated] = generated_record
+    for relative, record in EXPECTED_UV_SEED_FILES.items():
+        if relative in expected_records:
+            fail()
+        expected_records[relative] = dict(record)
+
+    observed: set[str] = set()
+    entry_count = 0
+    for target in site_packages.rglob("*"):
+        entry_count += 1
+        if entry_count > 20000:
+            fail()
+        try:
+            target_status = target.lstat()
+            relative = target.relative_to(site_packages).as_posix()
+        except (OSError, ValueError):
+            fail()
+        if (
+            stat.S_ISLNK(target_status.st_mode)
+            or target_status.st_uid != os.geteuid()
+            or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+        ):
+            fail()
+        if stat.S_ISDIR(target_status.st_mode):
+            continue
+        if (
+            not stat.S_ISREG(target_status.st_mode)
+            or target_status.st_nlink != 1
+            or relative in observed
+        ):
+            fail()
+        observed.add(relative)
+        if relative.endswith(".pth") and relative != "_virtualenv.pth":
+            fail()
+    if observed != set(expected_records):
+        fail()
+    for relative, record in expected_records.items():
+        if record.get("generated") is True:
+            continue
+        digest = record.get("digest")
+        size = record.get("size")
+        if (
+            not isinstance(digest, str)
+            or not re.fullmatch(r"[0-9a-f]{64}", digest)
+            or not isinstance(size, int)
+            or isinstance(size, bool)
+            or size < 0
+            or size > MAX_WHEEL_CONTENT_BYTES
+        ):
+            fail()
+        target = site_packages / relative
+        if sha256_regular_file(
+            target, maximum=MAX_WHEEL_CONTENT_BYTES
+        ) != digest or target.lstat().st_size != size:
+            fail()
+
+
+def canonical_json_bytes(value: object) -> bytes:
+    try:
+        return (
+            json.dumps(
+                value,
+                sort_keys=True,
+                separators=(",", ":"),
+                ensure_ascii=False,
+                allow_nan=False,
+            ).encode("utf-8", "strict")
+            + b"\n"
+        )
+    except (TypeError, ValueError, UnicodeError):
+        fail()
+
+
+def environment_entry_type(status: os.stat_result) -> str:
+    if stat.S_ISDIR(status.st_mode):
+        return "directory"
+    if stat.S_ISREG(status.st_mode):
+        return "file"
+    if stat.S_ISLNK(status.st_mode):
+        return "symlink"
+    fail()
+
+
+def expected_console_script(
+    environment_root: bytes, specification: dict[str, str]
+) -> bytes:
+    import_line = specification.get("import")
+    call_text = specification.get("call")
+    if not isinstance(import_line, str) or not isinstance(call_text, str):
+        fail()
+    try:
+        return (
+            b"#!"
+            + environment_root
+            + b"/bin/python\n"
+            + b"# -*- coding: utf-8 -*-\n"
+            + b"import sys\n"
+            + import_line.encode("ascii", "strict")
+            + b"\n"
+            + b'if __name__ == "__main__":\n'
+            + b'    if sys.argv[0].endswith("-script.pyw"):\n'
+            + b"        sys.argv[0] = sys.argv[0][:-11]\n"
+            + b'    elif sys.argv[0].endswith(".exe"):\n'
+            + b"        sys.argv[0] = sys.argv[0][:-4]\n"
+            + b"    sys.exit("
+            + call_text.encode("ascii", "strict")
+            + b")\n"
+        )
+    except UnicodeError:
+        fail()
+
+
+def canonical_record_content(
+    environment_root: Path, relative: str, content: bytes
+) -> bytes:
+    specifications = {
+        str(specification["record_path"]): (script_relative, specification)
+        for script_relative, specification in EXPECTED_CONSOLE_SCRIPTS.items()
+        if specification.get("record") == relative
+    }
+    if not specifications:
+        return content
+    try:
+        rows = list(
+            csv.reader(
+                io.StringIO(content.decode("utf-8", "strict"), newline="")
+            )
+        )
+    except (UnicodeDecodeError, csv.Error):
+        fail()
+    observed: set[str] = set()
+    canonical_rows: list[list[str]] = []
+    environment_bytes = os.fsencode(str(environment_root))
+    for row in rows:
+        if len(row) != 3:
+            fail()
+        record_path, encoded_digest, size_text = row
+        if record_path in observed:
+            fail()
+        observed.add(record_path)
+        selected = specifications.get(record_path)
+        if selected is None:
+            canonical_rows.append(row)
+            continue
+        script_relative, specification = selected
+        raw_script = open_bounded_regular_file(
+            environment_root / script_relative, maximum=64 * 1024
+        )
+        expected_raw = expected_console_script(
+            environment_bytes, specification
+        )
+        if raw_script != expected_raw:
+            fail()
+        raw_encoded = base64.urlsafe_b64encode(
+            hashlib.sha256(raw_script).digest()
+        ).rstrip(b"=").decode("ascii")
+        if (
+            encoded_digest != f"sha256={raw_encoded}"
+            or size_text != str(len(raw_script))
+        ):
+            fail()
+        canonical_script = expected_console_script(
+            CANONICAL_VENV_ROOT, specification
+        )
+        canonical_encoded = base64.urlsafe_b64encode(
+            hashlib.sha256(canonical_script).digest()
+        ).rstrip(b"=").decode("ascii")
+        canonical_rows.append(
+            [
+                record_path,
+                f"sha256={canonical_encoded}",
+                str(len(canonical_script)),
+            ]
+        )
+    if set(specifications) - observed or [row[0] for row in rows] != sorted(
+        observed
+    ):
+        fail()
+    source_buffer = io.StringIO(newline="")
+    canonical_buffer = io.StringIO(newline="")
+    try:
+        csv.writer(source_buffer, lineterminator="\n").writerows(rows)
+        csv.writer(canonical_buffer, lineterminator="\n").writerows(
+            canonical_rows
+        )
+        source_bytes = source_buffer.getvalue().encode("utf-8", "strict")
+        canonical_bytes = canonical_buffer.getvalue().encode(
+            "utf-8", "strict"
+        )
+    except (csv.Error, UnicodeError):
+        fail()
+    if source_bytes != content:
+        fail()
+    return canonical_bytes
+
+
+def canonical_environment_file_content(
+    environment_root: Path,
+    python: Path,
+    relative: str,
+    content: bytes,
+) -> bytes:
+    environment_bytes = os.fsencode(str(environment_root))
+    python_parent_bytes = os.fsencode(str(python.parent))
+    if not environment_bytes or not python_parent_bytes:
+        fail()
+    if relative == "pyvenv.cfg":
+        expected = (
+            b"home = "
+            + python_parent_bytes
+            + b"\nimplementation = CPython\n"
+            + f"uv = {EXPECTED_UV}\n".encode("ascii")
+            + f"version_info = {EXPECTED_PYTHON}\n".encode("ascii")
+            + b"include-system-site-packages = false\n"
+        )
+        if content != expected:
+            fail()
+        return expected.replace(
+            python_parent_bytes, CANONICAL_BOOTSTRAP_PYTHON, 1
+        )
+    activation_digest = EXPECTED_ACTIVATION_FILES.get(relative)
+    if activation_digest is not None:
+        if content.count(environment_bytes) != 1:
+            fail()
+        canonical = content.replace(
+            environment_bytes, CANONICAL_VENV_ROOT, 1
+        )
+        if hashlib.sha256(canonical).hexdigest() != activation_digest:
+            fail()
+        return canonical
+    console_specification = EXPECTED_CONSOLE_SCRIPTS.get(relative)
+    if console_specification is not None:
+        expected = expected_console_script(
+            environment_bytes, console_specification
+        )
+        if content != expected:
+            fail()
+        return expected_console_script(
+            CANONICAL_VENV_ROOT, console_specification
+        )
+    canonical = canonical_record_content(
+        environment_root, relative, content
+    )
+    if environment_bytes in canonical or python_parent_bytes in canonical:
+        fail()
+    return canonical
+
+
+def dependency_environment_inventory_bytes(
+    environment_root: Path, *, python: Path
+) -> bytes:
+    try:
+        root_status = environment_root.lstat()
+        if environment_root.resolve(strict=True) != environment_root:
+            fail()
+        candidates = [environment_root, *environment_root.rglob("*")]
+        candidates.sort(
+            key=lambda path: (
+                "."
+                if path == environment_root
+                else path.relative_to(environment_root).as_posix()
+            )
+        )
+    except (OSError, RuntimeError, ValueError, UnicodeError):
+        fail()
+    if (
+        stat.S_ISLNK(root_status.st_mode)
+        or not stat.S_ISDIR(root_status.st_mode)
+        or len(candidates) < 2
+        or len(candidates) > MAX_ENVIRONMENT_ENTRIES
+    ):
+        fail()
+    inventory: list[dict[str, object]] = []
+    observed_paths: set[str] = set()
+    total_bytes = 0
+    for target in candidates:
+        try:
+            status = target.lstat()
+            relative = (
+                "."
+                if target == environment_root
+                else target.relative_to(environment_root).as_posix()
+            )
+            entry_type = environment_entry_type(status)
+        except (OSError, RuntimeError, ValueError, UnicodeError):
+            fail()
+        if (
+            relative in observed_paths
+            or status.st_uid != os.geteuid()
+            or (
+                entry_type != "symlink"
+                and status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            )
+        ):
+            fail()
+        observed_paths.add(relative)
+        if entry_type == "file":
+            content = open_bounded_regular_file(
+                target, maximum=MAX_ENVIRONMENT_FILE_BYTES
+            )
+            content = canonical_environment_file_content(
+                environment_root, python, relative, content
+            )
+            size = len(content)
+            total_bytes += size
+            digest = hashlib.sha256(content).hexdigest()
+        elif entry_type == "symlink":
+            try:
+                target_bytes = os.fsencode(os.readlink(target))
+            except (OSError, UnicodeError):
+                fail()
+            if relative == "bin/python":
+                if target_bytes != os.fsencode(str(python)):
+                    fail()
+                target_bytes = CANONICAL_BOOTSTRAP_PYTHON
+            elif (
+                os.fsencode(str(environment_root)) in target_bytes
+                or os.fsencode(str(python.parent)) in target_bytes
+            ):
+                fail()
+            size = len(target_bytes)
+            digest = hashlib.sha256(target_bytes).hexdigest()
+        else:
+            children: list[dict[str, str]] = []
+            try:
+                child_paths = sorted(target.iterdir(), key=lambda child: child.name)
+                for child in child_paths:
+                    children.append(
+                        {
+                            "name": child.name,
+                            "type": environment_entry_type(child.lstat()),
+                        }
+                    )
+            except (OSError, RuntimeError, UnicodeError):
+                fail()
+            size = 0
+            digest = hashlib.sha256(canonical_json_bytes(children)).hexdigest()
+        if total_bytes > MAX_ENVIRONMENT_TOTAL_BYTES:
+            fail()
+        inventory.append(
+            {
+                "mode": f"{stat.S_IFMT(status.st_mode) | stat.S_IMODE(status.st_mode):06o}",
+                "path": relative,
+                "sha256": digest,
+                "size": size,
+                "type": entry_type,
+            }
+        )
+    if len(inventory) != len(observed_paths):
+        fail()
+    return canonical_json_bytes(inventory)
+
+
+def dependency_environment_sha256(
+    environment_root: Path, *, python: Path
+) -> str:
+    first = dependency_environment_inventory_bytes(
+        environment_root, python=python
+    )
+    second = dependency_environment_inventory_bytes(
+        environment_root, python=python
+    )
+    if first != second:
+        fail()
+    return hashlib.sha256(first).hexdigest()
+
+
+def write_exclusive_regular_file(path: Path, content: bytes) -> None:
+    flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL
+    if hasattr(os, "O_CLOEXEC"):
+        flags |= os.O_CLOEXEC
+    if hasattr(os, "O_NOFOLLOW"):
+        flags |= os.O_NOFOLLOW
+    descriptor: int | None = None
+    try:
+        descriptor = os.open(path, flags, 0o600)
+        offset = 0
+        while offset < len(content):
+            written = os.write(descriptor, content[offset : offset + 64 * 1024])
+            if written <= 0:
+                fail()
+            offset += written
+        os.fsync(descriptor)
+        status = os.fstat(descriptor)
+    except OSError:
+        fail()
+    finally:
+        if descriptor is not None:
+            try:
+                os.close(descriptor)
+            except OSError:
+                fail()
+    if (
+        not stat.S_ISREG(status.st_mode)
+        or stat.S_IMODE(status.st_mode) != 0o600
+        or status.st_uid != os.geteuid()
+        or status.st_nlink != 1
+        or status.st_size != len(content)
+    ):
+        fail()
+
+
+def verify_wheelhouse(
+    wheelhouse: Path, downloads: dict[str, dict[str, object]]
+) -> None:
+    try:
+        wheelhouse_status = wheelhouse.lstat()
+        entries = tuple(wheelhouse.iterdir())
+    except OSError:
+        fail()
+    expected_filenames = {
+        str(download.get("filename")) for download in downloads.values()
+    }
+    if (
+        stat.S_ISLNK(wheelhouse_status.st_mode)
+        or not stat.S_ISDIR(wheelhouse_status.st_mode)
+        or stat.S_IMODE(wheelhouse_status.st_mode) != 0o700
+        or wheelhouse_status.st_uid != os.geteuid()
+        or {entry.name for entry in entries} != expected_filenames
+    ):
+        fail()
+    for download in downloads.values():
+        filename = download.get("filename")
+        digest = download.get("digest")
+        size = download.get("size")
+        if (
+            not isinstance(filename, str)
+            or PurePosixPath(filename).name != filename
+            or not isinstance(digest, str)
+            or not re.fullmatch(r"[0-9a-f]{64}", digest)
+            or not isinstance(size, int)
+            or isinstance(size, bool)
+            or size <= 0
+            or size > MAX_WHEEL_BYTES
+        ):
+            fail()
+        target = wheelhouse / filename
+        try:
+            target_status = target.lstat()
+        except OSError:
+            fail()
+        if (
+            stat.S_ISLNK(target_status.st_mode)
+            or not stat.S_ISREG(target_status.st_mode)
+            or stat.S_IMODE(target_status.st_mode) != 0o600
+            or target_status.st_uid != os.geteuid()
+            or target_status.st_nlink != 1
+            or target_status.st_size != size
+            or sha256_regular_file(target, maximum=MAX_WHEEL_BYTES) != digest
+        ):
+            fail()
+
+
+def write_private_wheelhouse(
+    workspace: Path, downloads: dict[str, dict[str, object]]
+) -> tuple[Path, dict[str, dict[str, dict[str, object]]]]:
+    expected_filenames = current_platform_policy().get(
+        "lock_wheel_filenames"
+    )
+    if (
+        not isinstance(expected_filenames, dict)
+        or set(downloads) != set(expected_filenames)
+    ):
+        fail()
+    wheelhouse = workspace / "wheelhouse"
+    try:
+        wheelhouse.mkdir(mode=0o700)
+    except OSError:
+        fail()
+    attestations: dict[str, dict[str, dict[str, object]]] = {}
+    total_size = 0
+    for name in sorted(downloads):
+        download = downloads[name]
+        if not isinstance(download, dict) or set(download) != {
+            "body",
+            "digest",
+            "filename",
+            "records",
+            "size",
+        }:
+            fail()
+        body = download.get("body")
+        filename = download.get("filename")
+        digest = download.get("digest")
+        size = download.get("size")
+        records = download.get("records")
+        if (
+            not isinstance(body, bytes)
+            or not isinstance(filename, str)
+            or filename != expected_filenames.get(name)
+            or not isinstance(digest, str)
+            or not isinstance(size, int)
+            or isinstance(size, bool)
+            or len(body) != size
+            or hashlib.sha256(body).hexdigest() != digest
+            or not isinstance(records, dict)
+            or not records
+        ):
+            fail()
+        total_size += size
+        if total_size > MAX_WHEELHOUSE_BYTES:
+            fail()
+        write_exclusive_regular_file(wheelhouse / filename, body)
+        attestations[name] = records
+    verify_wheelhouse(wheelhouse, downloads)
+    return wheelhouse, attestations
+
+
+def claim_environment_directory(environment_root: Path) -> tuple[int, int]:
+    try:
+        environment_root.mkdir(mode=0o700)
+        status = environment_root.lstat()
+    except OSError:
+        fail()
+    if (
+        stat.S_ISLNK(status.st_mode)
+        or not stat.S_ISDIR(status.st_mode)
+        or stat.S_IMODE(status.st_mode) != 0o700
+        or status.st_uid != os.geteuid()
+    ):
+        fail()
+    return status.st_dev, status.st_ino
+
+
+def environment_has_identity(
+    environment_root: Path, expected_identity: tuple[int, int]
+) -> bool:
+    try:
+        status = environment_root.lstat()
+    except OSError:
+        return False
+    return (
+        not stat.S_ISLNK(status.st_mode)
+        and stat.S_ISDIR(status.st_mode)
+        and status.st_uid == os.geteuid()
+        and (status.st_dev, status.st_ino) == expected_identity
+    )
+
+
+def remove_owned_environment(
+    environment_root: Path, expected_identity: tuple[int, int]
+) -> None:
+    if not environment_has_identity(environment_root, expected_identity):
+        fail()
+    try:
+        shutil.rmtree(environment_root)
+    except OSError:
+        fail()
+    if os.path.lexists(environment_root):
+        fail()
+
+
+def harden_uv_environment_lock(environment_root: Path) -> None:
+    lock_path = environment_root / ".lock"
+    try:
+        lock_status = lock_path.lstat()
+    except OSError:
+        fail()
+    if (
+        stat.S_ISLNK(lock_status.st_mode)
+        or not stat.S_ISREG(lock_status.st_mode)
+        or lock_status.st_uid != os.geteuid()
+        or lock_status.st_nlink != 1
+        or stat.S_IMODE(lock_status.st_mode) != 0o666
+        or lock_status.st_size != 0
+    ):
+        fail()
+    try:
+        lock_path.chmod(0o600)
+        hardened_status = lock_path.lstat()
+    except OSError:
+        fail()
+    if (
+        hardened_status.st_dev != lock_status.st_dev
+        or hardened_status.st_ino != lock_status.st_ino
+        or hardened_status.st_uid != lock_status.st_uid
+        or hardened_status.st_nlink != 1
+        or stat.S_IMODE(hardened_status.st_mode) != 0o600
+    ):
+        fail()
+
+
+def parse_uv_cache_metadata(content: bytes) -> None:
+    def unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
+        result: dict[str, object] = {}
+        for key, value in pairs:
+            if not isinstance(key, str) or key in result:
+                fail()
+            result[key] = value
+        return result
+
+    try:
+        document = json.loads(
+            content.decode("utf-8", "strict"), object_pairs_hook=unique_object
+        )
+    except (UnicodeDecodeError, json.JSONDecodeError, RecursionError):
+        fail()
+    if (
+        not isinstance(document, dict)
+        or set(document)
+        != {"commit", "directories", "env", "tags", "timestamp"}
+        or document.get("commit") is not None
+        or document.get("directories") != {}
+        or document.get("env") != {}
+        or document.get("tags") is not None
+    ):
+        fail()
+    timestamp = document.get("timestamp")
+    if not isinstance(timestamp, dict) or set(timestamp) != {
+        "nanos_since_epoch",
+        "secs_since_epoch",
+    }:
+        fail()
+    seconds = timestamp.get("secs_since_epoch")
+    nanoseconds = timestamp.get("nanos_since_epoch")
+    if (
+        not isinstance(seconds, int)
+        or isinstance(seconds, bool)
+        or seconds < 0
+        or not isinstance(nanoseconds, int)
+        or isinstance(nanoseconds, bool)
+        or nanoseconds < 0
+        or nanoseconds >= 1_000_000_000
+    ):
+        fail()
+
+
+def normalize_uv_install_metadata(
+    environment_root: Path,
+    wheel_attestations: dict[str, dict[str, dict[str, object]]],
+) -> None:
+    site_packages = environment_root / "lib/python3.14/site-packages"
+    try:
+        site_status = site_packages.lstat()
+        site_packages.resolve(strict=True).relative_to(environment_root)
+    except (OSError, RuntimeError, ValueError):
+        fail()
+    if (
+        stat.S_ISLNK(site_status.st_mode)
+        or not stat.S_ISDIR(site_status.st_mode)
+        or site_status.st_uid != os.geteuid()
+        or site_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+
+    plans: list[tuple[Path, Path, bytes]] = []
+    for name in sorted(wheel_attestations):
+        records = wheel_attestations[name]
+        if not isinstance(records, dict) or not records:
+            fail()
+        dist_info_directories = {
+            PurePosixPath(relative).parent.as_posix()
+            for relative in records
+            if relative.endswith(".dist-info/WHEEL")
+        }
+        if len(dist_info_directories) != 1:
+            fail()
+        dist_info = next(iter(dist_info_directories))
+        record_relative = f"{dist_info}/RECORD"
+        installer_relative = f"{dist_info}/INSTALLER"
+        requested_relative = f"{dist_info}/REQUESTED"
+        cache_relative = f"{dist_info}/uv_cache.json"
+        expected_paths = (
+            set(records)
+            | {
+                record_relative,
+                installer_relative,
+                requested_relative,
+                cache_relative,
+            }
+            | EXPECTED_GENERATED_SCRIPT_PATHS.get(name, set())
+        )
+
+        record_path = site_packages / record_relative
+        record_content = open_bounded_regular_file(
+            record_path, maximum=4 * 1024 * 1024
+        )
+        try:
+            rows = list(
+                csv.reader(
+                    io.StringIO(
+                        record_content.decode("utf-8", "strict"), newline=""
+                    )
+                )
+            )
+        except (UnicodeDecodeError, csv.Error):
+            fail()
+        if len(rows) != len(expected_paths):
+            fail()
+        observed_paths: set[str] = set()
+        installed_content: dict[str, bytes] = {}
+        for row in rows:
+            if len(row) != 3:
+                fail()
+            relative, encoded_digest, size_text = row
+            relative_path = PurePosixPath(relative)
+            if (
+                not relative
+                or "\\" in relative
+                or relative_path.is_absolute()
+                or relative in observed_paths
+                or relative not in expected_paths
+            ):
+                fail()
+            observed_paths.add(relative)
+            if relative == record_relative:
+                if encoded_digest or size_text:
+                    fail()
+                continue
+            target = site_packages / relative
+            try:
+                target_status = target.lstat()
+                target.resolve(strict=True).relative_to(environment_root)
+            except (OSError, RuntimeError, ValueError):
+                fail()
+            if (
+                stat.S_ISLNK(target_status.st_mode)
+                or not stat.S_ISREG(target_status.st_mode)
+                or target_status.st_uid != os.geteuid()
+                or target_status.st_nlink != 1
+                or target_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            ):
+                fail()
+            content = open_bounded_regular_file(
+                target, maximum=MAX_WHEEL_CONTENT_BYTES
+            )
+            try:
+                expected_size = int(size_text, 10)
+            except ValueError:
+                fail()
+            observed_encoded = base64.urlsafe_b64encode(
+                hashlib.sha256(content).digest()
+            ).rstrip(b"=").decode("ascii")
+            if (
+                expected_size != len(content)
+                or encoded_digest != f"sha256={observed_encoded}"
+            ):
+                fail()
+            installed_content[relative] = content
+        if observed_paths != expected_paths:
+            fail()
+        for relative, expected in records.items():
+            content = installed_content.get(relative)
+            if (
+                not isinstance(expected, dict)
+                or set(expected) != {"digest", "size"}
+                or content is None
+                or expected.get("size") != len(content)
+                or expected.get("digest")
+                != hashlib.sha256(content).hexdigest()
+            ):
+                fail()
+        if (
+            installed_content.get(installer_relative) != b"uv"
+            or installed_content.get(requested_relative) != b""
+        ):
+            fail()
+        cache_path = site_packages / cache_relative
+        parse_uv_cache_metadata(installed_content[cache_relative])
+
+        normalized_rows: list[list[str]] = []
+        for relative in sorted(expected_paths - {cache_relative}):
+            if relative == record_relative:
+                normalized_rows.append([relative, "", ""])
+                continue
+            content = installed_content[relative]
+            encoded = base64.urlsafe_b64encode(
+                hashlib.sha256(content).digest()
+            ).rstrip(b"=").decode("ascii")
+            normalized_rows.append(
+                [relative, f"sha256={encoded}", str(len(content))]
+            )
+        buffer = io.StringIO(newline="")
+        try:
+            writer = csv.writer(buffer, lineterminator="\n")
+            writer.writerows(normalized_rows)
+            normalized_record = buffer.getvalue().encode("utf-8", "strict")
+        except (csv.Error, UnicodeError):
+            fail()
+        plans.append((cache_path, record_path, normalized_record))
+
+    for cache_path, record_path, normalized_record in plans:
+        temporary_record = record_path.with_name("RECORD.normalized")
+        try:
+            cache_path.unlink()
+        except OSError:
+            fail()
+        write_exclusive_regular_file(temporary_record, normalized_record)
+        try:
+            os.replace(temporary_record, record_path)
+        except OSError:
+            fail()
+        if os.path.lexists(cache_path) or os.path.lexists(temporary_record):
+            fail()
+
+
+def verify_environment_symlink_topology(
+    environment_root: Path, python: Path
+) -> None:
+    expected = {
+        "bin/python": str(python),
+        "bin/python3": "python",
+        "bin/python3.14": "python",
+    }
+    observed: dict[str, str] = {}
+    try:
+        candidates = tuple(environment_root.rglob("*"))
+    except (OSError, RuntimeError):
+        fail()
+    for target in candidates:
+        try:
+            status = target.lstat()
+        except OSError:
+            fail()
+        if not stat.S_ISLNK(status.st_mode):
+            continue
+        try:
+            relative = target.relative_to(environment_root).as_posix()
+            link_target = os.readlink(target)
+        except (OSError, ValueError, UnicodeError):
+            fail()
+        if (
+            relative in observed
+            or status.st_uid != os.geteuid()
+            or "\x00" in link_target
+        ):
+            fail()
+        observed[relative] = link_target
+    if observed != expected:
+        fail()
+
+
+def build_and_inspect_environment(
+    source: Path,
+    snapshot: dict[str, bytes],
+    downloaded_wheels: dict[str, dict[str, object]],
+    python: Path,
+    uv: Path,
+    cache: Path,
+    environment_directory: Path,
+) -> tuple[str, tuple[int, int]]:
+    system_temporary = system_temporary_root()
+    try:
+        workspace = Path(
+            tempfile.mkdtemp(prefix="travel-readiness-dependency-gate-", dir=system_temporary)
+        )
+    except OSError:
+        fail()
+    succeeded = False
+    environment_identity: tuple[int, int] | None = None
+    try:
+        if os.path.lexists(environment_directory):
+            fail()
+        if any(
+            paths_overlap(workspace, boundary)
+            for boundary in (
+                source,
+                private_root_for(python),
+                private_root_for(uv),
+                private_root_for(cache),
+                environment_directory,
+            )
+        ):
+            fail()
+        workspace_status = workspace.lstat()
+        if (
+            not stat.S_ISDIR(workspace_status.st_mode)
+            or stat.S_IMODE(workspace_status.st_mode) != 0o700
+            or workspace_status.st_uid != os.geteuid()
+        ):
+            fail()
+        environment_identity = claim_environment_directory(
+            environment_directory
+        )
+        wheelhouse, lock_wheel_attestations = write_private_wheelhouse(
+            workspace, downloaded_wheels
+        )
+        installed_names = set(current_platform_policy()["wheel_tags"])
+        if not installed_names.issubset(lock_wheel_attestations):
+            fail()
+        wheel_attestations = {
+            name: lock_wheel_attestations[name]
+            for name in sorted(installed_names)
+        }
+        project = workspace / "project"
+        project.mkdir(mode=0o700)
+        try:
+            attestation_bytes = json.dumps(
+                wheel_attestations,
+                sort_keys=True,
+                separators=(",", ":"),
+            ).encode("utf-8", "strict")
+        except (TypeError, UnicodeError):
+            fail()
+        if len(attestation_bytes) < 2 or len(attestation_bytes) > 8 * 1024 * 1024:
+            fail()
+        attestation_path = project / "wheel-attestation.json"
+        write_exclusive_regular_file(attestation_path, attestation_bytes)
+        copied_policy_files: dict[str, bytes] = {}
+        for relative in ("pyproject.toml", "uv.lock"):
+            destination = project / relative
+            copied_policy_files[relative] = snapshot_file(
+                snapshot, relative, maximum=8 * 1024 * 1024
+            )
+            write_exclusive_regular_file(
+                destination, copied_policy_files[relative]
+            )
+        locked = parse_lock(snapshot)
+        requirement_lines: list[str] = []
+        for name in sorted(installed_names):
+            version = locked.get(name)
+            download = downloaded_wheels.get(name)
+            digest = download.get("digest") if isinstance(download, dict) else None
+            if (
+                version != EXPECTED_LOCK_PACKAGES.get(name)
+                or not isinstance(digest, str)
+                or not re.fullmatch(r"[0-9a-f]{64}", digest)
+            ):
+                fail()
+            requirement_lines.append(
+                f"{name}=={version} --hash=sha256:{digest}\n"
+            )
+        try:
+            requirements_bytes = "".join(requirement_lines).encode(
+                "ascii", "strict"
+            )
+        except UnicodeError:
+            fail()
+        requirements_path = project / "requirements.txt"
+        write_exclusive_regular_file(requirements_path, requirements_bytes)
+        environment = fixed_child_environment(workspace, cache, python)
+        environment["UV_PROJECT_ENVIRONMENT"] = str(environment_directory)
+        exact_python_version(python, cwd=workspace, environment=environment)
+        exact_uv_version(uv, cwd=workspace, environment=environment)
+        run_fixed(
+            [
+                str(uv),
+                "lock",
+                "--check",
+                "--offline",
+                "--no-cache",
+                "--project",
+                str(project),
+                "--cache-dir",
+                str(cache),
+                "--python",
+                str(python),
+                "--no-python-downloads",
+            ],
+            cwd=project,
+            environment=environment,
+        )
+        if not directory_is_empty(cache):
+            fail()
+        run_fixed(
+            [
+                str(uv),
+                "venv",
+                str(environment_directory),
+                "--allow-existing",
+                "--no-project",
+                "--offline",
+                "--no-cache",
+                "--cache-dir",
+                str(cache),
+                "--python",
+                str(python),
+                "--no-python-downloads",
+                "--link-mode",
+                "copy",
+            ],
+            cwd=project,
+            environment=environment,
+        )
+        if not directory_is_empty(cache):
+            fail()
+        run_fixed(
+            [
+                str(uv),
+                "pip",
+                "install",
+                "--requirements",
+                str(requirements_path),
+                "--offline",
+                "--no-cache",
+                "--no-index",
+                "--find-links",
+                str(wheelhouse),
+                "--only-binary",
+                ":all:",
+                "--no-deps",
+                "--require-hashes",
+                "--exact",
+                "--strict",
+                "--cache-dir",
+                str(cache),
+                "--python",
+                str(environment_directory / "bin/python"),
+                "--no-python-downloads",
+                "--link-mode",
+                "copy",
+            ],
+            cwd=project,
+            environment=environment,
+        )
+        if not directory_is_empty(cache):
+            fail()
+        if not environment_has_identity(
+            environment_directory, environment_identity
+        ):
+            fail()
+        # Exact uv 0.12.6 creates this zero-byte lock at 0666 inside the
+        # invocation-owned 0700 root; harden it before inspection or import.
+        harden_uv_environment_lock(environment_directory)
+        normalize_uv_install_metadata(
+            environment_directory, wheel_attestations
+        )
+        verify_environment_symlink_topology(environment_directory, python)
+        try:
+            environment_status = environment_directory.lstat()
+        except OSError:
+            fail()
+        if (
+            stat.S_ISLNK(environment_status.st_mode)
+            or not stat.S_ISDIR(environment_status.st_mode)
+            or environment_status.st_uid != os.geteuid()
+            or environment_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+            or (environment_status.st_dev, environment_status.st_ino)
+            != environment_identity
+        ):
+            fail()
+        # Do not execute the new environment (including through a normal
+        # interpreter startup that could process a .pth file) until every
+        # installed byte is bound to the selected official wheels or to the
+        # exact uv virtual-environment seed policy.
+        verify_preinspection_site(environment_directory, wheel_attestations)
+        preimport_environment_digest = dependency_environment_sha256(
+            environment_directory, python=python
+        )
+        venv_python = environment_directory / (
+            "Scripts/python.exe" if os.name == "nt" else "bin/python"
+        )
+        try:
+            venv_python_status = venv_python.lstat()
+            venv_python_target = os.readlink(venv_python)
+        except OSError:
+            fail()
+        if (
+            not stat.S_ISLNK(venv_python_status.st_mode)
+            or venv_python_target != str(python)
+            or venv_python.resolve(strict=True) != python
+        ):
+            fail()
+        inspection_arguments = [
+            str(python),
+            "-I",
+            "-S",
+            "-B",
+            "-c",
+            INSPECTION_PROGRAM,
+            json.dumps(EXPECTED_LICENSES, sort_keys=True),
+            json.dumps(
+                {key: sorted(value) for key, value in EXPECTED_LICENSE_FILES.items()},
+                sort_keys=True,
+            ),
+            sys.platform,
+            platform.machine(),
+            json.dumps(current_platform_policy()["wheel_tags"], sort_keys=True),
+            str(attestation_path),
+            str(environment_directory),
+        ]
+        raw_report = run_fixed(
+            inspection_arguments,
+            cwd=project,
+            environment=environment,
+        )
+        if len(raw_report) > 64 * 1024:
+            fail()
+        try:
+            report = json.loads(raw_report.decode("utf-8", "strict"))
+        except (UnicodeDecodeError, json.JSONDecodeError):
+            fail()
+        verify_inspection_report(report)
+        site_packages = environment_directory / "lib/python3.14/site-packages"
+        raw_import_report = run_fixed(
+            [
+                str(python),
+                "-I",
+                "-S",
+                "-B",
+                "-c",
+                IMPORT_PROGRAM,
+                str(site_packages),
+                str(environment_directory),
+            ],
+            cwd=project,
+            environment=environment,
+        )
+        if len(raw_import_report) > 64 * 1024:
+            fail()
+        try:
+            import_report = json.loads(raw_import_report.decode("utf-8", "strict"))
+        except (UnicodeDecodeError, json.JSONDecodeError):
+            fail()
+        verify_import_report(import_report)
+        if run_fixed(
+            inspection_arguments,
+            cwd=project,
+            environment=environment,
+        ) != raw_report:
+            fail()
+        if (
+            stat.S_IMODE(attestation_path.lstat().st_mode) != 0o600
+            or open_bounded_regular_file(
+                attestation_path, maximum=8 * 1024 * 1024
+            )
+            != attestation_bytes
+        ):
+            fail()
+        for relative, expected_content in copied_policy_files.items():
+            destination = project / relative
+            if (
+                stat.S_IMODE(destination.lstat().st_mode) != 0o600
+                or open_bounded_regular_file(
+                    destination, maximum=8 * 1024 * 1024
+                )
+                != expected_content
+            ):
+                fail()
+        if (
+            stat.S_IMODE(requirements_path.lstat().st_mode) != 0o600
+            or open_bounded_regular_file(
+                requirements_path, maximum=64 * 1024
+            )
+            != requirements_bytes
+        ):
+            fail()
+        verify_wheelhouse(wheelhouse, downloaded_wheels)
+        if not directory_is_empty(cache):
+            fail()
+        environment_digest = dependency_environment_sha256(
+            environment_directory, python=python
+        )
+        if environment_digest != preimport_environment_digest:
+            fail()
+        succeeded = True
+        return environment_digest, environment_identity
+    finally:
+        workspace_removed = True
+        try:
+            shutil.rmtree(workspace)
+        except OSError:
+            workspace_removed = False
+        if workspace.exists():
+            workspace_removed = False
+        if not workspace_removed:
+            succeeded = False
+        if (
+            not succeeded
+            and environment_identity is not None
+            and os.path.lexists(environment_directory)
+        ):
+            remove_owned_environment(
+                environment_directory, environment_identity
+            )
+        if not workspace_removed:
+            fail()
+
+
+def validate_pypi_url(url: str, expected_path: str) -> None:
+    try:
+        parsed = urllib.parse.urlsplit(url)
+    except ValueError:
+        fail()
+    if (
+        parsed.scheme != "https"
+        or parsed.hostname != "pypi.org"
+        or parsed.username is not None
+        or parsed.password is not None
+        or parsed.port not in (None, 443)
+        or parsed.path != expected_path
+        or parsed.query
+        or parsed.fragment
+    ):
+        fail()
+
+
+class SameOfficialOriginRedirect(urllib.request.HTTPRedirectHandler):
+    def redirect_request(self, request, fp, code, message, headers, new_url):
+        expected_path = getattr(request, "dependency_gate_expected_path", None)
+        if not isinstance(expected_path, str):
+            fail()
+        validate_pypi_url(new_url, expected_path)
+        redirected = super().redirect_request(
+            request, fp, code, message, headers, new_url
+        )
+        if redirected is None:
+            fail()
+        setattr(redirected, "dependency_gate_expected_path", expected_path)
+        return redirected
+
+
+class RejectRedirect(urllib.request.HTTPRedirectHandler):
+    def redirect_request(self, request, fp, code, message, headers, new_url):
+        fail()
+
+
+def controlled_ssl_context(ca_file: Path) -> ssl.SSLContext:
+    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+    context.check_hostname = True
+    context.verify_mode = ssl.CERT_REQUIRED
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    try:
+        context.load_verify_locations(cafile=str(ca_file))
+    except (OSError, ssl.SSLError):
+        fail()
+    return context
+
+
+def official_opener(ca_file: Path) -> urllib.request.OpenerDirector:
+    return urllib.request.build_opener(
+        urllib.request.ProxyHandler({}),
+        SameOfficialOriginRedirect(),
+        urllib.request.HTTPSHandler(context=controlled_ssl_context(ca_file)),
+    )
+
+
+def official_artifact_opener(ca_file: Path) -> urllib.request.OpenerDirector:
+    return urllib.request.build_opener(
+        urllib.request.ProxyHandler({}),
+        RejectRedirect(),
+        urllib.request.HTTPSHandler(context=controlled_ssl_context(ca_file)),
+    )
+
+
+def validate_artifact_url(url: str) -> None:
+    try:
+        parsed = urllib.parse.urlsplit(url)
+    except ValueError:
+        fail()
+    parts = PurePosixPath(parsed.path).parts
+    if (
+        parsed.scheme != "https"
+        or parsed.hostname != "files.pythonhosted.org"
+        or parsed.username is not None
+        or parsed.password is not None
+        or parsed.port not in (None, 443)
+        or parsed.query
+        or parsed.fragment
+        or len(parts) != 6
+        or parts[1] != "packages"
+        or not parts[-1].endswith(".whl")
+        or urllib.parse.unquote(parts[-1]) != parts[-1]
+    ):
+        fail()
+
+
+def bounded_network_timeout(deadline: float | None) -> float:
+    if deadline is None:
+        return float(NETWORK_TIMEOUT_SECONDS)
+    remaining = deadline - time.monotonic()
+    if remaining <= 0:
+        fail()
+    return min(float(NETWORK_TIMEOUT_SECONDS), remaining)
+
+
+def require_network_deadline(deadline: float | None) -> None:
+    if deadline is not None and time.monotonic() > deadline:
+        fail()
+
+
+def begin_network_alarm(deadline: float | None) -> object | None:
+    if deadline is None:
+        return None
+    if not hasattr(signal, "setitimer") or not hasattr(signal, "getitimer"):
+        fail()
+    remaining = deadline - time.monotonic()
+    if remaining <= 0:
+        fail()
+    try:
+        previous_handler = signal.getsignal(signal.SIGALRM)
+        previous_timer = signal.getitimer(signal.ITIMER_REAL)
+    except (OSError, ValueError):
+        fail()
+    if previous_handler != signal.SIG_DFL or previous_timer != (0.0, 0.0):
+        fail()
+
+    def expired(_signum, _frame):
+        fail()
+
+    try:
+        signal.signal(signal.SIGALRM, expired)
+        signal.setitimer(signal.ITIMER_REAL, remaining)
+    except (OSError, ValueError):
+        fail()
+    return previous_handler
+
+
+def end_network_alarm(previous_handler: object | None) -> None:
+    if previous_handler is None:
+        return
+    failed = False
+    try:
+        signal.setitimer(signal.ITIMER_REAL, 0.0)
+        signal.signal(signal.SIGALRM, previous_handler)
+    except (OSError, ValueError):
+        failed = True
+    if failed:
+        fail()
+
+
+def response_header(response: object, name: str) -> str | None:
+    headers = getattr(response, "headers", None)
+    if headers is None:
+        fail()
+    if isinstance(headers, email.message.Message):
+        values = headers.get_all(name, [])
+        if len(values) > 1:
+            fail()
+        return values[0] if values else None
+    getter = getattr(headers, "get", None)
+    if not callable(getter):
+        fail()
+    value = getter(name)
+    if value is not None and not isinstance(value, str):
+        fail()
+    return value
+
+
+def validate_response_headers(response: object) -> None:
+    headers = getattr(response, "headers", None)
+    items = getattr(headers, "items", None)
+    if not callable(items):
+        fail()
+    total = 0
+    try:
+        pairs = list(items())
+    except (TypeError, ValueError):
+        fail()
+    for name, value in pairs:
+        if (
+            not isinstance(name, str)
+            or not isinstance(value, str)
+            or not re.fullmatch(r"[!-9;-~]+", name)
+            or "\r" in value
+            or "\n" in value
+        ):
+            fail()
+        try:
+            total += len(name.encode("ascii", "strict")) + len(
+                value.encode("latin-1", "strict")
+            ) + 4
+        except UnicodeEncodeError:
+            fail()
+        if total > MAX_RESPONSE_HEADERS_BYTES:
+            fail()
+
+
+def artifact_is_relevant_to_exact_lock(
+    name: str, filename: str, packagetype: str
+) -> bool:
+    if name == "psycopg-binary":
+        return packagetype == "bdist_wheel" and "-cp314-cp314-" in filename
+    return True
+
+
+def attest_exact_wheel(
+    body: bytes, expected_artifact: dict[str, object]
+) -> dict[str, dict[str, object]]:
+    expected_digest = expected_artifact.get("digest")
+    expected_size = expected_artifact.get("size")
+    if (
+        not isinstance(expected_digest, str)
+        or not re.fullmatch(r"[0-9a-f]{64}", expected_digest)
+        or not isinstance(expected_size, int)
+        or isinstance(expected_size, bool)
+        or len(body) != expected_size
+        or len(body) > MAX_WHEEL_BYTES
+        or hashlib.sha256(body).hexdigest() != expected_digest
+    ):
+        fail()
+    try:
+        archive = zipfile.ZipFile(io.BytesIO(body), mode="r")
+        entries = archive.infolist()
+    except (OSError, ValueError, zipfile.BadZipFile):
+        fail()
+    if not entries or len(entries) > 10000:
+        fail()
+    contents: dict[str, bytes] = {}
+    seen_names: set[str] = set()
+    total_content = 0
+    record_names: list[str] = []
+    try:
+        for entry in entries:
+            filename = entry.filename
+            relative = PurePosixPath(filename)
+            unix_mode = (entry.external_attr >> 16) & 0xFFFF
+            file_type = stat.S_IFMT(unix_mode)
+            if (
+                not filename
+                or "\\" in filename
+                or "\x00" in filename
+                or relative.is_absolute()
+                or ".." in relative.parts
+                or filename in seen_names
+                or entry.flag_bits & 0x1
+                or entry.compress_type
+                not in {zipfile.ZIP_STORED, zipfile.ZIP_DEFLATED}
+                or entry.file_size < 0
+                or entry.file_size > MAX_WHEEL_CONTENT_BYTES
+                or entry.compress_size < 0
+                or file_type not in {0, stat.S_IFREG, stat.S_IFDIR}
+            ):
+                fail()
+            seen_names.add(filename)
+            if entry.is_dir():
+                if file_type not in {0, stat.S_IFDIR}:
+                    fail()
+                continue
+            if file_type == stat.S_IFDIR:
+                fail()
+            total_content += entry.file_size
+            if total_content > MAX_WHEEL_CONTENT_BYTES:
+                fail()
+            content = archive.read(entry)
+            if len(content) != entry.file_size:
+                fail()
+            contents[filename] = content
+            if filename.endswith(".dist-info/RECORD"):
+                record_names.append(filename)
+    except (OSError, RuntimeError, zipfile.BadZipFile, zipfile.LargeZipFile):
+        fail()
+    finally:
+        archive.close()
+    if len(record_names) != 1:
+        fail()
+    record_name = record_names[0]
+    try:
+        rows = list(
+            csv.reader(
+                io.StringIO(contents[record_name].decode("utf-8", "strict"), newline="")
+            )
+        )
+    except (UnicodeDecodeError, csv.Error):
+        fail()
+    attestation: dict[str, dict[str, object]] = {}
+    recorded_names: set[str] = set()
+    for row in rows:
+        if len(row) != 3:
+            fail()
+        filename, encoded_digest, size_text = row
+        if filename in recorded_names or filename not in contents:
+            fail()
+        recorded_names.add(filename)
+        if filename == record_name:
+            if encoded_digest or size_text:
+                fail()
+            continue
+        content = contents[filename]
+        try:
+            recorded_size = int(size_text, 10)
+        except ValueError:
+            fail()
+        observed_encoded = base64.urlsafe_b64encode(
+            hashlib.sha256(content).digest()
+        ).rstrip(b"=").decode("ascii")
+        if (
+            recorded_size != len(content)
+            or encoded_digest != f"sha256={observed_encoded}"
+        ):
+            fail()
+        attestation[filename] = {
+            "digest": hashlib.sha256(content).hexdigest(),
+            "size": len(content),
+        }
+    if recorded_names != set(contents):
+        fail()
+    return attestation
+
+
+def fetch_exact_wheel(
+    expected_artifact: dict[str, object], *, opener: object,
+    deadline: float | None = None,
+) -> dict[str, object]:
+    url = expected_artifact.get("url")
+    expected_size = expected_artifact.get("size")
+    if (
+        expected_artifact.get("packagetype") != "bdist_wheel"
+        or not isinstance(url, str)
+        or not isinstance(expected_size, int)
+        or isinstance(expected_size, bool)
+        or expected_size <= 0
+        or expected_size > MAX_WHEEL_BYTES
+    ):
+        fail()
+    validate_artifact_url(url)
+    request = urllib.request.Request(
+        url,
+        headers={
+            "Accept": "application/octet-stream",
+            "Accept-Encoding": "identity",
+            "User-Agent": "travel-readiness-dependency-gate/1",
+        },
+        method="GET",
+    )
+    alarm = begin_network_alarm(deadline)
+    try:
+        response = opener.open(request, timeout=bounded_network_timeout(deadline))
+        try:
+            status = getattr(response, "status", None)
+            final_url_getter = getattr(response, "geturl", None)
+            final_url = final_url_getter() if callable(final_url_getter) else None
+            if status != 200 or final_url != url:
+                fail()
+            validate_response_headers(response)
+            content_type = response_header(response, "Content-Type")
+            if (
+                not isinstance(content_type, str)
+                or content_type.split(";", 1)[0].strip().lower()
+                not in {
+                    "application/octet-stream",
+                    "application/zip",
+                    "binary/octet-stream",
+                }
+            ):
+                fail()
+            if response_header(response, "Content-Encoding") not in (
+                None,
+                "",
+                "identity",
+            ):
+                fail()
+            content_length = response_header(response, "Content-Length")
+            try:
+                declared_length = int(content_length or "", 10)
+            except ValueError:
+                fail()
+            if declared_length != expected_size:
+                fail()
+            body = response.read(MAX_WHEEL_BYTES + 1)
+            if not isinstance(body, bytes) or len(body) != expected_size:
+                fail()
+            require_network_deadline(deadline)
+        finally:
+            close = getattr(response, "close", None)
+            if callable(close):
+                close()
+    except GateError:
+        raise
+    except (OSError, TimeoutError, urllib.error.URLError, urllib.error.HTTPError):
+        fail()
+    finally:
+        end_network_alarm(alarm)
+    filename = PurePosixPath(urllib.parse.urlsplit(url).path).name
+    return {
+        "body": body,
+        "digest": expected_artifact["digest"],
+        "filename": filename,
+        "records": attest_exact_wheel(body, expected_artifact),
+        "size": expected_size,
+    }
+
+
+def fetch_selected_wheels(
+    artifacts: dict[str, dict[str, dict[str, object]]], *, opener: object,
+    deadline: float | None = None,
+) -> dict[str, dict[str, object]]:
+    filenames = current_platform_policy().get("lock_wheel_filenames")
+    if not isinstance(filenames, dict):
+        fail()
+    downloads: dict[str, dict[str, object]] = {}
+    for name in sorted(filenames):
+        require_network_deadline(deadline)
+        filename = filenames[name]
+        expected_artifact = artifacts.get(name, {}).get(str(filename))
+        if not isinstance(expected_artifact, dict):
+            fail()
+        download = fetch_exact_wheel(
+            expected_artifact, opener=opener, deadline=deadline
+        )
+        if download.get("filename") != filename:
+            fail()
+        downloads[name] = download
+    if set(downloads) != set(filenames):
+        fail()
+    return downloads
+
+
+def check_one_advisory(
+    name: str,
+    version: str,
+    *,
+    expected_artifacts: dict[str, dict[str, object]],
+    opener: object,
+    deadline: float | None = None,
+) -> None:
+    quoted_name = urllib.parse.quote(name, safe="")
+    quoted_version = urllib.parse.quote(version, safe="")
+    expected_path = f"/pypi/{quoted_name}/{quoted_version}/json"
+    url = f"{PYPI_ORIGIN}{expected_path}"
+    request = urllib.request.Request(
+        url,
+        headers={
+            "Accept": "application/json",
+            "Accept-Encoding": "identity",
+            "User-Agent": "travel-readiness-dependency-gate/1",
+        },
+        method="GET",
+    )
+    setattr(request, "dependency_gate_expected_path", expected_path)
+    alarm = begin_network_alarm(deadline)
+    try:
+        response = opener.open(request, timeout=bounded_network_timeout(deadline))
+        try:
+            status = getattr(response, "status", None)
+            if status is None:
+                getter = getattr(response, "getcode", None)
+                status = getter() if callable(getter) else None
+            final_url_getter = getattr(response, "geturl", None)
+            final_url = final_url_getter() if callable(final_url_getter) else None
+            if status != 200 or not isinstance(final_url, str):
+                fail()
+            validate_pypi_url(final_url, expected_path)
+            validate_response_headers(response)
+            content_type = response_header(response, "Content-Type")
+            if (
+                not isinstance(content_type, str)
+                or content_type.split(";", 1)[0].strip().lower() != "application/json"
+            ):
+                fail()
+            content_encoding = response_header(response, "Content-Encoding")
+            if content_encoding not in (None, "", "identity"):
+                fail()
+            content_length = response_header(response, "Content-Length")
+            if content_length is None:
+                fail()
+            try:
+                declared_length = int(content_length, 10)
+            except ValueError:
+                fail()
+            if declared_length < 0 or declared_length > MAX_METADATA_BYTES:
+                fail()
+            body = response.read(MAX_METADATA_BYTES + 1)
+            if not isinstance(body, bytes) or len(body) > MAX_METADATA_BYTES:
+                fail()
+            if declared_length != len(body):
+                fail()
+            require_network_deadline(deadline)
+        finally:
+            close = getattr(response, "close", None)
+            if callable(close):
+                close()
+    except GateError:
+        raise
+    except (OSError, TimeoutError, urllib.error.URLError, urllib.error.HTTPError):
+        fail()
+    finally:
+        end_network_alarm(alarm)
+    try:
+        document = json.loads(body.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, json.JSONDecodeError, RecursionError):
+        fail()
+    if not isinstance(document, dict):
+        fail()
+    info = document.get("info")
+    urls = document.get("urls")
+    vulnerabilities = document.get("vulnerabilities")
+    last_serial = document.get("last_serial")
+    if (
+        not isinstance(info, dict)
+        or not isinstance(urls, list)
+        or not urls
+        or not isinstance(vulnerabilities, list)
+        or vulnerabilities
+        or not isinstance(last_serial, int)
+        or isinstance(last_serial, bool)
+        or last_serial <= 0
+        or canonical_name(info.get("name")) != name
+        or info.get("version") != version
+        or not isinstance(info.get("yanked"), bool)
+    ):
+        fail()
+    info_yanked_reason = info.get("yanked_reason")
+    if info_yanked_reason is not None and not isinstance(info_yanked_reason, str):
+        fail()
+    if info["yanked"]:
+        fail()
+    observed_artifacts: dict[str, dict[str, object]] = {}
+    for artifact in urls:
+        if not isinstance(artifact, dict):
+            fail()
+        filename = artifact.get("filename")
+        digests = artifact.get("digests")
+        packagetype = artifact.get("packagetype")
+        size = artifact.get("size")
+        artifact_url = artifact.get("url")
+        reason = artifact.get("yanked_reason")
+        requires_python = artifact.get("requires_python")
+        python_version = artifact.get("python_version")
+        upload_time = artifact.get("upload_time_iso_8601")
+        if (
+            not isinstance(filename, str)
+            or not filename
+            or PurePosixPath(filename).name != filename
+            or filename in observed_artifacts
+            or not isinstance(digests, dict)
+            or not isinstance(digests.get("sha256"), str)
+            or not re.fullmatch(r"[0-9a-f]{64}", digests["sha256"])
+            or packagetype not in {"bdist_wheel", "sdist"}
+            or not isinstance(size, int)
+            or isinstance(size, bool)
+            or size <= 0
+            or not isinstance(artifact_url, str)
+            or not isinstance(artifact.get("yanked"), bool)
+            or (reason is not None and not isinstance(reason, str))
+            or (requires_python is not None and not isinstance(requires_python, str))
+            or not isinstance(python_version, str)
+            or not isinstance(upload_time, str)
+            or not re.fullmatch(
+                r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z",
+                upload_time,
+            )
+        ):
+            fail()
+        if artifact["yanked"]:
+            fail()
+        if artifact_is_relevant_to_exact_lock(name, filename, packagetype):
+            observed_artifacts[filename] = {
+                "digest": digests["sha256"],
+                "packagetype": packagetype,
+                "size": size,
+                "url": artifact_url,
+            }
+    if observed_artifacts != expected_artifacts:
+        fail()
+
+
+def check_advisories(
+    locked: dict[str, str],
+    artifacts: dict[str, dict[str, dict[str, object]]],
+    *,
+    ca_file: Path | None = None,
+    opener: object | None = None,
+    deadline: float | None = None,
+) -> None:
+    registry = {
+        name: version
+        for name, version in locked.items()
+        if name != "audience-foundry-travel-readiness"
+    }
+    if registry != {
+        key: value
+        for key, value in EXPECTED_LOCK_PACKAGES.items()
+        if key != "audience-foundry-travel-readiness"
+    }:
+        fail()
+    if set(artifacts) != set(locked):
+        fail()
+    if opener is None:
+        if ca_file is None:
+            fail()
+        active_opener = official_opener(ca_file)
+    else:
+        active_opener = opener
+    for name in sorted(registry):
+        require_network_deadline(deadline)
+        expected_artifacts = artifacts.get(name)
+        if not isinstance(expected_artifacts, dict) or not expected_artifacts:
+            fail()
+        check_one_advisory(
+            name,
+            registry[name],
+            expected_artifacts=expected_artifacts,
+            opener=active_opener,
+            deadline=deadline,
+        )
+
+
+def parse_arguments(arguments: list[str]) -> dict[str, str]:
+    if len(arguments) != 12:
+        fail()
+    allowed = {
+        "--source-dir",
+        "--python-bin",
+        "--uv-bin",
+        "--cache-dir",
+        "--ca-file",
+        "--environment-dir",
+    }
+    parsed: dict[str, str] = {}
+    for index in range(0, len(arguments), 2):
+        option = arguments[index]
+        value = arguments[index + 1]
+        if option not in allowed or option in parsed or not value:
+            fail()
+        parsed[option] = value
+    if set(parsed) != allowed:
+        fail()
+    return parsed
+
+
+def run_gate(arguments: list[str]) -> tuple[str, str]:
+    enforce_gate_runner_flags()
+    parsed = parse_arguments(arguments)
+    enforce_gate_runner(parsed["--python-bin"])
+    source = safe_absolute_path(parsed["--source-dir"], kind="directory")
+    python = safe_absolute_path(parsed["--python-bin"], kind="executable")
+    uv = safe_absolute_path(parsed["--uv-bin"], kind="executable")
+    cache = safe_absolute_path(parsed["--cache-dir"], kind="directory")
+    ca_file = safe_absolute_path(parsed["--ca-file"], kind="file")
+    environment = safe_new_environment_path(
+        parsed["--environment-dir"], source=source
+    )
+    verify_input_boundaries(source, python, uv, cache, ca_file, environment)
+    environment_identity: tuple[int, int] | None = None
+    downloaded_wheels: dict[str, dict[str, object]] = {}
+    succeeded = False
+    try:
+        snapshot = capture_policy_snapshot(source)
+        parse_runtime(snapshot)
+        parse_project(snapshot)
+        locked, artifacts = parse_lock_contract(snapshot)
+        verify_notices(snapshot)
+        verify_tool_provenance(snapshot, python, uv)
+        verify_ca_provenance(ca_file)
+        network_deadline = time.monotonic() + NETWORK_TOTAL_SECONDS
+        check_advisories(
+            locked, artifacts, ca_file=ca_file, deadline=network_deadline
+        )
+        downloaded_wheels = fetch_selected_wheels(
+            artifacts,
+            opener=official_artifact_opener(ca_file),
+            deadline=network_deadline,
+        )
+        environment_digest, environment_identity = build_and_inspect_environment(
+            source,
+            snapshot,
+            downloaded_wheels,
+            python,
+            uv,
+            cache,
+            environment,
+        )
+        if environment_digest != current_platform_policy().get(
+            "dependency_environment_sha256"
+        ):
+            fail()
+        verify_policy_snapshot_unchanged(source, snapshot)
+        if dependency_environment_sha256(
+            environment, python=python
+        ) != environment_digest:
+            fail()
+        verify_tool_provenance(snapshot, python, uv)
+        verify_policy_snapshot_unchanged(source, snapshot)
+        bootstrap_python_tree_digest = current_platform_policy().get(
+            "python_tree_sha256"
+        )
+        if (
+            not isinstance(bootstrap_python_tree_digest, str)
+            or not re.fullmatch(
+                r"[0-9a-f]{64}", bootstrap_python_tree_digest
+            )
+        ):
+            fail()
+        succeeded = True
+        return environment_digest, bootstrap_python_tree_digest
+    finally:
+        for download in downloaded_wheels.values():
+            if isinstance(download, dict) and "body" in download:
+                download["body"] = b""
+        downloaded_wheels.clear()
+        if (
+            environment_identity is not None
+            and not succeeded
+            and os.path.lexists(environment)
+        ):
+            remove_owned_environment(environment, environment_identity)
+
+
+def main() -> int:
+    try:
+        environment_digest, bootstrap_python_tree_digest = run_gate(
+            sys.argv[1:]
+        )
+    except SystemExit as exc:
+        return int(exc.code or 0)
+    except Exception:
+        print(FAILURE_RECEIPT, file=sys.stderr)
+        return 1
+    print(
+        f"{SUCCESS_RECEIPT} dependency_environment_sha256={environment_digest} "
+        f"bootstrap_python_tree_sha256={bootstrap_python_tree_digest}"
+    )
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


