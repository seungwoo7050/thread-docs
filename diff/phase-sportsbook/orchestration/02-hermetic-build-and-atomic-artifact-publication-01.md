# 격리 빌드와 원자적 아티팩트 게시

## `build(shared): install the locked protocol artifact`

diff --git a/scripts/install-shared.sh b/scripts/install-shared.sh
new file mode 100755
index 0000000..202ab21
--- /dev/null
+++ b/scripts/install-shared.sh
@@ -0,0 +1,51 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT=$(git rev-parse --show-toplevel)
+SOURCE_ROOT=${1:-$ROOT/.runtime/sources}
+MAVEN_REPO=${2:-$ROOT/.runtime/m2/repository}
+LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
+RUNNER=${MAVEN_RUNNER:-}
+
+fail() {
+  printf 'shared-install: %s\n' "$1" >&2
+  exit 1
+}
+
+IFS='|' read -r logical branch commit artifact < <(
+  awk -F'|' '$1 == "shared" { print; found=1 } END { if (!found) exit 2 }' "$LOCK"
+)
+[[ $logical == shared && $artifact == shared-protocol-1.0.0.jar ]] \
+  || fail "invalid shared lock entry"
+
+SOURCE=$SOURCE_ROOT/shared
+[[ -f $SOURCE/.git && ! -L $SOURCE ]] || fail "shared source is not a detached worktree"
+[[ $(git -C "$SOURCE" rev-parse HEAD) == "$commit" ]] || fail "shared source SHA mismatch"
+! git -C "$SOURCE" symbolic-ref -q HEAD >/dev/null || fail "shared source is attached"
+[[ -z $(git -C "$SOURCE" status --porcelain) ]] || fail "shared source is dirty"
+
+JAVA_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}java
+JAVA_MAJOR=$($JAVA_BIN -version 2>&1 | awk -F'[."]' '/version/ {print $2; exit}')
+[[ $JAVA_MAJOR == 17 ]] || fail "Java 17 is required"
+
+mkdir -p "$MAVEN_REPO"
+MAVEN_REPO=$(cd "$MAVEN_REPO" && pwd -P)
+[[ $MAVEN_REPO != / && $MAVEN_REPO != "$ROOT" ]] || fail "unsafe Maven repository"
+[[ ! -L $MAVEN_REPO ]] || fail "Maven repository must not be a symlink"
+
+[[ -n $RUNNER ]] || RUNNER=$SOURCE/mvnw
+[[ -x $RUNNER ]] || fail "Maven runner is not executable"
+(
+  cd "$SOURCE"
+  "$RUNNER" -B "-Dmaven.repo.local=$MAVEN_REPO" -DskipTests clean install
+)
+
+SOURCE_JAR=$SOURCE/target/$artifact
+INSTALLED=$MAVEN_REPO/com/sportsbook/shared-protocol/1.0.0/$artifact
+INSTALLED_POM=$MAVEN_REPO/com/sportsbook/shared-protocol/1.0.0/shared-protocol-1.0.0.pom
+[[ -f $SOURCE_JAR && -f $INSTALLED && -f $INSTALLED_POM ]] \
+  || fail "shared 1.0.0 artifacts are incomplete"
+cmp -s "$SOURCE_JAR" "$INSTALLED" || fail "installed JAR differs from the build output"
+jar tf "$INSTALLED" | grep -qx 'com/sportsbook/protocol/value/Money.class' \
+  || fail "installed JAR is not shared-protocol 1.0.0"
+grep -q '<version>1.0.0</version>' "$INSTALLED_POM" || fail "installed POM version mismatch"


## `test(shared): verify installed artifact identity`

