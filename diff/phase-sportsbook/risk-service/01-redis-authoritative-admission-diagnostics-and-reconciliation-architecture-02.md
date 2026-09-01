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


