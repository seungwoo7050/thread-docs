## `feat(admin): register revision retries`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetry.java b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetry.java
new file mode 100644
index 0000000..7d0b341
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetry.java
@@ -0,0 +1,52 @@
+package com.sportsbook.settlement.admin;
+
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class AdminRevisionRetry {
+
+  private final AdminActionRepository actions;
+  private final AdminRevisionRetryRepository retries;
+  private final AdminRevisionQueryRepository revisions;
+
+  public AdminRevisionRetry(
+      AdminActionRepository actions,
+      AdminRevisionRetryRepository retries,
+      AdminRevisionQueryRepository revisions) {
+    this.actions = actions;
+    this.retries = retries;
+    this.revisions = revisions;
+  }
+
+  @Transactional
+  public Decision claim(UUID idempotencyKey, UUID revisionId) {
+    AdminAction.Kind kind = AdminAction.Kind.REVISION_RETRY;
+    String fingerprint = AdminRequestFingerprint.create(kind, revisionId, "");
+    var replay =
+        AdminActionReplay.requireExact(
+            actions.lockAndFind(idempotencyKey), kind, revisionId, fingerprint);
+    if (replay.isPresent()) {
+      return new Decision(replay.orElseThrow(), true);
+    }
+    UUID token = UUID.randomUUID();
+    if (retries.queue(revisionId).isEmpty()) {
+      if (revisions.find(revisionId).isEmpty()) {
+        throw AdminControlException.notFound("Settlement revision");
+      }
+      throw AdminControlException.conflict("Settlement revision is not paused for operator retry");
+    }
+    AdminAction action =
+        actions.append(
+            idempotencyKey,
+            kind,
+            revisionId,
+            fingerprint,
+            AdminAction.Outcome.REVISION_RETRY_QUEUED,
+            token);
+    return new Decision(action, false);
+  }
+
+  public record Decision(AdminAction action, boolean replay) {}
+}


## `feat(admin): claim manual revision retries`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetryRepository.java b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetryRepository.java
new file mode 100644
index 0000000..eddfc6c
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetryRepository.java
@@ -0,0 +1,50 @@
+package com.sportsbook.settlement.admin;
+
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class AdminRevisionRetryRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public AdminRevisionRetryRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(propagation = Propagation.MANDATORY)
+  public Optional<Queued> queue(UUID revisionId) {
+    return jdbc
+        .query(
+            """
+            update settlement_revision set
+                state = case when wallet_status = 'BLOCKED' then 'BLOCKED' else 'PENDING' end,
+                lease_token = null, lease_until = null, attempt_count = 0,
+                next_retry_at = case when wallet_status = 'BLOCKED'
+                    then greatest(current_timestamp, wallet_next_attempt_at)
+                    else current_timestamp end,
+                last_error_code = null,
+                updated_at = current_timestamp
+            where revision_id = ? and lease_token is null and (
+                state = 'EXHAUSTED'
+                or (state = 'BLOCKED' and next_retry_at is null
+                    and wallet_status = 'BLOCKED' and last_error_code is not null))
+            returning state, next_retry_at, wallet_status = 'BLOCKED' as blocked_proof
+            """,
+            (result, rowNumber) ->
+                new Queued(
+                    result.getString("state"),
+                    result.getBoolean("blocked_proof"),
+                    result.getTimestamp("next_retry_at").toInstant()),
+            revisionId)
+        .stream()
+        .findFirst();
+  }
+
+  public record Queued(String state, boolean blockedProof, Instant nextRetryAt) {}
+}


