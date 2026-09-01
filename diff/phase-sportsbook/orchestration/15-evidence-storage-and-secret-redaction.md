# 증거 저장과 비밀 제거

## `build(evidence): redact runtime credentials`

diff --git a/scripts/cold_gate/redaction.py b/scripts/cold_gate/redaction.py
new file mode 100644
index 0000000..5051c61
--- /dev/null
+++ b/scripts/cold_gate/redaction.py
@@ -0,0 +1,32 @@
+from __future__ import annotations
+
+import re
+from collections.abc import Iterable
+
+
+PEM_PATTERN = re.compile(
+    r"-----BEGIN [^-\r\n]*KEY-----.*?-----END [^-\r\n]*KEY-----",
+    re.DOTALL,
+)
+JWT_PATTERN = re.compile(r"\beyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\b")
+
+
+class EvidenceRedactor:
+    def __init__(self, secret_values: Iterable[str]) -> None:
+        values = {value for value in secret_values if value}
+        if any(len(value) < 8 for value in values):
+            raise ValueError("redaction secrets must contain at least eight characters")
+        self.secret_values = tuple(sorted(values, key=len, reverse=True))
+
+    def redact(self, value: str) -> str:
+        redacted = PEM_PATTERN.sub("[REDACTED PEM]", value)
+        redacted = JWT_PATTERN.sub("[REDACTED JWT]", redacted)
+        for secret in self.secret_values:
+            redacted = redacted.replace(secret, "[REDACTED SECRET]")
+        return redacted
+
+    def require_clean(self, value: str) -> None:
+        if PEM_PATTERN.search(value) or JWT_PATTERN.search(value):
+            raise RuntimeError("evidence contains key material or a JWT")
+        if any(secret in value for secret in self.secret_values):
+            raise RuntimeError("evidence contains an exact secret value")


## `test(evidence): remove secrets JWTs and keys`

