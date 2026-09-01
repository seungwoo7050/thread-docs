## `feat(routing): require betting credentials`

diff --git a/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java b/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java
index eccb460..8835545 100644
--- a/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java
+++ b/src/main/java/com/sportsbook/gateway/routing/BettingDownstreamProperties.java
@@ -2,10 +2,18 @@ package com.sportsbook.gateway.routing;
 
 import java.net.URI;
 import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.boot.context.properties.bind.ConstructorBinding;
 
 @ConfigurationProperties(prefix = "gateway.downstream")
-public record BettingDownstreamProperties(URI bettingUri) {
+public record BettingDownstreamProperties(URI bettingUri, String bettingApiKey) {
 
+  private static final int MINIMUM_API_KEY_LENGTH = 32;
+
+  public BettingDownstreamProperties(URI bettingUri) {
+    this(bettingUri, null);
+  }
+
+  @ConstructorBinding
   public BettingDownstreamProperties {
     String scheme = bettingUri == null ? null : bettingUri.getScheme();
     String path = bettingUri == null ? null : bettingUri.getRawPath();
@@ -21,4 +29,19 @@ public record BettingDownstreamProperties(URI bettingUri) {
       throw new IllegalArgumentException("gateway.downstream.betting-uri must be an HTTP base URI");
     }
   }
+
+  String requiredApiKey() {
+    if (bettingApiKey == null
+        || bettingApiKey.isBlank()
+        || bettingApiKey.length() < MINIMUM_API_KEY_LENGTH) {
+      throw new IllegalArgumentException(
+          "GATEWAY_BETTING_API_KEY must contain at least 32 characters");
+    }
+    return bettingApiKey;
+  }
+
+  @Override
+  public String toString() {
+    return "BettingDownstreamProperties[bettingUri=" + bettingUri + ", bettingApiKey=<redacted>]";
+  }
 }
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 04f69e7..53f7c03 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -65,6 +65,7 @@ management:
 gateway:
   downstream:
     betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}
