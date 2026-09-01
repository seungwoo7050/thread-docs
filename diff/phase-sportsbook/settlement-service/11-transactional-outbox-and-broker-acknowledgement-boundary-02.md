## `test(outbox): verify PostgreSQL skip locked claims`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresOutboxLockIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresOutboxLockIntegrationTest.java
new file mode 100644
index 0000000..5f34cb0
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresOutboxLockIntegrationTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.settlement.outbox.OutboxEvent;
+import com.sportsbook.settlement.outbox.OutboxEventRepository;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.support.TransactionTemplate;
+
+class PostgresOutboxLockIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private OutboxEventRepository outbox;
+  @Autowired private PlatformTransactionManager transactionManager;
+
+  @Test
+  void skipsRowsHeldByAnotherPublisherTransaction() throws Exception {
+    TransactionTemplate transactions = new TransactionTemplate(transactionManager);
+    OutboxEvent first = pending("first");
+    OutboxEvent second = pending("second");
+    transactions.executeWithoutResult(ignored -> outbox.saveAllAndFlush(List.of(first, second)));
+    CountDownLatch firstLocked = new CountDownLatch(1);
+    CountDownLatch releaseFirst = new CountDownLatch(1);
+    var workers = Executors.newSingleThreadExecutor();
+
+    try {
+      var held =
+          workers.submit(
+              () ->
+                  transactions.execute(
+                      ignored -> {
+                        UUID id = outbox.lockNextUnpublished(1).get(0).eventId();
+                        firstLocked.countDown();
+                        await(releaseFirst);
+                        return id;
+                      }));
+      assertThat(firstLocked.await(2, TimeUnit.SECONDS)).isTrue();
+
+      UUID skipped =
+          transactions.execute(ignored -> outbox.lockNextUnpublished(1).get(0).eventId());
+      releaseFirst.countDown();
+
+      assertThat(List.of(held.get(2, TimeUnit.SECONDS), skipped))
+          .containsExactlyInAnyOrder(first.eventId(), second.eventId());
+    } finally {
+      releaseFirst.countDown();
+      workers.shutdownNow();
+    }
+  }
+
+  private static OutboxEvent pending(String key) {
+    return OutboxEvent.pending("bet.settled.v1", key, "BetSettled", new byte[] {1}, Instant.EPOCH);
+  }
+
+  private static void await(CountDownLatch latch) {
+    try {
+      if (!latch.await(2, TimeUnit.SECONDS)) {
+        throw new AssertionError("Timed out waiting for publisher release");
+      }
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+      throw new AssertionError(exception);
+    }
+  }
+}


## `feat(outbox): create revision events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
index fc31957..156c876 100644
--- a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
+++ b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
@@ -1,13 +1,16 @@
 package com.sportsbook.settlement.outbox;
 
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
 import com.sportsbook.protocol.event.VoidReason;
 import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.correction.RevisionPlan;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.BetSelection;
 import com.sportsbook.settlement.domain.SettlementStatus;
+import java.time.Instant;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.UUID;
@@ -67,6 +70,30 @@ public final class SettlementEventFactory {
         bet.settledAt());
   }
 
+  public OutboxEvent revised(RevisionPlan plan, Instant revisedAt) {
+    var target = plan.target();
+    BetResolutionRevised event =
+        BetResolutionRevised.newBuilder()
+            .setRevisionId(plan.revisionId().toString())
+            .setRevisionNumber(target.revisionNumber())
+            .setBetId(target.betId().toString())
+            .setUserId(target.userId().toString())
+            .setEventId(target.eventId().toString())
+            .setPreviousResult(SettlementResultAvro.valueOf(target.previousResult().name()))
+            .setNewResult(SettlementResultAvro.valueOf(plan.newResult().name()))
+            .setPreviousPayout(money(target.previousPayout()))
+            .setNewPayout(money(plan.newPayout()))
+            .setSourceResultSettledAt(target.sourceResultSettledAt())
+            .setRevisedAt(revisedAt)
+            .build();
+    return OutboxEvent.pending(
+        topics.betResolutionRevised(),
+        target.betId().toString(),
+        BetResolutionRevised.class.getSimpleName(),
+        encoder.encode(event),
+        revisedAt);
+  }
+
   private static com.sportsbook.protocol.event.Money money(
       com.sportsbook.protocol.value.Money value) {
     return com.sportsbook.protocol.event.Money.newBuilder()


## `test(outbox): verify revision event contract`

diff --git a/src/test/java/com/sportsbook/settlement/outbox/RevisionEventFactoryTest.java b/src/test/java/com/sportsbook/settlement/outbox/RevisionEventFactoryTest.java
new file mode 100644
index 0000000..1cd3745
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/outbox/RevisionEventFactoryTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.settlement.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.correction.RevisionPlan;
+import com.sportsbook.settlement.correction.RevisionTarget;
+import com.sportsbook.settlement.event.StrictAvroDecoder;
+import com.sportsbook.settlement.resolver.ResolvedSelection;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RevisionEventFactoryTest {
+
+  @Test
+  void preservesRevisionIdentityAndUsesBetIdAsTheKafkaKey() {
+    RevisionPlan plan = plan();
+    Instant revisedAt = Instant.EPOCH.plusSeconds(2);
+    SettlementEventFactory factory =
+        new SettlementEventFactory(
+            new SettlementTopics(null, null, null, null, null, null), new StrictAvroEncoder());
+
+    OutboxEvent outbox = factory.revised(plan, revisedAt);
+    BetResolutionRevised event =
+        new StrictAvroDecoder().decode(outbox.payload(), BetResolutionRevised.class);
+
+    assertThat(outbox.topic()).isEqualTo("bet.resolution.revised.v1");
+    assertThat(outbox.partitionKey()).isEqualTo(plan.target().betId().toString());
+    assertThat(event.getRevisionId()).isEqualTo(plan.revisionId().toString());
+    assertThat(event.getRevisionNumber()).isEqualTo(1);
+    assertThat(event.getPreviousResult()).isEqualTo(SettlementResultAvro.WON);
+    assertThat(event.getNewResult()).isEqualTo(SettlementResultAvro.PUSH);
+    assertThat(event.getPreviousPayout().getAmount()).isEqualTo(200);
+    assertThat(event.getNewPayout().getAmount()).isEqualTo(100);
+    assertThat(event.getSourceResultSettledAt()).isEqualTo(Instant.EPOCH.plusSeconds(1));
+    assertThat(event.getRevisedAt()).isEqualTo(revisedAt);
+  }
+
+  private static RevisionPlan plan() {
+    RevisionTarget target =
+        new RevisionTarget(
+            UUID.randomUUID(),
+            1,
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            SettlementResult.WON,
+            Money.krw(200),
+            new BetSlipType.Single(),
+            Money.krw(100),
+            List.of(
+                new ResolvedSelection(
+                    UUID.randomUUID(), Odds.ofDecimal("2.0000"), SettlementResult.PUSH)),
+            Instant.EPOCH.plusSeconds(1));
+    return new RevisionPlan(
+        UUID.randomUUID(), target, SettlementResult.PUSH, Money.krw(100), Instant.EPOCH);
+  }
+}
