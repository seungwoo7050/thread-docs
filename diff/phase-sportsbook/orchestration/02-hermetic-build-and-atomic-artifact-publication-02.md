## `fix(shared): require one locked protocol input`

diff --git a/scripts/install-shared.sh b/scripts/install-shared.sh
index 18d7573..ffd0c1e 100755
--- a/scripts/install-shared.sh
+++ b/scripts/install-shared.sh
@@ -13,9 +13,15 @@ fail() {
   exit 1
 }
 
-IFS='|' read -r logical branch commit artifact < <(
-  awk -F'|' '$1 == "shared" { print; found=1 } END { if (!found) exit 2 }' "$LOCK"
-)
+entry=$(awk -F'|' '
+  NF && $1 !~ /^#/ {
+    if (NF != 4) exit 2
+    total++
+    if ($1 == "shared") { print; shared++ }
+  }
+  END { if (total != 8 || shared != 1) exit 2 }
+' "$LOCK") || fail "invalid services lock"
+IFS='|' read -r logical branch commit artifact <<<"$entry"
 [[ $logical == shared && $artifact == shared-protocol-1.0.0.jar ]] \
   || fail "invalid shared lock entry"
 


## `test(shared): reject duplicate protocol locks`

diff --git a/tests/test_shared_install.py b/tests/test_shared_install.py
index c172a45..6a790eb 100644
--- a/tests/test_shared_install.py
+++ b/tests/test_shared_install.py
@@ -98,6 +98,19 @@ class SharedInstallTest(unittest.TestCase):
                 (source / "target/shared-protocol-1.0.0.jar").read_bytes(),
                 (installed / "shared-protocol-1.0.0.jar").read_bytes(),
             )
+            lock = temporary_path / "duplicate.lock"
+            locked = (ROOT / "services.lock").read_text()
+            lock.write_text(locked + locked.splitlines()[1] + "\n")
+            failed = subprocess.run(
+                [str(SCRIPT), source_root.name, repository.name],
+                cwd=temporary_path,
+                env=environment | {"SERVICES_LOCK": str(lock)},
+                text=True,
+                capture_output=True,
+                check=False,
+            )
+            self.assertNotEqual(failed.returncode, 0)
+            self.assertIn("invalid services lock", failed.stderr)
             subprocess.run(
                 ["git", "worktree", "remove", "--force", str(source)],
                 cwd=ROOT,


## `build(gate): assemble exact release artifacts`

diff --git a/scripts/cold_gate/build.py b/scripts/cold_gate/build.py
new file mode 100644
index 0000000..0c454cc
--- /dev/null
+++ b/scripts/cold_gate/build.py
@@ -0,0 +1,79 @@
+from __future__ import annotations
+
+import dataclasses
+import os
+import subprocess
+from collections.abc import Callable
+from pathlib import Path
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.owned_path import ensure_directory
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+
+
+@dataclasses.dataclass(frozen=True)
+class ReleaseArtifacts:
+    sources: Path
+    maven_repository: Path
+    service_jars: Path
+    fixture_jar: Path
+
+
+class ReleaseBuilder:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        environment: dict[str, str],
+        runner: Runner = subprocess.run,
+    ) -> None:
+        self.context = context
+        self.environment = environment
+        self.runner = runner
+
+    def build(self) -> ReleaseArtifacts:
+        self.context.require_owned()
+        root = self.context.root
+        sources = self.context.runtime / "sources"
+        repository = self.context.runtime / "m2/repository"
+        fixture_output = self.context.runtime / "fixtures"
+        jars_link = root / "docker/jars"
+        generations = root / "docker/.jars"
+        if any(path.exists() or path.is_symlink() for path in (jars_link, generations)):
+            raise RuntimeError("release JAR staging is not empty")
+        ensure_directory(repository.parent)
+        ensure_directory(repository)
+        ensure_directory(fixture_output)
+
+        commands = (
+            [str(root / "scripts/materialize-sources.sh"), str(sources), "materialize"],
+            [str(root / "scripts/install-shared.sh"), str(sources), str(repository)],
+            [str(root / "scripts/stage-release-jars.sh"), str(sources), str(repository)],
+            [
+                str(root / "scripts/stage-fixture-publisher.sh"),
+                str(sources),
+                str(repository),
+                str(fixture_output),
+            ],
+        )
+        for command in commands:
+            environment = self.environment.copy()
+            if command[0].endswith("stage-release-jars.sh"):
+                environment["DOCKER_OUTPUT_ROOT"] = str(root / "docker")
+            self.runner(
+                command,
+                cwd=root,
+                env=environment,
+                text=True,
+                capture_output=True,
+                check=True,
+            )
+
+        fixture = fixture_output / "avro-fixture-publisher.jar"
+        if not jars_link.is_symlink() or not fixture.is_file():
+            raise RuntimeError("release artifacts are incomplete")
+        service_jars = (root / "docker" / os.readlink(jars_link)).resolve(strict=True)
+        if len(list(service_jars.glob("*.jar"))) != 7:
+            raise RuntimeError("service JAR generation is incomplete")
+        return ReleaseArtifacts(sources, repository, service_jars, fixture)


