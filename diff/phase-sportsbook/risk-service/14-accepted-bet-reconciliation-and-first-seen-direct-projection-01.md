# 수락 베팅 재조정과 최초 관측 직접 투영

## `feat(events): define accepted bet reconciliation outcomes`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java
new file mode 100644
index 0000000..b058b25
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciler.java
@@ -0,0 +1,7 @@
+package com.sportsbook.risk.event;
+
+/** Atomic boundary for reservation confirmation or first-seen accepted-bet projection. */
+@FunctionalInterface
+public interface AcceptedBetReconciler {
+  AcceptedBetReconciliation reconcile(AcceptedBetEnvelope envelope);
+}
diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java
new file mode 100644
index 0000000..9106969
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetReconciliation.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.event;
+
+/** Atomic result of reconciling an accepted event with reservation and committed state. */
+public enum AcceptedBetReconciliation {
+  CONFIRMED(null),
+  PROJECTED(null),
+  REPLAYED(null),
+  FINGERPRINT_MISMATCH(BetPlacedFailureReason.FINGERPRINT_MISMATCH),
+  TERMINAL_RESERVATION(BetPlacedFailureReason.TERMINAL_RESERVATION);
+
+  private final BetPlacedFailureReason failureReason;
+
+  AcceptedBetReconciliation(BetPlacedFailureReason failureReason) {
+    this.failureReason = failureReason;
+  }
+
+  public boolean permanentFailure() {
+    return failureReason != null;
+  }
+
+  public BetPlacedFailureReason failureReason() {
+    if (failureReason == null) {
+      throw new IllegalStateException("successful reconciliation has no failure reason");
+    }
+    return failureReason;
+  }
+}


## `test(events): verify reconciliation outcome semantics`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java
new file mode 100644
index 0000000..b122182
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetReconciliationTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class AcceptedBetReconciliationTest {
+  @Test
+  void exposesOnlyPermanentReservationFailuresAsDeadLetterReasons() {
+    assertThat(AcceptedBetReconciliation.CONFIRMED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.PROJECTED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.REPLAYED.permanentFailure()).isFalse();
+    assertThat(AcceptedBetReconciliation.FINGERPRINT_MISMATCH.failureReason())
+        .isEqualTo(BetPlacedFailureReason.FINGERPRINT_MISMATCH);
+    assertThat(AcceptedBetReconciliation.TERMINAL_RESERVATION.failureReason())
+        .isEqualTo(BetPlacedFailureReason.TERMINAL_RESERVATION);
+    assertThatThrownBy(AcceptedBetReconciliation.CONFIRMED::failureReason)
+        .isInstanceOf(IllegalStateException.class);
+  }
+}


## `feat(events): retain accepted projection identities`

