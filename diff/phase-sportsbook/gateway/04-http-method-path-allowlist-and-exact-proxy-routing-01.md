# HTTP 메서드·경로 허용목록과 정확한 프록시 라우팅

## `feat(security): protect HTTP access boundaries`

diff --git a/src/main/java/com/sportsbook/gateway/security/SecurityConfig.java b/src/main/java/com/sportsbook/gateway/security/SecurityConfig.java
new file mode 100644
index 0000000..8cd2f13
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/SecurityConfig.java
@@ -0,0 +1,68 @@
+package com.sportsbook.gateway.security;
+
+import com.sportsbook.gateway.error.GatewayErrorCode;
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import jakarta.servlet.DispatcherType;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.web.AuthenticationEntryPoint;
+import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.access.AccessDeniedHandler;
+
+@Configuration(proxyBeanMethods = false)
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class SecurityConfig {
+
+  @Bean
+  SecurityFilterChain securityFilterChain(HttpSecurity http, GatewayProblemWriter problems)
+      throws Exception {
+    AuthenticationEntryPoint unauthorized =
+        (request, response, failure) ->
+            problems.write(request, response, GatewayErrorCode.GATEWAY_UNAUTHORIZED);
+    AccessDeniedHandler forbidden =
+        (request, response, failure) ->
+            problems.write(request, response, GatewayErrorCode.GATEWAY_FORBIDDEN);
+
+    http.csrf(AbstractHttpConfigurer::disable)
+        .sessionManagement(
+            session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .authorizeHttpRequests(
+            access ->
+                access
+                    .dispatcherTypeMatchers(DispatcherType.ERROR)
+                    .permitAll()
+                    .requestMatchers(
+                        "/actuator/health/**", "/actuator/info", "/actuator/prometheus")
+                    .permitAll()
+                    .requestMatchers(
+                        HttpMethod.GET, "/api/v1/events", "/api/v1/events/*", "/api/v1/odds/*/*/*")
+                    .permitAll()
+                    .requestMatchers("/ws/v1/odds", "/ws/v1/bets")
+                    .permitAll()
+                    .requestMatchers(HttpMethod.POST, "/api/v1/bets")
+                    .authenticated()
+                    .requestMatchers(HttpMethod.GET, "/api/v1/bets", "/api/v1/bets/*")
+                    .authenticated()
+                    .requestMatchers(HttpMethod.GET, "/api/v1/wallet/balance")
+                    .authenticated()
+                    .anyRequest()
+                    .denyAll())
+        .exceptionHandling(
+            failures ->
+                failures.authenticationEntryPoint(unauthorized).accessDeniedHandler(forbidden))
+        .oauth2ResourceServer(
+            oauth2 ->
+                oauth2
+                    .authenticationEntryPoint(unauthorized)
+                    .jwt(
+                        jwt ->
+                            jwt.jwtAuthenticationConverter(
+                                new GatewayJwtAuthenticationConverter())));
+    return http.build();
+  }
+}


## `test(security): verify public and private paths`

diff --git a/src/test/java/com/sportsbook/gateway/security/HttpAccessBoundaryTest.java b/src/test/java/com/sportsbook/gateway/security/HttpAccessBoundaryTest.java
new file mode 100644
index 0000000..d1f0459
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/security/HttpAccessBoundaryTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.gateway.security;
+
+import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import jakarta.servlet.DispatcherType;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.annotation.Import;
+import org.springframework.http.MediaType;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.test.web.servlet.MockMvc;
+
+@WebMvcTest
+@Import({SecurityConfig.class, GatewayProblemWriter.class})
+class HttpAccessBoundaryTest {
+
+  @Autowired MockMvc mvc;
+  @MockBean JwtDecoder jwtDecoder;
+
+  @Test
+  void publicReadsAndWebsocketHandshakesDoNotRequireAuthentication() throws Exception {
+    mvc.perform(get("/api/v1/events/fixture")).andExpect(status().isNotFound());
+    mvc.perform(get("/api/v1/odds/event/market/selection")).andExpect(status().isNotFound());
+    mvc.perform(get("/ws/v1/odds")).andExpect(status().isNotFound());
+  }
+
+  @Test
+  void privateEndpointsRequireAuthentication() throws Exception {
+    mvc.perform(get("/api/v1/wallet/balance"))
+        .andExpect(status().isUnauthorized())
+        .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$.errorCode").value("GATEWAY_UNAUTHORIZED"));
+    mvc.perform(post("/api/v1/bets").with(jwt())).andExpect(status().isNotFound());
+    mvc.perform(get("/api/v1/bets/00000000-0000-0000-0000-000000000001").with(jwt()))
+        .andExpect(status().isNotFound());
+  }
+
+  @Test
+  void unexpectedMethodsAndPathsAreDenied() throws Exception {
+    mvc.perform(post("/api/v1/events").with(jwt()))
+        .andExpect(status().isForbidden())
+        .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$.errorCode").value("GATEWAY_FORBIDDEN"));
+    mvc.perform(get("/api/v1/bets/internal/hidden").with(jwt())).andExpect(status().isForbidden());
+    mvc.perform(get("/api/v1/odds/incomplete").with(jwt())).andExpect(status().isForbidden());
+    mvc.perform(get("/internal/health").with(jwt())).andExpect(status().isForbidden());
+  }
+
+  @Test
+  void errorDispatchIsNotReauthenticated() throws Exception {
+    mvc.perform(
+            get("/internal/health")
+                .with(
+                    request -> {
+                      request.setDispatcherType(DispatcherType.ERROR);
+                      return request;
+                    }))
+        .andExpect(status().isNotFound());
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


## `test(routing): add authenticated proxy fixture`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
new file mode 100644
index 0000000..77c5f0c
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -0,0 +1,89 @@
+package com.sportsbook.gateway.routing;
+
+import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;
+
+import com.github.tomakehurst.wiremock.WireMockServer;
+import com.nimbusds.jose.JWSAlgorithm;
+import com.nimbusds.jose.JWSHeader;
+import com.nimbusds.jose.crypto.RSASSASigner;
+import com.nimbusds.jwt.JWTClaimsSet;
+import com.nimbusds.jwt.SignedJWT;
+import java.security.KeyPair;
+import java.security.KeyPairGenerator;
+import java.security.interfaces.RSAPrivateKey;
+import java.time.Instant;
+import java.util.Base64;
+import java.util.Date;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpHeaders;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
+class GatewayRoutingIntegrationTest {
+
+  static final WireMockServer DOWNSTREAM = startDownstream();
+  private static final KeyPair KEYS = keyPair();
+
+  @Autowired TestRestTemplate http;
+
+  @DynamicPropertySource
+  static void properties(DynamicPropertyRegistry registry) {
+    registry.add("gateway.security.jwt.public-key", GatewayRoutingIntegrationTest::publicKey);
+    registry.add("gateway.ratelimit.enabled", () -> "false");
+    registry.add("gateway.downstream.betting-uri", DOWNSTREAM::baseUrl);
+  }
+
+  @BeforeEach
+  void resetDownstream() {
+    DOWNSTREAM.resetAll();
+  }
+
+  @AfterAll
+  static void stopDownstream() {
+    DOWNSTREAM.stop();
+  }
+
+  HttpHeaders authenticated(UUID subject) throws Exception {
+    JWTClaimsSet claims =
+        new JWTClaimsSet.Builder()
+            .subject(subject.toString())
+            .expirationTime(Date.from(Instant.now().plusSeconds(300)))
+            .claim("roles", List.of("USER"))
+            .build();
+    SignedJWT jwt = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
+    jwt.sign(new RSASSASigner((RSAPrivateKey) KEYS.getPrivate()));
+    HttpHeaders headers = new HttpHeaders();
+    headers.setBearerAuth(jwt.serialize());
+    return headers;
+  }
+
+  private static WireMockServer startDownstream() {
+    WireMockServer server =
+        new WireMockServer(
+            wireMockConfig().dynamicPort().http2PlainDisabled(true).http2TlsDisabled(true));
+    server.start();
+    return server;
+  }
+
+  private static KeyPair keyPair() {
+    try {
+      KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
+      generator.initialize(2048);
+      return generator.generateKeyPair();
+    } catch (Exception failure) {
+      throw new ExceptionInInitializerError(failure);
+    }
+  }
+
+  private static String publicKey() {
+    String encoded = Base64.getEncoder().encodeToString(KEYS.getPublic().getEncoded());
+    return "-----BEGIN PUBLIC KEY-----\n" + encoded + "\n-----END PUBLIC KEY-----";
+  }
+}


## `test(routing): verify betting route contracts`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 77c5f0c..6f5c996 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -1,6 +1,15 @@
 package com.sportsbook.gateway.routing;
 
+import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
+import static com.github.tomakehurst.wiremock.client.WireMock.equalTo;
+import static com.github.tomakehurst.wiremock.client.WireMock.get;
+import static com.github.tomakehurst.wiremock.client.WireMock.getRequestedFor;
+import static com.github.tomakehurst.wiremock.client.WireMock.post;
+import static com.github.tomakehurst.wiremock.client.WireMock.postRequestedFor;
+import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
 import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.github.tomakehurst.wiremock.WireMockServer;
 import com.nimbusds.jose.JWSAlgorithm;
@@ -8,6 +17,7 @@ import com.nimbusds.jose.JWSHeader;
 import com.nimbusds.jose.crypto.RSASSASigner;
 import com.nimbusds.jwt.JWTClaimsSet;
 import com.nimbusds.jwt.SignedJWT;
+import java.net.URI;
 import java.security.KeyPair;
 import java.security.KeyPairGenerator;
 import java.security.interfaces.RSAPrivateKey;
@@ -18,10 +28,16 @@ import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.AfterAll;
 import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpEntity;
 import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 
@@ -45,6 +61,75 @@ class GatewayRoutingIntegrationTest {
     DOWNSTREAM.resetAll();
   }
 
+  @Test
+  void forwardsBetPlacementBodyAndVerifiedIdentity() throws Exception {
+    UUID subject = UUID.randomUUID();
+    String body = "{\"stake\":\"10.00\"}";
+    DOWNSTREAM.stubFor(
+        post(urlPathEqualTo("/internal/v1/bets"))
+            .willReturn(aResponse().withStatus(201).withBody("{\"betId\":\"accepted\"}")));
+    HttpHeaders headers = authenticated(subject);
+    headers.setContentType(MediaType.APPLICATION_JSON);
+    headers.set("Idempotency-Key", "fixture-idempotency-key");
+    headers.set("X-User-Id", UUID.randomUUID().toString());
+    headers.set("X-Internal-Service", "attacker");
+    headers.set("X-Internal-Api-Key", "attacker-key");
+
+    ResponseEntity<String> response =
+        http.exchange(
+            "/api/v1/bets", HttpMethod.POST, new HttpEntity<>(body, headers), String.class);
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
+    DOWNSTREAM.verify(
+        postRequestedFor(urlPathEqualTo("/internal/v1/bets"))
+            .withHeader("X-User-Id", equalTo(subject.toString()))
+            .withHeader("X-User-Roles", equalTo("USER"))
+            .withHeader("Idempotency-Key", equalTo("fixture-idempotency-key"))
+            .withoutHeader(HttpHeaders.AUTHORIZATION)
+            .withoutHeader("X-Internal-Service")
+            .withoutHeader("X-Internal-Api-Key")
+            .withRequestBody(equalTo(body)));
+  }
+
+  @Test
+  void scopesBetReadsToTheVerifiedSubject() throws Exception {
+    UUID subject = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    DOWNSTREAM.stubFor(
+        get(urlPathEqualTo("/internal/v1/bets")).willReturn(aResponse().withStatus(200)));
+    DOWNSTREAM.stubFor(
+        get(urlPathEqualTo("/internal/v1/bets/" + betId)).willReturn(aResponse().withStatus(200)));
+
+    http.exchange(
+        "/api/v1/bets?userId=attacker&status=OPEN",
+        HttpMethod.GET,
+        new HttpEntity<>(authenticated(subject)),
+        String.class);
+    http.exchange(
+        "/api/v1/bets/" + betId,
+        HttpMethod.GET,
+        new HttpEntity<>(authenticated(subject)),
+        String.class);
+
+    DOWNSTREAM.verify(
+        getRequestedFor(urlPathEqualTo("/internal/v1/bets"))
+            .withQueryParam("userId", equalTo(subject.toString()))
+            .withQueryParam("status", equalTo("OPEN")));
+    DOWNSTREAM.verify(getRequestedFor(urlPathEqualTo("/internal/v1/bets/" + betId)));
+  }
+
+  @Test
+  void rejectsUnsafeBettingBaseUris() {
+    assertThatThrownBy(() -> bettingUri("ftp://betting.internal"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> bettingUri("http://user@betting.internal"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> bettingUri("http://betting.internal/base"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> bettingUri("http://betting.internal?query=1"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
   @AfterAll
   static void stopDownstream() {
     DOWNSTREAM.stop();
@@ -86,4 +171,8 @@ class GatewayRoutingIntegrationTest {
     String encoded = Base64.getEncoder().encodeToString(KEYS.getPublic().getEncoded());
     return "-----BEGIN PUBLIC KEY-----\n" + encoded + "\n-----END PUBLIC KEY-----";
   }
+
+  private static BettingDownstreamProperties bettingUri(String value) {
+    return new BettingDownstreamProperties(URI.create(value));
+  }
 }


