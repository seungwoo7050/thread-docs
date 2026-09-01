# 승인 결과 프로젝션과 기본 정산 팬아웃

## `feat(result): model accepted result projections`

diff --git a/src/main/java/com/sportsbook/settlement/result/AcceptedResult.java b/src/main/java/com/sportsbook/settlement/result/AcceptedResult.java
new file mode 100644
index 0000000..5dd1578
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/result/AcceptedResult.java
@@ -0,0 +1,38 @@
+package com.sportsbook.settlement.result;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import java.time.Instant;
+import java.util.Collections;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.Objects;
+import java.util.Optional;
+import java.util.UUID;
+
+/** Immutable projection of the currently accepted result evidence for one event. */
+public record AcceptedResult(
+    UUID eventId,
+    UUID candidateId,
+    MatchOutcomeMode mode,
+    Map<UUID, SettlementResult> outcomes,
+    Instant sourceSettledAt) {
+
+  public AcceptedResult {
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(candidateId, "candidateId");
+    Objects.requireNonNull(mode, "mode");
+    LinkedHashMap<UUID, SettlementResult> ordered = new LinkedHashMap<>();
+    Objects.requireNonNull(outcomes, "outcomes")
+        .forEach(
+            (selectionId, outcome) ->
+                ordered.put(
+                    Objects.requireNonNull(selectionId, "selectionId"),
+                    Objects.requireNonNull(outcome, "outcome")));
+    outcomes = Collections.unmodifiableMap(ordered);
+    Objects.requireNonNull(sourceSettledAt, "sourceSettledAt");
+  }
+
+  public Optional<SettlementResult> resolve(UUID selectionId) {
+    return mode.resolve(outcomes.get(Objects.requireNonNull(selectionId, "selectionId")));
+  }
+}


## `feat(result): read accepted result projections`

diff --git a/src/main/java/com/sportsbook/settlement/result/AcceptedResultRepository.java b/src/main/java/com/sportsbook/settlement/result/AcceptedResultRepository.java
new file mode 100644
index 0000000..6ab54a4
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/result/AcceptedResultRepository.java
@@ -0,0 +1,49 @@
+package com.sportsbook.settlement.result;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import java.util.LinkedHashMap;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+@Repository
+public class AcceptedResultRepository {
+
+  private final JdbcTemplate jdbc;
+
+  public AcceptedResultRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public Optional<AcceptedResult> findByEventId(UUID eventId) {
+    return jdbc.query(
+        """
+        select m.event_id, m.accepted_candidate_id, m.mode, m.settled_at,
+            s.selection_id, s.outcome
+        from match_result m
+        left join match_selection_result s on s.event_id = m.event_id
+        where m.event_id = ? and m.accepted_candidate_id is not null
+        order by s.selection_id
+        """,
+        result -> {
+          if (!result.next()) {
+            return Optional.empty();
+          }
+          UUID acceptedEventId = result.getObject("event_id", UUID.class);
+          UUID candidateId = result.getObject("accepted_candidate_id", UUID.class);
+          MatchOutcomeMode mode = MatchOutcomeMode.valueOf(result.getString("mode"));
+          var sourceSettledAt = result.getTimestamp("settled_at").toInstant();
+          var outcomes = new LinkedHashMap<UUID, SettlementResult>();
+          do {
+            UUID selectionId = result.getObject("selection_id", UUID.class);
+            if (selectionId != null) {
+              outcomes.put(selectionId, SettlementResult.valueOf(result.getString("outcome")));
+            }
+          } while (result.next());
+          return Optional.of(
+              new AcceptedResult(acceptedEventId, candidateId, mode, outcomes, sourceSettledAt));
+        },
+        eventId);
+  }
+}


## `feat(result): apply accepted result sources`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
index 16722c6..406d0de 100644
--- a/src/main/java/com/sportsbook/settlement/domain/Bet.java
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -187,6 +187,24 @@ public class Bet {
     return changed;
   }
 
