# JWT 만료에 맞춘 인증 WebSocket 세션 수명 주기

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


## `feat(websocket): track raw WebSocket sessions`

diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
index b1f3853..8ea8096 100644
--- a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -21,12 +21,15 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
 
   private final String[] allowedOrigins;
   private final StompAuthChannelInterceptor authentication;
+  private final WebSocketSessionRegistry sessions;
 
   public WebSocketConfig(
       @Value("${gateway.ws.allowed-origins}") String[] allowedOrigins,
-      StompAuthChannelInterceptor authentication) {
+      StompAuthChannelInterceptor authentication,
+      WebSocketSessionRegistry sessions) {
     this.allowedOrigins = allowedOrigins;
     this.authentication = authentication;
+    this.sessions = sessions;
   }
 
   @Override
@@ -51,6 +54,7 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
     registration
         .setMessageSizeLimit(MESSAGE_SIZE_LIMIT)
         .setSendBufferSizeLimit(SEND_BUFFER_LIMIT)
-        .setSendTimeLimit(SEND_TIME_LIMIT);
+        .setSendTimeLimit(SEND_TIME_LIMIT)
+        .addDecoratorFactory(sessions);
   }
 }
diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
new file mode 100644
index 0000000..8c80d5a
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
@@ -0,0 +1,46 @@
+package com.sportsbook.gateway.ws;
+
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.concurrent.ConcurrentMap;
+import org.springframework.stereotype.Component;
+import org.springframework.web.socket.CloseStatus;
+import org.springframework.web.socket.WebSocketHandler;
+import org.springframework.web.socket.WebSocketSession;
+import org.springframework.web.socket.handler.WebSocketHandlerDecorator;
+import org.springframework.web.socket.handler.WebSocketHandlerDecoratorFactory;
+
+@Component
+public class WebSocketSessionRegistry implements WebSocketHandlerDecoratorFactory {
+
+  private final ConcurrentMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
+
+  @Override
+  public WebSocketHandler decorate(WebSocketHandler handler) {
+    return new WebSocketHandlerDecorator(handler) {
+      @Override
+      public void afterConnectionEstablished(WebSocketSession session) throws Exception {
+        sessions.put(session.getId(), session);
+        try {
+          super.afterConnectionEstablished(session);
+        } catch (Exception failure) {
+          sessions.remove(session.getId(), session);
+          throw failure;
+        }
+      }
+
+      @Override
+      public void afterConnectionClosed(WebSocketSession session, CloseStatus status)
+          throws Exception {
+        try {
+          super.afterConnectionClosed(session, status);
+        } finally {
+          sessions.remove(session.getId(), session);
+        }
+      }
+    };
+  }
+
+  int size() {
+    return sessions.size();
+  }
+}


## `test(websocket): verify session registry lifecycle`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java
new file mode 100644
index 0000000..9980928
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java
@@ -0,0 +1,52 @@
+package com.sportsbook.gateway.ws;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.doThrow;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import java.io.IOException;
+import org.junit.jupiter.api.Test;
+import org.springframework.web.socket.CloseStatus;
+import org.springframework.web.socket.WebSocketHandler;
+import org.springframework.web.socket.WebSocketSession;
+
+class WebSocketSessionRegistryTest {
+
+  @Test
+  void tracksEstablishedSessionsUntilTransportClose() throws Exception {
+    WebSocketSessionRegistry registry = new WebSocketSessionRegistry();
+    WebSocketHandler delegate = mock(WebSocketHandler.class);
+    WebSocketSession session = session("session-1");
+    WebSocketHandler tracked = registry.decorate(delegate);
+
+    tracked.afterConnectionEstablished(session);
+    assertThat(registry.size()).isOne();
+    verify(delegate).afterConnectionEstablished(session);
+
+    tracked.afterConnectionClosed(session, CloseStatus.NORMAL);
+    assertThat(registry.size()).isZero();
+    verify(delegate).afterConnectionClosed(session, CloseStatus.NORMAL);
+  }
+
+  @Test
+  void removesSessionsWhenDelegateLifecycleCallbacksFail() throws Exception {
+    WebSocketSessionRegistry registry = new WebSocketSessionRegistry();
+    WebSocketHandler delegate = mock(WebSocketHandler.class);
+    WebSocketSession session = session("session-2");
+    IOException failure = new IOException("delegate failed");
+    doThrow(failure).when(delegate).afterConnectionEstablished(session);
+
+    assertThatThrownBy(() -> registry.decorate(delegate).afterConnectionEstablished(session))
+        .isSameAs(failure);
+    assertThat(registry.size()).isZero();
+  }
+
+  private static WebSocketSession session(String id) {
+    WebSocketSession session = mock(WebSocketSession.class);
+    when(session.getId()).thenReturn(id);
+    return session;
+  }
+}


## `feat(websocket): expire authenticated STOMP sessions`

