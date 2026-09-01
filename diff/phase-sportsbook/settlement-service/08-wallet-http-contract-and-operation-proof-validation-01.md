# Wallet HTTP 계약과 작업 증거 검증

## `feat(wallet): constrain settlement credit reasons`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java b/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java
new file mode 100644
index 0000000..ef4d579
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java
@@ -0,0 +1,23 @@
+package com.sportsbook.settlement.client;
+
+public enum WalletCreditPurpose {
+  WHOLE_SLIP_VOID("USER_LOCKED", "VOID"),
+  RETURNED_STAKE("USER_LOCKED", "REFUND"),
+  PROFIT_PAYOUT("HOUSE_POOL", "PAYOUT");
+
+  private final String source;
+  private final String reason;
+
+  WalletCreditPurpose(String source, String reason) {
+    this.source = source;
+    this.reason = reason;
+  }
+
+  public String source() {
+    return source;
+  }
+
+  public String reason() {
+    return reason;
+  }
+}


## `feat(wallet): submit authorized credit movements`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
new file mode 100644
index 0000000..f86ecbf
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -0,0 +1,48 @@
+package com.sportsbook.settlement.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+import com.sportsbook.protocol.value.Money;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+
+@Component
+public class WalletClient {
+
+  static final String CREDIT_PATH = "/internal/v1/wallet/transactions/credit";
+  static final String IDEMPOTENCY_HEADER = "Idempotency-Key";
+
+  private final RestClient http;
+
+  public WalletClient(
+      RestClient.Builder builder,
+      WalletEndpointProperties endpoint,
+      WalletAuthenticationHeaders authentication) {
+    this.http =
+        builder
+            .clone()
+            .baseUrl(endpoint.baseUrl().toString())
+            .defaultHeaders(authentication::apply)
+            .build();
+  }
+
+  public UUID credit(
+      String idempotencyKey, UUID userId, Money amount, WalletCreditPurpose purpose) {
+    CreditResponse response =
+        http.post()
+            .uri(CREDIT_PATH)
+            .header(IDEMPOTENCY_HEADER, idempotencyKey)
+            .contentType(MediaType.APPLICATION_JSON)
+            .body(new CreditRequest(userId, amount, purpose.source(), purpose.reason()))
+            .retrieve()
+            .body(CreditResponse.class);
+    return Objects.requireNonNull(response, "wallet credit response").operationGroupId();
+  }
+
+  private record CreditRequest(UUID userId, Money amount, String source, String reason) {}
+
+  @JsonIgnoreProperties(ignoreUnknown = true)
+  private record CreditResponse(UUID operationGroupId) {}
+}
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletEndpointProperties.java b/src/main/java/com/sportsbook/settlement/client/WalletEndpointProperties.java
new file mode 100644
index 0000000..9c5506e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletEndpointProperties.java
@@ -0,0 +1,26 @@
+package com.sportsbook.settlement.client;
+
+import java.net.URI;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("settlement.wallet")
+public record WalletEndpointProperties(URI baseUrl) {
+
+  private static final URI DEFAULT_BASE_URL = URI.create("http://localhost:8081");
+
+  public WalletEndpointProperties {
+    baseUrl = baseUrl == null ? DEFAULT_BASE_URL : baseUrl;
+    String path = baseUrl.getPath();
+    boolean rootPath = path != null && (path.isEmpty() || "/".equals(path));
+    if (!baseUrl.isAbsolute()
+        || (!"http".equals(baseUrl.getScheme()) && !"https".equals(baseUrl.getScheme()))
+        || baseUrl.getHost() == null
+        || baseUrl.getRawUserInfo() != null
+        || !rootPath
+        || baseUrl.getRawQuery() != null
+        || baseUrl.getRawFragment() != null) {
+      throw new IllegalArgumentException(
+          "settlement.wallet.base-url must be an HTTP(S) origin without credentials or suffixes");
+    }
+  }
+}


