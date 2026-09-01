## `test(reservation): verify committed footprint removal`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitFootprintScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitFootprintScriptTest.java
new file mode 100644
index 0000000..ab13c1b
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitFootprintScriptTest.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitFootprintScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void removesTheCompleteActiveFootprintOnCommit() {
+    var first = selection(1_100);
+    var second = selection(1_101);
+    var command = command(1_100, 40, Currency.KRW, first, second);
+    ReservationDecision reserved = reserve(command);
+
+    assertThat(commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.APPLIED);
+
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeStakes(USER, Currency.KRW).entries())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeStakes(USER, Currency.KRW).sum())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeSelections(USER).entries())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeSelections(USER).sum())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeSelection(USER, first))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeSelection(USER, second))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.ACTIVE_COUNT)).isFalse();
+  }
+}


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


## `feat(reservation): commit pattern history`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
index 6ff22fd..ab8b62b 100644
--- a/src/main/resources/scripts/risk-commit.lua
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -127,6 +127,15 @@ for _, dimension in ipairs(dimensions) do
   table.insert(plans, plan)
 end
 
+local historyBase = "risk:history:{" .. userId .. "}"
+local historyBets, historyStakes = historyBase .. ":bets", historyBase .. ":stakes:" .. currencyLower
+local historyError = typeError(historyBets, "zset") or typeError(historyStakes, "zset")
+if historyError then return redis.error_reply(historyError) end
+for _, selectionId in ipairs(selections) do
+  local itemError = typeError(historyBase .. ":selection:" .. selectionId, "zset")
+  if itemError then return redis.error_reply(itemError) end
+end
+
 removeActive()
 for _, plan in ipairs(plans) do
   redis.call("ZREMRANGEBYSCORE", plan[1], "-inf", now - plan[3])
@@ -134,6 +143,18 @@ for _, plan in ipairs(plans) do
   redis.call("SET", plan[2], string.format("%.0f", plan[5]), "PX", plan[3] + 300000)
   redis.call("PEXPIRE", plan[1], plan[3] + 300000)
 end
+local historyTtl, stakeLimit = tonumber(ARGV[11]), tonumber(ARGV[12])
+redis.call("ZREMRANGEBYSCORE", historyBets, "-inf", now - tonumber(ARGV[9]))
+redis.call("ZADD", historyBets, "NX", now, betId)
+redis.call("ZADD", historyStakes, "NX", now, betId .. "|" .. stakeText)
+local stakeCard = redis.call("ZCARD", historyStakes)
+if stakeCard > stakeLimit then redis.call("ZREMRANGEBYRANK", historyStakes, 0, stakeCard - stakeLimit - 1) end
+for _, selectionId in ipairs(selections) do
+  local key = historyBase .. ":selection:" .. selectionId
+  redis.call("ZREMRANGEBYSCORE", key, "-inf", now - tonumber(ARGV[10]))
+  redis.call("ZADD", key, "NX", now, betId); redis.call("PEXPIRE", key, historyTtl)
+end
+redis.call("PEXPIRE", historyBets, historyTtl); redis.call("PEXPIRE", historyStakes, historyTtl)
 redis.call("HSET", KEYS[1], "state", "COMMITTED", "committedAt", string.format("%.0f", now))
 redis.call("PEXPIRE", KEYS[1], retention)
 return "APPLIED"


## `test(reservation): verify committed pattern history`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationCommitHistoryScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitHistoryScriptTest.java
new file mode 100644
index 0000000..c97389c
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationCommitHistoryScriptTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.pattern.HistoryKeys;
+import org.junit.jupiter.api.Test;
+
+class ReservationCommitHistoryScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void projectsCommittedFactsIntoBoundedPatternHistory() {
+    var first = selection(1_300);
+    var second = selection(1_301);
+    var command = command(1_300, 40, Currency.USD, first, second);
+    ReservationDecision reserved = reserve(command);
+
+    assertThat(commit(command.betId(), reserved.token(), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.APPLIED);
+
+    assertThat(redis.opsForZSet().range(HistoryKeys.bets(USER), 0, -1))
+        .containsExactly(HistoryKeys.betMember(command.betId()));
+    assertThat(redis.opsForZSet().range(HistoryKeys.stakes(USER, Currency.USD), 0, -1))
+        .containsExactly(HistoryKeys.stakeMember(command.betId(), 40));
+    assertThat(redis.opsForZSet().range(HistoryKeys.selection(USER, first), 0, -1))
+        .containsExactly(HistoryKeys.betMember(command.betId()));
+    assertThat(redis.opsForZSet().range(HistoryKeys.selection(USER, second), 0, -1))
+        .containsExactly(HistoryKeys.betMember(command.betId()));
+  }
+}