diff --git a/src/main/resources/scripts/risk-project-accepted.lua b/src/main/resources/scripts/risk-project-accepted.lua
new file mode 100644
index 0000000..c516879
--- /dev/null
+++ b/src/main/resources/scripts/risk-project-accepted.lua
@@ -0,0 +1,82 @@
+local maxExact = 9007199254740991
+local now, retention = tonumber(ARGV[2]), tonumber(ARGV[3])
+local fingerprint, userId, betId = ARGV[4], ARGV[5], ARGV[6]
+local stake, currency, selectionCount = tonumber(ARGV[7]), ARGV[8], tonumber(ARGV[9])
+
+local function exact(text, positive)
+  if not text or not string.match(text, "^%d+$") then return nil end
+  local value = tonumber(text)
+  if not value or value > maxExact or (positive and value <= 0) then return nil end
+  return value
+end
+local function keyType(key) return redis.call("TYPE", key).ok end
+local function typeError(key, expected)
+  local actual = keyType(key)
+  if actual ~= "none" and actual ~= expected then
+    return "wrong Redis type for " .. key
+  end
+end
+local function split(encoded, expected)
+  local values = {}
+  for value in string.gmatch(encoded or "", "[^,]+") do table.insert(values, value) end
+  if #values == expected then return values end
+end
+
+if ARGV[1] ~= "1" or #KEYS ~= 2 or #ARGV ~= 18
+  or not exact(ARGV[2], false) or not exact(ARGV[3], true)
+  or not fingerprint or not string.match(fingerprint, "^[0-9a-f]+$") or #fingerprint ~= 64
+  or not userId or not betId or not string.match(userId, "^[0-9a-f%-]+$")
+  or not string.match(betId, "^[0-9a-f%-]+$") or not exact(ARGV[7], true)
+  or not currency or not string.match(currency, "^[A-Z]+$")
+  or not exact(ARGV[9], true) or not split(ARGV[10], selectionCount or -1) then
+  return redis.error_reply("invalid accepted projection request")
+end
+for index = 11, 18 do
+  if not exact(ARGV[index], true) then return redis.error_reply("invalid projection policy") end
+end
+
+if keyType(KEYS[1]) ~= "none" then
+  return redis.error_reply("reservation lifecycle appeared before accepted projection")
+end
+local markerError = typeError(KEYS[2], "string")
+if markerError then return redis.error_reply(markerError) end
+local retained = redis.call("GET", KEYS[2])
+if retained then
+  if retained == fingerprint then return "REPLAYED" end
+  return "CONFLICT"
+end
+
+local selections = split(ARGV[10], selectionCount)
+local function memberAmount(member)
+  return exact(string.match(member, "|([0-9]+)$"), true)
+end
+local function planWindow(entries, sum, amount, window)
+  local errorText = typeError(entries, "zset") or typeError(sum, "string")
+  if errorText then return nil, errorText end
+  local total = 0
+  if keyType(entries) ~= "none" then
+    total = exact(redis.call("GET", sum), false)
+    if not total then return nil, "missing or corrupt rolling sum" end
+    local calculated = 0
+    for _, member in ipairs(redis.call("ZRANGE", entries, 0, -1)) do
+      local amount = memberAmount(member)
+      if not amount or calculated > maxExact - amount then
+        return nil, "corrupt rolling member"
+      end
+      calculated = calculated + amount
+    end
+    if calculated ~= total then return nil, "inconsistent rolling sum" end
+  end
+  local expired = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window)
+  for _, member in ipairs(expired) do
+    local amount = memberAmount(member)
+    if not amount or total < amount then return nil, "corrupt rolling member" end
+    total = total - amount
+  end
+  if total > maxExact - amount then return nil, "rolling sum exceeds exact range" end
+  local amountText = string.format("%.0f", amount)
+  if redis.call("ZSCORE", entries, betId .. "|" .. amountText) then
+    return nil, "accepted projection already exists in rolling window"
+  end
+  return {entries, sum, window, amountText, total + amount}
+end


## `test(events): verify accepted projection identities`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
new file mode 100644
index 0000000..cb9ebc7
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.RedisSystemException;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class AcceptedProjectionIdentityScriptTest extends RedisTestSupport {
+  private static final RedisScript<String> SCRIPT =
+      RedisLuaScriptLoader.stringScript("risk-project-accepted.lua");
+  private static final String FINGERPRINT = "a".repeat(64);
+
+  @Test
+  void replaysOnlyTheRetainedFingerprint() {
+    redis.opsForValue().set("accepted", FINGERPRINT);
+
+    assertThat(execute(FINGERPRINT)).isEqualTo("REPLAYED");
+    assertThat(execute("b".repeat(64))).isEqualTo("CONFLICT");
+  }
+
+  @Test
+  void rejectsAnExistingReservationLifecycle() {
+    redis.opsForHash().put("lifecycle", "state", "RESERVED");
+
+    assertThatThrownBy(() -> execute(FINGERPRINT))
+        .isInstanceOf(RedisSystemException.class)
+        .rootCause()
+        .hasMessageContaining("reservation lifecycle appeared");
+    assertThat(redis.hasKey("accepted")).isFalse();
+  }
+
+  private String execute(String fingerprint) {
+    return redis.execute(SCRIPT, List.of("lifecycle", "accepted"), arguments(fingerprint));
+  }
+
+  private static Object[] arguments(String fingerprint) {
+    return new Object[] {
+      "1",
+      "2000000",
+      "2764800000",
+      fingerprint,
+      "00000000-0000-0000-0000-000000000001",
+      "00000000-0000-0000-0000-000000000010",
+      "50",
+      "KRW",
+      "1",
+      "00000000-0000-0000-0000-000000000020",
+      "86400000",
+      "604800000",
+      "2592000000",
+      "60000",
+      "60000",
+      "86400000",
+      "600000",
+      "100"
+    };
+  }
+}


