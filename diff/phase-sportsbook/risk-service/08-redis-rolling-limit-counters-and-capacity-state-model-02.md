## `feat(reservation): enforce currency rolling capacity`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index ae6a28d..a95fe77 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -69,8 +69,34 @@ if existing then
   return redis.error_reply("unknown reservation state")
 end
 
-local singleRaw = redis.call("HGET", KEYS[7], "SINGLE_BET_MAX:" .. currency) or ARGV[11]
-local singleLimit = exact(singleRaw, false)
+local function effective(field, fallback)
+  return exact(redis.call("HGET", KEYS[7], field) or fallback, false)
+end
+local function memberAmount(member)
+  return exact(string.match(member, "|([0-9]+)$"), true)
+end
+local function readWindow(entries, sum, window)
+  local errorText = typeError(entries, "zset") or typeError(sum, "string")
+  if errorText then return nil, errorText end
+  if keyType(entries) == "none" then redis.call("DEL", sum); return 0, nil end
+  local total = exact(redis.call("GET", sum), false)
+  if not total then return nil, "missing or corrupt rolling sum" end
+  local expired, removed = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window), 0
+  for _, member in ipairs(expired) do
+    local value = memberAmount(member)
+    if not value or removed > maxExact - value then return nil, "corrupt rolling member" end
+    removed = removed + value
+  end
+  if removed > total then return nil, "rolling sum underflow" end
+  if #expired > 0 then redis.call("ZREMRANGEBYSCORE", entries, "-inf", now - window) end
+  total = total - removed
+  if redis.call("ZCARD", entries) == 0 then redis.call("DEL", entries, sum); return 0, nil end
+  redis.call("SET", sum, string.format("%.0f", total), "PX", window + 300000)
+  redis.call("PEXPIRE", entries, window + 300000)
+  return total, nil
+end
+
+local singleLimit = effective("SINGLE_BET_MAX:" .. currency, ARGV[11])
 if not singleLimit then return redis.error_reply("corrupt single-bet limit") end
 local function persist(state, patternsJson)
   redis.call("HSET", KEYS[1], "state", state, "fingerprint", fingerprint, "token", fingerprint,
@@ -97,6 +123,24 @@ local expiresAt = checkedAdd(now, lease)
 if not nextStake or not nextSelections or not nextGauge or not expiresAt then
   return redis.error_reply("active reservation total exceeds exact range")
 end
+local names = {"STAKE_DAILY", "STAKE_WEEKLY", "STAKE_MONTHLY"}
+for index = 1, 3 do
+  local committed, committedError = readWindow(KEYS[6 + index * 2], KEYS[7 + index * 2],
+    tonumber(ARGV[15 + index]))
+  local limit = effective(names[index] .. ":" .. currency, ARGV[11 + index])
+  local current = committed and checkedAdd(committed, activeStake) or nil
+  local candidate = current and checkedAdd(current, stake) or nil
+  if committedError or not limit or not candidate then
+    return redis.error_reply(committedError or "invalid rolling capacity")
+  end
+  if candidate > limit then
+    persist("REJECTED", "[]")
+    local code = names[index] .. "_LIMIT_EXCEEDED"
+    redis.call("HSET", KEYS[1], "rejection", code, "rejectedAt", string.format("%.0f", now))
+    return response({status = "REJECTED", rejection = code,
+      replayed = false, patternsJson = "[]"})
+  end
+end
 persist("RESERVED", "[]")
 redis.call("HSET", KEYS[1], "reservedAt", string.format("%.0f", now),
   "expiresAt", string.format("%.0f", expiresAt))


## `test(reservation): verify concurrent rolling capacity`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationRollingCapacityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationRollingCapacityScriptTest.java
new file mode 100644
index 0000000..26a0498
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationRollingCapacityScriptTest.java
@@ -0,0 +1,76 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import java.util.List;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class ReservationRollingCapacityScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void isolatesActiveCapacityByCurrency() {
+    RiskPatternProperties patterns = new RiskPatternProperties(null, null, null);
+
+    assertThat(
+            execute(request(command(30, 60, Currency.KRW), limits(100), patterns))
+                .decision()
+                .approved())
+        .isTrue();
+    assertThat(
+            execute(request(command(31, 50, Currency.KRW), limits(100), patterns))
+                .decision()
+                .rejection())
+        .isEqualTo("STAKE_DAILY_LIMIT_EXCEEDED");
+    assertThat(
+            execute(request(command(32, 50, Currency.USD), limits(100), patterns))
+                .decision()
+                .approved())
+        .isTrue();
+  }
+
+  @Test
+  void serializesParallelRequestsAtTheLastAvailableCapacity() throws Exception {
+    var executor = Executors.newFixedThreadPool(8);
+    CountDownLatch start = new CountDownLatch(1);
+    try {
+      List<Future<ReservationDecision>> futures =
+          IntStream.range(0, 20)
+              .mapToObj(
+                  index ->
+                      executor.submit(
+                          () -> {
+                            start.await();
+                            return execute(
+                                    request(
+                                        command(
+                                            100 + index, 10, Currency.KRW, selection(100 + index)),
+                                        limits(100),
+                                        new RiskPatternProperties(null, null, null)))
+                                .decision();
+                          }))
+              .toList();
+      start.countDown();
+
+      List<ReservationDecision> decisions =
+          futures.stream().map(ReservationRollingCapacityScriptTest::result).toList();
+      assertThat(decisions.stream().filter(ReservationDecision::approved)).hasSize(10);
+      assertThat(decisions.stream().filter(decision -> !decision.approved()))
+          .allMatch(decision -> "STAKE_DAILY_LIMIT_EXCEEDED".equals(decision.rejection()));
+    } finally {
+      executor.shutdownNow();
+    }
+  }
+
+  private static ReservationDecision result(Future<ReservationDecision> future) {
+    try {
+      return future.get();
+    } catch (Exception failure) {
+      throw new AssertionError(failure);
+    }
+  }
+}


