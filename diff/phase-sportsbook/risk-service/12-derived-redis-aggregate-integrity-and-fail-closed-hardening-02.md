## `feat(reservation): validate active reservation aggregates`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 5e4eee3..a4d1e11 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -73,38 +73,53 @@ if errorText then return redis.error_reply(errorText) end
 
 local activeBase, cleanups, stakeDecrements =
   "risk:reservations:user:{" .. userId .. "}", {}, {}
-local selectionDecrement = 0
-for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[2], 0, -1)) do
+local selectionDecrement, expectedSelections = 0, 0
+local expectedStakes = {[KEYS[4]] = 0}
+local expectedStakeEntries = {[KEYS[3]] = 0}
+local expectedSelectionEntries = {}
+for index = 1, selectionCount do
+  expectedSelectionEntries[KEYS[18 + selectionCount + index]] = 0
+end
+local activeBetIds = redis.call("ZRANGE", KEYS[2], 0, -1)
+for _, activeBetId in ipairs(activeBetIds) do
   local lifecycle = "risk:reservation:" .. activeBetId
   if keyType(lifecycle) ~= "hash" then return redis.error_reply("missing active lifecycle") end
   local state = redis.call("HGET", lifecycle, "state")
   local expiresAt = exact(redis.call("HGET", lifecycle, "expiresAt"), false)
   if not expiresAt then return redis.error_reply("corrupt active expiry") end
-  if state ~= "RESERVED" or expiresAt <= now then
-    local oldStakeText = redis.call("HGET", lifecycle, "stake")
-    local oldCountText = redis.call("HGET", lifecycle, "selectionCount")
-    local oldCurrency = redis.call("HGET", lifecycle, "currency")
-    local oldStake, oldCount = exact(oldStakeText, true), exact(oldCountText, true)
-    local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
-    if redis.call("HGET", lifecycle, "userId") ~= userId or not oldStake
-      or not oldCount or not oldCurrency or not oldSelections then
-      return redis.error_reply("corrupt active lifecycle")
-    end
-    if not string.match(oldCurrency, "^[A-Z]+$") then return redis.error_reply("corrupt currency") end
-    local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
-    local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
-    local cleanupError = typeError(entries, "zset") or typeError(sum, "string")
-    if cleanupError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. oldStakeText)
-      or not redis.call("ZSCORE", KEYS[5], activeBetId .. "|" .. oldCountText) then
-      return redis.error_reply(cleanupError or "missing active footprint")
-    end
-    for _, selectionId in ipairs(oldSelections) do
-      local key = activeBase .. ":selection:" .. selectionId
-      local selectionError = typeError(key, "zset")
-      if selectionError or not redis.call("ZSCORE", key, activeBetId) then
-        return redis.error_reply(selectionError or "missing selection footprint")
-      end
+  local oldStakeText = redis.call("HGET", lifecycle, "stake")
+  local oldCountText = redis.call("HGET", lifecycle, "selectionCount")
+  local oldCurrency = redis.call("HGET", lifecycle, "currency")
+  local oldStake, oldCount = exact(oldStakeText, true), exact(oldCountText, true)
+  local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
+  if redis.call("HGET", lifecycle, "userId") ~= userId or not oldStake
+    or not oldCount or not oldCurrency or not oldSelections then
+    return redis.error_reply("corrupt active lifecycle")
+  end
+  if not string.match(oldCurrency, "^[A-Z]+$") then return redis.error_reply("corrupt currency") end
+  local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
+  local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
+  local cleanupError = typeError(entries, "zset") or typeError(sum, "string")
+  if cleanupError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. oldStakeText)
+    or not redis.call("ZSCORE", KEYS[5], activeBetId .. "|" .. oldCountText) then
+    return redis.error_reply(cleanupError or "missing active footprint")
+  end
+  local nextExpectedStake = checkedAdd(expectedStakes[sum] or 0, oldStake)
+  local nextExpectedSelections = checkedAdd(expectedSelections, oldCount)
+  if not nextExpectedStake or not nextExpectedSelections then
+    return redis.error_reply("active footprint exceeds exact range")
+  end
+  expectedStakes[sum], expectedSelections = nextExpectedStake, nextExpectedSelections
+  expectedStakeEntries[entries] = (expectedStakeEntries[entries] or 0) + 1
+  for _, selectionId in ipairs(oldSelections) do
+    local key = activeBase .. ":selection:" .. selectionId
+    local selectionError = typeError(key, "zset")
+    if selectionError or not redis.call("ZSCORE", key, activeBetId) then
+      return redis.error_reply(selectionError or "missing selection footprint")
     end
