# 선형 개발 히스토리 정책

## `docs(project): establish orchestration ownership`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..4667b71
--- /dev/null
+++ b/README.md
@@ -0,0 +1,3 @@
+# Sportsbook Orchestration
+
+Reproducible build, runtime, and integration boundary for the sportsbook services.


## `build(history): parse measured linear records`

diff --git a/scripts/cold_gate/history_repository.py b/scripts/cold_gate/history_repository.py
new file mode 100644
index 0000000..614f277
--- /dev/null
+++ b/scripts/cold_gate/history_repository.py
@@ -0,0 +1,70 @@
+from __future__ import annotations
+
+import dataclasses
+import subprocess
+from collections.abc import Callable
+from pathlib import Path
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+FORMAT = "%x1e%H%x1f%P%x1f%B%x1f"
+
+
+@dataclasses.dataclass(frozen=True)
+class Change:
+    path: str
+    additions: int
+    deletions: int
+
+
+@dataclasses.dataclass(frozen=True)
+class Commit:
+    sha: str
+    parents: tuple[str, ...]
+    message: str
+    changes: tuple[Change, ...]
+
+    @property
+    def subject(self) -> str:
+        return self.message.splitlines()[0] if self.message else ""
+
+    @property
+    def churn(self) -> int:
+        return sum(change.additions + change.deletions for change in self.changes)
+
+
+def load_history(
+    root: Path,
+    revision: str = "HEAD",
+    runner: Runner = subprocess.run,
+) -> tuple[Commit, ...]:
+    result = runner(
+        [
+            "git", "-C", str(root), "log", "--reverse", "--no-renames",
+            f"--format={FORMAT}", "--numstat", revision,
+        ],
+        text=True,
+        capture_output=True,
+        check=True,
+    )
+    commits = []
+    for raw in result.stdout.split("\x1e")[1:]:
+        fields = raw.split("\x1f", 3)
+        if len(fields) != 4:
+            raise RuntimeError("Git history record is malformed")
+        sha, parents, message, stats = fields
+        changes = []
+        for line in stats.splitlines():
+            if not line:
+                continue
+            columns = line.split("\t", 2)
+            if len(columns) != 3 or not columns[0].isdigit() or not columns[1].isdigit():
+                raise RuntimeError(f"{sha[:12]} contains an unmeasured change")
+            changes.append(Change(columns[2], int(columns[0]), int(columns[1])))
+        commits.append(
+            Commit(sha, tuple(parents.split()) if parents else (), message.rstrip("\n"),
+                   tuple(changes))
+        )
+    if not commits:
+        raise RuntimeError("Git history is empty")
+    return tuple(commits)


## `test(history): reject unmeasured Git records`

