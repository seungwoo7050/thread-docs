# 순서 역전 이벤트 캐치업

## `feat(result): catch up results before placement`

diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
index 447ffe2..7a35315 100644
--- a/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
@@ -1,13 +1,20 @@
 package com.sportsbook.settlement.readmodel;
 
 import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.BetSelection;
 import com.sportsbook.settlement.domain.EmbeddedMoney;
 import com.sportsbook.settlement.domain.SlipKind;
 import com.sportsbook.settlement.persistence.BetRepository;
+import com.sportsbook.settlement.result.MatchResultRecord;
+import com.sportsbook.settlement.result.MatchResultRepository;
 import java.time.Clock;
+import java.time.Instant;
+import java.util.LinkedHashMap;
 import java.util.List;
+import java.util.Map;
+import java.util.UUID;
 import org.springframework.stereotype.Service;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -23,16 +30,19 @@ public class BetReadModelWriter {
   private final BetRepository repository;
   private final BetPlacementValidator validator;
   private final BetPlacementFingerprinter fingerprinter;
+  private final MatchResultRepository results;
   private final Clock clock;
 
   public BetReadModelWriter(
       BetRepository repository,
       BetPlacementValidator validator,
       BetPlacementFingerprinter fingerprinter,
+      MatchResultRepository results,
       Clock clock) {
     this.repository = repository;
     this.validator = validator;
     this.fingerprinter = fingerprinter;
+    this.results = results;
     this.clock = clock;
   }
 
@@ -69,7 +79,8 @@ public class BetReadModelWriter {
       minimumWins = system.minWins();
       totalSelections = system.totalSelections();
     }
-    repository.save(
+    Instant now = clock.instant();
+    Bet bet =
         Bet.pending(
             placement.betId(),
             placement.userId(),
@@ -79,7 +90,25 @@ public class BetReadModelWriter {
             EmbeddedMoney.of(placement.unitStake()),
             placement.requestedAt(),
             selections,
-            clock.instant()));
+            now);
+    repository.save(bet);
+    placement.selections().stream()
+        .map(BetPlacement.Selection::eventId)
+        .distinct()
+        .forEach(eventId -> results.findById(eventId).ifPresent(result -> apply(bet, result, now)));
     return RecordResult.CREATED;
   }
+
+  private static void apply(Bet bet, MatchResultRecord result, Instant now) {
+    Map<UUID, SettlementResult> resolved = new LinkedHashMap<>();
+    bet.selections().stream()
+        .filter(selection -> selection.eventId().equals(result.eventId()))
+        .forEach(
+            selection ->
+                result
+                    .mode()
+                    .resolve(result.outcomes().get(selection.selectionId()))
+                    .ifPresent(outcome -> resolved.put(selection.selectionId(), outcome)));
+    bet.applySelectionSnapshot(result.eventId(), resolved, false, now);
+  }
 }


## `feat(lifecycle): catch up tombstones before results`

diff --git a/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
index 6c7e42e..2bb74cf 100644
--- a/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
+++ b/src/main/java/com/sportsbook/settlement/event/BetPlacedListener.java
@@ -1,6 +1,9 @@
 package com.sportsbook.settlement.event;
 
 import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.settlement.lifecycle.LifecycleFanout;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
+import com.sportsbook.settlement.readmodel.BetPlacement;
 import com.sportsbook.settlement.readmodel.BetReadModelWriter;
 import org.apache.kafka.clients.consumer.ConsumerRecord;
 import org.springframework.beans.factory.annotation.Autowired;
