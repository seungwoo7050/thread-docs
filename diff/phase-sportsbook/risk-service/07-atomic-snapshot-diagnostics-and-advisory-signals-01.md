# 원자적 스냅샷 기반 비권위 진단과 관측 신호

## `feat(snapshot): define combined risk facts`

diff --git a/src/main/java/com/sportsbook/risk/snapshot/LimitSnapshot.java b/src/main/java/com/sportsbook/risk/snapshot/LimitSnapshot.java
new file mode 100644
index 0000000..0b63e21
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/LimitSnapshot.java
@@ -0,0 +1,43 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.EnumMap;
+import java.util.Map;
+import java.util.Objects;
+
+/** Captured committed, active, and override values for all rolling dimensions. */
+public record LimitSnapshot(Map<LimitType, SnapshotSlot<Value>> values) {
+  public LimitSnapshot {
+    Objects.requireNonNull(values, "values");
+    EnumMap<LimitType, SnapshotSlot<Value>> copy = new EnumMap<>(LimitType.class);
+    copy.putAll(values);
+    for (LimitType type : LimitType.values()) {
+      Objects.requireNonNull(copy.get(type), "missing limit snapshot: " + type);
+    }
+    values = Map.copyOf(copy);
+  }
+
+  public Value require(LimitType type) {
+    return values.get(Objects.requireNonNull(type, "type")).valueOrThrow();
+  }
+
+  public record Value(long committed, long active, Long override) {
+    public Value {
+      SafeRedisNumber.requireNonNegative(committed, "committed");
+      SafeRedisNumber.requireNonNegative(active, "active");
+      if (override != null) {
+        SafeRedisNumber.requireNonNegative(override, "override");
+      }
+    }
+
+    public long current() {
+      return SafeRedisNumber.add(committed, active, "current risk capacity");
+    }
+
+    public long effectiveLimit(long deployedDefault) {
+      SafeRedisNumber.requireNonNegative(deployedDefault, "deployed default");
+      return override == null ? deployedDefault : override;
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshot.java b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshot.java
new file mode 100644
index 0000000..5a3e32d
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshot.java
@@ -0,0 +1,11 @@
+package com.sportsbook.risk.snapshot;
+
+import java.util.Objects;
+
+/** One atomic Redis view used by diagnostics and pure policy evaluation. */
+public record RiskSnapshot(LimitSnapshot limits, PatternSnapshot patterns) {
+  public RiskSnapshot {
+    Objects.requireNonNull(limits, "limits");
+    Objects.requireNonNull(patterns, "patterns");
+  }
+}


## `feat(snapshot): assemble atomic snapshot inputs`

diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotScriptRequest.java b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotScriptRequest.java
new file mode 100644
index 0000000..cb889ef
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotScriptRequest.java
@@ -0,0 +1,84 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideKeys;
+import com.sportsbook.risk.pattern.HistoryKeys;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.reservation.ReservationKeys;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Objects;
+
+/** Deterministic key and argument order consumed by {@code risk-snapshot.lua}. */
+public record RiskSnapshotScriptRequest(List<String> keys, List<String> arguments) {
+  public RiskSnapshotScriptRequest {
+    keys = List.copyOf(Objects.requireNonNull(keys, "keys"));
+    arguments = List.copyOf(Objects.requireNonNull(arguments, "arguments"));
+  }
+
+  public static RiskSnapshotScriptRequest from(
+      PatternContext context,
+      RiskPatternProperties patterns,
+      RiskReservationProperties reservations) {
+    Objects.requireNonNull(context, "context");
+    Objects.requireNonNull(patterns, "patterns");
+    Objects.requireNonNull(reservations, "reservations");
+    UserId userId = context.userId();
+    Currency currency = context.stake().currency();
+    List<String> keys = new ArrayList<>();
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_DAILY, currency));
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_WEEKLY, currency));
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_MONTHLY, currency));
+    add(keys, LimitKeys.selections(userId));
+    keys.add(LimitOverrideKeys.user(userId));
+    keys.add(ReservationKeys.activeBets(userId));
+    add(keys, ReservationKeys.activeStakes(userId, currency));
+    add(keys, ReservationKeys.activeSelections(userId));
+    keys.add(HistoryKeys.bets(userId));
+    keys.add(HistoryKeys.stakes(userId, currency));
+    keys.add(ReservationKeys.ACTIVE_COUNT);
+    context.selections().stream()
+        .map(selection -> HistoryKeys.selection(userId, selection))
+        .forEach(keys::add);
+    context.selections().stream()
+        .map(selection -> ReservationKeys.activeSelection(userId, selection))
+        .forEach(keys::add);
+
+    List<String> arguments = new ArrayList<>();
+    arguments.add(Long.toString(context.evaluatedAt().toEpochMilli()));
+    arguments.add(Long.toString(reservations.retention().toMillis()));
+    arguments.add(Long.toString(LimitType.STAKE_DAILY.window().toMillis()));
+    arguments.add(Long.toString(LimitType.STAKE_WEEKLY.window().toMillis()));
+    arguments.add(Long.toString(LimitType.STAKE_MONTHLY.window().toMillis()));
+    arguments.add(Long.toString(LimitType.SELECTIONS_PER_MINUTE.window().toMillis()));
+    arguments.add(enabled(patterns.rapidBetting().enabled()));
+    arguments.add(Long.toString(patterns.rapidBetting().window().toMillis()));
+    arguments.add(enabled(patterns.suddenStake().enabled()));
+    arguments.add(Integer.toString(patterns.suddenStake().lookbackBets()));
+    arguments.add(enabled(patterns.repeatedSelection().enabled()));
+    arguments.add(Long.toString(patterns.repeatedSelection().window().toMillis()));
+    arguments.add(userId.value().toString());
+    arguments.add(currency.name());
+    arguments.add(Integer.toString(context.selections().size()));
+    context.selections().stream()
+        .map(SelectionId::value)
+        .map(Object::toString)
+        .forEach(arguments::add);
+    return new RiskSnapshotScriptRequest(keys, arguments);
+  }
+
+  private static void add(List<String> keys, LimitKeys.Keys pair) {
+    keys.add(pair.entries());
+    keys.add(pair.sum());
+  }
+
+  private static String enabled(boolean value) {
+    return value ? "1" : "0";
+  }
+}


