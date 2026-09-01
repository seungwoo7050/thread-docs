## `feat(correction): allocate immutable revision plans`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlan.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlan.java
new file mode 100644
index 0000000..96d983f
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlan.java
@@ -0,0 +1,44 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.infrastructure.id.UuidV7;
+import com.sportsbook.settlement.resolver.SettlementOutcome;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Complete immutable correction plan allocated before any Wallet request. */
+public record RevisionPlan(
+    UUID revisionId,
+    RevisionTarget target,
+    SettlementResult newResult,
+    Money newPayout,
+    Instant createdAt) {
+
+  public RevisionPlan {
+    Objects.requireNonNull(revisionId, "revisionId");
+    Objects.requireNonNull(target, "target");
+    Objects.requireNonNull(newResult, "newResult");
+    Objects.requireNonNull(newPayout, "newPayout");
+    Objects.requireNonNull(createdAt, "createdAt");
+    if (newPayout.isNegative() || newPayout.currency() != target.previousPayout().currency()) {
+      throw new IllegalArgumentException("Revision payout must preserve currency and be nonnegative");
+    }
+  }
+
+  public static RevisionPlan allocate(
+      RevisionTarget target, SettlementOutcome resolution, Instant now) {
+    Objects.requireNonNull(resolution, "resolution");
+    return new RevisionPlan(
+        UuidV7.generate(), target, resolution.result(), resolution.payout(), now);
+  }
+
+  public long deltaAmount() {
+    return Math.subtractExact(newPayout.amount(), target.previousPayout().amount());
+  }
+
+  public boolean hasZeroDelta() {
+    return deltaAmount() == 0;
+  }
+}


## `feat(correction): persist plans before wallet`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
new file mode 100644
index 0000000..a5680ba
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -0,0 +1,97 @@
+package com.sportsbook.settlement.correction;
+
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
+
+import java.sql.Timestamp;
+import java.time.Duration;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class RevisionPlanRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public RevisionPlanRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional
+  public Persisted persist(RevisionPlan plan, Duration leaseDuration) {
+    long leaseMillis = leaseDuration == null ? 0 : leaseDuration.toMillis();
+    if (leaseMillis < 1) {
+      throw new IllegalArgumentException("Revision lease duration must be positive");
+    }
+    RevisionTarget target = plan.target();
+    RevisionSnapshot snapshot = RevisionSnapshot.capture(target);
+    UUID leaseToken = UUID.randomUUID();
+    List<Timestamp> claimed =
+        jdbc.query(
+            """
+            insert into settlement_revision (
+                revision_id, bet_id, revision_number, user_id, event_id, source_candidate_id,
+                previous_result, new_result, previous_payout_amount, new_payout_amount, currency,
+                slip_type, system_min_wins, system_total_selections, unit_stake_amount,
+                source_result_settled_at, state, lease_token, lease_until, attempt_count,
+                next_retry_at, created_at, updated_at)
+            select ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'PENDING', ?,
+                current_timestamp + (? * interval '1 millisecond'), 1, null, ?, ?
+            from bet where bet_id = ? and status = 'SETTLED' and revision_number = ?
+            on conflict do nothing
+            returning lease_until
+            """,
+            (result, rowNumber) -> result.getTimestamp("lease_until"),
+            plan.revisionId(),
+            target.betId(),
+            target.revisionNumber(),
+            target.userId(),
+            target.eventId(),
+            target.sourceCandidateId(),
+            target.previousResult().name(),
+            plan.newResult().name(),
+            target.previousPayout().amount(),
+            plan.newPayout().amount(),
+            plan.newPayout().currency().name(),
+            snapshot.slipType(),
+            snapshot.systemMinWins(),
+            snapshot.systemTotalSelections(),
+            snapshot.unitStakeAmount(),
+            required(target.sourceResultSettledAt()),
+            leaseToken,
+            leaseMillis,
+            required(plan.createdAt()),
+            required(plan.createdAt()),
+            target.betId(),
+            target.revisionNumber() - 1);
+    if (!claimed.isEmpty()) {
+      jdbc.batchUpdate(
+          """
+          insert into settlement_revision_selection (
+              revision_id, selection_id, leg_index, odds, outcome)
+          values (?, ?, ?, ?, ?)
+          """,
+          RevisionSelectionRows.from(plan.revisionId(), snapshot));
+      return new Persisted(
+          plan.revisionId(), true, new RevisionLease(leaseToken, claimed.get(0).toInstant()));
+    }
+    UUID existing =
+        jdbc
+            .query(
+                """
+                select revision_id from settlement_revision
+                where bet_id = ? and source_candidate_id = ?
+                """,
+                (result, rowNumber) -> result.getObject("revision_id", UUID.class),
+                target.betId(),
+                target.sourceCandidateId())
+            .stream()
+            .findFirst()
+            .orElseThrow(() -> new IllegalStateException("Concurrent revision allocation lost"));
+    return new Persisted(existing, false, null);
+  }
+
+  public record Persisted(UUID revisionId, boolean created, RevisionLease lease) {}
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


## `feat(outbox): create revision events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
index fc31957..156c876 100644
--- a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
+++ b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
@@ -1,13 +1,16 @@
 package com.sportsbook.settlement.outbox;
 
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
 import com.sportsbook.protocol.event.VoidReason;
 import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.correction.RevisionPlan;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.BetSelection;
 import com.sportsbook.settlement.domain.SettlementStatus;
+import java.time.Instant;
 import java.util.LinkedHashMap;
 import java.util.Map;
 import java.util.UUID;
@@ -67,6 +70,30 @@ public final class SettlementEventFactory {
         bet.settledAt());
   }
 
+  public OutboxEvent revised(RevisionPlan plan, Instant revisedAt) {
+    var target = plan.target();
+    BetResolutionRevised event =
+        BetResolutionRevised.newBuilder()
+            .setRevisionId(plan.revisionId().toString())
+            .setRevisionNumber(target.revisionNumber())
+            .setBetId(target.betId().toString())
+            .setUserId(target.userId().toString())
+            .setEventId(target.eventId().toString())
+            .setPreviousResult(SettlementResultAvro.valueOf(target.previousResult().name()))
+            .setNewResult(SettlementResultAvro.valueOf(plan.newResult().name()))
+            .setPreviousPayout(money(target.previousPayout()))
+            .setNewPayout(money(plan.newPayout()))
+            .setSourceResultSettledAt(target.sourceResultSettledAt())
+            .setRevisedAt(revisedAt)
+            .build();
+    return OutboxEvent.pending(
+        topics.betResolutionRevised(),
+        target.betId().toString(),
+        BetResolutionRevised.class.getSimpleName(),
+        encoder.encode(event),
+        revisedAt);
+  }
+
   private static com.sportsbook.protocol.event.Money money(
       com.sportsbook.protocol.value.Money value) {
     return com.sportsbook.protocol.event.Money.newBuilder()


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
