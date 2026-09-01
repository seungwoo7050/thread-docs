# 조정 증명과 최초 판정

## `feat(command): validate correction payout snapshots`

diff --git a/src/main/java/com/sportsbook/wallet/service/command/AdjustmentCommand.java b/src/main/java/com/sportsbook/wallet/service/command/AdjustmentCommand.java
new file mode 100644
index 0000000..ec47518
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/command/AdjustmentCommand.java
@@ -0,0 +1,54 @@
+package com.sportsbook.wallet.service.command;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Validated settlement revision whose nonzero payout delta must be applied or queued. */
+public record AdjustmentCommand(
+    UUID revisionId,
+    UUID betId,
+    long revisionNumber,
+    UUID userId,
+    Money previousPayout,
+    Money newPayout,
+    IdempotencyKey idempotencyKey) {
+
+  public AdjustmentCommand {
+    Objects.requireNonNull(revisionId, "revisionId");
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(previousPayout, "previousPayout");
+    Objects.requireNonNull(newPayout, "newPayout");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    if (SystemAccountIds.isSystemAccount(userId)) {
+      throw new IllegalArgumentException("System UUID cannot receive an adjustment");
+    }
+    if (revisionNumber < 1L) {
+      throw new IllegalArgumentException("Revision number must be at least one");
+    }
+    if (previousPayout.amount() < 0L || newPayout.amount() < 0L) {
+      throw new IllegalArgumentException("Payout snapshots cannot be negative");
+    }
+    if (previousPayout.currency() != newPayout.currency()) {
+      throw new IllegalArgumentException("Payout snapshot currencies must match");
+    }
+    if (previousPayout.amount() == newPayout.amount()) {
+      throw new IllegalArgumentException("Adjustment delta cannot be zero");
+    }
+    String expectedKey = "settlement:revision:" + revisionId;
+    if (!idempotencyKey.value().equals(expectedKey)) {
+      throw new IllegalArgumentException("Idempotency key must identify the revision");
+    }
+  }
+
+  public long deltaAmount() {
+    return Math.subtractExact(newPayout.amount(), previousPayout.amount());
+  }
+
+  public Money absoluteDelta() {
+    return new Money(Math.abs(deltaAmount()), previousPayout.currency());
+  }
+}


## `feat(adjustment): map revision request identity`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
new file mode 100644
index 0000000..35185e2
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
@@ -0,0 +1,83 @@
+package com.sportsbook.wallet.domain;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.util.UUID;
+
+/** Durable request identity and proof for one nonzero settlement payout correction. */
+@Entity
+@Table(name = "wallet_adjustment")
+public class WalletAdjustment {
+  @Id
+  @Column(name = "revision_id", nullable = false, updatable = false)
+  private UUID revisionId;
+
+  @Column(name = "idempotency_key", nullable = false, length = 128, updatable = false)
+  private String idempotencyKey;
+
+  @Column(name = "bet_id", nullable = false, updatable = false)
+  private UUID betId;
+
+  @Column(name = "revision_number", nullable = false, updatable = false)
+  private long revisionNumber;
+
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Column(name = "previous_payout_amount", nullable = false, updatable = false)
+  private long previousPayoutAmount;
+
+  @Column(name = "new_payout_amount", nullable = false, updatable = false)
+  private long newPayoutAmount;
+
+  @Column(name = "delta_amount", nullable = false, updatable = false)
+  private long deltaAmount;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "currency", nullable = false, length = 3, updatable = false)
+  private Currency currency;
+
+  protected WalletAdjustment() {}
+
+  public UUID revisionId() {
+    return revisionId;
+  }
+
+  public String idempotencyKey() {
+    return idempotencyKey;
+  }
+
+  public UUID betId() {
+    return betId;
+  }
+
+  public long revisionNumber() {
+    return revisionNumber;
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+
+  public Money previousPayout() {
+    return new Money(previousPayoutAmount, currency);
+  }
+
+  public Money newPayout() {
+    return new Money(newPayoutAmount, currency);
+  }
+
+  public long deltaAmount() {
+    return deltaAmount;
+  }
+
+  public Currency currency() {
+    return currency;
+  }
+}


