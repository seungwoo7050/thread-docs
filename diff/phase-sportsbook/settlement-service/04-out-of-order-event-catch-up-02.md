## `test(placement): verify tombstone first catchup passes`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedCatchupPassOrderTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedCatchupPassOrderTest.java
new file mode 100644
index 0000000..ca9749a
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedCatchupPassOrderTest.java
@@ -0,0 +1,91 @@
+package com.sportsbook.settlement.event;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.settlement.lifecycle.LifecycleFanout;
+import com.sportsbook.settlement.lifecycle.LifecycleObservation;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
+import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import com.sportsbook.settlement.result.AcceptedResult;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import com.sportsbook.settlement.result.ResultFanout;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedCatchupPassOrderTest {
+
+  @Test
+  void completesEveryLifecyclePassBeforeStartingResultCatchup() {
+    UUID first = UUID.fromString("00000000-0000-0000-0000-000000000001");
+    UUID second = UUID.fromString("00000000-0000-0000-0000-000000000002");
+    var event =
+        BetPlacedRequested.newBuilder(BetPlacedListenerTest.event())
+            .setSlipType(BetSlipTypeTag.MULTIPLE)
+            .setSelections(List.of(selection(second), selection(first)))
+            .build();
+    BetReadModelWriter writer = mock(BetReadModelWriter.class);
+    LifecycleStore lifecycles = mock(LifecycleStore.class);
+    LifecycleFanout lifecycleFanout = mock(LifecycleFanout.class);
+    AcceptedResultRepository acceptedResults = mock(AcceptedResultRepository.class);
+    ResultFanout resultFanout = mock(ResultFanout.class);
+    Acknowledgment acknowledgment = mock(Acknowledgment.class);
+    LifecycleObservation firstTombstone = tombstone(first);
+    LifecycleObservation secondTombstone = tombstone(second);
+    AcceptedResult firstResult = accepted(first);
+    AcceptedResult secondResult = accepted(second);
+    when(lifecycles.findTombstone(first)).thenReturn(Optional.of(firstTombstone));
+    when(lifecycles.findTombstone(second)).thenReturn(Optional.of(secondTombstone));
+    when(acceptedResults.findByEventId(first)).thenReturn(Optional.of(firstResult));
+    when(acceptedResults.findByEventId(second)).thenReturn(Optional.of(secondResult));
+    var listener =
+        new BetPlacedListener(writer, lifecycles, lifecycleFanout, acceptedResults, resultFanout);
+
+    listener.receive(
+        BetPlacedListenerTest.record(event, event.getUserId().toString()), acknowledgment);
+
+    var order =
+        inOrder(writer, lifecycles, lifecycleFanout, acceptedResults, resultFanout, acknowledgment);
+    order.verify(writer).record(any());
+    order.verify(lifecycles).findTombstone(first);
+    order.verify(lifecycleFanout).fanOut(firstTombstone);
+    order.verify(lifecycles).findTombstone(second);
+    order.verify(lifecycleFanout).fanOut(secondTombstone);
+    order.verify(acceptedResults).findByEventId(first);
+    order.verify(resultFanout).fanOut(firstResult);
+    order.verify(acceptedResults).findByEventId(second);
+    order.verify(resultFanout).fanOut(secondResult);
+    order.verify(acknowledgment).acknowledge();
+  }
+
+  private static RequestedSelection selection(UUID eventId) {
+    return RequestedSelection.newBuilder()
+        .setEventId(eventId.toString())
+        .setMarketId(UUID.randomUUID().toString())
+        .setSelectionId(UUID.randomUUID().toString())
+        .setOddsAtSubmission("2.0000")
+        .build();
+  }
+
+  private static LifecycleObservation tombstone(UUID eventId) {
+    return LifecycleObservation.observe(
+        eventId, EventLifecycleStatus.CANCELLED, Instant.EPOCH, null, Instant.EPOCH);
+  }
+
+  private static AcceptedResult accepted(UUID eventId) {
+    return new AcceptedResult(
+        eventId, UUID.randomUUID(), MatchOutcomeMode.VOIDED, Map.of(), Instant.EPOCH);
+  }
+}


## `test(placement): verify catchup ack order`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
index 0b31331..482d958 100644
--- a/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
@@ -18,6 +18,8 @@ import com.sportsbook.settlement.lifecycle.LifecycleFanout;
 import com.sportsbook.settlement.lifecycle.LifecycleObservation;
 import com.sportsbook.settlement.lifecycle.LifecycleStore;
 import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.ResultFanout;
 import java.io.ByteArrayOutputStream;
 import java.time.Instant;
 import java.util.List;
