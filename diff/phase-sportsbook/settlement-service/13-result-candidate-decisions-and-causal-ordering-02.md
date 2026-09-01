## `feat(correction): hold late result candidates`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 55effb2..5f3261f 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -1,6 +1,8 @@
 package com.sportsbook.settlement.correction;
 
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
 import com.sportsbook.settlement.result.MatchResultRecord;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.stereotype.Service;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -9,15 +11,22 @@ public class ResultCandidateIntake {
 
   private final ResultCandidateStore store;
   private final ResultCandidateFingerprinter fingerprints;
+  private final SettlementRuntimeProperties runtime;
 
   public ResultCandidateIntake(ResultCandidateStore store) {
+    this(store, new SettlementRuntimeProperties(null, null, null, 0));
+  }
+
+  @Autowired
+  public ResultCandidateIntake(ResultCandidateStore store, SettlementRuntimeProperties runtime) {
     this.store = store;
     this.fingerprints = new ResultCandidateFingerprinter();
+    this.runtime = runtime;
   }
 
   @Transactional
   public IntakeResult ingest(MatchResultRecord result) {
-    var accepted = store.findAcceptedCandidateId(result.eventId());
+    var accepted = store.findAcceptedCandidate(result.eventId());
     String fingerprint =
         fingerprints.fingerprint(result.eventId(), result.mode(), result.outcomes());
     ResultCandidate candidate =
@@ -28,7 +37,7 @@ public class ResultCandidateIntake {
             result.outcomes(),
             result.settledAt(),
             result.receivedAt(),
-            accepted.orElse(null));
+            accepted.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
     ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
     if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
       return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
@@ -40,8 +49,12 @@ public class ResultCandidateIntake {
           ? IntakeResult.FIRST_ACCEPTED
           : IntakeResult.CORRECTION_PENDING;
     }
+    ResultCandidateStore.AcceptedCandidate current = accepted.orElseThrow();
+    if (result.receivedAt().isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
+      return IntakeResult.LATE_HELD;
+    }
     return store.replaceAccepted(
-            candidate.candidateId(), accepted.orElseThrow(), result.receivedAt())
+            candidate.candidateId(), current.candidateId(), result.receivedAt())
         ? IntakeResult.AUTO_CORRECTION_ACCEPTED
         : IntakeResult.CORRECTION_PENDING;
   }
@@ -51,6 +64,7 @@ public class ResultCandidateIntake {
     NO_CHANGE,
     FIRST_ACCEPTED,
     AUTO_CORRECTION_ACCEPTED,
+    LATE_HELD,
     CORRECTION_PENDING
   }
 }
diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index a78e148..43f9b66 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -117,6 +117,24 @@ public class ResultCandidateStore {
         .findFirst();
   }
 
+  public Optional<AcceptedCandidate> findAcceptedCandidate(UUID eventId) {
+    return jdbc
+        .query(
+            """
+            select m.accepted_candidate_id, c.received_at
+            from match_result m join result_candidate c
+              on c.candidate_id = m.accepted_candidate_id
+            where m.event_id = ? and m.accepted_candidate_id is not null
+            """,
+            (result, rowNumber) ->
+                new AcceptedCandidate(
+                    result.getObject("accepted_candidate_id", UUID.class),
+                    result.getTimestamp("received_at").toInstant()),
+            eventId)
+        .stream()
+        .findFirst();
+  }
+
   @Transactional
   public boolean replaceAccepted(
       UUID candidateId, UUID expectedAcceptedId, java.time.Instant decidedAt) {
@@ -205,4 +223,6 @@ public class ResultCandidateStore {
   }
 
   public record RecordOutcome(RecordKind kind, UUID candidateId, ResultCandidateState state) {}
+
+  public record AcceptedCandidate(UUID candidateId, java.time.Instant receivedAt) {}
 }


## `feat(correction): supersede stale candidates`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 5f3261f..1c65ddf 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -45,17 +45,23 @@ public class ResultCandidateIntake {
           : IntakeResult.NO_CHANGE;
     }
     if (accepted.isEmpty()) {
-      return store.acceptFirst(candidate.candidateId(), result.receivedAt())
-          ? IntakeResult.FIRST_ACCEPTED
+      if (store.acceptFirst(candidate.candidateId(), result.receivedAt())) {
+        return IntakeResult.FIRST_ACCEPTED;
+      }
+      return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+          ? IntakeResult.CORRECTION_SUPERSEDED
           : IntakeResult.CORRECTION_PENDING;
     }
     ResultCandidateStore.AcceptedCandidate current = accepted.orElseThrow();
     if (result.receivedAt().isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
       return IntakeResult.LATE_HELD;
     }
