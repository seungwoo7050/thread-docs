# 이벤트 범위 공개 배당 STOMP 팬아웃

## `feat(websocket): publish odds updates`

diff --git a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
new file mode 100644
index 0000000..b43e087
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
@@ -0,0 +1,22 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.protocol.event.OddsChanged;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.messaging.simp.SimpMessagingTemplate;
+import org.springframework.stereotype.Component;
+
+/** Hands validated Kafka events to the local STOMP broker. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class GatewayPushPublisher {
+
+  private final SimpMessagingTemplate messaging;
+
+  public GatewayPushPublisher(SimpMessagingTemplate messaging) {
+    this.messaging = messaging;
+  }
+
+  public void publishOdds(OddsChanged event) {
+    messaging.convertAndSend("/topic/odds/" + event.getEventId(), OddsUpdate.from(event));
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java b/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java
new file mode 100644
index 0000000..6ae506d
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java
@@ -0,0 +1,31 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.gateway.events.GatewayEventContract;
+import com.sportsbook.protocol.event.OddsChanged;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+/** Publishes validated odds events to their public event stream. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class OddsStreamListener {
+
+  private final GatewayPushPublisher publisher;
+
+  public OddsStreamListener(GatewayPushPublisher publisher) {
+    this.publisher = publisher;
+  }
+
+  @KafkaListener(
+      id = "gateway-odds-listener",
+      topics = "${gateway.topics.odds-changed}",
+      groupId = "gateway-odds",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onOddsChanged(ConsumerRecord<byte[], byte[]> record) {
+    OddsChanged event = GatewayEventContract.oddsChanged(record);
+    publisher.publishOdds(event);
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java b/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java
new file mode 100644
index 0000000..e3765e5
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java
@@ -0,0 +1,24 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.protocol.event.OddsChanged;
+import java.time.Instant;
+
+/** Public odds projection delivered on an event-specific STOMP topic. */
+public record OddsUpdate(
+    String eventId,
+    String marketId,
+    String selectionId,
+    String previousOdds,
+    String newOdds,
+    Instant changedAt) {
+
+  static OddsUpdate from(OddsChanged event) {
+    return new OddsUpdate(
+        event.getEventId(),
+        event.getMarketId(),
+        event.getSelectionId(),
+        event.getPreviousOdds(),
+        event.getNewOdds(),
+        event.getChangedAt());
+  }
+}


