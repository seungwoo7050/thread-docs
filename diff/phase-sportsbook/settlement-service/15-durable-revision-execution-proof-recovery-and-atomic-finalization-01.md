# 내구성 있는 수정 실행, 증거 복구, 원자적 확정

## `feat(correction): lease revision ownership`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionLease.java b/src/main/java/com/sportsbook/settlement/correction/RevisionLease.java
new file mode 100644
index 0000000..0534195
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionLease.java
@@ -0,0 +1,26 @@
+package com.sportsbook.settlement.correction;
+
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record RevisionLease(UUID token, Instant until) {
+
+  public RevisionLease {
+    Objects.requireNonNull(token, "token");
+    Objects.requireNonNull(until, "until");
+  }
+
+  public static RevisionLease acquire(Instant now, Duration duration) {
+    Objects.requireNonNull(now, "now");
+    if (duration == null || duration.isZero() || duration.isNegative()) {
+      throw new IllegalArgumentException("Revision lease duration must be positive");
+    }
+    return new RevisionLease(UUID.randomUUID(), now.plus(duration));
+  }
+
+  public boolean isExpiredAt(Instant instant) {
+    return !until.isAfter(Objects.requireNonNull(instant, "instant"));
+  }
+}


## `feat(correction): submit exact wallet adjustments`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionProofValidator.java b/src/main/java/com/sportsbook/settlement/correction/RevisionProofValidator.java
new file mode 100644
index 0000000..d455948
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionProofValidator.java
@@ -0,0 +1,54 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.client.WalletAdjustmentProof;
+import com.sportsbook.settlement.client.WalletFailurePolicy;
+import java.util.Objects;
+
+final class RevisionProofValidator {
+
+  WalletAdjustmentProof requireExact(RevisionPlan plan, WalletAdjustmentProof proof) {
+    RevisionTarget target = plan.target();
+    if (proof == null
+        || !Objects.equals(proof.revisionId(), plan.revisionId())
+        || !Objects.equals(proof.betId(), target.betId())
+        || proof.revisionNumber() != target.revisionNumber()
+        || !Objects.equals(proof.userId(), target.userId())
+        || !Objects.equals(proof.previousPayout(), target.previousPayout())
+        || !Objects.equals(proof.newPayout(), plan.newPayout())
+        || proof.deltaAmount() != plan.deltaAmount()
+        || proof.currency() != plan.newPayout().currency()
+        || proof.status() == null) {
+      throw WalletFailurePolicy.malformedSuccess();
+    }
+    boolean valid =
+        switch (proof.status()) {
+          case APPLIED ->
+              proof.operationGroupId() != null
+                  && proof.appliedAt() != null
+                  && proof.nextAttemptAt() == null
+                  && ((proof.queueSequence() == null && proof.queuedAt() == null)
+                      || (plan.deltaAmount() < 0
+                          && proof.queueSequence() != null
+                          && proof.queueSequence() > 0
+                          && proof.queuedAt() != null));
+          case BLOCKED ->
+              plan.deltaAmount() < 0
+                  && proof.queueSequence() != null
+                  && proof.queueSequence() > 0
+                  && proof.queuedAt() != null
+                  && proof.nextAttemptAt() != null
+                  && proof.operationGroupId() == null
+                  && proof.appliedAt() == null;
+          case REJECTED ->
+              proof.queueSequence() == null
+                  && proof.operationGroupId() == null
+                  && proof.queuedAt() == null
+                  && proof.appliedAt() == null
+                  && proof.nextAttemptAt() == null;
+        };
+    if (!valid) {
+      throw WalletFailurePolicy.malformedSuccess();
+    }
+    return proof;
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
new file mode 100644
index 0000000..bb636f6
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
@@ -0,0 +1,29 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.client.WalletAdjustmentProof;
+import com.sportsbook.settlement.client.WalletClient;
+import org.springframework.stereotype.Component;
+
+@Component
+public class RevisionWalletGateway {
+
+  private final WalletClient wallet;
+  private final RevisionProofValidator proofs = new RevisionProofValidator();
+
+  public RevisionWalletGateway(WalletClient wallet) {
+    this.wallet = wallet;
+  }
+
+  public WalletAdjustmentProof submit(RevisionPlan plan) {
+    RevisionTarget target = plan.target();
+    WalletAdjustmentProof proof =
+        wallet.adjust(
+            plan.revisionId(),
+            target.betId(),
+            target.revisionNumber(),
+            target.userId(),
+            target.previousPayout(),
+            plan.newPayout());
+    return proofs.requireExact(plan, proof);
+  }
+}


## `feat(correction): persist blocked adjustments`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index a5680ba..15da9c5 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -2,9 +2,12 @@ package com.sportsbook.settlement.correction;
 
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
+import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import java.sql.Timestamp;
 import java.time.Duration;
+import java.time.Instant;
 import java.util.List;
+import java.util.Optional;
 import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
@@ -93,5 +96,39 @@ public class RevisionPlanRepository {
     return new Persisted(existing, false, null);
   }
 
+  public Optional<RevisionState> markBlocked(
+      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof, Instant now) {
+    if (proof.status() != WalletAdjustmentProof.Status.BLOCKED) {
+      throw new IllegalArgumentException("Only a blocked Wallet proof can block a revision");
+    }
+    return jdbc
+        .query(
+            """
+            update settlement_revision set
+                state = 'BLOCKED',
+                lease_token = null, lease_until = null,
+                last_error_code = case when attempt_count >= 12
+                    then 'WALLET_RETRY_EXHAUSTED' else null end,
+                wallet_status = 'BLOCKED', wallet_queue_sequence = ?,
+                wallet_queued_at = cast(? as timestamptz),
+                wallet_next_attempt_at = cast(? as timestamptz),
+                next_retry_at = case when attempt_count >= 12 then null
+                    else cast(? as timestamptz) end,
+                updated_at = ? where revision_id = ? and state = 'PENDING' and lease_token = ?
+                    and lease_until > current_timestamp
+            returning state
+            """,
+            (result, rowNumber) -> RevisionState.valueOf(result.getString("state")),
+            proof.queueSequence(),
+            required(proof.queuedAt()),
+            required(proof.nextAttemptAt()),
+            required(proof.nextAttemptAt()),
+            required(now),
+            revisionId,
+            lease.token())
+        .stream()
+        .findFirst();
+  }
+
   public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
 }


## `feat(correction): release transient revisions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index 15da9c5..f2aca78 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -3,6 +3,7 @@ package com.sportsbook.settlement.correction;
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import com.sportsbook.settlement.client.WalletAdjustmentProof;
+import com.sportsbook.settlement.client.WalletFailurePolicy;
 import java.sql.Timestamp;
 import java.time.Duration;
 import java.time.Instant;
@@ -130,5 +131,35 @@ public class RevisionPlanRepository {
         .findFirst();
   }
 
+  public Optional<RevisionState> releaseTransient(
+      UUID revisionId, RevisionLease lease, WalletFailurePolicy.TransientFailure failure) {
+    String code =
+        failure.errorCode().matches("[A-Z0-9_]{1,128}") ? failure.errorCode() : "WALLET_FAILURE";
+    return jdbc
+        .query(
+            """
+            update settlement_revision set
+                state = case when wallet_status = 'BLOCKED' then 'BLOCKED'
+                    when attempt_count >= 12 then 'EXHAUSTED' else 'PENDING' end,
+                lease_token = null, lease_until = null,
+                last_error_code = case when attempt_count >= 12
+                    then 'WALLET_RETRY_EXHAUSTED' else ? end,
+                updated_at = current_timestamp,
+                next_retry_at = case when attempt_count >= 12 then null
+                    else current_timestamp + least(interval '300 seconds',
+                        interval '1 second' * power(2,
+                            least(greatest(attempt_count - 1, 0), 9))) end
+            where revision_id = ? and state = 'PENDING' and lease_token = ?
+                and lease_until > current_timestamp
+            returning state
+            """,
+            (result, rowNumber) -> RevisionState.valueOf(result.getString("state")),
+            code,
+            revisionId,
+            lease.token())
+        .stream()
+        .findFirst();
+  }
+
   public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
 }