diff --git a/tests/test_evidence_redaction.py b/tests/test_evidence_redaction.py
new file mode 100644
index 0000000..3232ee9
--- /dev/null
+++ b/tests/test_evidence_redaction.py
@@ -0,0 +1,50 @@
+import unittest
+
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+class EvidenceRedactionTest(unittest.TestCase):
+    def test_redacts_exact_secrets_jwts_and_pem_blocks(self) -> None:
+        secrets = (
+            "wallet-secret-value-0000000000000001",
+            "database-password-value",
+        )
+        redactor = EvidenceRedactor(secrets)
+        source = """
+        X-API-Key: wallet-secret-value-0000000000000001
+        password=database-password-value
+        Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxIn0.signature
+        -----BEGIN PRIVATE KEY-----
+        private-material
+        -----END PRIVATE KEY-----
+        -----BEGIN PUBLIC KEY-----
+        public-material
+        -----END PUBLIC KEY-----
+        """
+
+        redacted = redactor.redact(source)
+
+        redactor.require_clean(redacted)
+        for secret in secrets:
+            self.assertNotIn(secret, redacted)
+        self.assertNotIn("eyJhbGci", redacted)
+        self.assertNotIn("BEGIN PRIVATE KEY", redacted)
+        self.assertNotIn("BEGIN PUBLIC KEY", redacted)
+        self.assertIn("[REDACTED SECRET]", redacted)
+        self.assertIn("[REDACTED JWT]", redacted)
+        self.assertIn("[REDACTED PEM]", redacted)
+
+    def test_rejects_unredacted_evidence_and_short_markers(self) -> None:
+        secret = "service-secret-value"
+        redactor = EvidenceRedactor((secret,))
+
+        with self.assertRaisesRegex(RuntimeError, "exact secret"):
+            redactor.require_clean(f"key={secret}")
+        with self.assertRaisesRegex(RuntimeError, "key material"):
+            redactor.require_clean("-----BEGIN PUBLIC KEY-----x-----END PUBLIC KEY-----")
+        with self.assertRaises(ValueError):
+            EvidenceRedactor(("short",))
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): capture a fixed artifact inventory`

diff --git a/scripts/cold_gate/evidence.py b/scripts/cold_gate/evidence.py
new file mode 100644
index 0000000..f4ba233
--- /dev/null
+++ b/scripts/cold_gate/evidence.py
@@ -0,0 +1,79 @@
+from __future__ import annotations
+
+import os
+import re
+import tempfile
+from pathlib import Path, PurePosixPath
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+REQUIRED_FILES = frozenset(
+    {
+        "run.tsv",
+        "services.lock",
+        "jars.sha256",
+        "images.tsv",
+        "compose.sha256",
+        "topics.tsv",
+        "migrations.tsv",
+        "readiness.tsv",
+        "scenarios.tsv",
+        "compose-ps.json",
+        "cleanup.tsv",
+    }
+)
+LOG_PATTERN = re.compile(r"logs/[a-z0-9][a-z0-9-]*\.log")
+
+
+class EvidenceStore:
+    def __init__(self, context: ColdGateContext, redactor: EvidenceRedactor) -> None:
+        self.context = context
+        self.redactor = redactor
+
+    def write(self, relative_name: str, content: str) -> Path:
+        self.context.require_owned()
+        self._validate_name(relative_name)
+        redacted = self.redactor.redact(content)
+        self.redactor.require_clean(redacted)
+        target = self.context.evidence / relative_name
+        target.parent.mkdir(parents=True, exist_ok=True)
+        if target.is_symlink():
+            raise RuntimeError("evidence target must not be a symlink")
+
+        pending: Path | None = None
+        try:
+            with tempfile.NamedTemporaryFile(
+                mode="w", dir=target.parent, prefix=".pending.", delete=False
+            ) as output:
+                output.write(redacted)
+                pending = Path(output.name)
+            os.replace(pending, target)
+        finally:
+            if pending is not None and pending.exists():
+                pending.unlink()
+        return target
+
+    def verify(self, complete: bool = False) -> None:
+        self.context.require_owned()
+        found = set()
+        for path in self.context.evidence.rglob("*"):
+            if path.is_symlink():
+                raise RuntimeError("evidence must not contain symlinks")
+            if path.is_dir():
+                continue
+            relative_name = path.relative_to(self.context.evidence).as_posix()
+            self._validate_name(relative_name)
+            found.add(relative_name)
+            self.redactor.require_clean(path.read_text())
+        if complete and not REQUIRED_FILES.issubset(found):
+            raise RuntimeError("evidence inventory is incomplete")
+
+    @staticmethod
+    def _validate_name(relative_name: str) -> None:
+        path = PurePosixPath(relative_name)
+        if path.is_absolute() or ".." in path.parts:
+            raise RuntimeError("evidence path escaped its root")
+        if relative_name not in REQUIRED_FILES and not LOG_PATTERN.fullmatch(relative_name):
+            raise RuntimeError("evidence file is not in the whitelist")


## `test(evidence): reject unsafe artifact capture`

diff --git a/tests/test_evidence_capture.py b/tests/test_evidence_capture.py
new file mode 100644
index 0000000..24bf0e1
--- /dev/null
+++ b/tests/test_evidence_capture.py
@@ -0,0 +1,66 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore, REQUIRED_FILES
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+SECRET = "wallet-secret-value-0000000000000001"
+
+
+class EvidenceCaptureTest(unittest.TestCase):
+    def store(self, root: pathlib.Path) -> EvidenceStore:
+        context = ColdGateContext.create(root, SHA, "00000001")
+        return EvidenceStore(context, EvidenceRedactor((SECRET,)))
+
+    def test_writes_only_whitelisted_redacted_artifacts(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            store = self.store(pathlib.Path(temporary))
+
+            target = store.write(
+                "logs/admin.log",
+                f"key={SECRET} token=eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxIn0.signature\n",
+            )
+
+            content = target.read_text()
+            self.assertNotIn(SECRET, content)
+            self.assertNotIn("eyJhbGci", content)
+            store.verify()
+            with self.assertRaisesRegex(RuntimeError, "whitelist"):
+                store.write("resolved-compose.yaml", "services: {}\n")
+            with self.assertRaisesRegex(RuntimeError, "escaped"):
+                store.write("../outside", "unsafe\n")
+
+    def test_requires_the_complete_fixed_inventory(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            store = self.store(pathlib.Path(temporary))
+
+            with self.assertRaisesRegex(RuntimeError, "incomplete"):
+                store.verify(complete=True)
+            for name in REQUIRED_FILES:
+                store.write(name, f"artifact\t{name}\n")
+
+            store.verify(complete=True)
+
+    def test_rejects_symlinks_and_directly_added_files(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            store = self.store(root)
+            foreign = root / "foreign"
+            foreign.write_text("foreign\n")
+            target = store.context.evidence / "run.tsv"
+            target.symlink_to(foreign)
+
+            with self.assertRaisesRegex(RuntimeError, "symlink"):
+                store.write("run.tsv", "run\n")
+            target.unlink()
+            (store.context.evidence / "unexpected.txt").write_text("unexpected\n")
+            with self.assertRaisesRegex(RuntimeError, "whitelist"):
+                store.verify()
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(evidence): bound owned artifact writes`

