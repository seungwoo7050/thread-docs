## `feat(correction): consume applied revision leases`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index 7d15fea..6a28666 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -1,6 +1,7 @@
 package com.sportsbook.settlement.correction;
 
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.nullable;
 
 import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import com.sportsbook.settlement.client.WalletFailurePolicy;
@@ -188,5 +189,35 @@ public class RevisionPlanRepository {
         .findFirst();
   }
 
+  public boolean markApplied(
+      UUID revisionId,
+      RevisionLease lease,
+      WalletAdjustmentProof proof,
+      Instant now) {
+    if (proof != null && proof.status() != WalletAdjustmentProof.Status.APPLIED) {
+      throw new IllegalArgumentException("Revision finalization requires an applied Wallet proof");
+    }
+    return jdbc.update(
+            """
+            update settlement_revision set state = 'APPLIED', lease_token = null,
+                lease_until = null, last_error_code = null, wallet_status = ?,
+                wallet_queue_sequence = ?, wallet_operation_group_id = ?, wallet_queued_at = ?,
+                wallet_applied_at = ?, wallet_next_attempt_at = null, next_retry_at = null,
+                updated_at = ?, applied_at = ?
+            where revision_id = ? and state = 'PENDING' and lease_token = ?
+                and lease_until > current_timestamp
+            """,
+            proof == null ? null : proof.status().name(),
+            proof == null ? null : proof.queueSequence(),
+            proof == null ? null : proof.operationGroupId(),
+            nullable(proof == null ? null : proof.queuedAt()),
+            nullable(proof == null ? null : proof.appliedAt()),
+            required(now),
+            required(now),
+            revisionId,
+            lease.token())
+        == 1;
+  }
+
   public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
 }


## `feat(correction): fence finalization with database time`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index 88a32f8..0880c0e 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -196,31 +196,40 @@ public class RevisionPlanRepository {
         .findFirst();
   }
 
