# 예약 확정·해제·만료 상태 머신

## `feat(reservation): configure lifecycle retention`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RiskReservationProperties.java b/src/main/java/com/sportsbook/risk/reservation/RiskReservationProperties.java
new file mode 100644
index 0000000..a5b0830
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/RiskReservationProperties.java
@@ -0,0 +1,26 @@
+package com.sportsbook.risk.reservation;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Lease and tombstone lifetimes for idempotent reservation state. */
+@ConfigurationProperties(prefix = "risk.reservations")
+public record RiskReservationProperties(Duration lease, Duration retention) {
+  public static final Duration DEFAULT_LEASE = Duration.ofMinutes(2);
+  public static final Duration DEFAULT_RETENTION = Duration.ofDays(32);
+  private static final Duration LONGEST_COUNTER_WINDOW = Duration.ofDays(30);
+
+  public RiskReservationProperties {
+    lease = lease == null ? DEFAULT_LEASE : lease;
+    retention = retention == null ? DEFAULT_RETENTION : retention;
+    if (lease.isZero() || lease.isNegative()) {
+      throw new IllegalArgumentException("reservation lease must be positive");
+    }
+    if (retention.compareTo(lease) <= 0) {
+      throw new IllegalArgumentException("reservation retention must exceed the lease");
+    }
+    if (retention.compareTo(LONGEST_COUNTER_WINDOW) <= 0) {
+      throw new IllegalArgumentException("reservation retention must exceed the monthly window");
+    }
+  }
+}