## `feat(events): project accepted risk capacity`

diff --git a/src/main/resources/scripts/risk-project-accepted.lua b/src/main/resources/scripts/risk-project-accepted.lua
index c516879..02703cb 100644
--- a/src/main/resources/scripts/risk-project-accepted.lua
+++ b/src/main/resources/scripts/risk-project-accepted.lua
@@ -80,3 +80,44 @@ local function planWindow(entries, sum, amount, window)
   end
   return {entries, sum, window, amountText, total + amount}
 end
+
+local limitBase, plans = "risk:limit:{" .. userId .. "}:", {}
+local currencyLower = string.lower(currency)
+local dimensions = {
+  {"stake-daily:" .. currencyLower, ARGV[11], stake},
+  {"stake-weekly:" .. currencyLower, ARGV[12], stake},
+  {"stake-monthly:" .. currencyLower, ARGV[13], stake},
+  {"selections-per-minute", ARGV[14], selectionCount}
+}
+for _, dimension in ipairs(dimensions) do
+  local prefix = limitBase .. dimension[1]
+  local plan, planError =
+    planWindow(prefix .. ":entries", prefix .. ":sum", dimension[3], tonumber(dimension[2]))
+  if planError then return redis.error_reply(planError) end
+  table.insert(plans, plan)
+end
+
+local historyBase = "risk:history:{" .. userId .. "}"
+local historyBets = historyBase .. ":bets"
+local historyStakes = historyBase .. ":stakes:" .. currencyLower
+local historyError = typeError(historyBets, "zset") or typeError(historyStakes, "zset")
+if historyError then return redis.error_reply(historyError) end
+if redis.call("ZSCORE", historyBets, betId)
+  or redis.call("ZSCORE", historyStakes, betId .. "|" .. ARGV[7]) then
+  return redis.error_reply("accepted projection already exists without retained marker")
+end
+for _, selectionId in ipairs(selections) do
+  local selectionKey = historyBase .. ":selection:" .. selectionId
+  local itemError = typeError(selectionKey, "zset")
+  if itemError then return redis.error_reply(itemError) end
+  if redis.call("ZSCORE", selectionKey, betId) then
+    return redis.error_reply("accepted selection projection exists without retained marker")
+  end
+end
+
+for _, plan in ipairs(plans) do
+  redis.call("ZREMRANGEBYSCORE", plan[1], "-inf", now - plan[3])
+  redis.call("ZADD", plan[1], now, betId .. "|" .. plan[4])
+  redis.call("SET", plan[2], string.format("%.0f", plan[5]), "PX", plan[3] + 300000)
+  redis.call("PEXPIRE", plan[1], plan[3] + 300000)
+end


## `test(events): verify accepted risk capacity`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
index cb9ebc7..087daab 100644
--- a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
@@ -3,9 +3,14 @@ package com.sportsbook.risk.reservation;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.counter.RedisLuaScriptLoader;
 import com.sportsbook.risk.support.RedisTestSupport;
 import java.util.List;
+import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.data.redis.RedisSystemException;
 import org.springframework.data.redis.core.script.RedisScript;
@@ -34,6 +39,19 @@ class AcceptedProjectionIdentityScriptTest extends RedisTestSupport {
     assertThat(redis.hasKey("accepted")).isFalse();
   }
 