## `feat(reservation): enforce selection capacity`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index a95fe77..3b94e26 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -141,6 +141,21 @@ for index = 1, 3 do
       replayed = false, patternsJson = "[]"})
   end
 end
+local committedSelections, committedSelectionError =
+  readWindow(KEYS[14], KEYS[15], tonumber(ARGV[19]))
+local selectionLimit = effective("SELECTIONS_PER_MINUTE", ARGV[15])
+local selectionCurrent = committedSelections and checkedAdd(committedSelections, activeSelections) or nil
+local selectionCandidate = selectionCurrent and checkedAdd(selectionCurrent, selectionCount) or nil
+if committedSelectionError or not selectionLimit or not selectionCandidate then
+  return redis.error_reply(committedSelectionError or "invalid selection capacity")
+end
+if selectionCandidate > selectionLimit then
+  persist("REJECTED", "[]")
+  redis.call("HSET", KEYS[1], "rejection", "SELECTIONS_PER_MINUTE_LIMIT_EXCEEDED",
+    "rejectedAt", string.format("%.0f", now))
+  return response({status = "REJECTED", rejection = "SELECTIONS_PER_MINUTE_LIMIT_EXCEEDED",
+    replayed = false, patternsJson = "[]"})
+end
 persist("RESERVED", "[]")
 redis.call("HSET", KEYS[1], "reservedAt", string.format("%.0f", now),
   "expiresAt", string.format("%.0f", expiresAt))


## `feat(reservation): commit rolling risk capacity`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
index f071421..6ff22fd 100644
--- a/src/main/resources/scripts/risk-commit.lua
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -85,13 +85,55 @@ local function removeActive()
   if redis.call("ZCARD", stakeEntries) == 0 then redis.call("DEL", stakeEntries) end
   if redis.call("ZCARD", selectionEntries) == 0 then redis.call("DEL", selectionEntries) end
 end
-
-removeActive()
 if expiresAt <= now then