diff --git a/tests/test_shared_install.py b/tests/test_shared_install.py
new file mode 100644
index 0000000..4b07997
--- /dev/null
+++ b/tests/test_shared_install.py
@@ -0,0 +1,96 @@
+import os
+import pathlib
+import subprocess
+import tempfile
+import textwrap
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SCRIPT = ROOT / "scripts/install-shared.sh"
+SHARED_SHA = "f9de6bc1e533761ab4bb1454d8d4ab8175cdf001"
+
+
+class SharedInstallTest(unittest.TestCase):
+    def test_installs_only_the_locked_shared_artifact(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = pathlib.Path(temporary)
+            source_root = temporary_path / "sources"
+            source = source_root / "shared"
+            repository = temporary_path / "m2"
+            runner = temporary_path / "fake-maven"
+            source_root.mkdir()
+            subprocess.run(
+                ["git", "worktree", "add", "--quiet", "--detach", str(source), SHARED_SHA],
+                cwd=ROOT,
+                check=True,
+            )
+            self.addCleanup(
+                subprocess.run,
+                ["git", "worktree", "remove", "--force", str(source)],
+                cwd=ROOT,
+                check=False,
+                stdout=subprocess.DEVNULL,
+                stderr=subprocess.DEVNULL,
+            )
+            runner.write_text(
+                textwrap.dedent(
+                    """\
+                    #!/usr/bin/env python3
+                    import pathlib
+                    import shutil
+                    import sys
+                    import zipfile
+
+                    repository = pathlib.Path(next(
+                        value.split("=", 1)[1]
+                        for value in sys.argv
+                        if value.startswith("-Dmaven.repo.local=")
+                    ))
+                    source = pathlib.Path.cwd()
+                    artifact = source / "target/shared-protocol-1.0.0.jar"
+                    artifact.parent.mkdir(parents=True)
+                    with zipfile.ZipFile(artifact, "w") as archive:
+                        archive.writestr("com/sportsbook/protocol/value/Money.class", b"class")
+                    destination = repository / "com/sportsbook/shared-protocol/1.0.0"
+                    destination.mkdir(parents=True)
+                    shutil.copy2(artifact, destination / artifact.name)
+                    (destination / "shared-protocol-1.0.0.pom").write_text(
+                        "<project><version>1.0.0</version></project>\\n"
+                    )
+                    """
+                )
+            )
+            runner.chmod(0o755)
+            environment = os.environ.copy()
+            environment["JAVA_HOME"] = "/opt/homebrew/opt/openjdk@17"
+            environment["MAVEN_RUNNER"] = str(runner)
+
+            result = subprocess.run(
+                [str(SCRIPT), str(source_root), str(repository)],
+                cwd=ROOT,
+                env=environment,
+                text=True,
+                capture_output=True,
+                check=False,
+            )
+
+            self.assertEqual(result.returncode, 0, result.stderr)
+            installed = repository / "com/sportsbook/shared-protocol/1.0.0"
+            self.assertEqual(
+                {path.name for path in installed.iterdir()},
+                {"shared-protocol-1.0.0.jar", "shared-protocol-1.0.0.pom"},
+            )
+            self.assertEqual(
+                (source / "target/shared-protocol-1.0.0.jar").read_bytes(),
+                (installed / "shared-protocol-1.0.0.jar").read_bytes(),
+            )
+            subprocess.run(
+                ["git", "worktree", "remove", "--force", str(source)],
+                cwd=ROOT,
+                check=True,
+            )
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(jars): stage exact release artifacts atomically`

diff --git a/scripts/stage-release-jars.sh b/scripts/stage-release-jars.sh
new file mode 100755
index 0000000..95fbe07
--- /dev/null
+++ b/scripts/stage-release-jars.sh
@@ -0,0 +1,80 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT=$(git rev-parse --show-toplevel)
+SOURCE_ROOT=${1:-$ROOT/.runtime/sources}
+MAVEN_REPO=${2:-$ROOT/.runtime/m2/repository}
+LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
+DOCKER_DIR=${DOCKER_OUTPUT_ROOT:-$ROOT/docker}
+GENERATIONS=$DOCKER_DIR/.jars
+JARS=$DOCKER_DIR/jars
+STAGING=
+LINK_TMP=
+
+fail() {
+  printf 'jar-stage: %s\n' "$1" >&2
+  exit 1
+}
+
+cleanup() {
+  local status=$?
+  [[ -z $LINK_TMP || ! -L $LINK_TMP ]] || rm -f "$LINK_TMP"
+  [[ -z $STAGING || ! -d $STAGING ]] || rm -rf -- "$STAGING"
+  exit "$status"
+}
+trap cleanup EXIT INT TERM
+
+[[ -d $SOURCE_ROOT && ! -L $SOURCE_ROOT ]] || fail "source root is not materialized"
+[[ -d $MAVEN_REPO && ! -L $MAVEN_REPO ]] || fail "isolated Maven repository is missing"
+[[ ! -L $GENERATIONS ]] || fail "generation root must not be a symlink"
+mkdir -p "$GENERATIONS"
+STAGING=$(mktemp -d "$GENERATIONS/generation.XXXXXX")
+
+count=0
+while IFS='|' read -r logical branch commit artifact; do
+  [[ $logical != shared ]] || continue
+  source=$SOURCE_ROOT/$logical
+  [[ -f $source/.git && $(git -C "$source" rev-parse HEAD) == "$commit" ]] \
+    || fail "$logical source mismatch"
+  ! git -C "$source" symbolic-ref -q HEAD >/dev/null || fail "$logical source is attached"
+  [[ -z $(git -C "$source" status --porcelain) ]] || fail "$logical source is dirty"
+  runner=${MAVEN_RUNNER:-$source/mvnw}
+  [[ -x $runner ]] || fail "$logical Maven runner is not executable"
+  (
+    cd "$source"
+    "$runner" -B "-Dmaven.repo.local=$MAVEN_REPO" -DskipTests clean package
+  )
+  built=$source/target/$artifact
+  [[ -f $built && ! -L $built ]] || fail "$logical exact release JAR is missing"
+  jar tf "$built" | grep -q '^BOOT-INF/classes/' || fail "$logical JAR is not executable"
+  cp "$built" "$STAGING/$logical.jar"
+  hash=$(shasum -a 256 "$STAGING/$logical.jar" | awk '{print $1}')
+  printf '%s  %s.jar\n' "$hash" "$logical" >>"$STAGING/SHA256SUMS"
+  count=$((count + 1))
+done < <(awk -F'|' 'NF && $1 !~ /^#/ { if (NF != 4) exit 2; print }' "$LOCK")
+[[ $count == 7 && $(wc -l <"$STAGING/SHA256SUMS" | tr -d ' ') == 7 ]] \
+  || fail "release JAR set is incomplete"
+
+old_target=
+if [[ -e $JARS && ! -L $JARS ]]; then
+  fail "docker/jars must be absent or a managed symlink"
+elif [[ -L $JARS ]]; then
+  old_target=$(readlink "$JARS")
+  [[ $old_target =~ ^\.jars/generation\.[A-Za-z0-9]+$ ]] \
+    || fail "docker/jars points outside managed generations"
+  [[ -d $DOCKER_DIR/$old_target && ! -L $DOCKER_DIR/$old_target ]] \
+    || fail "active generation is invalid"
+fi
+
+generation=$(basename "$STAGING")
+LINK_TMP=$GENERATIONS/link.$$
+ln -s ".jars/$generation" "$LINK_TMP"
+case $(uname -s) in
+  Darwin) mv -hf "$LINK_TMP" "$JARS" ;;
+  Linux) mv -Tf "$LINK_TMP" "$JARS" ;;
+  *) fail "atomic publication is unsupported" ;;
+esac
+LINK_TMP=
+STAGING=
+[[ -z $old_target ]] || rm -rf -- "$DOCKER_DIR/$old_target"
+trap - EXIT INT TERM


## `test(jars): verify complete release generation`

diff --git a/tests/test_jar_staging_completeness.py b/tests/test_jar_staging_completeness.py
new file mode 100644
index 0000000..70db884
--- /dev/null
+++ b/tests/test_jar_staging_completeness.py
@@ -0,0 +1,40 @@
+import hashlib
+import zipfile
+
+from tests.staging_fixture import StagingFixture
+
+
+SERVICES = {"wallet", "risk", "odds", "betting", "gateway", "settlement", "admin"}
+
+
+class JarStagingCompletenessTest(StagingFixture):
+    def test_publishes_exactly_one_complete_generation(self) -> None:
+        result = self.stage()
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        active = self.active_generation()
+        self.assertTrue((self.docker / "jars").is_symlink())
+        self.assertEqual(
+            {path.name for path in active.iterdir()},
+            {*(f"{service}.jar" for service in SERVICES), "SHA256SUMS"},
+        )
+
+        expected_sums = []
+        for service in sorted(SERVICES):
+            jar = active / f"{service}.jar"
+            with zipfile.ZipFile(jar) as archive:
+                self.assertEqual(
+                    archive.read("BOOT-INF/classes/Probe.class"), service.encode()
+                )
+            expected_sums.append(f"{hashlib.sha256(jar.read_bytes()).hexdigest()}  {jar.name}")
+
+        self.assertCountEqual(
+            (active / "SHA256SUMS").read_text().splitlines(), expected_sums
+        )
+        self.assertEqual(len(list((self.docker / ".jars").iterdir())), 1)
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `test(jars): preserve atomic generation on failure`

diff --git a/tests/test_jar_staging_atomicity.py b/tests/test_jar_staging_atomicity.py
new file mode 100644
index 0000000..8cb4ec2
--- /dev/null
+++ b/tests/test_jar_staging_atomicity.py
@@ -0,0 +1,34 @@
+import hashlib
+
+from tests.staging_fixture import StagingFixture
+
+
+class JarStagingAtomicityTest(StagingFixture):
+    def test_failed_rebuild_preserves_the_active_generation(self) -> None:
+        first = self.stage()
+        self.assertEqual(first.returncode, 0, first.stderr)
+        active = self.active_generation()
+        link_target = (self.docker / "jars").readlink()
+        before = {
+            path.name: hashlib.sha256(path.read_bytes()).hexdigest()
+            for path in active.iterdir()
+        }
+
+        failed = self.stage(FAIL_LOGICAL="risk")
+
+        self.assertNotEqual(failed.returncode, 0)
+        self.assertEqual((self.docker / "jars").readlink(), link_target)
+        self.assertEqual(
+            {
+                path.name: hashlib.sha256(path.read_bytes()).hexdigest()
+                for path in active.iterdir()
+            },
+            before,
+        )
+        self.assertEqual(list((self.docker / ".jars").iterdir()), [active])
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(shared): canonicalize release inputs`

diff --git a/scripts/install-shared.sh b/scripts/install-shared.sh
index 202ab21..5948331 100755
--- a/scripts/install-shared.sh
+++ b/scripts/install-shared.sh
@@ -1,7 +1,8 @@
 #!/usr/bin/env bash
 set -euo pipefail
 
-ROOT=$(git rev-parse --show-toplevel)
+SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)
+ROOT=$(git -C "$SCRIPT_DIR" rev-parse --show-toplevel)
 SOURCE_ROOT=${1:-$ROOT/.runtime/sources}
 MAVEN_REPO=${2:-$ROOT/.runtime/m2/repository}
 LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
