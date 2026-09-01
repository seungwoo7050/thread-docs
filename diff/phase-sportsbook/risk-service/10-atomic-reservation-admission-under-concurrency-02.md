## `feat(reservation): reserve rapid pattern capacity`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 3b94e26..4cdc168 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -156,7 +156,35 @@ if selectionCandidate > selectionLimit then
   return response({status = "REJECTED", rejection = "SELECTIONS_PER_MINUTE_LIMIT_EXCEEDED",
     replayed = false, patternsJson = "[]"})
 end
-persist("RESERVED", "[]")
+local function action(value)
+  if value == "SUSPECT" or value == "REVIEW" or value == "BLOCK" then return value end
+end
+local patterns, firstBlock = {}, nil
+local function addPattern(rule, configuredAction, reason)
+  table.insert(patterns, {rule = rule, action = configuredAction, reason = reason})
+  if configuredAction == "BLOCK" and not firstBlock then firstBlock = rule end
+end
+if ARGV[20] == "1" then
+  local rapidError = typeError(KEYS[16], "zset") or typeError(KEYS[2], "zset")
+  local rapidWindow, rapidMax, rapidAction = tonumber(ARGV[21]), tonumber(ARGV[22]), action(ARGV[23])
+  if rapidError or not rapidWindow or not rapidMax or not rapidAction then
+    return redis.error_reply(rapidError or "invalid rapid policy")
+  end
+  local cutoff = "(" .. (now - rapidWindow)
+  local rapidCount = redis.call("ZCOUNT", KEYS[16], cutoff, "+inf")
+    + redis.call("ZCOUNT", KEYS[2], cutoff, "+inf") + 1
+  if rapidCount >= rapidMax then
+    addPattern("RAPID_BETTING", rapidAction, "rapid betting threshold reached")
+  end
+end
+local patternsJson = #patterns == 0 and "[]" or cjson.encode(patterns)
+if firstBlock then
+  persist("REJECTED", patternsJson)
+  redis.call("HSET", KEYS[1], "rejection", firstBlock, "rejectedAt", string.format("%.0f", now))
+  return response({status = "REJECTED", rejection = firstBlock,
+    replayed = false, patternsJson = patternsJson})
+end
+persist("RESERVED", patternsJson)
 redis.call("HSET", KEYS[1], "reservedAt", string.format("%.0f", now),
   "expiresAt", string.format("%.0f", expiresAt))
 redis.call("ZADD", KEYS[2], now, betId); redis.call("ZADD", KEYS[3], now, betId .. "|" .. stakeText)
@@ -166,4 +194,4 @@ redis.call("SET", KEYS[6], string.format("%.0f", nextSelections))
 redis.call("SET", KEYS[18], string.format("%.0f", nextGauge))
 return response({status = "APPROVED", state = "RESERVED",
   expiresAt = string.format("%.0f", expiresAt), token = fingerprint,
-  replayed = false, patternsJson = "[]"})
+  replayed = false, patternsJson = patternsJson})


## `test(reservation): prevent parallel rapid bypass`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationRapidPatternScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationRapidPatternScriptTest.java
new file mode 100644
index 0000000..30c2d73
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationRapidPatternScriptTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import java.time.Duration;
+import java.util.List;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class ReservationRapidPatternScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void preventsParallelCandidatesFromBypassingTheRapidThreshold() throws Exception {
+    RiskPatternProperties patterns =
+        new RiskPatternProperties(
+            new RapidBettingPolicy(true, Duration.ofMinutes(1), 5, PatternAction.BLOCK),
+            SuddenStakePolicy.defaults(),
+            RepeatedSelectionPolicy.defaults());
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
+                                            200 + index, 10, Currency.KRW, selection(200 + index)),
+                                        limits(1_000),
+                                        patterns))
+                                .decision();
+                          }))
+              .toList();
+      start.countDown();
+
+      List<ReservationDecision> decisions =
+          futures.stream().map(ReservationRapidPatternScriptTest::result).toList();
+      assertThat(decisions.stream().filter(ReservationDecision::approved)).hasSize(4);
+      assertThat(decisions.stream().filter(decision -> !decision.approved()))
+          .allMatch(decision -> "RAPID_BETTING".equals(decision.rejection()));
+      assertThat(redis.opsForZSet().size(ReservationKeys.activeBets(USER))).isEqualTo(4);
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