+  @Test
+  void projectsCurrencyScopedCapacity() {
+    execute(FINGERPRINT);
+    UserId userId = UserId.of(new UUID(0, 1));
+
+    assertThat(
+            redis
+                .opsForValue()
+                .get(LimitKeys.monetary(userId, LimitType.STAKE_DAILY, Currency.KRW).sum()))
+        .isEqualTo("50");
+    assertThat(redis.opsForValue().get(LimitKeys.selections(userId).sum())).isEqualTo("1");
+  }
+
   private String execute(String fingerprint) {
     return redis.execute(SCRIPT, List.of("lifecycle", "accepted"), arguments(fingerprint));
   }


## `feat(events): project accepted pattern history`

diff --git a/src/main/resources/scripts/risk-project-accepted.lua b/src/main/resources/scripts/risk-project-accepted.lua
index 02703cb..d44275c 100644
--- a/src/main/resources/scripts/risk-project-accepted.lua
+++ b/src/main/resources/scripts/risk-project-accepted.lua
@@ -121,3 +121,23 @@ for _, plan in ipairs(plans) do
   redis.call("SET", plan[2], string.format("%.0f", plan[5]), "PX", plan[3] + 300000)
   redis.call("PEXPIRE", plan[1], plan[3] + 300000)
 end
+
+local rapidWindow, repeatedWindow = tonumber(ARGV[15]), tonumber(ARGV[16])
+local historyTtl, stakeLimit = tonumber(ARGV[17]), tonumber(ARGV[18])
+redis.call("ZREMRANGEBYSCORE", historyBets, "-inf", now - rapidWindow)
+redis.call("ZADD", historyBets, now, betId)
+redis.call("ZADD", historyStakes, now, betId .. "|" .. ARGV[7])
+local stakeCard = redis.call("ZCARD", historyStakes)
+if stakeCard > stakeLimit then
+  redis.call("ZREMRANGEBYRANK", historyStakes, 0, stakeCard - stakeLimit - 1)
+end
+for _, selectionId in ipairs(selections) do
+  local selectionKey = historyBase .. ":selection:" .. selectionId
+  redis.call("ZREMRANGEBYSCORE", selectionKey, "-inf", now - repeatedWindow)
+  redis.call("ZADD", selectionKey, now, betId)
+  redis.call("PEXPIRE", selectionKey, historyTtl)
+end
+redis.call("PEXPIRE", historyBets, historyTtl)
+redis.call("PEXPIRE", historyStakes, historyTtl)
+redis.call("SET", KEYS[2], fingerprint, "PX", retention)
+return "APPLIED"


## `test(events): verify accepted pattern history`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
index 087daab..90a09b6 100644
--- a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionIdentityScriptTest.java
@@ -8,6 +8,7 @@ import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.counter.LimitKeys;
 import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.pattern.HistoryKeys;
 import com.sportsbook.risk.support.RedisTestSupport;
 import java.util.List;
 import java.util.UUID;
@@ -52,6 +53,16 @@ class AcceptedProjectionIdentityScriptTest extends RedisTestSupport {
     assertThat(redis.opsForValue().get(LimitKeys.selections(userId).sum())).isEqualTo("1");
   }
 
+  @Test
+  void retainsAcceptedPatternHistory() {
+    assertThat(execute(FINGERPRINT)).isEqualTo("APPLIED");
+    UserId userId = UserId.of(new UUID(0, 1));
+
+    assertThat(redis.opsForValue().get("accepted")).isEqualTo(FINGERPRINT);
+    assertThat(redis.opsForZSet().size(HistoryKeys.bets(userId))).isEqualTo(1);
+    assertThat(redis.opsForZSet().size(HistoryKeys.stakes(userId, Currency.KRW))).isEqualTo(1);
+  }
+
   private String execute(String fingerprint) {
     return redis.execute(SCRIPT, List.of("lifecycle", "accepted"), arguments(fingerprint));
   }


## `feat(events): encode accepted projection requests`

