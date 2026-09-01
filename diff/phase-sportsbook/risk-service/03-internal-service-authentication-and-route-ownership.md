# 내부 서비스 인증과 경로 소유권

## `build(security): add internal request security`

diff --git a/pom.xml b/pom.xml
index 185faf8..fe980f3 100644
--- a/pom.xml
+++ b/pom.xml
@@ -64,6 +64,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-actuator</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-security</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-data-redis</artifactId>
@@ -98,6 +102,11 @@
             <artifactId>spring-boot-starter-test</artifactId>
             <scope>test</scope>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.security</groupId>
+            <artifactId>spring-security-test</artifactId>
+            <scope>test</scope>
+        </dependency>
         <dependency>
             <groupId>org.springframework.kafka</groupId>
             <artifactId>spring-kafka-test</artifactId>
diff --git a/src/main/java/com/sportsbook/risk/RiskServiceApplication.java b/src/main/java/com/sportsbook/risk/RiskServiceApplication.java
index 1ec92a3..83c137c 100644
--- a/src/main/java/com/sportsbook/risk/RiskServiceApplication.java
+++ b/src/main/java/com/sportsbook/risk/RiskServiceApplication.java
@@ -2,9 +2,10 @@ package com.sportsbook.risk;
 
 import org.springframework.boot.SpringApplication;
 import org.springframework.boot.autoconfigure.SpringBootApplication;
+import org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration;
 import org.springframework.boot.context.properties.ConfigurationPropertiesScan;
 