+    expectedSelectionEntries[key] = (expectedSelectionEntries[key] or 0) + 1
+  end
+  if state ~= "RESERVED" or expiresAt <= now then
     table.insert(cleanups, {activeBetId, lifecycle, state, oldStakeText, oldCountText,
       entries, sum, oldSelections})
     local previous = stakeDecrements[sum] or 0
@@ -114,6 +129,31 @@ for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[2], 0, -1)) do
     stakeDecrements[sum] = previous + oldStake; selectionDecrement = selectionDecrement + oldCount
   end
 end
+for sum, value in pairs(expectedStakes) do
+  if exact(redis.call("GET", sum) or "0", false) ~= value then
+    return redis.error_reply("corrupt active stake aggregate")
+  end
+end
+for entries, count in pairs(expectedStakeEntries) do
+  if redis.call("ZCARD", entries) ~= count then
+    return redis.error_reply("corrupt active stake entries")
+  end
+end
+if exact(redis.call("GET", KEYS[6]) or "0", false) ~= expectedSelections then
+  return redis.error_reply("corrupt active selection aggregate")
+end
+if redis.call("ZCARD", KEYS[5]) ~= #activeBetIds then
+  return redis.error_reply("corrupt active selection entries")
+end
+for key, count in pairs(expectedSelectionEntries) do
+  if redis.call("ZCARD", key) ~= count then
+    return redis.error_reply("corrupt per-selection entries")
+  end
+end
+local activeGauge = exact(redis.call("GET", KEYS[18]) or "0", false)
+if not activeGauge or activeGauge < #activeBetIds then
+  return redis.error_reply("corrupt active gauge")
+end
 for sum, value in pairs(stakeDecrements) do
   local current = exact(redis.call("GET", sum), false)
   if not current or current < value then return redis.error_reply("corrupt active stake sum") end


## `test(reservation): reject corrupt active aggregates`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationAggregateConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationAggregateConsistencyScriptTest.java
new file mode 100644
index 0000000..12a983f
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationAggregateConsistencyScriptTest.java
@@ -0,0 +1,54 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationAggregateConsistencyScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void rejectsOverreportedStakeAggregateBeforeCleanup() {
+    assertOverstatementAborts(
+        ReservationKeys.activeStakes(USER, Currency.KRW).sum(),
+        "41",
+        "corrupt active stake aggregate");
+  }
+
+  @Test
+  void rejectsOverreportedSelectionAggregateBeforeCleanup() {
+    assertOverstatementAborts(
+        ReservationKeys.activeSelections(USER).sum(), "2", "corrupt active selection aggregate");
+  }
+
+  private void assertOverstatementAborts(String key, String tampered, String error) {
+    RiskCheckCommand expired = command(800, 40, Currency.KRW, selection(800));
+    assertThat(reserve(expired).approved()).isTrue();
+    redis.opsForValue().set(key, tampered);
+
+    RiskCheckCommand next =
+        new RiskCheckCommand(
+            USER,
+            BetId.of(new UUID(0, 801)),
+            new Money(10, Currency.KRW),
+            List.of(selection(801)),
+            NOW.plus(RiskReservationProperties.DEFAULT_LEASE));
+    assertThatThrownBy(() -> execute(request(next))).hasStackTraceContaining(error);
+
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(expired.betId()), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(next.betId()))).isFalse();
+    assertThat(redis.opsForValue().get(key)).isEqualTo(tampered);
+    assertThat(redis.opsForZSet().range(ReservationKeys.activeBets(USER), 0, -1))
+        .containsExactly(expired.betId().value().toString());
+    assertThat(redis.opsForValue().get(ReservationKeys.ACTIVE_COUNT)).isEqualTo("1");
+  }
+}


## `feat(reservation): validate committed reservation aggregates`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index a4d1e11..22ab10f 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -222,19 +222,24 @@ end
 local function readWindow(entries, sum, window)
   local errorText = typeError(entries, "zset") or typeError(sum, "string")
   if errorText then return nil, errorText end
