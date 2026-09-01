## `feat(events): execute accepted projections`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
index 69be2ef..eec0d7b 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
@@ -7,6 +7,7 @@ import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.policy.RiskPatternProperties;
 import com.sportsbook.risk.service.RiskCheckCommand;
 import java.time.Instant;
+import java.util.List;
 import java.util.Objects;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.data.redis.core.script.RedisScript;
@@ -19,6 +20,8 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
       RedisLuaScriptLoader.stringScript("risk-reserve.lua");
   private static final RedisScript<String> COMMIT =
       RedisLuaScriptLoader.stringScript("risk-commit.lua");
+  private static final RedisScript<String> PROJECT_ACCEPTED =
+      RedisLuaScriptLoader.stringScript("risk-project-accepted.lua");
   private static final RedisScript<String> RELEASE =
       RedisLuaScriptLoader.stringScript("risk-release.lua");
 
@@ -56,19 +59,27 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
   public ReservationTransition commit(BetId betId, String token, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.commit(betId, token, now, reservations, patterns, history);
-    return executeTransition(COMMIT, request, "commit");
+    return executeTransition(COMMIT, request.keys(), request.arguments(), "commit");
+  }
+
+  @Override
+  public ReservationTransition projectAccepted(RiskCheckCommand command, String fingerprint) {
+    AcceptedProjectionRequest request =
+        AcceptedProjectionRequest.from(command, fingerprint, reservations, patterns, history);
+    return executeTransition(
+        PROJECT_ACCEPTED, request.keys(), request.arguments(), "accepted projection");
   }
 
   @Override
   public ReservationTransition release(BetId betId, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.release(betId, now, reservations);
-    return executeTransition(RELEASE, request, "release");
+    return executeTransition(RELEASE, request.keys(), request.arguments(), "release");
   }
 
   private ReservationTransition executeTransition(
-      RedisScript<String> script, ReservationTransitionRequest request, String operation) {
-    String raw = redis.execute(script, request.keys(), request.arguments().toArray());
+      RedisScript<String> script, List<String> keys, List<String> arguments, String operation) {
+    String raw = redis.execute(script, keys, arguments.toArray());
     if (raw == null) {
       throw new IllegalStateException("Redis " + operation + " script returned no result");
     }
diff --git a/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
index 4d91954..fb652c1 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
@@ -10,5 +10,7 @@ public interface RiskReservationStore {
 
   ReservationTransition commit(BetId betId, String token, Instant now);
 
+  ReservationTransition projectAccepted(RiskCheckCommand command, String fingerprint);
+
   ReservationTransition release(BetId betId, Instant now);
 }


## `test(events): verify accepted projection storage`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
index bf0a966..5cbf4e0 100644
--- a/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/RedisRiskReservationStoreTest.java
@@ -37,6 +37,18 @@ class RedisRiskReservationStoreTest extends RedisTestSupport {
         .isEqualTo(ReservationTransition.APPLIED);
   }
 
+  @Test
+  void executesAcceptedProjectionThroughTheTypedPort() {
+    RiskReservationStore store = store();
+    RiskCheckCommand accepted = command(3);
+    String fingerprint = ReservationFingerprint.of(accepted);
+
+    assertThat(store.projectAccepted(accepted, fingerprint))
+        .isEqualTo(ReservationTransition.APPLIED);
+    assertThat(store.projectAccepted(accepted, fingerprint))
+        .isEqualTo(ReservationTransition.REPLAYED);
+  }
+
   private RiskReservationStore store() {
     return new RedisRiskReservationStore(
         redis,


## `feat(events): bind accepted identities to admission`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 22ab10f..77b5271 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -65,11 +65,16 @@ for index = 1, selectionCount do
   end
   seen[selectionId] = true; table.insert(selections, selectionId)
 end
+local acceptedKey = "risk:event:fingerprint:" .. betId
 local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
   or typeError(KEYS[3], "zset") or typeError(KEYS[4], "string")
   or typeError(KEYS[5], "zset") or typeError(KEYS[6], "string")
   or typeError(KEYS[7], "hash") or typeError(KEYS[18], "string")
+  or typeError(acceptedKey, "string")
 if errorText then return redis.error_reply(errorText) end
+if redis.call("EXISTS", acceptedKey) == 1 then
+  return response({status = "CONFLICT", replayed = false})
+end
 
 local activeBase, cleanups, stakeDecrements =
   "risk:reservations:user:{" .. userId .. "}", {}, {}


## `test(events): prevent accepted identity readmission`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java
new file mode 100644
index 0000000..2474f7d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class AcceptedReservationBoundaryScriptTest extends RedisTestSupport {
+  private static final RedisScript<String> PROJECT =
+      RedisLuaScriptLoader.stringScript("risk-project-accepted.lua");
+  private static final RedisScript<String> RESERVE =
+      RedisLuaScriptLoader.stringScript("risk-reserve.lua");
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 10));
+  private static final RiskCheckCommand COMMAND =
+      new RiskCheckCommand(
+          USER,
+          BET,
+          Money.krw(50),
+          List.of(SelectionId.of(new UUID(0, 20))),
+          Instant.ofEpochMilli(2_000_000));
+
+  @Test
+  void acceptedProjectionPreventsReservationReadmission() {
+    var reservations = new RiskReservationProperties(null, null);
+    var patterns = new RiskPatternProperties(null, null, null);
+    var history = new RiskHistoryProperties(null, 0);
+    String fingerprint = ReservationFingerprint.of(COMMAND);
+    AcceptedProjectionRequest projection =
+        AcceptedProjectionRequest.from(COMMAND, fingerprint, reservations, patterns, history);
+    ReservationScriptRequest admission =
+        ReservationScriptRequest.from(
+            COMMAND,
+            new RiskLimitProperties(null, null, null, null, 0),
+            patterns,
+            reservations,
+            history);
+
+    assertThat(redis.execute(PROJECT, projection.keys(), projection.arguments().toArray()))
+        .isEqualTo("APPLIED");
+    ReservationDecision decision =
+        new ReservationWireMapper(new ObjectMapper())
+            .map(redis.execute(RESERVE, admission.keys(), admission.arguments().toArray()))
+            .decision();
+
+    assertThat(decision.status()).isEqualTo(ReservationDecision.Status.CONFLICT);
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(BET))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+    assertThat(
+            redis
+                .opsForValue()
+                .get(LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW).sum()))
+        .isEqualTo("50");
+  }
+}


