## `feat(error): render RFC 7807 security failures`

diff --git a/src/main/java/com/sportsbook/admin/error/Rfc7807Writer.java b/src/main/java/com/sportsbook/admin/error/Rfc7807Writer.java
new file mode 100644
index 0000000..21e76c8
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/error/Rfc7807Writer.java
@@ -0,0 +1,52 @@
+package com.sportsbook.admin.error;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.error.ProblemDetail;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.URI;
+import java.nio.charset.StandardCharsets;
+import org.slf4j.MDC;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class Rfc7807Writer {
+
+  public static final URI UNAUTHORIZED = URI.create("https://sportsbook/errors/unauthorized");
+  public static final URI FORBIDDEN = URI.create("https://sportsbook/errors/forbidden");
+  public static final URI IP_NOT_ALLOWED = URI.create("https://sportsbook/errors/ip-not-allowed");
+
+  private final ObjectMapper objectMapper;
+
+  public Rfc7807Writer(ObjectMapper objectMapper) {
+    this.objectMapper = objectMapper;
+  }
+
+  public void write(
+      HttpServletRequest request,
+      HttpServletResponse response,
+      HttpStatus status,
+      URI type,
+      String title,
+      String code,
+      String detail)
+      throws IOException {
+    ProblemDetail body =
+        new ProblemDetail(
+            type,
+            title,
+            status.value(),
+            code,
+            detail,
+            URI.create(request.getRequestURI()),
+            MDC.get("traceId"));
+    response.setStatus(status.value());
+    response.setCharacterEncoding(StandardCharsets.UTF_8.name());
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    response.setHeader("Cache-Control", "no-store");
+    objectMapper.writeValue(response.getOutputStream(), body);
+  }
+}


## `test(error): verify RFC 7807 rendering`

diff --git a/src/test/java/com/sportsbook/admin/error/Rfc7807WriterTest.java b/src/test/java/com/sportsbook/admin/error/Rfc7807WriterTest.java
new file mode 100644
index 0000000..6023342
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/error/Rfc7807WriterTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.admin.error;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.slf4j.MDC;
+import org.springframework.http.HttpStatus;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class Rfc7807WriterTest {
+
+  private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();
+  private final Rfc7807Writer writer = new Rfc7807Writer(objectMapper);
+
+  @AfterEach
+  void clearTraceContext() {
+    MDC.clear();
+  }
+
+  @Test
+  void rendersACompleteSecurityProblemWithoutCaching() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/admin/v1/audit-logs");
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    MDC.put("traceId", "trace-123");
+
+    writer.write(
+        request,
+        response,
+        HttpStatus.UNAUTHORIZED,
+        Rfc7807Writer.UNAUTHORIZED,
+        "Unauthorized",
+        "UNAUTHORIZED",
+        "Authentication is required");
+
+    JsonNode body = objectMapper.readTree(response.getContentAsByteArray());
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentType()).startsWith("application/problem+json");
+    assertThat(response.getHeader("Cache-Control")).isEqualTo("no-store");
+    assertThat(body.path("type").asText()).isEqualTo(Rfc7807Writer.UNAUTHORIZED.toString());
+    assertThat(body.path("title").asText()).isEqualTo("Unauthorized");
+    assertThat(body.path("status").asInt()).isEqualTo(401);
+    assertThat(body.path("errorCode").asText()).isEqualTo("UNAUTHORIZED");
+    assertThat(body.path("detail").asText()).isEqualTo("Authentication is required");
+    assertThat(body.path("instance").asText()).isEqualTo("/admin/v1/audit-logs");
+    assertThat(body.path("correlationId").asText()).isEqualTo("trace-123");
+  }
+}


## `feat(security): decode verified RS256 tokens`

diff --git a/src/main/java/com/sportsbook/admin/security/JwtDecoderConfiguration.java b/src/main/java/com/sportsbook/admin/security/JwtDecoderConfiguration.java
new file mode 100644
index 0000000..006a058
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/JwtDecoderConfiguration.java
@@ -0,0 +1,33 @@
+package com.sportsbook.admin.security;
+
+import java.util.ArrayList;
+import java.util.List;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.jose.jws.SignatureAlgorithm;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.security.oauth2.jwt.JwtIssuerValidator;
+import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
+
+@Configuration(proxyBeanMethods = false)
+class JwtDecoderConfiguration {
+
+  @Bean
+  JwtDecoder adminJwtDecoder(AdminJwtProperties properties) {
+    NimbusJwtDecoder decoder =
+        NimbusJwtDecoder.withPublicKey(new RsaPublicKeyParser().parse(properties.publicKey()))
+            .signatureAlgorithm(SignatureAlgorithm.RS256)
+            .build();
+
+    List<OAuth2TokenValidator<Jwt>> validators = new ArrayList<>();
+    validators.add(new AdminJwtTimestampValidator());
+    validators.add(new AdminJwtSubjectValidator());
+    validators.add(new AdminJwtRoleValidator());
+    properties.expectedIssuer().ifPresent(issuer -> validators.add(new JwtIssuerValidator(issuer)));
+    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(validators));
+    return decoder;
+  }
+}


