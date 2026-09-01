# 파생 Redis 집계의 무결성과 Fail-Closed 하드닝

## `feat(snapshot): validate active aggregate consistency`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index 19cfedf..e8f9b00 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -116,58 +116,92 @@ for index = 1, count do
 end
 
 local activeBase = "risk:reservations:user:{" .. userId .. "}"
-local plans, stakeTotals, selectionTotal = {}, {}, 0
+local plans, stakeTotals, expectedStakes = {}, {}, {}
+local expectedSelections, selectionTotal, expectedSelectionCards = 0, 0, {}
 local cleanupError = typeError(KEYS[10], "zset") or typeError(KEYS[13], "zset")
   or typeError(KEYS[14], "string") or typeError(KEYS[17], "string")
 if cleanupError then return redis.error_reply(cleanupError) end
-for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[10], 0, -1)) do
+local activeBetIds = redis.call("ZRANGE", KEYS[10], 0, -1)
+for _, activeBetId in ipairs(activeBetIds) do
   local lifecycle = "risk:reservation:" .. activeBetId
   if keyType(lifecycle) ~= "hash" then return redis.error_reply("missing lifecycle") end
   local state = redis.call("HGET", lifecycle, "state")
   local expiresAt = exact(redis.call("HGET", lifecycle, "expiresAt"), false)
   if not expiresAt then return redis.error_reply("corrupt lifecycle expiry") end
-  if state ~= "RESERVED" or expiresAt <= now then
-    local stakeText = redis.call("HGET", lifecycle, "stake")
-    local countText = redis.call("HGET", lifecycle, "selectionCount")
-    local oldCurrency = redis.call("HGET", lifecycle, "currency")
-    local stake = exact(stakeText, true)
-    local oldCount = exact(countText, true)
-    local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
-    if redis.call("HGET", lifecycle, "userId") ~= userId or not stake
-      or not oldCount or not oldCurrency or not oldSelections then
-      return redis.error_reply("corrupt active lifecycle")
-    end
-    local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
-    local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
-    local planError = typeError(entries, "zset") or typeError(sum, "string")
-    if planError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. stakeText)
-      or not redis.call("ZSCORE", KEYS[13], activeBetId .. "|" .. countText) then
-      return redis.error_reply(planError or "missing active footprint")
-    end
-    for _, selectionId in ipairs(oldSelections) do
-      local selectionKey = activeBase .. ":selection:" .. selectionId
-      local selectionError = typeError(selectionKey, "zset")
-      if selectionError or not redis.call("ZSCORE", selectionKey, activeBetId) then
-        return redis.error_reply(selectionError or "missing selection footprint")
-      end
+  local stakeText = redis.call("HGET", lifecycle, "stake")
+  local countText = redis.call("HGET", lifecycle, "selectionCount")
+  local oldCurrency = redis.call("HGET", lifecycle, "currency")
+  local stake, oldCount = exact(stakeText, true), exact(countText, true)
+  local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
+  if redis.call("HGET", lifecycle, "userId") ~= userId or not stake or not oldCount
+    or not oldCurrency or not string.match(oldCurrency, "^[A-Z]+$") or not oldSelections then
+    return redis.error_reply("corrupt active lifecycle")
+  end
+  local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
+  local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
+  local planError = typeError(entries, "zset") or typeError(sum, "string")
+  if planError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. stakeText)
+    or not redis.call("ZSCORE", KEYS[13], activeBetId .. "|" .. countText) then
+    return redis.error_reply(planError or "missing active footprint")
+  end
+  local expected = expectedStakes[sum] or {entries = entries, amount = 0, count = 0}
+  if expected.amount > maxExact - stake or expectedSelections > maxExact - oldCount then
+    return redis.error_reply("active aggregate exceeds exact range")
+  end
+  expected.amount, expected.count = expected.amount + stake, expected.count + 1
+  expectedStakes[sum], expectedSelections = expected, expectedSelections + oldCount
+  for _, selectionId in ipairs(oldSelections) do
+    local selectionKey = activeBase .. ":selection:" .. selectionId
+    local selectionError = typeError(selectionKey, "zset")
+    if selectionError or not redis.call("ZSCORE", selectionKey, activeBetId) then
+      return redis.error_reply(selectionError or "missing selection footprint")
     end
+    expectedSelectionCards[selectionKey] = (expectedSelectionCards[selectionKey] or 0) + 1
+  end
+  if state ~= "RESERVED" or expiresAt <= now then
     table.insert(plans, {activeBetId, lifecycle, state, stakeText, countText,
       entries, sum, oldSelections})
     stakeTotals[sum] = (stakeTotals[sum] or 0) + stake
     selectionTotal = selectionTotal + oldCount
   end
 end
