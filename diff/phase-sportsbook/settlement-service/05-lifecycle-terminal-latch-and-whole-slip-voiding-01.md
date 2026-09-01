# 이벤트 생명주기 터미널 래치와 전체 슬립 무효화

## `feat(lifecycle): capture semantic observations`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFingerprinter.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFingerprinter.java
new file mode 100644
index 0000000..0e3956c
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFingerprinter.java
@@ -0,0 +1,34 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.time.Instant;
+import java.util.HexFormat;
+import java.util.UUID;
+
+public final class LifecycleFingerprinter {
+
+  public String fingerprint(
+      UUID eventId, EventLifecycleStatus status, Instant occurredAt, Instant scheduledStartAt) {
+    try {
+      MessageDigest digest = MessageDigest.getInstance("SHA-256");
+      add(digest, "event-lifecycle-v1");
+      add(digest, eventId.toString());
+      add(digest, status.name());
+      add(digest, occurredAt.toString());
+      add(digest, scheduledStartAt == null ? "" : scheduledStartAt.toString());
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
diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleObservation.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleObservation.java
new file mode 100644
index 0000000..4e499a3
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleObservation.java
@@ -0,0 +1,40 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.settlement.infrastructure.id.UuidV7;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record LifecycleObservation(
+    UUID observationId,
+    UUID eventId,
+    EventLifecycleStatus status,
+    Instant occurredAt,
+    Instant scheduledStartAt,
+    Instant receivedAt,
+    String fingerprint) {
+
+  public LifecycleObservation {
+    Objects.requireNonNull(observationId, "observationId");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(status, "status");
+    Objects.requireNonNull(occurredAt, "occurredAt");
+    Objects.requireNonNull(receivedAt, "receivedAt");
+    if (fingerprint == null || !fingerprint.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("Lifecycle fingerprint must be lowercase SHA-256");
+    }
+  }
+
+  public static LifecycleObservation observe(
+      UUID eventId,
+      EventLifecycleStatus status,
+      Instant occurredAt,
+      Instant scheduledStartAt,
+      Instant receivedAt) {
+    String fingerprint =
+        new LifecycleFingerprinter().fingerprint(eventId, status, occurredAt, scheduledStartAt);
+    return new LifecycleObservation(
+        UuidV7.generate(), eventId, status, occurredAt, scheduledStartAt, receivedAt, fingerprint);
+  }
+}


## `feat(lifecycle): order observations causally`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleOrder.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleOrder.java
new file mode 100644
index 0000000..023d192
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleOrder.java
@@ -0,0 +1,28 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.util.Comparator;
+import java.util.List;
+import java.util.Optional;
+
+public final class LifecycleOrder {
+
+  private static final Comparator<LifecycleObservation> CAUSAL =
+      Comparator.comparing(LifecycleObservation::occurredAt)
+          .thenComparingInt(observation -> rank(observation.status()))
+          .thenComparing(LifecycleObservation::fingerprint);
+
+  public Optional<LifecycleObservation> latest(List<LifecycleObservation> observations) {
+    return observations.stream().max(CAUSAL);
+  }
+
+  private static int rank(EventLifecycleStatus status) {
+    return switch (status) {
+      case SCHEDULED -> 0;
+      case IN_PLAY -> 1;
+      case FINISHED -> 2;
+      case POSTPONED -> 3;
+      case CANCELLED -> 4;
+    };
+  }
+}


## `feat(lifecycle): add durable terminal tombstones`

diff --git a/src/main/resources/db/migration/V6__event_lifecycle.sql b/src/main/resources/db/migration/V6__event_lifecycle.sql
new file mode 100644
index 0000000..ff429e3
--- /dev/null
+++ b/src/main/resources/db/migration/V6__event_lifecycle.sql
@@ -0,0 +1,30 @@
+-- Durable observations preserve causal evidence; tombstones permanently latch terminal events.
+
+CREATE TABLE event_lifecycle_observation (
+    observation_id    UUID                     PRIMARY KEY,
+    event_id           UUID                     NOT NULL,
+    status             VARCHAR(16)              NOT NULL,
+    occurred_at        TIMESTAMP WITH TIME ZONE NOT NULL,
+    scheduled_start_at TIMESTAMP WITH TIME ZONE,
+    received_at        TIMESTAMP WITH TIME ZONE NOT NULL,
+    fingerprint        CHAR(64)                 NOT NULL,
+    CONSTRAINT uq_event_lifecycle_fingerprint UNIQUE (event_id, fingerprint),
+    CONSTRAINT ck_event_lifecycle_status CHECK (
+        status IN ('SCHEDULED', 'IN_PLAY', 'FINISHED', 'CANCELLED', 'POSTPONED'))
+);
+
+CREATE INDEX ix_event_lifecycle_order
+    ON event_lifecycle_observation (event_id, occurred_at, fingerprint);
+
+CREATE TABLE event_lifecycle_tombstone (
+    event_id        UUID                     PRIMARY KEY,
+    terminal_status VARCHAR(16)              NOT NULL,
+    occurred_at     TIMESTAMP WITH TIME ZONE NOT NULL,
+    received_at     TIMESTAMP WITH TIME ZONE NOT NULL,
+    fingerprint     CHAR(64)                 NOT NULL,
+    CONSTRAINT ck_event_lifecycle_terminal CHECK (
+        terminal_status IN ('CANCELLED', 'POSTPONED'))
+);
+
+COMMENT ON TABLE event_lifecycle_tombstone IS
+    'Non-expiring first terminal latch; late nonterminal observations cannot revive an event.';


## `feat(lifecycle): latch first terminal observation`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
new file mode 100644
index 0000000..c83c1a8
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
@@ -0,0 +1,70 @@
+package com.sportsbook.settlement.lifecycle;
+
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.nullable;
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class LifecycleStore {
+
+  private final JdbcTemplate jdbc;
+
+  public LifecycleStore(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional
+  public RecordResult record(LifecycleObservation observation) {
+    int inserted =
+        jdbc.update(
+            """
+            insert into event_lifecycle_observation (
+                observation_id, event_id, status, occurred_at,
+                scheduled_start_at, received_at, fingerprint)
+            values (?, ?, ?, ?, ?, ?, ?)
+            on conflict (event_id, fingerprint) do nothing
+            """,
+            observation.observationId(),
+            observation.eventId(),
+            observation.status().name(),
+            required(observation.occurredAt()),
+            nullable(observation.scheduledStartAt()),
+            required(observation.receivedAt()),
+            observation.fingerprint());
+    if (inserted == 0) {
+      return RecordResult.EXACT_REPLAY;
+    }
+    if (!terminal(observation.status())) {
+      return RecordResult.OBSERVED;
+    }
+    int latched =
+        jdbc.update(
+            """
+            insert into event_lifecycle_tombstone (
+                event_id, terminal_status, occurred_at, received_at, fingerprint)
+            values (?, ?, ?, ?, ?)
+            on conflict (event_id) do nothing
+            """,
+            observation.eventId(),
+            observation.status().name(),
+            required(observation.occurredAt()),
+            required(observation.receivedAt()),
+            observation.fingerprint());
+    return latched == 1 ? RecordResult.TERMINAL_LATCHED : RecordResult.TERMINAL_ALREADY_LATCHED;
+  }
+
+  private static boolean terminal(EventLifecycleStatus status) {
+    return status == EventLifecycleStatus.CANCELLED || status == EventLifecycleStatus.POSTPONED;
+  }
+
+  public enum RecordResult {
+    EXACT_REPLAY,
+    OBSERVED,
+    TERMINAL_LATCHED,
+    TERMINAL_ALREADY_LATCHED
+  }
+}


## `feat(lifecycle): consume raw lifecycle events`

diff --git a/src/main/java/com/sportsbook/settlement/event/EventLifecycleListener.java b/src/main/java/com/sportsbook/settlement/event/EventLifecycleListener.java
new file mode 100644
index 0000000..8e67502
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/EventLifecycleListener.java
@@ -0,0 +1,60 @@
+package com.sportsbook.settlement.event;
+
+import com.sportsbook.protocol.event.EventLifecycle;
+import com.sportsbook.settlement.lifecycle.LifecycleFanout;
+import com.sportsbook.settlement.lifecycle.LifecycleObservation;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
+import java.time.Clock;
+import java.util.Objects;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.support.Acknowledgment;
+import org.springframework.stereotype.Component;
+
+@Component
+public class EventLifecycleListener {
+
+  private final LifecycleStore store;
+  private final LifecycleFanout fanout;
+  private final Clock clock;
+  private final StrictAvroDecoder decoder;
+  private final KafkaUuidKeyValidator keys;
+
+  @Autowired
+  public EventLifecycleListener(LifecycleStore store, LifecycleFanout fanout, Clock clock) {
+    this(store, fanout, clock, new StrictAvroDecoder(), new KafkaUuidKeyValidator());
+  }
+
+  EventLifecycleListener(
+      LifecycleStore store,
+      LifecycleFanout fanout,
+      Clock clock,
+      StrictAvroDecoder decoder,
+      KafkaUuidKeyValidator keys) {
+    this.store = store;
+    this.fanout = fanout;
+    this.clock = clock;
+    this.decoder = decoder;
+    this.keys = keys;
+  }
+
+  @KafkaListener(
+      topics = "${settlement.topics.event-lifecycle:event.lifecycle}",
+      groupId = "settlement-service")
+  public void receive(ConsumerRecord<byte[], byte[]> record, Acknowledgment acknowledgment) {
+    EventLifecycle event = decoder.decode(record.value(), EventLifecycle.class);
+    UUID eventId = keys.requireMatching(record.key(), event.getEventId(), "eventId");
+    LifecycleObservation observation =
+        LifecycleObservation.observe(
+            eventId,
+            Objects.requireNonNull(event.getStatus(), "status"),
+            Objects.requireNonNull(event.getOccurredAt(), "occurredAt"),
+            event.getScheduledStartAt(),
+            clock.instant());
+    store.record(observation);
+    store.findTombstone(eventId).ifPresent(fanout::fanOut);
+    acknowledgment.acknowledge();
+  }
+}


## `feat(settlement): prepare whole slip void claims`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java
index ca717e6..34ee56c 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java
@@ -1,8 +1,10 @@
 package com.sportsbook.settlement.execution;
 
 import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
 import java.time.Instant;
 import java.util.Objects;
+import java.util.Set;
 import java.util.UUID;
 
 public record SettlementAttempt(
@@ -18,6 +20,9 @@ public record SettlementAttempt(
     Instant createdAt,
     Instant updatedAt) {
 
+  private static final Set<String> WHOLE_SLIP_VOID_REASONS =
+      Set.of("EVENT_CANCELLED", "EVENT_POSTPONED", "ADMIN_VOID");
+
   public SettlementAttempt {
     Objects.requireNonNull(betId, "betId");
     Objects.requireNonNull(action, "action");
@@ -28,7 +33,7 @@ public record SettlementAttempt(
     Objects.requireNonNull(updatedAt, "updatedAt");
     boolean resolved = action == Action.SETTLE && result != null && voidReason == null;
     boolean voided =
-        action == Action.VOID && result == null && voidReason != null && !voidReason.isBlank();
+        action == Action.VOID && result == null && WHOLE_SLIP_VOID_REASONS.contains(voidReason);
     if ((!resolved && !voided) || attemptCount < 1) {
       throw new IllegalArgumentException("Invalid settlement attempt action");
     }
@@ -45,6 +50,21 @@ public record SettlementAttempt(
         betId, Action.SETTLE, eventId, result, null, money, lease, 1, null, now, now);
   }
 
+  public static SettlementAttempt wholeSlipVoid(
+      UUID betId,
+      UUID eventId,
+      String reason,
+      Money totalExposure,
+      SettlementLease lease,
+      Instant now) {
+    Objects.requireNonNull(totalExposure, "totalExposure");
+    Money zero = Money.zero(totalExposure.currency());
+    SettlementMoneyPlan refund =
+        new SettlementMoneyPlan(totalExposure, totalExposure, totalExposure, zero, zero);
+    return new SettlementAttempt(
+        betId, Action.VOID, eventId, null, reason, refund, lease, 1, null, now, now);
+  }
+
   public enum Action {
     SETTLE,
     VOID


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


## `feat(lifecycle): scan actionable tombstones`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
index 3efa5ae..86229a2 100644
--- a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
@@ -5,6 +5,7 @@ import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import java.nio.charset.StandardCharsets;
+import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
@@ -83,6 +84,36 @@ public class LifecycleStore {
         .findFirst();
   }
 
+  public List<LifecycleObservation> findActionableTombstones(int limit) {
+    if (limit < 1 || limit > 1000) {
+      throw new IllegalArgumentException("Lifecycle scan limit must be between 1 and 1000");
+    }
+    return jdbc.query(
+        """
+        select t.event_id, t.terminal_status, t.occurred_at, t.received_at, t.fingerprint
+        from event_lifecycle_tombstone t
+        where exists (
+            select 1 from bet_selection s join bet b on b.bet_id = s.bet_id
+            where s.event_id = t.event_id and b.status = 'PENDING'
+              and not exists (
+                  select 1 from settlement_attempt a where a.bet_id = b.bet_id))
+        order by t.received_at, t.event_id limit ?
+        """,
+        (result, rowNumber) -> {
+          UUID eventId = result.getObject("event_id", UUID.class);
+          String fingerprint = result.getString("fingerprint");
+          return new LifecycleObservation(
+              UUID.nameUUIDFromBytes(fingerprint.getBytes(StandardCharsets.UTF_8)),
+              eventId,
+              EventLifecycleStatus.valueOf(result.getString("terminal_status")),
+              result.getTimestamp("occurred_at").toInstant(),
+              null,
+              result.getTimestamp("received_at").toInstant(),
+              fingerprint);
+        },
+        limit);
+  }
+
   private static boolean terminal(EventLifecycleStatus status) {
     return status == EventLifecycleStatus.CANCELLED || status == EventLifecycleStatus.POSTPONED;
   }
diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java
new file mode 100644
index 0000000..e845b48
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleTombstoneScanner.java
@@ -0,0 +1,31 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class LifecycleTombstoneScanner {
+
+  private final LifecycleStore store;
+  private final LifecycleFanout fanout;
+  private final SettlementRuntimeProperties runtime;
+
+  public LifecycleTombstoneScanner(
+      LifecycleStore store, LifecycleFanout fanout, SettlementRuntimeProperties runtime) {
+    this.store = store;
+    this.fanout = fanout;
+    this.runtime = runtime;
+  }
+
+  @Scheduled(
+      fixedDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      scheduler = SettlementWorkerConfiguration.LIFECYCLE)
+  public int scan() {
+    var tombstones = store.findActionableTombstones(runtime.batchSize());
+    tombstones.forEach(fanout::fanOut);
+    return tombstones.size();
+  }
+}