@@ -16,21 +19,34 @@ public class BetPlacedListener {
   private final StrictAvroDecoder decoder;
   private final KafkaUuidKeyValidator keys;
   private final BetPlacedMapper mapper;
+  private final LifecycleStore lifecycles;
+  private final LifecycleFanout lifecycleFanout;
 
   @Autowired
-  public BetPlacedListener(BetReadModelWriter writer) {
-    this(writer, new StrictAvroDecoder(), new KafkaUuidKeyValidator(), new BetPlacedMapper());
+  public BetPlacedListener(
+      BetReadModelWriter writer, LifecycleStore lifecycles, LifecycleFanout lifecycleFanout) {
+    this(
+        writer,
+        new StrictAvroDecoder(),
+        new KafkaUuidKeyValidator(),
+        new BetPlacedMapper(),
+        lifecycles,
+        lifecycleFanout);
   }
 
   BetPlacedListener(
       BetReadModelWriter writer,
       StrictAvroDecoder decoder,
       KafkaUuidKeyValidator keys,
-      BetPlacedMapper mapper) {
+      BetPlacedMapper mapper,
+      LifecycleStore lifecycles,
+      LifecycleFanout lifecycleFanout) {
     this.writer = writer;
     this.decoder = decoder;
     this.keys = keys;
     this.mapper = mapper;
+    this.lifecycles = lifecycles;
+    this.lifecycleFanout = lifecycleFanout;
   }
 
   @KafkaListener(
@@ -39,7 +55,13 @@ public class BetPlacedListener {
   public void receive(ConsumerRecord<byte[], byte[]> record, Acknowledgment acknowledgment) {
     BetPlacedRequested event = decoder.decode(record.value(), BetPlacedRequested.class);
     keys.requireMatching(record.key(), event.getUserId(), "userId");
-    writer.record(mapper.map(event));
+    BetPlacement placement = mapper.map(event);
+    writer.record(placement);
+    placement.selections().stream()
+        .map(BetPlacement.Selection::eventId)
+        .distinct()
+        .sorted()
+        .forEach(eventId -> lifecycles.findTombstone(eventId).ifPresent(lifecycleFanout::fanOut));
     acknowledgment.acknowledge();
   }
 }
diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
index c83c1a8..3efa5ae 100644
--- a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleStore.java
@@ -4,6 +4,9 @@ import static com.sportsbook.settlement.persistence.JdbcTimestamps.nullable;
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import com.sportsbook.protocol.event.EventLifecycleStatus;
+import java.nio.charset.StandardCharsets;
+import java.util.Optional;
+import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
 import org.springframework.transaction.annotation.Transactional;
@@ -57,6 +60,29 @@ public class LifecycleStore {
     return latched == 1 ? RecordResult.TERMINAL_LATCHED : RecordResult.TERMINAL_ALREADY_LATCHED;
   }
 
+  public Optional<LifecycleObservation> findTombstone(UUID eventId) {
+    return jdbc
+        .query(
+            """
+            select terminal_status, occurred_at, received_at, fingerprint
+            from event_lifecycle_tombstone where event_id = ?
+            """,
+            (result, rowNumber) -> {
+              String fingerprint = result.getString("fingerprint");
+              return new LifecycleObservation(
+                  UUID.nameUUIDFromBytes(fingerprint.getBytes(StandardCharsets.UTF_8)),
+                  eventId,
+                  EventLifecycleStatus.valueOf(result.getString("terminal_status")),
+                  result.getTimestamp("occurred_at").toInstant(),
+                  null,
+                  result.getTimestamp("received_at").toInstant(),
+                  fingerprint);
+            },
+            eventId)
+        .stream()
+        .findFirst();
+  }
+
   private static boolean terminal(EventLifecycleStatus status) {
     return status == EventLifecycleStatus.CANCELLED || status == EventLifecycleStatus.POSTPONED;
   }


## `refactor(placement): isolate immutable placement writes`

diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
index 7a35315..123eccd 100644
--- a/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
@@ -1,20 +1,14 @@
 package com.sportsbook.settlement.readmodel;
 
 import com.sportsbook.protocol.domain.BetSlipType;
-import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.BetSelection;
 import com.sportsbook.settlement.domain.EmbeddedMoney;
 import com.sportsbook.settlement.domain.SlipKind;
 import com.sportsbook.settlement.persistence.BetRepository;
-import com.sportsbook.settlement.result.MatchResultRecord;
-import com.sportsbook.settlement.result.MatchResultRepository;
 import java.time.Clock;
 import java.time.Instant;
-import java.util.LinkedHashMap;
 import java.util.List;
-import java.util.Map;
-import java.util.UUID;
 import org.springframework.stereotype.Service;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -30,19 +24,16 @@ public class BetReadModelWriter {
   private final BetRepository repository;
   private final BetPlacementValidator validator;
   private final BetPlacementFingerprinter fingerprinter;
-  private final MatchResultRepository results;
   private final Clock clock;
 
   public BetReadModelWriter(
       BetRepository repository,
       BetPlacementValidator validator,
       BetPlacementFingerprinter fingerprinter,
-      MatchResultRepository results,
       Clock clock) {
     this.repository = repository;
     this.validator = validator;
     this.fingerprinter = fingerprinter;
-    this.results = results;
     this.clock = clock;
   }
 
@@ -92,23 +83,6 @@ public class BetReadModelWriter {
             selections,
             now);
     repository.save(bet);
-    placement.selections().stream()
-        .map(BetPlacement.Selection::eventId)
-        .distinct()
-        .forEach(eventId -> results.findById(eventId).ifPresent(result -> apply(bet, result, now)));
     return RecordResult.CREATED;
   }
-
-  private static void apply(Bet bet, MatchResultRecord result, Instant now) {
-    Map<UUID, SettlementResult> resolved = new LinkedHashMap<>();
-    bet.selections().stream()
-        .filter(selection -> selection.eventId().equals(result.eventId()))
-        .forEach(
-            selection ->
-                result
-                    .mode()
-                    .resolve(result.outcomes().get(selection.selectionId()))
-                    .ifPresent(outcome -> resolved.put(selection.selectionId(), outcome)));
-    bet.applySelectionSnapshot(result.eventId(), resolved, false, now);
-  }
 }


## `test(placement): verify isolated placement writes`

diff --git a/src/test/java/com/sportsbook/settlement/readmodel/BetReadModelWriterTest.java b/src/test/java/com/sportsbook/settlement/readmodel/BetReadModelWriterTest.java
index 466c6e0..838df03 100644
--- a/src/test/java/com/sportsbook/settlement/readmodel/BetReadModelWriterTest.java
+++ b/src/test/java/com/sportsbook/settlement/readmodel/BetReadModelWriterTest.java
@@ -15,14 +15,10 @@ import com.sportsbook.settlement.domain.BetSelection;
 import com.sportsbook.settlement.domain.EmbeddedMoney;
 import com.sportsbook.settlement.domain.SlipKind;
 import com.sportsbook.settlement.persistence.BetRepository;
-import com.sportsbook.settlement.result.MatchOutcomeMode;
-import com.sportsbook.settlement.result.MatchResultRecord;
-import com.sportsbook.settlement.result.MatchResultRepository;
 import java.time.Clock;
 import java.time.Instant;
 import java.time.ZoneOffset;
 import java.util.List;
-import java.util.Map;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
@@ -31,13 +27,11 @@ import org.mockito.ArgumentCaptor;
 class BetReadModelWriterTest {
 
   private final BetRepository repository = mock(BetRepository.class);
-  private final MatchResultRepository results = mock(MatchResultRepository.class);
   private final BetReadModelWriter writer =
       new BetReadModelWriter(
           repository,
           new BetPlacementValidator(),
           new BetPlacementFingerprinter(),
-          results,
           Clock.fixed(Instant.parse("2026-01-02T00:00:00Z"), ZoneOffset.UTC));
 
   @Test
@@ -73,29 +67,6 @@ class BetReadModelWriterTest {
     verify(repository, never()).save(org.mockito.ArgumentMatchers.any());
   }
 
-  @Test
-  void appliesAResultThatArrivedBeforePlacement() {
-    BetPlacement placement = placement(100);
-    BetPlacement.Selection selection = placement.selections().get(0);
-    when(repository.findWithSelectionsById(placement.betId())).thenReturn(Optional.empty());
-    when(results.findById(selection.eventId()))
-        .thenReturn(
-            Optional.of(
-                new MatchResultRecord(
-                    selection.eventId(),
-                    MatchOutcomeMode.VOIDED,
-                    Map.of(),
-                    Instant.EPOCH,
-                    Instant.EPOCH)));
-
-    writer.record(placement);
-
-    ArgumentCaptor<Bet> saved = ArgumentCaptor.forClass(Bet.class);
-    verify(repository).save(saved.capture());
-    assertThat(saved.getValue().selections().get(0).outcome())
-        .isEqualTo(com.sportsbook.protocol.domain.SettlementResult.VOID);
-  }
-
   private static BetPlacement placement(long amount) {
     BetPlacement.Selection selection =
         new BetPlacement.Selection(


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


## `test(lifecycle): verify tombstone first catchup`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
index 1218ff5..0b31331 100644
--- a/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
@@ -6,18 +6,22 @@ import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.reset;
-import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
 import com.sportsbook.protocol.event.BetPlacedRequested;
 import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.event.Money;
 import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.settlement.lifecycle.LifecycleFanout;
+import com.sportsbook.settlement.lifecycle.LifecycleObservation;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
 import com.sportsbook.settlement.readmodel.BetReadModelWriter;
 import java.io.ByteArrayOutputStream;
 import java.time.Instant;
 import java.util.List;
+import java.util.Optional;
 import java.util.UUID;
 import org.apache.avro.io.EncoderFactory;
 import org.apache.avro.specific.SpecificDatumWriter;
@@ -29,7 +33,16 @@ import org.springframework.kafka.support.Acknowledgment;
 class BetPlacedListenerTest {
 
   private final BetReadModelWriter writer = mock(BetReadModelWriter.class);
-  private final BetPlacedListener listener = new BetPlacedListener(writer);
+  private final LifecycleStore lifecycles = mock(LifecycleStore.class);
+  private final LifecycleFanout lifecycleFanout = mock(LifecycleFanout.class);
+  private final BetPlacedListener listener =
+      new BetPlacedListener(
+          writer,
+          new StrictAvroDecoder(),
+          new KafkaUuidKeyValidator(),
+          new BetPlacedMapper(),
+          lifecycles,
+          lifecycleFanout);
   private final Acknowledgment acknowledgment = mock(Acknowledgment.class);
 
   @Test
@@ -60,6 +73,27 @@ class BetPlacedListenerTest {
     verifyNoInteractions(writer, acknowledgment);
   }
 
+  @Test
+  void fansOutStoredTerminalTombstoneBeforeAcknowledgment() {
+    BetPlacedRequested event = event();
+    UUID eventId = UUID.fromString(event.getSelections().get(0).getEventId().toString());
+    LifecycleObservation tombstone =
+        LifecycleObservation.observe(
+            eventId,
+            EventLifecycleStatus.POSTPONED,
+            Instant.EPOCH,
+            Instant.EPOCH.plusSeconds(3600),
+            Instant.EPOCH.plusSeconds(1));
+    when(lifecycles.findTombstone(eventId)).thenReturn(Optional.of(tombstone));
+
+    listener.receive(record(event, event.getUserId().toString()), acknowledgment);
+
+    InOrder caughtUp = inOrder(writer, lifecycleFanout, acknowledgment);
+    caughtUp.verify(writer).record(any());
+    caughtUp.verify(lifecycleFanout).fanOut(tombstone);
+    caughtUp.verify(acknowledgment).acknowledge();
+  }
+
   private static ConsumerRecord<byte[], byte[]> record(BetPlacedRequested event, String key) {
     return new ConsumerRecord<>("bet.placed.v1", 0, 0, key.getBytes(UTF_8), encode(event));
   }


