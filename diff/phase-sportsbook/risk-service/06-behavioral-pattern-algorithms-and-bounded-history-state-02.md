## `test(pattern): verify sudden stake boundaries`

diff --git a/src/test/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRuleTest.java b/src/test/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRuleTest.java
new file mode 100644
index 0000000..9820854
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRuleTest.java
@@ -0,0 +1,75 @@
+package com.sportsbook.risk.pattern.rule;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import com.sportsbook.risk.snapshot.SnapshotSlot;
+import java.time.Instant;
+import java.util.Arrays;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SuddenStakeIncreaseRuleTest {
+  private static final SelectionId SELECTION = SelectionId.of(new UUID(0, 3));
+
+  @Test
+  void requiresACompleteLookbackAndMatchesTheExactOddMedianThreshold() {
+    SuddenStakeIncreaseRule rule = rule(3, 3);
+
+    assertThat(rule.evaluate(context(100), snapshot(10, 20))).isEmpty();
+    assertThat(rule.evaluate(context(59), snapshot(10, 20, 30))).isEmpty();
+    assertThat(rule.evaluate(context(60), snapshot(10, 20, 30)))
+        .hasValueSatisfying(
+            match -> {
+              assertThat(match.rule()).isEqualTo(SuddenStakeIncreaseRule.NAME);
+              assertThat(match.action()).isEqualTo(PatternAction.REVIEW);
+            });
+  }
+
+  @Test
+  void usesAnExactEvenMedianWithoutOverflow() {
+    assertThat(rule(3, 4).evaluate(context(89), snapshot(10, 20, 40, 100))).isEmpty();
+    assertThat(rule(3, 4).evaluate(context(90), snapshot(10, 20, 40, 100))).isPresent();
+    assertThat(rule(2, 3).evaluate(context(1), snapshot(0, 0, 0))).isEmpty();
+    assertThat(
+            rule(2, 3)
+                .evaluate(
+                    context(SafeRedisNumber.MAX_VALUE),
+                    snapshot(
+                        SafeRedisNumber.MAX_VALUE,
+                        SafeRedisNumber.MAX_VALUE,
+                        SafeRedisNumber.MAX_VALUE)))
+        .isEmpty();
+  }
+
+  private static SuddenStakeIncreaseRule rule(int multiplier, int lookback) {
+    return new SuddenStakeIncreaseRule(
+        new SuddenStakePolicy(true, multiplier, lookback, PatternAction.REVIEW));
+  }
+
+  private static PatternContext context(long amount) {
+    return new PatternContext(
+        UserId.of(new UUID(0, 1)),
+        BetId.of(new UUID(0, 2)),
+        Money.krw(amount),
+        List.of(SELECTION),
+        Instant.EPOCH);
+  }
+
+  private static PatternSnapshot snapshot(long... stakes) {
+    return new PatternSnapshot(
+        SnapshotSlot.success(0L),
+        SnapshotSlot.success(Arrays.stream(stakes).boxed().toList()),
+        Map.of(SELECTION, SnapshotSlot.success(0L)));
+  }
+}


## `feat(pattern): detect repeated selections`

diff --git a/src/main/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRule.java b/src/main/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRule.java
new file mode 100644
index 0000000..7b460dd
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRule.java
@@ -0,0 +1,53 @@
+package com.sportsbook.risk.pattern.rule;
+
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.pattern.PatternRule;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import java.util.Objects;
+import java.util.Optional;
+
+/** Detects the first candidate selection that exceeds its global repeat cap. */
+public final class RepeatedSameSelectionRule implements PatternRule {
+  public static final String NAME = "REPEATED_SAME_SELECTION";
+  private static final int PRIORITY = 30;
+
+  private final RepeatedSelectionPolicy policy;
+
+  public RepeatedSameSelectionRule(RepeatedSelectionPolicy policy) {
+    this.policy = Objects.requireNonNull(policy, "policy");
+  }
+
+  @Override
+  public String name() {
+    return NAME;
+  }
+
+  @Override
+  public int priority() {
+    return PRIORITY;
+  }
+
+  @Override
+  public Optional<PatternMatch> evaluate(PatternContext context, PatternSnapshot snapshot) {
+    Objects.requireNonNull(context, "context");
+    Objects.requireNonNull(snapshot, "snapshot");
+    if (!policy.enabled()) {
+      return Optional.empty();
+    }
+    for (SelectionId selectionId : context.selections()) {
+      long candidateCount =
+          SafeRedisNumber.add(
+              snapshot.selectionCount(selectionId).valueOrThrow(), 1, "selection count");
+      if (candidateCount > policy.maxCount()) {
+        return Optional.of(
+            new PatternMatch(
+                NAME, policy.action(), "repeated selection threshold reached: " + selectionId));
+      }
+    }
+    return Optional.empty();
+  }
+}


## `test(pattern): verify repeated selection boundaries`