## `feat(reservation): reserve repeated selection capacity`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 4cdc168..432a7b8 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -177,6 +177,33 @@ if ARGV[20] == "1" then
     addPattern("RAPID_BETTING", rapidAction, "rapid betting threshold reached")
   end
 end
+for index = 1, selectionCount do
+  local activeKey = KEYS[18 + selectionCount + index]
+  local activeError = typeError(activeKey, "zset")
+  if activeError or redis.call("ZSCORE", activeKey, betId) then
+    return redis.error_reply(activeError or "orphan per-selection footprint")
+  end
+end
+if ARGV[28] == "1" then
+  local repeatedWindow, repeatedMax, repeatedAction =
+    tonumber(ARGV[29]), tonumber(ARGV[30]), action(ARGV[31])
+  if not repeatedWindow or not repeatedMax or not repeatedAction then
+    return redis.error_reply("invalid repeated policy")
+  end
+  for index, selectionId in ipairs(selections) do
+    local confirmedKey, activeKey = KEYS[18 + index], KEYS[18 + selectionCount + index]
+    local repeatedError = typeError(confirmedKey, "zset") or typeError(activeKey, "zset")
+    if repeatedError then return redis.error_reply(repeatedError) end
+    local cutoff = "(" .. (now - repeatedWindow)
+    local repeatedCount = redis.call("ZCOUNT", confirmedKey, cutoff, "+inf")
+      + redis.call("ZCOUNT", activeKey, cutoff, "+inf") + 1
+    if repeatedCount > repeatedMax then
+      addPattern("REPEATED_SAME_SELECTION", repeatedAction,
+        "repeated selection threshold reached: SelectionId[value=" .. selectionId .. "]")
+      break
+    end
+  end
+end
 local patternsJson = #patterns == 0 and "[]" or cjson.encode(patterns)
 if firstBlock then
   persist("REJECTED", patternsJson)
@@ -191,6 +218,10 @@ redis.call("ZADD", KEYS[2], now, betId); redis.call("ZADD", KEYS[3], now, betId
 redis.call("SET", KEYS[4], string.format("%.0f", nextStake))
 redis.call("ZADD", KEYS[5], now, betId .. "|" .. countText)
 redis.call("SET", KEYS[6], string.format("%.0f", nextSelections))
+for index = 1, selectionCount do
+  local activeKey = KEYS[18 + selectionCount + index]
+  redis.call("ZADD", activeKey, now, betId)
+end
 redis.call("SET", KEYS[18], string.format("%.0f", nextGauge))
 return response({status = "APPROVED", state = "RESERVED",
   expiresAt = string.format("%.0f", expiresAt), token = fingerprint,


## `test(reservation): prevent repeated selection bypass`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationRepeatedPatternScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationRepeatedPatternScriptTest.java
new file mode 100644
index 0000000..5e744ad
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationRepeatedPatternScriptTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class ReservationRepeatedPatternScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void sharesRepeatedSelectionCapacityAcrossCurrencies() {
+    RiskPatternProperties patterns =
+        new RiskPatternProperties(
+            RapidBettingPolicy.defaults(),
+            SuddenStakePolicy.defaults(),
+            new RepeatedSelectionPolicy(true, Duration.ofDays(1), 2, PatternAction.BLOCK));
+
+    ReservationDecision first =
+        execute(request(command(300, 10, Currency.KRW), limits(1_000), patterns)).decision();
+    ReservationDecision second =
+        execute(request(command(301, 10, Currency.USD), limits(1_000), patterns)).decision();
+    ReservationDecision blocked =
+        execute(request(command(302, 10, Currency.KRW), limits(1_000), patterns)).decision();
+
+    assertThat(first.approved()).isTrue();
+    assertThat(second.approved()).isTrue();
+    assertThat(blocked.rejection()).isEqualTo("REPEATED_SAME_SELECTION");
+    assertThat(redis.opsForZSet().size(ReservationKeys.activeSelection(USER, SELECTION)))
+        .isEqualTo(2);
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(command(302, 10, Currency.KRW).betId())))
+        .isTrue();
+  }
+}