@@ -18,6 +19,8 @@ IFS='|' read -r logical branch commit artifact < <(
 [[ $logical == shared && $artifact == shared-protocol-1.0.0.jar ]] \
   || fail "invalid shared lock entry"
 
+[[ -d $SOURCE_ROOT && ! -L $SOURCE_ROOT ]] || fail "source root is not materialized"
+SOURCE_ROOT=$(cd "$SOURCE_ROOT" && pwd -P)
 SOURCE=$SOURCE_ROOT/shared
 [[ -f $SOURCE/.git && ! -L $SOURCE ]] || fail "shared source is not a detached worktree"
 [[ $(git -C "$SOURCE" rev-parse HEAD) == "$commit" ]] || fail "shared source SHA mismatch"
@@ -33,7 +36,12 @@ MAVEN_REPO=$(cd "$MAVEN_REPO" && pwd -P)
 [[ $MAVEN_REPO != / && $MAVEN_REPO != "$ROOT" ]] || fail "unsafe Maven repository"
 [[ ! -L $MAVEN_REPO ]] || fail "Maven repository must not be a symlink"
 
-[[ -n $RUNNER ]] || RUNNER=$SOURCE/mvnw
+if [[ -n $RUNNER ]]; then
+  [[ -x $RUNNER ]] || fail "Maven runner is not executable"
+  RUNNER=$(cd "$(dirname "$RUNNER")" && pwd -P)/$(basename "$RUNNER")
+else
+  RUNNER=$SOURCE/mvnw
+fi
 [[ -x $RUNNER ]] || fail "Maven runner is not executable"
 (
   cd "$SOURCE"


## `build(shared): require a complete Java 17 JDK`

diff --git a/scripts/install-shared.sh b/scripts/install-shared.sh
index 5948331..18d7573 100755
--- a/scripts/install-shared.sh
+++ b/scripts/install-shared.sh
@@ -28,8 +28,10 @@ SOURCE=$SOURCE_ROOT/shared
 [[ -z $(git -C "$SOURCE" status --porcelain) ]] || fail "shared source is dirty"
 
 JAVA_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}java
+JAVAC_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}javac
 JAVA_MAJOR=$($JAVA_BIN -version 2>&1 | awk -F'[."]' '/version/ {print $2; exit}')