## `test(reservation): verify lifecycle retention bounds`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RiskReservationPropertiesTest.java b/src/test/java/com/sportsbook/risk/reservation/RiskReservationPropertiesTest.java
new file mode 100644
index 0000000..aa24bde
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/RiskReservationPropertiesTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class RiskReservationPropertiesTest {
+  @Test
+  void suppliesLeaseAndTombstoneDefaults() {
+    RiskReservationProperties properties = new RiskReservationProperties(null, null);
+
+    assertThat(properties.lease()).isEqualTo(Duration.ofMinutes(2));
+    assertThat(properties.retention()).isEqualTo(Duration.ofDays(32));
+  }
+
+  @Test
+  void requiresAValidLeaseAndMonthlyTombstone() {
+    assertThatThrownBy(() -> new RiskReservationProperties(Duration.ZERO, Duration.ofDays(32)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () -> new RiskReservationProperties(Duration.ofMinutes(2), Duration.ofDays(30)))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("monthly");
+    assertThatThrownBy(
+            () -> new RiskReservationProperties(Duration.ofDays(40), Duration.ofDays(32)))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("lease");
+  }
+}


## `feat(reservation): expire reservation footprints`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 8abaa62..48d78b6 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -2,6 +2,7 @@ local maxExact = 9007199254740991
 local now, lease, retention = tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
 local fingerprint, userId, betId = ARGV[5], ARGV[6], ARGV[7]
 local stakeText, currency, countText = ARGV[8], ARGV[9], ARGV[10]
+local expiredCount = 0
 
 local function exact(text, positive)
   if not text or not string.match(text, "^%d+$") then return nil end
@@ -18,8 +19,18 @@ local function checkedAdd(left, right)
   if left > maxExact - right then return nil end
   return left + right
 end
+local function split(encoded, expected)
+  local values = {}
+  for value in string.gmatch(encoded or "", "[^,]+") do table.insert(values, value) end
+  if #values == expected then return values end
+end
+local function decrement(key, amount)
+  local nextValue = exact(redis.call("GET", key), false) - amount
+  if nextValue == 0 then redis.call("DEL", key)
+  else redis.call("SET", key, string.format("%.0f", nextValue)) end
+end
 local function response(payload)
-  payload.version, payload.expired = "1", "0"
+  payload.version, payload.expired = "1", string.format("%.0f", expiredCount)
   return cjson.encode(payload)
 end
 
@@ -41,6 +52,74 @@ local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
   or typeError(KEYS[5], "zset") or typeError(KEYS[6], "string")
   or typeError(KEYS[7], "hash") or typeError(KEYS[18], "string")
 if errorText then return redis.error_reply(errorText) end
+
+local activeBase, cleanups, stakeDecrements =
+  "risk:reservations:user:{" .. userId .. "}", {}, {}
+local selectionDecrement = 0
+for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[2], 0, -1)) do
+  local lifecycle = "risk:reservation:" .. activeBetId
+  if keyType(lifecycle) ~= "hash" then return redis.error_reply("missing active lifecycle") end
+  local state = redis.call("HGET", lifecycle, "state")
+  local expiresAt = exact(redis.call("HGET", lifecycle, "expiresAt"), false)
+  if not expiresAt then return redis.error_reply("corrupt active expiry") end
+  if state ~= "RESERVED" or expiresAt <= now then
+    local oldStakeText = redis.call("HGET", lifecycle, "stake")
+    local oldCountText = redis.call("HGET", lifecycle, "selectionCount")
+    local oldCurrency = redis.call("HGET", lifecycle, "currency")
+    local oldStake, oldCount = exact(oldStakeText, true), exact(oldCountText, true)
+    local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
+    if redis.call("HGET", lifecycle, "userId") ~= userId or not oldStake
+      or not oldCount or not oldCurrency or not oldSelections then
+      return redis.error_reply("corrupt active lifecycle")
+    end
+    local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
+    local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
+    local cleanupError = typeError(entries, "zset") or typeError(sum, "string")
+    if cleanupError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. oldStakeText)
+      or not redis.call("ZSCORE", KEYS[5], activeBetId .. "|" .. oldCountText) then
+      return redis.error_reply(cleanupError or "missing active footprint")
+    end
+    for _, selectionId in ipairs(oldSelections) do
+      local key = activeBase .. ":selection:" .. selectionId
+      local selectionError = typeError(key, "zset")
+      if selectionError or not redis.call("ZSCORE", key, activeBetId) then
+        return redis.error_reply(selectionError or "missing selection footprint")
+      end
+    end
+    table.insert(cleanups, {activeBetId, lifecycle, state, oldStakeText, oldCountText,
+      entries, sum, oldSelections})
+    stakeDecrements[sum] = (stakeDecrements[sum] or 0) + oldStake
+    selectionDecrement = selectionDecrement + oldCount
+  end
+end
+for sum, value in pairs(stakeDecrements) do
+  local current = exact(redis.call("GET", sum), false)
+  if not current or current < value then return redis.error_reply("corrupt active stake sum") end
+end
+if #cleanups > 0 then
+  local selectionsTotal, gauge = exact(redis.call("GET", KEYS[6]), false),
+    exact(redis.call("GET", KEYS[18]), false)
+  if not selectionsTotal or selectionsTotal < selectionDecrement or not gauge or gauge < #cleanups then
+    return redis.error_reply("corrupt active totals")
+  end
+end
+for _, item in ipairs(cleanups) do
+  redis.call("ZREM", KEYS[2], item[1]); redis.call("ZREM", item[6], item[1] .. "|" .. item[4])
+  redis.call("ZREM", KEYS[5], item[1] .. "|" .. item[5])
+  for _, selectionId in ipairs(item[8]) do
+    local key = activeBase .. ":selection:" .. selectionId
+    redis.call("ZREM", key, item[1]); if redis.call("ZCARD", key) == 0 then redis.call("DEL", key) end
+  end
+  if item[3] == "RESERVED" then
+    redis.call("HSET", item[2], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
+    redis.call("PEXPIRE", item[2], retention); expiredCount = expiredCount + 1
+  end
+end
+for sum, value in pairs(stakeDecrements) do decrement(sum, value) end
+if selectionDecrement > 0 then decrement(KEYS[6], selectionDecrement) end
+if #cleanups > 0 then decrement(KEYS[18], #cleanups) end
+if redis.call("ZCARD", KEYS[2]) == 0 then redis.call("DEL", KEYS[2]) end
+if redis.call("ZCARD", KEYS[5]) == 0 then redis.call("DEL", KEYS[5]) end
 local existing = redis.call("HGET", KEYS[1], "state")
 if existing then
   if redis.call("HGET", KEYS[1], "fingerprint") ~= fingerprint then


## `test(reservation): verify reservation expiry recovery`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationExpiryCleanupScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationExpiryCleanupScriptTest.java
new file mode 100644
index 0000000..5374db1
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationExpiryCleanupScriptTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationExpiryCleanupScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void removesEveryExpiredCapacityFootprintBeforeAdmission() {
+    RiskCheckCommand expired = command(600, 40, Currency.KRW, selection(600));
+    RiskCheckCommand next =
+        new RiskCheckCommand(
+            USER,
+            BetId.of(new UUID(0, 601)),
+            new Money(10, Currency.KRW),
+            List.of(selection(601)),
+            NOW.plus(RiskReservationProperties.DEFAULT_LEASE));
+
+    assertThat(reserve(expired).approved()).isTrue();
+    ReservationWireMapper.Decoded admitted = execute(request(next));
+
+    assertThat(admitted.decision().approved()).isTrue();
+    assertThat(admitted.expired()).isEqualTo(1);
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(expired.betId()), "state"))
+        .isEqualTo("EXPIRED");
+    assertThat(redis.hasKey(ReservationKeys.activeSelection(USER, selection(600)))).isFalse();
+    assertThat(redis.opsForZSet().range(ReservationKeys.activeBets(USER), 0, -1))
+        .containsExactly(next.betId().value().toString());
+    assertThat(redis.opsForValue().get(ReservationKeys.activeStakes(USER, Currency.KRW).sum()))
+        .isEqualTo("10");
+    assertThat(redis.opsForValue().get(ReservationKeys.activeSelections(USER).sum()))
+        .isEqualTo("1");
+    assertThat(redis.opsForValue().get(ReservationKeys.ACTIVE_COUNT)).isEqualTo("1");
+  }
+}


