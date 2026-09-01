# 멱등 운영 제어와 감사 로그

## `build(flyway): add V10 admin actions`

diff --git a/src/main/resources/db/migration/V10__admin_action.sql b/src/main/resources/db/migration/V10__admin_action.sql
new file mode 100644
index 0000000..1a43524
--- /dev/null
+++ b/src/main/resources/db/migration/V10__admin_action.sql
@@ -0,0 +1,38 @@
+-- Record each authenticated operator command exactly once.
+
+CREATE TABLE settlement_admin_action (
+    idempotency_key     UUID                     PRIMARY KEY,
+    action_kind        VARCHAR(32)              NOT NULL,
+    target_id          UUID                     NOT NULL,
+    request_fingerprint CHAR(64)                NOT NULL,
+    outcome            VARCHAR(32)              NOT NULL,
+    execution_token    UUID,
+    created_at         TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
+    completed_at       TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
+    CONSTRAINT ck_settlement_admin_action_kind CHECK (
+        action_kind IN ('CANDIDATE_APPROVE', 'CANDIDATE_REJECT', 'REVISION_RETRY')),
+    CONSTRAINT ck_settlement_admin_action_fingerprint CHECK (
+        request_fingerprint ~ '^[0-9a-f]{64}$'),
+    CONSTRAINT ck_settlement_admin_action_outcome CHECK (
+        (action_kind = 'CANDIDATE_APPROVE' AND outcome = 'CANDIDATE_APPROVED')
+        OR (action_kind = 'CANDIDATE_REJECT' AND outcome = 'CANDIDATE_REJECTED')
+        OR (action_kind = 'REVISION_RETRY' AND outcome = 'REVISION_RETRY_QUEUED')),
+    CONSTRAINT ck_settlement_admin_action_execution CHECK (
+        (action_kind = 'REVISION_RETRY' AND execution_token IS NOT NULL)
+        OR (action_kind <> 'REVISION_RETRY' AND execution_token IS NULL)),
+    CONSTRAINT ck_settlement_admin_action_time CHECK (completed_at >= created_at)
+);
+
+CREATE INDEX ix_settlement_admin_action_target
+    ON settlement_admin_action (target_id, action_kind, created_at);
+
+CREATE FUNCTION reject_settlement_admin_action_mutation() RETURNS TRIGGER
+LANGUAGE plpgsql AS $$
+BEGIN
+    RAISE EXCEPTION 'settlement admin actions are append-only';
+END;
+$$;
+
+CREATE TRIGGER settlement_admin_action_append_only
+    BEFORE UPDATE OR DELETE ON settlement_admin_action
+    FOR EACH ROW EXECUTE FUNCTION reject_settlement_admin_action_mutation();