diff --git a/tests/test_history_repository.py b/tests/test_history_repository.py
new file mode 100644
index 0000000..bc43933
--- /dev/null
+++ b/tests/test_history_repository.py
@@ -0,0 +1,49 @@
+import pathlib
+import subprocess
+import unittest
+
+from scripts.cold_gate.history_repository import FORMAT, load_history
+
+
+ROOT_SHA = "a" * 40
+TEST_SHA = "b" * 40
+
+
+class HistoryRepositoryTest(unittest.TestCase):
+    def test_parses_ordered_messages_parents_and_measured_changes(self):
+        output = (
+            f"\x1e{ROOT_SHA}\x1f\x1fdocs(project): root\n\x1f\n3\t0\tREADME.md\n"
+            f"\x1e{TEST_SHA}\x1f{ROOT_SHA}\x1ftest(repo): verify root\n\x1f\n"
+            "4\t1\ttests/test_root.py\n"
+        )
+
+        def runner(command, **_options):
+            self.assertIn(f"--format={FORMAT}", command)
+            self.assertIn("--no-renames", command)
+            return subprocess.CompletedProcess(command, 0, stdout=output)
+
+        commits = load_history(pathlib.Path("/release"), runner=runner)
+
+        self.assertEqual([commit.sha for commit in commits], [ROOT_SHA, TEST_SHA])
+        self.assertEqual(commits[0].parents, ())
+        self.assertEqual(commits[1].parents, (ROOT_SHA,))
+        self.assertEqual(commits[0].subject, "docs(project): root")
+        self.assertEqual(commits[1].changes[0].path, "tests/test_root.py")
+        self.assertEqual(commits[1].churn, 5)
+
+    def test_rejects_binary_or_empty_history_receipts(self):
+        outputs = (
+            "",
+            f"\x1e{ROOT_SHA}\x1f\x1fbuild(repo): binary\n\x1f\n-\t-\tasset.bin\n",
+        )
+        for output in outputs:
+            with self.subTest(output=output):
+                runner = lambda command, **_options: subprocess.CompletedProcess(
+                    command, 0, stdout=output
+                )
+                with self.assertRaisesRegex(RuntimeError, "empty|unmeasured"):
+                    load_history(pathlib.Path("/release"), runner=runner)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(history): classify policy boundaries`

diff --git a/scripts/cold_gate/history_rules.py b/scripts/cold_gate/history_rules.py
new file mode 100644
index 0000000..d5f98f2
--- /dev/null
+++ b/scripts/cold_gate/history_rules.py
@@ -0,0 +1,54 @@
+from __future__ import annotations
+
+import re
+
+from scripts.cold_gate.history_repository import Change
+
+
+ROOT_SUBJECT = "docs(project): establish orchestration ownership"
+RELEASE_SUBJECT = "build(release): release orchestration 1.0.0"
+DOCS_SUBJECT = "docs(project): document full-stack operations"
+SUBJECT = re.compile(r"^(build|chore|ci|docs|feat|fix|test)\([a-z0-9-]+\): \S(?:.*\S)?$")
+BANNED_TERMS = ("fixup!", "squash!", "reconstruct", "provenance", "devlog", "changelog")
+GENERATED_DIRECTORIES = frozenset(
+    {".runtime", "target", "build", "evidence", "results", "reports", "__pycache__"}
+)
+GENERATED_SUFFIXES = (".class", ".jar", ".log", ".pyc")
+LARGE_EXCEPTIONS = (
+    re.compile(r"(^|/)mvnw(?:\.cmd)?$"),
+    re.compile(r"(^|/)\.mvn/wrapper/maven-wrapper\.properties$"),
+    re.compile(r"(^|/)pom\.xml$"),
+    re.compile(r"(^|/)src/main/resources/db/migration/V[1-9][0-9]*__[^/]+\.sql$"),
+    re.compile(r"\.avsc$"),
+)
+
+
+def is_test_path(path: str) -> bool:
+    if path.startswith(("tests/", "e2e/")):
+        return True
+    parts = path.split("/")
+    return len(parts) >= 5 and parts[0] == "fixtures" and parts[2:4] == ["src", "test"]
+
+
+def is_documentation(path: str) -> bool:
+    return path.lower().endswith((".md", ".adoc", ".rst"))
+
+
+def forbidden_path(path: str) -> bool:
+    lower = path.lower()
+    parts = lower.split("/")
+    if any(term in part for term in BANNED_TERMS for part in parts):
+        return True
+    if any(part in GENERATED_DIRECTORIES for part in parts[:-1]):
+        return True
+    return (
+        lower == "docker/jars"
+        or lower.startswith("docker/.jars/")
+        or lower.endswith(GENERATED_SUFFIXES)
+    )
+
+
+def is_large_exception(changes: tuple[Change, ...]) -> bool:
+    return len(changes) == 1 and any(
+        pattern.search(changes[0].path) for pattern in LARGE_EXCEPTIONS
+    )


## `test(history): verify path policy boundaries`

diff --git a/tests/test_history_rules.py b/tests/test_history_rules.py
new file mode 100644
index 0000000..ab3035f
--- /dev/null
+++ b/tests/test_history_rules.py
@@ -0,0 +1,62 @@
+import unittest
+
+from scripts.cold_gate.history_repository import Change
+from scripts.cold_gate.history_rules import (
+    forbidden_path,
+    is_documentation,
+    is_large_exception,
+    is_test_path,
+)
+
+
+class HistoryRulesTest(unittest.TestCase):
+    def test_classifies_only_owned_test_boundaries(self):
+        accepted = (
+            "tests/test_gate.py",
+            "e2e/scenario_01.py",
+            "fixtures/avro-publisher/src/test/java/PublisherTest.java",
+        )
+        rejected = (
+            "scripts/cold_gate/gate.py",
+            "fixtures/avro-publisher/src/main/java/Publisher.java",
+            "compose.yaml",
+        )
+        self.assertTrue(all(is_test_path(path) for path in accepted))
+        self.assertFalse(any(is_test_path(path) for path in rejected))
+
+    def test_detects_documentation_and_generated_or_process_files(self):
+        self.assertTrue(is_documentation("README.md"))
+        self.assertFalse(is_documentation("config/required-secrets.txt"))
+        forbidden = (
+            ".runtime/run/owner",
+            "evidence/cold-gate/run.tsv",
+            "fixtures/tool/target/tool.jar",
+            "docker/.jars/generation/app.jar",
+            "docker/jars",
+            "notes/devlog.md",
+            "src/App.class",
+            "logs/runtime.log",
+        )
+        self.assertTrue(all(forbidden_path(path) for path in forbidden))
+        self.assertFalse(forbidden_path("scripts/cold_gate/log_evidence.py"))
+
+    def test_allows_only_one_responsibility_large_exceptions(self):
+        allowed = (
+            "mvnw",
+            "service/pom.xml",
+            "src/main/resources/db/migration/V2__lifecycle.sql",
+            "src/main/avro/AdminAction.avsc",
+        )
+        for path in allowed:
+            with self.subTest(path=path):
+                self.assertTrue(is_large_exception((Change(path, 101, 0),)))
+        self.assertFalse(
+            is_large_exception(
+                (Change("pom.xml", 101, 0), Change("src/App.java", 1, 0))
+            )
+        )
+        self.assertFalse(is_large_exception((Change("src/App.java", 101, 0),)))
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(history): close process artifact aliases`