-for sum, value in pairs(stakeTotals) do
+for sum, expected in pairs(expectedStakes) do
   local current = exact(redis.call("GET", sum), false)
-  if not current or current < value then return redis.error_reply("corrupt active stake sum") end
+  if current ~= expected.amount or redis.call("ZCARD", expected.entries) ~= expected.count then
+    return redis.error_reply("inconsistent active stake aggregate")
+  end
 end
-if #plans > 0 then
-  local currentSelections = exact(redis.call("GET", KEYS[14]), false)
-  local gauge = exact(redis.call("GET", KEYS[17]), false)
-  if not currentSelections or currentSelections < selectionTotal or not gauge or gauge < #plans then
-    return redis.error_reply("corrupt active totals")
+if not expectedStakes[KEYS[12]]
+  and (keyType(KEYS[11]) ~= "none" or keyType(KEYS[12]) ~= "none") then
+  return redis.error_reply("orphan active stake aggregate")
+end
+local currentSelections = exact(redis.call("GET", KEYS[14]), false)
+if #activeBetIds == 0 then
+  if keyType(KEYS[13]) ~= "none" or keyType(KEYS[14]) ~= "none" then
+    return redis.error_reply("orphan active selection aggregate")
+  end
+elseif currentSelections ~= expectedSelections or redis.call("ZCARD", KEYS[13]) ~= #activeBetIds then
+  return redis.error_reply("inconsistent active selection aggregate")
+end
+for selectionKey, expected in pairs(expectedSelectionCards) do
+  if redis.call("ZCARD", selectionKey) ~= expected then
+    return redis.error_reply("inconsistent active selection footprint")
+  end
+end
+for index = 1, count do
+  local selectionKey = KEYS[17 + count + index]
+  if typeError(selectionKey, "zset")
+    or redis.call("ZCARD", selectionKey) ~= (expectedSelectionCards[selectionKey] or 0) then
+    return redis.error_reply("orphan active selection footprint")
   end
 end
+local gauge = exact(redis.call("GET", KEYS[17]), false)
+if keyType(KEYS[17]) ~= "none" and not gauge then
+  return redis.error_reply("corrupt active gauge")
+end
+if #activeBetIds > 0 and (not gauge or gauge < #activeBetIds) then
+  return redis.error_reply("corrupt active gauge")
+end
 for _, plan in ipairs(plans) do
   redis.call("ZREM", KEYS[10], plan[1])
   redis.call("ZREM", plan[6], plan[1] .. "|" .. plan[4])


## `test(snapshot): reject inconsistent active aggregates`

diff --git a/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotAggregateConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotAggregateConsistencyScriptTest.java
new file mode 100644
index 0000000..d1cb89f
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotAggregateConsistencyScriptTest.java
@@ -0,0 +1,84 @@
+package com.sportsbook.risk.snapshot;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.reservation.ReservationKeys;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskSnapshotAggregateConsistencyScriptTest extends RedisTestSupport {
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 2));
+  private static final SelectionId SELECTION = SelectionId.of(new UUID(0, 3));
+  private static final Instant NOW = Instant.ofEpochMilli(2_000_000);
+
+  @Test
+  void rejectsOverreportedActiveStakeWithoutPartiallyExpiring() {
+    String bet = BET.value().toString();
+    redis
+        .opsForHash()
+        .putAll(
+            ReservationKeys.lifecycle(BET),
+            Map.of(
+                "state",
+                "RESERVED",
+                "userId",
+                USER.value().toString(),
+                "stake",
+                "10",
+                "currency",
+                "KRW",
+                "selectionCount",
+                "1",
+                "selections",
+                SELECTION.value().toString(),
+                "expiresAt",
+                Long.toString(NOW.toEpochMilli())));
+    redis
+        .opsForZSet()
+        .add(ReservationKeys.activeBets(USER), bet, NOW.minusMillis(1).toEpochMilli());
+    var stakes = ReservationKeys.activeStakes(USER, Currency.KRW);
+    redis.opsForZSet().add(stakes.entries(), bet + "|10", NOW.minusMillis(1).toEpochMilli());
+    redis.opsForValue().set(stakes.sum(), "20");
+    var selections = ReservationKeys.activeSelections(USER);
+    redis.opsForZSet().add(selections.entries(), bet + "|1", NOW.minusMillis(1).toEpochMilli());
+    redis.opsForValue().set(selections.sum(), "1");
+    redis
+        .opsForZSet()
+        .add(ReservationKeys.activeSelection(USER, SELECTION), bet, NOW.toEpochMilli());
+    redis.opsForValue().set(ReservationKeys.ACTIVE_COUNT, "1");
+
+    assertThatThrownBy(() -> reader().read(context()))
+        .hasStackTraceContaining("inconsistent active stake aggregate");
+    assertThat(redis.opsForHash().get(ReservationKeys.lifecycle(BET), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.opsForZSet().size(ReservationKeys.activeBets(USER))).isEqualTo(1);
+    assertThat(redis.opsForValue().get(stakes.sum())).isEqualTo("20");
+  }
+
+  private RedisRiskSnapshotReader reader() {
+    return new RedisRiskSnapshotReader(
+        redis,
+        new RiskPatternProperties(null, null, null),
+        new RiskReservationProperties(null, null),
+        new RiskSnapshotWireMapper(new ObjectMapper()));
+  }
+
+  private static PatternContext context() {
+    return new PatternContext(USER, BET, new Money(1, Currency.KRW), List.of(SELECTION), NOW);
+  }
+}


