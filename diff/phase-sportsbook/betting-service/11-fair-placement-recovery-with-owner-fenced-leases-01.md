# 소유자 펜싱 리스를 이용한 공정한 접수 복구

## `feat(placement): resume stale placement checkpoints`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java b/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java
new file mode 100644
index 0000000..dae6053
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java
@@ -0,0 +1,53 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.protocol.domain.BetStatus;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.data.domain.PageRequest;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetReconciliationJob {
+
+  private static final Logger log = LoggerFactory.getLogger(BetReconciliationJob.class);
+  static final int BATCH_SIZE = 100;
+
+  private final BetRepository bets;
+  private final BetPlacementService placement;
+  private final Clock clock;
+  private final Duration pendingTimeout;
+
+  public BetReconciliationJob(
+      BetRepository bets,
+      BetPlacementService placement,
+      Clock clock,
+      @Value("${betting.reconciliation.pending-timeout:30s}") Duration pendingTimeout) {
+    this.bets = bets;
+    this.placement = placement;
+    this.clock = clock;
+    this.pendingTimeout = pendingTimeout;
+  }
+
+  @Scheduled(fixedDelayString = "${betting.reconciliation.poll-interval-ms:10000}")
+  public void reconcile() {
+    Instant threshold = clock.instant().minus(pendingTimeout);
+    List<Bet> stale =
+        bets.findByStatusAndCreatedAtBefore(
+            BetStatus.PENDING, threshold, PageRequest.of(0, BATCH_SIZE));
+    for (Bet bet : stale) {
+      try {
+        placement.reconcile(bet.betId());
+      } catch (RuntimeException unexpected) {
+        log.error("Placement reconciliation failed for bet {}", bet.betId(), unexpected);
+      }
+    }
+  }
+}


## `test(placement): verify bounded checkpoint recovery`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java b/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java
new file mode 100644
index 0000000..54d8e96
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.betting.placement;
+
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.protocol.domain.BetStatus;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.PageRequest;
+
+class BetReconciliationJobTest {
+
+  @Test
+  void resumesOnlyStalePendingBetsInBoundedBatches() {
+    BetRepository bets = mock(BetRepository.class);
+    BetPlacementService placement = mock(BetPlacementService.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    Instant now = Instant.parse("2026-08-22T00:00:00Z");
+    when(bet.betId()).thenReturn(betId);
+    when(bets.findByStatusAndCreatedAtBefore(
+            BetStatus.PENDING,
+            now.minusSeconds(30),
+            PageRequest.of(0, BetReconciliationJob.BATCH_SIZE)))
+        .thenReturn(List.of(bet));
+
+    new BetReconciliationJob(
+            bets, placement, Clock.fixed(now, ZoneOffset.UTC), Duration.ofSeconds(30))
+        .reconcile();
+
+    verify(placement).reconcile(betId);
+  }
+}


## `feat(recovery): add owner-fenced reconciliation leases`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 87e5a86..5d051ad 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -158,6 +158,15 @@ public class Bet {
   @Column(name = "reconciliation_requested_at")
   private Instant reconciliationRequestedAt;
 
+  @Column(name = "reconciliation_eligible_at")
+  private Instant reconciliationEligibleAt;
+
+  @Column(name = "reconciliation_claim_owner", length = 128)
+  private String reconciliationClaimOwner;
+
+  @Column(name = "reconciliation_claim_until")
+  private Instant reconciliationClaimUntil;
+
   @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
   @OrderBy("legIndex ASC")
   private List<BetLeg> legs = new ArrayList<>();
@@ -600,12 +609,17 @@ public class Bet {
 
   public void requestReconciliation(Instant at) {
     reconciliationRequestedAt = Objects.requireNonNull(at, "at");
+    reconciliationEligibleAt = at;
   }
 
   public Instant reconciliationRequestedAt() {
     return reconciliationRequestedAt;
   }
 
+  public Instant reconciliationEligibleAt() {
+    return reconciliationEligibleAt;
+  }
+
   public List<BetLeg> legs() {
     return List.copyOf(legs);
   }
diff --git a/src/main/resources/db/migration/V10__reconciliation_lease.sql b/src/main/resources/db/migration/V10__reconciliation_lease.sql
new file mode 100644
index 0000000..183323c
--- /dev/null
+++ b/src/main/resources/db/migration/V10__reconciliation_lease.sql
@@ -0,0 +1,28 @@
+-- Claim stale placement recovery exclusively without changing user-visible timestamps.
+ALTER TABLE bet
+    ADD COLUMN reconciliation_eligible_at TIMESTAMP WITH TIME ZONE,
+    ADD COLUMN reconciliation_claim_owner VARCHAR(128),
+    ADD COLUMN reconciliation_claim_until TIMESTAMP WITH TIME ZONE;
+
+ALTER TABLE bet
+    ADD CONSTRAINT bet_reconciliation_claim_pair CHECK (
+        (reconciliation_claim_owner IS NULL AND reconciliation_claim_until IS NULL)
+        OR
+        (reconciliation_claim_owner IS NOT NULL AND reconciliation_claim_until IS NOT NULL)
+    );
+
+CREATE INDEX ix_bet_reconciliation_claim
+    ON bet (
+        reconciliation_eligible_at,
+        reconciliation_requested_at,
+        created_at,
+        bet_id
+    )
+    WHERE status = 'PENDING';
+
+COMMENT ON COLUMN bet.reconciliation_eligible_at IS
+    'Database-time retry eligibility advanced when a worker claims recovery.';
+COMMENT ON COLUMN bet.reconciliation_claim_owner IS
+    'Instance-scoped owner fencing concurrent recovery workers.';
+COMMENT ON COLUMN bet.reconciliation_claim_until IS
+    'Database-time lease expiry permitting crash recovery.';


## `test(recovery): verify lease schema and wake eligibility`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index 3f0f118..8e4fb53 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -70,6 +70,7 @@ class BetTest {
     bet.requestReconciliation(requestedAt);
 
     assertThat(bet.reconciliationRequestedAt()).isEqualTo(requestedAt);
+    assertThat(bet.reconciliationEligibleAt()).isEqualTo(requestedAt);
   }
 
   @Test
diff --git a/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java b/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java
index f38442d..232b954 100644
--- a/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java
+++ b/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java
@@ -87,6 +87,6 @@ class PostgresSagaIntegrationTest extends PostgresIntegrationSupport {
     assertThat(
             jdbc.queryForObject(
                 "select count(*) from flyway_schema_history where success", Integer.class))
-        .isEqualTo(9);
+        .isEqualTo(10);
   }
 }
diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index 4363787..7f34265 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -71,6 +71,15 @@ class MigrationContractTest {
         .contains("WHERE resolution_revision_id IS NOT NULL");
   }
 
+  @Test
+  void addsOwnerFencedRecoveryWithoutChangingEarlierMigrations() {
+    assertThat(migrationText("V10__reconciliation_lease.sql"))
+        .contains("reconciliation_eligible_at TIMESTAMP WITH TIME ZONE")
+        .contains("reconciliation_claim_owner VARCHAR(128)")
+        .contains("reconciliation_claim_until TIMESTAMP WITH TIME ZONE")
+        .contains("WHERE status = 'PENDING'");
+  }
+
   private String migrationText(String migration) {
     try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {


## `feat(recovery): claim fair reconciliation batches`

diff --git a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
index 800a93b..04aa823 100644
--- a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
+++ b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
@@ -11,6 +11,10 @@ import org.springframework.data.domain.Pageable;
 import org.springframework.data.jpa.repository.EntityGraph;
 import org.springframework.data.jpa.repository.JpaRepository;
 import org.springframework.data.jpa.repository.Lock;
+import org.springframework.data.jpa.repository.Modifying;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
+import org.springframework.transaction.annotation.Transactional;
 
 public interface BetRepository extends JpaRepository<Bet, UUID> {
 
@@ -27,6 +31,63 @@ public interface BetRepository extends JpaRepository<Bet, UUID> {
   @EntityGraph(attributePaths = "legs")
   List<Bet> findByStatusAndCreatedAtBefore(BetStatus status, Instant threshold, Pageable pageable);
 
+  @Transactional
+  @Query(
+      nativeQuery = true,
+      value =
+          """
+          WITH candidates AS (
+              SELECT b.bet_id
+              FROM bet b
+              WHERE b.status = 'PENDING'
+                AND COALESCE(
+                      b.reconciliation_eligible_at,
+                      b.reconciliation_requested_at,
+                      b.created_at + CAST(:pendingDelayMs AS bigint) * INTERVAL '1 millisecond'
+                    ) <= CURRENT_TIMESTAMP
+                AND (b.reconciliation_claim_until IS NULL
+                     OR b.reconciliation_claim_until <= CURRENT_TIMESTAMP)
+              ORDER BY COALESCE(
+                         b.reconciliation_eligible_at,
+                         b.reconciliation_requested_at,
+                         b.created_at + CAST(:pendingDelayMs AS bigint) * INTERVAL '1 millisecond'
+                       ), b.bet_id
+              FOR UPDATE SKIP LOCKED
+              LIMIT :batchSize
+          ), claimed AS (
+              UPDATE bet b
+              SET reconciliation_claim_owner = :owner,
+                  reconciliation_claim_until = CURRENT_TIMESTAMP
+                      + CAST(:leaseMs AS bigint) * INTERVAL '1 millisecond',
+                  reconciliation_eligible_at = CURRENT_TIMESTAMP
+                      + CAST(:retryDelayMs AS bigint) * INTERVAL '1 millisecond'
+              FROM candidates c
+              WHERE b.bet_id = c.bet_id
+              RETURNING b.bet_id
+          )
+          SELECT bet_id FROM claimed
+          """)
+  List<UUID> claimReconciliationBatch(
+      @Param("owner") String owner,
+      @Param("pendingDelayMs") long pendingDelayMs,
+      @Param("leaseMs") long leaseMs,
+      @Param("retryDelayMs") long retryDelayMs,
+      @Param("batchSize") int batchSize);
+
+  @Transactional
+  @Modifying(clearAutomatically = true, flushAutomatically = true)
+  @Query(
+      nativeQuery = true,
+      value =
+          """
+          UPDATE bet
+          SET reconciliation_claim_owner = NULL,
+              reconciliation_claim_until = NULL
+          WHERE bet_id = :betId
+            AND reconciliation_claim_owner = :owner
+          """)
+  int clearReconciliationClaim(@Param("betId") UUID betId, @Param("owner") String owner);
+
   @EntityGraph(attributePaths = "legs")
   List<Bet> findByUserIdOrderByBetIdDesc(UUID userId, Pageable pageable);
 


## `test(recovery): verify owner-fenced claim contract`

diff --git a/src/test/java/com/sportsbook/betting/persistence/BetRepositoryLeaseTest.java b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryLeaseTest.java
new file mode 100644
index 0000000..22c7c94
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryLeaseTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.betting.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.jpa.repository.Modifying;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.transaction.annotation.Transactional;
+
+class BetRepositoryLeaseTest {
+
+  @Test
+  void claimsFairBoundedRowsWithDatabaseTimeAndSkipLocked() throws Exception {
+    Method claim =
+        BetRepository.class.getMethod(
+            "claimReconciliationBatch",
+            String.class,
+            long.class,
+            long.class,
+            long.class,
+            int.class);
+    String sql = claim.getAnnotation(Query.class).value();
+
+    assertThat(claim.getAnnotation(Transactional.class)).isNotNull();
+    assertThat(sql)
+        .contains("CURRENT_TIMESTAMP")
+        .contains("ORDER BY COALESCE")
+        .contains("FOR UPDATE SKIP LOCKED")
+        .contains("LIMIT :batchSize")
+        .contains("RETURNING b.bet_id");
+  }
+
+  @Test
+  void clearsOnlyTheClaimOwnedByThisWorker() throws Exception {
+    Method clear =
+        BetRepository.class.getMethod("clearReconciliationClaim", UUID.class, String.class);
+    String sql = clear.getAnnotation(Query.class).value();
+
+    assertThat(clear.getAnnotation(Modifying.class)).isNotNull();
+    assertThat(sql)
+        .contains("reconciliation_claim_owner = NULL")
+        .contains("reconciliation_claim_owner = :owner");
+  }
+}


## `feat(recovery): consume owner-fenced reconciliation claims`

diff --git a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
index 04aa823..252100e 100644
--- a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
+++ b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
@@ -1,9 +1,7 @@
 package com.sportsbook.betting.persistence;
 
 import com.sportsbook.betting.domain.Bet;
-import com.sportsbook.protocol.domain.BetStatus;
 import jakarta.persistence.LockModeType;
-import java.time.Instant;
 import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
@@ -28,9 +26,6 @@ public interface BetRepository extends JpaRepository<Bet, UUID> {
   @EntityGraph(attributePaths = "legs")
   Optional<Bet> findLockedByBetId(UUID betId);
 
-  @EntityGraph(attributePaths = "legs")
-  List<Bet> findByStatusAndCreatedAtBefore(BetStatus status, Instant threshold, Pageable pageable);
-
   @Transactional
   @Query(
       nativeQuery = true,
diff --git a/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java b/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java
index dae6053..d530633 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetReconciliationJob.java
@@ -1,16 +1,12 @@
 package com.sportsbook.betting.placement;
 
-import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.persistence.BetRepository;
-import com.sportsbook.protocol.domain.BetStatus;
-import java.time.Clock;
 import java.time.Duration;
-import java.time.Instant;
 import java.util.List;
+import java.util.UUID;
 import org.slf4j.Logger;
 import org.slf4j.LoggerFactory;
 import org.springframework.beans.factory.annotation.Value;
-import org.springframework.data.domain.PageRequest;
 import org.springframework.scheduling.annotation.Scheduled;
 import org.springframework.stereotype.Component;
 
@@ -22,32 +18,57 @@ public class BetReconciliationJob {
 
   private final BetRepository bets;
   private final BetPlacementService placement;
-  private final Clock clock;
+  private final String owner;
   private final Duration pendingTimeout;
+  private final Duration leaseDuration;
+  private final Duration retryDelay;
 
   public BetReconciliationJob(
       BetRepository bets,
       BetPlacementService placement,
-      Clock clock,
-      @Value("${betting.reconciliation.pending-timeout:30s}") Duration pendingTimeout) {
+      @Value("${random.uuid}") String owner,
+      @Value("${betting.reconciliation.pending-timeout:30s}") Duration pendingTimeout,
+      @Value("${betting.reconciliation.lease-duration:30s}") Duration leaseDuration,
+      @Value("${betting.reconciliation.retry-delay:10s}") Duration retryDelay) {
     this.bets = bets;
     this.placement = placement;
-    this.clock = clock;
-    this.pendingTimeout = pendingTimeout;
+    if (owner == null || owner.isBlank() || owner.length() > 128) {
+      throw new IllegalArgumentException("Reconciliation claim owner must be 1-128 characters");
+    }
+    this.owner = owner;
+    this.pendingTimeout = positive(pendingTimeout, "pendingTimeout");
+    this.leaseDuration = positive(leaseDuration, "leaseDuration");
+    this.retryDelay = positive(retryDelay, "retryDelay");
   }
 
-  @Scheduled(fixedDelayString = "${betting.reconciliation.poll-interval-ms:10000}")
+  @Scheduled(
+      fixedDelayString = "${betting.reconciliation.poll-interval-ms:10000}",
+      scheduler = "reconciliationTaskScheduler")
   public void reconcile() {
-    Instant threshold = clock.instant().minus(pendingTimeout);
-    List<Bet> stale =
-        bets.findByStatusAndCreatedAtBefore(
-            BetStatus.PENDING, threshold, PageRequest.of(0, BATCH_SIZE));
-    for (Bet bet : stale) {
+    List<UUID> claimed =
+        bets.claimReconciliationBatch(
+            owner,
+            pendingTimeout.toMillis(),
+            leaseDuration.toMillis(),
+            retryDelay.toMillis(),
+            BATCH_SIZE);
+    for (UUID betId : claimed) {
       try {
-        placement.reconcile(bet.betId());
+        placement.reconcile(betId);
       } catch (RuntimeException unexpected) {
-        log.error("Placement reconciliation failed for bet {}", bet.betId(), unexpected);
+        log.error("Placement reconciliation failed for bet {}", betId, unexpected);
+      } finally {
+        if (bets.clearReconciliationClaim(betId, owner) == 0) {
+          log.warn("Reconciliation claim was no longer owned for bet {}", betId);
+        }
       }
     }
   }
+
+  private static Duration positive(Duration value, String name) {
+    if (value == null || value.isZero() || value.isNegative()) {
+      throw new IllegalArgumentException(name + " must be positive");
+    }
+    return value;
+  }
 }


## `test(recovery): clear claims after bounded replay`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java b/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java
index 54d8e96..6ae1641 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetReconciliationJobTest.java
@@ -4,38 +4,77 @@ import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
-import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.persistence.BetRepository;
-import com.sportsbook.protocol.domain.BetStatus;
-import java.time.Clock;
+import java.lang.reflect.Constructor;
 import java.time.Duration;
-import java.time.Instant;
-import java.time.ZoneOffset;
 import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
-import org.springframework.data.domain.PageRequest;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.scheduling.annotation.Scheduled;
 
 class BetReconciliationJobTest {
 
   @Test
-  void resumesOnlyStalePendingBetsInBoundedBatches() {
+  void reconcilesOnlyAnOwnedBoundedClaimAndClearsIt() {
     BetRepository bets = mock(BetRepository.class);
     BetPlacementService placement = mock(BetPlacementService.class);
-    Bet bet = mock(Bet.class);
     UUID betId = UUID.randomUUID();
-    Instant now = Instant.parse("2026-08-22T00:00:00Z");
-    when(bet.betId()).thenReturn(betId);
-    when(bets.findByStatusAndCreatedAtBefore(
-            BetStatus.PENDING,
-            now.minusSeconds(30),
-            PageRequest.of(0, BetReconciliationJob.BATCH_SIZE)))
-        .thenReturn(List.of(bet));
+    when(bets.claimReconciliationBatch("worker-1", 30_000, 20_000, 10_000, 100))
+        .thenReturn(List.of(betId));
+    when(bets.clearReconciliationClaim(betId, "worker-1")).thenReturn(1);
 
     new BetReconciliationJob(
-            bets, placement, Clock.fixed(now, ZoneOffset.UTC), Duration.ofSeconds(30))
+            bets,
+            placement,
+            "worker-1",
+            Duration.ofSeconds(30),
+            Duration.ofSeconds(20),
+            Duration.ofSeconds(10))
         .reconcile();
 
     verify(placement).reconcile(betId);
+    verify(bets).clearReconciliationClaim(betId, "worker-1");
+  }
+
+  @Test
+  void releasesItsClaimAfterAReplayFailure() {
+    BetRepository bets = mock(BetRepository.class);
+    BetPlacementService placement = mock(BetPlacementService.class);
+    UUID betId = UUID.randomUUID();
+    when(bets.claimReconciliationBatch("worker-2", 30_000, 20_000, 10_000, 100))
+        .thenReturn(List.of(betId));
+    when(bets.clearReconciliationClaim(betId, "worker-2")).thenReturn(1);
+    org.mockito.Mockito.doThrow(new IllegalStateException("failure"))
+        .when(placement)
+        .reconcile(betId);
+
+    new BetReconciliationJob(
+            bets,
+            placement,
+            "worker-2",
+            Duration.ofSeconds(30),
+            Duration.ofSeconds(20),
+            Duration.ofSeconds(10))
+        .reconcile();
+
+    verify(bets).clearReconciliationClaim(betId, "worker-2");
+  }
+
+  @Test
+  void alwaysUsesAnInstanceUniqueOwner() {
+    Constructor<?> constructor = BetReconciliationJob.class.getConstructors()[0];
+    Value owner = constructor.getParameters()[2].getAnnotation(Value.class);
+
+    org.assertj.core.api.Assertions.assertThat(owner.value()).isEqualTo("${random.uuid}");
+  }
+
+  @Test
+  void runsOnTheDedicatedReconciliationScheduler() throws Exception {
+    Scheduled scheduled =
+        BetReconciliationJob.class.getMethod("reconcile").getAnnotation(Scheduled.class);
+
+    org.assertj.core.api.Assertions.assertThat(scheduled.scheduler())
+        .isEqualTo("reconciliationTaskScheduler");
   }
 }
diff --git a/src/test/resources/application-test.yml b/src/test/resources/application-test.yml
index 23b3f3c..69513c1 100644
--- a/src/test/resources/application-test.yml
+++ b/src/test/resources/application-test.yml
@@ -20,4 +20,4 @@ betting:
     poll-interval-ms: 86400000
   reconciliation:
     poll-interval-ms: 86400000
-    pending-timeout: 0s
+    pending-timeout: 1ms