+  removeActive()
   redis.call("HSET", KEYS[1], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
   redis.call("PEXPIRE", KEYS[1], retention)
   return "EXPIRED"
 end
+
+local function memberAmount(member) return exact(string.match(member, "|([0-9]+)$"), true) end
+local function planWindow(entries, sum, amount, window)
+  local errorValue = typeError(entries, "zset") or typeError(sum, "string")
+  if errorValue then return nil, errorValue end
+  local total = 0
+  if keyType(entries) ~= "none" then
+    total = exact(redis.call("GET", sum), false)
+    if not total then return nil, "missing or corrupt rolling sum" end
+  end
+  local expired = redis.call("ZRANGEBYSCORE", entries, "-inf", now - window)
+  for _, member in ipairs(expired) do
+    local value = memberAmount(member)
+    if not value or total < value then return nil, "corrupt rolling member" end
+    total = total - value
+  end
+  if total > maxExact - amount then return nil, "rolling sum exceeds exact range" end
+  local amountText = string.format("%.0f", amount)
+  if redis.call("ZSCORE", entries, betId .. "|" .. amountText) then
+    return nil, "reservation already exists in committed window"
+  end
+  return {entries, sum, window, amountText, total + amount}
+end
+
+local limitBase, plans = "risk:limit:{" .. userId .. "}:", {}
+local currencyLower = string.lower(currency)
+local dimensions = {{"stake-daily:" .. currencyLower, ARGV[5], stake},
+  {"stake-weekly:" .. currencyLower, ARGV[6], stake}, {"stake-monthly:" .. currencyLower, ARGV[7], stake},
+  {"selections-per-minute", ARGV[8], count}}
+for _, dimension in ipairs(dimensions) do
+  local prefix = limitBase .. dimension[1]
+  local plan, planError = planWindow(prefix .. ":entries", prefix .. ":sum", dimension[3], tonumber(dimension[2]))
+  if planError then return redis.error_reply(planError) end
+  table.insert(plans, plan)
+end
+
+removeActive()
+for _, plan in ipairs(plans) do
+  redis.call("ZREMRANGEBYSCORE", plan[1], "-inf", now - plan[3])
+  redis.call("ZADD", plan[1], now, betId .. "|" .. plan[4])
+  redis.call("SET", plan[2], string.format("%.0f", plan[5]), "PX", plan[3] + 300000)
+  redis.call("PEXPIRE", plan[1], plan[3] + 300000)
+end
 redis.call("HSET", KEYS[1], "state", "COMMITTED", "committedAt", string.format("%.0f", now))
 redis.call("PEXPIRE", KEYS[1], retention)
 return "APPLIED"


## `test(reservation): verify committed rolling capacity`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitCapacityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitCapacityScriptTest.java
new file mode 100644
index 0000000..b2185f0
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitCapacityScriptTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitCapacityScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void movesReservedExposureIntoCommittedWindows() {
+    var command = command(1_200, 40, Currency.USD, selection(1_200), selection(1_201));
+    ReservationDecision reserved = reserve(command);
+
+    assertThat(commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.APPLIED);
+
+    assertMonetaryWindow(command, LimitType.STAKE_DAILY);
+    assertMonetaryWindow(command, LimitType.STAKE_WEEKLY);
+    assertMonetaryWindow(command, LimitType.STAKE_MONTHLY);
+    LimitKeys.Keys selections = LimitKeys.selections(command.userId());
+    assertThat(redis.opsForValue().get(selections.sum())).isEqualTo("2");
+    assertThat(redis.opsForZSet().range(selections.entries(), 0, -1))
+        .containsExactly(LimitKeys.member(command.betId(), 2));
+  }
+
+  private void assertMonetaryWindow(
+      com.sportsbook.risk.service.RiskCheckCommand command, LimitType type) {
+    LimitKeys.Keys keys = LimitKeys.monetary(command.userId(), type, command.stake().currency());
+    assertThat(redis.opsForValue().get(keys.sum())).isEqualTo("40");
+    assertThat(redis.opsForZSet().range(keys.entries(), 0, -1))
+        .containsExactly(LimitKeys.member(command.betId(), 40));
+  }
+}
