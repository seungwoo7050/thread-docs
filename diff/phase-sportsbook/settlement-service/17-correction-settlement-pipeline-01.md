# 수정 정산 처리 파이프라인

## `feat(event): consume match result events`

diff --git a/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java b/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java
new file mode 100644
index 0000000..6c5afdc
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java
@@ -0,0 +1,76 @@
+package com.sportsbook.settlement.event;
+
+import com.sportsbook.protocol.event.MatchResult;
+import com.sportsbook.settlement.correction.ResultCandidateIntake;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.ResultFanout;
+import java.time.Clock;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.support.Acknowledgment;
+import org.springframework.stereotype.Component;
+
+@Component
+public class MatchResultListener {
+
+  private final ResultCandidateIntake intake;
+  private final AcceptedResultRepository acceptedResults;
+  private final ResultFanout fanout;
+  private final Clock clock;
+  private final StrictAvroDecoder decoder;
+  private final KafkaUuidKeyValidator keys;
+  private final MatchResultMapper mapper;
+
+  @Autowired
+  public MatchResultListener(
+      ResultCandidateIntake intake,
+      AcceptedResultRepository acceptedResults,
+      ResultFanout fanout,
+      Clock clock) {
+    this(
+        intake,
+        acceptedResults,
+        fanout,
+        clock,
+        new StrictAvroDecoder(),
+        new KafkaUuidKeyValidator(),
+        new MatchResultMapper());
+  }
+
+  MatchResultListener(
+      ResultCandidateIntake intake,
+      AcceptedResultRepository acceptedResults,
+      ResultFanout fanout,
+      Clock clock,
+      StrictAvroDecoder decoder,
+      KafkaUuidKeyValidator keys,
+      MatchResultMapper mapper) {
+    this.intake = intake;
+    this.acceptedResults = acceptedResults;
+    this.fanout = fanout;
+    this.clock = clock;
+    this.decoder = decoder;
+    this.keys = keys;
+    this.mapper = mapper;
+  }
+
+  @KafkaListener(
+      topics = "${settlement.topics.match-result:match.result}",
+      groupId = "settlement-service")
+  public void receive(ConsumerRecord<byte[], byte[]> record, Acknowledgment acknowledgment) {
+    MatchResult event = decoder.decode(record.value(), MatchResult.class);
+    var eventId = keys.requireMatching(record.key(), event.getEventId(), "eventId");
+    var result = intake.ingest(mapper.map(event, clock.instant()));
+    if (result == ResultCandidateIntake.IntakeResult.FIRST_ACCEPTED
+        || result == ResultCandidateIntake.IntakeResult.ACCEPTED_REPLAY) {
+      var accepted =
+          acceptedResults
+              .findByEventId(eventId)
+              .orElseThrow(
+                  () -> new IllegalStateException("Accepted result projection is missing"));
+      fanout.fanOut(accepted);
+    }
+    acknowledgment.acknowledge();
+  }
+}


## `feat(correction): add immutable result candidates`

