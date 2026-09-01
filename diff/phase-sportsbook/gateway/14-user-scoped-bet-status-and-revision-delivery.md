# 사용자 범위 베팅 상태와 정정 이벤트 전달

## `feat(websocket): project terminal bet updates`

diff --git a/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java b/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java
new file mode 100644
index 0000000..96c9c68
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java
@@ -0,0 +1,54 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import com.sportsbook.protocol.event.Money;
+import java.time.Instant;
+
+/** Private projection of a terminal bet state. */
+public record BetStatusUpdate(
+    String betId,
+    String userId,
+    String eventId,
+    String status,
+    String result,
+    MoneyView amount,
+    String reason,
+    String revisionId,
+    Long revisionNumber,
+    Instant updatedAt) {
+
+  public record MoneyView(long amount, String currency) {
+    static MoneyView from(Money money) {
+      return new MoneyView(money.getAmount(), money.getCurrency());
+    }
+  }
+
+  static BetStatusUpdate settled(BetSettled event) {
+    return new BetStatusUpdate(
+        event.getBetId(),
+        event.getUserId(),
+        event.getEventId(),
+        "SETTLED",
+        event.getResult().name(),
+        MoneyView.from(event.getPayout()),
+        null,
+        null,
+        0L,
+        event.getSettledAt());
+  }
+
+  static BetStatusUpdate voided(BetVoided event) {
+    return new BetStatusUpdate(
+        event.getBetId(),
+        event.getUserId(),
+        event.getEventId(),
+        "VOIDED",
+        null,
+        MoneyView.from(event.getRefund()),
+        event.getReason().name(),
+        null,
+        null,
+        event.getVoidedAt());
+  }
+}


## `feat(websocket): publish terminal bet updates`