## `feat(reservation): release active reservation capacity`

diff --git a/src/main/resources/scripts/risk-release.lua b/src/main/resources/scripts/risk-release.lua
new file mode 100644
index 0000000..eb9392d
--- /dev/null
+++ b/src/main/resources/scripts/risk-release.lua
@@ -0,0 +1,84 @@
+local maxExact, now, retention = 9007199254740991, tonumber(ARGV[2]), tonumber(ARGV[3])
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
+
+if ARGV[1] ~= "1" or #KEYS ~= 2 or #ARGV ~= 3
+  or not exact(ARGV[2], false) or not exact(ARGV[3], true) then
+  return redis.error_reply("invalid release request")
+end
+local lifecycleType = keyType(KEYS[1])
+if lifecycleType == "none" then return "NOT_FOUND" end
+if lifecycleType ~= "hash" then return redis.error_reply("wrong reservation lifecycle type") end
+local state = redis.call("HGET", KEYS[1], "state")
+if state == "RELEASED" then return "REPLAYED" end
+if state == "COMMITTED" then return "CONFLICT" end
+if state == "EXPIRED" or state == "REJECTED" then return "TOMBSTONED" end
+if state ~= "RESERVED" then return redis.error_reply("unknown reservation state") end
+
+local userId, betId = redis.call("HGET", KEYS[1], "userId"), redis.call("HGET", KEYS[1], "betId")
+local stakeText, currency = redis.call("HGET", KEYS[1], "stake"), redis.call("HGET", KEYS[1], "currency")
+local countText = redis.call("HGET", KEYS[1], "selectionCount")
+local stake, count = exact(stakeText, true), exact(countText, true)
+local selections = split(redis.call("HGET", KEYS[1], "selections"), count or -1)
+if not userId or not betId or not stake or not currency or not count or not selections
+  or not string.match(userId, "^[0-9a-f%-]+$") or not string.match(betId, "^[0-9a-f%-]+$")
+  or not string.match(currency, "^[A-Z]+$") then
+  return redis.error_reply("corrupt reservation identity")
+end
+
+local base = "risk:reservations:user:{" .. userId .. "}"
+local bets, stakeEntries = base .. ":bets", base .. ":stakes:" .. string.lower(currency) .. ":entries"
+local stakeSum, selectionEntries = base .. ":stakes:" .. string.lower(currency) .. ":sum", base .. ":selections:entries"
+local selectionSum = base .. ":selections:sum"
+local errorText = typeError(bets, "zset") or typeError(stakeEntries, "zset")
+  or typeError(stakeSum, "string") or typeError(selectionEntries, "zset")
+  or typeError(selectionSum, "string") or typeError(KEYS[2], "string")
+if errorText or not redis.call("ZSCORE", bets, betId)
+  or not redis.call("ZSCORE", stakeEntries, betId .. "|" .. stakeText)
+  or not redis.call("ZSCORE", selectionEntries, betId .. "|" .. countText) then
+  return redis.error_reply(errorText or "missing active reservation footprint")
+end
+for _, selectionId in ipairs(selections) do
+  local key = base .. ":selection:" .. selectionId
+  local selectionError = typeError(key, "zset")
+  if selectionError or not redis.call("ZSCORE", key, betId) then
+    return redis.error_reply(selectionError or "missing per-selection footprint")
+  end
+end
+local stakeTotal, selectionTotal, gauge = exact(redis.call("GET", stakeSum), false),
+  exact(redis.call("GET", selectionSum), false), exact(redis.call("GET", KEYS[2]), false)
+if not stakeTotal or stakeTotal < stake or not selectionTotal or selectionTotal < count
+  or not gauge or gauge < 1 then return redis.error_reply("corrupt active total") end
+
+redis.call("ZREM", bets, betId); redis.call("ZREM", stakeEntries, betId .. "|" .. stakeText)
+redis.call("ZREM", selectionEntries, betId .. "|" .. countText)
+for _, selectionId in ipairs(selections) do
+  local key = base .. ":selection:" .. selectionId
+  redis.call("ZREM", key, betId); if redis.call("ZCARD", key) == 0 then redis.call("DEL", key) end
+end
+decrement(stakeSum, stake); decrement(selectionSum, count); decrement(KEYS[2], 1)
+if redis.call("ZCARD", bets) == 0 then redis.call("DEL", bets) end
+if redis.call("ZCARD", stakeEntries) == 0 then redis.call("DEL", stakeEntries) end
+if redis.call("ZCARD", selectionEntries) == 0 then redis.call("DEL", selectionEntries) end
+redis.call("HSET", KEYS[1], "state", "RELEASED", "releasedAt", string.format("%.0f", now))
+redis.call("PEXPIRE", KEYS[1], retention)
+return "APPLIED"


