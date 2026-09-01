# 트레이스 전파, 다운스트림 실패 정규화, 응답 계약 보존

## `feat(routing): bound downstream failures`

diff --git a/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java b/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java
index 5884681..a4c1b86 100644
--- a/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java
+++ b/src/main/java/com/sportsbook/gateway/error/GatewayProblemWriter.java
@@ -32,7 +32,7 @@ public final class GatewayProblemWriter {
     objectMapper.writeValue(response.getOutputStream(), problem(request, error));
   }
 
-  ProblemDetail problem(HttpServletRequest request, GatewayErrorCode error) {
+  public ProblemDetail problem(HttpServletRequest request, GatewayErrorCode error) {
     return new ProblemDetail(
         error.type(),
         error.title(),
diff --git a/src/main/java/com/sportsbook/gateway/routing/DownstreamFailureBoundary.java b/src/main/java/com/sportsbook/gateway/routing/DownstreamFailureBoundary.java
new file mode 100644
index 0000000..441326c
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/DownstreamFailureBoundary.java
@@ -0,0 +1,57 @@
+package com.sportsbook.gateway.routing;
+
+import com.sportsbook.gateway.error.GatewayErrorCode;
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import java.net.SocketTimeoutException;
+import java.net.http.HttpTimeoutException;
+import java.util.concurrent.TimeoutException;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.ResourceAccessException;
+import org.springframework.web.servlet.function.HandlerFilterFunction;
+import org.springframework.web.servlet.function.HandlerFunction;
+import org.springframework.web.servlet.function.ServerRequest;
+import org.springframework.web.servlet.function.ServerResponse;
+
+@Component
+public final class DownstreamFailureBoundary
+    implements HandlerFilterFunction<ServerResponse, ServerResponse> {
+
+  private final GatewayProblemWriter problems;
+
+  public DownstreamFailureBoundary(GatewayProblemWriter problems) {
+    this.problems = problems;
+  }
+
+  @Override
+  public ServerResponse filter(ServerRequest request, HandlerFunction<ServerResponse> downstream)
+      throws Exception {
+    try {
+      return downstream.handle(request);
+    } catch (ResourceAccessException failure) {
+      GatewayErrorCode error =
+          hasCause(
+                  failure,
+                  SocketTimeoutException.class,
+                  HttpTimeoutException.class,
+                  TimeoutException.class)
+              ? GatewayErrorCode.GATEWAY_TIMEOUT
+              : GatewayErrorCode.GATEWAY_BAD_GATEWAY;
+      return ServerResponse.status(error.status())
+          .contentType(MediaType.APPLICATION_PROBLEM_JSON)
+          .body(problems.problem(request.servletRequest(), error));
+    }
+  }
+
+  @SafeVarargs
+  private static boolean hasCause(Throwable failure, Class<? extends Throwable>... causeTypes) {
+    for (Throwable cause = failure; cause != null; cause = cause.getCause()) {
+      for (Class<? extends Throwable> causeType : causeTypes) {
+        if (causeType.isInstance(cause)) {
+          return true;
+        }
+      }
+    }
+    return false;
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index b2f045e..e5c4d5d 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -1,6 +1,12 @@
 spring:
   application:
     name: gateway
+  cloud:
+    gateway:
+      mvc:
+        http-client:
+          connect-timeout: ${GATEWAY_DOWNSTREAM_CONNECT_TIMEOUT:500ms}
+          read-timeout: ${GATEWAY_DOWNSTREAM_READ_TIMEOUT:3s}
   data:
     redis:
       host: ${GATEWAY_REDIS_HOST:localhost}


## `test(routing): verify upstream failure responses`

diff --git a/src/test/java/com/sportsbook/gateway/routing/DownstreamFailureBoundaryTest.java b/src/test/java/com/sportsbook/gateway/routing/DownstreamFailureBoundaryTest.java
new file mode 100644
index 0000000..24a89bf
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/DownstreamFailureBoundaryTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.gateway.routing;
+
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+import static org.springframework.web.servlet.function.RequestPredicates.GET;
+import static org.springframework.web.servlet.function.RouterFunctions.route;
+
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import com.sportsbook.gateway.security.SecurityConfig;
+import java.io.IOException;
+import java.net.ConnectException;
+import java.util.concurrent.TimeoutException;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.boot.test.context.TestConfiguration;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Import;
+import org.springframework.http.MediaType;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.web.client.ResourceAccessException;
+import org.springframework.web.servlet.function.RouterFunction;
+import org.springframework.web.servlet.function.ServerResponse;
+
+@WebMvcTest
+@Import({SecurityConfig.class, GatewayProblemWriter.class, DownstreamFailureBoundary.class})
+class DownstreamFailureBoundaryTest {
+
+  @Autowired MockMvc mvc;
+  @MockBean JwtDecoder jwtDecoder;
+
+  @TestConfiguration(proxyBeanMethods = false)
+  static class FailureRoutes {
+
+    @Bean
+    RouterFunction<ServerResponse> failureRoute(DownstreamFailureBoundary failures) {
+      return route(
+              GET("/api/v1/events/{failure}"),
+              request -> {
+                if ("timeout".equals(request.pathVariable("failure"))) {
+                  throw new ResourceAccessException(
+                      "timed out", new IOException(new TimeoutException()));
+                }
+                throw new ResourceAccessException("unavailable", new ConnectException());
+              })
+          .filter(failures);
+    }
+  }
+
+  @Test
+  void returnsPublicGatewayTimeoutWithoutReauthentication() throws Exception {
+    mvc.perform(get("/api/v1/events/timeout"))
+        .andExpect(status().isGatewayTimeout())
+        .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$.errorCode").value("GATEWAY_TIMEOUT"));
+  }
+
+  @Test
+  void returnsPublicBadGatewayWithoutReauthentication() throws Exception {
+    mvc.perform(get("/api/v1/events/unavailable"))
+        .andExpect(status().isBadGateway())
+        .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$.errorCode").value("GATEWAY_BAD_GATEWAY"));
+  }
+}


## `feat(routing): propagate trace context`

diff --git a/src/main/java/com/sportsbook/gateway/routing/TraceForwarding.java b/src/main/java/com/sportsbook/gateway/routing/TraceForwarding.java
new file mode 100644
index 0000000..148ae77
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/TraceForwarding.java
@@ -0,0 +1,65 @@
+package com.sportsbook.gateway.routing;
+
+import io.micrometer.tracing.Span;
+import io.micrometer.tracing.TraceContext;
+import io.micrometer.tracing.Tracer;
+import java.util.List;
+import java.util.regex.Pattern;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.stereotype.Component;
+import org.springframework.web.servlet.function.ServerRequest;
+
+@Component
+public class TraceForwarding {
+
+  private static final String TRACEPARENT = "traceparent";
+  private static final Pattern VALID_TRACEPARENT =
+      Pattern.compile("00-[0-9a-f]{32}-[0-9a-f]{16}-(00|01)");
+  private static final String ZERO_TRACE_ID = "00000000000000000000000000000000";
+  private static final String ZERO_SPAN_ID = "0000000000000000";
+
+  private final ObjectProvider<Tracer> tracer;
+
+  public TraceForwarding(ObjectProvider<Tracer> tracer) {
+    this.tracer = tracer;
+  }
+
+  public ServerRequest apply(ServerRequest request) {
+    List<String> inbound = request.headers().header(TRACEPARENT);
+    if (inbound.size() == 1 && isValid(inbound.get(0))) {
+      return request;
+    }
+    ServerRequest sanitized = request;
+    if (!inbound.isEmpty()) {
+      sanitized =
+          ServerRequest.from(request).headers(headers -> headers.remove(TRACEPARENT)).build();
+    }
+    String traceparent = currentTraceparent();
+    if (!isValid(traceparent)) {
+      return sanitized;
+    }
+    return ServerRequest.from(sanitized).header(TRACEPARENT, traceparent).build();
+  }
+
+  private boolean isValid(String value) {
+    if (value == null || !VALID_TRACEPARENT.matcher(value).matches()) {
+      return false;
+    }
+    return !value.regionMatches(3, ZERO_TRACE_ID, 0, ZERO_TRACE_ID.length())
+        && !value.regionMatches(36, ZERO_SPAN_ID, 0, ZERO_SPAN_ID.length());
+  }
+
+  private String currentTraceparent() {
+    Tracer activeTracer = tracer.getIfAvailable();
+    if (activeTracer == null) {
+      return null;
+    }
+    Span span = activeTracer.currentSpan();
+    if (span == null) {
+      return null;
+    }
+    TraceContext context = span.context();
+    String flags = Boolean.TRUE.equals(context.sampled()) ? "01" : "00";
+    return "00-" + context.traceId() + "-" + context.spanId() + "-" + flags;
+  }
+}


## `test(routing): verify trace propagation`

diff --git a/src/test/java/com/sportsbook/gateway/routing/TraceForwardingTest.java b/src/test/java/com/sportsbook/gateway/routing/TraceForwardingTest.java
new file mode 100644
index 0000000..dc996ca
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/TraceForwardingTest.java
@@ -0,0 +1,79 @@
+package com.sportsbook.gateway.routing;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import io.micrometer.tracing.Span;
+import io.micrometer.tracing.TraceContext;
+import io.micrometer.tracing.Tracer;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.web.servlet.function.ServerRequest;
+
+class TraceForwardingTest {
+
+  private static final String SAMPLED = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01";
+  private static final String UNSAMPLED = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00";
+
+  @Test
+  void preservesSingleValidSampledAndUnsampledValues() {
+    TraceForwarding forwarding = new TraceForwarding(provider(null));
+
+    for (String traceparent : List.of(SAMPLED, UNSAMPLED)) {
+      assertThat(forwarding.apply(request(traceparent)).headers().firstHeader("traceparent"))
+          .isEqualTo(traceparent);
+    }
+  }
+
+  @Test
+  void removesMalformedAndAllZeroValuesWithoutAnActiveSpan() {
+    TraceForwarding forwarding = new TraceForwarding(provider(null));
+    List<String> invalid =
+        List.of(
+            "00-4BF92F3577B34DA6A3CE929D0E0E4736-00f067aa0ba902b7-01",
+            "00-00000000000000000000000000000000-00f067aa0ba902b7-01",
+            "00-4bf92f3577b34da6a3ce929d0e0e4736-0000000000000000-01",
+            "invalid");
+
+    for (String traceparent : invalid) {
+      assertThat(forwarding.apply(request(traceparent)).headers().header("traceparent")).isEmpty();
+    }
+    MockHttpServletRequest servletRequest = new MockHttpServletRequest();
+    servletRequest.addHeader("traceparent", SAMPLED);
+    servletRequest.addHeader("traceparent", UNSAMPLED);
+    ServerRequest result = forwarding.apply(ServerRequest.create(servletRequest, List.of()));
+    assertThat(result.headers().header("traceparent")).isEmpty();
+  }
+
+  @Test
+  void createsAValidValueFromTheActiveSpan() {
+    TraceContext context = mock(TraceContext.class);
+    when(context.traceId()).thenReturn("4bf92f3577b34da6a3ce929d0e0e4736");
+    when(context.spanId()).thenReturn("00f067aa0ba902b7");
+    when(context.sampled()).thenReturn(true);
+    Span span = mock(Span.class);
+    when(span.context()).thenReturn(context);
+    Tracer tracer = mock(Tracer.class);
+    when(tracer.currentSpan()).thenReturn(span);
+
+    ServerRequest result = new TraceForwarding(provider(tracer)).apply(request("invalid"));
+
+    assertThat(result.headers().firstHeader("traceparent")).isEqualTo(SAMPLED);
+  }
+
+  private static ServerRequest request(String traceparent) {
+    MockHttpServletRequest servletRequest = new MockHttpServletRequest();
+    servletRequest.addHeader("traceparent", traceparent);
+    return ServerRequest.create(servletRequest, List.of());
+  }
+
+  @SuppressWarnings("unchecked")
+  private static ObjectProvider<Tracer> provider(Tracer tracer) {
+    ObjectProvider<Tracer> provider = mock(ObjectProvider.class);
+    when(provider.getIfAvailable()).thenReturn(tracer);
+    return provider;
+  }
+}


## `test(routing): preserve downstream response contracts`

diff --git a/src/test/java/com/sportsbook/gateway/routing/ClosedPortProxyTest.java b/src/test/java/com/sportsbook/gateway/routing/ClosedPortProxyTest.java
new file mode 100644
index 0000000..9ee7d47
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/ClosedPortProxyTest.java
@@ -0,0 +1,61 @@
+package com.sportsbook.gateway.routing;
+
+import static com.jayway.jsonpath.JsonPath.read;
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.net.ServerSocket;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
+    properties = {
+      "gateway.ratelimit.enabled=false",
+      "gateway.downstream.wallet.api-key=fixture-wallet-key-32-characters-long"
+    })
+class ClosedPortProxyTest {
+
+  private static final ServerSocket RESERVED_PORT = reservedPort();
+
+  @Autowired TestRestTemplate http;
+  @MockBean JwtDecoder jwtDecoder;
+
+  @DynamicPropertySource
+  static void downstream(DynamicPropertyRegistry registry) {
+    registry.add(
+        "gateway.downstream.odds-feed-uri",
+        () -> "http://127.0.0.1:" + RESERVED_PORT.getLocalPort());
+  }
+
+  @Test
+  void mapsAnonymousConnectionRefusalWithoutReauthentication() throws IOException {
+    RESERVED_PORT.close();
+    ResponseEntity<String> response = http.getForEntity("/api/v1/events/unavailable", String.class);
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_GATEWAY);
+    assertThat(response.getHeaders().getContentType())
+        .isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(read(response.getBody(), "$.status").toString()).isEqualTo("502");
+    assertThat(read(response.getBody(), "$.errorCode").toString()).isEqualTo("GATEWAY_BAD_GATEWAY");
+    assertThat(read(response.getBody(), "$.instance").toString())
+        .isEqualTo("/api/v1/events/unavailable");
+  }
+
+  private static ServerSocket reservedPort() {
+    try {
+      return new ServerSocket(0);
+    } catch (IOException failure) {
+      throw new ExceptionInInitializerError(failure);
+    }
+  }
+}
diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 8e3c796..1c820e3 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -8,6 +8,7 @@ import static com.github.tomakehurst.wiremock.client.WireMock.post;
 import static com.github.tomakehurst.wiremock.client.WireMock.postRequestedFor;
 import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
 import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;
+import static com.jayway.jsonpath.JsonPath.read;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
@@ -222,6 +223,43 @@ class GatewayRoutingIntegrationTest {
     }
   }
 
+  @Test
+  void preservesDownstreamStatusHeadersAndBody() {
+    String problem = "{\"errorCode\":\"ODDS_MARKET_CLOSED\",\"detail\":\"closed\"}";
+    DOWNSTREAM.stubFor(
+        get(urlPathEqualTo("/api/v1/events/problem"))
+            .willReturn(
+                aResponse()
+                    .withStatus(409)
+                    .withHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_PROBLEM_JSON_VALUE)
+                    .withHeader(HttpHeaders.LOCATION, "/api/v1/events/next")
+                    .withHeader(HttpHeaders.RETRY_AFTER, "7")
+                    .withBody(problem)));
+
+    ResponseEntity<String> response = http.getForEntity("/api/v1/events/problem", String.class);
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CONFLICT);
+    assertThat(response.getHeaders().getContentType())
+        .isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(response.getHeaders().getLocation()).hasToString("/api/v1/events/next");
+    assertThat(response.getHeaders().getFirst(HttpHeaders.RETRY_AFTER)).isEqualTo("7");
+    assertThat(response.getBody()).isEqualTo(problem);
+  }
+
+  @Test
+  void mapsPublicProxyTimeoutWithoutReauthentication() {
+    DOWNSTREAM.stubFor(
+        get(urlPathEqualTo("/api/v1/events/timeout"))
+            .willReturn(aResponse().withFixedDelay(4_000).withStatus(200)));
+
+    ResponseEntity<String> timeout = http.getForEntity("/api/v1/events/timeout", String.class);
+
+    assertThat(timeout.getStatusCode()).isEqualTo(HttpStatus.GATEWAY_TIMEOUT);
+    assertThat(timeout.getHeaders().getContentType()).isEqualTo(MediaType.APPLICATION_PROBLEM_JSON);
+    assertThat(read(timeout.getBody(), "$.status").toString()).isEqualTo("504");
+    assertThat(read(timeout.getBody(), "$.errorCode").toString()).isEqualTo("GATEWAY_TIMEOUT");
+  }
+
   @Test
   void rejectsUnsafeBettingBaseUris() {
     assertThatThrownBy(() -> bettingUri("ftp://betting.internal"))
