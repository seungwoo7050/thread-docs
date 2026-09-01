## `test(wallet): reject non-authoritative success statuses`

diff --git a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
index e36c212..011f4d3 100644
--- a/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/WalletClientTest.java
@@ -157,6 +157,34 @@ class WalletClientTest {
         .isEqualTo("refund");
   }
 
+  @Test
+  void acceptsOnlyOkForWalletProofs() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    for (HttpStatus status :
+        java.util.List.of(
+            HttpStatus.CREATED, HttpStatus.ACCEPTED, HttpStatus.NO_CONTENT, HttpStatus.FOUND)) {
+      server
+          .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit"))
+          .andRespond(withStatus(status));
+      assertThatThrownBy(() -> client.debit(betId, userId, Money.krw(1_000)))
+          .isInstanceOf(com.sportsbook.betting.error.DependencyUnavailableException.class);
+      server.reset();
+      server
+          .expect(requestTo("http://wallet/internal/v1/wallet/transactions/debit/" + betId))
+          .andRespond(withStatus(status));
+      assertThatThrownBy(() -> client.findDebit(betId, userId, Money.krw(1_000)))
+          .isInstanceOf(com.sportsbook.betting.error.DependencyUnavailableException.class);
+      server.reset();
+      server
+          .expect(requestTo("http://wallet/internal/v1/wallet/transactions/credit"))
+          .andRespond(withStatus(status));
+      assertThatThrownBy(() -> client.refund(betId, userId, Money.krw(1_000)))
+          .isInstanceOf(com.sportsbook.betting.error.DependencyUnavailableException.class);
+      server.reset();
+    }
+  }
+
   private static String proof(UUID operationId, UUID userId, long amount, String reason) {
     return "{\"operationGroupId\":\""
         + operationId
