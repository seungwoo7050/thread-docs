# 내부 호출 인증과 자격증명 분리

## `feat(admin): validate operator credentials`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminCredentials.java b/src/main/java/com/sportsbook/settlement/admin/AdminCredentials.java
new file mode 100644
index 0000000..c45f30d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminCredentials.java
@@ -0,0 +1,24 @@
+package com.sportsbook.settlement.admin;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("settlement.admin")
+public record AdminCredentials(String apiKey) {
+
+  public static final String CALLER = "admin-api";
+  public static final String SERVICE_HEADER = "X-Service-Name";
+  public static final String API_KEY_HEADER = "X-API-Key";
+  private static final int MINIMUM_SECRET_LENGTH = 32;
+
+  public AdminCredentials {
+    if (apiKey == null || apiKey.isBlank() || apiKey.length() < MINIMUM_SECRET_LENGTH) {
+      throw new IllegalArgumentException(
+          "SETTLEMENT_ADMIN_API_KEY must contain at least 32 characters");
+    }
+  }
+
+  @Override
+  public String toString() {
+    return "AdminCredentials[apiKey=<redacted>]";
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 5e2ccf7..05f8481 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -20,5 +20,7 @@ management:
         include: health,info,prometheus
 
 settlement:
+  admin:
+    api-key: ${SETTLEMENT_ADMIN_API_KEY:}
   wallet:
     api-key: ${SETTLEMENT_WALLET_API_KEY:}


## `feat(admin): authenticate internal requests`

diff --git a/src/main/java/com/sportsbook/settlement/admin/AdminAuthenticationFilter.java b/src/main/java/com/sportsbook/settlement/admin/AdminAuthenticationFilter.java
new file mode 100644
index 0000000..ae7a86d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/admin/AdminAuthenticationFilter.java
@@ -0,0 +1,63 @@
+package com.sportsbook.settlement.admin;
+
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.util.Collections;
+import java.util.List;
+import org.springframework.core.Ordered;
+import org.springframework.core.annotation.Order;
+import org.springframework.http.HttpStatus;
+import org.springframework.stereotype.Component;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+@Component
+@Order(Ordered.HIGHEST_PRECEDENCE + 10)
+public final class AdminAuthenticationFilter extends OncePerRequestFilter {
+
+  private static final String ADMIN_PREFIX = "/internal/admin";
+
+  private final byte[] expectedSecret;
+  private final AdminProblemWriter problems;
+
+  public AdminAuthenticationFilter(AdminCredentials credentials, AdminProblemWriter problems) {
+    this.expectedSecret = credentials.apiKey().getBytes(StandardCharsets.UTF_8);
+    this.problems = problems;
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    String path = request.getRequestURI();
+    return !path.equals(ADMIN_PREFIX) && !path.startsWith(ADMIN_PREFIX + "/");
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    List<String> callers = Collections.list(request.getHeaders(AdminCredentials.SERVICE_HEADER));
+    List<String> secrets = Collections.list(request.getHeaders(AdminCredentials.API_KEY_HEADER));
+    if (callers.isEmpty() || secrets.isEmpty()) {
+      problems.write(request, response, HttpStatus.UNAUTHORIZED, "Admin credentials are required");
+      return;
+    }
+    if (callers.size() != 1 || secrets.size() != 1) {
+      problems.write(request, response, HttpStatus.FORBIDDEN, "Admin credentials are invalid");
+      return;
+    }
+    String caller = callers.get(0);
+    String supplied = secrets.get(0);
+    boolean callerMatches = AdminCredentials.CALLER.equals(caller);
+    boolean secretMatches =
+        MessageDigest.isEqual(expectedSecret, supplied.getBytes(StandardCharsets.UTF_8));
+    if (!callerMatches || !secretMatches) {
+      problems.write(request, response, HttpStatus.FORBIDDEN, "Admin credentials are invalid");
+      return;
+    }
+    chain.doFilter(request, response);
+  }
+}


## `test(admin): verify internal authentication`

diff --git a/src/test/java/com/sportsbook/settlement/admin/AdminAuthenticationFilterTest.java b/src/test/java/com/sportsbook/settlement/admin/AdminAuthenticationFilterTest.java
new file mode 100644
index 0000000..e125f81
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/admin/AdminAuthenticationFilterTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.settlement.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import jakarta.servlet.FilterChain;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class AdminAuthenticationFilterTest {
+
+  private static final String SECRET = "abcdef0123456789abcdef0123456789";
+  private final FilterChain chain = mock(FilterChain.class);
+  private final AdminAuthenticationFilter filter =
+      new AdminAuthenticationFilter(
+          new AdminCredentials(SECRET),
+          new AdminProblemWriter(new ObjectMapper().findAndRegisterModules()));
+
+  @Test
+  void returnsUnauthorizedWhenEitherCredentialHeaderIsMissing() throws Exception {
+    MockHttpServletRequest request = adminRequest();
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, chain);
+
+    assertThat(response.getStatus()).isEqualTo(401);
+    assertThat(response.getContentAsString()).doesNotContain(SECRET);
+    verify(chain, never()).doFilter(request, response);
+  }
+
+  @Test
+  void returnsForbiddenForWrongCallerOrSecret() throws Exception {
+    assertForbidden("other-service", SECRET);
+    assertForbidden(AdminCredentials.CALLER, "x".repeat(SECRET.length()));
+  }
+
+  @Test
+  void returnsForbiddenForAmbiguousCredentialHeaders() throws Exception {
+    MockHttpServletRequest request = adminRequest();
+    request.addHeader(AdminCredentials.SERVICE_HEADER, AdminCredentials.CALLER);
+    request.addHeader(AdminCredentials.SERVICE_HEADER, AdminCredentials.CALLER);
+    request.addHeader(AdminCredentials.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, chain);
+
+    assertThat(response.getStatus()).isEqualTo(403);
+    verify(chain, never()).doFilter(request, response);
+  }
+
+  @Test
+  void permitsExactCredentialsAndLeavesOtherPathsUnfiltered() throws Exception {
+    MockHttpServletRequest admin = adminRequest();
+    admin.addHeader(AdminCredentials.SERVICE_HEADER, AdminCredentials.CALLER);
+    admin.addHeader(AdminCredentials.API_KEY_HEADER, SECRET);
+    MockHttpServletResponse adminResponse = new MockHttpServletResponse();
+    MockHttpServletRequest health = new MockHttpServletRequest("GET", "/actuator/health");
+    MockHttpServletResponse healthResponse = new MockHttpServletResponse();
+
+    filter.doFilter(admin, adminResponse, chain);
+    filter.doFilter(health, healthResponse, chain);
+
+    verify(chain).doFilter(admin, adminResponse);
+    verify(chain).doFilter(health, healthResponse);
+  }
+
+  private void assertForbidden(String caller, String secret) throws Exception {
+    MockHttpServletRequest request = adminRequest();
+    request.addHeader(AdminCredentials.SERVICE_HEADER, caller);
+    request.addHeader(AdminCredentials.API_KEY_HEADER, secret);
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, chain);
+
+    assertThat(response.getStatus()).isEqualTo(403);
+    assertThat(response.getContentAsString()).doesNotContain(SECRET, secret);
+    verify(chain, never()).doFilter(request, response);
+  }
+
+  private static MockHttpServletRequest adminRequest() {
+    return new MockHttpServletRequest("GET", "/internal/admin/revisions/one");
+  }
+}


