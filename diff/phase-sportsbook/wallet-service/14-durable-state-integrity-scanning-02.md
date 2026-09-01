## `feat(integrity): scan durable wallet invariants`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScanner.java b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScanner.java
new file mode 100644
index 0000000..7d9d30a
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScanner.java
@@ -0,0 +1,61 @@
+package com.sportsbook.wallet.integrity;
+
+import java.time.Clock;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Isolation;
+import org.springframework.transaction.annotation.Transactional;
+
+/** Runs all durable wallet invariants against one repeatable database view. */
+@Service
+public class WalletIntegrityScanner {
+
+  private final AccountIntegrityRepository accounts;
+  private final OperationIntegrityRepository operations;
+  private final RecoveryQueueIntegrityRepository recovery;
+  private final AdjustmentOperationIntegrityRepository adjustmentOutcomes;
+  private final AdjustmentFailureIntegrityRepository adjustmentFailures;
+  private final AdjustmentFingerprintIntegrityRepository adjustmentFingerprints;
+  private final AdjustmentLedgerIntegrityRepository adjustmentLedgers;
+  private final Clock clock;
+
+  public WalletIntegrityScanner(
+      AccountIntegrityRepository accounts,
+      OperationIntegrityRepository operations,
+      RecoveryQueueIntegrityRepository recovery,
+      AdjustmentOperationIntegrityRepository adjustmentOutcomes,
+      AdjustmentFailureIntegrityRepository adjustmentFailures,
+      AdjustmentFingerprintIntegrityRepository adjustmentFingerprints,
+      AdjustmentLedgerIntegrityRepository adjustmentLedgers,
+      Clock clock) {
+    this.accounts = accounts;
+    this.operations = operations;
+    this.recovery = recovery;
+    this.adjustmentOutcomes = adjustmentOutcomes;
+    this.adjustmentFailures = adjustmentFailures;
+    this.adjustmentFingerprints = adjustmentFingerprints;
+    this.adjustmentLedgers = adjustmentLedgers;
+    this.clock = clock;
+  }
+
+  @Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
+  public WalletIntegritySnapshot scan() {
+    long accountSnapshotDrift = accounts.findSnapshotDrift().size();
+    long orphanAccountLedgers = accounts.findOrphanLedgerAccountIds().size();
+    long operationGroupDrift = operations.findGroupDriftKeys().size();
+    long recoveryQueueDrift = recovery.findQueueDriftUsers().size();
+    long adjustmentOutcomeDrift = adjustmentOutcomes.findOutcomeDriftKeys().size();
+    long adjustmentFailureDrift = adjustmentFailures.findFailureDriftKeys().size();
+    long adjustmentFingerprintDrift = adjustmentFingerprints.findFingerprintDriftKeys().size();
+    long adjustmentLedgerDrift = adjustmentLedgers.findLedgerDriftKeys().size();
+    return new WalletIntegritySnapshot(
+        clock.instant(),
+        accountSnapshotDrift,
+        orphanAccountLedgers,
+        operationGroupDrift,
+        recoveryQueueDrift,
+        adjustmentOutcomeDrift,
+        adjustmentFailureDrift,
+        adjustmentFingerprintDrift,
+        adjustmentLedgerDrift);
+  }
+}


## `test(integrity): combine wallet invariant scans`

