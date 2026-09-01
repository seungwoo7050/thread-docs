# 동시 요청을 직렬화하는 원자적 예약 승인

## `feat(reservation): define atomic reservation entrypoint`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
new file mode 100644
index 0000000..f3ca683
--- /dev/null
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -0,0 +1,84 @@
+local maxExact = 9007199254740991
+local now, lease, retention = tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
+local fingerprint, userId, betId = ARGV[5], ARGV[6], ARGV[7]
+local stakeText, currency, countText = ARGV[8], ARGV[9], ARGV[10]
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
+  if actual ~= "none" and actual ~= expected then return "wrong Redis type for " .. key end
+end
+local function checkedAdd(left, right)
+  if left > maxExact - right then return nil end
+  return left + right
+end
+local function response(payload)
+  payload.version, payload.expired = "1", "0"
+  return cjson.encode(payload)
+end
+
+local stake, selectionCount = exact(stakeText, true), exact(countText, true)
+if ARGV[1] ~= "1" or not exact(ARGV[2], false) or not exact(ARGV[3], true)
+  or not exact(ARGV[4], true) or retention <= lease or not stake or not selectionCount
+  or not fingerprint or not string.match(fingerprint, "^[0-9a-f]+$") or #fingerprint ~= 64
+  or #KEYS ~= 18 + selectionCount * 2 or #ARGV ~= 33 + selectionCount then
+  return redis.error_reply("invalid reservation request")
+end
+local selections, seen = {}, {}
+for index = 1, selectionCount do
+  local selectionId = ARGV[33 + index]
+  if not selectionId or seen[selectionId] then return redis.error_reply("invalid selection") end
+  seen[selectionId] = true; table.insert(selections, selectionId)
+end
+local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
+  or typeError(KEYS[3], "zset") or typeError(KEYS[4], "string")
+  or typeError(KEYS[5], "zset") or typeError(KEYS[6], "string")
+  or typeError(KEYS[7], "hash") or typeError(KEYS[18], "string")
+if errorText then return redis.error_reply(errorText) end
+if redis.call("EXISTS", KEYS[1]) == 1 then return redis.error_reply("reservation already exists") end
+
+local singleRaw = redis.call("HGET", KEYS[7], "SINGLE_BET_MAX:" .. currency) or ARGV[11]
+local singleLimit = exact(singleRaw, false)
+if not singleLimit then return redis.error_reply("corrupt single-bet limit") end
+local function persist(state, patternsJson)
+  redis.call("HSET", KEYS[1], "state", state, "fingerprint", fingerprint, "token", fingerprint,
+    "userId", userId, "betId", betId, "stake", stakeText, "currency", currency,
+    "selectionCount", countText, "selections", table.concat(selections, ","),
+    "patternsJson", patternsJson)
+  redis.call("PEXPIRE", KEYS[1], retention)
+end
+if stake > singleLimit then
+  persist("REJECTED", "[]")
+  redis.call("HSET", KEYS[1], "rejection", "SINGLE_BET_MAX_EXCEEDED",
+    "rejectedAt", string.format("%.0f", now))
+  return response({status = "REJECTED", rejection = "SINGLE_BET_MAX_EXCEEDED",
+    replayed = false, patternsJson = "[]"})
+end
+
+local activeStake = exact(redis.call("GET", KEYS[4]) or "0", false)
+local activeSelections = exact(redis.call("GET", KEYS[6]) or "0", false)
+local gauge = exact(redis.call("GET", KEYS[18]) or "0", false)
+local nextStake = activeStake and checkedAdd(activeStake, stake) or nil
+local nextSelections = activeSelections and checkedAdd(activeSelections, selectionCount) or nil
+local nextGauge = gauge and checkedAdd(gauge, 1) or nil
+local expiresAt = checkedAdd(now, lease)
+if not nextStake or not nextSelections or not nextGauge or not expiresAt then
+  return redis.error_reply("active reservation total exceeds exact range")
+end
+persist("RESERVED", "[]")
+redis.call("HSET", KEYS[1], "reservedAt", string.format("%.0f", now),
+  "expiresAt", string.format("%.0f", expiresAt))
+redis.call("ZADD", KEYS[2], now, betId); redis.call("ZADD", KEYS[3], now, betId .. "|" .. stakeText)
+redis.call("SET", KEYS[4], string.format("%.0f", nextStake))
+redis.call("ZADD", KEYS[5], now, betId .. "|" .. countText)
+redis.call("SET", KEYS[6], string.format("%.0f", nextSelections))
+redis.call("SET", KEYS[18], string.format("%.0f", nextGauge))
+return response({status = "APPROVED", state = "RESERVED",
+  expiresAt = string.format("%.0f", expiresAt), token = fingerprint,
+  replayed = false, patternsJson = "[]"})