diff --git a/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
new file mode 100644
index 0000000..2cad337
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
@@ -0,0 +1,43 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.gateway.events.GatewayEventContract;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+/** Publishes validated terminal bet events to their owning user. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class BetStatusStreamListener {
+
+  private final GatewayPushPublisher publisher;
+
+  public BetStatusStreamListener(GatewayPushPublisher publisher) {
+    this.publisher = publisher;
+  }
+
+  @KafkaListener(
+      id = "gateway-settled-listener",
+      topics = "${gateway.topics.bet-settled}",
+      groupId = "gateway-bets",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onBetSettled(ConsumerRecord<byte[], byte[]> record) {
+    BetSettled event = GatewayEventContract.betSettled(record);
+    publisher.publishBet(event.getUserId(), BetStatusUpdate.settled(event));
+  }
+
+  @KafkaListener(
+      id = "gateway-voided-listener",
+      topics = "${gateway.topics.bet-voided}",
+      groupId = "gateway-bets",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onBetVoided(ConsumerRecord<byte[], byte[]> record) {
+    BetVoided event = GatewayEventContract.betVoided(record);
+    publisher.publishBet(event.getUserId(), BetStatusUpdate.voided(event));
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
index b43e087..f2794c7 100644
--- a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
+++ b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
@@ -19,4 +19,8 @@ public class GatewayPushPublisher {
   public void publishOdds(OddsChanged event) {
     messaging.convertAndSend("/topic/odds/" + event.getEventId(), OddsUpdate.from(event));
   }
+
+  public void publishBet(String userId, BetStatusUpdate update) {
+    messaging.convertAndSendToUser(userId, "/queue/bets", update);
+  }
 }


## `test(websocket): verify settled bet fan-out`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
index e40ccc9..4a2ffd8 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
@@ -3,13 +3,18 @@ package com.sportsbook.gateway.ws;
 import static java.util.concurrent.TimeUnit.SECONDS;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.awaitility.Awaitility.await;
+import static org.mockito.Mockito.when;
 
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.Money;
 import com.sportsbook.protocol.event.OddsChanged;
+import com.sportsbook.protocol.event.SettlementResultAvro;
 import java.time.Instant;
 import java.util.UUID;
 import java.util.concurrent.BlockingQueue;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.http.HttpHeaders;
 import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
 import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
 import org.springframework.messaging.simp.SimpMessageType;
@@ -17,6 +22,7 @@ import org.springframework.messaging.simp.broker.SimpleBrokerMessageHandler;
 import org.springframework.messaging.simp.stomp.StompHeaders;
 import org.springframework.messaging.simp.stomp.StompSession;
 import org.springframework.messaging.support.MessageBuilder;
+import org.springframework.security.oauth2.jwt.Jwt;
 
 class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
 
@@ -70,6 +76,67 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
     }
   }
 
+  @Test
+  void deliversSettledBetOnlyToOwningUser() throws Exception {
+    String owner = UUID.randomUUID().toString();
+    String other = UUID.randomUUID().toString();
+    BetSettled event = betSettled(owner);
+    StompSession ownerSession = connect("/ws/v1/bets", authHeaders(owner));
+    StompSession otherSession = connect("/ws/v1/bets", authHeaders(other));
+    try {
+      BlockingQueue<String> ownerMessages = subscribe(ownerSession, "/user/queue/bets");
+      BlockingQueue<String> otherMessages = subscribe(otherSession, "/user/queue/bets");
+      awaitListener("gateway-settled-listener");
+
+      publish(topics.betSettled(), event.getEventId(), event);
+
+      assertThat(ownerMessages.poll(5, SECONDS))
+          .contains(
+              event.getBetId(),
+              owner,
+              "\"status\":\"SETTLED\"",
+              "\"result\":\"WON\"",
+              "\"revisionNumber\":0");
+      assertThat(ownerMessages.poll(1, SECONDS)).isNull();
+      assertThat(otherMessages.poll(1, SECONDS)).isNull();
+    } finally {
+      ownerSession.disconnect();
+      otherSession.disconnect();
+    }
+  }
+
+  protected StompHeaders authHeaders(String userId) {
+    when(jwtDecoder.decode(userId))
+        .thenReturn(
+            Jwt.withTokenValue(userId)
+                .header("alg", "RS256")
+                .subject(userId)
+                .expiresAt(Instant.now().plusSeconds(60))
+                .build());
+    StompHeaders headers = new StompHeaders();
+    headers.add(HttpHeaders.AUTHORIZATION, "Bearer " + userId);
+    return headers;
+  }
+
+  protected void awaitListener(String id) {
+    await()
+        .atMost(5, SECONDS)
+        .until(() -> !listeners.getListenerContainer(id).getAssignedPartitions().isEmpty());
+  }
+
+  protected static BetSettled betSettled(String userId) {
+    Money stake = Money.newBuilder().setAmount(10_000).setCurrency("KRW").build();
+    return BetSettled.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(userId)
+        .setEventId(UUID.randomUUID().toString())
+        .setResult(SettlementResultAvro.WON)
+        .setStake(stake)
+        .setPayout(Money.newBuilder(stake).setAmount(18_500).build())
+        .setSettledAt(Instant.parse("2026-08-21T00:00:01Z"))
+        .build();
+  }
+
   protected static OddsChanged oddsChanged(String eventId) {
     return OddsChanged.newBuilder()
         .setEventId(eventId)


## `test(websocket): verify voided bet fan-out`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
index 4a2ffd8..c83c66d 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
@@ -6,9 +6,11 @@ import static org.awaitility.Awaitility.await;
 import static org.mockito.Mockito.when;
 
 import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.Money;
 import com.sportsbook.protocol.event.OddsChanged;
 import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.event.VoidReason;
 import java.time.Instant;
 import java.util.UUID;
 import java.util.concurrent.BlockingQueue;
@@ -105,6 +107,36 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
     }
   }
 
+  @Test
+  void deliversVoidedBetOnlyToOwningUser() throws Exception {
+    String owner = UUID.randomUUID().toString();
+    String other = UUID.randomUUID().toString();
+    BetVoided event = betVoided(owner);
+    StompSession ownerSession = connect("/ws/v1/bets", authHeaders(owner));
+    StompSession otherSession = connect("/ws/v1/bets", authHeaders(other));
+    try {
+      BlockingQueue<String> ownerMessages = subscribe(ownerSession, "/user/queue/bets");
+      BlockingQueue<String> otherMessages = subscribe(otherSession, "/user/queue/bets");
+      awaitListener("gateway-voided-listener");
+
+      publish(topics.betVoided(), event.getEventId(), event);
+
+      assertThat(ownerMessages.poll(5, SECONDS))
+          .contains(
+              event.getBetId(),
+              owner,
+              "\"status\":\"VOIDED\"",
+              "\"reason\":\"EVENT_POSTPONED\"",
+              "\"amount\":{\"amount\":10000,\"currency\":\"KRW\"}",
+              "\"revisionNumber\":null");
+      assertThat(ownerMessages.poll(1, SECONDS)).isNull();
+      assertThat(otherMessages.poll(1, SECONDS)).isNull();
+    } finally {
+      ownerSession.disconnect();
+      otherSession.disconnect();
+    }
+  }
+
   protected StompHeaders authHeaders(String userId) {
     when(jwtDecoder.decode(userId))
         .thenReturn(
@@ -137,6 +169,17 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
         .build();
   }
 
+  protected static BetVoided betVoided(String userId) {
+    return BetVoided.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(userId)
+        .setEventId(UUID.randomUUID().toString())
+        .setReason(VoidReason.EVENT_POSTPONED)
+        .setRefund(Money.newBuilder().setAmount(10_000).setCurrency("KRW").build())
+        .setVoidedAt(Instant.parse("2026-08-21T00:00:02Z"))
+        .build();
+  }
+
   protected static OddsChanged oddsChanged(String eventId) {
     return OddsChanged.newBuilder()
         .setEventId(eventId)


## `feat(websocket): publish resolution revisions`

diff --git a/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
index 2cad337..ebc81e5 100644
--- a/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
+++ b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
@@ -1,6 +1,7 @@
 package com.sportsbook.gateway.ws;
 
 import com.sportsbook.gateway.events.GatewayEventContract;
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import org.apache.kafka.clients.consumer.ConsumerRecord;
@@ -40,4 +41,15 @@ public class BetStatusStreamListener {
     BetVoided event = GatewayEventContract.betVoided(record);
     publisher.publishBet(event.getUserId(), BetStatusUpdate.voided(event));
   }
+
+  @KafkaListener(
+      id = "gateway-revision-listener",
+      topics = "${gateway.topics.bet-resolution-revised}",
+      groupId = "gateway-bets",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onBetResolutionRevised(ConsumerRecord<byte[], byte[]> record) {
+    BetResolutionRevised event = GatewayEventContract.betResolutionRevised(record);
+    publisher.publishBet(event.getUserId(), BetStatusUpdate.revised(event));
+  }
 }
diff --git a/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java b/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java
index 96c9c68..ebe1a63 100644
--- a/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java
+++ b/src/main/java/com/sportsbook/gateway/ws/BetStatusUpdate.java
@@ -1,5 +1,6 @@
 package com.sportsbook.gateway.ws;
 
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.Money;
@@ -51,4 +52,18 @@ public record BetStatusUpdate(
         null,
         event.getVoidedAt());
   }
+
+  static BetStatusUpdate revised(BetResolutionRevised event) {
+    return new BetStatusUpdate(
+        event.getBetId(),
+        event.getUserId(),
+        event.getEventId(),
+        "SETTLED",
+        event.getNewResult().name(),
+        MoneyView.from(event.getNewPayout()),
+        null,
+        event.getRevisionId(),
+        event.getRevisionNumber(),
+        event.getRevisedAt());
+  }
 }


## `test(websocket): verify revision projections`

diff --git a/src/test/java/com/sportsbook/gateway/ws/BetStatusUpdateTest.java b/src/test/java/com/sportsbook/gateway/ws/BetStatusUpdateTest.java
new file mode 100644
index 0000000..d01ea9d
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/BetStatusUpdateTest.java
@@ -0,0 +1,55 @@
+package com.sportsbook.gateway.ws;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.CsvSource;
+
+class BetStatusUpdateTest {
+
+  @ParameterizedTest
+  @CsvSource({"LOST,WON,0,18500", "WON,LOST,18500,0"})
+  void projectsTheLatestRevisionSnapshot(
+      SettlementResultAvro previousResult,
+      SettlementResultAvro newResult,
+      long previousPayout,
+      long newPayout) {
+    BetResolutionRevised event =
+        BetResolutionRevised.newBuilder()
+            .setRevisionId(UUID.randomUUID().toString())
+            .setRevisionNumber(3)
+            .setBetId(UUID.randomUUID().toString())
+            .setUserId(UUID.randomUUID().toString())
+            .setEventId(UUID.randomUUID().toString())
+            .setPreviousResult(previousResult)
+            .setNewResult(newResult)
+            .setPreviousPayout(money(previousPayout))
+            .setNewPayout(money(newPayout))
+            .setSourceResultSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+            .setRevisedAt(Instant.parse("2026-08-21T00:00:03Z"))
+            .build();
+
+    assertThat(BetStatusUpdate.revised(event))
+        .isEqualTo(
+            new BetStatusUpdate(
+                event.getBetId(),
+                event.getUserId(),
+                event.getEventId(),
+                "SETTLED",
+                newResult.name(),
+                new BetStatusUpdate.MoneyView(newPayout, "KRW"),
+                null,
+                event.getRevisionId(),
+                3L,
+                event.getRevisedAt()));
+  }
+
+  private static Money money(long amount) {
+    return Money.newBuilder().setAmount(amount).setCurrency("KRW").build();
+  }
+}


## `test(websocket): verify revision fan-out`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
index c83c66d..9d5bf80 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketStreamIntegrationTest.java
@@ -5,6 +5,7 @@ import static org.assertj.core.api.Assertions.assertThat;
 import static org.awaitility.Awaitility.await;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.Money;
@@ -137,6 +138,38 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
     }
   }
 
+  @Test
+  void deliversResolutionRevisionOnlyToOwningUser() throws Exception {
+    String owner = UUID.randomUUID().toString();
+    String other = UUID.randomUUID().toString();
+    BetResolutionRevised event = betResolutionRevised(owner);
+    StompSession ownerSession = connect("/ws/v1/bets", authHeaders(owner));
+    StompSession otherSession = connect("/ws/v1/bets", authHeaders(other));
+    try {
+      BlockingQueue<String> ownerMessages = subscribe(ownerSession, "/user/queue/bets");
+      BlockingQueue<String> otherMessages = subscribe(otherSession, "/user/queue/bets");
+      awaitListener("gateway-revision-listener");
+
+      publish(topics.betResolutionRevised(), event.getBetId(), event);
+
+      assertThat(ownerMessages.poll(5, SECONDS))
+          .contains(
+              event.getBetId(),
+              owner,
+              "\"status\":\"SETTLED\"",
+              "\"result\":\"WON\"",
+              "\"amount\":{\"amount\":18500,\"currency\":\"KRW\"}",
+              "\"revisionId\":\"" + event.getRevisionId() + "\"",
+              "\"revisionNumber\":3",
+              "\"updatedAt\":\"2026-08-21T00:00:03Z\"");
+      assertThat(ownerMessages.poll(1, SECONDS)).isNull();
+      assertThat(otherMessages.poll(1, SECONDS)).isNull();
+    } finally {
+      ownerSession.disconnect();
+      otherSession.disconnect();
+    }
+  }
+
   protected StompHeaders authHeaders(String userId) {
     when(jwtDecoder.decode(userId))
         .thenReturn(
@@ -180,6 +213,22 @@ class WebSocketStreamIntegrationTest extends WebSocketStreamFixture {
         .build();
   }
 
+  protected static BetResolutionRevised betResolutionRevised(String userId) {
+    return BetResolutionRevised.newBuilder()
+        .setRevisionId(UUID.randomUUID().toString())
+        .setRevisionNumber(3)
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(userId)
+        .setEventId(UUID.randomUUID().toString())
+        .setPreviousResult(SettlementResultAvro.LOST)
+        .setNewResult(SettlementResultAvro.WON)
+        .setPreviousPayout(Money.newBuilder().setAmount(0).setCurrency("KRW").build())
+        .setNewPayout(Money.newBuilder().setAmount(18_500).setCurrency("KRW").build())
+        .setSourceResultSettledAt(Instant.parse("2026-08-21T00:00:00Z"))
+        .setRevisedAt(Instant.parse("2026-08-21T00:00:03Z"))
+        .build();
+  }
+
   protected static OddsChanged oddsChanged(String eventId) {
     return OddsChanged.newBuilder()
         .setEventId(eventId)
