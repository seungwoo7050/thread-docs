# 신뢰 헤더 차단과 다운스트림 신원·자격 증명 재구성

## `feat(security): isolate inbound trust headers`

diff --git a/src/main/java/com/sportsbook/gateway/security/GatewayHeaders.java b/src/main/java/com/sportsbook/gateway/security/GatewayHeaders.java
new file mode 100644
index 0000000..8ce0ab6
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/GatewayHeaders.java
@@ -0,0 +1,11 @@
+package com.sportsbook.gateway.security;
+
+public final class GatewayHeaders {
+
+  public static final String USER_ID = "X-User-Id";
+  public static final String USER_ROLES = "X-User-Roles";
+  public static final String INTERNAL_SERVICE = "X-Internal-Service";
+  public static final String INTERNAL_API_KEY = "X-Internal-Api-Key";
+
+  private GatewayHeaders() {}
+}
diff --git a/src/main/java/com/sportsbook/gateway/security/TrustedHeaderFilter.java b/src/main/java/com/sportsbook/gateway/security/TrustedHeaderFilter.java
new file mode 100644
index 0000000..82d2a43
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/TrustedHeaderFilter.java
@@ -0,0 +1,77 @@
+package com.sportsbook.gateway.security;
+
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletRequestWrapper;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.util.Collections;
+import java.util.Enumeration;
+import java.util.List;
+import org.springframework.core.Ordered;
+import org.springframework.core.annotation.Order;
+import org.springframework.stereotype.Component;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+@Component
+@Order(Ordered.HIGHEST_PRECEDENCE)
+public final class TrustedHeaderFilter extends OncePerRequestFilter {
+
+  private static final List<String> HIDDEN =
+      List.of(
+          GatewayHeaders.USER_ID,
+          GatewayHeaders.USER_ROLES,
+          GatewayHeaders.INTERNAL_SERVICE,
+          GatewayHeaders.INTERNAL_API_KEY);
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    chain.doFilter(new TrustedHeaderRequest(request), response);
+  }
+
+  @Override
+  protected boolean shouldNotFilterErrorDispatch() {
+    return false;
+  }
+
+  @Override
+  protected void doFilterNestedErrorDispatch(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    doFilterInternal(request, response, chain);
+  }
+
+  private static boolean hidden(String name) {
+    return name != null && HIDDEN.stream().anyMatch(name::equalsIgnoreCase);
+  }
+
+  private static final class TrustedHeaderRequest extends HttpServletRequestWrapper {
+
+    private TrustedHeaderRequest(HttpServletRequest request) {
+      super(request);
+    }
+
+    @Override
+    public String getHeader(String name) {
+      return hidden(name) ? null : super.getHeader(name);
+    }
+
+    @Override
+    public Enumeration<String> getHeaders(String name) {
+      return hidden(name) ? Collections.emptyEnumeration() : super.getHeaders(name);
+    }
+
+    @Override
+    public Enumeration<String> getHeaderNames() {
+      Enumeration<String> names = super.getHeaderNames();
+      if (names == null) {
+        return null;
+      }
+      return Collections.enumeration(
+          Collections.list(names).stream().filter(name -> !hidden(name)).toList());
+    }
+  }
+}


## `test(security): reject spoofed trust headers`

