## `test(load): add audit read fixture`

diff --git a/load-test/scenarios/audit-read.js b/load-test/scenarios/audit-read.js
new file mode 100644
index 0000000..75eff66
--- /dev/null
+++ b/load-test/scenarios/audit-read.js
@@ -0,0 +1,30 @@
+import http from 'k6/http';
+import { check } from 'k6';
+
+const BASE_URL = __ENV.ADMIN_BASE_URL || 'http://127.0.0.1:8090';
+const TOKEN = __ENV.ADMIN_TOKEN || '';
+
+export const options = {
+  scenarios: {
+    audit_reads: {
+      executor: 'ramping-vus',
+      startVUs: 0,
+      stages: [
+        { duration: '15s', target: 30 },
+        { duration: '30s', target: 30 },
+        { duration: '15s', target: 0 },
+      ],
+    },
+  },
+  thresholds: {
+    http_req_failed: ['rate<0.01'],
+    http_req_duration: ['p(95)<100', 'p(99)<200'],
+  },
+};
+
+export default function () {
+  const response = http.get(`${BASE_URL}/admin/v1/audit-logs?size=20`, {
+    headers: { Authorization: `Bearer ${TOKEN}` },
+  });
+  check(response, { 'status is 200': (result) => result.status === 200 });
+}


## `test(load): prevent persistent load evidence`

diff --git a/src/test/java/com/sportsbook/admin/ops/AuditLoadFixtureTest.java b/src/test/java/com/sportsbook/admin/ops/AuditLoadFixtureTest.java
new file mode 100644
index 0000000..bf1e0b6
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/AuditLoadFixtureTest.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.ops;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+
+class AuditLoadFixtureTest {
+
+  private static final Path ROOT = Path.of(System.getProperty("user.dir"));
+
+  @Test
+  void exercisesTheAuthenticatedAuditReadWithoutPersistingEvidence() throws Exception {
+    String script = Files.readString(ROOT.resolve("load-test/scenarios/audit-read.js"));
+
+    assertThat(script)
+        .contains(
+            "http.get(`${BASE_URL}/admin/v1/audit-logs?size=20`",
+            "Authorization: `Bearer ${TOKEN}`",
+            "http_req_failed",
+            "http_req_duration")
+        .doesNotContain(
+            "handleSummary", "summary-export", "load-test/results", "writeFile", "appendFile");
+  }
+
+  @Test
+  void keepsAllLoadEvidenceIgnored() throws Exception {
+    String ignores = Files.readString(ROOT.resolve(".gitignore"));
+
+    assertThat(ignores).contains("load-test/results/");
+    assertThat(ROOT.resolve("load-test/README.md")).doesNotExist();
+  }
+}