## `feat(correction): quarantine rejected revisions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index f2aca78..7d15fea 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -161,5 +161,32 @@ public class RevisionPlanRepository {
         .findFirst();
   }
 
+  public Optional<RevisionState> rejectPermanent(
+      UUID revisionId,
+      RevisionLease lease,
+      WalletFailurePolicy.PermanentFailure failure,
+      Instant now) {
+    String code =
+        failure.errorCode().matches("[A-Z0-9_]{1,128}") ? failure.errorCode() : "WALLET_FAILURE";
+    return jdbc
+        .query(
+            """
+            update settlement_revision set
+                state = case when wallet_status = 'BLOCKED' then 'BLOCKED' else 'REJECTED' end,
+                lease_token = null, lease_until = null, next_retry_at = null,
+                last_error_code = ?, updated_at = ?
+            where revision_id = ? and state = 'PENDING' and lease_token = ?
+                and lease_until > current_timestamp
+            returning state
+            """,
+            (result, rowNumber) -> RevisionState.valueOf(result.getString("state")),
+            code,
+            required(now),
+            revisionId,
+            lease.token())
+        .stream()
+        .findFirst();
+  }
+
   public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
 }


## `feat(correction): execute durable revision plans`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java b/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java
new file mode 100644
index 0000000..8f0d7dd
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionExecutionRunner.java
@@ -0,0 +1,80 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.client.WalletAdjustmentProof;
+import com.sportsbook.settlement.client.WalletFailurePolicy;
+import java.time.Clock;
+import java.time.Instant;
+import org.springframework.stereotype.Component;
+
+@Component
+public class RevisionExecutionRunner {
+
+  private final RevisionWalletGateway wallet;
+  private final RevisionPlanRepository revisions;
+  private final RevisionFinalizer finalizer;
+  private final Clock clock;
+
+  public RevisionExecutionRunner(
+      RevisionWalletGateway wallet,
+      RevisionPlanRepository revisions,
+      RevisionFinalizer finalizer,
+      Clock clock) {
+    this.wallet = wallet;
+    this.revisions = revisions;
+    this.finalizer = finalizer;
+    this.clock = clock;
+  }
+
+  public Result execute(RevisionPlan plan, RevisionLease lease, boolean recoverFirst) {
+    if (!plan.requiresWalletAdjustment()) {
+      return finalizer.apply(plan, lease, null, clock.instant())
+          ? Result.APPLIED
+          : Result.LOST_OWNERSHIP;
+    }
+    try {
+      WalletAdjustmentProof proof =
+          recoverFirst ? wallet.recoverAmbiguous(plan) : wallet.submit(plan);
+      Instant completedAt = clock.instant();
+      return switch (proof.status()) {
+        case APPLIED ->
+            finalizer.apply(plan, lease, proof, completedAt)
+                ? Result.APPLIED
+                : Result.LOST_OWNERSHIP;
+        case BLOCKED ->
+            revisions
+                .markBlocked(plan.revisionId(), lease, proof, completedAt)
+                .map(state -> Result.BLOCKED)
+                .orElse(Result.LOST_OWNERSHIP);
+        case REJECTED ->
+            revisions.markRejected(plan.revisionId(), lease, proof, completedAt)
+                ? Result.REJECTED
+                : Result.LOST_OWNERSHIP;
+      };
+    } catch (WalletFailurePolicy.TransientFailure failure) {
+      return revisions
+          .releaseTransient(plan.revisionId(), lease, failure)
+          .map(
+              state ->
+                  switch (state) {
+                    case BLOCKED -> Result.BLOCKED;
+                    case EXHAUSTED -> Result.EXHAUSTED;
+                    default -> Result.RETRY;
+                  })
+          .orElse(Result.LOST_OWNERSHIP);
+    } catch (WalletFailurePolicy.PermanentFailure failure) {
+      return revisions
+          .rejectPermanent(plan.revisionId(), lease, failure, clock.instant())
+          .map(state -> state == RevisionState.BLOCKED ? Result.BLOCKED : Result.REJECTED)
+          .orElse(Result.LOST_OWNERSHIP);
+    }
+  }
+
+  public enum Result {
+    APPLIED,
+    BLOCKED,
+    EXHAUSTED,
+    REJECTED,
+    RETRY,
+    LOST_OWNERSHIP
+  }
+}