-@SpringBootApplication
+@SpringBootApplication(exclude = UserDetailsServiceAutoConfiguration.class)
 @ConfigurationPropertiesScan
 @SuppressWarnings("HideUtilityClassConstructor")
 public class RiskServiceApplication {


## `feat(auth): bind internal caller credentials`

diff --git a/src/main/java/com/sportsbook/risk/auth/InternalAuthProperties.java b/src/main/java/com/sportsbook/risk/auth/InternalAuthProperties.java
new file mode 100644
index 0000000..b8ef7cf
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/auth/InternalAuthProperties.java
@@ -0,0 +1,78 @@
+package com.sportsbook.risk.auth;
+
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.EnumMap;
+import java.util.HashSet;
+import java.util.Map;
+import java.util.Optional;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Validates internal caller secrets and retains only constant-time comparable digests. */
+@ConfigurationProperties(prefix = "risk.auth")
+public final class InternalAuthProperties {
+  public static final int MIN_SECRET_LENGTH = 32;
+
+  private final Map<Caller, byte[]> digests;
+
+  public InternalAuthProperties(
+      String bettingServiceApiKey, String adminApiKey, String platformApiKey) {
+    Map<Caller, String> secrets =
+        Map.of(
+            Caller.BETTING_SERVICE, requireSecret(bettingServiceApiKey, Caller.BETTING_SERVICE),
+            Caller.ADMIN_API, requireSecret(adminApiKey, Caller.ADMIN_API),
+            Caller.PLATFORM, requireSecret(platformApiKey, Caller.PLATFORM));
+    if (new HashSet<>(secrets.values()).size() != Caller.values().length) {
+      throw new IllegalArgumentException("internal caller secrets must be distinct");
+    }
+    EnumMap<Caller, byte[]> result = new EnumMap<>(Caller.class);
+    secrets.forEach((caller, secret) -> result.put(caller, digest(secret)));
+    digests = Map.copyOf(result);
+  }
+
+  public boolean matches(Caller caller, String candidate) {
+    if (caller == null || candidate == null) {
+      return false;
+    }
+    return MessageDigest.isEqual(digests.get(caller), digest(candidate));
+  }
+
+  private static String requireSecret(String secret, Caller caller) {
+    if (secret == null || secret.isBlank() || secret.length() < MIN_SECRET_LENGTH) {
+      throw new IllegalArgumentException(
+          caller.wireName + " secret must contain at least 32 characters");
+    }
+    return secret;
+  }
+
+  private static byte[] digest(String value) {
+    try {
+      return MessageDigest.getInstance("SHA-256").digest(value.getBytes(StandardCharsets.UTF_8));
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("SHA-256 is unavailable", impossible);
+    }
+  }
+
+  public enum Caller {
+    BETTING_SERVICE("betting-service"),
+    ADMIN_API("admin-api"),
+    PLATFORM("platform");
+
+    private final String wireName;
+
+    Caller(String wireName) {
+      this.wireName = wireName;
+    }
+
+    public String wireName() {
+      return wireName;
+    }
+
+    public static Optional<Caller> fromWire(String value) {
+      return java.util.Arrays.stream(values())
+          .filter(caller -> caller.wireName.equals(value))
+          .findFirst();
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 618b68d..aa4e9af 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -29,6 +29,10 @@ server:
   port: ${SERVER_PORT:8083}
 
 risk:
+  auth:
+    betting-service-api-key: ${INTERNAL_BETTING_SERVICE_API_KEY}
+    admin-api-key: ${INTERNAL_ADMIN_API_KEY}
+    platform-api-key: ${INTERNAL_PLATFORM_API_KEY}
   limits:
     stake-daily: { KRW: 1000000, USD: 100000 }
     stake-weekly: { KRW: 5000000, USD: 500000 }


## `test(auth): reject invalid caller credentials`

diff --git a/src/test/java/com/sportsbook/risk/auth/InternalAuthPropertiesTest.java b/src/test/java/com/sportsbook/risk/auth/InternalAuthPropertiesTest.java
new file mode 100644
index 0000000..968d1e5
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/auth/InternalAuthPropertiesTest.java
@@ -0,0 +1,43 @@
+package com.sportsbook.risk.auth;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.risk.auth.InternalAuthProperties.Caller;
+import org.junit.jupiter.api.Test;
+
+class InternalAuthPropertiesTest {
+  private static final String BETTING = "b".repeat(32);
+  private static final String ADMIN = "a".repeat(32);
+  private static final String PLATFORM = "p".repeat(32);
+
+  @Test
+  void rejectsMissingShortAndDuplicateSecrets() {
+    assertThatThrownBy(() -> new InternalAuthProperties(null, ADMIN, PLATFORM))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new InternalAuthProperties("short", ADMIN, PLATFORM))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new InternalAuthProperties(BETTING, BETTING, PLATFORM))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("distinct");
+  }
+
+  @Test
+  void comparesCallerSpecificDigests() {
+    InternalAuthProperties properties = new InternalAuthProperties(BETTING, ADMIN, PLATFORM);
+
+    assertThat(properties.matches(Caller.BETTING_SERVICE, BETTING)).isTrue();
+    assertThat(properties.matches(Caller.BETTING_SERVICE, ADMIN)).isFalse();
+    assertThat(properties.matches(Caller.ADMIN_API, ADMIN)).isTrue();
+    assertThat(properties.matches(Caller.PLATFORM, PLATFORM)).isTrue();
+    assertThat(properties.toString()).doesNotContain(BETTING, ADMIN, PLATFORM);
+  }
+
+  @Test
+  void resolvesOnlyCanonicalCallerNames() {
+    assertThat(Caller.fromWire("betting-service")).contains(Caller.BETTING_SERVICE);
+    assertThat(Caller.fromWire("admin-api")).contains(Caller.ADMIN_API);
+    assertThat(Caller.fromWire("platform")).contains(Caller.PLATFORM);
+    assertThat(Caller.fromWire("BETTING-SERVICE")).isEmpty();
+  }
+}


## `feat(auth): authenticate internal callers`

diff --git a/src/main/java/com/sportsbook/risk/auth/InternalAuthenticationFilter.java b/src/main/java/com/sportsbook/risk/auth/InternalAuthenticationFilter.java
new file mode 100644
index 0000000..94a6fb7
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/auth/InternalAuthenticationFilter.java
@@ -0,0 +1,78 @@
+package com.sportsbook.risk.auth;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.error.ProblemDetail;
+import com.sportsbook.risk.auth.InternalAuthProperties.Caller;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.URI;
+import java.util.List;
+import org.springframework.http.MediaType;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.stereotype.Component;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+/** Authenticates internal callers with a caller-specific API key. */
+@Component
+public class InternalAuthenticationFilter extends OncePerRequestFilter {
+  public static final String SERVICE_HEADER = "X-Internal-Service";
+  public static final String API_KEY_HEADER = "X-Internal-Api-Key";
+
+  private final InternalAuthProperties properties;
+  private final ObjectMapper mapper;
+
+  public InternalAuthenticationFilter(InternalAuthProperties properties, ObjectMapper mapper) {
+    this.properties = properties;
+    this.mapper = mapper;
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    Caller caller = Caller.fromWire(request.getHeader(SERVICE_HEADER)).orElse(null);
+    if (caller == null || !properties.matches(caller, request.getHeader(API_KEY_HEADER))) {
+      unauthorized(request, response);
+      return;
+    }
+    var authority = new SimpleGrantedAuthority("ROLE_" + caller.name());
+    var authentication =
+        UsernamePasswordAuthenticationToken.authenticated(
+            caller.wireName(), null, List.of(authority));
+    SecurityContextHolder.getContext().setAuthentication(authentication);
+    request.setAttribute(Caller.class.getName(), caller);
+    chain.doFilter(request, response);
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    String path = request.getRequestURI();
+    if (path.equals("/actuator/health")
+        || path.startsWith("/actuator/health/")
+        || path.equals("/actuator/prometheus")) {
+      return true;
+    }
+    return !path.startsWith("/internal/") && !path.startsWith("/actuator/");
+  }
+
+  private void unauthorized(HttpServletRequest request, HttpServletResponse response)
+      throws IOException {
+    ProblemDetail problem =
+        new ProblemDetail(
+            URI.create("https://sportsbook/errors/unauthorized"),
+            "Unauthorized",
+            HttpServletResponse.SC_UNAUTHORIZED,
+            "UNAUTHORIZED",
+            "Missing or invalid internal credentials",
+            URI.create(request.getRequestURI()),
+            null);
+    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    mapper.writeValue(response.getOutputStream(), problem);
+  }
+}


## `test(auth): reject invalid internal callers`

diff --git a/src/test/java/com/sportsbook/risk/auth/InternalAuthenticationFilterTest.java b/src/test/java/com/sportsbook/risk/auth/InternalAuthenticationFilterTest.java
new file mode 100644
index 0000000..24c11d2
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/auth/InternalAuthenticationFilterTest.java
@@ -0,0 +1,94 @@
+package com.sportsbook.risk.auth;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.risk.auth.InternalAuthProperties.Caller;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.core.Authentication;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class InternalAuthenticationFilterTest {
+  private static final String BETTING = "b".repeat(32);
+  private static final String ADMIN = "a".repeat(32);
+  private static final String PLATFORM = "p".repeat(32);
+
+  private final InternalAuthenticationFilter filter =
+      new InternalAuthenticationFilter(
+          new InternalAuthProperties(BETTING, ADMIN, PLATFORM), new ObjectMapper());
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void rejectsMissingUnknownAndInvalidCredentials() throws Exception {
+    assertUnauthorized(null, null);
+    assertUnauthorized("unknown", BETTING);
+    assertUnauthorized("betting-service", ADMIN);
+  }
+
+  @Test
+  void authenticatesTheMatchingCallerWithoutRetainingTheSecret() throws Exception {
+    MockHttpServletRequest request = request("/internal/v1/risk/reservations");
+    request.addHeader(InternalAuthenticationFilter.SERVICE_HEADER, "betting-service");
+    request.addHeader(InternalAuthenticationFilter.API_KEY_HEADER, BETTING);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    AtomicReference<Authentication> observed = new AtomicReference<>();
+
+    filter.doFilter(
+        request,
+        response,
+        (ignoredRequest, ignoredResponse) ->
+            observed.set(SecurityContextHolder.getContext().getAuthentication()));
+
+    assertThat(response.getStatus()).isEqualTo(200);
+    assertThat(observed.get().getName()).isEqualTo("betting-service");
+    assertThat(observed.get().getAuthorities())
+        .extracting("authority")
+        .containsExactly("ROLE_BETTING_SERVICE");
+    assertThat(request.getAttribute(Caller.class.getName())).isEqualTo(Caller.BETTING_SERVICE);
+    assertThat(observed.get().getCredentials()).isNull();
+  }
+
+  @Test
+  void leavesAnonymousHealthRequestsUntouched() throws Exception {
+    MockHttpServletRequest request = request("/actuator/health/readiness");
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    var invoked = new java.util.concurrent.atomic.AtomicBoolean();
+
+    filter.doFilter(request, response, (ignoredRequest, ignoredResponse) -> invoked.set(true));
+
+    assertThat(invoked).isTrue();
+  }
+
+  private void assertUnauthorized(String caller, String secret) throws Exception {
+    MockHttpServletRequest request = request("/internal/v1/risk/check");
+    if (caller != null) {
+      request.addHeader(InternalAuthenticationFilter.SERVICE_HEADER, caller);
+    }
+    if (secret != null) {
+      request.addHeader(InternalAuthenticationFilter.API_KEY_HEADER, secret);
+    }
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    var invoked = new java.util.concurrent.atomic.AtomicBoolean();
+
+    filter.doFilter(request, response, (ignoredRequest, ignoredResponse) -> invoked.set(true));
+
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentType()).isEqualTo("application/problem+json");
+    assertThat(response.getContentAsString())
+        .contains("UNAUTHORIZED")
+        .doesNotContain(BETTING, ADMIN);
+    assertThat(invoked).isFalse();
+  }
+
+  private static MockHttpServletRequest request(String path) {
+    return new MockHttpServletRequest("POST", path);
+  }
+}


## `feat(auth): authorize internal service routes`

diff --git a/src/main/java/com/sportsbook/risk/auth/InternalSecurityConfiguration.java b/src/main/java/com/sportsbook/risk/auth/InternalSecurityConfiguration.java
new file mode 100644
index 0000000..65e3149
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/auth/InternalSecurityConfiguration.java
@@ -0,0 +1,75 @@
+package com.sportsbook.risk.auth;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.error.ProblemDetail;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.URI;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.MediaType;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.access.intercept.AuthorizationFilter;
+
+/** Restricts every internal operation to its owning service principal. */
+@Configuration
+public class InternalSecurityConfiguration {
+  @Bean
+  SecurityFilterChain internalSecurity(
+      HttpSecurity http, InternalAuthenticationFilter authentication, ObjectMapper mapper)
+      throws Exception {
+    return http.csrf(csrf -> csrf.disable())
+        .sessionManagement(
+            sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .requestCache(cache -> cache.disable())
+        .httpBasic(basic -> basic.disable())
+        .formLogin(form -> form.disable())
+        .logout(logout -> logout.disable())
+        .authorizeHttpRequests(
+            requests ->
+                requests
+                    .requestMatchers(
+                        "/actuator/health", "/actuator/health/**", "/actuator/prometheus")
+                    .permitAll()
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/risk/reservations")
+                    .hasRole("BETTING_SERVICE")
+                    .requestMatchers(HttpMethod.PUT, "/internal/v1/risk/reservations/*/commit")
+                    .hasRole("BETTING_SERVICE")
+                    .requestMatchers(HttpMethod.DELETE, "/internal/v1/risk/reservations/*")
+                    .hasRole("BETTING_SERVICE")
+                    .requestMatchers("/internal/v1/risk/limits/**")
+                    .hasRole("ADMIN_API")
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/risk/check")
+                    .hasRole("PLATFORM")
+                    .requestMatchers("/actuator/**")
+                    .hasRole("PLATFORM")
+                    .anyRequest()
+                    .denyAll())
+        .exceptionHandling(
+            errors ->
+                errors.accessDeniedHandler(
+                    (request, response, denied) ->
+                        forbidden(request.getRequestURI(), response, mapper)))
+        .addFilterBefore(authentication, AuthorizationFilter.class)
+        .build();
+  }
+
+  private static void forbidden(String path, HttpServletResponse response, ObjectMapper mapper)
+      throws IOException {
+    ProblemDetail problem =
+        new ProblemDetail(
+            URI.create("https://sportsbook/errors/forbidden"),
+            "Forbidden",
+            HttpServletResponse.SC_FORBIDDEN,
+            "FORBIDDEN",
+            "The authenticated caller does not own this operation",
+            URI.create(path),
+            null);
+    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    mapper.writeValue(response.getOutputStream(), problem);
+  }
+}


## `test(auth): verify internal route ownership`

diff --git a/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java b/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java
new file mode 100644
index 0000000..28b3cc1
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/auth/InternalSecurityIntegrationTest.java
@@ -0,0 +1,96 @@
+package com.sportsbook.risk.auth;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.patch;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
+
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.http.HttpMethod;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder;
+
+@SpringBootTest(
+    properties = {
+      "risk.auth.betting-service-api-key=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
+      "risk.auth.admin-api-key=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
+      "risk.auth.platform-api-key=pppppppppppppppppppppppppppppppp",
+      "management.health.redis.enabled=false"
+    })
+@AutoConfigureMockMvc
+class InternalSecurityIntegrationTest {
+  private static final String BETTING = "b".repeat(32);
+  private static final String ADMIN = "a".repeat(32);
+  private static final String PLATFORM = "p".repeat(32);
+
+  @Autowired private MockMvc mvc;
+
+  @Test
+  void permitsOnlyTheOwnerOfEachInternalRoute() throws Exception {
+    List<Route> routes =
+        List.of(
+            new Route(
+                HttpMethod.POST, "/internal/v1/risk/reservations", "betting-service", BETTING),
+            new Route(
+                HttpMethod.PUT,
+                "/internal/v1/risk/reservations/00000000-0000-0000-0000-000000000001/commit",
+                "betting-service",
+                BETTING),
+            new Route(
+                HttpMethod.DELETE,
+                "/internal/v1/risk/reservations/00000000-0000-0000-0000-000000000001",
+                "betting-service",
+                BETTING),
+            new Route(HttpMethod.GET, "/internal/v1/risk/limits/x", "admin-api", ADMIN),
+            new Route(HttpMethod.PATCH, "/internal/v1/risk/limits/x", "admin-api", ADMIN),
+            new Route(HttpMethod.POST, "/internal/v1/risk/check", "platform", PLATFORM));
+
+    for (Route route : routes) {
+      int ownerStatus = mvc.perform(route.request()).andReturn().getResponse().getStatus();
+      assertThat(ownerStatus).isNotIn(401, 403);
+      String otherCaller = route.caller.equals("platform") ? "admin-api" : "platform";
+      String otherSecret = route.caller.equals("platform") ? ADMIN : PLATFORM;
+      assertThat(
+              mvc.perform(route.request(otherCaller, otherSecret))
+                  .andReturn()
+                  .getResponse()
+                  .getStatus())
+          .isEqualTo(403);
+    }
+  }
+
+  @Test
+  void distinguishesAnonymousHealthFromProtectedRequests() throws Exception {
+    assertThat(mvc.perform(get("/actuator/health/liveness")).andReturn().getResponse().getStatus())
+        .isEqualTo(200);
+    assertThat(mvc.perform(post("/internal/v1/risk/check")).andReturn().getResponse().getStatus())
+        .isEqualTo(401);
+  }
+
+  private record Route(HttpMethod method, String path, String caller, String secret) {
+    MockHttpServletRequestBuilder request() {
+      return request(caller, secret);
+    }
+
+    MockHttpServletRequestBuilder request(String service, String key) {
+      MockHttpServletRequestBuilder request =
+          switch (method.name()) {
+            case "GET" -> get(path);
+            case "PATCH" -> patch(path);
+            case "POST" -> post(path);
+            case "PUT" -> put(path);
+            case "DELETE" -> delete(path);
+            default -> throw new IllegalArgumentException("unsupported method");
+          };
+      return request
+          .header(InternalAuthenticationFilter.SERVICE_HEADER, service)
+          .header(InternalAuthenticationFilter.API_KEY_HEADER, key);
+    }
+  }
+}
diff --git a/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
new file mode 100644
index 0000000..fdbd0b1
--- /dev/null
+++ b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
@@ -0,0 +1 @@
+mock-maker-subclass