## `test(security): provide signed JWT fixture`

diff --git a/src/test/java/com/sportsbook/admin/security/TestJwtKeys.java b/src/test/java/com/sportsbook/admin/security/TestJwtKeys.java
new file mode 100644
index 0000000..0ce30a4
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/TestJwtKeys.java
@@ -0,0 +1,54 @@
+package com.sportsbook.admin.security;
+
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
+
+public final class TestJwtKeys {
+
+  private static final KeyPair TRUSTED = generate();
+
+  private TestJwtKeys() {}
+
+  public static String publicKeyPem() {
+    String body = Base64.getEncoder().encodeToString(TRUSTED.getPublic().getEncoded());
+    return "-----BEGIN PUBLIC KEY-----\n" + body + "\n-----END PUBLIC KEY-----";
+  }
+
+  public static String bearer(String subject, String role) {
+    try {
+      Instant now = Instant.now();
+      JWTClaimsSet claims =
+          new JWTClaimsSet.Builder()
+              .subject(subject)
+              .claim("role", role)
+              .issueTime(Date.from(now))
+              .notBeforeTime(Date.from(now.minusSeconds(1)))
+              .expirationTime(Date.from(now.plusSeconds(300)))
+              .build();
+      SignedJWT jwt = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
+      jwt.sign(new RSASSASigner((RSAPrivateKey) TRUSTED.getPrivate()));
+      return "Bearer " + jwt.serialize();
+    } catch (Exception exception) {
+      throw new IllegalStateException("Could not sign test JWT", exception);
+    }
+  }
+
+  private static KeyPair generate() {
+    try {
+      KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
+      generator.initialize(2048);
+      return generator.generateKeyPair();
+    } catch (Exception exception) {
+      throw new IllegalStateException("Could not generate test key", exception);
+    }
+  }
+}


## `feat(security): enforce authenticated admin requests`

diff --git a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
new file mode 100644
index 0000000..f11885e
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
@@ -0,0 +1,85 @@
+package com.sportsbook.admin.security;
+
+import com.sportsbook.admin.error.Rfc7807Writer;
+import java.util.List;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.core.GrantedAuthority;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
+import org.springframework.security.web.SecurityFilterChain;
+
+@Configuration(proxyBeanMethods = false)
+@EnableMethodSecurity
+class SecurityConfig {
+
+  @Bean
+  SecurityFilterChain adminSecurityFilterChain(HttpSecurity http, Rfc7807Writer problems)
+      throws Exception {
+    return http.csrf(csrf -> csrf.disable())
+        .sessionManagement(
+            sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .authorizeHttpRequests(
+            requests ->
+                requests
+                    .requestMatchers(
+                        "/actuator/health/liveness", "/actuator/health/readiness", "/error")
+                    .permitAll()
+                    .requestMatchers("/admin/**")
+                    .authenticated()
+                    .anyRequest()
+                    .denyAll())
+        .exceptionHandling(
+            failures ->
+                failures
+                    .authenticationEntryPoint(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.UNAUTHORIZED,
+                                Rfc7807Writer.UNAUTHORIZED,
+                                "Unauthorized",
+                                "UNAUTHORIZED",
+                                "Authentication is required"))
+                    .accessDeniedHandler(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.FORBIDDEN,
+                                Rfc7807Writer.FORBIDDEN,
+                                "Forbidden",
+                                "FORBIDDEN",
+                                "The operator is not allowed to perform this action")))
+        .oauth2ResourceServer(
+            resourceServer ->
+                resourceServer
+                    .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
+                    .authenticationEntryPoint(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.UNAUTHORIZED,
+                                Rfc7807Writer.UNAUTHORIZED,
+                                "Unauthorized",
+                                "UNAUTHORIZED",
+                                "Authentication is required")))
+        .build();
+  }
+
+  private static JwtAuthenticationConverter jwtAuthenticationConverter() {
+    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
+    converter.setJwtGrantedAuthoritiesConverter(
+        jwt ->
+            AdminRole.fromClaim(jwt.getClaims().get("role"))
+                .map(role -> List.<GrantedAuthority>of(new SimpleGrantedAuthority(role.authority())))
+                .orElseGet(List::of));
+    return converter;
+  }
+}