## `feat(admin): query result candidates`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateQueryRepository.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateQueryRepository.java
new file mode 100644
index 0000000..a27f944
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateQueryRepository.java
@@ -0,0 +1,61 @@
+package com.sportsbook.settlement.admin;
+
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class AdminCandidateQueryRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public AdminCandidateQueryRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(readOnly = true)
+  public Optional<View> find(UUID candidateId) {
+    return jdbc
+        .query(
+            """
+            select c.candidate_id, c.event_id, c.mode, c.settled_at, c.received_at,
+                c.state, c.replaces_candidate_id, c.decision_reason, c.decided_at,
+                coalesce(m.accepted_candidate_id = c.candidate_id, false) as accepted
+            from result_candidate c
+            left join match_result m on m.event_id = c.event_id
+            where c.candidate_id = ?
+            """,
+            (result, rowNumber) ->
+                new View(
+                    result.getObject("candidate_id", UUID.class),
+                    result.getObject("event_id", UUID.class),
+                    result.getString("mode"),
+                    result.getTimestamp("settled_at").toInstant(),
+                    result.getTimestamp("received_at").toInstant(),
+                    result.getString("state"),
+                    result.getObject("replaces_candidate_id", UUID.class),
+                    result.getString("decision_reason"),
+                    result.getTimestamp("decided_at") == null
+                        ? null
+                        : result.getTimestamp("decided_at").toInstant(),
+                    result.getBoolean("accepted")),
+            candidateId)
+        .stream()
+        .findFirst();
+  }
+
+  public record View(
+      UUID candidateId,
+      UUID eventId,
+      String mode,
+      Instant settledAt,
+      Instant receivedAt,
+      String state,
+      UUID replacesCandidateId,
+      String decisionReason,
+      Instant decidedAt,
+      boolean accepted) {}
+}


## `feat(admin): query settlement revisions`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionQueryRepository.java b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionQueryRepository.java
new file mode 100644
index 0000000..0108622
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionQueryRepository.java
@@ -0,0 +1,82 @@
+package com.sportsbook.settlement.admin;
+
+import java.sql.Timestamp;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class AdminRevisionQueryRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public AdminRevisionQueryRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(readOnly = true)
+  public Optional<View> find(UUID revisionId) {
+    return jdbc
+        .query(
+            """
+            select revision_id, bet_id, revision_number, event_id, source_candidate_id,
+                state, attempt_count, next_retry_at, last_error_code, lease_until,
+                wallet_status, wallet_queue_sequence, wallet_operation_group_id,
+                wallet_queued_at, wallet_applied_at, wallet_next_attempt_at,
+                created_at, updated_at, applied_at
+            from settlement_revision where revision_id = ?
+            """,
+            (result, rowNumber) ->
+                new View(
+                    result.getObject("revision_id", UUID.class),
+                    result.getObject("bet_id", UUID.class),
+                    result.getLong("revision_number"),
+                    result.getObject("event_id", UUID.class),
+                    result.getObject("source_candidate_id", UUID.class),
+                    result.getString("state"),
+                    result.getInt("attempt_count"),
+                    instant(result.getTimestamp("next_retry_at")),
+                    result.getString("last_error_code"),
+                    instant(result.getTimestamp("lease_until")),
+                    result.getString("wallet_status"),
+                    (Long) result.getObject("wallet_queue_sequence"),
+                    result.getObject("wallet_operation_group_id", UUID.class),
+                    instant(result.getTimestamp("wallet_queued_at")),
+                    instant(result.getTimestamp("wallet_applied_at")),
+                    instant(result.getTimestamp("wallet_next_attempt_at")),
+                    result.getTimestamp("created_at").toInstant(),
+                    result.getTimestamp("updated_at").toInstant(),
+                    instant(result.getTimestamp("applied_at"))),
+            revisionId)
+        .stream()
+        .findFirst();
+  }
+
+  private static Instant instant(Timestamp value) {
+    return value == null ? null : value.toInstant();
+  }
+
+  public record View(
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber,
+      UUID eventId,
+      UUID sourceCandidateId,
+      String state,
+      int attemptCount,
+      Instant nextRetryAt,
+      String lastErrorCode,
+      Instant leaseUntil,
+      String walletStatus,
+      Long walletQueueSequence,
+      UUID walletOperationGroupId,
+      Instant walletQueuedAt,
+      Instant walletAppliedAt,
+      Instant walletNextAttemptAt,
+      Instant createdAt,
+      Instant updatedAt,
+      Instant appliedAt) {}
+}


