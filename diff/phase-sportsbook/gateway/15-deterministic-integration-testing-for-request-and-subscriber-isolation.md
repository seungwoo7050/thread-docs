# 동시 요청·구독자 격리를 위한 결정론적 통합 테스트

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


## `test(websocket): observe broker subscription registration`

diff --git a/src/test/java/com/sportsbook/gateway/ws/SubscriptionRegistrationProbe.java b/src/test/java/com/sportsbook/gateway/ws/SubscriptionRegistrationProbe.java
new file mode 100644
index 0000000..3451b30
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/SubscriptionRegistrationProbe.java
@@ -0,0 +1,92 @@
+package com.sportsbook.gateway.ws;
+
+import java.util.Set;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.concurrent.ConcurrentMap;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.messaging.Message;
+import org.springframework.messaging.MessageChannel;
+import org.springframework.messaging.MessageHandler;
+import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
+import org.springframework.messaging.simp.SimpMessageType;
+import org.springframework.messaging.simp.broker.SimpleBrokerMessageHandler;
+import org.springframework.messaging.simp.config.ChannelRegistration;
+import org.springframework.messaging.simp.config.MessageBrokerRegistry;
+import org.springframework.messaging.support.ExecutorChannelInterceptor;
+import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
+
+@TestConfiguration(proxyBeanMethods = false)
+final class SubscriptionRegistrationProbe
+    implements ExecutorChannelInterceptor, WebSocketMessageBrokerConfigurer {
+
+  private final ConcurrentMap<String, CompletableFuture<Void>> expectations =
+      new ConcurrentHashMap<>();
+  private final ConcurrentMap<String, Set<String>> sessionSubscriptions = new ConcurrentHashMap<>();
+
+  CompletableFuture<Void> expect(String subscriptionId) {
+    CompletableFuture<Void> expected = new CompletableFuture<>();
+    if (expectations.putIfAbsent(subscriptionId, expected) != null) {
+      throw new IllegalStateException("duplicate subscription expectation");
+    }
+    return expected;
+  }
+
+  void release(String subscriptionId) {
+    expectations.remove(subscriptionId);
+  }
+
+  @Override
+  public void configureClientInboundChannel(ChannelRegistration registration) {
+    registration.interceptors(this);
+  }
+
+  @Override
+  public void configureMessageBroker(MessageBrokerRegistry registry) {
+    registry.configureBrokerChannel().interceptors(this);
+  }
+
+  @Override
+  public void afterMessageHandled(
+      Message<?> message, MessageChannel channel, MessageHandler handler, Exception failure) {
+    if (!(handler instanceof SimpleBrokerMessageHandler broker)) {
+      return;
+    }
+    String sessionId = SimpMessageHeaderAccessor.getSessionId(message.getHeaders());
+    SimpMessageType type = SimpMessageHeaderAccessor.getMessageType(message.getHeaders());
+    if (type == SimpMessageType.DISCONNECT) {
+      clearSession(sessionId);
+      return;
+    }
+    if (failure != null || type != SimpMessageType.SUBSCRIBE || !broker.isRunning()) {
+      return;
+    }
+    String subscriptionId = SimpMessageHeaderAccessor.getSubscriptionId(message.getHeaders());
+    CompletableFuture<Void> expected = expectations.get(subscriptionId);
+    if (sessionId == null || expected == null) {
+      return;
+    }
+    sessionSubscriptions
+        .computeIfAbsent(sessionId, ignored -> ConcurrentHashMap.newKeySet())
+        .add(subscriptionId);
+    String destination = SimpMessageHeaderAccessor.getDestination(message.getHeaders());
+    if (destination != null
+        && broker.getDestinationPrefixes().stream().anyMatch(destination::startsWith)) {
+      expected.complete(null);
+    }
+  }
+
+  private void clearSession(String sessionId) {
+    Set<String> subscriptions = sessionId == null ? null : sessionSubscriptions.remove(sessionId);
+    if (subscriptions != null) {
+      subscriptions.forEach(this::failPendingExpectation);
+    }
+  }
+
+  private void failPendingExpectation(String subscriptionId) {
+    CompletableFuture<Void> expected = expectations.remove(subscriptionId);
+    if (expected != null) {
+      expected.completeExceptionally(new IllegalStateException("session disconnected"));
+    }
+  }
+}