## `feat(reservation): bind lifecycle transition scripts`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationTransitionRequest.java b/src/main/java/com/sportsbook/risk/reservation/ReservationTransitionRequest.java
new file mode 100644
index 0000000..6ac240a
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationTransitionRequest.java
@@ -0,0 +1,65 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+
+/** Typed key and argument boundary for commit and release lifecycle scripts. */
+public record ReservationTransitionRequest(List<String> keys, List<String> arguments) {
+  public ReservationTransitionRequest {
+    keys = List.copyOf(Objects.requireNonNull(keys, "keys"));
+    arguments = List.copyOf(Objects.requireNonNull(arguments, "arguments"));
+  }
+
+  public static ReservationTransitionRequest commit(
+      BetId betId,
+      String token,
+      Instant now,
+      RiskReservationProperties reservations,
+      RiskPatternProperties patterns,
+      RiskHistoryProperties history) {
+    requireToken(token);
+    Objects.requireNonNull(now, "now");
+    return new ReservationTransitionRequest(
+        keys(betId),
+        List.of(
+            "1",
+            Long.toString(now.toEpochMilli()),
+            Long.toString(reservations.retention().toMillis()),
+            token,
+            Long.toString(LimitType.STAKE_DAILY.window().toMillis()),
+            Long.toString(LimitType.STAKE_WEEKLY.window().toMillis()),
+            Long.toString(LimitType.STAKE_MONTHLY.window().toMillis()),
+            Long.toString(LimitType.SELECTIONS_PER_MINUTE.window().toMillis()),
+            Long.toString(patterns.rapidBetting().window().toMillis()),
+            Long.toString(patterns.repeatedSelection().window().toMillis()),
+            Long.toString(history.idleRetention().toMillis()),
+            Integer.toString(history.maxStakeSamples())));
+  }
+
+  public static ReservationTransitionRequest release(
+      BetId betId, Instant now, RiskReservationProperties reservations) {
+    Objects.requireNonNull(now, "now");
+    return new ReservationTransitionRequest(
+        keys(betId),
+        List.of(
+            "1",
+            Long.toString(now.toEpochMilli()),
+            Long.toString(reservations.retention().toMillis())));
+  }
+
+  private static List<String> keys(BetId betId) {
+    return List.of(ReservationKeys.lifecycle(betId), ReservationKeys.ACTIVE_COUNT);
+  }
+
+  private static void requireToken(String token) {
+    if (token == null || !token.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException(
+          "reservation token must be a 64-character lowercase hex value");
+    }
+  }
+}