diff --git a/src/test/java/com/sportsbook/wallet/integrity/WalletIntegrityScannerTest.java b/src/test/java/com/sportsbook/wallet/integrity/WalletIntegrityScannerTest.java
new file mode 100644
index 0000000..2df5440
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/integrity/WalletIntegrityScannerTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.wallet.integrity;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.Collections;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+class WalletIntegrityScannerTest {
+
+  @Test
+  void combinesEveryInvariantIntoOneIdentifierFreeSnapshot() {
+    AccountIntegrityRepository accounts = mock(AccountIntegrityRepository.class);
+    OperationIntegrityRepository operations = mock(OperationIntegrityRepository.class);
+    RecoveryQueueIntegrityRepository recovery = mock(RecoveryQueueIntegrityRepository.class);
+    AdjustmentOperationIntegrityRepository outcomes =
+        mock(AdjustmentOperationIntegrityRepository.class);
+    AdjustmentFailureIntegrityRepository failures =
+        mock(AdjustmentFailureIntegrityRepository.class);
+    AdjustmentFingerprintIntegrityRepository fingerprints =
+        mock(AdjustmentFingerprintIntegrityRepository.class);
+    AdjustmentLedgerIntegrityRepository ledgers = mock(AdjustmentLedgerIntegrityRepository.class);
+    when(accounts.findSnapshotDrift()).thenReturn(entries(1));
+    when(accounts.findOrphanLedgerAccountIds()).thenReturn(entries(2));
+    when(operations.findGroupDriftKeys()).thenReturn(entries(3));
+    when(recovery.findQueueDriftUsers()).thenReturn(entries(4));
+    when(outcomes.findOutcomeDriftKeys()).thenReturn(entries(5));
+    when(failures.findFailureDriftKeys()).thenReturn(entries(6));
+    when(fingerprints.findFingerprintDriftKeys()).thenReturn(entries(7));
+    when(ledgers.findLedgerDriftKeys()).thenReturn(entries(8));
+    Instant checkedAt = Instant.parse("2026-08-21T11:00:00Z");
+    WalletIntegrityScanner scanner =
+        new WalletIntegrityScanner(
+            accounts,
+            operations,
+            recovery,
+            outcomes,
+            failures,
+            fingerprints,
+            ledgers,
+            Clock.fixed(checkedAt, ZoneOffset.UTC));
+
+    WalletIntegritySnapshot snapshot = scanner.scan();
+
+    assertThat(snapshot.lastCheckedAt()).isEqualTo(checkedAt);
+    assertThat(snapshot.accountSnapshotDrift()).isEqualTo(1);
+    assertThat(snapshot.orphanAccountLedgers()).isEqualTo(2);
+    assertThat(snapshot.operationGroupDrift()).isEqualTo(3);
+    assertThat(snapshot.recoveryQueueDrift()).isEqualTo(4);
+    assertThat(snapshot.adjustmentOutcomeDrift()).isEqualTo(5);
+    assertThat(snapshot.adjustmentFailureDrift()).isEqualTo(6);
+    assertThat(snapshot.adjustmentFingerprintDrift()).isEqualTo(7);
+    assertThat(snapshot.adjustmentLedgerDrift()).isEqualTo(8);
+  }
+
+  private static <T> List<T> entries(int count) {
+    return Collections.nCopies(count, null);
+  }
+}


## `feat(integrity): summarize scan drift counts`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/WalletIntegritySnapshot.java b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegritySnapshot.java
new file mode 100644
index 0000000..7772579
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegritySnapshot.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.integrity;
+
+import java.time.Instant;
+import java.util.Objects;
+
+/** Immutable, identifier-free summary of the last completed integrity scan. */
+public record WalletIntegritySnapshot(
+    Instant lastCheckedAt,
+    long accountSnapshotDrift,
+    long orphanAccountLedgers,
+    long operationGroupDrift,
+    long recoveryQueueDrift,
+    long adjustmentOutcomeDrift,
+    long adjustmentFailureDrift,
+    long adjustmentFingerprintDrift,
+    long adjustmentLedgerDrift) {
+
+  public WalletIntegritySnapshot {
+    Objects.requireNonNull(lastCheckedAt, "lastCheckedAt");
+    if (accountSnapshotDrift < 0
+        || orphanAccountLedgers < 0
+        || operationGroupDrift < 0
+        || recoveryQueueDrift < 0
+        || adjustmentOutcomeDrift < 0
+        || adjustmentFailureDrift < 0
+        || adjustmentFingerprintDrift < 0
+        || adjustmentLedgerDrift < 0) {
+      throw new IllegalArgumentException("integrity drift counts cannot be negative");
+    }
+  }
+
+  public long totalDrift() {
+    return accountSnapshotDrift
+        + orphanAccountLedgers
+        + operationGroupDrift
+        + recoveryQueueDrift
+        + adjustmentOutcomeDrift
+        + adjustmentFailureDrift
+        + adjustmentFingerprintDrift
+        + adjustmentLedgerDrift;
+  }
+
+  public boolean hasDrift() {
+    return totalDrift() > 0;
+  }
+}