diff --git a/src/main/java/com/sportsbook/gateway/ws/AuthenticatedSessionExpiryInterceptor.java b/src/main/java/com/sportsbook/gateway/ws/AuthenticatedSessionExpiryInterceptor.java
new file mode 100644
index 0000000..3843fd2
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/AuthenticatedSessionExpiryInterceptor.java
@@ -0,0 +1,87 @@
+package com.sportsbook.gateway.ws;
+
+import java.io.IOException;
+import java.time.Instant;
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.concurrent.ConcurrentMap;
+import java.util.concurrent.Future;
+import java.util.concurrent.FutureTask;
+import java.util.concurrent.ScheduledFuture;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.event.EventListener;
+import org.springframework.messaging.Message;
+import org.springframework.messaging.MessageChannel;
+import org.springframework.messaging.MessageDeliveryException;
+import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
+import org.springframework.messaging.support.ChannelInterceptor;
+import org.springframework.scheduling.TaskScheduler;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.stereotype.Component;
+import org.springframework.web.socket.messaging.SessionDisconnectEvent;
+
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class AuthenticatedSessionExpiryInterceptor implements ChannelInterceptor {
+
+  private final WebSocketSessionRegistry sessions;
+  private final ObjectProvider<TaskScheduler> scheduler;
+  private final ConcurrentMap<String, Future<?>> expiryTasks = new ConcurrentHashMap<>();
+
+  public AuthenticatedSessionExpiryInterceptor(
+      WebSocketSessionRegistry sessions,
+      @Qualifier("messageBrokerTaskScheduler") ObjectProvider<TaskScheduler> scheduler) {
+    this.sessions = sessions;
+    this.scheduler = scheduler;
+  }
+
+  @Override
+  public Message<?> preSend(Message<?> message, MessageChannel channel) {
+    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
+    if (StompAuthChannelInterceptor.isConnect(accessor.getCommand())
+        && accessor.getUser() instanceof JwtAuthenticationToken authentication) {
+      schedule(accessor.getSessionId(), authentication.getToken().getExpiresAt());
+    }
+    return message;
+  }
+
+  @EventListener
+  void disconnect(SessionDisconnectEvent event) {
+    Future<?> expiry = expiryTasks.remove(event.getSessionId());
+    if (expiry != null) {
+      expiry.cancel(false);
+    }
+  }
+
+  private void schedule(String sessionId, Instant expiresAt) {
+    if (sessionId == null || expiresAt == null) {
+      throw new MessageDeliveryException("Authenticated session expiry is unavailable");
+    }
+    FutureTask<Void> expiry = new FutureTask<>(() -> expire(sessionId));
+    if (expiryTasks.putIfAbsent(sessionId, expiry) != null) {
+      throw new MessageDeliveryException("WebSocket session expiry is unavailable");
+    }
+    ScheduledFuture<?> scheduled = null;
+    try {
+      scheduled = scheduler.getObject().schedule(expiry, expiresAt);
+      if (scheduled == null || !expiryTasks.replace(sessionId, expiry, scheduled)) {
+        throw new IllegalStateException("expiry task was not registered");
+      }
+    } catch (RuntimeException failure) {
+      expiryTasks.remove(sessionId, expiry);
+      expiry.cancel(false);
+      if (scheduled != null) {
+        scheduled.cancel(false);
+      }
+      throw new MessageDeliveryException(null, "WebSocket expiry could not be scheduled", failure);
+    }
+  }
+
+  private Void expire(String sessionId) throws IOException {
+    if (expiryTasks.remove(sessionId) != null) {
+      sessions.closeExpired(sessionId);
+    }
+    return null;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
index 8ea8096..42ce447 100644
--- a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -21,14 +21,17 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
 
   private final String[] allowedOrigins;
   private final StompAuthChannelInterceptor authentication;
+  private final AuthenticatedSessionExpiryInterceptor expiry;
   private final WebSocketSessionRegistry sessions;
 
   public WebSocketConfig(
       @Value("${gateway.ws.allowed-origins}") String[] allowedOrigins,
       StompAuthChannelInterceptor authentication,
+      AuthenticatedSessionExpiryInterceptor expiry,
       WebSocketSessionRegistry sessions) {
     this.allowedOrigins = allowedOrigins;
     this.authentication = authentication;
+    this.expiry = expiry;
     this.sessions = sessions;
   }
 
@@ -47,6 +50,7 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
   @Override
   public void configureClientInboundChannel(ChannelRegistration registration) {
     registration.interceptors(authentication);
+    registration.interceptors(expiry);
   }
 
   @Override
diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
index 8c80d5a..744caa2 100644
--- a/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
@@ -1,5 +1,6 @@
 package com.sportsbook.gateway.ws;
 
+import java.io.IOException;
 import java.util.concurrent.ConcurrentHashMap;
 import java.util.concurrent.ConcurrentMap;
 import org.springframework.stereotype.Component;
@@ -40,6 +41,13 @@ public class WebSocketSessionRegistry implements WebSocketHandlerDecoratorFactor
     };
   }
 