## `feat(snapshot): read committed limits atomically`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
new file mode 100644
index 0000000..aaee7a4
--- /dev/null
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -0,0 +1,86 @@
+local maxExact = 9007199254740991
+local now = tonumber(ARGV[1])
+local currency = ARGV[14]
+local count = tonumber(ARGV[15])
+
+local function exact(text, positive)
+  if not text or not string.match(text, "^%d+$") then return nil end
+  local value = tonumber(text)
+  if not value or value > maxExact or (positive and value <= 0) then return nil end
+  return value
+end
+
+local function keyType(key) return redis.call("TYPE", key).ok end
+
+local function typeError(key, expected)
+  local actual = keyType(key)
+  if actual ~= "none" and actual ~= expected then return "wrong Redis type for " .. key end
+  return nil
+end
+
+local function failure(message) return {ok = false, error = message} end
+
+local function amount(member)
+  return exact(string.match(member, "|([0-9]+)$"), true)
+end
+
+local function capture(entries, sum, window)
+  local errorText = typeError(entries, "zset") or typeError(sum, "string")
+  if errorText then return failure(errorText) end
+  if keyType(entries) == "none" then
+    redis.call("DEL", sum)
+    return {ok = true, value = "0"}
+  end
+  local total = exact(redis.call("GET", sum), false)
+  if not total then return failure("missing or corrupt rolling sum") end
+  local expired = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window)
+  local removed = 0
+  for _, member in ipairs(expired) do
+    local value = amount(member)
+    if not value or removed > maxExact - value then return failure("corrupt rolling member") end
+    removed = removed + value
+  end
+  if removed > total then return failure("rolling sum underflow") end
+  if #expired > 0 then redis.call("ZREMRANGEBYSCORE", entries, "-inf", now - window) end
+  total = total - removed
+  if redis.call("ZCARD", entries) == 0 then
+    redis.call("DEL", entries, sum)
+    total = 0
+  else
+    redis.call("SET", sum, string.format("%.0f", total), "PX", window + 300000)
+    redis.call("PEXPIRE", entries, window + 300000)
+  end
+  return {ok = true, value = string.format("%.0f", total)}
+end
+
+local function limitSlot(counter, field)
+  if not counter.ok then return counter end
+  local errorText = typeError(KEYS[9], "hash")
+  if errorText then return failure(errorText) end
+  local raw = redis.call("HGET", KEYS[9], field)
+  if raw and not exact(raw, false) then return failure("corrupt limit override") end
+  return {ok = true, committed = counter.value, active = "0", override = raw or cjson.null}
+end
+
+if not now or now < 0 or now > maxExact or not count or count < 1
+  or #KEYS ~= 17 + count * 2 then
+  return redis.error_reply("invalid snapshot request")
+end
+local limits = {
+  STAKE_DAILY = limitSlot(capture(KEYS[1], KEYS[2], tonumber(ARGV[3])),
+    "STAKE_DAILY:" .. currency),
+  STAKE_WEEKLY = limitSlot(capture(KEYS[3], KEYS[4], tonumber(ARGV[4])),
+    "STAKE_WEEKLY:" .. currency),
+  STAKE_MONTHLY = limitSlot(capture(KEYS[5], KEYS[6], tonumber(ARGV[5])),
+    "STAKE_MONTHLY:" .. currency),
+  SELECTIONS_PER_MINUTE = limitSlot(capture(KEYS[7], KEYS[8], tonumber(ARGV[6])),
+    "SELECTIONS_PER_MINUTE")
+}
+local selectionFacts = cjson.decode("[]")
+for index = 1, count do
+  table.insert(selectionFacts, {selectionId = ARGV[15 + index],
+    slot = {ok = true, value = "0"}})
+end
+return cjson.encode({version = "1", expired = "0", limits = limits,
+  patterns = {rapid = {ok = true, value = "0"},
+    stakes = {ok = true, value = ""}, selections = selectionFacts}})