## `feat(reservation): evaluate currency stake patterns`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 432a7b8..4588232 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -204,6 +204,71 @@ if ARGV[28] == "1" then
     end
   end
 end
+local function addText(left, right)
+  local output, carry, leftIndex, rightIndex = {}, 0, #left, #right
+  while leftIndex > 0 or rightIndex > 0 or carry > 0 do
+    local l = leftIndex > 0 and tonumber(string.sub(left, leftIndex, leftIndex)) or 0
+    local r = rightIndex > 0 and tonumber(string.sub(right, rightIndex, rightIndex)) or 0
+    local sum = l + r + carry
+    table.insert(output, 1, tostring(sum % 10)); carry = math.floor(sum / 10)
+    leftIndex, rightIndex = leftIndex - 1, rightIndex - 1
+  end
+  return table.concat(output)
+end
+local function multiplyText(value, multiplier)
+  local output, carry = {}, 0
+  for index = #value, 1, -1 do
+    local product = tonumber(string.sub(value, index, index)) * multiplier + carry
+    table.insert(output, 1, tostring(product % 10)); carry = math.floor(product / 10)
+  end
+  while carry > 0 do table.insert(output, 1, tostring(carry % 10)); carry = math.floor(carry / 10) end
+  return table.concat(output)
+end
+local function greaterOrEqual(left, right)
+  left, right = string.gsub(left, "^0+", ""), string.gsub(right, "^0+", "")
+  if #left ~= #right then return #left > #right end
+  return left >= right
+end
+local function suddenMatch()
+  if ARGV[24] ~= "1" then return false end
+  local errorText = typeError(KEYS[17], "zset") or typeError(KEYS[3], "zset")
+  local multiplier, lookback = tonumber(ARGV[25]), tonumber(ARGV[26])
+  if errorText or not multiplier or multiplier <= 1 or not lookback or lookback < 1 then
+    return nil, errorText or "invalid sudden policy"
+  end
+  local samples = {}
+  for _, key in ipairs({KEYS[17], KEYS[3]}) do
+    local values = redis.call("ZRANGE", key, 0, -1, "WITHSCORES")
+    for index = 1, #values, 2 do
+      local text = string.match(values[index], "|([0-9]+)$")
+      local amount = exact(text, true)
+      if not amount then return nil, "corrupt sudden stake member" end
+      table.insert(samples, {amount = amount, text = text,
+        score = tonumber(values[index + 1]), member = values[index]})
+    end
+  end
+  table.sort(samples, function(left, right)
+    if left.score == right.score then return left.member < right.member end
+    return left.score < right.score
+  end)
+  if #samples < lookback then return false end
+  local recent = {}
+  for index = #samples - lookback + 1, #samples do table.insert(recent, samples[index]) end
+  table.sort(recent, function(left, right) return left.amount < right.amount end)
+  local middle = math.floor(#recent / 2) + 1
+  if #recent % 2 == 1 then
+    return greaterOrEqual(stakeText, multiplyText(recent[middle].text, multiplier))
+  end
+  local medianSum = addText(recent[middle - 1].text, recent[middle].text)
+  return greaterOrEqual(multiplyText(stakeText, 2), multiplyText(medianSum, multiplier))
+end
+local sudden, suddenError = suddenMatch()
+if suddenError then return redis.error_reply(suddenError) end
+if sudden then
+  local suddenAction = action(ARGV[27])
+  if not suddenAction then return redis.error_reply("invalid sudden action") end
+  addPattern("SUDDEN_STAKE_INCREASE", suddenAction, "sudden stake threshold reached")
+end
 local patternsJson = #patterns == 0 and "[]" or cjson.encode(patterns)
 if firstBlock then
   persist("REJECTED", patternsJson)


## `test(reservation): verify atomic sudden stake decisions`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationSuddenPatternScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationSuddenPatternScriptTest.java
new file mode 100644
index 0000000..b3b1ca1
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationSuddenPatternScriptTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import org.junit.jupiter.api.Test;
+
+class ReservationSuddenPatternScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void evaluatesSuddenStakeAgainstSameCurrencyActiveFacts() {
+    RiskPatternProperties patterns =
+        new RiskPatternProperties(
+            RapidBettingPolicy.defaults(),
+            new SuddenStakePolicy(true, 3, 2, PatternAction.BLOCK),
+            RepeatedSelectionPolicy.defaults());
+
+    assertThat(
+            execute(
+                    request(
+                        command(400, 10, Currency.KRW, selection(400)), limits(1_000), patterns))
+                .decision()
+                .approved())
+        .isTrue();
+    assertThat(
+            execute(
+                    request(
+                        command(401, 10, Currency.KRW, selection(401)), limits(1_000), patterns))
+                .decision()
+                .approved())
+        .isTrue();
+    assertThat(
+            execute(
+                    request(
+                        command(402, 30, Currency.KRW, selection(402)), limits(1_000), patterns))
+                .decision()
+                .rejection())
+        .isEqualTo("SUDDEN_STAKE_INCREASE");
+    assertThat(
+            execute(
+                    request(
+                        command(403, 30, Currency.USD, selection(403)), limits(1_000), patterns))
+                .decision()
+                .approved())
+        .isTrue();
+  }
+}


