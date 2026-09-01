# 범위와 완전성을 보존하는 위험 한도 관리

## `feat(risk): model scoped limit updates`

diff --git a/src/main/java/com/sportsbook/admin/client/RiskLimitPayload.java b/src/main/java/com/sportsbook/admin/client/RiskLimitPayload.java
new file mode 100644
index 0000000..85c03b7
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/RiskLimitPayload.java
@@ -0,0 +1,22 @@
+package com.sportsbook.admin.client;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import com.sportsbook.protocol.value.Currency;
+import java.util.Objects;
+
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record RiskLimitPayload(RiskLimitType type, Currency currency, Long value) {
+
+  public static final long MAX_SAFE_VALUE = 9_007_199_254_740_991L;
+
+  public RiskLimitPayload {
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(value, "value");
+    if (value < 0 || value > MAX_SAFE_VALUE) {
+      throw new IllegalArgumentException("Risk limit value is outside the safe range");
+    }
+    if (type.requiresCurrency() != (currency != null)) {
+      throw new IllegalArgumentException("Risk limit currency scope does not match its type");
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/RiskLimitType.java b/src/main/java/com/sportsbook/admin/client/RiskLimitType.java
new file mode 100644
index 0000000..1af55f4
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/RiskLimitType.java
@@ -0,0 +1,12 @@
+package com.sportsbook.admin.client;
+
+public enum RiskLimitType {
+  STAKE_DAILY,
+  STAKE_WEEKLY,
+  STAKE_MONTHLY,
+  SELECTIONS_PER_MINUTE;
+
+  public boolean requiresCurrency() {
+    return this != SELECTIONS_PER_MINUTE;
+  }
+}


## `test(risk): enforce limit type currency scope`

diff --git a/src/test/java/com/sportsbook/admin/client/RiskLimitPayloadTest.java b/src/test/java/com/sportsbook/admin/client/RiskLimitPayloadTest.java
new file mode 100644
index 0000000..7547b86
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/RiskLimitPayloadTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class RiskLimitPayloadTest {
+
+  private final ObjectMapper json = new ObjectMapper();
+
+  @Test
+  void acceptsEveryValidScopeAndSafeBoundary() {
+    assertThatCode(() -> new RiskLimitPayload(RiskLimitType.STAKE_DAILY, Currency.KRW, 0L))
+        .doesNotThrowAnyException();
+    assertThatCode(
+            () ->
+                new RiskLimitPayload(
+                    RiskLimitType.STAKE_MONTHLY, Currency.USD, RiskLimitPayload.MAX_SAFE_VALUE))
+        .doesNotThrowAnyException();
+    assertThatCode(() -> new RiskLimitPayload(RiskLimitType.SELECTIONS_PER_MINUTE, null, 20L))
+        .doesNotThrowAnyException();
+  }
+
+  @Test
+  void rejectsMismatchedScopesAndUnsafeValues() {
+    assertThatThrownBy(() -> new RiskLimitPayload(RiskLimitType.STAKE_WEEKLY, null, 1L))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () -> new RiskLimitPayload(RiskLimitType.SELECTIONS_PER_MINUTE, Currency.KRW, 1L))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new RiskLimitPayload(RiskLimitType.STAKE_DAILY, Currency.KRW, -1L))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                new RiskLimitPayload(
+                    RiskLimitType.STAKE_DAILY, Currency.KRW, RiskLimitPayload.MAX_SAFE_VALUE + 1))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  @Test
+  void omitsCurrencyForSelectionLimits() throws Exception {
+    var payload = new RiskLimitPayload(RiskLimitType.SELECTIONS_PER_MINUTE, null, 20L);
+
+    assertThat(json.readTree(json.writeValueAsBytes(payload)))
+        .isEqualTo(json.readTree("{\"type\":\"SELECTIONS_PER_MINUTE\",\"value\":20}"));
+  }
+}


## `feat(risk): verify complete limits snapshots`

diff --git a/src/main/java/com/sportsbook/admin/client/RiskLimitsResponse.java b/src/main/java/com/sportsbook/admin/client/RiskLimitsResponse.java
new file mode 100644
index 0000000..276c230
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/RiskLimitsResponse.java
@@ -0,0 +1,66 @@
+package com.sportsbook.admin.client;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import com.sportsbook.protocol.value.Currency;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Set;
+import java.util.UUID;
+
+public record RiskLimitsResponse(UUID userId, List<Entry> limits) {
+
+  private static final Set<Target> REQUIRED_TARGETS =
+      Set.of(
+          new Target(RiskLimitType.STAKE_DAILY, Currency.KRW),
+          new Target(RiskLimitType.STAKE_DAILY, Currency.USD),
+          new Target(RiskLimitType.STAKE_WEEKLY, Currency.KRW),
+          new Target(RiskLimitType.STAKE_WEEKLY, Currency.USD),
+          new Target(RiskLimitType.STAKE_MONTHLY, Currency.KRW),
+          new Target(RiskLimitType.STAKE_MONTHLY, Currency.USD),
+          new Target(RiskLimitType.SELECTIONS_PER_MINUTE, null));
+
+  public static RiskLimitsResponse verify(UUID requestedUser, RiskLimitsResponse response) {
+    if (response == null
+        || !requestedUser.equals(response.userId())
+        || response.limits() == null
+        || response.limits().size() != REQUIRED_TARGETS.size()) {
+      throw violation();
+    }
+    Set<Target> targets = new HashSet<>();
+    for (Entry entry : response.limits()) {
+      if (!valid(entry) || !targets.add(new Target(entry.type(), entry.currency()))) {
+        throw violation();
+      }
+    }
+    if (!targets.equals(REQUIRED_TARGETS)) {
+      throw violation();
+    }
+    return response;
+  }
+
+  private static boolean valid(Entry entry) {
+    if (entry == null
+        || entry.type() == null
+        || entry.value() == null
+        || entry.source() == null
+        || entry.value() < 0
+        || entry.value() > RiskLimitPayload.MAX_SAFE_VALUE) {
+      return false;
+    }
+    return entry.type().requiresCurrency() == (entry.currency() != null);
+  }
+
+  private static DownstreamContractException violation() {
+    return new DownstreamContractException("complete seven-entry Risk limits response");
+  }
+
+  @JsonInclude(JsonInclude.Include.NON_NULL)
+  public record Entry(RiskLimitType type, Currency currency, Long value, Source source) {}
+
+  public enum Source {
+    POLICY,
+    OVERRIDE
+  }
+
+  private record Target(RiskLimitType type, Currency currency) {}
+}


## `test(risk): reject incomplete limits snapshots`

diff --git a/src/test/java/com/sportsbook/admin/client/RiskLimitsResponseTest.java b/src/test/java/com/sportsbook/admin/client/RiskLimitsResponseTest.java
new file mode 100644
index 0000000..136a45a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/RiskLimitsResponseTest.java
@@ -0,0 +1,65 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.admin.client.RiskLimitsResponse.Entry;
+import com.sportsbook.admin.client.RiskLimitsResponse.Source;
+import com.sportsbook.protocol.value.Currency;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskLimitsResponseTest {
+
+  private static final UUID USER = UUID.fromString("018f0000-0000-7000-8000-000000000121");
+
+  @Test
+  void acceptsTheSevenUniqueSupportedTargets() {
+    RiskLimitsResponse response = new RiskLimitsResponse(USER, validEntries());
+
+    assertThat(RiskLimitsResponse.verify(USER, response)).isSameAs(response);
+  }
+
+  @Test
+  void rejectsMissingDuplicateMismatchedAndUnsafeEntries() {
+    UUID other = UUID.fromString("018f0000-0000-7000-8000-000000000122");
+    List<RiskLimitsResponse> invalid = new ArrayList<>();
+    invalid.add(new RiskLimitsResponse(other, validEntries()));
+    invalid.add(new RiskLimitsResponse(USER, validEntries().subList(0, 6)));
+
+    List<Entry> duplicate = new ArrayList<>(validEntries());
+    duplicate.set(6, duplicate.get(0));
+    invalid.add(new RiskLimitsResponse(USER, duplicate));
+
+    List<Entry> wrongScope = new ArrayList<>(validEntries());
+    wrongScope.set(
+        6, new Entry(RiskLimitType.SELECTIONS_PER_MINUTE, Currency.KRW, 20L, Source.POLICY));
+    invalid.add(new RiskLimitsResponse(USER, wrongScope));
+
+    List<Entry> unsafe = new ArrayList<>(validEntries());
+    unsafe.set(0, new Entry(RiskLimitType.STAKE_DAILY, Currency.KRW, -1L, Source.POLICY));
+    invalid.add(new RiskLimitsResponse(USER, unsafe));
+
+    invalid.forEach(
+        response ->
+            assertThatThrownBy(() -> RiskLimitsResponse.verify(USER, response))
+                .isInstanceOf(DownstreamContractException.class));
+  }
+
+  private static List<Entry> validEntries() {
+    return List.of(
+        entry(RiskLimitType.STAKE_DAILY, Currency.KRW, 1_000L),
+        entry(RiskLimitType.STAKE_DAILY, Currency.USD, 100L),
+        entry(RiskLimitType.STAKE_WEEKLY, Currency.KRW, 5_000L),
+        entry(RiskLimitType.STAKE_WEEKLY, Currency.USD, 500L),
+        entry(RiskLimitType.STAKE_MONTHLY, Currency.KRW, 20_000L),
+        entry(RiskLimitType.STAKE_MONTHLY, Currency.USD, 2_000L),
+        entry(RiskLimitType.SELECTIONS_PER_MINUTE, null, 20L));
+  }
+
+  private static Entry entry(RiskLimitType type, Currency currency, long value) {
+    return new Entry(type, currency, value, Source.POLICY);
+  }
+}


## `feat(risk): fetch complete user limits`

diff --git a/src/main/java/com/sportsbook/admin/client/RiskClient.java b/src/main/java/com/sportsbook/admin/client/RiskClient.java
new file mode 100644
index 0000000..d9d6fa6
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/RiskClient.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin.client;
+
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+
+@Component
+public class RiskClient {
+
+  private final RestClient http;
+  private final DownstreamFailureMapper failures = new DownstreamFailureMapper();
+
+  public RiskClient(@Qualifier("riskRestClient") RestClient http) {
+    this.http = http;
+  }
+
+  public RiskLimitsResponse getLimits(UUID userId) {
+    var response =
+        failures.execute(
+            () ->
+                http.get()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "v1", "risk", "limits")
+                                .pathSegment(userId.toString())
+                                .build())
+                    .retrieve()
+                    .toEntity(RiskLimitsResponse.class));
+    RiskLimitsResponse body =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Risk limits GET must return HTTP 200 with a body");
+    return RiskLimitsResponse.verify(userId, body);
+  }
+}