diff --git a/scripts/cold_gate/evidence.py b/scripts/cold_gate/evidence.py
index f4ba233..52726ec 100644
--- a/scripts/cold_gate/evidence.py
+++ b/scripts/cold_gate/evidence.py
@@ -6,6 +6,11 @@ import tempfile
 from pathlib import Path, PurePosixPath
 
 from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.owned_path import (
+    ensure_directory,
+    require_directory,
+    require_regular_file,
+)
 from scripts.cold_gate.redaction import EvidenceRedactor
 
 
@@ -24,7 +29,15 @@ REQUIRED_FILES = frozenset(
         "cleanup.tsv",
     }
 )
-LOG_PATTERN = re.compile(r"logs/[a-z0-9][a-z0-9-]*\.log")
+LOG_SERVICES = frozenset(
+    {
+        "postgres", "kafka", "topic-init", "secret-preflight",
+        "redis-risk", "redis-odds", "redis-wallet", "redis-gateway",
+        "wallet", "risk", "odds", "betting", "gateway", "settlement", "admin",
+        "consumer-assignment", "toxiproxy", "prometheus", "loki", "grafana", "promtail",
+    }
+)
+LOG_PATTERN = re.compile(r"logs/([a-z0-9][a-z0-9-]*)\.log")
 
 
 class EvidenceStore:
@@ -35,12 +48,20 @@ class EvidenceStore:
     def write(self, relative_name: str, content: str) -> Path:
         self.context.require_owned()
         self._validate_name(relative_name)
+        limit = 1024 * 1024 if relative_name.startswith("logs/") else 256 * 1024
+        if "\0" in content or len(content.encode()) > limit:
+            raise RuntimeError("evidence content is unsafe or too large")
         redacted = self.redactor.redact(content)
         self.redactor.require_clean(redacted)
         target = self.context.evidence / relative_name
-        target.parent.mkdir(parents=True, exist_ok=True)
-        if target.is_symlink():
-            raise RuntimeError("evidence target must not be a symlink")
+        if target.parent == self.context.evidence:
+            require_directory(target.parent)
+        elif target.parent == self.context.evidence / "logs":
+            ensure_directory(target.parent)
+        else:
+            raise RuntimeError("evidence parent is not owned")
+        if target.exists() or target.is_symlink():
+            raise RuntimeError("evidence files are write-once")
 
         pending: Path | None = None
         try:
@@ -50,6 +71,7 @@ class EvidenceStore:
                 output.write(redacted)
                 pending = Path(output.name)
             os.replace(pending, target)
+            require_regular_file(target)
         finally:
             if pending is not None and pending.exists():
                 pending.unlink()
@@ -59,10 +81,12 @@ class EvidenceStore:
         self.context.require_owned()
         found = set()
         for path in self.context.evidence.rglob("*"):
-            if path.is_symlink():
-                raise RuntimeError("evidence must not contain symlinks")
             if path.is_dir():
+                if path != self.context.evidence / "logs":
+                    raise RuntimeError("evidence contains an unknown directory")
+                require_directory(path)
                 continue
+            require_regular_file(path)
             relative_name = path.relative_to(self.context.evidence).as_posix()
             self._validate_name(relative_name)
             found.add(relative_name)
@@ -75,5 +99,8 @@ class EvidenceStore:
         path = PurePosixPath(relative_name)
         if path.is_absolute() or ".." in path.parts:
             raise RuntimeError("evidence path escaped its root")
-        if relative_name not in REQUIRED_FILES and not LOG_PATTERN.fullmatch(relative_name):
+        log_match = LOG_PATTERN.fullmatch(relative_name)
+        if relative_name not in REQUIRED_FILES and (
+            log_match is None or log_match.group(1) not in LOG_SERVICES
+        ):
             raise RuntimeError("evidence file is not in the whitelist")


