# 하위 서비스 자격 증명과 RestClient 격리

## `feat(client): validate isolated downstream credentials`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java b/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java
new file mode 100644
index 0000000..5d33f10
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java
@@ -0,0 +1,33 @@
+package com.sportsbook.admin.client;
+
+import jakarta.validation.constraints.NotBlank;
+import jakarta.validation.constraints.Size;
+import java.util.HashSet;
+import java.util.Objects;
+import java.util.stream.Stream;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.validation.annotation.Validated;
+
+@Validated
+@ConfigurationProperties("admin.downstream.credentials")
+public record DownstreamCredentials(
+    @NotBlank @Size(min = 32) String walletApiKey,
+    @NotBlank @Size(min = 32) String riskApiKey,
+    @NotBlank @Size(min = 32) String oddsFeedApiKey,
+    @NotBlank @Size(min = 32) String settlementApiKey) {
+
+  public DownstreamCredentials {
+    var configured =
+        Stream.of(walletApiKey, riskApiKey, oddsFeedApiKey, settlementApiKey)
+            .filter(Objects::nonNull)
+            .toList();
+    if (new HashSet<>(configured).size() != configured.size()) {
+      throw new IllegalArgumentException("Admin downstream API keys must be distinct");
+    }
+  }
+
+  @Override
+  public String toString() {
+    return "DownstreamCredentials[REDACTED]";
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 7d15dcc..3ed9980 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -61,3 +61,9 @@ admin:
       issuer: ${ADMIN_JWT_ISSUER:}
     ip-allowlist: ${ADMIN_IP_ALLOWLIST:127.0.0.1/32,::1/128}
     trusted-proxy-cidrs: ${ADMIN_TRUSTED_PROXY_CIDRS:}
+  downstream:
+    credentials:
+      wallet-api-key: ${ADMIN_WALLET_API_KEY:}
+      risk-api-key: ${ADMIN_RISK_API_KEY:}
+      odds-feed-api-key: ${ADMIN_ODDS_FEED_API_KEY:}
+      settlement-api-key: ${ADMIN_SETTLEMENT_API_KEY:}


## `test(client): reject missing and reused credentials`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamCredentialsTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamCredentialsTest.java
new file mode 100644
index 0000000..d4f57f0
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamCredentialsTest.java
@@ -0,0 +1,73 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+
+class DownstreamCredentialsTest {
+
+  private static final String WALLET = "wallet-admin-test-key-000000000001";
+  private static final String RISK = "risk-admin-test-key-00000000000002";
+  private static final String ODDS = "odds-admin-test-key-00000000000003";
+  private static final String SETTLEMENT = "settlement-admin-test-key-000000004";
+
+  private final ApplicationContextRunner contextRunner =
+      new ApplicationContextRunner().withUserConfiguration(CredentialsConfiguration.class);
+
+  @Test
+  void bindsFourDistinctLongCredentialsWithoutRenderingThem() {
+    contextRunner
+        .withPropertyValues(validProperties())
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              DownstreamCredentials credentials = context.getBean(DownstreamCredentials.class);
+              assertThat(credentials.walletApiKey()).isEqualTo(WALLET);
+              assertThat(credentials.riskApiKey()).isEqualTo(RISK);
+              assertThat(credentials.oddsFeedApiKey()).isEqualTo(ODDS);
+              assertThat(credentials.settlementApiKey()).isEqualTo(SETTLEMENT);
+              assertThat(credentials.toString())
+                  .isEqualTo("DownstreamCredentials[REDACTED]")
+                  .doesNotContain(WALLET, RISK, ODDS, SETTLEMENT);
+            });
+  }
+
+  @Test
+  void rejectsMissingAndShortCredentials() {
+    contextRunner.run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues(
+            "admin.downstream.credentials.wallet-api-key=short",
+            "admin.downstream.credentials.risk-api-key=" + RISK,
+            "admin.downstream.credentials.odds-feed-api-key=" + ODDS,
+            "admin.downstream.credentials.settlement-api-key=" + SETTLEMENT)
+        .run(context -> assertThat(context).hasFailed());
+  }
+
+  @Test
+  void rejectsCredentialReuseAcrossProviders() {
+    contextRunner
+        .withPropertyValues(
+            "admin.downstream.credentials.wallet-api-key=" + WALLET,
+            "admin.downstream.credentials.risk-api-key=" + WALLET,
+            "admin.downstream.credentials.odds-feed-api-key=" + ODDS,
+            "admin.downstream.credentials.settlement-api-key=" + SETTLEMENT)
+        .run(context -> assertThat(context).hasFailed());
+  }
+
+  private static String[] validProperties() {
+    return new String[] {
+      "admin.downstream.credentials.wallet-api-key=" + WALLET,
+      "admin.downstream.credentials.risk-api-key=" + RISK,
+      "admin.downstream.credentials.odds-feed-api-key=" + ODDS,
+      "admin.downstream.credentials.settlement-api-key=" + SETTLEMENT
+    };
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(DownstreamCredentials.class)
+  static class CredentialsConfiguration {}
+}


## `feat(client): validate downstream HTTP origins`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamProperties.java b/src/main/java/com/sportsbook/admin/client/DownstreamProperties.java
new file mode 100644
index 0000000..b79b08a
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamProperties.java
@@ -0,0 +1,45 @@
+package com.sportsbook.admin.client;
+
+import jakarta.validation.constraints.NotNull;
+import java.net.URI;
+import java.time.Duration;
+import java.util.stream.Stream;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.validation.annotation.Validated;
+
+@Validated
+@ConfigurationProperties("admin.downstream")
+public record DownstreamProperties(
+    @NotNull URI walletBaseUrl,
+    @NotNull URI riskBaseUrl,
+    @NotNull URI oddsFeedBaseUrl,
+    @NotNull URI settlementBaseUrl,
+    @NotNull Duration connectTimeout,
+    @NotNull Duration readTimeout) {
+
+  public DownstreamProperties {
+    Stream.of(walletBaseUrl, riskBaseUrl, oddsFeedBaseUrl, settlementBaseUrl)
+        .filter(uri -> uri != null)
+        .forEach(DownstreamProperties::requireHttpOrigin);
+    requirePositive(connectTimeout);
+    requirePositive(readTimeout);
+  }
+
+  private static void requireHttpOrigin(URI uri) {
+    boolean supportedScheme = "http".equals(uri.getScheme()) || "https".equals(uri.getScheme());
+    if (!uri.isAbsolute()
+        || !supportedScheme
+        || uri.getHost() == null
+        || uri.getUserInfo() != null
+        || uri.getQuery() != null
+        || uri.getFragment() != null) {
+      throw new IllegalArgumentException("Downstream base URLs must be HTTP origins");
+    }
+  }
+
+  private static void requirePositive(Duration duration) {
+    if (duration != null && (duration.isZero() || duration.isNegative())) {
+      throw new IllegalArgumentException("Downstream timeouts must be positive");
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 3ed9980..b7eec1e 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -62,6 +62,12 @@ admin:
     ip-allowlist: ${ADMIN_IP_ALLOWLIST:127.0.0.1/32,::1/128}
     trusted-proxy-cidrs: ${ADMIN_TRUSTED_PROXY_CIDRS:}
   downstream:
+    wallet-base-url: ${ADMIN_WALLET_BASE_URL:http://localhost:8081}
+    risk-base-url: ${ADMIN_RISK_BASE_URL:http://localhost:8083}
+    odds-feed-base-url: ${ADMIN_ODDS_FEED_BASE_URL:http://localhost:8085}
+    settlement-base-url: ${ADMIN_SETTLEMENT_BASE_URL:http://localhost:8084}
+    connect-timeout: ${ADMIN_DOWNSTREAM_CONNECT_TIMEOUT:200ms}
+    read-timeout: ${ADMIN_DOWNSTREAM_READ_TIMEOUT:2s}
     credentials:
       wallet-api-key: ${ADMIN_WALLET_API_KEY:}
       risk-api-key: ${ADMIN_RISK_API_KEY:}


## `test(client): reject invalid downstream origins`

diff --git a/src/test/java/com/sportsbook/admin/client/DownstreamPropertiesTest.java b/src/test/java/com/sportsbook/admin/client/DownstreamPropertiesTest.java
new file mode 100644
index 0000000..41462ed
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/DownstreamPropertiesTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+
+class DownstreamPropertiesTest {
+
+  private final ApplicationContextRunner contextRunner =
+      new ApplicationContextRunner().withUserConfiguration(HttpOriginsConfiguration.class);
+
+  @Test
+  void bindsFourHttpOriginsAndPositiveTimeouts() {
+    contextRunner
+        .withPropertyValues(validProperties())
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              DownstreamProperties properties = context.getBean(DownstreamProperties.class);
+              assertThat(properties.walletBaseUrl().toString()).isEqualTo("http://wallet:8081");
+              assertThat(properties.riskBaseUrl().toString()).isEqualTo("http://risk:8083");
+              assertThat(properties.connectTimeout()).isEqualTo(Duration.ofMillis(200));
+              assertThat(properties.readTimeout()).isEqualTo(Duration.ofSeconds(2));
+            });
+  }
+
+  @Test
+  void rejectsMissingAndNonHttpOrigins() {
+    contextRunner.run(context -> assertThat(context).hasFailed());
+    String[] invalid = validProperties();
+    invalid[0] = "admin.downstream.wallet-base-url=ftp://wallet:21";
+    contextRunner.withPropertyValues(invalid).run(context -> assertThat(context).hasFailed());
+  }
+
+  @Test
+  void rejectsZeroOrNegativeTimeouts() {
+    String[] invalid = validProperties();
+    invalid[4] = "admin.downstream.connect-timeout=0ms";
+    contextRunner.withPropertyValues(invalid).run(context -> assertThat(context).hasFailed());
+  }
+
+  private static String[] validProperties() {
+    return new String[] {
+      "admin.downstream.wallet-base-url=http://wallet:8081",
+      "admin.downstream.risk-base-url=http://risk:8083",
+      "admin.downstream.odds-feed-base-url=http://odds:8085",
+      "admin.downstream.settlement-base-url=http://settlement:8084",
+      "admin.downstream.connect-timeout=200ms",
+      "admin.downstream.read-timeout=2s"
+    };
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(DownstreamProperties.class)
+  static class HttpOriginsConfiguration {}
+}


## `feat(client): isolate the Wallet RestClient`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
new file mode 100644
index 0000000..0e1b3d8
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
@@ -0,0 +1,38 @@
+package com.sportsbook.admin.client;
+
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.client.SimpleClientHttpRequestFactory;
+import org.springframework.web.client.RestClient;
+
+@Configuration(proxyBeanMethods = false)
+class DownstreamClientConfiguration {
+
+  @Bean
+  @Qualifier("walletRestClient")
+  RestClient walletRestClient(
+      RestClient.Builder builder,
+      DownstreamProperties properties,
+      DownstreamCredentials credentials) {
+    return internalClient(
+        builder, properties.walletBaseUrl().toString(), credentials.walletApiKey(), properties);
+  }
+
+  private static RestClient internalClient(
+      RestClient.Builder builder,
+      String baseUrl,
+      String apiKey,
+      DownstreamProperties properties) {
+    SimpleClientHttpRequestFactory requests = new SimpleClientHttpRequestFactory();
+    requests.setConnectTimeout(properties.connectTimeout());
+    requests.setReadTimeout(properties.readTimeout());
+    return builder
+        .clone()
+        .baseUrl(baseUrl)
+        .requestFactory(requests)
+        .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
+        .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, apiKey)
+        .build();
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java b/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java
new file mode 100644
index 0000000..02cdbfb
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamHeaders.java
@@ -0,0 +1,12 @@
+package com.sportsbook.admin.client;
+
+public final class DownstreamHeaders {
+
+  public static final String INTERNAL_SERVICE = "X-Internal-Service";
+  public static final String INTERNAL_API_KEY = "X-Internal-Api-Key";
+  public static final String SERVICE_NAME = "X-Service-Name";
+  public static final String API_KEY = "X-API-Key";
+  public static final String ADMIN_API = "admin-api";
+
+  private DownstreamHeaders() {}
+}


## `feat(client): isolate the Risk RestClient`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
index 0e1b3d8..15c5697 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
@@ -19,6 +19,16 @@ class DownstreamClientConfiguration {
         builder, properties.walletBaseUrl().toString(), credentials.walletApiKey(), properties);
   }
 
+  @Bean
+  @Qualifier("riskRestClient")
+  RestClient riskRestClient(
+      RestClient.Builder builder,
+      DownstreamProperties properties,
+      DownstreamCredentials credentials) {
+    return internalClient(
+        builder, properties.riskBaseUrl().toString(), credentials.riskApiKey(), properties);
+  }
+
   private static RestClient internalClient(
       RestClient.Builder builder,
       String baseUrl,


## `feat(client): isolate the Odds RestClient`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
index 15c5697..228329d 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
@@ -29,6 +29,16 @@ class DownstreamClientConfiguration {
         builder, properties.riskBaseUrl().toString(), credentials.riskApiKey(), properties);
   }
 
+  @Bean
+  @Qualifier("oddsRestClient")
+  RestClient oddsRestClient(
+      RestClient.Builder builder,
+      DownstreamProperties properties,
+      DownstreamCredentials credentials) {
+    return internalClient(
+        builder, properties.oddsFeedBaseUrl().toString(), credentials.oddsFeedApiKey(), properties);
+  }
+
   private static RestClient internalClient(
       RestClient.Builder builder,
       String baseUrl,


## `feat(client): isolate the Settlement RestClient`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
index 228329d..05c0e28 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamClientConfiguration.java
@@ -39,20 +39,36 @@ class DownstreamClientConfiguration {
         builder, properties.oddsFeedBaseUrl().toString(), credentials.oddsFeedApiKey(), properties);
   }
 
-  private static RestClient internalClient(
+  @Bean
+  @Qualifier("settlementRestClient")
+  RestClient settlementRestClient(
       RestClient.Builder builder,
-      String baseUrl,
-      String apiKey,
-      DownstreamProperties properties) {
-    SimpleClientHttpRequestFactory requests = new SimpleClientHttpRequestFactory();
-    requests.setConnectTimeout(properties.connectTimeout());
-    requests.setReadTimeout(properties.readTimeout());
+      DownstreamProperties properties,
+      DownstreamCredentials credentials) {
+    return builder
+        .clone()
+        .baseUrl(properties.settlementBaseUrl().toString())
+        .requestFactory(requestFactory(properties))
+        .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API)
+        .defaultHeader(DownstreamHeaders.API_KEY, credentials.settlementApiKey())
+        .build();
+  }
+
+  private static RestClient internalClient(
+      RestClient.Builder builder, String baseUrl, String apiKey, DownstreamProperties properties) {
     return builder
         .clone()
         .baseUrl(baseUrl)
-        .requestFactory(requests)
+        .requestFactory(requestFactory(properties))
         .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
         .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, apiKey)
         .build();
   }
+
+  private static SimpleClientHttpRequestFactory requestFactory(DownstreamProperties properties) {
+    SimpleClientHttpRequestFactory requests = new SimpleClientHttpRequestFactory();
+    requests.setConnectTimeout(properties.connectTimeout());
+    requests.setReadTimeout(properties.readTimeout());
+    return requests;
+  }
 }


## `test(client): provide cross-client fixture`

diff --git a/src/test/java/com/sportsbook/admin/client/CrossClientIntegrationFixture.java b/src/test/java/com/sportsbook/admin/client/CrossClientIntegrationFixture.java
new file mode 100644
index 0000000..2f0ef0f
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/CrossClientIntegrationFixture.java
@@ -0,0 +1,69 @@
+package com.sportsbook.admin.client;
+
+import com.sun.net.httpserver.HttpServer;
+import java.io.IOException;
+import java.net.InetSocketAddress;
+import java.net.URI;
+import java.util.List;
+import java.util.Locale;
+import java.util.Map;
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.stream.Collectors;
+import org.springframework.context.annotation.AnnotationConfigApplicationContext;
+import org.springframework.web.client.RestClient;
+
+final class CrossClientIntegrationFixture implements AutoCloseable {
+
+  private final Map<String, Map<String, List<String>>> captured = new ConcurrentHashMap<>();
+  private final HttpServer server;
+  private final AnnotationConfigApplicationContext context;
+
+  CrossClientIntegrationFixture() throws IOException {
+    server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
+    server.createContext(
+        "/",
+        exchange -> {
+          captured.put(exchange.getRequestURI().getPath(), lowerCase(exchange.getRequestHeaders()));
+          exchange.sendResponseHeaders(204, -1);
+          exchange.close();
+        });
+    server.start();
+    context = context();
+  }
+
+  void invoke(String beanName, String path) {
+    context.getBean(beanName, RestClient.class).get().uri(path).retrieve().toBodilessEntity();
+  }
+
+  Map<String, List<String>> headers(String path) {
+    return captured.get(path);
+  }
+
+  @Override
+  public void close() {
+    context.close();
+    server.stop(0);
+  }
+
+  private AnnotationConfigApplicationContext context() {
+    URI origin = URI.create("http://127.0.0.1:" + server.getAddress().getPort());
+    DownstreamProperties defaults = ClientIsolationFixture.properties();
+    DownstreamProperties properties =
+        new DownstreamProperties(
+            origin, origin, origin, origin, defaults.connectTimeout(), defaults.readTimeout());
+    AnnotationConfigApplicationContext created = new AnnotationConfigApplicationContext();
+    created.registerBean(RestClient.Builder.class, () -> RestClient.builder());
+    created.registerBean(DownstreamProperties.class, () -> properties);
+    created.registerBean(DownstreamCredentials.class, ClientIsolationFixture::credentials);
+    created.register(DownstreamClientConfiguration.class);
+    created.refresh();
+    return created;
+  }
+
+  private static Map<String, List<String>> lowerCase(Map<String, List<String>> headers) {
+    return headers.entrySet().stream()
+        .collect(
+            Collectors.toUnmodifiableMap(
+                entry -> entry.getKey().toLowerCase(Locale.ROOT), Map.Entry::getValue));
+  }
+}


