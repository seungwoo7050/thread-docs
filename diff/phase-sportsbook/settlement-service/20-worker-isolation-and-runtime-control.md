# 작업자 격리와 런타임 제어

## `feat(scheduling): isolate settlement workers`

diff --git a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
new file mode 100644
index 0000000..b28fb4c
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
@@ -0,0 +1,39 @@
+package com.sportsbook.settlement.config;
+
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.scheduling.annotation.EnableScheduling;
+import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
+
+@Configuration(proxyBeanMethods = false)
+@EnableScheduling
+public class SettlementWorkerConfiguration {
+
+  public static final String OUTBOX = "settlementOutboxScheduler";
+  public static final String LIFECYCLE = "settlementLifecycleScheduler";
+  public static final String RECOVERY = "settlementRecoveryScheduler";
+
+  @Bean(OUTBOX)
+  ThreadPoolTaskScheduler outboxScheduler() {
+    return worker("settlement-outbox-");
+  }
+
+  @Bean(LIFECYCLE)
+  ThreadPoolTaskScheduler lifecycleScheduler() {
+    return worker("settlement-lifecycle-");
+  }
+
+  @Bean(RECOVERY)
+  ThreadPoolTaskScheduler recoveryScheduler() {
+    return worker("settlement-recovery-");
+  }
+
+  private static ThreadPoolTaskScheduler worker(String threadPrefix) {
+    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
+    scheduler.setPoolSize(1);
+    scheduler.setThreadNamePrefix(threadPrefix);
+    scheduler.setWaitForTasksToCompleteOnShutdown(true);
+    scheduler.setAwaitTerminationSeconds(10);
+    return scheduler;
+  }
+}


## `feat(config): make workers runtime-switchable`

diff --git a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
index b28fb4c..2951598 100644
--- a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
+++ b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
@@ -1,5 +1,6 @@
 package com.sportsbook.settlement.config;
 
+import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.scheduling.annotation.EnableScheduling;
@@ -7,6 +8,11 @@ import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
 
 @Configuration(proxyBeanMethods = false)
 @EnableScheduling