diff --git a/src/test/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRuleTest.java b/src/test/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRuleTest.java
new file mode 100644
index 0000000..d2e6867
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/pattern/rule/RepeatedSameSelectionRuleTest.java
@@ -0,0 +1,67 @@
+package com.sportsbook.risk.pattern.rule;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import com.sportsbook.risk.snapshot.SnapshotSlot;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RepeatedSameSelectionRuleTest {
+  private static final SelectionId FIRST = SelectionId.of(new UUID(0, 3));
+  private static final SelectionId SECOND = SelectionId.of(new UUID(0, 4));
+  private static final PatternContext CONTEXT =
+      new PatternContext(
+          UserId.of(new UUID(0, 1)),
+          BetId.of(new UUID(0, 2)),
+          Money.krw(10),
+          List.of(FIRST, SECOND),
+          Instant.EPOCH);
+
+  @Test
+  void matchesOnlyAfterTheConfiguredCapIsExceeded() {
+    RepeatedSameSelectionRule rule =
+        new RepeatedSameSelectionRule(
+            new RepeatedSelectionPolicy(true, Duration.ofDays(1), 2, PatternAction.REVIEW));
+
+    assertThat(rule.evaluate(CONTEXT, snapshot(1, 0))).isEmpty();
+    assertThat(rule.evaluate(CONTEXT, snapshot(2, 0)))
+        .hasValueSatisfying(
+            match -> {
+              assertThat(match.rule()).isEqualTo(RepeatedSameSelectionRule.NAME);
+              assertThat(match.reason()).contains(FIRST.toString());
+            });
+  }
+
+  @Test
+  void reportsTheFirstHotInputSelectionDeterministically() {
+    RepeatedSameSelectionRule rule =
+        new RepeatedSameSelectionRule(
+            new RepeatedSelectionPolicy(true, Duration.ofDays(1), 1, PatternAction.BLOCK));
+
+    assertThat(rule.evaluate(CONTEXT, snapshot(1, 1)))
+        .hasValueSatisfying(match -> assertThat(match.reason()).contains(FIRST.toString()));
+    assertThat(
+            new RepeatedSameSelectionRule(RepeatedSelectionPolicy.defaults())
+                .evaluate(CONTEXT, snapshot(100, 100)))
+        .isEmpty();
+  }
+
+  private static PatternSnapshot snapshot(long first, long second) {
+    return new PatternSnapshot(
+        SnapshotSlot.success(0L),
+        SnapshotSlot.success(List.of()),
+        Map.of(FIRST, SnapshotSlot.success(first), SECOND, SnapshotSlot.success(second)));
+  }
+}


## `feat(history): configure bounded retention`

diff --git a/src/main/java/com/sportsbook/risk/pattern/RiskHistoryProperties.java b/src/main/java/com/sportsbook/risk/pattern/RiskHistoryProperties.java
new file mode 100644
index 0000000..8334754
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/RiskHistoryProperties.java
@@ -0,0 +1,22 @@
+package com.sportsbook.risk.pattern;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Bounds retained confirmed pattern facts for active and idle users. */
+@ConfigurationProperties(prefix = "risk.history")
+public record RiskHistoryProperties(Duration idleRetention, int maxStakeSamples) {
+  private static final Duration DEFAULT_IDLE_RETENTION = Duration.ofDays(7);
+  private static final int DEFAULT_MAX_STAKE_SAMPLES = 100;
+
+  public RiskHistoryProperties {
+    idleRetention = idleRetention == null ? DEFAULT_IDLE_RETENTION : idleRetention;
+    maxStakeSamples = maxStakeSamples == 0 ? DEFAULT_MAX_STAKE_SAMPLES : maxStakeSamples;
+    if (idleRetention.isZero() || idleRetention.isNegative()) {
+      throw new IllegalArgumentException("history idle retention must be positive");
+    }
+    if (maxStakeSamples <= 0) {
+      throw new IllegalArgumentException("history stake sample bound must be positive");
+    }
+  }
+}


## `feat(history): define currency-aware history keys`

diff --git a/src/main/java/com/sportsbook/risk/pattern/HistoryKeys.java b/src/main/java/com/sportsbook/risk/pattern/HistoryKeys.java
new file mode 100644
index 0000000..7cc56d3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/HistoryKeys.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.pattern;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.Objects;
+
+/** Redis keys for confirmed pattern history owned by one user hash slot. */
+public final class HistoryKeys {
+  private static final String PREFIX = "risk:history:";
+
+  private HistoryKeys() {}
+
+  public static String bets(UserId userId) {
+    return base(userId) + ":bets";
+  }
+
+  public static String stakes(UserId userId, Currency currency) {
+    return base(userId)
+        + ":stakes:"
+        + Objects.requireNonNull(currency, "currency").name().toLowerCase();
+  }
+
+  public static String selection(UserId userId, SelectionId selectionId) {
+    Objects.requireNonNull(selectionId, "selectionId");
+    return base(userId) + ":selection:" + selectionId.value();
+  }
+
+  public static String betMember(BetId betId) {
+    return Objects.requireNonNull(betId, "betId").value().toString();
+  }
+
+  public static String stakeMember(BetId betId, long amount) {
+    SafeRedisNumber.requirePositive(amount, "amount");
+    return betMember(betId) + "|" + amount;
+  }
+
+  private static String base(UserId userId) {
+    Objects.requireNonNull(userId, "userId");
+    return PREFIX + "{" + userId.value() + "}";
+  }
+}