-[[ $JAVA_MAJOR == 17 ]] || fail "Java 17 is required"
+JAVAC_MAJOR=$($JAVAC_BIN -version 2>&1 | awk '{print $2; exit}' | cut -d. -f1)
+[[ $JAVA_MAJOR == 17 && $JAVAC_MAJOR == 17 ]] || fail "Java 17 JDK is required"
 
 mkdir -p "$MAVEN_REPO"
 MAVEN_REPO=$(cd "$MAVEN_REPO" && pwd -P)


## `test(shared): verify Java 17 compiler preflight`

diff --git a/tests/test_shared_install.py b/tests/test_shared_install.py
index e9eb2cb..c172a45 100644
--- a/tests/test_shared_install.py
+++ b/tests/test_shared_install.py
@@ -66,6 +66,9 @@ class SharedInstallTest(unittest.TestCase):
             java.parent.mkdir(parents=True)
             java.write_text('#!/bin/sh\nprintf \'openjdk version "17.0.0"\\n\' >&2\n')
             java.chmod(0o755)
+            javac = java.parent / "javac"
+            javac.write_text("#!/bin/sh\nprintf 'javac 17.0.0\\n' >&2\n")
+            javac.chmod(0o755)
             jar = java.parent / "jar"
             jar.write_text(
                 "#!/bin/sh\nprintf 'com/sportsbook/protocol/value/Money.class\\n'\n"


## `build(jars): canonicalize staging inputs`

diff --git a/scripts/stage-release-jars.sh b/scripts/stage-release-jars.sh
index 95fbe07..9a6a09d 100755
--- a/scripts/stage-release-jars.sh
+++ b/scripts/stage-release-jars.sh
@@ -1,13 +1,12 @@
 #!/usr/bin/env bash
 set -euo pipefail
 
-ROOT=$(git rev-parse --show-toplevel)
+SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)
+ROOT=$(git -C "$SCRIPT_DIR" rev-parse --show-toplevel)
 SOURCE_ROOT=${1:-$ROOT/.runtime/sources}
 MAVEN_REPO=${2:-$ROOT/.runtime/m2/repository}
 LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
 DOCKER_DIR=${DOCKER_OUTPUT_ROOT:-$ROOT/docker}