## `feat(wallet): submit locked stake forfeits`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index 045d51d..995ec67 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -15,6 +15,7 @@ import org.springframework.web.client.RestClient;
 public class WalletClient {
 
   static final String CREDIT_PATH = "/internal/v1/wallet/transactions/credit";
+  static final String FORFEIT_PATH = "/internal/v1/wallet/transactions/forfeit";
   static final String IDEMPOTENCY_HEADER = "Idempotency-Key";
 
   private final RestClient http;
@@ -56,8 +57,22 @@ public class WalletClient {
     return Objects.requireNonNull(response, "wallet credit response").operationGroupId();
   }
 
+  public UUID forfeit(String idempotencyKey, UUID userId, Money amount) {
+    CreditResponse response =
+        http.post()
+            .uri(FORFEIT_PATH)
+            .header(IDEMPOTENCY_HEADER, idempotencyKey)
+            .contentType(MediaType.APPLICATION_JSON)
+            .body(new ForfeitRequest(userId, amount))
+            .retrieve()
+            .body(CreditResponse.class);
+    return Objects.requireNonNull(response, "wallet forfeit response").operationGroupId();
+  }
+
   private record CreditRequest(UUID userId, Money amount, String source, String reason) {}
 
+  private record ForfeitRequest(UUID userId, Money amount) {}
+
   @JsonIgnoreProperties(ignoreUnknown = true)
   private record CreditResponse(UUID operationGroupId) {}
 }