diff --git a/scripts/cold_gate/history_rules.py b/scripts/cold_gate/history_rules.py
index d5f98f2..59c0b10 100644
--- a/scripts/cold_gate/history_rules.py
+++ b/scripts/cold_gate/history_rules.py
@@ -9,7 +9,7 @@ ROOT_SUBJECT = "docs(project): establish orchestration ownership"
 RELEASE_SUBJECT = "build(release): release orchestration 1.0.0"
 DOCS_SUBJECT = "docs(project): document full-stack operations"
 SUBJECT = re.compile(r"^(build|chore|ci|docs|feat|fix|test)\([a-z0-9-]+\): \S(?:.*\S)?$")
-BANNED_TERMS = ("fixup!", "squash!", "reconstruct", "provenance", "devlog", "changelog")
+BANNED_TERMS = ("fixup", "squash", "reconstruct", "provenance", "devlog", "changelog")
 GENERATED_DIRECTORIES = frozenset(
     {".runtime", "target", "build", "evidence", "results", "reports", "__pycache__"}
 )
@@ -31,7 +31,13 @@ def is_test_path(path: str) -> bool:
 
 
 def is_documentation(path: str) -> bool:
-    return path.lower().endswith((".md", ".adoc", ".rst"))
+    lower = path.lower()
+    basename = lower.rsplit("/", 1)[-1]
+    return (
+        lower.startswith(("docs/", "handoffs/"))
+        or lower.endswith((".md", ".adoc", ".rst"))
+        or basename.startswith(("readme", "changelog", "devlog"))
+    )
 
 
 def forbidden_path(path: str) -> bool:


## `build(history): enforce atomic development blocks`

