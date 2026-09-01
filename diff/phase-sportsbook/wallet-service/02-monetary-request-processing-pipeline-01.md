# 금전 요청 처리 파이프라인

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


## `feat(api): parse wallet request identities`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java b/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java
new file mode 100644
index 0000000..4ffffdc
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java
@@ -0,0 +1,47 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.Enumeration;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Parses single-valued wallet request identity headers without normalization. */
+public final class WalletRequestHeaders {
+  public static final String IDEMPOTENCY_KEY = "Idempotency-Key";
+
+  private WalletRequestHeaders() {}
+
+  public static IdempotencyKey requireIdempotencyKey(HttpServletRequest request) {
+    Objects.requireNonNull(request, "request");
+    Enumeration<String> values = request.getHeaders(IDEMPOTENCY_KEY);
+    if (values == null || !values.hasMoreElements()) {
+      throw new IllegalArgumentException("Exactly one Idempotency-Key header is required");
+    }
+    String value = values.nextElement();
+    if (values.hasMoreElements()) {
+      throw new IllegalArgumentException("Exactly one Idempotency-Key header is required");
+    }
+    return IdempotencyKey.of(value);
+  }
+
+  public static IdempotencyKey requireCanonicalDebitKey(HttpServletRequest request) {
+    IdempotencyKey key = requireIdempotencyKey(request);
+    requireCanonicalDebitId(key.value());
+    return key;
+  }
+
+  public static UUID requireCanonicalDebitId(String raw) {
+    Objects.requireNonNull(raw, "raw");
+    UUID parsed;
+    try {
+      parsed = UUID.fromString(raw);
+    } catch (IllegalArgumentException invalid) {
+      throw new IllegalArgumentException("Debit identity must be a canonical UUID");
+    }
+    if (!parsed.toString().equals(raw)) {
+      throw new IllegalArgumentException("Debit identity must be a canonical UUID");
+    }
+    return parsed;
+  }
+}


## `feat(platform): expose deposit and withdrawal endpoints`

diff --git a/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java b/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java
new file mode 100644
index 0000000..3320843
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java
@@ -0,0 +1,42 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.DepositCommand;
+import com.sportsbook.wallet.service.command.WithdrawCommand;
+import com.sportsbook.wallet.web.dto.TransactionRequest;
+import com.sportsbook.wallet.web.dto.WalletOperationResponse;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes platform-owned external payment transfers. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions")
+public class PlatformTransactionController {
+  private final WalletService wallet;
+
+  public PlatformTransactionController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping("/deposit")
+  WalletOperationResponse deposit(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    return WalletOperationResponse.from(
+        wallet.deposit(new DepositCommand(body.userId(), body.amount(), key)));
+  }
+
+  @PostMapping("/withdraw")
+  WalletOperationResponse withdraw(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    return WalletOperationResponse.from(
+        wallet.withdraw(new WithdrawCommand(body.userId(), body.amount(), key)));
+  }
+}


## `feat(operation): fingerprint canonical wallet requests`

