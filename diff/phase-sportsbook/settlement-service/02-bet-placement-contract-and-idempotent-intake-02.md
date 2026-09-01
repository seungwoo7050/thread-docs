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


## `test(placement): verify listener key and ack boundaries`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
new file mode 100644
index 0000000..1218ff5
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedListenerTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.settlement.event;
+
+import static java.nio.charset.StandardCharsets.UTF_8;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.reset;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.settlement.readmodel.BetReadModelWriter;
+import java.io.ByteArrayOutputStream;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+import org.springframework.kafka.support.Acknowledgment;
+
+class BetPlacedListenerTest {
+
+  private final BetReadModelWriter writer = mock(BetReadModelWriter.class);
+  private final BetPlacedListener listener = new BetPlacedListener(writer);
+  private final Acknowledgment acknowledgment = mock(Acknowledgment.class);
+
+  @Test
+  void acknowledgesOnlyAfterTheWriterReturns() {
+    BetPlacedRequested event = event();
+    ConsumerRecord<byte[], byte[]> record = record(event, event.getUserId().toString());
+
+    listener.receive(record, acknowledgment);
+
+    InOrder committed = inOrder(writer, acknowledgment);
+    committed.verify(writer).record(any());
+    committed.verify(acknowledgment).acknowledge();
+
+    reset(writer, acknowledgment);
+    when(writer.record(any())).thenThrow(new IllegalStateException("database unavailable"));
+    assertThatThrownBy(() -> listener.receive(record, acknowledgment))
+        .isInstanceOf(IllegalStateException.class);
+    verifyNoInteractions(acknowledgment);
+  }
+
+  @Test
+  void rejectsAMismatchedRawUserKeyBeforePersistence() {
+    BetPlacedRequested event = event();
+
+    assertThatThrownBy(
+            () -> listener.receive(record(event, UUID.randomUUID().toString()), acknowledgment))
+        .isInstanceOf(IllegalArgumentException.class);
+    verifyNoInteractions(writer, acknowledgment);
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(BetPlacedRequested event, String key) {
+    return new ConsumerRecord<>("bet.placed.v1", 0, 0, key.getBytes(UTF_8), encode(event));
+  }
+
+  private static BetPlacedRequested event() {
+    RequestedSelection selected =
+        RequestedSelection.newBuilder()
+            .setEventId(UUID.randomUUID().toString())
+            .setMarketId(UUID.randomUUID().toString())
+            .setSelectionId(UUID.randomUUID().toString())
+            .setOddsAtSubmission("2.0000")
+            .build();
+    return BetPlacedRequested.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(UUID.randomUUID().toString())
+        .setSlipType(BetSlipTypeTag.SINGLE)
+        .setSystemMinWins(null)
+        .setSystemTotalSelections(null)
+        .setSelections(List.of(selected))
+        .setStake(Money.newBuilder().setAmount(100).setCurrency("KRW").build())
+        .setIdempotencyKey("placement-key")
+        .setRequestedAt(Instant.EPOCH)
+        .build();
+  }
+
+  private static byte[] encode(BetPlacedRequested event) {
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      var encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<BetPlacedRequested>(event.getSchema()).write(event, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (Exception exception) {
+      throw new AssertionError(exception);
+    }
+  }
+}
