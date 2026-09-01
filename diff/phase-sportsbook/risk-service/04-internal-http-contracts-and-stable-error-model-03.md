## `feat(api): expose reservation transitions`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskReservationController.java b/src/main/java/com/sportsbook/risk/api/RiskReservationController.java
index 2fab0b4..f2f7fea 100644
--- a/src/main/java/com/sportsbook/risk/api/RiskReservationController.java
+++ b/src/main/java/com/sportsbook/risk/api/RiskReservationController.java
@@ -1,26 +1,51 @@
 package com.sportsbook.risk.api;
 
+import com.sportsbook.protocol.value.BetId;
 import com.sportsbook.risk.reservation.ReservationDecision;
+import com.sportsbook.risk.reservation.ReservationTransition;
 import com.sportsbook.risk.reservation.RiskReservationService;
 import com.sportsbook.risk.service.RiskCheckCommand;
 import jakarta.validation.Valid;
 import java.time.Clock;
+import java.time.Instant;
+import java.util.UUID;
 import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.DeleteMapping;
+import org.springframework.web.bind.annotation.PathVariable;
 import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.PutMapping;
 import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestHeader;
 import org.springframework.web.bind.annotation.RequestMapping;
 import org.springframework.web.bind.annotation.RestController;
 
-/** Betting-owned atomic admission API. */
+/** Betting-owned atomic admission and reservation lifecycle API. */
 @RestController
 @RequestMapping("/internal/v1/risk/reservations")
 public class RiskReservationController {
+  public static final String TOKEN_HEADER = "X-Risk-Reservation-Token";
+
   private final Operations operations;
   private final Clock clock;
 
   @Autowired
   public RiskReservationController(RiskReservationService service, Clock clock) {
-    this(service::reserve, clock);
+    this(
+        new Operations() {
+          public ReservationDecision reserve(RiskCheckCommand command) {
+            return service.reserve(command);
+          }
+
+          public ReservationTransition commit(BetId betId, String token, Instant at) {
+            return service.commit(betId, token, at);
+          }
+
+          public ReservationTransition release(BetId betId, Instant at) {
+            return service.release(betId, at);
+          }
+        },
+        clock);
   }
 
   RiskReservationController(Operations operations, Clock clock) {
@@ -37,8 +62,37 @@ public class RiskReservationController {
     return RiskReservationResponse.from(decision);
   }
 
-  @FunctionalInterface
+  @PutMapping("/{betId}/commit")
+  public ResponseEntity<Void> commit(
+      @PathVariable UUID betId, @RequestHeader(TOKEN_HEADER) String reservationToken) {
+    BetId typedBetId = BetId.of(betId);
+    if (!reservationToken.matches("[0-9a-f]{64}")) {
+      throw RiskApiException.validation("reservation token must be lowercase SHA-256 hex");
+    }
+    ReservationTransition result = operations.commit(typedBetId, reservationToken, clock.instant());
+    switch (result) {
+      case APPLIED, REPLAYED -> {}
+      case NOT_FOUND, EXPIRED, TOMBSTONED -> throw RiskApiException.notFound(typedBetId);
+      case CONFLICT -> throw RiskApiException.duplicate(typedBetId);
+    }
+    return ResponseEntity.noContent().build();
+  }
+
+  @DeleteMapping("/{betId}")
+  public ResponseEntity<Void> release(@PathVariable UUID betId) {
+    BetId typedBetId = BetId.of(betId);
+    ReservationTransition result = operations.release(typedBetId, clock.instant());
+    if (result == ReservationTransition.CONFLICT) {
+      throw RiskApiException.committed(typedBetId);
+    }
+    return ResponseEntity.noContent().build();
+  }
+
   interface Operations {
     ReservationDecision reserve(RiskCheckCommand command);
+
+    ReservationTransition commit(BetId betId, String token, Instant at);
+
+    ReservationTransition release(BetId betId, Instant at);
   }
 }


## `test(api): verify reservation transition contracts`

diff --git a/src/test/java/com/sportsbook/risk/api/RiskReservationTransitionTest.java b/src/test/java/com/sportsbook/risk/api/RiskReservationTransitionTest.java
new file mode 100644
index 0000000..f2a8a0f
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/RiskReservationTransitionTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.risk.api;
+
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.risk.reservation.ReservationTransition;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+@ExtendWith(MockitoExtension.class)
+class RiskReservationTransitionTest {
+  private static final Instant NOW = Instant.parse("2026-08-21T10:00:00Z");
+  private static final UUID BET_VALUE = UUID.fromString("00000000-0000-0000-0000-000000000002");
+  private static final BetId BET = BetId.of(BET_VALUE);
+  private static final String TOKEN = "a".repeat(64);
+  private static final String OTHER_TOKEN = "b".repeat(64);
+
+  @Mock private RiskReservationController.Operations operations;
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    var controller = new RiskReservationController(operations, Clock.fixed(NOW, ZoneOffset.UTC));
+    mvc =
+        MockMvcBuilders.standaloneSetup(controller)
+            .setControllerAdvice(new RestExceptionHandler())
+            .build();
+  }
+
+  @Test
+  void commitRequiresAndForwardsTheOpaqueToken() throws Exception {
+    when(operations.commit(BET, TOKEN, NOW)).thenReturn(ReservationTransition.APPLIED);
+    mvc.perform(put(path("/commit")).header(RiskReservationController.TOKEN_HEADER, TOKEN))
+        .andExpect(status().isNoContent());
+    verify(operations).commit(BET, TOKEN, NOW);
+
+    mvc.perform(put(path("/commit")))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    mvc.perform(put(path("/commit")).header(RiskReservationController.TOKEN_HEADER, " "))
+        .andExpect(status().isBadRequest());
+    mvc.perform(put(path("/commit")).header(RiskReservationController.TOKEN_HEADER, "opaque-token"))
+        .andExpect(status().isBadRequest());
+  }
+
+  @Test
+  void commitDistinguishesMissingAndConflictingReservations() throws Exception {
+    when(operations.commit(eq(BET), eq(TOKEN), eq(NOW))).thenReturn(ReservationTransition.EXPIRED);
+    when(operations.commit(eq(BET), eq(OTHER_TOKEN), eq(NOW)))
+        .thenReturn(ReservationTransition.CONFLICT);
+    mvc.perform(put(path("/commit")).header(RiskReservationController.TOKEN_HEADER, TOKEN))
+        .andExpect(status().isNotFound())
+        .andExpect(jsonPath("$.errorCode").value("RISK_RESERVATION_NOT_FOUND"));
+    mvc.perform(put(path("/commit")).header(RiskReservationController.TOKEN_HEADER, OTHER_TOKEN))
+        .andExpect(status().isConflict())
+        .andExpect(jsonPath("$.errorCode").value("DUPLICATE_BET"));
+  }
+
+  @Test
+  void releaseIsIdempotentUntilTheReservationIsCommitted() throws Exception {
+    when(operations.release(BET, NOW))
+        .thenReturn(ReservationTransition.NOT_FOUND, ReservationTransition.CONFLICT);
+    mvc.perform(delete(path(""))).andExpect(status().isNoContent());
+    mvc.perform(delete(path("")))
+        .andExpect(status().isConflict())
+        .andExpect(jsonPath("$.errorCode").value("RISK_RESERVATION_COMMITTED"));
+  }
+
+  private String path(String suffix) {
+    return "/internal/v1/risk/reservations/" + BET_VALUE + suffix;
+  }
+}