diff --git a/src/test/java/com/sportsbook/gateway/security/TrustedHeaderFilterTest.java b/src/test/java/com/sportsbook/gateway/security/TrustedHeaderFilterTest.java
new file mode 100644
index 0000000..14dcc17
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/security/TrustedHeaderFilterTest.java
@@ -0,0 +1,74 @@
+package com.sportsbook.gateway.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import jakarta.servlet.DispatcherType;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.Collections;
+import java.util.List;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class TrustedHeaderFilterTest {
+
+  private static final List<String> TRUST_HEADERS =
+      List.of(
+          GatewayHeaders.USER_ID,
+          GatewayHeaders.USER_ROLES,
+          GatewayHeaders.INTERNAL_SERVICE,
+          GatewayHeaders.INTERNAL_API_KEY);
+
+  @Test
+  void hidesEveryTrustHeaderAcrossServletHeaderApis() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader("x-user-id", "spoofed-user");
+    request.addHeader("X-USER-ROLES", "ADMIN");
+    request.addHeader("x-Internal-Service", "attacker");
+    request.addHeader("X-Internal-Api-Key", "first");
+    request.addHeader("X-Internal-Api-Key", "second");
+
+    HttpServletRequest filtered = filter(request);
+
+    for (String name : TRUST_HEADERS) {
+      assertThat(filtered.getHeader(name)).isNull();
+      assertThat(Collections.list(filtered.getHeaders(name.toLowerCase()))).isEmpty();
+    }
+    assertThat(Collections.list(filtered.getHeaderNames()))
+        .noneMatch(name -> TRUST_HEADERS.stream().anyMatch(name::equalsIgnoreCase));
+  }
+
+  @Test
+  void retainsAuthorizationAndUnrelatedHeaders() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader("Authorization", "Bearer signed-token");
+    request.addHeader("X-Request-Id", "request-1");
+
+    HttpServletRequest filtered = filter(request);
+
+    assertThat(filtered.getHeader("Authorization")).isEqualTo("Bearer signed-token");
+    assertThat(filtered.getHeader("X-Request-Id")).isEqualTo("request-1");
+    assertThat(Collections.list(filtered.getHeaderNames()))
+        .containsExactlyInAnyOrder("Authorization", "X-Request-Id");
+  }
+
+  @Test
+  void stripsTrustHeadersAgainOnErrorDispatch() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.setDispatcherType(DispatcherType.ERROR);
+    request.addHeader(GatewayHeaders.INTERNAL_API_KEY, "must-not-survive");
+
+    assertThat(filter(request).getHeader(GatewayHeaders.INTERNAL_API_KEY)).isNull();
+  }
+
+  private static HttpServletRequest filter(MockHttpServletRequest request) throws Exception {
+    AtomicReference<HttpServletRequest> captured = new AtomicReference<>();
+    new TrustedHeaderFilter()
+        .doFilter(
+            request,
+            new MockHttpServletResponse(),
+            (filtered, response) -> captured.set((HttpServletRequest) filtered));
+    return captured.get();
+  }
+}


## `feat(routing): establish downstream identity boundary`