## `feat(events): reconcile accepted reservations`

diff --git a/src/main/java/com/sportsbook/risk/event/ReservationAcceptedBetReconciler.java b/src/main/java/com/sportsbook/risk/event/ReservationAcceptedBetReconciler.java
new file mode 100644
index 0000000..a0edc31
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/ReservationAcceptedBetReconciler.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.risk.reservation.ReservationTransition;
+import com.sportsbook.risk.reservation.RiskReservationStore;
+import java.util.Objects;
+import org.springframework.stereotype.Component;
+
+/** Confirms reserved bets and atomically projects first-seen accepted bets. */
+@Component
+public final class ReservationAcceptedBetReconciler implements AcceptedBetReconciler {
+  private final RiskReservationStore reservations;
+
+  public ReservationAcceptedBetReconciler(RiskReservationStore reservations) {
+    this.reservations = Objects.requireNonNull(reservations, "reservations");
+  }
+
+  @Override
+  public AcceptedBetReconciliation reconcile(AcceptedBetEnvelope envelope) {
+    Objects.requireNonNull(envelope, "envelope");
+    String fingerprint = envelope.reservationFingerprint();
+    ReservationTransition transition =
+        reservations.commit(envelope.command().betId(), fingerprint, envelope.command().now());
+    if (transition == ReservationTransition.NOT_FOUND) {
+      transition = reservations.projectAccepted(envelope.command(), fingerprint);
+      return switch (transition) {
+        case APPLIED -> AcceptedBetReconciliation.PROJECTED;
+        case REPLAYED -> AcceptedBetReconciliation.REPLAYED;
+        case CONFLICT -> AcceptedBetReconciliation.FINGERPRINT_MISMATCH;
+        default -> throw unexpected(transition);
+      };
+    }
+    return switch (transition) {
+      case APPLIED -> AcceptedBetReconciliation.CONFIRMED;
+      case REPLAYED -> AcceptedBetReconciliation.REPLAYED;
+      case CONFLICT -> AcceptedBetReconciliation.FINGERPRINT_MISMATCH;
+      case EXPIRED, TOMBSTONED -> AcceptedBetReconciliation.TERMINAL_RESERVATION;
+      case NOT_FOUND -> throw unexpected(transition);
+    };
+  }
+
+  private static IllegalStateException unexpected(ReservationTransition transition) {
+    return new IllegalStateException("unexpected accepted projection result: " + transition);
+  }
+}


## `test(events): verify accepted reconciliation outcomes`