@@ -35,6 +37,8 @@ class BetPlacedListenerTest {
   private final BetReadModelWriter writer = mock(BetReadModelWriter.class);
   private final LifecycleStore lifecycles = mock(LifecycleStore.class);
   private final LifecycleFanout lifecycleFanout = mock(LifecycleFanout.class);
+  private final AcceptedResultRepository acceptedResults = mock(AcceptedResultRepository.class);
+  private final ResultFanout resultFanout = mock(ResultFanout.class);
   private final BetPlacedListener listener =
       new BetPlacedListener(
           writer,
@@ -42,7 +46,9 @@ class BetPlacedListenerTest {
           new KafkaUuidKeyValidator(),
           new BetPlacedMapper(),
           lifecycles,
-          lifecycleFanout);
+          lifecycleFanout,
+          acceptedResults,
+          resultFanout);
   private final Acknowledgment acknowledgment = mock(Acknowledgment.class);
 
   @Test
@@ -88,17 +94,20 @@ class BetPlacedListenerTest {
 
     listener.receive(record(event, event.getUserId().toString()), acknowledgment);
 
-    InOrder caughtUp = inOrder(writer, lifecycleFanout, acknowledgment);
+    InOrder caughtUp =
+        inOrder(writer, lifecycles, lifecycleFanout, acceptedResults, acknowledgment);
     caughtUp.verify(writer).record(any());
+    caughtUp.verify(lifecycles).findTombstone(eventId);
     caughtUp.verify(lifecycleFanout).fanOut(tombstone);
+    caughtUp.verify(acceptedResults).findByEventId(eventId);
     caughtUp.verify(acknowledgment).acknowledge();
   }
 
-  private static ConsumerRecord<byte[], byte[]> record(BetPlacedRequested event, String key) {
+  static ConsumerRecord<byte[], byte[]> record(BetPlacedRequested event, String key) {
     return new ConsumerRecord<>("bet.placed.v1", 0, 0, key.getBytes(UTF_8), encode(event));
   }
 
-  private static BetPlacedRequested event() {
+  static BetPlacedRequested event() {
     RequestedSelection selected =
         RequestedSelection.newBuilder()
             .setEventId(UUID.randomUUID().toString())


## `test(placement): verify result catchup redelivery`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedResultReplayTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedResultReplayTest.java
new file mode 100644
index 0000000..74a1d3a
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedResultReplayTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.settlement.event;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.settlement.lifecycle.LifecycleFanout;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
+import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import com.sportsbook.settlement.result.AcceptedResult;
+import com.sportsbook.settlement.result.AcceptedResultRepository;
+import com.sportsbook.settlement.result.MatchOutcomeMode;
+import com.sportsbook.settlement.result.ResultFanout;
+import java.time.Instant;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedResultReplayTest {
+
+  @Test
+  void exactPlacementReplayRestartsAcceptedResultCatchupBeforeAck() {
+    BetReadModelWriter writer = mock(BetReadModelWriter.class);
+    LifecycleStore lifecycles = mock(LifecycleStore.class);
+    LifecycleFanout lifecycleFanout = mock(LifecycleFanout.class);
+    AcceptedResultRepository acceptedResults = mock(AcceptedResultRepository.class);
+    ResultFanout resultFanout = mock(ResultFanout.class);
+    Acknowledgment acknowledgment = mock(Acknowledgment.class);
+    var event = BetPlacedListenerTest.event();
+    UUID eventId = UUID.fromString(event.getSelections().get(0).getEventId().toString());
+    AcceptedResult accepted =
+        new AcceptedResult(
+            eventId, UUID.randomUUID(), MatchOutcomeMode.VOIDED, Map.of(), Instant.EPOCH);
+    when(writer.record(any())).thenReturn(BetReadModelWriter.RecordResult.EXACT_REPLAY);
+    when(acceptedResults.findByEventId(eventId)).thenReturn(Optional.of(accepted));
+    var listener =
+        new BetPlacedListener(writer, lifecycles, lifecycleFanout, acceptedResults, resultFanout);
+
+    listener.receive(
+        BetPlacedListenerTest.record(event, event.getUserId().toString()), acknowledgment);
+
+    var order = inOrder(writer, acceptedResults, resultFanout, acknowledgment);
+    order.verify(writer).record(any());
+    order.verify(acceptedResults).findByEventId(eventId);
+    order.verify(resultFanout).fanOut(accepted);
+    order.verify(acknowledgment).acknowledge();
+  }
+}


## `test(result): execute placement before result settlement`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresPlacementBeforeResultIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresPlacementBeforeResultIntegrationTest.java
new file mode 100644
index 0000000..74f1adc
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresPlacementBeforeResultIntegrationTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.times;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.client.WalletClient;
+import com.sportsbook.settlement.client.WalletCreditPurpose;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.event.BaseResultEvents;
+import com.sportsbook.settlement.event.BetPlacedListener;
+import com.sportsbook.settlement.event.MatchResultListener;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.kafka.support.Acknowledgment;
+
+class PostgresPlacementBeforeResultIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private MatchResultListener results;
+  @Autowired private BetPlacedListener placements;
+  @Autowired private BetRepository bets;
+  @MockBean private WalletClient wallet;
+
+  @Test
+  void settlesPlacementWhenItsFirstResultAndExactReplayArrive() {
+    UUID eventId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    var placement = BaseResultEvents.single(betId, userId, eventId, UUID.randomUUID());
+    var result = BaseResultEvents.voided(eventId);
+    Acknowledgment placementAck = mock(Acknowledgment.class);
+    Acknowledgment resultAck = mock(Acknowledgment.class);
+    when(wallet.credit(
+            "settle:refund:" + betId, userId, Money.krw(100), WalletCreditPurpose.RETURNED_STAKE))
+        .thenReturn(UUID.randomUUID());
+
+    placements.receive(BaseResultEvents.placementRecord(placement), placementAck);
+    assertThat(bets.findById(betId).orElseThrow().status()).isEqualTo(SettlementStatus.PENDING);
+    results.receive(BaseResultEvents.resultRecord(result), resultAck);
+    results.receive(BaseResultEvents.resultRecord(result), resultAck);
+
+    var settled = bets.findWithSelectionsById(betId).orElseThrow();
+    assertThat(settled.status()).isEqualTo(SettlementStatus.SETTLED);
+    assertThat(settled.selections().get(0).sourceCandidateId()).isNotNull();
+    assertThat(
+            jdbc.queryForObject(
+                "select count(*) from outbox_event where schema_name='BetSettled'", Integer.class))
+        .isEqualTo(1);
+    verify(wallet, times(1))
+        .credit(
+            "settle:refund:" + betId, userId, Money.krw(100), WalletCreditPurpose.RETURNED_STAKE);
+    verify(placementAck).acknowledge();
+    verify(resultAck, times(2)).acknowledge();
+  }
+}


## `test(result): execute result before placement catchup`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresResultBeforePlacementIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresResultBeforePlacementIntegrationTest.java
new file mode 100644
index 0000000..29a8f18
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresResultBeforePlacementIntegrationTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.times;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.client.WalletClient;
+import com.sportsbook.settlement.client.WalletCreditPurpose;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.event.BaseResultEvents;
+import com.sportsbook.settlement.event.BetPlacedListener;
+import com.sportsbook.settlement.event.MatchResultListener;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.kafka.support.Acknowledgment;
+
+class PostgresResultBeforePlacementIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private MatchResultListener results;
+  @Autowired private BetPlacedListener placements;
+  @Autowired private BetRepository bets;
+  @MockBean private WalletClient wallet;
+
+  @Test
+  void catchesUpVoidedResultIntoOneSettledEventAndOutboxRecord() {
+    UUID eventId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    var result = BaseResultEvents.voided(eventId);
+    var placement = BaseResultEvents.single(betId, userId, eventId, selectionId);
+    Acknowledgment resultAck = mock(Acknowledgment.class);
+    Acknowledgment placementAck = mock(Acknowledgment.class);
+    when(wallet.credit(
+            "settle:refund:" + betId, userId, Money.krw(100), WalletCreditPurpose.RETURNED_STAKE))
+        .thenReturn(UUID.randomUUID());
+
+    results.receive(BaseResultEvents.resultRecord(result), resultAck);
+    placements.receive(BaseResultEvents.placementRecord(placement), placementAck);
+    placements.receive(BaseResultEvents.placementRecord(placement), placementAck);
+
+    var settled = bets.findWithSelectionsById(betId).orElseThrow();
+    UUID acceptedCandidate =
+        jdbc.queryForObject(
+            "select accepted_candidate_id from match_result where event_id=?", UUID.class, eventId);
+    assertThat(settled.status()).isEqualTo(SettlementStatus.SETTLED);
+    assertThat(settled.result()).isEqualTo(SettlementResult.VOID);
+    assertThat(settled.payout()).isEqualTo(Money.krw(100));
+    assertThat(settled.selections().get(0).sourceCandidateId()).isEqualTo(acceptedCandidate);
+    assertThat(jdbc.queryForObject("select count(*) from settlement_attempt", Integer.class))
+        .isZero();
+    assertThat(
+            jdbc.queryForObject(
+                "select count(*) from outbox_event where schema_name='BetSettled'", Integer.class))
+        .isEqualTo(1);
+    assertThat(
+            jdbc.queryForObject(
+                "select count(*) from outbox_event where schema_name='BetVoided'", Integer.class))
+        .isZero();
+    verify(wallet, times(1))
+        .credit(
+            "settle:refund:" + betId, userId, Money.krw(100), WalletCreditPurpose.RETURNED_STAKE);
+    verify(resultAck).acknowledge();
+    verify(placementAck, times(2)).acknowledge();
+  }
+}
