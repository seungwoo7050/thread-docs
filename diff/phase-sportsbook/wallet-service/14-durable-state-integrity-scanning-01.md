# 영속 상태 무결성 스캔

## `feat(integrity): reconcile account ledger snapshots`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/AccountIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/AccountIntegrityRepository.java
new file mode 100644
index 0000000..9aede26
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/AccountIntegrityRepository.java
@@ -0,0 +1,77 @@
+package com.sportsbook.wallet.integrity;
+
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import java.math.BigInteger;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Reconciles materialized account buckets against unbounded numeric ledger nets. */
+@Repository
+public class AccountIntegrityRepository {
+  private static final String SNAPSHOT_DRIFT_SQL =
+      """
+      WITH ledger_net AS (
+        SELECT a.user_id,
+          COALESCE(SUM(CASE WHEN l.bucket = 'AVAILABLE'
+            THEN CASE l.side WHEN 'DEBIT' THEN l.amount ELSE -l.amount END
+            ELSE 0 END), 0)::NUMERIC AS available_net,
+          COALESCE(SUM(CASE WHEN l.bucket = 'LOCKED'
+            THEN CASE l.side WHEN 'DEBIT' THEN l.amount ELSE -l.amount END
+            ELSE 0 END), 0)::NUMERIC AS locked_net,
+          COALESCE(BOOL_OR(l.currency <> a.available_currency), FALSE) AS currency_drift
+        FROM account a
+        LEFT JOIN ledger_entry l ON l.account_id = a.user_id
+        GROUP BY a.user_id
+      )
+      SELECT a.user_id, a.available_amount, n.available_net,
+        a.locked_amount, n.locked_net, a.available_currency
+      FROM account a
+      JOIN ledger_net n ON n.user_id = a.user_id
+      WHERE a.available_amount::NUMERIC <> n.available_net
+        OR a.locked_amount::NUMERIC <> n.locked_net
+        OR n.currency_drift
+      ORDER BY a.user_id
+      """;
+  private static final String ORPHAN_LEDGER_SQL =
+      """
+      SELECT DISTINCT l.account_id
+      FROM ledger_entry l
+      LEFT JOIN account a ON a.user_id = l.account_id
+      WHERE a.user_id IS NULL AND l.account_id NOT IN (?, ?)
+      ORDER BY l.account_id
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AccountIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<AccountSnapshotDrift> findSnapshotDrift() {
+    return jdbc.query(
+        SNAPSHOT_DRIFT_SQL,
+        (row, number) ->
+            new AccountSnapshotDrift(
+                row.getObject("user_id", UUID.class),
+                BigInteger.valueOf(row.getLong("available_amount")),
+                row.getBigDecimal("available_net").toBigIntegerExact(),
+                BigInteger.valueOf(row.getLong("locked_amount")),
+                row.getBigDecimal("locked_net").toBigIntegerExact(),
+                row.getString("available_currency")));
+  }
+
+  public List<UUID> findOrphanLedgerAccountIds() {
+    return jdbc.queryForList(
+        ORPHAN_LEDGER_SQL, UUID.class, SystemAccountIds.HOUSE, SystemAccountIds.EXTERNAL_PAYMENT);
+  }
+
+  public record AccountSnapshotDrift(
+      UUID userId,
+      BigInteger availableSnapshot,
+      BigInteger availableLedgerNet,
+      BigInteger lockedSnapshot,
+      BigInteger lockedLedgerNet,
+      String currency) {}
+}


## `feat(integrity): project operation ledger groups`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/OperationIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/OperationIntegrityRepository.java
new file mode 100644
index 0000000..7107a3d
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/OperationIntegrityRepository.java
@@ -0,0 +1,87 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.List;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Projects durable outcomes onto their exact two-row ledger groups. */
+@Repository
+public class OperationIntegrityRepository {
+  private static final String GROUP_DRIFT_SQL =
+      """
+      WITH succeeded AS (
+        SELECT o.idempotency_key, COUNT(l.entry_id) AS entries,
+          COUNT(l.entry_id) FILTER (WHERE l.side = 'DEBIT') AS debits,
+          COUNT(l.entry_id) FILTER (WHERE l.side = 'CREDIT') AS credits,
+          COALESCE(BOOL_AND(
+            l.idempotency_key = o.idempotency_key
+            AND l.amount = o.request_amount
+            AND l.currency = o.request_currency
+            AND l.reason = o.operation_kind
+            AND l.created_at = o.completed_at
+          ), FALSE) AS matches_request,
+          MIN(l.created_at) IS NOT DISTINCT FROM MAX(l.created_at) AS one_timestamp,
+          CASE o.operation_kind
+            WHEN 'DEPOSIT' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+              AND BOOL_OR(l.side = 'CREDIT' AND l.account_id =
+                '00000000-0000-7000-8000-000000000002' AND l.bucket = 'AVAILABLE')
+            WHEN 'WITHDRAW' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id =
+                '00000000-0000-7000-8000-000000000002' AND l.bucket = 'AVAILABLE')
+              AND BOOL_OR(l.side = 'CREDIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+            WHEN 'BET_DEBIT' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id = o.user_id AND l.bucket = 'LOCKED')
+              AND BOOL_OR(l.side = 'CREDIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+            WHEN 'BET_PAYOUT' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+              AND BOOL_OR(l.side = 'CREDIT' AND l.account_id =
+                '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE')
+            WHEN 'BET_REFUND' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+              AND BOOL_OR(l.side = 'CREDIT' AND ((l.account_id = o.user_id AND l.bucket = 'LOCKED')
+                OR (l.account_id = '00000000-0000-7000-8000-000000000001'
+                  AND l.bucket = 'AVAILABLE')))
+            WHEN 'BET_FORFEIT' THEN
+              BOOL_OR(l.side = 'DEBIT' AND l.account_id =
+                '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE')
+              AND BOOL_OR(l.side = 'CREDIT' AND l.account_id = o.user_id AND l.bucket = 'LOCKED')
+            WHEN 'BET_ADJUSTMENT' THEN
+              (BOOL_OR(l.side = 'DEBIT' AND l.account_id = o.user_id AND l.bucket = 'AVAILABLE')
+                AND BOOL_OR(l.side = 'CREDIT' AND l.account_id =
+                  '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE'))
+              OR (BOOL_OR(l.side = 'DEBIT' AND l.account_id =
+                  '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE')
+                AND BOOL_OR(l.side = 'CREDIT' AND l.account_id = o.user_id
+                  AND l.bucket = 'AVAILABLE'))
+          END AS matches_topology
+        FROM wallet_operation o
+        LEFT JOIN ledger_entry l ON l.operation_group_id = o.operation_group_id
+        WHERE o.status = 'SUCCEEDED'
+        GROUP BY o.idempotency_key
+      ), bad_succeeded AS (
+        SELECT idempotency_key FROM succeeded
+        WHERE entries <> 2 OR debits <> 1 OR credits <> 1
+          OR NOT matches_request OR NOT one_timestamp OR NOT matches_topology
+      ), orphan_ledger AS (
+        SELECT DISTINCT l.idempotency_key
+        FROM ledger_entry l
+        LEFT JOIN wallet_operation o ON o.operation_group_id = l.operation_group_id
+        WHERE o.operation_group_id IS NULL OR o.status <> 'SUCCEEDED'
+      )
+      SELECT idempotency_key FROM bad_succeeded
+      UNION
+      SELECT idempotency_key FROM orphan_ledger
+      ORDER BY idempotency_key
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public OperationIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<String> findGroupDriftKeys() {
+    return jdbc.queryForList(GROUP_DRIFT_SQL, String.class);
+  }
+}


