# 릴리스 입력 잠금과 소스 구체화

## `build(lock): pin service release inputs`

diff --git a/services.lock b/services.lock
new file mode 100644
index 0000000..733f842
--- /dev/null
+++ b/services.lock
@@ -0,0 +1,9 @@
+# logical-name|branch|commit|artifact
+shared|shared-protocol|f9de6bc1e533761ab4bb1454d8d4ab8175cdf001|shared-protocol-1.0.0.jar
+wallet|wallet-service|c9a05f4d652f24ac97d3e1cd753f69cef2725ff3|wallet-service-1.0.0.jar
+risk|risk-service|c64f67dbc437a18640dc4984dea4d8194fb5b164|risk-service-1.0.0.jar
+odds|odds-feed-service|574e83d2862f086ae07ff56fd95a8336f78a72da|odds-feed-service-1.0.0.jar
+betting|betting-service|f712bdf389ee3fb63d8cdc84c49e2b84a346edde|betting-service-1.0.0.jar
+gateway|gateway|8248a3233f0fce7ca36a503ee71b7a8a0802d733|gateway-1.0.0.jar
+settlement|settlement-service|fc53ee8bfbb99b083f504d414d84ae5a994e4b57|settlement-service-1.0.0.jar
+admin|admin-api|2fb55910475b31084e6489bf01c34cc970c96874|admin-api-1.0.0.jar


## `test(lock): verify exact service SHAs`