diff --git a/src/test/java/com/sportsbook/risk/event/ReservationAcceptedBetReconcilerTest.java b/src/test/java/com/sportsbook/risk/event/ReservationAcceptedBetReconcilerTest.java
new file mode 100644
index 0000000..20a2106
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/ReservationAcceptedBetReconcilerTest.java
@@ -0,0 +1,77 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.reservation.ReservationFingerprint;
+import com.sportsbook.risk.reservation.ReservationTransition;
+import com.sportsbook.risk.reservation.RiskReservationStore;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationAcceptedBetReconcilerTest {
+  private static final Instant NOW = Instant.ofEpochMilli(2_000_000);
+  private final RiskCheckCommand command =
+      new RiskCheckCommand(
+          UserId.of(new UUID(0, 1)),
+          BetId.of(new UUID(0, 2)),
+          new Money(10, Currency.KRW),
+          List.of(SelectionId.of(new UUID(0, 3))),
+          NOW);
+  private final AcceptedBetEnvelope envelope = new AcceptedBetEnvelope(command, NOW);
+  private final String fingerprint = ReservationFingerprint.of(command);
+  private final RiskReservationStore store = mock(RiskReservationStore.class);
+  private final ReservationAcceptedBetReconciler reconciler =
+      new ReservationAcceptedBetReconciler(store);
+
+  @Test
+  void projectsOnlyWhenTheReservationIsNotFound() {
+    when(store.commit(command.betId(), fingerprint, NOW))
+        .thenReturn(ReservationTransition.NOT_FOUND);
+    when(store.projectAccepted(command, fingerprint)).thenReturn(ReservationTransition.APPLIED);
+
+    assertThat(reconciler.reconcile(envelope)).isEqualTo(AcceptedBetReconciliation.PROJECTED);
+    verify(store).projectAccepted(command, fingerprint);
+  }
+
+  @Test
+  void mapsProjectionReplayAndConflict() {
+    when(store.commit(command.betId(), fingerprint, NOW))
+        .thenReturn(ReservationTransition.NOT_FOUND);
+    when(store.projectAccepted(command, fingerprint))
+        .thenReturn(ReservationTransition.REPLAYED, ReservationTransition.CONFLICT);
+
+    assertThat(reconciler.reconcile(envelope)).isEqualTo(AcceptedBetReconciliation.REPLAYED);
+    assertThat(reconciler.reconcile(envelope))
+        .isEqualTo(AcceptedBetReconciliation.FINGERPRINT_MISMATCH);
+  }
+
+  @Test
+  void mapsReservedLifecycleOutcomesWithoutProjection() {
+    when(store.commit(command.betId(), fingerprint, NOW))
+        .thenReturn(
+            ReservationTransition.APPLIED,
+            ReservationTransition.REPLAYED,
+            ReservationTransition.CONFLICT,
+            ReservationTransition.EXPIRED);
+
+    assertThat(reconciler.reconcile(envelope)).isEqualTo(AcceptedBetReconciliation.CONFIRMED);
+    assertThat(reconciler.reconcile(envelope)).isEqualTo(AcceptedBetReconciliation.REPLAYED);
+    assertThat(reconciler.reconcile(envelope))
+        .isEqualTo(AcceptedBetReconciliation.FINGERPRINT_MISMATCH);
+    assertThat(reconciler.reconcile(envelope))
+        .isEqualTo(AcceptedBetReconciliation.TERMINAL_RESERVATION);
+    verify(store, never()).projectAccepted(command, fingerprint);
+  }
+}


## `test(events): verify accepted-bet Redis projection`

diff --git a/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java
new file mode 100644
index 0000000..23a8f83
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java
@@ -0,0 +1,36 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.awaitility.Awaitility.await;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedKafkaRedisIntegrationTest extends BetPlacedKafkaRedisIntegrationSupport {
+  private static final String BET_ID = "20000000-0000-4000-8000-000000000001";
+  private static final String DAILY_BASE =
+      "risk:limit:{" + BetPlacedEventFixture.USER_ID + "}:stake-daily:krw";
+
+  @Test
+  void acceptedBetProjectsOnceAcrossBrokerRedelivery() throws Exception {
+    publishAcceptedBet();
+
+    await()
+        .atMost(Duration.ofSeconds(10))
+        .untilAsserted(
+            () -> {
+              assertThat(redis.opsForValue().get(DAILY_BASE + ":sum")).isEqualTo("10000");
+              assertThat(redis.opsForValue().get("risk:event:fingerprint:" + BET_ID))
+                  .matches("[0-9a-f]{64}");
+              assertThat(committedSourceOffset()).isEqualTo(1L);
+            });
+
+    publishAcceptedBet();
+
+    await()
+        .atMost(Duration.ofSeconds(10))
+        .untilAsserted(() -> assertThat(committedSourceOffset()).isEqualTo(2L));
+    assertThat(redis.opsForValue().get(DAILY_BASE + ":sum")).isEqualTo("10000");
+    assertThat(redis.opsForZSet().size(DAILY_BASE + ":entries")).isEqualTo(1L);
+  }
+}
