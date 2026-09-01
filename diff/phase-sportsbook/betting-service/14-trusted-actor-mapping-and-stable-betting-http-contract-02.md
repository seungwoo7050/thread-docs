## `test(api): verify stable problem responses`

diff --git a/src/test/java/com/sportsbook/betting/api/BetExceptionHandlerTest.java b/src/test/java/com/sportsbook/betting/api/BetExceptionHandlerTest.java
new file mode 100644
index 0000000..4b0b6e5
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/api/BetExceptionHandlerTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.betting.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.error.BetNotFoundException;
+import com.sportsbook.betting.error.InsufficientBalanceException;
+import com.sportsbook.protocol.error.ErrorCode;
+import java.net.URI;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class BetExceptionHandlerTest {
+
+  private final BetExceptionHandler handler = new BetExceptionHandler();
+
+  @Test
+  void preservesSharedPlacementStatusAndErrorCode() {
+    var response =
+        handler.placement(
+            new InsufficientBalanceException("declined"), request("/internal/v1/bets"));
+
+    assertThat(response.getStatusCode().value()).isEqualTo(409);
+    assertThat(response.getBody().errorCode()).isEqualTo(ErrorCode.INSUFFICIENT_BALANCE.name());
+    assertThat(response.getBody().instance()).isEqualTo(URI.create("/internal/v1/bets"));
+  }
+
+  @Test
+  void rendersActorScopedMissingBetsAsNotFound() {
+    var response =
+        handler.missing(new BetNotFoundException("hidden"), request("/internal/v1/bets/unknown"));
+
+    assertThat(response.getStatusCode().value()).isEqualTo(404);
+    assertThat(response.getBody().errorCode()).isEqualTo("BET_NOT_FOUND");
+  }
+
+  @Test
+  void normalizesMalformedWireValuesAsValidationProblems() {
+    var response =
+        handler.invalid(new IllegalArgumentException("bad uuid"), request("/internal/v1/bets"));
+
+    assertThat(response.getStatusCode().value()).isEqualTo(400);
+    assertThat(response.getBody().errorCode()).isEqualTo(ErrorCode.VALIDATION_FAILED.name());
+    assertThat(response.getBody().detail()).isEqualTo("Request validation failed");
+    assertThat(response.getBody().detail()).doesNotContain("bad uuid");
+  }
+
+  private static MockHttpServletRequest request(String path) {
+    return new MockHttpServletRequest("POST", path);
+  }
+}


## `feat(api): expose terminal resolution snapshot`