diff --git a/tests/test_services_lock.py b/tests/test_services_lock.py
new file mode 100644
index 0000000..695c501
--- /dev/null
+++ b/tests/test_services_lock.py
@@ -0,0 +1,49 @@
+import pathlib
+import re
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+EXPECTED = {
+    "shared": ("shared-protocol", "shared-protocol-1.0.0.jar"),
+    "wallet": ("wallet-service", "wallet-service-1.0.0.jar"),
+    "risk": ("risk-service", "risk-service-1.0.0.jar"),
+    "odds": ("odds-feed-service", "odds-feed-service-1.0.0.jar"),
+    "betting": ("betting-service", "betting-service-1.0.0.jar"),
+    "gateway": ("gateway", "gateway-1.0.0.jar"),
+    "settlement": ("settlement-service", "settlement-service-1.0.0.jar"),
+    "admin": ("admin-api", "admin-api-1.0.0.jar"),
+}
+
+
+def entries() -> list[tuple[str, str, str, str]]:
+    lines = (ROOT / "services.lock").read_text().splitlines()
+    return [tuple(line.split("|")) for line in lines if line and not line.startswith("#")]
+
+
+class ServicesLockTest(unittest.TestCase):
+    def test_pins_every_release_branch_to_a_full_commit(self) -> None:
+        locked = entries()
+        self.assertEqual([entry[0] for entry in locked], list(EXPECTED))
+        self.assertEqual(len({entry[2] for entry in locked}), len(locked))
+
+        for logical, branch, commit, artifact in locked:
+            with self.subTest(service=logical):
+                self.assertEqual((branch, artifact), EXPECTED[logical])
+                self.assertRegex(commit, re.compile(r"^[0-9a-f]{40}$"))
+                object_type = subprocess.check_output(
+                    ["git", "cat-file", "-t", commit], cwd=ROOT, text=True
+                ).strip()
+                self.assertEqual(object_type, "commit")
+                branch_tip = subprocess.check_output(
+                    ["git", "rev-parse", f"refs/heads/{branch}"], cwd=ROOT, text=True
+                ).strip()
+                self.assertEqual(branch_tip, commit)
+
+    def test_does_not_create_an_orchestration_lock_cycle(self) -> None:
+        self.assertNotIn("orchestration", {entry[0] for entry in entries()})
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(source): materialize detached release worktrees`

diff --git a/scripts/materialize-sources.sh b/scripts/materialize-sources.sh
new file mode 100755
index 0000000..0f0701d
--- /dev/null
+++ b/scripts/materialize-sources.sh
@@ -0,0 +1,80 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT=$(git rev-parse --show-toplevel)
+LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
+TARGET=${1:-$ROOT/.runtime/sources}
+MODE=${2:-materialize}
+CREATED=()
+
+fail() {
+  printf 'materialize: %s\n' "$1" >&2
+  exit 1
+}
+
+absolute_target() {
+  local parent base
+  parent=$(dirname "$TARGET")
+  base=$(basename "$TARGET")
+  mkdir -p "$parent"
+  parent=$(cd "$parent" && pwd -P)
+  printf '%s/%s\n' "$parent" "$base"
+}
+
+validate_target() {
+  TARGET=$(absolute_target)
+  [[ $TARGET != / && $TARGET != "$ROOT" && $TARGET != "$(dirname "$ROOT")" ]] \
+    || fail "refusing broad target: $TARGET"
+  [[ ! -L $TARGET ]] || fail "target must not be a symlink"
+}
+
+locked_entries() {
+  awk -F'|' 'NF && $1 !~ /^#/ { if (NF != 4) exit 2; print }' "$LOCK"
+}
+
+cleanup_created() {
+  local path
+  for ((index=${#CREATED[@]} - 1; index >= 0; index--)); do
+    path=${CREATED[$index]}
+    git -C "$ROOT" worktree remove --force "$path" >/dev/null 2>&1 || true
+  done
+  [[ ! -d $TARGET ]] || rmdir "$TARGET" >/dev/null 2>&1 || true
+}
+
+materialize() {
+  local logical branch commit artifact path
+  [[ ! -e $TARGET ]] || fail "target already exists: $TARGET"
+  mkdir "$TARGET"
+  trap cleanup_created EXIT ERR INT TERM
+  while IFS='|' read -r logical branch commit artifact; do
+    git -C "$ROOT" cat-file -e "$commit^{commit}"
+    [[ $(git -C "$ROOT" rev-parse "refs/heads/$branch") == "$commit" ]] \
+      || fail "$branch no longer matches $commit"
+    path=$TARGET/$logical
+    git -C "$ROOT" worktree add --quiet --detach "$path" "$commit"
+    CREATED+=("$path")
+    [[ $(git -C "$path" rev-parse HEAD) == "$commit" ]] || fail "$logical checkout mismatch"
+    ! git -C "$path" symbolic-ref -q HEAD >/dev/null || fail "$logical is not detached"
+  done < <(locked_entries)
+  trap - EXIT ERR INT TERM
+}
+
+cleanup() {
+  local logical branch commit artifact path
+  [[ -d $TARGET && ! -L $TARGET ]] || fail "cleanup target is not a directory"
+  while IFS='|' read -r logical branch commit artifact; do
+    path=$TARGET/$logical
+    [[ -f $path/.git ]] || fail "unmanaged worktree: $path"
+    [[ $(git -C "$path" rev-parse HEAD) == "$commit" ]] || fail "changed worktree: $path"
+    ! git -C "$path" symbolic-ref -q HEAD >/dev/null || fail "attached worktree: $path"
+    git -C "$ROOT" worktree remove --force "$path"
+  done < <(locked_entries)
+  rmdir "$TARGET"
+}
+
+validate_target
+case "$MODE" in
+  materialize) materialize ;;
+  cleanup) cleanup ;;
+  *) fail "mode must be materialize or cleanup" ;;
+esac


## `test(source): verify detached cleanup safety`

diff --git a/tests/test_materializer.py b/tests/test_materializer.py
new file mode 100644
index 0000000..e3df0cd
--- /dev/null
+++ b/tests/test_materializer.py
@@ -0,0 +1,71 @@
+import os
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SCRIPT = ROOT / "scripts/materialize-sources.sh"
+
+
+def locked_entries() -> list[tuple[str, str, str, str]]:
+    lines = (ROOT / "services.lock").read_text().splitlines()
+    return [tuple(line.split("|")) for line in lines if line and not line.startswith("#")]
+
+
+class MaterializerTest(unittest.TestCase):
+    def run_script(
+        self, target: pathlib.Path, mode: str = "materialize", lock: pathlib.Path | None = None
+    ) -> subprocess.CompletedProcess[str]:
+        environment = os.environ.copy()
+        if lock is not None:
+            environment["SERVICES_LOCK"] = str(lock)
+        return subprocess.run(
+            [str(SCRIPT), str(target), mode],
+            cwd=ROOT,
+            env=environment,
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+
+    def test_materializes_exact_detached_commits_and_cleans_them(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            target = pathlib.Path(temporary) / "sources"
+            self.assertEqual(self.run_script(target).returncode, 0)
+
+            for logical, _branch, commit, _artifact in locked_entries():
+                with self.subTest(service=logical):
+                    source = target / logical
+                    self.assertTrue((source / ".git").is_file())
+                    head = subprocess.check_output(
+                        ["git", "rev-parse", "HEAD"], cwd=source, text=True
+                    ).strip()
+                    self.assertEqual(head, commit)
+                    attached = subprocess.run(
+                        ["git", "symbolic-ref", "-q", "HEAD"], cwd=source, check=False
+                    )
+                    self.assertNotEqual(attached.returncode, 0)
+
+            self.assertEqual(self.run_script(target, "cleanup").returncode, 0)
+            self.assertFalse(target.exists())
+
+    def test_removes_partial_worktrees_after_a_lock_failure(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = pathlib.Path(temporary)
+            target = temporary_path / "sources"
+            valid = (ROOT / "services.lock").read_text().splitlines()[1]
+            bad_lock = temporary_path / "bad.lock"
+            bad_lock.write_text(
+                valid
+                + "\n"
+                + "risk|risk-service|0000000000000000000000000000000000000000|risk-service-1.0.0.jar\n"
+            )
+
+            self.assertNotEqual(self.run_script(target, lock=bad_lock).returncode, 0)
+            self.assertFalse(target.exists())
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(source): resolve repository from script path`

