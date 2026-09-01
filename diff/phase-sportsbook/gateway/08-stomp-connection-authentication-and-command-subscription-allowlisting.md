# STOMP 연결 인증과 명령·구독 허용목록

## `feat(websocket): configure STOMP transport`

diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
new file mode 100644
index 0000000..a800280
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -0,0 +1,46 @@
+package com.sportsbook.gateway.ws;
+
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.messaging.simp.config.MessageBrokerRegistry;
+import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
+import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
+import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
+import org.springframework.web.socket.config.annotation.WebSocketTransportRegistration;
+
+@Configuration
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+@EnableWebSocketMessageBroker
+public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
+
+  private static final int MESSAGE_SIZE_LIMIT = 64 * 1024;
+  private static final int SEND_BUFFER_LIMIT = 512 * 1024;
+  private static final int SEND_TIME_LIMIT = 10_000;
+
+  private final String[] allowedOrigins;
+
+  public WebSocketConfig(@Value("${gateway.ws.allowed-origins}") String[] allowedOrigins) {
+    this.allowedOrigins = allowedOrigins;
+  }
+
+  @Override
+  public void registerStompEndpoints(StompEndpointRegistry registry) {
+    registry.addEndpoint("/ws/v1/odds", "/ws/v1/bets").setAllowedOriginPatterns(allowedOrigins);
+  }
+
+  @Override
+  public void configureMessageBroker(MessageBrokerRegistry registry) {
+    registry.enableSimpleBroker("/topic", "/queue");
+    registry.setApplicationDestinationPrefixes("/app");
+    registry.setUserDestinationPrefix("/user");
+  }
+
+  @Override
+  public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
+    registration
+        .setMessageSizeLimit(MESSAGE_SIZE_LIMIT)
+        .setSendBufferSizeLimit(SEND_BUFFER_LIMIT)
+        .setSendTimeLimit(SEND_TIME_LIMIT);
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 81faa7d..61fe403 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -42,3 +42,5 @@ gateway:
       capacity: ${GATEWAY_RATELIMIT_IP_CAPACITY:60}
       refill-period: ${GATEWAY_RATELIMIT_IP_PERIOD:1m}
     trusted-proxy-cidrs: ${GATEWAY_TRUSTED_PROXY_CIDRS:}