## `feat(admin): expose candidate decisions`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCandidateController.java b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateController.java
new file mode 100644
index 0000000..ea49e02
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCandidateController.java
@@ -0,0 +1,38 @@
+package com.sportsbook.settlement.admin;
+
+import java.util.UUID;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestHeader;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/internal/admin/result-candidates")
+public final class AdminCandidateController {
+
+  private static final String IDEMPOTENCY_HEADER = "Idempotency-Key";
+
+  private final AdminCandidateCommands commands;
+
+  public AdminCandidateController(AdminCandidateCommands commands) {
+    this.commands = commands;
+  }
+
+  @PostMapping("/{candidateId}/approve")
+  AdminCandidateCommands.Receipt approve(
+      @RequestHeader(IDEMPOTENCY_HEADER) UUID idempotencyKey, @PathVariable UUID candidateId) {
+    return commands.approve(idempotencyKey, candidateId);
+  }
+
+  @PostMapping("/{candidateId}/reject")
+  AdminCandidateCommands.Receipt reject(
+      @RequestHeader(IDEMPOTENCY_HEADER) UUID idempotencyKey,
+      @PathVariable UUID candidateId,
+      @RequestBody Rejection request) {
+    return commands.reject(idempotencyKey, candidateId, request.reason());
+  }
+
+  record Rejection(String reason) {}
+}


## `feat(admin): expose revision retry`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminRevisionController.java b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionController.java
new file mode 100644
index 0000000..e097b77
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminRevisionController.java
@@ -0,0 +1,26 @@
+package com.sportsbook.settlement.admin;
+
+import java.util.UUID;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestHeader;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/internal/admin/revisions")
+public final class AdminRevisionController {
+
+  private final AdminRevisionCommands commands;
+
+  public AdminRevisionController(AdminRevisionCommands commands) {
+    this.commands = commands;
+  }
+
+  @PostMapping("/{revisionId}/retry")
+  ResponseEntity<AdminRevisionCommands.Receipt> retry(
+      @RequestHeader("Idempotency-Key") UUID idempotencyKey, @PathVariable UUID revisionId) {
+    return ResponseEntity.accepted().body(commands.retry(idempotencyKey, revisionId));
+  }
+}


## `test(admin): verify concurrent candidate decisions`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresAdminCandidateConcurrencyIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresAdminCandidateConcurrencyIntegrationTest.java
new file mode 100644
index 0000000..6d68540
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresAdminCandidateConcurrencyIntegrationTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.admin.AdminCandidateApproval;
+import com.sportsbook.settlement.admin.AdminCandidateRejection;
+import com.sportsbook.settlement.admin.AdminControlException;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+
+class PostgresAdminCandidateConcurrencyIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private AdminCandidateApproval approvals;
+  @Autowired private AdminCandidateRejection rejections;
+
+  @BeforeEach
+  void clearAdminActions() {
+    jdbc.execute("truncate table settlement_admin_action");
+  }
+
+  @Test
+  void commitsOnlyOneConcurrentDecisionAndAction() throws Exception {
+    PendingBet bet = insertPendingBet(UUID.randomUUID());
+    UUID candidateId =
+        insertResultCandidate(
+            bet.eventId(), bet.selectionId(), SettlementResult.WON, Instant.EPOCH, "PENDING");
+    CountDownLatch start = new CountDownLatch(1);
+    var workers = Executors.newFixedThreadPool(2);
+    try {
+      Future<Boolean> approve =
+          workers.submit(
+              () -> decide(start, () -> approvals.decide(UUID.randomUUID(), candidateId)));
+      Future<Boolean> reject =
+          workers.submit(
+              () ->
+                  decide(
+                      start,
+                      () ->
+                          rejections.decide(
+                              UUID.randomUUID(), candidateId, "concurrent rejection")));
+      start.countDown();
+
+      assertThat(List.of(approve.get(), reject.get())).containsExactlyInAnyOrder(true, false);
+    } finally {
+      workers.shutdownNow();
+    }
+    assertThat(jdbc.queryForObject("select count(*) from settlement_admin_action", Integer.class))
+        .isOne();
+    assertThat(
+            jdbc.queryForObject(
+                "select state in ('ACCEPTED','REJECTED') from result_candidate "
+                    + "where candidate_id=?",
+                Boolean.class,
+                candidateId))
+        .isTrue();
+  }
+
+  private static boolean decide(CountDownLatch start, Command command) throws Exception {
+    start.await();
+    try {
+      command.run();
+      return true;
+    } catch (AdminControlException conflict) {
+      return false;
+    }
+  }
+
+  @FunctionalInterface
+  private interface Command {
+    void run();
+  }
+}


