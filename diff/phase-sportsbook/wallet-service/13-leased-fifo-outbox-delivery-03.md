## `feat(outbox): expose lease retry and oldest-pending metrics`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxBacklogSampler.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxBacklogSampler.java
new file mode 100644
index 0000000..f159733
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxBacklogSampler.java
@@ -0,0 +1,25 @@
+package com.sportsbook.wallet.outbox;
+
+import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+/** Refreshes gauges separately so a monitoring scrape never queries PostgreSQL. */
+@Component
+@ConditionalOnProperty(name = "wallet.outbox.scheduling-enabled", havingValue = "true")
+public class OutboxBacklogSampler {
+
+  private final OutboxDeliveryRepository delivery;
+  private final OutboxMetrics metrics;
+
+  public OutboxBacklogSampler(OutboxDeliveryRepository delivery, OutboxMetrics metrics) {
+    this.delivery = delivery;
+    this.metrics = metrics;
+  }
+
+  @Scheduled(fixedDelayString = "${wallet.outbox.metrics-interval:PT5S}")
+  public void sample() {
+    metrics.sample(delivery.snapshot());
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxMetrics.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxMetrics.java
index 89ec2c8..d242126 100644
--- a/src/main/java/com/sportsbook/wallet/outbox/OutboxMetrics.java
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxMetrics.java
@@ -1,9 +1,11 @@
 package com.sportsbook.wallet.outbox;
 
 import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.Gauge;
 import io.micrometer.core.instrument.MeterRegistry;
 import java.util.List;
 import java.util.Objects;
+import java.util.concurrent.atomic.AtomicReference;
 import org.springframework.stereotype.Component;
 
 /** Delivery counters and a scrape-safe in-memory backlog view. */
@@ -15,6 +17,8 @@ public class OutboxMetrics {
   private final Counter retried;
   private final Counter fencedCompletion;
   private final Counter leaseTakeovers;
+  private final AtomicReference<OutboxBacklogSnapshot> backlog =
+      new AtomicReference<>(OutboxBacklogSnapshot.EMPTY);
 
   public OutboxMetrics(MeterRegistry registry) {
     claimed = registry.counter("wallet.outbox.claimed");
@@ -22,6 +26,15 @@ public class OutboxMetrics {
     retried = registry.counter("wallet.outbox.retried");
     fencedCompletion = registry.counter("wallet.outbox.fenced.completion");
     leaseTakeovers = registry.counter("wallet.outbox.lease.takeovers");
+    Gauge.builder("wallet.outbox.pending", backlog, value -> value.get().pending())
+        .register(registry);
+    Gauge.builder("wallet.outbox.leased", backlog, value -> value.get().leased())
+        .register(registry);
+    Gauge.builder(
+            "wallet.outbox.oldest.pending.seconds",
+            backlog,
+            value -> value.get().oldestPendingSeconds())
+        .register(registry);
   }
 
   public void claimed(List<LeasedOutboxMessage> messages) {
@@ -38,6 +51,10 @@ public class OutboxMetrics {
     recordCompletion(retried, fenceWon);
   }
 
+  public void sample(OutboxBacklogSnapshot snapshot) {
+    backlog.set(Objects.requireNonNull(snapshot, "snapshot"));
+  }
+
   private void recordCompletion(Counter accepted, boolean fenceWon) {
     if (fenceWon) {
       accepted.increment();
