# 행동 패턴 알고리즘과 제한된 이력 상태

## `feat(policy): classify pattern actions`

diff --git a/src/main/java/com/sportsbook/risk/policy/PatternAction.java b/src/main/java/com/sportsbook/risk/policy/PatternAction.java
new file mode 100644
index 0000000..a35fdbe
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/PatternAction.java
@@ -0,0 +1,8 @@
+package com.sportsbook.risk.policy;
+
+/** Operator-selected consequence when a suspicious activity rule matches. */
+public enum PatternAction {
+  SUSPECT,
+  REVIEW,
+  BLOCK
+}


## `feat(policy): configure rapid betting`

diff --git a/src/main/java/com/sportsbook/risk/policy/RapidBettingPolicy.java b/src/main/java/com/sportsbook/risk/policy/RapidBettingPolicy.java
new file mode 100644
index 0000000..ea6331d
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/RapidBettingPolicy.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.policy;
+
+import java.time.Duration;
+
+/** Threshold for bursts of accepted and currently reserved bets. */
+public record RapidBettingPolicy(
+    boolean enabled, Duration window, int maxBets, PatternAction action) {
+
+  private static final Duration DEFAULT_WINDOW = Duration.ofMinutes(1);
+  private static final int DEFAULT_MAX_BETS = 30;
+
+  public RapidBettingPolicy {
+    window = window == null ? DEFAULT_WINDOW : window;
+    maxBets = maxBets == 0 ? DEFAULT_MAX_BETS : maxBets;
+    action = action == null ? PatternAction.SUSPECT : action;
+    if (enabled && (window.isZero() || window.isNegative())) {
+      throw new IllegalArgumentException("rapid betting window must be positive");
+    }
+    if (enabled && maxBets <= 0) {
+      throw new IllegalArgumentException("rapid betting max-bets must be positive");
+    }
+  }
+
+  public static RapidBettingPolicy defaults() {
+    return new RapidBettingPolicy(false, DEFAULT_WINDOW, DEFAULT_MAX_BETS, PatternAction.SUSPECT);
+  }
+}


## `feat(policy): configure sudden stake increases`

diff --git a/src/main/java/com/sportsbook/risk/policy/SuddenStakePolicy.java b/src/main/java/com/sportsbook/risk/policy/SuddenStakePolicy.java
new file mode 100644
index 0000000..3c8a7eb
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/SuddenStakePolicy.java
@@ -0,0 +1,26 @@
+package com.sportsbook.risk.policy;
+
+/** Threshold for a candidate stake relative to recent same-currency stakes. */
+public record SuddenStakePolicy(
+    boolean enabled, int multiplier, int lookbackBets, PatternAction action) {
+
+  private static final int DEFAULT_MULTIPLIER = 10;
+  private static final int DEFAULT_LOOKBACK = 10;
+
+  public SuddenStakePolicy {
+    multiplier = multiplier == 0 ? DEFAULT_MULTIPLIER : multiplier;
+    lookbackBets = lookbackBets == 0 ? DEFAULT_LOOKBACK : lookbackBets;
+    action = action == null ? PatternAction.SUSPECT : action;
+    if (enabled && multiplier <= 1) {
+      throw new IllegalArgumentException("sudden stake multiplier must be greater than one");
+    }
+    if (enabled && lookbackBets <= 0) {
+      throw new IllegalArgumentException("sudden stake lookback must be positive");
+    }
+  }
+
+  public static SuddenStakePolicy defaults() {
+    return new SuddenStakePolicy(
+        false, DEFAULT_MULTIPLIER, DEFAULT_LOOKBACK, PatternAction.SUSPECT);
+  }
+}


## `feat(policy): configure repeated selections`

diff --git a/src/main/java/com/sportsbook/risk/policy/RepeatedSelectionPolicy.java b/src/main/java/com/sportsbook/risk/policy/RepeatedSelectionPolicy.java
new file mode 100644
index 0000000..1066914
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/policy/RepeatedSelectionPolicy.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.policy;
+
+import java.time.Duration;
+
+/** Threshold for repeatedly betting the same selection across currencies. */
+public record RepeatedSelectionPolicy(
+    boolean enabled, Duration window, int maxCount, PatternAction action) {
+
+  private static final Duration DEFAULT_WINDOW = Duration.ofHours(24);
+  private static final int DEFAULT_MAX_COUNT = 5;
+
+  public RepeatedSelectionPolicy {
+    window = window == null ? DEFAULT_WINDOW : window;
+    maxCount = maxCount == 0 ? DEFAULT_MAX_COUNT : maxCount;
+    action = action == null ? PatternAction.REVIEW : action;
+    if (enabled && (window.isZero() || window.isNegative())) {
+      throw new IllegalArgumentException("repeated selection window must be positive");
+    }
+    if (enabled && maxCount <= 0) {
+      throw new IllegalArgumentException("repeated selection max-count must be positive");
+    }
+  }
+
+  public static RepeatedSelectionPolicy defaults() {
+    return new RepeatedSelectionPolicy(
+        false, DEFAULT_WINDOW, DEFAULT_MAX_COUNT, PatternAction.REVIEW);
+  }
+}


