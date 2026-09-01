## `feat(routing): route authenticated wallet reads`

diff --git a/src/main/java/com/sportsbook/gateway/routing/WalletRequestAuthentication.java b/src/main/java/com/sportsbook/gateway/routing/WalletRequestAuthentication.java
new file mode 100644
index 0000000..6e7aadd
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/WalletRequestAuthentication.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.routing;
+
+import com.sportsbook.gateway.security.GatewayHeaders;
+import org.springframework.web.servlet.function.ServerRequest;
+
+final class WalletRequestAuthentication {
+
+  private final String apiKey;
+
+  WalletRequestAuthentication(WalletDownstreamProperties properties) {
+    this.apiKey = properties.requiredApiKey();
+  }
+
+  ServerRequest apply(ServerRequest request) {
+    return ServerRequest.from(request)
+        .headers(
+            headers -> {
+              headers.set(GatewayHeaders.INTERNAL_SERVICE, "gateway");
+              headers.set(GatewayHeaders.INTERNAL_API_KEY, apiKey);
+            })
+        .build();
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/routing/WalletRoutes.java b/src/main/java/com/sportsbook/gateway/routing/WalletRoutes.java
new file mode 100644
index 0000000..767539e
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/WalletRoutes.java
@@ -0,0 +1,65 @@
+package com.sportsbook.gateway.routing;
+
+import static org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions.route;
+import static org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions.http;
+import static org.springframework.web.servlet.function.RequestPredicates.GET;
+
+import java.net.URI;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.web.servlet.function.RouterFunction;
+import org.springframework.web.servlet.function.ServerRequest;
+import org.springframework.web.servlet.function.ServerResponse;
+import org.springframework.web.util.UriComponentsBuilder;
+
+@Configuration
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+@EnableConfigurationProperties(WalletDownstreamProperties.class)
+public class WalletRoutes {
+
+  private final WalletDownstreamProperties downstream;
+  private final DownstreamRequestSanitizer sanitizer;
+  private final IdentityForwarding identity;
+  private final WalletRequestAuthentication authentication;
+  private final TraceForwarding trace;
+  private final DownstreamFailureBoundary failures;
+
+  public WalletRoutes(
+      WalletDownstreamProperties downstream,
+      DownstreamRequestSanitizer sanitizer,
+      IdentityForwarding identity,
+      TraceForwarding trace,
+      DownstreamFailureBoundary failures) {
+    this.downstream = downstream;
+    this.sanitizer = sanitizer;
+    this.identity = identity;
+    this.authentication = new WalletRequestAuthentication(downstream);
+    this.trace = trace;
+    this.failures = failures;
+  }
+
+  @Bean
+  RouterFunction<ServerResponse> walletBalanceRoute() {
+    return route("wallet-balance")
+        .route(GET("/api/v1/wallet/balance"), http(downstream.uri().toString()))
+        .before(sanitizer::apply)
+        .before(identity::apply)
+        .before(authentication::apply)
+        .before(trace::apply)
+        .before(this::rewrite)
+        .filter(failures)
+        .build();
+  }
+
+  private ServerRequest rewrite(ServerRequest request) {
+    String subject = identity.currentSubject().orElseThrow();
+    URI uri =
+        UriComponentsBuilder.fromUri(request.uri())
+            .replacePath("/internal/v1/wallet/accounts/" + subject + "/balance")
+            .build(true)
+            .toUri();
+    return ServerRequest.from(request).uri(uri).build();
+  }
+}


## `test(routing): verify wallet authentication boundary`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 772cb53..8fd2b41 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -121,6 +121,43 @@ class GatewayRoutingIntegrationTest {
     DOWNSTREAM.verify(getRequestedFor(urlPathEqualTo("/internal/v1/bets/" + betId)));
   }
 
+  @Test
+  void injectsWalletCredentialForTheVerifiedSubject() throws Exception {
+    UUID subject = UUID.randomUUID();
+    String path = "/internal/v1/wallet/accounts/" + subject + "/balance";
+    DOWNSTREAM.stubFor(get(urlPathEqualTo(path)).willReturn(aResponse().withStatus(200)));
+    HttpHeaders headers = authenticated(subject);
+    headers.set("X-Internal-Service", "attacker");
+    headers.set("X-Internal-Api-Key", "attacker-key");
+
+    ResponseEntity<String> response =
+        http.exchange(
+            "/api/v1/wallet/balance", HttpMethod.GET, new HttpEntity<>(headers), String.class);
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
+    DOWNSTREAM.verify(
+        getRequestedFor(urlPathEqualTo(path))
+            .withHeader("X-User-Id", equalTo(subject.toString()))
+            .withHeader("X-Internal-Service", equalTo("gateway"))
+            .withHeader("X-Internal-Api-Key", equalTo(WALLET_KEY))
+            .withoutHeader(HttpHeaders.AUTHORIZATION));
+  }
+
+  @Test
+  void rejectsAnonymousAndUnexpectedWalletRequests() throws Exception {
+    assertThat(http.getForEntity("/api/v1/wallet/balance", String.class).getStatusCode())
+        .isEqualTo(HttpStatus.UNAUTHORIZED);
+    assertThat(
+            http.exchange(
+                    "/api/v1/wallet/balance",
+                    HttpMethod.POST,
+                    new HttpEntity<>(authenticated(UUID.randomUUID())),
+                    String.class)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.FORBIDDEN);
+    assertThat(DOWNSTREAM.getAllServeEvents()).isEmpty();
+  }
+
   @Test
   void rejectsUnsafeBettingBaseUris() {
     assertThatThrownBy(() -> bettingUri("ftp://betting.internal"))
diff --git a/src/test/java/com/sportsbook/gateway/routing/WalletRoutesStartupTest.java b/src/test/java/com/sportsbook/gateway/routing/WalletRoutesStartupTest.java
new file mode 100644
index 0000000..87542b4
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/WalletRoutesStartupTest.java
@@ -0,0 +1,45 @@
+package com.sportsbook.gateway.routing;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.AutoConfigurations;
+import org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration;
+import org.springframework.boot.test.context.runner.WebApplicationContextRunner;
+
+class WalletRoutesStartupTest {
+
+  private final WebApplicationContextRunner context =
+      new WebApplicationContextRunner()
+          .withConfiguration(AutoConfigurations.of(ConfigurationPropertiesAutoConfiguration.class))
+          .withUserConfiguration(WalletRoutes.class)
+          .withBean(DownstreamRequestSanitizer.class)
+          .withBean(IdentityForwarding.class)
+          .withBean(ObjectMapper.class, ObjectMapper::new)
+          .withBean(GatewayProblemWriter.class)
+          .withBean(TraceForwarding.class)
+          .withBean(DownstreamFailureBoundary.class)
+          .withPropertyValues("gateway.downstream.wallet.uri=http://wallet.internal");
+
+  @Test
+  void failsStartupWhenWalletKeyIsMissing() {
+    assertRejected(context);
+  }
+
+  @Test
+  void failsStartupWhenWalletKeyIsShort() {
+    assertRejected(context.withPropertyValues("gateway.downstream.wallet.api-key=short"));
+  }
+
+  private static void assertRejected(WebApplicationContextRunner runner) {
+    runner.run(
+        context -> {
+          assertThat(context).hasFailed();
+          assertThat(context.getStartupFailure())
+              .hasRootCauseInstanceOf(IllegalArgumentException.class)
+              .hasRootCauseMessage("GATEWAY_WALLET_API_KEY must contain at least 32 characters");
+        });
+  }
+}


## `feat(routing): expose public event and odds reads`

diff --git a/src/main/java/com/sportsbook/gateway/routing/OddsFeedDownstreamProperties.java b/src/main/java/com/sportsbook/gateway/routing/OddsFeedDownstreamProperties.java
new file mode 100644
index 0000000..26a5bbb
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/OddsFeedDownstreamProperties.java
@@ -0,0 +1,25 @@
+package com.sportsbook.gateway.routing;
+
+import java.net.URI;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "gateway.downstream")
+public record OddsFeedDownstreamProperties(URI oddsFeedUri) {
+
+  public OddsFeedDownstreamProperties {
+    String scheme = oddsFeedUri == null ? null : oddsFeedUri.getScheme();
+    String path = oddsFeedUri == null ? null : oddsFeedUri.getRawPath();
+    if (oddsFeedUri == null
+        || !oddsFeedUri.isAbsolute()
+        || oddsFeedUri.isOpaque()
+        || oddsFeedUri.getHost() == null
+        || !("http".equals(scheme) || "https".equals(scheme))
+        || oddsFeedUri.getUserInfo() != null
+        || oddsFeedUri.getRawQuery() != null
+        || oddsFeedUri.getRawFragment() != null
+        || !(path == null || path.isEmpty() || "/".equals(path))) {
+      throw new IllegalArgumentException(
+          "gateway.downstream.odds-feed-uri must be an HTTP base URI");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/routing/PublicReadRoutes.java b/src/main/java/com/sportsbook/gateway/routing/PublicReadRoutes.java
new file mode 100644
index 0000000..70db9ab
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/PublicReadRoutes.java
@@ -0,0 +1,46 @@
+package com.sportsbook.gateway.routing;
+
+import static org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions.route;
+import static org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions.http;
+import static org.springframework.web.servlet.function.RequestPredicates.GET;
+
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.web.servlet.function.RouterFunction;
+import org.springframework.web.servlet.function.ServerResponse;
+
+@Configuration
+@EnableConfigurationProperties(OddsFeedDownstreamProperties.class)
+public class PublicReadRoutes {
+
+  private final OddsFeedDownstreamProperties downstream;
+  private final DownstreamRequestSanitizer sanitizer;
+  private final TraceForwarding trace;
+  private final DownstreamFailureBoundary failures;
+
+  public PublicReadRoutes(
+      OddsFeedDownstreamProperties downstream,
+      DownstreamRequestSanitizer sanitizer,
+      TraceForwarding trace,
+      DownstreamFailureBoundary failures) {
+    this.downstream = downstream;
+    this.sanitizer = sanitizer;
+    this.trace = trace;
+    this.failures = failures;
+  }
+
+  @Bean
+  RouterFunction<ServerResponse> publicReadRoute() {
+    return route("public-reads")
+        .route(
+            GET("/api/v1/events")
+                .or(GET("/api/v1/events/{eventId}"))
+                .or(GET("/api/v1/odds/{eventId}/{marketId}/{selectionId}")),
+            http(downstream.oddsFeedUri().toString()))
+        .before(sanitizer::apply)
+        .before(trace::apply)
+        .filter(failures)
+        .build();
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 5a12da8..81faa7d 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -26,6 +26,7 @@ server:
 gateway:
   downstream:
     betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}
+    odds-feed-uri: ${GATEWAY_ODDS_FEED_URI:http://localhost:8085}
     wallet:
       uri: ${GATEWAY_WALLET_URI:http://localhost:8081}
       api-key: ${GATEWAY_WALLET_API_KEY:}


## `test(routing): verify public read route boundaries`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 8fd2b41..8e3c796 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -57,6 +57,7 @@ class GatewayRoutingIntegrationTest {
     registry.add("gateway.downstream.betting-uri", DOWNSTREAM::baseUrl);
     registry.add("gateway.downstream.wallet.uri", DOWNSTREAM::baseUrl);
     registry.add("gateway.downstream.wallet.api-key", () -> WALLET_KEY);
+    registry.add("gateway.downstream.odds-feed-uri", DOWNSTREAM::baseUrl);
   }
 
   @BeforeEach
@@ -158,6 +159,69 @@ class GatewayRoutingIntegrationTest {
     assertThat(DOWNSTREAM.getAllServeEvents()).isEmpty();
   }
 
+  @Test
+  void proxiesExactPublicReadsWithoutForwardingCredentials() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    List<String> paths =
+        List.of(
+            "/api/v1/events",
+            "/api/v1/events/" + eventId,
+            "/api/v1/odds/" + eventId + "/" + marketId + "/" + selectionId);
+    for (String path : paths) {
+      DOWNSTREAM.stubFor(get(urlPathEqualTo(path)).willReturn(aResponse().withStatus(200)));
+    }
+    HttpHeaders headers = authenticated(UUID.randomUUID());
+    headers.set("X-User-Id", "attacker");
+    headers.set("X-Internal-Service", "attacker");
+    headers.set("X-Internal-Api-Key", "attacker-key");
+
+    for (String path : paths) {
+      HttpEntity<?> request =
+          path.equals("/api/v1/events") ? HttpEntity.EMPTY : new HttpEntity<>(headers);
+      assertThat(http.exchange(path, HttpMethod.GET, request, String.class).getStatusCode())
+          .isEqualTo(HttpStatus.OK);
+      DOWNSTREAM.verify(
+          getRequestedFor(urlPathEqualTo(path))
+              .withoutHeader(HttpHeaders.AUTHORIZATION)
+              .withoutHeader("X-User-Id")
+              .withoutHeader("X-Internal-Service")
+              .withoutHeader("X-Internal-Api-Key"));
+    }
+  }
+
+  @Test
+  void rejectsUnexpectedPublicMethodsAndPathShapes() throws Exception {
+    HttpEntity<Void> request = new HttpEntity<>(authenticated(UUID.randomUUID()));
+    List<String> paths =
+        List.of(
+            "/api/v1/events/one/extra",
+            "/api/v1/odds/event/market",
+            "/api/v1/odds/event/market/selection/extra");
+    assertThat(
+            http.exchange("/api/v1/events", HttpMethod.POST, request, String.class).getStatusCode())
+        .isEqualTo(HttpStatus.FORBIDDEN);
+    for (String path : paths) {
+      assertThat(http.exchange(path, HttpMethod.GET, request, String.class).getStatusCode())
+          .isEqualTo(HttpStatus.FORBIDDEN);
+    }
+    assertThat(DOWNSTREAM.getAllServeEvents()).isEmpty();
+  }
+
+  @Test
+  void rejectsUnsafeOddsFeedBaseUris() {
+    for (String uri :
+        List.of(
+            "ftp://odds.internal",
+            "http://user@odds.internal",
+            "http://odds.internal/base",
+            "http://odds.internal?query=1")) {
+      assertThatThrownBy(() -> new OddsFeedDownstreamProperties(URI.create(uri)))
+          .isInstanceOf(IllegalArgumentException.class);
+    }
+  }
+
   @Test
   void rejectsUnsafeBettingBaseUris() {
     assertThatThrownBy(() -> bettingUri("ftp://betting.internal"))