diff --git a/src/main/resources/db/migration/V7__result_candidate.sql b/src/main/resources/db/migration/V7__result_candidate.sql
new file mode 100644
index 0000000..8d6aa8c
--- /dev/null
+++ b/src/main/resources/db/migration/V7__result_candidate.sql
@@ -0,0 +1,44 @@
+-- Every distinct MatchResult is immutable evidence before it can replace the accepted snapshot.
+
+CREATE TABLE result_candidate (
+    candidate_id          UUID                     PRIMARY KEY,
+    candidate_sequence    BIGINT GENERATED ALWAYS AS IDENTITY UNIQUE,
+    event_id              UUID                     NOT NULL,
+    fingerprint           CHAR(64)                 NOT NULL,
+    mode                  VARCHAR(16)              NOT NULL,
+    settled_at            TIMESTAMP WITH TIME ZONE NOT NULL,
+    received_at           TIMESTAMP WITH TIME ZONE NOT NULL,
+    state                 VARCHAR(24)              NOT NULL,
+    replaces_candidate_id UUID REFERENCES result_candidate (candidate_id),
+    decided_at            TIMESTAMP WITH TIME ZONE,
+    decision_reason       VARCHAR(256),
+    CONSTRAINT uq_result_candidate_fingerprint UNIQUE (event_id, fingerprint),
+    CONSTRAINT ck_result_candidate_mode CHECK (
+        mode IN ('COMPLETED', 'ABANDONED', 'VOIDED')),
+    CONSTRAINT ck_result_candidate_state CHECK (
+        state IN ('PENDING', 'ACCEPTED', 'SUPERSEDED', 'REJECTED')),
+    CONSTRAINT ck_result_candidate_decision CHECK (
+        (state = 'PENDING' AND decided_at IS NULL)
+        OR (state <> 'PENDING' AND decided_at IS NOT NULL))
+);
+
+CREATE TABLE result_candidate_selection (
+    candidate_id UUID       NOT NULL REFERENCES result_candidate (candidate_id),
+    selection_id UUID       NOT NULL,
+    outcome      VARCHAR(8) NOT NULL,
+    CONSTRAINT pk_result_candidate_selection PRIMARY KEY (candidate_id, selection_id),
+    CONSTRAINT ck_result_candidate_outcome CHECK (
+        outcome IN ('WON', 'LOST', 'PUSH', 'VOID'))
+);
+
+CREATE INDEX ix_result_candidate_review
+    ON result_candidate (state, received_at, candidate_sequence);
+
+CREATE INDEX ix_result_candidate_event_order
+    ON result_candidate (event_id, candidate_sequence);
+
+ALTER TABLE match_result
+    ADD COLUMN accepted_candidate_id UUID REFERENCES result_candidate (candidate_id);
+
+CREATE UNIQUE INDEX uq_match_result_accepted_candidate
+    ON match_result (accepted_candidate_id) WHERE accepted_candidate_id IS NOT NULL;


## `feat(correction): auto accept replacement results`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 801165d..55effb2 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -17,6 +17,7 @@ public class ResultCandidateIntake {
 
   @Transactional
   public IntakeResult ingest(MatchResultRecord result) {
+    var accepted = store.findAcceptedCandidateId(result.eventId());
     String fingerprint =
         fingerprints.fingerprint(result.eventId(), result.mode(), result.outcomes());
     ResultCandidate candidate =
@@ -27,15 +28,21 @@ public class ResultCandidateIntake {
             result.outcomes(),
             result.settledAt(),
             result.receivedAt(),
-            null);
+            accepted.orElse(null));
     ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
     if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
       return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
           ? IntakeResult.EXACT_REPLAY
           : IntakeResult.NO_CHANGE;
     }
-    return store.acceptFirst(candidate.candidateId(), result.receivedAt())
-        ? IntakeResult.FIRST_ACCEPTED
+    if (accepted.isEmpty()) {
+      return store.acceptFirst(candidate.candidateId(), result.receivedAt())
+          ? IntakeResult.FIRST_ACCEPTED
+          : IntakeResult.CORRECTION_PENDING;
+    }
+    return store.replaceAccepted(
+            candidate.candidateId(), accepted.orElseThrow(), result.receivedAt())
+        ? IntakeResult.AUTO_CORRECTION_ACCEPTED
         : IntakeResult.CORRECTION_PENDING;
   }
 
@@ -43,6 +50,7 @@ public class ResultCandidateIntake {
     EXACT_REPLAY,
     NO_CHANGE,
     FIRST_ACCEPTED,
+    AUTO_CORRECTION_ACCEPTED,
     CORRECTION_PENDING
   }
 }
diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index d9f0cd9..a78e148 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -4,6 +4,7 @@ import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import java.util.Comparator;
 import java.util.List;
+import java.util.Optional;
 import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
@@ -103,6 +104,78 @@ public class ResultCandidateStore {
     return true;
   }
 