## `feat(snapshot): include confirmed pattern facts`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index aaee7a4..e7d8600 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -76,11 +76,35 @@ local limits = {
   SELECTIONS_PER_MINUTE = limitSlot(capture(KEYS[7], KEYS[8], tonumber(ARGV[6])),
     "SELECTIONS_PER_MINUTE")
 }
+
+local function confirmedCount(key, enabled, window)
+  if not enabled then return {ok = true, value = "0"} end
+  local errorText = typeError(key, "zset")
+  if errorText then return failure(errorText) end
+  local value = redis.call("ZCOUNT", key, "(" .. (now - window), "+inf")
+  if value > maxExact then return failure("pattern count exceeds exact range") end
+  return {ok = true, value = string.format("%.0f", value)}
+end
+
+local function confirmedStakes()
+  if ARGV[9] ~= "1" then return {ok = true, value = ""} end
+  local errorText = typeError(KEYS[16], "zset")
+  if errorText then return failure(errorText) end
+  local raw = redis.call("ZREVRANGE", KEYS[16], 0, tonumber(ARGV[10]) - 1)
+  local values = cjson.decode("[]")
+  for index = #raw, 1, -1 do
+    local encoded = string.match(raw[index], "|([0-9]+)$")
+    if not exact(encoded, true) then return failure("corrupt stake history member") end
+    table.insert(values, encoded)
+  end
+  return {ok = true, value = table.concat(values, ",")}
+end
+
+local rapid = confirmedCount(KEYS[15], ARGV[7] == "1", tonumber(ARGV[8]))
 local selectionFacts = cjson.decode("[]")
 for index = 1, count do
   table.insert(selectionFacts, {selectionId = ARGV[15 + index],
-    slot = {ok = true, value = "0"}})
+    slot = confirmedCount(KEYS[17 + index], ARGV[11] == "1", tonumber(ARGV[12]))})
 end
 return cjson.encode({version = "1", expired = "0", limits = limits,
-  patterns = {rapid = {ok = true, value = "0"},
-    stakes = {ok = true, value = ""}, selections = selectionFacts}})
+  patterns = {rapid = rapid, stakes = confirmedStakes(), selections = selectionFacts}})