## `feat(wallet): submit payout adjustments`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletAdjustmentProof.java b/src/main/java/com/sportsbook/settlement/client/WalletAdjustmentProof.java
new file mode 100644
index 0000000..4ee550b
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletAdjustmentProof.java
@@ -0,0 +1,31 @@
+package com.sportsbook.settlement.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+public record WalletAdjustmentProof(
+    UUID revisionId,
+    UUID betId,
+    long revisionNumber,
+    UUID userId,
+    Money previousPayout,
+    Money newPayout,
+    long deltaAmount,
+    Currency currency,
+    Status status,
+    Long queueSequence,
+    UUID operationGroupId,
+    Instant queuedAt,
+    Instant appliedAt,
+    Instant nextAttemptAt) {
+
+  public enum Status {
+    APPLIED,
+    BLOCKED,
+    REJECTED
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index d647bcd..3a59f65 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -16,6 +16,7 @@ public class WalletClient {
 
   static final String CREDIT_PATH = "/internal/v1/wallet/transactions/credit";
   static final String FORFEIT_PATH = "/internal/v1/wallet/transactions/forfeit";
+  static final String ADJUSTMENT_PATH = "/internal/v1/wallet/transactions/adjustment";
   static final String IDEMPOTENCY_HEADER = "Idempotency-Key";
 
   private final RestClient http;
@@ -75,6 +76,32 @@ public class WalletClient {
     return requireOperationGroupId(response);
   }
 
+  public WalletAdjustmentProof adjust(
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber,
+      UUID userId,
+      Money previousPayout,
+      Money newPayout) {
+    WalletAdjustmentProof proof =
+        http.post()
+            .uri(ADJUSTMENT_PATH)
+            .header(IDEMPOTENCY_HEADER, "settlement:revision:" + revisionId)
+            .contentType(MediaType.APPLICATION_JSON)
+            .body(
+                new AdjustmentRequest(
+                    revisionId, betId, revisionNumber, userId, previousPayout, newPayout))
+            .retrieve()
+            .onStatus(
+                HttpStatusCode::isError,
+                (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
+            .body(WalletAdjustmentProof.class);
+    if (proof == null || proof.revisionId() == null || proof.status() == null) {
+      throw WalletFailurePolicy.malformedSuccess();
+    }
+    return proof;
+  }
+
   private static UUID requireOperationGroupId(CreditResponse response) {
     if (response == null || response.operationGroupId() == null) {
       throw WalletFailurePolicy.malformedSuccess();
@@ -86,6 +113,14 @@ public class WalletClient {
 
   private record ForfeitRequest(UUID userId, Money amount) {}
 
+  private record AdjustmentRequest(
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber,
+      UUID userId,
+      Money previousPayout,
+      Money newPayout) {}
+
   @JsonIgnoreProperties(ignoreUnknown = true)
   private record CreditResponse(UUID operationGroupId) {}
 }


## `feat(wallet): retrieve durable adjustment proofs`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index 3a59f65..1d61ac1 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -102,6 +102,21 @@ public class WalletClient {
     return proof;
   }
 
+  public WalletAdjustmentProof findAdjustment(UUID revisionId) {
+    WalletAdjustmentProof proof =
+        http.get()
+            .uri(ADJUSTMENT_PATH + "/{revisionId}", revisionId)
+            .retrieve()
+            .onStatus(
+                HttpStatusCode::isError,
+                (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
+            .body(WalletAdjustmentProof.class);
+    if (proof == null || !revisionId.equals(proof.revisionId()) || proof.status() == null) {
+      throw WalletFailurePolicy.malformedSuccess();
+    }
+    return proof;
+  }
+
   private static UUID requireOperationGroupId(CreditResponse response) {
     if (response == null || response.operationGroupId() == null) {
       throw WalletFailurePolicy.malformedSuccess();


## `feat(wallet): model problem detail responses`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletProblemDetail.java b/src/main/java/com/sportsbook/settlement/client/WalletProblemDetail.java
new file mode 100644
index 0000000..fe5a32b
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletProblemDetail.java
@@ -0,0 +1,8 @@
+package com.sportsbook.settlement.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+import java.net.URI;
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+public record WalletProblemDetail(
+    URI type, String title, int status, String detail, URI instance, String errorCode) {}


## `feat(wallet): classify dependency failures`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index 995ec67..c9ee386 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -6,6 +6,7 @@ import java.util.Objects;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatusCode;
 import org.springframework.http.MediaType;
 import org.springframework.http.client.ClientHttpRequestFactory;
 import org.springframework.stereotype.Component;
@@ -53,6 +54,9 @@ public class WalletClient {
             .contentType(MediaType.APPLICATION_JSON)
             .body(new CreditRequest(userId, amount, purpose.source(), purpose.reason()))
             .retrieve()
+            .onStatus(
+                HttpStatusCode::isError,
+                (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
             .body(CreditResponse.class);
     return Objects.requireNonNull(response, "wallet credit response").operationGroupId();
   }
@@ -65,6 +69,9 @@ public class WalletClient {
             .contentType(MediaType.APPLICATION_JSON)
             .body(new ForfeitRequest(userId, amount))
             .retrieve()
+            .onStatus(
+                HttpStatusCode::isError,
+                (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
             .body(CreditResponse.class);
     return Objects.requireNonNull(response, "wallet forfeit response").operationGroupId();
   }
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
new file mode 100644
index 0000000..deae4f0
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletFailurePolicy.java
@@ -0,0 +1,63 @@
+package com.sportsbook.settlement.client;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.io.IOException;
+import org.springframework.http.client.ClientHttpResponse;
+
+public final class WalletFailurePolicy {
+
+  private static final ObjectMapper JSON = new ObjectMapper();
+
+  private WalletFailurePolicy() {}
+
+  public static void throwFor(ClientHttpResponse response) throws IOException {
+    int status = response.getStatusCode().value();
+    String errorCode = readErrorCode(response, status);
+    if (status == 429 || status >= 500) {
+      throw new TransientFailure(status, errorCode);
+    }
+    throw new PermanentFailure(status, errorCode);
+  }
+
+  private static String readErrorCode(ClientHttpResponse response, int status) throws IOException {
+    try {
+      WalletProblemDetail problem = JSON.readValue(response.getBody(), WalletProblemDetail.class);
+      return problem.errorCode() == null || problem.errorCode().isBlank()
+          ? "WALLET_HTTP_" + status
+          : problem.errorCode();
+    } catch (RuntimeException | IOException malformed) {
+      return "WALLET_HTTP_" + status;
+    }
+  }
+
+  public abstract static class Failure extends RuntimeException {
+    private final int status;
+    private final String errorCode;
+
+    private Failure(int status, String errorCode) {
+      super("wallet request failed: status=" + status + " errorCode=" + errorCode);
+      this.status = status;
+      this.errorCode = errorCode;
+    }
+
+    public int status() {
+      return status;
+    }
+
+    public String errorCode() {
+      return errorCode;
+    }
+  }
+
+  public static final class TransientFailure extends Failure {
+    private TransientFailure(int status, String errorCode) {
+      super(status, errorCode);
+    }
+  }
+
+  public static final class PermanentFailure extends Failure {
+    private PermanentFailure(int status, String errorCode) {
+      super(status, errorCode);
+    }
+  }
+}


## `feat(wallet): bound dependency timeouts`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletHttpProperties.java b/src/main/java/com/sportsbook/settlement/client/WalletHttpProperties.java
new file mode 100644
index 0000000..8a5494a
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletHttpProperties.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.client;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("settlement.wallet.http")
+public record WalletHttpProperties(Duration connectTimeout, Duration readTimeout) {
+
+  private static final Duration MAX_TIMEOUT = Duration.ofSeconds(5);
+
+  public WalletHttpProperties {
+    connectTimeout = connectTimeout == null ? Duration.ofSeconds(2) : connectTimeout;
+    readTimeout = readTimeout == null ? Duration.ofSeconds(5) : readTimeout;
+    if (invalid(connectTimeout) || invalid(readTimeout)) {
+      throw new IllegalArgumentException("Wallet HTTP timeouts must be in (0, 5s]");
+    }
+  }
+
+  private static boolean invalid(Duration timeout) {
+    return timeout.isZero() || timeout.isNegative() || timeout.compareTo(MAX_TIMEOUT) > 0;
+  }
+}


## `feat(wallet): enforce bounded HTTP transport`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index f86ecbf..045d51d 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -4,7 +4,10 @@ import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
 import com.sportsbook.protocol.value.Money;
 import java.util.Objects;
 import java.util.UUID;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.http.MediaType;
+import org.springframework.http.client.ClientHttpRequestFactory;
 import org.springframework.stereotype.Component;
 import org.springframework.web.client.RestClient;
 
@@ -16,13 +19,25 @@ public class WalletClient {
 
   private final RestClient http;
 
-  public WalletClient(
+  WalletClient(
       RestClient.Builder builder,
       WalletEndpointProperties endpoint,
       WalletAuthenticationHeaders authentication) {
+    this(builder, endpoint, authentication, null);
+  }
+
+  @Autowired
+  public WalletClient(
+      RestClient.Builder builder,
+      WalletEndpointProperties endpoint,
+      WalletAuthenticationHeaders authentication,
+      @Qualifier(WalletHttpConfiguration.REQUEST_FACTORY) ClientHttpRequestFactory requestFactory) {
+    RestClient.Builder configured = builder.clone();
+    if (requestFactory != null) {
+      configured.requestFactory(requestFactory);
+    }
     this.http =
-        builder
-            .clone()
+        configured
             .baseUrl(endpoint.baseUrl().toString())
             .defaultHeaders(authentication::apply)
             .build();
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletHttpConfiguration.java b/src/main/java/com/sportsbook/settlement/client/WalletHttpConfiguration.java
new file mode 100644
index 0000000..a99891d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/client/WalletHttpConfiguration.java
@@ -0,0 +1,21 @@
+package com.sportsbook.settlement.client;
+
+import java.net.http.HttpClient;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.client.ClientHttpRequestFactory;
+import org.springframework.http.client.JdkClientHttpRequestFactory;
+
+@Configuration(proxyBeanMethods = false)
+public class WalletHttpConfiguration {
+
+  static final String REQUEST_FACTORY = "settlementWalletRequestFactory";
+
+  @Bean(REQUEST_FACTORY)
+  ClientHttpRequestFactory settlementWalletRequestFactory(WalletHttpProperties properties) {
+    HttpClient client = HttpClient.newBuilder().connectTimeout(properties.connectTimeout()).build();
+    JdkClientHttpRequestFactory factory = new JdkClientHttpRequestFactory(client);
+    factory.setReadTimeout(properties.readTimeout());
+    return factory;
+  }
+}


## `feat(wallet): verify monetary operation proofs`

diff --git a/src/main/java/com/sportsbook/settlement/client/WalletClient.java b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
index 1d61ac1..747f1b1 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletClient.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletClient.java
@@ -2,6 +2,8 @@ package com.sportsbook.settlement.client;
 
 import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
 import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.Objects;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Qualifier;
@@ -47,7 +49,7 @@ public class WalletClient {
 
   public UUID credit(
       String idempotencyKey, UUID userId, Money amount, WalletCreditPurpose purpose) {
-    CreditResponse response =
+    OperationProof response =
         http.post()
             .uri(CREDIT_PATH)
             .header(IDEMPOTENCY_HEADER, idempotencyKey)
@@ -55,14 +57,14 @@ public class WalletClient {
             .body(new CreditRequest(userId, amount, purpose.source(), purpose.reason()))
             .retrieve()
             .onStatus(
-                HttpStatusCode::isError,
+                status -> status.value() != 200,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
-            .body(CreditResponse.class);
-    return requireOperationGroupId(response);
+            .body(OperationProof.class);
+    return requireOperationProof(response, userId, amount, purpose.proofReason());
   }
 
   public UUID forfeit(String idempotencyKey, UUID userId, Money amount) {
-    CreditResponse response =
+    OperationProof response =
         http.post()
             .uri(FORFEIT_PATH)
             .header(IDEMPOTENCY_HEADER, idempotencyKey)
@@ -70,10 +72,10 @@ public class WalletClient {
             .body(new ForfeitRequest(userId, amount))
             .retrieve()
             .onStatus(
-                HttpStatusCode::isError,
+                status -> status.value() != 200,
                 (request, httpResponse) -> WalletFailurePolicy.throwFor(httpResponse))
-            .body(CreditResponse.class);
-    return requireOperationGroupId(response);
+            .body(OperationProof.class);
+    return requireOperationProof(response, userId, amount, "BET_FORFEIT");
   }
 
   public WalletAdjustmentProof adjust(
@@ -117,8 +119,14 @@ public class WalletClient {
     return proof;
   }
 
-  private static UUID requireOperationGroupId(CreditResponse response) {
-    if (response == null || response.operationGroupId() == null) {
+  private static UUID requireOperationProof(
+      OperationProof response, UUID userId, Money amount, String reason) {
+    if (response == null
+        || response.operationGroupId() == null
+        || response.at() == null
+        || !Objects.equals(response.userId(), userId)
+        || !Objects.equals(response.amount(), amount)
+        || !Objects.equals(response.reason(), reason)) {
       throw WalletFailurePolicy.malformedSuccess();
     }
     return response.operationGroupId();
@@ -137,5 +145,6 @@ public class WalletClient {
       Money newPayout) {}
 
   @JsonIgnoreProperties(ignoreUnknown = true)
-  private record CreditResponse(UUID operationGroupId) {}
+  private record OperationProof(
+      UUID operationGroupId, UUID userId, Money amount, String reason, Instant at) {}
 }
diff --git a/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java b/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java
index ef4d579..7769296 100644
--- a/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java
+++ b/src/main/java/com/sportsbook/settlement/client/WalletCreditPurpose.java
@@ -1,16 +1,18 @@
 package com.sportsbook.settlement.client;
 
 public enum WalletCreditPurpose {
-  WHOLE_SLIP_VOID("USER_LOCKED", "VOID"),
-  RETURNED_STAKE("USER_LOCKED", "REFUND"),
-  PROFIT_PAYOUT("HOUSE_POOL", "PAYOUT");
+  WHOLE_SLIP_VOID("USER_LOCKED", "VOID", "BET_REFUND"),
+  RETURNED_STAKE("USER_LOCKED", "REFUND", "BET_REFUND"),
+  PROFIT_PAYOUT("HOUSE_POOL", "PAYOUT", "BET_PAYOUT");
 
   private final String source;
   private final String reason;
+  private final String proofReason;
 
-  WalletCreditPurpose(String source, String reason) {
+  WalletCreditPurpose(String source, String reason, String proofReason) {
     this.source = source;
     this.reason = reason;
+    this.proofReason = proofReason;
   }
 
   public String source() {
@@ -20,4 +22,8 @@ public enum WalletCreditPurpose {
   public String reason() {
     return reason;
   }
+
+  public String proofReason() {
+    return proofReason;
+  }
 }