-    return store.replaceAccepted(
-            candidate.candidateId(), current.candidateId(), result.receivedAt())
-        ? IntakeResult.AUTO_CORRECTION_ACCEPTED
+    if (store.replaceAccepted(
+        candidate.candidateId(), current.candidateId(), result.receivedAt())) {
+      return IntakeResult.AUTO_CORRECTION_ACCEPTED;
+    }
+    return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+        ? IntakeResult.CORRECTION_SUPERSEDED
         : IntakeResult.CORRECTION_PENDING;
   }
 
@@ -65,6 +71,7 @@ public class ResultCandidateIntake {
     FIRST_ACCEPTED,
     AUTO_CORRECTION_ACCEPTED,
     LATE_HELD,
+    CORRECTION_SUPERSEDED,
     CORRECTION_PENDING
   }
 }
diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index 43f9b66..66ef62e 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -194,6 +194,18 @@ public class ResultCandidateStore {
     return true;
   }
 
+  public boolean supersedeStale(UUID candidateId, java.time.Instant decidedAt) {
+    return jdbc.update(
+            """
+            update result_candidate set state = 'SUPERSEDED', decided_at = ?,
+                decision_reason = 'STALE_BASE'
+            where candidate_id = ? and state = 'PENDING'
+            """,
+            required(decidedAt),
+            candidateId)
+        == 1;
+  }
+
   private RecordOutcome find(UUID eventId, String fingerprint) {
     return jdbc
         .query(


## `feat(correction): enforce candidate causal order`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index 1a037bb..861cf1b 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -173,9 +173,13 @@ public class ResultCandidateStore {
                 accepted_candidate_id = c.candidate_id
             from result_candidate c
             where c.candidate_id = ? and c.state = 'PENDING'
+              and c.replaces_candidate_id = ?
+              and c.candidate_sequence > (select candidate_sequence from result_candidate
+                  where candidate_id = c.replaces_candidate_id)
               and m.event_id = c.event_id and m.accepted_candidate_id = ?
             """,
             candidateId,
+            expectedAcceptedId,
             expectedAcceptedId);
     if (replaced == 0) {
       return false;


## `fix(correction): fence future result decisions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index 065a4fb..820c9f1 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -94,6 +94,7 @@ public class ResultCandidateStore {
                 event_id, mode, settled_at, received_at, accepted_candidate_id)
             select event_id, mode, settled_at, received_at, candidate_id
             from result_candidate where candidate_id = ? and state = 'PENDING'
+                and settled_at <= current_timestamp
             on conflict (event_id) do nothing
             """,
             candidateId);
@@ -193,6 +194,7 @@ public class ResultCandidateStore {
                 accepted_candidate_id = c.candidate_id
             from result_candidate c
             where c.candidate_id = ? and c.state = 'PENDING'
+              and c.settled_at <= current_timestamp
               and c.replaces_candidate_id = ?
               and c.candidate_sequence > (select candidate_sequence from result_candidate
                   where candidate_id = c.replaces_candidate_id)


## `feat(correction): gate candidate decisions by database time`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 3ddb556..458b2c7 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -26,7 +26,7 @@ public class ResultCandidateIntake {
 
   @Transactional
   public IntakeResult ingest(MatchResultRecord result) {
-    var accepted = store.findAcceptedCandidate(result.eventId());
+    var acceptedAtRecord = store.findAcceptedCandidate(result.eventId());
     String fingerprint =
         fingerprints.fingerprint(result.eventId(), result.mode(), result.outcomes());
     ResultCandidate candidate =
@@ -37,33 +37,36 @@ public class ResultCandidateIntake {
             result.outcomes(),
             result.settledAt(),
             result.receivedAt(),
-            accepted.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
+            acceptedAtRecord.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
     ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
-    if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
-      if (recorded.state() == ResultCandidateState.ACCEPTED) {
-        return IntakeResult.ACCEPTED_REPLAY;
-      }
-      return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
-          ? IntakeResult.EXACT_REPLAY
-          : IntakeResult.NO_CHANGE;
+    if (recorded.state() == ResultCandidateState.ACCEPTED) {
+      return IntakeResult.ACCEPTED_REPLAY;
+    }
+    if (recorded.state() != ResultCandidateState.PENDING) {
+      return IntakeResult.NO_CHANGE;
     }
+    if (store.holdWhileFuture(recorded.candidateId())) {
+      return IntakeResult.FUTURE_HELD;
+    }
+    var accepted = store.findAcceptedCandidate(result.eventId());
+    var candidateReceivedAt =
+        recorded.receivedAt() == null ? result.receivedAt() : recorded.receivedAt();
     if (accepted.isEmpty()) {
-      if (store.acceptFirst(candidate.candidateId(), result.receivedAt())) {
+      if (store.acceptFirst(recorded.candidateId(), result.receivedAt())) {
         return IntakeResult.FIRST_ACCEPTED;
       }
-      return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+      return store.supersedeStale(recorded.candidateId(), result.receivedAt())
           ? IntakeResult.CORRECTION_SUPERSEDED
           : IntakeResult.CORRECTION_PENDING;
     }
     ResultCandidateStore.AcceptedCandidate current = accepted.orElseThrow();
-    if (result.receivedAt().isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
+    if (candidateReceivedAt.isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
       return IntakeResult.LATE_HELD;
     }
-    if (store.replaceAccepted(
-        candidate.candidateId(), current.candidateId(), result.receivedAt())) {
+    if (store.replaceAccepted(recorded.candidateId(), current.candidateId(), result.receivedAt())) {
       return IntakeResult.AUTO_CORRECTION_ACCEPTED;
     }
-    return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+    return store.supersedeStale(recorded.candidateId(), result.receivedAt())
         ? IntakeResult.CORRECTION_SUPERSEDED
         : IntakeResult.CORRECTION_PENDING;
   }
@@ -74,6 +77,7 @@ public class ResultCandidateIntake {
     NO_CHANGE,
     FIRST_ACCEPTED,
     AUTO_CORRECTION_ACCEPTED,
+    FUTURE_HELD,
     LATE_HELD,
     CORRECTION_SUPERSEDED,
     CORRECTION_PENDING


## `test(correction): verify PostgreSQL first result race`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresFirstCandidateRaceIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresFirstCandidateRaceIntegrationTest.java
new file mode 100644
index 0000000..7733661
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresFirstCandidateRaceIntegrationTest.java
@@ -0,0 +1,92 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.correction.ResultCandidateIntake;
+import com.sportsbook.settlement.correction.ResultCandidateStore;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import com.sportsbook.settlement.result.MatchResultRecord;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import java.util.concurrent.CyclicBarrier;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Import;
+import org.springframework.context.annotation.Primary;
+import org.springframework.jdbc.core.JdbcTemplate;
+
+@Import(PostgresFirstCandidateRaceIntegrationTest.RaceConfiguration.class)
+class PostgresFirstCandidateRaceIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private ResultCandidateIntake intake;
+
+  @Test
+  void acceptsOneConcurrentFirstResultAndSupersedesTheLoser() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    Instant now = Instant.parse("2026-08-22T04:00:00Z");
+    var workers = Executors.newFixedThreadPool(2);
+
+    try {
+      var one =
+          workers.submit(
+              () -> intake.ingest(result(eventId, selectionId, SettlementResult.WON, now)));
+      var two =
+          workers.submit(
+              () -> intake.ingest(result(eventId, selectionId, SettlementResult.LOST, now)));
+
+      assertThat(List.of(one.get(5, TimeUnit.SECONDS), two.get(5, TimeUnit.SECONDS)))
+          .containsExactlyInAnyOrder(
+              ResultCandidateIntake.IntakeResult.FIRST_ACCEPTED,
+              ResultCandidateIntake.IntakeResult.CORRECTION_SUPERSEDED);
+      assertThat(
+              jdbc.queryForList(
+                  "select state from result_candidate where event_id = ?", String.class, eventId))
+          .containsExactlyInAnyOrder("ACCEPTED", "SUPERSEDED");
+    } finally {
+      workers.shutdownNow();
+    }
+  }
+
+  private MatchResultRecord result(
+      UUID eventId, UUID selectionId, SettlementResult outcome, Instant now) {
+    return new MatchResultRecord(
+        eventId, MatchOutcomeMode.COMPLETED, Map.of(selectionId, outcome), now, now);
+  }
+
+  @TestConfiguration(proxyBeanMethods = false)
+  static class RaceConfiguration {
+    @Bean
+    @Primary
+    ResultCandidateStore raceStore(JdbcTemplate jdbc) {
+      return new BarrierStore(jdbc);
+    }
+  }
+
+  static class BarrierStore extends ResultCandidateStore {
+    private final CyclicBarrier barrier = new CyclicBarrier(2);
+
+    BarrierStore(JdbcTemplate jdbc) {
+      super(jdbc);
+    }
+
+    @Override
+    public Optional<AcceptedCandidate> findAcceptedCandidate(UUID eventId) {
+      Optional<AcceptedCandidate> accepted = super.findAcceptedCandidate(eventId);
+      try {
+        barrier.await(2, TimeUnit.SECONDS);
+        return accepted;
+      } catch (Exception exception) {
+        throw new AssertionError(exception);
+      }
+    }
+  }
+}