## `feat(correction): recover ambiguous adjustments`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
index bb636f6..ef27800 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionWalletGateway.java
@@ -2,6 +2,7 @@ package com.sportsbook.settlement.correction;
 
 import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import com.sportsbook.settlement.client.WalletClient;
+import com.sportsbook.settlement.client.WalletFailurePolicy;
 import org.springframework.stereotype.Component;
 
 @Component
@@ -26,4 +27,15 @@ public class RevisionWalletGateway {
             plan.newPayout());
     return proofs.requireExact(plan, proof);
   }
+
+  public WalletAdjustmentProof recoverAmbiguous(RevisionPlan plan) {
+    try {
+      return proofs.requireExact(plan, wallet.findAdjustment(plan.revisionId()));
+    } catch (WalletFailurePolicy.PermanentFailure failure) {
+      if (failure.status() == 404 && "WALLET_ADJUSTMENT_NOT_FOUND".equals(failure.errorCode())) {
+        return submit(plan);
+      }
+      throw failure;
+    }
+  }
 }


## `feat(correction): persist rejected adjustment proofs`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index 6a28666..2dce2f1 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -1,7 +1,7 @@
 package com.sportsbook.settlement.correction;
 
-import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.nullable;
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import com.sportsbook.settlement.client.WalletAdjustmentProof;
 import com.sportsbook.settlement.client.WalletFailurePolicy;