## `feat(reservation): persist ordered pattern verdicts`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 4588232..8abaa62 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -159,10 +159,9 @@ end
 local function action(value)
   if value == "SUSPECT" or value == "REVIEW" or value == "BLOCK" then return value end
 end
-local patterns, firstBlock = {}, nil
+local matches = {}
 local function addPattern(rule, configuredAction, reason)
-  table.insert(patterns, {rule = rule, action = configuredAction, reason = reason})
-  if configuredAction == "BLOCK" and not firstBlock then firstBlock = rule end
+  matches[rule] = {rule = rule, action = configuredAction, reason = reason}
 end
 if ARGV[20] == "1" then
   local rapidError = typeError(KEYS[16], "zset") or typeError(KEYS[2], "zset")
@@ -269,6 +268,14 @@ if sudden then
   if not suddenAction then return redis.error_reply("invalid sudden action") end
   addPattern("SUDDEN_STAKE_INCREASE", suddenAction, "sudden stake threshold reached")
 end
+local patterns, firstBlock = {}, nil
+for _, rule in ipairs({"RAPID_BETTING", "SUDDEN_STAKE_INCREASE", "REPEATED_SAME_SELECTION"}) do
+  local match = matches[rule]
+  if match then
+    table.insert(patterns, match)
+    if match.action == "BLOCK" and not firstBlock then firstBlock = rule end
+  end
+end
 local patternsJson = #patterns == 0 and "[]" or cjson.encode(patterns)
 if firstBlock then
   persist("REJECTED", patternsJson)


## `test(reservation): verify stable pattern verdicts`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationPatternVerdictScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationPatternVerdictScriptTest.java
new file mode 100644
index 0000000..720b3bf
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationPatternVerdictScriptTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class ReservationPatternVerdictScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void persistsStableRuleOrderAndFirstBlockingVerdict() {
+    reserve(command(500, 10, Currency.KRW));
+    reserve(command(501, 10, Currency.KRW));
+    RiskPatternProperties patterns =
+        new RiskPatternProperties(
+            new RapidBettingPolicy(true, Duration.ofMinutes(1), 3, PatternAction.SUSPECT),
+            new SuddenStakePolicy(true, 2, 2, PatternAction.BLOCK),
+            new RepeatedSelectionPolicy(true, Duration.ofDays(1), 2, PatternAction.REVIEW));
+    var request = request(command(502, 30, Currency.KRW), limits(1_000), patterns);
+
+    ReservationDecision first = execute(request).decision();
+    ReservationDecision replay = execute(request).decision();
+
+    assertThat(first.rejection()).isEqualTo("SUDDEN_STAKE_INCREASE");
+    assertThat(first.patterns())
+        .extracting(match -> match.rule())
+        .containsExactly("RAPID_BETTING", "SUDDEN_STAKE_INCREASE", "REPEATED_SAME_SELECTION");
+    assertThat(replay.replayed()).isTrue();
+    assertThat(replay.rejection()).isEqualTo(first.rejection());
+    assertThat(replay.patterns()).isEqualTo(first.patterns());
+  }
+}


