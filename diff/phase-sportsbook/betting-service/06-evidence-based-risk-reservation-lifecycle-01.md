# 리스크 예약의 증거 기반 수명 주기

## `feat(error): separate dependency and risk outcomes`

diff --git a/src/main/java/com/sportsbook/betting/error/DependencyUnavailableException.java b/src/main/java/com/sportsbook/betting/error/DependencyUnavailableException.java
new file mode 100644
index 0000000..70b9af1
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/DependencyUnavailableException.java
@@ -0,0 +1,15 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class DependencyUnavailableException extends BetPlacementException {
+
+  public DependencyUnavailableException(String message) {
+    super(ErrorCode.SERVICE_UNAVAILABLE, message);
+  }
+
+  public DependencyUnavailableException(String message, Throwable cause) {
+    super(ErrorCode.SERVICE_UNAVAILABLE, message);
+    initCause(cause);
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/RiskLimitException.java b/src/main/java/com/sportsbook/betting/error/RiskLimitException.java
new file mode 100644
index 0000000..29970a3
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/RiskLimitException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class RiskLimitException extends BetPlacementException {
+
+  public RiskLimitException(String message) {
+    super(ErrorCode.LIMIT_EXCEEDED, message == null ? "Risk limit exceeded" : message);
+  }
+}


## `feat(risk): define reservation wire contract`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskReservationRequest.java b/src/main/java/com/sportsbook/betting/client/RiskReservationRequest.java
new file mode 100644
index 0000000..dd76354
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/RiskReservationRequest.java
@@ -0,0 +1,26 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.protocol.value.Money;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+
+public record RiskReservationRequest(
+    String userId, String betId, Money stake, List<String> selectionIds) {
+
+  public RiskReservationRequest {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(stake, "stake");
+    selectionIds = List.copyOf(Objects.requireNonNull(selectionIds, "selectionIds"));
+  }
+
+  static RiskReservationRequest of(
+      UUID betId, UUID userId, Money fullExposure, List<UUID> selectionIds) {
+    return new RiskReservationRequest(
+        userId.toString(),
+        betId.toString(),
+        fullExposure,
+        selectionIds.stream().map(UUID::toString).toList());
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java b/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java
new file mode 100644
index 0000000..e953820
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java
@@ -0,0 +1,40 @@
+package com.sportsbook.betting.client;
+
+import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.ValidationFailedException;
+import java.io.IOException;
+import java.time.Instant;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.client.ClientHttpResponse;
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+public record RiskReservationResponse(
+    boolean approved,
+    boolean replayed,
+    String rejectionReason,
+    String reservationState,
+    Instant expiresAt,
+    String reservationToken) {}
+
+@JsonIgnoreProperties(ignoreUnknown = true)
+record RiskProblem(String errorCode) {
+
+  private static final ObjectMapper JSON = new ObjectMapper();
+
+  static RuntimeException mapReservation(ClientHttpResponse response) {
+    try {
+      if (response.getStatusCode().value() != HttpStatus.BAD_REQUEST.value()) {
+        return new DependencyUnavailableException("Risk reservation was not accepted");
+      }
+      RiskProblem problem = JSON.readValue(response.getBody(), RiskProblem.class);
+      if ("VALIDATION_FAILED".equals(problem.errorCode())) {
+        return new ValidationFailedException("Risk rejected an invalid reservation");
+      }
+      return new DependencyUnavailableException("Risk returned an unexpected validation problem");
+    } catch (IOException unreadable) {
+      return new DependencyUnavailableException("Risk returned an unreadable problem", unreadable);
+    }
+  }
+}


## `test(risk): verify reservation wire fields`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskReservationWireTest.java b/src/test/java/com/sportsbook/betting/client/RiskReservationWireTest.java
new file mode 100644
index 0000000..53a42f6
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/RiskReservationWireTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.protocol.value.Money;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.mock.http.client.MockClientHttpResponse;
+
+class RiskReservationWireTest {
+
+  private final ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
+
+  @Test
+  void sendsFullExposureAndReadsReservationToken() throws Exception {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    String json =
+        mapper.writeValueAsString(
+            RiskReservationRequest.of(betId, userId, Money.krw(6_000), List.of(selectionId)));
+
+    assertThat(json).contains("\"betId\":\"" + betId + "\"");
+    assertThat(json).contains("\"amount\":6000");
+    assertThat(json).contains("\"selectionIds\":[\"" + selectionId + "\"]");
+
+    RiskReservationResponse response =
+        mapper.readValue(
+            "{\"approved\":true,\"replayed\":false,\"reservationState\":\"RESERVED\","
+                + "\"expiresAt\":\"2026-08-22T00:02:00Z\",\"reservationToken\":\""
+                + "a".repeat(64)
+                + "\"}",
+            RiskReservationResponse.class);
+    assertThat(response.reservationToken()).isEqualTo("a".repeat(64));
+    assertThat(response.expiresAt()).isEqualTo(Instant.parse("2026-08-22T00:02:00Z"));
+  }
+
+  @Test
+  void recognizesOnlyTheFixedValidationProblemWithoutItsDetail() {
+    String body =
+        "{\"status\":400,\"errorCode\":\"VALIDATION_FAILED\","
+            + "\"detail\":\"provider internal validation detail\"}";
+    RuntimeException verdict =
+        RiskProblem.mapReservation(
+            new MockClientHttpResponse(
+                body.getBytes(StandardCharsets.UTF_8), HttpStatus.BAD_REQUEST));
+
+    assertThat(verdict)
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessage("Risk rejected an invalid reservation");
+  }
+}


## `feat(risk): reserve full betting exposure`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskClient.java b/src/main/java/com/sportsbook/betting/client/RiskClient.java
new file mode 100644
index 0000000..e5a2d7e
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/RiskClient.java
@@ -0,0 +1,93 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.DuplicateBetException;
+import com.sportsbook.betting.error.RiskLimitException;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+import org.springframework.web.client.RestClientException;
+
+@Component
+public class RiskClient {
+
+  private static final String RESERVATIONS = "/internal/v1/risk/reservations";
+
+  private final RestClient http;
+
+  public RiskClient(@Qualifier("riskRestClient") RestClient http) {
+    this.http = http;
+  }
+
+  public Reservation reserve(UUID betId, UUID userId, Money fullExposure, List<UUID> selectionIds) {
+    try {
+      RiskReservationResponse response =
+          http.post()
+              .uri(RESERVATIONS)
+              .contentType(MediaType.APPLICATION_JSON)
+              .body(RiskReservationRequest.of(betId, userId, fullExposure, selectionIds))
+              .retrieve()
+              .onStatus(
+                  status -> status.value() == HttpStatus.CONFLICT.value(),
+                  (request, ignored) -> {
+                    throw new DuplicateBetException("Risk reservation identity conflicts");
+                  })
+              .onStatus(
+                  status -> status.value() != HttpStatus.OK.value(),
+                  (request, error) -> {
+                    throw RiskProblem.mapReservation(error);
+                  })
+              .body(RiskReservationResponse.class);
+      return requireApproved(response);
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Risk reservation is unavailable", exception);
+    }
+  }
+
+  private static Reservation requireApproved(RiskReservationResponse response) {
+    if (response == null) {
+      throw new DependencyUnavailableException("Risk returned an empty response");
+    }
+    if (!response.approved()) {
+      throw new RiskLimitException(response.rejectionReason());
+    }
+    try {
+      return new Reservation(
+          ReservationState.valueOf(response.reservationState()),
+          Objects.requireNonNull(response.expiresAt(), "expiresAt"),
+          response.reservationToken());
+    } catch (RuntimeException exception) {
+      throw new DependencyUnavailableException("Risk returned an invalid reservation", exception);
+    }
+  }
+
+  public enum ReservationState {
+    RESERVED,
+    COMMITTED
+  }
+
+  public record Reservation(ReservationState state, Instant expiresAt, String token) {
+    public Reservation {
+      Objects.requireNonNull(state, "state");
+      Objects.requireNonNull(expiresAt, "expiresAt");
+      if (token == null || !token.matches("[0-9a-f]{64}")) {
+        throw new IllegalArgumentException("reservation token must be lowercase SHA-256");
+      }
+    }
+
+    public boolean alreadyCommitted() {
+      return state == ReservationState.COMMITTED;
+    }
+  }
+}


## `test(risk): verify reservation outcomes`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
new file mode 100644
index 0000000..c762c84
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.jsonPath;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withStatus;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.RiskLimitException;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.protocol.value.Money;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class RiskClientTest {
+
+  private MockRestServiceServer server;
+  private RiskClient client;
+
+  @BeforeEach
+  void setUp() {
+    RestClient.Builder builder = RestClient.builder().baseUrl("http://risk");
+    server = MockRestServiceServer.bindTo(builder).build();
+    client = new RiskClient(builder.build());
+  }
+
+  @Test
+  void returnsOpaqueTokenForApprovedFullExposure() {
+    String token = "a".repeat(64);
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations"))
+        .andExpect(method(HttpMethod.POST))
+        .andExpect(jsonPath("$.stake.amount").value(6_000))
+        .andRespond(
+            withSuccess(
+                "{\"approved\":true,\"replayed\":false,\"reservationState\":\"RESERVED\","
+                    + "\"expiresAt\":\"2026-08-22T00:02:00Z\",\"reservationToken\":\""
+                    + token
+                    + "\"}",
+                MediaType.APPLICATION_JSON));
+
+    RiskClient.Reservation reservation = reserve();
+
+    assertThat(reservation.token()).isEqualTo(token);
+    server.verify();
+  }
+
+  @Test
+  void treatsApprovedFalseAsBusinessVerdict() {
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations"))
+        .andRespond(
+            withSuccess(
+                "{\"approved\":false,\"replayed\":false,"
+                    + "\"rejectionReason\":\"daily limit\",\"patterns\":[]}",
+                MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(this::reserve)
+        .isInstanceOf(RiskLimitException.class)
+        .hasMessage("daily limit");
+  }
+
+  @Test
+  void acceptsOnlyHttp200OrTheFixedValidationVerdict() {
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations"))
+        .andRespond(
+            withStatus(HttpStatus.BAD_REQUEST)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body("{\"errorCode\":\"VALIDATION_FAILED\",\"detail\":\"hidden\"}"));
+    assertThatThrownBy(this::reserve)
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessage("Risk rejected an invalid reservation");
+    server.reset();
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations"))
+        .andRespond(withStatus(HttpStatus.CREATED));
+    assertThatThrownBy(this::reserve).isInstanceOf(DependencyUnavailableException.class);
+  }
+
+  private RiskClient.Reservation reserve() {
+    return client.reserve(
+        UUID.randomUUID(), UUID.randomUUID(), Money.krw(6_000), List.of(UUID.randomUUID()));
+  }
+}


## `feat(risk): commit reservations with opaque token`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskClient.java b/src/main/java/com/sportsbook/betting/client/RiskClient.java
index e5a2d7e..dd262fc 100644
--- a/src/main/java/com/sportsbook/betting/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/betting/client/RiskClient.java
@@ -55,6 +55,43 @@ public class RiskClient {
     }
   }
 
+  public CommitResult commit(UUID betId, String reservationToken) {
+    if (reservationToken == null || !reservationToken.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("reservationToken must be lowercase SHA-256");
+    }
+    try {
+      http.put()
+          .uri(RESERVATIONS + "/{betId}/commit", betId)
+          .header("X-Risk-Reservation-Token", reservationToken)
+          .retrieve()
+          .onStatus(
+              status -> status.value() == HttpStatus.NOT_FOUND.value(),
+              (request, ignored) -> {
+                throw new ReservationMissing();
+              })
+          .onStatus(
+              status -> status.value() == HttpStatus.CONFLICT.value(),
+              (request, ignored) -> {
+                throw new ReservationConflict();
+              })
+          .onStatus(
+              status -> status.value() != HttpStatus.NO_CONTENT.value(),
+              (request, ignored) -> {
+                throw new DependencyUnavailableException("Risk commit was not accepted");
+              })
+          .toBodilessEntity();
+      return CommitResult.COMMITTED;
+    } catch (ReservationMissing exception) {
+      return CommitResult.NOT_FOUND;
+    } catch (ReservationConflict exception) {
+      return CommitResult.CONFLICT;
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Risk commit is unavailable", exception);
+    }
+  }
+
   private static Reservation requireApproved(RiskReservationResponse response) {
     if (response == null) {
       throw new DependencyUnavailableException("Risk returned an empty response");
@@ -77,6 +114,12 @@ public class RiskClient {
     COMMITTED
   }
 
+  public enum CommitResult {
+    COMMITTED,
+    NOT_FOUND,
+    CONFLICT
+  }
+
   public record Reservation(ReservationState state, Instant expiresAt, String token) {
     public Reservation {
       Objects.requireNonNull(state, "state");
@@ -90,4 +133,12 @@ public class RiskClient {
       return state == ReservationState.COMMITTED;
     }
   }
+
+  private static final class ReservationMissing extends RuntimeException {
+    private static final long serialVersionUID = 1L;
+  }
+
+  private static final class ReservationConflict extends RuntimeException {
+    private static final long serialVersionUID = 1L;
+  }
 }


## `test(risk): verify token-bound commit`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
index c762c84..0a09235 100644
--- a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
@@ -2,9 +2,12 @@ package com.sportsbook.betting.client;
 
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
 import static org.springframework.test.web.client.match.MockRestRequestMatchers.jsonPath;
 import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
 import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withNoContent;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withResourceNotFound;
 import static org.springframework.test.web.client.response.MockRestResponseCreators.withStatus;
 import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
 
@@ -88,6 +91,54 @@ class RiskClientTest {
     assertThatThrownBy(this::reserve).isInstanceOf(DependencyUnavailableException.class);
   }
 
+  @Test
+  void presentsPersistedTokenWhenCommitting() {
+    UUID betId = UUID.randomUUID();
+    String token = "b".repeat(64);
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId + "/commit"))
+        .andExpect(method(HttpMethod.PUT))
+        .andExpect(header("X-Risk-Reservation-Token", token))
+        .andRespond(withNoContent());
+
+    assertThat(client.commit(betId, token)).isEqualTo(RiskClient.CommitResult.COMMITTED);
+    server.verify();
+  }
+
+  @Test
+  void returnsFalseForExpiredReservation() {
+    UUID betId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId + "/commit"))
+        .andRespond(withResourceNotFound());
+
+    assertThat(client.commit(betId, "c".repeat(64))).isEqualTo(RiskClient.CommitResult.NOT_FOUND);
+  }
+
+  @Test
+  void returnsDefinitiveConflictWithoutRetryingCommit() {
+    UUID betId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId + "/commit"))
+        .andRespond(
+            org.springframework.test.web.client.response.MockRestResponseCreators.withStatus(
+                HttpStatus.CONFLICT));
+
+    assertThat(client.commit(betId, "d".repeat(64))).isEqualTo(RiskClient.CommitResult.CONFLICT);
+  }
+
+  @Test
+  void acceptsOnlyNoContentForCommit() {
+    UUID betId = UUID.randomUUID();
+    for (HttpStatus status : List.of(HttpStatus.OK, HttpStatus.FOUND)) {
+      server
+          .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId + "/commit"))
+          .andRespond(withStatus(status));
+      assertThatThrownBy(() -> client.commit(betId, "e".repeat(64)))
+          .isInstanceOf(DependencyUnavailableException.class);
+      server.reset();
+    }
+  }
   private RiskClient.Reservation reserve() {
     return client.reserve(
         UUID.randomUUID(), UUID.randomUUID(), Money.krw(6_000), List.of(UUID.randomUUID()));