-GENERATIONS=$DOCKER_DIR/.jars
-JARS=$DOCKER_DIR/jars
 STAGING=
 LINK_TMP=
 
@@ -25,7 +24,17 @@ cleanup() {
 trap cleanup EXIT INT TERM
 
 [[ -d $SOURCE_ROOT && ! -L $SOURCE_ROOT ]] || fail "source root is not materialized"
+SOURCE_ROOT=$(cd "$SOURCE_ROOT" && pwd -P)
 [[ -d $MAVEN_REPO && ! -L $MAVEN_REPO ]] || fail "isolated Maven repository is missing"
+MAVEN_REPO=$(cd "$MAVEN_REPO" && pwd -P)
+mkdir -p "$(dirname "$DOCKER_DIR")"
+DOCKER_DIR=$(cd "$(dirname "$DOCKER_DIR")" && pwd -P)/$(basename "$DOCKER_DIR")
+GENERATIONS=$DOCKER_DIR/.jars
+JARS=$DOCKER_DIR/jars
+if [[ -n ${MAVEN_RUNNER:-} ]]; then
+  [[ -x $MAVEN_RUNNER ]] || fail "Maven runner is not executable"
+  MAVEN_RUNNER=$(cd "$(dirname "$MAVEN_RUNNER")" && pwd -P)/$(basename "$MAVEN_RUNNER")
+fi
 [[ ! -L $GENERATIONS ]] || fail "generation root must not be a symlink"
 mkdir -p "$GENERATIONS"
 STAGING=$(mktemp -d "$GENERATIONS/generation.XXXXXX")


## `build(jars): require the release Java 17 JDK`

diff --git a/scripts/stage-release-jars.sh b/scripts/stage-release-jars.sh
index 9a6a09d..b2009c8 100755
--- a/scripts/stage-release-jars.sh
+++ b/scripts/stage-release-jars.sh
@@ -35,6 +35,11 @@ if [[ -n ${MAVEN_RUNNER:-} ]]; then
   [[ -x $MAVEN_RUNNER ]] || fail "Maven runner is not executable"
   MAVEN_RUNNER=$(cd "$(dirname "$MAVEN_RUNNER")" && pwd -P)/$(basename "$MAVEN_RUNNER")
 fi
+JAVA_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}java
+JAVAC_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}javac
+JAVA_MAJOR=$($JAVA_BIN -version 2>&1 | awk -F'[."]' '/version/ {print $2; exit}')
+JAVAC_MAJOR=$($JAVAC_BIN -version 2>&1 | awk '{print $2; exit}' | cut -d. -f1)
+[[ $JAVA_MAJOR == 17 && $JAVAC_MAJOR == 17 ]] || fail "Java 17 JDK is required"
 [[ ! -L $GENERATIONS ]] || fail "generation root must not be a symlink"
 mkdir -p "$GENERATIONS"
 STAGING=$(mktemp -d "$GENERATIONS/generation.XXXXXX")