## `test(gate): verify locked build sequence`

diff --git a/tests/test_release_builder.py b/tests/test_release_builder.py
new file mode 100644
index 0000000..91db791
--- /dev/null
+++ b/tests/test_release_builder.py
@@ -0,0 +1,78 @@
+import os
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.build import ReleaseBuilder
+from scripts.cold_gate.context import ColdGateContext
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+SERVICES = ("wallet", "risk", "odds", "betting", "gateway", "settlement", "admin")
+
+
+class ReleaseBuilderTest(unittest.TestCase):
+    def context(self, root: pathlib.Path) -> ColdGateContext:
+        (root / "docker").mkdir()
+        (root / "scripts").mkdir()
+        return ColdGateContext.create(root, SHA, "00000001")
+
+    def test_builds_in_locked_order_and_returns_exact_artifacts(self) -> None:
+        calls = []
+
+        def runner(command, **options):
+            calls.append((command, options))
+            name = pathlib.Path(command[0]).name
+            if name == "materialize-sources.sh":
+                pathlib.Path(command[1]).mkdir()
+            elif name == "stage-release-jars.sh":
+                docker = pathlib.Path(options["env"]["DOCKER_OUTPUT_ROOT"])
+                generation = docker / ".jars/generation.test"
+                generation.mkdir(parents=True)
+                for service in SERVICES:
+                    (generation / f"{service}.jar").write_bytes(service.encode())
+                (docker / "jars").symlink_to(".jars/generation.test")
+            elif name == "stage-fixture-publisher.sh":
+                pathlib.Path(command[3], "avro-fixture-publisher.jar").write_bytes(
+                    b"fixture"
+                )
+            return subprocess.CompletedProcess(command, 0, stdout="")
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+
+            artifacts = ReleaseBuilder(context, {"JAVA_HOME": "/jdk17"}, runner).build()
+
+            self.assertEqual(
+                [pathlib.Path(call[0][0]).name for call in calls],
+                [
+                    "materialize-sources.sh",
+                    "install-shared.sh",
+                    "stage-release-jars.sh",
+                    "stage-fixture-publisher.sh",
+                ],
+            )
+            self.assertEqual(
+                {path.name for path in artifacts.service_jars.glob("*.jar")},
+                {f"{service}.jar" for service in SERVICES},
+            )
+            self.assertEqual(artifacts.sources, context.runtime / "sources")
+            self.assertEqual(artifacts.fixture_jar.read_bytes(), b"fixture")
+
+    def test_rejects_preexisting_staging_without_running_commands(self) -> None:
+        calls = []
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            (root / "docker/.jars").mkdir()
+
+            with self.assertRaisesRegex(RuntimeError, "not empty"):
+                ReleaseBuilder(context, os.environ.copy(), calls.append).build()
+
+            self.assertEqual(calls, [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(gate): remove only staged release jars`

diff --git a/scripts/cold_gate/cleanup.py b/scripts/cold_gate/cleanup.py
index c81608b..7c9252a 100644
--- a/scripts/cold_gate/cleanup.py
+++ b/scripts/cold_gate/cleanup.py
@@ -1,5 +1,6 @@
 from __future__ import annotations
 
+import os
 import shutil
 import subprocess
 from collections.abc import Callable
@@ -7,6 +8,7 @@ from pathlib import Path
 
 from scripts.cold_gate.compose import ComposeProject
 from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.owned_path import require_directory
 
 
 Runner = Callable[..., subprocess.CompletedProcess[str]]
@@ -25,12 +27,23 @@ class ScopedCleanup:
             raise RuntimeError("cleanup Compose project has different ownership")
         self.runner = runner
 
-    def run(self, sources: Path | None = None) -> None:
+    def run(self, sources: Path | None = None, service_jars: Path | None = None) -> None:
         self.context.require_owned()
         if sources is not None:
             expected = self.context.runtime / "sources"
             if sources != expected or sources.is_symlink() or not sources.is_dir():
                 raise RuntimeError("source cleanup target is not owned by this run")
+        jars_link = self.context.root / "docker/jars"
+        if service_jars is not None:
+            expected_parent = self.context.root / "docker/.jars"
+            expected_link = f".jars/{service_jars.name}"
+            if (
+                service_jars.parent != expected_parent
+                or not jars_link.is_symlink()
+                or os.readlink(jars_link) != expected_link
+            ):
+                raise RuntimeError("service JAR cleanup target is not owned")
+            require_directory(service_jars)
 
         self.compose.run(
             "down",
@@ -57,6 +70,10 @@ class ScopedCleanup:
             )
             if sources.exists() or sources.is_symlink():
                 raise RuntimeError("materialized sources remain after cleanup")
+        if service_jars is not None:
+            jars_link.unlink()
+            shutil.rmtree(service_jars)
+            service_jars.parent.rmdir()
 
         self.context.require_owned()
         shutil.rmtree(self.context.runtime)


## `test(gate): verify exact JAR generation cleanup`

diff --git a/tests/test_scoped_cleanup.py b/tests/test_scoped_cleanup.py
index 87dd6d3..adbc103 100644
--- a/tests/test_scoped_cleanup.py
+++ b/tests/test_scoped_cleanup.py
@@ -37,9 +37,13 @@ class ScopedCleanupTest(unittest.TestCase):
             context = ColdGateContext.create(root, SHA, "00000001")
             sources = context.runtime / "sources"
             sources.mkdir()
+            generation = root / "docker/.jars/generation.test"
+            generation.mkdir(parents=True)
+            (generation / "wallet.jar").write_bytes(b"jar")
+            (root / "docker/jars").symlink_to(".jars/generation.test")
             compose = FakeCompose(context)
 
-            ScopedCleanup(context, compose, runner).run(sources)
+            ScopedCleanup(context, compose, runner).run(sources, generation)
 
             self.assertEqual(
                 compose.calls,
@@ -60,6 +64,8 @@ class ScopedCleanupTest(unittest.TestCase):
             self.assertFalse(context.runtime.exists())
             self.assertFalse(context.lock.exists())
             self.assertTrue(context.evidence.is_dir())
+            self.assertFalse((root / "docker/jars").exists())
+            self.assertFalse((root / "docker/.jars").exists())
 
     def test_rejects_foreign_or_symlinked_source_targets(self) -> None:
         with tempfile.TemporaryDirectory() as temporary:


## `build(gate): verify staged release identity`

diff --git a/scripts/cold_gate/artifacts.py b/scripts/cold_gate/artifacts.py
new file mode 100644
index 0000000..4d4564d
--- /dev/null
+++ b/scripts/cold_gate/artifacts.py
@@ -0,0 +1,66 @@
+from __future__ import annotations
+
+import hashlib
+import re
+import zipfile
+from pathlib import Path
+
+from scripts.cold_gate.owned_path import require_directory, require_regular_file
+
+
+SERVICES = ("wallet", "risk", "odds", "betting", "gateway", "settlement", "admin")
+SUM_PATTERN = re.compile(r"^([0-9a-f]{64})  ([a-z]+\.jar)$")
+
+
+def verify_release_artifacts(directory: Path) -> dict[str, str]:
+    require_directory(directory)
+    expected = {f"{service}.jar" for service in SERVICES}
+    sums_file = directory / "SHA256SUMS"
+    require_regular_file(sums_file)
+    if {path.name for path in directory.iterdir()} != expected | {"SHA256SUMS"}:
+        raise RuntimeError("release JAR inventory is not exact")
+
+    declared: dict[str, str] = {}
+    for line in sums_file.read_text().splitlines():
+        match = SUM_PATTERN.fullmatch(line)
+        if match is None or match.group(2) in declared:
+            raise RuntimeError("release checksum manifest is invalid")
+        declared[match.group(2)] = match.group(1)
+    if set(declared) != expected:
+        raise RuntimeError("release checksum inventory is incomplete")
+
+    for name in sorted(expected):
+        jar = directory / name
+        require_regular_file(jar)
+        actual = hashlib.sha256(jar.read_bytes()).hexdigest()
+        if declared[name] != actual:
+            raise RuntimeError(f"{name} checksum mismatch")
+        _verify_executable_jar(jar)
+    return declared
+
+
+def _verify_executable_jar(path: Path) -> None:
+    try:
+        with zipfile.ZipFile(path) as archive:
+            names = archive.namelist()
+            manifest = archive.read("META-INF/MANIFEST.MF").decode("utf-8")
+            start_class = _manifest_value(manifest, "Start-Class")
+            main_path = "BOOT-INF/classes/" + start_class.replace(".", "/") + ".class"
+            bytecode = archive.read(main_path)
+            shared = [name for name in names if name.startswith("BOOT-INF/lib/shared-")]
+    except (KeyError, UnicodeDecodeError, zipfile.BadZipFile) as error:
+        raise RuntimeError(f"{path.name} is not an executable release JAR") from error
+    if len(bytecode) < 8 or bytecode[:4] != b"\xca\xfe\xba\xbe":
+        raise RuntimeError(f"{path.name} application entrypoint is not bytecode")
+    if int.from_bytes(bytecode[6:8], "big") != 61:
+        raise RuntimeError(f"{path.name} application class is not Java 17")
+    if shared != ["BOOT-INF/lib/shared-1.0.0.jar"]:
+        raise RuntimeError(f"{path.name} shared protocol dependency drifted")
+
+
+def _manifest_value(manifest: str, key: str) -> str:
+    prefix = f"{key}: "
+    values = [line[len(prefix) :] for line in manifest.splitlines() if line.startswith(prefix)]
+    if len(values) != 1 or not values[0]:
+        raise RuntimeError(f"manifest must contain one {key}")
+    return values[0]


## `test(gate): reject release artifact drift`

diff --git a/tests/test_release_artifact_identity.py b/tests/test_release_artifact_identity.py
new file mode 100644
index 0000000..9e5f16d
--- /dev/null
+++ b/tests/test_release_artifact_identity.py
@@ -0,0 +1,70 @@
+import hashlib
+import pathlib
+import tempfile
+import unittest
+import zipfile
+
+from scripts.cold_gate.artifacts import SERVICES, verify_release_artifacts
+
+
+def class_bytes(major: int) -> bytes:
+    return b"\xca\xfe\xba\xbe\x00\x00" + major.to_bytes(2, "big")
+
+
+def write_release(root: pathlib.Path, major: int = 61, shared: str = "1.0.0") -> None:
+    sums = []
+    for service in SERVICES:
+        jar = root / f"{service}.jar"
+        start = f"com.sportsbook.{service}.Application"
+        with zipfile.ZipFile(jar, "w") as archive:
+            archive.writestr(
+                "META-INF/MANIFEST.MF",
+                f"Manifest-Version: 1.0\nStart-Class: {start}\n",
+            )
+            archive.writestr(
+                "BOOT-INF/classes/" + start.replace(".", "/") + ".class",
+                class_bytes(major),
+            )
+            archive.writestr(f"BOOT-INF/lib/shared-{shared}.jar", b"shared")
+        sums.append(f"{hashlib.sha256(jar.read_bytes()).hexdigest()}  {jar.name}")
+    (root / "SHA256SUMS").write_text("\n".join(sums) + "\n")
+
+
+class ReleaseArtifactIdentityTest(unittest.TestCase):
+    def test_verifies_hash_class_and_shared_protocol_identity(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            write_release(root)
+
+            actual = verify_release_artifacts(root)
+
+            self.assertEqual(set(actual), {f"{service}.jar" for service in SERVICES})
+
+    def test_rejects_checksum_drift(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            write_release(root)
+            (root / "wallet.jar").write_bytes(b"tampered")
+
+            with self.assertRaisesRegex(RuntimeError, "checksum mismatch"):
+                verify_release_artifacts(root)
+
+    def test_rejects_non_java_17_application_class(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            write_release(root, major=65)
+
+            with self.assertRaisesRegex(RuntimeError, "not Java 17"):
+                verify_release_artifacts(root)
+
+    def test_rejects_shared_protocol_version_drift(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            write_release(root, shared="1.0.1")
+
+            with self.assertRaisesRegex(RuntimeError, "protocol dependency drifted"):
+                verify_release_artifacts(root)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(lock): pin executable settlement release`

diff --git a/services.lock b/services.lock
index 733f842..9f89636 100644
--- a/services.lock
+++ b/services.lock
@@ -5,5 +5,5 @@ risk|risk-service|c64f67dbc437a18640dc4984dea4d8194fb5b164|risk-service-1.0.0.ja
 odds|odds-feed-service|574e83d2862f086ae07ff56fd95a8336f78a72da|odds-feed-service-1.0.0.jar
 betting|betting-service|f712bdf389ee3fb63d8cdc84c49e2b84a346edde|betting-service-1.0.0.jar
 gateway|gateway|8248a3233f0fce7ca36a503ee71b7a8a0802d733|gateway-1.0.0.jar
-settlement|settlement-service|fc53ee8bfbb99b083f504d414d84ae5a994e4b57|settlement-service-1.0.0.jar
+settlement|settlement-service|e935873660aad4ceb28788521f7657289f97bc15|settlement-service-1.0.0.jar
 admin|admin-api|2fb55910475b31084e6489bf01c34cc970c96874|admin-api-1.0.0.jar


## `test(lock): require executable settlement packaging`

diff --git a/tests/test_services_lock.py b/tests/test_services_lock.py
index 695c501..31f323f 100644
--- a/tests/test_services_lock.py
+++ b/tests/test_services_lock.py
@@ -44,6 +44,14 @@ class ServicesLockTest(unittest.TestCase):
     def test_does_not_create_an_orchestration_lock_cycle(self) -> None:
         self.assertNotIn("orchestration", {entry[0] for entry in entries()})
 
+    def test_settlement_release_declares_executable_packaging(self) -> None:
+        commit = next(entry[2] for entry in entries() if entry[0] == "settlement")
+        pom = subprocess.check_output(
+            ["git", "show", f"{commit}:pom.xml"], cwd=ROOT, text=True
+        )
+
+        self.assertIn("<artifactId>spring-boot-maven-plugin</artifactId>", pom)
+
 
 if __name__ == "__main__":
     unittest.main()
