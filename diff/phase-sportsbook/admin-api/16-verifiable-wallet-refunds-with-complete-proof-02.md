## `test(wallet): guard and audit refunds`

diff --git a/src/test/java/com/sportsbook/admin/api/WalletAdminControllerTest.java b/src/test/java/com/sportsbook/admin/api/WalletAdminControllerTest.java
new file mode 100644
index 0000000..5ef7887
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/WalletAdminControllerTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.audit.AdminAction;
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.WalletClient;
+import com.sportsbook.admin.context.AdminContext;
+import com.sportsbook.admin.security.AdminRole;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import jakarta.servlet.http.HttpServletRequest;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.access.prepost.PreAuthorize;
+
+class WalletAdminControllerTest {
+
+  @Test
+  void returnsWalletAndAuditIdentities() {
+    WalletClient wallet = mock(WalletClient.class);
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000112");
+    UUID operationGroup = UUID.fromString("018f0000-0000-7000-8000-000000000113");
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000114");
+    AdminContext context = new AdminContext("operator-1", AdminRole.CS, actionId, "trace-1");
+    RefundRequest body = new RefundRequest(750, Currency.KRW, "goodwill");
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(AdminRequestHeaders.IDEMPOTENCY_KEY, "refund retry 01");
+    when(wallet.refund(userId, Money.krw(750), IdempotencyKey.of("refund retry 01")))
+        .thenReturn(operationGroup);
+
+    RefundResponse response =
+        new WalletAdminController(wallet).refund(userId, body, context, request);
+
+    assertThat(response).isEqualTo(new RefundResponse(operationGroup, actionId));
+    verify(wallet).refund(userId, Money.krw(750), IdempotencyKey.of("refund retry 01"));
+  }
+
+  @Test
+  void restrictsAndAuditsTheMutation() throws NoSuchMethodException {
+    Method refund =
+        WalletAdminController.class.getMethod(
+            "refund",
+            UUID.class,
+            RefundRequest.class,
+            AdminContext.class,
+            HttpServletRequest.class);
+
+    assertThat(refund.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','CS')");
+    Audited audited = refund.getAnnotation(Audited.class);
+    assertThat(audited.action()).isEqualTo(AdminAction.WALLET_REFUND);
+    assertThat(audited.target()).isEqualTo("#userId");
+    assertThat(audited.reason()).isEqualTo("#body.reason()");
+  }
+}