## `test(reservation): verify idempotent release lifecycle`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseLifecycleScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseLifecycleScriptTest.java
new file mode 100644
index 0000000..691ac86
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationReleaseLifecycleScriptTest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import org.junit.jupiter.api.Test;
+
+class ReservationReleaseLifecycleScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void releasesCapacityAndReplaysTheTerminalResult() {
+    var command = command(1_600, 20, Currency.KRW, selection(1_600), selection(1_601));
+    reserve(command);
+
+    assertThat(release(command.betId(), NOW.plusMillis(1)))
+        .isEqualTo(ReservationTransition.APPLIED);
+    assertThat(release(command.betId(), NOW.plusMillis(2)))
+        .isEqualTo(ReservationTransition.REPLAYED);
+    assertThat(
+            redis
+                .<String, String>opsForHash()
+                .get(ReservationKeys.lifecycle(command.betId()), "state"))
+        .isEqualTo("RELEASED");
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeStakes(USER, Currency.KRW).sum())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeSelections(USER).sum())).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.ACTIVE_COUNT)).isFalse();
+  }
+
+  @Test
+  void preventsCommittedReservationsFromBeingReleased() {
+    var command = command(1_601, 10, Currency.KRW);
+    ReservationDecision reserved = reserve(command);
+    commit(command.betId(), reserved.token(), NOW.plusMillis(1));
+
+    assertThat(release(command.betId(), NOW.plusMillis(2)))
+        .isEqualTo(ReservationTransition.CONFLICT);
+  }
+}


## `test(load): verify reservation lifecycle replay`

diff --git a/load-test/lifecycle-replay.sh b/load-test/lifecycle-replay.sh
new file mode 100644
index 0000000..aabe3fa
--- /dev/null
+++ b/load-test/lifecycle-replay.sh
@@ -0,0 +1,51 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+base_url=${RISK_BASE_URL:?}
+output_dir=${RISK_GATE_OUTPUT:?}
+betting_key=${INTERNAL_BETTING_SERVICE_API_KEY:?}
+auth=(-H "X-Internal-Service: betting-service" -H "X-Internal-Api-Key: ${betting_key}")
+json=(-H "Content-Type: application/json")
+user_id=61000000-0000-4000-8000-000000000001
+selection_id=71000000-0000-4000-8000-000000000001
+
+reserve() {
+  local bet_id=$1 target=$2
+  curl -fsS "${auth[@]}" "${json[@]}" -X POST \
+    -d "{\"userId\":\"${user_id}\",\"betId\":\"${bet_id}\",\"stake\":{\"amount\":10,\"currency\":\"KRW\"},\"selectionIds\":[\"${selection_id}\"]}" \
+    "${base_url}/internal/v1/risk/reservations" -o "${target}"
+  jq -e '.approved == true and .replayed == false and
+    .reservationState == "RESERVED" and (.reservationToken | type == "string")' \
+    "${target}" > /dev/null
+}
+
+committed=81000000-0000-4000-8000-000000000001
+reserve "${committed}" "${output_dir}/committed.json"
+token=$(jq -er '.reservationToken' "${output_dir}/committed.json")
+for attempt in 1 2; do
+  status=$(curl -sS "${auth[@]}" -H "X-Risk-Reservation-Token: ${token}" \
+    -X PUT -o /dev/null -w '%{http_code}' \
+    "${base_url}/internal/v1/risk/reservations/${committed}/commit")
+  test "${status}" = 204
+done
+status=$(curl -sS "${auth[@]}" -X DELETE \
+  -o "${output_dir}/committed-release.json" -w '%{http_code}' \
+  "${base_url}/internal/v1/risk/reservations/${committed}")
+test "${status}" = 409
+jq -e '.errorCode == "RISK_RESERVATION_COMMITTED"' \
+  "${output_dir}/committed-release.json" > /dev/null
+
+released=81000000-0000-4000-8000-000000000002
+reserve "${released}" "${output_dir}/released.json"
+released_token=$(jq -er '.reservationToken' "${output_dir}/released.json")
+for attempt in 1 2; do
+  status=$(curl -sS "${auth[@]}" -X DELETE -o /dev/null -w '%{http_code}' \
+    "${base_url}/internal/v1/risk/reservations/${released}")
+  test "${status}" = 204
+done
+status=$(curl -sS "${auth[@]}" -H "X-Risk-Reservation-Token: ${released_token}" \
+  -X PUT -o "${output_dir}/released-commit.json" -w '%{http_code}' \
+  "${base_url}/internal/v1/risk/reservations/${released}/commit")
+test "${status}" = 404
+jq -e '.errorCode == "RISK_RESERVATION_NOT_FOUND"' \
+  "${output_dir}/released-commit.json" > /dev/null
