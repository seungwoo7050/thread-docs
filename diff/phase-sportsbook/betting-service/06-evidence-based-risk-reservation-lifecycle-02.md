## `feat(risk): release unused reservations`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskClient.java b/src/main/java/com/sportsbook/betting/client/RiskClient.java
index dd262fc..f835ad5 100644
--- a/src/main/java/com/sportsbook/betting/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/betting/client/RiskClient.java
@@ -11,7 +11,6 @@ import java.util.Objects;
 import java.util.UUID;
 import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.http.HttpStatus;
-import org.springframework.http.HttpStatusCode;
 import org.springframework.http.MediaType;
 import org.springframework.stereotype.Component;
 import org.springframework.web.client.RestClient;
@@ -92,6 +91,32 @@ public class RiskClient {
     }
   }
 
+  public ReleaseResult release(UUID betId) {
+    try {
+      http.delete()
+          .uri(RESERVATIONS + "/{betId}", betId)
+          .retrieve()
+          .onStatus(
+              status -> status.value() == HttpStatus.CONFLICT.value(),
+              (request, ignored) -> {
+                throw new ReservationCommitted();
+              })
+          .onStatus(
+              status -> status.value() != HttpStatus.NO_CONTENT.value(),
+              (request, ignored) -> {
+                throw new DependencyUnavailableException("Risk release was not accepted");
+              })
+          .toBodilessEntity();
+      return ReleaseResult.RELEASED;
+    } catch (ReservationCommitted exception) {
+      return ReleaseResult.COMMITTED;
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Risk release is unavailable", exception);
+    }
+  }
+
   private static Reservation requireApproved(RiskReservationResponse response) {
     if (response == null) {
       throw new DependencyUnavailableException("Risk returned an empty response");
@@ -120,6 +145,11 @@ public class RiskClient {
     CONFLICT
   }
 
+  public enum ReleaseResult {
+    RELEASED,
+    COMMITTED
+  }
+
   public record Reservation(ReservationState state, Instant expiresAt, String token) {
     public Reservation {
       Objects.requireNonNull(state, "state");
@@ -141,4 +171,8 @@ public class RiskClient {
   private static final class ReservationConflict extends RuntimeException {
     private static final long serialVersionUID = 1L;
   }
+
+  private static final class ReservationCommitted extends RuntimeException {
+    private static final long serialVersionUID = 1L;
+  }
 }


## `test(risk): verify reservation release`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
index 0a09235..14d853f 100644
--- a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
@@ -139,6 +139,45 @@ class RiskClientTest {
       server.reset();
     }
   }
+
+  @Test
+  void releasesReservationIdempotently() {
+    UUID betId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId))
+        .andExpect(method(HttpMethod.DELETE))
+        .andRespond(withNoContent());
+
+    assertThat(client.release(betId)).isEqualTo(RiskClient.ReleaseResult.RELEASED);
+
+    server.verify();
+  }
+
+  @Test
+  void reportsCommittedReservationAsDefinitiveReleaseConflict() {
+    UUID betId = UUID.randomUUID();
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId))
+        .andRespond(
+            org.springframework.test.web.client.response.MockRestResponseCreators.withStatus(
+                HttpStatus.CONFLICT));
+
+    assertThat(client.release(betId)).isEqualTo(RiskClient.ReleaseResult.COMMITTED);
+  }
+
+  @Test
+  void acceptsOnlyNoContentForRelease() {
+    UUID betId = UUID.randomUUID();
+    for (HttpStatus status : List.of(HttpStatus.OK, HttpStatus.FOUND)) {
+      server
+          .expect(requestTo("http://risk/internal/v1/risk/reservations/" + betId))
+          .andRespond(withStatus(status));
+      assertThatThrownBy(() -> client.release(betId))
+          .isInstanceOf(DependencyUnavailableException.class);
+      server.reset();
+    }
+  }
+
   private RiskClient.Reservation reserve() {
     return client.reserve(
         UUID.randomUUID(), UUID.randomUUID(), Money.krw(6_000), List.of(UUID.randomUUID()));


## `feat(risk): bound dependency failures with circuit breaker`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskClient.java b/src/main/java/com/sportsbook/betting/client/RiskClient.java
index f835ad5..c220526 100644
--- a/src/main/java/com/sportsbook/betting/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/betting/client/RiskClient.java
@@ -5,6 +5,7 @@ import com.sportsbook.betting.error.DependencyUnavailableException;
 import com.sportsbook.betting.error.DuplicateBetException;
 import com.sportsbook.betting.error.RiskLimitException;
 import com.sportsbook.protocol.value.Money;
+import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
 import java.time.Instant;
 import java.util.List;
 import java.util.Objects;
@@ -27,6 +28,7 @@ public class RiskClient {
     this.http = http;
   }
 
+  @CircuitBreaker(name = "riskClient", fallbackMethod = "reserveFallback")
   public Reservation reserve(UUID betId, UUID userId, Money fullExposure, List<UUID> selectionIds) {
     try {
       RiskReservationResponse response =
@@ -54,6 +56,7 @@ public class RiskClient {
     }
   }
 
+  @CircuitBreaker(name = "riskClient", fallbackMethod = "commitFallback")
   public CommitResult commit(UUID betId, String reservationToken) {
     if (reservationToken == null || !reservationToken.matches("[0-9a-f]{64}")) {
       throw new IllegalArgumentException("reservationToken must be lowercase SHA-256");
@@ -91,6 +94,7 @@ public class RiskClient {
     }
   }
 
+  @CircuitBreaker(name = "riskClient", fallbackMethod = "releaseFallback")
   public ReleaseResult release(UUID betId) {
     try {
       http.delete()
@@ -134,6 +138,26 @@ public class RiskClient {
     }
   }
 
+  private Reservation reserveFallback(
+      UUID betId, UUID userId, Money exposure, List<UUID> selections, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private CommitResult commitFallback(UUID betId, String token, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private ReleaseResult releaseFallback(UUID betId, Throwable failure) {
+    throw fallback(failure);
+  }
+
+  private static RuntimeException fallback(Throwable failure) {
+    if (failure instanceof BetPlacementException verdict) {
+      return verdict;
+    }
+    return new DependencyUnavailableException("Risk circuit is unavailable", failure);
+  }
+
   public enum ReservationState {
     RESERVED,
     COMMITTED


## `test(risk): verify lifecycle circuit coverage`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskClientResilienceTest.java b/src/test/java/com/sportsbook/betting/client/RiskClientResilienceTest.java
new file mode 100644
index 0000000..53f578f
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/client/RiskClientResilienceTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.betting.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Money;
+import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
+import java.lang.reflect.Method;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskClientResilienceTest {
+
+  @Test
+  void protectsEveryRiskLifecycleCall() throws NoSuchMethodException {
+    assertFallback(
+        RiskClient.class.getMethod("reserve", UUID.class, UUID.class, Money.class, List.class),
+        "reserveFallback");
+    assertFallback(
+        RiskClient.class.getMethod("commit", UUID.class, String.class), "commitFallback");
+    assertFallback(RiskClient.class.getMethod("release", UUID.class), "releaseFallback");
+  }
+
+  private void assertFallback(Method method, String expected) {
+    CircuitBreaker breaker = method.getAnnotation(CircuitBreaker.class);
+    assertThat(breaker).isNotNull();
+    assertThat(breaker.name()).isEqualTo("riskClient");
+    assertThat(breaker.fallbackMethod()).isEqualTo(expected);
+  }
+}


## `feat(database): persist risk reservation tokens`

diff --git a/src/main/resources/db/migration/V7__risk_reservation_token.sql b/src/main/resources/db/migration/V7__risk_reservation_token.sql
new file mode 100644
index 0000000..4b4b4d6
--- /dev/null
+++ b/src/main/resources/db/migration/V7__risk_reservation_token.sql
@@ -0,0 +1,12 @@
+-- Persist the opaque lease proof before any wallet side effect.
+ALTER TABLE bet
+    ADD COLUMN risk_reservation_token VARCHAR(64);
+
+ALTER TABLE bet
+    ADD CONSTRAINT bet_risk_reservation_token_valid CHECK (
+        risk_reservation_token IS NULL
+        OR risk_reservation_token ~ '^[0-9a-f]{64}$'
+    );
+
+COMMENT ON COLUMN bet.risk_reservation_token IS
+    'Opaque token required to commit the retained risk reservation; null only before reserve or for legacy rows.';


## `test(database): verify risk token constraint`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index dc5db21..b368a85 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -47,6 +47,24 @@ class MigrationContractTest {
         .isEqualTo("1625d9d3140aa8f1888bd8b61dd9d2d00ba612d9c2c5ce35b8d15620368e8e67");
   }
 
+  @Test
+  void requiresCanonicalPersistedRiskToken() {
+    assertThat(migrationText("V7__risk_reservation_token.sql"))
+        .contains("risk_reservation_token VARCHAR(64)")
+        .contains("^[0-9a-f]{64}$");
+  }
+
+  private String migrationText(String migration) {
+    try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
+      if (input == null) {
+        throw new IllegalStateException("Missing migration " + migration);
+      }
+      return new String(input.readAllBytes(), java.nio.charset.StandardCharsets.UTF_8);
+    } catch (IOException exception) {
+      throw new IllegalStateException(exception);
+    }
+  }
+
   private String sha256(String migration) {
     try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {


## `feat(risk): require explicit reservation approval`

diff --git a/src/main/java/com/sportsbook/betting/client/RiskClient.java b/src/main/java/com/sportsbook/betting/client/RiskClient.java
index c220526..cb11960 100644
--- a/src/main/java/com/sportsbook/betting/client/RiskClient.java
+++ b/src/main/java/com/sportsbook/betting/client/RiskClient.java
@@ -125,6 +125,9 @@ public class RiskClient {
     if (response == null) {
       throw new DependencyUnavailableException("Risk returned an empty response");
     }
+    if (response.approved() == null) {
+      throw new DependencyUnavailableException("Risk omitted its approval verdict");
+    }
     if (!response.approved()) {
       throw new RiskLimitException(response.rejectionReason());
     }
diff --git a/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java b/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java
index e953820..1ea3eda 100644
--- a/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java
+++ b/src/main/java/com/sportsbook/betting/client/RiskReservationResponse.java
@@ -11,7 +11,7 @@ import org.springframework.http.client.ClientHttpResponse;
 
 @JsonIgnoreProperties(ignoreUnknown = true)
 public record RiskReservationResponse(
-    boolean approved,
+    Boolean approved,
     boolean replayed,
     String rejectionReason,
     String reservationState,


## `test(risk): reject missing reservation approval`

diff --git a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
index 14d853f..753efce 100644
--- a/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
+++ b/src/test/java/com/sportsbook/betting/client/RiskClientTest.java
@@ -73,6 +73,17 @@ class RiskClientTest {
         .hasMessage("daily limit");
   }
 
+  @Test
+  void rejectsAReservationResponseWithoutAnApprovalVerdict() {
+    server
+        .expect(requestTo("http://risk/internal/v1/risk/reservations"))
+        .andRespond(withSuccess("{}", MediaType.APPLICATION_JSON));
+
+    assertThatThrownBy(this::reserve)
+        .isInstanceOf(DependencyUnavailableException.class)
+        .hasMessage("Risk omitted its approval verdict");
+  }
+
   @Test
   void acceptsOnlyHttp200OrTheFixedValidationVerdict() {
     server