diff --git a/src/main/java/com/sportsbook/risk/reservation/AcceptedProjectionRequest.java b/src/main/java/com/sportsbook/risk/reservation/AcceptedProjectionRequest.java
new file mode 100644
index 0000000..a77a0f3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/AcceptedProjectionRequest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.List;
+import java.util.Objects;
+import java.util.stream.Collectors;
+
+/** Typed input for an atomic first-seen accepted-bet projection. */
+public record AcceptedProjectionRequest(List<String> keys, List<String> arguments) {
+  public AcceptedProjectionRequest {
+    keys = List.copyOf(Objects.requireNonNull(keys, "keys"));
+    arguments = List.copyOf(Objects.requireNonNull(arguments, "arguments"));
+  }
+
+  public static AcceptedProjectionRequest from(
+      RiskCheckCommand command,
+      String fingerprint,
+      RiskReservationProperties reservations,
+      RiskPatternProperties patterns,
+      RiskHistoryProperties history) {
+    Objects.requireNonNull(command, "command");
+    if (fingerprint == null || !fingerprint.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("accepted fingerprint must be lowercase SHA-256 hex");
+    }
+    String selections =
+        command.selectionIds().stream()
+            .map(selection -> selection.value().toString())
+            .collect(Collectors.joining(","));
+    return new AcceptedProjectionRequest(
+        List.of(
+            ReservationKeys.lifecycle(command.betId()),
+            ReservationKeys.acceptedFingerprint(command.betId())),
+        List.of(
+            "1",
+            Long.toString(command.now().toEpochMilli()),
+            Long.toString(reservations.retention().toMillis()),
+            fingerprint,
+            command.userId().value().toString(),
+            command.betId().value().toString(),
+            Long.toString(command.stake().amount()),
+            command.stake().currency().name(),
+            Integer.toString(command.selectionIds().size()),
+            selections,
+            Long.toString(LimitType.STAKE_DAILY.window().toMillis()),
+            Long.toString(LimitType.STAKE_WEEKLY.window().toMillis()),
+            Long.toString(LimitType.STAKE_MONTHLY.window().toMillis()),
+            Long.toString(LimitType.SELECTIONS_PER_MINUTE.window().toMillis()),
+            Long.toString(patterns.rapidBetting().window().toMillis()),
+            Long.toString(patterns.repeatedSelection().window().toMillis()),
+            Long.toString(history.idleRetention().toMillis()),
+            Integer.toString(history.maxStakeSamples())));
+  }
+}


## `test(events): verify accepted projection encoding`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionRequestTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionRequestTest.java
new file mode 100644
index 0000000..f68afb2
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedProjectionRequestTest.java
@@ -0,0 +1,70 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AcceptedProjectionRequestTest {
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 2));
+  private static final String FINGERPRINT = "a".repeat(64);
+
+  @Test
+  void encodesIdentityExposureAndPolicyArguments() {
+    AcceptedProjectionRequest request =
+        AcceptedProjectionRequest.from(
+            command(),
+            FINGERPRINT,
+            new RiskReservationProperties(null, null),
+            new RiskPatternProperties(null, null, null),
+            new RiskHistoryProperties(null, 2));
+
+    assertThat(request.keys())
+        .containsExactly(ReservationKeys.lifecycle(BET), ReservationKeys.acceptedFingerprint(BET));
+    assertThat(request.arguments()).hasSize(18);
+    assertThat(request.arguments().subList(3, 10))
+        .containsExactly(
+            FINGERPRINT,
+            USER.value().toString(),
+            BET.value().toString(),
+            "25",
+            "KRW",
+            "2",
+            "00000000-0000-0000-0000-000000000003,00000000-0000-0000-0000-000000000004");
+  }
+
+  @Test
+  void rejectsNoncanonicalFingerprints() {
+    assertThatThrownBy(
+            () ->
+                AcceptedProjectionRequest.from(
+                    command(),
+                    "A".repeat(64),
+                    new RiskReservationProperties(null, null),
+                    new RiskPatternProperties(null, null, null),
+                    new RiskHistoryProperties(null, 2)))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("lowercase SHA-256");
+  }
+
+  private static RiskCheckCommand command() {
+    return new RiskCheckCommand(
+        USER,
+        BET,
+        new Money(25, Currency.KRW),
+        List.of(SelectionId.of(new UUID(0, 3)), SelectionId.of(new UUID(0, 4))),
+        Instant.ofEpochMilli(2_000_000));
+  }
+}