## `test(risk): fetch exact typed limits response`

diff --git a/src/test/java/com/sportsbook/admin/client/RiskClientGetTest.java b/src/test/java/com/sportsbook/admin/client/RiskClientGetTest.java
new file mode 100644
index 0000000..76213f5
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/RiskClientGetTest.java
@@ -0,0 +1,57 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.http.HttpMethod.GET;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class RiskClientGetTest {
+
+  @Test
+  void fetchesTheExactUserAndValidatesAllSevenEntries() {
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000123");
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://risk.test")
+            .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
+            .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://risk.test/internal/v1/risk/limits/" + userId))
+        .andExpect(method(GET))
+        .andExpect(header(DownstreamHeaders.INTERNAL_SERVICE, "admin-api"))
+        .andExpect(header(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK))
+        .andRespond(withSuccess(response(userId), MediaType.APPLICATION_JSON));
+
+    RiskLimitsResponse result = new RiskClient(builder.build()).getLimits(userId);
+
+    assertThat(result.userId()).isEqualTo(userId);
+    assertThat(result.limits()).hasSize(7);
+    assertThat(result.limits().get(6).type()).isEqualTo(RiskLimitType.SELECTIONS_PER_MINUTE);
+    assertThat(result.limits().get(6).currency()).isNull();
+    server.verify();
+  }
+
+  private static String response(UUID userId) {
+    return """
+        {"userId":"%s","limits":[
+          {"type":"STAKE_DAILY","currency":"KRW","value":1000,"source":"POLICY"},
+          {"type":"STAKE_DAILY","currency":"USD","value":100,"source":"POLICY"},
+          {"type":"STAKE_WEEKLY","currency":"KRW","value":5000,"source":"POLICY"},
+          {"type":"STAKE_WEEKLY","currency":"USD","value":500,"source":"POLICY"},
+          {"type":"STAKE_MONTHLY","currency":"KRW","value":20000,"source":"POLICY"},
+          {"type":"STAKE_MONTHLY","currency":"USD","value":2000,"source":"POLICY"},
+          {"type":"SELECTIONS_PER_MINUTE","value":20,"source":"OVERRIDE"}
+        ]}
+        """
+        .formatted(userId);
+  }
+}


