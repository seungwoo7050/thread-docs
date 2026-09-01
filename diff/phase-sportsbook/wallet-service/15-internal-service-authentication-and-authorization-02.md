## `test(security): establish authenticated caller principals`

diff --git a/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilterTest.java b/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilterTest.java
new file mode 100644
index 0000000..72d2532
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationFilterTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.wallet.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.EnumSource;
+import org.springframework.http.ProblemDetail;
+import org.springframework.http.converter.json.ProblemDetailJacksonMixin;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.core.Authentication;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class InternalApiKeyAuthenticationFilterTest {
+  private final WalletCredentials credentials = new WalletCredentials(properties());
+  private final WalletSecurityFailureHandler failureHandler =
+      new WalletSecurityFailureHandler(
+          new ObjectMapper().addMixIn(ProblemDetail.class, ProblemDetailJacksonMixin.class));
+  private final InternalApiKeyAuthenticationFilter filter =
+      new InternalApiKeyAuthenticationFilter(credentials, failureHandler);
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void continuesRequestsThatDoNotPresentCredentials() throws Exception {
+    AtomicBoolean invoked = new AtomicBoolean();
+    AtomicReference<Authentication> observed = new AtomicReference<>();
+
+    filter.doFilter(
+        request(),
+        new MockHttpServletResponse(),
+        (ignoredRequest, ignoredResponse) -> {
+          invoked.set(true);
+          observed.set(SecurityContextHolder.getContext().getAuthentication());
+        });
+
+    assertThat(invoked).isTrue();
+    assertThat(observed.get()).isNull();
+  }
+
+  @ParameterizedTest
+  @EnumSource(WalletCaller.class)
+  void authenticatesEachExactPairWithoutRetainingTheKey(WalletCaller caller) throws Exception {
+    MockHttpServletRequest request = request();
+    request.addHeader(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, caller.wireName());
+    request.addHeader(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, key(caller));
+    AtomicReference<Authentication> observed = new AtomicReference<>();
+
+    filter.doFilter(
+        request,
+        new MockHttpServletResponse(),
+        (ignoredRequest, ignoredResponse) ->
+            observed.set(SecurityContextHolder.getContext().getAuthentication()));
+
+    assertThat(observed.get().getPrincipal()).isEqualTo(caller);
+    assertThat(observed.get().getCredentials()).isNull();
+    assertThat(observed.get().getAuthorities()).isEmpty();
+    assertThat(observed.get().isAuthenticated()).isTrue();
+  }
+
+  @Test
+  void rejectsMissingDependencies() {
+    assertThatNullPointerException()
+        .isThrownBy(() -> new InternalApiKeyAuthenticationFilter(null, failureHandler));
+    assertThatNullPointerException()
+        .isThrownBy(() -> new InternalApiKeyAuthenticationFilter(credentials, null));
+  }
+
+  private MockHttpServletRequest request() {
+    return new MockHttpServletRequest("GET", "/internal/v1/wallet/balance");
+  }
+
+  private static WalletSecurityProperties properties() {
+    return new WalletSecurityProperties(
+        key(WalletCaller.PLATFORM),
+        key(WalletCaller.GATEWAY),
+        key(WalletCaller.BETTING),
+        key(WalletCaller.SETTLEMENT),
+        key(WalletCaller.ADMIN));
+  }
+
+  private static String key(WalletCaller caller) {
+    return caller.wireName() + ":" + caller.name().repeat(8);
+  }
+}


## `test(security): reject ambiguous service credentials`

