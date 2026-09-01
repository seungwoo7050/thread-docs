# 게이트웨이 신뢰 경계와 기본 거부 인가

## `feat(api): authenticate the gateway boundary`

diff --git a/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java b/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java
new file mode 100644
index 0000000..6e8466b
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java
@@ -0,0 +1,47 @@
+package com.sportsbook.betting.api;
+
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+@Component
+public class GatewayAuthFilter extends OncePerRequestFilter {
+
+  static final String SERVICE_HEADER = "X-Internal-Service";
+  static final String API_KEY_HEADER = "X-Internal-Api-Key";
+
+  private final GatewayAuthProperties credentials;
+
+  public GatewayAuthFilter(GatewayAuthProperties credentials) {
+    this.credentials = credentials;
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    return !request.getRequestURI().startsWith("/internal/v1/bets");
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    boolean trustedCaller = "gateway".equals(request.getHeader(SERVICE_HEADER));
+    boolean trustedKey = credentials.matches(request.getHeader(API_KEY_HEADER));
+    if (!trustedCaller || !trustedKey) {
+      response.setStatus(HttpServletResponse.SC_FORBIDDEN);
+      response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+      response
+          .getWriter()
+          .write(
+              "{\"type\":\"https://sportsbook/errors/forbidden\","
+                  + "\"title\":\"Forbidden\",\"status\":403,\"errorCode\":\"FORBIDDEN\"}");
+      return;
+    }
+    chain.doFilter(request, response);
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/api/GatewayAuthProperties.java b/src/main/java/com/sportsbook/betting/api/GatewayAuthProperties.java
new file mode 100644
index 0000000..1ac6746
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/GatewayAuthProperties.java
@@ -0,0 +1,45 @@
+package com.sportsbook.betting.api;
+
+import com.sportsbook.betting.client.ClientProperties;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class GatewayAuthProperties {
+
+  private final byte[] apiKey;
+
+  @Autowired
+  public GatewayAuthProperties(
+      @Value("${BETTING_GATEWAY_API_KEY}") String apiKey, ClientProperties clients) {
+    this(apiKey, clients.riskApiKey(), clients.walletApiKey());
+  }
+
+  GatewayAuthProperties(String apiKey) {
+    this(apiKey, null, null);
+  }
+
+  private GatewayAuthProperties(String apiKey, String riskKey, String walletKey) {
+    if (apiKey == null || apiKey.isBlank() || apiKey.length() < 32) {
+      throw new IllegalArgumentException(
+          "BETTING_GATEWAY_API_KEY must contain at least 32 characters");
+    }
+    if (apiKey.equals(riskKey) || apiKey.equals(walletKey)) {
+      throw new IllegalArgumentException("Gateway and dependency API keys must be distinct");
+    }
+    this.apiKey = apiKey.getBytes(StandardCharsets.UTF_8);
+  }
+
+  boolean matches(String candidate) {
+    byte[] supplied = candidate == null ? new byte[0] : candidate.getBytes(StandardCharsets.UTF_8);
+    return MessageDigest.isEqual(apiKey, supplied);
+  }
+
+  @Override
+  public String toString() {
+    return "GatewayAuthProperties[apiKey=<redacted>]";
+  }
+}


## `test(api): verify constant time gateway authentication`

diff --git a/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java b/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java
new file mode 100644
index 0000000..e5a3711
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.betting.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.betting.client.ClientProperties;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockFilterChain;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class GatewayAuthFilterTest {
+
+  private static final String SECRET = "a".repeat(32);
+
+  @Test
+  void failsStartupForMissingStrengthAndNeverRendersTheSecret() {
+    assertThatThrownBy(() -> new GatewayAuthProperties("short"))
+        .isInstanceOf(IllegalArgumentException.class);
+
+    assertThat(new GatewayAuthProperties(SECRET).toString())
+        .doesNotContain(SECRET)
+        .contains("redacted");
+  }
+
+  @Test
+  void failsStartupWhenGatewayReusesADependencyKey() {
+    ClientProperties clients =
+        new ClientProperties(
+            "http://risk", "http://wallet", null, null, "r".repeat(32), "w".repeat(32));
+
+    assertThatThrownBy(() -> new GatewayAuthProperties("r".repeat(32), clients))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("distinct");
+    assertThatThrownBy(() -> new GatewayAuthProperties("w".repeat(32), clients))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("distinct");
+  }
+
+  @Test
+  void permitsOnlyTheExactGatewayCallerAndSecret() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/internal/v1/bets");
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    request.addHeader(GatewayAuthFilter.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    new GatewayAuthFilter(new GatewayAuthProperties(SECRET))
+        .doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(200);
+  }
+
+  @Test
+  void rejectsSpoofedOrMismatchedInternalHeaders() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/internal/v1/bets");
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "client");
+    request.addHeader(GatewayAuthFilter.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    new GatewayAuthFilter(new GatewayAuthProperties(SECRET))
+        .doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(403);
+    assertThat(response.getContentAsString()).contains("FORBIDDEN");
+  }
+}


## `feat(api): enforce exact gateway ingress`

diff --git a/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java b/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java
index 6e8466b..39e5383 100644
--- a/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java
+++ b/src/main/java/com/sportsbook/betting/api/GatewayAuthFilter.java
@@ -5,6 +5,8 @@ import jakarta.servlet.ServletException;
 import jakarta.servlet.http.HttpServletRequest;
 import jakarta.servlet.http.HttpServletResponse;
 import java.io.IOException;
+import java.util.Collections;
+import java.util.List;
 import org.springframework.http.MediaType;
 import org.springframework.stereotype.Component;
 import org.springframework.web.filter.OncePerRequestFilter;
@@ -23,25 +25,60 @@ public class GatewayAuthFilter extends OncePerRequestFilter {
 
   @Override
   protected boolean shouldNotFilter(HttpServletRequest request) {
-    return !request.getRequestURI().startsWith("/internal/v1/bets");
+    String path = businessPath(request);
+    return !path.startsWith("/internal/") && !path.startsWith("/api/");
   }
 
   @Override
   protected void doFilterInternal(
       HttpServletRequest request, HttpServletResponse response, FilterChain chain)
       throws ServletException, IOException {
-    boolean trustedCaller = "gateway".equals(request.getHeader(SERVICE_HEADER));
-    boolean trustedKey = credentials.matches(request.getHeader(API_KEY_HEADER));
-    if (!trustedCaller || !trustedKey) {
-      response.setStatus(HttpServletResponse.SC_FORBIDDEN);
-      response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
-      response
-          .getWriter()
-          .write(
-              "{\"type\":\"https://sportsbook/errors/forbidden\","
-                  + "\"title\":\"Forbidden\",\"status\":403,\"errorCode\":\"FORBIDDEN\"}");
+    List<String> callers = Collections.list(request.getHeaders(SERVICE_HEADER));
+    List<String> keys = Collections.list(request.getHeaders(API_KEY_HEADER));
+    boolean exactKey = keys.size() == 1 && credentials.matches(keys.get(0));
+    if (!exactKey || callers.size() != 1) {
+      reject(response, HttpServletResponse.SC_UNAUTHORIZED, "UNAUTHORIZED");
+      return;
+    }
+    boolean trustedCaller = "gateway".equals(callers.get(0));
+    String path = businessPath(request);
+    String method = request.getMethod();
+    boolean collection = path.equals("/internal/v1/bets");
+    boolean item =
+        path.matches(
+            "/internal/v1/bets/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}");
+    boolean allowedRoute =
+        (collection && (method.equals("GET") || method.equals("POST")))
+            || (item && method.equals("GET"));
+    if (!trustedCaller || !allowedRoute) {
+      reject(response, HttpServletResponse.SC_FORBIDDEN, "FORBIDDEN");
       return;
     }
     chain.doFilter(request, response);
   }
+
+  private static String businessPath(HttpServletRequest request) {
+    String uri = request.getRequestURI();
+    String context = request.getContextPath();
+    return context.isEmpty() || !uri.startsWith(context) ? uri : uri.substring(context.length());
+  }
+
+  private static void reject(HttpServletResponse response, int status, String code)
+      throws IOException {
+    response.setStatus(status);
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    response
+        .getWriter()
+        .write(
+            "{\"type\":\"https://sportsbook/errors/"
+                + code.toLowerCase()
+                + "\","
+                + "\"title\":\""
+                + code
+                + "\",\"status\":"
+                + status
+                + ",\"errorCode\":\""
+                + code
+                + "\"}");
+  }
 }


## `test(api): verify default deny gateway boundary`

diff --git a/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java b/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java
index e5a3711..ed53057 100644
--- a/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java
+++ b/src/test/java/com/sportsbook/betting/api/GatewayAuthFilterTest.java
@@ -17,6 +17,8 @@ class GatewayAuthFilterTest {
   void failsStartupForMissingStrengthAndNeverRendersTheSecret() {
     assertThatThrownBy(() -> new GatewayAuthProperties("short"))
         .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new GatewayAuthProperties(" ".repeat(32)))
+        .isInstanceOf(IllegalArgumentException.class);
 
     assertThat(new GatewayAuthProperties(SECRET).toString())
         .doesNotContain(SECRET)
@@ -63,4 +65,74 @@ class GatewayAuthFilterTest {
     assertThat(response.getStatus()).isEqualTo(403);
     assertThat(response.getContentAsString()).contains("FORBIDDEN");
   }
+
+  @Test
+  void returnsUnauthorizedForMissingOrDuplicateCredentials() throws Exception {
+    MockHttpServletRequest missing = new MockHttpServletRequest("GET", "/internal/v1/bets");
+    missing.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    MockHttpServletResponse missingResponse = new MockHttpServletResponse();
+    GatewayAuthFilter filter = new GatewayAuthFilter(new GatewayAuthProperties(SECRET));
+
+    filter.doFilter(missing, missingResponse, new MockFilterChain());
+
+    assertThat(missingResponse.getStatus()).isEqualTo(401);
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/internal/v1/bets");
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    request.addHeader(GatewayAuthFilter.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentAsString()).contains("UNAUTHORIZED");
+  }
+
+  @Test
+  void deniesUnlistedBusinessRoutesEvenForTheGateway() throws Exception {
+    MockHttpServletRequest request =
+        new MockHttpServletRequest("POST", "/internal/v1/admin/reprice");
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    request.addHeader(GatewayAuthFilter.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    new GatewayAuthFilter(new GatewayAuthProperties(SECRET))
+        .doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(403);
+  }
+
+  @Test
+  void deniesWrongMethodsAndNestedPathsBeforeControllerMapping() throws Exception {
+    GatewayAuthFilter filter = new GatewayAuthFilter(new GatewayAuthProperties(SECRET));
+    MockHttpServletRequest wrongMethod = authenticated("PUT", "/internal/v1/bets");
+    MockHttpServletResponse wrongMethodResponse = new MockHttpServletResponse();
+    MockHttpServletRequest nested = authenticated("GET", "/internal/v1/bets/one/export");
+    MockHttpServletResponse nestedResponse = new MockHttpServletResponse();
+
+    filter.doFilter(wrongMethod, wrongMethodResponse, new MockFilterChain());
+    filter.doFilter(nested, nestedResponse, new MockFilterChain());
+
+    assertThat(wrongMethodResponse.getStatus()).isEqualTo(403);
+    assertThat(nestedResponse.getStatus()).isEqualTo(403);
+  }
+
+  @Test
+  void matchesRoutesBelowTheConfiguredContextPath() throws Exception {
+    MockHttpServletRequest request = authenticated("GET", "/sportsbook/internal/v1/bets");
+    request.setContextPath("/sportsbook");
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    new GatewayAuthFilter(new GatewayAuthProperties(SECRET))
+        .doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(200);
+  }
+
+  private static MockHttpServletRequest authenticated(String method, String path) {
+    MockHttpServletRequest request = new MockHttpServletRequest(method, path);
+    request.addHeader(GatewayAuthFilter.SERVICE_HEADER, "gateway");
+    request.addHeader(GatewayAuthFilter.API_KEY_HEADER, SECRET);
+    return request;
+  }
 }