## `test(evidence): preserve external path victims`

diff --git a/tests/test_evidence_capture.py b/tests/test_evidence_capture.py
index 24bf0e1..9e319af 100644
--- a/tests/test_evidence_capture.py
+++ b/tests/test_evidence_capture.py
@@ -54,7 +54,7 @@ class EvidenceCaptureTest(unittest.TestCase):
             target = store.context.evidence / "run.tsv"
             target.symlink_to(foreign)
 
-            with self.assertRaisesRegex(RuntimeError, "symlink"):
+            with self.assertRaisesRegex(RuntimeError, "write-once"):
                 store.write("run.tsv", "run\n")
             target.unlink()
             (store.context.evidence / "unexpected.txt").write_text("unexpected\n")
diff --git a/tests/test_evidence_path_safety.py b/tests/test_evidence_path_safety.py
new file mode 100644
index 0000000..b582b52
--- /dev/null
+++ b/tests/test_evidence_path_safety.py
@@ -0,0 +1,62 @@
+import os
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class EvidencePathSafetyTest(unittest.TestCase):
+    def store(self, root: pathlib.Path) -> EvidenceStore:
+        context = ColdGateContext.create(root, SHA, "00000001")
+        return EvidenceStore(context, EvidenceRedactor(("service-secret-value",)))
+
+    def test_rejects_symlinked_log_parent_without_touching_victim(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            store = self.store(root)
+            victim = root / "victim"
+            victim.mkdir()
+            (store.context.evidence / "logs").symlink_to(
+                victim, target_is_directory=True
+            )
+
+            with self.assertRaisesRegex(RuntimeError, "physical directory"):
+                store.write("logs/admin.log", "safe\n")
+            self.assertEqual(list(victim.iterdir()), [])
+
+    def test_rejects_unknown_logs_overwrites_and_unsafe_content(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            store = self.store(pathlib.Path(temporary))
+
+            with self.assertRaisesRegex(RuntimeError, "whitelist"):
+                store.write("logs/foreign.log", "safe\n")
+            store.write("run.tsv", "run\tPASS\n")
+            with self.assertRaisesRegex(RuntimeError, "write-once"):
+                store.write("run.tsv", "replacement\n")
+            with self.assertRaisesRegex(RuntimeError, "unsafe or too large"):
+                store.write("readiness.tsv", "contains\0nul")
+            with self.assertRaisesRegex(RuntimeError, "unsafe or too large"):
+                store.write("topics.tsv", "x" * (256 * 1024 + 1))
+
+    def test_rejects_special_files_and_unknown_directories(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            store = self.store(pathlib.Path(temporary))
+            fifo = store.context.evidence / "run.tsv"
+            os.mkfifo(fifo)
+
+            with self.assertRaisesRegex(RuntimeError, "regular file"):
+                store.verify()
+            fifo.unlink()
+            (store.context.evidence / "foreign").mkdir()
+            with self.assertRaisesRegex(RuntimeError, "unknown directory"):
+                store.verify()
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(evidence): redact encoded credentials`

diff --git a/scripts/cold_gate/redaction.py b/scripts/cold_gate/redaction.py
index 5051c61..253fd8a 100644
--- a/scripts/cold_gate/redaction.py
+++ b/scripts/cold_gate/redaction.py
@@ -1,6 +1,8 @@
 from __future__ import annotations
 
+import json
 import re
+import urllib.parse
 from collections.abc import Iterable
 
 
@@ -9,6 +11,15 @@ PEM_PATTERN = re.compile(
     re.DOTALL,
 )
 JWT_PATTERN = re.compile(r"\beyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\b")
+BEARER_PATTERN = re.compile(r"(?i)\bBearer\s+[A-Za-z0-9._~+/=-]+")
+JSON_SENSITIVE_PATTERN = re.compile(
+    r'(?i)("[^"]*(?:api[_-]?key|password|secret|token)[^"]*"\s*:\s*)'
+    r'"(?:\\.|[^"])*"'
+)
+PLAIN_SENSITIVE_PATTERN = re.compile(
+    r"(?im)^(\s*(?:x-api-key|api[_-]?key|authorization|proxy-authorization|"
+    r"[^\s:=]*(?:password|secret|token))\s*[:=]\s*).+$"
+)
 
 
 class EvidenceRedactor:
@@ -16,13 +27,20 @@ class EvidenceRedactor:
         values = {value for value in secret_values if value}
         if any(len(value) < 8 for value in values):
             raise ValueError("redaction secrets must contain at least eight characters")
-        self.secret_values = tuple(sorted(values, key=len, reverse=True))
+        variants = set(values)
+        for value in values:
+            variants.add(json.dumps(value)[1:-1])
+            variants.add(urllib.parse.quote(value, safe=""))
+        self.secret_values = tuple(sorted(variants, key=len, reverse=True))
 
     def redact(self, value: str) -> str:
         redacted = PEM_PATTERN.sub("[REDACTED PEM]", value)
         redacted = JWT_PATTERN.sub("[REDACTED JWT]", redacted)
+        redacted = BEARER_PATTERN.sub("Bearer [REDACTED]", redacted)
         for secret in self.secret_values:
             redacted = redacted.replace(secret, "[REDACTED SECRET]")
+        redacted = JSON_SENSITIVE_PATTERN.sub(r'\1"[REDACTED]"', redacted)
+        redacted = PLAIN_SENSITIVE_PATTERN.sub(r"\1[REDACTED]", redacted)
         return redacted
 
     def require_clean(self, value: str) -> None:
@@ -30,3 +48,5 @@ class EvidenceRedactor:
             raise RuntimeError("evidence contains key material or a JWT")
         if any(secret in value for secret in self.secret_values):
             raise RuntimeError("evidence contains an exact secret value")
+        if self.redact(value) != value:
+            raise RuntimeError("evidence contains credential material")


## `test(evidence): reject encoded credential leaks`

diff --git a/tests/test_evidence_redaction.py b/tests/test_evidence_redaction.py
index 3232ee9..03e5c32 100644
--- a/tests/test_evidence_redaction.py
+++ b/tests/test_evidence_redaction.py
@@ -30,8 +30,7 @@ class EvidenceRedactionTest(unittest.TestCase):
         self.assertNotIn("eyJhbGci", redacted)
         self.assertNotIn("BEGIN PRIVATE KEY", redacted)
         self.assertNotIn("BEGIN PUBLIC KEY", redacted)
-        self.assertIn("[REDACTED SECRET]", redacted)
-        self.assertIn("[REDACTED JWT]", redacted)
+        self.assertIn("[REDACTED]", redacted)
         self.assertIn("[REDACTED PEM]", redacted)
 
     def test_rejects_unredacted_evidence_and_short_markers(self) -> None:
diff --git a/tests/test_evidence_redaction_formats.py b/tests/test_evidence_redaction_formats.py
new file mode 100644
index 0000000..b12d820
--- /dev/null
+++ b/tests/test_evidence_redaction_formats.py
@@ -0,0 +1,40 @@
+import unittest
+
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+class EvidenceRedactionFormatsTest(unittest.TestCase):
+    def test_redacts_json_url_and_opaque_authorization_values(self) -> None:
+        secret = 'quoted-"secret"-value'
+        redactor = EvidenceRedactor((secret,))
+        source = '''
+        {"x-api-key":"quoted-\\"secret\\"-value",
+         "url":"quoted-%22secret%22-value",
+         "access_token":"unregistered-opaque-token",
+         "Authorization":"Bearer opaque.header.signature"}
+        Proxy-Authorization: Basic opaque-credential
+        sessionToken=another-opaque-value
+        '''
+
+        redacted = redactor.redact(source)
+
+        redactor.require_clean(redacted)
+        self.assertNotIn("quoted", redacted)
+        self.assertNotIn("unregistered-opaque-token", redacted)
+        self.assertNotIn("opaque.header.signature", redacted)
+        self.assertNotIn("opaque-credential", redacted)
+        self.assertNotIn("another-opaque-value", redacted)
+
+    def test_post_scan_rejects_unknown_bearer_and_literal_newline_pem(self) -> None:
+        redactor = EvidenceRedactor(("registered-secret-value",))
+
+        with self.assertRaisesRegex(RuntimeError, "credential material"):
+            redactor.require_clean("Authorization: Bearer opaque-token-value")
+        with self.assertRaisesRegex(RuntimeError, "key material"):
+            redactor.require_clean(
+                r"-----BEGIN PRIVATE KEY-----\nmaterial\n-----END PRIVATE KEY-----"
+            )
+
+
+if __name__ == "__main__":
+    unittest.main()