+  void closeExpired(String sessionId) throws IOException {
+    WebSocketSession session = sessions.remove(sessionId);
+    if (session != null && session.isOpen()) {
+      session.close(CloseStatus.POLICY_VIOLATION);
+    }
+  }
+
   int size() {
     return sessions.size();
   }


## `test(websocket): close expired authenticated sessions`

diff --git a/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java b/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java
index 9980928..af57446 100644
--- a/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java
+++ b/src/test/java/com/sportsbook/gateway/ws/WebSocketSessionRegistryTest.java
@@ -2,16 +2,31 @@ package com.sportsbook.gateway.ws;
 
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.doReturn;
 import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
 import java.io.IOException;
+import java.time.Instant;
+import java.util.List;
+import java.util.concurrent.ScheduledFuture;
 import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.messaging.Message;
+import org.springframework.messaging.simp.stomp.StompCommand;
+import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
+import org.springframework.messaging.support.MessageBuilder;
+import org.springframework.scheduling.TaskScheduler;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
 import org.springframework.web.socket.CloseStatus;
 import org.springframework.web.socket.WebSocketHandler;
 import org.springframework.web.socket.WebSocketSession;
+import org.springframework.web.socket.messaging.SessionDisconnectEvent;
 
 class WebSocketSessionRegistryTest {
 
@@ -44,9 +59,70 @@ class WebSocketSessionRegistryTest {
     assertThat(registry.size()).isZero();
   }
 
+  @Test
+  @SuppressWarnings("unchecked")
+  void closesAuthenticatedSessionWithPolicyViolationAtExpiry() throws Exception {
+    WebSocketSessionRegistry registry = new WebSocketSessionRegistry();
+    WebSocketSession session = session("expiring-session");
+    when(session.isOpen()).thenReturn(true);
+    registry.decorate(mock(WebSocketHandler.class)).afterConnectionEstablished(session);
+    TaskScheduler scheduler = mock(TaskScheduler.class);
+    ObjectProvider<TaskScheduler> schedulers = mock(ObjectProvider.class);
+    ScheduledFuture<?> scheduled = mock(ScheduledFuture.class);
+    Instant expiresAt = Instant.parse("2030-01-01T00:00:00Z");
+    when(schedulers.getObject()).thenReturn(scheduler);
+    doReturn(scheduled).when(scheduler).schedule(any(Runnable.class), eq(expiresAt));
+    AuthenticatedSessionExpiryInterceptor expiry =
+        new AuthenticatedSessionExpiryInterceptor(registry, schedulers);
+
+    expiry.preSend(connect("expiring-session", expiresAt, StompCommand.STOMP), null);
+
+    var task = org.mockito.ArgumentCaptor.forClass(Runnable.class);
+    verify(scheduler).schedule(task.capture(), eq(expiresAt));
+    task.getValue().run();
+    verify(session).close(CloseStatus.POLICY_VIOLATION);
+    assertThat(registry.size()).isZero();
+  }
+
+  @Test
+  @SuppressWarnings("unchecked")
+  void cancelsExpiryWhenTheTransportDisconnectsEarly() {
+    TaskScheduler scheduler = mock(TaskScheduler.class);
+    ObjectProvider<TaskScheduler> schedulers = mock(ObjectProvider.class);
+    ScheduledFuture<?> scheduled = mock(ScheduledFuture.class);
+    Instant expiresAt = Instant.parse("2030-01-01T00:00:00Z");
+    when(schedulers.getObject()).thenReturn(scheduler);
+    doReturn(scheduled).when(scheduler).schedule(any(Runnable.class), eq(expiresAt));
+    AuthenticatedSessionExpiryInterceptor expiry =
+        new AuthenticatedSessionExpiryInterceptor(new WebSocketSessionRegistry(), schedulers);
+    expiry.preSend(connect("disconnected-session", expiresAt, StompCommand.CONNECT), null);
+    SessionDisconnectEvent disconnect = mock(SessionDisconnectEvent.class);
+    when(disconnect.getSessionId()).thenReturn("disconnected-session");
+
+    expiry.disconnect(disconnect);
+
+    verify(scheduled).cancel(false);
+  }
+
   private static WebSocketSession session(String id) {
     WebSocketSession session = mock(WebSocketSession.class);
     when(session.getId()).thenReturn(id);
     return session;
   }
+
+  private static Message<byte[]> connect(
+      String sessionId, Instant expiresAt, StompCommand command) {
+    Jwt jwt =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .subject("11111111-1111-4111-8111-111111111111")
+            .claim("roles", List.of("USER"))
+            .expiresAt(expiresAt)
+            .build();
+    StompHeaderAccessor accessor = StompHeaderAccessor.create(command);
+    accessor.setSessionId(sessionId);
+    accessor.setUser(new JwtAuthenticationToken(jwt));
+    accessor.setLeaveMutable(true);
+    return MessageBuilder.createMessage(new byte[0], accessor.getMessageHeaders());
+  }
 }