## `feat(snapshot): validate committed aggregate consistency`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index e8f9b00..80867d2 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -50,6 +50,15 @@ local function capture(entries, sum, window)
   end
   local total = exact(redis.call("GET", sum), false)
   if not total then return failure("missing or corrupt rolling sum") end
+  local calculated = 0
+  for _, member in ipairs(redis.call("ZRANGE", entries, 0, -1)) do
+    local value = amount(member)
+    if not value or calculated > maxExact - value then
+      return failure("corrupt rolling member")
+    end
+    calculated = calculated + value
+  end
+  if calculated ~= total then return failure("inconsistent rolling sum") end
   local expired = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window)
   local removed = 0
   for _, member in ipairs(expired) do


## `test(snapshot): reject inconsistent committed aggregates`

diff --git a/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotCommittedConsistencyScriptTest.java b/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotCommittedConsistencyScriptTest.java
new file mode 100644
index 0000000..8f76660
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/snapshot/RiskSnapshotCommittedConsistencyScriptTest.java
@@ -0,0 +1,58 @@
+package com.sportsbook.risk.snapshot;
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
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RiskSnapshotCommittedConsistencyScriptTest extends RedisTestSupport {
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 2));
+  private static final Instant NOW = Instant.ofEpochMilli(2_000_000);
+
+  @Test
+  void reportsAnOverstatedCommittedSumWithoutMutatingTheWindow() {
+    LimitKeys.Keys daily = LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW);
+    String member = LimitKeys.member(BET, 10);
+    redis.opsForZSet().add(daily.entries(), member, NOW.minusMillis(1).toEpochMilli());
+    redis.opsForValue().set(daily.sum(), "20");
+
+    RiskSnapshot snapshot = reader().read(context());
+
+    assertThat(snapshot.limits().values().get(LimitType.STAKE_DAILY).failure())
+        .contains("inconsistent rolling sum");
+    assertThat(redis.opsForZSet().range(daily.entries(), 0, -1)).containsExactly(member);
+    assertThat(redis.opsForValue().get(daily.sum())).isEqualTo("20");
+  }
+
+  private RedisRiskSnapshotReader reader() {
+    return new RedisRiskSnapshotReader(
+        redis,
+        new RiskPatternProperties(null, null, null),
+        new RiskReservationProperties(null, null),
+        new RiskSnapshotWireMapper(new ObjectMapper()));
+  }
+
+  private static PatternContext context() {
+    return new PatternContext(
+        USER,
+        BetId.of(new UUID(0, 9)),
+        new Money(1, Currency.KRW),
+        List.of(SelectionId.of(new UUID(0, 3))),
+        NOW);
+  }
+}


## `feat(reservation): harden reservation footprints`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 48d78b6..5e4eee3 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -41,10 +41,28 @@ if ARGV[1] ~= "1" or not exact(ARGV[2], false) or not exact(ARGV[3], true)
   or #KEYS ~= 18 + selectionCount * 2 or #ARGV ~= 33 + selectionCount then
   return redis.error_reply("invalid reservation request")
 end
+for _, index in ipairs({8, 10, 16, 17, 18, 19, 21, 22, 25, 26, 29, 30, 32, 33}) do
+  if not exact(ARGV[index], true) then return redis.error_reply("invalid reservation number") end
+end
+for index = 11, 15 do
+  if not exact(ARGV[index], false) then return redis.error_reply("invalid reservation limit") end
+end
+for _, index in ipairs({20, 24, 28}) do
+  if ARGV[index] ~= "0" and ARGV[index] ~= "1" then
+    return redis.error_reply("invalid reservation policy flag")
+  end
+end
+if not string.match(userId or "", "^[0-9a-f%-]+$")
+  or not string.match(betId or "", "^[0-9a-f%-]+$")
+  or not string.match(currency or "", "^[A-Z]+$") then
+  return redis.error_reply("invalid reservation identity")
+end
 local selections, seen = {}, {}
 for index = 1, selectionCount do
   local selectionId = ARGV[33 + index]
