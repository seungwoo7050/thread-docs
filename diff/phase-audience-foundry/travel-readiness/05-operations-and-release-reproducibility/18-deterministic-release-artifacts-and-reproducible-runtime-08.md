## `feat(release): bind runtime bootstrap identity`

diff --git a/operations/gunicorn_logging.py b/operations/gunicorn_logging.py
index 8c8811f..362ad53 100644
--- a/operations/gunicorn_logging.py
+++ b/operations/gunicorn_logging.py
@@ -4,16 +4,32 @@ from __future__ import annotations
 
 import json
 import logging
+import os
 
 from gunicorn.glogging import Logger
 
+from .telemetry import bounded_release_sha, utc_timestamp
+
 
 class _RedactedGunicornFilter(logging.Filter):
     def filter(self, record: logging.LogRecord) -> bool:
+        level = record.levelname
+        if type(level) is not str or level not in {
+            "DEBUG",
+            "INFO",
+            "WARNING",
+            "ERROR",
+            "CRITICAL",
+        }:
+            level = "OTHER"
         record.msg = json.dumps(
             {
                 "event": "gunicorn",
-                "level": record.levelname,
+                "timestamp": utc_timestamp(),
+                "release_sha": bounded_release_sha(
+                    os.environ.get("TRAVEL_READINESS_RELEASE_SHA")
+                ),
+                "level": level,
                 "redacted": True,
             },
             separators=(",", ":"),
diff --git a/operations/telemetry.py b/operations/telemetry.py
new file mode 100644
index 0000000..bffeaff
--- /dev/null
+++ b/operations/telemetry.py
@@ -0,0 +1,25 @@
+"""Bounded release identity and UTC timestamps for fixed-shape telemetry."""
+
+from __future__ import annotations
+
+from datetime import UTC, datetime
+import re
+
+
+_RELEASE_SHA = re.compile(r"[0-9a-f]{40}", re.ASCII)
+
+
+def bounded_release_sha(value: object) -> str:
+    """Return only an exact Git SHA-1 identity or a fixed unavailable marker."""
+
+    if type(value) is str and _RELEASE_SHA.fullmatch(value) is not None:
+        return value
+    return "unavailable"
+
+
+def utc_timestamp() -> str:
+    """Return an RFC 3339 UTC timestamp with stable millisecond precision."""
+
+    return datetime.now(UTC).isoformat(timespec="milliseconds").replace(
+        "+00:00", "Z"
+    )
diff --git a/operations/tests/test_dependency_gate.py b/operations/tests/test_dependency_gate.py
index 4903736..de8c188 100644
--- a/operations/tests/test_dependency_gate.py
+++ b/operations/tests/test_dependency_gate.py
@@ -534,6 +534,7 @@ class DependencyContractTests(unittest.TestCase):
         for field in (
             "dependency_environment_sha256",
             "python_executable_sha256",
+            "python_mtree_sha256",
             "python_tree_sha256",
             "uv_executable_sha256",
             "ca_sha256",
@@ -575,6 +576,69 @@ class DependencyContractTests(unittest.TestCase):
             gate.EXPECTED_PYTHON_ARCHIVE_SHA256,
         )
 
+    def test_gate_generates_and_replays_the_exact_content_bearing_mtree(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            output = Path(temporary) / "bootstrap.mtree"
+            output, parent_identity = gate.safe_new_bootstrap_tree_spec_path(
+                str(output)
+            )
+            digest = gate.generate_bootstrap_tree_spec(
+                Path(sys.executable),
+                output,
+                output_parent_identity=parent_identity,
+            )
+            content = output.read_bytes()
+            policy = gate.current_platform_policy()
+            self.assertEqual(digest, gate.EXPECTED_BOOTSTRAP_MTREE_SHA256)
+            self.assertEqual(digest, policy["python_mtree_sha256"])
+            self.assertEqual(stat.S_IMODE(output.stat().st_mode), 0o444)
+            self.assertGreater(len(content), 100_000)
+            self.assertTrue(
+                content.startswith(gate.bootstrap_mtree_header(policy))
+            )
+            self.assertIn(b"sha256digest=", content)
+            self.assertIn(b" type=link link=", content)
+            replay = subprocess.run(
+                [
+                    str(gate.MTREE),
+                    "-f",
+                    str(output),
+                    "-p",
+                    str(Path(sys.executable).resolve().parents[1]),
+                ],
+                cwd=gate.system_temporary_root(),
+                env=gate.fixed_mtree_environment(),
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                check=False,
+                timeout=30,
+            )
+            self.assertEqual(replay.returncode, 0)
+            self.assertEqual(replay.stdout, b"")
+            self.assertEqual(replay.stderr, b"")
+
+    def test_tree_spec_creation_is_exclusive_and_preserves_a_caller_file(self):
+        with tempfile.TemporaryDirectory(
+            dir=gate.system_temporary_root()
+        ) as temporary:
+            root = Path(temporary)
+            target = root / "bootstrap.mtree"
+            target.write_bytes(b"caller-owned\n")
+            parent_status = root.stat()
+            with self.assertRaises(gate.GateError):
+                gate.write_bootstrap_tree_spec(
+                    target,
+                    b"replacement\n",
+                    parent_identity=(
+                        parent_status.st_dev,
+                        parent_status.st_ino,
+                    ),
+                )
+            self.assertEqual(target.read_bytes(), b"caller-owned\n")
+
     def test_python_tree_rejects_added_or_mutated_bytecode(self):
         with tempfile.TemporaryDirectory(
             dir=gate.system_temporary_root()
@@ -630,6 +694,8 @@ class DependencyContractTests(unittest.TestCase):
                 "/private/etc/ssl/cert.pem",
                 "--environment-dir",
                 str(ambient / ".venv"),
+                "--bootstrap-tree-spec-output",
+                str(ambient / "bootstrap.mtree"),
             ]
             unsafe = subprocess.run(
                 [sys.executable, str(SCRIPT), *arguments],
@@ -1346,6 +1412,7 @@ class DependencyContractTests(unittest.TestCase):
             self.addCleanup(temporary.cleanup)
         digests = []
         bootstrap_digests = []
+        mtree_digests = []
         for index in range(2):
             python_root = Path(temporary_roots[index * 4].name)
             cache = Path(temporary_roots[index * 4 + 1].name)
@@ -1375,6 +1442,7 @@ class DependencyContractTests(unittest.TestCase):
             self.assertEqual(extracted.stderr, b"")
             python = python_root / "python/bin/python3.14"
             environment = output / ".venv"
+            tree_spec = output / "bootstrap.mtree"
             result = subprocess.run(
                 [
                     str(python),
@@ -1394,6 +1462,8 @@ class DependencyContractTests(unittest.TestCase):
                     "/private/etc/ssl/cert.pem",
                     "--environment-dir",
                     str(environment),
+                    "--bootstrap-tree-spec-output",
+                    str(tree_spec),
                 ],
                 cwd=SCRIPT.parent.parent,
                 env={
@@ -1419,15 +1489,23 @@ class DependencyContractTests(unittest.TestCase):
             match = gate.re.fullmatch(
                 gate.re.escape(gate.SUCCESS_RECEIPT)
                 + r" dependency_environment_sha256=([0-9a-f]{64})"
-                + r" bootstrap_python_tree_sha256=([0-9a-f]{64})",
+                + r" bootstrap_python_tree_sha256=([0-9a-f]{64})"
+                + r" bootstrap_python_mtree_sha256=([0-9a-f]{64})",
                 receipt,
             )
             self.assertIsNotNone(match)
             digests.append(match.group(1))
             bootstrap_digests.append(match.group(2))
+            mtree_digests.append(match.group(3))
+            self.assertEqual(
+                hashlib.sha256(tree_spec.read_bytes()).hexdigest(),
+                match.group(3),
+            )
+            self.assertEqual(stat.S_IMODE(tree_spec.stat().st_mode), 0o444)
             self.assertEqual(tuple(cache.iterdir()), ())
         self.assertEqual(len(set(digests)), 1)
         self.assertEqual(len(set(bootstrap_digests)), 1)
+        self.assertEqual(len(set(mtree_digests)), 1)
         self.assertEqual(
             digests[0],
             gate.current_platform_policy()["dependency_environment_sha256"],
@@ -1436,6 +1514,10 @@ class DependencyContractTests(unittest.TestCase):
             bootstrap_digests[0],
             gate.current_platform_policy()["python_tree_sha256"],
         )
+        self.assertEqual(
+            mtree_digests[0],
+            gate.current_platform_policy()["python_mtree_sha256"],
+        )
 
     def test_unreviewed_embedded_environment_paths_fail_closed(self):
         with tempfile.TemporaryDirectory(
@@ -1550,6 +1632,7 @@ class DependencyContractTests(unittest.TestCase):
                 "--cache-dir": "/fixture/cache",
                 "--ca-file": "/fixture/ca",
                 "--environment-dir": str(environment),
+                "--bootstrap-tree-spec-output": str(root / "bootstrap.mtree"),
             }
             resolved = {
                 "/fixture/source": Path("/fixture/source"),
@@ -1576,6 +1659,11 @@ class DependencyContractTests(unittest.TestCase):
                 mock.patch.object(
                     gate, "safe_new_environment_path", return_value=environment
                 ),
+                mock.patch.object(
+                    gate,
+                    "safe_new_bootstrap_tree_spec_path",
+                    return_value=(root / "bootstrap.mtree", (1, 1)),
+                ),
                 mock.patch.object(gate, "verify_input_boundaries"),
                 mock.patch.object(gate, "capture_policy_snapshot", return_value={}),
                 mock.patch.object(gate, "parse_runtime"),
@@ -1651,9 +1739,12 @@ class DependencyContractTests(unittest.TestCase):
     def test_main_emits_only_fixed_receipt_and_environment_digest(self):
         digest = "a" * 64
         bootstrap_digest = "b" * 64
+        mtree_digest = "c" * 64
         with (
             mock.patch.object(
-                gate, "run_gate", return_value=(digest, bootstrap_digest)
+                gate,
+                "run_gate",
+                return_value=(digest, bootstrap_digest, mtree_digest),
             ),
             mock.patch.object(gate.sys, "argv", ["check-dependencies"]),
             mock.patch("builtins.print") as printed,
@@ -1661,7 +1752,8 @@ class DependencyContractTests(unittest.TestCase):
             self.assertEqual(gate.main(), 0)
         printed.assert_called_once_with(
             f"{gate.SUCCESS_RECEIPT} dependency_environment_sha256={digest} "
-            f"bootstrap_python_tree_sha256={bootstrap_digest}"
+            f"bootstrap_python_tree_sha256={bootstrap_digest} "
+            f"bootstrap_python_mtree_sha256={mtree_digest}"
         )
 
     def test_main_redacts_unexpected_failure_details(self):
diff --git a/operations/tests/test_runtime_config.py b/operations/tests/test_runtime_config.py
index c42213e..2aed2b6 100644
--- a/operations/tests/test_runtime_config.py
+++ b/operations/tests/test_runtime_config.py
@@ -1,14 +1,22 @@
 import io
+import base64
+import ast
+import hashlib
 import json
 import logging
 import os
 from pathlib import Path
+import re
 import runpy
+import shutil
+import signal
 import ssl
 import stat
 import subprocess
 import sys
+import sysconfig
 import tempfile
+import tomllib
 from unittest.mock import patch
 
 from django.conf import settings
@@ -19,12 +27,1451 @@ from operations.gunicorn_logging import (
     FixedGunicornLogger,
     _RedactedGunicornFilter,
 )
+from operations.telemetry import bounded_release_sha
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
+def _sha256(value: bytes) -> str:
+    return hashlib.sha256(value).hexdigest()
+
+
+_DEPENDENCY_GATE: dict[str, object] | None = None
+_BOOTSTRAP_MTREE_BYTES: bytes | None = None
+
+
+def _dependency_gate() -> dict[str, object]:
+    global _DEPENDENCY_GATE
+    if _DEPENDENCY_GATE is None:
+        _DEPENDENCY_GATE = runpy.run_path(
+            str(settings.BASE_DIR / "scripts" / "check-dependencies")
+        )
+    return _DEPENDENCY_GATE
+
+
+def _bootstrap_mtree_bytes() -> bytes:
+    global _BOOTSTRAP_MTREE_BYTES
+    if _BOOTSTRAP_MTREE_BYTES is None:
+        gate = _dependency_gate()
+        policy_function = gate["current_platform_policy"]
+        header_function = gate["bootstrap_mtree_header"]
+        if not callable(policy_function) or not callable(header_function):
+            raise AssertionError("bootstrap mtree policy is unavailable")
+        policy = policy_function()
+        executable = Path(sys.executable).resolve()
+        result = subprocess.run(
+            [
+                "/usr/sbin/mtree",
+                "-c",
+                "-n",
+                "-k",
+                "type,mode,nlink,sha256digest,link",
+                "-p",
+                str(executable.parents[1]),
+            ],
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "TZ": "UTC",
+            },
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+            timeout=30,
+        )
+        if result.returncode != 0 or result.stderr:
+            raise AssertionError("bootstrap mtree generation failed")
+        content = header_function(policy) + result.stdout
+        if _sha256(content) != policy["python_mtree_sha256"]:
+            raise AssertionError("bootstrap mtree digest is not pinned")
+        _BOOTSTRAP_MTREE_BYTES = content
+    return _BOOTSTRAP_MTREE_BYTES
+
+
+class _RuntimeReleaseFixture:
+    release_sha = "a" * 40
+
+    def __init__(
+        self,
+        root: Path,
+        repository: Path,
+        *,
+        full_source: bool = False,
+        settings_source: bytes | None = None,
+    ):
+        self.repository = repository
+        self.release = root / "release"
+        self.source = self.release / "source"
+        self.manifest = self.release / "manifest.json"
+        self.receipt = root / "runtime-receipt.json"
+        self.bootstrap_tree_spec = root / "bootstrap-tree.mtree"
+        self.bootstrap_tree_spec.write_bytes(_bootstrap_mtree_bytes())
+        self.bootstrap_tree_spec.chmod(0o444)
+        self.receipt_sha256: str | None = None
+        self.expected_dependency_environment_sha256: str | None = None
+        self.files = self._full_files() if full_source else self._minimal_files()
+        if settings_source is not None:
+            self.files["travel_readiness/settings.py"] = (settings_source, 0o644)
+            self.files.setdefault("travel_readiness/__init__.py", (b"", 0o644))
+        self._write_source()
+        self.manifest_value = self._manifest_value()
+        self.write_manifest()
+        self.expected_manifest_sha256 = self.manifest_sha256
+
+    def _minimal_files(self) -> dict[str, tuple[bytes, int]]:
+        return {
+            "runtime/versions.toml": (
+                b"[runtime]\npython = \"3.14.7\"\n",
+                0o644,
+            ),
+            "sample/migrations/0001_initial.py": (b"fixture = True\n", 0o644),
+            "sample/static/site.css": (b"body { color: #123456; }\n", 0o644),
+            "scripts/run-production": (
+                (self.repository / "scripts" / "run-production").read_bytes(),
+                0o755,
+            ),
+            "scripts/verify-release-runtime": (
+                (
+                    self.repository / "scripts" / "verify-release-runtime"
+                ).read_bytes(),
+                0o755,
+            ),
+            "uv.lock": (
+                (self.repository / "uv.lock").read_bytes(),
+                0o644,
+            ),
+        }
+
+    def _full_files(self) -> dict[str, tuple[bytes, int]]:
+        result = subprocess.run(
+            ["git", "ls-files", "-z"],
+            cwd=self.repository,
+            capture_output=True,
+            check=False,
+        )
+        if result.returncode != 0:
+            raise AssertionError("tracked fixture inventory is unavailable")
+        relatives = {
+            raw.decode("utf-8", "strict")
+            for raw in result.stdout.split(b"\0")
+            if raw
+        }
+        # Release-candidate tests also exercise newly created, not-yet-committed
+        # runtime files during the atom that introduces them.
+        relatives.update(
+            {
+                "operations/telemetry.py",
+                "scripts/verify-release-runtime",
+            }
+        )
+        files: dict[str, tuple[bytes, int]] = {}
+        for relative in sorted(relatives):
+            path = self.repository.joinpath(*Path(relative).parts)
+            mode = 0o755 if path.stat().st_mode & 0o111 else 0o644
+            files[relative] = (path.read_bytes(), mode)
+        return files
+
+    def _write_source(self) -> None:
+        self.source.mkdir(parents=True, mode=0o755)
+        for relative, (data, mode) in self.files.items():
+            path = self.source.joinpath(*Path(relative).parts)
+            path.parent.mkdir(parents=True, exist_ok=True, mode=0o755)
+            path.write_bytes(data)
+            path.chmod(mode)
+        for directory in sorted(
+            (path for path in self.source.rglob("*") if path.is_dir()),
+            key=lambda path: len(path.parts),
+            reverse=True,
+        ):
+            directory.chmod(0o755)
+        self.source.chmod(0o755)
+        self.release.chmod(0o755)
+
+    def _manifest_value(self) -> dict[str, object]:
+        tracked = [
+            {
+                "git_mode": "100755" if mode == 0o755 else "100644",
+                "path": relative,
+                "sha256": _sha256(data),
+            }
+            for relative, (data, mode) in sorted(self.files.items())
+        ]
+        migration_path = (
+            "entry_requirements/migrations/0001_initial.py"
+            if "entry_requirements/migrations/0001_initial.py" in self.files
+            else "sample/migrations/0001_initial.py"
+        )
+        migration_digest = _sha256(self.files[migration_path][0])
+        leaves = [
+            {
+                "app": "entry_requirements",
+                "module": "entry_requirements.migrations.0001_initial",
+                "name": "0001_initial",
+                "origin": "source",
+                "path": migration_path,
+                "sha256": migration_digest,
+            }
+        ]
+        static_source = (
+            "public_web/static/public_web/site.css"
+            if "public_web/static/public_web/site.css" in self.files
+            else "sample/static/site.css"
+        )
+        static_digest = _sha256(self.files[static_source][0])
+        collected = [{"path": "public_web/site.css", "sha256": static_digest}]
+        origins = [
+            {
+                "collected_path": "public_web/site.css",
+                "origin": "source",
+                "path": static_source,
+                "sha256": static_digest,
+            }
+        ]
+        tracked_static = [
+            {
+                "collected_path": "public_web/site.css",
+                "path": static_source,
+                "sha256": static_digest,
+            }
+        ]
+        lock = tomllib.loads(self.files["uv.lock"][0].decode("utf-8"))
+        packages = sorted(
+            (
+                {"name": package["name"], "version": package["version"]}
+                for package in lock["package"]
+            ),
+            key=lambda package: (package["name"], package["version"]),
+        )
+        lock_digest = _sha256(self.files["uv.lock"][0])
+        runtime_digest = _sha256(self.files["runtime/versions.toml"][0])
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
+                "package_set_sha256": _sha256(_canonical_json(packages)),
+                "packages": packages,
+            },
+            "git_object_format": "sha1",
+            "migrations": {
+                "leaf_set_sha256": _sha256(_canonical_json(leaves)),
+                "leaves": leaves,
+            },
+            "release_sha": self.release_sha,
+            "runtime": {
+                "path": "runtime/versions.toml",
+                "sha256": runtime_digest,
+                "versions": {
+                    "django": "5.2.17",
+                    "gunicorn": "26.2.0",
+                    "postgresql": "18.6",
+                    "psycopg": "3.3.4",
+                    "psycopg_distribution": "binary-wheel",
+                    "python": "3.14.7",
+                    "uv": "0.12.6",
+                },
+            },
+            "schema_version": 1,
+            "source": {
+                "tracked": tracked,
+                "tracked_set_sha256": _sha256(_canonical_json(tracked)),
+            },
+            "static": {
+                "collected": collected,
+                "collected_set_sha256": _sha256(_canonical_json(collected)),
+                "origins": origins,
+                "tracked": tracked_static,
+                "tracked_set_sha256": _sha256(_canonical_json(tracked_static)),
+            },
+        }
+
+    def write_manifest(
+        self,
+        value: dict[str, object] | None = None,
+        *,
+        canonical: bool = True,
+    ) -> None:
+        payload = self.manifest_value if value is None else value
+        data = (
+            _canonical_json(payload)
+            if canonical
+            else json.dumps(payload, indent=2).encode("utf-8")
+        )
+        self.manifest.write_bytes(data)
+        self.manifest.chmod(0o644)
+
+    @property
+    def verifier(self) -> Path:
+        return self.source / "scripts" / "verify-release-runtime"
+
+    @property
+    def startup(self) -> Path:
+        return self.source / "scripts" / "run-production"
+
+    def add_venv(self) -> None:
+        binary = self.source / ".venv" / "bin"
+        binary.mkdir(parents=True, mode=0o755)
+        python = binary / "python"
+        python.symlink_to(Path(sys.executable).resolve())
+        (binary / "python3").symlink_to("python")
+        (self.source / ".venv" / "pyvenv.cfg").write_text(
+            f"home = {self.bootstrap_python.parent}\n"
+            "implementation = CPython\n"
+            "uv = 0.12.6\n"
+            "version_info = 3.14.7\n"
+            "include-system-site-packages = false\n",
+            encoding="utf-8",
+        )
+        site_packages = (
+            self.source
+            / ".venv"
+            / "lib"
+            / f"python{sys.version_info.major}.{sys.version_info.minor}"
+            / "site-packages"
+        )
+        def record_hash(path: Path) -> str:
+            digest = hashlib.sha256(path.read_bytes()).digest()
+            encoded = base64.urlsafe_b64encode(digest).rstrip(b"=")
+            return "sha256=" + encoded.decode("ascii")
+
+        lock = tomllib.loads(self.files["uv.lock"][0].decode("utf-8"))
+        selected = {
+            package["name"]: package["version"]
+            for package in lock["package"]
+            if package.get("source", {}).get("virtual") != "."
+            and not (
+                package["name"] == "tzdata" and sys.platform != "win32"
+            )
+        }
+        gate = _dependency_gate()
+        console_scripts = gate["EXPECTED_CONSOLE_SCRIPTS"]
+        console_source = gate["expected_console_script"]
+        if not isinstance(console_scripts, dict) or not callable(console_source):
+            raise AssertionError("dependency gate console policy is unavailable")
+        environment_bytes = os.fsencode(str(self.source / ".venv"))
+        for name, version in sorted(selected.items()):
+            dist_name = name.replace("-", "_")
+            dist_info = site_packages / f"{dist_name}-{version}.dist-info"
+            dist_info.mkdir(parents=True, mode=0o755)
+            metadata = dist_info / "METADATA"
+            wheel = dist_info / "WHEEL"
+            record = dist_info / "RECORD"
+            metadata.write_text(
+                f"Metadata-Version: 2.4\nName: {name}\nVersion: {version}\n\n",
+                encoding="utf-8",
+            )
+            wheel.write_text(
+                "Wheel-Version: 1.0\n"
+                "Generator: runtime-fixture\n"
+                "Root-Is-Purelib: true\n"
+                "Tag: py3-none-any\n\n",
+                encoding="utf-8",
+            )
+            metadata_relative = f"{dist_info.name}/METADATA"
+            wheel_relative = f"{dist_info.name}/WHEEL"
+            record_relative = f"{dist_info.name}/RECORD"
+            record_rows = [
+                f"{metadata_relative},{record_hash(metadata)},{metadata.stat().st_size}",
+                f"{wheel_relative},{record_hash(wheel)},{wheel.stat().st_size}",
+                f"{record_relative},,",
+            ]
+            for script_relative, specification in console_scripts.items():
+                if specification.get("record") != (
+                    f"lib/python3.14/site-packages/{record_relative}"
+                ):
+                    continue
+                script_path = self.source / ".venv" / script_relative
+                script_path.write_bytes(
+                    console_source(environment_bytes, specification)
+                )
+                script_path.chmod(0o755)
+                record_rows.append(
+                    f"{specification['record_path']},{record_hash(script_path)},"
+                    f"{script_path.stat().st_size}"
+                )
+            record_rows.sort(key=lambda row: row.split(",", 1)[0])
+            record.write_text(
+                "\n".join(record_rows) + "\n", encoding="utf-8"
+            )
+        actual_site_packages = Path(sysconfig.get_path("purelib")).resolve()
+        for package_name in (
+            "asgiref",
+            "django",
+            "gunicorn",
+            "psycopg",
+            "psycopg_binary",
+            "sqlparse",
+        ):
+            (site_packages / package_name).symlink_to(
+                actual_site_packages / package_name
+            )
+        (self.source / ".venv").chmod(0o755)
+
+    @property
+    def bootstrap_python(self) -> Path:
+        return Path(sys.executable).resolve()
+
+    @property
+    def bootstrap_sha256(self) -> str:
+        return _sha256(self.bootstrap_python.read_bytes())
+
+    @property
+    def bootstrap_prefix(self) -> Path:
+        return self.bootstrap_python.parents[1]
+
+    @property
+    def bootstrap_pristine_tree_sha256(self) -> str:
+        policy_function = _dependency_gate()["current_platform_policy"]
+        if not callable(policy_function):
+            raise AssertionError("bootstrap tree policy is unavailable")
+        return str(policy_function()["python_tree_sha256"])
+
+    @property
+    def bootstrap_tree_spec_sha256(self) -> str:
+        return _sha256(self.bootstrap_tree_spec.read_bytes())
+
+    @property
+    def verifier_sha256(self) -> str:
+        return _sha256(self.verifier.read_bytes())
+
+    @property
+    def manifest_sha256(self) -> str:
+        return _sha256(self.manifest.read_bytes())
+
+    @property
+    def dependency_environment_sha256(self) -> str:
+        digest = _dependency_gate()["dependency_environment_sha256"]
+        if not callable(digest):
+            raise AssertionError("dependency gate digest is unavailable")
+        return digest(
+            self.source / ".venv", python=self.bootstrap_python
+        )
+
+    @property
+    def trusted_owner_uid(self) -> str:
+        return str(os.geteuid())
+
+    def verifier_arguments(self, mode: str) -> list[str]:
+        arguments = [
+            str(self.bootstrap_python),
+            "-I",
+            "-S",
+            "-B",
+            str(self.verifier),
+            mode,
+            "--bootstrap-prefix",
+            str(self.bootstrap_prefix),
+            "--bootstrap-pristine-tree-sha256",
+            self.bootstrap_pristine_tree_sha256,
+            "--bootstrap-python",
+            str(self.bootstrap_python),
+            "--bootstrap-python-sha256",
+            self.bootstrap_sha256,
+            "--bootstrap-tree-spec",
+            str(self.bootstrap_tree_spec),
+            "--bootstrap-tree-spec-sha256",
+            self.bootstrap_tree_spec_sha256,
+            "--expected-dependency-environment-sha256",
+            (
+                self.expected_dependency_environment_sha256
+                or self.dependency_environment_sha256
+            ),
+            "--expected-manifest-sha256",
+            self.expected_manifest_sha256,
+            "--expected-release-sha",
+            self.release_sha,
+            "--expected-verifier-sha256",
+            self.verifier_sha256,
+        ]
+        if mode != "prepare":
+            if self.receipt_sha256 is None:
+                raise AssertionError("runtime receipt is not prepared")
+            arguments.extend(
+                [
+                    "--expected-runtime-receipt-sha256",
+                    self.receipt_sha256,
+                ]
+            )
+        arguments.extend(
+            [
+                "--runtime-receipt",
+                str(self.receipt),
+                "--trusted-owner-uid",
+                self.trusted_owner_uid,
+            ]
+        )
+        return arguments
+
+    def prepare_receipt(self) -> subprocess.CompletedProcess[str]:
+        self.expected_dependency_environment_sha256 = (
+            self.dependency_environment_sha256
+        )
+        result = subprocess.run(
+            self.verifier_arguments("prepare"),
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+                "TZ": "UTC",
+            },
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=30,
+        )
+        if result.returncode == 0:
+            prefix = "runtime_receipt_sha256="
+            if not result.stdout.startswith(prefix):
+                raise AssertionError("runtime receipt digest is unavailable")
+            self.receipt_sha256 = result.stdout.removeprefix(prefix).strip()
+        return result
+
+    def startup_environment(
+        self, cert_file: Path, key_file: Path, **overrides: str
+    ) -> dict[str, str]:
+        if self.receipt_sha256 is None:
+            raise AssertionError("runtime receipt is not prepared")
+        environment = {
+            "PATH": "/usr/bin:/bin",
+            "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
+            "TRAVEL_READINESS_BOOTSTRAP_PYTHON": str(self.bootstrap_python),
+            "TRAVEL_READINESS_BOOTSTRAP_PYTHON_SHA256": self.bootstrap_sha256,
+            "TRAVEL_READINESS_BOOTSTRAP_PREFIX": str(self.bootstrap_prefix),
+            "TRAVEL_READINESS_BOOTSTRAP_PRISTINE_TREE_SHA256": (
+                self.bootstrap_pristine_tree_sha256
+            ),
+            "TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_FILE": str(
+                self.bootstrap_tree_spec
+            ),
+            "TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_SHA256": (
+                self.bootstrap_tree_spec_sha256
+            ),
+            "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+            "TRAVEL_READINESS_DB_NAME": "travel_readiness",
+            "TRAVEL_READINESS_DB_PASSWORD": "test-only-database-value",
+            "TRAVEL_READINESS_DB_PORT": "1",
+            "TRAVEL_READINESS_DB_USER": "travel_readiness",
+            "TRAVEL_READINESS_DEBUG": "0",
+            "TRAVEL_READINESS_DEPENDENCY_ENVIRONMENT_SHA256": (
+                self.expected_dependency_environment_sha256 or ""
+            ),
+            "TRAVEL_READINESS_EXPECTED_MANIFEST_SHA256": (
+                self.expected_manifest_sha256
+            ),
+            "TRAVEL_READINESS_EXPECTED_RELEASE_SHA": self.release_sha,
+            "TRAVEL_READINESS_HTTPS": "1",
+            "TRAVEL_READINESS_RELEASE_OWNER_UID": self.trusted_owner_uid,
+            "TRAVEL_READINESS_RUNTIME_RECEIPT_FILE": str(self.receipt),
+            "TRAVEL_READINESS_RUNTIME_RECEIPT_SHA256": self.receipt_sha256,
+            "TRAVEL_READINESS_SECRET_KEY": (
+                "test-only-startup-secret-with-enough-entropy-not-a-credential"
+            ),
+            "TRAVEL_READINESS_VERIFIER_SHA256": self.verifier_sha256,
+            "TRAVEL_READINESS_TLS_CERT_FILE": str(cert_file),
+            "TRAVEL_READINESS_TLS_KEY_FILE": str(key_file),
+        }
+        environment.update(overrides)
+        return environment
 
 
 class RuntimeConfigTests(SimpleTestCase):
     def setUp(self):
         self.config_path = settings.BASE_DIR / "runtime" / "gunicorn.conf.py"
 
+    def _runtime_fixture(self, temporary: str, **options) -> _RuntimeReleaseFixture:
+        return _RuntimeReleaseFixture(
+            Path(temporary).resolve(),
+            settings.BASE_DIR,
+            **options,
+        )
+
+    def _tls_pair(self, root: Path) -> tuple[Path, Path]:
+        root = root.resolve()
+        cert_file = root / "candidate-cert.pem"
+        key_file = root / "candidate-key.pem"
+        generated = subprocess.run(
+            [
+                "/usr/bin/openssl",
+                "req",
+                "-x509",
+                "-newkey",
+                "rsa:2048",
+                "-nodes",
+                "-days",
+                "1",
+                "-subj",
+                "/CN=127.0.0.1",
+                "-keyout",
+                str(key_file),
+                "-out",
+                str(cert_file),
+            ],
+            env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=15,
+        )
+        self.assertEqual(generated.returncode, 0)
+        os.chown(key_file, -1, os.getegid())
+        key_file.chmod(0o440)
+        return cert_file, key_file
+
+    def _run_verifier(self, fixture: _RuntimeReleaseFixture):
+        return subprocess.run(
+            fixture.verifier_arguments("verify"),
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+            },
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=15,
+        )
+
+    def _run_launch_unit(
+        self,
+        fixture: _RuntimeReleaseFixture,
+        cert_file: Path,
+        key_file: Path,
+        *,
+        runtime_values: dict[str, str] | None = None,
+        ambient_environment: dict[str, str] | None = None,
+    ) -> subprocess.CompletedProcess[str]:
+        values = {
+            "TRAVEL_READINESS_SECRET_KEY": (
+                "test-only-startup-secret-with-enough-entropy-not-a-credential"
+            ),
+            "TRAVEL_READINESS_DB_PASSWORD": "test-only-database-value",
+            "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
+            "TRAVEL_READINESS_DB_NAME": "travel_readiness",
+            "TRAVEL_READINESS_DB_USER": "travel_readiness",
+            "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+            "TRAVEL_READINESS_DB_PORT": "1",
+            "TRAVEL_READINESS_HTTPS": "1",
+            "TRAVEL_READINESS_DEBUG": "0",
+        }
+        values.update(runtime_values or {})
+        payload = "\0".join(values[name] for name in (
+            "TRAVEL_READINESS_SECRET_KEY",
+            "TRAVEL_READINESS_DB_PASSWORD",
+            "TRAVEL_READINESS_ALLOWED_HOSTS",
+            "TRAVEL_READINESS_DB_NAME",
+            "TRAVEL_READINESS_DB_USER",
+            "TRAVEL_READINESS_DB_HOST",
+            "TRAVEL_READINESS_DB_PORT",
+            "TRAVEL_READINESS_HTTPS",
+            "TRAVEL_READINESS_DEBUG",
+        )) + "\0"
+        environment = {
+            "LANG": "C",
+            "LC_ALL": "C",
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "TZ": "UTC",
+        }
+        environment.update(ambient_environment or {})
+        code = (
+            "import os,pathlib,runpy,sys\n"
+            "m=runpy.run_path(sys.argv[1])\n"
+            "m['launch_runtime'].__globals__['runtime_principal']="
+            "lambda trusted:(trusted+1,os.getegid())\n"
+            "try:\n"
+            " m['launch_runtime'](source_root=pathlib.Path(sys.argv[2]),"
+            "release_sha=sys.argv[3],bootstrap_python=pathlib.Path(sys.argv[4]),"
+            "bootstrap_digest=sys.argv[5],trusted_owner_uid=int(sys.argv[6]),"
+            "expected_dependency_environment_sha256=sys.argv[7],"
+            "tls_cert=sys.argv[8],tls_key=sys.argv[9],bind='127.0.0.1:8000')\n"
+            "except m['LaunchError'] as error:\n"
+            " print('startup_error='+error.code,file=sys.stderr)\n"
+            " raise SystemExit(error.exit_code)\n"
+        )
+        return subprocess.run(
+            [
+                sys.executable,
+                "-I",
+                "-S",
+                "-B",
+                "-c",
+                code,
+                str(settings.BASE_DIR / "scripts" / "verify-release-runtime"),
+                str(fixture.source),
+                fixture.release_sha,
+                str(fixture.bootstrap_python),
+                fixture.bootstrap_sha256,
+                fixture.trusted_owner_uid,
+                fixture.expected_dependency_environment_sha256
+                or fixture.dependency_environment_sha256,
+                str(cert_file),
+                str(key_file),
+            ],
+            cwd="/private/tmp",
+            env=environment,
+            input=payload,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=20,
+        )
+
+    def _assert_verifier_rejected(self, fixture: _RuntimeReleaseFixture):
+        result = self._run_verifier(fixture)
+        self.assertEqual(result.returncode, 1)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "release_runtime=invalid\n")
+
+    def _assert_prepare_rejected(self, fixture: _RuntimeReleaseFixture):
+        if fixture.expected_dependency_environment_sha256 is None:
+            fixture.expected_dependency_environment_sha256 = (
+                fixture.dependency_environment_sha256
+            )
+        result = subprocess.run(
+            fixture.verifier_arguments("prepare"),
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+                "TZ": "UTC",
+            },
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=30,
+        )
+        self.assertEqual(result.returncode, 1)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "runtime_receipt=invalid\n")
+
+    def test_release_runtime_verifier_accepts_exact_source_and_venv_exception(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            result = self._run_verifier(fixture)
+        self.assertEqual(result.returncode, 0)
+        self.assertEqual(result.stdout, f"{fixture.release_sha}\n")
+        self.assertEqual(result.stderr, "")
+
+    def test_prepare_rejects_a_comment_only_caller_mtree_specification(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            fixture.bootstrap_tree_spec.chmod(0o644)
+            fixture.bootstrap_tree_spec.write_bytes(
+                b"# caller-provided comment is not a tree specification\n"
+            )
+            fixture.bootstrap_tree_spec.chmod(0o444)
+            self._assert_prepare_rejected(fixture)
+
+    def test_release_runtime_verifier_rejects_source_tamper_and_extra_entry(self):
+        mutations = {
+            "tamper": lambda fixture: (
+                fixture.source / "sample" / "static" / "site.css"
+            ).write_text("tampered\n", encoding="utf-8"),
+            "extra": lambda fixture: (
+                fixture.source / "unexpected.txt"
+            ).write_text("unexpected\n", encoding="utf-8"),
+        }
+        for name, mutate in mutations.items():
+            with self.subTest(name=name), tempfile.TemporaryDirectory() as temporary:
+                fixture = self._runtime_fixture(temporary)
+                fixture.add_venv()
+                prepared = fixture.prepare_receipt()
+                self.assertEqual(prepared.returncode, 0, prepared.stderr)
+                mutate(fixture)
+                self._assert_verifier_rejected(fixture)
+
+    def test_release_runtime_verifier_rejects_links_and_noncanonical_layout(self):
+        with self.subTest(case="tracked-symlink"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            target = fixture.source / "sample" / "static" / "site.css"
+            target.unlink()
+            target.symlink_to(fixture.source / "uv.lock")
+            self._assert_verifier_rejected(fixture)
+
+        with self.subTest(case="venv-symlink"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            shutil.rmtree(fixture.source / ".venv")
+            (fixture.source / ".venv").symlink_to("runtime")
+            self._assert_verifier_rejected(fixture)
+
+        with self.subTest(case="ancestor-symlink"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            alias = Path(temporary) / "alias" / "release"
+            alias.parent.mkdir()
+            alias.symlink_to(fixture.release)
+            alias_verifier = alias / "source" / "scripts" / "verify-release-runtime"
+            result = subprocess.run(
+                [
+                    *fixture.verifier_arguments("verify")[:4],
+                    str(alias_verifier),
+                    *fixture.verifier_arguments("verify")[5:],
+                ],
+                cwd="/private/tmp",
+                env={
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                },
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=15,
+            )
+            self.assertEqual(result.returncode, 1)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(result.stderr, "release_runtime=invalid\n")
+
+    def test_release_runtime_verifier_rejects_manifest_schema_and_encoding_drift(self):
+        with self.subTest(case="noncanonical-json"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            fixture.write_manifest(canonical=False)
+            self._assert_verifier_rejected(fixture)
+
+        with self.subTest(case="source-schema"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            manifest = json.loads(fixture.manifest.read_text(encoding="utf-8"))
+            manifest["source"]["tracked"][0]["unexpected"] = True
+            manifest["source"]["tracked_set_sha256"] = _sha256(
+                _canonical_json(manifest["source"]["tracked"])
+            )
+            fixture.write_manifest(manifest)
+            self._assert_verifier_rejected(fixture)
+
+        with self.subTest(case="release-sha"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            manifest = json.loads(fixture.manifest.read_text(encoding="utf-8"))
+            manifest["release_sha"] = "A" * 40
+            fixture.write_manifest(manifest)
+            self._assert_verifier_rejected(fixture)
+
+    def test_release_runtime_verifier_rejects_unsafe_permissions(self):
+        mutations = {
+            "release-root": lambda fixture: fixture.release.chmod(0o775),
+            "source-root": lambda fixture: fixture.source.chmod(0o775),
+            "manifest": lambda fixture: fixture.manifest.chmod(0o664),
+            "tracked-mode": lambda fixture: (
+                fixture.source / "uv.lock"
+            ).chmod(0o664),
+            "venv-root": lambda fixture: (
+                fixture.source / ".venv"
+            ).chmod(0o777),
+        }
+        for name, mutate in mutations.items():
+            with self.subTest(name=name), tempfile.TemporaryDirectory() as temporary:
+                fixture = self._runtime_fixture(temporary)
+                fixture.add_venv()
+                prepared = fixture.prepare_receipt()
+                self.assertEqual(prepared.returncode, 0, prepared.stderr)
+                mutate(fixture)
+                self._assert_verifier_rejected(fixture)
+
+    def test_runtime_receipt_binds_exact_dependency_environment_and_lock_selection(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary, full_source=True)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            receipt = json.loads(fixture.receipt.read_text(encoding="utf-8"))
+            self.assertEqual(stat.S_IMODE(fixture.receipt.stat().st_mode), 0o444)
+            self.assertEqual(
+                receipt["dependency_environment_sha256"],
+                receipt["canonical_inventory_sha256"],
+            )
+            self.assertEqual(
+                receipt["dependency_environment_sha256"],
+                fixture.expected_dependency_environment_sha256,
+            )
+            self.assertEqual(receipt["verifier_sha256"], fixture.verifier_sha256)
+            self.assertEqual(
+                receipt["bootstrap"]["pristine_tree_sha256"],
+                fixture.bootstrap_pristine_tree_sha256,
+            )
+            self.assertEqual(
+                receipt["bootstrap"]["tree_spec_sha256"],
+                fixture.bootstrap_tree_spec_sha256,
+            )
+            self.assertEqual(
+                receipt["bootstrap"]["tree_spec_format"],
+                "travel-readiness-bootstrap-mtree-v1",
+            )
+            selected = {
+                package["name"]
+                for package in receipt["lock_selection"]["packages"]
+            }
+            self.assertEqual(
+                selected,
+                {
+                    "asgiref",
+                    "django",
+                    "gunicorn",
+                    "psycopg",
+                    "psycopg-binary",
+                    "sqlparse",
+                },
+            )
+            self.assertNotIn("audience-foundry-travel-readiness", selected)
+            self.assertNotIn("tzdata", selected)
+            observed = {
+                distribution["name"] for distribution in receipt["distributions"]
+            }
+            self.assertEqual(observed, selected)
+
+    def test_prepare_requires_external_verifier_identity_anchor(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            fixture.expected_dependency_environment_sha256 = (
+                fixture.dependency_environment_sha256
+            )
+            arguments = fixture.verifier_arguments("prepare")
+            digest_index = arguments.index("--expected-verifier-sha256") + 1
+            arguments[digest_index] = "0" * 64
+            result = subprocess.run(
+                arguments,
+                cwd="/private/tmp",
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                    "TZ": "UTC",
+                },
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=30,
+            )
+        self.assertEqual(result.returncode, 1)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "runtime_receipt=invalid\n")
+
+    def test_verifier_fails_closed_without_isolated_no_site_no_bytecode_flags(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            fixture.expected_dependency_environment_sha256 = (
+                fixture.dependency_environment_sha256
+            )
+            exact_arguments = fixture.verifier_arguments("prepare")
+            insecure_arguments = [exact_arguments[0], *exact_arguments[4:]]
+            result = subprocess.run(
+                insecure_arguments,
+                cwd="/private/tmp",
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "TZ": "UTC",
+                },
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=30,
+            )
+        self.assertEqual(result.returncode, 1)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "")
+
+    def test_dependency_gate_and_runtime_verifier_share_exact_inventory_digest(self):
+        gate = _dependency_gate()
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            gate_digest = gate["dependency_environment_sha256"](
+                fixture.source / ".venv", python=fixture.bootstrap_python
+            )
+            verifier_probe = subprocess.run(
+                [
+                    sys.executable,
+                    "-I",
+                    "-S",
+                    "-B",
+                    "-c",
+                    (
+                        "import pathlib,runpy,sys;"
+                        "m=runpy.run_path(sys.argv[1]);"
+                        "i,_=m['inventory_venv'](pathlib.Path(sys.argv[2]),"
+                        "trusted_owner_uid=int(sys.argv[3]),"
+                        "bootstrap_python=pathlib.Path(sys.argv[4]));"
+                        "print(m['sha256'](m['canonical_json'](i)))"
+                    ),
+                    str(verifier_path),
+                    str(fixture.source),
+                    str(os.geteuid()),
+                    str(fixture.bootstrap_python),
+                ],
+                cwd="/private/tmp",
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                    "TZ": "UTC",
+                },
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=30,
+            )
+            self.assertEqual(verifier_probe.returncode, 0, verifier_probe.stderr)
+            verifier_digest = verifier_probe.stdout.strip()
+            fixture_digest = fixture.dependency_environment_sha256
+        self.assertEqual(gate_digest, verifier_digest)
+        self.assertEqual(gate_digest, fixture_digest)
+
+    def test_canonical_dependency_digest_is_path_independent_in_both_implementations(self):
+        gate = _dependency_gate()
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        observed: list[tuple[str, str]] = []
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            for name in ("first-root", "second-root-with-a-longer-name"):
+                fixture_root = root / name
+                fixture_root.mkdir()
+                fixture = self._runtime_fixture(str(fixture_root))
+                fixture.add_venv()
+                gate_digest = gate["dependency_environment_sha256"](
+                    fixture.source / ".venv",
+                    python=fixture.bootstrap_python,
+                )
+                verifier_probe = subprocess.run(
+                    [
+                        sys.executable,
+                        "-I",
+                        "-S",
+                        "-B",
+                        "-c",
+                        (
+                            "import pathlib,runpy,sys;"
+                            "m=runpy.run_path(sys.argv[1]);"
+                            "i,_=m['inventory_venv'](pathlib.Path(sys.argv[2]),"
+                            "trusted_owner_uid=int(sys.argv[3]),"
+                            "bootstrap_python=pathlib.Path(sys.argv[4]));"
+                            "print(m['sha256'](m['canonical_json'](i)))"
+                        ),
+                        str(verifier_path),
+                        str(fixture.source),
+                        str(os.geteuid()),
+                        str(fixture.bootstrap_python),
+                    ],
+                    cwd="/private/tmp",
+                    env={
+                        "LANG": "C",
+                        "LC_ALL": "C",
+                        "PATH": "/usr/bin:/bin",
+                        "PYTHONDONTWRITEBYTECODE": "1",
+                        "TZ": "UTC",
+                    },
+                    capture_output=True,
+                    text=True,
+                    check=False,
+                    timeout=30,
+                )
+                self.assertEqual(
+                    verifier_probe.returncode, 0, verifier_probe.stderr
+                )
+                verifier_digest = verifier_probe.stdout.strip()
+                observed.append((gate_digest, verifier_digest))
+                prepared = fixture.prepare_receipt()
+                self.assertEqual(prepared.returncode, 0, prepared.stderr)
+                receipt = fixture.receipt.read_bytes()
+                self.assertNotIn(
+                    os.fsencode(str(fixture.source / ".venv")), receipt
+                )
+                self.assertNotIn(
+                    os.fsencode(str(fixture.bootstrap_python.parent)), receipt
+                )
+        self.assertEqual(len({digest for pair in observed for digest in pair}), 1)
+
+    def test_runtime_receipt_rejects_environment_record_and_receipt_tamper(self):
+        mutations = {
+            "environment": lambda fixture: (
+                fixture.source / ".venv" / "pyvenv.cfg"
+            ).write_text("tampered\n", encoding="utf-8"),
+            "record": lambda fixture: next(
+                (fixture.source / ".venv").rglob("RECORD")
+            ).write_text("tampered,,\n", encoding="utf-8"),
+            "console": lambda fixture: (
+                fixture.source / ".venv" / "bin" / "gunicorn"
+            ).write_text("tampered\n", encoding="utf-8"),
+            "receipt": lambda fixture: (
+                fixture.receipt.chmod(0o600),
+                fixture.receipt.write_text("{}\n", encoding="utf-8"),
+                fixture.receipt.chmod(0o444),
+            ),
+        }
+        for name, mutate in mutations.items():
+            with self.subTest(name=name), tempfile.TemporaryDirectory() as temporary:
+                fixture = self._runtime_fixture(temporary)
+                fixture.add_venv()
+                prepared = fixture.prepare_receipt()
+                self.assertEqual(prepared.returncode, 0, prepared.stderr)
+                mutate(fixture)
+                self._assert_verifier_rejected(fixture)
+
+    def test_prepare_rejects_venv_hardlinks_inside_and_across_boundary(self):
+        with self.subTest(case="inside"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            fixture.expected_dependency_environment_sha256 = (
+                fixture.dependency_environment_sha256
+            )
+            original = fixture.source / ".venv" / "pyvenv.cfg"
+            os.link(original, fixture.source / ".venv" / "duplicate.cfg")
+            self._assert_prepare_rejected(fixture)
+
+        with self.subTest(case="outside"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            fixture.expected_dependency_environment_sha256 = (
+                fixture.dependency_environment_sha256
+            )
+            original = fixture.source / ".venv" / "pyvenv.cfg"
+            os.link(original, Path(temporary).resolve() / "outside-hardlink")
+            self._assert_prepare_rejected(fixture)
+
+    def test_runtime_principal_must_be_distinct_non_root(self):
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        code = (
+            "import os,runpy,sys\n"
+            "from unittest.mock import patch\n"
+            "m=runpy.run_path(sys.argv[1])\n"
+            "f=m['runtime_principal']; error=m['VerificationError']\n"
+            "def rejected(uid,owner):\n"
+            " try:\n"
+            "  with patch.object(os,'geteuid',return_value=uid),"
+            "patch.object(os,'getegid',return_value=77): f(owner)\n"
+            " except error: return True\n"
+            " return False\n"
+            "assert rejected(0,1000)\n"
+            "assert rejected(1000,1000)\n"
+            "with patch.object(os,'geteuid',return_value=1001),"
+            "patch.object(os,'getegid',return_value=77):\n"
+            " assert f(1000)==(1001,77)\n"
+        )
+        result = subprocess.run(
+            [sys.executable, "-I", "-S", "-B", "-c", code, str(verifier_path)],
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+                "TZ": "UTC",
+            },
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=30,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stdout, "")
+
+    def test_receipt_write_is_bounded_and_removes_partial_output(self):
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        with tempfile.TemporaryDirectory() as temporary:
+            receipt = Path(temporary).resolve() / "oversized-receipt.json"
+            code = (
+                "import pathlib,runpy,sys\n"
+                "m=runpy.run_path(sys.argv[1])\n"
+                "m['write_receipt'].__globals__['RECEIPT_LIMIT']=4\n"
+                "failed=False\n"
+                "try: m['write_receipt'](pathlib.Path(sys.argv[2]),b'12345',"
+                "trusted_owner_uid=int(sys.argv[3]))\n"
+                "except m['VerificationError']: failed=True\n"
+                "raise SystemExit(0 if failed and not pathlib.Path(sys.argv[2]).exists() else 1)\n"
+            )
+            result = subprocess.run(
+                [
+                    sys.executable,
+                    "-I",
+                    "-S",
+                    "-B",
+                    "-c",
+                    code,
+                    str(verifier_path),
+                    str(receipt),
+                    str(os.geteuid()),
+                ],
+                cwd="/private/tmp",
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "PYTHONDONTWRITEBYTECODE": "1",
+                    "TZ": "UTC",
+                },
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=30,
+            )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stdout, "")
+
+    def test_prepare_rejects_extra_distribution_even_with_matching_environment_digest(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            site_packages = next((fixture.source / ".venv").rglob("site-packages"))
+            dist_info = site_packages / "unselected-9.9.dist-info"
+            dist_info.mkdir()
+            metadata = dist_info / "METADATA"
+            metadata.write_text(
+                "Metadata-Version: 2.4\nName: unselected\nVersion: 9.9\n\n",
+                encoding="utf-8",
+            )
+            digest = base64.urlsafe_b64encode(
+                hashlib.sha256(metadata.read_bytes()).digest()
+            ).rstrip(b"=").decode("ascii")
+            (dist_info / "RECORD").write_text(
+                f"{dist_info.name}/METADATA,sha256={digest},{metadata.stat().st_size}\n"
+                f"{dist_info.name}/RECORD,,\n",
+                encoding="utf-8",
+            )
+            self._assert_prepare_rejected(fixture)
+
+    def test_prepare_rejects_nested_manifest_schema_and_venv_static_paths(self):
+        def nested_extra(manifest: dict[str, object], path: tuple[object, ...]) -> None:
+            target: object = manifest
+            for part in path:
+                target = target[part]  # type: ignore[index]
+            target["unexpected"] = True  # type: ignore[index]
+
+        cases = (
+            ("top-level", ()),
+            ("archive", ("archive",)),
+            ("dependency", ("dependencies",)),
+            ("dependency-package", ("dependencies", "packages", 0)),
+            ("migration", ("migrations",)),
+            ("migration-leaf", ("migrations", "leaves", 0)),
+            ("runtime", ("runtime",)),
+            ("runtime-version", ("runtime", "versions")),
+            ("source", ("source",)),
+            ("source-entry", ("source", "tracked", 0)),
+            ("static", ("static",)),
+            ("static-collected", ("static", "collected", 0)),
+            ("static-origin", ("static", "origins", 0)),
+            ("static-tracked", ("static", "tracked", 0)),
+        )
+        for name, path in cases:
+            with self.subTest(name=name), tempfile.TemporaryDirectory() as temporary:
+                fixture = self._runtime_fixture(temporary)
+                fixture.add_venv()
+                manifest = json.loads(fixture.manifest.read_text(encoding="utf-8"))
+                nested_extra(manifest, path)
+                fixture.write_manifest(manifest)
+                fixture.expected_manifest_sha256 = fixture.manifest_sha256
+                self._assert_prepare_rejected(fixture)
+
+        with self.subTest(name="venv-static"), tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            manifest = json.loads(fixture.manifest.read_text(encoding="utf-8"))
+            manifest["static"]["collected"][0]["path"] = ".venv/site.css"
+            manifest["static"]["origins"][0]["collected_path"] = ".venv/site.css"
+            manifest["static"]["tracked"][0]["collected_path"] = ".venv/site.css"
+            for key in ("collected", "tracked"):
+                manifest["static"][f"{key}_set_sha256"] = _sha256(
+                    _canonical_json(manifest["static"][key])
+                )
+            fixture.write_manifest(manifest)
+            fixture.expected_manifest_sha256 = fixture.manifest_sha256
+            self._assert_prepare_rejected(fixture)
+
+    def test_prepare_rejects_bootstrap_under_replaceable_ancestor(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            unsafe = Path(temporary).resolve() / "replaceable"
+            unsafe.mkdir(mode=0o700)
+            copied_python = unsafe / "bootstrap-python"
+            shutil.copy2(fixture.bootstrap_python, copied_python)
+            copied_python.chmod(0o755)
+            unsafe.chmod(0o777)
+            arguments = fixture.verifier_arguments("prepare")
+            arguments = [
+                (
+                    str(copied_python)
+                    if index > 0 and value == str(fixture.bootstrap_python)
+                    else value
+                )
+                for index, value in enumerate(arguments)
+            ]
+            result = subprocess.run(
+                arguments,
+                cwd="/private/tmp",
+                env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=30,
+            )
+            self.assertEqual(result.returncode, 1)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(result.stderr, "runtime_receipt=invalid\n")
+
+    def test_prepare_rejects_release_and_receipt_replaceable_ancestors(self):
+        with self.subTest(boundary="release"), tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            root.chmod(0o777)
+            self._assert_prepare_rejected(fixture)
+
+        with self.subTest(boundary="receipt"), tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            receipt_parent = root / "replaceable-receipt-parent"
+            receipt_parent.mkdir(mode=0o700)
+            fixture.receipt = receipt_parent / "runtime-receipt.json"
+            receipt_parent.chmod(0o777)
+            self._assert_prepare_rejected(fixture)
+
+    def test_bootstrap_identity_rejects_an_untrusted_executable_owner(self):
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        bootstrap = Path(sys.executable).resolve()
+        bootstrap_owner = bootstrap.stat().st_uid
+        if bootstrap_owner == 0:
+            self.assertIn(
+                "allowed_owner_uids={0, trusted_owner_uid}",
+                (settings.BASE_DIR / "scripts" / "verify-release-runtime").read_text(
+                    encoding="utf-8"
+                ),
+            )
+            return
+        result = subprocess.run(
+            [
+                sys.executable,
+                "-I",
+                "-S",
+                "-B",
+                "-c",
+                (
+                    "import runpy,sys;"
+                    "m=runpy.run_path(sys.argv[1]);"
+                    "failed=False;"
+                    "\ntry:\n"
+                    " m['bootstrap_identity'](sys.argv[2],sys.argv[3],"
+                    "sys.argv[6],sys.argv[7],sys.argv[8],sys.argv[9],"
+                    "{'python':sys.argv[4]},trusted_owner_uid=int(sys.argv[5]),"
+                    "require_root_tree=False)\n"
+                    "except m['VerificationError']:\n failed=True\n"
+                    "raise SystemExit(0 if failed else 1)"
+                ),
+                str(verifier_path),
+                str(bootstrap),
+                _sha256(bootstrap.read_bytes()),
+                ".".join(str(part) for part in sys.version_info[:3]),
+                str(bootstrap_owner + 1),
+                str(bootstrap.parents[1]),
+                str(Path(tempfile.gettempdir()).resolve() / "not-used-tree-spec"),
+                "c" * 64,
+                "d" * 64,
+            ],
+            cwd="/private/tmp",
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+                "TZ": "UTC",
+            },
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=30,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stdout, "")
+
+    def test_verifier_child_processes_use_explicit_environments_and_owner_model(self):
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        source = verifier_path.read_text(encoding="utf-8")
+        tree = ast.parse(source)
+        subprocess_calls = [
+            node
+            for node in ast.walk(tree)
+            if isinstance(node, ast.Call)
+            and isinstance(node.func, ast.Attribute)
+            and isinstance(node.func.value, ast.Name)
+            and node.func.value.id == "subprocess"
+            and node.func.attr == "run"
+        ]
+        self.assertTrue(subprocess_calls)
+        for call in subprocess_calls:
+            self.assertIn("env", {keyword.arg for keyword in call.keywords})
+        self.assertIn("trusted_owner_uid", source)
+        ownership_functions = {
+            node.name: ast.get_source_segment(source, node) or ""
+            for node in tree.body
+            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef))
+            and node.name
+            in {
+                "safe_status",
+                "scan_source_tree",
+                "verify_files",
+                "safe_inventory_status",
+                "inventory_venv",
+            }
+        }
+        self.assertEqual(len(ownership_functions), 5)
+        for function_source in ownership_functions.values():
+            self.assertNotIn("os.geteuid", function_source)
+        self.assertIn("0o444", source)
+
     def test_static_root_is_outside_tracked_source_assets(self):
         self.assertEqual(settings.STATIC_ROOT, settings.BASE_DIR / "staticfiles")
         self.assertFalse(
@@ -74,12 +1521,32 @@ class RuntimeConfigTests(SimpleTestCase):
 
         self.assertTrue(_RedactedGunicornFilter().filter(record))
         payload = json.loads(record.getMessage())
+        timestamp = payload.pop("timestamp")
+        self.assertRegex(
+            timestamp,
+            r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
+        )
         self.assertEqual(
             payload,
-            {"event": "gunicorn", "level": "ERROR", "redacted": True},
+            {
+                "event": "gunicorn",
+                "release_sha": bounded_release_sha(
+                    os.environ.get("TRAVEL_READINESS_RELEASE_SHA")
+                ),
+                "level": "ERROR",
+                "redacted": True,
+            },
         )
         self.assertNotIn(marker, repr(record.__dict__))
 
+        record = logging.LogRecord(
+            "gunicorn.error", logging.INFO, __file__, 1, "discarded", (), None
+        )
+        record.levelname = marker
+        _RedactedGunicornFilter().filter(record)
+        self.assertEqual(json.loads(record.getMessage())["level"], "OTHER")
+        self.assertNotIn(marker, record.getMessage())
+
     def test_configured_gunicorn_handler_emits_only_fixed_json(self):
         marker = "synthetic-uri-ip-header-marker"
         arguments = [
@@ -93,9 +1560,22 @@ class RuntimeConfigTests(SimpleTestCase):
             application = WSGIApplication()
             logger = application.cfg.logger_class(application.cfg)
             logger.warning("Invalid request from %s: %s", marker, marker)
+        payload = json.loads(captured.getvalue())
+        timestamp = payload.pop("timestamp")
+        self.assertRegex(
+            timestamp,
+            r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
+        )
         self.assertEqual(
-            captured.getvalue(),
-            '{"event":"gunicorn","level":"WARNING","redacted":true}\n',
+            payload,
+            {
+                "event": "gunicorn",
+                "release_sha": bounded_release_sha(
+                    os.environ.get("TRAVEL_READINESS_RELEASE_SHA")
+                ),
+                "level": "WARNING",
+                "redacted": True,
+            },
         )
         self.assertNotIn(marker, captured.getvalue())
 
@@ -131,12 +1611,35 @@ class RuntimeConfigTests(SimpleTestCase):
     def test_startup_script_uses_only_the_pinned_config_and_wsgi_app(self):
         script_path = settings.BASE_DIR / "scripts" / "run-production"
         script = script_path.read_text(encoding="utf-8")
+        verifier = (
+            settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        ).read_text(encoding="utf-8")
+        self.assertTrue(script.startswith("#!/bin/sh -p\n"))
+        self.assertTrue(
+            script.startswith(
+                "#!/bin/sh -p\nPATH=/usr/bin:/bin\nexport PATH\nset +x\n"
+            )
+        )
+        self.assertLess(
+            script.index("unset BASH_ENV CDPATH ENV"),
+            script.index("/usr/bin/stat"),
+        )
         self.assertEqual(stat.S_IMODE(script_path.stat().st_mode), 0o755)
-        self.assertIn('"$python_bin" -I -m gunicorn', script)
-        self.assertIn('--chdir "$project_dir"', script)
-        self.assertIn('"$project_dir/runtime/gunicorn.conf.py"', script)
-        self.assertIn("travel_readiness.wsgi:application", script)
-        self.assertIn("DJANGO_SETTINGS_MODULE=travel_readiness.settings", script)
+        self.assertIn("/usr/bin/env -i", script)
+        self.assertIn("/usr/bin/mkfifo -m 600", script)
+        self.assertIn("runtime_values >&5", script)
+        self.assertNotIn("runtime_values |", script)
+        self.assertIn('"$bootstrap_python" -I -S -B "$verifier" launch', script)
+        self.assertIn('script_dir=${0%/*}', script)
+        self.assertNotIn("dirname", script)
+        self.assertIn("???) boundary_permissions=0$boundary_mode", script)
+        self.assertIn("from gunicorn.app.wsgiapp import run", verifier)
+        self.assertIn('sys.argv = ["gunicorn", *sys.argv[3:]]', verifier)
+        self.assertNotIn('"-m",\n                "gunicorn"', verifier)
+        self.assertNotIn('[str(python_path), "-I", "-B"', verifier)
+        self.assertIn('"DEPENDENCY_ENVIRONMENT_MUTATED"', verifier)
+        self.assertIn('"travel_readiness.wsgi:application"', verifier)
+        self.assertIn('"DJANGO_SETTINGS_MODULE": "travel_readiness.settings"', verifier)
         self.assertIn("GUNICORN_CMD_ARGS_FORBIDDEN", script)
         for dependency in (
             '"Django": "5.2.17"',
@@ -144,16 +1647,579 @@ class RuntimeConfigTests(SimpleTestCase):
             '"psycopg": "3.3.4"',
             '"psycopg-binary": "3.3.4"',
         ):
-            self.assertIn(dependency, script)
-        self.assertIn('call_command("check", deploy=True', script)
-        self.assertIn('call_command("migrate", check=True', script)
+            self.assertIn(dependency, verifier)
+        self.assertIn('call_command("check", deploy=True', verifier)
+        self.assertIn('call_command("migrate", check=True', verifier)
         self.assertIn("unset MOFA_TRAVEL_ALARM_SERVICE_KEY", script)
-        self.assertIn("DIRECT_TLS_REQUIRED", script)
-        self.assertIn("TLS_MATERIAL_INVALID", script)
-        self.assertIn("LOOPBACK_BIND_REQUIRED", script)
+        self.assertIn("unset TRAVEL_READINESS_RELEASE_SHA", script)
+        self.assertIn("PYTHONDONTWRITEBYTECODE=1", script)
+        self.assertIn("RELEASE_RUNTIME_INVALID", script)
+        for anchor in (
+            "TRAVEL_READINESS_BOOTSTRAP_PREFIX",
+            "TRAVEL_READINESS_BOOTSTRAP_PRISTINE_TREE_SHA256",
+            "TRAVEL_READINESS_BOOTSTRAP_PYTHON_SHA256",
+            "TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_FILE",
+            "TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_SHA256",
+            "TRAVEL_READINESS_DEPENDENCY_ENVIRONMENT_SHA256",
+            "TRAVEL_READINESS_EXPECTED_MANIFEST_SHA256",
+            "TRAVEL_READINESS_EXPECTED_RELEASE_SHA",
+            "TRAVEL_READINESS_RUNTIME_RECEIPT_SHA256",
+            "TRAVEL_READINESS_VERIFIER_SHA256",
+        ):
+            self.assertIn(anchor, script)
         for forbidden in ("--access-logfile", "--reload", "--preload", "eval"):
             self.assertNotIn(forbidden, script)
 
+    def test_startup_freezes_the_full_bootstrap_tree_before_python(self):
+        script = (
+            settings.BASE_DIR / "scripts" / "run-production"
+        ).read_text(encoding="utf-8")
+        verifier = (
+            settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        ).read_text(encoding="utf-8")
+        tree_scan = script.index('/usr/bin/find "$bootstrap_prefix"')
+        tree_verify = script.index(
+            '/usr/sbin/mtree -f "$bootstrap_tree_spec" -p "$bootstrap_prefix"'
+        )
+        verifier_start = script.index(
+            '"$bootstrap_python" -I -S -B "$verifier" launch'
+        )
+        self.assertLess(tree_scan, tree_verify)
+        self.assertLess(tree_verify, verifier_start)
+        self.assertIn("! -uid 0", script)
+        self.assertIn("-perm -020", script)
+        self.assertIn("-perm -002", script)
+        self.assertIn(
+            "pinned_bootstrap_tree_sha256="
+            "28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110",
+            script,
+        )
+        self.assertIn(
+            "pinned_bootstrap_mtree_sha256="
+            "939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1",
+            script,
+        )
+        self.assertIn('bootstrap_spec_status" != 0:444:1', script)
+        self.assertIn("expected_tree_spec_header=", script)
+        self.assertIn(
+            "allowed_tree_owners = {0} if require_root_tree",
+            verifier,
+        )
+        self.assertIn("require_root_tree=mode == \"launch\"", verifier)
+        execute_start = verifier.index("def execute(")
+        self.assertLess(
+            verifier.index("runtime_principal(trusted_owner_uid)", execute_start),
+            verifier.index("verify_source(", execute_start),
+        )
+        self.assertIn('"pristine_tree_sha256"', verifier)
+        self.assertIn('"tree_spec_sha256"', verifier)
+        self.assertIn(
+            '"tree_spec_format": PINNED_BOOTSTRAP_MTREE_SCHEMA',
+            verifier,
+        )
+
+    def test_mtree_contract_rejects_content_topology_and_metadata_mutations(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            tree = root / "python"
+            tree.mkdir(mode=0o755)
+            payload = tree / "payload"
+            payload.write_bytes(b"pinned\n")
+            payload.chmod(0o644)
+            link = tree / "payload-link"
+            link.symlink_to("payload")
+            generated = subprocess.run(
+                [
+                    "/usr/sbin/mtree",
+                    "-c",
+                    "-n",
+                    "-k",
+                    "type,mode,nlink,sha256digest,link",
+                    "-p",
+                    str(tree),
+                ],
+                cwd="/private/tmp",
+                env={
+                    "LANG": "C",
+                    "LC_ALL": "C",
+                    "PATH": "/usr/bin:/bin",
+                    "TZ": "UTC",
+                },
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                check=False,
+                timeout=10,
+            )
+            self.assertEqual(generated.returncode, 0, generated.stderr)
+            spec = root / "bootstrap.mtree"
+            header = (
+                b"# travel-readiness-bootstrap-mtree-v1 "
+                b"python_tree_sha256=" + b"a" * 64 + b" "
+                b"python_executable_sha256=" + b"b" * 64 + b"\n"
+            )
+            spec.write_bytes(header + generated.stdout)
+
+            def replay() -> bool:
+                result = subprocess.run(
+                    [
+                        "/usr/sbin/mtree",
+                        "-f",
+                        str(spec),
+                        "-p",
+                        str(tree),
+                    ],
+                    cwd="/private/tmp",
+                    env={
+                        "LANG": "C",
+                        "LC_ALL": "C",
+                        "PATH": "/usr/bin:/bin",
+                        "TZ": "UTC",
+                    },
+                    stdin=subprocess.DEVNULL,
+                    stdout=subprocess.PIPE,
+                    stderr=subprocess.DEVNULL,
+                    check=False,
+                    timeout=10,
+                )
+                return result.returncode == 0 and result.stdout == b""
+
+            self.assertTrue(replay())
+            payload.write_bytes(b"mutated\n")
+            self.assertFalse(replay())
+            payload.write_bytes(b"pinned\n")
+            link.unlink()
+            link.symlink_to("missing-target")
+            self.assertFalse(replay())
+            link.unlink()
+            link.symlink_to("payload")
+            payload.chmod(0o600)
+            self.assertFalse(replay())
+            payload.chmod(0o644)
+            outside_link = root / "outside-hardlink"
+            os.link(payload, outside_link)
+            self.assertFalse(replay())
+            outside_link.unlink()
+            extra = tree / "extra"
+            extra.write_bytes(b"extra")
+            self.assertFalse(replay())
+            extra.unlink()
+            payload.rename(root / "missing-payload")
+            self.assertFalse(replay())
+            (root / "missing-payload").rename(payload)
+            self.assertTrue(replay())
+            pinned_spec_digest = _sha256(spec.read_bytes())
+            spec.write_bytes(b"# tampered-header\n" + generated.stdout)
+            self.assertNotEqual(_sha256(spec.read_bytes()), pinned_spec_digest)
+            body = bytearray(generated.stdout)
+            digest_offset = body.index(b"sha256digest=") + len(b"sha256digest=")
+            body[digest_offset] = ord("0") if body[digest_offset] != ord("0") else ord("1")
+            spec.write_bytes(header + bytes(body))
+            self.assertFalse(replay())
+
+    def test_bootstrap_provisioner_is_fresh_root_owned_and_replays_the_spec(self):
+        provisioner_path = (
+            settings.BASE_DIR / "scripts" / "provision-runtime-bootstrap"
+        )
+        provisioner = provisioner_path.read_text(encoding="utf-8")
+        self.assertEqual(stat.S_IMODE(provisioner_path.stat().st_mode), 0o755)
+        self.assertTrue(provisioner.startswith("#!/bin/sh -p\n"))
+        self.assertIn('"$(/usr/bin/id -u 2>/dev/null)" != 0', provisioner)
+        self.assertIn("/bin/cp -R -P -p", provisioner)
+        self.assertIn("/usr/sbin/chown -R 0:0", provisioner)
+        self.assertIn('/bin/chmod 0444 "$stage_spec"', provisioner)
+        self.assertGreaterEqual(provisioner.count("/usr/sbin/mtree -f"), 3)
+        self.assertIn("pinned_mtree_sha256=939d99ae", provisioner)
+        self.assertNotIn("python3.14 -", provisioner)
+
+    def test_startup_signal_handlers_cover_child_start_and_reap_failures(self):
+        script = (
+            settings.BASE_DIR / "scripts" / "run-production"
+        ).read_text(encoding="utf-8")
+        child_start = script.index(
+            '"$bootstrap_python" -I -S -B "$verifier" launch'
+        )
+        wait_start = script.index('wait "$runtime_pid"', child_start)
+        active_child_region = script[child_start:wait_start]
+        self.assertLess(script.index("trap forward_hup HUP"), child_start)
+        self.assertLess(script.index("trap forward_interrupt INT"), child_start)
+        self.assertLess(script.index("trap forward_term TERM"), child_start)
+        self.assertLess(script.index("trap abort_runtime_pipe PIPE"), child_start)
+        self.assertNotIn("trap - ", active_child_region)
+        signal_tree = script[
+            script.index("signal_runtime_tree() {") :
+            script.index("terminate_runtime() {")
+        ]
+        self.assertIn(
+            'kill -"$runtime_signal" "-$runtime_pid"', signal_tree
+        )
+        self.assertIn(
+            'kill -"$runtime_signal" "$runtime_pid"', signal_tree
+        )
+        terminate = script[
+            script.index("terminate_runtime() {") :
+            script.index("abort_runtime_pipe() {")
+        ]
+        self.assertIn("signal_runtime_tree TERM", terminate)
+        self.assertIn("signal_runtime_tree KILL", terminate)
+        self.assertIn('wait "$runtime_pid"', terminate)
+        self.assertIn('"$termination_attempt" -lt 50', terminate)
+        self.assertIn("runtime_identity_is_current", terminate)
+        self.assertIn("runtime_group_live_inventory", terminate)
+        self.assertIn("runtime_group_inventory_is_subset", terminate)
+        self.assertNotIn("last_async_pid", terminate)
+        self.assertNotIn("$!", terminate)
+        abort = script[
+            script.index("abort_runtime_pipe() {") :
+            script.index("forward_hup() {")
+        ]
+        self.assertIn("trap '' HUP INT TERM PIPE", abort)
+        self.assertLess(
+            abort.index("close_runtime_descriptors"),
+            abort.index("terminate_runtime TERM"),
+        )
+        self.assertIn("terminate_runtime TERM", abort)
+        handler_region = script[
+            script.index("forward_hup() {") :
+            script.index("trap cleanup_runtime_pipe EXIT")
+        ]
+        self.assertNotIn("trap -", handler_region)
+        self.assertEqual(
+            handler_region.count("trap '' HUP INT TERM PIPE"), 3
+        )
+        self.assertIn('4>&9', script)
+        self.assertIn(
+            '[ "$runtime_ready_pid" != "$runtime_pid" ]', script
+        )
+        self.assertIn("capture_runtime_identity direct", script)
+        self.assertIn("capture_runtime_identity group", script)
+        self.assertIn("-o lstart=", script)
+
+        verifier = (
+            settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        ).read_text(encoding="utf-8")
+        isolate = verifier[
+            verifier.index("def isolate_runtime_process()") :
+            verifier.index("def validate_tls_material(")
+        ]
+        self.assertIn("os.setsid()", isolate)
+        self.assertIn("os.getpgrp() != runtime_pid", isolate)
+        self.assertIn("os.getsid(0) != runtime_pid", isolate)
+        self.assertIn("os.fstat(4)", isolate)
+
+    def test_runtime_process_isolation_acknowledges_exact_child_pid(self):
+        verifier_path = settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        code = (
+            "import os,runpy,sys\n"
+            "passed=int(sys.argv[2])\n"
+            "if passed != 4:\n"
+            " os.dup2(passed,4)\n"
+            " os.close(passed)\n"
+            "m=runpy.run_path(sys.argv[1])\n"
+            "m['isolate_runtime_process']()\n"
+            "assert os.getpid()==os.getpgrp()==os.getsid(0)\n"
+            "print('isolated=true')\n"
+        )
+        with tempfile.TemporaryDirectory() as temporary:
+            ready_path = Path(temporary).resolve() / "ready"
+            os.mkfifo(ready_path, 0o600)
+            ready_fd = os.open(ready_path, os.O_RDWR | os.O_NONBLOCK)
+            try:
+                process = subprocess.Popen(
+                    [
+                        sys.executable,
+                        "-I",
+                        "-S",
+                        "-B",
+                        "-c",
+                        code,
+                        str(verifier_path),
+                        str(ready_fd),
+                    ],
+                    cwd="/private/tmp",
+                    env={
+                        "LANG": "C",
+                        "LC_ALL": "C",
+                        "PATH": "/usr/bin:/bin",
+                        "PYTHONDONTWRITEBYTECODE": "1",
+                        "TZ": "UTC",
+                    },
+                    pass_fds=(ready_fd,),
+                    stdout=subprocess.PIPE,
+                    stderr=subprocess.PIPE,
+                    text=True,
+                )
+                stdout, stderr = process.communicate(timeout=15)
+                ready = os.read(ready_fd, 64)
+            finally:
+                os.close(ready_fd)
+        self.assertEqual(process.returncode, 0, stderr)
+        self.assertEqual(stdout, "isolated=true\n")
+        self.assertEqual(ready, f"{process.pid}\n".encode("ascii"))
+
+    def test_shutdown_ignores_second_signals_and_kills_orphan_worker_group(self):
+        startup = (
+            settings.BASE_DIR / "scripts" / "run-production"
+        ).read_text(encoding="utf-8")
+        functions = startup[
+            startup.index("close_runtime_descriptors() {") :
+            startup.index("trap cleanup_runtime_pipe EXIT")
+        ]
+        helper_source = (
+            "import os,signal,sys,time\n"
+            "os.setsid()\n"
+            "worker=os.fork()\n"
+            "if worker==0:\n"
+            " for name in ('SIGHUP','SIGINT','SIGTERM'):\n"
+            "  signal.signal(getattr(signal,name),signal.SIG_IGN)\n"
+            " while True: time.sleep(1)\n"
+            "with open(sys.argv[1],'w',encoding='ascii') as output:\n"
+            " output.write(f'{os.getpid()} {worker}\\n')\n"
+            " output.flush()\n"
+            " os.fsync(output.fileno())\n"
+            "while True: time.sleep(1)\n"
+        )
+        harness_source = (
+            "#!/bin/sh -p\n"
+            "PATH=/usr/bin:/bin\nexport PATH\nset -eu\n"
+            "runtime_pipe_dir=\nruntime_fifo=\nruntime_ack_fifo=\n"
+            "runtime_pid=\nruntime_identity=\nruntime_group_identity=\n"
+            "runtime_group_ready=0\n"
+            "runtime_owner_uid=$(/usr/bin/id -u)\n"
+            "runtime_identity_python=$1\n"
+            "cleanup_runtime_pipe() { :; }\n"
+            + functions
+            + "trap cleanup_runtime_pipe EXIT\n"
+            + "trap forward_hup HUP\n"
+            + "trap forward_interrupt INT\n"
+            + "trap forward_term TERM\n"
+            + "trap abort_runtime_pipe PIPE\n"
+            + '"$1" -I -S -B "$2" "$3" &\n'
+            + "runtime_pid=$!\n"
+            + 'while [ ! -s "$3" ]; do /bin/sleep 0.05; done\n'
+            + "capture_runtime_identity group\n"
+            + "runtime_group_ready=1\n"
+            + 'printf "%s\\n" "$runtime_pid" >"$4"\n'
+            + 'wait "$runtime_pid"\n'
+        )
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            helper = root / "runtime-tree.py"
+            harness = root / "signal-harness"
+            identities = root / "identities"
+            ready = root / "harness-ready"
+            helper.write_text(helper_source, encoding="utf-8")
+            harness.write_text(harness_source, encoding="utf-8")
+            harness.chmod(0o755)
+            process = subprocess.Popen(
+                [str(harness), sys.executable, str(helper), str(identities), str(ready)],
+                cwd="/private/tmp",
+                env={"LANG": "C", "LC_ALL": "C", "PATH": "/usr/bin:/bin"},
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                text=True,
+            )
+            leader_pid = worker_pid = None
+            try:
+                deadline = 10.0
+                while not ready.exists() and deadline > 0:
+                    import time
+
+                    time.sleep(0.05)
+                    deadline -= 0.05
+                self.assertTrue(ready.exists(), "runtime tree did not become ready")
+                leader_text, worker_text = identities.read_text(
+                    encoding="ascii"
+                ).split()
+                leader_pid, worker_pid = int(leader_text), int(worker_text)
+                self.assertEqual(
+                    ready.read_text(encoding="ascii"), f"{leader_pid}\n"
+                )
+                os.kill(process.pid, signal.SIGTERM)
+                time.sleep(0.25)
+                os.kill(process.pid, signal.SIGINT)
+                os.kill(process.pid, signal.SIGHUP)
+                stdout, stderr = process.communicate(timeout=12)
+                self.assertEqual(process.returncode, 143, stderr)
+                self.assertEqual(stdout, "")
+                self.assertEqual(stderr, "")
+                for runtime_pid in (leader_pid, worker_pid):
+                    for _attempt in range(40):
+                        try:
+                            os.kill(runtime_pid, 0)
+                        except ProcessLookupError:
+                            break
+                        time.sleep(0.05)
+                    else:
+                        self.fail(f"runtime process remained alive: {runtime_pid}")
+            finally:
+                if process.poll() is None:
+                    process.kill()
+                    process.wait(timeout=5)
+                if leader_pid is None and identities.exists():
+                    leader_text, worker_text = identities.read_text(
+                        encoding="ascii"
+                    ).split()
+                    leader_pid, worker_pid = int(leader_text), int(worker_text)
+                if leader_pid is not None:
+                    try:
+                        os.killpg(leader_pid, signal.SIGKILL)
+                    except (PermissionError, ProcessLookupError):
+                        pass
+
+    def test_shutdown_breaks_promptly_when_the_exact_group_exits_on_term(self):
+        startup = (
+            settings.BASE_DIR / "scripts" / "run-production"
+        ).read_text(encoding="utf-8")
+        functions = startup[
+            startup.index("close_runtime_descriptors() {") :
+            startup.index("trap cleanup_runtime_pipe EXIT")
+        ]
+        helper_source = (
+            "import os,signal,sys,time\n"
+            "os.setsid()\n"
+            "def stop(_number,_frame):\n"
+            " with open(sys.argv[2],'w',encoding='ascii') as output:\n"
+            "  output.write('TERM\\n')\n"
+            "  output.flush()\n"
+            "  os.fsync(output.fileno())\n"
+            " raise SystemExit(0)\n"
+            "signal.signal(signal.SIGTERM,stop)\n"
+            "with open(sys.argv[1],'w',encoding='ascii') as output:\n"
+            " output.write(f'{os.getpid()}\\n')\n"
+            " output.flush()\n"
+            " os.fsync(output.fileno())\n"
+            "while True: time.sleep(1)\n"
+        )
+        harness_source = (
+            "#!/bin/sh -p\n"
+            "PATH=/usr/bin:/bin\nexport PATH\nset -eu\n"
+            "runtime_pipe_dir=\nruntime_fifo=\nruntime_ack_fifo=\n"
+            "runtime_pid=\nruntime_identity=\nruntime_group_identity=\n"
+            "runtime_group_ready=0\n"
+            "runtime_owner_uid=$(/usr/bin/id -u)\n"
+            "runtime_identity_python=$1\n"
+            "cleanup_runtime_pipe() { :; }\n"
+            + functions
+            + '"$1" -I -S -B "$2" "$3" "$4" &\n'
+            + "runtime_pid=$!\n"
+            + 'while [ ! -s "$3" ]; do /bin/sleep 0.05; done\n'
+            + "capture_runtime_identity group\n"
+            + "runtime_group_ready=1\n"
+            + "terminate_runtime\n"
+        )
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            helper = root / "prompt-runtime.py"
+            harness = root / "prompt-harness"
+            ready = root / "ready"
+            received = root / "received"
+            helper.write_text(helper_source, encoding="utf-8")
+            harness.write_text(harness_source, encoding="utf-8")
+            harness.chmod(0o755)
+            import time
+
+            started = time.monotonic()
+            result = subprocess.run(
+                [
+                    str(harness),
+                    sys.executable,
+                    str(helper),
+                    str(ready),
+                    str(received),
+                ],
+                cwd="/private/tmp",
+                env={"LANG": "C", "LC_ALL": "C", "PATH": "/usr/bin:/bin"},
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                text=True,
+                check=False,
+                timeout=5,
+            )
+            elapsed = time.monotonic() - started
+            self.assertEqual(result.returncode, 0, result.stderr)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(result.stderr, "")
+            self.assertLess(elapsed, 2.0)
+            self.assertEqual(received.read_text(encoding="ascii"), "TERM\n")
+
+    def test_shutdown_refuses_a_simulated_reused_pid_identity(self):
+        startup = (
+            settings.BASE_DIR / "scripts" / "run-production"
+        ).read_text(encoding="utf-8")
+        functions = startup[
+            startup.index("close_runtime_descriptors() {") :
+            startup.index("trap cleanup_runtime_pipe EXIT")
+        ]
+        harness_source = (
+            "#!/bin/sh -p\n"
+            "PATH=/usr/bin:/bin\nexport PATH\nset -eu\n"
+            "runtime_pipe_dir=\nruntime_fifo=\nruntime_ack_fifo=\n"
+            "runtime_pid=$1\n"
+            "kill_marker=$2\n"
+            "runtime_owner_uid=$(/usr/bin/id -u)\n"
+            "runtime_identity=\"$1 $$ $1 $1 $runtime_owner_uid "
+            "Sun Aug 31 00:00:00 2026\"\n"
+            "runtime_group_identity=\nruntime_group_ready=1\n"
+            "cleanup_runtime_pipe() { :; }\n"
+            + functions
+            + "runtime_process_snapshot() {\n"
+            + " printf '%s\\n' \"$1 1 $1 $1 $runtime_owner_uid "
+            + "Sun Aug 31 00:00:01 2026\"\n"
+            + "}\n"
+            + "kill() { printf '%s\\n' called >\"$kill_marker\"; }\n"
+            + "terminate_runtime\n"
+        )
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            harness = root / "reuse-harness"
+            marker = root / "kill-called"
+            harness.write_text(harness_source, encoding="utf-8")
+            harness.chmod(0o755)
+            sentinel = subprocess.Popen(
+                ["/bin/sleep", "10"],
+                cwd="/private/tmp",
+                env={"LANG": "C", "LC_ALL": "C", "PATH": "/usr/bin:/bin"},
+                stdout=subprocess.DEVNULL,
+                stderr=subprocess.DEVNULL,
+            )
+            try:
+                result = subprocess.run(
+                    [str(harness), str(sentinel.pid), str(marker)],
+                    cwd="/private/tmp",
+                    env={"LANG": "C", "LC_ALL": "C", "PATH": "/usr/bin:/bin"},
+                    stdout=subprocess.PIPE,
+                    stderr=subprocess.PIPE,
+                    text=True,
+                    check=False,
+                    timeout=5,
+                )
+                self.assertEqual(result.returncode, 0, result.stderr)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, "")
+                self.assertIsNone(sentinel.poll())
+                self.assertFalse(marker.exists())
+            finally:
+                if sentinel.poll() is None:
+                    sentinel.terminate()
+                    sentinel.wait(timeout=5)
+
+    def test_receipt_write_fsyncs_file_and_parent_after_a_size_bound(self):
+        source = (
+            settings.BASE_DIR / "scripts" / "verify-release-runtime"
+        ).read_text(encoding="utf-8")
+        tree = ast.parse(source)
+        function = next(
+            node
+            for node in tree.body
+            if isinstance(node, ast.FunctionDef) and node.name == "write_receipt"
+        )
+        segment = ast.get_source_segment(source, function) or ""
+        self.assertIn("len(data) > RECEIPT_LIMIT", segment)
+        self.assertGreaterEqual(segment.count("os.fsync(fd)"), 2)
+        self.assertGreaterEqual(segment.count("os.fsync(parent_fd)"), 1)
+        self.assertIn("os.O_NOFOLLOW", segment)
+
     def test_startup_rejects_inherited_gunicorn_arguments_from_any_cwd(self):
         script_path = settings.BASE_DIR / "scripts" / "run-production"
         environment = os.environ.copy()
@@ -212,25 +2278,19 @@ class RuntimeConfigTests(SimpleTestCase):
         )
 
     def test_startup_rejects_invalid_tls_material_with_fixed_error(self):
-        script_path = settings.BASE_DIR / "scripts" / "run-production"
         with tempfile.TemporaryDirectory() as temporary:
             temporary_path = Path(temporary)
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
             cert_file = temporary_path / "invalid-cert.pem"
             key_file = temporary_path / "invalid-key.pem"
             cert_file.write_text("invalid-test-certificate\n")
             key_file.write_text("invalid-test-key\n")
-            key_file.chmod(0o600)
-            environment = os.environ.copy()
-            environment.pop("GUNICORN_CMD_ARGS", None)
-            environment["TRAVEL_READINESS_TLS_CERT_FILE"] = str(cert_file)
-            environment["TRAVEL_READINESS_TLS_KEY_FILE"] = str(key_file)
-            result = subprocess.run(
-                [str(script_path)],
-                cwd="/private/tmp",
-                env=environment,
-                capture_output=True,
-                text=True,
-                check=False,
+            key_file.chmod(0o440)
+            result = self._run_launch_unit(
+                fixture, cert_file, key_file
             )
         self.assertEqual(result.returncode, 64)
         self.assertEqual(result.stdout, "")
@@ -239,75 +2299,377 @@ class RuntimeConfigTests(SimpleTestCase):
             "startup_error=TLS_MATERIAL_INVALID\n",
         )
 
-    def test_startup_preflights_deployment_then_current_migrations(self):
-        script_path = settings.BASE_DIR / "scripts" / "run-production"
+    def test_tls_key_requires_immutable_group_read_contract_and_single_link(self):
+        def wrong_mode(_root: Path, _cert: Path, key: Path) -> None:
+            key.chmod(0o400)
+
+        def hardlinked_key(root: Path, _cert: Path, key: Path) -> None:
+            os.link(key, root / "candidate-key-alias.pem")
+
+        def hardlinked_cert(root: Path, cert: Path, _key: Path) -> None:
+            os.link(cert, root / "candidate-cert-alias.pem")
+
+        for name, mutate in (
+            ("key-mode", wrong_mode),
+            ("key-hardlink", hardlinked_key),
+            ("cert-hardlink", hardlinked_cert),
+        ):
+            with self.subTest(name=name), tempfile.TemporaryDirectory() as temporary:
+                root = Path(temporary).resolve()
+                fixture = self._runtime_fixture(temporary)
+                fixture.add_venv()
+                prepared = fixture.prepare_receipt()
+                self.assertEqual(prepared.returncode, 0, prepared.stderr)
+                cert_file, key_file = self._tls_pair(root)
+                mutate(root, cert_file, key_file)
+                result = self._run_launch_unit(
+                    fixture, cert_file, key_file
+                )
+                self.assertEqual(result.returncode, 64)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(
+                    result.stderr,
+                    "startup_error=TLS_MATERIAL_INVALID\n",
+                )
+
+    def test_startup_exports_only_manifest_verified_release_identity(self):
         with tempfile.TemporaryDirectory() as temporary:
             temporary_path = Path(temporary)
-            cert_file = temporary_path / "candidate-cert.pem"
-            key_file = temporary_path / "candidate-key.pem"
-            generated = subprocess.run(
-                [
-                    "openssl",
-                    "req",
-                    "-x509",
-                    "-newkey",
-                    "rsa:2048",
-                    "-nodes",
-                    "-days",
-                    "1",
-                    "-subj",
-                    "/CN=127.0.0.1",
-                    "-keyout",
-                    str(key_file),
-                    "-out",
-                    str(cert_file),
-                ],
+            marker = temporary_path / "observed-release-sha"
+            settings_source = f"""
+import os
+from pathlib import Path
+
+expected = {(_RuntimeReleaseFixture.release_sha)!r}
+if os.environ.get("TRAVEL_READINESS_RELEASE_SHA") != expected:
+    raise RuntimeError("unverified release identity")
+if os.environ.get("PYTHONDONTWRITEBYTECODE") != "1":
+    raise RuntimeError("bytecode boundary is not fixed")
+if os.environ.get("UNKNOWN_SECRET_MARKER") is not None:
+    raise RuntimeError("unknown inherited environment reached runtime")
+Path({str(marker)!r}).write_text(expected + "\\n", encoding="ascii")
+
+SECRET_KEY = "test-only-release-runtime-secret-value-with-enough-entropy"
+ALLOWED_HOSTS = ["candidate.invalid"]
+DEBUG = False
+INSTALLED_APPS = []
+MIDDLEWARE = []
+ROOT_URLCONF = __name__
+urlpatterns = []
+DATABASES = {{}}
+TEMPLATES = []
+USE_TZ = True
+SECURE_SSL_REDIRECT = True
+SECURE_HSTS_SECONDS = 31536000
+SECURE_HSTS_INCLUDE_SUBDOMAINS = True
+SECURE_HSTS_PRELOAD = True
+SECURE_CONTENT_TYPE_NOSNIFF = True
+SECURE_REFERRER_POLICY = "no-referrer"
+SECURE_CROSS_ORIGIN_OPENER_POLICY = "same-origin"
+SESSION_COOKIE_SECURE = True
+CSRF_COOKIE_SECURE = True
+""".lstrip().encode("utf-8")
+            fixture = self._runtime_fixture(
+                temporary,
+                settings_source=settings_source,
+            )
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(temporary_path)
+            result = self._run_launch_unit(
+                fixture,
+                cert_file,
+                key_file,
+                ambient_environment={
+                    "TRAVEL_READINESS_RELEASE_SHA": "b" * 40,
+                    "UNKNOWN_SECRET_MARKER": "must-not-reach-any-child",
+                },
+            )
+            self.assertEqual(result.returncode, 78)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr,
+                "startup_error=DEPLOYMENT_CHECK_FAILED\n",
+            )
+            self.assertEqual(marker.read_text(encoding="ascii"), "a" * 40 + "\n")
+
+            marker.unlink()
+            settings_file = fixture.source / "travel_readiness" / "settings.py"
+            settings_file.write_bytes(settings_file.read_bytes() + b"\n")
+            self._assert_verifier_rejected(fixture)
+            self.assertFalse(marker.exists())
+
+    def test_startup_ignores_poisoned_path_python_hooks_and_unknown_environment(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary, full_source=True)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(root)
+            marker = root / "poison-executed"
+            poison = root / "poison-bin"
+            poison.mkdir()
+            payload = (
+                "#!/bin/sh\n"
+                f"/usr/bin/touch {marker}\n"
+                "exit 97\n"
+            )
+            for name in ("dirname", "python", "shasum", "sha256sum"):
+                candidate = poison / name
+                candidate.write_text(payload, encoding="utf-8")
+                candidate.chmod(0o755)
+            (poison / "sitecustomize.py").write_text(
+                f"from pathlib import Path\nPath({str(marker)!r}).touch()\n",
+                encoding="utf-8",
+            )
+            shell_hook = poison / "shell-hook"
+            shell_hook.write_text(
+                f"/usr/bin/touch {marker}\n",
+                encoding="utf-8",
+            )
+            secret_marker = "unknown-parent-secret-must-not-reach-child"
+            environment = fixture.startup_environment(
+                cert_file,
+                key_file,
+                BASH_ENV=str(shell_hook),
+                ENV=str(shell_hook),
+                PATH=str(poison),
+                PYTHONPATH=str(poison),
+                UNKNOWN_SECRET_MARKER=secret_marker,
+            )
+            result = subprocess.run(
+                [str(fixture.startup)],
+                cwd="/private/tmp",
+                env=environment,
                 capture_output=True,
                 text=True,
                 check=False,
                 timeout=15,
             )
-            self.assertEqual(generated.returncode, 0)
-            key_file.chmod(0o600)
-            base_environment = os.environ.copy()
-            for name in (
-                "GUNICORN_CMD_ARGS",
-                "TRAVEL_READINESS_BUILD",
-                "MOFA_TRAVEL_ALARM_SERVICE_KEY",
-            ):
-                base_environment.pop(name, None)
-            base_environment.update(
-                {
-                    "TRAVEL_READINESS_SECRET_KEY": (
-                        "test-only-startup-secret-0123456789-"
-                        "ABCDEFGHIJKLMNOPQRSTUVWXYZ-not-a-credential"
-                    ),
-                    "TRAVEL_READINESS_DB_PASSWORD": (
-                        "test-only-database-value"
-                    ),
-                    "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
-                    "TRAVEL_READINESS_DB_PORT": "1",
-                    "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
-                    "TRAVEL_READINESS_TLS_CERT_FILE": str(cert_file),
-                    "TRAVEL_READINESS_TLS_KEY_FILE": str(key_file),
-                }
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr, "startup_error=BOOTSTRAP_RUNTIME_INVALID\n"
             )
+            self.assertFalse(marker.exists())
+            self.assertNotIn(secret_marker, result.stdout + result.stderr)
+
+    def test_startup_never_executes_venv_python_before_receipt_verification(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary, full_source=True)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            marker = root / "untrusted-venv-python-executed"
+            python = fixture.source / ".venv" / "bin" / "python"
+            python.unlink()
+            python.write_text(
+                "#!/bin/sh\n"
+                f"/usr/bin/touch {marker}\n"
+                "exit 97\n",
+                encoding="utf-8",
+            )
+            python.chmod(0o755)
+            self._assert_verifier_rejected(fixture)
+            self.assertFalse(marker.exists())
+
+    def test_startup_rejects_tampered_verifier_before_executing_it(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(root)
+            environment = fixture.startup_environment(cert_file, key_file)
+            marker = root / "tampered-verifier-executed"
+            fixture.verifier.write_text(
+                "#!/bin/sh\n"
+                f"/usr/bin/touch {marker}\n"
+                "exit 97\n",
+                encoding="utf-8",
+            )
+            fixture.verifier.chmod(0o755)
+            result = subprocess.run(
+                [str(fixture.startup)],
+                cwd="/private/tmp",
+                env=environment,
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=15,
+            )
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr, "startup_error=BOOTSTRAP_RUNTIME_INVALID\n"
+            )
+            self.assertFalse(marker.exists())
+
+    def test_runtime_disables_pth_hooks_and_bytecode_mutation(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary, full_source=True)
+            fixture.add_venv()
+            marker = root / "venv-pth-executed"
+            site_packages = next((fixture.source / ".venv").rglob("site-packages"))
+            (site_packages / "startup-hook.pth").write_text(
+                f"import pathlib; pathlib.Path({str(marker)!r}).touch()\n",
+                encoding="utf-8",
+            )
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            expected_environment = fixture.dependency_environment_sha256
+            cert_file, key_file = self._tls_pair(root)
+            result = self._run_launch_unit(
+                fixture, cert_file, key_file
+            )
+            self.assertEqual(result.returncode, 78)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr, "startup_error=MIGRATION_CHECK_FAILED\n"
+            )
+            self.assertFalse(marker.exists())
+            self.assertEqual(
+                fixture.dependency_environment_sha256,
+                expected_environment,
+            )
+
+    def test_runtime_detects_dependency_environment_mutation_by_preflight(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            mutation_target = root / "release" / "source" / ".venv" / "pyvenv.cfg"
+            original_settings = (
+                settings.BASE_DIR / "travel_readiness" / "settings.py"
+            ).read_bytes()
+            mutation = (
+                "from pathlib import Path\n"
+                f"Path({str(mutation_target)!r}).write_text('mutated\\n', encoding='utf-8')\n"
+            ).encode("utf-8")
+            fixture = self._runtime_fixture(
+                temporary,
+                full_source=True,
+                settings_source=mutation + original_settings,
+            )
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(root)
+            result = self._run_launch_unit(
+                fixture, cert_file, key_file
+            )
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr,
+                "startup_error=DEPENDENCY_ENVIRONMENT_MUTATED\n",
+            )
+
+    def test_startup_rejects_unanchored_bootstrap_without_executing_it(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(root)
+            marker = root / "unanchored-bootstrap-executed"
+            bootstrap = root / "unanchored-bootstrap"
+            bootstrap.write_text(
+                "#!/bin/sh\n"
+                f"/usr/bin/touch {marker}\n"
+                "exit 97\n",
+                encoding="utf-8",
+            )
+            bootstrap.chmod(0o755)
+            environment = fixture.startup_environment(
+                cert_file,
+                key_file,
+                TRAVEL_READINESS_BOOTSTRAP_PYTHON=str(bootstrap),
+            )
+            result = subprocess.run(
+                [str(fixture.startup)],
+                cwd="/private/tmp",
+                env=environment,
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=15,
+            )
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr, "startup_error=BOOTSTRAP_RUNTIME_INVALID\n"
+            )
+            self.assertFalse(marker.exists())
+
+    def test_startup_rejects_bootstrap_in_replaceable_ancestor_before_execution(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary).resolve()
+            fixture = self._runtime_fixture(temporary)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(root)
+            marker = root / "replaceable-bootstrap-executed"
+            replaceable = root / "replaceable"
+            replaceable.mkdir(mode=0o700)
+            bootstrap = replaceable / "bootstrap"
+            bootstrap.write_text(
+                "#!/bin/sh\n"
+                f"/usr/bin/touch {marker}\n"
+                "exit 97\n",
+                encoding="utf-8",
+            )
+            bootstrap.chmod(0o755)
+            bootstrap_digest = _sha256(bootstrap.read_bytes())
+            replaceable.chmod(0o777)
+            environment = fixture.startup_environment(
+                cert_file,
+                key_file,
+                TRAVEL_READINESS_BOOTSTRAP_PYTHON=str(bootstrap),
+                TRAVEL_READINESS_BOOTSTRAP_PYTHON_SHA256=bootstrap_digest,
+            )
+            result = subprocess.run(
+                [str(fixture.startup)],
+                cwd="/private/tmp",
+                env=environment,
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=15,
+            )
+            self.assertEqual(result.returncode, 64)
+            self.assertEqual(result.stdout, "")
+            self.assertEqual(
+                result.stderr, "startup_error=BOOTSTRAP_RUNTIME_INVALID\n"
+            )
+            self.assertFalse(marker.exists())
+
+    def test_startup_preflights_deployment_then_current_migrations(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            fixture = self._runtime_fixture(temporary, full_source=True)
+            fixture.add_venv()
+            prepared = fixture.prepare_receipt()
+            self.assertEqual(prepared.returncode, 0, prepared.stderr)
+            cert_file, key_file = self._tls_pair(temporary_path)
             cases = (
                 ("0", "startup_error=DEPLOYMENT_CHECK_FAILED\n"),
                 ("1", "startup_error=MIGRATION_CHECK_FAILED\n"),
             )
             for https_flag, expected_error in cases:
                 with self.subTest(https_flag=https_flag):
-                    environment = base_environment.copy()
-                    environment["TRAVEL_READINESS_HTTPS"] = https_flag
-                    result = subprocess.run(
-                        [str(script_path)],
-                        cwd="/private/tmp",
-                        env=environment,
-                        capture_output=True,
-                        text=True,
-                        check=False,
-                        timeout=15,
+                    result = self._run_launch_unit(
+                        fixture,
+                        cert_file,
+                        key_file,
+                        runtime_values={
+                            "TRAVEL_READINESS_HTTPS": https_flag
+                        },
                     )
                     self.assertEqual(result.returncode, 78)
                     self.assertEqual(result.stdout, "")
diff --git a/runtime/versions.toml b/runtime/versions.toml
index 6350928..3244b17 100644
--- a/runtime/versions.toml
+++ b/runtime/versions.toml
@@ -15,6 +15,9 @@ python_macos_arm64_asset = "cpython-3.14.7+20260825-aarch64-apple-darwin-install
 python_macos_arm64_archive_url = "https://github.com/astral-sh/python-build-standalone/releases/download/20260825/cpython-3.14.7%2B20260825-aarch64-apple-darwin-install_only_stripped.tar.gz"
 python_macos_arm64_archive_size = 26488085
 python_macos_arm64_archive_sha256 = "17ecb3d29c49765856370cbb47d948a24ec518be40f607362c1bbb3ebbc5c442"
+python_macos_arm64_tree_sha256 = "28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110"
+python_macos_arm64_mtree_schema = "travel-readiness-bootstrap-mtree-v1"
+python_macos_arm64_mtree_sha256 = "939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1"
 python_verification = "official-github-immutable-release-api-digest-local-sha256-and-signed-release-commit"
 uv_release_commit = "7938ca5d53dbb9c614a4a030df406e41ff101ab9"
 uv_macos_arm64_archive_sha256 = "14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d"
diff --git a/scripts/check-dependencies b/scripts/check-dependencies
index 0612876..859a2ff 100755
--- a/scripts/check-dependencies
+++ b/scripts/check-dependencies
@@ -52,6 +52,7 @@ MAX_ENVIRONMENT_TOTAL_BYTES = 256 * 1024 * 1024
 MAX_WHEELHOUSE_BYTES = 64 * 1024 * 1024
 MAX_PYTHON_TREE_ENTRIES = 5000
 MAX_PYTHON_TREE_BYTES = 256 * 1024 * 1024
+MAX_BOOTSTRAP_MTREE_BYTES = 1024 * 1024
 MIN_PYTHON_TREE_ENTRIES = 1000
 MIN_PYTHON_TREE_BYTES = 32 * 1024 * 1024
 NETWORK_TIMEOUT_SECONDS = 5
@@ -77,6 +78,11 @@ EXPECTED_PYTHON_ARCHIVE_SIZE = 26488085
 EXPECTED_PYTHON_RELEASE_COMMIT = (
     "c0aa3bbdc2fff56a77ad1ecec68b1e47794d8779"
 )
+BOOTSTRAP_MTREE_SCHEMA = "travel-readiness-bootstrap-mtree-v1"
+EXPECTED_BOOTSTRAP_MTREE_SHA256 = (
+    "939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1"
+)
+MTREE = Path("/usr/sbin/mtree")
 EXPECTED_PYTHON_ASSET = (
     "cpython-3.14.7+20260825-aarch64-apple-darwin-"
     "install_only_stripped.tar.gz"
@@ -158,6 +164,7 @@ EXPECTED_PLATFORM_POLICY = {
         "python_tree_sha256": (
             "28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110"
         ),
+        "python_mtree_sha256": EXPECTED_BOOTSTRAP_MTREE_SHA256,
         "dependency_environment_sha256": (
             "ade0e58f6e5e3cc08d4c2a2631bea93fae5c1d86b460971afd34e3de3119ee69"
         ),
@@ -440,6 +447,29 @@ def safe_new_environment_path(value: str, *, source: Path) -> Path:
     return candidate
 
 
+def safe_new_bootstrap_tree_spec_path(
+    value: str,
+) -> tuple[Path, tuple[int, int]]:
+    candidate = Path(value)
+    if (
+        not candidate.is_absolute()
+        or ".." in candidate.parts
+        or candidate.name in ("", ".", "..")
+        or os.path.lexists(candidate)
+    ):
+        fail()
+    parent = candidate.parent
+    chain = absolute_path_chain(parent)
+    parent_status = chain[-1][1]
+    if (
+        not stat.S_ISDIR(parent_status.st_mode)
+        or parent_status.st_uid != os.geteuid()
+        or stat.S_IMODE(parent_status.st_mode) != 0o700
+    ):
+        fail()
+    return candidate, (parent_status.st_dev, parent_status.st_ino)
+
+
 def open_bounded_regular_file(path: Path, *, maximum: int) -> bytes:
     flags = os.O_RDONLY
     if hasattr(os, "O_CLOEXEC"):
@@ -793,6 +823,12 @@ def verify_tool_provenance(
         != EXPECTED_PYTHON_ARCHIVE_SIZE
         or integrity.get("python_macos_arm64_archive_sha256")
         != EXPECTED_PYTHON_ARCHIVE_SHA256
+        or integrity.get("python_macos_arm64_tree_sha256")
+        != policy["python_tree_sha256"]
+        or integrity.get("python_macos_arm64_mtree_schema")
+        != BOOTSTRAP_MTREE_SCHEMA
+        or integrity.get("python_macos_arm64_mtree_sha256")
+        != policy["python_mtree_sha256"]
         or integrity.get("python_verification")
         != (
             "official-github-immutable-release-api-digest-local-sha256-"
@@ -837,6 +873,13 @@ def parse_runtime(
         "python_macos_arm64_archive_url": EXPECTED_PYTHON_ASSET_URL,
         "python_macos_arm64_archive_size": EXPECTED_PYTHON_ARCHIVE_SIZE,
         "python_macos_arm64_archive_sha256": EXPECTED_PYTHON_ARCHIVE_SHA256,
+        "python_macos_arm64_tree_sha256": current_platform_policy()[
+            "python_tree_sha256"
+        ],
+        "python_macos_arm64_mtree_schema": BOOTSTRAP_MTREE_SCHEMA,
+        "python_macos_arm64_mtree_sha256": current_platform_policy()[
+            "python_mtree_sha256"
+        ],
         "python_verification": (
             "official-github-immutable-release-api-digest-local-sha256-"
             "and-signed-release-commit"
@@ -1244,6 +1287,267 @@ def run_fixed(
     return b"".join(stdout_chunks)
 
 
+def fixed_mtree_environment() -> dict[str, str]:
+    return {
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "TZ": "UTC",
+    }
+
+
+def bootstrap_mtree_header(policy: dict[str, object]) -> bytes:
+    tree_digest = policy.get("python_tree_sha256")
+    executable_digest = policy.get("python_executable_sha256")
+    if (
+        not isinstance(tree_digest, str)
+        or not re.fullmatch(r"[0-9a-f]{64}", tree_digest)
+        or not isinstance(executable_digest, str)
+        or not re.fullmatch(r"[0-9a-f]{64}", executable_digest)
+    ):
+        fail()
+    return (
+        f"# {BOOTSTRAP_MTREE_SCHEMA} python_tree_sha256={tree_digest} "
+        f"python_executable_sha256={executable_digest}\n"
+    ).encode("ascii", "strict")
+
+
+def write_bootstrap_tree_spec(
+    path: Path,
+    content: bytes,
+    *,
+    parent_identity: tuple[int, int],
+) -> tuple[int, int]:
+    parent_flags = os.O_RDONLY
+    file_flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL
+    if hasattr(os, "O_CLOEXEC"):
+        parent_flags |= os.O_CLOEXEC
+        file_flags |= os.O_CLOEXEC
+    if hasattr(os, "O_DIRECTORY"):
+        parent_flags |= os.O_DIRECTORY
+    if hasattr(os, "O_NOFOLLOW"):
+        parent_flags |= os.O_NOFOLLOW
+        file_flags |= os.O_NOFOLLOW
+    parent_descriptor: int | None = None
+    descriptor: int | None = None
+    created_identity: tuple[int, int] | None = None
+    try:
+        parent_descriptor = os.open(path.parent, parent_flags)
+        parent_status = os.fstat(parent_descriptor)
+        if (
+            not stat.S_ISDIR(parent_status.st_mode)
+            or parent_status.st_uid != os.geteuid()
+            or stat.S_IMODE(parent_status.st_mode) != 0o700
+            or (parent_status.st_dev, parent_status.st_ino) != parent_identity
+        ):
+            fail()
+        descriptor = os.open(
+            path.name,
+            file_flags,
+            0o400,
+            dir_fd=parent_descriptor,
+        )
+        initial_status = os.fstat(descriptor)
+        created_identity = (initial_status.st_dev, initial_status.st_ino)
+        if (
+            not stat.S_ISREG(initial_status.st_mode)
+            or initial_status.st_uid != os.geteuid()
+            or initial_status.st_nlink != 1
+            or stat.S_IMODE(initial_status.st_mode) != 0o400
+        ):
+            fail()
+        offset = 0
+        while offset < len(content):
+            written = os.write(descriptor, content[offset : offset + 64 * 1024])
+            if written <= 0:
+                fail()
+            offset += written
+        os.fsync(descriptor)
+        os.fchmod(descriptor, 0o444)
+        os.fsync(descriptor)
+        final_status = os.fstat(descriptor)
+        if (
+            (final_status.st_dev, final_status.st_ino) != created_identity
+            or not stat.S_ISREG(final_status.st_mode)
+            or final_status.st_uid != os.geteuid()
+            or final_status.st_nlink != 1
+            or stat.S_IMODE(final_status.st_mode) != 0o444
+            or final_status.st_size != len(content)
+        ):
+            fail()
+        os.fsync(parent_descriptor)
+        return created_identity
+    except BaseException as error:
+        if descriptor is not None:
+            try:
+                os.close(descriptor)
+            except OSError:
+                pass
+            descriptor = None
+        if parent_descriptor is not None and created_identity is not None:
+            try:
+                observed = os.stat(
+                    path.name,
+                    dir_fd=parent_descriptor,
+                    follow_symlinks=False,
+                )
+                if (
+                    stat.S_ISREG(observed.st_mode)
+                    and observed.st_uid == os.geteuid()
+                    and observed.st_nlink == 1
+                    and (observed.st_dev, observed.st_ino) == created_identity
+                ):
+                    os.unlink(path.name, dir_fd=parent_descriptor)
+                    os.fsync(parent_descriptor)
+            except OSError:
+                pass
+        if isinstance(error, OSError):
+            fail()
+        raise
+    finally:
+        if descriptor is not None:
+            try:
+                os.close(descriptor)
+            except OSError:
+                fail()
+        if parent_descriptor is not None:
+            try:
+                os.close(parent_descriptor)
+            except OSError:
+                fail()
+
+
+def remove_owned_bootstrap_tree_spec(
+    path: Path,
+    *,
+    identity: tuple[int, int],
+    parent_identity: tuple[int, int],
+) -> None:
+    parent_flags = os.O_RDONLY
+    if hasattr(os, "O_CLOEXEC"):
+        parent_flags |= os.O_CLOEXEC
+    if hasattr(os, "O_DIRECTORY"):
+        parent_flags |= os.O_DIRECTORY
+    if hasattr(os, "O_NOFOLLOW"):
+        parent_flags |= os.O_NOFOLLOW
+    parent_descriptor: int | None = None
+    try:
+        parent_descriptor = os.open(path.parent, parent_flags)
+        parent_status = os.fstat(parent_descriptor)
+        observed = os.stat(
+            path.name,
+            dir_fd=parent_descriptor,
+            follow_symlinks=False,
+        )
+        if (
+            (parent_status.st_dev, parent_status.st_ino) != parent_identity
+            or not stat.S_ISREG(observed.st_mode)
+            or observed.st_uid != os.geteuid()
+            or observed.st_nlink != 1
+            or (observed.st_dev, observed.st_ino) != identity
+        ):
+            fail()
+        os.unlink(path.name, dir_fd=parent_descriptor)
+        os.fsync(parent_descriptor)
+    except OSError:
+        fail()
+    finally:
+        if parent_descriptor is not None:
+            try:
+                os.close(parent_descriptor)
+            except OSError:
+                fail()
+
+
+def generate_bootstrap_tree_spec(
+    python: Path,
+    output: Path,
+    *,
+    output_parent_identity: tuple[int, int],
+) -> str:
+    policy = current_platform_policy()
+    mtree = safe_absolute_path(str(MTREE), kind="executable")
+    try:
+        mtree_status = mtree.lstat()
+    except OSError:
+        fail()
+    if (
+        mtree_status.st_uid != 0
+        or mtree_status.st_nlink != 1
+        or mtree_status.st_mode & (stat.S_IWGRP | stat.S_IWOTH)
+    ):
+        fail()
+    python_root = private_root_for(python)
+    artifact_root = python_root / Path(str(policy["python_relative"])).parents[1]
+    if (
+        sha256_regular_file(python, maximum=128 * 1024 * 1024)
+        != policy.get("python_executable_sha256")
+        or python_artifact_tree_sha256(artifact_root)
+        != policy.get("python_tree_sha256")
+    ):
+        fail()
+    arguments = [
+        str(mtree),
+        "-c",
+        "-n",
+        "-k",
+        "type,mode,nlink,sha256digest,link",
+        "-p",
+        str(artifact_root),
+    ]
+    environment = fixed_mtree_environment()
+    first_body = run_fixed(
+        arguments, cwd=system_temporary_root(), environment=environment
+    )
+    second_body = run_fixed(
+        arguments, cwd=system_temporary_root(), environment=environment
+    )
+    if (
+        first_body != second_body
+        or not first_body.startswith(b"\n/set type=file mode=0755 nlink=1\n")
+        or b"\x00" in first_body
+    ):
+        fail()
+    content = bootstrap_mtree_header(policy) + first_body
+    expected_digest = policy.get("python_mtree_sha256")
+    digest = hashlib.sha256(content).hexdigest()
+    if (
+        len(content) > MAX_BOOTSTRAP_MTREE_BYTES
+        or expected_digest != EXPECTED_BOOTSTRAP_MTREE_SHA256
+        or digest != expected_digest
+    ):
+        fail()
+    identity = write_bootstrap_tree_spec(
+        output, content, parent_identity=output_parent_identity
+    )
+    try:
+        if (
+            sha256_regular_file(output, maximum=MAX_BOOTSTRAP_MTREE_BYTES)
+            != digest
+            or run_fixed(
+                [str(mtree), "-f", str(output), "-p", str(artifact_root)],
+                cwd=system_temporary_root(),
+                environment=environment,
+            )
+            != b""
+            or sha256_regular_file(output, maximum=MAX_BOOTSTRAP_MTREE_BYTES)
+            != digest
+            or python_artifact_tree_sha256(artifact_root)
+            != policy.get("python_tree_sha256")
+            or sha256_regular_file(python, maximum=128 * 1024 * 1024)
+            != policy.get("python_executable_sha256")
+        ):
+            fail()
+    except BaseException:
+        remove_owned_bootstrap_tree_spec(
+            output,
+            identity=identity,
+            parent_identity=output_parent_identity,
+        )
+        raise
+    return digest
+
+
 def exact_python_version(python: Path, *, cwd: Path, environment: dict[str, str]) -> None:
     output = run_fixed(
         [str(python), "-I", "-S", "-B", "--version"],
@@ -3778,9 +4082,10 @@ def check_advisories(
 
 
 def parse_arguments(arguments: list[str]) -> dict[str, str]:
-    if len(arguments) != 12:
+    if len(arguments) != 14:
         fail()
     allowed = {
+        "--bootstrap-tree-spec-output",
         "--source-dir",
         "--python-bin",
         "--uv-bin",
@@ -3800,7 +4105,7 @@ def parse_arguments(arguments: list[str]) -> dict[str, str]:
     return parsed
 
 
-def run_gate(arguments: list[str]) -> tuple[str, str]:
+def run_gate(arguments: list[str]) -> tuple[str, str, str]:
     enforce_gate_runner_flags()
     parsed = parse_arguments(arguments)
     enforce_gate_runner(parsed["--python-bin"])
@@ -3812,6 +4117,11 @@ def run_gate(arguments: list[str]) -> tuple[str, str]:
     environment = safe_new_environment_path(
         parsed["--environment-dir"], source=source
     )
+    tree_spec, tree_spec_parent_identity = safe_new_bootstrap_tree_spec_path(
+        parsed["--bootstrap-tree-spec-output"]
+    )
+    if paths_overlap(tree_spec, environment):
+        fail()
     verify_input_boundaries(source, python, uv, cache, ca_file, environment)
     environment_identity: tuple[int, int] | None = None
     downloaded_wheels: dict[str, dict[str, object]] = {}
@@ -3863,8 +4173,17 @@ def run_gate(arguments: list[str]) -> tuple[str, str]:
             )
         ):
             fail()
+        bootstrap_python_mtree_digest = generate_bootstrap_tree_spec(
+            python,
+            tree_spec,
+            output_parent_identity=tree_spec_parent_identity,
+        )
         succeeded = True
-        return environment_digest, bootstrap_python_tree_digest
+        return (
+            environment_digest,
+            bootstrap_python_tree_digest,
+            bootstrap_python_mtree_digest,
+        )
     finally:
         for download in downloaded_wheels.values():
             if isinstance(download, dict) and "body" in download:
@@ -3880,9 +4199,11 @@ def run_gate(arguments: list[str]) -> tuple[str, str]:
 
 def main() -> int:
     try:
-        environment_digest, bootstrap_python_tree_digest = run_gate(
-            sys.argv[1:]
-        )
+        (
+            environment_digest,
+            bootstrap_python_tree_digest,
+            bootstrap_python_mtree_digest,
+        ) = run_gate(sys.argv[1:])
     except SystemExit as exc:
         return int(exc.code or 0)
     except Exception:
@@ -3890,7 +4211,8 @@ def main() -> int:
         return 1
     print(
         f"{SUCCESS_RECEIPT} dependency_environment_sha256={environment_digest} "
-        f"bootstrap_python_tree_sha256={bootstrap_python_tree_digest}"
+        f"bootstrap_python_tree_sha256={bootstrap_python_tree_digest} "
+        f"bootstrap_python_mtree_sha256={bootstrap_python_mtree_digest}"
     )
     return 0
 
diff --git a/scripts/provision-runtime-bootstrap b/scripts/provision-runtime-bootstrap
new file mode 100755
index 0000000..079d497
--- /dev/null
+++ b/scripts/provision-runtime-bootstrap
@@ -0,0 +1,239 @@
+#!/bin/sh -p
+PATH=/usr/bin:/bin
+export PATH
+set +x
+set -eu
+IFS=' '
+umask 077
+
+pinned_python_sha256=1bfa9a829d950ecd4870a3d7a6826eb57edb4aa93f69d07cd3bb21e9fcc6d439
+pinned_tree_sha256=28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110
+pinned_mtree_schema=travel-readiness-bootstrap-mtree-v1
+pinned_mtree_sha256=939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1
+
+fail() {
+  printf '%s\n' 'bootstrap_provision=failed' >&2
+  exit 1
+}
+
+if [ "$#" -ne 8 ] || [ "${1-}" != --source-prefix ] \
+  || [ "${3-}" != --source-tree-spec ] \
+  || [ "${5-}" != --destination-prefix ] \
+  || [ "${7-}" != --destination-tree-spec ]; then
+  fail
+fi
+source_prefix=$2
+source_tree_spec=$4
+destination_prefix=$6
+destination_tree_spec=$8
+unset BASH_ENV CDPATH ENV IFS
+unset DYLD_INSERT_LIBRARIES DYLD_LIBRARY_PATH LD_LIBRARY_PATH
+unset PYTHONHOME PYTHONPATH PYTHONSTARTUP
+IFS=' '
+
+if [ "$(/usr/bin/id -u 2>/dev/null)" != 0 ] \
+  || [ "$(/usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC /usr/bin/uname -s 2>/dev/null)" != Darwin ] \
+  || [ "$(/usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC /usr/bin/uname -m 2>/dev/null)" != arm64 ]; then
+  fail
+fi
+
+case "$source_prefix:$source_tree_spec:$destination_prefix:$destination_tree_spec" in
+  /*:/*:/*:/*) ;;
+  *) fail ;;
+esac
+for checked_path in "$source_prefix" "$source_tree_spec" \
+  "$destination_prefix" "$destination_tree_spec"; do
+  case "$checked_path" in
+    /|*//*|*/./*|*/../*|*/.|*/..|*/)
+      fail
+      ;;
+  esac
+done
+unset checked_path
+case "$destination_prefix/" in
+  "$source_prefix/"|"$source_prefix/"*) fail ;;
+esac
+case "$source_prefix/" in
+  "$destination_prefix/"*) fail ;;
+esac
+if [ "$source_tree_spec" = "$destination_tree_spec" ] \
+  || [ "$source_tree_spec" = "$destination_prefix" ] \
+  || [ "$destination_tree_spec" = "$source_prefix" ] \
+  || [ "$destination_tree_spec" = "$destination_prefix" ] \
+  || [ -e "$destination_prefix" ] || [ -L "$destination_prefix" ] \
+  || [ -e "$destination_tree_spec" ] || [ -L "$destination_tree_spec" ] \
+  || [ ! -d "$source_prefix" ] || [ -L "$source_prefix" ] \
+  || [ ! -f "$source_tree_spec" ] || [ -L "$source_tree_spec" ] \
+  || [ ! -x /usr/sbin/mtree ] || [ ! -x /usr/bin/find ] \
+  || [ ! -x /bin/cp ] || [ ! -x /bin/mv ] \
+  || [ ! -x /bin/mkdir ] || [ ! -x /bin/rmdir ] \
+  || [ ! -x /usr/sbin/chown ]; then
+  fail
+fi
+
+stat_style=
+if /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+  /usr/bin/stat -f '%u:%Lp:%l' -- / >/dev/null 2>&1; then
+  stat_style=bsd
+else
+  fail
+fi
+
+trusted_root_boundary() {
+  boundary=$1
+  while :; do
+    [ ! -L "$boundary" ] || return 1
+    status=$(
+      /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+        /usr/bin/stat -f '%u:%p' -- "$boundary" 2>/dev/null
+    ) || return 1
+    uid=${status%%:*}
+    mode=${status#*:}
+    [ "$uid" = 0 ] || return 1
+    case "$mode" in
+      ???) permissions=0$mode ;;
+      ????) permissions=$mode ;;
+      ?????|??????) permissions=${mode#${mode%????}} ;;
+      *) return 1 ;;
+    esac
+    other=${permissions#${permissions%?}}
+    without_other=${permissions%?}
+    group=${without_other#${without_other%?}}
+    special=0
+    [ "${#permissions}" -ne 4 ] || special=${permissions%???}
+    case "$group:$other" in
+      *[2367]:*|*:*[2367])
+        case "$special:$uid:$other" in
+          [1357]:0:[2367]) [ -d "$boundary" ] || return 1 ;;
+          *) return 1 ;;
+        esac
+        ;;
+    esac
+    [ "$boundary" != / ] || return 0
+    boundary=${boundary%/*}
+    [ -n "$boundary" ] || boundary=/
+  done
+}
+
+source_python=$source_prefix/bin/python3.14
+source_parent=${source_prefix%/*}
+destination_parent=${destination_prefix%/*}
+destination_spec_parent=${destination_tree_spec%/*}
+[ -n "$source_parent" ] || source_parent=/
+[ -n "$destination_parent" ] || destination_parent=/
+[ -n "$destination_spec_parent" ] || destination_spec_parent=/
+if ! trusted_root_boundary "$source_prefix" \
+  || ! trusted_root_boundary "$source_tree_spec" \
+  || ! trusted_root_boundary "$destination_parent" \
+  || ! trusted_root_boundary "$destination_spec_parent" \
+  || [ ! -f "$source_python" ] || [ ! -x "$source_python" ] \
+  || [ -L "$source_python" ]; then
+  fail
+fi
+source_spec_status=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/stat -f '%u:%Lp:%l' -- "$source_tree_spec" 2>/dev/null
+) || fail
+[ "$source_spec_status" = 0:444:1 ] || fail
+
+sha256_file() {
+  hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/shasum -a 256 -- "$1" 2>/dev/null
+  ) || return 1
+  hash=${hash_line%% *}
+  case "$hash" in
+    *[!0123456789abcdef]*|'') return 1 ;;
+  esac
+  [ "${#hash}" -eq 64 ] || return 1
+  printf '%s\n' "$hash"
+}
+
+expected_header="# $pinned_mtree_schema python_tree_sha256=$pinned_tree_sha256 python_executable_sha256=$pinned_python_sha256"
+observed_header=
+IFS= read -r observed_header <"$source_tree_spec" || fail
+[ "$observed_header" = "$expected_header" ] || fail
+[ "$(sha256_file "$source_tree_spec")" = "$pinned_mtree_sha256" ] || fail
+[ "$(sha256_file "$source_python")" = "$pinned_python_sha256" ] || fail
+source_violation=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/find "$source_prefix" \
+      \( ! -uid 0 -o -perm -020 -o -perm -002 \
+        -o ! \( -type d -o -type f -o -type l \) \) \
+      -print -quit 2>/dev/null
+) || fail
+[ -z "$source_violation" ] || fail
+source_mtree_output=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/sbin/mtree -f "$source_tree_spec" -p "$source_prefix" \
+      2>/dev/null
+) || fail
+[ -z "$source_mtree_output" ] || fail
+
+stage_prefix=$destination_prefix.provisioning.$$
+stage_spec_directory=$destination_tree_spec.provisioning.$$
+stage_spec=$stage_spec_directory/spec
+if [ -e "$stage_prefix" ] || [ -L "$stage_prefix" ] \
+  || [ -e "$stage_spec_directory" ] || [ -L "$stage_spec_directory" ]; then
+  fail
+fi
+/bin/mkdir -m 700 "$stage_prefix" >/dev/null 2>&1 || fail
+/bin/mkdir -m 700 "$stage_spec_directory" >/dev/null 2>&1 || fail
+/bin/cp -R -P -p "$source_prefix/." "$stage_prefix/" \
+  >/dev/null 2>&1 || fail
+/bin/cp -P -p "$source_tree_spec" "$stage_spec" \
+  >/dev/null 2>&1 || fail
+/usr/sbin/chown -R 0:0 "$stage_prefix" >/dev/null 2>&1 || fail
+/usr/sbin/chown 0:0 "$stage_spec" >/dev/null 2>&1 || fail
+/bin/chmod 0444 "$stage_spec" >/dev/null 2>&1 || fail
+stage_spec_status=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/stat -f '%u:%Lp:%l' -- "$stage_spec" 2>/dev/null
+) || fail
+[ "$stage_spec_status" = 0:444:1 ] || fail
+
+stage_violation=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/find "$stage_prefix" \
+      \( ! -uid 0 -o -perm -020 -o -perm -002 \
+        -o ! \( -type d -o -type f -o -type l \) \) \
+      -print -quit 2>/dev/null
+) || fail
+[ -z "$stage_violation" ] || fail
+[ "$(sha256_file "$stage_spec")" = "$pinned_mtree_sha256" ] || fail
+[ "$(sha256_file "$stage_prefix/bin/python3.14")" = "$pinned_python_sha256" ] || fail
+stage_mtree_output=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/sbin/mtree -f "$stage_spec" -p "$stage_prefix" \
+      2>/dev/null
+) || fail
+[ -z "$stage_mtree_output" ] || fail
+
+/bin/mv -n "$stage_spec" "$destination_tree_spec" >/dev/null 2>&1 || fail
+[ ! -e "$stage_spec" ] && [ ! -L "$stage_spec" ] || fail
+/bin/rmdir "$stage_spec_directory" >/dev/null 2>&1 || fail
+/bin/mv -n "$stage_prefix" "$destination_prefix" >/dev/null 2>&1 || fail
+[ ! -e "$stage_prefix" ] && [ ! -L "$stage_prefix" ] || fail
+destination_spec_status=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/stat -f '%u:%Lp:%l' -- "$destination_tree_spec" 2>/dev/null
+) || fail
+[ "$destination_spec_status" = 0:444:1 ] || fail
+[ "$(sha256_file "$destination_tree_spec")" = "$pinned_mtree_sha256" ] || fail
+[ "$(sha256_file "$destination_prefix/bin/python3.14")" = "$pinned_python_sha256" ] || fail
+destination_violation=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/find "$destination_prefix" \
+      \( ! -uid 0 -o -perm -020 -o -perm -002 \
+        -o ! \( -type d -o -type f -o -type l \) \) \
+      -print -quit 2>/dev/null
+) || fail
+[ -z "$destination_violation" ] || fail
+destination_mtree_output=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/sbin/mtree -f "$destination_tree_spec" \
+      -p "$destination_prefix" 2>/dev/null
+) || fail
+[ -z "$destination_mtree_output" ] || fail
+
+printf '%s\n' 'bootstrap_provision=ok'
diff --git a/scripts/run-production b/scripts/run-production
index 0ed9f4d..e32ea2c 100755
--- a/scripts/run-production
+++ b/scripts/run-production
@@ -1,15 +1,75 @@
-#!/bin/sh
+#!/bin/sh -p
+PATH=/usr/bin:/bin
+export PATH
+set +x
 set -eu
+IFS=' '
+umask 077
 
-if [ "${GUNICORN_CMD_ARGS+x}" = x ]; then
-  printf '%s\n' 'startup_error=GUNICORN_CMD_ARGS_FORBIDDEN' >&2
-  exit 64
-fi
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY
+pinned_bootstrap_python_sha256=1bfa9a829d950ecd4870a3d7a6826eb57edb4aa93f69d07cd3bb21e9fcc6d439
+pinned_bootstrap_tree_sha256=28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110
+pinned_bootstrap_mtree_schema=travel-readiness-bootstrap-mtree-v1
+pinned_bootstrap_mtree_sha256=939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1
+
+gunicorn_arguments_present=${GUNICORN_CMD_ARGS+x}
+bootstrap_python=${TRAVEL_READINESS_BOOTSTRAP_PYTHON-}
+bootstrap_python_sha256=${TRAVEL_READINESS_BOOTSTRAP_PYTHON_SHA256-}
+bootstrap_prefix=${TRAVEL_READINESS_BOOTSTRAP_PREFIX-}
+bootstrap_pristine_tree_sha256=${TRAVEL_READINESS_BOOTSTRAP_PRISTINE_TREE_SHA256-}
+bootstrap_tree_spec=${TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_FILE-}
+bootstrap_tree_spec_sha256=${TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_SHA256-}
+dependency_environment_sha256=${TRAVEL_READINESS_DEPENDENCY_ENVIRONMENT_SHA256-}
+expected_release_sha=${TRAVEL_READINESS_EXPECTED_RELEASE_SHA-}
+expected_manifest_sha256=${TRAVEL_READINESS_EXPECTED_MANIFEST_SHA256-}
+verifier_sha256=${TRAVEL_READINESS_VERIFIER_SHA256-}
+runtime_receipt=${TRAVEL_READINESS_RUNTIME_RECEIPT_FILE-}
+runtime_receipt_sha256=${TRAVEL_READINESS_RUNTIME_RECEIPT_SHA256-}
+trusted_owner_uid=${TRAVEL_READINESS_RELEASE_OWNER_UID-}
+tls_cert=${TRAVEL_READINESS_TLS_CERT_FILE-}
+tls_key=${TRAVEL_READINESS_TLS_KEY_FILE-}
+bind=${TRAVEL_READINESS_BIND-127.0.0.1:8000}
+secret_key=${TRAVEL_READINESS_SECRET_KEY-}
+database_password=${TRAVEL_READINESS_DB_PASSWORD-}
+allowed_hosts=${TRAVEL_READINESS_ALLOWED_HOSTS-}
+database_name=${TRAVEL_READINESS_DB_NAME-travel_readiness}
+database_user=${TRAVEL_READINESS_DB_USER-travel_readiness}
+database_host=${TRAVEL_READINESS_DB_HOST-127.0.0.1}
+database_port=${TRAVEL_READINESS_DB_PORT-5432}
+https_enabled=${TRAVEL_READINESS_HTTPS-1}
+debug_enabled=${TRAVEL_READINESS_DEBUG-0}
+
+unset BASH_ENV CDPATH ENV GUNICORN_CMD_ARGS IFS
+unset LD_LIBRARY_PATH DYLD_LIBRARY_PATH DYLD_INSERT_LIBRARIES
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
+unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
 unset TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD
 unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD
+unset TRAVEL_READINESS_BOOTSTRAP_PYTHON
+unset TRAVEL_READINESS_BOOTSTRAP_PYTHON_SHA256
+unset TRAVEL_READINESS_BOOTSTRAP_PREFIX
+unset TRAVEL_READINESS_BOOTSTRAP_PRISTINE_TREE_SHA256
+unset TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_FILE
+unset TRAVEL_READINESS_BOOTSTRAP_TREE_SPEC_SHA256
+unset TRAVEL_READINESS_EXPECTED_MANIFEST_SHA256
+unset TRAVEL_READINESS_EXPECTED_RELEASE_SHA
+unset TRAVEL_READINESS_VERIFIER_SHA256
+unset TRAVEL_READINESS_DEPENDENCY_ENVIRONMENT_SHA256
+unset TRAVEL_READINESS_RELEASE_OWNER_UID
+unset TRAVEL_READINESS_RELEASE_SHA
+unset TRAVEL_READINESS_RUNTIME_RECEIPT_FILE
+unset TRAVEL_READINESS_RUNTIME_RECEIPT_SHA256
+unset TRAVEL_READINESS_TLS_CERT_FILE TRAVEL_READINESS_TLS_KEY_FILE
+unset TRAVEL_READINESS_BIND TRAVEL_READINESS_SECRET_KEY
+unset TRAVEL_READINESS_DB_PASSWORD TRAVEL_READINESS_ALLOWED_HOSTS
+unset TRAVEL_READINESS_DB_NAME TRAVEL_READINESS_DB_USER
+unset TRAVEL_READINESS_DB_HOST TRAVEL_READINESS_DB_PORT
+unset TRAVEL_READINESS_HTTPS TRAVEL_READINESS_DEBUG TRAVEL_READINESS_BUILD
+IFS=' '
 
-bind=${TRAVEL_READINESS_BIND-127.0.0.1:8000}
+if [ "$gunicorn_arguments_present" = x ]; then
+  printf '%s\n' 'startup_error=GUNICORN_CMD_ARGS_FORBIDDEN' >&2
+  exit 64
+fi
 case "$bind" in
   127.0.0.1:*) bind_port=${bind#127.0.0.1:} ;;
   *)
@@ -17,115 +77,768 @@ case "$bind" in
     exit 64
     ;;
 esac
-case "$bind_port" in ''|*[!0-9]*)
-  printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
-  exit 64
-  ;;
+case "$bind_port" in
+  ''|*[!0-9]*)
+    printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
+    exit 64
+    ;;
 esac
 if [ "${#bind_port}" -gt 5 ] || [ "$bind_port" -lt 1 ] \
   || [ "$bind_port" -gt 65535 ]; then
   printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
   exit 64
 fi
+unset bind_port
+if [ -z "$tls_cert" ] || [ -z "$tls_key" ]; then
+  printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+  exit 64
+fi
 
-tls_cert=${TRAVEL_READINESS_TLS_CERT_FILE-}
-tls_key=${TRAVEL_READINESS_TLS_KEY_FILE-}
-case "$tls_cert:$tls_key" in
-  /*:/*) ;;
+case "$0" in
+  /*) ;;
   *)
-    printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+    printf '%s\n' 'startup_error=RELEASE_RUNTIME_INVALID' >&2
     exit 64
     ;;
 esac
-if [ ! -f "$tls_cert" ] || [ -L "$tls_cert" ] \
-  || [ ! -f "$tls_key" ] || [ -L "$tls_key" ]; then
-  printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+script_dir=${0%/*}
+project_dir=${script_dir%/*}
+verifier=$project_dir/scripts/verify-release-runtime
+
+if [ -z "$bootstrap_python" ] || [ -z "$bootstrap_python_sha256" ] \
+  || [ -z "$bootstrap_prefix" ] \
+  || [ -z "$bootstrap_pristine_tree_sha256" ] \
+  || [ -z "$bootstrap_tree_spec" ] \
+  || [ -z "$bootstrap_tree_spec_sha256" ] \
+  || [ -z "$dependency_environment_sha256" ] \
+  || [ -z "$expected_release_sha" ] || [ -z "$expected_manifest_sha256" ] \
+  || [ -z "$verifier_sha256" ] \
+  || [ -z "$runtime_receipt" ] || [ -z "$runtime_receipt_sha256" ] \
+  || [ -z "$trusted_owner_uid" ] || [ -z "$secret_key" ] \
+  || [ -z "$database_password" ] || [ -z "$allowed_hosts" ]; then
+  printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_MISSING' >&2
+  exit 64
+fi
+case "$bootstrap_python:$bootstrap_prefix:$bootstrap_tree_spec:$runtime_receipt:$tls_cert:$tls_key" in
+  /*:/*:/*:/*:/*:/*) ;;
+  *)
+    printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_INVALID' >&2
+    exit 64
+    ;;
+esac
+case "$bootstrap_python_sha256:$bootstrap_pristine_tree_sha256:$bootstrap_tree_spec_sha256:$dependency_environment_sha256:$expected_manifest_sha256:$runtime_receipt_sha256:$verifier_sha256" in
+  *[!0123456789abcdef:]*|*::*|:*)
+    printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_INVALID' >&2
+    exit 64
+    ;;
+esac
+if [ "${#bootstrap_python_sha256}" -ne 64 ] \
+  || [ "${#bootstrap_pristine_tree_sha256}" -ne 64 ] \
+  || [ "${#bootstrap_tree_spec_sha256}" -ne 64 ] \
+  || [ "${#dependency_environment_sha256}" -ne 64 ] \
+  || [ "${#expected_manifest_sha256}" -ne 64 ] \
+  || [ "${#runtime_receipt_sha256}" -ne 64 ]; then
+  printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_INVALID' >&2
+  exit 64
+fi
+if [ "${#verifier_sha256}" -ne 64 ]; then
+  printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_INVALID' >&2
+  exit 64
+fi
+if [ "$bootstrap_python_sha256" != "$pinned_bootstrap_python_sha256" ] \
+  || [ "$bootstrap_pristine_tree_sha256" != "$pinned_bootstrap_tree_sha256" ] \
+  || [ "$bootstrap_tree_spec_sha256" != "$pinned_bootstrap_mtree_sha256" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+case "$trusted_owner_uid" in
+  ''|*[!0-9]*)
+    printf '%s\n' 'startup_error=STARTUP_CONFIGURATION_INVALID' >&2
+    exit 64
+    ;;
+esac
+if [ ! -f "$bootstrap_python" ] || [ ! -x "$bootstrap_python" ] \
+  || [ -L "$bootstrap_python" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+if [ "$bootstrap_python" != "$bootstrap_prefix/bin/python3.14" ] \
+  || [ ! -d "$bootstrap_prefix" ] || [ -L "$bootstrap_prefix" ] \
+  || [ ! -f "$bootstrap_tree_spec" ] || [ -L "$bootstrap_tree_spec" ] \
+  || [ ! -x /usr/bin/find ] || [ ! -x /usr/sbin/mtree ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
   exit 64
 fi
 
-script_dir=$(CDPATH='' cd -- "$(dirname -- "$0")" && pwd)
-project_dir=$(dirname -- "$script_dir")
-python_bin="$project_dir/.venv/bin/python"
+stat_style=
+if /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+  /usr/bin/stat -f '%u:%p' -- / >/dev/null 2>&1; then
+  stat_style=bsd
+elif /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+  /usr/bin/stat -c '%u:%a' -- / >/dev/null 2>&1; then
+  stat_style=gnu
+else
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
 
-if [ ! -x "$python_bin" ]; then
-  printf '%s\n' 'startup_error=FROZEN_ENV_MISSING' >&2
+trusted_path_boundary() {
+  checked_boundary=$1
+  allow_trusted_boundary_owner=${2-1}
+  while :; do
+    if [ -L "$checked_boundary" ]; then
+      return 1
+    fi
+    if [ "$stat_style" = bsd ]; then
+      boundary_status=$(
+        /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+          /usr/bin/stat -f '%u:%p' -- "$checked_boundary" 2>/dev/null
+      ) || return 1
+    else
+      boundary_status=$(
+        /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+          /usr/bin/stat -c '%u:%a' -- "$checked_boundary" 2>/dev/null
+      ) || return 1
+    fi
+    boundary_uid=${boundary_status%%:*}
+    boundary_mode=${boundary_status#*:}
+    case "$boundary_uid" in ''|*[!0123456789]*) return 1 ;; esac
+    case "$boundary_mode" in ''|*[!01234567]*) return 1 ;; esac
+    if [ "$boundary_uid" != 0 ] \
+      && { [ "$allow_trusted_boundary_owner" != 1 ] \
+        || [ "$boundary_uid" != "$trusted_owner_uid" ]; }; then
+      return 1
+    fi
+    case "$boundary_mode" in
+      ???) boundary_permissions=0$boundary_mode ;;
+      ????) boundary_permissions=$boundary_mode ;;
+      ?????|??????)
+        boundary_permissions=${boundary_mode#${boundary_mode%????}}
+        ;;
+      *) return 1 ;;
+    esac
+    boundary_other=${boundary_permissions#${boundary_permissions%?}}
+    boundary_without_other=${boundary_permissions%?}
+    boundary_group=${boundary_without_other#${boundary_without_other%?}}
+    boundary_special=0
+    if [ "${#boundary_permissions}" -eq 4 ]; then
+      boundary_special=${boundary_permissions%???}
+    fi
+    boundary_writable=0
+    case "$boundary_group:$boundary_other" in
+      *[2367]:*|*:*[2367]) boundary_writable=1 ;;
+    esac
+    if [ "$boundary_writable" -eq 1 ]; then
+      boundary_sticky=0
+      case "$boundary_special" in 1|3|5|7) boundary_sticky=1 ;; esac
+      if [ ! -d "$checked_boundary" ] || [ "$boundary_uid" != 0 ] \
+        || [ "$boundary_sticky" -ne 1 ] \
+        || { [ "$boundary_other" != 2 ] && [ "$boundary_other" != 3 ] \
+          && [ "$boundary_other" != 6 ] && [ "$boundary_other" != 7 ]; }; then
+        return 1
+      fi
+    fi
+    if [ "$checked_boundary" = / ]; then
+      return 0
+    fi
+    checked_boundary=${checked_boundary%/*}
+    if [ -z "$checked_boundary" ]; then
+      checked_boundary=/
+    fi
+  done
+}
+if ! trusted_path_boundary "$bootstrap_python" 0 \
+  || ! trusted_path_boundary "$bootstrap_prefix" 0 \
+  || ! trusted_path_boundary "$bootstrap_tree_spec" 0; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+if [ "$stat_style" = bsd ]; then
+  bootstrap_spec_status=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/stat -f '%u:%Lp:%l' -- "$bootstrap_tree_spec" 2>/dev/null
+  ) || {
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  }
+else
+  bootstrap_spec_status=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/stat -c '%u:%a:%h' -- "$bootstrap_tree_spec" 2>/dev/null
+  ) || {
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  }
+fi
+if [ "$bootstrap_spec_status" != 0:444:1 ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
   exit 64
 fi
-if ! "$python_bin" -I -c 'import sys; raise SystemExit(0 if sys.version_info[:3] == (3, 14, 7) else 1)' \
-  >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=PYTHON_VERSION_MISMATCH' >&2
+unset bootstrap_spec_status
+if [ ! -f "$verifier" ] || [ ! -x "$verifier" ] || [ -L "$verifier" ] \
+  || ! trusted_path_boundary "$verifier"; then
+  printf '%s\n' 'startup_error=RELEASE_RUNTIME_INVALID' >&2
   exit 64
 fi
-if ! "$python_bin" -I -c '
-import os
-import stat
-import sys
-cert = os.lstat(sys.argv[1])
-key = os.lstat(sys.argv[2])
-valid = (
-    stat.S_ISREG(cert.st_mode)
-    and stat.S_ISREG(key.st_mode)
-    and cert.st_uid == os.geteuid()
-    and key.st_uid == os.geteuid()
-    and key.st_mode & 0o077 == 0
-)
-raise SystemExit(0 if valid else 1)
-' "$tls_cert" "$tls_key" >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+unset allow_trusted_boundary_owner
+unset boundary_group boundary_mode boundary_other boundary_permissions
+unset boundary_special boundary_status boundary_uid boundary_without_other
+unset boundary_writable boundary_sticky checked_boundary stat_style
+
+bootstrap_hash_line=
+if [ -x /usr/bin/shasum ]; then
+  if ! bootstrap_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/shasum -a 256 -- "$bootstrap_python" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+elif [ -x /usr/bin/sha256sum ]; then
+  if ! bootstrap_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/sha256sum -- "$bootstrap_python" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+else
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+bootstrap_hash=${bootstrap_hash_line%% *}
+unset bootstrap_hash_line
+if [ "$bootstrap_hash" != "$bootstrap_python_sha256" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
   exit 64
 fi
-if ! "$python_bin" -I -c '
-import ssl
+unset bootstrap_hash
+
+tree_spec_hash_line=
+if [ -x /usr/bin/shasum ]; then
+  if ! tree_spec_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/shasum -a 256 -- "$bootstrap_tree_spec" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+elif [ -x /usr/bin/sha256sum ]; then
+  if ! tree_spec_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/sha256sum -- "$bootstrap_tree_spec" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+else
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+tree_spec_hash=${tree_spec_hash_line%% *}
+unset tree_spec_hash_line
+if [ "$tree_spec_hash" != "$bootstrap_tree_spec_sha256" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+unset tree_spec_hash
+tree_spec_header=
+if ! IFS= read -r tree_spec_header <"$bootstrap_tree_spec"; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+expected_tree_spec_header="# $pinned_bootstrap_mtree_schema python_tree_sha256=$pinned_bootstrap_tree_sha256 python_executable_sha256=$pinned_bootstrap_python_sha256"
+if [ "$tree_spec_header" != "$expected_tree_spec_header" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+unset expected_tree_spec_header tree_spec_header
+
+bootstrap_tree_violation=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/find "$bootstrap_prefix" \
+      \( ! -uid 0 -o -perm -020 -o -perm -002 \
+        -o ! \( -type d -o -type f -o -type l \) \) \
+      -print -quit 2>/dev/null
+) || {
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+}
+if [ -n "$bootstrap_tree_violation" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+unset bootstrap_tree_violation
+bootstrap_mtree_output=
+if ! bootstrap_mtree_output=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/sbin/mtree -f "$bootstrap_tree_spec" -p "$bootstrap_prefix" \
+      2>/dev/null
+); then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+if [ -n "$bootstrap_mtree_output" ]; then
+  printf '%s\n' 'startup_error=BOOTSTRAP_RUNTIME_INVALID' >&2
+  exit 64
+fi
+unset bootstrap_mtree_output
+
+verifier_hash_line=
+if [ -x /usr/bin/shasum ]; then
+  if ! verifier_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/shasum -a 256 -- "$verifier" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=RELEASE_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+else
+  if ! verifier_hash_line=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /usr/bin/sha256sum -- "$verifier" 2>/dev/null
+  ); then
+    printf '%s\n' 'startup_error=RELEASE_RUNTIME_INVALID' >&2
+    exit 64
+  fi
+fi
+verifier_hash=${verifier_hash_line%% *}
+unset verifier_hash_line
+if [ "$verifier_hash" != "$verifier_sha256" ]; then
+  printf '%s\n' 'startup_error=RELEASE_RUNTIME_INVALID' >&2
+  exit 64
+fi
+unset verifier_hash
+
+runtime_values() {
+  printf '%s\0' "$secret_key"
+  printf '%s\0' "$database_password"
+  printf '%s\0' "$allowed_hosts"
+  printf '%s\0' "$database_name"
+  printf '%s\0' "$database_user"
+  printf '%s\0' "$database_host"
+  printf '%s\0' "$database_port"
+  printf '%s\0' "$https_enabled"
+  printf '%s\0' "$debug_enabled"
+}
+
+runtime_pipe_dir=
+runtime_fifo=
+runtime_ack_fifo=
+runtime_pid=
+runtime_identity=
+runtime_group_identity=
+runtime_group_ready=0
+runtime_identity_python=$bootstrap_python
+runtime_owner_uid=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/id -u 2>/dev/null
+) || {
+  printf '%s\n' 'startup_error=RUNTIME_PIPE_INVALID' >&2
+  exit 64
+}
+case "$runtime_owner_uid" in
+  ''|*[!0123456789]*)
+    printf '%s\n' 'startup_error=RUNTIME_PIPE_INVALID' >&2
+    exit 64
+    ;;
+esac
+cleanup_runtime_pipe() {
+  if [ -n "$runtime_fifo" ]; then
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/rm -f -- "$runtime_fifo" >/dev/null 2>&1 || :
+  fi
+  if [ -n "$runtime_ack_fifo" ]; then
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/rm -f -- "$runtime_ack_fifo" >/dev/null 2>&1 || :
+  fi
+  if [ -n "$runtime_pipe_dir" ]; then
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/rmdir -- "$runtime_pipe_dir" >/dev/null 2>&1 || :
+  fi
+  runtime_fifo=
+  runtime_ack_fifo=
+  runtime_pipe_dir=
+}
+close_runtime_descriptors() {
+  exec 5>&- 6<&- 8<&- 9>&-
+}
+runtime_process_snapshot() {
+  snapshot_pid=$1
+  first_process_status=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/ps -o pid= -o ppid= -o pgid= -o uid= -o lstart= \
+        -p "$snapshot_pid" 2>/dev/null \
+    | /usr/bin/awk -v expected_pid="$snapshot_pid" '
+        NF == 9 && $1 == expected_pid {
+          count += 1
+          print $1, $2, $3, $4, $5, $6, $7, $8, $9
+          next
+        }
+        NF != 0 { invalid = 1 }
+        END { if (count != 1 || invalid) exit 2 }
+      '
+  ) || return 1
+  process_session=$(runtime_kernel_session_id "$snapshot_pid") || return 1
+  second_process_status=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/ps -o pid= -o ppid= -o pgid= -o uid= -o lstart= \
+        -p "$snapshot_pid" 2>/dev/null \
+    | /usr/bin/awk -v expected_pid="$snapshot_pid" '
+        NF == 9 && $1 == expected_pid {
+          count += 1
+          print $1, $2, $3, $4, $5, $6, $7, $8, $9
+          next
+        }
+        NF != 0 { invalid = 1 }
+        END { if (count != 1 || invalid) exit 2 }
+      '
+  ) || return 1
+  [ "$first_process_status" = "$second_process_status" ] || return 1
+  set -- $first_process_status
+  [ "$#" -eq 9 ] || return 1
+  printf '%s %s %s %s %s %s %s %s %s %s\n' \
+    "$1" "$2" "$3" "$process_session" "$4" \
+    "$5" "$6" "$7" "$8" "$9"
+  unset first_process_status process_session second_process_status
+}
+runtime_kernel_session_id() {
+  kernel_pid=$1
+  kernel_session=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      PYTHONDONTWRITEBYTECODE=1 \
+      "$runtime_identity_python" -I -S -B -c '
+import os
 import sys
-context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
-context.minimum_version = ssl.TLSVersion.TLSv1_2
-context.load_cert_chain(sys.argv[1], sys.argv[2])
-' "$tls_cert" "$tls_key" >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=TLS_MATERIAL_INVALID' >&2
-  exit 64
-fi
-if ! "$python_bin" -I -c '
-from importlib.metadata import version
-expected = {
-    "Django": "5.2.17",
-    "gunicorn": "26.2.0",
-    "psycopg": "3.3.4",
-    "psycopg-binary": "3.3.4",
-}
-if any(version(name) != wanted for name, wanted in expected.items()):
+try:
+    pid = int(sys.argv[1])
+    session = os.getsid(pid)
+except (OSError, TypeError, ValueError):
     raise SystemExit(1)
-import django, gunicorn, psycopg  # noqa: F401
-' >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=DEPENDENCY_VERSION_MISMATCH' >&2
+if pid <= 0 or session <= 0:
+    raise SystemExit(1)
+sys.stdout.write(f"{session}\n")
+' "$kernel_pid" 2>/dev/null
+  ) || return 1
+  case "$kernel_session" in
+    ''|*[!0123456789]*) return 1 ;;
+  esac
+  printf '%s\n' "$kernel_session"
+  unset kernel_pid kernel_session
+}
+capture_runtime_identity() {
+  identity_phase=$1
+  [ -n "$runtime_pid" ] || return 1
+  captured_identity=$(runtime_process_snapshot "$runtime_pid") || return 1
+  set -- $captured_identity
+  if [ "$#" -ne 10 ] || [ "$1" != "$runtime_pid" ] \
+    || [ "$2" != "$$" ] || [ "$5" != "$runtime_owner_uid" ]; then
+    return 1
+  fi
+  if [ "$identity_phase" = group ] \
+    && { [ "$3" != "$runtime_pid" ] || [ "$4" != "$runtime_pid" ]; }; then
+    return 1
+  fi
+  runtime_identity=$captured_identity
+  unset captured_identity identity_phase
+}
+runtime_identity_is_current() {
+  [ -n "$runtime_pid" ] && [ -n "$runtime_identity" ] || return 1
+  current_identity=$(runtime_process_snapshot "$runtime_pid") || return 1
+  [ "$current_identity" = "$runtime_identity" ] || return 1
+  unset current_identity
+}
+runtime_group_live_inventory() {
+  [ -n "$runtime_pid" ] || return 1
+  raw_group_inventory=$(
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/ps -ax -o pid= -o pgid= -o uid= -o state= \
+        -o lstart= 2>/dev/null \
+    | /usr/bin/awk -v expected_group="$runtime_pid" \
+        -v expected_uid="$runtime_owner_uid" '
+        $2 == expected_group {
+          if (NF != 9 || $3 != expected_uid) {
+            invalid = 1
+            next
+          }
+          if ($4 !~ /^Z/) {
+            print $1, $2, $3, $5, $6, $7, $8, $9
+          }
+        }
+        END { if (invalid) exit 2 }
+      '
+  ) || return 1
+  remaining_group_inventory=$raw_group_inventory
+  checked_group_inventory=
+  while [ -n "$remaining_group_inventory" ]; do
+    case "$remaining_group_inventory" in
+      *'
+'*)
+        raw_group_member=${remaining_group_inventory%%'
+'*}
+        remaining_group_inventory=${remaining_group_inventory#*'
+'}
+        ;;
+      *)
+        raw_group_member=$remaining_group_inventory
+        remaining_group_inventory=
+        ;;
+    esac
+    set -- $raw_group_member
+    [ "$#" -eq 8 ] || return 1
+    member_session=$(runtime_kernel_session_id "$1") || continue
+    [ "$member_session" = "$runtime_pid" ] || return 1
+    checked_group_member="$1 $2 $member_session $3 $4 $5 $6 $7 $8"
+    if [ -n "$checked_group_inventory" ]; then
+      checked_group_inventory="$checked_group_inventory
+$checked_group_member"
+    else
+      checked_group_inventory=$checked_group_member
+    fi
+  done
+  [ -z "$checked_group_inventory" ] \
+    || printf '%s\n' "$checked_group_inventory"
+  unset checked_group_inventory checked_group_member member_session
+  unset raw_group_inventory raw_group_member remaining_group_inventory
+}
+runtime_leader_group_identity() {
+  set -- $runtime_identity
+  [ "$#" -eq 10 ] || return 1
+  printf '%s %s %s %s %s %s %s %s %s\n' \
+    "$1" "$3" "$4" "$5" "$6" "$7" "$8" "$9" "${10}"
+}
+runtime_group_inventory_is_subset() {
+  candidate_inventory=$1
+  remaining_inventory=$candidate_inventory
+  while [ -n "$remaining_inventory" ]; do
+    case "$remaining_inventory" in
+      *'
+'*)
+        runtime_member=${remaining_inventory%%'
+'*}
+        remaining_inventory=${remaining_inventory#*'
+'}
+        ;;
+      *)
+        runtime_member=$remaining_inventory
+        remaining_inventory=
+        ;;
+    esac
+    case "
+$runtime_group_identity
+" in
+      *"
+$runtime_member
+"*) ;;
+      *) return 1 ;;
+    esac
+  done
+  unset candidate_inventory remaining_inventory runtime_member
+}
+signal_runtime_tree() {
+  runtime_signal=$1
+  if [ "$runtime_group_ready" -eq 1 ]; then
+    signal_inventory=$(runtime_group_live_inventory) || return 1
+    [ -n "$signal_inventory" ] || return 1
+    runtime_group_inventory_is_subset "$signal_inventory" || return 1
+    kill -"$runtime_signal" "-$runtime_pid" 2>/dev/null || return 1
+    unset signal_inventory
+  else
+    runtime_identity_is_current || return 1
+    kill -"$runtime_signal" "$runtime_pid" 2>/dev/null || return 1
+  fi
+  unset runtime_signal
+}
+terminate_runtime() {
+  [ -n "$runtime_pid" ] && [ -n "$runtime_identity" ] || return 0
+  termination_safe=1
+  termination_live=
+  if [ "$runtime_group_ready" -eq 1 ]; then
+    runtime_identity_is_current || termination_safe=0
+    if [ "$termination_safe" -eq 1 ]; then
+      runtime_group_identity=$(runtime_group_live_inventory) \
+        || termination_safe=0
+    fi
+    if [ "$termination_safe" -eq 1 ]; then
+      leader_group_identity=$(runtime_leader_group_identity) \
+        || termination_safe=0
+    fi
+    if [ "$termination_safe" -eq 1 ]; then
+      case "
+$runtime_group_identity
+" in
+        *"
+$leader_group_identity
+"*) ;;
+        *) termination_safe=0 ;;
+      esac
+    fi
+  fi
+  if [ "$termination_safe" -eq 1 ]; then
+    signal_runtime_tree TERM || termination_safe=0
+  fi
+  termination_attempt=0
+  while [ "$termination_safe" -eq 1 ] \
+    && [ "$termination_attempt" -lt 50 ]; do
+    if [ "$runtime_group_ready" -eq 1 ]; then
+      termination_live=$(runtime_group_live_inventory) \
+        || termination_safe=0
+      if [ "$termination_safe" -eq 1 ] \
+        && [ -n "$termination_live" ]; then
+        runtime_group_inventory_is_subset "$termination_live" \
+          || termination_safe=0
+      fi
+    elif runtime_identity_is_current; then
+      termination_live=$runtime_identity
+    else
+      termination_live=
+    fi
+    [ "$termination_safe" -eq 1 ] || break
+    [ -n "$termination_live" ] || break
+    /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+      /bin/sleep 0.1 >/dev/null 2>&1 || termination_safe=0
+    termination_attempt=$((termination_attempt + 1))
+  done
+  if [ "$termination_safe" -eq 1 ] && [ -n "$termination_live" ]; then
+    signal_runtime_tree KILL || termination_safe=0
+  fi
+  wait "$runtime_pid" 2>/dev/null || :
+  runtime_pid=
+  runtime_identity=
+  runtime_group_identity=
+  runtime_group_ready=0
+  unset leader_group_identity termination_attempt termination_live
+  unset termination_safe
+}
+abort_runtime_pipe() {
+  trap '' HUP INT TERM PIPE
+  close_runtime_descriptors
+  terminate_runtime TERM
+  cleanup_runtime_pipe
+  printf '%s\n' 'startup_error=RUNTIME_PIPE_INVALID' >&2
   exit 64
+}
+forward_hup() {
+  trap '' HUP INT TERM PIPE
+  close_runtime_descriptors
+  terminate_runtime TERM
+  cleanup_runtime_pipe
+  exit 129
+}
+forward_interrupt() {
+  trap '' HUP INT TERM PIPE
+  close_runtime_descriptors
+  terminate_runtime TERM
+  cleanup_runtime_pipe
+  exit 130
+}
+forward_term() {
+  trap '' HUP INT TERM PIPE
+  close_runtime_descriptors
+  terminate_runtime TERM
+  cleanup_runtime_pipe
+  exit 143
+}
+trap cleanup_runtime_pipe EXIT
+trap forward_hup HUP
+trap forward_interrupt INT
+trap forward_term TERM
+trap abort_runtime_pipe PIPE
+
+runtime_pipe_dir=$(
+  /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+    /usr/bin/mktemp -d /tmp/travel-readiness-runtime.XXXXXXXX 2>/dev/null
+) || {
+  abort_runtime_pipe
+}
+case "$runtime_pipe_dir" in
+  /tmp/travel-readiness-runtime.????????) ;;
+  *)
+    abort_runtime_pipe
+    ;;
+esac
+runtime_fifo=$runtime_pipe_dir/values
+runtime_ack_fifo=$runtime_pipe_dir/ready
+if ! /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+  /usr/bin/mkfifo -m 600 -- "$runtime_fifo" "$runtime_ack_fifo" \
+    >/dev/null 2>&1; then
+  abort_runtime_pipe
 fi
+if ! { exec 4<>"$runtime_fifo"; } 2>/dev/null \
+  || ! { exec 5>"$runtime_fifo"; } 2>/dev/null \
+  || ! { exec 6<"$runtime_fifo"; } 2>/dev/null \
+  || ! { exec 7<>"$runtime_ack_fifo"; } 2>/dev/null \
+  || ! { exec 8<"$runtime_ack_fifo"; } 2>/dev/null \
+  || ! { exec 9>"$runtime_ack_fifo"; } 2>/dev/null; then
+  abort_runtime_pipe
+fi
+exec 4>&-
+exec 7>&-
 
-cd "$project_dir"
-DJANGO_SETTINGS_MODULE=travel_readiness.settings
-export DJANGO_SETTINGS_MODULE
-if ! "$python_bin" -I -c '
-import sys
-sys.path.insert(0, sys.argv[1])
-import django
-django.setup()
-from django.core.management import call_command
-call_command("check", deploy=True, fail_level="WARNING", verbosity=0)
-' "$project_dir" >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=DEPLOYMENT_CHECK_FAILED' >&2
-  exit 78
-fi
-if ! "$python_bin" -I -c '
-import sys
-sys.path.insert(0, sys.argv[1])
-import django
-django.setup()
-from django.core.management import call_command
-call_command("migrate", check=True, interactive=False, verbosity=0)
-' "$project_dir" >/dev/null 2>&1; then
-  printf '%s\n' 'startup_error=MIGRATION_CHECK_FAILED' >&2
-  exit 78
-fi
-exec "$python_bin" -I -m gunicorn \
-  --chdir "$project_dir" \
-  --config "$project_dir/runtime/gunicorn.conf.py" \
-  travel_readiness.wsgi:application
+/usr/bin/env -i \
+  PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC PYTHONDONTWRITEBYTECODE=1 \
+  "$bootstrap_python" -I -S -B "$verifier" launch \
+  --bootstrap-prefix "$bootstrap_prefix" \
+  --bootstrap-pristine-tree-sha256 "$bootstrap_pristine_tree_sha256" \
+  --bootstrap-python "$bootstrap_python" \
+  --bootstrap-python-sha256 "$bootstrap_python_sha256" \
+  --bootstrap-tree-spec "$bootstrap_tree_spec" \
+  --bootstrap-tree-spec-sha256 "$bootstrap_tree_spec_sha256" \
+  --expected-dependency-environment-sha256 "$dependency_environment_sha256" \
+  --expected-manifest-sha256 "$expected_manifest_sha256" \
+  --expected-release-sha "$expected_release_sha" \
+  --expected-verifier-sha256 "$verifier_sha256" \
+  --expected-runtime-receipt-sha256 "$runtime_receipt_sha256" \
+  --runtime-receipt "$runtime_receipt" \
+  --trusted-owner-uid "$trusted_owner_uid" \
+  --tls-cert "$tls_cert" \
+  --tls-key "$tls_key" \
+  --bind "$bind" <&6 4>&9 5>&- 6<&- 8<&- 9>&- 3>&2 2>/dev/null &
+runtime_pid=$!
+exec 6<&- 9>&-
+if ! capture_runtime_identity direct; then
+  exec 5>&- 8<&-
+  abort_runtime_pipe
+fi
+
+runtime_ready_pid=
+if ! IFS= read -r runtime_ready_pid <&8 \
+  || [ "$runtime_ready_pid" != "$runtime_pid" ]; then
+  exec 5>&- 8<&-
+  abort_runtime_pipe
+fi
+exec 8<&-
+unset runtime_ready_pid
+if ! capture_runtime_identity group; then
+  exec 5>&-
+  abort_runtime_pipe
+fi
+runtime_group_ready=1
+
+if ! runtime_values >&5; then
+  exec 5>&-
+  abort_runtime_pipe
+fi
+exec 5>&-
+cleanup_runtime_pipe
+
+unset secret_key database_password allowed_hosts
+unset database_name database_user database_host database_port
+unset https_enabled debug_enabled
+unset bootstrap_python bootstrap_python_sha256 bootstrap_prefix
+unset bootstrap_pristine_tree_sha256 bootstrap_tree_spec
+unset bootstrap_tree_spec_sha256 dependency_environment_sha256
+unset expected_release_sha
+unset expected_manifest_sha256 runtime_receipt runtime_receipt_sha256
+unset verifier_sha256
+unset trusted_owner_uid tls_cert tls_key bind verifier project_dir script_dir
+
+set +e
+wait "$runtime_pid"
+runtime_status=$?
+set -e
+runtime_pid=
+runtime_group_ready=0
+trap - EXIT HUP INT TERM PIPE
+cleanup_runtime_pipe
+exit "$runtime_status"
diff --git a/scripts/verify-release-runtime b/scripts/verify-release-runtime
new file mode 100755
index 0000000..b74259f
--- /dev/null
+++ b/scripts/verify-release-runtime
@@ -0,0 +1,2628 @@
+#!/usr/bin/env python3
+"""Bind a runtime process to one exact extracted release source tree."""
+
+from __future__ import annotations
+
+import sys
+
+if not (
+    sys.flags.isolated
+    and sys.flags.no_site
+    and sys.flags.dont_write_bytecode
+):
+    raise SystemExit(1)
+
+import base64
+import csv
+from email.parser import BytesParser
+import hashlib
+import io
+import json
+import os
+from pathlib import Path, PurePosixPath
+import platform
+import re
+import stat
+import subprocess
+import tomllib
+from typing import NoReturn
+
+
+MANIFEST_LIMIT = 8 * 1024 * 1024
+TRACKED_FILE_LIMIT = 64 * 1024 * 1024
+TRACKED_TOTAL_LIMIT = 512 * 1024 * 1024
+VENV_FILE_LIMIT = 64 * 1024 * 1024
+VENV_TOTAL_LIMIT = 256 * 1024 * 1024
+VENV_ENTRY_LIMIT = 20_000
+RECEIPT_LIMIT = 64 * 1024 * 1024
+PINNED_BOOTSTRAP_PYTHON_SHA256 = (
+    "1bfa9a829d950ecd4870a3d7a6826eb57edb4aa93f69d07cd3bb21e9fcc6d439"
+)
+PINNED_BOOTSTRAP_TREE_SHA256 = (
+    "28796411ad33b7aa638710849ef1ec150a92e9ee51c581ed3c5a7d56ce613110"
+)
+PINNED_BOOTSTRAP_MTREE_SCHEMA = "travel-readiness-bootstrap-mtree-v1"
+PINNED_BOOTSTRAP_MTREE_SHA256 = (
+    "939d99ae9466194c03136cfb9128931da7f01ec94786a67528fa416c9d5935d1"
+)
+EXPECTED_GIT_MODES = {"100644": 0o644, "100755": 0o755}
+TOP_LEVEL_KEYS = {
+    "archive",
+    "dependencies",
+    "git_object_format",
+    "migrations",
+    "release_sha",
+    "runtime",
+    "schema_version",
+    "source",
+    "static",
+}
+RUNTIME_VERSION_KEYS = {
+    "django",
+    "gunicorn",
+    "postgresql",
+    "psycopg",
+    "psycopg_distribution",
+    "python",
+    "uv",
+}
+OPEN_FLAGS = os.O_RDONLY | os.O_CLOEXEC | os.O_NOFOLLOW | os.O_NONBLOCK
+DIRECTORY_FLAGS = OPEN_FLAGS | os.O_DIRECTORY
+RECEIPT_KEYS = {
+    "bootstrap",
+    "canonical_inventory",
+    "canonical_inventory_sha256",
+    "dependency_environment_sha256",
+    "distributions",
+    "lock_selection",
+    "lock_sha256",
+    "manifest_sha256",
+    "owner_uid",
+    "release_sha",
+    "schema_version",
+    "verifier_sha256",
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
+BOOTSTRAP_KEYS = {
+    "implementation",
+    "machine",
+    "python_sha256",
+    "python_version",
+    "pristine_tree_sha256",
+    "system",
+    "tree_spec_format",
+    "tree_spec_sha256",
+}
+LOCK_SELECTION_KEYS = {"environment", "package_set_sha256", "packages"}
+LOCK_ENVIRONMENT_KEYS = {
+    "implementation_name",
+    "platform_machine",
+    "python_full_version",
+    "python_version",
+    "sys_platform",
+}
+RUNTIME_ENV_NAMES = (
+    "TRAVEL_READINESS_SECRET_KEY",
+    "TRAVEL_READINESS_DB_PASSWORD",
+    "TRAVEL_READINESS_ALLOWED_HOSTS",
+    "TRAVEL_READINESS_DB_NAME",
+    "TRAVEL_READINESS_DB_USER",
+    "TRAVEL_READINESS_DB_HOST",
+    "TRAVEL_READINESS_DB_PORT",
+    "TRAVEL_READINESS_HTTPS",
+    "TRAVEL_READINESS_DEBUG",
+)
+
+
+class VerificationError(RuntimeError):
+    pass
+
+
+def fail() -> NoReturn:
+    raise VerificationError
+
+
+def sha256(data: bytes) -> str:
+    return hashlib.sha256(data).hexdigest()
+
+
+def canonical_json(value: object) -> bytes:
+    try:
+        encoded = json.dumps(
+            value,
+            ensure_ascii=False,
+            allow_nan=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        )
+    except (TypeError, ValueError, RecursionError):
+        fail()
+    return (encoded + "\n").encode("utf-8")
+
+
+def exact_dict(value: object, keys: set[str]) -> dict[str, object]:
+    if type(value) is not dict or set(value) != keys:
+        fail()
+    if any(type(key) is not str for key in value):
+        fail()
+    return value
+
+
+def exact_list(value: object) -> list[object]:
+    if type(value) is not list:
+        fail()
+    return value
+
+
+def bounded_string(value: object, *, limit: int = 4096) -> str:
+    if type(value) is not str or not value or len(value) > limit:
+        fail()
+    if any(ord(character) < 0x20 for character in value):
+        fail()
+    return value
+
+
+def digest_string(value: object) -> str:
+    digest = bounded_string(value, limit=64)
+    if len(digest) != 64 or any(
+        character not in "0123456789abcdef" for character in digest
+    ):
+        fail()
+    return digest
+
+
+def safe_path(value: object) -> str:
+    path = bounded_string(value, limit=1024)
+    pure = PurePosixPath(path)
+    if (
+        pure.is_absolute()
+        or not pure.parts
+        or path != pure.as_posix()
+        or any(part in {"", ".", ".."} or len(part) > 255 for part in pure.parts)
+        or pure.parts[0] == ".venv"
+    ):
+        fail()
+    return path
+
+
+def canonical_digest(value: object, expected: object) -> None:
+    if sha256(canonical_json(value)) != digest_string(expected):
+        fail()
+
+
+def validate_packages(value: object, expected_digest: object) -> None:
+    packages = exact_list(value)
+    if not packages:
+        fail()
+    identities: list[tuple[str, str]] = []
+    for raw in packages:
+        package = exact_dict(raw, {"name", "version"})
+        identities.append(
+            (
+                bounded_string(package["name"], limit=256),
+                bounded_string(package["version"], limit=256),
+            )
+        )
+    if identities != sorted(set(identities)):
+        fail()
+    canonical_digest(packages, expected_digest)
+
+
+def validate_source(value: object) -> tuple[list[dict[str, object]], dict[str, str]]:
+    source = exact_dict(value, {"tracked", "tracked_set_sha256"})
+    tracked_raw = exact_list(source["tracked"])
+    if not tracked_raw or len(tracked_raw) > 100_000:
+        fail()
+    tracked: list[dict[str, object]] = []
+    hashes: dict[str, str] = {}
+    for raw in tracked_raw:
+        entry = exact_dict(raw, {"git_mode", "path", "sha256"})
+        git_mode = bounded_string(entry["git_mode"], limit=6)
+        if git_mode not in EXPECTED_GIT_MODES:
+            fail()
+        path = safe_path(entry["path"])
+        digest = digest_string(entry["sha256"])
+        if path in hashes:
+            fail()
+        hashes[path] = digest
+        tracked.append({"git_mode": git_mode, "path": path, "sha256": digest})
+    if [entry["path"] for entry in tracked] != sorted(hashes):
+        fail()
+    if "scripts/verify-release-runtime" not in hashes:
+        fail()
+    canonical_digest(tracked, source["tracked_set_sha256"])
+    return tracked, hashes
+
+
+def validate_dependencies(value: object, tracked_hashes: dict[str, str]) -> None:
+    dependencies = exact_dict(
+        value,
+        {"lock_path", "lock_sha256", "package_set_sha256", "packages"},
+    )
+    lock_path = safe_path(dependencies["lock_path"])
+    if lock_path != "uv.lock":
+        fail()
+    lock_digest = digest_string(dependencies["lock_sha256"])
+    if tracked_hashes.get(lock_path) != lock_digest:
+        fail()
+    validate_packages(dependencies["packages"], dependencies["package_set_sha256"])
+
+
+def validate_migrations(value: object, tracked_hashes: dict[str, str]) -> None:
+    migrations = exact_dict(value, {"leaf_set_sha256", "leaves"})
+    leaves = exact_list(migrations["leaves"])
+    if not leaves or len(leaves) > 100_000:
+        fail()
+    identities: list[tuple[str, str]] = []
+    for raw in leaves:
+        leaf = exact_dict(
+            raw,
+            {"app", "module", "name", "origin", "path", "sha256"},
+        )
+        app = bounded_string(leaf["app"], limit=512)
+        bounded_string(leaf["module"], limit=1024)
+        name = bounded_string(leaf["name"], limit=512)
+        origin = bounded_string(leaf["origin"], limit=16)
+        path = safe_path(leaf["path"])
+        digest = digest_string(leaf["sha256"])
+        if origin == "source":
+            if tracked_hashes.get(path) != digest:
+                fail()
+        elif origin == "django":
+            if not path.startswith("django/"):
+                fail()
+        else:
+            fail()
+        identities.append((app, name))
+    if identities != sorted(set(identities)):
+        fail()
+    canonical_digest(leaves, migrations["leaf_set_sha256"])
+
+
+def validate_runtime(value: object, tracked_hashes: dict[str, str]) -> None:
+    runtime = exact_dict(value, {"path", "sha256", "versions"})
+    path = safe_path(runtime["path"])
+    digest = digest_string(runtime["sha256"])
+    if path != "runtime/versions.toml" or tracked_hashes.get(path) != digest:
+        fail()
+    versions = exact_dict(runtime["versions"], RUNTIME_VERSION_KEYS)
+    for version in versions.values():
+        bounded_string(version, limit=256)
+    if versions["psycopg_distribution"] != "binary-wheel":
+        fail()
+
+
+def validate_static(value: object, tracked_hashes: dict[str, str]) -> None:
+    static_data = exact_dict(
+        value,
+        {
+            "collected",
+            "collected_set_sha256",
+            "origins",
+            "tracked",
+            "tracked_set_sha256",
+        },
+    )
+    collected = exact_list(static_data["collected"])
+    origins = exact_list(static_data["origins"])
+    tracked = exact_list(static_data["tracked"])
+    if not collected or len(collected) > 100_000 or len(origins) != len(collected):
+        fail()
+    collected_hashes: dict[str, str] = {}
+    for raw in collected:
+        entry = exact_dict(raw, {"path", "sha256"})
+        path = safe_path(entry["path"])
+        digest = digest_string(entry["sha256"])
+        if path in collected_hashes:
+            fail()
+        collected_hashes[path] = digest
+    if list(collected_hashes) != sorted(collected_hashes):
+        fail()
+    canonical_digest(collected, static_data["collected_set_sha256"])
+
+    expected_tracked: list[dict[str, object]] = []
+    origin_paths: list[str] = []
+    for raw in origins:
+        origin = exact_dict(
+            raw, {"collected_path", "origin", "path", "sha256"}
+        )
+        collected_path = safe_path(origin["collected_path"])
+        source_origin = bounded_string(origin["origin"], limit=16)
+        path = safe_path(origin["path"])
+        digest = digest_string(origin["sha256"])
+        if collected_hashes.get(collected_path) != digest:
+            fail()
+        if source_origin == "source":
+            if tracked_hashes.get(path) != digest:
+                fail()
+            expected_tracked.append(
+                {
+                    "collected_path": collected_path,
+                    "path": path,
+                    "sha256": digest,
+                }
+            )
+        elif source_origin == "django":
+            if not path.startswith("django/"):
+                fail()
+        else:
+            fail()
+        origin_paths.append(collected_path)
+    if origin_paths != sorted(collected_hashes):
+        fail()
+    expected_tracked.sort(key=lambda item: (item["collected_path"], item["path"]))
+
+    normalized_tracked: list[dict[str, object]] = []
+    for raw in tracked:
+        entry = exact_dict(raw, {"collected_path", "path", "sha256"})
+        normalized_tracked.append(
+            {
+                "collected_path": safe_path(entry["collected_path"]),
+                "path": safe_path(entry["path"]),
+                "sha256": digest_string(entry["sha256"]),
+            }
+        )
+    if normalized_tracked != expected_tracked:
+        fail()
+    canonical_digest(normalized_tracked, static_data["tracked_set_sha256"])
+
+
+def validate_manifest(data: bytes) -> tuple[str, list[dict[str, object]]]:
+    try:
+        manifest = json.loads(data.decode("utf-8"))
+    except (UnicodeDecodeError, json.JSONDecodeError, RecursionError):
+        fail()
+    if canonical_json(manifest) != data:
+        fail()
+    manifest = exact_dict(manifest, TOP_LEVEL_KEYS)
+    if type(manifest["schema_version"]) is not int or manifest["schema_version"] != 1:
+        fail()
+
+    archive = exact_dict(
+        manifest["archive"],
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
+    if archive != {
+        "compression": "none",
+        "format": "ustar",
+        "gid": 0,
+        "group": "root",
+        "mtime": 0,
+        "regular_modes": ["0644", "0755"],
+        "uid": 0,
+        "user": "root",
+    }:
+        fail()
+
+    object_format = bounded_string(manifest["git_object_format"], limit=6)
+    sha_length = {"sha1": 40, "sha256": 64}.get(object_format)
+    release_sha = bounded_string(manifest["release_sha"], limit=64)
+    if sha_length is None or len(release_sha) != sha_length or any(
+        character not in "0123456789abcdef" for character in release_sha
+    ):
+        fail()
+
+    tracked, tracked_hashes = validate_source(manifest["source"])
+    validate_dependencies(manifest["dependencies"], tracked_hashes)
+    validate_migrations(manifest["migrations"], tracked_hashes)
+    validate_runtime(manifest["runtime"], tracked_hashes)
+    validate_static(manifest["static"], tracked_hashes)
+    return release_sha, tracked
+
+
+def safe_status(
+    status: os.stat_result, *, directory: bool, owner_uid: int
+) -> None:
+    expected_type = stat.S_ISDIR if directory else stat.S_ISREG
+    if (
+        not expected_type(status.st_mode)
+        or status.st_uid != owner_uid
+        or status.st_mode & 0o022
+    ):
+        fail()
+
+
+def matching_status(left: os.stat_result, right: os.stat_result) -> None:
+    if (left.st_dev, left.st_ino) != (right.st_dev, right.st_ino):
+        fail()
+
+
+def read_bounded(fd: int, limit: int) -> tuple[bytes, os.stat_result]:
+    before = os.fstat(fd)
+    if not stat.S_ISREG(before.st_mode) or before.st_size < 0 or before.st_size > limit:
+        fail()
+    chunks: list[bytes] = []
+    remaining = before.st_size
+    while remaining:
+        chunk = os.read(fd, min(remaining, 1024 * 1024))
+        if not chunk:
+            fail()
+        chunks.append(chunk)
+        remaining -= len(chunk)
+    if os.read(fd, 1):
+        fail()
+    after = os.fstat(fd)
+    if (
+        (before.st_dev, before.st_ino, before.st_size, before.st_mode, before.st_uid)
+        != (after.st_dev, after.st_ino, after.st_size, after.st_mode, after.st_uid)
+    ):
+        fail()
+    return b"".join(chunks), after
+
+
+def open_child_directory(parent_fd: int, name: str, *, owner_uid: int) -> int:
+    try:
+        child_fd = os.open(name, DIRECTORY_FLAGS, dir_fd=parent_fd)
+        status = os.fstat(child_fd)
+    except OSError:
+        fail()
+    safe_status(status, directory=True, owner_uid=owner_uid)
+    return child_fd
+
+
+def open_tracked_file(
+    source_fd: int, relative: str, *, owner_uid: int
+) -> int:
+    parts = PurePosixPath(relative).parts
+    directory_fd = os.dup(source_fd)
+    try:
+        for part in parts[:-1]:
+            next_fd = open_child_directory(
+                directory_fd, part, owner_uid=owner_uid
+            )
+            os.close(directory_fd)
+            directory_fd = next_fd
+        try:
+            return os.open(parts[-1], OPEN_FLAGS, dir_fd=directory_fd)
+        except OSError:
+            fail()
+    finally:
+        os.close(directory_fd)
+
+
+def scan_source_tree(source_fd: int, *, owner_uid: int) -> dict[str, str]:
+    observed: dict[str, str] = {}
+    stack: list[tuple[int, tuple[str, ...]]] = [(os.dup(source_fd), ())]
+    try:
+        while stack:
+            directory_fd, prefix = stack.pop()
+            try:
+                entries = list(os.scandir(directory_fd))
+            except OSError:
+                os.close(directory_fd)
+                fail()
+            for entry in entries:
+                if type(entry.name) is not str or entry.name in {"", ".", ".."}:
+                    os.close(directory_fd)
+                    fail()
+                parts = (*prefix, entry.name)
+                relative = PurePosixPath(*parts).as_posix()
+                try:
+                    status = entry.stat(follow_symlinks=False)
+                except OSError:
+                    os.close(directory_fd)
+                    fail()
+                if not prefix and entry.name == ".venv":
+                    safe_status(status, directory=True, owner_uid=owner_uid)
+                    observed[relative] = "venv"
+                    continue
+                if stat.S_ISDIR(status.st_mode):
+                    safe_status(status, directory=True, owner_uid=owner_uid)
+                    observed[relative] = "directory"
+                    child_fd = open_child_directory(
+                        directory_fd, entry.name, owner_uid=owner_uid
+                    )
+                    stack.append((child_fd, parts))
+                elif stat.S_ISREG(status.st_mode):
+                    if status.st_uid != owner_uid:
+                        os.close(directory_fd)
+                        fail()
+                    observed[relative] = "file"
+                else:
+                    os.close(directory_fd)
+                    fail()
+            os.close(directory_fd)
+    except Exception:
+        for pending_fd, _prefix in stack:
+            os.close(pending_fd)
+        raise
+    return observed
+
+
+def expected_tree(tracked: list[dict[str, object]]) -> dict[str, str]:
+    expected: dict[str, str] = {}
+    for entry in tracked:
+        pure = PurePosixPath(str(entry["path"]))
+        for index in range(1, len(pure.parts)):
+            directory = PurePosixPath(*pure.parts[:index]).as_posix()
+            if expected.get(directory) == "file":
+                fail()
+            expected[directory] = "directory"
+        path = pure.as_posix()
+        if path in expected:
+            fail()
+        expected[path] = "file"
+    return expected
+
+
+def verify_files(
+    source_fd: int,
+    tracked: list[dict[str, object]],
+    *,
+    owner_uid: int,
+) -> None:
+    total = 0
+    for entry in tracked:
+        file_fd = open_tracked_file(
+            source_fd, str(entry["path"]), owner_uid=owner_uid
+        )
+        try:
+            data, status = read_bounded(file_fd, TRACKED_FILE_LIMIT)
+        finally:
+            os.close(file_fd)
+        expected_mode = EXPECTED_GIT_MODES[str(entry["git_mode"])]
+        if (
+            status.st_uid != owner_uid
+            or stat.S_IMODE(status.st_mode) != expected_mode
+            or sha256(data) != entry["sha256"]
+        ):
+            fail()
+        total += len(data)
+        if total > TRACKED_TOTAL_LIMIT:
+            fail()
+
+
+def canonical_file_path(
+    path: Path, *, directory: bool, owner_uid: int
+) -> os.stat_result:
+    if not path.is_absolute() or Path(os.path.abspath(path)) != path:
+        fail()
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except (OSError, RuntimeError):
+        fail()
+    if resolved != path:
+        fail()
+    safe_status(status, directory=directory, owner_uid=owner_uid)
+    return status
+
+
+def owner_uid(value: str) -> int:
+    if not value or len(value) > 20 or not value.isascii() or not value.isdigit():
+        fail()
+    parsed = int(value, 10)
+    if parsed < 0 or parsed > 2**32 - 2:
+        fail()
+    return parsed
+
+
+def expected_release_sha(value: str) -> str:
+    value = bounded_string(value, limit=64)
+    if len(value) not in {40, 64} or any(
+        character not in "0123456789abcdef" for character in value
+    ):
+        fail()
+    return value
+
+
+def canonical_external_file(
+    value: str,
+    *,
+    allowed_owner_uids: set[int] | None = None,
+    executable: bool = False,
+) -> tuple[Path, os.stat_result]:
+    if type(value) is not str or not value or len(value) > 4096:
+        fail()
+    path = Path(value)
+    if not path.is_absolute() or Path(os.path.abspath(path)) != path:
+        fail()
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except (OSError, RuntimeError):
+        fail()
+    if resolved != path or not stat.S_ISREG(status.st_mode) or status.st_mode & 0o022:
+        fail()
+    if allowed_owner_uids is not None and status.st_uid not in allowed_owner_uids:
+        fail()
+    if executable and not status.st_mode & 0o111:
+        fail()
+    validate_external_ancestors(
+        path, allowed_owner_uids=allowed_owner_uids or {status.st_uid}
+    )
+    return path, status
+
+
+def canonical_external_directory(
+    value: str,
+    *,
+    allowed_owner_uids: set[int],
+) -> tuple[Path, os.stat_result]:
+    if type(value) is not str or not value or len(value) > 4096:
+        fail()
+    path = Path(value)
+    if not path.is_absolute() or Path(os.path.abspath(path)) != path:
+        fail()
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except (OSError, RuntimeError):
+        fail()
+    if (
+        resolved != path
+        or not stat.S_ISDIR(status.st_mode)
+        or status.st_uid not in allowed_owner_uids
+        or status.st_mode & 0o022
+    ):
+        fail()
+    validate_external_ancestors(path, allowed_owner_uids=allowed_owner_uids)
+    return path, status
+
+
+def validate_external_ancestors(
+    path: Path, *, allowed_owner_uids: set[int]
+) -> None:
+    allowed = {0, *allowed_owner_uids}
+    current = path.parent
+    while True:
+        try:
+            status = current.lstat()
+            resolved = current.resolve(strict=True)
+        except (OSError, RuntimeError):
+            fail()
+        if (
+            resolved != current
+            or not stat.S_ISDIR(status.st_mode)
+            or status.st_uid not in allowed
+        ):
+            fail()
+        writable = status.st_mode & 0o022
+        root_sticky_boundary = (
+            status.st_uid == 0
+            and bool(status.st_mode & stat.S_ISVTX)
+            and bool(status.st_mode & 0o002)
+        )
+        if writable and not root_sticky_boundary:
+            fail()
+        if current == current.parent:
+            return
+        current = current.parent
+
+
+def hash_external_file(path: Path, *, limit: int = VENV_FILE_LIMIT) -> str:
+    try:
+        fd = os.open(path, OPEN_FLAGS)
+    except OSError:
+        fail()
+    try:
+        before = os.fstat(fd)
+        if (
+            not stat.S_ISREG(before.st_mode)
+            or before.st_size < 0
+            or before.st_size > limit
+        ):
+            fail()
+        digest = hashlib.sha256()
+        remaining = before.st_size
+        while remaining:
+            block = os.read(fd, min(remaining, 1024 * 1024))
+            if not block:
+                fail()
+            digest.update(block)
+            remaining -= len(block)
+        if os.read(fd, 1):
+            fail()
+        after = os.fstat(fd)
+        if (
+            before.st_dev,
+            before.st_ino,
+            before.st_size,
+            before.st_mode,
+            before.st_uid,
+        ) != (
+            after.st_dev,
+            after.st_ino,
+            after.st_size,
+            after.st_mode,
+            after.st_uid,
+        ):
+            fail()
+        return digest.hexdigest()
+    finally:
+        os.close(fd)
+
+
+def bootstrap_identity(
+    bootstrap_python: str,
+    expected_digest: str,
+    bootstrap_prefix: str,
+    tree_spec: str,
+    expected_tree_spec_digest: str,
+    expected_pristine_tree_digest: str,
+    manifest_versions: dict[str, object],
+    *,
+    trusted_owner_uid: int,
+    require_root_tree: bool,
+) -> tuple[Path, dict[str, object]]:
+    digest = digest_string(expected_digest)
+    allowed_tree_owners = {0} if require_root_tree else {0, trusted_owner_uid}
+    prefix, prefix_status = canonical_external_directory(
+        bootstrap_prefix, allowed_owner_uids=allowed_tree_owners
+    )
+    spec, spec_status = canonical_external_file(
+        tree_spec, allowed_owner_uids=allowed_tree_owners
+    )
+    tree_spec_digest = digest_string(expected_tree_spec_digest)
+    pristine_tree_digest = digest_string(expected_pristine_tree_digest)
+    if (
+        digest != PINNED_BOOTSTRAP_PYTHON_SHA256
+        or pristine_tree_digest != PINNED_BOOTSTRAP_TREE_SHA256
+        or tree_spec_digest != PINNED_BOOTSTRAP_MTREE_SHA256
+        or hash_external_file(spec, limit=RECEIPT_LIMIT) != tree_spec_digest
+        or (require_root_tree and (prefix_status.st_uid != 0 or spec_status.st_uid != 0))
+    ):
+        fail()
+    path, _status = canonical_external_file(
+        bootstrap_python,
+        allowed_owner_uids=allowed_tree_owners,
+        executable=True,
+    )
+    expected_binary = (
+        prefix
+        / "bin"
+        / f"python{sys.version_info.major}.{sys.version_info.minor}"
+    )
+    if path != expected_binary:
+        fail()
+    try:
+        same_executable = os.path.samefile(path, sys.executable)
+    except OSError:
+        fail()
+    if not same_executable or hash_external_file(path) != digest:
+        fail()
+    version = ".".join(str(part) for part in sys.version_info[:3])
+    expected_version = bounded_string(manifest_versions["python"], limit=256)
+    if version != expected_version or platform.python_implementation() != "CPython":
+        fail()
+    identity = {
+        "implementation": "CPython",
+        "machine": bounded_string(platform.machine(), limit=256),
+        "python_sha256": digest,
+        "python_version": version,
+        "pristine_tree_sha256": pristine_tree_digest,
+        "system": bounded_string(platform.system(), limit=256),
+        "tree_spec_format": PINNED_BOOTSTRAP_MTREE_SCHEMA,
+        "tree_spec_sha256": tree_spec_digest,
+    }
+    return path, identity
+
+
+def verify_source(
+    *,
+    trusted_owner_uid: int,
+    expected_release: str,
+    expected_manifest_digest: str,
+    expected_verifier_digest: str,
+) -> tuple[Path, Path, str, str, dict[str, object]]:
+    script_path = Path(__file__)
+    if (
+        script_path.name != "verify-release-runtime"
+        or script_path.parent.name != "scripts"
+        or script_path.parent.parent.name != "source"
+        or script_path.parent.parent.parent.name != "release"
+    ):
+        fail()
+    canonical_file_path(
+        script_path, directory=False, owner_uid=trusted_owner_uid
+    )
+    validate_external_ancestors(
+        script_path, allowed_owner_uids={trusted_owner_uid}
+    )
+    scripts_root = script_path.parent
+    source_root = scripts_root.parent
+    release_root = source_root.parent
+    scripts_status = canonical_file_path(
+        scripts_root, directory=True, owner_uid=trusted_owner_uid
+    )
+    source_status = canonical_file_path(
+        source_root, directory=True, owner_uid=trusted_owner_uid
+    )
+    release_status = canonical_file_path(
+        release_root, directory=True, owner_uid=trusted_owner_uid
+    )
+
+    try:
+        release_fd = os.open(release_root, DIRECTORY_FLAGS)
+    except OSError:
+        fail()
+    try:
+        matching_status(release_status, os.fstat(release_fd))
+        source_fd = open_child_directory(
+            release_fd, "source", owner_uid=trusted_owner_uid
+        )
+        try:
+            matching_status(source_status, os.fstat(source_fd))
+            scripts_fd = open_child_directory(
+                source_fd, "scripts", owner_uid=trusted_owner_uid
+            )
+            try:
+                matching_status(scripts_status, os.fstat(scripts_fd))
+            finally:
+                os.close(scripts_fd)
+            try:
+                manifest_fd = os.open("manifest.json", OPEN_FLAGS, dir_fd=release_fd)
+            except OSError:
+                fail()
+            try:
+                manifest_data, manifest_status = read_bounded(
+                    manifest_fd, MANIFEST_LIMIT
+                )
+            finally:
+                os.close(manifest_fd)
+            safe_status(
+                manifest_status,
+                directory=False,
+                owner_uid=trusted_owner_uid,
+            )
+            if stat.S_IMODE(manifest_status.st_mode) not in {0o400, 0o440, 0o444, 0o600, 0o640, 0o644}:
+                fail()
+            manifest_digest = sha256(manifest_data)
+            if manifest_digest != digest_string(expected_manifest_digest):
+                fail()
+            release_sha, tracked = validate_manifest(manifest_data)
+            if release_sha != expected_release_sha(expected_release):
+                fail()
+            verifier_entries = [
+                entry
+                for entry in tracked
+                if entry["path"] == "scripts/verify-release-runtime"
+            ]
+            if (
+                len(verifier_entries) != 1
+                or verifier_entries[0]["sha256"]
+                != digest_string(expected_verifier_digest)
+            ):
+                fail()
+            manifest_value = json.loads(manifest_data.decode("utf-8"))
+            actual_tree = scan_source_tree(
+                source_fd, owner_uid=trusted_owner_uid
+            )
+            expected = expected_tree(tracked)
+            if actual_tree.get(".venv") == "venv":
+                expected[".venv"] = "venv"
+            if actual_tree != expected:
+                fail()
+            verify_files(source_fd, tracked, owner_uid=trusted_owner_uid)
+            if (
+                scan_source_tree(source_fd, owner_uid=trusted_owner_uid)
+                != actual_tree
+            ):
+                fail()
+            matching_status(source_status, os.fstat(source_fd))
+        finally:
+            os.close(source_fd)
+        matching_status(release_status, os.fstat(release_fd))
+    finally:
+        os.close(release_fd)
+    return source_root, release_root, release_sha, manifest_digest, manifest_value
+
+
+def inventory_mode(status: os.stat_result) -> str:
+    return f"{stat.S_IFMT(status.st_mode) | stat.S_IMODE(status.st_mode):06o}"
+
+
+def read_inventory_file(
+    fd: int,
+) -> tuple[bytes, str, int, os.stat_result]:
+    before = os.fstat(fd)
+    if (
+        not stat.S_ISREG(before.st_mode)
+        or before.st_nlink != 1
+        or before.st_size < 0
+        or before.st_size > VENV_FILE_LIMIT
+    ):
+        fail()
+    digest = hashlib.sha256()
+    chunks: list[bytes] = []
+    remaining = before.st_size
+    while remaining:
+        block = os.read(fd, min(remaining, 1024 * 1024))
+        if not block:
+            fail()
+        digest.update(block)
+        chunks.append(block)
+        remaining -= len(block)
+    if os.read(fd, 1):
+        fail()
+    after = os.fstat(fd)
+    if (
+        before.st_dev,
+        before.st_ino,
+        before.st_size,
+        before.st_mode,
+        before.st_uid,
+        before.st_nlink,
+    ) != (
+        after.st_dev,
+        after.st_ino,
+        after.st_size,
+        after.st_mode,
+        after.st_uid,
+        after.st_nlink,
+    ):
+        fail()
+    return b"".join(chunks), digest.hexdigest(), before.st_size, after
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
+    environment_root: Path,
+    relative: str,
+    content: bytes,
+    console_scripts: dict[str, bytes],
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
+                io.StringIO(
+                    content.decode("utf-8", "strict"), newline=""
+                )
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
+        raw_script = console_scripts.get(script_relative)
+        if raw_script is None:
+            fail()
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
+    bootstrap_python: Path,
+    relative: str,
+    content: bytes,
+    console_scripts: dict[str, bytes],
+) -> bytes:
+    environment_bytes = os.fsencode(str(environment_root))
+    python_parent_bytes = os.fsencode(str(bootstrap_python.parent))
+    if not environment_bytes or not python_parent_bytes:
+        fail()
+    if relative == "pyvenv.cfg":
+        expected = (
+            b"home = "
+            + python_parent_bytes
+            + b"\nimplementation = CPython\n"
+            + b"uv = 0.12.6\n"
+            + b"version_info = 3.14.7\n"
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
+        environment_root, relative, content, console_scripts
+    )
+    if environment_bytes in canonical or python_parent_bytes in canonical:
+        fail()
+    return canonical
+
+
+def safe_inventory_status(
+    status: os.stat_result, *, owner_uid: int, symlink: bool = False
+) -> None:
+    if status.st_uid != owner_uid:
+        fail()
+    if not symlink and status.st_mode & 0o022:
+        fail()
+
+
+def inventory_venv(
+    source_root: Path,
+    *,
+    trusted_owner_uid: int,
+    bootstrap_python: Path,
+) -> tuple[list[dict[str, object]], dict[str, dict[str, object]]]:
+    try:
+        source_fd = os.open(source_root, DIRECTORY_FLAGS)
+    except OSError:
+        fail()
+    try:
+        venv_fd = open_child_directory(
+            source_fd, ".venv", owner_uid=trusted_owner_uid
+        )
+    finally:
+        os.close(source_fd)
+    environment_root = source_root / ".venv"
+    entries: list[dict[str, object]] = []
+    raw_by_path: dict[str, dict[str, object]] = {}
+    console_scripts: dict[str, bytes] = {}
+    regular_identities: set[tuple[int, int]] = set()
+    total_size = 0
+    stack: list[tuple[int, tuple[str, ...]]] = [(venv_fd, ())]
+    try:
+        while stack:
+            directory_fd, prefix = stack.pop()
+            try:
+                directory_status = os.fstat(directory_fd)
+                safe_inventory_status(
+                    directory_status, owner_uid=trusted_owner_uid
+                )
+                names = sorted(os.listdir(directory_fd))
+            except OSError:
+                os.close(directory_fd)
+                fail()
+            if len(entries) + len(names) + 1 > VENV_ENTRY_LIMIT:
+                os.close(directory_fd)
+                fail()
+            children: list[dict[str, str]] = []
+            pending_directories: list[tuple[int, tuple[str, ...]]] = []
+            for name in names:
+                if (
+                    type(name) is not str
+                    or not name
+                    or name in {".", ".."}
+                    or "/" in name
+                    or len(name.encode("utf-8")) > 255
+                    or any(ord(character) < 0x20 for character in name)
+                ):
+                    os.close(directory_fd)
+                    fail()
+                parts = (*prefix, name)
+                relative = PurePosixPath(*parts).as_posix()
+                try:
+                    status = os.stat(
+                        name, dir_fd=directory_fd, follow_symlinks=False
+                    )
+                except OSError:
+                    os.close(directory_fd)
+                    fail()
+                if stat.S_ISDIR(status.st_mode):
+                    safe_inventory_status(status, owner_uid=trusted_owner_uid)
+                    child_fd = open_child_directory(
+                        directory_fd, name, owner_uid=trusted_owner_uid
+                    )
+                    pending_directories.append((child_fd, parts))
+                    children.append({"name": name, "type": "directory"})
+                    continue
+                if stat.S_ISREG(status.st_mode):
+                    safe_inventory_status(status, owner_uid=trusted_owner_uid)
+                    identity = (status.st_dev, status.st_ino)
+                    if status.st_nlink != 1 or identity in regular_identities:
+                        os.close(directory_fd)
+                        fail()
+                    regular_identities.add(identity)
+                    try:
+                        file_fd = os.open(
+                            name, OPEN_FLAGS, dir_fd=directory_fd
+                        )
+                    except OSError:
+                        os.close(directory_fd)
+                        fail()
+                    try:
+                        (
+                            content,
+                            raw_digest,
+                            raw_size,
+                            final_status,
+                        ) = read_inventory_file(file_fd)
+                    finally:
+                        os.close(file_fd)
+                    if (
+                        status.st_dev,
+                        status.st_ino,
+                        status.st_mode,
+                        status.st_uid,
+                        status.st_nlink,
+                    ) != (
+                        final_status.st_dev,
+                        final_status.st_ino,
+                        final_status.st_mode,
+                        final_status.st_uid,
+                        final_status.st_nlink,
+                    ):
+                        os.close(directory_fd)
+                        fail()
+                    if relative in EXPECTED_CONSOLE_SCRIPTS:
+                        console_scripts[relative] = content
+                    canonical_content = canonical_environment_file_content(
+                        environment_root,
+                        bootstrap_python,
+                        relative,
+                        content,
+                        console_scripts,
+                    )
+                    canonical_size = len(canonical_content)
+                    canonical_digest = sha256(canonical_content)
+                    total_size += canonical_size
+                    if total_size > VENV_TOTAL_LIMIT:
+                        os.close(directory_fd)
+                        fail()
+                    entries.append(
+                        {
+                            "mode": inventory_mode(status),
+                            "path": relative,
+                            "sha256": canonical_digest,
+                            "size": canonical_size,
+                            "type": "file",
+                        }
+                    )
+                    raw_by_path[relative] = {
+                        "mode": inventory_mode(status),
+                        "path": relative,
+                        "sha256": raw_digest,
+                        "size": raw_size,
+                        "type": "file",
+                    }
+                    children.append({"name": name, "type": "file"})
+                    continue
+                if stat.S_ISLNK(status.st_mode):
+                    safe_inventory_status(
+                        status, owner_uid=trusted_owner_uid, symlink=True
+                    )
+                    try:
+                        target = os.readlink(name, dir_fd=directory_fd)
+                        after = os.stat(
+                            name, dir_fd=directory_fd, follow_symlinks=False
+                        )
+                    except OSError:
+                        os.close(directory_fd)
+                        fail()
+                    if (
+                        status.st_dev,
+                        status.st_ino,
+                        status.st_mode,
+                        status.st_uid,
+                    ) != (
+                        after.st_dev,
+                        after.st_ino,
+                        after.st_mode,
+                        after.st_uid,
+                    ):
+                        os.close(directory_fd)
+                        fail()
+                    target_bytes = os.fsencode(target)
+                    if not target_bytes or len(target_bytes) > 4096:
+                        os.close(directory_fd)
+                        fail()
+                    canonical_target = target_bytes
+                    if relative == "bin/python":
+                        if target_bytes != os.fsencode(str(bootstrap_python)):
+                            os.close(directory_fd)
+                            fail()
+                        canonical_target = CANONICAL_BOOTSTRAP_PYTHON
+                    elif (
+                        os.fsencode(str(environment_root)) in target_bytes
+                        or os.fsencode(str(bootstrap_python.parent))
+                        in target_bytes
+                    ):
+                        os.close(directory_fd)
+                        fail()
+                    entries.append(
+                        {
+                            "mode": inventory_mode(status),
+                            "path": relative,
+                            "sha256": sha256(canonical_target),
+                            "size": len(canonical_target),
+                            "type": "symlink",
+                        }
+                    )
+                    raw_by_path[relative] = {
+                        "mode": inventory_mode(status),
+                        "path": relative,
+                        "sha256": sha256(target_bytes),
+                        "size": len(target_bytes),
+                        "type": "symlink",
+                    }
+                    children.append({"name": name, "type": "symlink"})
+                    continue
+                os.close(directory_fd)
+                fail()
+            directory_path = (
+                PurePosixPath(*prefix).as_posix() if prefix else "."
+            )
+            entries.append(
+                directory_entry := {
+                    "mode": inventory_mode(directory_status),
+                    "path": directory_path,
+                    "sha256": sha256(canonical_json(children)),
+                    "size": 0,
+                    "type": "directory",
+                }
+            )
+            raw_by_path[directory_path] = dict(directory_entry)
+            os.close(directory_fd)
+            stack.extend(reversed(pending_directories))
+    except Exception:
+        for pending_fd, _prefix in stack:
+            try:
+                os.close(pending_fd)
+            except OSError:
+                pass
+        raise
+    entries.sort(key=lambda item: str(item["path"]))
+    by_path = {str(entry["path"]): entry for entry in entries}
+    if (
+        len(by_path) != len(entries)
+        or len(raw_by_path) != len(entries)
+        or by_path.get(".", {}).get("type") != "directory"
+        or raw_by_path.get(".", {}).get("type") != "directory"
+    ):
+        fail()
+    return entries, raw_by_path
+
+
+def open_venv_relative(venv_fd: int, relative: str) -> int:
+    pure = PurePosixPath(relative)
+    if (
+        pure.is_absolute()
+        or not pure.parts
+        or relative != pure.as_posix()
+        or any(part in {"", ".", ".."} for part in pure.parts)
+    ):
+        fail()
+    directory_fd = os.dup(venv_fd)
+    try:
+        for part in pure.parts[:-1]:
+            try:
+                next_fd = os.open(
+                    part, DIRECTORY_FLAGS, dir_fd=directory_fd
+                )
+            except OSError:
+                fail()
+            os.close(directory_fd)
+            directory_fd = next_fd
+        try:
+            return os.open(pure.parts[-1], OPEN_FLAGS, dir_fd=directory_fd)
+        except OSError:
+            fail()
+    finally:
+        os.close(directory_fd)
+
+
+def read_venv_file(venv_fd: int, relative: str, *, limit: int) -> bytes:
+    fd = open_venv_relative(venv_fd, relative)
+    try:
+        data, _status = read_bounded(fd, limit)
+        return data
+    finally:
+        os.close(fd)
+
+
+def normalized_distribution_name(value: str) -> str:
+    normalized = re.sub(r"[-_.]+", "-", value).lower()
+    if (
+        not normalized
+        or len(normalized) > 256
+        or re.fullmatch(r"[a-z0-9]+(?:-[a-z0-9]+)*", normalized) is None
+    ):
+        fail()
+    return normalized
+
+
+def marker_environment() -> dict[str, str]:
+    full_version = ".".join(str(part) for part in sys.version_info[:3])
+    return {
+        "implementation_name": sys.implementation.name,
+        "platform_machine": platform.machine(),
+        "python_full_version": full_version,
+        "python_version": ".".join(str(part) for part in sys.version_info[:2]),
+        "sys_platform": sys.platform,
+    }
+
+
+def marker_applies(value: object, environment: dict[str, str]) -> bool:
+    marker = bounded_string(value, limit=512)
+    match = re.fullmatch(
+        r"(implementation_name|platform_machine|python_full_version|"
+        r"python_version|sys_platform) (==|!=) '([A-Za-z0-9_.-]{1,128})'",
+        marker,
+    )
+    if match is None:
+        fail()
+    name, operator, expected = match.groups()
+    actual = environment[name]
+    return actual == expected if operator == "==" else actual != expected
+
+
+def lock_dependency(value: object) -> tuple[str, tuple[str, ...]] | None:
+    if type(value) is not dict or not value:
+        fail()
+    allowed = {"name", "marker", "extra"}
+    if set(value) - allowed or "name" not in value:
+        fail()
+    environment = marker_environment()
+    if "marker" in value and not marker_applies(value["marker"], environment):
+        return None
+    name = normalized_distribution_name(
+        bounded_string(value["name"], limit=256)
+    )
+    extras_raw = value.get("extra", [])
+    extras: list[str] = []
+    if type(extras_raw) is not list:
+        fail()
+    for raw in extras_raw:
+        extras.append(
+            normalized_distribution_name(bounded_string(raw, limit=256))
+        )
+    if extras != sorted(set(extras)):
+        fail()
+    return name, tuple(extras)
+
+
+def selected_lock_packages(
+    source_root: Path,
+    manifest_packages: object,
+    expected_lock_digest: str,
+) -> dict[str, object]:
+    lock_path = source_root / "uv.lock"
+    try:
+        fd = os.open(lock_path, OPEN_FLAGS)
+    except OSError:
+        fail()
+    try:
+        lock_data, _status = read_bounded(fd, 64 * 1024 * 1024)
+    finally:
+        os.close(fd)
+    if sha256(lock_data) != digest_string(expected_lock_digest):
+        fail()
+    try:
+        document = tomllib.loads(lock_data.decode("utf-8", "strict"))
+    except (UnicodeDecodeError, tomllib.TOMLDecodeError):
+        fail()
+    packages_raw = document.get("package")
+    if type(packages_raw) is not list or not packages_raw:
+        fail()
+    packages: dict[str, dict[str, object]] = {}
+    virtual_roots: list[str] = []
+    all_locked: list[tuple[str, str]] = []
+    for raw in packages_raw:
+        if type(raw) is not dict:
+            fail()
+        name = normalized_distribution_name(
+            bounded_string(raw.get("name"), limit=256)
+        )
+        version = bounded_string(raw.get("version"), limit=256)
+        if name in packages:
+            fail()
+        source = raw.get("source")
+        if type(source) is not dict or len(source) != 1:
+            fail()
+        if source.get("virtual") == ".":
+            virtual_roots.append(name)
+        elif "registry" not in source:
+            fail()
+        packages[name] = raw
+        all_locked.append((name, version))
+    if len(virtual_roots) != 1:
+        fail()
+    manifested: list[tuple[str, str]] = []
+    for raw in exact_list(manifest_packages):
+        package = exact_dict(raw, {"name", "version"})
+        manifested.append(
+            (
+                normalized_distribution_name(
+                    bounded_string(package["name"], limit=256)
+                ),
+                bounded_string(package["version"], limit=256),
+            )
+        )
+    if sorted(all_locked) != manifested:
+        fail()
+
+    root_name = virtual_roots[0]
+    root = packages[root_name]
+    queue: list[tuple[str, tuple[str, ...]]] = []
+    for dependency in exact_list(root.get("dependencies", [])):
+        selected = lock_dependency(dependency)
+        if selected is not None:
+            queue.append(selected)
+    requested_extras: dict[str, set[str]] = {}
+    selected_names: set[str] = set()
+    while queue:
+        name, extras = queue.pop(0)
+        package = packages.get(name)
+        if package is None or name == root_name:
+            fail()
+        new_extras = set(extras) - requested_extras.setdefault(name, set())
+        first_visit = name not in selected_names
+        selected_names.add(name)
+        requested_extras[name].update(extras)
+        if first_visit:
+            for dependency in exact_list(package.get("dependencies", [])):
+                selected = lock_dependency(dependency)
+                if selected is not None:
+                    queue.append(selected)
+        optional = package.get("optional-dependencies", {})
+        if type(optional) is not dict:
+            fail()
+        for extra in sorted(new_extras):
+            candidates = optional.get(extra)
+            if type(candidates) is not list:
+                fail()
+            for dependency in candidates:
+                selected = lock_dependency(dependency)
+                if selected is not None:
+                    queue.append(selected)
+    selected_packages = [
+        {
+            "name": name,
+            "version": bounded_string(packages[name]["version"], limit=256),
+        }
+        for name in sorted(selected_names)
+    ]
+    if not selected_packages:
+        fail()
+    environment = marker_environment()
+    exact_dict(environment, LOCK_ENVIRONMENT_KEYS)
+    return {
+        "environment": environment,
+        "package_set_sha256": sha256(canonical_json(selected_packages)),
+        "packages": selected_packages,
+    }
+
+
+def resolve_record_path(site_packages: PurePosixPath, raw: str) -> str:
+    if not raw or len(raw) > 4096 or "\\" in raw or raw.startswith("/"):
+        fail()
+    parts = list(site_packages.parts)
+    for part in PurePosixPath(raw).parts:
+        if part in {"", "."}:
+            continue
+        if part == "..":
+            if not parts:
+                fail()
+            parts.pop()
+            continue
+        if len(part.encode("utf-8")) > 255 or any(
+            ord(character) < 0x20 for character in part
+        ):
+            fail()
+        parts.append(part)
+    if not parts:
+        fail()
+    return PurePosixPath(*parts).as_posix()
+
+
+def distribution_evidence(
+    source_root: Path,
+    inventory: dict[str, dict[str, object]],
+    manifest_packages: object,
+    *,
+    trusted_owner_uid: int,
+) -> list[dict[str, object]]:
+    expected: list[tuple[str, str]] = []
+    for raw in exact_list(manifest_packages):
+        package = exact_dict(raw, {"name", "version"})
+        expected.append(
+            (
+                normalized_distribution_name(
+                    bounded_string(package["name"], limit=256)
+                ),
+                bounded_string(package["version"], limit=256),
+            )
+        )
+    if expected != sorted(set(expected)):
+        fail()
+
+    dist_infos = sorted(
+        path
+        for path, entry in inventory.items()
+        if path != "."
+        and entry["type"] == "directory"
+        and PurePosixPath(path).name.endswith(".dist-info")
+    )
+    if not dist_infos:
+        fail()
+    try:
+        source_fd = os.open(source_root, DIRECTORY_FLAGS)
+    except OSError:
+        fail()
+    try:
+        venv_fd = open_child_directory(
+            source_fd, ".venv", owner_uid=trusted_owner_uid
+        )
+    finally:
+        os.close(source_fd)
+    evidence: list[dict[str, object]] = []
+    try:
+        for dist_info in dist_infos:
+            pure_dist = PurePosixPath(dist_info)
+            if len(pure_dist.parts) < 3 or pure_dist.parts[-2] != "site-packages":
+                fail()
+            metadata_path = (pure_dist / "METADATA").as_posix()
+            wheel_path = (pure_dist / "WHEEL").as_posix()
+            record_path = (pure_dist / "RECORD").as_posix()
+            if (
+                inventory.get(metadata_path, {}).get("type") != "file"
+                or inventory.get(wheel_path, {}).get("type") != "file"
+                or inventory.get(record_path, {}).get("type") != "file"
+            ):
+                fail()
+            metadata = read_venv_file(
+                venv_fd, metadata_path, limit=8 * 1024 * 1024
+            )
+            message = BytesParser().parsebytes(metadata, headersonly=True)
+            names = message.get_all("Name", [])
+            versions = message.get_all("Version", [])
+            if len(names) != 1 or len(versions) != 1:
+                fail()
+            name = normalized_distribution_name(
+                bounded_string(names[0], limit=256)
+            )
+            version = bounded_string(versions[0], limit=256)
+            wheel_data = read_venv_file(
+                venv_fd, wheel_path, limit=1024 * 1024
+            )
+            wheel_message = BytesParser().parsebytes(
+                wheel_data, headersonly=True
+            )
+            wheel_versions = wheel_message.get_all("Wheel-Version", [])
+            purelib_values = wheel_message.get_all("Root-Is-Purelib", [])
+            wheel_tags = wheel_message.get_all("Tag", [])
+            if (
+                wheel_versions != ["1.0"]
+                or len(purelib_values) != 1
+                or purelib_values[0] not in {"true", "false"}
+                or not wheel_tags
+            ):
+                fail()
+            normalized_tags = sorted(
+                bounded_string(tag, limit=256) for tag in wheel_tags
+            )
+            if normalized_tags != sorted(set(normalized_tags)) or any(
+                re.fullmatch(r"[A-Za-z0-9_.]+-[A-Za-z0-9_.]+-[A-Za-z0-9_.]+", tag)
+                is None
+                for tag in normalized_tags
+            ):
+                fail()
+            record_data = read_venv_file(
+                venv_fd, record_path, limit=16 * 1024 * 1024
+            )
+            try:
+                record_text = record_data.decode("utf-8", "strict")
+                rows = list(csv.reader(io.StringIO(record_text, newline="")))
+            except (UnicodeDecodeError, csv.Error):
+                fail()
+            normalized_rows: list[dict[str, object]] = []
+            observed_paths: set[str] = set()
+            verified_hashes = 0
+            site_packages = pure_dist.parent
+            for row in rows:
+                if len(row) != 3:
+                    fail()
+                resolved = resolve_record_path(site_packages, row[0])
+                if resolved in observed_paths:
+                    fail()
+                observed_paths.add(resolved)
+                installed = inventory.get(resolved)
+                if installed is None or installed["type"] != "file":
+                    fail()
+                hash_field, size_field = row[1], row[2]
+                if not hash_field:
+                    if resolved != record_path or size_field:
+                        fail()
+                    digest = ""
+                    size: int | str = ""
+                else:
+                    if not hash_field.startswith("sha256="):
+                        fail()
+                    encoded = hash_field.removeprefix("sha256=")
+                    try:
+                        decoded = base64.urlsafe_b64decode(
+                            encoded + "=" * (-len(encoded) % 4)
+                        )
+                    except (ValueError, TypeError):
+                        fail()
+                    if len(decoded) != 32 or decoded.hex() != installed["sha256"]:
+                        fail()
+                    if not size_field.isascii() or not size_field.isdigit():
+                        fail()
+                    size = int(size_field, 10)
+                    if size != installed["size"]:
+                        fail()
+                    digest = decoded.hex()
+                    verified_hashes += 1
+                normalized_rows.append(
+                    {
+                        "path": resolved,
+                        "sha256": digest,
+                        "size": size,
+                    }
+                )
+            normalized_rows.sort(key=lambda item: str(item["path"]))
+            if not normalized_rows or not {
+                metadata_path,
+                wheel_path,
+                record_path,
+            }.issubset(observed_paths):
+                fail()
+            evidence.append(
+                {
+                    "dist_info": dist_info,
+                    "name": name,
+                    "record_entries_sha256": sha256(
+                        canonical_json(normalized_rows)
+                    ),
+                    "record_entry_count": len(normalized_rows),
+                    "record_sha256": sha256(record_data),
+                    "verified_file_count": verified_hashes,
+                    "version": version,
+                    "wheel_root_is_purelib": purelib_values[0] == "true",
+                    "wheel_sha256": sha256(wheel_data),
+                    "wheel_tags": normalized_tags,
+                }
+            )
+    finally:
+        os.close(venv_fd)
+    evidence.sort(key=lambda item: (str(item["name"]), str(item["version"])))
+    observed = [(str(item["name"]), str(item["version"])) for item in evidence]
+    if observed != expected:
+        fail()
+    return evidence
+
+
+def verify_venv_python(source_root: Path, bootstrap_python: Path, digest: str) -> None:
+    python_path = source_root / ".venv" / "bin" / "python"
+    try:
+        status = python_path.lstat()
+        resolved = python_path.resolve(strict=True)
+    except (OSError, RuntimeError):
+        fail()
+    if not (stat.S_ISREG(status.st_mode) or stat.S_ISLNK(status.st_mode)):
+        fail()
+    if not os.access(python_path, os.X_OK):
+        fail()
+    try:
+        same = os.path.samefile(resolved, bootstrap_python)
+    except OSError:
+        same = False
+    if not same and hash_external_file(resolved) != digest:
+        fail()
+
+
+def runtime_receipt_value(
+    *,
+    source_root: Path,
+    release_sha: str,
+    manifest_digest: str,
+    manifest: dict[str, object],
+    trusted_owner_uid: int,
+    bootstrap: dict[str, object],
+    bootstrap_python: Path,
+    expected_dependency_environment_sha256: str,
+    expected_verifier_sha256: str,
+) -> dict[str, object]:
+    inventory, inventory_by_path = inventory_venv(
+        source_root,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap_python=bootstrap_python,
+    )
+    inventory_digest = sha256(canonical_json(inventory))
+    if inventory_digest != digest_string(
+        expected_dependency_environment_sha256
+    ):
+        fail()
+    verify_venv_python(
+        source_root, bootstrap_python, str(bootstrap["python_sha256"])
+    )
+    dependencies = exact_dict(
+        manifest["dependencies"],
+        {"lock_path", "lock_sha256", "package_set_sha256", "packages"},
+    )
+    lock_selection = selected_lock_packages(
+        source_root,
+        dependencies["packages"],
+        str(dependencies["lock_sha256"]),
+    )
+    distributions = distribution_evidence(
+        source_root,
+        inventory_by_path,
+        exact_dict(lock_selection, LOCK_SELECTION_KEYS)["packages"],
+        trusted_owner_uid=trusted_owner_uid,
+    )
+    return {
+        "bootstrap": bootstrap,
+        "canonical_inventory": inventory,
+        "canonical_inventory_sha256": inventory_digest,
+        "dependency_environment_sha256": inventory_digest,
+        "distributions": distributions,
+        "lock_sha256": digest_string(dependencies["lock_sha256"]),
+        "lock_selection": lock_selection,
+        "manifest_sha256": manifest_digest,
+        "owner_uid": trusted_owner_uid,
+        "release_sha": release_sha,
+        "schema_version": 1,
+        "verifier_sha256": digest_string(expected_verifier_sha256),
+    }
+
+
+def receipt_path(
+    value: str,
+    *,
+    release_root: Path,
+    trusted_owner_uid: int,
+    must_exist: bool,
+) -> Path:
+    if type(value) is not str or not value or len(value) > 4096:
+        fail()
+    path = Path(value)
+    if not path.is_absolute() or Path(os.path.abspath(path)) != path:
+        fail()
+    try:
+        canonical_parent = path.parent.resolve(strict=True)
+        parent_status = canonical_parent.lstat()
+    except (OSError, RuntimeError):
+        fail()
+    if (
+        canonical_parent != path.parent
+        or not stat.S_ISDIR(parent_status.st_mode)
+        or parent_status.st_uid != trusted_owner_uid
+        or parent_status.st_mode & 0o022
+    ):
+        fail()
+    validate_external_ancestors(
+        path, allowed_owner_uids={trusted_owner_uid}
+    )
+    try:
+        if path.is_relative_to(release_root):
+            fail()
+    except ValueError:
+        pass
+    if must_exist:
+        canonical_external_file(
+            value, allowed_owner_uids={trusted_owner_uid}
+        )
+    elif path.exists() or path.is_symlink():
+        fail()
+    return path
+
+
+def write_receipt(path: Path, data: bytes, *, trusted_owner_uid: int) -> None:
+    if not data or len(data) > RECEIPT_LIMIT:
+        fail()
+    try:
+        parent_fd = os.open(path.parent, DIRECTORY_FLAGS)
+    except OSError:
+        fail()
+    fd: int | None = None
+    created = False
+    try:
+        try:
+            fd = os.open(
+                path.name,
+                os.O_WRONLY
+                | os.O_CREAT
+                | os.O_EXCL
+                | os.O_CLOEXEC
+                | os.O_NOFOLLOW,
+                0o600,
+                dir_fd=parent_fd,
+            )
+            created = True
+        except OSError:
+            fail()
+        offset = 0
+        while offset < len(data):
+            written = os.write(fd, data[offset:])
+            if written <= 0:
+                fail()
+            offset += written
+        os.fsync(fd)
+        os.fchmod(fd, 0o444)
+        os.fsync(fd)
+        status = os.fstat(fd)
+        if (
+            status.st_uid != trusted_owner_uid
+            or stat.S_IMODE(status.st_mode) != 0o444
+            or status.st_size != len(data)
+        ):
+            fail()
+        os.fsync(parent_fd)
+        created = False
+    finally:
+        if fd is not None:
+            os.close(fd)
+        if created:
+            try:
+                os.unlink(path.name, dir_fd=parent_fd)
+                os.fsync(parent_fd)
+            except OSError:
+                pass
+        os.close(parent_fd)
+
+
+def verify_receipt(
+    path: Path,
+    expected_digest: str,
+    expected_value: dict[str, object],
+    *,
+    trusted_owner_uid: int,
+) -> None:
+    digest = digest_string(expected_digest)
+    try:
+        fd = os.open(path, OPEN_FLAGS)
+    except OSError:
+        fail()
+    try:
+        data, status = read_bounded(fd, RECEIPT_LIMIT)
+    finally:
+        os.close(fd)
+    if (
+        status.st_uid != trusted_owner_uid
+        or status.st_mode & 0o222
+        or stat.S_IMODE(status.st_mode)
+        not in {0o400, 0o440, 0o444}
+        or sha256(data) != digest
+    ):
+        fail()
+    try:
+        parsed = json.loads(data.decode("utf-8"))
+    except (UnicodeDecodeError, json.JSONDecodeError, RecursionError):
+        fail()
+    if canonical_json(parsed) != data or parsed != expected_value:
+        fail()
+    receipt = exact_dict(parsed, RECEIPT_KEYS)
+    exact_dict(receipt["bootstrap"], BOOTSTRAP_KEYS)
+
+
+def fixed_runtime_environment(
+    values: dict[str, str],
+    *,
+    release_sha: str,
+    tls_cert: str,
+    tls_key: str,
+    bind: str,
+) -> dict[str, str]:
+    environment = {
+        "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "PYTHONDONTWRITEBYTECODE": "1",
+        "TZ": "UTC",
+        "TRAVEL_READINESS_BIND": bind,
+        "TRAVEL_READINESS_BUILD": "0",
+        "TRAVEL_READINESS_RELEASE_SHA": release_sha,
+        "TRAVEL_READINESS_TLS_CERT_FILE": tls_cert,
+        "TRAVEL_READINESS_TLS_KEY_FILE": tls_key,
+    }
+    environment.update(values)
+    if set(environment) != {
+        "DJANGO_SETTINGS_MODULE",
+        "LANG",
+        "LC_ALL",
+        "PATH",
+        "PYTHONDONTWRITEBYTECODE",
+        "TZ",
+        "TRAVEL_READINESS_ALLOWED_HOSTS",
+        "TRAVEL_READINESS_BIND",
+        "TRAVEL_READINESS_BUILD",
+        "TRAVEL_READINESS_DB_HOST",
+        "TRAVEL_READINESS_DB_NAME",
+        "TRAVEL_READINESS_DB_PASSWORD",
+        "TRAVEL_READINESS_DB_PORT",
+        "TRAVEL_READINESS_DB_USER",
+        "TRAVEL_READINESS_DEBUG",
+        "TRAVEL_READINESS_HTTPS",
+        "TRAVEL_READINESS_RELEASE_SHA",
+        "TRAVEL_READINESS_SECRET_KEY",
+        "TRAVEL_READINESS_TLS_CERT_FILE",
+        "TRAVEL_READINESS_TLS_KEY_FILE",
+    }:
+        fail()
+    return environment
+
+
+def read_runtime_values() -> dict[str, str]:
+    payload = sys.stdin.buffer.read(128 * 1024 + 1)
+    if len(payload) > 128 * 1024 or not payload.endswith(b"\0"):
+        fail()
+    fields = payload.split(b"\0")
+    if len(fields) != len(RUNTIME_ENV_NAMES) + 1 or fields[-1] != b"":
+        fail()
+    values: dict[str, str] = {}
+    for name, raw in zip(RUNTIME_ENV_NAMES, fields[:-1], strict=True):
+        if not raw or len(raw) > 16 * 1024 or b"\0" in raw:
+            fail()
+        try:
+            value = raw.decode("utf-8", "strict")
+        except UnicodeDecodeError:
+            fail()
+        if any(character in value for character in ("\r", "\n")):
+            fail()
+        values[name] = value
+    for flag in ("TRAVEL_READINESS_HTTPS", "TRAVEL_READINESS_DEBUG"):
+        if values[flag] not in {"0", "1"}:
+            fail()
+    return values
+
+
+def validate_bind(value: str) -> str:
+    if not value.startswith("127.0.0.1:"):
+        fail()
+    port = value.removeprefix("127.0.0.1:")
+    if (
+        not port
+        or len(port) > 5
+        or not port.isascii()
+        or not port.isdigit()
+        or not 1 <= int(port, 10) <= 65535
+    ):
+        fail()
+    return value
+
+
+def runtime_principal(trusted_owner_uid: int) -> tuple[int, int]:
+    runtime_uid = os.geteuid()
+    runtime_gid = os.getegid()
+    if runtime_uid == 0 or runtime_uid == trusted_owner_uid:
+        fail()
+    return runtime_uid, runtime_gid
+
+
+def isolate_runtime_process() -> None:
+    try:
+        os.setsid()
+        runtime_pid = os.getpid()
+        ready_status = os.fstat(4)
+        if (
+            runtime_pid <= 1
+            or os.getpgrp() != runtime_pid
+            or os.getsid(0) != runtime_pid
+            or not stat.S_ISFIFO(ready_status.st_mode)
+            or ready_status.st_uid != os.geteuid()
+            or stat.S_IMODE(ready_status.st_mode) != 0o600
+        ):
+            fail()
+        ready = f"{runtime_pid}\n".encode("ascii")
+        offset = 0
+        while offset < len(ready):
+            written = os.write(4, ready[offset:])
+            if written <= 0:
+                fail()
+            offset += written
+    except (OSError, ValueError):
+        raise LaunchError("RUNTIME_PROCESS_GROUP_INVALID", 64)
+    except VerificationError:
+        raise LaunchError("RUNTIME_PROCESS_GROUP_INVALID", 64)
+    finally:
+        try:
+            os.close(4)
+        except OSError:
+            pass
+
+
+def validate_tls_material(
+    cert_value: str,
+    key_value: str,
+    *,
+    trusted_owner_uid: int,
+    runtime_uid: int,
+    runtime_gid: int,
+) -> tuple[str, str]:
+    cert, cert_status = canonical_external_file(
+        cert_value,
+        allowed_owner_uids={0, trusted_owner_uid},
+    )
+    key, key_status = canonical_external_file(
+        key_value, allowed_owner_uids={trusted_owner_uid}
+    )
+    if (
+        runtime_uid in {0, trusted_owner_uid}
+        or cert_status.st_uid not in {0, trusted_owner_uid}
+        or cert_status.st_nlink != 1
+        or cert_status.st_mode & 0o022
+        or key_status.st_uid != trusted_owner_uid
+        or key_status.st_gid != runtime_gid
+        or key_status.st_nlink != 1
+        or stat.S_IMODE(key_status.st_mode) != 0o440
+        or not os.access(cert, os.R_OK)
+        or not os.access(key, os.R_OK)
+    ):
+        fail()
+    try:
+        import ssl
+
+        context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+        context.minimum_version = ssl.TLSVersion.TLSv1_2
+        context.load_cert_chain(cert, key)
+    except Exception:
+        fail()
+    return str(cert), str(key)
+
+
+def run_checked_child(
+    arguments: list[str],
+    *,
+    environment: dict[str, str],
+    cwd: Path,
+    timeout: int,
+) -> bool:
+    try:
+        result = subprocess.run(
+            arguments,
+            cwd=cwd,
+            env=environment,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+            check=False,
+            timeout=timeout,
+        )
+    except (OSError, subprocess.SubprocessError):
+        return False
+    return result.returncode == 0
+
+
+def require_dependency_environment_unchanged(
+    source_root: Path,
+    *,
+    trusted_owner_uid: int,
+    bootstrap_python: Path,
+    expected_digest: str,
+) -> None:
+    try:
+        inventory, _by_path = inventory_venv(
+            source_root,
+            trusted_owner_uid=trusted_owner_uid,
+            bootstrap_python=bootstrap_python,
+        )
+        observed_digest = sha256(canonical_json(inventory))
+    except VerificationError:
+        raise LaunchError("DEPENDENCY_ENVIRONMENT_MUTATED", 64)
+    if observed_digest != digest_string(expected_digest):
+        raise LaunchError("DEPENDENCY_ENVIRONMENT_MUTATED", 64)
+
+
+def launch_runtime(
+    *,
+    source_root: Path,
+    release_sha: str,
+    bootstrap_python: Path,
+    bootstrap_digest: str,
+    trusted_owner_uid: int,
+    expected_dependency_environment_sha256: str,
+    tls_cert: str,
+    tls_key: str,
+    bind: str,
+) -> NoReturn:
+    try:
+        runtime_uid, runtime_gid = runtime_principal(trusted_owner_uid)
+    except VerificationError:
+        raise LaunchError("RUNTIME_PRINCIPAL_INVALID", 64)
+    try:
+        values = read_runtime_values()
+    except VerificationError:
+        raise LaunchError("STARTUP_CONFIGURATION_INVALID", 64)
+    try:
+        bind = validate_bind(bind)
+    except VerificationError:
+        raise LaunchError("LOOPBACK_BIND_REQUIRED", 64)
+    try:
+        cert, key = validate_tls_material(
+            tls_cert,
+            tls_key,
+            trusted_owner_uid=trusted_owner_uid,
+            runtime_uid=runtime_uid,
+            runtime_gid=runtime_gid,
+        )
+    except VerificationError:
+        raise LaunchError("TLS_MATERIAL_INVALID", 64)
+    python_path = source_root / ".venv" / "bin" / "python"
+    verify_venv_python(source_root, bootstrap_python, bootstrap_digest)
+    site_packages = (
+        source_root
+        / ".venv"
+        / "lib"
+        / f"python{sys.version_info.major}.{sys.version_info.minor}"
+        / "site-packages"
+    )
+    canonical_file_path(
+        site_packages, directory=True, owner_uid=trusted_owner_uid
+    )
+    public_environment = {
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "PYTHONDONTWRITEBYTECODE": "1",
+        "TZ": "UTC",
+    }
+    runtime_environment = fixed_runtime_environment(
+        values,
+        release_sha=release_sha,
+        tls_cert=cert,
+        tls_key=key,
+        bind=bind,
+    )
+    dependency_probe = r'''
+import sys
+sys.path.insert(0, sys.argv[1])
+from importlib.metadata import version
+expected = {
+    "Django": "5.2.17",
+    "gunicorn": "26.2.0",
+    "psycopg": "3.3.4",
+    "psycopg-binary": "3.3.4",
+}
+if any(version(name) != wanted for name, wanted in expected.items()):
+    raise SystemExit(1)
+import django, gunicorn, psycopg
+'''
+    require_dependency_environment_unchanged(
+        source_root,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap_python=bootstrap_python,
+        expected_digest=expected_dependency_environment_sha256,
+    )
+    dependency_probe_ok = run_checked_child(
+        [
+            str(python_path),
+            "-I",
+            "-S",
+            "-B",
+            "-c",
+            dependency_probe,
+            str(site_packages),
+        ],
+        environment=public_environment,
+        cwd=source_root,
+        timeout=30,
+    )
+    require_dependency_environment_unchanged(
+        source_root,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap_python=bootstrap_python,
+        expected_digest=expected_dependency_environment_sha256,
+    )
+    if not dependency_probe_ok:
+        raise LaunchError("DEPENDENCY_VERSION_MISMATCH", 64)
+    preflight = r'''
+import sys
+sys.path.insert(0, sys.argv[1])
+sys.path.insert(0, sys.argv[2])
+import django
+django.setup()
+from django.core.management import call_command
+if sys.argv[3] == "deploy":
+    call_command("check", deploy=True, fail_level="WARNING", verbosity=0)
+elif sys.argv[3] == "migrations":
+    call_command("migrate", check=True, interactive=False, verbosity=0)
+else:
+    raise SystemExit(2)
+'''
+    deployment_check_ok = run_checked_child(
+        [
+            str(python_path),
+            "-I",
+            "-S",
+            "-B",
+            "-c",
+            preflight,
+            str(site_packages),
+            str(source_root),
+            "deploy",
+        ],
+        environment=runtime_environment,
+        cwd=source_root,
+        timeout=30,
+    )
+    require_dependency_environment_unchanged(
+        source_root,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap_python=bootstrap_python,
+        expected_digest=expected_dependency_environment_sha256,
+    )
+    if not deployment_check_ok:
+        raise LaunchError("DEPLOYMENT_CHECK_FAILED", 78)
+    migration_check_ok = run_checked_child(
+        [
+            str(python_path),
+            "-I",
+            "-S",
+            "-B",
+            "-c",
+            preflight,
+            str(site_packages),
+            str(source_root),
+            "migrations",
+        ],
+        environment=runtime_environment,
+        cwd=source_root,
+        timeout=30,
+    )
+    require_dependency_environment_unchanged(
+        source_root,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap_python=bootstrap_python,
+        expected_digest=expected_dependency_environment_sha256,
+    )
+    if not migration_check_ok:
+        raise LaunchError("MIGRATION_CHECK_FAILED", 78)
+    gunicorn_launcher = r'''
+import sys
+sys.path.insert(0, sys.argv[1])
+sys.path.insert(0, sys.argv[2])
+from gunicorn.app.wsgiapp import run
+sys.argv = ["gunicorn", *sys.argv[3:]]
+run()
+'''
+    try:
+        os.chdir(source_root)
+        os.execve(
+            python_path,
+            [
+                str(python_path),
+                "-I",
+                "-S",
+                "-B",
+                "-c",
+                gunicorn_launcher,
+                str(site_packages),
+                str(source_root),
+                "--chdir",
+                str(source_root),
+                "--config",
+                str(source_root / "runtime" / "gunicorn.conf.py"),
+                "travel_readiness.wsgi:application",
+            ],
+            runtime_environment,
+        )
+    except OSError:
+        raise LaunchError("GUNICORN_START_FAILED", 78)
+
+
+class LaunchError(RuntimeError):
+    def __init__(self, code: str, exit_code: int):
+        super().__init__(code)
+        self.code = code
+        self.exit_code = exit_code
+
+
+def parsed_options(arguments: list[str]) -> tuple[str, dict[str, str]]:
+    if not arguments:
+        fail()
+    mode = arguments[0]
+    expected_by_mode = {
+        "prepare": {
+            "--bootstrap-prefix",
+            "--bootstrap-pristine-tree-sha256",
+            "--bootstrap-python",
+            "--bootstrap-python-sha256",
+            "--bootstrap-tree-spec",
+            "--bootstrap-tree-spec-sha256",
+            "--expected-dependency-environment-sha256",
+            "--expected-manifest-sha256",
+            "--expected-release-sha",
+            "--expected-verifier-sha256",
+            "--runtime-receipt",
+            "--trusted-owner-uid",
+        },
+        "verify": {
+            "--bootstrap-prefix",
+            "--bootstrap-pristine-tree-sha256",
+            "--bootstrap-python",
+            "--bootstrap-python-sha256",
+            "--bootstrap-tree-spec",
+            "--bootstrap-tree-spec-sha256",
+            "--expected-dependency-environment-sha256",
+            "--expected-manifest-sha256",
+            "--expected-release-sha",
+            "--expected-verifier-sha256",
+            "--expected-runtime-receipt-sha256",
+            "--runtime-receipt",
+            "--trusted-owner-uid",
+        },
+        "launch": {
+            "--bind",
+            "--bootstrap-prefix",
+            "--bootstrap-pristine-tree-sha256",
+            "--bootstrap-python",
+            "--bootstrap-python-sha256",
+            "--bootstrap-tree-spec",
+            "--bootstrap-tree-spec-sha256",
+            "--expected-dependency-environment-sha256",
+            "--expected-manifest-sha256",
+            "--expected-release-sha",
+            "--expected-verifier-sha256",
+            "--expected-runtime-receipt-sha256",
+            "--runtime-receipt",
+            "--tls-cert",
+            "--tls-key",
+            "--trusted-owner-uid",
+        },
+    }
+    expected = expected_by_mode.get(mode)
+    if expected is None or len(arguments[1:]) != 2 * len(expected):
+        fail()
+    options: dict[str, str] = {}
+    pairs = arguments[1:]
+    for index in range(0, len(pairs), 2):
+        name, value = pairs[index : index + 2]
+        if name not in expected or name in options or not value:
+            fail()
+        options[name] = value
+    if set(options) != expected:
+        fail()
+    return mode, options
+
+
+def execute(arguments: list[str]) -> tuple[str, str | None]:
+    mode, options = parsed_options(arguments)
+    trusted_owner_uid = owner_uid(options["--trusted-owner-uid"])
+    if mode == "launch":
+        try:
+            runtime_principal(trusted_owner_uid)
+        except VerificationError:
+            raise LaunchError("RUNTIME_PRINCIPAL_INVALID", 64)
+        isolate_runtime_process()
+    expected_release = expected_release_sha(options["--expected-release-sha"])
+    expected_manifest_digest = digest_string(
+        options["--expected-manifest-sha256"]
+    )
+    source_root, release_root, release_sha, manifest_digest, manifest = verify_source(
+        trusted_owner_uid=trusted_owner_uid,
+        expected_release=expected_release,
+        expected_manifest_digest=expected_manifest_digest,
+        expected_verifier_digest=options["--expected-verifier-sha256"],
+    )
+    runtime = exact_dict(manifest["runtime"], {"path", "sha256", "versions"})
+    versions = exact_dict(runtime["versions"], RUNTIME_VERSION_KEYS)
+    bootstrap_python, bootstrap = bootstrap_identity(
+        options["--bootstrap-python"],
+        options["--bootstrap-python-sha256"],
+        options["--bootstrap-prefix"],
+        options["--bootstrap-tree-spec"],
+        options["--bootstrap-tree-spec-sha256"],
+        options["--bootstrap-pristine-tree-sha256"],
+        versions,
+        trusted_owner_uid=trusted_owner_uid,
+        require_root_tree=mode == "launch",
+    )
+    expected_receipt = runtime_receipt_value(
+        source_root=source_root,
+        release_sha=release_sha,
+        manifest_digest=manifest_digest,
+        manifest=manifest,
+        trusted_owner_uid=trusted_owner_uid,
+        bootstrap=bootstrap,
+        bootstrap_python=bootstrap_python,
+        expected_dependency_environment_sha256=options[
+            "--expected-dependency-environment-sha256"
+        ],
+        expected_verifier_sha256=options["--expected-verifier-sha256"],
+    )
+    runtime_receipt = receipt_path(
+        options["--runtime-receipt"],
+        release_root=release_root,
+        trusted_owner_uid=trusted_owner_uid,
+        must_exist=mode != "prepare",
+    )
+    if mode == "prepare":
+        data = canonical_json(expected_receipt)
+        write_receipt(
+            runtime_receipt, data, trusted_owner_uid=trusted_owner_uid
+        )
+        return mode, sha256(data)
+    verify_receipt(
+        runtime_receipt,
+        options["--expected-runtime-receipt-sha256"],
+        expected_receipt,
+        trusted_owner_uid=trusted_owner_uid,
+    )
+    if mode == "verify":
+        return mode, release_sha
+    launch_runtime(
+        source_root=source_root,
+        release_sha=release_sha,
+        bootstrap_python=bootstrap_python,
+        bootstrap_digest=str(bootstrap["python_sha256"]),
+        trusted_owner_uid=trusted_owner_uid,
+        expected_dependency_environment_sha256=options[
+            "--expected-dependency-environment-sha256"
+        ],
+        tls_cert=options["--tls-cert"],
+        tls_key=options["--tls-key"],
+        bind=options["--bind"],
+    )
+
+
+def main() -> int:
+    try:
+        os.fstat(3)
+    except OSError:
+        pass
+    else:
+        try:
+            os.dup2(3, 2)
+        finally:
+            os.close(3)
+    mode = sys.argv[1] if len(sys.argv) > 1 else "verify"
+    if mode == "launch":
+        os.closerange(5, 1_048_576)
+    else:
+        os.closerange(3, 1_048_576)
+    try:
+        completed_mode, value = execute(sys.argv[1:])
+    except LaunchError as error:
+        print(f"startup_error={error.code}", file=sys.stderr)
+        return error.exit_code
+    except BaseException:
+        if mode == "launch":
+            print("startup_error=RELEASE_RUNTIME_INVALID", file=sys.stderr)
+            return 64
+        if mode == "prepare":
+            print("runtime_receipt=invalid", file=sys.stderr)
+            return 1
+        print("release_runtime=invalid", file=sys.stderr)
+        return 1
+    if completed_mode == "prepare":
+        print(f"runtime_receipt_sha256={value}")
+    else:
+        print(value)
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