-  public boolean markApplied(
-      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof, Instant now) {
+  public Optional<Instant> markApplied(
+      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof) {
     if (proof != null && proof.status() != WalletAdjustmentProof.Status.APPLIED) {
       throw new IllegalArgumentException("Revision finalization requires an applied Wallet proof");
     }
-    return jdbc.update(
+    return jdbc
+        .query(
             """
             update settlement_revision set state = 'APPLIED', lease_token = null,
                 lease_until = null, last_error_code = null, wallet_status = ?,
                 wallet_queue_sequence = ?, wallet_operation_group_id = ?, wallet_queued_at = ?,
                 wallet_applied_at = ?, wallet_next_attempt_at = null, next_retry_at = null,
-                updated_at = ?, applied_at = ?
+                updated_at = date_trunc('milliseconds', current_timestamp),
+                applied_at = date_trunc('milliseconds', current_timestamp)
             where revision_id = ? and state = 'PENDING' and lease_token = ?
                 and lease_until > current_timestamp
+                and source_result_settled_at <= current_timestamp
+            returning applied_at
             """,
+            (result, rowNumber) -> result.getTimestamp("applied_at").toInstant(),
             proof == null ? null : proof.status().name(),
             proof == null ? null : proof.queueSequence(),
             proof == null ? null : proof.operationGroupId(),
             nullable(proof == null ? null : proof.queuedAt()),
             nullable(proof == null ? null : proof.appliedAt()),
-            required(now),
-            required(now),
             revisionId,
             lease.token())
-        == 1;
+        .stream()
+        .findFirst();
+  }
+
+  public boolean markApplied(
+      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof, Instant ignored) {
+    return markApplied(revisionId, lease, proof).isPresent();
   }
 
   public boolean markRejected(


## `feat(recovery): claim due revision leases`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java
new file mode 100644
index 0000000..d1b1253
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java
@@ -0,0 +1,77 @@
+package com.sportsbook.settlement.correction;
+
+import java.sql.Timestamp;
+import java.time.Duration;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class RevisionRecoveryRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public RevisionRecoveryRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional
+  public List<Claim> claimDue(Duration leaseDuration, int limit) {
+    long leaseMillis = leaseDuration == null ? 0 : leaseDuration.toMillis();
+    if (leaseMillis < 1 || limit < 1 || limit > 1000) {
+      throw new IllegalArgumentException("Invalid revision recovery bounds");
+    }
+    jdbc.update(
+        """
+        update settlement_revision set
+            state = case when wallet_status = 'BLOCKED' then 'BLOCKED' else 'EXHAUSTED' end,
+            lease_token = null, lease_until = null, next_retry_at = null,
+            last_error_code = 'WALLET_RETRY_EXHAUSTED', updated_at = current_timestamp
+        where state in ('PENDING', 'BLOCKED') and attempt_count >= 12
+            and lease_token is not null and lease_until <= current_timestamp
+        """);
+    List<UUID> due =
+        jdbc.query(
+            """
+            select revision_id from settlement_revision
+            where attempt_count < 12 and state in ('PENDING', 'BLOCKED') and (
+                (lease_token is null and next_retry_at <= current_timestamp)
+                or (lease_token is not null and lease_until <= current_timestamp))
+            order by coalesce(next_retry_at, lease_until), revision_id
+            limit ? for update skip locked
+            """,
+            (result, rowNumber) -> result.getObject("revision_id", UUID.class),
+            limit);
+    List<Claim> claimed = new ArrayList<>(due.size());
+    for (UUID revisionId : due) {
+      UUID token = UUID.randomUUID();
+      Timestamp until =
+          jdbc
+              .query(
+                  """
+                  update settlement_revision set state = 'PENDING', lease_token = ?,
+                      lease_until = current_timestamp + (? * interval '1 millisecond'),
+                      attempt_count = attempt_count + 1, last_error_code = null,
+                      next_retry_at = null,
+                      updated_at = current_timestamp
+                  where revision_id = ? and attempt_count < 12
+                      and state in ('PENDING', 'BLOCKED')
+                  returning lease_until
+                  """,
+                  (result, rowNumber) -> result.getTimestamp("lease_until"),
+                  token,
+                  leaseMillis,
+                  revisionId)
+              .stream()
+              .findFirst()
+              .orElseThrow(() -> new IllegalStateException("Revision claim lost its locked row"));
+      claimed.add(new Claim(revisionId, new RevisionLease(token, until.toInstant())));
+    }
+    return List.copyOf(claimed);
+  }
+
+  public record Claim(UUID revisionId, RevisionLease lease) {}
+}


## `feat(recovery): scan durable revision claims`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
new file mode 100644
index 0000000..52deda6
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java
@@ -0,0 +1,45 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import java.util.ArrayList;
+import java.util.List;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class RevisionRecoveryScanner {
+
+  private final RevisionRecoveryRepository recovery;
+  private final RevisionPlanReader plans;
+  private final RevisionExecutionRunner runner;
+  private final SettlementRuntimeProperties runtime;
+
+  public RevisionRecoveryScanner(
+      RevisionRecoveryRepository recovery,
+      RevisionPlanReader plans,
+      RevisionExecutionRunner runner,
+      SettlementRuntimeProperties runtime) {
+    this.recovery = recovery;
+    this.plans = plans;
+    this.runner = runner;
+    this.runtime = runtime;
+  }
+
+  @Scheduled(
+      fixedDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      scheduler = SettlementWorkerConfiguration.RECOVERY)
+  public List<RevisionExecutionRunner.Result> recover() {
+    var claims = recovery.claimDue(runtime.leaseDuration(), runtime.batchSize());
+    List<RevisionExecutionRunner.Result> results = new ArrayList<>(claims.size());
+    for (var claim : claims) {
+      RevisionPlan plan =
+          plans
+              .find(claim.revisionId())
+              .orElseThrow(() -> new IllegalStateException("Claimed revision plan is missing"));
+      results.add(runner.execute(plan, claim.lease(), true, !claim.blockedProof()));
+    }
+    return List.copyOf(results);
+  }
+}


## `feat(recovery): protect blocked wallet proofs`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java b/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java
index 8f0d7dd..c699dc6 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java
@@ -26,6 +26,11 @@ public class RevisionExecutionRunner {
   }
 
   public Result execute(RevisionPlan plan, RevisionLease lease, boolean recoverFirst) {
+    return execute(plan, lease, recoverFirst, true);
+  }
+
+  public Result execute(
+      RevisionPlan plan, RevisionLease lease, boolean recoverFirst, boolean submitWhenMissing) {
     if (!plan.requiresWalletAdjustment()) {
       return finalizer.apply(plan, lease, null, clock.instant())
           ? Result.APPLIED
@@ -33,7 +38,7 @@ public class RevisionExecutionRunner {
     }
     try {
       WalletAdjustmentProof proof =
-          recoverFirst ? wallet.recoverAmbiguous(plan) : wallet.submit(plan);
+          recoverFirst ? wallet.recoverAmbiguous(plan, submitWhenMissing) : wallet.submit(plan);
       Instant completedAt = clock.instant();
       return switch (proof.status()) {
         case APPLIED ->
diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
index ef27800..236f667 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
@@ -28,11 +28,13 @@ public class RevisionWalletGateway {
     return proofs.requireExact(plan, proof);
   }
 
-  public WalletAdjustmentProof recoverAmbiguous(RevisionPlan plan) {
+  public WalletAdjustmentProof recoverAmbiguous(RevisionPlan plan, boolean submitWhenMissing) {
     try {
       return proofs.requireExact(plan, wallet.findAdjustment(plan.revisionId()));
     } catch (WalletFailurePolicy.PermanentFailure failure) {
-      if (failure.status() == 404 && "WALLET_ADJUSTMENT_NOT_FOUND".equals(failure.errorCode())) {
+      if (submitWhenMissing
+          && failure.status() == 404
+          && "WALLET_ADJUSTMENT_NOT_FOUND".equals(failure.errorCode())) {
         return submit(plan);
       }
       throw failure;


## `feat(recovery): retain blocked proof claims`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java
index d1b1253..8376d66 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryRepository.java
@@ -33,20 +33,27 @@ public class RevisionRecoveryRepository {
         where state in ('PENDING', 'BLOCKED') and attempt_count >= 12
             and lease_token is not null and lease_until <= current_timestamp
         """);
-    List<UUID> due =
+    List<Candidate> due =
         jdbc.query(
             """
-            select revision_id from settlement_revision
+            select revision_id, wallet_status from settlement_revision
             where attempt_count < 12 and state in ('PENDING', 'BLOCKED') and (
-                (lease_token is null and next_retry_at <= current_timestamp)
+                (lease_token is null and (
+                    (state = 'PENDING' and next_retry_at <= current_timestamp)
+                    or (state = 'BLOCKED' and next_retry_at <= current_timestamp
+                        and wallet_status = 'BLOCKED'
+                        and wallet_next_attempt_at <= current_timestamp)))
                 or (lease_token is not null and lease_until <= current_timestamp))
             order by coalesce(next_retry_at, lease_until), revision_id
             limit ? for update skip locked
             """,
-            (result, rowNumber) -> result.getObject("revision_id", UUID.class),
+            (result, rowNumber) ->
+                new Candidate(
+                    result.getObject("revision_id", UUID.class),
+                    "BLOCKED".equals(result.getString("wallet_status"))),
             limit);
     List<Claim> claimed = new ArrayList<>(due.size());
-    for (UUID revisionId : due) {
+    for (Candidate candidate : due) {
       UUID token = UUID.randomUUID();
       Timestamp until =
           jdbc
@@ -64,14 +71,20 @@ public class RevisionRecoveryRepository {
                   (result, rowNumber) -> result.getTimestamp("lease_until"),
                   token,
                   leaseMillis,
-                  revisionId)
+                  candidate.revisionId())
               .stream()
               .findFirst()
               .orElseThrow(() -> new IllegalStateException("Revision claim lost its locked row"));
-      claimed.add(new Claim(revisionId, new RevisionLease(token, until.toInstant())));
+      claimed.add(
+          new Claim(
+              candidate.revisionId(),
+              new RevisionLease(token, until.toInstant()),
+              candidate.blockedProof()));
     }
     return List.copyOf(claimed);
   }
 
-  public record Claim(UUID revisionId, RevisionLease lease) {}
+  record Candidate(UUID revisionId, boolean blockedProof) {}
+
+  public record Claim(UUID revisionId, RevisionLease lease, boolean blockedProof) {}
 }


## `test(correction): verify GET first ambiguity recovery`

diff --git a/src/test/java/com/sportsbook/settlement/correction/RevisionWalletGatewayTest.java b/src/test/java/com/sportsbook/settlement/correction/RevisionWalletGatewayTest.java
index 8e189ff..6379071 100644
--- a/src/test/java/com/sportsbook/settlement/correction/RevisionWalletGatewayTest.java
+++ b/src/test/java/com/sportsbook/settlement/correction/RevisionWalletGatewayTest.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.correction;
 
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.when;
 
@@ -14,10 +15,14 @@ import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import com.sportsbook.settlement.client.WalletClient;
 import com.sportsbook.settlement.client.WalletFailurePolicy;
 import com.sportsbook.settlement.resolver.ResolvedSelection;
+import java.io.ByteArrayInputStream;
+import java.nio.charset.StandardCharsets;
 import java.time.Instant;
 import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.client.ClientHttpResponse;
 
 class RevisionWalletGatewayTest {
 
@@ -56,6 +61,52 @@ class RevisionWalletGatewayTest {
         .hasMessageContaining("WALLET_MALFORMED_RESPONSE");
   }
 
+  @Test
+  void getsBeforeRepostingOnlyWhenNoWalletProofExists() throws Exception {
+    WalletClient wallet = mock(WalletClient.class);
+    RevisionPlan plan = plan();
+    WalletFailurePolicy.PermanentFailure missing = notFound();
+    when(wallet.findAdjustment(plan.revisionId())).thenThrow(missing);
+    when(wallet.adjust(
+            plan.revisionId(),
+            plan.target().betId(),
+            1,
+            plan.target().userId(),
+            Money.krw(200),
+            Money.krw(100)))
+        .thenReturn(applied(plan, plan.target().userId()));
+
+    new RevisionWalletGateway(wallet).recoverAmbiguous(plan);
+
+    var ordered = inOrder(wallet);
+    ordered.verify(wallet).findAdjustment(plan.revisionId());
+    ordered
+        .verify(wallet)
+        .adjust(
+            plan.revisionId(),
+            plan.target().betId(),
+            1,
+            plan.target().userId(),
+            Money.krw(200),
+            Money.krw(100));
+  }
+
+  private static WalletFailurePolicy.PermanentFailure notFound() throws Exception {
+    ClientHttpResponse response = mock(ClientHttpResponse.class);
+    when(response.getStatusCode()).thenReturn(HttpStatus.NOT_FOUND);
+    when(response.getBody())
+        .thenReturn(
+            new ByteArrayInputStream(
+                "{\"errorCode\":\"WALLET_ADJUSTMENT_NOT_FOUND\"}"
+                    .getBytes(StandardCharsets.UTF_8)));
+    try {
+      WalletFailurePolicy.throwFor(response);
+      throw new AssertionError("Expected missing adjustment");
+    } catch (WalletFailurePolicy.PermanentFailure failure) {
+      return failure;
+    }
+  }
+
   private static WalletAdjustmentProof applied(RevisionPlan plan, UUID userId) {
     return new WalletAdjustmentProof(
         plan.revisionId(),