## `test(websocket): await broker subscription registration`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java
index a4ac422..c61ea4d 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamFixture.java
@@ -6,13 +6,16 @@ import static java.util.concurrent.TimeUnit.SECONDS;
 import com.sportsbook.gateway.events.AvroTestSupport;
 import com.sportsbook.gateway.kafka.GatewayTopicProperties;
 import java.lang.reflect.Type;
+import java.util.UUID;
 import java.util.concurrent.BlockingQueue;
+import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.LinkedBlockingQueue;
 import org.apache.avro.specific.SpecificRecord;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
 import org.springframework.boot.test.web.server.LocalServerPort;
+import org.springframework.context.annotation.Import;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.kafka.test.context.EmbeddedKafka;
 import org.springframework.messaging.simp.stomp.StompFrameHandler;
@@ -45,11 +48,13 @@ import org.springframework.web.socket.messaging.WebSocketStompClient;
       "bet.resolution.revised.v1.DLT"
     },
     bootstrapServersProperty = "spring.kafka.bootstrap-servers")
+@Import(SubscriptionRegistrationProbe.class)
 abstract class WebSocketStreamFixture {
 
   @LocalServerPort protected int port;
   @Autowired protected KafkaTemplate<byte[], byte[]> kafka;
   @Autowired protected GatewayTopicProperties topics;
+  @Autowired protected SubscriptionRegistrationProbe registrations;
   @MockBean protected JwtDecoder jwtDecoder;
 
   protected StompSession connect(String path, StompHeaders headers) throws Exception {
@@ -66,18 +71,28 @@ abstract class WebSocketStreamFixture {
   protected BlockingQueue<String> subscribe(StompSession session, String destination)
       throws Exception {
     BlockingQueue<String> messages = new LinkedBlockingQueue<>();
-    session.subscribe(
-        destination,
-        new StompFrameHandler() {
-          public Type getPayloadType(StompHeaders headers) {
-            return byte[].class;
-          }
+    String subscriptionId = UUID.randomUUID().toString();
+    StompHeaders headers = new StompHeaders();
+    headers.setDestination(destination);
+    headers.setId(subscriptionId);
+    CompletableFuture<Void> registered = registrations.expect(subscriptionId);
+    try {
+      session.subscribe(
+          headers,
+          new StompFrameHandler() {
+            public Type getPayloadType(StompHeaders ignored) {
+              return byte[].class;
+            }
 
-          public void handleFrame(StompHeaders headers, Object payload) {
-            messages.add(new String((byte[]) payload, UTF_8));
-          }
-        });
-    return messages;
+            public void handleFrame(StompHeaders ignored, Object payload) {
+              messages.add(new String((byte[]) payload, UTF_8));
+            }
+          });
+      registered.get(5, SECONDS);
+      return messages;
+    } finally {
+      registrations.release(subscriptionId);
+    }
   }
 
   protected void publish(String topic, String key, SpecificRecord event) throws Exception {


## `test(routing): verify concurrent request isolation`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 1c820e3..655b214 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -23,10 +23,16 @@ import java.security.KeyPair;
 import java.security.KeyPairGenerator;
 import java.security.interfaces.RSAPrivateKey;
 import java.time.Instant;
+import java.util.ArrayList;
 import java.util.Base64;
 import java.util.Date;
 import java.util.List;
 import java.util.UUID;
+import java.util.concurrent.CyclicBarrier;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import java.util.concurrent.TimeUnit;
 import org.junit.jupiter.api.AfterAll;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
@@ -260,6 +266,62 @@ class GatewayRoutingIntegrationTest {
     assertThat(read(timeout.getBody(), "$.errorCode").toString()).isEqualTo("GATEWAY_TIMEOUT");
   }
 
+  @Test
+  void isolatesConcurrentIdentityIdempotencyAndTraceTuples() throws Exception {
+    int clients = 24;
+    DOWNSTREAM.stubFor(
+        post(urlPathEqualTo("/internal/v1/bets"))
+            .willReturn(aResponse().withStatus(201).withBody("{\"accepted\":true}")));
+    ExecutorService pool = Executors.newFixedThreadPool(clients);
+    CyclicBarrier start = new CyclicBarrier(clients);
+    List<Future<ResponseEntity<String>>> responses = new ArrayList<>();
+    try {
+      for (int index = 0; index < clients; index++) {
+        int request = index;
+        responses.add(
+            pool.submit(
+                () -> {
+                  start.await(5, TimeUnit.SECONDS);
+                  HttpHeaders headers = authenticated(new UUID(0, request + 1));
+                  headers.setContentType(MediaType.APPLICATION_JSON);
+                  headers.set("Idempotency-Key", "fixture-" + request);
+                  headers.set("traceparent", traceparent(request));
+                  headers.set("X-User-Id", new UUID(-1, request + 1).toString());
+                  headers.set("X-User-Roles", "ADMIN");
+                  headers.set("X-Internal-Service", "attacker");
+                  headers.set("X-Internal-Api-Key", "attacker-key");
+                  return http.exchange(
+                      "/api/v1/bets",
+                      HttpMethod.POST,
+                      new HttpEntity<>(requestBody(request), headers),
+                      String.class);
+                }));
+      }
+      for (Future<ResponseEntity<String>> response : responses) {
+        assertThat(response.get(30, TimeUnit.SECONDS).getStatusCode())
+            .isEqualTo(HttpStatus.CREATED);
+      }
+    } finally {
+      pool.shutdownNow();
+      assertThat(pool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
+    }
+
+    DOWNSTREAM.verify(clients, postRequestedFor(urlPathEqualTo("/internal/v1/bets")));
+    for (int index = 0; index < clients; index++) {
+      DOWNSTREAM.verify(
+          1,
+          postRequestedFor(urlPathEqualTo("/internal/v1/bets"))
+              .withHeader("X-User-Id", equalTo(new UUID(0, index + 1).toString()))
+              .withHeader("X-User-Roles", equalTo("USER"))
+              .withHeader("Idempotency-Key", equalTo("fixture-" + index))
+              .withHeader("traceparent", equalTo(traceparent(index)))
+              .withoutHeader(HttpHeaders.AUTHORIZATION)
+              .withoutHeader("X-Internal-Service")
+              .withoutHeader("X-Internal-Api-Key")
+              .withRequestBody(equalTo(requestBody(index))));
+    }
+  }
+
   @Test
   void rejectsUnsafeBettingBaseUris() {
     assertThatThrownBy(() -> bettingUri("ftp://betting.internal"))
@@ -317,4 +379,12 @@ class GatewayRoutingIntegrationTest {
   private static BettingDownstreamProperties bettingUri(String value) {
     return new BettingDownstreamProperties(URI.create(value));
   }
+
+  private static String requestBody(int index) {
+    return "{\"request\":" + index + "}";
+  }
+
+  private static String traceparent(int index) {
+    return String.format("00-%032x-%016x-01", index + 1, index + 1);
+  }
 }


## `test(realtime): verify concurrent subscriber isolation`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
index 9d5bf80..890acce 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
@@ -13,6 +13,8 @@ import com.sportsbook.protocol.event.OddsChanged;
 import com.sportsbook.protocol.event.SettlementResultAvro;
 import com.sportsbook.protocol.event.VoidReason;
 import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
 import java.util.UUID;
 import java.util.concurrent.BlockingQueue;
 import org.junit.jupiter.api.Test;
@@ -32,6 +34,76 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
   @Autowired SimpleBrokerMessageHandler broker;
   @Autowired KafkaListenerEndpointRegistry listeners;
 
+  @Test
+  void broadcastsOneOddsEventToEightSubscribedSessionsExactlyOnce() throws Exception {
+    OddsChanged event = oddsChanged(UUID.randomUUID().toString());
+    String destination = "/topic/odds/" + event.getEventId();
+    List<StompSession> sessions = new ArrayList<>();
+    List<BlockingQueue<String>> messages = new ArrayList<>();
+    try {
+      for (int index = 0; index < 8; index++) {
+        StompSession session = connect("/ws/v1/odds", new StompHeaders());
+        sessions.add(session);
+        messages.add(subscribe(session, destination));
+      }
+      awaitListener("gateway-odds-listener");
+
+      publish(topics.oddsChanged(), event.getEventId(), event);
+
+      String first = null;
+      for (BlockingQueue<String> subscriber : messages) {
+        String payload = subscriber.poll(5, SECONDS);
+        assertThat(payload)
+            .contains(event.getEventId(), event.getMarketId(), event.getSelectionId());
+        if (first == null) {
+          first = payload;
+        } else {
+          assertThat(payload).isEqualTo(first);
+        }
+        assertThat(subscriber.poll(1, SECONDS)).isNull();
+      }
+    } finally {
+      disconnect(sessions);
+    }
+  }
+
+  @Test
+  void isolatesEightOwnersDuringConcurrentBetFanOut() throws Exception {
+    List<StompSession> sessions = new ArrayList<>();
+    List<BlockingQueue<String>> messages = new ArrayList<>();
+    List<BetSettled> events = new ArrayList<>();
+    try {
+      for (int index = 0; index < 8; index++) {
+        String owner = UUID.randomUUID().toString();
+        BetSettled event = betSettled(owner);
+        StompSession session = connect("/ws/v1/bets", authHeaders(owner));
+        sessions.add(session);
+        messages.add(subscribe(session, "/user/queue/bets"));
+        events.add(event);
+      }
+      awaitListener("gateway-settled-listener");
+
+      for (BetSettled event : events) {
+        publish(topics.betSettled(), event.getEventId(), event);
+      }
+
+      List<String> deliveries = new ArrayList<>();
+      for (BlockingQueue<String> owner : messages) {
+        deliveries.add(owner.poll(5, SECONDS));
+      }
+      for (int index = 0; index < events.size(); index++) {
+        BetSettled event = events.get(index);
+        assertThat(deliveries.get(index))
+            .contains(event.getBetId(), event.getUserId(), "\"status\":\"SETTLED\"");
+      }
+      for (BlockingQueue<String> owner : messages) {
+        assertThat(owner.poll(1, SECONDS)).isNull();
+      }
+    } finally {
+      disconnect(sessions);
+    }
+  }
+
   @Test
   void broadcastsKafkaOddsToEverySubscribedSessionExactlyOnce() throws Exception {
     OddsChanged event = oddsChanged(UUID.randomUUID().toString());
@@ -176,7 +248,7 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
             Jwt.withTokenValue(userId)
                 .header("alg", "RS256")
                 .subject(userId)
-                .expiresAt(Instant.now().plusSeconds(60))
+                .expiresAt(Instant.now().plusSeconds(300))
                 .build());
     StompHeaders headers = new StompHeaders();
     headers.add(HttpHeaders.AUTHORIZATION, "Bearer " + userId);
@@ -239,4 +311,8 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
         .setChangedAt(Instant.parse("2026-08-21T00:00:00Z"))
         .build();
   }
+
+  private static void disconnect(List<StompSession> sessions) {
+    sessions.stream().filter(StompSession::isConnected).forEach(StompSession::disconnect);
+  }
 }