## `feat(integrity): reconcile recovery queues`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/RecoveryQueueIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/RecoveryQueueIntegrityRepository.java
new file mode 100644
index 0000000..a4566f1
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/RecoveryQueueIntegrityRepository.java
@@ -0,0 +1,60 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Reconciles recovery debt, freeze state, and the durable FIFO sequence. */
+@Repository
+public class RecoveryQueueIntegrityRepository {
+  private static final String QUEUE_DRIFT_SQL =
+      """
+      WITH queue_summary AS (
+        SELECT user_id,
+          COUNT(queue_sequence) AS queued_count,
+          MAX(queue_sequence) AS max_sequence,
+          COUNT(*) FILTER (WHERE status = 'BLOCKED') AS blocked_count,
+          COALESCE(SUM((-delta_amount)::NUMERIC) FILTER (WHERE status = 'BLOCKED'), 0)
+            AS blocked_debt
+        FROM wallet_adjustment
+        WHERE queue_sequence IS NOT NULL
+        GROUP BY user_id
+      ), identities AS (
+        SELECT user_id FROM account
+        UNION
+        SELECT user_id FROM queue_summary
+      )
+      SELECT i.user_id
+      FROM identities i
+      LEFT JOIN account a ON a.user_id = i.user_id
+      LEFT JOIN queue_summary q ON q.user_id = i.user_id
+      WHERE a.user_id IS NULL
+        OR a.recovery_debt_amount <> COALESCE(q.blocked_debt, 0)
+        OR (a.recovery_debt_amount > 0) <> (a.recovery_frozen_at IS NOT NULL)
+        OR (a.recovery_debt_amount > 0) <> (COALESCE(q.blocked_count, 0) > 0)
+        OR a.next_adjustment_sequence::NUMERIC <>
+          COALESCE(q.max_sequence::NUMERIC + 1, 1)
+        OR COALESCE(q.queued_count::NUMERIC, 0) <> COALESCE(q.max_sequence::NUMERIC, 0)
+        OR EXISTS (
+          SELECT 1
+          FROM wallet_adjustment applied
+          JOIN wallet_adjustment blocked ON blocked.user_id = applied.user_id
+            AND blocked.status = 'BLOCKED'
+            AND blocked.queue_sequence < applied.queue_sequence
+          WHERE applied.user_id = i.user_id
+            AND applied.status = 'APPLIED' AND applied.queue_sequence IS NOT NULL
+        )
+      ORDER BY i.user_id
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public RecoveryQueueIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<UUID> findQueueDriftUsers() {
+    return jdbc.queryForList(QUEUE_DRIFT_SQL, UUID.class);
+  }
+}


## `feat(integrity): reconcile adjustment failures`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFailureIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFailureIntegrityRepository.java
new file mode 100644
index 0000000..6e05e36
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFailureIntegrityRepository.java
@@ -0,0 +1,45 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.List;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Reconciles rejected adjustments with the exact failure shape they can produce. */
+@Repository
+public class AdjustmentFailureIntegrityRepository {
+  private static final String FAILURE_DRIFT_SQL =
+      """
+      SELECT a.idempotency_key
+      FROM wallet_adjustment a
+      JOIN wallet_operation o ON o.idempotency_key = a.idempotency_key
+      WHERE a.status = 'REJECTED' AND (
+        o.failure_detail IS NOT NULL AND (
+          (o.failure_code = 'ACCOUNT_NOT_FOUND'
+            AND o.failure_http_status = 404 AND o.failure_title = 'Account not found'
+            AND o.failure_balance_amount IS NULL AND o.failure_balance_currency IS NULL
+            AND o.failure_expected_currency IS NULL)
+          OR (o.failure_code = 'CURRENCY_MISMATCH'
+            AND o.failure_http_status = 422 AND o.failure_title = 'Currency mismatch'
+            AND o.failure_balance_amount IS NULL AND o.failure_balance_currency IS NULL
+            AND o.failure_expected_currency IN ('KRW', 'USD')
+            AND o.failure_expected_currency <> o.request_currency)
+          OR (o.failure_code = 'AMOUNT_OUT_OF_RANGE' AND a.delta_amount > 0
+            AND o.failure_http_status = 422 AND o.failure_title = 'Amount out of range'
+            AND o.failure_balance_amount IS NOT NULL AND o.failure_balance_amount >= 0
+            AND o.failure_balance_currency = o.request_currency
+            AND o.failure_expected_currency IS NULL)
+        )
+      ) IS NOT TRUE
+      ORDER BY a.idempotency_key
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AdjustmentFailureIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<String> findFailureDriftKeys() {
+    return jdbc.queryForList(FAILURE_DRIFT_SQL, String.class);
+  }
+}