## `test(websocket): establish realtime delivery fixture`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java
new file mode 100644
index 0000000..a4ac422
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java
@@ -0,0 +1,86 @@
+package com.sportsbook.gateway.ws;
+
+import static java.nio.charset.StandardCharsets.UTF_8;
+import static java.util.concurrent.TimeUnit.SECONDS;
+
+import com.sportsbook.gateway.events.AvroTestSupport;
+import com.sportsbook.gateway.kafka.GatewayTopicProperties;
+import java.lang.reflect.Type;
+import java.util.concurrent.BlockingQueue;
+import java.util.concurrent.LinkedBlockingQueue;
+import org.apache.avro.specific.SpecificRecord;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.web.server.LocalServerPort;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.messaging.simp.stomp.StompFrameHandler;
+import org.springframework.messaging.simp.stomp.StompHeaders;
+import org.springframework.messaging.simp.stomp.StompSession;
+import org.springframework.messaging.simp.stomp.StompSessionHandlerAdapter;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.web.socket.WebSocketHttpHeaders;
+import org.springframework.web.socket.client.standard.StandardWebSocketClient;
+import org.springframework.web.socket.messaging.WebSocketStompClient;
+
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
+    properties = {
+      "gateway.ratelimit.enabled=false",
+      "gateway.downstream.wallet.api-key=fixture-wallet-key-32-characters-long",
+      "spring.kafka.consumer.auto-offset-reset=earliest",
+      "spring.kafka.listener.auto-startup=true"
+    })
+@EmbeddedKafka(
+    partitions = 1,
+    topics = {
+      "odds.changed",
+      "odds.changed.DLT",
+      "bet.settled.v1",
+      "bet.settled.v1.DLT",
+      "bet.voided.v1",
+      "bet.voided.v1.DLT",
+      "bet.resolution.revised.v1",
+      "bet.resolution.revised.v1.DLT"
+    },
+    bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+abstract class WebSocketStreamFixture {
+
+  @LocalServerPort protected int port;
+  @Autowired protected KafkaTemplate<byte[], byte[]> kafka;
+  @Autowired protected GatewayTopicProperties topics;
+  @MockBean protected JwtDecoder jwtDecoder;
+
+  protected StompSession connect(String path, StompHeaders headers) throws Exception {
+    WebSocketStompClient client = new WebSocketStompClient(new StandardWebSocketClient());
+    return client
+        .connectAsync(
+            "ws://localhost:" + port + path,
+            new WebSocketHttpHeaders(),
+            headers,
+            new StompSessionHandlerAdapter() {})
+        .get(5, SECONDS);
+  }
+
+  protected BlockingQueue<String> subscribe(StompSession session, String destination)
+      throws Exception {
+    BlockingQueue<String> messages = new LinkedBlockingQueue<>();
+    session.subscribe(
+        destination,
+        new StompFrameHandler() {
+          public Type getPayloadType(StompHeaders headers) {
+            return byte[].class;
+          }
+
+          public void handleFrame(StompHeaders headers, Object payload) {
+            messages.add(new String((byte[]) payload, UTF_8));
+          }
+        });
+    return messages;
+  }
+
+  protected void publish(String topic, String key, SpecificRecord event) throws Exception {
+    kafka.send(topic, key.getBytes(UTF_8), AvroTestSupport.encode(event)).get(5, SECONDS);
+  }
+}


## `test(websocket): verify odds fan-out`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
new file mode 100644
index 0000000..e40ccc9
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
@@ -0,0 +1,83 @@
+package com.sportsbook.gateway.ws;
+
+import static java.util.concurrent.TimeUnit.SECONDS;
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.awaitility.Awaitility.await;
+
+import com.sportsbook.protocol.event.OddsChanged;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.concurrent.BlockingQueue;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
+import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
+import org.springframework.messaging.simp.SimpMessageType;
+import org.springframework.messaging.simp.broker.SimpleBrokerMessageHandler;
+import org.springframework.messaging.simp.stomp.StompHeaders;
+import org.springframework.messaging.simp.stomp.StompSession;
+import org.springframework.messaging.support.MessageBuilder;
+
+class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
+
+  @Autowired SimpleBrokerMessageHandler broker;
+  @Autowired KafkaListenerEndpointRegistry listeners;
+
+  @Test
+  void broadcastsKafkaOddsToEverySubscribedSessionExactlyOnce() throws Exception {
+    OddsChanged event = oddsChanged(UUID.randomUUID().toString());
+    StompSession first = connect("/ws/v1/odds", new StompHeaders());
+    StompSession second = connect("/ws/v1/odds", new StompHeaders());
+    try {
+      String destination = "/topic/odds/" + event.getEventId();
+      BlockingQueue<String> firstMessages = subscribe(first, destination);
+      BlockingQueue<String> secondMessages = subscribe(second, destination);
+      await()
+          .atMost(5, SECONDS)
+          .until(
+              () ->
+                  broker
+                          .getSubscriptionRegistry()
+                          .findSubscriptions(
+                              MessageBuilder.withPayload(new byte[0])
+                                  .setHeader(
+                                      SimpMessageHeaderAccessor.MESSAGE_TYPE_HEADER,
+                                      SimpMessageType.MESSAGE)
+                                  .setHeader(
+                                      SimpMessageHeaderAccessor.DESTINATION_HEADER, destination)
+                                  .build())
+                          .size()
+                      == 2);
+      await()
+          .atMost(5, SECONDS)
+          .until(
+              () ->
+                  !listeners
+                      .getListenerContainer("gateway-odds-listener")
+                      .getAssignedPartitions()
+                      .isEmpty());
+
+      publish(topics.oddsChanged(), event.getEventId(), event);
+
+      String payload = firstMessages.poll(5, SECONDS);
+      assertThat(payload).contains(event.getEventId(), event.getMarketId(), "1.8500", "1.9000");
+      assertThat(secondMessages.poll(5, SECONDS)).isEqualTo(payload);
+      assertThat(firstMessages.poll(1, SECONDS)).isNull();
+      assertThat(secondMessages.poll(1, SECONDS)).isNull();
+    } finally {
+      first.disconnect();
+      second.disconnect();
+    }
+  }
+
+  protected static OddsChanged oddsChanged(String eventId) {
+    return OddsChanged.newBuilder()
+        .setEventId(eventId)
+        .setMarketId(UUID.randomUUID().toString())
+        .setSelectionId(UUID.randomUUID().toString())
+        .setPreviousOdds("1.8500")
+        .setNewOdds("1.9000")
+        .setChangedAt(Instant.parse("2026-08-21T00:00:00Z"))
+        .build();
+  }
+}