## `feat(admin): canonicalize operator actions`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminAction.java b/src/main/java/com/sportsbook/settlement/admin/AdminAction.java
new file mode 100644
index 0000000..00426fd
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminAction.java
@@ -0,0 +1,41 @@
+package com.sportsbook.settlement.admin;
+
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record AdminAction(
+    UUID idempotencyKey,
+    Kind kind,
+    UUID targetId,
+    String requestFingerprint,
+    Outcome outcome,
+    UUID executionToken,
+    Instant createdAt,
+    Instant completedAt) {
+
+  public AdminAction {
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    Objects.requireNonNull(kind, "kind");
+    Objects.requireNonNull(targetId, "targetId");
+    Objects.requireNonNull(requestFingerprint, "requestFingerprint");
+    Objects.requireNonNull(outcome, "outcome");
+    Objects.requireNonNull(createdAt, "createdAt");
+    Objects.requireNonNull(completedAt, "completedAt");
+    if ((kind == Kind.REVISION_RETRY) != (executionToken != null)) {
+      throw new IllegalArgumentException("Only revision retries carry an execution token");
+    }
+  }
+
+  public enum Kind {
+    CANDIDATE_APPROVE,
+    CANDIDATE_REJECT,
+    REVISION_RETRY
+  }
+
+  public enum Outcome {
+    CANDIDATE_APPROVED,
+    CANDIDATE_REJECTED,
+    REVISION_RETRY_QUEUED
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRequestFingerprint.java b/src/main/java/com/sportsbook/settlement/admin/AdminRequestFingerprint.java
new file mode 100644
index 0000000..e5b52b1
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRequestFingerprint.java
@@ -0,0 +1,28 @@
+package com.sportsbook.settlement.admin;
+
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Objects;
+import java.util.UUID;
+
+public final class AdminRequestFingerprint {
+
+  private AdminRequestFingerprint() {}
+
+  public static String create(AdminAction.Kind kind, UUID targetId, String canonicalPayload) {
+    Objects.requireNonNull(kind, "kind");
+    Objects.requireNonNull(targetId, "targetId");
+    Objects.requireNonNull(canonicalPayload, "canonicalPayload");
+    String canonical = "admin-command-v1\n" + kind + "\n" + targetId + "\n" + canonicalPayload;
+    try {
+      return HexFormat.of()
+          .formatHex(
+              MessageDigest.getInstance("SHA-256")
+                  .digest(canonical.getBytes(StandardCharsets.UTF_8)));
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("SHA-256 is unavailable", impossible);
+    }
+  }
+}


## `feat(admin): persist idempotent actions`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminActionRepository.java b/src/main/java/com/sportsbook/settlement/admin/AdminActionRepository.java
new file mode 100644
index 0000000..4601eba
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminActionRepository.java
@@ -0,0 +1,76 @@
+package com.sportsbook.settlement.admin;
+
+import java.sql.ResultSet;
+import java.sql.SQLException;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class AdminActionRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public AdminActionRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(propagation = Propagation.MANDATORY)
+  public Optional<AdminAction> lockAndFind(UUID idempotencyKey) {
+    jdbc.query(
+        "select pg_advisory_xact_lock(hashtextextended(cast(? as text), 0))",
+        (result, rowNumber) -> Boolean.TRUE,
+        idempotencyKey);
+    return jdbc
+        .query(
+            "select * from settlement_admin_action where idempotency_key = ?",
+            (result, rowNumber) -> read(result),
+            idempotencyKey)
+        .stream()
+        .findFirst();
+  }
+
+  @Transactional(propagation = Propagation.MANDATORY)
+  public AdminAction append(
+      UUID idempotencyKey,
+      AdminAction.Kind kind,
+      UUID targetId,
+      String fingerprint,
+      AdminAction.Outcome outcome,
+      UUID executionToken) {
+    return jdbc
+        .query(
+            """
+            insert into settlement_admin_action (
+                idempotency_key, action_kind, target_id, request_fingerprint,
+                outcome, execution_token, created_at, completed_at)
+            values (?, ?, ?, ?, ?, ?, current_timestamp, current_timestamp)
+            returning *
+            """,
+            (result, rowNumber) -> read(result),
+            idempotencyKey,
+            kind.name(),
+            targetId,
+            fingerprint,
+            outcome.name(),
+            executionToken)
+        .stream()
+        .findFirst()
+        .orElseThrow(() -> new IllegalStateException("Admin action insert returned no row"));
+  }
+
+  private static AdminAction read(ResultSet result) throws SQLException {
+    return new AdminAction(
+        result.getObject("idempotency_key", UUID.class),
+        AdminAction.Kind.valueOf(result.getString("action_kind")),
+        result.getObject("target_id", UUID.class),
+        result.getString("request_fingerprint"),
+        AdminAction.Outcome.valueOf(result.getString("outcome")),
+        result.getObject("execution_token", UUID.class),
+        result.getTimestamp("created_at").toInstant(),
+        result.getTimestamp("completed_at").toInstant());
+  }
+}


## `feat(admin): validate idempotent replays`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminActionReplay.java b/src/main/java/com/sportsbook/settlement/admin/AdminActionReplay.java
new file mode 100644
index 0000000..b81a0fd
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminActionReplay.java
@@ -0,0 +1,25 @@
+package com.sportsbook.settlement.admin;
+
+import java.util.Optional;
+import java.util.UUID;
+
+public final class AdminActionReplay {
+
+  private AdminActionReplay() {}
+
+  public static Optional<AdminAction> requireExact(
+      Optional<AdminAction> existing, AdminAction.Kind kind, UUID targetId, String fingerprint) {
+    if (existing.isEmpty()) {
+      return Optional.empty();
+    }
+    AdminAction action = existing.orElseThrow();
+    boolean exact =
+        action.kind() == kind
+            && action.targetId().equals(targetId)
+            && action.requestFingerprint().equals(fingerprint);
+    if (!exact) {
+      throw AdminControlException.conflict("Idempotency-Key is already bound to another request");
+    }
+    return Optional.of(action);
+  }
+}


## `feat(admin): lock candidate decisions`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
index 820c9f1..3323ddd 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateStore.java
@@ -87,6 +87,15 @@ public class ResultCandidateStore {
 
   @Transactional
   public boolean acceptFirst(UUID candidateId, java.time.Instant decidedAt) {
+    return acceptFirst(candidateId, decidedAt, "FIRST_RESULT");
+  }
+
+  @Transactional
+  public boolean approveFirst(UUID candidateId, java.time.Instant decidedAt) {
+    return acceptFirst(candidateId, decidedAt, "OPERATOR_APPROVED");
+  }
+
+  private boolean acceptFirst(UUID candidateId, java.time.Instant decidedAt, String reason) {
     int current =
         jdbc.update(
             """
@@ -114,10 +123,11 @@ public class ResultCandidateStore {
         jdbc.update(
             """
             update result_candidate set state = 'ACCEPTED', decided_at = ?,
-                decision_reason = 'FIRST_RESULT'
+                decision_reason = ?
             where candidate_id = ? and state = 'PENDING'
             """,
             required(decidedAt),
+            reason,
             candidateId);
     if (accepted != 1) {
       throw new IllegalStateException("First result decision lost its candidate");
@@ -125,6 +135,28 @@ public class ResultCandidateStore {
     return true;
   }
 
+  public Optional<AdminCandidate> lockForAdmin(UUID candidateId) {
+    return jdbc
+        .query(
+            """
+            select c.event_id, c.state, c.settled_at, c.replaces_candidate_id,
+                m.accepted_candidate_id
+            from result_candidate c
+            left join match_result m on m.event_id = c.event_id
+            where c.candidate_id = ? for update of c
+            """,
+            (result, rowNumber) ->
+                new AdminCandidate(
+                    result.getObject("event_id", UUID.class),
+                    ResultCandidateState.valueOf(result.getString("state")),
+                    result.getTimestamp("settled_at").toInstant(),
+                    result.getObject("replaces_candidate_id", UUID.class),
+                    result.getObject("accepted_candidate_id", UUID.class)),
+            candidateId)
+        .stream()
+        .findFirst();
+  }
+
   public Optional<UUID> findAcceptedCandidateId(UUID eventId) {
     return jdbc
         .query(
@@ -318,4 +350,11 @@ public class ResultCandidateStore {
   }
 
   public record AcceptedCandidate(UUID candidateId, java.time.Instant receivedAt) {}
+
+  public record AdminCandidate(
+      UUID eventId,
+      ResultCandidateState state,
+      Instant settledAt,
+      UUID replacesCandidateId,
+      UUID acceptedCandidateId) {}
 }


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


## `feat(admin): reject result candidates`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateRejection.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateRejection.java
new file mode 100644
index 0000000..c80dafe
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateRejection.java
@@ -0,0 +1,68 @@
+package com.sportsbook.settlement.admin;
+
+import com.sportsbook.settlement.correction.ResultCandidateState;
+import com.sportsbook.settlement.correction.ResultCandidateStore;
+import com.sportsbook.settlement.persistence.DatabaseTimeSource;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class AdminCandidateRejection {
+
+  private final AdminActionRepository actions;
+  private final ResultCandidateStore candidates;
+  private final DatabaseTimeSource databaseTime;
+
+  public AdminCandidateRejection(
+      AdminActionRepository actions,
+      ResultCandidateStore candidates,
+      DatabaseTimeSource databaseTime) {
+    this.actions = actions;
+    this.candidates = candidates;
+    this.databaseTime = databaseTime;
+  }
+
+  @Transactional
+  public Decision decide(UUID idempotencyKey, UUID candidateId, String requestedReason) {
+    String reason = normalize(requestedReason);
+    AdminAction.Kind kind = AdminAction.Kind.CANDIDATE_REJECT;
+    String fingerprint = AdminRequestFingerprint.create(kind, candidateId, reason);
+    var replay =
+        AdminActionReplay.requireExact(
+            actions.lockAndFind(idempotencyKey), kind, candidateId, fingerprint);
+    if (replay.isPresent()) {
+      return new Decision(replay.orElseThrow(), true);
+    }
+    ResultCandidateStore.AdminCandidate candidate =
+        candidates
+            .lockForAdmin(candidateId)
+            .orElseThrow(() -> AdminControlException.notFound("Result candidate"));
+    if (candidate.state() != ResultCandidateState.PENDING) {
+      throw AdminControlException.conflict("Result candidate is already decided");
+    }
+    if (!candidates.reject(candidateId, databaseTime.currentTimestamp(), reason)) {
+      throw AdminControlException.conflict("Result candidate decision changed concurrently");
+    }
+    AdminAction action =
+        actions.append(
+            idempotencyKey,
+            kind,
+            candidateId,
+            fingerprint,
+            AdminAction.Outcome.CANDIDATE_REJECTED,
+            null);
+    return new Decision(action, false);
+  }
+
+  private static String normalize(String reason) {
+    String normalized = reason == null ? "" : reason.strip();
+    boolean control = normalized.codePoints().anyMatch(Character::isISOControl);
+    if (normalized.isEmpty() || normalized.length() > 256 || control) {
+      throw AdminControlException.invalid("Rejection reason must be 1 to 256 printable characters");
+    }
+    return normalized;
+  }
+
+  public record Decision(AdminAction action, boolean replay) {}
+}


## `feat(admin): redrive approved result fanout`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java
new file mode 100644
index 0000000..7690831
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateCommands.java
@@ -0,0 +1,51 @@
+package com.sportsbook.settlement.admin;
+
+import com.sportsbook.settlement.correction.CorrectionFanout;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.ResultFanout;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+
+@Service
+public class AdminCandidateCommands {
+
+  private final AdminCandidateApproval approvals;
+  private final AdminCandidateRejection rejections;
+  private final AcceptedResultRepository acceptedResults;
+  private final ResultFanout baseFanout;
+  private final CorrectionFanout correctionFanout;
+
+  public AdminCandidateCommands(
+      AdminCandidateApproval approvals,
+      AdminCandidateRejection rejections,
+      AcceptedResultRepository acceptedResults,
+      ResultFanout baseFanout,
+      CorrectionFanout correctionFanout) {
+    this.approvals = approvals;
+    this.rejections = rejections;
+    this.acceptedResults = acceptedResults;
+    this.baseFanout = baseFanout;
+    this.correctionFanout = correctionFanout;
+  }
+
+  public Receipt approve(UUID idempotencyKey, UUID candidateId) {
+    AdminCandidateApproval.Decision decision = approvals.decide(idempotencyKey, candidateId);
+    var accepted =
+        acceptedResults
+            .findByEventId(decision.eventId())
+            .orElseThrow(() -> new IllegalStateException("Approved result projection is missing"));
+    baseFanout.fanOut(accepted);
+    correctionFanout.fanOut(accepted);
+    return new Receipt(
+        decision.action().idempotencyKey(), decision.action().outcome().name(), decision.replay());
+  }
+
+  public Receipt reject(UUID idempotencyKey, UUID candidateId, String reason) {
+    AdminCandidateRejection.Decision decision =
+        rejections.decide(idempotencyKey, candidateId, reason);
+    return new Receipt(
+        decision.action().idempotencyKey(), decision.action().outcome().name(), decision.replay());
+  }
+
+  public record Receipt(UUID idempotencyKey, String outcome, boolean replay) {}
+}