-  if keyType(entries) == "none" then redis.call("DEL", sum); return 0, nil end
-  local total = exact(redis.call("GET", sum), false)
-  if not total then return nil, "missing or corrupt rolling sum" end
-  local expired, removed = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window), 0
-  for _, member in ipairs(expired) do
-    local value = memberAmount(member)
-    if not value or removed > maxExact - value then return nil, "corrupt rolling member" end
-    removed = removed + value
-  end
-  if removed > total then return nil, "rolling sum underflow" end
-  if #expired > 0 then redis.call("ZREMRANGEBYSCORE", entries, "-inf", now - window) end
-  total = total - removed
-  if redis.call("ZCARD", entries) == 0 then redis.call("DEL", entries, sum); return 0, nil end
+  local values, actual, removed = redis.call("ZRANGE", entries, 0, -1, "WITHSCORES"), 0, 0
+  local cutoff = now - window
+  for index = 1, #values, 2 do
+    local value, score = memberAmount(values[index]), tonumber(values[index + 1])
+    local nextActual = value and checkedAdd(actual, value) or nil
+    if not nextActual or not score then return nil, "corrupt rolling member" end
+    actual = nextActual
+    if score <= cutoff then
+      local nextRemoved = checkedAdd(removed, value)
+      if not nextRemoved then return nil, "corrupt rolling member" end
+      removed = nextRemoved
+    end
+  end
+  local stored = exact(redis.call("GET", sum) or "0", false)
+  if not stored or stored ~= actual then return nil, "corrupt rolling aggregate" end
+  if removed > 0 then redis.call("ZREMRANGEBYSCORE", entries, "-inf", cutoff) end
+  local total = actual - removed
+  if total == 0 then redis.call("DEL", entries, sum); return 0, nil end
   redis.call("SET", sum, string.format("%.0f", total), "PX", window + 300000)
   redis.call("PEXPIRE", entries, window + 300000)
   return total, nil


## `test(reservation): reject corrupt committed aggregates`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommittedConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommittedConsistencyScriptTest.java
new file mode 100644
index 0000000..637a003
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommittedConsistencyScriptTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommittedConsistencyScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void preservesCorruptCommittedWindowAndRejectsAdmission() {
+    Instant now = Instant.ofEpochMilli(3_000_000_000L);
+    LimitKeys.Keys daily = LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW);
+    String expired = LimitKeys.member(BetId.of(new UUID(0, 900)), 30);
+    String live = LimitKeys.member(BetId.of(new UUID(0, 901)), 20);
+    redis
+        .opsForZSet()
+        .add(
+            daily.entries(),
+            expired,
+            now.minus(LimitType.STAKE_DAILY.window()).minusMillis(1).toEpochMilli());
+    redis.opsForZSet().add(daily.entries(), live, now.minusMillis(1).toEpochMilli());
+    redis.opsForValue().set(daily.sum(), "51");
+    RiskCheckCommand candidate =
+        new RiskCheckCommand(
+            USER,
+            BetId.of(new UUID(0, 902)),
+            new Money(10, Currency.KRW),
+            List.of(selection(902)),
+            now);
+
+    assertThatThrownBy(() -> execute(request(candidate)))
+        .hasStackTraceContaining("corrupt rolling aggregate");
+
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(candidate.betId()))).isFalse();
+    assertThat(redis.opsForValue().get(daily.sum())).isEqualTo("51");
+    assertThat(redis.opsForZSet().range(daily.entries(), 0, -1)).containsExactly(expired, live);
+  }
+}


## `feat(reservation): validate active commit aggregates`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
index ab8b62b..5100152 100644
--- a/src/main/resources/scripts/risk-commit.lua
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -53,25 +53,66 @@ local base = "risk:reservations:user:{" .. userId .. "}"
 local bets, stakeEntries = base .. ":bets", base .. ":stakes:" .. string.lower(currency) .. ":entries"
 local stakeSum = base .. ":stakes:" .. string.lower(currency) .. ":sum"
 local selectionEntries, selectionSum = base .. ":selections:entries", base .. ":selections:sum"
-local errorText = typeError(bets, "zset") or typeError(stakeEntries, "zset")
-  or typeError(stakeSum, "string") or typeError(selectionEntries, "zset")
+local errorText = typeError(bets, "zset") or typeError(selectionEntries, "zset")
   or typeError(selectionSum, "string") or typeError(KEYS[2], "string")