+  public boolean applyAcceptedResult(
+      UUID eventId, UUID candidateId, Map<UUID, SettlementResult> resolvedOutcomes, Instant now) {
+    if (status != SettlementStatus.PENDING) {
+      throw new IllegalStateException("Cannot project results after terminal settlement");
+    }
+    boolean changed = false;
+    for (BetSelection selection : selections) {
+      if (selection.eventId().equals(eventId)) {
+        changed |=
+            selection.applyCandidate(candidateId, resolvedOutcomes.get(selection.selectionId()));
+      }
+    }
+    if (changed) {
+      updatedAt = Objects.requireNonNull(now, "now");
+    }
+    return changed;
+  }
+
   public boolean allSelectionsResolved() {
     return !selections.isEmpty() && selections.stream().allMatch(s -> s.outcome() != null);
   }


## `feat(result): query unowned pending bets`

diff --git a/src/main/java/com/sportsbook/settlement/persistence/BetRepository.java b/src/main/java/com/sportsbook/settlement/persistence/BetRepository.java
index 82f5c16..6bfe722 100644
--- a/src/main/java/com/sportsbook/settlement/persistence/BetRepository.java
+++ b/src/main/java/com/sportsbook/settlement/persistence/BetRepository.java
@@ -28,6 +28,15 @@ public interface BetRepository extends JpaRepository<Bet, UUID> {
           + "com.sportsbook.settlement.domain.SettlementStatus.PENDING order by b.betId")
   List<UUID> findPendingIdsByEvent(@Param("eventId") UUID eventId);
 