diff --git a/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationRejectionTest.java b/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationRejectionTest.java
new file mode 100644
index 0000000..e9fe5b6
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/security/InternalApiKeyAuthenticationRejectionTest.java
@@ -0,0 +1,94 @@
+package com.sportsbook.wallet.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.List;
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.ProblemDetail;
+import org.springframework.http.converter.json.ProblemDetailJacksonMixin;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class InternalApiKeyAuthenticationRejectionTest {
+  private static final String PLATFORM_KEY = "platform:" + "p".repeat(32);
+
+  private final InternalApiKeyAuthenticationFilter filter =
+      new InternalApiKeyAuthenticationFilter(
+          new WalletCredentials(
+              new WalletSecurityProperties(
+                  PLATFORM_KEY,
+                  "gateway:" + "g".repeat(32),
+                  "betting:" + "b".repeat(32),
+                  "settlement:" + "s".repeat(32),
+                  "admin:" + "a".repeat(32))),
+          new WalletSecurityFailureHandler(
+              new ObjectMapper().addMixIn(ProblemDetail.class, ProblemDetailJacksonMixin.class)));
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void rejectsPartialDuplicateAndInvalidCredentials() throws Exception {
+    assertRejected(withHeader(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "platform"));
+    assertRejected(withHeader(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, PLATFORM_KEY));
+    assertRejected(duplicate(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "platform"));
+    assertRejected(duplicate(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, PLATFORM_KEY));
+    assertRejected(pair("platform", "invalid"));
+    assertRejected(pair(" ", PLATFORM_KEY));
+    assertRejected(pair("platform", " "));
+    assertRejected(pair("gateway", PLATFORM_KEY));
+    assertRejected(pair("unknown", PLATFORM_KEY));
+  }
+
+  @Test
+  void clearsPreexistingAuthenticationWhenCredentialsAreInvalid() throws Exception {
+    SecurityContextHolder.getContext()
+        .setAuthentication(
+            new UsernamePasswordAuthenticationToken(WalletCaller.ADMIN, null, List.of()));
+
+    assertRejected(pair("platform", "invalid"));
+  }
+
+  private void assertRejected(MockHttpServletRequest request) throws Exception {
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    AtomicBoolean invoked = new AtomicBoolean();
+    filter.doFilter(request, response, (ignoredRequest, ignoredResponse) -> invoked.set(true));
+    assertThat(invoked).isFalse();
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentAsString()).contains("WALLET_AUTHENTICATION_REQUIRED");
+    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
+  }
+
+  private MockHttpServletRequest pair(String caller, String key) {
+    MockHttpServletRequest request =
+        withHeader(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, caller);
+    request.addHeader(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, key);
+    return request;
+  }
+
+  private MockHttpServletRequest duplicate(String name, String value) {
+    MockHttpServletRequest request = withHeader(name, value);
+    request.addHeader(name, value);
+    if (name.equals(InternalApiKeyAuthenticationFilter.SERVICE_HEADER)) {
+      request.addHeader(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, PLATFORM_KEY);
+    } else {
+      request.addHeader(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "platform");
+    }
+    return request;
+  }
+
+  private MockHttpServletRequest withHeader(String name, String value) {
+    MockHttpServletRequest request =
+        new MockHttpServletRequest("GET", "/internal/v1/wallet/balance");
+    request.addHeader(name, value);
+    return request;
+  }
+}


## `feat(security): activate internal request authentication`

diff --git a/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
index 394cd32..96e0507 100644
--- a/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
+++ b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
@@ -1,19 +1,29 @@
 package com.sportsbook.wallet.security;
 
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.wallet.domain.WalletCaller;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.http.HttpMethod;
 import org.springframework.security.authentication.AuthenticationManager;
+import org.springframework.security.authorization.AuthorizationDecision;
 import org.springframework.security.config.annotation.web.builders.HttpSecurity;
 import org.springframework.security.config.http.SessionCreationPolicy;
 import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.authentication.AnonymousAuthenticationFilter;
 
 /** Establishes the closed HTTP boundary before monetary routes are exposed. */
 @Configuration