diff --git a/scripts/cold_gate/history_policy.py b/scripts/cold_gate/history_policy.py
new file mode 100644
index 0000000..c1525cd
--- /dev/null
+++ b/scripts/cold_gate/history_policy.py
@@ -0,0 +1,93 @@
+from __future__ import annotations
+
+from scripts.cold_gate.history_repository import Commit
+from scripts.cold_gate.history_rules import (
+    BANNED_TERMS,
+    DOCS_SUBJECT,
+    RELEASE_SUBJECT,
+    ROOT_SUBJECT,
+    SUBJECT,
+    forbidden_path,
+    is_documentation,
+    is_large_exception,
+    is_test_path,
+)
+
+
+def verify_history(commits: tuple[Commit, ...], minimum: int = 240) -> None:
+    if len(commits) < minimum:
+        raise RuntimeError(f"history has only {len(commits)} commits; expected at least {minimum}")
+    _verify_release_shape(commits)
+    for index, commit in enumerate(commits):
+        label = commit.sha[:12]
+        if index == 0:
+            if commit.parents or commit.subject != ROOT_SUBJECT or _paths(commit) != ("README.md",):
+                raise RuntimeError("history root is not the README-only ownership commit")
+        elif commit.parents != (commits[index - 1].sha,):
+            raise RuntimeError(f"{label} is not on the single linear parent chain")
+        if not commit.message or commit.message != commit.subject:
+            raise RuntimeError(f"{label} has an empty or multi-line message")
+        if SUBJECT.fullmatch(commit.subject) is None:
+            raise RuntimeError(f"{label} subject is not conventional")
+        lower_subject = commit.subject.lower()
+        if any(term in lower_subject for term in BANNED_TERMS):
+            raise RuntimeError(f"{label} subject records forbidden process metadata")
+        if not commit.changes or len(_paths(commit)) != len(commit.changes):
+            raise RuntimeError(f"{label} has an empty or duplicate change set")
+        if any(forbidden_path(path) for path in _paths(commit)):
+            raise RuntimeError(f"{label} tracks generated or process artifacts")
+        final_docs = index == len(commits) - 1 and commit.subject == DOCS_SUBJECT
+        if commit.churn > 100 and not final_docs and not is_large_exception(commit.changes):
+            raise RuntimeError(f"{label} exceeds the 100-line responsibility limit")
+        _verify_documentation(commit, index, len(commits))
+        terminal_release = index == len(commits) - 2 and commit.subject == RELEASE_SUBJECT
+        if index != 0 and not terminal_release and not final_docs:
+            _verify_development_boundary(commits, index)
+
+
+def _verify_release_shape(commits: tuple[Commit, ...]) -> None:
+    releases = [index for index, commit in enumerate(commits) if commit.subject == RELEASE_SUBJECT]
+    documents = [index for index, commit in enumerate(commits) if commit.subject == DOCS_SUBJECT]
+    if not releases and not documents:
+        return
+    if releases != [len(commits) - 2] or documents != [len(commits) - 1]:
+        raise RuntimeError("release and final documentation commits are not terminal")
+    if _paths(commits[-2]) != ("VERSION",) or _paths(commits[-1]) != ("README.md",):
+        raise RuntimeError("terminal release commits have an invalid file boundary")
+
+
+def _verify_documentation(commit: Commit, index: int, count: int) -> None:
+    for path in _paths(commit):
+        allowed_readme = path == "README.md" and (
+            index == 0 or (index == count - 1 and commit.subject == DOCS_SUBJECT)
+        )
+        allowed_version = path == "VERSION" and (
+            index == count - 2 and commit.subject == RELEASE_SUBJECT
+        )
+        if (is_documentation(path) and not allowed_readme) or (
+            path in {"README.md", "VERSION"} and not (allowed_readme or allowed_version)
+        ):
+            raise RuntimeError(f"{commit.sha[:12]} mixes documentation into development")
+
+
+def _verify_development_boundary(commits: tuple[Commit, ...], index: int) -> None:
+    commit = commits[index]
+    if commit.subject.startswith("docs("):
+        raise RuntimeError(f"{commit.sha[:12]} is an intermediate documentation commit")
+    tests = tuple(change for change in commit.changes if is_test_path(change.path))
+    production = tuple(change for change in commit.changes if not is_test_path(change.path))
+    if tests and production:
+        raise RuntimeError(f"{commit.sha[:12]} mixes production and tests")
+    if bool(tests) != commit.subject.startswith("test("):
+        raise RuntimeError(f"{commit.sha[:12]} subject does not match its change boundary")
+    if production and len(production) > 2:
+        raise RuntimeError(f"{commit.sha[:12]} changes more than two production files")
+    if production:
+        if index + 1 >= len(commits) or not all(
+            is_test_path(change.path) for change in commits[index + 1].changes
+        ):
+            raise RuntimeError(f"{commit.sha[:12]} is not followed by an adjacent test commit")
+
+
+def _paths(commit: Commit) -> tuple[str, ...]:
+    return tuple(change.path for change in commit.changes)


## `test(history): verify linear release structure`

