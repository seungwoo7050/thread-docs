# 결과 후보 판정과 인과 순서

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


## `feat(correction): capture immutable result snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidate.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidate.java
new file mode 100644
index 0000000..fa8eaf2
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidate.java
@@ -0,0 +1,63 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.infrastructure.id.UuidV7;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import java.time.Instant;
+import java.util.Map;
+import java.util.Objects;
+import java.util.UUID;
+
+public record ResultCandidate(
+    UUID candidateId,
+    Long sequence,
+    UUID eventId,
+    String fingerprint,
+    MatchOutcomeMode mode,
+    Map<UUID, SettlementResult> outcomes,
+    Instant settledAt,
+    Instant receivedAt,
+    ResultCandidateState state,
+    UUID replacesCandidateId,
+    Instant decidedAt,
+    String decisionReason) {
+
+  public ResultCandidate {
+    Objects.requireNonNull(candidateId, "candidateId");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(mode, "mode");
+    outcomes = Map.copyOf(Objects.requireNonNull(outcomes, "outcomes"));
+    Objects.requireNonNull(settledAt, "settledAt");
+    Objects.requireNonNull(receivedAt, "receivedAt");
+    Objects.requireNonNull(state, "state");
+    if (fingerprint == null || !fingerprint.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("Candidate fingerprint must be lowercase SHA-256");
+    }
+    if ((state == ResultCandidateState.PENDING) != (decidedAt == null)) {
+      throw new IllegalArgumentException("Candidate decision timestamp must match state");
+    }
+  }
+
+  public static ResultCandidate pending(
+      UUID eventId,
+      String fingerprint,
+      MatchOutcomeMode mode,
+      Map<UUID, SettlementResult> outcomes,
+      Instant settledAt,
+      Instant receivedAt,
+      UUID replacesCandidateId) {
+    return new ResultCandidate(
+        UuidV7.generate(),
+        null,
+        eventId,
+        fingerprint,
+        mode,
+        outcomes,
+        settledAt,
+        receivedAt,
+        ResultCandidateState.PENDING,
+        replacesCandidateId,
+        null,
+        null);
+  }
+}


## `feat(correction): fingerprint semantic resolutions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateFingerprinter.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateFingerprinter.java
new file mode 100644
index 0000000..b10b161
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateFingerprinter.java
@@ -0,0 +1,42 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Map;
+import java.util.UUID;
+
+public final class ResultCandidateFingerprinter {
+
+  public String fingerprint(
+      UUID eventId, MatchOutcomeMode mode, Map<UUID, SettlementResult> outcomes) {
+    try {
+      MessageDigest digest = MessageDigest.getInstance("SHA-256");
+      add(digest, "result-candidate-v1");
+      add(digest, eventId.toString());
+      add(digest, mode.name());
+      if (mode != MatchOutcomeMode.VOIDED) {
+        outcomes.entrySet().stream()
+            .sorted(Map.Entry.comparingByKey())
+            .forEach(
+                entry -> {
+                  add(digest, entry.getKey().toString());
+                  add(digest, entry.getValue().name());
+                });
+      }
+      return HexFormat.of().formatHex(digest.digest());
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("JDK must provide SHA-256", exception);
+    }
+  }
+
+  private static void add(MessageDigest digest, String value) {
+    byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
+    digest.update(ByteBuffer.allocate(Integer.BYTES).putInt(bytes.length).array());
+    digest.update(bytes);
+  }
+}