@@ -190,10 +190,7 @@ public class RevisionPlanRepository {
   }
 
   public boolean markApplied(
-      UUID revisionId,
-      RevisionLease lease,
-      WalletAdjustmentProof proof,
-      Instant now) {
+      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof, Instant now) {
     if (proof != null && proof.status() != WalletAdjustmentProof.Status.APPLIED) {
       throw new IllegalArgumentException("Revision finalization requires an applied Wallet proof");
     }
@@ -219,5 +216,27 @@ public class RevisionPlanRepository {
         == 1;
   }
 
+  public boolean markRejected(
+      UUID revisionId, RevisionLease lease, WalletAdjustmentProof proof, Instant now) {
+    if (proof.status() != WalletAdjustmentProof.Status.REJECTED) {
+      throw new IllegalArgumentException("Only a rejected Wallet proof can reject a revision");
+    }
+    return jdbc.update(
+            """
+            update settlement_revision set state = 'REJECTED', lease_token = null,
+                lease_until = null, last_error_code = 'WALLET_REJECTED',
+                wallet_status = 'REJECTED', wallet_queue_sequence = null,
+                wallet_operation_group_id = null, wallet_queued_at = null,
+                wallet_applied_at = null, wallet_next_attempt_at = null, next_retry_at = null,
+                updated_at = ?
+            where revision_id = ? and state = 'PENDING' and lease_token = ?
+                and lease_until > current_timestamp
+            """,
+            required(now),
+            revisionId,
+            lease.token())
+        == 1;
+  }
+
   public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
 }


## `feat(correction): finalize revisions atomically`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionFinalizer.java b/src/main/java/com/sportsbook/settlement/correction/RevisionFinalizer.java
new file mode 100644
index 0000000..1d24280
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionFinalizer.java
@@ -0,0 +1,84 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.client.WalletAdjustmentProof;
+import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.outbox.OutboxEventRepository;
+import com.sportsbook.settlement.outbox.SettlementEventFactory;
+import com.sportsbook.settlement.outbox.StrictAvroEncoder;
+import com.sportsbook.settlement.persistence.BetRepository;
+import com.sportsbook.settlement.resolver.ResolvedSelection;
+import java.time.Instant;
+import java.util.Map;
+import java.util.function.Function;
+import java.util.stream.Collectors;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+@Component
+public class RevisionFinalizer {
+
+  private final BetRepository bets;
+  private final RevisionPlanRepository revisions;
+  private final OutboxEventRepository outbox;
+  private final SettlementEventFactory events;
+
+  public RevisionFinalizer(
+      BetRepository bets,
+      RevisionPlanRepository revisions,
+      OutboxEventRepository outbox,
+      SettlementTopics topics) {
+    this.bets = bets;
+    this.revisions = revisions;
+    this.outbox = outbox;
+    this.events = new SettlementEventFactory(topics, new StrictAvroEncoder());
+  }
+
+  @Transactional
+  public boolean apply(
+      RevisionPlan plan, RevisionLease lease, WalletAdjustmentProof proof, Instant now) {
+    if (plan.requiresWalletAdjustment()) {
+      new RevisionProofValidator().requireExact(plan, proof);
+      if (proof.status() != WalletAdjustmentProof.Status.APPLIED) {
+        throw new IllegalArgumentException("Only applied adjustments can finalize a revision");
+      }
+    } else if (proof != null) {
+      throw new IllegalArgumentException("Zero-delta revisions must not contact Wallet");
+    }
+    RevisionTarget target = plan.target();
+    Bet bet = bets.findForUpdateById(target.betId()).orElseThrow();
+    Map<java.util.UUID, ResolvedSelection> snapshot =
+        target.selections().stream()
+            .collect(Collectors.toMap(ResolvedSelection::selectionId, Function.identity()));
+    boolean stale =
+        bet.status() != SettlementStatus.SETTLED
+            || bet.revisionNumber() != target.revisionNumber() - 1
+            || !bet.userId().equals(target.userId())
+            || bet.result() != target.previousResult()
+            || !target.previousPayout().equals(bet.payout())
+            || snapshot.size() != bet.selections().size()
+            || bet.selections().stream()
+                .anyMatch(selection -> !snapshot.containsKey(selection.selectionId()))
+            || bet.selections().stream()
+                .noneMatch(selection -> selection.eventId().equals(target.eventId()));
+    if (stale || !revisions.markApplied(plan.revisionId(), lease, proof, now)) {
+      return false;
+    }
+    bet.selections()
+        .forEach(
+            selection -> {
+              var resolved = snapshot.get(selection.selectionId());
+              if (selection.eventId().equals(target.eventId())) {
+                selection.applyCandidate(target.sourceCandidateId(), resolved.outcome());
+              } else if (selection.outcome() != resolved.outcome()) {
+                throw new IllegalStateException("Unrelated selection changed during correction");
+              }
+            });
+    if (bet.recordRevision(plan.newResult(), plan.newPayout(), now) != target.revisionNumber()) {
+      throw new IllegalStateException("Bet revision sequence diverged");
+    }
+    outbox.save(events.revised(plan, now));
+    return true;
+  }
+}