-  if not selectionId or seen[selectionId] then return redis.error_reply("invalid selection") end
+  if not selectionId or seen[selectionId] or not string.match(selectionId, "^[0-9a-f%-]+$") then
+    return redis.error_reply("invalid selection")
+  end
   seen[selectionId] = true; table.insert(selections, selectionId)
 end
 local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
@@ -72,6 +90,7 @@ for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[2], 0, -1)) do
       or not oldCount or not oldCurrency or not oldSelections then
       return redis.error_reply("corrupt active lifecycle")
     end
+    if not string.match(oldCurrency, "^[A-Z]+$") then return redis.error_reply("corrupt currency") end
     local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
     local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
     local cleanupError = typeError(entries, "zset") or typeError(sum, "string")
@@ -88,8 +107,11 @@ for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[2], 0, -1)) do
     end
     table.insert(cleanups, {activeBetId, lifecycle, state, oldStakeText, oldCountText,
       entries, sum, oldSelections})
-    stakeDecrements[sum] = (stakeDecrements[sum] or 0) + oldStake
-    selectionDecrement = selectionDecrement + oldCount
+    local previous = stakeDecrements[sum] or 0
+    if previous > maxExact - oldStake or selectionDecrement > maxExact - oldCount then
+      return redis.error_reply("active cleanup exceeds exact range")
+    end
+    stakeDecrements[sum] = previous + oldStake; selectionDecrement = selectionDecrement + oldCount
   end
 end
 for sum, value in pairs(stakeDecrements) do
@@ -120,6 +142,9 @@ if selectionDecrement > 0 then decrement(KEYS[6], selectionDecrement) end
 if #cleanups > 0 then decrement(KEYS[18], #cleanups) end
 if redis.call("ZCARD", KEYS[2]) == 0 then redis.call("DEL", KEYS[2]) end
 if redis.call("ZCARD", KEYS[5]) == 0 then redis.call("DEL", KEYS[5]) end
+for _, item in ipairs(cleanups) do
+  if redis.call("ZCARD", item[6]) == 0 then redis.call("DEL", item[6]) end
+end
 local existing = redis.call("HGET", KEYS[1], "state")
 if existing then
   if redis.call("HGET", KEYS[1], "fingerprint") ~= fingerprint then
@@ -202,6 +227,10 @@ local expiresAt = checkedAdd(now, lease)
 if not nextStake or not nextSelections or not nextGauge or not expiresAt then
   return redis.error_reply("active reservation total exceeds exact range")
 end
+if redis.call("ZSCORE", KEYS[2], betId) or redis.call("ZSCORE", KEYS[3], betId .. "|" .. stakeText)
+  or redis.call("ZSCORE", KEYS[5], betId .. "|" .. countText) then
+  return redis.error_reply("orphan active reservation footprint")
+end
 local names = {"STAKE_DAILY", "STAKE_WEEKLY", "STAKE_MONTHLY"}
 for index = 1, 3 do
   local committed, committedError = readWindow(KEYS[6 + index * 2], KEYS[7 + index * 2],


## `test(reservation): reject corrupt expiry footprints`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCorruptFootprintScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCorruptFootprintScriptTest.java
new file mode 100644
index 0000000..4edb54e
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCorruptFootprintScriptTest.java
@@ -0,0 +1,45 @@
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
+class ReservationCorruptFootprintScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void rejectsIncompleteExpiryCleanupWithoutMutation() {
+    RiskCheckCommand expired = command(700, 40, Currency.KRW, selection(700));
+    RiskCheckCommand next =
+        new RiskCheckCommand(
+            USER,
+            BetId.of(new UUID(0, 701)),
+            new Money(10, Currency.KRW),
+            List.of(selection(701)),
+            NOW.plus(RiskReservationProperties.DEFAULT_LEASE));
+    assertThat(reserve(expired).approved()).isTrue();
+    redis.delete(ReservationKeys.activeSelection(USER, selection(700)));
+
+    assertThatThrownBy(() -> execute(request(next)))
+        .hasStackTraceContaining("missing selection footprint");
+
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(expired.betId()), "state"))
+        .isEqualTo("RESERVED");
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(next.betId()))).isFalse();
+    assertThat(redis.opsForZSet().range(ReservationKeys.activeBets(USER), 0, -1))
+        .containsExactly(expired.betId().value().toString());
+    assertThat(redis.opsForValue().get(ReservationKeys.activeStakes(USER, Currency.KRW).sum()))
+        .isEqualTo("40");
+    assertThat(redis.opsForValue().get(ReservationKeys.activeSelections(USER).sum()))
+        .isEqualTo("1");
+    assertThat(redis.opsForValue().get(ReservationKeys.ACTIVE_COUNT)).isEqualTo("1");
+  }
+}