## `feat(reservation): define atomic commit lifecycle`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
new file mode 100644
index 0000000..fefcb31
--- /dev/null
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -0,0 +1,48 @@
+local maxExact = 9007199254740991
+local now = tonumber(ARGV[2])
+local retention = tonumber(ARGV[3])
+local token = ARGV[4]
+
+local function exact(text, positive)
+  if not text or not string.match(text, "^%d+$") then return nil end
+  local value = tonumber(text)
+  if not value or value > maxExact or (positive and value <= 0) then return nil end
+  return value
+end
+
+local function keyType(key)
+  return redis.call("TYPE", key).ok
+end
+
+if ARGV[1] ~= "1" or #KEYS ~= 2 or #ARGV ~= 12
+  or not exact(ARGV[2], false) or not exact(ARGV[3], true)
+  or not token or #token ~= 64 or not string.match(token, "^[0-9a-f]+$") then
+  return redis.error_reply("invalid commit request")
+end
+for index = 5, 12 do
+  if not exact(ARGV[index], true) then return redis.error_reply("invalid commit policy") end
+end
+
+local lifecycleType = keyType(KEYS[1])
+if lifecycleType == "none" then return "NOT_FOUND" end
+if lifecycleType ~= "hash" then return redis.error_reply("wrong reservation lifecycle type") end
+if redis.call("HGET", KEYS[1], "fingerprint") ~= token then return "CONFLICT" end
+
+local state = redis.call("HGET", KEYS[1], "state")
+if state == "COMMITTED" then return "REPLAYED" end
+if state == "EXPIRED" or state == "RELEASED" or state == "REJECTED" then
+  return "TOMBSTONED"
+end
+if state ~= "RESERVED" then return redis.error_reply("unknown reservation state") end
+
+local expiresAt = exact(redis.call("HGET", KEYS[1], "expiresAt"), false)
+if not expiresAt then return redis.error_reply("corrupt reservation expiry") end
+if expiresAt <= now then
+  redis.call("HSET", KEYS[1], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
+  redis.call("PEXPIRE", KEYS[1], retention)
+  return "EXPIRED"
+end
+
+redis.call("HSET", KEYS[1], "state", "COMMITTED", "committedAt", string.format("%.0f", now))
+redis.call("PEXPIRE", KEYS[1], retention)
+return "APPLIED"


## `test(reservation): verify token-bound commit lifecycle`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitLifecycleScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitLifecycleScriptTest.java
new file mode 100644
index 0000000..53c8012
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitLifecycleScriptTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitLifecycleScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void requiresTheOpaqueReservationTokenAndReplaysCommit() {
+    var command = command(1_000, 10, Currency.KRW);
+    ReservationDecision reserved = reserve(command);
+
+    assertThat(commit(command.betId(), "0".repeat(64), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.CONFLICT);
+    assertThat(commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.APPLIED);
+    assertThat(commit(command.betId(), reserved.token(), NOW.plusMillis(2)))
+        .isEqualTo(ReservationTransition.REPLAYED);
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("COMMITTED");
+  }
+
+  @Test
+  void tombstonesExpiredReservations() {
+    var command = command(1_001, 10, Currency.KRW);
+    ReservationDecision reserved = reserve(command);
+
+    assertThat(
+            commit(
+                command.betId(),
+                reserved.token(),
+                NOW.plus(RiskReservationProperties.DEFAULT_LEASE)))
+        .isEqualTo(ReservationTransition.EXPIRED);
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("EXPIRED");
+  }
+}


## `feat(reservation): consume active commit footprints`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
index fefcb31..f071421 100644
--- a/src/main/resources/scripts/risk-commit.lua
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -1,17 +1,25 @@
-local maxExact = 9007199254740991
-local now = tonumber(ARGV[2])
-local retention = tonumber(ARGV[3])
+local maxExact, now, retention = 9007199254740991, tonumber(ARGV[2]), tonumber(ARGV[3])
 local token = ARGV[4]
-
 local function exact(text, positive)
   if not text or not string.match(text, "^%d+$") then return nil end
   local value = tonumber(text)
   if not value or value > maxExact or (positive and value <= 0) then return nil end
   return value
 end
-
-local function keyType(key)
-  return redis.call("TYPE", key).ok
+local function keyType(key) return redis.call("TYPE", key).ok end
+local function typeError(key, expected)
+  local actual = keyType(key)
+  if actual ~= "none" and actual ~= expected then return "wrong Redis type for " .. key end
+end
+local function split(encoded, expected)
+  local values = {}
+  for value in string.gmatch(encoded or "", "[^,]+") do table.insert(values, value) end
+  if #values == expected then return values end
+end
+local function decrement(key, amount)
+  local nextValue = exact(redis.call("GET", key), false) - amount
+  if nextValue == 0 then redis.call("DEL", key)
+  else redis.call("SET", key, string.format("%.0f", nextValue)) end
 end
 
 if ARGV[1] ~= "1" or #KEYS ~= 2 or #ARGV ~= 12
@@ -22,27 +30,68 @@ end
 for index = 5, 12 do
   if not exact(ARGV[index], true) then return redis.error_reply("invalid commit policy") end
 end
-
 local lifecycleType = keyType(KEYS[1])
 if lifecycleType == "none" then return "NOT_FOUND" end
 if lifecycleType ~= "hash" then return redis.error_reply("wrong reservation lifecycle type") end
 if redis.call("HGET", KEYS[1], "fingerprint") ~= token then return "CONFLICT" end
-
 local state = redis.call("HGET", KEYS[1], "state")
 if state == "COMMITTED" then return "REPLAYED" end
-if state == "EXPIRED" or state == "RELEASED" or state == "REJECTED" then
-  return "TOMBSTONED"
-end
+if state == "EXPIRED" or state == "RELEASED" or state == "REJECTED" then return "TOMBSTONED" end
 if state ~= "RESERVED" then return redis.error_reply("unknown reservation state") end
 
+local userId, betId = redis.call("HGET", KEYS[1], "userId"), redis.call("HGET", KEYS[1], "betId")
+local stakeText, currency = redis.call("HGET", KEYS[1], "stake"), redis.call("HGET", KEYS[1], "currency")
+local countText = redis.call("HGET", KEYS[1], "selectionCount")
+local stake, count = exact(stakeText, true), exact(countText, true)
 local expiresAt = exact(redis.call("HGET", KEYS[1], "expiresAt"), false)
-if not expiresAt then return redis.error_reply("corrupt reservation expiry") end
+local selections = split(redis.call("HGET", KEYS[1], "selections"), count or -1)
+if not userId or not betId or not stake or not currency or not count or not expiresAt or not selections
+  or not string.match(userId, "^[0-9a-f%-]+$") or not string.match(betId, "^[0-9a-f%-]+$")
+  or not string.match(currency, "^[A-Z]+$") then return redis.error_reply("corrupt reservation identity") end
+
+local base = "risk:reservations:user:{" .. userId .. "}"
+local bets, stakeEntries = base .. ":bets", base .. ":stakes:" .. string.lower(currency) .. ":entries"
+local stakeSum = base .. ":stakes:" .. string.lower(currency) .. ":sum"
+local selectionEntries, selectionSum = base .. ":selections:entries", base .. ":selections:sum"
+local errorText = typeError(bets, "zset") or typeError(stakeEntries, "zset")
+  or typeError(stakeSum, "string") or typeError(selectionEntries, "zset")
+  or typeError(selectionSum, "string") or typeError(KEYS[2], "string")
+if errorText or not redis.call("ZSCORE", bets, betId)
+  or not redis.call("ZSCORE", stakeEntries, betId .. "|" .. stakeText)
+  or not redis.call("ZSCORE", selectionEntries, betId .. "|" .. countText) then
+  return redis.error_reply(errorText or "missing active reservation footprint")
+end
+for _, selectionId in ipairs(selections) do
+  local key, itemError = base .. ":selection:" .. selectionId
+  itemError = typeError(key, "zset")
+  if itemError or not redis.call("ZSCORE", key, betId) then
+    return redis.error_reply(itemError or "missing per-selection footprint")
+  end
+end
+local stakeTotal, selectionTotal = exact(redis.call("GET", stakeSum), false), exact(redis.call("GET", selectionSum), false)
+local gauge = exact(redis.call("GET", KEYS[2]), false)
+if not stakeTotal or stakeTotal < stake or not selectionTotal or selectionTotal < count
+  or not gauge or gauge < 1 then return redis.error_reply("corrupt active total") end
+
+local function removeActive()
+  redis.call("ZREM", bets, betId); redis.call("ZREM", stakeEntries, betId .. "|" .. stakeText)
+  redis.call("ZREM", selectionEntries, betId .. "|" .. countText)
+  for _, selectionId in ipairs(selections) do
+    local key = base .. ":selection:" .. selectionId
+    redis.call("ZREM", key, betId); if redis.call("ZCARD", key) == 0 then redis.call("DEL", key) end
+  end
+  decrement(stakeSum, stake); decrement(selectionSum, count); decrement(KEYS[2], 1)
+  if redis.call("ZCARD", bets) == 0 then redis.call("DEL", bets) end
+  if redis.call("ZCARD", stakeEntries) == 0 then redis.call("DEL", stakeEntries) end
+  if redis.call("ZCARD", selectionEntries) == 0 then redis.call("DEL", selectionEntries) end
+end
+
+removeActive()
 if expiresAt <= now then
   redis.call("HSET", KEYS[1], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
   redis.call("PEXPIRE", KEYS[1], retention)
   return "EXPIRED"
 end
-
 redis.call("HSET", KEYS[1], "state", "COMMITTED", "committedAt", string.format("%.0f", now))
 redis.call("PEXPIRE", KEYS[1], retention)
 return "APPLIED"