-if errorText or not redis.call("ZSCORE", bets, betId)
-  or not redis.call("ZSCORE", stakeEntries, betId .. "|" .. stakeText)
-  or not redis.call("ZSCORE", selectionEntries, betId .. "|" .. countText) then
-  return redis.error_reply(errorText or "missing active reservation footprint")
+if errorText then return redis.error_reply(errorText) end
+local expectedStakes, expectedStakeCards, expectedSelectionCards = {}, {}, {}
+local expectedSelections, seenCurrent = 0, false
+local activeBetIds = redis.call("ZRANGE", bets, 0, -1)
+for _, activeBetId in ipairs(activeBetIds) do
+  local lifecycle = "risk:reservation:" .. activeBetId
+  if keyType(lifecycle) ~= "hash" then return redis.error_reply("missing active lifecycle") end
+  local oldStakeText = redis.call("HGET", lifecycle, "stake")
+  local oldCountText = redis.call("HGET", lifecycle, "selectionCount")
+  local oldCurrency = redis.call("HGET", lifecycle, "currency")
+  local oldStake, oldCount = exact(oldStakeText, true), exact(oldCountText, true)
+  local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
+  if redis.call("HGET", lifecycle, "userId") ~= userId
+    or redis.call("HGET", lifecycle, "betId") ~= activeBetId or not oldStake or not oldCount
+    or not oldCurrency or not string.match(oldCurrency, "^[A-Z]+$") or not oldSelections then
+    return redis.error_reply("corrupt active lifecycle")
+  end
+  local prefix = base .. ":stakes:" .. string.lower(oldCurrency)
+  local entries, sum = prefix .. ":entries", prefix .. ":sum"
+  local footprintError = typeError(entries, "zset") or typeError(sum, "string")
+  if footprintError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. oldStakeText)
+    or not redis.call("ZSCORE", selectionEntries, activeBetId .. "|" .. oldCountText) then
+    return redis.error_reply(footprintError or "missing active reservation footprint")
+  end
+  local expectedStake = expectedStakes[sum] or 0
+  if expectedStake > maxExact - oldStake or expectedSelections > maxExact - oldCount then
+    return redis.error_reply("active aggregate exceeds exact range")
+  end
+  expectedStakes[sum], expectedSelections = expectedStake + oldStake, expectedSelections + oldCount
+  expectedStakeCards[entries] = (expectedStakeCards[entries] or 0) + 1
+  for _, selectionId in ipairs(oldSelections) do
+    local key = base .. ":selection:" .. selectionId
+    local itemError = typeError(key, "zset")
+    if itemError or not redis.call("ZSCORE", key, activeBetId) then
+      return redis.error_reply(itemError or "missing per-selection footprint")
+    end
+    expectedSelectionCards[key] = (expectedSelectionCards[key] or 0) + 1
+  end
+  if activeBetId == betId then seenCurrent = true end
 end
-for _, selectionId in ipairs(selections) do
-  local key, itemError = base .. ":selection:" .. selectionId
-  itemError = typeError(key, "zset")
-  if itemError or not redis.call("ZSCORE", key, betId) then
-    return redis.error_reply(itemError or "missing per-selection footprint")
+if not seenCurrent then return redis.error_reply("missing active reservation footprint") end
+for sum, expected in pairs(expectedStakes) do
+  if exact(redis.call("GET", sum), false) ~= expected then
+    return redis.error_reply("inconsistent active stake aggregate")
   end
 end
-local stakeTotal, selectionTotal = exact(redis.call("GET", stakeSum), false), exact(redis.call("GET", selectionSum), false)
+for entries, expected in pairs(expectedStakeCards) do
+  if redis.call("ZCARD", entries) ~= expected then return redis.error_reply("orphan active stake entry") end
+end
+if exact(redis.call("GET", selectionSum), false) ~= expectedSelections
+  or redis.call("ZCARD", selectionEntries) ~= #activeBetIds then
+  return redis.error_reply("inconsistent active selection aggregate")
+end
+for key, expected in pairs(expectedSelectionCards) do
+  if redis.call("ZCARD", key) ~= expected then return redis.error_reply("orphan active selection entry") end
+end
 local gauge = exact(redis.call("GET", KEYS[2]), false)
-if not stakeTotal or stakeTotal < stake or not selectionTotal or selectionTotal < count
-  or not gauge or gauge < 1 then return redis.error_reply("corrupt active total") end
+if not gauge or gauge < #activeBetIds then return redis.error_reply("corrupt active gauge") end
 
 local function removeActive()
   redis.call("ZREM", bets, betId); redis.call("ZREM", stakeEntries, betId .. "|" .. stakeText)


## `test(reservation): reject corrupt active commit state`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitActiveConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitActiveConsistencyScriptTest.java
new file mode 100644
index 0000000..a5b05a9
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitActiveConsistencyScriptTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitActiveConsistencyScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void rejectsOverreportedActiveStakeBeforeCommitMutation() {
+    var command = command(1_400, 10, Currency.KRW);
+    ReservationDecision reserved = reserve(command);
+    var active = ReservationKeys.activeStakes(USER, Currency.KRW);
+    redis.opsForValue().set(active.sum(), "11");
+
+    assertThatThrownBy(() -> commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .hasStackTraceContaining("inconsistent active stake aggregate");
+
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.opsForValue().get(active.sum())).isEqualTo("11");
+    assertThat(redis.opsForZSet().range(active.entries(), 0, -1))
+        .containsExactly(command.betId().value() + "|10");
+  }
+}