+    betting-api-key: ${GATEWAY_BETTING_API_KEY:}
     odds-feed-uri: ${GATEWAY_ODDS_FEED_URI:http://localhost:8085}
     wallet:
       uri: ${GATEWAY_WALLET_URI:http://localhost:8081}


## `test(routing): reject invalid betting credentials`

diff --git a/src/test/java/com/sportsbook/gateway/routing/BettingDownstreamPropertiesTest.java b/src/test/java/com/sportsbook/gateway/routing/BettingDownstreamPropertiesTest.java
new file mode 100644
index 0000000..0c94057
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/BettingDownstreamPropertiesTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.gateway.routing;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.AutoConfigurations;
+import org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+class BettingDownstreamPropertiesTest {
+
+  private static final String KEY = "01234567890123456789012345678901";
+
+  @Test
+  void acceptsARequiredCredentialWithoutExposingIt() {
+    runner("gateway.downstream.betting-api-key=" + KEY)
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              assertThat(context.getBean("bettingApiKey")).isEqualTo(KEY);
+              assertThat(context.getBean(BettingDownstreamProperties.class).toString())
+                  .doesNotContain(KEY)
+                  .contains("<redacted>");
+            });
+  }
+
+  @Test
+  void rejectsMissingBlankAndShortCredentialsWhenRequired() {
+    assertRejected(runner());
+    assertRejected(runner("gateway.downstream.betting-api-key= "));
+    assertRejected(runner("gateway.downstream.betting-api-key=too-short"));
+  }
+
+  private static ApplicationContextRunner runner(String... properties) {
+    return new ApplicationContextRunner()
+        .withConfiguration(AutoConfigurations.of(ConfigurationPropertiesAutoConfiguration.class))
+        .withUserConfiguration(PropertyConfiguration.class)
+        .withPropertyValues("gateway.downstream.betting-uri=http://betting.internal")
+        .withPropertyValues(properties);
+  }
+
+  private static void assertRejected(ApplicationContextRunner runner) {
+    runner.run(
+        context -> {
+          assertThat(context).hasFailed();
+          assertThat(context.getStartupFailure())
+              .hasRootCauseInstanceOf(IllegalArgumentException.class)
+              .hasRootCauseMessage("GATEWAY_BETTING_API_KEY must contain at least 32 characters");
+        });
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(BettingDownstreamProperties.class)
+  static class PropertyConfiguration {
+    @Bean
+    String bettingApiKey(BettingDownstreamProperties properties) {
+      return properties.requiredApiKey();
+    }
+  }
+}


## `feat(routing): authenticate betting requests`

diff --git a/src/main/java/com/sportsbook/gateway/routing/BettingRequestAuthentication.java b/src/main/java/com/sportsbook/gateway/routing/BettingRequestAuthentication.java
new file mode 100644
index 0000000..7f99720
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/BettingRequestAuthentication.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.routing;
+
+import com.sportsbook.gateway.security.GatewayHeaders;
+import org.springframework.web.servlet.function.ServerRequest;
+
+final class BettingRequestAuthentication {
+
+  private final String apiKey;
+
+  BettingRequestAuthentication(BettingDownstreamProperties properties) {
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
diff --git a/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java b/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java
index 177522e..d655e88 100644
--- a/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java
+++ b/src/main/java/com/sportsbook/gateway/routing/BettingRoutes.java
@@ -22,6 +22,7 @@ public class BettingRoutes {
   private final BettingDownstreamProperties downstream;
   private final DownstreamRequestSanitizer sanitizer;
   private final IdentityForwarding identity;
+  private final BettingRequestAuthentication authentication;
   private final TraceForwarding trace;
   private final DownstreamFailureBoundary failures;
 
@@ -34,6 +35,7 @@ public class BettingRoutes {
     this.downstream = downstream;
     this.sanitizer = sanitizer;
     this.identity = identity;
+    this.authentication = new BettingRequestAuthentication(downstream);
     this.trace = trace;
     this.failures = failures;
   }
@@ -46,6 +48,7 @@ public class BettingRoutes {
             http(downstream.bettingUri().toString()))
         .before(sanitizer::apply)
         .before(identity::apply)
+        .before(authentication::apply)
         .before(trace::apply)
         .before(this::rewrite)
         .filter(failures)


## `test(routing): isolate betting credentials`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
index 655b214..4f18a60 100644
--- a/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayRoutingIntegrationTest.java
@@ -51,6 +51,7 @@ import org.springframework.test.context.DynamicPropertySource;
 @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
 class GatewayRoutingIntegrationTest {
 
+  static final String BETTING_KEY = "fixture-betting-key-32-characters-long";
   static final String WALLET_KEY = "fixture-wallet-key-32-characters-long";
   static final WireMockServer DOWNSTREAM = startDownstream();
   private static final KeyPair KEYS = keyPair();
@@ -62,6 +63,7 @@ class GatewayRoutingIntegrationTest {
     registry.add("gateway.security.jwt.public-key", GatewayRoutingIntegrationTest::publicKey);
     registry.add("gateway.ratelimit.enabled", () -> "false");
     registry.add("gateway.downstream.betting-uri", DOWNSTREAM::baseUrl);
+    registry.add("gateway.downstream.betting-api-key", () -> BETTING_KEY);
     registry.add("gateway.downstream.wallet.uri", DOWNSTREAM::baseUrl);
     registry.add("gateway.downstream.wallet.api-key", () -> WALLET_KEY);
     registry.add("gateway.downstream.odds-feed-uri", DOWNSTREAM::baseUrl);
@@ -96,9 +98,9 @@ class GatewayRoutingIntegrationTest {
             .withHeader("X-User-Id", equalTo(subject.toString()))
             .withHeader("X-User-Roles", equalTo("USER"))
             .withHeader("Idempotency-Key", equalTo("fixture-idempotency-key"))
+            .withHeader("X-Internal-Service", equalTo("gateway"))
+            .withHeader("X-Internal-Api-Key", equalTo(BETTING_KEY))
             .withoutHeader(HttpHeaders.AUTHORIZATION)
-            .withoutHeader("X-Internal-Service")
-            .withoutHeader("X-Internal-Api-Key")
             .withRequestBody(equalTo(body)));
   }
 
@@ -315,9 +317,9 @@ class GatewayRoutingIntegrationTest {
               .withHeader("X-User-Roles", equalTo("USER"))
               .withHeader("Idempotency-Key", equalTo("fixture-" + index))
               .withHeader("traceparent", equalTo(traceparent(index)))
+              .withHeader("X-Internal-Service", equalTo("gateway"))
+              .withHeader("X-Internal-Api-Key", equalTo(BETTING_KEY))
               .withoutHeader(HttpHeaders.AUTHORIZATION)
-              .withoutHeader("X-Internal-Service")
-              .withoutHeader("X-Internal-Api-Key")
               .withRequestBody(equalTo(requestBody(index))));
     }
   }
diff --git a/src/test/resources/application.properties b/src/test/resources/application.properties
index baebbb8..e2dd548 100644
--- a/src/test/resources/application.properties
+++ b/src/test/resources/application.properties
@@ -1 +1,2 @@
 spring.kafka.listener.auto-startup=false
+gateway.downstream.betting-api-key=fixture-betting-key-32-characters-long


## `feat(routing): require distinct downstream credentials`

diff --git a/src/main/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicy.java b/src/main/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicy.java
new file mode 100644
index 0000000..219a240
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicy.java
@@ -0,0 +1,21 @@
+package com.sportsbook.gateway.routing;
+
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Configuration;
+
+@Configuration(proxyBeanMethods = false)
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+@EnableConfigurationProperties({
+  BettingDownstreamProperties.class,
+  WalletDownstreamProperties.class
+})
+class GatewayDownstreamCredentialPolicy {
+
+  GatewayDownstreamCredentialPolicy(
+      BettingDownstreamProperties betting, WalletDownstreamProperties wallet) {
+    if (betting.requiredApiKey().equals(wallet.requiredApiKey())) {
+      throw new IllegalArgumentException("Gateway downstream API keys must be distinct");
+    }
+  }
+}


## `test(routing): reject reused downstream credentials`

diff --git a/src/test/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicyTest.java b/src/test/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicyTest.java
new file mode 100644
index 0000000..8682de9
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/routing/GatewayDownstreamCredentialPolicyTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.gateway.routing;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.AutoConfigurations;
+import org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration;
+import org.springframework.boot.test.context.runner.WebApplicationContextRunner;
+
+class GatewayDownstreamCredentialPolicyTest {
+
+  private static final String BETTING_KEY = "betting-downstream-credential-0001";
+  private static final String WALLET_KEY = "wallet-downstream-credential-00002";
+  private static final String SHARED_KEY = "shared-downstream-credential-00002";
+
+  @Test
+  void acceptsDistinctCredentialsWithoutRenderingSecrets() {
+    runner(BETTING_KEY, WALLET_KEY)
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              assertThat(context.getBean(BettingDownstreamProperties.class).toString())
+                  .doesNotContain(BETTING_KEY, WALLET_KEY)
+                  .contains("<redacted>");
+              assertThat(context.getBean(WalletDownstreamProperties.class).toString())
+                  .doesNotContain(BETTING_KEY, WALLET_KEY)
+                  .contains("<redacted>");
+            });
+  }
+
+  @Test
+  void rejectsAReusedCredentialWithoutRenderingIt() {
+    runner(SHARED_KEY, SHARED_KEY)
+        .run(
+            context -> {
+              assertThat(context).hasFailed();
+              Throwable root = rootCause(context.getStartupFailure());
+              assertThat(root)
+                  .isInstanceOf(IllegalArgumentException.class)
+                  .hasMessage("Gateway downstream API keys must be distinct");
+              assertThat(root.getMessage()).doesNotContain(SHARED_KEY);
+            });
+  }
+
+  private static WebApplicationContextRunner runner(String bettingKey, String walletKey) {
+    return new WebApplicationContextRunner()
+        .withConfiguration(AutoConfigurations.of(ConfigurationPropertiesAutoConfiguration.class))
+        .withUserConfiguration(GatewayDownstreamCredentialPolicy.class)
+        .withPropertyValues(
+            "gateway.downstream.betting-uri=http://betting.internal",
+            "gateway.downstream.betting-api-key=" + bettingKey,
+            "gateway.downstream.wallet.uri=http://wallet.internal",
+            "gateway.downstream.wallet.api-key=" + walletKey);
+  }
+
+  private static Throwable rootCause(Throwable failure) {
+    Throwable root = failure;
+    while (root.getCause() != null) {
+      root = root.getCause();
+    }
+    return root;
+  }
+}