## `test(admin): execute queued revision recovery`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresRevisionClaimIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresRevisionClaimIntegrationTest.java
index 7128deb..3053de0 100644
--- a/src/test/java/com/sportsbook/settlement/persistence/PostgresRevisionClaimIntegrationTest.java
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresRevisionClaimIntegrationTest.java
@@ -1,16 +1,97 @@
 package com.sportsbook.settlement.persistence;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
+import com.sportsbook.settlement.admin.AdminCredentials;
+import com.sportsbook.settlement.admin.AdminRevisionRetry;
 import com.sportsbook.settlement.correction.RevisionRecoveryRepository;
 import java.time.Duration;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
+import org.springframework.test.web.servlet.MockMvc;
 
+@AutoConfigureMockMvc
 class PostgresRevisionClaimIntegrationTest extends PostgresIntegrationSupport {
 
+  private static final String ADMIN_KEY = "abcdef0123456789abcdef0123456789";
+
   @Autowired private RevisionRecoveryRepository recovery;
+  @Autowired private AdminRevisionRetry adminRetry;
+  @Autowired private MockMvc mvc;
+
+  @Test
+  void queuesAnExhaustedRevisionOnceBeforeTheScannerClaimsAttemptOne() throws Exception {
+    UUID revisionId = insertRevision(11);
+    UUID actionKey = UUID.randomUUID();
+    jdbc.update(
+        "update settlement_revision set state='EXHAUSTED', attempt_count=12, "
+            + "next_retry_at=null, last_error_code='WALLET_RETRY_EXHAUSTED' "
+            + "where revision_id=?",
+        revisionId);
+
+    retry(actionKey, revisionId, "QUEUED");
+    retry(actionKey, revisionId, "REPLAY");
+    assertThat(
+            jdbc.queryForObject(
+                "select state='PENDING' and attempt_count=0 and lease_token is null "
+                    + "and next_retry_at <= current_timestamp from settlement_revision "
+                    + "where revision_id=?",
+                Boolean.class,
+                revisionId))
+        .isTrue();
+
+    var claim = recovery.claimDue(Duration.ofSeconds(30), 1).get(0);
+
+    assertThat(claim.revisionId()).isEqualTo(revisionId);
+    assertThat(claim.blockedProof()).isFalse();
+    assertThat(
+            jdbc.queryForObject(
+                "select attempt_count=1 and lease_token is not null and next_retry_at is null "
+                    + "from settlement_revision where revision_id=?",
+                Boolean.class,
+                revisionId))
+        .isTrue();
+  }
+
+  private void retry(UUID actionKey, UUID revisionId, String outcome) throws Exception {
+    mvc.perform(
+            post("/internal/admin/revisions/{id}/retry", revisionId)
+                .header("Idempotency-Key", actionKey)
+                .header(AdminCredentials.SERVICE_HEADER, AdminCredentials.CALLER)
+                .header(AdminCredentials.API_KEY_HEADER, ADMIN_KEY))
+        .andExpect(status().isAccepted())
+        .andExpect(jsonPath("$.outcome").value(outcome));
+  }
+
+  @Test
+  void preservesAPausedBlockedProofUntilItsWalletDueTime() {
+    UUID revisionId = insertRevision(11);
+    jdbc.update(
+        "update settlement_revision set state='BLOCKED', attempt_count=12, "
+            + "next_retry_at=null, last_error_code='WALLET_RETRY_EXHAUSTED', "
+            + "wallet_status='BLOCKED', wallet_queue_sequence=9, "
+            + "wallet_queued_at=current_timestamp, "
+            + "wallet_next_attempt_at=current_timestamp + interval '1 hour' "
+            + "where revision_id=?",
+        revisionId);
+
+    adminRetry.claim(UUID.randomUUID(), revisionId);
+
+    assertThat(recovery.claimDue(Duration.ofSeconds(30), 1)).isEmpty();
+    assertThat(
+            jdbc.queryForObject(
+                "select state='BLOCKED' and attempt_count=0 and wallet_status='BLOCKED' "
+                    + "and wallet_queue_sequence=9 and next_retry_at=wallet_next_attempt_at "
+                    + "from settlement_revision where revision_id=?",
+                Boolean.class,
+                revisionId))
+        .isTrue();
+  }
 
   @Test
   void claimsDuePendingAndBlockedRowsButExcludesFutureAndExhaustedWork() {