+  @Query(
+      value =
+          "select distinct b.bet_id from bet b join bet_selection s on s.bet_id = b.bet_id "
+              + "where b.status = 'PENDING' and s.event_id = :eventId "
+              + "and not exists (select 1 from settlement_attempt a where a.bet_id = b.bet_id) "
+              + "order by b.bet_id",
+      nativeQuery = true)
+  List<UUID> findResultActionableIdsByEvent(@Param("eventId") UUID eventId);
+
   @Query(
       "select distinct b.betId from Bet b join b.selections s "
           + "where b.status = com.sportsbook.settlement.domain.SettlementStatus.SETTLED "


## `feat(result): prepare accepted result claims`

diff --git a/src/main/java/com/sportsbook/settlement/result/ResultSettlementPreparer.java b/src/main/java/com/sportsbook/settlement/result/ResultSettlementPreparer.java
new file mode 100644
index 0000000..8cde33b
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/result/ResultSettlementPreparer.java
@@ -0,0 +1,71 @@
+package com.sportsbook.settlement.result;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Clock;
+import java.util.LinkedHashMap;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class ResultSettlementPreparer {
+
+  private final BetRepository bets;
+  private final SettlementAttemptRepository attempts;
+  private final BaseSettlementPlanner planner;
+  private final SettlementRuntimeProperties runtime;
+  private final Clock clock;
+
+  public ResultSettlementPreparer(
+      BetRepository bets,
+      SettlementAttemptRepository attempts,
+      BaseSettlementPlanner planner,
+      SettlementRuntimeProperties runtime,
+      Clock clock) {
+    this.bets = bets;
+    this.attempts = attempts;
+    this.planner = planner;
+    this.runtime = runtime;
+    this.clock = clock;
+  }
+
+  @Transactional
+  public Optional<SettlementExecution> prepare(UUID betId, AcceptedResult accepted) {
+    Bet bet = bets.findForUpdateById(betId).orElseThrow();
+    if (bet.status() != SettlementStatus.PENDING || attempts.exists(betId)) {
+      return Optional.empty();
+    }
+    var outcomes = new LinkedHashMap<UUID, SettlementResult>();
+    boolean related = false;
+    for (var selection : bet.selections()) {
+      if (selection.eventId().equals(accepted.eventId())) {
+        related = true;
+        accepted
+            .resolve(selection.selectionId())
+            .ifPresent(value -> outcomes.put(selection.selectionId(), value));
+      }
+    }
+    if (!related) {
+      throw new IllegalArgumentException("Accepted result is unrelated to pending bet");
+    }
+    bet.applyAcceptedResult(accepted.eventId(), accepted.candidateId(), outcomes, clock.instant());
+    if (!bet.allSelectionsResolved()) {
+      return Optional.empty();
+    }
+    var draft = planner.plan(bet, accepted.eventId());
+    return attempts
+        .claimPending(draft, runtime.leaseDuration())
+        .map(claimed -> new SettlementExecution(claimed, bet.userId()))
+        .or(
+            () -> {
+              throw new IllegalStateException("Initial result claim lost after bet lock");
+            });
+  }
+}


## `feat(result): fan out accepted results`

diff --git a/src/main/java/com/sportsbook/settlement/result/ResultFanout.java b/src/main/java/com/sportsbook/settlement/result/ResultFanout.java
new file mode 100644
index 0000000..7b3d5f6
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/result/ResultFanout.java
@@ -0,0 +1,31 @@
+package com.sportsbook.settlement.result;
+
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.execution.SettlementExecutionRunner;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.util.ArrayList;
+import java.util.List;
+import org.springframework.stereotype.Service;
+
+@Service
+public class ResultFanout {
+
+  private final BetRepository bets;
+  private final ResultSettlementPreparer preparer;
+  private final SettlementExecutionRunner runner;
+
+  public ResultFanout(
+      BetRepository bets, ResultSettlementPreparer preparer, SettlementExecutionRunner runner) {
+    this.bets = bets;
+    this.preparer = preparer;
+    this.runner = runner;
+  }
+
+  public SettlementExecutionRunner.BatchResult fanOut(AcceptedResult accepted) {
+    List<SettlementExecution> executions = new ArrayList<>();
+    for (var betId : bets.findResultActionableIdsByEvent(accepted.eventId())) {
+      preparer.prepare(betId, accepted).ifPresent(executions::add);
+    }
+    return runner.fanOut(List.copyOf(executions));
+  }
+}


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


## `feat(result): classify accepted candidate replays`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 1c65ddf..3ddb556 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -40,6 +40,9 @@ public class ResultCandidateIntake {
             accepted.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
     ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
     if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
+      if (recorded.state() == ResultCandidateState.ACCEPTED) {
+        return IntakeResult.ACCEPTED_REPLAY;
+      }
       return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
           ? IntakeResult.EXACT_REPLAY
           : IntakeResult.NO_CHANGE;
@@ -67,6 +70,7 @@ public class ResultCandidateIntake {
 
   public enum IntakeResult {
     EXACT_REPLAY,
+    ACCEPTED_REPLAY,
     NO_CHANGE,
     FIRST_ACCEPTED,
     AUTO_CORRECTION_ACCEPTED,


## `test(result): execute concurrent result claims`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresConcurrentResultClaimIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresConcurrentResultClaimIntegrationTest.java
new file mode 100644
index 0000000..17d1d75
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresConcurrentResultClaimIntegrationTest.java
@@ -0,0 +1,88 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.settlement.correction.ResultCandidateIntake;
+import com.sportsbook.settlement.result.AcceptedResult;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import com.sportsbook.settlement.result.MatchResultRecord;
+import com.sportsbook.settlement.result.ResultSettlementPreparer;
+import java.time.Instant;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.CyclicBarrier;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+
+class PostgresConcurrentResultClaimIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private ResultCandidateIntake intake;
+  @Autowired private AcceptedResultRepository acceptedResults;
+  @Autowired private ResultSettlementPreparer preparer;
+  @Autowired private BetRepository bets;
+
+  @Test
+  void serializesTwoCompletingEventsIntoOneImmutableAttempt() throws Exception {
+    UUID firstEvent = UUID.randomUUID();
+    UUID secondEvent = UUID.randomUUID();
+    UUID firstSelection = UUID.randomUUID();
+    UUID secondSelection = UUID.randomUUID();
+    var selections = new LinkedHashMap<UUID, UUID>();
+    selections.put(firstEvent, firstSelection);
+    selections.put(secondEvent, secondSelection);
+    PendingMultiple bet = insertPendingMultiple(selections);
+    accept(firstEvent, firstSelection);
+    accept(secondEvent, secondSelection);
+    AcceptedResult first = acceptedResults.findByEventId(firstEvent).orElseThrow();
+    AcceptedResult second = acceptedResults.findByEventId(secondEvent).orElseThrow();
+    CyclicBarrier start = new CyclicBarrier(2);
+    var workers = Executors.newFixedThreadPool(2);
+
+    try {
+      var firstClaim = workers.submit(() -> prepare(start, bet.betId(), first));
+      var secondClaim = workers.submit(() -> prepare(start, bet.betId(), second));
+
+      assertThat(java.util.List.of(firstClaim.get(), secondClaim.get()))
+          .filteredOn(java.util.Optional::isPresent)
+          .hasSize(1);
+    } finally {
+      workers.shutdownNow();
+      assertThat(workers.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
+    }
+
+    assertThat(
+            jdbc.queryForMap(
+                "select action,result,payout_amount,attempt_count from settlement_attempt "
+                    + "where bet_id=?",
+                bet.betId()))
+        .containsEntry("action", "SETTLE")
+        .containsEntry("result", "WON")
+        .containsEntry("payout_amount", 400L)
+        .containsEntry("attempt_count", 1);
+    assertThat(bets.findWithSelectionsById(bet.betId()).orElseThrow().selections())
+        .extracting(selection -> selection.sourceCandidateId())
+        .containsExactly(first.candidateId(), second.candidateId());
+  }
+
+  private void accept(UUID eventId, UUID selectionId) {
+    var result =
+        new MatchResultRecord(
+            eventId,
+            MatchOutcomeMode.COMPLETED,
+            Map.of(selectionId, SettlementResult.WON),
+            Instant.EPOCH,
+            Instant.now());
+    assertThat(intake.ingest(result)).isEqualTo(ResultCandidateIntake.IntakeResult.FIRST_ACCEPTED);
+  }
+
+  private java.util.Optional<?> prepare(CyclicBarrier start, UUID betId, AcceptedResult accepted)
+      throws Exception {
+    start.await(5, TimeUnit.SECONDS);
+    return preparer.prepare(betId, accepted);
+  }
+}


## `test(result): verify accepted result fanout`

diff --git a/src/test/java/com/sportsbook/settlement/result/ResultFanoutTest.java b/src/test/java/com/sportsbook/settlement/result/ResultFanoutTest.java
new file mode 100644
index 0000000..998d4d1
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/result/ResultFanoutTest.java
@@ -0,0 +1,61 @@
+package com.sportsbook.settlement.result;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.execution.SettlementAttempt;
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.execution.SettlementExecutionRunner;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.execution.SettlementMoneyPlan;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ResultFanoutTest {
+
+  @Test
+  void preparesEveryActionableBetBeforeExecutingTheClaimedBatch() {
+    BetRepository bets = mock(BetRepository.class);
+    ResultSettlementPreparer preparer = mock(ResultSettlementPreparer.class);
+    SettlementExecutionRunner runner = mock(SettlementExecutionRunner.class);
+    UUID eventId = UUID.randomUUID();
+    UUID first = UUID.randomUUID();
+    UUID partial = UUID.randomUUID();
+    AcceptedResult accepted =
+        new AcceptedResult(
+            eventId, UUID.randomUUID(), MatchOutcomeMode.VOIDED, Map.of(), Instant.EPOCH);
+    SettlementMoneyPlan money =
+        new SettlementMoneyPlan(
+            Money.krw(100), Money.krw(100), Money.krw(100), Money.krw(0), Money.krw(0));
+    SettlementAttempt attempt =
+        SettlementAttempt.resolved(
+            first,
+            eventId,
+            SettlementResult.VOID,
+            money,
+            SettlementLease.acquire(Instant.EPOCH, Duration.ofSeconds(30)),
+            Instant.EPOCH);
+    SettlementExecution execution = new SettlementExecution(attempt, UUID.randomUUID());
+    when(bets.findResultActionableIdsByEvent(eventId)).thenReturn(List.of(first, partial));
+    when(preparer.prepare(first, accepted)).thenReturn(Optional.of(execution));
+    when(preparer.prepare(partial, accepted)).thenReturn(Optional.empty());
+    var expected = new SettlementExecutionRunner.BatchResult(1, 0);
+    when(runner.fanOut(List.of(execution))).thenReturn(expected);
+
+    assertThat(new ResultFanout(bets, preparer, runner).fanOut(accepted)).isEqualTo(expected);
+
+    verify(preparer).prepare(first, accepted);
+    verify(preparer).prepare(partial, accepted);
+    verify(runner).fanOut(List.of(execution));
+  }
+}