## `feat(risk): replace one user limit`

diff --git a/src/main/java/com/sportsbook/admin/client/RiskClient.java b/src/main/java/com/sportsbook/admin/client/RiskClient.java
index d9d6fa6..ec1b8b5 100644
--- a/src/main/java/com/sportsbook/admin/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/admin/client/RiskClient.java
@@ -3,6 +3,7 @@ package com.sportsbook.admin.client;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
 import org.springframework.stereotype.Component;
 import org.springframework.web.client.RestClient;
 
@@ -37,4 +38,23 @@ public class RiskClient {
             "Risk limits GET must return HTTP 200 with a body");
     return RiskLimitsResponse.verify(userId, body);
   }
+
+  public void setLimit(UUID userId, RiskLimitPayload limit) {
+    var response =
+        failures.execute(
+            () ->
+                http.patch()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "v1", "risk", "limits")
+                                .pathSegment(userId.toString())
+                                .build())
+                    .contentType(MediaType.APPLICATION_JSON)
+                    .body(limit)
+                    .retrieve()
+                    .toEntity(byte[].class));
+    DownstreamContract.requireEmpty(
+        response, HttpStatus.NO_CONTENT, "Risk limit PATCH must return empty HTTP 204");
+  }
 }


## `test(risk): patch the exact limit target`

