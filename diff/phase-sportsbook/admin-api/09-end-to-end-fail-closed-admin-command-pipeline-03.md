## `test(audit): record exact mutation statuses`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditDeclaredResponseStatusTest.java b/src/test/java/com/sportsbook/admin/audit/AuditDeclaredResponseStatusTest.java
new file mode 100644
index 0000000..9169c82
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditDeclaredResponseStatusTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.api.MarketAdminController;
+import com.sportsbook.admin.api.RiskAdminController;
+import com.sportsbook.admin.api.SettlementRevisionCommandController;
+import com.sportsbook.admin.client.MarketStatusPayload;
+import com.sportsbook.admin.client.RiskLimitPayload;
+import com.sportsbook.admin.context.AdminContext;
+import jakarta.servlet.http.HttpServletRequest;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AuditDeclaredResponseStatusTest {
+
+  private final AuditOutcomeClassifier classifier = new AuditOutcomeClassifier();
+
+  @Test
+  void recordsTheExactSuccessStatusDeclaredByMutationEndpoints() throws NoSuchMethodException {
+    Method risk =
+        RiskAdminController.class.getMethod(
+            "setLimit", UUID.class, RiskLimitPayload.class, AdminContext.class);
+    Method odds =
+        MarketAdminController.class.getMethod(
+            "suspend",
+            UUID.class,
+            UUID.class,
+            MarketStatusPayload.class,
+            AdminContext.class,
+            HttpServletRequest.class);
+    Method settlement =
+        SettlementRevisionCommandController.class.getMethod(
+            "retry", UUID.class, AdminContext.class, HttpServletRequest.class);
+
+    assertStatus(risk, 204);
+    assertStatus(odds, 202);
+    assertStatus(settlement, 202);
+  }
+
+  private void assertStatus(Method method, int expectedStatus) {
+    AuditOutcomeClassifier.AuditDecision decision = classifier.result(null, method);
+    assertThat(decision.outcome()).isEqualTo(AuditOutcome.SUCCESS);
+    assertThat(decision.httpStatus()).isEqualTo(expectedStatus);
+  }
+}
