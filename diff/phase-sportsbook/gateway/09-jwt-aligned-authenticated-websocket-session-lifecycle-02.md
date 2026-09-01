## `test(websocket): reconnect after token expiry`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java
new file mode 100644
index 0000000..ddb097c
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.gateway.ws;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.when;
+
+import java.time.Instant;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.web.server.LocalServerPort;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.web.socket.CloseStatus;
+import org.springframework.web.socket.TextMessage;
+import org.springframework.web.socket.WebSocketSession;
+import org.springframework.web.socket.client.standard.StandardWebSocketClient;
+import org.springframework.web.socket.handler.TextWebSocketHandler;
+
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
+    properties = {
+      "gateway.ratelimit.enabled=false",
+      "gateway.downstream.wallet.api-key=fixture-wallet-key-32-characters-long"
+    })
+class WebSocketTokenExpiryTest {
+
+  @LocalServerPort private int port;
+  @MockBean JwtDecoder decoder;
+
+  @Test
+  void reconnectsWithFreshCredentialsAfterTokenExpiry() throws Exception {
+    when(decoder.decode("expiring")).thenReturn(jwt(Instant.now().plusSeconds(3)));
+    RawStompConnection expired = connect("expiring");
+
+    assertThat(expired.connected.await(2, TimeUnit.SECONDS)).isTrue();
+    assertThat(expired.closed.await(5, TimeUnit.SECONDS)).isTrue();
+    assertThat(expired.closeStatus).isEqualTo(CloseStatus.POLICY_VIOLATION);
+
+    when(decoder.decode("fresh")).thenReturn(jwt(Instant.now().plusSeconds(30)));
+    RawStompConnection fresh = connect("fresh");
+    assertThat(fresh.connected.await(2, TimeUnit.SECONDS)).isTrue();
+    assertThat(fresh.closed.await(1, TimeUnit.SECONDS)).isFalse();
+    assertThat(fresh.session.isOpen()).isTrue();
+    fresh.session.close();
+  }
+
+  private RawStompConnection connect(String token) throws Exception {
+    RawStompConnection connection = new RawStompConnection(token);
+    new StandardWebSocketClient()
+        .execute(connection, "ws://localhost:" + port + "/ws/v1/odds")
+        .get(3, TimeUnit.SECONDS);
+    return connection;
+  }
+
+  private static Jwt jwt(Instant expiresAt) {
+    return Jwt.withTokenValue("token")
+        .header("alg", "RS256")
+        .subject("11111111-1111-4111-8111-111111111111")
+        .expiresAt(expiresAt)
+        .build();
+  }
+
+  private static final class RawStompConnection extends TextWebSocketHandler {
+    private final String token;
+    private final CountDownLatch connected = new CountDownLatch(1);
+    private final CountDownLatch closed = new CountDownLatch(1);
+    private volatile WebSocketSession session;
+    private volatile CloseStatus closeStatus;
+
+    private RawStompConnection(String token) {
+      this.token = token;
+    }
+
+    @Override
+    public void afterConnectionEstablished(WebSocketSession established) throws Exception {
+      session = established;
+      established.sendMessage(
+          new TextMessage(
+              "CONNECT\naccept-version:1.2\nhost:localhost\nAuthorization:Bearer "
+                  + token
+                  + "\n\n\0"));
+    }
+
+    @Override
+    protected void handleTextMessage(WebSocketSession current, TextMessage message) {
+      if (message.getPayload().startsWith("CONNECTED")) {
+        connected.countDown();
+      }
+    }
+
+    @Override
+    public void afterConnectionClosed(WebSocketSession current, CloseStatus status) {
+      closeStatus = status;
+      closed.countDown();
+    }
+  }
+}


## `test(websocket): preserve anonymous odds sessions`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java
index ddb097c..2e9d16a 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketTokenExpiryTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.gateway.ws;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
 import java.time.Instant;
@@ -46,6 +47,17 @@ class WebSocketTokenExpiryTest {
     fresh.session.close();
   }
 
+  @Test
+  void preservesAnonymousOddsSessionWithoutAnExpiryTask() throws Exception {
+    RawStompConnection anonymous = connect(null);
+
+    assertThat(anonymous.connected.await(2, TimeUnit.SECONDS)).isTrue();
+    assertThat(anonymous.closed.await(2, TimeUnit.SECONDS)).isFalse();
+    assertThat(anonymous.session.isOpen()).isTrue();
+    verifyNoInteractions(decoder);
+    anonymous.session.close();
+  }
+
   private RawStompConnection connect(String token) throws Exception {
     RawStompConnection connection = new RawStompConnection(token);
     new StandardWebSocketClient()
@@ -76,11 +88,10 @@ class WebSocketTokenExpiryTest {
     @Override
     public void afterConnectionEstablished(WebSocketSession established) throws Exception {
       session = established;
+      String authorization = token == null ? "" : "Authorization:Bearer " + token + "\n";
       established.sendMessage(
           new TextMessage(
-              "CONNECT\naccept-version:1.2\nhost:localhost\nAuthorization:Bearer "
-                  + token
-                  + "\n\n\0"));
+              "CONNECT\naccept-version:1.2\nhost:localhost\n" + authorization + "\n\0"));
     }
 
     @Override