+  ws:
+    allowed-origins: ${GATEWAY_WS_ALLOWED_ORIGINS:http://localhost:[*],http://127.0.0.1:[*]}


## `test(websocket): verify endpoint and origin policy`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketEndpointTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketEndpointTest.java
new file mode 100644
index 0000000..0d613e4
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketEndpointTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.gateway.ws;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.net.URI;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.web.server.LocalServerPort;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.web.socket.WebSocketHttpHeaders;
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
+class WebSocketEndpointTest {
+
+  @LocalServerPort private int port;
+  @MockBean JwtDecoder jwtDecoder;
+
+  @Test
+  void exposesDeclaredEndpointsForAllowedBrowserOrigins() throws Exception {
+    WebSocketHttpHeaders headers = new WebSocketHttpHeaders();
+    headers.setOrigin("http://localhost:3000");
+
+    for (String path : new String[] {"/ws/v1/odds", "/ws/v1/bets"}) {
+      WebSocketSession session =
+          new StandardWebSocketClient()
+              .execute(new TextWebSocketHandler() {}, headers, endpoint(path))
+              .get(3, TimeUnit.SECONDS);
+      assertThat(session.isOpen()).isTrue();
+      session.close();
+    }
+  }
+
+  @Test
+  void rejectsUntrustedOriginsAndUndeclaredEndpoints() {
+    WebSocketHttpHeaders untrusted = new WebSocketHttpHeaders();
+    untrusted.setOrigin("https://untrusted.example");
+
+    assertThatThrownBy(
+            () ->
+                new StandardWebSocketClient()
+                    .execute(new TextWebSocketHandler() {}, untrusted, endpoint("/ws/v1/odds"))
+                    .get(3, TimeUnit.SECONDS))
+        .rootCause()
+        .hasMessageContaining("[403]");
+    assertThatThrownBy(
+            () ->
+                new StandardWebSocketClient()
+                    .execute(
+                        new TextWebSocketHandler() {}, endpoint("/ws/v1/undeclared").toString())
+                    .get(3, TimeUnit.SECONDS))
+        .rootCause()
+        .hasMessageContaining("[401]");
+  }
+
+  private URI endpoint(String path) {
+    return URI.create("ws://localhost:" + port + path);
+  }
+}


## `feat(websocket): authenticate STOMP sessions`

diff --git a/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java b/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java
new file mode 100644
index 0000000..f8b8097
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java
@@ -0,0 +1,75 @@
+package com.sportsbook.gateway.ws;
+
+import java.util.Collection;
+import java.util.List;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.http.HttpHeaders;
+import org.springframework.messaging.Message;
+import org.springframework.messaging.MessageChannel;
+import org.springframework.messaging.MessageDeliveryException;
+import org.springframework.messaging.simp.stomp.StompCommand;
+import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
+import org.springframework.messaging.support.ChannelInterceptor;
+import org.springframework.messaging.support.MessageHeaderAccessor;
+import org.springframework.security.core.GrantedAuthority;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.security.oauth2.jwt.JwtException;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.stereotype.Component;
+
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class StompAuthChannelInterceptor implements ChannelInterceptor {
+
+  private static final String BEARER_PREFIX = "Bearer ";
+
+  private final JwtDecoder jwtDecoder;
+
+  public StompAuthChannelInterceptor(JwtDecoder jwtDecoder) {
+    this.jwtDecoder = jwtDecoder;
+  }
+
+  @Override
+  public Message<?> preSend(Message<?> message, MessageChannel channel) {
+    StompHeaderAccessor accessor =
+        MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
+    if (accessor != null && isConnect(accessor.getCommand())) {
+      authenticate(accessor);
+    }
+    return message;
+  }
+
+  private void authenticate(StompHeaderAccessor accessor) {
+    List<String> headers = accessor.getNativeHeader(HttpHeaders.AUTHORIZATION);
+    if (headers == null || headers.isEmpty()) {
+      accessor.setUser(null);
+      return;
+    }
+    if (headers.size() != 1
+        || !headers.get(0).startsWith(BEARER_PREFIX)
+        || headers.get(0).length() == BEARER_PREFIX.length()) {
+      throw new MessageDeliveryException("Invalid Authorization header");
+    }
+    try {
+      Jwt jwt = jwtDecoder.decode(headers.get(0).substring(BEARER_PREFIX.length()));
+      accessor.setUser(new JwtAuthenticationToken(jwt, authorities(jwt), jwt.getSubject()));
+    } catch (JwtException failure) {
+      throw new MessageDeliveryException("Invalid or expired token");
+    }
+  }
+
+  private static Collection<GrantedAuthority> authorities(Jwt jwt) {
+    List<String> roles = jwt.getClaimAsStringList("roles");
+    return roles == null
+        ? List.of()
+        : roles.stream()
+            .<GrantedAuthority>map(role -> new SimpleGrantedAuthority("ROLE_" + role))
+            .toList();
+  }
+
+  static boolean isConnect(StompCommand command) {
+    return StompCommand.CONNECT.equals(command) || StompCommand.STOMP.equals(command);
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
index a800280..b1f3853 100644
--- a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -3,6 +3,7 @@ package com.sportsbook.gateway.ws;
 import org.springframework.beans.factory.annotation.Value;
 import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
 import org.springframework.context.annotation.Configuration;
+import org.springframework.messaging.simp.config.ChannelRegistration;
 import org.springframework.messaging.simp.config.MessageBrokerRegistry;
 import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
 import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
@@ -19,9 +20,13 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
   private static final int SEND_TIME_LIMIT = 10_000;
 
   private final String[] allowedOrigins;
+  private final StompAuthChannelInterceptor authentication;
 
-  public WebSocketConfig(@Value("${gateway.ws.allowed-origins}") String[] allowedOrigins) {
+  public WebSocketConfig(
+      @Value("${gateway.ws.allowed-origins}") String[] allowedOrigins,
+      StompAuthChannelInterceptor authentication) {
     this.allowedOrigins = allowedOrigins;
+    this.authentication = authentication;
   }
 
   @Override
@@ -36,6 +41,11 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
     registry.setUserDestinationPrefix("/user");
   }
 
+  @Override
+  public void configureClientInboundChannel(ChannelRegistration registration) {
+    registration.interceptors(authentication);
+  }
+
   @Override
   public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
     registration


## `test(websocket): verify CONNECT authentication`

diff --git a/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java b/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java
new file mode 100644
index 0000000..49a5576
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java
@@ -0,0 +1,100 @@
+package com.sportsbook.gateway.ws;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Instant;
+import java.util.List;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
+import org.springframework.messaging.Message;
+import org.springframework.messaging.MessageDeliveryException;
+import org.springframework.messaging.simp.stomp.StompCommand;
+import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
+import org.springframework.messaging.support.MessageBuilder;
+import org.springframework.messaging.support.MessageHeaderAccessor;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtException;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+
+class StompAuthChannelInterceptorTest {
+
+  private static final String USER_ID = "11111111-1111-4111-8111-111111111111";
+
+  @Test
+  void authenticatesConnectWithTheSharedDecoder() {
+    AtomicReference<String> decoded = new AtomicReference<>();
+    StompAuthChannelInterceptor interceptor =
+        new StompAuthChannelInterceptor(
+            token -> {
+              decoded.set(token);
+              return jwt();
+            });
+    Message<byte[]> frame = connect(StompCommand.STOMP, "Bearer signed-token");
+
+    interceptor.preSend(frame, null);
+
+    JwtAuthenticationToken authentication =
+        (JwtAuthenticationToken) StompHeaderAccessor.wrap(frame).getUser();
+    assertThat(decoded).hasValue("signed-token");
+    assertThat(authentication.getName()).isEqualTo(USER_ID);
+    assertThat(authentication.getAuthorities())
+        .extracting("authority")
+        .containsExactly("ROLE_USER", "ROLE_TRADER");
+  }
+
+  @Test
+  void permitsAnonymousConnectWithoutDecoding() {
+    StompAuthChannelInterceptor interceptor =
+        new StompAuthChannelInterceptor(
+            token -> {
+              throw new AssertionError("anonymous CONNECT must not decode a token");
+            });
+    Message<byte[]> frame = connect(StompCommand.CONNECT);
+    MessageHeaderAccessor.getAccessor(frame, StompHeaderAccessor.class)
+        .setUser(new JwtAuthenticationToken(jwt()));
+
+    assertThatCode(() -> interceptor.preSend(frame, null)).doesNotThrowAnyException();
+    assertThat(StompHeaderAccessor.wrap(frame).getUser()).isNull();
+  }
+
+  @Test
+  void rejectsMalformedDuplicateAndUnverifiableCredentials() {
+    StompAuthChannelInterceptor interceptor =
+        new StompAuthChannelInterceptor(
+            token -> {
+              throw new JwtException("rejected");
+            });
+
+    assertRejected(interceptor, "Basic token");
+    assertRejected(interceptor, "Bearer ");
+    assertRejected(interceptor, "Bearer one", "Bearer two");
+    assertRejected(interceptor, "Bearer rejected-token");
+  }
+
+  private static void assertRejected(
+      StompAuthChannelInterceptor interceptor, String... credentials) {
+    assertThatThrownBy(() -> interceptor.preSend(connect(StompCommand.CONNECT, credentials), null))
+        .isInstanceOf(MessageDeliveryException.class);
+  }
+
+  private static Message<byte[]> connect(StompCommand command, String... credentials) {
+    StompHeaderAccessor accessor = StompHeaderAccessor.create(command);
+    for (String credential : credentials) {
+      accessor.addNativeHeader("Authorization", credential);
+    }
+    accessor.setLeaveMutable(true);
+    return MessageBuilder.createMessage(new byte[0], accessor.getMessageHeaders());
+  }
+
+  private static Jwt jwt() {
+    return Jwt.withTokenValue("signed-token")
+        .header("alg", "RS256")
+        .subject(USER_ID)
+        .claim("roles", List.of("USER", "TRADER"))
+        .issuedAt(Instant.now().minusSeconds(1))
+        .expiresAt(Instant.now().plusSeconds(60))
+        .build();
+  }
+}


## `feat(websocket): restrict client STOMP commands`

diff --git a/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java b/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java
index f8b8097..2742f6a 100644
--- a/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java
+++ b/src/main/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptor.java
@@ -2,6 +2,7 @@ package com.sportsbook.gateway.ws;
 
 import java.util.Collection;
 import java.util.List;
+import java.util.UUID;
 import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
 import org.springframework.http.HttpHeaders;
 import org.springframework.messaging.Message;
@@ -35,8 +36,17 @@ public class StompAuthChannelInterceptor implements ChannelInterceptor {
   public Message<?> preSend(Message<?> message, MessageChannel channel) {
     StompHeaderAccessor accessor =
         MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
-    if (accessor != null && isConnect(accessor.getCommand())) {
+    if (accessor == null || accessor.getCommand() == null) {
+      return message;
+    }
+    StompCommand command = accessor.getCommand();
+    if (isConnect(command)) {
       authenticate(accessor);
+    } else if (StompCommand.SUBSCRIBE.equals(command)) {
+      authorizeSubscription(accessor);
+    } else if (!StompCommand.UNSUBSCRIBE.equals(command)
+        && !StompCommand.DISCONNECT.equals(command)) {
+      throw new MessageDeliveryException("Client STOMP command is not supported");
     }
     return message;
   }
@@ -72,4 +82,27 @@ public class StompAuthChannelInterceptor implements ChannelInterceptor {
   static boolean isConnect(StompCommand command) {
     return StompCommand.CONNECT.equals(command) || StompCommand.STOMP.equals(command);
   }
+
+  private static void authorizeSubscription(StompHeaderAccessor accessor) {
+    String destination = accessor.getDestination();
+    if (isCanonicalOddsDestination(destination)
+        || ("/user/queue/bets".equals(destination)
+            && accessor.getUser() instanceof JwtAuthenticationToken)) {
+      return;
+    }
+    throw new MessageDeliveryException("Subscription destination is not allowed");
+  }
+
+  private static boolean isCanonicalOddsDestination(String destination) {
+    String prefix = "/topic/odds/";
+    if (destination == null || !destination.startsWith(prefix)) {
+      return false;
+    }
+    String eventId = destination.substring(prefix.length());
+    try {
+      return UUID.fromString(eventId).toString().equals(eventId);
+    } catch (IllegalArgumentException ignored) {
+      return false;
+    }
+  }
 }


## `test(websocket): verify destination permissions`

diff --git a/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java b/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java
index 49a5576..650f462 100644
--- a/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/StompAuthChannelInterceptorTest.java
@@ -4,6 +4,7 @@ import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatCode;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
+import java.security.Principal;
 import java.time.Instant;
 import java.util.List;
 import java.util.concurrent.atomic.AtomicReference;
@@ -21,6 +22,7 @@ import org.springframework.security.oauth2.server.resource.authentication.JwtAut
 class StompAuthChannelInterceptorTest {
 
   private static final String USER_ID = "11111111-1111-4111-8111-111111111111";
+  private static final String EVENT_ID = "abcdefab-cdef-4abc-8def-abcdefabcdef";
 
   @Test
   void authenticatesConnectWithTheSharedDecoder() {
@@ -73,12 +75,77 @@ class StompAuthChannelInterceptorTest {
     assertRejected(interceptor, "Bearer rejected-token");
   }
 
+  @Test
+  void rejectsEveryUnsupportedClientCommand() {
+    for (StompCommand command :
+        List.of(
+            StompCommand.SEND,
+            StompCommand.MESSAGE,
+            StompCommand.CONNECTED,
+            StompCommand.RECEIPT,
+            StompCommand.ERROR,
+            StompCommand.ACK,
+            StompCommand.NACK,
+            StompCommand.BEGIN,
+            StompCommand.COMMIT,
+            StompCommand.ABORT)) {
+      assertDenied(frame(command, "/topic/odds/" + EVENT_ID, null));
+    }
+  }
+
+  @Test
+  void allowsOnlyPublicOddsAndAuthenticatedBetSubscriptions() {
+    StompAuthChannelInterceptor interceptor = interceptor();
+    for (StompCommand command : List.of(StompCommand.UNSUBSCRIBE, StompCommand.DISCONNECT)) {
+      assertThatCode(() -> interceptor.preSend(frame(command, null, null), null))
+          .doesNotThrowAnyException();
+    }
+    assertThatCode(
+            () ->
+                interceptor.preSend(
+                    frame(StompCommand.SUBSCRIBE, "/topic/odds/" + EVENT_ID, null), null))
+        .doesNotThrowAnyException();
+    assertThatCode(
+            () ->
+                interceptor.preSend(
+                    frame(
+                        StompCommand.SUBSCRIBE,
+                        "/user/queue/bets",
+                        new JwtAuthenticationToken(jwt())),
+                    null))
+        .doesNotThrowAnyException();
+
+    assertDenied(frame(StompCommand.SUBSCRIBE, "/user/queue/bets", null));
+    assertDenied(frame(StompCommand.SUBSCRIBE, "/user/queue/bets", (Principal) () -> USER_ID));
+    assertDenied(frame(StompCommand.SUBSCRIBE, "/topic/odds/" + EVENT_ID + "/extra", null));
+    assertDenied(frame(StompCommand.SUBSCRIBE, "/topic/odds/" + EVENT_ID.toUpperCase(), null));
+    assertDenied(frame(StompCommand.SUBSCRIBE, "/queue/internal", null));
+  }
+
   private static void assertRejected(
       StompAuthChannelInterceptor interceptor, String... credentials) {
     assertThatThrownBy(() -> interceptor.preSend(connect(StompCommand.CONNECT, credentials), null))
         .isInstanceOf(MessageDeliveryException.class);
   }
 
+  private static void assertDenied(Message<byte[]> frame) {
+    assertThatThrownBy(() -> interceptor().preSend(frame, null))
+        .isInstanceOf(MessageDeliveryException.class);
+  }
+
+  private static StompAuthChannelInterceptor interceptor() {
+    return new StompAuthChannelInterceptor(token -> jwt());
+  }
+
+  private static Message<byte[]> frame(
+      StompCommand command, String destination, Principal principal) {
+    StompHeaderAccessor accessor = StompHeaderAccessor.create(command);
+    accessor.setDestination(destination);
+    accessor.setUser(principal);
+    accessor.setLeaveMutable(true);
+    return MessageBuilder.createMessage(new byte[0], accessor.getMessageHeaders());
+  }
+
   private static Message<byte[]> connect(StompCommand command, String... credentials) {
     StompHeaderAccessor accessor = StompHeaderAccessor.create(command);
     for (String credential : credentials) {