## `feat(integrity): publish scan metrics`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityMetrics.java b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityMetrics.java
new file mode 100644
index 0000000..3ab97ba
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityMetrics.java
@@ -0,0 +1,71 @@
+package com.sportsbook.wallet.integrity;
+
+import io.micrometer.core.instrument.Gauge;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.Map;
+import java.util.Objects;
+import java.util.concurrent.atomic.AtomicReference;
+import java.util.function.ToDoubleFunction;
+import org.springframework.stereotype.Component;
+
+/** Publishes bounded, scrape-safe gauges from the last completed integrity scan. */
+@Component
+public class WalletIntegrityMetrics {
+
+  private static final Map<String, ToDoubleFunction<WalletIntegritySnapshot>> DRIFT_GAUGES =
+      Map.of(
+          "wallet.integrity.account.snapshot.drift", WalletIntegritySnapshot::accountSnapshotDrift,
+          "wallet.integrity.account.orphan.ledgers", WalletIntegritySnapshot::orphanAccountLedgers,
+          "wallet.integrity.operation.group.drift", WalletIntegritySnapshot::operationGroupDrift,
+          "wallet.integrity.recovery.queue.drift", WalletIntegritySnapshot::recoveryQueueDrift,
+          "wallet.integrity.adjustment.outcome.drift",
+              WalletIntegritySnapshot::adjustmentOutcomeDrift,
+          "wallet.integrity.adjustment.failure.drift",
+              WalletIntegritySnapshot::adjustmentFailureDrift,
+          "wallet.integrity.adjustment.fingerprint.drift",
+              WalletIntegritySnapshot::adjustmentFingerprintDrift,
+          "wallet.integrity.adjustment.ledger.drift",
+              WalletIntegritySnapshot::adjustmentLedgerDrift,
+          "wallet.integrity.total.drift", WalletIntegritySnapshot::totalDrift);
+
+  private final AtomicReference<Status> state = new AtomicReference<>(new Status(null, false));
+
+  public WalletIntegrityMetrics(MeterRegistry registry) {
+    DRIFT_GAUGES.forEach((name, valueFunction) -> gauge(registry, name, valueFunction));
+    Gauge.builder("wallet.integrity.scan.failed", state, value -> value.get().scanFailed() ? 1 : 0)
+        .register(registry);
+    Gauge.builder(
+            "wallet.integrity.last.checked.epoch.seconds",
+            state,
+            value ->
+                value.get().snapshot() == null
+                    ? 0
+                    : value.get().snapshot().lastCheckedAt().getEpochSecond())
+        .register(registry);
+  }
+
+  public void record(WalletIntegritySnapshot snapshot) {
+    state.set(new Status(Objects.requireNonNull(snapshot, "snapshot"), false));
+  }
+
+  public void recordFailure() {
+    state.updateAndGet(previous -> new Status(previous.snapshot(), true));
+  }
+
+  Status status() {
+    return state.get();
+  }
+
+  private void gauge(
+      MeterRegistry registry,
+      String name,
+      ToDoubleFunction<WalletIntegritySnapshot> valueFunction) {
+    Gauge.builder(name, state, value -> metric(value.get(), valueFunction)).register(registry);
+  }
+
+  private double metric(Status current, ToDoubleFunction<WalletIntegritySnapshot> valueFunction) {
+    return current.snapshot() == null ? 0 : valueFunction.applyAsDouble(current.snapshot());
+  }
+
+  record Status(WalletIntegritySnapshot snapshot, boolean scanFailed) {}
+}


## `feat(integrity): report scan health`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityHealth.java b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityHealth.java
new file mode 100644
index 0000000..366e5f2
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityHealth.java
@@ -0,0 +1,33 @@
+package com.sportsbook.wallet.integrity;
+
+import org.springframework.boot.actuate.health.Health;
+import org.springframework.boot.actuate.health.HealthIndicator;
+import org.springframework.stereotype.Component;
+
+/** Degrades health only for detected drift or a failed integrity scan. */
+@Component
+public class WalletIntegrityHealth implements HealthIndicator {
+
+  private final WalletIntegrityMetrics metrics;
+
+  public WalletIntegrityHealth(WalletIntegrityMetrics metrics) {
+    this.metrics = metrics;
+  }
+
+  @Override
+  public Health health() {
+    WalletIntegrityMetrics.Status status = metrics.status();
+    if (status.scanFailed()) {
+      return Health.down().withDetail("reason", "integrity_scan_failed").build();
+    }
+    if (status.snapshot() == null) {
+      return Health.unknown().withDetail("reason", "integrity_not_checked").build();
+    }
+    WalletIntegritySnapshot snapshot = status.snapshot();
+    Health.Builder health = snapshot.hasDrift() ? Health.down() : Health.up();
+    return health
+        .withDetail("lastCheckedAt", snapshot.lastCheckedAt())
+        .withDetail("driftCount", snapshot.totalDrift())
+        .build();
+  }
+}