## `test(reservation): provide reserve script fixtures`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationScriptTestSupport.java b/src/test/java/com/sportsbook/risk/reservation/ReservationScriptTestSupport.java
new file mode 100644
index 0000000..6705d0c
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationScriptTestSupport.java
@@ -0,0 +1,72 @@
+package com.sportsbook.risk.reservation;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.springframework.data.redis.core.script.RedisScript;
+
+abstract class ReservationScriptTestSupport extends RedisTestSupport {
+  protected static final UserId USER = UserId.of(new UUID(0, 1));
+  protected static final SelectionId SELECTION = SelectionId.of(new UUID(0, 2));
+  protected static final Instant NOW = Instant.ofEpochMilli(2_000_000);
+  private static final RedisScript<String> SCRIPT =
+      RedisLuaScriptLoader.stringScript("risk-reserve.lua");
+
+  protected ReservationWireMapper.Decoded execute(ReservationScriptRequest request) {
+    String raw = redis.execute(SCRIPT, request.keys(), request.arguments().toArray());
+    return new ReservationWireMapper(new ObjectMapper()).map(raw);
+  }
+
+  protected ReservationDecision reserve(RiskCheckCommand command) {
+    return execute(request(command)).decision();
+  }
+
+  protected ReservationScriptRequest request(RiskCheckCommand command) {
+    return request(
+        command,
+        new RiskLimitProperties(null, null, null, null, 0),
+        new RiskPatternProperties(null, null, null));
+  }
+
+  protected ReservationScriptRequest request(
+      RiskCheckCommand command, RiskLimitProperties limits, RiskPatternProperties patterns) {
+    return ReservationScriptRequest.from(
+        command,
+        limits,
+        patterns,
+        new RiskReservationProperties(null, null),
+        new RiskHistoryProperties(null, 0));
+  }
+
+  protected static RiskLimitProperties limits(long amount) {
+    Map<Currency, Long> values = Map.of(Currency.KRW, amount, Currency.USD, amount);
+    return new RiskLimitProperties(values, values, values, values, 30);
+  }
+
+  protected static RiskCheckCommand command(long bet, long amount, Currency currency) {
+    return command(bet, amount, currency, SELECTION);
+  }
+
+  protected static RiskCheckCommand command(
+      long bet, long amount, Currency currency, SelectionId... selections) {
+    return new RiskCheckCommand(
+        USER, BetId.of(new UUID(0, bet)), new Money(amount, currency), List.of(selections), NOW);
+  }
+
+  protected static SelectionId selection(long value) {
+    return SelectionId.of(new UUID(0, value));
+  }
+}


## `test(reservation): verify initial reservation decisions`

diff --git a/src/test/java/com/sportsbook/risk/reservation/RiskReserveScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/RiskReserveScriptTest.java
new file mode 100644
index 0000000..48e6ee9
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/RiskReserveScriptTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.ArrayList;
+import org.junit.jupiter.api.Test;
+
+class RiskReserveScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void approvesAValidCandidateAndPersistsItsToken() {
+    RiskCheckCommand command = command(10, 100, Currency.KRW);
+
+    ReservationDecision decision = reserve(command);
+
+    assertThat(decision.approved()).isTrue();
+    assertThat(decision.state()).isEqualTo(ReservationState.RESERVED);
+    assertThat(decision.token()).isEqualTo(ReservationFingerprint.of(command));
+    assertThat(redis.opsForHash().get(ReservationKeys.lifecycle(command.betId()), "fingerprint"))
+        .isEqualTo(decision.token());
+    assertThat(redis.opsForValue().get(ReservationKeys.activeStakes(USER, Currency.KRW).sum()))
+        .isEqualTo("100");
+    assertThat(redis.opsForValue().get(ReservationKeys.ACTIVE_COUNT)).isEqualTo("1");
+  }
+
+  @Test
+  void rejectsSingleBetLimitWithoutActiveCapacity() {
+    RiskCheckCommand command = command(11, 60, Currency.KRW);
+
+    ReservationDecision decision =
+        execute(request(command, limits(50), new RiskPatternProperties(null, null, null)))
+            .decision();
+
+    assertThat(decision.approved()).isFalse();
+    assertThat(decision.rejection()).isEqualTo("SINGLE_BET_MAX_EXCEEDED");
+    assertThat(redis.opsForHash().get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("REJECTED");
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+    assertThat(redis.opsForValue().get(ReservationKeys.ACTIVE_COUNT)).isNull();
+  }
+
+  @Test
+  void rejectsMalformedScriptArgumentsBeforeMutation() {
+    RiskCheckCommand command = command(12, 10, Currency.KRW);
+    ReservationScriptRequest valid = request(command);
+    ArrayList<String> arguments = new ArrayList<>(valid.arguments());
+    arguments.set(0, "2");
+
+    assertThatThrownBy(() -> execute(new ReservationScriptRequest(valid.keys(), arguments)))
+        .hasStackTraceContaining("invalid reservation request");
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(command.betId()))).isFalse();
+  }
+}


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


## `test(reservation): verify neutral selection capacity`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationSelectionCapacityScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationSelectionCapacityScriptTest.java
new file mode 100644
index 0000000..938c19d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationSelectionCapacityScriptTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.limit.LimitOverrideField;
+import com.sportsbook.risk.limit.LimitOverrideKeys;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+
+class ReservationSelectionCapacityScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void sharesSelectionCapacityAcrossCurrenciesAndHonorsNeutralOverrides() {
+    Map<Currency, Long> monetary = Map.of(Currency.KRW, 1_000L, Currency.USD, 1_000L);
+    RiskLimitProperties limits = new RiskLimitProperties(monetary, monetary, monetary, monetary, 2);
+    RiskPatternProperties patterns = new RiskPatternProperties(null, null, null);
+
+    assertThat(
+            execute(
+                    request(
+                        command(40, 10, Currency.KRW, selection(40), selection(41)),
+                        limits,
+                        patterns))
+                .decision()
+                .approved())
+        .isTrue();
+    assertThat(
+            execute(request(command(41, 10, Currency.USD), limits, patterns))
+                .decision()
+                .rejection())
+        .isEqualTo("SELECTIONS_PER_MINUTE_LIMIT_EXCEEDED");
+
+    redis
+        .opsForHash()
+        .put(LimitOverrideKeys.user(USER), LimitOverrideField.selections().redisField(), "3");
+    assertThat(
+            execute(request(command(42, 10, Currency.USD), limits, patterns)).decision().approved())
+        .isTrue();
+    assertThat(redis.opsForValue().get(ReservationKeys.activeSelections(USER).sum()))
+        .isEqualTo("3");
+  }
+}