## `feat(history): project accepted histories atomically`

diff --git a/src/main/resources/scripts/history-record.lua b/src/main/resources/scripts/history-record.lua
new file mode 100644
index 0000000..a784578
--- /dev/null
+++ b/src/main/resources/scripts/history-record.lua
@@ -0,0 +1,56 @@
+local now = tonumber(ARGV[1])
+local betMember = ARGV[2]
+local stakeMember = ARGV[3]
+local rapidWindow = tonumber(ARGV[4])
+local stakeLimit = tonumber(ARGV[5])
+local repeatedWindow = tonumber(ARGV[6])
+local ttl = tonumber(ARGV[7])
+local maxExact = 9007199254740991
+
+if #KEYS < 3 or not now or now < 0 or now > maxExact then
+  return redis.error_reply("invalid history identity")
+end
+if not betMember or betMember == "" or not stakeMember or stakeMember == "" then
+  return redis.error_reply("invalid history member")
+end
+local encodedAmount = string.match(stakeMember, "|([0-9]+)$")
+local amount = encodedAmount and tonumber(encodedAmount) or nil
+if not amount or amount <= 0 or amount > maxExact then
+  return redis.error_reply("invalid history stake")
+end
+if not rapidWindow or rapidWindow <= 0 or not stakeLimit or stakeLimit <= 0 then
+  return redis.error_reply("invalid history bound")
+end
+if not repeatedWindow or repeatedWindow <= 0 or not ttl or ttl < repeatedWindow then
+  return redis.error_reply("invalid history retention")
+end
+for _, key in ipairs(KEYS) do
+  local keyType = redis.call("TYPE", key).ok
+  if keyType ~= "none" and keyType ~= "zset" then
+    return redis.error_reply("history key has wrong type")
+  end
+end
+
+redis.call("ZREMRANGEBYSCORE", KEYS[1], "-inf", now - rapidWindow)
+for index = 3, #KEYS do
+  redis.call("ZREMRANGEBYSCORE", KEYS[index], "-inf", now - repeatedWindow)
+end
+
+local betAdded = redis.call("ZADD", KEYS[1], "NX", now, betMember)
+local stakeAdded = redis.call("ZADD", KEYS[2], "NX", now, stakeMember)
+for index = 3, #KEYS do
+  redis.call("ZADD", KEYS[index], "NX", now, betMember)
+end
+
+local stakeCount = redis.call("ZCARD", KEYS[2])
+if stakeCount > stakeLimit then
+  redis.call("ZREMRANGEBYRANK", KEYS[2], 0, stakeCount - stakeLimit - 1)
+end
+for _, key in ipairs(KEYS) do
+  if redis.call("ZCARD", key) == 0 then
+    redis.call("DEL", key)
+  else
+    redis.call("PEXPIRE", key, ttl)
+  end
+end
+return {tostring(betAdded), tostring(stakeAdded)}


## `test(history): verify bounded hot-user histories`

diff --git a/src/test/java/com/sportsbook/risk/pattern/HistoryProjectionScriptTest.java b/src/test/java/com/sportsbook/risk/pattern/HistoryProjectionScriptTest.java
index 0558def..71a25cd 100644
--- a/src/test/java/com/sportsbook/risk/pattern/HistoryProjectionScriptTest.java
+++ b/src/test/java/com/sportsbook/risk/pattern/HistoryProjectionScriptTest.java
@@ -38,6 +38,22 @@ class HistoryProjectionScriptTest extends RedisTestSupport {
     assertThat(redis.hasKey(KEYS.get(2))).isFalse();
   }
 
+  @Test
+  void trimsHotHistoriesAndRefreshesIdleRetention() {
+    RedisScript<List> script = script("history-record.lua");
+    execute(script, 1, "bet-a", "bet-a|10");
+    execute(script, 2, "bet-b", "bet-b|20");
+    execute(script, 3, "bet-c", "bet-c|30");
+    execute(script, 4, "bet-d", "bet-d|40");
+
+    assertThat(redis.opsForZSet().range(KEYS.get(1), 0, -1))
+        .containsExactly("bet-b|20", "bet-c|30", "bet-d|40");
+    assertThat(redis.getExpire(KEYS.get(0))).isPositive();
+    execute(script, 60002, "bet-e", "bet-e|50");
+    assertThat(redis.opsForZSet().range(KEYS.get(0), 0, -1))
+        .containsExactly("bet-c", "bet-d", "bet-e");
+  }
+
   @SuppressWarnings("unchecked")
   private List<String> execute(
       RedisScript<List> script, long now, String betMember, String stakeMember) {


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
