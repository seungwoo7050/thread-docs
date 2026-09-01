# 권위 상태와 일관성 경계

## `docs(project): introduce wallet ownership`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..e5a4179
--- /dev/null
+++ b/README.md
@@ -0,0 +1,21 @@
+# Sportsbook Wallet Service
+
+The wallet service owns user balances and the append-only double-entry ledger for the
+sportsbook backend. It is the only service allowed to mutate available or locked funds.
+
+## Responsibilities
+
+- keep available and locked balances in one currency per account;
+- record every money movement as a balanced debit-credit pair;
+- make caller retries resolve to one durable operation outcome;
+- publish wallet integration events through a transactional outbox;
+- expose internal account, transfer, and settlement-adjustment APIs.
+
+## Runtime
+
+The service uses Java 17, Spring Boot, PostgreSQL, Redis, Kafka, Avro, and Maven. PostgreSQL
+owns correctness. Redis is only an optional replay hint, and Kafka publication is driven from
+the database outbox.
+
+Build and runtime instructions are added as the executable project is introduced. The final
+project documentation records the API, security, recovery, and operational contracts.


## `build(flyway): create the final account and ledger schema`

diff --git a/src/main/resources/db/migration/V1__account_and_ledger.sql b/src/main/resources/db/migration/V1__account_and_ledger.sql
new file mode 100644
index 0000000..216a928
--- /dev/null
+++ b/src/main/resources/db/migration/V1__account_and_ledger.sql
@@ -0,0 +1,64 @@
+CREATE TABLE account (
+    user_id UUID PRIMARY KEY,
+    available_amount BIGINT NOT NULL DEFAULT 0,
+    available_currency VARCHAR(3) NOT NULL,
+    locked_amount BIGINT NOT NULL DEFAULT 0,
+    locked_currency VARCHAR(3) NOT NULL,
+    recovery_debt_amount NUMERIC(38, 0) NOT NULL DEFAULT 0,
+    recovery_frozen_at TIMESTAMPTZ,
+    next_adjustment_sequence BIGINT NOT NULL DEFAULT 1,
+    version BIGINT NOT NULL DEFAULT 0,
+    created_at TIMESTAMPTZ NOT NULL,
+    updated_at TIMESTAMPTZ NOT NULL,
+    CONSTRAINT ck_account_user_uuid CHECK (
+        user_id NOT IN (
+            '00000000-0000-7000-8000-000000000001',
+            '00000000-0000-7000-8000-000000000002'
+        )
+    ),
+    CONSTRAINT ck_account_available_nonnegative CHECK (available_amount >= 0),
+    CONSTRAINT ck_account_locked_nonnegative CHECK (locked_amount >= 0),
+    CONSTRAINT ck_account_currency CHECK (
+        available_currency = locked_currency
+        AND available_currency IN ('KRW', 'USD')
+    ),
+    CONSTRAINT ck_account_aggregate_limit CHECK (
+        available_amount <= 9223372036854775807 - locked_amount
+    ),
+    CONSTRAINT ck_account_recovery_debt CHECK (recovery_debt_amount >= 0),
+    CONSTRAINT ck_account_recovery_freeze CHECK (
+        (recovery_debt_amount = 0 AND recovery_frozen_at IS NULL)
+        OR (recovery_debt_amount > 0 AND recovery_frozen_at IS NOT NULL)
+    ),
+    CONSTRAINT ck_account_adjustment_sequence CHECK (next_adjustment_sequence > 0)
+);
+
+CREATE TABLE ledger_entry (
+    entry_id UUID PRIMARY KEY,
+    account_id UUID NOT NULL,
+    bucket VARCHAR(16) NOT NULL,
+    side VARCHAR(6) NOT NULL,
+    amount BIGINT NOT NULL,
+    currency VARCHAR(3) NOT NULL,
+    reason VARCHAR(24) NOT NULL,
+    idempotency_key VARCHAR(128) NOT NULL,
+    operation_group_id UUID NOT NULL,
+    created_at TIMESTAMPTZ NOT NULL,
+    CONSTRAINT ck_ledger_bucket CHECK (bucket IN ('AVAILABLE', 'LOCKED')),
+    CONSTRAINT ck_ledger_side CHECK (side IN ('DEBIT', 'CREDIT')),
+    CONSTRAINT ck_ledger_amount CHECK (amount > 0),
+    CONSTRAINT ck_ledger_currency CHECK (currency IN ('KRW', 'USD')),
+    CONSTRAINT ck_ledger_reason CHECK (
+        reason IN (
+            'DEPOSIT', 'WITHDRAW', 'BET_DEBIT', 'BET_PAYOUT',
+            'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        )
+    ),
+    CONSTRAINT uk_ledger_entry_idempotency_side UNIQUE (idempotency_key, side),
+    CONSTRAINT uk_ledger_entry_group_side UNIQUE (operation_group_id, side)
+);
+
+CREATE INDEX ix_ledger_entry_account_created
+    ON ledger_entry (account_id, created_at);
+CREATE INDEX ix_ledger_entry_idempotency_key
+    ON ledger_entry (idempotency_key);


## `build(flyway): create authoritative wallet outcomes`

diff --git a/src/main/resources/db/migration/V2__wallet_operation.sql b/src/main/resources/db/migration/V2__wallet_operation.sql
new file mode 100644
index 0000000..5a8a800
--- /dev/null
+++ b/src/main/resources/db/migration/V2__wallet_operation.sql
@@ -0,0 +1,82 @@
+CREATE TABLE wallet_operation (
+    idempotency_key VARCHAR(128) PRIMARY KEY,
+    caller_id VARCHAR(16) NOT NULL,
+    operation_kind VARCHAR(32) NOT NULL,
+    user_id UUID NOT NULL,
+    request_amount BIGINT NOT NULL,
+    request_currency VARCHAR(3) NOT NULL,
+    request_fingerprint VARCHAR(64) NOT NULL,
+    status VARCHAR(24) NOT NULL,
+    operation_group_id UUID UNIQUE,
+    failure_code VARCHAR(32),
+    failure_http_status SMALLINT,
+    failure_title VARCHAR(128),
+    failure_detail VARCHAR(1024),
+    failure_balance_amount BIGINT,
+    failure_balance_currency VARCHAR(3),
+    failure_expected_currency VARCHAR(3),
+    requested_at TIMESTAMPTZ NOT NULL,
+    updated_at TIMESTAMPTZ NOT NULL,
+    completed_at TIMESTAMPTZ,
+    version BIGINT NOT NULL DEFAULT 0,
+    CONSTRAINT ck_wallet_operation_caller CHECK (
+        caller_id IN ('PLATFORM', 'BETTING', 'SETTLEMENT', 'ADMIN')
+    ),
+    CONSTRAINT ck_wallet_operation_kind CHECK (
+        operation_kind IN (
+            'DEPOSIT', 'WITHDRAW', 'BET_DEBIT', 'BET_PAYOUT',
+            'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        )
+    ),
+    CONSTRAINT ck_wallet_operation_caller_kind CHECK (
+        (caller_id = 'PLATFORM' AND operation_kind IN ('DEPOSIT', 'WITHDRAW'))
+        OR (caller_id = 'BETTING' AND operation_kind IN ('BET_DEBIT', 'BET_REFUND'))
+        OR (caller_id = 'SETTLEMENT' AND operation_kind IN (
+            'BET_PAYOUT', 'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        ))
+        OR (caller_id = 'ADMIN' AND operation_kind = 'BET_REFUND')
+    ),
+    CONSTRAINT ck_wallet_operation_request CHECK (
+        request_amount > 0
+        AND request_currency IN ('KRW', 'USD')
+        AND request_fingerprint ~ '^[0-9a-f]{64}$'
+    ),
+    CONSTRAINT ck_wallet_operation_status CHECK (
+        status IN ('SUCCEEDED', 'REJECTED', 'BLOCKED_FUNDS')
+    ),
+    CONSTRAINT ck_wallet_operation_outcome CHECK (
+        (
+            status = 'SUCCEEDED'
+            AND operation_group_id IS NOT NULL
+            AND completed_at IS NOT NULL
+            AND failure_code IS NULL
+            AND failure_http_status IS NULL
+            AND failure_title IS NULL
+            AND failure_detail IS NULL
+        )
+        OR (
+            status = 'REJECTED'
+            AND operation_group_id IS NULL
+            AND completed_at IS NOT NULL
+            AND failure_code IS NOT NULL
+            AND failure_http_status BETWEEN 400 AND 499
+            AND failure_title IS NOT NULL
+            AND failure_detail IS NOT NULL
+        )
+        OR (
+            status = 'BLOCKED_FUNDS'
+            AND operation_kind = 'BET_ADJUSTMENT'
+            AND operation_group_id IS NULL
+            AND completed_at IS NULL
+            AND failure_code IS NULL
+            AND failure_http_status IS NULL
+            AND failure_title IS NULL
+            AND failure_detail IS NULL
+        )
+    ),
+    CONSTRAINT uk_wallet_operation_key_group
+        UNIQUE (idempotency_key, operation_group_id)
+);
+
+CREATE INDEX ix_wallet_operation_user_requested
+    ON wallet_operation (user_id, requested_at);


## `feat(cache): treat Redis idempotency data as a fallible hint`

diff --git a/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java b/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java
new file mode 100644
index 0000000..140eb17
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.time.Duration;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+/** Best-effort existence marker; PostgreSQL remains the only owner of request outcomes. */
+@Component
+public class IdempotencyCache {
+
+  static final Duration TTL = Duration.ofHours(24L);
+  static final String KEY_PREFIX = "idempotency:wallet:";
+
+  private static final Logger log = LoggerFactory.getLogger(IdempotencyCache.class);
+
+  private final StringRedisTemplate redis;
+
+  public IdempotencyCache(StringRedisTemplate redis) {
+    this.redis = redis;
+  }
+
+  public boolean mightContain(IdempotencyKey key) {
+    try {
+      return Boolean.TRUE.equals(redis.hasKey(redisKey(key)));
+    } catch (DataAccessException unavailable) {
+      log.warn("Redis idempotency lookup failed; using PostgreSQL", unavailable);
+      return false;
+    }
+  }
+
+  public void mark(IdempotencyKey key) {
+    try {
+      redis.opsForValue().set(redisKey(key), "1", TTL);
+    } catch (DataAccessException unavailable) {
+      log.warn("Redis idempotency marker failed after PostgreSQL outcome", unavailable);
+    }
+  }
+
+  private static String redisKey(IdempotencyKey key) {
+    return KEY_PREFIX + key.value();
+  }
+}


## `build(flyway): create an ordered transactional outbox`

diff --git a/src/main/resources/db/migration/V3__transactional_outbox.sql b/src/main/resources/db/migration/V3__transactional_outbox.sql
new file mode 100644
index 0000000..0e3ede1
--- /dev/null
+++ b/src/main/resources/db/migration/V3__transactional_outbox.sql
@@ -0,0 +1,66 @@
+CREATE TABLE outbox_stream (
+    topic VARCHAR(128) NOT NULL,
+    partition_key VARCHAR(128) NOT NULL,
+    last_sequence BIGINT NOT NULL DEFAULT 0,
+    PRIMARY KEY (topic, partition_key),
+    CONSTRAINT ck_outbox_stream_sequence CHECK (last_sequence >= 0)
+);
+
+CREATE TABLE outbox_event (
+    event_id UUID PRIMARY KEY,
+    operation_key VARCHAR(128) NOT NULL,
+    topic VARCHAR(128) NOT NULL,
+    partition_key VARCHAR(128) NOT NULL,
+    schema_name VARCHAR(128) NOT NULL,
+    deduplication_key VARCHAR(128) NOT NULL,
+    stream_sequence BIGINT NOT NULL,
+    payload BYTEA NOT NULL,
+    created_at TIMESTAMPTZ NOT NULL,
+    available_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
+    published_at TIMESTAMPTZ,
+    lease_owner VARCHAR(128),
+    lease_until TIMESTAMPTZ,
+    lease_version BIGINT NOT NULL DEFAULT 0,
+    attempt_count INTEGER NOT NULL DEFAULT 0,
+    last_error VARCHAR(1024),
+    CONSTRAINT fk_outbox_operation
+        FOREIGN KEY (operation_key)
+        REFERENCES wallet_operation(idempotency_key)
+        DEFERRABLE INITIALLY DEFERRED,
+    CONSTRAINT fk_outbox_stream
+        FOREIGN KEY (topic, partition_key)
+        REFERENCES outbox_stream(topic, partition_key),
+    CONSTRAINT uq_outbox_semantic_event UNIQUE (topic, deduplication_key),
+    CONSTRAINT uq_outbox_operation UNIQUE (operation_key),
+    CONSTRAINT uq_outbox_stream_sequence UNIQUE (topic, partition_key, stream_sequence),
+    CONSTRAINT ck_outbox_strings CHECK (
+        btrim(operation_key) <> ''
+        AND btrim(topic) <> ''
+        AND btrim(partition_key) <> ''
+        AND btrim(schema_name) <> ''
+        AND btrim(deduplication_key) <> ''
+    ),
+    CONSTRAINT ck_outbox_payload CHECK (octet_length(payload) > 0),
+    CONSTRAINT ck_outbox_stream_position CHECK (stream_sequence > 0),
+    CONSTRAINT ck_outbox_attempt_count CHECK (attempt_count >= 0),
+    CONSTRAINT ck_outbox_lease_version CHECK (lease_version >= 0),
+    CONSTRAINT ck_outbox_lease_pair CHECK (
+        (lease_owner IS NULL AND lease_until IS NULL)
+        OR (lease_owner IS NOT NULL AND lease_until IS NOT NULL)
+    ),
+    CONSTRAINT ck_outbox_published_lease CHECK (
+        published_at IS NULL OR (lease_owner IS NULL AND lease_until IS NULL)
+    )
+);
+
+CREATE INDEX ix_outbox_claim_due
+    ON outbox_event (available_at, stream_sequence)
+    WHERE published_at IS NULL;
+
+CREATE INDEX ix_outbox_fifo
+    ON outbox_event (topic, partition_key, stream_sequence)
+    WHERE published_at IS NULL;
+
+CREATE INDEX ix_outbox_lease_expiry
+    ON outbox_event (lease_until)
+    WHERE published_at IS NULL AND lease_owner IS NOT NULL;


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


## `feat(persistence): read transaction timestamps from PostgreSQL`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/DatabaseClock.java b/src/main/java/com/sportsbook/wallet/persistence/DatabaseClock.java
new file mode 100644
index 0000000..f125159
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/DatabaseClock.java
@@ -0,0 +1,27 @@
+package com.sportsbook.wallet.persistence;
+
+import java.time.Instant;
+import java.time.OffsetDateTime;
+import java.util.Objects;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+
+/** Reads the authoritative PostgreSQL wall clock once after wallet locks are held. */
+@Component
+public class DatabaseClock {
+  private final JdbcTemplate jdbc;
+
+  public DatabaseClock(JdbcTemplate jdbc) {
+    this.jdbc = Objects.requireNonNull(jdbc, "jdbc");
+  }
+
+  public Instant now() {
+    if (!TransactionSynchronizationManager.isActualTransactionActive()) {
+      throw new IllegalStateException("Database clock requires an active transaction");
+    }
+    return jdbc.queryForObject(
+        "SELECT clock_timestamp()",
+        (result, row) -> result.getObject(1, OffsetDateTime.class).toInstant());
+  }
+}


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


