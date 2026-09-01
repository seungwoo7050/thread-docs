# 내부 호출자 인증과 공개·내부 라우트 경계

## `build(security): add route security support`

diff --git a/pom.xml b/pom.xml
index 384340e..d430885 100644
--- a/pom.xml
+++ b/pom.xml
@@ -43,6 +43,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-validation</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-security</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-data-redis</artifactId>
diff --git a/src/test/resources/application.properties b/src/test/resources/application.properties
new file mode 100644
index 0000000..57e9a22
--- /dev/null
+++ b/src/test/resources/application.properties
@@ -0,0 +1 @@
+oddsfeed.security.internal.api-key=test-internal-api-key-0123456789abcdef


## `feat(security): bind internal caller credentials`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/InternalSecurityProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/InternalSecurityProperties.java
new file mode 100644
index 0000000..93b5774
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/InternalSecurityProperties.java
@@ -0,0 +1,17 @@
+package com.sportsbook.oddsfeed.config;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Credentials accepted at the internal administration boundary. */
+@ConfigurationProperties(prefix = "oddsfeed.security.internal")
+public record InternalSecurityProperties(String apiKey) {
+
+  public static final int MINIMUM_API_KEY_LENGTH = 32;
+
+  public InternalSecurityProperties {
+    if (apiKey == null || apiKey.isBlank() || apiKey.length() < MINIMUM_API_KEY_LENGTH) {
+      throw new IllegalArgumentException(
+          "ADMIN_API_INTERNAL_KEY must contain at least " + MINIMUM_API_KEY_LENGTH + " characters");
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index f051506..0d52aa8 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -70,5 +70,8 @@ oddsfeed:
     batch-size: 50
     claim-idle: ${CRITICAL_EVENT_CLAIM_IDLE:5s}
     poll-interval-ms: ${CRITICAL_EVENT_POLL_INTERVAL_MS:250}
+  security:
+    internal:
+      api-key: ${ADMIN_API_INTERNAL_KEY}
   cache:
     ttl: 24h


## `test(security): verify internal credential settings`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/InternalSecurityPropertiesTest.java b/src/test/java/com/sportsbook/oddsfeed/config/InternalSecurityPropertiesTest.java
new file mode 100644
index 0000000..008bbff
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/InternalSecurityPropertiesTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+
+class InternalSecurityPropertiesTest {
+
+  private final ApplicationContextRunner contextRunner =
+      new ApplicationContextRunner().withUserConfiguration(PropertiesConfiguration.class);
+
+  @Test
+  void acceptsASecretWithAtLeastThirtyTwoCharacters() {
+    String secret = "0123456789abcdef0123456789abcdef";
+
+    assertThat(new InternalSecurityProperties(secret).apiKey()).isEqualTo(secret);
+  }
+
+  @Test
+  void rejectsMissingBlankAndShortSecrets() {
+    assertThatThrownBy(() -> new InternalSecurityProperties(null))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("ADMIN_API_INTERNAL_KEY");
+    assertThatThrownBy(() -> new InternalSecurityProperties("too-short"))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("32");
+    assertThatThrownBy(() -> new InternalSecurityProperties(" ".repeat(32)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void bindingEnforcesTheStartupBoundary() {
+    contextRunner.run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues("oddsfeed.security.internal.api-key=too-short")
+        .run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues("oddsfeed.security.internal.api-key=0123456789abcdef0123456789abcdef")
+        .run(context -> assertThat(context).hasNotFailed());
+  }
+
+  @EnableConfigurationProperties(InternalSecurityProperties.class)
+  static class PropertiesConfiguration {}
+}


## `feat(security): authenticate internal callers`

diff --git a/src/main/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilter.java b/src/main/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilter.java
new file mode 100644
index 0000000..e7b45f0
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilter.java
@@ -0,0 +1,76 @@
+package com.sportsbook.oddsfeed.security;
+
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.List;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.core.context.SecurityContext;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+/** Authenticates internal callers using constant-time digest comparisons. */
+public class InternalApiKeyAuthenticationFilter extends OncePerRequestFilter {
+
+  public static final String SERVICE_HEADER = "X-Internal-Service";
+  public static final String API_KEY_HEADER = "X-Internal-Api-Key";
+  public static final String EXPECTED_SERVICE = "admin-api";
+  public static final String AUTHORITY = "ODDS_INTERNAL_ADMIN";
+
+  private final byte[] expectedServiceDigest;
+  private final byte[] expectedApiKeyDigest;
+
+  public InternalApiKeyAuthenticationFilter(InternalSecurityProperties properties) {
+    this.expectedServiceDigest = sha256(EXPECTED_SERVICE);
+    this.expectedApiKeyDigest = sha256(properties.apiKey());
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    return !request.getRequestURI().startsWith("/internal/");
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
+      throws ServletException, IOException {
+    String service = request.getHeader(SERVICE_HEADER);
+    String apiKey = request.getHeader(API_KEY_HEADER);
+    if (service == null || service.isBlank() || !matches(expectedApiKeyDigest, apiKey)) {
+      SecurityContextHolder.clearContext();
+      response.sendError(HttpStatus.UNAUTHORIZED.value());
+      return;
+    }
+
+    List<SimpleGrantedAuthority> authorities =
+        matches(expectedServiceDigest, service)
+            ? List.of(new SimpleGrantedAuthority(AUTHORITY))
+            : List.of();
+    UsernamePasswordAuthenticationToken authentication =
+        UsernamePasswordAuthenticationToken.authenticated(service, null, authorities);
+    SecurityContext context = SecurityContextHolder.createEmptyContext();
+    context.setAuthentication(authentication);
+    SecurityContextHolder.setContext(context);
+    filterChain.doFilter(request, response);
+  }
+
+  private static boolean matches(byte[] expectedDigest, String supplied) {
+    return MessageDigest.isEqual(expectedDigest, sha256(supplied == null ? "" : supplied));
+  }
+
+  private static byte[] sha256(String value) {
+    try {
+      return MessageDigest.getInstance("SHA-256").digest(value.getBytes(StandardCharsets.UTF_8));
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("SHA-256 is unavailable", exception);
+    }
+  }
+}


## `test(security): verify internal authentication`

diff --git a/src/test/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilterTest.java b/src/test/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilterTest.java
new file mode 100644
index 0000000..0721062
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/security/InternalApiKeyAuthenticationFilterTest.java
@@ -0,0 +1,84 @@
+package com.sportsbook.oddsfeed.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.MethodSource;
+import org.springframework.mock.web.MockFilterChain;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.core.Authentication;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class InternalApiKeyAuthenticationFilterTest {
+
+  private static final String SECRET = "0123456789abcdef0123456789abcdef";
+
+  private final InternalApiKeyAuthenticationFilter filter =
+      new InternalApiKeyAuthenticationFilter(new InternalSecurityProperties(SECRET));
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @ParameterizedTest
+  @MethodSource("invalidCredentials")
+  void rejectsMissingOrInvalidCredentials(String service, String key) throws Exception {
+    MockHttpServletRequest request = internalRequest(service, key);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, new MockFilterChain());
+
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
+  }
+
+  @Test
+  void authenticatesNonAdminCallerWithoutAuthority() throws Exception {
+    filter.doFilter(
+        internalRequest("settlement-service", SECRET),
+        new MockHttpServletResponse(),
+        new MockFilterChain());
+
+    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
+    assertThat(authentication.getName()).isEqualTo("settlement-service");
+    assertThat(authentication.getAuthorities()).isEmpty();
+  }
+
+  @Test
+  void grantsOnlyTheAdminCallerAuthority() throws Exception {
+    filter.doFilter(
+        internalRequest("admin-api", SECRET), new MockHttpServletResponse(), new MockFilterChain());
+
+    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
+    assertThat(authentication.getName()).isEqualTo("admin-api");
+    assertThat(authentication.getAuthorities())
+        .extracting("authority")
+        .containsExactly(InternalApiKeyAuthenticationFilter.AUTHORITY);
+  }
+
+  private static Stream<Arguments> invalidCredentials() {
+    return Stream.of(
+        Arguments.of(null, SECRET),
+        Arguments.of("", SECRET),
+        Arguments.of("admin-api", null),
+        Arguments.of("admin-api", "wrong-key"));
+  }
+
+  private static MockHttpServletRequest internalRequest(String service, String key) {
+    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/internal/v1/action");
+    if (service != null) {
+      request.addHeader(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, service);
+    }
+    if (key != null) {
+      request.addHeader(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, key);
+    }
+    return request;
+  }
+}


## `feat(security): authorize public and internal routes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/security/SecurityConfig.java b/src/main/java/com/sportsbook/oddsfeed/security/SecurityConfig.java
new file mode 100644
index 0000000..fbebd91
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/security/SecurityConfig.java
@@ -0,0 +1,53 @@
+package com.sportsbook.oddsfeed.security;
+
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.authentication.AnonymousAuthenticationFilter;
+
+/** Defines the complete HTTP exposure boundary. */
+@Configuration(proxyBeanMethods = false)
+@EnableConfigurationProperties(InternalSecurityProperties.class)
+public class SecurityConfig {
+
+  @Bean
+  public InternalApiKeyAuthenticationFilter internalApiKeyAuthenticationFilter(
+      InternalSecurityProperties properties) {
+    return new InternalApiKeyAuthenticationFilter(properties);
+  }
+
+  @Bean
+  public SecurityFilterChain securityFilterChain(
+      HttpSecurity http, InternalApiKeyAuthenticationFilter internalFilter) throws Exception {
+    return http.csrf(csrf -> csrf.disable())
+        .sessionManagement(
+            sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .authorizeHttpRequests(
+            requests ->
+                requests
+                    .requestMatchers(
+                        HttpMethod.GET,
+                        "/api/v1/events",
+                        "/api/v1/events/**",
+                        "/api/v1/odds/**",
+                        "/actuator/health",
+                        "/actuator/health/**",
+                        "/actuator/prometheus")
+                    .permitAll()
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/events/*/markets/*/suspend")
+                    .hasAuthority(InternalApiKeyAuthenticationFilter.AUTHORITY)
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/events/*/markets/*/close")
+                    .hasAuthority(InternalApiKeyAuthenticationFilter.AUTHORITY)
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/events/*/markets/*/reopen")
+                    .hasAuthority(InternalApiKeyAuthenticationFilter.AUTHORITY)
+                    .anyRequest()
+                    .denyAll())
+        .addFilterBefore(internalFilter, AnonymousAuthenticationFilter.class)
+        .build();
+  }
+}


## `test(security): verify route authorization`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
index d91b1b5..27cddcc 100644
--- a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
@@ -6,8 +6,10 @@ import static org.springframework.test.web.servlet.request.MockMvcRequestBuilder
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.oddsfeed.security.SecurityConfig;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.EventId;
 import java.time.Instant;
@@ -16,14 +18,15 @@ import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
-import org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
 import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.annotation.Import;
 import org.springframework.test.web.servlet.MockMvc;
 
-@WebMvcTest(
-    controllers = EventReadController.class,
-    excludeAutoConfiguration = SecurityAutoConfiguration.class)
+@WebMvcTest(controllers = EventReadController.class)
+@Import(SecurityConfig.class)
+@EnableConfigurationProperties(InternalSecurityProperties.class)
 class EventReadControllerTest {
 
   @Autowired private MockMvc mockMvc;
diff --git a/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java
index c606242..4bde3de 100644
--- a/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java
@@ -2,10 +2,14 @@ package com.sportsbook.oddsfeed.api;
 
 import static org.mockito.Mockito.when;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
 import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import com.sportsbook.oddsfeed.security.InternalApiKeyAuthenticationFilter;
+import com.sportsbook.oddsfeed.security.SecurityConfig;
 import com.sportsbook.protocol.value.EventId;
 import com.sportsbook.protocol.value.MarketId;
 import com.sportsbook.protocol.value.Odds;
@@ -14,16 +18,19 @@ import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
-import org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
 import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
 import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.annotation.Import;
 import org.springframework.test.web.servlet.MockMvc;
 
-@WebMvcTest(
-    controllers = OddsReadController.class,
-    excludeAutoConfiguration = SecurityAutoConfiguration.class)
+@WebMvcTest(controllers = OddsReadController.class)
+@Import(SecurityConfig.class)
+@EnableConfigurationProperties(InternalSecurityProperties.class)
 class OddsReadControllerTest {
 
+  private static final String API_KEY = "test-internal-api-key-0123456789abcdef";
+
   @Autowired private MockMvc mockMvc;
   @MockBean private RedisOddsCache cache;
 
@@ -56,4 +63,31 @@ class OddsReadControllerTest {
         .perform(get("/api/v1/odds/{e}/{m}/{s}", eventId, marketId, selectionId))
         .andExpect(status().isNotFound());
   }
+
+  @Test
+  void internalRoutesRequireTheAdminCaller() throws Exception {
+    String path = "/internal/v1/events/{event}/markets/{market}/suspend";
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+
+    mockMvc.perform(post(path, eventId, marketId)).andExpect(status().isUnauthorized());
+    mockMvc
+        .perform(
+            post(path, eventId, marketId)
+                .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "settlement-service")
+                .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, API_KEY))
+        .andExpect(status().isForbidden());
+    mockMvc
+        .perform(
+            post(path, eventId, marketId)
+                .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "admin-api")
+                .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, API_KEY))
+        .andExpect(status().isNotFound());
+    mockMvc
+        .perform(
+            post("/internal/v1/unknown")
+                .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "admin-api")
+                .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, API_KEY))
+        .andExpect(status().isForbidden());
+  }
 }
