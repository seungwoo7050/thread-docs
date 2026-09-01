# 내부 서비스 인증과 권한 경계

## `build(security): establish a default-deny HTTP boundary`

diff --git a/pom.xml b/pom.xml
index 90278bc..eaa092a 100644
--- a/pom.xml
+++ b/pom.xml
@@ -51,6 +51,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-web</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-security</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-validation</artifactId>
diff --git a/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
new file mode 100644
index 0000000..394cd32
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
@@ -0,0 +1,44 @@
+package com.sportsbook.wallet.security;
+
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpMethod;
+import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.web.SecurityFilterChain;
+
+/** Establishes the closed HTTP boundary before monetary routes are exposed. */
+@Configuration
+public class WalletSecurityConfig {
+
+  @Bean
+  SecurityFilterChain walletSecurityFilterChain(HttpSecurity http) throws Exception {
+    return http.csrf(csrf -> csrf.disable())
+        .formLogin(form -> form.disable())
+        .httpBasic(basic -> basic.disable())
+        .logout(logout -> logout.disable())
+        .requestCache(cache -> cache.disable())
+        .sessionManagement(
+            sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .authorizeHttpRequests(
+            requests ->
+                requests
+                    .requestMatchers(
+                        HttpMethod.GET,
+                        "/actuator/health",
+                        "/actuator/health/**",
+                        "/actuator/prometheus")
+                    .permitAll()
+                    .anyRequest()
+                    .denyAll())
+        .build();
+  }
+
+  @Bean
+  AuthenticationManager rejectingAuthenticationManager() {
+    return authentication -> {
+      throw new UnsupportedOperationException("No configured wallet authentication mechanism");
+    };
+  }
+}


## `test(security): verify anonymous management exceptions`

diff --git a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
new file mode 100644
index 0000000..e441f22
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.wallet.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.ListableBeanFactory;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.userdetails.UserDetailsService;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.MvcResult;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@WebMvcTest(controllers = WalletSecurityConfigTest.TestEndpoints.class)
+@Import({WalletSecurityConfig.class, WalletSecurityConfigTest.TestEndpoints.class})
+class WalletSecurityConfigTest {
+
+  @Autowired private MockMvc mvc;
+  @Autowired private ListableBeanFactory beans;
+  @Autowired private AuthenticationManager authenticationManager;
+
+  @Test
+  void allowsOnlyAnonymousManagementProbes() throws Exception {
+    MvcResult health = mvc.perform(get("/actuator/health")).andExpect(status().isOk()).andReturn();
+    mvc.perform(get("/actuator/health/liveness")).andExpect(status().isOk());
+    mvc.perform(get("/actuator/prometheus")).andExpect(status().isOk());
+
+    mvc.perform(get("/actuator/info")).andExpect(status().is4xxClientError());
+    mvc.perform(get("/actuator/metrics")).andExpect(status().is4xxClientError());
+    mvc.perform(get("/internal/test")).andExpect(status().is4xxClientError());
+    mvc.perform(post("/actuator/health")).andExpect(status().is4xxClientError());
+    assertThat(health.getRequest().getSession(false)).isNull();
+  }
+
+  @Test
+  void createsNoGeneratedUserStore() {
+    assertThat(beans.getBeanProvider(UserDetailsService.class).stream()).isEmpty();
+    assertThatThrownBy(
+            () ->
+                authenticationManager.authenticate(
+                    UsernamePasswordAuthenticationToken.unauthenticated("caller", "secret")))
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
+
+  @RestController
+  static class TestEndpoints {
+    @GetMapping({
+      "/actuator/health",
+      "/actuator/health/liveness",
+      "/actuator/prometheus",
+      "/actuator/info",
+      "/internal/test"
+    })
+    String getEndpoint() {
+      return "ok";
+    }
+
+    @PostMapping("/actuator/health")
+    String postHealth() {
+      return "ok";
+    }
+  }
+}


## `feat(security): define caller wire names`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletCaller.java b/src/main/java/com/sportsbook/wallet/domain/WalletCaller.java
index 4221e35..e006f49 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletCaller.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletCaller.java
@@ -1,10 +1,27 @@
 package com.sportsbook.wallet.domain;
 
+import java.util.Arrays;
+import java.util.Optional;
+
 /** Authenticated service identity participating in request fingerprints and audit records. */
 public enum WalletCaller {
-  PLATFORM,
-  GATEWAY,
-  BETTING,
-  SETTLEMENT,
-  ADMIN
+  PLATFORM("platform"),
+  GATEWAY("gateway"),
+  BETTING("betting-service"),
+  SETTLEMENT("settlement-service"),
+  ADMIN("admin-api");
+
+  private final String wireName;
+
+  WalletCaller(String wireName) {
+    this.wireName = wireName;
+  }
+
+  public String wireName() {
+    return wireName;
+  }
+
+  public static Optional<WalletCaller> fromWireName(String wireName) {
+    return Arrays.stream(values()).filter(caller -> caller.wireName.equals(wireName)).findFirst();
+  }
 }


## `feat(security): validate caller API keys`

diff --git a/src/main/java/com/sportsbook/wallet/security/WalletSecurityProperties.java b/src/main/java/com/sportsbook/wallet/security/WalletSecurityProperties.java
new file mode 100644
index 0000000..e41404e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/security/WalletSecurityProperties.java
@@ -0,0 +1,48 @@
+package com.sportsbook.wallet.security;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.HashSet;
+import java.util.Map;
+import java.util.Objects;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Environment-bound internal credentials, validated before any HTTP request is accepted. */
+@ConfigurationProperties("wallet.security")
+public final class WalletSecurityProperties {
+  static final int MINIMUM_KEY_LENGTH = 32;
+
+  private final Map<WalletCaller, String> apiKeys;
+
+  public WalletSecurityProperties(
+      String platformApiKey,
+      String gatewayApiKey,
+      String bettingServiceApiKey,
+      String settlementServiceApiKey,
+      String adminApiKey) {
+    apiKeys =
+        Map.of(
+            WalletCaller.PLATFORM, requireKey(WalletCaller.PLATFORM, platformApiKey),
+            WalletCaller.GATEWAY, requireKey(WalletCaller.GATEWAY, gatewayApiKey),
+            WalletCaller.BETTING, requireKey(WalletCaller.BETTING, bettingServiceApiKey),
+            WalletCaller.SETTLEMENT, requireKey(WalletCaller.SETTLEMENT, settlementServiceApiKey),
+            WalletCaller.ADMIN, requireKey(WalletCaller.ADMIN, adminApiKey));
+    if (new HashSet<>(apiKeys.values()).size() != apiKeys.size()) {
+      throw new IllegalArgumentException("Wallet caller API keys must be distinct");
+    }
+  }
+
+  String apiKey(WalletCaller caller) {
+    return apiKeys.get(Objects.requireNonNull(caller, "caller"));
+  }
+
+  private static String requireKey(WalletCaller caller, String key) {
+    if (key == null || key.isBlank() || key.length() < MINIMUM_KEY_LENGTH) {
+      throw new IllegalArgumentException(
+          caller.wireName()
+              + " API key must contain at least "
+              + MINIMUM_KEY_LENGTH
+              + " characters");
+    }
+    return key;
+  }
+}


## `feat(security): compare caller credential digests`

diff --git a/src/main/java/com/sportsbook/wallet/security/WalletCredentials.java b/src/main/java/com/sportsbook/wallet/security/WalletCredentials.java
new file mode 100644
index 0000000..891b63e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/security/WalletCredentials.java
@@ -0,0 +1,40 @@
+package com.sportsbook.wallet.security;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.EnumMap;
+import java.util.Map;
+import java.util.Optional;
+
+/** Holds only credential digests and compares every presented key in constant time. */
+public final class WalletCredentials {
+  private static final byte[] UNKNOWN_CALLER_DIGEST = digest("wallet-unknown-caller");
+
+  private final Map<WalletCaller, byte[]> callerDigests;
+
+  public WalletCredentials(WalletSecurityProperties properties) {
+    Map<WalletCaller, byte[]> digests = new EnumMap<>(WalletCaller.class);
+    for (WalletCaller caller : WalletCaller.values()) {
+      digests.put(caller, digest(properties.apiKey(caller)));
+    }
+    callerDigests = Map.copyOf(digests);
+  }
+
+  public Optional<WalletCaller> authenticate(String wireName, String apiKey) {
+    Optional<WalletCaller> caller = WalletCaller.fromWireName(wireName);
+    byte[] expected = caller.map(callerDigests::get).orElse(UNKNOWN_CALLER_DIGEST);
+    byte[] presented = digest(apiKey == null ? "" : apiKey);
+    boolean matches = MessageDigest.isEqual(expected, presented);
+    return matches ? caller : Optional.empty();
+  }
+
+  private static byte[] digest(String value) {
+    try {
+      return MessageDigest.getInstance("SHA-256").digest(value.getBytes(StandardCharsets.UTF_8));
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("SHA-256 is unavailable", impossible);
+    }
+  }
+}


## `test(security): reject cross-caller credentials`

diff --git a/src/test/java/com/sportsbook/wallet/security/WalletCredentialsTest.java b/src/test/java/com/sportsbook/wallet/security/WalletCredentialsTest.java
new file mode 100644
index 0000000..a593954
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/security/WalletCredentialsTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.wallet.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.junit.jupiter.params.provider.Arguments.arguments;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.Arrays;
+import java.util.Map;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.EnumSource;
+import org.junit.jupiter.params.provider.MethodSource;
+
+class WalletCredentialsTest {
+  private static final Map<WalletCaller, String> KEYS =
+      Map.of(
+          WalletCaller.PLATFORM, "platform:" + "p".repeat(32),
+          WalletCaller.GATEWAY, "gateway:" + "g".repeat(32),
+          WalletCaller.BETTING, "betting:" + "b".repeat(32),
+          WalletCaller.SETTLEMENT, "settlement:" + "s".repeat(32),
+          WalletCaller.ADMIN, "admin:" + "a".repeat(32));
+
+  private final WalletCredentials credentials =
+      new WalletCredentials(
+          new WalletSecurityProperties(
+              KEYS.get(WalletCaller.PLATFORM),
+              KEYS.get(WalletCaller.GATEWAY),
+              KEYS.get(WalletCaller.BETTING),
+              KEYS.get(WalletCaller.SETTLEMENT),
+              KEYS.get(WalletCaller.ADMIN)));
+
+  @ParameterizedTest
+  @EnumSource(WalletCaller.class)
+  void authenticatesEveryExactCallerKeyPair(WalletCaller caller) {
+    assertThat(credentials.authenticate(caller.wireName(), KEYS.get(caller))).contains(caller);
+  }
+
+  @ParameterizedTest
+  @MethodSource("crossCallerPairs")
+  void rejectsEveryCrossCallerKeyPair(WalletCaller claimed, WalletCaller keyOwner) {
+    assertThat(credentials.authenticate(claimed.wireName(), KEYS.get(keyOwner))).isEmpty();
+  }
+
+  @Test
+  void rejectsMalformedIdentitiesAndKeys() {
+    assertThat(credentials.authenticate(null, KEYS.get(WalletCaller.PLATFORM))).isEmpty();
+    assertThat(credentials.authenticate("unknown", KEYS.get(WalletCaller.PLATFORM))).isEmpty();
+    assertThat(credentials.authenticate("unknown", "wallet-unknown-caller")).isEmpty();
+    assertThat(credentials.authenticate("PLATFORM", KEYS.get(WalletCaller.PLATFORM))).isEmpty();
+    assertThat(credentials.authenticate("platform", null)).isEmpty();
+    assertThat(credentials.authenticate("platform", "wrong-key")).isEmpty();
+  }
+
+  private static Stream<Arguments> crossCallerPairs() {
+    return Arrays.stream(WalletCaller.values())
+        .flatMap(
+            claimed ->
+                Arrays.stream(WalletCaller.values())
+                    .filter(keyOwner -> keyOwner != claimed)
+                    .map(keyOwner -> arguments(claimed, keyOwner)));
+  }
+}


## `feat(security): render security failures as problems`

diff --git a/src/main/java/com/sportsbook/wallet/security/WalletSecurityFailureHandler.java b/src/main/java/com/sportsbook/wallet/security/WalletSecurityFailureHandler.java
new file mode 100644
index 0000000..46dcf57
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/security/WalletSecurityFailureHandler.java
@@ -0,0 +1,62 @@
+package com.sportsbook.wallet.security;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.wallet.web.WalletError;
+import com.sportsbook.wallet.web.WalletProblems;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.URI;
+import java.nio.charset.StandardCharsets;
+import java.util.Objects;
+import org.springframework.http.MediaType;
+import org.springframework.http.ProblemDetail;
+import org.springframework.security.core.AuthenticationException;
+import org.springframework.security.web.AuthenticationEntryPoint;
+import org.springframework.security.web.access.AccessDeniedHandler;
+
+/** Emits fixed problem bodies without reflecting credentials or exception messages. */
+public final class WalletSecurityFailureHandler
+    implements AuthenticationEntryPoint, AccessDeniedHandler {
+  static final String AUTHENTICATION_DETAIL = "Valid internal service credentials are required";
+  static final String ACCESS_DETAIL = "Authenticated caller cannot access this wallet resource";
+
+  private final ObjectMapper objectMapper;
+
+  public WalletSecurityFailureHandler(ObjectMapper objectMapper) {
+    this.objectMapper = Objects.requireNonNull(objectMapper, "objectMapper");
+  }
+
+  @Override
+  public void commence(
+      HttpServletRequest request, HttpServletResponse response, AuthenticationException exception)
+      throws IOException, ServletException {
+    authenticationRequired(request, response);
+  }
+
+  @Override
+  public void handle(
+      HttpServletRequest request,
+      HttpServletResponse response,
+      org.springframework.security.access.AccessDeniedException exception)
+      throws IOException, ServletException {
+    write(request, response, WalletError.ACCESS_DENIED, ACCESS_DETAIL);
+  }
+
+  void authenticationRequired(HttpServletRequest request, HttpServletResponse response)
+      throws IOException {
+    write(request, response, WalletError.AUTHENTICATION_REQUIRED, AUTHENTICATION_DETAIL);
+  }
+
+  private void write(
+      HttpServletRequest request, HttpServletResponse response, WalletError error, String detail)
+      throws IOException {
+    ProblemDetail problem = WalletProblems.from(error, detail);
+    problem.setInstance(URI.create(request.getRequestURI()));
+    response.setStatus(error.httpStatus());
+    response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
+    response.setCharacterEncoding(StandardCharsets.UTF_8.name());
+    objectMapper.writeValue(response.getOutputStream(), problem);
+  }
+}


## `feat(security): authenticate internal API keys`

diff --git a/src/main/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilter.java b/src/main/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilter.java
new file mode 100644
index 0000000..2c07f52
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilter.java
@@ -0,0 +1,68 @@
+package com.sportsbook.wallet.security;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.util.Collections;
+import java.util.List;
+import java.util.Objects;
+import java.util.Optional;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.context.SecurityContext;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+/** Authenticates one unambiguous internal caller header pair without retaining its API key. */
+public final class InternalApiKeyAuthenticationFilter extends OncePerRequestFilter {
+  public static final String SERVICE_HEADER = "X-Internal-Service";
+  public static final String API_KEY_HEADER = "X-Internal-Api-Key";
+
+  private final WalletCredentials credentials;
+  private final WalletSecurityFailureHandler failureHandler;
+
+  public InternalApiKeyAuthenticationFilter(
+      WalletCredentials credentials, WalletSecurityFailureHandler failureHandler) {
+    this.credentials = Objects.requireNonNull(credentials, "credentials");
+    this.failureHandler = Objects.requireNonNull(failureHandler, "failureHandler");
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
+      throws ServletException, IOException {
+    List<String> callers = headers(request, SERVICE_HEADER);
+    List<String> apiKeys = headers(request, API_KEY_HEADER);
+    if (callers.isEmpty() && apiKeys.isEmpty()) {
+      filterChain.doFilter(request, response);
+      return;
+    }
+    if (callers.size() != 1 || apiKeys.size() != 1) {
+      reject(request, response);
+      return;
+    }
+
+    Optional<WalletCaller> caller = credentials.authenticate(callers.get(0), apiKeys.get(0));
+    if (caller.isEmpty()) {
+      reject(request, response);
+      return;
+    }
+
+    SecurityContext context = SecurityContextHolder.createEmptyContext();
+    context.setAuthentication(
+        new UsernamePasswordAuthenticationToken(caller.orElseThrow(), null, List.of()));
+    SecurityContextHolder.setContext(context);
+    filterChain.doFilter(request, response);
+  }
+
+  private List<String> headers(HttpServletRequest request, String name) {
+    return Collections.list(request.getHeaders(name));
+  }
+
+  private void reject(HttpServletRequest request, HttpServletResponse response) throws IOException {
+    SecurityContextHolder.clearContext();
+    failureHandler.authenticationRequired(request, response);
+  }
+}


