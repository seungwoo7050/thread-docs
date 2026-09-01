## `test(security): deny unsupported wallet requests`

diff --git a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
index 0494ec3..7383f5b 100644
--- a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
+++ b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
@@ -133,6 +133,51 @@ class WalletSecurityConfigTest {
     }
   }
 
+  @ParameterizedTest
+  @MethodSource("walletRoutes")
+  void rejectsTheOppositeMethodForEveryWalletRoute(WalletRoute route) throws Exception {
+    HttpMethod opposite = route.method() == HttpMethod.GET ? HttpMethod.POST : HttpMethod.GET;
+    WalletCaller allowed = route.allowed().iterator().next();
+
+    mvc.perform(internalRequest(opposite, route.path(), allowed)).andExpect(status().isForbidden());
+  }
+
+  @Test
+  void rejectsUnauthenticatedAndInvalidWalletCredentials() throws Exception {
+    mvc.perform(post("/internal/v1/wallet/accounts"))
+        .andExpect(status().isUnauthorized())
+        .andExpect(jsonPath("$.errorCode").value("WALLET_AUTHENTICATION_REQUIRED"));
+    mvc.perform(
+            post("/internal/v1/wallet/accounts")
+                .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "platform")
+                .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, "invalid"))
+        .andExpect(status().isUnauthorized())
+        .andExpect(jsonPath("$.errorCode").value("WALLET_AUTHENTICATION_REQUIRED"));
+  }
+
+  @Test
+  void rejectsUnknownTrailingAndDeeperWalletPaths() throws Exception {
+    mvc.perform(
+            internalRequest(HttpMethod.GET, "/internal/v1/wallet/unknown", WalletCaller.PLATFORM))
+        .andExpect(status().isForbidden());
+    mvc.perform(
+            internalRequest(
+                HttpMethod.POST, "/internal/v1/wallet/accounts/", WalletCaller.PLATFORM))
+        .andExpect(status().isForbidden());
+    mvc.perform(
+            internalRequest(
+                HttpMethod.GET,
+                "/internal/v1/wallet/accounts/user/balance/extra",
+                WalletCaller.PLATFORM))
+        .andExpect(status().isForbidden());
+    mvc.perform(
+            internalRequest(
+                HttpMethod.GET,
+                "/internal/v1/wallet/transactions/debit/bet/extra",
+                WalletCaller.BETTING))
+        .andExpect(status().isForbidden());
+  }
+
   private org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder internalGet(
       String path, WalletCaller caller) {
     return internalRequest(HttpMethod.GET, path, caller);


## `feat(security): reject unauthorized credit semantics`

diff --git a/src/main/java/com/sportsbook/wallet/domain/error/WalletAccessDeniedException.java b/src/main/java/com/sportsbook/wallet/domain/error/WalletAccessDeniedException.java
new file mode 100644
index 0000000..44eb804
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/WalletAccessDeniedException.java
@@ -0,0 +1,27 @@
+package com.sportsbook.wallet.domain.error;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.Objects;
+
+/** Rejects an authenticated caller whose requested wallet capability is not allowed. */
+public final class WalletAccessDeniedException extends RuntimeException {
+  private final WalletCaller caller;
+  private final String capability;
+
+  public WalletAccessDeniedException(WalletCaller caller, String capability) {
+    super(
+        Objects.requireNonNull(caller, "caller").wireName()
+            + " cannot perform "
+            + Objects.requireNonNull(capability, "capability"));
+    this.caller = caller;
+    this.capability = capability;
+  }
+
+  public WalletCaller caller() {
+    return caller;
+  }
+
+  public String capability() {
+    return capability;
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
index bf5adf9..c0b13dd 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -9,6 +9,7 @@ import com.sportsbook.wallet.domain.WalletOperation;
 import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.domain.error.AccountNotFoundException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.WalletAccessDeniedException;
 import com.sportsbook.wallet.outbox.OutboxAppender;
 import com.sportsbook.wallet.outbox.WalletEventFactory;
 import com.sportsbook.wallet.persistence.AccountRepository;
@@ -184,14 +185,15 @@ public class WalletTransferExecutor {
                 && command.reason() == CreditReason.REFUND)
             || (caller == WalletCaller.SETTLEMENT
                 && ((command.source() == CreditCommand.Source.USER_LOCKED
-                        && command.reason() != CreditReason.PAYOUT)
+                        && (command.reason() == CreditReason.VOID
+                            || command.reason() == CreditReason.REFUND))
                     || (command.source() == CreditCommand.Source.HOUSE_POOL
                         && command.reason() == CreditReason.PAYOUT)))
             || (caller == WalletCaller.ADMIN
                 && command.source() == CreditCommand.Source.HOUSE_POOL
                 && command.reason() == CreditReason.REFUND);
     if (!allowed) {
-      throw new IllegalArgumentException("Caller is not allowed for credit source and reason");
+      throw new WalletAccessDeniedException(caller, "credit source and reason");
     }
   }
 


## `test(security): lock credit semantic permissions`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java b/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
index 4451d4f..79c5532 100644
--- a/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
@@ -1,11 +1,14 @@
 package com.sportsbook.wallet.service;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatCode;
-import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+import static org.assertj.core.api.Assertions.catchThrowable;
 
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.error.WalletAccessDeniedException;
 import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.CreditReason;
 import java.util.List;
@@ -18,18 +21,9 @@ class WalletCreditAuthorizationTest {
 
   @Test
   void acceptsTheCallerSourceAndReasonAllowlist() {
-    List<Attempt> allowed =
-        List.of(
-            new Attempt(
-                WalletCaller.BETTING, CreditCommand.Source.USER_LOCKED, CreditReason.REFUND),
-            new Attempt(
-                WalletCaller.SETTLEMENT, CreditCommand.Source.USER_LOCKED, CreditReason.VOID),
-            new Attempt(
-                WalletCaller.SETTLEMENT, CreditCommand.Source.USER_LOCKED, CreditReason.REFUND),
-            new Attempt(
-                WalletCaller.SETTLEMENT, CreditCommand.Source.HOUSE_POOL, CreditReason.PAYOUT),
-            new Attempt(WalletCaller.ADMIN, CreditCommand.Source.HOUSE_POOL, CreditReason.REFUND));
+    List<Attempt> allowed = allowed();
 
+    assertThat(allowed).hasSize(5);
     allowed.forEach(
         attempt ->
             assertThatCode(
@@ -41,27 +35,49 @@ class WalletCreditAuthorizationTest {
 
   @Test
   void rejectsEveryCallerSourceAndReasonOutsideTheAllowlist() {
-    List<Attempt> forbidden =
-        List.of(
-            new Attempt(
-                WalletCaller.PLATFORM, CreditCommand.Source.HOUSE_POOL, CreditReason.PAYOUT),
-            new Attempt(WalletCaller.GATEWAY, CreditCommand.Source.HOUSE_POOL, CreditReason.REFUND),
-            new Attempt(WalletCaller.BETTING, CreditCommand.Source.USER_LOCKED, CreditReason.VOID),
-            new Attempt(WalletCaller.BETTING, CreditCommand.Source.HOUSE_POOL, CreditReason.REFUND),
-            new Attempt(
-                WalletCaller.SETTLEMENT, CreditCommand.Source.HOUSE_POOL, CreditReason.REFUND),
-            new Attempt(
-                WalletCaller.SETTLEMENT, CreditCommand.Source.USER_LOCKED, CreditReason.PAYOUT),
-            new Attempt(WalletCaller.ADMIN, CreditCommand.Source.HOUSE_POOL, CreditReason.PAYOUT),
-            new Attempt(WalletCaller.ADMIN, CreditCommand.Source.USER_LOCKED, CreditReason.REFUND));
+    List<Attempt> allowed = allowed();
+    int rejected = 0;
+    for (WalletCaller caller : WalletCaller.values()) {
+      for (CreditCommand.Source source : CreditCommand.Source.values()) {
+        for (CreditReason reason : CreditReason.values()) {
+          Attempt attempt = new Attempt(caller, source, reason);
+          if (!allowed.contains(attempt)) {
+            Throwable error =
+                catchThrowable(
+                    () -> WalletTransferExecutor.requireAllowedCredit(caller, command(attempt)));
+            assertThat(error)
+                .isExactlyInstanceOf(WalletAccessDeniedException.class)
+                .isNotInstanceOf(IllegalArgumentException.class);
+            assertThat(((WalletAccessDeniedException) error).caller()).isEqualTo(caller);
+            rejected++;
+          }
+        }
+      }
+    }
+    assertThat(rejected).isEqualTo(25);
+  }
 
-    forbidden.forEach(
-        attempt ->
-            assertThatThrownBy(
-                    () ->
-                        WalletTransferExecutor.requireAllowedCredit(
-                            attempt.caller(), command(attempt)))
-                .isInstanceOf(RuntimeException.class));
+  @Test
+  void describesCreditCapabilityDenialsWithoutUsingArgumentErrors() {
+    WalletAccessDeniedException denied =
+        new WalletAccessDeniedException(WalletCaller.ADMIN, "credit source and reason");
+
+    assertThat(denied.caller()).isEqualTo(WalletCaller.ADMIN);
+    assertThat(denied.capability()).isEqualTo("credit source and reason");
+    assertThat(denied).isNotInstanceOf(IllegalArgumentException.class);
+    assertThatNullPointerException()
+        .isThrownBy(() -> new WalletAccessDeniedException(null, "credit"));
+    assertThatNullPointerException()
+        .isThrownBy(() -> new WalletAccessDeniedException(WalletCaller.ADMIN, null));
+  }
+
+  private static List<Attempt> allowed() {
+    return List.of(
+        new Attempt(WalletCaller.BETTING, CreditCommand.Source.USER_LOCKED, CreditReason.REFUND),
+        new Attempt(WalletCaller.SETTLEMENT, CreditCommand.Source.USER_LOCKED, CreditReason.VOID),
+        new Attempt(WalletCaller.SETTLEMENT, CreditCommand.Source.USER_LOCKED, CreditReason.REFUND),
+        new Attempt(WalletCaller.SETTLEMENT, CreditCommand.Source.HOUSE_POOL, CreditReason.PAYOUT),
+        new Attempt(WalletCaller.ADMIN, CreditCommand.Source.HOUSE_POOL, CreditReason.REFUND));
   }
 
   private static CreditCommand command(Attempt attempt) {