diff --git a/src/main/java/com/sportsbook/gateway/routing/DownstreamRequestSanitizer.java b/src/main/java/com/sportsbook/gateway/routing/DownstreamRequestSanitizer.java
new file mode 100644
index 0000000..3e54292
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/DownstreamRequestSanitizer.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.routing;
+
+import com.sportsbook.gateway.security.GatewayHeaders;
+import org.springframework.http.HttpHeaders;
+import org.springframework.stereotype.Component;
+import org.springframework.web.servlet.function.ServerRequest;
+
+@Component
+public final class DownstreamRequestSanitizer {
+
+  public ServerRequest apply(ServerRequest request) {
+    return ServerRequest.from(request)
+        .headers(
+            headers -> {
+              headers.remove(HttpHeaders.AUTHORIZATION);
+              headers.remove(GatewayHeaders.USER_ID);
+              headers.remove(GatewayHeaders.USER_ROLES);
+              headers.remove(GatewayHeaders.INTERNAL_SERVICE);
+              headers.remove(GatewayHeaders.INTERNAL_API_KEY);
+            })
+        .build();
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/routing/IdentityForwarding.java b/src/main/java/com/sportsbook/gateway/routing/IdentityForwarding.java
new file mode 100644
index 0000000..39f86d3
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/IdentityForwarding.java
@@ -0,0 +1,43 @@
+package com.sportsbook.gateway.routing;
+
+import com.sportsbook.gateway.security.GatewayHeaders;
+import java.util.List;
+import java.util.Optional;
+import org.springframework.security.core.Authentication;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.stereotype.Component;
+import org.springframework.web.servlet.function.ServerRequest;
+
+@Component
+public final class IdentityForwarding {
+
+  public ServerRequest apply(ServerRequest request) {
+    Jwt jwt = currentJwt();
+    if (jwt == null) {
+      return request;
+    }
+    ServerRequest.Builder forwarded =
+        ServerRequest.from(request).header(GatewayHeaders.USER_ID, jwt.getSubject());
+    List<String> roles = jwt.getClaimAsStringList("roles");
+    if (roles != null && !roles.isEmpty()) {
+      forwarded.header(GatewayHeaders.USER_ROLES, String.join(",", roles));
+    }
+    return forwarded.build();
+  }
+
+  public Optional<String> currentSubject() {
+    Jwt jwt = currentJwt();
+    return jwt == null ? Optional.empty() : Optional.of(jwt.getSubject());
+  }
+
+  private static Jwt currentJwt() {
+    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
+    if (authentication != null
+        && authentication.isAuthenticated()
+        && authentication.getPrincipal() instanceof Jwt jwt) {
+      return jwt;
+    }
+    return null;
+  }
+}


## `test(routing): verify proxied credential isolation`

diff --git a/src/test/java/com/sportsbook/gateway/routing/DownstreamIdentityBoundaryTest.java b/src/test/java/com/sportsbook/gateway/routing/DownstreamIdentityBoundaryTest.java
new file mode 100644
index 0000000..813faee
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/DownstreamIdentityBoundaryTest.java
@@ -0,0 +1,71 @@
+package com.sportsbook.gateway.routing;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.gateway.security.GatewayHeaders;
+import java.time.Instant;
+import java.util.List;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpHeaders;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.web.servlet.function.ServerRequest;
+
+class DownstreamIdentityBoundaryTest {
+
+  private static final String USER_ID = "11111111-1111-4111-8111-111111111111";
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void removesCallerCredentialsBeforeProxying() {
+    MockHttpServletRequest servletRequest = new MockHttpServletRequest();
+    servletRequest.addHeader(HttpHeaders.AUTHORIZATION, "Bearer external-token");
+    servletRequest.addHeader(GatewayHeaders.USER_ID, "spoofed-user");
+    servletRequest.addHeader(GatewayHeaders.USER_ROLES, "ADMIN");
+    servletRequest.addHeader(GatewayHeaders.INTERNAL_SERVICE, "spoofed-service");
+    servletRequest.addHeader(GatewayHeaders.INTERNAL_API_KEY, "spoofed-key");
+    servletRequest.addHeader("traceparent", "00-trace-span-01");
+
+    ServerRequest sanitized = new DownstreamRequestSanitizer().apply(request(servletRequest));
+
+    assertThat(sanitized.headers().firstHeader(HttpHeaders.AUTHORIZATION)).isNull();
+    assertThat(sanitized.headers().firstHeader(GatewayHeaders.USER_ID)).isNull();
+    assertThat(sanitized.headers().firstHeader(GatewayHeaders.USER_ROLES)).isNull();
+    assertThat(sanitized.headers().firstHeader(GatewayHeaders.INTERNAL_SERVICE)).isNull();
+    assertThat(sanitized.headers().firstHeader(GatewayHeaders.INTERNAL_API_KEY)).isNull();
+    assertThat(sanitized.headers().firstHeader("traceparent")).isEqualTo("00-trace-span-01");
+  }
+
+  @Test
+  void forwardsOnlyIdentityDerivedFromTheVerifiedJwt() {
+    Jwt jwt =
+        Jwt.withTokenValue("verified-token")
+            .header("alg", "RS256")
+            .subject(USER_ID)
+            .claim("roles", List.of("USER", "TRADER"))
+            .issuedAt(Instant.now().minusSeconds(1))
+            .expiresAt(Instant.now().plusSeconds(60))
+            .build();
+    SecurityContextHolder.getContext()
+        .setAuthentication(new JwtAuthenticationToken(jwt, List.of()));
+
+    ServerRequest forwarded =
+        new IdentityForwarding()
+            .apply(new DownstreamRequestSanitizer().apply(request(new MockHttpServletRequest())));
+
+    assertThat(forwarded.headers().firstHeader(GatewayHeaders.USER_ID)).isEqualTo(USER_ID);
+    assertThat(forwarded.headers().firstHeader(GatewayHeaders.USER_ROLES)).isEqualTo("USER,TRADER");
+    assertThat(new IdentityForwarding().currentSubject()).contains(USER_ID);
+  }
+
+  private static ServerRequest request(MockHttpServletRequest servletRequest) {
+    return ServerRequest.create(servletRequest, List.of());
+  }
+}


## `feat(routing): route betting requests`

diff --git a/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java b/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java
new file mode 100644
index 0000000..eccb460
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java
@@ -0,0 +1,24 @@
+package com.sportsbook.gateway.routing;
+
+import java.net.URI;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "gateway.downstream")
+public record BettingDownstreamProperties(URI bettingUri) {
+
+  public BettingDownstreamProperties {
+    String scheme = bettingUri == null ? null : bettingUri.getScheme();
+    String path = bettingUri == null ? null : bettingUri.getRawPath();
+    if (bettingUri == null
+        || !bettingUri.isAbsolute()
+        || bettingUri.isOpaque()
+        || bettingUri.getHost() == null
+        || !("http".equals(scheme) || "https".equals(scheme))
+        || bettingUri.getUserInfo() != null
+        || bettingUri.getRawQuery() != null
+        || bettingUri.getRawFragment() != null
+        || !(path == null || path.isEmpty() || "/".equals(path))) {
+      throw new IllegalArgumentException("gateway.downstream.betting-uri must be an HTTP base URI");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java b/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java
new file mode 100644
index 0000000..177522e
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java
@@ -0,0 +1,66 @@
+package com.sportsbook.gateway.routing;
+
+import static org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions.route;
+import static org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions.http;
+import static org.springframework.web.servlet.function.RequestPredicates.GET;
+import static org.springframework.web.servlet.function.RequestPredicates.POST;
+
+import java.net.URI;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.web.servlet.function.RouterFunction;
+import org.springframework.web.servlet.function.ServerRequest;
+import org.springframework.web.servlet.function.ServerResponse;
+import org.springframework.web.util.UriComponentsBuilder;
+
+@Configuration
+@EnableConfigurationProperties(BettingDownstreamProperties.class)
+public class BettingRoutes {
+
+  private final BettingDownstreamProperties downstream;
+  private final DownstreamRequestSanitizer sanitizer;
+  private final IdentityForwarding identity;
+  private final TraceForwarding trace;
+  private final DownstreamFailureBoundary failures;
+
+  public BettingRoutes(
+      BettingDownstreamProperties downstream,
+      DownstreamRequestSanitizer sanitizer,
+      IdentityForwarding identity,
+      TraceForwarding trace,
+      DownstreamFailureBoundary failures) {
+    this.downstream = downstream;
+    this.sanitizer = sanitizer;
+    this.identity = identity;
+    this.trace = trace;
+    this.failures = failures;
+  }
+
+  @Bean
+  RouterFunction<ServerResponse> bettingRoute() {
+    return route("betting")
+        .route(
+            POST("/api/v1/bets").or(GET("/api/v1/bets")).or(GET("/api/v1/bets/{betId}")),
+            http(downstream.bettingUri().toString()))
+        .before(sanitizer::apply)
+        .before(identity::apply)
+        .before(trace::apply)
+        .before(this::rewrite)
+        .filter(failures)
+        .build();
+  }
+
+  private ServerRequest rewrite(ServerRequest request) {
+    String path = request.uri().getRawPath().replaceFirst("^/api/v1/bets", "/internal/v1/bets");
+    URI uri = UriComponentsBuilder.fromUri(request.uri()).replacePath(path).build(true).toUri();
+    ServerRequest.Builder forwarded = ServerRequest.from(request).uri(uri);
+    if (request.method() == HttpMethod.GET && path.equals("/internal/v1/bets")) {
+      identity
+          .currentSubject()
+          .ifPresent(subject -> forwarded.params(params -> params.set("userId", subject)));
+    }
+    return forwarded.build();
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index e5c4d5d..6425d3e 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -24,6 +24,8 @@ server:
   shutdown: graceful
 
 gateway:
+  downstream:
+    betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}
   security:
     jwt:
       public-key: ${GATEWAY_JWT_PUBLIC_KEY:}


## `feat(routing): validate wallet service credentials`

diff --git a/src/main/java/com/sportsbook/gateway/routing/WalletDownstreamProperties.java b/src/main/java/com/sportsbook/gateway/routing/WalletDownstreamProperties.java
new file mode 100644
index 0000000..a330168
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/WalletDownstreamProperties.java
@@ -0,0 +1,39 @@
+package com.sportsbook.gateway.routing;
+
+import java.net.URI;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "gateway.downstream.wallet")
+public record WalletDownstreamProperties(URI uri, String apiKey) {
+
+  private static final int MINIMUM_API_KEY_LENGTH = 32;
+
+  public WalletDownstreamProperties {
+    String scheme = uri == null ? null : uri.getScheme();
+    String path = uri == null ? null : uri.getRawPath();
+    if (uri == null
+        || !uri.isAbsolute()
+        || uri.isOpaque()
+        || uri.getHost() == null
+        || !("http".equals(scheme) || "https".equals(scheme))
+        || uri.getUserInfo() != null
+        || uri.getRawQuery() != null
+        || uri.getRawFragment() != null
+        || !(path == null || path.isEmpty() || "/".equals(path))) {
+      throw new IllegalArgumentException("gateway.downstream.wallet.uri must be an HTTP base URI");
+    }
+  }
+
+  String requiredApiKey() {
+    if (apiKey == null || apiKey.isBlank() || apiKey.length() < MINIMUM_API_KEY_LENGTH) {
+      throw new IllegalArgumentException(
+          "GATEWAY_WALLET_API_KEY must contain at least 32 characters");
+    }
+    return apiKey;
+  }
+
+  @Override
+  public String toString() {
+    return "WalletDownstreamProperties[uri=" + uri + ", apiKey=<redacted>]";
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 6425d3e..5a12da8 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -26,6 +26,9 @@ server:
 gateway:
   downstream:
     betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}
+    wallet:
+      uri: ${GATEWAY_WALLET_URI:http://localhost:8081}
+      api-key: ${GATEWAY_WALLET_API_KEY:}
   security:
     jwt:
       public-key: ${GATEWAY_JWT_PUBLIC_KEY:}