## `feat(integrity): schedule wallet scans`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScheduler.java b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScheduler.java
new file mode 100644
index 0000000..f5e75d9
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/WalletIntegrityScheduler.java
@@ -0,0 +1,29 @@
+package com.sportsbook.wallet.integrity;
+
+import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+/** Periodically refreshes the cached integrity view without hiding database failures. */
+@Component
+@ConditionalOnProperty(name = "wallet.integrity.scheduling-enabled", havingValue = "true")
+public class WalletIntegrityScheduler {
+
+  private final WalletIntegrityScanner scanner;
+  private final WalletIntegrityMetrics metrics;
+
+  public WalletIntegrityScheduler(WalletIntegrityScanner scanner, WalletIntegrityMetrics metrics) {
+    this.scanner = scanner;
+    this.metrics = metrics;
+  }
+
+  @Scheduled(fixedDelayString = "${wallet.integrity.poll-interval:PT30S}")
+  public void scan() {
+    try {
+      metrics.record(scanner.scan());
+    } catch (RuntimeException failure) {
+      metrics.recordFailure();
+      throw failure;
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 24702bf..359af9c 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -70,6 +70,9 @@ logging:
     org.hibernate.SQL: WARN
 
 wallet:
+  integrity:
+    scheduling-enabled: ${WALLET_INTEGRITY_ENABLED:true}
+    poll-interval: ${WALLET_INTEGRITY_POLL_INTERVAL:PT30S}
   outbox:
     scheduling-enabled: ${WALLET_OUTBOX_ENABLED:false}
   recovery:


## `test(integrity): verify scheduled scan outcomes`

diff --git a/src/test/java/com/sportsbook/wallet/integrity/WalletIntegritySchedulerTest.java b/src/test/java/com/sportsbook/wallet/integrity/WalletIntegritySchedulerTest.java
new file mode 100644
index 0000000..a511f70
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/integrity/WalletIntegritySchedulerTest.java
@@ -0,0 +1,67 @@
+package com.sportsbook.wallet.integrity;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.io.IOException;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.env.YamlPropertySourceLoader;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.core.io.ClassPathResource;
+
+class WalletIntegritySchedulerTest {
+
+  @Test
+  void productionConfigurationEnablesPeriodicScans() throws IOException {
+    var properties =
+        new YamlPropertySourceLoader()
+            .load("wallet", new ClassPathResource("application.yml"))
+            .get(0);
+
+    assertThat(properties.getProperty("wallet.integrity.scheduling-enabled"))
+        .isEqualTo("${WALLET_INTEGRITY_ENABLED:true}");
+    assertThat(properties.getProperty("wallet.integrity.poll-interval"))
+        .isEqualTo("${WALLET_INTEGRITY_POLL_INTERVAL:PT30S}");
+  }
+
+  @Test
+  void schedulingPropertyControlsTheScanner() {
+    ApplicationContextRunner context =
+        new ApplicationContextRunner()
+            .withBean(WalletIntegrityScanner.class, () -> mock(WalletIntegrityScanner.class))
+            .withBean(WalletIntegrityMetrics.class, () -> mock(WalletIntegrityMetrics.class))
+            .withUserConfiguration(WalletIntegrityScheduler.class);
+
+    context
+        .withPropertyValues("wallet.integrity.scheduling-enabled=true")
+        .run(
+            enabled ->
+                assertThat(enabled.getBeansOfType(WalletIntegrityScheduler.class)).hasSize(1));
+    context
+        .withPropertyValues("wallet.integrity.scheduling-enabled=false")
+        .run(
+            disabled ->
+                assertThat(disabled.getBeansOfType(WalletIntegrityScheduler.class)).isEmpty());
+  }
+
+  @Test
+  void recordsCompletedAndFailedScans() {
+    WalletIntegrityScanner scanner = mock(WalletIntegrityScanner.class);
+    WalletIntegrityMetrics metrics = new WalletIntegrityMetrics(new SimpleMeterRegistry());
+    WalletIntegrityScheduler scheduler = new WalletIntegrityScheduler(scanner, metrics);
+    WalletIntegritySnapshot snapshot =
+        new WalletIntegritySnapshot(Instant.parse("2026-08-21T14:00:00Z"), 0, 0, 0, 0, 0, 0, 0, 0);
+    IllegalStateException failure = new IllegalStateException("database unavailable");
+    when(scanner.scan()).thenReturn(snapshot).thenThrow(failure);
+
+    scheduler.scan();
+    assertThat(metrics.status()).isEqualTo(new WalletIntegrityMetrics.Status(snapshot, false));
+
+    assertThatThrownBy(scheduler::scan).isSameAs(failure);
+    assertThat(metrics.status()).isEqualTo(new WalletIntegrityMetrics.Status(snapshot, true));
+  }
+}