## `feat(snapshot): include active reservation capacity`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index e7d8600..9fd059e 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -53,58 +53,89 @@ local function capture(entries, sum, window)
   return {ok = true, value = string.format("%.0f", total)}
 end
 
-local function limitSlot(counter, field)
+local function active(entries, sum)
+  local errorText = typeError(entries, "zset") or typeError(sum, "string")
+  if errorText then return nil, errorText end
+  if keyType(entries) == "none" then redis.call("DEL", sum); return "0", nil end
+  local value = exact(redis.call("GET", sum), false)
+  if not value then return nil, "missing or corrupt active sum" end
+  return string.format("%.0f", value), nil
+end
+
+local function limitSlot(counter, activeValue, activeError, field)
   if not counter.ok then return counter end
+  if activeError then return failure(activeError) end
   local errorText = typeError(KEYS[9], "hash")
   if errorText then return failure(errorText) end
   local raw = redis.call("HGET", KEYS[9], field)
   if raw and not exact(raw, false) then return failure("corrupt limit override") end
-  return {ok = true, committed = counter.value, active = "0", override = raw or cjson.null}
+  return {ok = true, committed = counter.value, active = activeValue,
+    override = raw or cjson.null}
 end
 
 if not now or now < 0 or now > maxExact or not count or count < 1
   or #KEYS ~= 17 + count * 2 then
   return redis.error_reply("invalid snapshot request")
 end
+local activeStake, activeStakeError = active(KEYS[11], KEYS[12])
+local activeSelections, activeSelectionError = active(KEYS[13], KEYS[14])
 local limits = {
   STAKE_DAILY = limitSlot(capture(KEYS[1], KEYS[2], tonumber(ARGV[3])),
+    activeStake, activeStakeError,
     "STAKE_DAILY:" .. currency),
   STAKE_WEEKLY = limitSlot(capture(KEYS[3], KEYS[4], tonumber(ARGV[4])),
+    activeStake, activeStakeError,
     "STAKE_WEEKLY:" .. currency),
   STAKE_MONTHLY = limitSlot(capture(KEYS[5], KEYS[6], tonumber(ARGV[5])),
+    activeStake, activeStakeError,
     "STAKE_MONTHLY:" .. currency),
   SELECTIONS_PER_MINUTE = limitSlot(capture(KEYS[7], KEYS[8], tonumber(ARGV[6])),
+    activeSelections, activeSelectionError,
     "SELECTIONS_PER_MINUTE")
 }
 
-local function confirmedCount(key, enabled, window)
+local function patternCount(confirmed, activeKey, enabled, window)
   if not enabled then return {ok = true, value = "0"} end
-  local errorText = typeError(key, "zset")
+  local errorText = typeError(confirmed, "zset") or typeError(activeKey, "zset")
   if errorText then return failure(errorText) end
-  local value = redis.call("ZCOUNT", key, "(" .. (now - window), "+inf")
+  local cutoff = "(" .. (now - window)
+  local value = redis.call("ZCOUNT", confirmed, cutoff, "+inf")
+    + redis.call("ZCOUNT", activeKey, cutoff, "+inf")
   if value > maxExact then return failure("pattern count exceeds exact range") end
   return {ok = true, value = string.format("%.0f", value)}
 end
 
 local function confirmedStakes()
   if ARGV[9] ~= "1" then return {ok = true, value = ""} end
-  local errorText = typeError(KEYS[16], "zset")
+  local errorText = typeError(KEYS[16], "zset") or typeError(KEYS[11], "zset")
   if errorText then return failure(errorText) end