diff --git a/src/test/java/com/sportsbook/admin/client/RiskClientPatchTest.java b/src/test/java/com/sportsbook/admin/client/RiskClientPatchTest.java
new file mode 100644
index 0000000..5ae2719
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/RiskClientPatchTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin.client;
+
+import static org.springframework.http.HttpMethod.PATCH;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.content;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withNoContent;
+
+import com.sportsbook.protocol.value.Currency;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class RiskClientPatchTest {
+
+  @Test
+  void replacesTheExactTypedCurrencyLimit() {
+    UUID userId = UUID.fromString("018f0000-0000-7000-8000-000000000124");
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://risk.test")
+            .defaultHeader(DownstreamHeaders.INTERNAL_SERVICE, DownstreamHeaders.ADMIN_API)
+            .defaultHeader(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://risk.test/internal/v1/risk/limits/" + userId))
+        .andExpect(method(PATCH))
+        .andExpect(header(DownstreamHeaders.INTERNAL_SERVICE, "admin-api"))
+        .andExpect(header(DownstreamHeaders.INTERNAL_API_KEY, ClientIsolationFixture.RISK))
+        .andExpect(content().json("{\"type\":\"STAKE_DAILY\",\"currency\":\"KRW\",\"value\":750}"))
+        .andRespond(withNoContent());
+
+    new RiskClient(builder.build())
+        .setLimit(userId, new RiskLimitPayload(RiskLimitType.STAKE_DAILY, Currency.KRW, 750L));
+
+    server.verify();
+  }
+}


## `feat(risk): clear one user limit override`

diff --git a/src/main/java/com/sportsbook/admin/client/RiskClient.java b/src/main/java/com/sportsbook/admin/client/RiskClient.java
index ec1b8b5..a165dd3 100644
--- a/src/main/java/com/sportsbook/admin/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/admin/client/RiskClient.java
@@ -1,5 +1,6 @@
 package com.sportsbook.admin.client;
 
+import com.sportsbook.protocol.value.Currency;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.http.HttpStatus;
@@ -57,4 +58,29 @@ public class RiskClient {
     DownstreamContract.requireEmpty(
         response, HttpStatus.NO_CONTENT, "Risk limit PATCH must return empty HTTP 204");
   }
+
+  public void clearLimit(UUID userId, RiskLimitType type, Currency currency) {
+    if (type.requiresCurrency() != (currency != null)) {
+      throw new IllegalArgumentException("Risk limit currency scope does not match its type");
+    }
+    var response =
+        failures.execute(
+            () ->
+                http.delete()
+                    .uri(
+                        builder -> {
+                          var path =
+                              builder
+                                  .pathSegment("internal", "v1", "risk", "limits")
+                                  .pathSegment(userId.toString(), type.name());
+                          if (currency != null) {
+                            path.queryParam("currency", currency.name());
+                          }
+                          return path.build();
+                        })
+                    .retrieve()
+                    .toEntity(byte[].class));
+    DownstreamContract.requireEmpty(
+        response, HttpStatus.NO_CONTENT, "Risk limit DELETE must return empty HTTP 204");
+  }
 }