## `feat(pattern): define typed pattern evaluations`

diff --git a/src/main/java/com/sportsbook/risk/pattern/PatternContext.java b/src/main/java/com/sportsbook/risk/pattern/PatternContext.java
new file mode 100644
index 0000000..bdd5b85
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/PatternContext.java
@@ -0,0 +1,32 @@
+package com.sportsbook.risk.pattern;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+
+/** Immutable candidate facts shared by every pattern rule. */
+public record PatternContext(
+    UserId userId, BetId betId, Money stake, List<SelectionId> selections, Instant evaluatedAt) {
+  public PatternContext {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(stake, "stake");
+    Objects.requireNonNull(selections, "selections");
+    Objects.requireNonNull(evaluatedAt, "evaluatedAt");
+    if (selections.isEmpty()) {
+      throw new IllegalArgumentException("selections must not be empty");
+    }
+    selections = List.copyOf(selections);
+  }
+
+  public static PatternContext from(RiskCheckCommand command) {
+    Objects.requireNonNull(command, "command");
+    return new PatternContext(
+        command.userId(), command.betId(), command.stake(), command.selectionIds(), command.now());
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/pattern/PatternMatch.java b/src/main/java/com/sportsbook/risk/pattern/PatternMatch.java
new file mode 100644
index 0000000..bb6e2ec
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/PatternMatch.java
@@ -0,0 +1,20 @@
+package com.sportsbook.risk.pattern;
+
+import com.sportsbook.risk.policy.PatternAction;
+import java.util.Objects;
+
+/** Stable evidence emitted when one configured rule matches a candidate. */
+public record PatternMatch(String rule, PatternAction action, String reason) {
+  public PatternMatch {
+    rule = requireText(rule, "rule");
+    Objects.requireNonNull(action, "action");
+    reason = requireText(reason, "reason");
+  }
+
+  private static String requireText(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
+  }
+}


## `feat(pattern): evaluate rules in stable order`

diff --git a/src/main/java/com/sportsbook/risk/pattern/PatternRule.java b/src/main/java/com/sportsbook/risk/pattern/PatternRule.java
new file mode 100644
index 0000000..f85bb1e
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/PatternRule.java
@@ -0,0 +1,13 @@
+package com.sportsbook.risk.pattern;
+
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import java.util.Optional;
+
+/** Pure pattern policy evaluated only from one candidate and one atomic snapshot. */
+public interface PatternRule {
+  String name();
+
+  int priority();
+
+  Optional<PatternMatch> evaluate(PatternContext context, PatternSnapshot snapshot);
+}
diff --git a/src/main/java/com/sportsbook/risk/pattern/RuleEngine.java b/src/main/java/com/sportsbook/risk/pattern/RuleEngine.java
new file mode 100644
index 0000000..34b82bf
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/RuleEngine.java
@@ -0,0 +1,36 @@
+package com.sportsbook.risk.pattern;
+
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import java.util.Comparator;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Objects;
+
+/** Evaluates every configured rule in an explicit deterministic order. */
+public final class RuleEngine {
+  private final List<PatternRule> rules;
+
+  public RuleEngine(List<PatternRule> rules) {
+    Objects.requireNonNull(rules, "rules");
+    if (rules.stream().anyMatch(Objects::isNull)) {
+      throw new IllegalArgumentException("rules must not contain null");
+    }
+    if (new HashSet<>(rules.stream().map(PatternRule::name).toList()).size() != rules.size()) {
+      throw new IllegalArgumentException("rule names must be unique");
+    }
+    this.rules =
+        rules.stream()
+            .sorted(Comparator.comparingInt(PatternRule::priority).thenComparing(PatternRule::name))
+            .toList();
+  }
+
+  public List<PatternMatch> evaluate(PatternContext context, PatternSnapshot snapshot) {
+    Objects.requireNonNull(context, "context");
+    Objects.requireNonNull(snapshot, "snapshot");
+    return rules.stream().flatMap(rule -> rule.evaluate(context, snapshot).stream()).toList();
+  }
+
+  public List<String> ruleOrder() {
+    return rules.stream().map(PatternRule::name).toList();
+  }
+}


## `test(pattern): verify ordered rule evaluation`

diff --git a/src/test/java/com/sportsbook/risk/pattern/RuleEngineTest.java b/src/test/java/com/sportsbook/risk/pattern/RuleEngineTest.java
new file mode 100644
index 0000000..08de3f5
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/pattern/RuleEngineTest.java
@@ -0,0 +1,77 @@
+package com.sportsbook.risk.pattern;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import com.sportsbook.risk.snapshot.SnapshotSlot;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.UUID;
+import java.util.concurrent.atomic.AtomicInteger;
+import org.junit.jupiter.api.Test;
+
+class RuleEngineTest {
+  private static final SelectionId SELECTION = SelectionId.of(new UUID(0, 3));
+  private static final PatternContext CONTEXT =
+      new PatternContext(
+          UserId.of(new UUID(0, 1)),
+          BetId.of(new UUID(0, 2)),
+          Money.krw(10),
+          List.of(SELECTION),
+          Instant.EPOCH);
+  private static final PatternSnapshot SNAPSHOT =
+      new PatternSnapshot(
+          SnapshotSlot.success(0L),
+          SnapshotSlot.success(List.of()),
+          Map.of(SELECTION, SnapshotSlot.success(0L)));
+
+  @Test
+  void evaluatesEveryMatchInPriorityOrder() {
+    AtomicInteger calls = new AtomicInteger();
+    PatternRule later = rule("later", 20, PatternAction.REVIEW, calls);
+    PatternRule blocker = rule("blocker", 10, PatternAction.BLOCK, calls);
+    RuleEngine engine = new RuleEngine(List.of(later, blocker));
+
+    assertThat(engine.ruleOrder()).containsExactly("blocker", "later");
+    assertThat(engine.evaluate(CONTEXT, SNAPSHOT))
+        .extracting(PatternMatch::rule)
+        .containsExactly("blocker", "later");
+    assertThat(calls).hasValue(2);
+  }
+
+  @Test
+  void rejectsAmbiguousRuleNames() {
+    PatternRule first = rule("same", 1, PatternAction.SUSPECT, new AtomicInteger());
+    PatternRule second = rule("same", 2, PatternAction.REVIEW, new AtomicInteger());
+
+    assertThatThrownBy(() -> new RuleEngine(List.of(first, second)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThat(new RuleEngine(List.of()).evaluate(CONTEXT, SNAPSHOT)).isEmpty();
+  }
+
+  private static PatternRule rule(
+      String name, int priority, PatternAction action, AtomicInteger calls) {
+    return new PatternRule() {
+      public String name() {
+        return name;
+      }
+
+      public int priority() {
+        return priority;
+      }
+
+      public Optional<PatternMatch> evaluate(PatternContext context, PatternSnapshot snapshot) {
+        calls.incrementAndGet();
+        return Optional.of(new PatternMatch(name, action, "matched"));
+      }
+    };
+  }
+}


## `feat(pattern): detect rapid betting`

diff --git a/src/main/java/com/sportsbook/risk/pattern/rule/RapidBettingRule.java b/src/main/java/com/sportsbook/risk/pattern/rule/RapidBettingRule.java
new file mode 100644
index 0000000..79fcb55
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/rule/RapidBettingRule.java
@@ -0,0 +1,46 @@
+package com.sportsbook.risk.pattern.rule;
+
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.pattern.PatternRule;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import java.util.Objects;
+import java.util.Optional;
+
+/** Detects a candidate that reaches the configured rapid-bet threshold. */
+public final class RapidBettingRule implements PatternRule {
+  public static final String NAME = "RAPID_BETTING";
+  private static final int PRIORITY = 10;
+  private final RapidBettingPolicy policy;
+
+  public RapidBettingRule(RapidBettingPolicy policy) {
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
+    long candidateCount =
+        SafeRedisNumber.add(snapshot.recentBetCount().valueOrThrow(), 1, "rapid bet count");
+    if (candidateCount < policy.maxBets()) {
+      return Optional.empty();
+    }
+    return Optional.of(new PatternMatch(NAME, policy.action(), "rapid betting threshold reached"));
+  }
+}


## `test(pattern): verify rapid betting boundaries`

diff --git a/src/test/java/com/sportsbook/risk/pattern/rule/RapidBettingRuleTest.java b/src/test/java/com/sportsbook/risk/pattern/rule/RapidBettingRuleTest.java
new file mode 100644
index 0000000..30d0db8
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/pattern/rule/RapidBettingRuleTest.java
@@ -0,0 +1,64 @@
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
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import com.sportsbook.risk.snapshot.SnapshotSlot;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RapidBettingRuleTest {
+  private static final SelectionId SELECTION = SelectionId.of(new UUID(0, 3));
+  private static final PatternContext CONTEXT =
+      new PatternContext(
+          UserId.of(new UUID(0, 1)),
+          BetId.of(new UUID(0, 2)),
+          Money.krw(10),
+          List.of(SELECTION),
+          Instant.EPOCH);
+
+  @Test
+  void matchesWhenTheCandidateReachesTheThreshold() {
+    RapidBettingRule rule =
+        new RapidBettingRule(
+            new RapidBettingPolicy(true, Duration.ofMinutes(1), 3, PatternAction.BLOCK));
+
+    assertThat(rule.evaluate(CONTEXT, snapshot(1))).isEmpty();
+    assertThat(rule.evaluate(CONTEXT, snapshot(2)))
+        .hasValueSatisfying(
+            match -> {
+              assertThat(match.rule()).isEqualTo(RapidBettingRule.NAME);
+              assertThat(match.action()).isEqualTo(PatternAction.BLOCK);
+            });
+  }
+
+  @Test
+  void disabledRulesDoNotConsumeUnavailableFacts() {
+    RapidBettingRule rule = new RapidBettingRule(RapidBettingPolicy.defaults());
+    PatternSnapshot unavailable =
+        new PatternSnapshot(
+            SnapshotSlot.failure("redis unavailable"),
+            SnapshotSlot.success(List.of()),
+            Map.of(SELECTION, SnapshotSlot.success(0L)));
+
+    assertThat(rule.evaluate(CONTEXT, unavailable)).isEmpty();
+  }
+
+  private static PatternSnapshot snapshot(long recent) {
+    return new PatternSnapshot(
+        SnapshotSlot.success(recent),
+        SnapshotSlot.success(List.of()),
+        Map.of(SELECTION, SnapshotSlot.success(0L)));
+  }
+}


## `feat(pattern): detect sudden stake increases`

diff --git a/src/main/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRule.java b/src/main/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRule.java
new file mode 100644
index 0000000..c6f5fbe
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/pattern/rule/SuddenStakeIncreaseRule.java
@@ -0,0 +1,70 @@
+package com.sportsbook.risk.pattern.rule;
+
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.pattern.PatternRule;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import com.sportsbook.risk.snapshot.PatternSnapshot;
+import java.math.BigInteger;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Objects;
+import java.util.Optional;
+
+/** Detects a large candidate relative to the median recent same-currency stake. */
+public final class SuddenStakeIncreaseRule implements PatternRule {
+  public static final String NAME = "SUDDEN_STAKE_INCREASE";
+  private static final int PRIORITY = 20;
+  private static final BigInteger TWO = BigInteger.valueOf(2);
+
+  private final SuddenStakePolicy policy;
+
+  public SuddenStakeIncreaseRule(SuddenStakePolicy policy) {
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
+    List<Long> recent = snapshot.recentStakes().valueOrThrow();
+    if (recent.size() < policy.lookbackBets()) {
+      return Optional.empty();
+    }
+    List<Long> sample =
+        new ArrayList<>(recent.subList(recent.size() - policy.lookbackBets(), recent.size()));
+    sample.forEach(value -> SafeRedisNumber.requireNonNegative(value, "recent stake"));
+    sample.sort(Long::compareTo);
+    BigInteger doubledMedian = doubledMedian(sample);
+    if (doubledMedian.signum() == 0) {
+      return Optional.empty();
+    }
+    BigInteger candidate = BigInteger.valueOf(context.stake().amount()).multiply(TWO);
+    BigInteger threshold = doubledMedian.multiply(BigInteger.valueOf(policy.multiplier()));
+    return candidate.compareTo(threshold) < 0
+        ? Optional.empty()
+        : Optional.of(new PatternMatch(NAME, policy.action(), "sudden stake threshold reached"));
+  }
+
+  private static BigInteger doubledMedian(List<Long> sorted) {
+    int middle = sorted.size() / 2;
+    if (sorted.size() % 2 == 1) {
+      return BigInteger.valueOf(sorted.get(middle)).multiply(TWO);
+    }
+    return BigInteger.valueOf(sorted.get(middle - 1)).add(BigInteger.valueOf(sorted.get(middle)));
+  }
+}