diff --git a/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java b/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java
new file mode 100644
index 0000000..619e36f
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java
@@ -0,0 +1,87 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Objects;
+import java.util.UUID;
+
+/** SHA-256 of a versioned binary representation of semantic request identity. */
+public record OperationFingerprint(String value) {
+  private static final int HASH_LENGTH = 64;
+  private static final int USER_TAG = 3;
+  private static final int AMOUNT_TAG = 4;
+  private static final int CURRENCY_TAG = 5;
+  private static final int REVISION_TAG = 6;
+  private static final int BET_TAG = 7;
+  private static final int REVISION_NUMBER_TAG = 8;
+  private static final int PREVIOUS_PAYOUT_TAG = 9;
+  private static final int NEW_PAYOUT_TAG = 10;
+
+  public OperationFingerprint {
+    Objects.requireNonNull(value, "value");
+    if (value.length() != HASH_LENGTH || !value.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("Operation fingerprint must be lower-case SHA-256 hex");
+    }
+  }
+
+  public static OperationFingerprint transfer(
+      WalletCaller caller, WalletOperationKind kind, UUID userId, Money amount) {
+    return digest(base(caller, kind, userId, amount).toByteArray());
+  }
+
+  public static OperationFingerprint adjustment(
+      WalletCaller caller,
+      UUID userId,
+      Money previousPayout,
+      Money newPayout,
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber) {
+    Objects.requireNonNull(previousPayout, "previousPayout");
+    Objects.requireNonNull(newPayout, "newPayout");
+    if (previousPayout.currency() != newPayout.currency()) {
+      throw new IllegalArgumentException("Adjustment payout currencies must match");
+    }
+    long delta = Math.subtractExact(newPayout.amount(), previousPayout.amount());
+    if (delta == Long.MIN_VALUE) {
+      throw new ArithmeticException("Adjustment delta is not representable");
+    }
+    CanonicalRequestEncoder encoded =
+        base(
+            caller,
+            WalletOperationKind.BET_ADJUSTMENT,
+            userId,
+            new Money(Math.abs(delta), previousPayout.currency()));
+    encoded
+        .uuid(REVISION_TAG, Objects.requireNonNull(revisionId, "revisionId"))
+        .uuid(BET_TAG, Objects.requireNonNull(betId, "betId"))
+        .number(REVISION_NUMBER_TAG, revisionNumber)
+        .number(PREVIOUS_PAYOUT_TAG, previousPayout.amount())
+        .number(NEW_PAYOUT_TAG, newPayout.amount());
+    return digest(encoded.toByteArray());
+  }
+
+  private static CanonicalRequestEncoder base(
+      WalletCaller caller, WalletOperationKind kind, UUID userId, Money amount) {
+    Objects.requireNonNull(amount, "amount");
+    return new CanonicalRequestEncoder()
+        .text(1, Objects.requireNonNull(caller, "caller").name())
+        .text(2, Objects.requireNonNull(kind, "kind").name())
+        .uuid(USER_TAG, Objects.requireNonNull(userId, "userId"))
+        .number(AMOUNT_TAG, amount.amount())
+        .text(CURRENCY_TAG, amount.currency().name());
+  }
+
+  private static OperationFingerprint digest(byte[] canonical) {
+    try {
+      byte[] hash = MessageDigest.getInstance("SHA-256").digest(canonical);
+      return new OperationFingerprint(HexFormat.of().formatHex(hash));
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("JVM lacks SHA-256", impossible);
+    }
+  }
+}


## `feat(locking): acquire transaction-scoped idempotency locks`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java b/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java
new file mode 100644
index 0000000..4999660
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java
@@ -0,0 +1,37 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.Objects;
+import org.springframework.dao.DataAccessException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+
+/** Serializes first writers for one full idempotency key until their transaction completes. */
+@Component
+public class IdempotencyKeyLock {
+
+  private static final String LOCK_NAMESPACE = "wallet:idempotency:";
+  private static final String LOCK_SQL = "select pg_advisory_xact_lock(hashtextextended(?, 0))";
+
+  private final JdbcTemplate jdbc;
+
+  public IdempotencyKeyLock(JdbcTemplate jdbc) {
+    this.jdbc = Objects.requireNonNull(jdbc, "jdbc");
+  }
+
+  public void acquire(IdempotencyKey key) {
+    Objects.requireNonNull(key, "key");
+    if (!TransactionSynchronizationManager.isActualTransactionActive()) {
+      throw new IllegalStateException("Idempotency advisory lock requires an active transaction");
+    }
+    try {
+      jdbc.query(
+          LOCK_SQL,
+          statement -> statement.setString(1, LOCK_NAMESPACE + key.value()),
+          resultSet -> null);
+    } catch (DataAccessException failure) {
+      throw PostgresFailureTranslator.translate(key, failure);
+    }
+  }
+}