diff --git a/tests/test_history_policy_structure.py b/tests/test_history_policy_structure.py
new file mode 100644
index 0000000..0e3ad0f
--- /dev/null
+++ b/tests/test_history_policy_structure.py
@@ -0,0 +1,58 @@
+import unittest
+
+from scripts.cold_gate.history_policy import verify_history
+from scripts.cold_gate.history_repository import Change
+from scripts.cold_gate.history_rules import ROOT_SUBJECT
+from tests.history_policy_fixture import changed, valid_development, valid_release
+
+
+class HistoryPolicyStructureTest(unittest.TestCase):
+    def test_accepts_complete_prerelease_and_terminal_release_shapes(self):
+        verify_history(valid_development(), minimum=1)
+        verify_history(valid_release(), minimum=1)
+
+    def test_requires_full_linear_history_and_exact_root(self):
+        valid = valid_development()
+        invalid = (
+            (changed(valid, 0, message="docs(project): wrong root"), "root"),
+            (changed(valid, 0, changes=(Change("OTHER", 1, 0),)), "root"),
+            (changed(valid, 1, parents=(valid[0].sha, "f" * 40)), "linear"),
+        )
+        for commits, message in invalid:
+            with self.subTest(message=message):
+                with self.assertRaisesRegex(RuntimeError, message):
+                    verify_history(commits, minimum=1)
+        with self.assertRaisesRegex(RuntimeError, "only 3 commits"):
+            verify_history(valid, minimum=4)
+
+    def test_rejects_message_bodies_and_process_markers(self):
+        valid = valid_development()
+        invalid = (
+            changed(valid, 1, message="build(app): add behavior\n\nimplementation note"),
+            changed(valid, 1, message="not conventional"),
+            changed(valid, 1, message="fix(app): squash temporary work"),
+            changed(valid, 1, message="fix(app): record provenance"),
+        )
+        for commits in invalid:
+            with self.subTest(subject=commits[1].subject):
+                with self.assertRaises(RuntimeError):
+                    verify_history(commits, minimum=1)
+
+    def test_requires_exact_terminal_release_pair(self):
+        release = valid_release()
+        invalid = (
+            release[:-1],
+            changed(release, -2, changes=(Change("version.txt", 1, 0),)),
+            changed(release, -1, changes=(Change("docs/README.md", 1, 0),)),
+        )
+        for commits in invalid:
+            with self.subTest(length=len(commits)):
+                with self.assertRaisesRegex(RuntimeError, "release|terminal"):
+                    verify_history(commits, minimum=1)
+
+    def test_root_constant_remains_the_owned_document_subject(self):
+        self.assertEqual(ROOT_SUBJECT, "docs(project): establish orchestration ownership")
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(history): detect duplicate changed paths`

diff --git a/scripts/cold_gate/history_policy.py b/scripts/cold_gate/history_policy.py
index c1525cd..3e923c1 100644
--- a/scripts/cold_gate/history_policy.py
+++ b/scripts/cold_gate/history_policy.py
@@ -32,7 +32,7 @@ def verify_history(commits: tuple[Commit, ...], minimum: int = 240) -> None:
         lower_subject = commit.subject.lower()
         if any(term in lower_subject for term in BANNED_TERMS):
             raise RuntimeError(f"{label} subject records forbidden process metadata")
-        if not commit.changes or len(_paths(commit)) != len(commit.changes):
+        if not commit.changes or len(set(_paths(commit))) != len(commit.changes):
             raise RuntimeError(f"{label} has an empty or duplicate change set")
         if any(forbidden_path(path) for path in _paths(commit)):
             raise RuntimeError(f"{label} tracks generated or process artifacts")


## `build(history): preserve conventional type vocabulary`

diff --git a/scripts/cold_gate/history_rules.py b/scripts/cold_gate/history_rules.py
index 59c0b10..5c5dd9e 100644
--- a/scripts/cold_gate/history_rules.py
+++ b/scripts/cold_gate/history_rules.py
@@ -8,7 +8,10 @@ from scripts.cold_gate.history_repository import Change
 ROOT_SUBJECT = "docs(project): establish orchestration ownership"
 RELEASE_SUBJECT = "build(release): release orchestration 1.0.0"
 DOCS_SUBJECT = "docs(project): document full-stack operations"
-SUBJECT = re.compile(r"^(build|chore|ci|docs|feat|fix|test)\([a-z0-9-]+\): \S(?:.*\S)?$")
+SUBJECT = re.compile(
+    r"^(build|chore|ci|docs|feat|fix|perf|refactor|style|test)"
+    r"\([a-z0-9-]+\): \S(?:.*\S)?$"
+)
 BANNED_TERMS = ("fixup", "squash", "reconstruct", "provenance", "devlog", "changelog")
 GENERATED_DIRECTORIES = frozenset(
     {".runtime", "target", "build", "evidence", "results", "reports", "__pycache__"}