diff --git a/src/main/java/com/sportsbook/betting/api/BetResponse.java b/src/main/java/com/sportsbook/betting/api/BetResponse.java
index db6bb21..5898ee9 100644
--- a/src/main/java/com/sportsbook/betting/api/BetResponse.java
+++ b/src/main/java/com/sportsbook/betting/api/BetResponse.java
@@ -19,6 +19,7 @@ public record BetResponse(
     Money maxPayout,
     List<SelectionView> selections,
     String rejectionReason,
+    ResolutionView resolution,
     Instant createdAt) {
 
   public record SlipTypeView(String type, Integer minWins, Integer totalSelections) {}
@@ -26,6 +27,16 @@ public record BetResponse(
   public record SelectionView(
       UUID eventId, UUID marketId, UUID selectionId, String oddsAtSubmission) {}
 
+  @JsonInclude(JsonInclude.Include.NON_NULL)
+  public record ResolutionView(
+      String settlementResult,
+      Money settledPayout,
+      String voidReason,
+      Instant resolvedAt,
+      UUID resolutionEventId,
+      UUID resolutionRevisionId,
+      Long resolutionRevisionNumber) {}
+
   public static BetResponse from(Bet bet) {
     BetSlipType type = bet.slipType();
     SlipTypeView slip =
@@ -43,6 +54,17 @@ public record BetResponse(
                         leg.selectionId(),
                         leg.oddsAtSubmission().decimal().toPlainString()))
             .toList();
+    ResolutionView resolution =
+        bet.resolvedAt() == null
+            ? null
+            : new ResolutionView(
+                bet.settlementResult() == null ? null : bet.settlementResult().name(),
+                bet.settledPayout(),
+                bet.voidReason() == null ? null : bet.voidReason().name(),
+                bet.resolvedAt(),
+                bet.resolutionEventId(),
+                bet.resolutionRevisionId(),
+                Math.max(0L, bet.resolutionRevisionNumber()));
     return new BetResponse(
         bet.betId(),
         bet.betReference(),
@@ -53,6 +75,7 @@ public record BetResponse(
         bet.maxPayout(),
         selections,
         bet.rejectionReason(),
+        resolution,
         bet.createdAt());
   }
 }
diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 5d051ad..a266166 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -582,6 +582,10 @@ public class Bet {
     return resolvedAt;
   }
 
+  public UUID resolutionEventId() {
+    return resolutionEventId;
+  }
+
   public long resolutionRevisionNumber() {
     return resolutionRevisionNumber == null ? -1 : resolutionRevisionNumber;
   }


## `test(api): verify base resolution response`

diff --git a/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java b/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java
new file mode 100644
index 0000000..ffbc0f4
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java
@@ -0,0 +1,78 @@
+package com.sportsbook.betting.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.test.util.ReflectionTestUtils;
+
+class BetResponseResolutionTest {
+
+  private static final Instant NOW = Instant.parse("2026-08-22T00:00:00Z");
+
+  @Test
+  void omitsResolutionBeforeAResult() throws Exception {
+    BetResponse response = BetResponse.from(accepted(UUID.randomUUID()));
+
+    assertThat(response.resolution()).isNull();
+    assertThat(new ObjectMapper().findAndRegisterModules().writeValueAsString(response))
+        .doesNotContain("\"resolution\"");
+  }
+
+  @Test
+  void exposesBaseSettlementAsLogicalRevisionZero() {
+    UUID eventId = UUID.randomUUID();
+    Bet bet = accepted(eventId);
+    bet.settleBase(eventId, SettlementResult.WON, Money.krw(1_000), Money.krw(2_000), NOW, hash());
+
+    assertThat(BetResponse.from(bet).resolution())
+        .isEqualTo(
+            new BetResponse.ResolutionView("WON", Money.krw(2_000), null, NOW, eventId, null, 0L));
+  }
+
+  @Test
+  void normalizesLegacyTerminalRowsWithoutRevisionProofToZero() {
+    UUID eventId = UUID.randomUUID();
+    Bet bet = accepted(eventId);
+    bet.settleBase(eventId, SettlementResult.WON, Money.krw(1_000), Money.krw(2_000), NOW, hash());
+    ReflectionTestUtils.setField(bet, "resolutionRevisionNumber", null);
+
+    assertThat(BetResponse.from(bet).resolution().resolutionRevisionNumber()).isZero();
+  }
+  private static Bet accepted(UUID eventId) {
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                UUID.randomUUID(),
+                UUID.randomUUID(),
+                "B-2026-08-22-00000001",
+                new BetSlipType.Single(),
+                Money.krw(1_000),
+                Money.krw(2_000),
+                IdempotencyKey.of("response-resolution"),
+                hash(),
+                NOW),
+            List.of(
+                BetLeg.create(eventId, UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"))));
+    bet.recordRiskReservation(NOW.plusSeconds(60), hash(), true, NOW);
+    bet.confirmWallet(UUID.randomUUID(), NOW);
+    bet.commitRisk(NOW);
+    bet.accept(NOW);
+    return bet;
+  }
+
+  private static String hash() {
+    return "a".repeat(64);
+  }
+}


## `test(api): verify revised and void responses`

diff --git a/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java b/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java
index ffbc0f4..ea63096 100644
--- a/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java
+++ b/src/test/java/com/sportsbook/betting/api/BetResponseResolutionTest.java
@@ -6,6 +6,7 @@ import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetDraft;
 import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.VoidReason;
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.value.IdempotencyKey;
@@ -50,6 +51,41 @@ class BetResponseResolutionTest {
 
     assertThat(BetResponse.from(bet).resolution().resolutionRevisionNumber()).isZero();
   }
+
+  @Test
+  void exposesTheCurrentCorrectionWhenRevisionArrivesBeforeBase() {
+    UUID eventId = UUID.randomUUID();
+    UUID revisionId = UUID.randomUUID();
+    Bet bet = accepted(eventId);
+    bet.applyRevision(
+        eventId,
+        revisionId,
+        1,
+        SettlementResult.LOST,
+        SettlementResult.WON,
+        Money.krw(0),
+        Money.krw(2_000),
+        NOW,
+        NOW.plusSeconds(1),
+        hash());
+
+    assertThat(BetResponse.from(bet).resolution())
+        .isEqualTo(
+            new BetResponse.ResolutionView(
+                "WON", Money.krw(2_000), null, NOW.plusSeconds(1), eventId, revisionId, 1L));
+  }
+
+  @Test
+  void exposesVoidWithoutInventingARefund() {
+    UUID eventId = UUID.randomUUID();
+    Bet bet = accepted(eventId);
+    bet.voidBase(eventId, VoidReason.EVENT_CANCELLED, NOW, hash());
+
+    assertThat(BetResponse.from(bet).resolution())
+        .isEqualTo(
+            new BetResponse.ResolutionView(null, null, "EVENT_CANCELLED", NOW, eventId, null, 0L));
+  }
+
   private static Bet accepted(UUID eventId) {
     Bet bet =
         Bet.pending(