## `feat(correction): deduplicate semantic candidates`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
new file mode 100644
index 0000000..fee82b4
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -0,0 +1,96 @@
+package com.sportsbook.settlement.correction;
+
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
+
+import java.util.Comparator;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class ResultCandidateStore {
+
+  private final JdbcTemplate jdbc;
+
+  public ResultCandidateStore(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional
+  public RecordOutcome record(ResultCandidate candidate) {
+    RecordOutcome existing = find(candidate.eventId(), candidate.fingerprint());
+    if (existing != null) {
+      return existing;
+    }
+    int inserted =
+        jdbc.update(
+            """
+            insert into result_candidate (
+                candidate_id, event_id, fingerprint, mode, settled_at, received_at,
+                state, replaces_candidate_id, decided_at, decision_reason)
+            values (?, ?, ?, ?, ?, ?, 'PENDING', ?, null, null)
+            on conflict (event_id, fingerprint) do nothing
+            """,
+            candidate.candidateId(),
+            candidate.eventId(),
+            candidate.fingerprint(),
+            candidate.mode().name(),
+            required(candidate.settledAt()),
+            required(candidate.receivedAt()),
+            candidate.replacesCandidateId());
+    if (inserted == 0) {
+      RecordOutcome raced = find(candidate.eventId(), candidate.fingerprint());
+      if (raced == null) {
+        throw new IllegalStateException("Conflicting candidate insert has no durable row");
+      }
+      return raced;
+    }
+    List<Object[]> selections =
+        candidate.outcomes().entrySet().stream()
+            .sorted(Comparator.comparing(entry -> entry.getKey().toString()))
+            .map(
+                entry ->
+                    new Object[] {candidate.candidateId(), entry.getKey(), entry.getValue().name()})
+            .toList();
+    jdbc.batchUpdate(
+        """
+        insert into result_candidate_selection (candidate_id, selection_id, outcome)
+        values (?, ?, ?)
+        """,
+        selections);
+    return new RecordOutcome(
+        RecordKind.CREATED, candidate.candidateId(), ResultCandidateState.PENDING);
+  }
+
+  private RecordOutcome find(UUID eventId, String fingerprint) {
+    return jdbc
+        .query(
+            """
+            select candidate_id, state from result_candidate
+            where event_id = ? and fingerprint = ?
+            """,
+            (result, rowNumber) -> {
+              ResultCandidateState state = ResultCandidateState.valueOf(result.getString("state"));
+              RecordKind kind =
+                  state == ResultCandidateState.PENDING
+                      ? RecordKind.EXACT_REPLAY
+                      : RecordKind.NO_CHANGE;
+              return new RecordOutcome(kind, result.getObject("candidate_id", UUID.class), state);
+            },
+            eventId,
+            fingerprint)
+        .stream()
+        .findFirst()
+        .orElse(null);
+  }
+
+  public enum RecordKind {
+    CREATED,
+    EXACT_REPLAY,
+    NO_CHANGE
+  }
+
+  public record RecordOutcome(RecordKind kind, UUID candidateId, ResultCandidateState state) {}
+}


## `feat(correction): define candidate state transitions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateState.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateState.java
new file mode 100644
index 0000000..753c72e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateState.java
@@ -0,0 +1,19 @@
+package com.sportsbook.settlement.correction;
+
+public enum ResultCandidateState {
+  PENDING,
+  ACCEPTED,
+  SUPERSEDED,
+  REJECTED;
+
+  public boolean canTransitionTo(ResultCandidateState target) {
+    if (target == null || target == this) {
+      return false;
+    }
+    return switch (this) {
+      case PENDING -> target == ACCEPTED || target == SUPERSEDED || target == REJECTED;
+      case ACCEPTED -> target == SUPERSEDED;
+      case SUPERSEDED, REJECTED -> false;
+    };
+  }
+}


## `feat(correction): accept first result candidate`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
new file mode 100644
index 0000000..801165d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -0,0 +1,48 @@
+package com.sportsbook.settlement.correction;
+
+import com.sportsbook.settlement.result.MatchResultRecord;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class ResultCandidateIntake {
+
+  private final ResultCandidateStore store;
+  private final ResultCandidateFingerprinter fingerprints;
+
+  public ResultCandidateIntake(ResultCandidateStore store) {
+    this.store = store;
+    this.fingerprints = new ResultCandidateFingerprinter();
+  }
+
+  @Transactional
+  public IntakeResult ingest(MatchResultRecord result) {
+    String fingerprint =
+        fingerprints.fingerprint(result.eventId(), result.mode(), result.outcomes());
+    ResultCandidate candidate =
+        ResultCandidate.pending(
+            result.eventId(),
+            fingerprint,
+            result.mode(),
+            result.outcomes(),
+            result.settledAt(),
+            result.receivedAt(),
+            null);
+    ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
+    if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
+      return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
+          ? IntakeResult.EXACT_REPLAY
+          : IntakeResult.NO_CHANGE;
+    }
+    return store.acceptFirst(candidate.candidateId(), result.receivedAt())
+        ? IntakeResult.FIRST_ACCEPTED
+        : IntakeResult.CORRECTION_PENDING;
+  }
+
+  public enum IntakeResult {
+    EXACT_REPLAY,
+    NO_CHANGE,
+    FIRST_ACCEPTED,
+    CORRECTION_PENDING
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index fee82b4..d9f0cd9 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -64,6 +64,45 @@ public class ResultCandidateStore {
         RecordKind.CREATED, candidate.candidateId(), ResultCandidateState.PENDING);
   }
 
+  @Transactional
+  public boolean acceptFirst(UUID candidateId, java.time.Instant decidedAt) {
+    int current =
+        jdbc.update(
+            """
+            insert into match_result (
+                event_id, mode, settled_at, received_at, accepted_candidate_id)
+            select event_id, mode, settled_at, received_at, candidate_id
+            from result_candidate where candidate_id = ? and state = 'PENDING'
+            on conflict (event_id) do nothing
+            """,
+            candidateId);
+    if (current == 0) {
+      return false;
+    }
+    jdbc.update(
+        """
+        insert into match_selection_result (event_id, selection_id, outcome)
+        select c.event_id, s.selection_id, s.outcome
+        from result_candidate c join result_candidate_selection s
+          on s.candidate_id = c.candidate_id
+        where c.candidate_id = ?
+        """,
+        candidateId);
+    int accepted =
+        jdbc.update(
+            """
+            update result_candidate set state = 'ACCEPTED', decided_at = ?,
+                decision_reason = 'FIRST_RESULT'
+            where candidate_id = ? and state = 'PENDING'
+            """,
+            required(decidedAt),
+            candidateId);
+    if (accepted != 1) {
+      throw new IllegalStateException("First result decision lost its candidate");
+    }
+    return true;
+  }
+
   private RecordOutcome find(UUID eventId, String fingerprint) {
     return jdbc
         .query(


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


