## `build(release): create deterministic artifact`

diff --git a/operations/tests/test_release_artifact.py b/operations/tests/test_release_artifact.py
new file mode 100644
index 0000000..efed315
--- /dev/null
+++ b/operations/tests/test_release_artifact.py
@@ -0,0 +1,762 @@
+from __future__ import annotations
+
+import hashlib
+import importlib.metadata
+import json
+from pathlib import Path
+import shutil
+import shlex
+import stat
+import subprocess
+import sys
+import tarfile
+import tempfile
+import textwrap
+import unittest
+
+
+class ReleaseArtifactTests(unittest.TestCase):
+    @staticmethod
+    def python_version() -> str:
+        return ".".join(str(part) for part in sys.version_info[:3])
+
+    @staticmethod
+    def canonical(value: object) -> bytes:
+        return (
+            json.dumps(
+                value,
+                ensure_ascii=False,
+                allow_nan=False,
+                sort_keys=True,
+                separators=(",", ":"),
+            )
+            + "\n"
+        ).encode("utf-8")
+
+    def setUp(self):
+        self.temporary = tempfile.TemporaryDirectory()
+        self.repository = Path(self.temporary.name) / "candidate"
+        self.repository.mkdir()
+        self.source_builder = (
+            Path(__file__).resolve().parents[2] / "scripts" / "build-release"
+        )
+        self.secret_marker = "synthetic-secret-must-not-cross-build-boundary"
+        self._create_repository()
+
+    def tearDown(self):
+        self.temporary.cleanup()
+
+    def _write(self, relative: str, content: str, *, executable: bool = False):
+        path = self.repository / relative
+        path.parent.mkdir(parents=True, exist_ok=True)
+        path.write_text(textwrap.dedent(content).lstrip(), encoding="utf-8")
+        if executable:
+            path.chmod(0o755)
+        return path
+
+    def _git(self, *arguments: str) -> str:
+        result = subprocess.run(
+            ["git", *arguments],
+            cwd=self.repository,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        return result.stdout.strip()
+
+    def _create_repository(self):
+        self._git("init", "--initial-branch=main")
+        self._git("config", "user.name", "Release Test")
+        self._git("config", "user.email", "release-test@example.invalid")
+        self._write(".gitignore", "/output/\n")
+        self._write(
+            "runtime/versions.toml",
+            f"""
+            [runtime]
+            python = "{self.python_version()}"
+            django = "{importlib.metadata.version('Django')}"
+            postgresql = "18.6"
+            uv = "0.12.6"
+            psycopg = "{importlib.metadata.version('psycopg')}"
+            psycopg_distribution = "binary-wheel"
+            gunicorn = "{importlib.metadata.version('gunicorn')}"
+            """,
+        )
+        self._write(
+            "uv.lock",
+            f"""
+            version = 1
+            revision = 3
+            requires-python = "=={self.python_version()}"
+
+            [[package]]
+            name = "django"
+            version = "{importlib.metadata.version('Django')}"
+            """,
+        )
+        self._write(
+            "travel_readiness/settings.py",
+            """
+            import os
+            from pathlib import Path
+            import django
+
+            if os.environ.get("MOFA_TRAVEL_ALARM_SERVICE_KEY"):
+                raise RuntimeError("credential crossed build boundary")
+            if os.environ.get("UNRELATED_RELEASE_PARENT_MARKER"):
+                raise RuntimeError("parent environment crossed build boundary")
+            BASE_DIR = Path(__file__).resolve().parent.parent
+            for ancestor in (BASE_DIR, *BASE_DIR.parents):
+                if (ancestor / ".env.local").exists():
+                    raise RuntimeError(
+                        "ignored environment file crossed source ancestry boundary"
+                    )
+            SECRET_KEY = "credential-free-build-test"
+            INSTALLED_APPS = ["sample", "django.contrib.staticfiles"]
+            STATIC_URL = "/static/"
+            STATIC_ROOT = BASE_DIR / "staticfiles"
+            STATICFILES_DIRS = [
+                Path(django.__file__).resolve().parent / "contrib" / "admin" / "static"
+            ]
+            """,
+        )
+        self._write("travel_readiness/__init__.py", "")
+        self._write("sample/__init__.py", "")
+        self._write("sample/migrations/__init__.py", "")
+        self.migration = self._write(
+            "sample/migrations/0001_initial.py",
+            """
+            from django.db import migrations
+
+
+            class Migration(migrations.Migration):
+                initial = True
+                dependencies = []
+                operations = []
+            """,
+        )
+        self.static = self._write(
+            "sample/static/sample/site.css", "body { color: #123456; }\n"
+        )
+        builder = self.repository / "scripts" / "build-release"
+        builder.parent.mkdir(parents=True)
+        shutil.copyfile(self.source_builder, builder)
+        builder.chmod(0o755)
+        self.uv = self._write(
+            "tools/uv",
+            """
+            #!/bin/sh
+            [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 97
+            [ -z "${UNRELATED_RELEASE_PARENT_MARKER:-}" ] || exit 97
+            [ "${PATH:-}" = "/usr/bin:/bin" ] || exit 97
+            if [ "${1:-}" = "--version" ]; then
+                printf '%s\\n' 'uv 0.12.6 (fixture build metadata)'
+                exit 0
+            fi
+            if [ "${1:-}" = "lock" ]; then
+                [ "${2:-}" = "--check" ] || exit 98
+                [ "${3:-}" = "--offline" ] || exit 98
+                exit 0
+            fi
+            [ "${1:-}" = "pip" ] || exit 98
+            [ "${2:-}" = "check" ] || exit 98
+            exit 0
+            """,
+            executable=True,
+        )
+        self._git("add", ".")
+        self._git("commit", "-m", "test fixture")
+        self.release_sha = self._git("rev-parse", "HEAD")
+
+    def _build(self, output_name: str, *, uv_bin: str | None = None):
+        environment = {
+            "LANG": "C",
+            "LC_ALL": "C",
+            "MOFA_TRAVEL_ALARM_SERVICE_KEY": self.secret_marker,
+            "PATH": "/private/untrusted-parent-path",
+            "TZ": "UTC",
+            "UNRELATED_RELEASE_PARENT_MARKER": self.secret_marker,
+        }
+        return subprocess.run(
+            [
+                sys.executable,
+                str(self.repository / "scripts" / "build-release"),
+                "--output-dir",
+                f"output/{output_name}",
+                "--uv-bin",
+                str(self.uv) if uv_bin is None else uv_bin,
+            ],
+            cwd=self.repository.parent,
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=30,
+        )
+
+    def test_two_builds_are_byte_identical_and_fully_manifested(self):
+        first = self._build("first")
+        second = self._build("second")
+        self.assertEqual(first.returncode, 0, first.stderr)
+        self.assertEqual(second.returncode, 0, second.stderr)
+        self.assertNotIn(self.secret_marker, first.stdout + first.stderr)
+        self.assertNotIn(self.secret_marker, second.stdout + second.stderr)
+
+        first_dir = self.repository / "output" / "first"
+        second_dir = self.repository / "output" / "second"
+        expected_files = {
+            "artifact.sha256",
+            "release-manifest.json",
+            "travel-readiness-release.tar",
+        }
+        self.assertEqual({path.name for path in first_dir.iterdir()}, expected_files)
+        self.assertEqual({path.name for path in second_dir.iterdir()}, expected_files)
+        self.assertEqual(stat.S_IMODE(first_dir.stat().st_mode), 0o500)
+        for name in expected_files:
+            self.assertEqual(stat.S_IMODE((first_dir / name).stat().st_mode), 0o400)
+            self.assertEqual(
+                (first_dir / name).read_bytes(), (second_dir / name).read_bytes()
+            )
+
+        archive = first_dir / "travel-readiness-release.tar"
+        archive_bytes = archive.read_bytes()
+        artifact_digest = hashlib.sha256(archive_bytes).hexdigest()
+        self.assertEqual(
+            (first_dir / "artifact.sha256").read_text(encoding="ascii"),
+            f"{artifact_digest}  travel-readiness-release.tar\n",
+        )
+
+        manifest_bytes = (first_dir / "release-manifest.json").read_bytes()
+        manifest = json.loads(manifest_bytes)
+        self.assertEqual(
+            set(manifest),
+            {
+                "archive",
+                "dependencies",
+                "git_object_format",
+                "migrations",
+                "release_sha",
+                "runtime",
+                "schema_version",
+                "source",
+                "static",
+            },
+        )
+        self.assertEqual(
+            set(manifest["archive"]),
+            {
+                "compression",
+                "format",
+                "gid",
+                "group",
+                "mtime",
+                "regular_modes",
+                "uid",
+                "user",
+            },
+        )
+        self.assertEqual(
+            set(manifest["dependencies"]),
+            {"lock_path", "lock_sha256", "package_set_sha256", "packages"},
+        )
+        self.assertEqual(
+            set(manifest["migrations"]), {"leaf_set_sha256", "leaves"}
+        )
+        self.assertEqual(
+            set(manifest["runtime"]), {"path", "sha256", "versions"}
+        )
+        self.assertEqual(
+            set(manifest["static"]),
+            {
+                "collected",
+                "collected_set_sha256",
+                "origins",
+                "tracked",
+                "tracked_set_sha256",
+            },
+        )
+        self.assertEqual(
+            set(manifest["source"]), {"tracked", "tracked_set_sha256"}
+        )
+        self.assertEqual(manifest["schema_version"], 1)
+        self.assertEqual(manifest["release_sha"], self.release_sha)
+        self.assertEqual(manifest["git_object_format"], "sha1")
+        self.assertEqual(manifest["archive"]["compression"], "none")
+        self.assertNotIn("timestamp", json.dumps(manifest).lower())
+        self.assertNotIn("built_at", manifest)
+        self.assertEqual(
+            manifest["dependencies"]["lock_sha256"],
+            hashlib.sha256((self.repository / "uv.lock").read_bytes()).hexdigest(),
+        )
+        self.assertEqual(
+            manifest["dependencies"]["package_set_sha256"],
+            hashlib.sha256(
+                self.canonical(manifest["dependencies"]["packages"])
+            ).hexdigest(),
+        )
+        self.assertEqual(
+            manifest["runtime"]["sha256"],
+            hashlib.sha256(
+                (self.repository / "runtime/versions.toml").read_bytes()
+            ).hexdigest(),
+        )
+        self.assertEqual(
+            manifest["runtime"]["versions"]["python"],
+            self.python_version(),
+        )
+        self.assertEqual(
+            set(manifest["runtime"]["versions"]),
+            {
+                "django",
+                "gunicorn",
+                "postgresql",
+                "psycopg",
+                "psycopg_distribution",
+                "python",
+                "uv",
+            },
+        )
+        self.assertEqual(
+            {item["path"] for item in manifest["source"]["tracked"]},
+            set(self._git("ls-files").splitlines()),
+        )
+        for item in manifest["source"]["tracked"]:
+            self.assertEqual(set(item), {"git_mode", "path", "sha256"})
+        for item in manifest["dependencies"]["packages"]:
+            self.assertEqual(set(item), {"name", "version"})
+        for item in manifest["migrations"]["leaves"]:
+            self.assertEqual(
+                set(item),
+                {"app", "module", "name", "origin", "path", "sha256"},
+            )
+        for item in manifest["static"]["collected"]:
+            self.assertEqual(set(item), {"path", "sha256"})
+        for item in manifest["static"]["origins"]:
+            self.assertEqual(
+                set(item), {"collected_path", "origin", "path", "sha256"}
+            )
+        for item in manifest["static"]["tracked"]:
+            self.assertEqual(
+                set(item), {"collected_path", "path", "sha256"}
+            )
+        leaf = next(
+            leaf for leaf in manifest["migrations"]["leaves"] if leaf["app"] == "sample"
+        )
+        self.assertEqual(leaf["name"], "0001_initial")
+        self.assertEqual(
+            leaf["sha256"], hashlib.sha256(self.migration.read_bytes()).hexdigest()
+        )
+        self.assertEqual(
+            manifest["migrations"]["leaf_set_sha256"],
+            hashlib.sha256(
+                self.canonical(manifest["migrations"]["leaves"])
+            ).hexdigest(),
+        )
+        tracked_static = {
+            item["path"]: item["sha256"]
+            for item in manifest["static"]["tracked"]
+        }
+        collected_static = {
+            item["path"]: item["sha256"] for item in manifest["static"]["collected"]
+        }
+        static_digest = hashlib.sha256(self.static.read_bytes()).hexdigest()
+        self.assertEqual(tracked_static["sample/static/sample/site.css"], static_digest)
+        self.assertEqual(collected_static["sample/site.css"], static_digest)
+        django_origin = next(
+            item
+            for item in manifest["static"]["origins"]
+            if item["origin"] == "django"
+        )
+        self.assertTrue(
+            django_origin["path"].startswith("django/contrib/admin/static/")
+        )
+        self.assertEqual(
+            django_origin["sha256"],
+            collected_static[django_origin["collected_path"]],
+        )
+        self.assertEqual(
+            manifest["static"]["collected_set_sha256"],
+            hashlib.sha256(
+                self.canonical(manifest["static"]["collected"])
+            ).hexdigest(),
+        )
+        self.assertEqual(
+            manifest["static"]["tracked_set_sha256"],
+            hashlib.sha256(
+                self.canonical(manifest["static"]["tracked"])
+            ).hexdigest(),
+        )
+        self.assertEqual(
+            manifest["source"]["tracked_set_sha256"],
+            hashlib.sha256(
+                self.canonical(manifest["source"]["tracked"])
+            ).hexdigest(),
+        )
+
+        with tarfile.open(archive, "r:") as release:
+            members = release.getmembers()
+            expected_archive_files = {
+                "release/manifest.json",
+                *(
+                    f"release/source/{item['path']}"
+                    for item in manifest["source"]["tracked"]
+                ),
+                *(
+                    f"release/static/{item['path']}"
+                    for item in manifest["static"]["collected"]
+                ),
+            }
+            expected_directories = {"release"}
+            for name in expected_archive_files:
+                parent = Path(name).parent
+                while parent.as_posix() not in {"", "."}:
+                    expected_directories.add(parent.as_posix())
+                    parent = parent.parent
+            expected_member_names = sorted(
+                expected_directories,
+                key=lambda value: (value.count("/"), value),
+            ) + sorted(expected_archive_files)
+            self.assertEqual(
+                [member.name for member in members], expected_member_names
+            )
+            self.assertEqual(
+                release.extractfile("release/manifest.json").read(), manifest_bytes
+            )
+            self.assertEqual(
+                release.extractfile("release/static/sample/site.css").read(),
+                self.static.read_bytes(),
+            )
+            for source_entry in manifest["source"]["tracked"]:
+                source_member = release.extractfile(
+                    f"release/source/{source_entry['path']}"
+                )
+                self.assertIsNotNone(source_member)
+                self.assertEqual(
+                    hashlib.sha256(source_member.read()).hexdigest(),
+                    source_entry["sha256"],
+                )
+            for static_entry in manifest["static"]["collected"]:
+                static_member = release.extractfile(
+                    f"release/static/{static_entry['path']}"
+                )
+                self.assertIsNotNone(static_member)
+                self.assertEqual(
+                    hashlib.sha256(static_member.read()).hexdigest(),
+                    static_entry["sha256"],
+                )
+            for member in members:
+                self.assertEqual(member.uid, 0)
+                self.assertEqual(member.gid, 0)
+                self.assertEqual(member.uname, "root")
+                self.assertEqual(member.gname, "root")
+                self.assertEqual(member.mtime, 0)
+                if member.isdir():
+                    self.assertEqual(member.mode, 0o755)
+                    continue
+                self.assertTrue(member.isfile())
+                expected_mode = 0o644
+                source_prefix = "release/source/"
+                if member.name.startswith(source_prefix):
+                    source_path = member.name.removeprefix(source_prefix)
+                    source_entry = next(
+                        item
+                        for item in manifest["source"]["tracked"]
+                        if item["path"] == source_path
+                    )
+                    expected_mode = (
+                        0o755 if source_entry["git_mode"] == "100755" else 0o644
+                    )
+                self.assertEqual(member.mode, expected_mode)
+                self.assertNotIn(
+                    self.secret_marker.encode(), release.extractfile(member).read()
+                )
+        self.assertEqual(self._git("status", "--porcelain=v1"), "")
+
+    def test_dirty_repository_is_rejected_before_output_creation(self):
+        dirty_marker = "dirty-content-must-not-be-archived"
+        self.static.write_text(dirty_marker, encoding="utf-8")
+
+        result = self._build("dirty")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("the Git worktree must be clean", result.stderr)
+        self.assertNotIn(dirty_marker, result.stdout + result.stderr)
+        self.assertFalse((self.repository / "output" / "dirty").exists())
+
+    def test_nonignored_output_boundary_is_rejected(self):
+        (self.repository / ".gitignore").write_text(
+            "\n".join(
+                (
+                    "/output/.release-boundary-probe",
+                    "/output/.release-boundary-probe/",
+                    "/output/not-ignored/",
+                    "/output/.not-ignored.incomplete-*/",
+                    "",
+                )
+            ),
+            encoding="utf-8",
+        )
+        self._git("add", ".gitignore")
+        self._git("commit", "-m", "ignore only anticipated release paths")
+
+        result = self._build("not-ignored")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "the complete output boundary must be ignored by Git", result.stderr
+        )
+        self.assertFalse((self.repository / "output" / "not-ignored").exists())
+
+    def test_relative_uv_binary_is_rejected_before_output_creation(self):
+        result = self._build("relative-uv", uv_bin="tools/uv")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "--uv-bin must identify an absolute executable file", result.stderr
+        )
+        self.assertFalse((self.repository / "output" / "relative-uv").exists())
+
+    def test_tracked_environment_file_is_rejected_without_reading_it(self):
+        unread_marker = "noncredential-environment-file-marker"
+        environment_file = self._write(".env.local", unread_marker)
+        self._git("add", ".env.local")
+        self._git("commit", "-m", "add forbidden empty environment file")
+
+        result = self._build("forbidden-environment")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "a credential-bearing environment file is tracked", result.stderr
+        )
+        self.assertNotIn(unread_marker, result.stdout + result.stderr)
+        self.assertEqual(
+            environment_file.read_text(encoding="utf-8"), unread_marker
+        )
+        self.assertFalse(
+            (self.repository / "output" / "forbidden-environment").exists()
+        )
+
+    def test_ignored_migration_static_and_environment_do_not_enter_build(self):
+        (self.repository / ".gitignore").write_text(
+            "\n".join(
+                (
+                    "/output/",
+                    "/.env.local",
+                    "/sample/migrations/0002_ignored.py",
+                    "/sample/static/sample/ignored.css",
+                    "",
+                )
+            ),
+            encoding="utf-8",
+        )
+        self._git("add", ".gitignore")
+        self._git("commit", "-m", "ignore adversarial build inputs")
+        ignored_marker = "ignored-content-must-not-enter-release"
+        self._write(".env.local", "")
+        self._write(
+            "sample/migrations/0002_ignored.py",
+            f"""
+            from django.db import migrations
+
+
+            MARKER = "{ignored_marker}"
+
+
+            class Migration(migrations.Migration):
+                dependencies = [("sample", "0001_initial")]
+                operations = []
+            """,
+        )
+        self._write(
+            "sample/static/sample/ignored.css", f"/* {ignored_marker} */\n"
+        )
+
+        result = self._build("isolated")
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        output = self.repository / "output" / "isolated"
+        manifest_bytes = (output / "release-manifest.json").read_bytes()
+        archive_bytes = (output / "travel-readiness-release.tar").read_bytes()
+        self.assertNotIn(ignored_marker.encode(), manifest_bytes)
+        self.assertNotIn(ignored_marker.encode(), archive_bytes)
+        manifest = json.loads(manifest_bytes)
+        self.assertNotIn(
+            "0002_ignored",
+            {leaf["name"] for leaf in manifest["migrations"]["leaves"]},
+        )
+        self.assertNotIn(
+            "sample/ignored.css",
+            {item["path"] for item in manifest["static"]["collected"]},
+        )
+        self.assertEqual(self._git("status", "--porcelain=v1"), "")
+
+    def test_repository_change_during_build_prevents_publication(self):
+        mutation_marker = "concurrent-change-must-abort-release"
+        self.uv.write_text(
+            textwrap.dedent(
+                f"""
+                #!/bin/sh
+                if [ "${{1:-}}" = "--version" ]; then
+                    printf '%s\\n' 'uv 0.12.6 (mutation fixture)'
+                    exit 0
+                fi
+                if [ "${{1:-}}" = "lock" ]; then
+                    printf '%s\\n' '{mutation_marker}' >> '{self.static}'
+                    exit 0
+                fi
+                [ "${{1:-}}" = "pip" ] || exit 98
+                exit 0
+                """
+            ).lstrip(),
+            encoding="utf-8",
+        )
+        self.uv.chmod(0o755)
+        self._git("add", "tools/uv")
+        self._git("commit", "-m", "mutate live repository during build")
+
+        result = self._build("concurrent-change")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "the Git worktree changed during the release build", result.stderr
+        )
+        self.assertIn(mutation_marker, self.static.read_text(encoding="utf-8"))
+        self.assertFalse(
+            (self.repository / "output" / "concurrent-change").exists()
+        )
+
+    def test_assume_unchanged_worktree_difference_is_rejected(self):
+        hidden_marker = "assume-unchanged-content-must-abort-release"
+        self.static.write_text(hidden_marker, encoding="utf-8")
+        self._git(
+            "update-index",
+            "--assume-unchanged",
+            "sample/static/sample/site.css",
+        )
+        self.assertEqual(self._git("status", "--porcelain=v1"), "")
+
+        result = self._build("assume-unchanged")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "the Git worktree does not exactly match HEAD", result.stderr
+        )
+        self.assertNotIn(hidden_marker, result.stdout + result.stderr)
+        self.assertFalse(
+            (self.repository / "output" / "assume-unchanged").exists()
+        )
+
+    def test_head_change_during_build_prevents_publication(self):
+        repository_argument = shlex.quote(str(self.repository))
+        head_mutation_command = (
+            f"/usr/bin/git -C {repository_argument} commit "
+            "--allow-empty --no-gpg-sign -m concurrent-head-change"
+        )
+        self.uv.write_text(
+            textwrap.dedent(
+                f"""
+                #!/bin/sh
+                if [ "${{1:-}}" = "--version" ]; then
+                    printf '%s\\n' 'uv 0.12.6 (HEAD mutation fixture)'
+                    exit 0
+                fi
+                if [ "${{1:-}}" = "lock" ]; then
+                    {head_mutation_command} >/dev/null 2>&1
+                    exit $?
+                fi
+                [ "${{1:-}}" = "pip" ] || exit 98
+                exit 0
+                """
+            ).lstrip(),
+            encoding="utf-8",
+        )
+        self.uv.chmod(0o755)
+        self._git("add", "tools/uv")
+        self._git("commit", "-m", "mutate repository HEAD during build")
+
+        result = self._build("concurrent-head-change")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("HEAD changed during the release build", result.stderr)
+        self.assertFalse(
+            (self.repository / "output" / "concurrent-head-change").exists()
+        )
+
+    def test_output_parent_symlink_swap_during_uv_prevents_publication(self):
+        output_argument = shlex.quote(str(self.repository / "output"))
+        moved_output = self.repository.parent / "moved-output"
+        moved_output_argument = shlex.quote(str(moved_output))
+        self.uv.write_text(
+            textwrap.dedent(
+                f"""
+                #!/bin/sh
+                if [ "${{1:-}}" = "--version" ]; then
+                    printf '%s\\n' 'uv 0.12.6 (output mutation fixture)'
+                    exit 0
+                fi
+                if [ "${{1:-}}" = "lock" ]; then
+                    /bin/mv {output_argument} {moved_output_argument} || exit $?
+                    /bin/ln -s {moved_output_argument} {output_argument} || exit $?
+                    exit 0
+                fi
+                [ "${{1:-}}" = "pip" ] || exit 98
+                exit 0
+                """
+            ).lstrip(),
+            encoding="utf-8",
+        )
+        self.uv.chmod(0o755)
+        self._git("add", "tools/uv")
+        self._git("commit", "-m", "swap the release output parent during build")
+
+        result = self._build("symlink-swap")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "the output directory may not traverse a symlink", result.stderr
+        )
+        self.assertFalse((moved_output / "symlink-swap").exists())
+
+    def test_repository_root_symlink_swap_during_uv_prevents_publication(self):
+        repository_argument = shlex.quote(str(self.repository))
+        moved_repository = self.repository.parent / "moved-candidate"
+        moved_repository_argument = shlex.quote(str(moved_repository))
+        move_command = f"/bin/mv {repository_argument} {moved_repository_argument}"
+        link_command = f"/bin/ln -s {moved_repository_argument} {repository_argument}"
+        self.uv.write_text(
+            textwrap.dedent(
+                f"""
+                #!/bin/sh
+                if [ "${{1:-}}" = "--version" ]; then
+                    printf '%s\\n' 'uv 0.12.6 (root mutation fixture)'
+                    exit 0
+                fi
+                if [ "${{1:-}}" = "lock" ]; then
+                    {move_command} || exit $?
+                    {link_command} || exit $?
+                    exit 0
+                fi
+                [ "${{1:-}}" = "pip" ] || exit 98
+                exit 0
+                """
+            ).lstrip(),
+            encoding="utf-8",
+        )
+        self.uv.chmod(0o755)
+        self._git("add", "tools/uv")
+        self._git("commit", "-m", "swap the repository root during build")
+
+        result = self._build("root-symlink-swap")
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn(
+            "the repository root changed during the release build", result.stderr
+        )
+        self.assertFalse(
+            (moved_repository / "output" / "root-symlink-swap").exists()
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/build-release b/scripts/build-release
new file mode 100755
index 0000000..3707432
--- /dev/null
+++ b/scripts/build-release
@@ -0,0 +1,1006 @@
+#!/usr/bin/env python3
+"""Build a credential-free, byte-reproducible local release archive."""
+
+from __future__ import annotations
+
+import argparse
+import hashlib
+import io
+import importlib.metadata
+import json
+import os
+from pathlib import Path, PurePosixPath
+import shutil
+import stat
+import subprocess
+import sys
+import tarfile
+import tempfile
+import tomllib
+
+
+SCHEMA_VERSION = 1
+ARCHIVE_BASENAME = "travel-readiness-release.tar"
+MANIFEST_BASENAME = "release-manifest.json"
+DIGEST_BASENAME = "artifact.sha256"
+EXPECTED_GIT_MODES = {"100644", "100755"}
+GIT_BINARY = "/usr/bin/git"
+
+
+class BuildError(RuntimeError):
+    pass
+
+
+def fail(message: str) -> None:
+    raise BuildError(message)
+
+
+def run(
+    arguments: list[str],
+    *,
+    cwd: Path,
+    environment: dict[str, str] | None = None,
+) -> bytes:
+    if environment is None:
+        environment = nonsecret_environment()
+    result = subprocess.run(
+        arguments,
+        cwd=cwd,
+        env=environment,
+        capture_output=True,
+        check=False,
+        timeout=60,
+    )
+    if result.returncode != 0:
+        fail("a release prerequisite command failed")
+    return result.stdout
+
+
+def sha256(data: bytes) -> str:
+    return hashlib.sha256(data).hexdigest()
+
+
+def nonsecret_environment() -> dict[str, str]:
+    return {
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "TZ": "UTC",
+    }
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
+def runtime_configuration(data: bytes) -> dict[str, object]:
+    configuration = tomllib.loads(data.decode("utf-8"))
+    runtime = configuration.get("runtime")
+    if not isinstance(runtime, dict) or any(
+        not isinstance(key, str) or not isinstance(value, str)
+        for key, value in runtime.items()
+    ):
+        fail("runtime/versions.toml has an invalid runtime table")
+    required = {
+        "python",
+        "django",
+        "postgresql",
+        "uv",
+        "psycopg",
+        "psycopg_distribution",
+        "gunicorn",
+    }
+    if set(runtime) != required:
+        fail("runtime/versions.toml has an unexpected runtime version set")
+    if runtime["psycopg_distribution"] != "binary-wheel":
+        fail("runtime/versions.toml has an unsupported psycopg distribution")
+    return runtime
+
+
+def maybe_reexec_with_pinned_python(root: Path, runtime: dict[str, object]) -> None:
+    expected = str(runtime["python"])
+    actual = ".".join(str(part) for part in sys.version_info[:3])
+    if actual == expected:
+        return
+    candidate = root / ".venv" / "bin" / "python"
+    if not candidate.is_file() or not candidate.resolve().is_file():
+        fail("the pinned Python runtime is unavailable")
+    version = run(
+        [str(candidate), "--version"],
+        cwd=root,
+        environment=credential_free_environment(root),
+    ).decode("ascii", "strict").strip()
+    if version != f"Python {expected}":
+        fail("the local virtual environment does not use the pinned Python")
+    os.execve(
+        candidate,
+        [str(candidate), str(Path(__file__).resolve()), *sys.argv[1:]],
+        credential_free_environment(root),
+    )
+
+
+def credential_free_environment(root: Path) -> dict[str, str]:
+    environment = nonsecret_environment()
+    environment.update(
+        {
+            "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "PYTHONHASHSEED": "0",
+            "PYTHONPATH": str(root),
+            "SOURCE_DATE_EPOCH": "0",
+            "TRAVEL_READINESS_BUILD": "1",
+        }
+    )
+    return environment
+
+
+def verify_installed_runtime(runtime: dict[str, object]) -> None:
+    expected = {
+        "Django": str(runtime["django"]),
+        "gunicorn": str(runtime["gunicorn"]),
+        "psycopg": str(runtime["psycopg"]),
+        "psycopg-binary": str(runtime["psycopg"]),
+    }
+    for distribution, version in expected.items():
+        try:
+            installed = importlib.metadata.version(distribution)
+        except importlib.metadata.PackageNotFoundError:
+            fail("a pinned runtime distribution is unavailable")
+        if installed != version:
+            fail("an installed runtime distribution does not match the pin")
+
+
+def repository_root() -> Path:
+    script_root = Path(__file__).resolve().parent.parent
+    if not Path(GIT_BINARY).is_file():
+        fail("the fixed Git executable is unavailable")
+    probe = subprocess.run(
+        [GIT_BINARY, "rev-parse", "--show-toplevel"],
+        cwd=script_root,
+        env=nonsecret_environment(),
+        capture_output=True,
+        check=False,
+        timeout=10,
+    )
+    if probe.returncode != 0:
+        fail("the release builder must run inside a Git repository")
+    try:
+        root = Path(probe.stdout.decode("utf-8", "strict").strip()).resolve()
+    except UnicodeDecodeError:
+        fail("the repository path must be UTF-8")
+    expected_script = root / "scripts" / "build-release"
+    if script_root != root or Path(__file__).resolve() != expected_script:
+        fail("the release builder must be the tracked repository script")
+    return root
+
+
+def repository_identity(root: Path) -> tuple[int, int]:
+    try:
+        root_status = root.lstat()
+        resolved = root.resolve(strict=True)
+    except (OSError, RuntimeError):
+        fail("the repository root identity is unavailable")
+    if (
+        stat.S_ISLNK(root_status.st_mode)
+        or not stat.S_ISDIR(root_status.st_mode)
+        or resolved != root
+    ):
+        fail("the repository root identity is unsafe")
+    return root_status.st_dev, root_status.st_ino
+
+
+def verify_repository_identity(root: Path, expected: tuple[int, int]) -> None:
+    try:
+        actual = repository_identity(root)
+    except BuildError:
+        fail("the repository root changed during the release build")
+    if actual != expected:
+        fail("the repository root changed during the release build")
+
+
+def git_snapshot(
+    root: Path,
+) -> tuple[str, str, list[dict[str, object]], dict[str, bytes]]:
+    if run(
+        [GIT_BINARY, "status", "--porcelain=v1", "--untracked-files=all", "-z"],
+        cwd=root,
+    ):
+        fail("the Git worktree must be clean")
+    release_sha = run(
+        [GIT_BINARY, "rev-parse", "--verify", "HEAD^{commit}"], cwd=root
+    ).decode("ascii", "strict").strip()
+    object_format = run(
+        [GIT_BINARY, "rev-parse", "--show-object-format"], cwd=root
+    ).decode("ascii", "strict").strip()
+    expected_sha_length = {"sha1": 40, "sha256": 64}.get(object_format)
+    if (
+        expected_sha_length is None
+        or len(release_sha) != expected_sha_length
+        or any(character not in "0123456789abcdef" for character in release_sha)
+    ):
+        fail("HEAD is not a full lowercase commit object ID")
+
+    tracked: list[dict[str, object]] = []
+    blobs: dict[str, bytes] = {}
+    records = run(
+        [GIT_BINARY, "ls-tree", "-r", "-z", "--full-tree", release_sha],
+        cwd=root,
+    ).split(b"\0")
+    for record in records:
+        if not record:
+            continue
+        metadata, separator, path_bytes = record.partition(b"\t")
+        if not separator:
+            fail("Git returned an invalid tracked-file record")
+        fields = metadata.decode("ascii", "strict").split()
+        if (
+            len(fields) != 3
+            or fields[1] != "blob"
+            or fields[0] not in EXPECTED_GIT_MODES
+        ):
+            fail("the release source contains an unsupported Git entry")
+        try:
+            path = path_bytes.decode("utf-8", "strict")
+        except UnicodeDecodeError:
+            fail("tracked release paths must be UTF-8")
+        pure_path = PurePosixPath(path)
+        if pure_path.is_absolute() or ".." in pure_path.parts or not pure_path.parts:
+            fail("Git returned an unsafe tracked path")
+        basename = pure_path.name
+        if basename == ".env" or (
+            basename.startswith(".env.") and basename != ".env.example"
+        ):
+            fail("a credential-bearing environment file is tracked")
+        blob = run([GIT_BINARY, "cat-file", "blob", fields[2]], cwd=root)
+        blobs[path] = blob
+        tracked.append(
+            {
+                "git_mode": fields[0],
+                "path": path,
+                "sha256": sha256(blob),
+            }
+        )
+    tracked.sort(key=lambda item: str(item["path"]))
+    if not tracked:
+        fail("the release source contains no tracked files")
+    builder = blobs.get("scripts/build-release")
+    if builder is None or Path(__file__).resolve().read_bytes() != builder:
+        fail("the running release builder does not match HEAD")
+    return release_sha, object_format, tracked, blobs
+
+
+def verify_worktree_matches_snapshot(
+    root: Path,
+    tracked: list[dict[str, object]],
+    blobs: dict[str, bytes],
+) -> None:
+    for entry in tracked:
+        relative = str(entry["path"])
+        target = root.joinpath(*PurePosixPath(relative).parts)
+        try:
+            target_status = target.lstat()
+        except OSError:
+            fail("the Git worktree does not exactly match HEAD")
+        if not stat.S_ISREG(target_status.st_mode):
+            fail("the Git worktree does not exactly match HEAD")
+        actual_mode = "100755" if target_status.st_mode & 0o111 else "100644"
+        if actual_mode != entry["git_mode"]:
+            fail("the Git worktree does not exactly match HEAD")
+        try:
+            actual = target.read_bytes()
+        except OSError:
+            fail("the Git worktree does not exactly match HEAD")
+        if actual != blobs[relative]:
+            fail("the Git worktree does not exactly match HEAD")
+
+
+def verify_repository_unchanged(
+    root: Path,
+    expected_root_identity: tuple[int, int],
+    expected_sha: str,
+    expected_object_format: str,
+    expected_tracked: list[dict[str, object]],
+    expected_blobs: dict[str, bytes],
+) -> None:
+    verify_repository_identity(root, expected_root_identity)
+    if run(
+        [GIT_BINARY, "status", "--porcelain=v1", "--untracked-files=all", "-z"],
+        cwd=root,
+    ):
+        fail("the Git worktree changed during the release build")
+    actual_sha, actual_object_format, actual_tracked, actual_blobs = git_snapshot(root)
+    if actual_sha != expected_sha:
+        fail("HEAD changed during the release build")
+    if (
+        actual_object_format != expected_object_format
+        or actual_tracked != expected_tracked
+        or actual_blobs != expected_blobs
+    ):
+        fail("the tracked HEAD snapshot changed during the release build")
+    verify_worktree_matches_snapshot(root, actual_tracked, actual_blobs)
+
+
+def git_ignores(root: Path, path: Path, *, directory: bool = False) -> bool:
+    candidate = str(path)
+    if directory:
+        candidate = candidate.rstrip(os.sep) + os.sep
+    probe = subprocess.run(
+        [
+            GIT_BINARY,
+            "check-ignore",
+            "--quiet",
+            "--no-index",
+            "--",
+            candidate,
+        ],
+        cwd=root,
+        env=nonsecret_environment(),
+        capture_output=True,
+        check=False,
+        timeout=10,
+    )
+    return probe.returncode == 0
+
+
+def validate_output_path(
+    root: Path,
+    expected_root_identity: tuple[int, int],
+    requested: str,
+    blobs: dict[str, bytes],
+) -> Path:
+    verify_repository_identity(root, expected_root_identity)
+    ignore_blob = blobs.get(".gitignore")
+    if ignore_blob is None:
+        fail("the complete output boundary must be ignored by Git")
+    try:
+        ignore_lines = ignore_blob.decode("utf-8", "strict").splitlines()
+    except UnicodeDecodeError:
+        fail("the complete output boundary must be ignored by Git")
+    if "/output/" not in ignore_lines:
+        fail("the complete output boundary must be ignored by Git")
+
+    output = Path(requested)
+    if not output.is_absolute():
+        output = root / output
+    output = Path(os.path.abspath(output))
+    output_root = root / "output"
+    try:
+        relative = output.relative_to(output_root)
+    except ValueError:
+        fail("the output directory must be below the repository output directory")
+    if not relative.parts or output == output_root:
+        fail("the output directory must be a new child of output")
+
+    current = root
+    for part in output.relative_to(root).parts:
+        current /= part
+        try:
+            current_status = current.lstat()
+        except FileNotFoundError:
+            continue
+        except OSError:
+            fail("the output boundary cannot be inspected safely")
+        if stat.S_ISLNK(current_status.st_mode):
+            fail("the output directory may not traverse a symlink")
+        if current == output:
+            fail("the output directory must be a new child of output")
+        if not stat.S_ISDIR(current_status.st_mode):
+            fail("the output directory may only traverse directories")
+
+    boundary_probes = (
+        (output_root, True),
+        (output_root / ".release-boundary-probe", False),
+        (output_root / ".release-boundary-probe" / "nested", False),
+        (output, True),
+        (output / ".release-child-probe", False),
+    )
+    if any(
+        not git_ignores(root, path, directory=directory)
+        for path, directory in boundary_probes
+    ):
+        fail("the complete output boundary must be ignored by Git")
+
+    output.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
+    current = root
+    for part in output.parent.relative_to(root).parts:
+        current /= part
+        try:
+            current_status = current.lstat()
+        except OSError:
+            fail("the output boundary cannot be inspected safely")
+        if stat.S_ISLNK(current_status.st_mode) or not stat.S_ISDIR(
+            current_status.st_mode
+        ):
+            fail("the output directory may not traverse a symlink")
+    return output
+
+
+def create_isolated_workspace(root: Path) -> Path:
+    try:
+        system_temporary_root = Path(tempfile.gettempdir()).resolve(strict=True)
+    except OSError:
+        fail("an isolated build workspace is unavailable")
+    workspace = Path(
+        tempfile.mkdtemp(
+            prefix="travel-readiness-release-build-",
+            dir=system_temporary_root,
+        )
+    ).resolve(strict=True)
+    try:
+        workspace.relative_to(root)
+    except ValueError:
+        pass
+    else:
+        shutil.rmtree(workspace, ignore_errors=True)
+        fail("the isolated build workspace may not descend from the repository")
+    workspace_status = workspace.lstat()
+    if (
+        not stat.S_ISDIR(workspace_status.st_mode)
+        or stat.S_IMODE(workspace_status.st_mode) != 0o700
+        or workspace_status.st_uid != os.geteuid()
+    ):
+        shutil.rmtree(workspace, ignore_errors=True)
+        fail("the isolated build workspace has unsafe permissions")
+    return workspace
+
+
+def export_source_tree(
+    destination: Path,
+    tracked: list[dict[str, object]],
+    blobs: dict[str, bytes],
+) -> None:
+    destination.mkdir(mode=0o700)
+    for entry in tracked:
+        relative = str(entry["path"])
+        target = destination / relative
+        target.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
+        target.write_bytes(blobs[relative])
+        target.chmod(0o755 if entry["git_mode"] == "100755" else 0o644)
+
+
+def absolute_uv_binary(value: str) -> str:
+    candidate = Path(value)
+    if not candidate.is_absolute() or not candidate.is_file():
+        fail("--uv-bin must identify an absolute executable file")
+    resolved = candidate.resolve(strict=True)
+    if not os.access(resolved, os.X_OK):
+        fail("--uv-bin must identify an absolute executable file")
+    return str(resolved)
+
+
+def check_uv(
+    source_root: Path,
+    uv_binary: str,
+    expected_version: str,
+    cache: Path,
+) -> None:
+    environment = credential_free_environment(source_root)
+    environment.update(
+        {
+            "UV_CACHE_DIR": str(cache),
+            "UV_NO_CONFIG": "1",
+            "UV_OFFLINE": "1",
+            "UV_PYTHON_DOWNLOADS": "never",
+        }
+    )
+    version = run(
+        [uv_binary, "--version"], cwd=source_root, environment=environment
+    )
+    version_fields = version.decode("ascii", "strict").strip().split()
+    if version_fields[:2] != ["uv", expected_version]:
+        fail("uv does not match the pinned version")
+    run(
+        [uv_binary, "lock", "--check", "--offline"],
+        cwd=source_root,
+        environment=environment,
+    )
+    run(
+        [uv_binary, "pip", "check", "--python", sys.executable],
+        cwd=source_root,
+        environment=environment,
+    )
+
+
+def collect_build_metadata(source_root: Path, destination: Path) -> dict[str, object]:
+    metadata_path = destination / "build-metadata.json"
+    code = """
+import hashlib
+import importlib
+import json
+from pathlib import Path
+import sys
+import django
+
+source_root = Path(sys.argv[1]).resolve()
+static_root = Path(sys.argv[2]).resolve()
+django.setup()
+from django.conf import settings
+from django.contrib.staticfiles import finders
+from django.core.management import call_command
+from django.db.migrations.loader import MigrationLoader
+
+django_root = Path(django.__file__).resolve().parent
+settings.STATIC_ROOT = static_root
+call_command("collectstatic", interactive=False, clear=True, verbosity=0)
+
+def classify(path):
+    if path.is_symlink():
+        raise SystemExit(2)
+    resolved = path.resolve()
+    if not resolved.is_file():
+        raise SystemExit(2)
+    try:
+        relative = resolved.relative_to(source_root)
+        origin = "source"
+    except ValueError:
+        try:
+            relative = resolved.relative_to(django_root)
+        except ValueError:
+            raise SystemExit(2)
+        relative = Path("django") / relative
+        origin = "django"
+    return {
+        "origin": origin,
+        "path": relative.as_posix(),
+        "sha256": hashlib.sha256(resolved.read_bytes()).hexdigest(),
+    }
+
+leaves = []
+loader = MigrationLoader(None, ignore_no_migrations=True)
+for app, name in sorted(loader.graph.leaf_nodes()):
+    migration = loader.disk_migrations[(app, name)]
+    module = importlib.import_module(migration.__module__)
+    path = Path(module.__file__)
+    if path.suffix != ".py":
+        raise SystemExit(2)
+    leaf = {
+        "app": app,
+        "module": migration.__module__,
+        "name": name,
+    }
+    leaf.update(classify(path))
+    leaves.append(leaf)
+
+static_origins = []
+for collected in sorted(path for path in static_root.rglob("*") if path.is_file()):
+    relative = collected.relative_to(static_root).as_posix()
+    found = finders.find(relative, all=True)
+    if not isinstance(found, (list, tuple)) or len(found) != 1:
+        raise SystemExit(2)
+    item = {"collected_path": relative}
+    item.update(classify(Path(found[0])))
+    static_origins.append(item)
+
+Path(sys.argv[3]).write_text(
+    json.dumps(
+        {"leaves": leaves, "static_origins": static_origins},
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ) + "\\n",
+    encoding="utf-8",
+)
+"""
+    run(
+        [
+            sys.executable,
+            "-c",
+            code,
+            str(source_root),
+            str(destination / "static"),
+            str(metadata_path),
+        ],
+        cwd=source_root,
+        environment=credential_free_environment(source_root),
+    )
+    try:
+        metadata = json.loads(metadata_path.read_text(encoding="utf-8"))
+    except (json.JSONDecodeError, UnicodeDecodeError):
+        fail("the isolated build metadata is invalid")
+    metadata_path.unlink()
+    if not isinstance(metadata, dict) or not isinstance(
+        metadata.get("leaves"), list
+    ):
+        fail("the isolated build metadata has an invalid shape")
+    if not metadata["leaves"]:
+        fail("the migration graph has no leaves")
+    if not isinstance(metadata.get("static_origins"), list):
+        fail("the static origin metadata has an invalid shape")
+    return metadata
+
+
+def regular_file_hashes(root: Path) -> tuple[list[dict[str, str]], dict[str, bytes]]:
+    entries: list[dict[str, str]] = []
+    content: dict[str, bytes] = {}
+    for path in sorted(root.rglob("*")):
+        if path.is_symlink():
+            fail("collected static files may not contain symlinks")
+        if path.is_dir():
+            continue
+        if not path.is_file():
+            fail("collected static output contains an unsupported entry")
+        relative = path.relative_to(root).as_posix()
+        data = path.read_bytes()
+        entries.append({"path": relative, "sha256": sha256(data)})
+        content[relative] = data
+    if not entries:
+        fail("collectstatic produced no files")
+    return entries, content
+
+
+def validated_build_metadata(
+    metadata: dict[str, object],
+    blobs: dict[str, bytes],
+    static_entries: list[dict[str, str]],
+) -> tuple[list[dict[str, str]], list[dict[str, object]], list[dict[str, str]]]:
+    leaves = metadata["leaves"]
+    origins = metadata["static_origins"]
+    expected_static = {entry["path"] for entry in static_entries}
+    collected_hashes = {
+        entry["path"]: entry["sha256"] for entry in static_entries
+    }
+    observed_static: set[str] = set()
+    tracked_static: list[dict[str, str]] = []
+
+    def validate_source(item: dict[str, object]) -> None:
+        origin = item.get("origin")
+        path = item.get("path")
+        digest = item.get("sha256")
+        if origin == "source":
+            if not isinstance(path, str) or blobs.get(path) is None:
+                fail("isolated metadata references a non-HEAD source file")
+            if digest != sha256(blobs[path]):
+                fail("isolated source metadata does not match HEAD")
+        elif origin == "django":
+            if (
+                not isinstance(path, str)
+                or not path.startswith("django/")
+                or not isinstance(digest, str)
+                or len(digest) != 64
+                or any(character not in "0123456789abcdef" for character in digest)
+            ):
+                fail("isolated metadata references an unpinned dependency")
+            pure_path = PurePosixPath(path)
+            if pure_path.is_absolute() or ".." in pure_path.parts:
+                fail("isolated metadata references an unpinned dependency")
+            try:
+                django_root = Path(
+                    importlib.metadata.distribution("Django").locate_file("django")
+                ).resolve(strict=True)
+                installed_path = django_root.joinpath(*pure_path.parts[1:])
+                installed_status = installed_path.lstat()
+            except (OSError, importlib.metadata.PackageNotFoundError):
+                fail("isolated metadata references an unpinned dependency")
+            if stat.S_ISLNK(installed_status.st_mode) or not stat.S_ISREG(
+                installed_status.st_mode
+            ):
+                fail("isolated metadata references an unpinned dependency")
+            try:
+                installed_relative = installed_path.resolve(strict=True).relative_to(
+                    django_root
+                )
+            except (OSError, ValueError):
+                fail("isolated metadata references an unpinned dependency")
+            expected_relative = PurePosixPath(*pure_path.parts[1:]).as_posix()
+            if installed_relative.as_posix() != expected_relative:
+                fail("isolated metadata references an unpinned dependency")
+            if sha256(installed_path.read_bytes()) != digest:
+                fail("isolated dependency metadata does not match the pinned runtime")
+        else:
+            fail("isolated metadata has an unknown source origin")
+
+    for leaf in leaves:
+        if not isinstance(leaf, dict) or set(leaf) != {
+            "app",
+            "module",
+            "name",
+            "origin",
+            "path",
+            "sha256",
+        }:
+            fail("the migration leaf metadata has an invalid shape")
+        if any(not isinstance(leaf[key], str) for key in ("app", "module", "name")):
+            fail("the migration leaf metadata has an invalid identity")
+        validate_source(leaf)
+
+    for origin in origins:
+        if not isinstance(origin, dict) or set(origin) != {
+            "collected_path",
+            "origin",
+            "path",
+            "sha256",
+        }:
+            fail("the static origin metadata has an invalid shape")
+        collected_path = origin["collected_path"]
+        if not isinstance(collected_path, str) or collected_path in observed_static:
+            fail("the static origin metadata has an invalid collected path")
+        observed_static.add(collected_path)
+        validate_source(origin)
+        if origin["sha256"] != collected_hashes.get(collected_path):
+            fail("collected static content does not match its authoritative origin")
+        if origin["origin"] == "source":
+            tracked_static.append(
+                {
+                    "collected_path": collected_path,
+                    "path": str(origin["path"]),
+                    "sha256": str(origin["sha256"]),
+                }
+            )
+    if observed_static != expected_static:
+        fail("static origins do not match collected static files")
+    tracked_static.sort(key=lambda item: (item["collected_path"], item["path"]))
+    return leaves, origins, tracked_static
+
+
+def normalized_tar(
+    path: Path,
+    *,
+    manifest: bytes,
+    tracked: list[dict[str, object]],
+    blobs: dict[str, bytes],
+    static: dict[str, bytes],
+) -> None:
+    files: dict[str, tuple[bytes, int]] = {
+        "release/manifest.json": (manifest, 0o644),
+    }
+    for entry in tracked:
+        source_path = str(entry["path"])
+        mode = 0o755 if entry["git_mode"] == "100755" else 0o644
+        files[f"release/source/{source_path}"] = (blobs[source_path], mode)
+    for static_path, data in static.items():
+        files[f"release/static/{static_path}"] = (data, 0o644)
+
+    directories = {"release"}
+    for name in files:
+        parent = PurePosixPath(name).parent
+        while str(parent) not in {"", "."}:
+            directories.add(parent.as_posix())
+            parent = parent.parent
+
+    with tarfile.open(path, mode="x", format=tarfile.USTAR_FORMAT) as archive:
+        ordered_directories = sorted(
+            directories, key=lambda value: (value.count("/"), value)
+        )
+        for name in ordered_directories:
+            info = tarfile.TarInfo(name)
+            normalize_tar_info(info, mode=0o755)
+            info.type = tarfile.DIRTYPE
+            archive.addfile(info)
+        for name in sorted(files):
+            data, mode = files[name]
+            info = tarfile.TarInfo(name)
+            normalize_tar_info(info, mode=mode)
+            info.size = len(data)
+            archive.addfile(info, fileobj=io.BytesIO(data))
+
+
+def normalize_tar_info(info: tarfile.TarInfo, *, mode: int) -> None:
+    info.uid = 0
+    info.gid = 0
+    info.uname = "root"
+    info.gname = "root"
+    info.mtime = 0
+    info.mode = mode
+
+
+def build(arguments: argparse.Namespace) -> tuple[str, str]:
+    root = repository_root()
+    root_identity = repository_identity(root)
+    release_sha, object_format, tracked, blobs = git_snapshot(root)
+    verify_worktree_matches_snapshot(root, tracked, blobs)
+    runtime_bytes = blobs.get("runtime/versions.toml")
+    if runtime_bytes is None:
+        fail("runtime/versions.toml is not tracked")
+    runtime = runtime_configuration(runtime_bytes)
+    maybe_reexec_with_pinned_python(root, runtime)
+    verify_installed_runtime(runtime)
+    uv_binary = absolute_uv_binary(arguments.uv_bin)
+    output = validate_output_path(
+        root, root_identity, arguments.output_dir, blobs
+    )
+
+    uv_workspace: Path | None = None
+    build_workspace: Path | None = None
+    artifact_workspace: Path | None = None
+    temporary: Path | None = None
+    try:
+        uv_workspace = create_isolated_workspace(root)
+        uv_source_root = uv_workspace / "source-checkout"
+        export_source_tree(uv_source_root, tracked, blobs)
+        check_uv(
+            uv_source_root,
+            uv_binary,
+            str(runtime["uv"]),
+            uv_workspace / "uv-cache",
+        )
+        shutil.rmtree(uv_workspace)
+        uv_workspace = None
+
+        build_workspace = create_isolated_workspace(root)
+        source_root = build_workspace / "source-checkout"
+        generated = build_workspace / "generated"
+        generated.mkdir(mode=0o700)
+        export_source_tree(source_root, tracked, blobs)
+        metadata = collect_build_metadata(source_root, generated)
+        static_entries, static_content = regular_file_hashes(generated / "static")
+        leaves, static_origins, tracked_static = validated_build_metadata(
+            metadata, blobs, static_entries
+        )
+        lock_bytes = blobs.get("uv.lock")
+        if lock_bytes is None:
+            fail("uv.lock is not tracked")
+        lock = tomllib.loads(lock_bytes.decode("utf-8"))
+        locked_packages = lock.get("package")
+        if not isinstance(locked_packages, list) or not locked_packages:
+            fail("uv.lock contains no package inventory")
+        if any(
+            not isinstance(package, dict)
+            or not isinstance(package.get("name"), str)
+            or not isinstance(package.get("version"), str)
+            for package in locked_packages
+        ):
+            fail("uv.lock contains an invalid package inventory")
+        packages = sorted(
+            {(package["name"], package["version"]) for package in locked_packages}
+        )
+        package_entries = [
+            {"name": name, "version": version} for name, version in packages
+        ]
+        leaf_bytes = canonical_json(leaves)
+        package_bytes = canonical_json(package_entries)
+        static_bytes = canonical_json(static_entries)
+        tracked_static_bytes = canonical_json(tracked_static)
+        tracked_bytes = canonical_json(tracked)
+        manifest_value = {
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
+                "lock_sha256": sha256(lock_bytes),
+                "package_set_sha256": sha256(package_bytes),
+                "packages": package_entries,
+            },
+            "git_object_format": object_format,
+            "migrations": {
+                "leaf_set_sha256": sha256(leaf_bytes),
+                "leaves": leaves,
+            },
+            "release_sha": release_sha,
+            "runtime": {
+                "path": "runtime/versions.toml",
+                "sha256": sha256(runtime_bytes),
+                "versions": runtime,
+            },
+            "schema_version": SCHEMA_VERSION,
+            "static": {
+                "collected": static_entries,
+                "collected_set_sha256": sha256(static_bytes),
+                "origins": static_origins,
+                "tracked": tracked_static,
+                "tracked_set_sha256": sha256(tracked_static_bytes),
+            },
+            "source": {
+                "tracked": tracked,
+                "tracked_set_sha256": sha256(tracked_bytes),
+            },
+        }
+        manifest = canonical_json(manifest_value)
+        shutil.rmtree(build_workspace)
+        build_workspace = None
+
+        artifact_workspace = create_isolated_workspace(root)
+        archive_path = artifact_workspace / ARCHIVE_BASENAME
+        normalized_tar(
+            archive_path,
+            manifest=manifest,
+            tracked=tracked,
+            blobs=blobs,
+            static=static_content,
+        )
+        artifact_digest = sha256(archive_path.read_bytes())
+        (artifact_workspace / MANIFEST_BASENAME).write_bytes(manifest)
+        (artifact_workspace / DIGEST_BASENAME).write_text(
+            f"{artifact_digest}  {ARCHIVE_BASENAME}\n", encoding="ascii"
+        )
+
+        output = validate_output_path(root, root_identity, str(output), blobs)
+        temporary = Path(
+            tempfile.mkdtemp(
+                prefix=f".{output.name}.incomplete-", dir=output.parent
+            )
+        )
+        os.chmod(temporary, 0o700)
+        if not git_ignores(root, temporary, directory=True) or not git_ignores(
+            root, temporary / ".release-staging-probe"
+        ):
+            fail("the release staging directory must be ignored by Git")
+
+        expected_output = {
+            ARCHIVE_BASENAME,
+            MANIFEST_BASENAME,
+            DIGEST_BASENAME,
+        }
+        if {child.name for child in artifact_workspace.iterdir()} != expected_output:
+            fail("the isolated artifact boundary is incomplete")
+        for name in sorted(expected_output):
+            source = artifact_workspace / name
+            source_status = source.lstat()
+            if source.is_symlink() or not stat.S_ISREG(source_status.st_mode):
+                fail("the isolated artifact boundary contains an unexpected entry")
+            shutil.copyfile(source, temporary / name)
+        shutil.rmtree(artifact_workspace)
+        artifact_workspace = None
+
+        for child in temporary.iterdir():
+            if child.is_symlink() or not child.is_file():
+                fail("the release output boundary contains an unexpected entry")
+            child.chmod(0o600)
+        if {child.name for child in temporary.iterdir()} != expected_output:
+            fail("the release output boundary is incomplete")
+        if any(not git_ignores(root, child) for child in temporary.iterdir()):
+            fail("the release staging contents must be ignored by Git")
+        verify_repository_unchanged(
+            root,
+            root_identity,
+            release_sha,
+            object_format,
+            tracked,
+            blobs,
+        )
+        temporary.rename(output)
+        output.chmod(0o500)
+        for child in output.iterdir():
+            child.chmod(0o400)
+        return release_sha, artifact_digest
+    except Exception:
+        if temporary is not None:
+            shutil.rmtree(temporary, ignore_errors=True)
+        for workspace in (uv_workspace, build_workspace, artifact_workspace):
+            if workspace is not None:
+                shutil.rmtree(workspace, ignore_errors=True)
+        raise
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(
+        description="Build a deterministic local release below ignored output/."
+    )
+    parser.add_argument("--output-dir", required=True)
+    parser.add_argument("--uv-bin", required=True)
+    return parser.parse_args()
+
+
+def main() -> int:
+    try:
+        release_sha, artifact_digest = build(parse_arguments())
+    except BuildError as error:
+        print(f"release_build=failed reason={error}", file=sys.stderr)
+        return 1
+    except Exception:
+        print("release_build=failed reason=internal-build-error", file=sys.stderr)
+        return 1
+    print("release_build=ok")
+    print(f"release_sha={release_sha}")
+    print(f"artifact_sha256={artifact_digest}")
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