+@EnableConfigurationProperties(WalletSecurityProperties.class)
 public class WalletSecurityConfig {
 
   @Bean
-  SecurityFilterChain walletSecurityFilterChain(HttpSecurity http) throws Exception {
+  SecurityFilterChain walletSecurityFilterChain(
+      HttpSecurity http, WalletCredentials credentials, WalletSecurityFailureHandler failures)
+      throws Exception {
+    InternalApiKeyAuthenticationFilter authentication =
+        new InternalApiKeyAuthenticationFilter(credentials, failures);
     return http.csrf(csrf -> csrf.disable())
         .formLogin(form -> form.disable())
         .httpBasic(basic -> basic.disable())
@@ -21,6 +31,9 @@ public class WalletSecurityConfig {
         .requestCache(cache -> cache.disable())
         .sessionManagement(
             sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .exceptionHandling(
+            exceptions ->
+                exceptions.authenticationEntryPoint(failures).accessDeniedHandler(failures))
         .authorizeHttpRequests(
             requests ->
                 requests
@@ -30,11 +43,27 @@ public class WalletSecurityConfig {
                         "/actuator/health/**",
                         "/actuator/prometheus")
                     .permitAll()
+                    .requestMatchers("/actuator", "/actuator/**")
+                    .access(
+                        (authenticated, context) ->
+                            new AuthorizationDecision(
+                                WalletCaller.PLATFORM.equals(authenticated.get().getPrincipal())))
                     .anyRequest()
                     .denyAll())
+        .addFilterBefore(authentication, AnonymousAuthenticationFilter.class)
         .build();
   }
 
+  @Bean
+  WalletCredentials walletCredentials(WalletSecurityProperties properties) {
+    return new WalletCredentials(properties);
+  }
+
+  @Bean
+  WalletSecurityFailureHandler walletSecurityFailureHandler(ObjectMapper objectMapper) {
+    return new WalletSecurityFailureHandler(objectMapper);
+  }
+
   @Bean
   AuthenticationManager rejectingAuthenticationManager() {
     return authentication -> {
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 8c84a11..a20a6ed 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -76,6 +76,12 @@ logging:
     org.hibernate.SQL: WARN
 
 wallet:
+  security:
+    platform-api-key: ${WALLET_PLATFORM_API_KEY}
+    gateway-api-key: ${WALLET_GATEWAY_API_KEY}
+    betting-service-api-key: ${WALLET_BETTING_SERVICE_API_KEY}
+    settlement-service-api-key: ${WALLET_SETTLEMENT_SERVICE_API_KEY}
+    admin-api-key: ${WALLET_ADMIN_API_KEY}
   integrity:
     scheduling-enabled: ${WALLET_INTEGRITY_ENABLED:true}
     poll-interval: ${WALLET_INTEGRITY_POLL_INTERVAL:PT30S}


## `feat(security): authorize wallet route capabilities`

diff --git a/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
index 96e0507..c9f8c9c 100644
--- a/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
+++ b/src/main/java/com/sportsbook/wallet/security/WalletSecurityConfig.java
@@ -2,15 +2,18 @@ package com.sportsbook.wallet.security;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.Set;
 import org.springframework.boot.context.properties.EnableConfigurationProperties;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 import org.springframework.http.HttpMethod;
 import org.springframework.security.authentication.AuthenticationManager;
 import org.springframework.security.authorization.AuthorizationDecision;
+import org.springframework.security.authorization.AuthorizationManager;
 import org.springframework.security.config.annotation.web.builders.HttpSecurity;
 import org.springframework.security.config.http.SessionCreationPolicy;
 import org.springframework.security.web.SecurityFilterChain;
+import org.springframework.security.web.access.intercept.RequestAuthorizationContext;
 import org.springframework.security.web.authentication.AnonymousAuthenticationFilter;
 
 /** Establishes the closed HTTP boundary before monetary routes are exposed. */
@@ -44,10 +47,31 @@ public class WalletSecurityConfig {
                         "/actuator/prometheus")
                     .permitAll()
                     .requestMatchers("/actuator", "/actuator/**")
+                    .access(callers(WalletCaller.PLATFORM))
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/wallet/accounts")
+                    .access(callers(WalletCaller.PLATFORM))
+                    .requestMatchers(HttpMethod.GET, "/internal/v1/wallet/accounts/*/balance")
+                    .access(callers(WalletCaller.PLATFORM, WalletCaller.GATEWAY))
+                    .requestMatchers(
+                        HttpMethod.POST,
+                        "/internal/v1/wallet/transactions/deposit",
+                        "/internal/v1/wallet/transactions/withdraw")
+                    .access(callers(WalletCaller.PLATFORM))
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/wallet/transactions/debit")
+                    .access(callers(WalletCaller.BETTING))
+                    .requestMatchers(HttpMethod.GET, "/internal/v1/wallet/transactions/debit/*")
+                    .access(callers(WalletCaller.BETTING))
+                    .requestMatchers(HttpMethod.POST, "/internal/v1/wallet/transactions/credit")
                     .access(
-                        (authenticated, context) ->
-                            new AuthorizationDecision(
-                                WalletCaller.PLATFORM.equals(authenticated.get().getPrincipal())))
+                        callers(WalletCaller.BETTING, WalletCaller.SETTLEMENT, WalletCaller.ADMIN))
+                    .requestMatchers(
+                        HttpMethod.POST,
+                        "/internal/v1/wallet/transactions/forfeit",
+                        "/internal/v1/wallet/transactions/adjustment")
+                    .access(callers(WalletCaller.SETTLEMENT))
+                    .requestMatchers(
+                        HttpMethod.GET, "/internal/v1/wallet/transactions/adjustment/*")
+                    .access(callers(WalletCaller.SETTLEMENT))
                     .anyRequest()
                     .denyAll())
         .addFilterBefore(authentication, AnonymousAuthenticationFilter.class)
@@ -64,6 +88,13 @@ public class WalletSecurityConfig {
     return new WalletSecurityFailureHandler(objectMapper);
   }
 
+  private static AuthorizationManager<RequestAuthorizationContext> callers(
+      WalletCaller... allowedCallers) {
+    Set<WalletCaller> allowed = Set.of(allowedCallers);
+    return (authentication, context) ->
+        new AuthorizationDecision(allowed.contains(authentication.get().getPrincipal()));
+  }
+
   @Bean
   AuthenticationManager rejectingAuthenticationManager() {
     return authentication -> {


## `test(security): lock caller route capabilities`

diff --git a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
index a19bede..0494ec3 100644
--- a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
+++ b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
@@ -4,17 +4,23 @@ import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.request;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
 import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.Set;
+import java.util.stream.Stream;
 import org.junit.jupiter.api.Test;
 import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
 import org.junit.jupiter.params.provider.EnumSource;
+import org.junit.jupiter.params.provider.MethodSource;
 import org.springframework.beans.factory.ListableBeanFactory;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
 import org.springframework.context.annotation.Import;
+import org.springframework.http.HttpMethod;
 import org.springframework.security.authentication.AuthenticationManager;
 import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
 import org.springframework.security.core.userdetails.UserDetailsService;
@@ -27,6 +33,8 @@ import org.springframework.test.web.servlet.MockMvc;
 import org.springframework.test.web.servlet.MvcResult;
 import org.springframework.web.bind.annotation.GetMapping;
 import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestMethod;
 import org.springframework.web.bind.annotation.RestController;
 
 @WebMvcTest(controllers = WalletSecurityConfigTest.TestEndpoints.class)
@@ -115,13 +123,63 @@ class WalletSecurityConfigTest {
     assertThat(beans.getBeanProvider(InternalApiKeyAuthenticationFilter.class).stream()).isEmpty();
   }
 
+  @ParameterizedTest
+  @MethodSource("walletRoutes")
+  void enforcesEveryWalletRouteCapability(WalletRoute route) throws Exception {
+    for (WalletCaller caller : WalletCaller.values()) {
+      int expected = route.allowed().contains(caller) ? 200 : 403;
+      mvc.perform(internalRequest(route.method(), route.path(), caller))
+          .andExpect(status().is(expected));
+    }
+  }
+
   private org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder internalGet(
       String path, WalletCaller caller) {
-    return get(path)
+    return internalRequest(HttpMethod.GET, path, caller);
+  }
+
+  private org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder
+      internalRequest(HttpMethod method, String path, WalletCaller caller) {
+    return request(method, path)
         .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, caller.wireName())
         .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, TestInternalApiKeys.key(caller));
   }
 
+  static Stream<Arguments> walletRoutes() {
+    return Stream.of(
+        route(HttpMethod.POST, "/internal/v1/wallet/accounts", WalletCaller.PLATFORM),
+        route(
+            HttpMethod.GET,
+            "/internal/v1/wallet/accounts/user/balance",
+            WalletCaller.PLATFORM,
+            WalletCaller.GATEWAY),
+        route(HttpMethod.POST, "/internal/v1/wallet/transactions/deposit", WalletCaller.PLATFORM),
+        route(HttpMethod.POST, "/internal/v1/wallet/transactions/withdraw", WalletCaller.PLATFORM),
+        route(HttpMethod.POST, "/internal/v1/wallet/transactions/debit", WalletCaller.BETTING),
+        route(HttpMethod.GET, "/internal/v1/wallet/transactions/debit/bet", WalletCaller.BETTING),
+        route(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/credit",
+            WalletCaller.BETTING,
+            WalletCaller.SETTLEMENT,
+            WalletCaller.ADMIN),
+        route(HttpMethod.POST, "/internal/v1/wallet/transactions/forfeit", WalletCaller.SETTLEMENT),
+        route(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/adjustment",
+            WalletCaller.SETTLEMENT),
+        route(
+            HttpMethod.GET,
+            "/internal/v1/wallet/transactions/adjustment/revision",
+            WalletCaller.SETTLEMENT));
+  }
+
+  private static Arguments route(HttpMethod method, String path, WalletCaller... allowed) {
+    return Arguments.of(new WalletRoute(method, path, Set.of(allowed)));
+  }
+
+  private record WalletRoute(HttpMethod method, String path, Set<WalletCaller> allowed) {}
+
   @RestController
   static class TestEndpoints {
     @GetMapping({
@@ -137,6 +195,24 @@ class WalletSecurityConfigTest {
       return "ok";
     }
 
+    @RequestMapping(
+        path = {
+          "/internal/v1/wallet/accounts",
+          "/internal/v1/wallet/accounts/{userId}/balance",
+          "/internal/v1/wallet/transactions/deposit",
+          "/internal/v1/wallet/transactions/withdraw",
+          "/internal/v1/wallet/transactions/debit",
+          "/internal/v1/wallet/transactions/debit/{betId}",
+          "/internal/v1/wallet/transactions/credit",
+          "/internal/v1/wallet/transactions/forfeit",
+          "/internal/v1/wallet/transactions/adjustment",
+          "/internal/v1/wallet/transactions/adjustment/{revisionId}"
+        },
+        method = {RequestMethod.GET, RequestMethod.POST})
+    String walletEndpoint() {
+      return "ok";
+    }
+
     @PostMapping("/actuator/health")
     String postHealth() {
       return "ok";