+@ConditionalOnProperty(
+    prefix = "settlement.workers",
+    name = "enabled",
+    havingValue = "true",
+    matchIfMissing = true)
 public class SettlementWorkerConfiguration {
 
   public static final String OUTBOX = "settlementOutboxScheduler";


## `feat(workers): isolate correction schedulers`

diff --git a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
index 2951598..9e4831d 100644
--- a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
+++ b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
@@ -18,6 +18,8 @@ public class SettlementWorkerConfiguration {
   public static final String OUTBOX = "settlementOutboxScheduler";
   public static final String LIFECYCLE = "settlementLifecycleScheduler";
   public static final String RECOVERY = "settlementRecoveryScheduler";
+  public static final String REVISION_RECOVERY = "settlementRevisionRecoveryScheduler";
+  public static final String CORRECTION = "settlementCorrectionScheduler";
 
   @Bean(OUTBOX)
   ThreadPoolTaskScheduler outboxScheduler() {
@@ -34,6 +36,16 @@ public class SettlementWorkerConfiguration {
     return worker("settlement-recovery-");
   }
 
+  @Bean(REVISION_RECOVERY)
+  ThreadPoolTaskScheduler revisionRecoveryScheduler() {
+    return worker("settlement-revision-recovery-");
+  }
+
+  @Bean(CORRECTION)
+  ThreadPoolTaskScheduler correctionScheduler() {
+    return worker("settlement-correction-");
+  }
+
   private static ThreadPoolTaskScheduler worker(String threadPrefix) {
     ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
     scheduler.setPoolSize(1);


## `feat(workers): route correction workers independently`

diff --git a/src/main/java/com/sportsbook/settlement/correction/CorrectionCatchupScanner.java b/src/main/java/com/sportsbook/settlement/correction/CorrectionCatchupScanner.java
index 6977cab..653ee25 100644
--- a/src/main/java/com/sportsbook/settlement/correction/CorrectionCatchupScanner.java
+++ b/src/main/java/com/sportsbook/settlement/correction/CorrectionCatchupScanner.java
@@ -25,7 +25,7 @@ public class CorrectionCatchupScanner {
   @Scheduled(
       fixedDelayString = "${settlement.runtime.recovery-interval:PT1S}",
       initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
-      scheduler = SettlementWorkerConfiguration.RECOVERY)
+      scheduler = SettlementWorkerConfiguration.CORRECTION)
   public List<RevisionExecutionRunner.Result> catchUp() {
     var eventId = targets.findNextActionableEvent();
     if (eventId.isEmpty()) {
diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
index c995841..7304c56 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
@@ -33,7 +33,7 @@ public class RevisionRecoveryScanner {
   @Scheduled(
       fixedDelayString = "${settlement.runtime.recovery-interval:PT1S}",
       initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
-      scheduler = SettlementWorkerConfiguration.RECOVERY)
+      scheduler = SettlementWorkerConfiguration.REVISION_RECOVERY)
   public List<RevisionExecutionRunner.Result> recover() {
     var sample = metrics.start();
     try {


## `feat(workers): bound scheduler shutdown`

diff --git a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
index 9e4831d..17e4db6 100644
--- a/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
+++ b/src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java
@@ -52,6 +52,9 @@ public class SettlementWorkerConfiguration {
     scheduler.setThreadNamePrefix(threadPrefix);
     scheduler.setWaitForTasksToCompleteOnShutdown(true);
     scheduler.setAwaitTerminationSeconds(10);
+    scheduler.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
+    scheduler.setContinueExistingPeriodicTasksAfterShutdownPolicy(false);
+    scheduler.setRemoveOnCancelPolicy(true);
     return scheduler;
   }
 }


## `test(scheduling): verify worker isolation`

diff --git a/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java b/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java
new file mode 100644
index 0000000..3db21f9
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java
@@ -0,0 +1,48 @@
+package com.sportsbook.settlement.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+
+class SettlementWorkerConfigurationTest {
+
+  @Test
+  void aBlockedOutboxWorkerCannotStarveLifecycleWork() throws Exception {
+    SettlementWorkerConfiguration configuration = new SettlementWorkerConfiguration();
+    var outbox = configuration.outboxScheduler();
+    var lifecycle = configuration.lifecycleScheduler();
+    outbox.initialize();
+    lifecycle.initialize();
+    CountDownLatch outboxStarted = new CountDownLatch(1);
+    CountDownLatch releaseOutbox = new CountDownLatch(1);
+    CountDownLatch lifecycleRan = new CountDownLatch(1);
+
+    try {
+      outbox.execute(
+          () -> {
+            outboxStarted.countDown();
+            await(releaseOutbox);
+          });
+      assertThat(outboxStarted.await(1, TimeUnit.SECONDS)).isTrue();
+
+      lifecycle.execute(lifecycleRan::countDown);
+
+      assertThat(lifecycleRan.await(1, TimeUnit.SECONDS)).isTrue();
+    } finally {
+      releaseOutbox.countDown();
+      outbox.shutdown();
+      lifecycle.shutdown();
+    }
+  }
+
+  private static void await(CountDownLatch latch) {
+    try {
+      latch.await();
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+      throw new AssertionError(exception);
+    }
+  }
+}


## `test(workers): verify recovery worker isolation`

diff --git a/src/test/java/com/sportsbook/settlement/config/SettlementRecoveryIsolationTest.java b/src/test/java/com/sportsbook/settlement/config/SettlementRecoveryIsolationTest.java
new file mode 100644
index 0000000..0a7b21e
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/config/SettlementRecoveryIsolationTest.java
@@ -0,0 +1,48 @@
+package com.sportsbook.settlement.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+
+class SettlementRecoveryIsolationTest {
+
+  @Test
+  void blockedWalletWorkersCannotStarveCorrectionCatchup() throws Exception {
+    SettlementWorkerConfiguration configuration = new SettlementWorkerConfiguration();
+    var base = configuration.recoveryScheduler();
+    var revision = configuration.revisionRecoveryScheduler();
+    var correction = configuration.correctionScheduler();
+    base.initialize();
+    revision.initialize();
+    correction.initialize();
+    CountDownLatch release = new CountDownLatch(1);
+    CountDownLatch blocked = new CountDownLatch(2);
+    CountDownLatch catchupRan = new CountDownLatch(1);
+
+    try {
+      base.execute(() -> block(blocked, release));
+      revision.execute(() -> block(blocked, release));
+      assertThat(blocked.await(1, TimeUnit.SECONDS)).isTrue();
+
+      correction.execute(catchupRan::countDown);
+
+      assertThat(catchupRan.await(1, TimeUnit.SECONDS)).isTrue();
+    } finally {
+      release.countDown();
+      base.shutdown();
+      revision.shutdown();
+      correction.shutdown();
+    }
+  }
+
+  private static void block(CountDownLatch started, CountDownLatch release) {
+    started.countDown();
+    try {
+      release.await();
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+    }
+  }
+}


## `test(workers): verify correction scheduler isolation`

diff --git a/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java b/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java
index b8b3fd8..8963b51 100644
--- a/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java
+++ b/src/test/java/com/sportsbook/settlement/config/SettlementWorkerConfigurationTest.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.config;
 
 import static org.assertj.core.api.Assertions.assertThat;
 
+import java.util.List;
 import java.util.concurrent.CountDownLatch;
 import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.Test;
@@ -19,7 +20,25 @@ class SettlementWorkerConfigurationTest {
                 assertThat(context)
                     .doesNotHaveBean(SettlementWorkerConfiguration.OUTBOX)
                     .doesNotHaveBean(SettlementWorkerConfiguration.LIFECYCLE)
-                    .doesNotHaveBean(SettlementWorkerConfiguration.RECOVERY));
+                    .doesNotHaveBean(SettlementWorkerConfiguration.RECOVERY)
+                    .doesNotHaveBean(SettlementWorkerConfiguration.REVISION_RECOVERY)
+                    .doesNotHaveBean(SettlementWorkerConfiguration.CORRECTION));
+  }
+
+  @Test
+  void createsIndependentSchedulersForEachWorkerClass() {
+    new ApplicationContextRunner()
+        .withUserConfiguration(SettlementWorkerConfiguration.class)
+        .run(
+            context ->
+                assertThat(
+                        List.of(
+                            context.getBean(SettlementWorkerConfiguration.OUTBOX),
+                            context.getBean(SettlementWorkerConfiguration.LIFECYCLE),
+                            context.getBean(SettlementWorkerConfiguration.RECOVERY),
+                            context.getBean(SettlementWorkerConfiguration.REVISION_RECOVERY),
+                            context.getBean(SettlementWorkerConfiguration.CORRECTION)))
+                    .doesNotHaveDuplicates());
   }
 
   @Test
