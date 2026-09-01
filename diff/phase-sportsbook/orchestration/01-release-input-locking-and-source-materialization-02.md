## `fix(materialization): pin the corrected betting release`

diff --git a/services.lock b/services.lock
index 9f89636..ec4fb72 100644
--- a/services.lock
+++ b/services.lock
@@ -3,7 +3,7 @@ shared|shared-protocol|f9de6bc1e533761ab4bb1454d8d4ab8175cdf001|shared-protocol-
 wallet|wallet-service|c9a05f4d652f24ac97d3e1cd753f69cef2725ff3|wallet-service-1.0.0.jar
 risk|risk-service|c64f67dbc437a18640dc4984dea4d8194fb5b164|risk-service-1.0.0.jar
 odds|odds-feed-service|574e83d2862f086ae07ff56fd95a8336f78a72da|odds-feed-service-1.0.0.jar
-betting|betting-service|f712bdf389ee3fb63d8cdc84c49e2b84a346edde|betting-service-1.0.0.jar
+betting|betting-service|40f040e2eff9638d7d6ff1983d86584b02cfebbc|betting-service-1.0.0.jar
 gateway|gateway|8248a3233f0fce7ca36a503ee71b7a8a0802d733|gateway-1.0.0.jar
 settlement|settlement-service|e935873660aad4ceb28788521f7657289f97bc15|settlement-service-1.0.0.jar
 admin|admin-api|2fb55910475b31084e6489bf01c34cc970c96874|admin-api-1.0.0.jar


## `test(materialization): require the corrected betting lock`

diff --git a/tests/test_services_lock.py b/tests/test_services_lock.py
index 31f323f..7e1f73f 100644
--- a/tests/test_services_lock.py
+++ b/tests/test_services_lock.py
@@ -52,6 +52,21 @@ class ServicesLockTest(unittest.TestCase):
 
         self.assertIn("<artifactId>spring-boot-maven-plugin</artifactId>", pom)
 
+    def test_betting_release_locks_the_root_before_loading_legs(self) -> None:
+        commit = next(entry[2] for entry in entries() if entry[0] == "betting")
+        repository = subprocess.check_output(
+            [
+                "git",
+                "show",
+                f"{commit}:src/main/java/com/sportsbook/betting/persistence/BetRepository.java",
+            ],
+            cwd=ROOT,
+            text=True,
+        )
+
+        self.assertIn("default Optional<Bet> findLockedByBetId", repository)
+        self.assertIn("Optional<Bet> findLockedRootByBetId", repository)
+
 
 if __name__ == "__main__":
     unittest.main()


## `test(lock): resolve fetched release refs`

diff --git a/tests/test_services_lock.py b/tests/test_services_lock.py
index 7e1f73f..3f90b67 100644
--- a/tests/test_services_lock.py
+++ b/tests/test_services_lock.py
@@ -22,6 +22,20 @@ def entries() -> list[tuple[str, str, str, str]]:
     return [tuple(line.split("|")) for line in lines if line and not line.startswith("#")]
 
 
+def branch_tip(branch: str) -> str:
+    for namespace in ("refs/heads", "refs/remotes/origin"):
+        result = subprocess.run(
+            ["git", "rev-parse", "--verify", f"{namespace}/{branch}^{{commit}}"],
+            cwd=ROOT,
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+        if result.returncode == 0:
+            return result.stdout.strip()
+    raise AssertionError(f"{branch} ref is unavailable")
+
+
 class ServicesLockTest(unittest.TestCase):
     def test_pins_every_release_branch_to_a_full_commit(self) -> None:
         locked = entries()
@@ -36,10 +50,7 @@ class ServicesLockTest(unittest.TestCase):
                     ["git", "cat-file", "-t", commit], cwd=ROOT, text=True
                 ).strip()
                 self.assertEqual(object_type, "commit")
-                branch_tip = subprocess.check_output(
-                    ["git", "rev-parse", f"refs/heads/{branch}"], cwd=ROOT, text=True
-                ).strip()
-                self.assertEqual(branch_tip, commit)
+                self.assertEqual(branch_tip(branch), commit)
 
     def test_does_not_create_an_orchestration_lock_cycle(self) -> None:
         self.assertNotIn("orchestration", {entry[0] for entry in entries()})