-  local raw = redis.call("ZREVRANGE", KEYS[16], 0, tonumber(ARGV[10]) - 1)
+  local samples = {}
+  for _, key in ipairs({KEYS[16], KEYS[11]}) do
+    local raw = redis.call("ZRANGE", key, 0, -1, "WITHSCORES")
+    for index = 1, #raw, 2 do
+      local encoded = string.match(raw[index], "|([0-9]+)$")
+      if not exact(encoded, true) then return failure("corrupt stake history member") end
+      table.insert(samples, {value = encoded, score = tonumber(raw[index + 1]), member = raw[index]})
+    end
+  end
+  table.sort(samples, function(left, right)
+    if left.score == right.score then return left.member < right.member end
+    return left.score < right.score
+  end)
   local values = cjson.decode("[]")
-  for index = #raw, 1, -1 do
-    local encoded = string.match(raw[index], "|([0-9]+)$")
-    if not exact(encoded, true) then return failure("corrupt stake history member") end
-    table.insert(values, encoded)
+  local first = math.max(1, #samples - tonumber(ARGV[10]) + 1)
+  for index = first, #samples do
+    table.insert(values, samples[index].value)
   end
   return {ok = true, value = table.concat(values, ",")}
 end
 
-local rapid = confirmedCount(KEYS[15], ARGV[7] == "1", tonumber(ARGV[8]))
+local rapid = patternCount(KEYS[15], KEYS[10], ARGV[7] == "1", tonumber(ARGV[8]))
 local selectionFacts = cjson.decode("[]")
 for index = 1, count do
   table.insert(selectionFacts, {selectionId = ARGV[15 + index],
-    slot = confirmedCount(KEYS[17 + index], ARGV[11] == "1", tonumber(ARGV[12]))})
+    slot = patternCount(KEYS[17 + index], KEYS[17 + count + index],
+      ARGV[11] == "1", tonumber(ARGV[12]))})
 end
 return cjson.encode({version = "1", expired = "0", limits = limits,
   patterns = {rapid = rapid, stakes = confirmedStakes(), selections = selectionFacts}})


## `feat(snapshot): expire stale reservation footprints`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index 9fd059e..33e15eb 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -1,7 +1,10 @@
 local maxExact = 9007199254740991
 local now = tonumber(ARGV[1])
+local retention = tonumber(ARGV[2])
+local userId = ARGV[13]
 local currency = ARGV[14]
 local count = tonumber(ARGV[15])
+local expiredCount = 0
 
 local function exact(text, positive)
   if not text or not string.match(text, "^%d+$") then return nil end
@@ -20,6 +23,20 @@ end
 
 local function failure(message) return {ok = false, error = message} end
 
+local function split(encoded, expected)
+  local values = {}
+  for value in string.gmatch(encoded or "", "[^,]+") do table.insert(values, value) end
+  if #values ~= expected then return nil end
+  return values
+end
+
+local function decrement(key, amount)
+  local current = exact(redis.call("GET", key), false)
+  local nextValue = current - amount
+  if nextValue == 0 then redis.call("DEL", key)
+  else redis.call("SET", key, string.format("%.0f", nextValue)) end
+end
+
 local function amount(member)
   return exact(string.match(member, "|([0-9]+)$"), true)
 end
@@ -74,9 +91,79 @@ local function limitSlot(counter, activeValue, activeError, field)
 end
 
 if not now or now < 0 or now > maxExact or not count or count < 1
-  or #KEYS ~= 17 + count * 2 then
+  or not retention or retention <= 0 or #KEYS ~= 17 + count * 2 then
   return redis.error_reply("invalid snapshot request")
 end
