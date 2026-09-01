# 기본 정산 처리 파이프라인

## `feat(placement): consume accepted placement events`

diff --git a/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
new file mode 100644
index 0000000..6c7e42e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
@@ -0,0 +1,45 @@
+package com.sportsbook.settlement.event;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.support.Acknowledgment;
+import org.springframework.stereotype.Component;
+
+/** Records accepted placements and acknowledges only after the database transaction returns. */
+@Component
+public class BetPlacedListener {
+
+  private final BetReadModelWriter writer;
+  private final StrictAvroDecoder decoder;
+  private final KafkaUuidKeyValidator keys;
+  private final BetPlacedMapper mapper;
+
+  @Autowired
+  public BetPlacedListener(BetReadModelWriter writer) {
+    this(writer, new StrictAvroDecoder(), new KafkaUuidKeyValidator(), new BetPlacedMapper());
+  }
+
+  BetPlacedListener(
+      BetReadModelWriter writer,
+      StrictAvroDecoder decoder,
+      KafkaUuidKeyValidator keys,
+      BetPlacedMapper mapper) {
+    this.writer = writer;
+    this.decoder = decoder;
+    this.keys = keys;
+    this.mapper = mapper;
+  }
+
+  @KafkaListener(
+      topics = "${settlement.topics.bet-placed:bet.placed.v1}",
+      groupId = "settlement-service")
+  public void receive(ConsumerRecord<byte[], byte[]> record, Acknowledgment acknowledgment) {
+    BetPlacedRequested event = decoder.decode(record.value(), BetPlacedRequested.class);
+    keys.requireMatching(record.key(), event.getUserId(), "userId");
+    writer.record(mapper.map(event));
+    acknowledgment.acknowledge();
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
new file mode 100644
index 0000000..a14aa8e
--- /dev/null
+++ b/src/main/resources/application.yml
@@ -0,0 +1,17 @@
+spring:
+  kafka:
+    consumer:
+      group-id: settlement-service
+      enable-auto-commit: false
+      key-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
+      value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
+      properties:
+        allow.auto.create.topics: false
+    listener:
+      ack-mode: manual_immediate
+
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,info,prometheus


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


## `feat(lifecycle): fan out terminal void claims`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java
new file mode 100644
index 0000000..04c75b7
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java
@@ -0,0 +1,86 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.execution.SettlementAttempt;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.execution.SettlementExecutionRunner;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Clock;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import org.springframework.stereotype.Service;
+
+@Service
+public class LifecycleFanout {
+
+  private final BetRepository bets;
+  private final SettlementAttemptRepository attempts;
+  private final SettlementExecutionRunner runner;
+  private final SettlementRuntimeProperties runtime;
+  private final Clock clock;
+
+  public LifecycleFanout(
+      BetRepository bets,
+      SettlementAttemptRepository attempts,
+      SettlementExecutionRunner runner,
+      SettlementRuntimeProperties runtime,
+      Clock clock) {
+    this.bets = bets;
+    this.attempts = attempts;
+    this.runner = runner;
+    this.runtime = runtime;
+    this.clock = clock;
+  }
+
+  public SettlementExecutionRunner.BatchResult fanOut(LifecycleObservation tombstone) {
+    String reason = reason(tombstone.status());
+    Instant now = clock.instant();
+    List<SettlementExecution> executions = new ArrayList<>();
+    for (var betId : bets.findPendingIdsByEvent(tombstone.eventId())) {
+      Bet bet = bets.findWithSelectionsById(betId).orElseThrow();
+      SettlementAttempt attempt =
+          SettlementAttempt.wholeSlipVoid(
+              bet.betId(),
+              tombstone.eventId(),
+              reason,
+              totalExposure(bet),
+              SettlementLease.acquire(now, runtime.leaseDuration()),
+              now);
+      if (attempts.claimPending(attempt)) {
+        executions.add(new SettlementExecution(attempt, bet.userId()));
+      }
+    }
+    return runner.fanOut(executions);
+  }
+
+  private static String reason(EventLifecycleStatus status) {
+    return switch (status) {
+      case CANCELLED -> "EVENT_CANCELLED";
+      case POSTPONED -> "EVENT_POSTPONED";
+      default -> throw new IllegalArgumentException("Lifecycle fanout requires terminal status");
+    };
+  }
+
+  private static Money totalExposure(Bet bet) {
+    long lines = 1;
+    if (bet.slipType() instanceof BetSlipType.System system) {
+      lines = combinations(system.totalSelections(), system.minWins());
+    }
+    return bet.stake().multiply(lines);
+  }
+
+  private static long combinations(int n, int k) {
+    long result = 1;
+    for (int factor = 1; factor <= k; factor++) {
+      result = Math.multiplyExact(result, n - k + factor) / factor;
+    }
+    return result;
+  }
+}


## `feat(placement): catch up accepted results`

diff --git a/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
index 2bb74cf..d32bc6a 100644
--- a/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
+++ b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
@@ -5,6 +5,10 @@ import com.sportsbook.settlement.lifecycle.LifecycleFanout;
 import com.sportsbook.settlement.lifecycle.LifecycleStore;
 import com.sportsbook.settlement.readmodel.BetPlacement;
 import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.ResultFanout;
+import java.util.List;
+import java.util.UUID;
 import org.apache.kafka.clients.consumer.ConsumerRecord;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.kafka.annotation.KafkaListener;
@@ -21,17 +25,25 @@ public class BetPlacedListener {
   private final BetPlacedMapper mapper;
   private final LifecycleStore lifecycles;
   private final LifecycleFanout lifecycleFanout;
+  private final AcceptedResultRepository acceptedResults;
+  private final ResultFanout resultFanout;
 
   @Autowired
   public BetPlacedListener(
-      BetReadModelWriter writer, LifecycleStore lifecycles, LifecycleFanout lifecycleFanout) {
+      BetReadModelWriter writer,
+      LifecycleStore lifecycles,
+      LifecycleFanout lifecycleFanout,
+      AcceptedResultRepository acceptedResults,
+      ResultFanout resultFanout) {
     this(
         writer,
         new StrictAvroDecoder(),
         new KafkaUuidKeyValidator(),
         new BetPlacedMapper(),
         lifecycles,
-        lifecycleFanout);
+        lifecycleFanout,
+        acceptedResults,
+        resultFanout);
   }
 
   BetPlacedListener(
@@ -40,13 +52,17 @@ public class BetPlacedListener {
       KafkaUuidKeyValidator keys,
       BetPlacedMapper mapper,
       LifecycleStore lifecycles,
-      LifecycleFanout lifecycleFanout) {
+      LifecycleFanout lifecycleFanout,
+      AcceptedResultRepository acceptedResults,
+      ResultFanout resultFanout) {
     this.writer = writer;
     this.decoder = decoder;
     this.keys = keys;
     this.mapper = mapper;
     this.lifecycles = lifecycles;
     this.lifecycleFanout = lifecycleFanout;
+    this.acceptedResults = acceptedResults;
+    this.resultFanout = resultFanout;
   }
 
   @KafkaListener(
@@ -57,11 +73,16 @@ public class BetPlacedListener {
     keys.requireMatching(record.key(), event.getUserId(), "userId");
     BetPlacement placement = mapper.map(event);
     writer.record(placement);
-    placement.selections().stream()
-        .map(BetPlacement.Selection::eventId)
-        .distinct()
-        .sorted()
-        .forEach(eventId -> lifecycles.findTombstone(eventId).ifPresent(lifecycleFanout::fanOut));
+    List<UUID> eventIds =
+        placement.selections().stream()
+            .map(BetPlacement.Selection::eventId)
+            .distinct()
+            .sorted()
+            .toList();
+    eventIds.forEach(
+        eventId -> lifecycles.findTombstone(eventId).ifPresent(lifecycleFanout::fanOut));
+    eventIds.forEach(
+        eventId -> acceptedResults.findByEventId(eventId).ifPresent(resultFanout::fanOut));
     acknowledgment.acknowledge();
   }
 }


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


## `feat(resolver): classify base settlement outcomes`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java
new file mode 100644
index 0000000..6bb8eb7
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementOutcome.java
@@ -0,0 +1,18 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+
+/** Full base resolution snapshot stored and emitted for a bet. */
+public record SettlementOutcome(
+    SettlementResult result, Money payout, int survivingLines, int totalLines) {
+
+  public SettlementOutcome {
+    Objects.requireNonNull(result, "result");
+    Objects.requireNonNull(payout, "payout");
+    if (survivingLines < 0 || totalLines < 1 || survivingLines > totalLines) {
+      throw new IllegalArgumentException("Invalid settlement outcome line counts");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java b/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java
new file mode 100644
index 0000000..08c743e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/SettlementResolver.java
@@ -0,0 +1,36 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.util.List;
+
+/** Resolves base slip result and payout from authoritative selection outcomes. */
+public final class SettlementResolver {
+
+  private final SettlementLineFactory lines = new SettlementLineFactory();
+  private final SettlementPayoutCalculator payouts = new SettlementPayoutCalculator();
+
+  public SettlementOutcome resolve(
+      BetSlipType slipType, List<ResolvedSelection> selections, Money unitStake) {
+    PayoutCalculation payout = payouts.calculate(lines.lines(slipType, selections), unitStake);
+    return new SettlementOutcome(
+        classify(selections, payout),
+        payout.payout(),
+        payout.survivingLines(),
+        payout.totalLines());
+  }
+
+  private static SettlementResult classify(
+      List<ResolvedSelection> selections, PayoutCalculation payout) {
+    if (payout.payout().isZero()) {
+      return SettlementResult.LOST;
+    }
+    if (selections.stream().anyMatch(ResolvedSelection::wins)) {
+      return SettlementResult.WON;
+    }
+    boolean allVoid =
+        selections.stream().allMatch(selection -> selection.outcome() == SettlementResult.VOID);
+    return allVoid ? SettlementResult.VOID : SettlementResult.PUSH;
+  }
+}


## `feat(resolver): split wallet settlement movements`

diff --git a/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java
new file mode 100644
index 0000000..3c61142
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlan.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+
+/** Split between locked stake, house pool, and forfeited exposure for one resolution. */
+public record WalletMovementPlan(
+    Money returnedStake, Money housePayout, Money forfeitedStake, Money totalExposure) {
+
+  public WalletMovementPlan {
+    Objects.requireNonNull(returnedStake, "returnedStake");
+    Objects.requireNonNull(housePayout, "housePayout");
+    Objects.requireNonNull(forfeitedStake, "forfeitedStake");
+    Objects.requireNonNull(totalExposure, "totalExposure");
+    if (returnedStake.isNegative()
+        || housePayout.isNegative()
+        || forfeitedStake.isNegative()
+        || !returnedStake.add(forfeitedStake).equals(totalExposure)) {
+      throw new IllegalArgumentException("Invalid wallet movement split");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java
new file mode 100644
index 0000000..ada9ee2
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/resolver/WalletMovementPlanner.java
@@ -0,0 +1,21 @@
+package com.sportsbook.settlement.resolver;
+
+import com.sportsbook.protocol.value.Money;
+
+/** Separates unit-stake exposure from returned stake and house-funded profit. */
+public final class WalletMovementPlanner {
+
+  public WalletMovementPlan plan(Money unitStake, SettlementOutcome outcome) {
+    if (unitStake == null || unitStake.amount() <= 0 || outcome == null) {
+      throw new IllegalArgumentException("Wallet movement planning requires a positive unit stake");
+    }
+    Money totalExposure = unitStake.multiply(outcome.totalLines());
+    Money returnedStake = unitStake.multiply(outcome.survivingLines());
+    Money forfeitedStake = totalExposure.subtract(returnedStake);
+    Money housePayout = outcome.payout().subtract(returnedStake);
+    if (housePayout.isNegative()) {
+      throw new IllegalArgumentException("Payout cannot be lower than surviving returned stake");
+    }
+    return new WalletMovementPlan(returnedStake, housePayout, forfeitedStake, totalExposure);
+  }
+}