## `feat(wallet): write exact authentication headers`

diff --git a/pom.xml b/pom.xml
index 50166a8..005253f 100644
--- a/pom.xml
+++ b/pom.xml
@@ -56,6 +56,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-actuator</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework</groupId>
+            <artifactId>spring-web</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.flywaydb</groupId>
             <artifactId>flyway-core</artifactId>
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletAuthenticationHeaders.java b/src/main/java/com/sportsbook/settlement/client/WalletAuthenticationHeaders.java
new file mode 100644
index 0000000..aa99409
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletAuthenticationHeaders.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.client;
+
+import org.springframework.http.HttpHeaders;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class WalletAuthenticationHeaders {
+
+  public static final String SERVICE_HEADER = "X-Internal-Service";
+  public static final String API_KEY_HEADER = "X-Internal-Api-Key";
+
+  private final WalletCredentials credentials;
+
+  public WalletAuthenticationHeaders(WalletCredentials credentials) {
+    this.credentials = credentials;
+  }
+
+  public void apply(HttpHeaders headers) {
+    headers.set(SERVICE_HEADER, WalletCredentials.CALLER);
+    headers.set(API_KEY_HEADER, credentials.apiKey());
+  }
+}


## `feat(wallet): require isolated settlement credentials`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletCredentials.java b/src/main/java/com/sportsbook/settlement/client/WalletCredentials.java
new file mode 100644
index 0000000..dddf759
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletCredentials.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.client;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("settlement.wallet")
+public record WalletCredentials(String apiKey) {
+
+  public static final String CALLER = "settlement-service";
+  private static final int MINIMUM_SECRET_LENGTH = 32;
+
+  public WalletCredentials {
+    if (apiKey == null || apiKey.isBlank() || apiKey.length() < MINIMUM_SECRET_LENGTH) {
+      throw new IllegalArgumentException(
+          "SETTLEMENT_WALLET_API_KEY must contain at least 32 characters");
+    }
+  }
+
+  @Override
+  public String toString() {
+    return "WalletCredentials[apiKey=<redacted>]";
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index a94dcf4..5e2ccf7 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -18,3 +18,7 @@ management:
     web:
       exposure:
         include: health,info,prometheus
+
+settlement:
+  wallet:
+    api-key: ${SETTLEMENT_WALLET_API_KEY:}


## `test(wallet): verify credential isolation and redaction`

diff --git a/src/test/java/com/sportsbook/settlement/client/WalletCredentialsTest.java b/src/test/java/com/sportsbook/settlement/client/WalletCredentialsTest.java
new file mode 100644
index 0000000..f39eb2a
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/client/WalletCredentialsTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.settlement.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+
+class WalletCredentialsTest {
+
+  private static final String SECRET = "0123456789abcdef0123456789abcdef";
+  private final ApplicationContextRunner runner =
+      new ApplicationContextRunner().withUserConfiguration(CredentialsConfiguration.class);
+
+  @Test
+  void failsStartupWhenSettlementCredentialIsMissingOrShort() {
+    runner.run(context -> assertThat(context).hasFailed());
+    runner
+        .withPropertyValues("settlement.wallet.api-key=too-short")
+        .run(context -> assertThat(context).hasFailed());
+  }
+
+  @Test
+  void exposesOnlyCallerIdentityAndRedactedCredentialText() {
+    runner
+        .withPropertyValues("settlement.wallet.api-key=" + SECRET)
+        .run(
+            context -> {
+              WalletCredentials credentials = context.getBean(WalletCredentials.class);
+              assertThat(WalletCredentials.CALLER).isEqualTo("settlement-service");
+              assertThat(credentials.apiKey()).isEqualTo(SECRET);
+              assertThat(credentials.toString()).doesNotContain(SECRET).contains("<redacted>");
+            });
+  }
+
+  @Test
+  void productionConfigurationBindsNoOtherServiceSecret() throws Exception {
+    String yaml = Files.readString(Path.of("src/main/resources/application.yml"));
+
+    assertThat(yaml).contains("SETTLEMENT_WALLET_API_KEY");
+    assertThat(yaml)
+        .doesNotContain("WALLET_PLATFORM_API_KEY", "WALLET_ADMIN_API_KEY", "WALLET_BETTING");
+  }
+
+  @EnableConfigurationProperties(WalletCredentials.class)
+  static class CredentialsConfiguration {}
+}
diff --git a/src/test/resources/application.properties b/src/test/resources/application.properties
index c4cdc06..c8112b7 100644
--- a/src/test/resources/application.properties
+++ b/src/test/resources/application.properties
@@ -1,3 +1,4 @@
 spring.kafka.listener.auto-startup=false
 spring.flyway.enabled=false
 spring.jpa.hibernate.ddl-auto=create-drop
+settlement.wallet.api-key=0123456789abcdef0123456789abcdef


## `fix(security): reject reused settlement credentials`

diff --git a/src/main/java/com/sportsbook/settlement/config/SettlementCredentialPolicy.java b/src/main/java/com/sportsbook/settlement/config/SettlementCredentialPolicy.java
new file mode 100644
index 0000000..08a87a7
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/config/SettlementCredentialPolicy.java
@@ -0,0 +1,21 @@
+package com.sportsbook.settlement.config;
+
+import com.sportsbook.settlement.admin.AdminCredentials;
+import com.sportsbook.settlement.client.WalletCredentials;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class SettlementCredentialPolicy {
+
+  public SettlementCredentialPolicy(
+      AdminCredentials adminCredentials, WalletCredentials walletCredentials) {
+    byte[] admin = adminCredentials.apiKey().getBytes(StandardCharsets.UTF_8);
+    byte[] wallet = walletCredentials.apiKey().getBytes(StandardCharsets.UTF_8);
+    if (MessageDigest.isEqual(admin, wallet)) {
+      throw new IllegalArgumentException(
+          "Settlement admin and Wallet credentials must be distinct");
+    }
+  }
+}


## `test(security): verify settlement credential isolation`

diff --git a/src/test/java/com/sportsbook/settlement/config/SettlementCredentialPolicyTest.java b/src/test/java/com/sportsbook/settlement/config/SettlementCredentialPolicyTest.java
new file mode 100644
index 0000000..70a4fc9
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/config/SettlementCredentialPolicyTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.settlement.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.settlement.admin.AdminCredentials;
+import com.sportsbook.settlement.client.WalletCredentials;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.context.annotation.Import;
+
+class SettlementCredentialPolicyTest {
+
+  private static final String ADMIN = "admin-0123456789abcdef0123456789ab";
+  private static final String WALLET = "wallet-0123456789abcdef0123456789a";
+  private static final String REUSED = "reused-0123456789abcdef0123456789";
+
+  private final ApplicationContextRunner runner =
+      new ApplicationContextRunner().withUserConfiguration(CredentialConfiguration.class);
+
+  @Test
+  void acceptsDistinctCredentialsWithoutExposingTheirValues() {
+    runner
+        .withPropertyValues(
+            "settlement.admin.api-key=" + ADMIN, "settlement.wallet.api-key=" + WALLET)
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              assertThat(context.getBean(AdminCredentials.class).toString())
+                  .doesNotContain(ADMIN)
+                  .contains("<redacted>");
+              assertThat(context.getBean(WalletCredentials.class).toString())
+                  .doesNotContain(WALLET)
+                  .contains("<redacted>");
+              assertThat(context.getBean(SettlementCredentialPolicy.class).toString())
+                  .doesNotContain(ADMIN, WALLET);
+            });
+  }
+
+  @Test
+  void rejectsASecretReusedAcrossAdminAndWalletDirections() {
+    runner
+        .withPropertyValues(
+            "settlement.admin.api-key=" + REUSED, "settlement.wallet.api-key=" + REUSED)
+        .run(
+            context -> {
+              assertThat(context).hasFailed();
+              assertThat(context.getStartupFailure())
+                  .hasRootCauseMessage("Settlement admin and Wallet credentials must be distinct");
+              assertThat(context.getStartupFailure().toString()).doesNotContain(REUSED);
+            });
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties({AdminCredentials.class, WalletCredentials.class})
+  @Import(SettlementCredentialPolicy.class)
+  static class CredentialConfiguration {}
+}