diff --git a/scripts/materialize-sources.sh b/scripts/materialize-sources.sh
index 0f0701d..c50b041 100755
--- a/scripts/materialize-sources.sh
+++ b/scripts/materialize-sources.sh
@@ -1,7 +1,8 @@
 #!/usr/bin/env bash
 set -euo pipefail
 
-ROOT=$(git rev-parse --show-toplevel)
+SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)
+ROOT=$(git -C "$SCRIPT_DIR" rev-parse --show-toplevel)
 LOCK=${SERVICES_LOCK:-$ROOT/services.lock}
 TARGET=${1:-$ROOT/.runtime/sources}
 MODE=${2:-materialize}


## `test(source): invoke materializer outside repository`

diff --git a/tests/test_materializer.py b/tests/test_materializer.py
index e3df0cd..1c6a29f 100644
--- a/tests/test_materializer.py
+++ b/tests/test_materializer.py
@@ -23,7 +23,7 @@ class MaterializerTest(unittest.TestCase):
             environment["SERVICES_LOCK"] = str(lock)
         return subprocess.run(
             [str(SCRIPT), str(target), mode],
-            cwd=ROOT,
+            cwd=target.parent,
             env=environment,
             text=True,
             capture_output=True,


## `build(source): preflight cleanup ownership`

diff --git a/scripts/materialize-sources.sh b/scripts/materialize-sources.sh
index c50b041..89bda98 100755
--- a/scripts/materialize-sources.sh
+++ b/scripts/materialize-sources.sh
@@ -62,14 +62,19 @@ materialize() {
 
 cleanup() {
   local logical branch commit artifact path
+  local -a paths=()
   [[ -d $TARGET && ! -L $TARGET ]] || fail "cleanup target is not a directory"
   while IFS='|' read -r logical branch commit artifact; do
     path=$TARGET/$logical
     [[ -f $path/.git ]] || fail "unmanaged worktree: $path"
     [[ $(git -C "$path" rev-parse HEAD) == "$commit" ]] || fail "changed worktree: $path"
     ! git -C "$path" symbolic-ref -q HEAD >/dev/null || fail "attached worktree: $path"
-    git -C "$ROOT" worktree remove --force "$path"
+    [[ -z $(git -C "$path" status --porcelain) ]] || fail "dirty worktree: $path"
+    paths+=("$path")
   done < <(locked_entries)
+  for path in "${paths[@]}"; do
+    git -C "$ROOT" worktree remove --force "$path"
+  done
   rmdir "$TARGET"
 }
 


## `test(source): preserve worktrees on cleanup mismatch`

diff --git a/tests/test_materializer.py b/tests/test_materializer.py
index 1c6a29f..d9d5eb6 100644
--- a/tests/test_materializer.py
+++ b/tests/test_materializer.py
@@ -66,6 +66,30 @@ class MaterializerTest(unittest.TestCase):
             self.assertNotEqual(self.run_script(target, lock=bad_lock).returncode, 0)
             self.assertFalse(target.exists())
 
+    def test_preflights_every_worktree_before_cleanup_removes_any(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            target = pathlib.Path(temporary) / "sources"
+            self.assertEqual(self.run_script(target).returncode, 0)
+            entries = locked_entries()
+            last_name, _branch, last_commit, _artifact = entries[-1]
+            last_source = target / last_name
+            subprocess.run(
+                ["git", "checkout", "--quiet", "--detach", "HEAD^"],
+                cwd=last_source,
+                check=True,
+            )
+
+            failed = self.run_script(target, "cleanup")
+
+            self.assertNotEqual(failed.returncode, 0)
+            self.assertTrue(all((target / entry[0]).is_dir() for entry in entries))
+            subprocess.run(
+                ["git", "checkout", "--quiet", "--detach", last_commit],
+                cwd=last_source,
+                check=True,
+            )
+            self.assertEqual(self.run_script(target, "cleanup").returncode, 0)
+
 
 if __name__ == "__main__":
     unittest.main()


## `fix(lock): propagate malformed release manifests`

diff --git a/scripts/materialize-sources.sh b/scripts/materialize-sources.sh
index 89bda98..06c8b1f 100755
--- a/scripts/materialize-sources.sh
+++ b/scripts/materialize-sources.sh
@@ -30,7 +30,10 @@ validate_target() {
 }
 
 locked_entries() {
-  awk -F'|' 'NF && $1 !~ /^#/ { if (NF != 4) exit 2; print }' "$LOCK"
+  awk -F'|' '
+    NF && $1 !~ /^#/ { if (NF != 4) exit 2; print; count++ }
+    END { if (count != 8) exit 2 }
+  ' "$LOCK"
 }
 
 cleanup_created() {
@@ -56,7 +59,7 @@ materialize() {
     CREATED+=("$path")
     [[ $(git -C "$path" rev-parse HEAD) == "$commit" ]] || fail "$logical checkout mismatch"
     ! git -C "$path" symbolic-ref -q HEAD >/dev/null || fail "$logical is not detached"
-  done < <(locked_entries)
+  done <<<"$ENTRIES"
   trap - EXIT ERR INT TERM
 }
 
@@ -71,7 +74,7 @@ cleanup() {
     ! git -C "$path" symbolic-ref -q HEAD >/dev/null || fail "attached worktree: $path"
     [[ -z $(git -C "$path" status --porcelain) ]] || fail "dirty worktree: $path"
     paths+=("$path")
-  done < <(locked_entries)
+  done <<<"$ENTRIES"
   for path in "${paths[@]}"; do
     git -C "$ROOT" worktree remove --force "$path"
   done
@@ -79,6 +82,7 @@ cleanup() {
 }
 
 validate_target
+ENTRIES=$(locked_entries) || fail "invalid services lock"
 case "$MODE" in
   materialize) materialize ;;
   cleanup) cleanup ;;
diff --git a/scripts/stage-release-jars.sh b/scripts/stage-release-jars.sh
index b2009c8..522e9f1 100755
--- a/scripts/stage-release-jars.sh
+++ b/scripts/stage-release-jars.sh
@@ -43,6 +43,10 @@ JAVAC_MAJOR=$($JAVAC_BIN -version 2>&1 | awk '{print $2; exit}' | cut -d. -f1)
 [[ ! -L $GENERATIONS ]] || fail "generation root must not be a symlink"
 mkdir -p "$GENERATIONS"
 STAGING=$(mktemp -d "$GENERATIONS/generation.XXXXXX")
+ENTRIES=$(awk -F'|' '
+  NF && $1 !~ /^#/ { if (NF != 4) exit 2; print; count++ }
+  END { if (count != 8) exit 2 }
+' "$LOCK") || fail "invalid services lock"
 
 count=0
 while IFS='|' read -r logical branch commit artifact; do
@@ -65,7 +69,7 @@ while IFS='|' read -r logical branch commit artifact; do
   hash=$(shasum -a 256 "$STAGING/$logical.jar" | awk '{print $1}')
   printf '%s  %s.jar\n' "$hash" "$logical" >>"$STAGING/SHA256SUMS"
   count=$((count + 1))
-done < <(awk -F'|' 'NF && $1 !~ /^#/ { if (NF != 4) exit 2; print }' "$LOCK")
+done <<<"$ENTRIES"
 [[ $count == 7 && $(wc -l <"$STAGING/SHA256SUMS" | tr -d ' ') == 7 ]] \
   || fail "release JAR set is incomplete"
 


## `test(lock): reject partial manifest consumption`

diff --git a/tests/test_materializer.py b/tests/test_materializer.py
index d9d5eb6..a1665fc 100644
--- a/tests/test_materializer.py
+++ b/tests/test_materializer.py
@@ -63,7 +63,9 @@ class MaterializerTest(unittest.TestCase):
                 + "risk|risk-service|0000000000000000000000000000000000000000|risk-service-1.0.0.jar\n"
             )
 
-            self.assertNotEqual(self.run_script(target, lock=bad_lock).returncode, 0)
+            failed = self.run_script(target, lock=bad_lock)
+            self.assertNotEqual(failed.returncode, 0)
+            self.assertIn("invalid services lock", failed.stderr)
             self.assertFalse(target.exists())
 
     def test_preflights_every_worktree_before_cleanup_removes_any(self) -> None:
diff --git a/tests/test_release_lock_fail_closed.py b/tests/test_release_lock_fail_closed.py
new file mode 100644
index 0000000..095d49c
--- /dev/null
+++ b/tests/test_release_lock_fail_closed.py
@@ -0,0 +1,27 @@
+import pathlib
+
+from tests.staging_fixture import StagingFixture
+
+
+class ReleaseLockFailClosedTest(StagingFixture):
+    def test_rejects_partial_manifest_before_building_services(self) -> None:
+        lock = self.temporary_path / "partial.lock"
+        lock.write_text(
+            "shared|shared-protocol|"
+            "f9de6bc1e533761ab4bb1454d8d4ab8175cdf001|shared-protocol-1.0.0.jar\n"
+        )
+
+        result = self.stage(SERVICES_LOCK=str(lock))
+
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("invalid services lock", result.stderr)
+        self.assertFalse((self.docker / "jars").exists())
+        for source in self.sources.iterdir():
+            if source.name != "shared":
+                self.assertFalse((source / "target").exists())
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `fix(source): resolve archive remote refs`

diff --git a/scripts/materialize-sources.sh b/scripts/materialize-sources.sh
index 06c8b1f..6da059b 100755
--- a/scripts/materialize-sources.sh
+++ b/scripts/materialize-sources.sh
@@ -36,6 +36,19 @@ locked_entries() {
   ' "$LOCK"
 }
 
+locked_branch_ref() {
+  local branch=$1
+  local local_ref=refs/heads/$branch
+  local remote_ref=refs/remotes/origin/$branch
+  if git -C "$ROOT" show-ref --verify --quiet "$local_ref"; then
+    printf '%s\n' "$local_ref"
+  elif git -C "$ROOT" show-ref --verify --quiet "$remote_ref"; then
+    printf '%s\n' "$remote_ref"
+  else
+    fail "$branch ref is unavailable"
+  fi
+}
+
 cleanup_created() {
   local path
   for ((index=${#CREATED[@]} - 1; index >= 0; index--)); do
@@ -46,13 +59,14 @@ cleanup_created() {
 }
 
 materialize() {
-  local logical branch commit artifact path
+  local logical branch commit artifact path branch_ref
   [[ ! -e $TARGET ]] || fail "target already exists: $TARGET"
   mkdir "$TARGET"
   trap cleanup_created EXIT ERR INT TERM
   while IFS='|' read -r logical branch commit artifact; do
     git -C "$ROOT" cat-file -e "$commit^{commit}"
-    [[ $(git -C "$ROOT" rev-parse "refs/heads/$branch") == "$commit" ]] \
+    branch_ref=$(locked_branch_ref "$branch")
+    [[ $(git -C "$ROOT" rev-parse "$branch_ref") == "$commit" ]] \
       || fail "$branch no longer matches $commit"
     path=$TARGET/$logical
     git -C "$ROOT" worktree add --quiet --detach "$path" "$commit"


## `test(source): materialize remote-only archive refs`

diff --git a/tests/test_materializer_remote_refs.py b/tests/test_materializer_remote_refs.py
new file mode 100644
index 0000000..127f291
--- /dev/null
+++ b/tests/test_materializer_remote_refs.py
@@ -0,0 +1,99 @@
+import os
+import pathlib
+import shutil
+import subprocess
+import tempfile
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SOURCE_SCRIPT = ROOT / "scripts/materialize-sources.sh"
+
+
+class MaterializerRemoteRefsTest(unittest.TestCase):
+    def fixture(self, parent: pathlib.Path) -> tuple[pathlib.Path, pathlib.Path, str, str]:
+        repository = parent / "repository"
+        repository.mkdir()
+        subprocess.run(["git", "init", "--quiet"], cwd=repository, check=True)
+        subprocess.run(["git", "config", "user.name", "Fixture"], cwd=repository, check=True)
+        subprocess.run(
+            ["git", "config", "user.email", "fixture@example.invalid"],
+            cwd=repository,
+            check=True,
+        )
+        (repository / "README.md").write_text("locked\n")
+        subprocess.run(["git", "add", "README.md"], cwd=repository, check=True)
+        subprocess.run(["git", "commit", "--quiet", "-m", "locked"], cwd=repository, check=True)
+        locked = subprocess.check_output(
+            ["git", "rev-parse", "HEAD"], cwd=repository, text=True
+        ).strip()
+        (repository / "README.md").write_text("later\n")
+        subprocess.run(["git", "commit", "--quiet", "-am", "later"], cwd=repository, check=True)
+        later = subprocess.check_output(
+            ["git", "rev-parse", "HEAD"], cwd=repository, text=True
+        ).strip()
+
+        scripts = repository / "scripts"
+        scripts.mkdir()
+        script = scripts / SOURCE_SCRIPT.name
+        shutil.copyfile(SOURCE_SCRIPT, script)
+        script.chmod(0o755)
+        lock = parent / "services.lock"
+        lines = []
+        for index in range(8):
+            branch = f"service-{index}"
+            subprocess.run(
+                ["git", "update-ref", f"refs/remotes/origin/{branch}", locked],
+                cwd=repository,
+                check=True,
+            )
+            lines.append(f"service{index}|{branch}|{locked}|service{index}.jar")
+        lock.write_text("\n".join(lines) + "\n")
+        return script, lock, locked, later
+
+    def invoke(self, script: pathlib.Path, lock: pathlib.Path, target: pathlib.Path, mode="materialize"):
+        environment = os.environ.copy()
+        environment["SERVICES_LOCK"] = str(lock)
+        return subprocess.run(
+            [str(script), str(target), mode],
+            cwd=script.parent.parent,
+            env=environment,
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+
+    def test_materializes_when_only_remote_tracking_refs_exist(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            parent = pathlib.Path(temporary).resolve()
+            script, lock, locked, _later = self.fixture(parent)
+            target = parent / "sources"
+
+            result = self.invoke(script, lock, target)
+
+            self.assertEqual(result.returncode, 0, result.stderr)
+            for source in target.iterdir():
+                self.assertEqual(
+                    subprocess.check_output(["git", "rev-parse", "HEAD"], cwd=source, text=True).strip(),
+                    locked,
+                )
+            self.assertEqual(self.invoke(script, lock, target, "cleanup").returncode, 0)
+
+    def test_does_not_hide_a_diverged_local_branch(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            parent = pathlib.Path(temporary).resolve()
+            script, lock, _locked, later = self.fixture(parent)
+            repository = script.parent.parent
+            subprocess.run(
+                ["git", "update-ref", "refs/heads/service-0", later], cwd=repository, check=True
+            )
+
+            result = self.invoke(script, lock, parent / "sources")
+
+            self.assertNotEqual(result.returncode, 0)
+            self.assertIn("no longer matches", result.stderr)
+            self.assertFalse((parent / "sources").exists())
+
+
+if __name__ == "__main__":
+    unittest.main()


