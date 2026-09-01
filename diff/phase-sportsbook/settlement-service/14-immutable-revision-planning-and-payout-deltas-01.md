# 불변 수정 계획과 지급 차액

## `build(flyway): add V8 source revisions`

diff --git a/src/main/resources/db/migration/V8__source_revision.sql b/src/main/resources/db/migration/V8__source_revision.sql
new file mode 100644
index 0000000..6ccb11c
--- /dev/null
+++ b/src/main/resources/db/migration/V8__source_revision.sql
@@ -0,0 +1,11 @@
+-- Track the accepted result evidence used by each terminal selection.
+
+ALTER TABLE bet
+    ADD COLUMN revision_number BIGINT NOT NULL DEFAULT 0,
+    ADD CONSTRAINT ck_bet_revision_nonnegative CHECK (revision_number >= 0);
+
+ALTER TABLE bet_selection
+    ADD COLUMN source_candidate_id UUID REFERENCES result_candidate (candidate_id);
+
+CREATE INDEX ix_bet_selection_stale_source
+    ON bet_selection (event_id, source_candidate_id, bet_id);


## `feat(correction): retain selection source candidates`

diff --git a/src/main/java/com/sportsbook/settlement/domain/BetSelection.java b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
index 865f3ec..1f1eceb 100644
--- a/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
+++ b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
@@ -48,6 +48,9 @@ public class BetSelection {
   @Column(name = "outcome", length = 8)
   private SettlementResult outcome;
 
+  @Column(name = "source_candidate_id")
+  private UUID sourceCandidateId;
+
   protected BetSelection() {}
 
   public BetSelection(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
@@ -71,6 +74,15 @@ public class BetSelection {
     return true;
   }
 
+  public boolean applyCandidate(UUID candidateId, SettlementResult replacement) {
+    boolean changed = replaceOutcome(replacement);
+    if (!Objects.equals(sourceCandidateId, candidateId)) {
+      sourceCandidateId = Objects.requireNonNull(candidateId, "candidateId");
+      changed = true;
+    }
+    return changed;
+  }
+
   public UUID selectionRowId() {
     return selectionRowId;
   }
@@ -98,4 +110,8 @@ public class BetSelection {
   public SettlementResult outcome() {
     return outcome;
   }
+
+  public UUID sourceCandidateId() {
+    return sourceCandidateId;
+  }
 }


## `build(flyway): add V9 revision plans`

diff --git a/src/main/resources/db/migration/V9__settlement_revision.sql b/src/main/resources/db/migration/V9__settlement_revision.sql
new file mode 100644
index 0000000..25e66c7
--- /dev/null
+++ b/src/main/resources/db/migration/V9__settlement_revision.sql
@@ -0,0 +1,128 @@
+-- Persist each correction plan before contacting Wallet.
+
+CREATE TABLE settlement_revision (
+    revision_id             UUID                     PRIMARY KEY,
+    bet_id                  UUID                     NOT NULL REFERENCES bet (bet_id),
+    revision_number         BIGINT                   NOT NULL,
+    user_id                 UUID                     NOT NULL,
+    event_id                UUID                     NOT NULL,
+    source_candidate_id     UUID                     NOT NULL REFERENCES result_candidate (candidate_id),
+    previous_result         VARCHAR(8)               NOT NULL,
+    new_result              VARCHAR(8)               NOT NULL,
+    previous_payout_amount  BIGINT                   NOT NULL,
+    new_payout_amount       BIGINT                   NOT NULL,
+    currency                VARCHAR(3)               NOT NULL,
+    slip_type               VARCHAR(16)              NOT NULL,
+    system_min_wins         INT,
+    system_total_selections INT,
+    unit_stake_amount       BIGINT                   NOT NULL,
+    source_result_settled_at TIMESTAMP WITH TIME ZONE NOT NULL,
+    state                   VARCHAR(16)              NOT NULL,
+    lease_token             UUID,
+    lease_until             TIMESTAMP WITH TIME ZONE,
+    attempt_count           INT                      NOT NULL DEFAULT 0,
+    next_retry_at           TIMESTAMP WITH TIME ZONE,
+    last_error_code         VARCHAR(128),
+    wallet_status           VARCHAR(16),
+    wallet_queue_sequence   BIGINT,
+    wallet_operation_group_id UUID,
+    wallet_queued_at        TIMESTAMP WITH TIME ZONE,
+    wallet_applied_at       TIMESTAMP WITH TIME ZONE,
+    wallet_next_attempt_at  TIMESTAMP WITH TIME ZONE,
+    created_at              TIMESTAMP WITH TIME ZONE NOT NULL,
+    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL,
+    applied_at              TIMESTAMP WITH TIME ZONE,
+    CONSTRAINT uq_settlement_revision_number UNIQUE (bet_id, revision_number),
+    CONSTRAINT uq_settlement_revision_source UNIQUE (bet_id, source_candidate_id),
+    CONSTRAINT ck_settlement_revision_number CHECK (revision_number >= 1),
+    CONSTRAINT ck_settlement_revision_result CHECK (
+        previous_result IN ('WON', 'LOST', 'PUSH', 'VOID')
+        AND new_result IN ('WON', 'LOST', 'PUSH', 'VOID')),
+    CONSTRAINT ck_settlement_revision_payout CHECK (
+        previous_payout_amount >= 0 AND new_payout_amount >= 0),
+    CONSTRAINT ck_settlement_revision_stake CHECK (unit_stake_amount > 0),
+    CONSTRAINT ck_settlement_revision_slip CHECK (
+        (slip_type IN ('SINGLE', 'MULTIPLE')
+            AND system_min_wins IS NULL AND system_total_selections IS NULL)
+        OR (slip_type = 'SYSTEM' AND system_min_wins BETWEEN 1 AND system_total_selections
+            AND system_total_selections BETWEEN 1 AND 15)),
+    CONSTRAINT ck_settlement_revision_state CHECK (
+        state IN ('PENDING', 'BLOCKED', 'EXHAUSTED', 'APPLIED', 'REJECTED')),
+    CONSTRAINT ck_settlement_revision_lease CHECK (
+        (lease_token IS NULL AND lease_until IS NULL)
+        OR (lease_token IS NOT NULL AND lease_until IS NOT NULL
+            AND state IN ('PENDING', 'BLOCKED'))),
+    CONSTRAINT ck_settlement_revision_applied CHECK (
+        (state = 'APPLIED' AND applied_at IS NOT NULL)
+        OR (state <> 'APPLIED' AND applied_at IS NULL)),
+    CONSTRAINT ck_settlement_revision_blocked_proof CHECK (
+        (wallet_status IS NOT NULL AND wallet_status = 'BLOCKED'
+            AND state IN ('PENDING', 'BLOCKED')
+            AND wallet_queue_sequence IS NOT NULL AND wallet_queued_at IS NOT NULL
+            AND wallet_operation_group_id IS NULL AND wallet_applied_at IS NULL
+            AND wallet_next_attempt_at IS NOT NULL)
+        OR ((wallet_status IS NULL
+                OR (wallet_status IS NOT NULL AND wallet_status <> 'BLOCKED'))
+            AND wallet_next_attempt_at IS NULL)),
+    CONSTRAINT ck_settlement_revision_wallet_state CHECK (
+        wallet_status IS NULL
+        OR (wallet_status IS NOT NULL AND wallet_status = 'BLOCKED'
+            AND state IN ('PENDING', 'BLOCKED'))
+        OR (wallet_status IS NOT NULL AND wallet_status = 'APPLIED' AND state = 'APPLIED')
+        OR (wallet_status IS NOT NULL AND wallet_status = 'REJECTED' AND state = 'REJECTED')),
+    CONSTRAINT ck_settlement_revision_applied_proof CHECK (
+        (wallet_status IS NOT NULL AND wallet_status = 'APPLIED'
+            AND wallet_operation_group_id IS NOT NULL AND wallet_applied_at IS NOT NULL
+            AND wallet_next_attempt_at IS NULL)
+        OR ((wallet_status IS NULL
+                OR (wallet_status IS NOT NULL AND wallet_status <> 'APPLIED'))
+            AND wallet_operation_group_id IS NULL AND wallet_applied_at IS NULL)),
+    CONSTRAINT ck_settlement_revision_rejected_proof CHECK (
+        (wallet_status IS NOT NULL AND wallet_status = 'REJECTED'
+            AND wallet_queue_sequence IS NULL AND wallet_operation_group_id IS NULL
+            AND wallet_queued_at IS NULL AND wallet_applied_at IS NULL
+            AND wallet_next_attempt_at IS NULL)
+        OR (wallet_status IS NULL
+            OR (wallet_status IS NOT NULL AND wallet_status <> 'REJECTED'))),
+    CONSTRAINT ck_settlement_revision_queue_identity CHECK (
+        (wallet_queue_sequence IS NULL AND wallet_queued_at IS NULL)
+        OR (wallet_queue_sequence IS NOT NULL AND wallet_queued_at IS NOT NULL
+            AND wallet_queue_sequence > 0 AND new_payout_amount < previous_payout_amount)),
+    CONSTRAINT ck_settlement_revision_attempts CHECK (attempt_count BETWEEN 0 AND 12),
+    CONSTRAINT ck_settlement_revision_schedule CHECK (
+        (lease_token IS NOT NULL AND next_retry_at IS NULL)
+        OR (lease_token IS NULL AND state IN ('PENDING', 'BLOCKED')
+            AND attempt_count < 12 AND next_retry_at IS NOT NULL)
+        OR (lease_token IS NULL AND state = 'BLOCKED' AND next_retry_at IS NULL
+            AND wallet_status IS NOT NULL AND wallet_status = 'BLOCKED'
+            AND last_error_code IS NOT NULL)
+        OR (lease_token IS NULL AND state IN ('EXHAUSTED', 'APPLIED', 'REJECTED')
+            AND next_retry_at IS NULL)),
+    CONSTRAINT ck_settlement_revision_exhausted CHECK (
+        state <> 'EXHAUSTED'
+        OR (last_error_code IS NOT NULL AND wallet_status IS NULL
+            AND wallet_queue_sequence IS NULL AND wallet_operation_group_id IS NULL
+            AND wallet_queued_at IS NULL AND wallet_applied_at IS NULL
+            AND wallet_next_attempt_at IS NULL)),
+    CONSTRAINT ck_settlement_revision_rejected_state CHECK (
+        state <> 'REJECTED'
+        OR (wallet_status IS NOT NULL AND wallet_status = 'REJECTED')
+        OR (wallet_status IS NULL AND last_error_code IS NOT NULL))
+);
+
+CREATE INDEX ix_settlement_revision_recovery
+    ON settlement_revision (state, next_retry_at, wallet_next_attempt_at, revision_id);
+
+CREATE TABLE settlement_revision_selection (
+    revision_id UUID       NOT NULL REFERENCES settlement_revision (revision_id) ON DELETE CASCADE,
+    selection_id UUID      NOT NULL,
+    leg_index    INT       NOT NULL,
+    odds         NUMERIC(9,4) NOT NULL,
+    outcome      VARCHAR(8) NOT NULL,
+    CONSTRAINT pk_settlement_revision_selection PRIMARY KEY (revision_id, selection_id),
+    CONSTRAINT uq_settlement_revision_selection_order UNIQUE (revision_id, leg_index),
+    CONSTRAINT ck_settlement_revision_selection_index CHECK (leg_index >= 0),
+    CONSTRAINT ck_settlement_revision_selection_odds CHECK (odds >= 1),
+    CONSTRAINT ck_settlement_revision_selection_outcome CHECK (
+        outcome IN ('WON', 'LOST', 'PUSH', 'VOID'))
+);


## `feat(correction): define revision lifecycle`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionState.java b/src/main/java/com/sportsbook/settlement/correction/RevisionState.java
new file mode 100644
index 0000000..84a5085
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionState.java
@@ -0,0 +1,29 @@
+package com.sportsbook.settlement.correction;
+
+/** Durable lifecycle for one immutable correction plan. */
+public enum RevisionState {
+  PENDING,
+  BLOCKED,
+  EXHAUSTED,
+  APPLIED,
+  REJECTED;
+
+  public boolean canTransitionTo(RevisionState target) {
+    if (this == target) {
+      return true;
+    }
+    return switch (this) {
+      case PENDING ->
+          target == BLOCKED || target == EXHAUSTED || target == APPLIED || target == REJECTED;
+      case BLOCKED -> target == PENDING || target == APPLIED || target == REJECTED;
+      case EXHAUSTED, REJECTED, APPLIED -> false;
+    };
+  }
+
+  public void requireTransitionTo(RevisionState target) {
+    if (!canTransitionTo(target)) {
+      throw new IllegalStateException(
+          "Invalid revision state transition: " + this + " -> " + target);
+    }
+  }
+}


## `feat(correction): capture revision targets`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionTarget.java b/src/main/java/com/sportsbook/settlement/correction/RevisionTarget.java
new file mode 100644
index 0000000..c74de41
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionTarget.java
@@ -0,0 +1,44 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.resolver.ResolvedSelection;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Immutable bet and accepted-result snapshot used to calculate one correction. */
+public record RevisionTarget(
+    UUID betId,
+    long revisionNumber,
+    UUID userId,
+    UUID eventId,
+    UUID sourceCandidateId,
+    SettlementResult previousResult,
+    Money previousPayout,
+    BetSlipType slipType,
+    Money unitStake,
+    List<ResolvedSelection> selections,
+    Instant sourceResultSettledAt) {
+
+  public RevisionTarget {
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(sourceCandidateId, "sourceCandidateId");
+    Objects.requireNonNull(previousResult, "previousResult");
+    Objects.requireNonNull(previousPayout, "previousPayout");
+    Objects.requireNonNull(slipType, "slipType");
+    Objects.requireNonNull(unitStake, "unitStake");
+    selections = List.copyOf(Objects.requireNonNull(selections, "selections"));
+    Objects.requireNonNull(sourceResultSettledAt, "sourceResultSettledAt");
+    if (revisionNumber < 1 || selections.isEmpty()) {
+      throw new IllegalArgumentException("Revision target must have a sequence and selections");
+    }
+    if (previousPayout.currency() != unitStake.currency()) {
+      throw new IllegalArgumentException("Revision target money currencies must match");
+    }
+  }
+}


## `feat(correction): project replacement snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ReplacementSnapshotProjector.java b/src/main/java/com/sportsbook/settlement/correction/ReplacementSnapshotProjector.java
new file mode 100644
index 0000000..74eaf92
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ReplacementSnapshotProjector.java
@@ -0,0 +1,50 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.resolver.ResolvedSelection;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Optional;
+
+public final class ReplacementSnapshotProjector {
+
+  public Optional<RevisionTarget> project(Bet bet, ResultCandidate source) {
+    if (bet.status() != SettlementStatus.SETTLED
+        || source.state() != ResultCandidateState.ACCEPTED) {
+      throw new IllegalArgumentException("Correction requires settled bet and accepted result");
+    }
+    List<ResolvedSelection> resolved = new ArrayList<>(bet.selections().size());
+    boolean sourceEventPresent = false;
+    for (var selection : bet.selections()) {
+      SettlementResult outcome = selection.outcome();
+      if (selection.eventId().equals(source.eventId())) {
+        sourceEventPresent = true;
+        Optional<SettlementResult> replacement =
+            source.mode().resolve(source.outcomes().get(selection.selectionId()));
+        if (replacement.isEmpty()) {
+          return Optional.empty();
+        }
+        outcome = replacement.orElseThrow();
+      }
+      resolved.add(new ResolvedSelection(selection.selectionId(), selection.odds(), outcome));
+    }
+    if (!sourceEventPresent) {
+      throw new IllegalArgumentException("Accepted result is unrelated to the bet");
+    }
+    return Optional.of(
+        new RevisionTarget(
+            bet.betId(),
+            Math.incrementExact(bet.revisionNumber()),
+            bet.userId(),
+            source.eventId(),
+            source.candidateId(),
+            bet.result(),
+            bet.payout(),
+            bet.slipType(),
+            bet.stake(),
+            resolved,
+            source.settledAt()));
+  }
+}


## `feat(correction): resolve revision payouts`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionResolver.java b/src/main/java/com/sportsbook/settlement/correction/RevisionResolver.java
new file mode 100644
index 0000000..612b79c
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionResolver.java
@@ -0,0 +1,14 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.resolver.SettlementOutcome;
+import com.sportsbook.settlement.resolver.SettlementResolver;
+
+/** Reuses base payout rules against an immutable replacement snapshot. */
+public final class RevisionResolver {
+
+  private final SettlementResolver settlements = new SettlementResolver();
+
+  public SettlementOutcome resolve(RevisionTarget target) {
+    return settlements.resolve(target.slipType(), target.selections(), target.unitStake());
+  }
+}


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


## `feat(correction): capture immutable revision snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionSnapshot.java b/src/main/java/com/sportsbook/settlement/correction/RevisionSnapshot.java
new file mode 100644
index 0000000..2c9b306
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionSnapshot.java
@@ -0,0 +1,43 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import java.math.BigDecimal;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+
+record RevisionSnapshot(
+    String slipType,
+    Integer systemMinWins,
+    Integer systemTotalSelections,
+    long unitStakeAmount,
+    List<Selection> selections) {
+
+  RevisionSnapshot {
+    selections = List.copyOf(selections);
+  }
+
+  static RevisionSnapshot capture(RevisionTarget target) {
+    BetSlipType slip = target.slipType();
+    String type = slip instanceof BetSlipType.Single ? "SINGLE" : "MULTIPLE";
+    Integer minimumWins = null;
+    Integer totalSelections = null;
+    if (slip instanceof BetSlipType.System system) {
+      type = "SYSTEM";
+      minimumWins = system.minWins();
+      totalSelections = system.totalSelections();
+    }
+    List<Selection> selections = new ArrayList<>();
+    for (int index = 0; index < target.selections().size(); index++) {
+      var selection = target.selections().get(index);
+      selections.add(
+          new Selection(
+              selection.selectionId(), index, selection.odds().decimal(), selection.outcome()));
+    }
+    return new RevisionSnapshot(
+        type, minimumWins, totalSelections, target.unitStake().amount(), selections);
+  }
+
+  record Selection(UUID selectionId, int legIndex, BigDecimal odds, SettlementResult outcome) {}
+}