## `test(security): verify request authentication boundaries`

diff --git a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
new file mode 100644
index 0000000..4ff864f
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
@@ -0,0 +1,79 @@
+package com.sportsbook.admin.security;
+
+import static org.springframework.http.HttpHeaders.AUTHORIZATION;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@SpringBootTest(
+    properties = {
+      "spring.autoconfigure.exclude="
+          + "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration",
+      "admin.downstream.credentials.wallet-api-key=wallet-admin-test-key-000000000001",
+      "admin.downstream.credentials.risk-api-key=risk-admin-test-key-00000000000002",
+      "admin.downstream.credentials.odds-feed-api-key=odds-admin-test-key-00000000000003",
+      "admin.downstream.credentials.settlement-api-key=settlement-admin-test-key-000000004"
+    })
+@AutoConfigureMockMvc
+@Import(SecurityChainTest.RoleProbeController.class)
+class SecurityChainTest {
+
+  private static final String ADMIN_ONLY = "/admin/v1/test/admin-only";
+
+  @DynamicPropertySource
+  static void jwtKey(DynamicPropertyRegistry registry) {
+    registry.add("admin.security.jwt.public-key", TestJwtKeys::publicKeyPem);
+  }
+
+  @Autowired private MockMvc mvc;
+
+  @Test
+  void rejectsAnonymousRequestsWithAnRfc7807Response() throws Exception {
+    mvc.perform(get(ADMIN_ONLY))
+        .andExpect(status().isUnauthorized())
+        .andExpect(content().contentTypeCompatibleWith("application/problem+json"))
+        .andExpect(jsonPath("$.errorCode").value("UNAUTHORIZED"));
+  }
+
+  @Test
+  void rejectsAnAuthenticatedRoleWithoutMethodAuthority() throws Exception {
+    mvc.perform(get(ADMIN_ONLY).header(AUTHORIZATION, TestJwtKeys.bearer("reader-1", "READONLY")))
+        .andExpect(status().isForbidden())
+        .andExpect(jsonPath("$.errorCode").value("FORBIDDEN"));
+  }
+
+  @Test
+  void permitsTheRequiredMethodRole() throws Exception {
+    mvc.perform(get(ADMIN_ONLY).header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+        .andExpect(status().isOk())
+        .andExpect(content().string("ok"));
+  }
+
+  @RestController
+  static class RoleProbeController {
+
+    @GetMapping(ADMIN_ONLY)
+    @PreAuthorize("hasRole('ADMIN')")
+    String adminOnly() {
+      return "ok";
+    }
+  }
+}


## `feat(security): permit metric scraping`

diff --git a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
index f11885e..273beee 100644
--- a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
+++ b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
@@ -27,7 +27,10 @@ class SecurityConfig {
             requests ->
                 requests
                     .requestMatchers(
-                        "/actuator/health/liveness", "/actuator/health/readiness", "/error")
+                        "/actuator/health/liveness",
+                        "/actuator/health/readiness",
+                        "/actuator/prometheus",
+                        "/error")
                     .permitAll()
                     .requestMatchers("/admin/**")
                     .authenticated()


## `test(security): isolate metric scrape access`

diff --git a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
index 4ff864f..b71bb9e 100644
--- a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
+++ b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
@@ -8,6 +8,7 @@ import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.
 
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
 import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
 import org.springframework.boot.test.context.SpringBootTest;
 import org.springframework.context.annotation.Import;
@@ -33,6 +34,7 @@ import org.springframework.web.bind.annotation.RestController;
       "admin.downstream.credentials.settlement-api-key=settlement-admin-test-key-000000004"
     })
 @AutoConfigureMockMvc
+@AutoConfigureObservability
 @Import(SecurityChainTest.RoleProbeController.class)
 class SecurityChainTest {
 
@@ -67,6 +69,22 @@ class SecurityChainTest {
         .andExpect(content().string("ok"));
   }
 
+  @Test
+  void exposesOnlyThePrometheusScrapeEndpointOutsideHealth() throws Exception {
+    mvc.perform(get("/actuator/prometheus"))
+        .andExpect(status().isOk())
+        .andExpect(content().contentTypeCompatibleWith("text/plain"));
+
+    mvc.perform(
+            get("/actuator/metrics")
+                .header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+        .andExpect(status().isForbidden());
+    mvc.perform(
+            get("/actuator/env")
+                .header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+        .andExpect(status().isForbidden());
+  }
+
   @RestController
   static class RoleProbeController {
 