## `test(jars): reject non-release JDK staging`

diff --git a/tests/staging_fixture.py b/tests/staging_fixture.py
index d54b8b6..b1584fb 100644
--- a/tests/staging_fixture.py
+++ b/tests/staging_fixture.py
@@ -18,6 +18,7 @@ class StagingFixture(unittest.TestCase):
         self.sources = self.temporary_path / "sources"
         self.repository = self.temporary_path / "m2"
         self.docker = self.temporary_path / "docker"
+        self.jdk = self.temporary_path / "jdk"
         self.repository.mkdir()
         materialized = subprocess.run(
             [str(MATERIALIZER), str(self.sources)],
@@ -28,6 +29,18 @@ class StagingFixture(unittest.TestCase):
         )
         self.assertEqual(materialized.returncode, 0, materialized.stderr)
         self.environment = os.environ.copy()
+        (self.jdk / "bin").mkdir(parents=True)
+        for name, output in (
+            ("java", 'openjdk version "17.0.0"'),
+            ("javac", "javac 17.0.0"),
+            ("jar", "BOOT-INF/classes/Probe.class"),
+        ):
+            executable = self.jdk / "bin" / name
+            redirect = "" if name == "jar" else " >&2"
+            executable.write_text(f"#!/bin/sh\nprintf '{output}\\n'{redirect}\n")
+            executable.chmod(0o755)
+        self.environment["JAVA_HOME"] = str(self.jdk)
+        self.environment["PATH"] = f"{self.jdk / 'bin'}:{self.environment['PATH']}"
         self.environment["MAVEN_RUNNER"] = str(FAKE_MAVEN)
         self.environment["DOCKER_OUTPUT_ROOT"] = self.docker.name
 
diff --git a/tests/test_jar_staging_completeness.py b/tests/test_jar_staging_completeness.py
index 70db884..82e151c 100644
--- a/tests/test_jar_staging_completeness.py
+++ b/tests/test_jar_staging_completeness.py
@@ -33,6 +33,17 @@ class JarStagingCompletenessTest(StagingFixture):
         )
         self.assertEqual(len(list((self.docker / ".jars").iterdir())), 1)
 
+    def test_rejects_non_release_jdk_before_publication(self) -> None:
+        java = self.jdk / "bin/java"
+        java.write_text('#!/bin/sh\nprintf \'openjdk version "21.0.0"\\n\' >&2\n')
+        java.chmod(0o755)
+
+        result = self.stage()
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("Java 17 JDK is required", result.stderr)
+        self.assertFalse((self.docker / "jars").exists())
+
 
 if __name__ == "__main__":
     import unittest


