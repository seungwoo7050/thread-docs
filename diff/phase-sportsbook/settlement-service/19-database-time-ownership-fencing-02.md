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


## `test(correction): verify database timed finalization fence`

diff --git a/src/test/java/com/sportsbook/settlement/correction/AppliedRevisionLeaseTest.java b/src/test/java/com/sportsbook/settlement/correction/AppliedRevisionLeaseTest.java
index 7ac5e80..59a4f8c 100644
--- a/src/test/java/com/sportsbook/settlement/correction/AppliedRevisionLeaseTest.java
+++ b/src/test/java/com/sportsbook/settlement/correction/AppliedRevisionLeaseTest.java
@@ -12,6 +12,7 @@ import com.sportsbook.protocol.value.Money;
 import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import java.sql.Timestamp;
 import java.time.Instant;
+import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.mockito.ArgumentCaptor;
@@ -22,28 +23,33 @@ class AppliedRevisionLeaseTest {
   @Test
   void consumesTheExactOwnerAndRetainsRecoveredWalletEvidence() {
     JdbcTemplate jdbc = mock(JdbcTemplate.class);
-    when(jdbc.update(anyString(), any(Object[].class))).thenReturn(1);
+    Instant appliedAt = Instant.EPOCH.plusSeconds(3);
+    when(jdbc.query(
+            anyString(), any(org.springframework.jdbc.core.RowMapper.class), any(Object[].class)))
+        .thenReturn(List.of(appliedAt));
     RevisionLease lease = new RevisionLease(UUID.randomUUID(), Instant.EPOCH.plusSeconds(30));
 
-    assertThat(
-            new RevisionPlanRepository(jdbc)
-                .markApplied(UUID.randomUUID(), lease, applied(), Instant.EPOCH.plusSeconds(3)))
-        .isTrue();
+    assertThat(new RevisionPlanRepository(jdbc).markApplied(UUID.randomUUID(), lease, applied()))
+        .contains(appliedAt);
 
     ArgumentCaptor<String> sql = ArgumentCaptor.forClass(String.class);
     ArgumentCaptor<Object[]> values = ArgumentCaptor.forClass(Object[].class);
-    verify(jdbc).update(sql.capture(), values.capture());
+    verify(jdbc)
+        .query(sql.capture(), any(org.springframework.jdbc.core.RowMapper.class), values.capture());
     assertThat(sql.getValue())
         .contains(
             "state = 'APPLIED'",
             "lease_token = null",
             "next_retry_at = null",
-            "lease_until > current_timestamp");
+            "lease_until > current_timestamp",
+            "source_result_settled_at <= current_timestamp",
+            "applied_at = date_trunc('milliseconds', current_timestamp)",
+            "returning applied_at");
     assertThat(values.getValue()[0]).isEqualTo("APPLIED");
     assertThat(values.getValue()[1]).isEqualTo(7L);
     assertThat(values.getValue()[2]).isInstanceOf(UUID.class);
     assertThat(values.getValue()[3]).isInstanceOf(Timestamp.class);
-    assertThat(values.getValue()[8]).isEqualTo(lease.token());
+    assertThat(values.getValue()[6]).isEqualTo(lease.token());
   }
 
   private static WalletAdjustmentProof applied() {