## `test(load): verify concurrent reservation cardinality`

diff --git a/load-test/concurrent-admission.sh b/load-test/concurrent-admission.sh
new file mode 100644
index 0000000..4a66a37
--- /dev/null
+++ b/load-test/concurrent-admission.sh
@@ -0,0 +1,66 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+base_url=${RISK_BASE_URL:?}
+output_dir=${RISK_GATE_OUTPUT:?}
+betting_key=${INTERNAL_BETTING_SERVICE_API_KEY:?}
+admin_key=${INTERNAL_ADMIN_API_KEY:?}
+user_id=60000000-0000-4000-8000-000000000001
+selection_id=70000000-0000-4000-8000-000000000001
+
+auth=(-H "X-Internal-Service: betting-service" -H "X-Internal-Api-Key: ${betting_key}")
+admin=(-H "X-Internal-Service: admin-api" -H "X-Internal-Api-Key: ${admin_key}")
+json=(-H "Content-Type: application/json")
+
+body() {
+  local bet_id=$1 amount=$2
+  printf '{"userId":"%s","betId":"%s","stake":{"amount":%s,"currency":"KRW"},"selectionIds":["%s"]}' \
+    "${user_id}" "${bet_id}" "${amount}" "${selection_id}"
+}
+
+reserve() {
+  local bet_id=$1 amount=$2 target=$3
+  curl -sS "${auth[@]}" "${json[@]}" -X POST -d "$(body "${bet_id}" "${amount}")" \
+    "${base_url}/internal/v1/risk/reservations" -o "${target}.json" \
+    -w '%{http_code}' > "${target}.status"
+}
+
+same_bet=80000000-0000-4000-8000-000000000001
+pids=()
+for attempt in $(seq 1 100); do
+  reserve "${same_bet}" 10 "${output_dir}/same-${attempt}" &
+  pids+=("$!")
+done
+for pid in "${pids[@]}"; do wait "${pid}"; done
+
+test "$(sort -u "${output_dir}"/same-*.status)" = 200
+jq -s -e '
+  length == 100 and
+  all(.approved == true and .reservationState == "RESERVED") and
+  (map(select(.replayed == false)) | length) == 1 and
+  (map(select(.replayed == true)) | length) == 99 and
+  (map(.reservationToken) | unique | length) == 1
+' "${output_dir}"/same-*.json > /dev/null
+
+for type in STAKE_DAILY STAKE_WEEKLY STAKE_MONTHLY; do
+  curl -fsS "${admin[@]}" "${json[@]}" -X PATCH \
+    -d "{\"type\":\"${type}\",\"currency\":\"KRW\",\"value\":100}" \
+    "${base_url}/internal/v1/risk/limits/${user_id}" > /dev/null
+done
+
+pids=()
+for ordinal in 2 3; do
+  bet_id=80000000-0000-4000-8000-$(printf '%012d' "${ordinal}")
+  reserve "${bet_id}" 60 "${output_dir}/capacity-${ordinal}" &
+  pids+=("$!")
+done
+for pid in "${pids[@]}"; do wait "${pid}"; done
+
+test "$(sort -u "${output_dir}"/capacity-*.status)" = 200
+jq -s -e '
+  length == 2 and
+  (map(select(.approved == true and .replayed == false and
+    .reservationState == "RESERVED" and (.reservationToken | type == "string"))) | length) == 1 and
+  (map(select(.approved == false and .replayed == false and
+    .rejectionReason == "STAKE_DAILY_LIMIT_EXCEEDED" and (has("reservationToken") | not))) | length) == 1
+' "${output_dir}"/capacity-*.json > /dev/null