+  public Optional<UUID> findAcceptedCandidateId(UUID eventId) {
+    return jdbc
+        .query(
+            """
+            select accepted_candidate_id from match_result
+            where event_id = ? and accepted_candidate_id is not null
+            """,
+            (result, rowNumber) -> result.getObject("accepted_candidate_id", UUID.class),
+            eventId)
+        .stream()
+        .findFirst();
+  }
+
+  @Transactional
+  public boolean replaceAccepted(
+      UUID candidateId, UUID expectedAcceptedId, java.time.Instant decidedAt) {
+    int replaced =
+        jdbc.update(
+            """
+            update match_result m set
+                mode = c.mode, settled_at = c.settled_at, received_at = c.received_at,
+                accepted_candidate_id = c.candidate_id
+            from result_candidate c
+            where c.candidate_id = ? and c.state = 'PENDING'
+              and m.event_id = c.event_id and m.accepted_candidate_id = ?
+            """,
+            candidateId,
+            expectedAcceptedId);
+    if (replaced == 0) {
+      return false;
+    }
+    jdbc.update(
+        """
+        delete from match_selection_result where event_id =
+            (select event_id from result_candidate where candidate_id = ?)
+        """,
+        candidateId);
+    jdbc.update(
+        """
+        insert into match_selection_result (event_id, selection_id, outcome)
+        select c.event_id, s.selection_id, s.outcome
+        from result_candidate c join result_candidate_selection s
+          on s.candidate_id = c.candidate_id where c.candidate_id = ?
+        """,
+        candidateId);
+    int superseded =
+        jdbc.update(
+            """
+            update result_candidate set state = 'SUPERSEDED', decided_at = ?,
+                decision_reason = 'AUTO_CORRECTION'
+            where candidate_id = ? and state = 'ACCEPTED'
+            """,
+            required(decidedAt),
+            expectedAcceptedId);
+    if (superseded != 1) {
+      throw new IllegalStateException("Replacement result lost its accepted candidate");
+    }
+    int accepted =
+        jdbc.update(
+            """
+            update result_candidate set state = 'ACCEPTED', decided_at = ?,
+                decision_reason = 'AUTO_CORRECTION'
+            where candidate_id = ? and state = 'PENDING'
+            """,
+            required(decidedAt),
+            candidateId);
+    if (accepted != 1) {
+      throw new IllegalStateException("Replacement result lost its pending candidate");
+    }
+    return true;
+  }
+
   private RecordOutcome find(UUID eventId, String fingerprint) {
     return jdbc
         .query(


## `feat(admin): approve result candidates`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateApproval.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateApproval.java
new file mode 100644
index 0000000..0821736
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateApproval.java
@@ -0,0 +1,77 @@
+package com.sportsbook.settlement.admin;
+
+import com.sportsbook.settlement.correction.ResultCandidateState;
+import com.sportsbook.settlement.correction.ResultCandidateStore;
+import com.sportsbook.settlement.persistence.DatabaseTimeSource;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class AdminCandidateApproval {
+
+  private final AdminActionRepository actions;
+  private final ResultCandidateStore candidates;
+  private final DatabaseTimeSource databaseTime;
+
+  public AdminCandidateApproval(
+      AdminActionRepository actions,
+      ResultCandidateStore candidates,
+      DatabaseTimeSource databaseTime) {
+    this.actions = actions;
+    this.candidates = candidates;
+    this.databaseTime = databaseTime;
+  }
+
+  @Transactional
+  public Decision decide(UUID idempotencyKey, UUID candidateId) {
+    AdminAction.Kind kind = AdminAction.Kind.CANDIDATE_APPROVE;
+    String fingerprint = AdminRequestFingerprint.create(kind, candidateId, "");
+    var replay =
+        AdminActionReplay.requireExact(
+            actions.lockAndFind(idempotencyKey), kind, candidateId, fingerprint);
+    ResultCandidateStore.AdminCandidate candidate =
+        candidates
+            .lockForAdmin(candidateId)
+            .orElseThrow(() -> AdminControlException.notFound("Result candidate"));
+    if (replay.isPresent()) {
+      return new Decision(replay.orElseThrow(), candidate.eventId(), true);
+    }
+    Instant decidedAt = databaseTime.currentTimestamp();
+    requireEligible(candidate, decidedAt);
+    boolean approved =
+        candidate.acceptedCandidateId() == null
+            ? candidates.approveFirst(candidateId, decidedAt)
+            : candidates.approve(candidateId, decidedAt);
+    if (!approved) {
+      throw AdminControlException.conflict("Result candidate decision changed concurrently");
+    }
+    AdminAction action =
+        actions.append(
+            idempotencyKey,
+            kind,
+            candidateId,
+            fingerprint,
+            AdminAction.Outcome.CANDIDATE_APPROVED,
+            null);
+    return new Decision(action, candidate.eventId(), false);
+  }
+
+  private static void requireEligible(
+      ResultCandidateStore.AdminCandidate candidate, Instant databaseNow) {
+    if (candidate.state() != ResultCandidateState.PENDING) {
+      throw AdminControlException.conflict("Result candidate is already decided");
+    }
+    if (candidate.settledAt().isAfter(databaseNow)) {
+      throw AdminControlException.conflict("Result candidate is not due");
+    }
+    if (candidate.acceptedCandidateId() != null
+        && !Objects.equals(candidate.replacesCandidateId(), candidate.acceptedCandidateId())) {
+      throw AdminControlException.conflict("Result candidate predecessor is stale");
+    }
+  }
+
+  public record Decision(AdminAction action, UUID eventId, boolean replay) {}
+}


## `feat(event): fan out accepted corrections`

diff --git a/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java b/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java
index 6c5afdc..d485d91 100644
--- a/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java
+++ b/src/main/java/com/sportsbook/settlement/event/MatchResultListener.java
@@ -1,6 +1,7 @@
 package com.sportsbook.settlement.event;
 
 import com.sportsbook.protocol.event.MatchResult;
+import com.sportsbook.settlement.correction.CorrectionFanout;
 import com.sportsbook.settlement.correction.ResultCandidateIntake;
 import com.sportsbook.settlement.result.AcceptedResultRepository;
 import com.sportsbook.settlement.result.ResultFanout;
@@ -17,6 +18,7 @@ public class MatchResultListener {
   private final ResultCandidateIntake intake;
   private final AcceptedResultRepository acceptedResults;
   private final ResultFanout fanout;
+  private final CorrectionFanout corrections;
   private final Clock clock;
   private final StrictAvroDecoder decoder;
   private final KafkaUuidKeyValidator keys;
@@ -27,11 +29,13 @@ public class MatchResultListener {
       ResultCandidateIntake intake,
       AcceptedResultRepository acceptedResults,
       ResultFanout fanout,
+      CorrectionFanout corrections,
       Clock clock) {
     this(
         intake,
         acceptedResults,
         fanout,
+        corrections,
         clock,
         new StrictAvroDecoder(),
         new KafkaUuidKeyValidator(),
@@ -42,6 +46,7 @@ public class MatchResultListener {
       ResultCandidateIntake intake,
       AcceptedResultRepository acceptedResults,
       ResultFanout fanout,
+      CorrectionFanout corrections,
       Clock clock,
       StrictAvroDecoder decoder,
       KafkaUuidKeyValidator keys,
@@ -49,6 +54,7 @@ public class MatchResultListener {
     this.intake = intake;
     this.acceptedResults = acceptedResults;
     this.fanout = fanout;
+    this.corrections = corrections;
     this.clock = clock;
     this.decoder = decoder;
     this.keys = keys;
@@ -62,14 +68,19 @@ public class MatchResultListener {
     MatchResult event = decoder.decode(record.value(), MatchResult.class);
     var eventId = keys.requireMatching(record.key(), event.getEventId(), "eventId");
     var result = intake.ingest(mapper.map(event, clock.instant()));
-    if (result == ResultCandidateIntake.IntakeResult.FIRST_ACCEPTED
-        || result == ResultCandidateIntake.IntakeResult.ACCEPTED_REPLAY) {
+    boolean correction =
+        result == ResultCandidateIntake.IntakeResult.AUTO_CORRECTION_ACCEPTED
+            || result == ResultCandidateIntake.IntakeResult.ACCEPTED_REPLAY;
+    if (result == ResultCandidateIntake.IntakeResult.FIRST_ACCEPTED || correction) {
       var accepted =
           acceptedResults
               .findByEventId(eventId)
               .orElseThrow(
                   () -> new IllegalStateException("Accepted result projection is missing"));
       fanout.fanOut(accepted);
+      if (correction) {
+        corrections.fanOut(accepted);
+      }
     }
     acknowledgment.acknowledge();
   }


## `feat(correction): prepare immutable revision claims`

diff --git a/src/main/java/com/sportsbook/settlement/correction/CorrectionRevisionPreparer.java b/src/main/java/com/sportsbook/settlement/correction/CorrectionRevisionPreparer.java
new file mode 100644
index 0000000..8c8978e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/CorrectionRevisionPreparer.java
@@ -0,0 +1,75 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.persistence.BetRepository;
+import com.sportsbook.settlement.persistence.DatabaseTimeSource;
+import com.sportsbook.settlement.result.AcceptedResult;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class CorrectionRevisionPreparer {
+
+  private final BetRepository bets;
+  private final AcceptedResultRepository acceptedResults;
+  private final RevisionPlanRepository revisions;
+  private final SettlementRuntimeProperties runtime;
+  private final DatabaseTimeSource databaseTime;
+  private final ReplacementSnapshotProjector projector = new ReplacementSnapshotProjector();
+  private final RevisionResolver resolver = new RevisionResolver();
+
+  public CorrectionRevisionPreparer(
+      BetRepository bets,
+      AcceptedResultRepository acceptedResults,
+      RevisionPlanRepository revisions,
+      SettlementRuntimeProperties runtime,
+      DatabaseTimeSource databaseTime) {
+    this.bets = bets;
+    this.acceptedResults = acceptedResults;
+    this.revisions = revisions;
+    this.runtime = runtime;
+    this.databaseTime = databaseTime;
+  }
+
+  @Transactional
+  public Optional<PreparedRevision> prepare(UUID betId, AcceptedResult expected) {
+    Bet bet = bets.findForUpdateById(betId).orElseThrow();
+    if (bet.status() != SettlementStatus.SETTLED || !isStale(bet, expected)) {
+      return Optional.empty();
+    }
+    AcceptedResult current =
+        acceptedResults
+            .findByEventId(expected.eventId())
+            .filter(result -> result.candidateId().equals(expected.candidateId()))
+            .orElse(null);
+    long nextRevision = Math.incrementExact(bet.revisionNumber());
+    if (current == null || revisions.exists(betId, nextRevision)) {
+      return Optional.empty();
+    }
+    var target = projector.project(bet, current);
+    if (target.isEmpty()) {
+      return Optional.empty();
+    }
+    var resolution = resolver.resolve(target.orElseThrow());
+    var requested =
+        RevisionPlan.allocate(target.orElseThrow(), resolution, databaseTime.currentTimestamp());
+    var persisted = revisions.persist(requested, runtime.leaseDuration());
+    if (!persisted.created()) {
+      return Optional.empty();
+    }
+    return Optional.of(new PreparedRevision(persisted.durablePlan(requested), persisted.lease()));
+  }
+
+  private static boolean isStale(Bet bet, AcceptedResult accepted) {
+    return bet.selections().stream()
+        .filter(selection -> selection.eventId().equals(accepted.eventId()))
+        .anyMatch(selection -> !accepted.candidateId().equals(selection.sourceCandidateId()));
+  }
+
+  public record PreparedRevision(RevisionPlan plan, RevisionLease lease) {}
+}