## `feat(integrity): reconcile adjustment outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/AdjustmentOperationIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentOperationIntegrityRepository.java
new file mode 100644
index 0000000..5ff7f3d
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentOperationIntegrityRepository.java
@@ -0,0 +1,68 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.List;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Reconciles every adjustment proof with its durable operation outcome. */
+@Repository
+public class AdjustmentOperationIntegrityRepository {
+  private static final String OUTCOME_DRIFT_SQL =
+      """
+      WITH adjustment_operations AS (
+        SELECT * FROM wallet_operation WHERE operation_kind = 'BET_ADJUSTMENT'
+      ), paired AS (
+        SELECT COALESCE(a.idempotency_key, o.idempotency_key) AS outcome_key,
+          a.idempotency_key AS proof_key, o.idempotency_key AS operation_key,
+          a.status AS proof_status, o.status AS operation_status,
+          a.operation_group_id AS proof_group, o.operation_group_id AS operation_group,
+          a.applied_at, o.completed_at, a.created_at, o.requested_at,
+          o.failure_code, o.failure_http_status, o.failure_title, o.failure_detail,
+          o.failure_balance_amount, o.failure_balance_currency, o.failure_expected_currency,
+          (SELECT COUNT(*) FROM ledger_entry l
+            WHERE l.idempotency_key = COALESCE(a.idempotency_key, o.idempotency_key))
+            AS ledger_entries,
+          a.user_id AS proof_user, o.user_id AS operation_user,
+          a.delta_amount, o.request_amount,
+          a.currency AS proof_currency, o.request_currency AS operation_currency,
+          o.caller_id
+        FROM wallet_adjustment a
+        FULL OUTER JOIN adjustment_operations o ON o.idempotency_key = a.idempotency_key
+      )
+      SELECT outcome_key FROM paired
+      WHERE proof_key IS NULL OR operation_key IS NULL
+        OR caller_id <> 'SETTLEMENT'
+        OR proof_user <> operation_user
+        OR ABS(delta_amount::NUMERIC) <> request_amount::NUMERIC
+        OR proof_currency <> operation_currency
+        OR created_at IS DISTINCT FROM requested_at
+        OR NOT (
+          (proof_status = 'REJECTED' AND operation_status = 'REJECTED'
+            AND proof_group IS NULL AND operation_group IS NULL AND ledger_entries = 0
+            AND failure_code IS NOT NULL AND failure_http_status BETWEEN 400 AND 499
+            AND failure_title IS NOT NULL AND failure_detail IS NOT NULL
+            AND created_at = completed_at)
+          OR (proof_status = 'BLOCKED' AND operation_status = 'BLOCKED_FUNDS'
+            AND proof_group IS NULL AND operation_group IS NULL AND ledger_entries = 0
+            AND completed_at IS NULL
+            AND failure_balance_amount IS NULL AND failure_balance_currency IS NULL
+            AND failure_expected_currency IS NULL)
+          OR (proof_status = 'APPLIED' AND operation_status = 'SUCCEEDED'
+            AND proof_group IS NOT NULL AND proof_group = operation_group
+            AND applied_at = completed_at
+            AND failure_balance_amount IS NULL AND failure_balance_currency IS NULL
+            AND failure_expected_currency IS NULL)
+        )
+      ORDER BY outcome_key
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AdjustmentOperationIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<String> findOutcomeDriftKeys() {
+    return jdbc.queryForList(OUTCOME_DRIFT_SQL, String.class);
+  }
+}


## `feat(integrity): verify adjustment fingerprints`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFingerprintIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFingerprintIntegrityRepository.java
new file mode 100644
index 0000000..bea7db0
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentFingerprintIntegrityRepository.java
@@ -0,0 +1,74 @@
+package com.sportsbook.wallet.integrity;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.service.OperationFingerprint;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Recomputes canonical adjustment identity without duplicating its binary encoder in SQL. */
+@Repository
+public class AdjustmentFingerprintIntegrityRepository {
+  private static final String IDENTITIES_SQL =
+      """
+      SELECT a.idempotency_key, a.revision_id, a.bet_id, a.revision_number, a.user_id,
+        a.previous_payout_amount, a.new_payout_amount, a.currency, o.request_fingerprint
+      FROM wallet_adjustment a
+      JOIN wallet_operation o ON o.idempotency_key = a.idempotency_key
+      ORDER BY a.idempotency_key
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AdjustmentFingerprintIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<String> findFingerprintDriftKeys() {
+    return jdbc
+        .query(
+            IDENTITIES_SQL,
+            (row, number) ->
+                new AdjustmentIdentity(
+                    row.getString("idempotency_key"),
+                    row.getObject("revision_id", UUID.class),
+                    row.getObject("bet_id", UUID.class),
+                    row.getLong("revision_number"),
+                    row.getObject("user_id", UUID.class),
+                    row.getLong("previous_payout_amount"),
+                    row.getLong("new_payout_amount"),
+                    Currency.valueOf(row.getString("currency")),
+                    row.getString("request_fingerprint")))
+        .stream()
+        .filter(identity -> !identity.hasCanonicalFingerprint())
+        .map(AdjustmentIdentity::key)
+        .toList();
+  }
+
+  private record AdjustmentIdentity(
+      String key,
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber,
+      UUID userId,
+      long previousPayout,
+      long newPayout,
+      Currency currency,
+      String storedFingerprint) {
+    private boolean hasCanonicalFingerprint() {
+      return OperationFingerprint.adjustment(
+              WalletCaller.SETTLEMENT,
+              userId,
+              new Money(previousPayout, currency),
+              new Money(newPayout, currency),
+              revisionId,
+              betId,
+              revisionNumber)
+          .value()
+          .equals(storedFingerprint);
+    }
+  }
+}


