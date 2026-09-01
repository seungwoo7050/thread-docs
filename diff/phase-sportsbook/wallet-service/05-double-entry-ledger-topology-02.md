## `feat(service): validate every final transfer topology`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
index 67113b7..b680a5a 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
@@ -1,6 +1,7 @@
 package com.sportsbook.wallet.service;
 
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.BalanceBucket;
 import com.sportsbook.wallet.domain.LedgerEntry;
 import com.sportsbook.wallet.domain.LedgerReason;
 import com.sportsbook.wallet.domain.LedgerSide;
@@ -51,11 +52,52 @@ public record WalletOperationResult(
           "Ledger result must identify exactly one user account (got " + userIds.size() + ")");
     }
 
+    UUID userId = userIds.iterator().next();
+    LedgerEntry debit =
+        pair.stream().filter(entry -> entry.side() == LedgerSide.DEBIT).findFirst().orElseThrow();
+    LedgerEntry credit =
+        pair.stream().filter(entry -> entry.side() == LedgerSide.CREDIT).findFirst().orElseThrow();
+    validateTopology(first.reason(), userId, debit, credit);
+
     return new WalletOperationResult(
-        first.operationGroupId(),
-        userIds.iterator().next(),
-        first.money(),
-        first.reason(),
-        first.createdAt());
+        first.operationGroupId(), userId, first.money(), first.reason(), first.createdAt());
+  }
+
+  private static void validateTopology(
+      LedgerReason reason, UUID userId, LedgerEntry debit, LedgerEntry credit) {
+    boolean valid =
+        switch (reason) {
+          case DEPOSIT ->
+              matches(debit, userId, BalanceBucket.AVAILABLE)
+                  && matches(credit, SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE);
+          case WITHDRAW ->
+              matches(debit, SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE)
+                  && matches(credit, userId, BalanceBucket.AVAILABLE);
+          case BET_DEBIT ->
+              matches(debit, userId, BalanceBucket.LOCKED)
+                  && matches(credit, userId, BalanceBucket.AVAILABLE);
+          case BET_PAYOUT ->
+              matches(debit, userId, BalanceBucket.AVAILABLE)
+                  && matches(credit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE);
+          case BET_REFUND ->
+              matches(debit, userId, BalanceBucket.AVAILABLE)
+                  && matches(credit, userId, BalanceBucket.LOCKED);
+          case BET_FORFEIT ->
+              matches(debit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE)
+                  && matches(credit, userId, BalanceBucket.LOCKED);
+          case BET_ADJUSTMENT ->
+              (matches(debit, userId, BalanceBucket.AVAILABLE)
+                      && matches(credit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE))
+                  || (matches(debit, SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE)
+                      && matches(credit, userId, BalanceBucket.AVAILABLE));
+        };
+    if (!valid) {
+      throw new IllegalStateException(
+          "Ledger result topology does not match reason " + reason.name());
+    }
+  }
+
+  private static boolean matches(LedgerEntry entry, UUID accountId, BalanceBucket bucket) {
+    return entry.accountId().equals(accountId) && entry.bucket() == bucket;
   }
 }


## `test(service): reject reason topology mismatches`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
index 5409ba4..201ec39 100644
--- a/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.wallet.service;
 
 import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
@@ -69,6 +70,26 @@ class WalletTransferTopologyTest {
         .doesNotThrowAnyException();
   }
 
+  @Test
+  void rejectsCounterpartyAndBucketMismatches() {
+    List<Topology> invalid =
+        List.of(
+            new Topology(
+                LedgerReason.DEPOSIT,
+                USER,
+                BalanceBucket.AVAILABLE,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.BET_DEBIT, USER, BalanceBucket.AVAILABLE, USER, BalanceBucket.LOCKED));
+
+    invalid.forEach(
+        topology ->
+            assertThatThrownBy(() -> WalletOperationResult.fromExisting(topology.entries()))
+                .isInstanceOf(IllegalStateException.class)
+                .hasMessageContaining("topology does not match reason"));
+  }
+
   private record Topology(
       LedgerReason reason,
       UUID destination,


## `test(service): cover deposit withdrawal betting and adjustment topology`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
new file mode 100644
index 0000000..5409ba4
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/service/WalletTransferTopologyTest.java
@@ -0,0 +1,92 @@
+package com.sportsbook.wallet.service;
+
+import static org.assertj.core.api.Assertions.assertThatCode;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.BalanceBucket;
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class WalletTransferTopologyTest {
+
+  private static final UUID USER = UUID.fromString("019b76da-a000-7000-8000-000000000111");
+
+  @Test
+  void acceptsEveryFinalTransferTopology() {
+    List<Topology> topologies =
+        List.of(
+            new Topology(
+                LedgerReason.DEPOSIT,
+                USER,
+                BalanceBucket.AVAILABLE,
+                SystemAccountIds.EXTERNAL_PAYMENT,
+                BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.WITHDRAW,
+                SystemAccountIds.EXTERNAL_PAYMENT,
+                BalanceBucket.AVAILABLE,
+                USER,
+                BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.BET_DEBIT, USER, BalanceBucket.LOCKED, USER, BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.BET_PAYOUT,
+                USER,
+                BalanceBucket.AVAILABLE,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.BET_REFUND, USER, BalanceBucket.AVAILABLE, USER, BalanceBucket.LOCKED),
+            new Topology(
+                LedgerReason.BET_FORFEIT,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE,
+                USER,
+                BalanceBucket.LOCKED),
+            new Topology(
+                LedgerReason.BET_ADJUSTMENT,
+                USER,
+                BalanceBucket.AVAILABLE,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE),
+            new Topology(
+                LedgerReason.BET_ADJUSTMENT,
+                SystemAccountIds.HOUSE,
+                BalanceBucket.AVAILABLE,
+                USER,
+                BalanceBucket.AVAILABLE));
+
+    assertThatCode(
+            () ->
+                topologies.forEach(
+                    topology -> WalletOperationResult.fromExisting(topology.entries())))
+        .doesNotThrowAnyException();
+  }
+
+  private record Topology(
+      LedgerReason reason,
+      UUID destination,
+      BalanceBucket destinationBucket,
+      UUID source,
+      BalanceBucket sourceBucket) {
+
+    private List<LedgerEntry> entries() {
+      LedgerEntry.Pair pair =
+          LedgerEntry.pair(
+              new LedgerEntry.TransferLeg(destination, destinationBucket),
+              new LedgerEntry.TransferLeg(source, sourceBucket),
+              Money.krw(10_000L),
+              reason,
+              IdempotencyKey.of("topology:" + reason),
+              UUID.fromString("019b76da-a000-7000-8000-000000000211"),
+              Instant.parse("2026-07-14T00:00:00Z"));
+      return List.of(pair.debit(), pair.credit());
+    }
+  }
+}