## `feat(adjustment): map proof lifecycle metadata`

diff --git a/src/main/java/com/sportsbook/wallet/domain/AdjustmentStatus.java b/src/main/java/com/sportsbook/wallet/domain/AdjustmentStatus.java
new file mode 100644
index 0000000..43b3b87
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/AdjustmentStatus.java
@@ -0,0 +1,8 @@
+package com.sportsbook.wallet.domain;
+
+/** Durable settlement correction proof state. */
+public enum AdjustmentStatus {
+  APPLIED,
+  BLOCKED,
+  REJECTED
+}
diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
index 35185e2..b9699c0 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
@@ -8,6 +8,7 @@ import jakarta.persistence.EnumType;
 import jakarta.persistence.Enumerated;
 import jakarta.persistence.Id;
 import jakarta.persistence.Table;
+import java.time.Instant;
 import java.util.UUID;
 
 /** Durable request identity and proof for one nonzero settlement payout correction. */
@@ -43,6 +44,34 @@ public class WalletAdjustment {
   @Column(name = "currency", nullable = false, length = 3, updatable = false)
   private Currency currency;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "status", nullable = false, length = 16)
+  private AdjustmentStatus status;
+
+  @Column(name = "queue_sequence")
+  private Long queueSequence;
+
+  @Column(name = "operation_group_id")
+  private UUID operationGroupId;
+
+  @Column(name = "queued_at")
+  private Instant queuedAt;
+
+  @Column(name = "applied_at")
+  private Instant appliedAt;
+
+  @Column(name = "next_attempt_at")
+  private Instant nextAttemptAt;
+
+  @Column(name = "retry_count", nullable = false)
+  private int retryCount;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "updated_at", nullable = false)
+  private Instant updatedAt;
+
   protected WalletAdjustment() {}
 
   public UUID revisionId() {
@@ -80,4 +109,40 @@ public class WalletAdjustment {
   public Currency currency() {
     return currency;
   }
+
+  public AdjustmentStatus status() {
+    return status;
+  }
+
+  public Long queueSequence() {
+    return queueSequence;
+  }
+
+  public UUID operationGroupId() {
+    return operationGroupId;
+  }
+
+  public Instant queuedAt() {
+    return queuedAt;
+  }
+
+  public Instant appliedAt() {
+    return appliedAt;
+  }
+
+  public Instant nextAttemptAt() {
+    return nextAttemptAt;
+  }
+
+  public int retryCount() {
+    return retryCount;
+  }
+
+  public Instant createdAt() {
+    return createdAt;
+  }
+
+  public Instant updatedAt() {
+    return updatedAt;
+  }
 }


## `feat(adjustment): construct terminal and blocked proofs`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
index b9699c0..01b314d 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
@@ -2,6 +2,7 @@ package com.sportsbook.wallet.domain;
 
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
 import jakarta.persistence.Column;
 import jakarta.persistence.Entity;
 import jakarta.persistence.EnumType;
@@ -9,6 +10,7 @@ import jakarta.persistence.Enumerated;
 import jakarta.persistence.Id;
 import jakarta.persistence.Table;
 import java.time.Instant;
+import java.util.Objects;
 import java.util.UUID;
 
 /** Durable request identity and proof for one nonzero settlement payout correction. */
@@ -74,6 +76,52 @@ public class WalletAdjustment {
 
   protected WalletAdjustment() {}
 
+  private WalletAdjustment(AdjustmentCommand command, Instant now) {
+    Objects.requireNonNull(command, "command");
+    this.revisionId = command.revisionId();
+    this.idempotencyKey = command.idempotencyKey().value();
+    this.betId = command.betId();
+    this.revisionNumber = command.revisionNumber();
+    this.userId = command.userId();
+    this.previousPayoutAmount = command.previousPayout().amount();
+    this.newPayoutAmount = command.newPayout().amount();
+    this.deltaAmount = command.deltaAmount();
+    this.currency = command.previousPayout().currency();
+    this.createdAt = Objects.requireNonNull(now, "now");
+    this.updatedAt = now;
+  }
+
+  public static WalletAdjustment applied(
+      AdjustmentCommand command, UUID operationGroupId, Instant now) {
+    WalletAdjustment proof = new WalletAdjustment(command, now);
+    proof.status = AdjustmentStatus.APPLIED;
+    proof.operationGroupId = Objects.requireNonNull(operationGroupId, "operationGroupId");
+    proof.appliedAt = now;
+    return proof;
+  }
+
+  public static WalletAdjustment blocked(
+      AdjustmentCommand command, long queueSequence, Instant now) {
+    if (command.deltaAmount() >= 0L) {
+      throw new IllegalArgumentException("Only negative adjustments can be blocked");
+    }
+    if (queueSequence < 1L) {
+      throw new IllegalArgumentException("Queue sequence must be positive");
+    }
+    WalletAdjustment proof = new WalletAdjustment(command, now);
+    proof.status = AdjustmentStatus.BLOCKED;
+    proof.queueSequence = queueSequence;
+    proof.queuedAt = now;
+    proof.nextAttemptAt = now;
+    return proof;
+  }
+
+  public static WalletAdjustment rejected(AdjustmentCommand command, Instant now) {
+    WalletAdjustment proof = new WalletAdjustment(command, now);
+    proof.status = AdjustmentStatus.REJECTED;
+    return proof;
+  }
+
   public UUID revisionId() {
     return revisionId;
   }


## `build(flyway): create adjustment proof and recovery table`

diff --git a/src/main/resources/db/migration/V4__wallet_adjustment.sql b/src/main/resources/db/migration/V4__wallet_adjustment.sql
new file mode 100644
index 0000000..8f99df7
--- /dev/null
+++ b/src/main/resources/db/migration/V4__wallet_adjustment.sql
@@ -0,0 +1,87 @@
+CREATE TABLE wallet_adjustment (
+    revision_id UUID PRIMARY KEY,
+    idempotency_key VARCHAR(128) NOT NULL UNIQUE,
+    bet_id UUID NOT NULL,
+    revision_number BIGINT NOT NULL,
+    user_id UUID NOT NULL,
+    previous_payout_amount BIGINT NOT NULL,
+    new_payout_amount BIGINT NOT NULL,
+    delta_amount BIGINT NOT NULL,
+    currency VARCHAR(3) NOT NULL,
+    status VARCHAR(16) NOT NULL,
+    queue_sequence BIGINT,
+    operation_group_id UUID UNIQUE,
+    queued_at TIMESTAMPTZ,
+    applied_at TIMESTAMPTZ,
+    next_attempt_at TIMESTAMPTZ,
+    retry_count INTEGER NOT NULL DEFAULT 0,
+    created_at TIMESTAMPTZ NOT NULL,
+    updated_at TIMESTAMPTZ NOT NULL,
+    CONSTRAINT uq_wallet_adjustment_bet_revision
+        UNIQUE (bet_id, revision_number),
+    CONSTRAINT uq_wallet_adjustment_user_sequence
+        UNIQUE (user_id, queue_sequence),
+    CONSTRAINT fk_wallet_adjustment_operation
+        FOREIGN KEY (idempotency_key)
+        REFERENCES wallet_operation(idempotency_key)
+        DEFERRABLE INITIALLY DEFERRED,
+    CONSTRAINT fk_wallet_adjustment_operation_group
+        FOREIGN KEY (idempotency_key, operation_group_id)
+        REFERENCES wallet_operation(idempotency_key, operation_group_id)
+        DEFERRABLE INITIALLY DEFERRED,
+    CONSTRAINT ck_wallet_adjustment_request CHECK (
+        revision_number >= 1
+        AND idempotency_key = 'settlement:revision:' || revision_id::text
+        AND user_id NOT IN (
+            '00000000-0000-7000-8000-000000000001',
+            '00000000-0000-7000-8000-000000000002'
+        )
+        AND previous_payout_amount >= 0
+        AND new_payout_amount >= 0
+        AND delta_amount <> 0
+        AND delta_amount = new_payout_amount - previous_payout_amount
+        AND currency IN ('KRW', 'USD')
+    ),
+    CONSTRAINT ck_wallet_adjustment_status CHECK (
+        status IN ('APPLIED', 'BLOCKED', 'REJECTED')
+    ),
+    CONSTRAINT ck_wallet_adjustment_queue_pair CHECK (
+        (queue_sequence IS NULL AND queued_at IS NULL)
+        OR (queue_sequence > 0 AND queued_at IS NOT NULL)
+    ),
+    CONSTRAINT ck_wallet_adjustment_outcome CHECK (
+        (
+            status = 'APPLIED'
+            AND operation_group_id IS NOT NULL
+            AND applied_at IS NOT NULL
+            AND next_attempt_at IS NULL
+            AND (queue_sequence IS NULL OR delta_amount < 0)
+            AND (queue_sequence IS NOT NULL OR retry_count = 0)
+        )
+        OR (
+            status = 'BLOCKED'
+            AND operation_group_id IS NULL
+            AND queue_sequence IS NOT NULL
+            AND next_attempt_at IS NOT NULL
+            AND applied_at IS NULL
+            AND delta_amount < 0
+        )
+        OR (
+            status = 'REJECTED'
+            AND operation_group_id IS NULL
+            AND queue_sequence IS NULL
+            AND applied_at IS NULL
+            AND next_attempt_at IS NULL
+            AND retry_count = 0
+        )
+    ),
+    CONSTRAINT ck_wallet_adjustment_retry CHECK (retry_count >= 0)
+);
+
+CREATE INDEX ix_wallet_adjustment_fifo
+    ON wallet_adjustment (user_id, queue_sequence)
+    WHERE status = 'BLOCKED';
+
+CREATE INDEX ix_wallet_adjustment_due
+    ON wallet_adjustment (next_attempt_at, user_id)
+    WHERE status = 'BLOCKED';


## `feat(repository): query adjustment proofs and FIFO heads`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/WalletAdjustmentRepository.java b/src/main/java/com/sportsbook/wallet/persistence/WalletAdjustmentRepository.java
new file mode 100644
index 0000000..3d569f5
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/WalletAdjustmentRepository.java
@@ -0,0 +1,31 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.domain.AdjustmentStatus;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
+
+/** Durable correction proofs and account-scoped FIFO head selection. */
+public interface WalletAdjustmentRepository extends JpaRepository<WalletAdjustment, UUID> {
+
+  Optional<WalletAdjustment> findByIdempotencyKey(String idempotencyKey);
+
+  Optional<WalletAdjustment> findByBetIdAndRevisionNumber(UUID betId, long revisionNumber);
+
+  boolean existsByUserIdAndStatus(UUID userId, AdjustmentStatus status);
+
+  @Query(
+      value =
+          """
+          SELECT * FROM wallet_adjustment
+          WHERE user_id = :userId AND status = 'BLOCKED'
+          ORDER BY queue_sequence
+          LIMIT 1
+          FOR UPDATE
+          """,
+      nativeQuery = true)
+  Optional<WalletAdjustment> findOldestBlockedForUpdate(@Param("userId") UUID userId);
+}


## `feat(adjustment): serialize bet revision claims`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/AdjustmentPairLock.java b/src/main/java/com/sportsbook/wallet/persistence/AdjustmentPairLock.java
new file mode 100644
index 0000000..199f839
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/AdjustmentPairLock.java
@@ -0,0 +1,34 @@
+package com.sportsbook.wallet.persistence;
+
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+
+/** Serializes one bet revision pair in a namespace disjoint from idempotency-key locks. */
+@Component
+public class AdjustmentPairLock {
+  static final int NAMESPACE = 0x57414C4C;
+  private static final String LOCK_SQL = "SELECT pg_advisory_xact_lock(?, hashtext(?))";
+
+  private final JdbcTemplate jdbc;
+
+  public AdjustmentPairLock(JdbcTemplate jdbc) {
+    this.jdbc = Objects.requireNonNull(jdbc, "jdbc");
+  }
+
+  public void acquire(UUID betId, long revisionNumber) {
+    if (!TransactionSynchronizationManager.isActualTransactionActive()) {
+      throw new IllegalStateException("Adjustment pair lock requires an active transaction");
+    }
+    String pair = Objects.requireNonNull(betId, "betId") + ":" + revisionNumber;
+    jdbc.query(
+        LOCK_SQL,
+        statement -> {
+          statement.setInt(1, NAMESPACE);
+          statement.setString(2, pair);
+        },
+        resultSet -> null);
+  }
+}


## `test(adjustment): serialize concurrent revision claims`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/AdjustmentPairLockTest.java b/src/test/java/com/sportsbook/wallet/persistence/AdjustmentPairLockTest.java
new file mode 100644
index 0000000..4660fae
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/persistence/AdjustmentPairLockTest.java
@@ -0,0 +1,92 @@
+package com.sportsbook.wallet.persistence;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.CountDownLatch;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.transaction.PlatformTransactionManager;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+import org.springframework.transaction.support.TransactionTemplate;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = "spring.test.database.replace=NONE")
+@Testcontainers
+@Transactional(propagation = Propagation.NOT_SUPPORTED)
+@Import(AdjustmentPairLock.class)
+class AdjustmentPairLockTest {
+  @Container
+  static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @Autowired AdjustmentPairLock pairLock;
+  @Autowired PlatformTransactionManager transactions;
+
+  @DynamicPropertySource
+  static void databaseProperties(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+  }
+
+  @Test
+  void serializesTheSameBetRevisionUntilTransactionCompletion() throws Exception {
+    UUID betId = UUID.fromString("019b76da-a000-7000-8000-000000000121");
+    TransactionTemplate transaction = new TransactionTemplate(transactions);
+    CountDownLatch acquired = new CountDownLatch(1);
+    CountDownLatch release = new CountDownLatch(1);
+    CountDownLatch contenderStarted = new CountDownLatch(1);
+    CompletableFuture<Void> first =
+        CompletableFuture.runAsync(
+            () ->
+                transaction.executeWithoutResult(
+                    ignored -> {
+                      pairLock.acquire(betId, 7L);
+                      acquired.countDown();
+                      await(release);
+                    }));
+    acquired.await();
+    CompletableFuture<Void> second =
+        CompletableFuture.runAsync(
+            () ->
+                transaction.executeWithoutResult(
+                    ignored -> {
+                      contenderStarted.countDown();
+                      pairLock.acquire(betId, 7L);
+                    }));
+    contenderStarted.await();
+
+    try {
+      assertThatThrownBy(() -> second.get(300L, java.util.concurrent.TimeUnit.MILLISECONDS))
+          .isInstanceOf(java.util.concurrent.TimeoutException.class);
+    } finally {
+      release.countDown();
+    }
+    first.get();
+    second.get();
+  }
+
+  @Test
+  void refusesAnAutocommitLockThatWouldReleaseImmediately() {
+    assertThatThrownBy(() -> pairLock.acquire(UUID.randomUUID(), 1L))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("Adjustment pair lock requires an active transaction");
+  }
+
+  private static void await(CountDownLatch latch) {
+    try {
+      latch.await();
+    } catch (InterruptedException interrupted) {
+      Thread.currentThread().interrupt();
+      throw new IllegalStateException(interrupted);
+    }
+  }
+}