## `feat(integrity): reconcile adjustment ledgers`

diff --git a/src/main/java/com/sportsbook/wallet/integrity/AdjustmentLedgerIntegrityRepository.java b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentLedgerIntegrityRepository.java
new file mode 100644
index 0000000..fa2c477
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/integrity/AdjustmentLedgerIntegrityRepository.java
@@ -0,0 +1,56 @@
+package com.sportsbook.wallet.integrity;
+
+import java.util.List;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+/** Reconciles applied correction snapshots with their signed two-leg ledger transfer. */
+@Repository
+public class AdjustmentLedgerIntegrityRepository {
+  private static final String LEDGER_DRIFT_SQL =
+      """
+      WITH applied AS (
+        SELECT a.idempotency_key, a.delta_amount,
+          COUNT(l.entry_id) AS entries,
+          COUNT(l.entry_id) FILTER (WHERE l.side = 'DEBIT') AS debits,
+          COUNT(l.entry_id) FILTER (WHERE l.side = 'CREDIT') AS credits,
+          COALESCE(BOOL_AND(
+            l.idempotency_key = a.idempotency_key
+            AND l.operation_group_id = a.operation_group_id
+            AND l.reason = 'BET_ADJUSTMENT'
+            AND l.amount::NUMERIC = ABS(a.delta_amount::NUMERIC)
+            AND l.currency = a.currency
+            AND l.created_at = a.applied_at
+          ), FALSE) AS matches_snapshot,
+          CASE WHEN a.delta_amount > 0 THEN
+            BOOL_OR(l.side = 'DEBIT' AND l.account_id = a.user_id
+              AND l.bucket = 'AVAILABLE')
+            AND BOOL_OR(l.side = 'CREDIT' AND l.account_id =
+              '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE')
+          ELSE
+            BOOL_OR(l.side = 'DEBIT' AND l.account_id =
+              '00000000-0000-7000-8000-000000000001' AND l.bucket = 'AVAILABLE')
+            AND BOOL_OR(l.side = 'CREDIT' AND l.account_id = a.user_id
+              AND l.bucket = 'AVAILABLE')
+          END AS matches_signed_topology
+        FROM wallet_adjustment a
+        LEFT JOIN ledger_entry l ON l.operation_group_id = a.operation_group_id
+        WHERE a.status = 'APPLIED'
+        GROUP BY a.revision_id
+      )
+      SELECT idempotency_key FROM applied
+      WHERE entries <> 2 OR debits <> 1 OR credits <> 1
+        OR NOT matches_snapshot OR NOT matches_signed_topology
+      ORDER BY idempotency_key
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public AdjustmentLedgerIntegrityRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public List<String> findLedgerDriftKeys() {
+    return jdbc.queryForList(LEDGER_DRIFT_SQL, String.class);
+  }
+}