+
+local activeBase = "risk:reservations:user:{" .. userId .. "}"
+local plans, stakeTotals, selectionTotal = {}, {}, 0
+local cleanupError = typeError(KEYS[10], "zset") or typeError(KEYS[13], "zset")
+  or typeError(KEYS[14], "string") or typeError(KEYS[17], "string")
+if cleanupError then return redis.error_reply(cleanupError) end
+for _, activeBetId in ipairs(redis.call("ZRANGE", KEYS[10], 0, -1)) do
+  local lifecycle = "risk:reservation:" .. activeBetId
+  if keyType(lifecycle) ~= "hash" then return redis.error_reply("missing lifecycle") end
+  local state = redis.call("HGET", lifecycle, "state")
+  local expiresAt = exact(redis.call("HGET", lifecycle, "expiresAt"), false)
+  if not expiresAt then return redis.error_reply("corrupt lifecycle expiry") end
+  if state ~= "RESERVED" or expiresAt <= now then
+    local stakeText = redis.call("HGET", lifecycle, "stake")
+    local countText = redis.call("HGET", lifecycle, "selectionCount")
+    local oldCurrency = redis.call("HGET", lifecycle, "currency")
+    local stake = exact(stakeText, true)
+    local oldCount = exact(countText, true)
+    local oldSelections = split(redis.call("HGET", lifecycle, "selections"), oldCount or -1)
+    if redis.call("HGET", lifecycle, "userId") ~= userId or not stake
+      or not oldCount or not oldCurrency or not oldSelections then
+      return redis.error_reply("corrupt active lifecycle")
+    end
+    local stakeBase = activeBase .. ":stakes:" .. string.lower(oldCurrency)
+    local entries, sum = stakeBase .. ":entries", stakeBase .. ":sum"
+    local planError = typeError(entries, "zset") or typeError(sum, "string")
+    if planError or not redis.call("ZSCORE", entries, activeBetId .. "|" .. stakeText)
+      or not redis.call("ZSCORE", KEYS[13], activeBetId .. "|" .. countText) then
+      return redis.error_reply(planError or "missing active footprint")
+    end
+    for _, selectionId in ipairs(oldSelections) do
+      local selectionKey = activeBase .. ":selection:" .. selectionId
+      local selectionError = typeError(selectionKey, "zset")
+      if selectionError or not redis.call("ZSCORE", selectionKey, activeBetId) then
+        return redis.error_reply(selectionError or "missing selection footprint")
+      end
+    end
+    table.insert(plans, {activeBetId, lifecycle, state, stakeText, countText,
+      entries, sum, oldSelections})
+    stakeTotals[sum] = (stakeTotals[sum] or 0) + stake
+    selectionTotal = selectionTotal + oldCount
+  end
+end
+for sum, value in pairs(stakeTotals) do
+  local current = exact(redis.call("GET", sum), false)
+  if not current or current < value then return redis.error_reply("corrupt active stake sum") end
+end
+if #plans > 0 then
+  local currentSelections = exact(redis.call("GET", KEYS[14]), false)
+  local gauge = exact(redis.call("GET", KEYS[17]), false)
+  if not currentSelections or currentSelections < selectionTotal or not gauge or gauge < #plans then
+    return redis.error_reply("corrupt active totals")
+  end
+end
+for _, plan in ipairs(plans) do
+  redis.call("ZREM", KEYS[10], plan[1])
+  redis.call("ZREM", plan[6], plan[1] .. "|" .. plan[4])
+  redis.call("ZREM", KEYS[13], plan[1] .. "|" .. plan[5])
+  for _, selectionId in ipairs(plan[8]) do
+    redis.call("ZREM", activeBase .. ":selection:" .. selectionId, plan[1])
+  end
+  if plan[3] == "RESERVED" then
+    redis.call("HSET", plan[2], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
+    redis.call("PEXPIRE", plan[2], retention)
+    expiredCount = expiredCount + 1
+  end
+end
+for sum, value in pairs(stakeTotals) do decrement(sum, value) end
+if selectionTotal > 0 then decrement(KEYS[14], selectionTotal) end
+if #plans > 0 then decrement(KEYS[17], #plans) end
 local activeStake, activeStakeError = active(KEYS[11], KEYS[12])
 local activeSelections, activeSelectionError = active(KEYS[13], KEYS[14])
 local limits = {
@@ -137,5 +224,5 @@ for index = 1, count do
     slot = patternCount(KEYS[17 + index], KEYS[17 + count + index],
       ARGV[11] == "1", tonumber(ARGV[12]))})
 end
-return cjson.encode({version = "1", expired = "0", limits = limits,
+return cjson.encode({version = "1", expired = string.format("%.0f", expiredCount), limits = limits,
   patterns = {rapid = rapid, stakes = confirmedStakes(), selections = selectionFacts}})


