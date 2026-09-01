## `feat(reservation): validate committed commit aggregates`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
index 5100152..7a46b6c 100644
--- a/src/main/resources/scripts/risk-commit.lua
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -141,6 +141,13 @@ local function planWindow(entries, sum, amount, window)
   if keyType(entries) ~= "none" then
     total = exact(redis.call("GET", sum), false)
     if not total then return nil, "missing or corrupt rolling sum" end
+    local calculated = 0
+    for _, member in ipairs(redis.call("ZRANGE", entries, 0, -1)) do
+      local value = memberAmount(member)
+      if not value or calculated > maxExact - value then return nil, "corrupt rolling member" end
+      calculated = calculated + value
+    end
+    if calculated ~= total then return nil, "inconsistent rolling sum" end
   end
   local expired = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window)
   for _, member in ipairs(expired) do


## `test(reservation): reject corrupt committed windows`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitWindowConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitWindowConsistencyScriptTest.java
new file mode 100644
index 0000000..feb0ec3
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitWindowConsistencyScriptTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitWindowConsistencyScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void preservesActiveCapacityWhenCommittedWindowIsCorrupt() {
+    var command = command(1_500, 10, Currency.KRW);
+    ReservationDecision reserved = reserve(command);
+    LimitKeys.Keys daily = LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW);
+    redis
+        .opsForZSet()
+        .add(daily.entries(), new UUID(0, 1_599) + "|10", NOW.minusMillis(1).toEpochMilli());
+    redis.opsForValue().set(daily.sum(), "11");
+
+    assertThatThrownBy(() -> commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .hasStackTraceContaining("inconsistent rolling sum");
+
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.opsForValue().get(ReservationKeys.activeStakes(USER, Currency.KRW).sum()))
+        .isEqualTo("10");
+    assertThat(redis.opsForValue().get(daily.sum())).isEqualTo("11");
+  }
+}


## `feat(reservation): validate release aggregates`

diff --git a/src/main/resources/scripts/risk-release.lua b/src/main/resources/scripts/risk-release.lua
index eb9392d..3fb56e2 100644
--- a/src/main/resources/scripts/risk-release.lua
+++ b/src/main/resources/scripts/risk-release.lua
@@ -49,25 +49,66 @@ local base = "risk:reservations:user:{" .. userId .. "}"
 local bets, stakeEntries = base .. ":bets", base .. ":stakes:" .. string.lower(currency) .. ":entries"
 local stakeSum, selectionEntries = base .. ":stakes:" .. string.lower(currency) .. ":sum", base .. ":selections:entries"
 local selectionSum = base .. ":selections:sum"
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
-  local key = base .. ":selection:" .. selectionId
-  local selectionError = typeError(key, "zset")
-  if selectionError or not redis.call("ZSCORE", key, betId) then
-    return redis.error_reply(selectionError or "missing per-selection footprint")
+if not seenCurrent then return redis.error_reply("missing active reservation footprint") end
+for sum, expected in pairs(expectedStakes) do
+  if exact(redis.call("GET", sum), false) ~= expected then
+    return redis.error_reply("inconsistent active stake aggregate")
   end
 end
-local stakeTotal, selectionTotal, gauge = exact(redis.call("GET", stakeSum), false),
-  exact(redis.call("GET", selectionSum), false), exact(redis.call("GET", KEYS[2]), false)
-if not stakeTotal or stakeTotal < stake or not selectionTotal or selectionTotal < count
-  or not gauge or gauge < 1 then return redis.error_reply("corrupt active total") end
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
+local gauge = exact(redis.call("GET", KEYS[2]), false)
+if not gauge or gauge < #activeBetIds then return redis.error_reply("corrupt active gauge") end
 
 redis.call("ZREM", bets, betId); redis.call("ZREM", stakeEntries, betId .. "|" .. stakeText)
 redis.call("ZREM", selectionEntries, betId .. "|" .. countText)


## `test(reservation): reject corrupt release state`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseConsistencyScriptTest.java
new file mode 100644
index 0000000..e76777b
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseConsistencyScriptTest.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class ReservationReleaseConsistencyScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void rejectsOverreportedSelectionsBeforeReleaseMutation() {
+    var command = command(1_700, 10, Currency.KRW);
+    assertThat(reserve(command).approved()).isTrue();
+    redis.opsForValue().set(ReservationKeys.activeSelections(USER).sum(), "2");
+
+    assertThatThrownBy(() -> release(command.betId(), NOW.plusMillis(1)))
+        .hasStackTraceContaining("inconsistent active selection aggregate");
+
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.opsForValue().get(ReservationKeys.activeSelections(USER).sum()))
+        .isEqualTo("2");
+    assertThat(redis.opsForZSet().size(ReservationKeys.activeBets(USER))).isEqualTo(1);
+  }
+}
