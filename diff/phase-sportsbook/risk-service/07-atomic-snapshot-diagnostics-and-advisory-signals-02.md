## `feat(snapshot): validate snapshot policy inputs`

diff --git a/src/main/resources/scripts/risk-snapshot.lua b/src/main/resources/scripts/risk-snapshot.lua
index 33e15eb..19cfedf 100644
--- a/src/main/resources/scripts/risk-snapshot.lua
+++ b/src/main/resources/scripts/risk-snapshot.lua
@@ -94,6 +94,26 @@ if not now or now < 0 or now > maxExact or not count or count < 1
   or not retention or retention <= 0 or #KEYS ~= 17 + count * 2 then
   return redis.error_reply("invalid snapshot request")
 end
+for _, index in ipairs({3, 4, 5, 6, 8, 10, 12}) do
+  local value = tonumber(ARGV[index])
+  if not value or value <= 0 or value > maxExact then
+    return redis.error_reply("invalid snapshot policy")
+  end
+end
+for _, index in ipairs({7, 9, 11}) do
+  if ARGV[index] ~= "0" and ARGV[index] ~= "1" then
+    return redis.error_reply("invalid snapshot policy flag")
+  end
+end
+if not userId or not string.match(userId, "^[0-9a-f%-]+$")
+  or not currency or not string.match(currency, "^[A-Z]+$") then
+  return redis.error_reply("invalid snapshot identity")
+end
+for index = 1, count do
+  if not string.match(ARGV[15 + index] or "", "^[0-9a-f%-]+$") then
+    return redis.error_reply("invalid snapshot selection")
+  end
+end
 
 local activeBase = "risk:reservations:user:{" .. userId .. "}"
 local plans, stakeTotals, selectionTotal = {}, {}, 0


## `feat(snapshot): define precision-safe snapshot wire`

diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java
new file mode 100644
index 0000000..a49c532
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotWire.java
@@ -0,0 +1,16 @@
+package com.sportsbook.risk.snapshot;
+
+import java.util.List;
+import java.util.Map;
+
+/** Precision-safe JSON contract returned by the combined Redis snapshot script. */
+record RiskSnapshotWire(
+    String version, String expired, Map<String, LimitSlot> limits, PatternFacts patterns) {
+  record LimitSlot(Boolean ok, String committed, String active, String override, String error) {}
+
+  record FactSlot(Boolean ok, String value, List<String> values, String error) {}
+
+  record PatternFacts(FactSlot rapid, FactSlot stakes, List<SelectionFact> selections) {}
+
+  record SelectionFact(String selectionId, FactSlot slot) {}
+}
diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java b/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java
new file mode 100644
index 0000000..7561aff
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskWireNumbers.java
@@ -0,0 +1,23 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.risk.policy.SafeRedisNumber;
+
+/** Canonical integer parser for values kept as strings across Redis JSON. */
+final class RiskWireNumbers {
+  private RiskWireNumbers() {}
+
+  static long exact(String raw, String name) {
+    if (raw == null || !raw.matches("0|[1-9][0-9]*")) {
+      throw malformed(name);
+    }
+    try {
+      return SafeRedisNumber.requireNonNegative(Long.parseLong(raw), name);
+    } catch (IllegalArgumentException failure) {
+      throw new IllegalStateException("malformed snapshot integer: " + name, failure);
+    }
+  }
+
+  static IllegalStateException malformed(String name) {
+    return new IllegalStateException("malformed snapshot field: " + name);
+  }
+}


## `feat(snapshot): expose atomic Redis snapshots`

diff --git a/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java b/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java
index c810620..17d984f 100644
--- a/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java
+++ b/src/main/java/com/sportsbook/risk/counter/RedisLuaScriptLoader.java
@@ -14,4 +14,11 @@ public final class RedisLuaScriptLoader {
     script.setResultType(List.class);
     return script;
   }
+
+  public static DefaultRedisScript<String> stringScript(String name) {
+    DefaultRedisScript<String> script = new DefaultRedisScript<>();
+    script.setLocation(new ClassPathResource("scripts/" + name));
+    script.setResultType(String.class);
+    return script;
+  }
 }
diff --git a/src/main/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReader.java b/src/main/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReader.java
new file mode 100644
index 0000000..a2f33bc
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReader.java
@@ -0,0 +1,41 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import java.util.Objects;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.script.RedisScript;
+import org.springframework.stereotype.Component;
+
+/** Standalone-Redis implementation of the combined snapshot boundary. */
+@Component
+public final class RedisRiskSnapshotReader implements RiskSnapshotReader {
+  private static final RedisScript<String> SCRIPT =
+      RedisLuaScriptLoader.stringScript("risk-snapshot.lua");
+
+  private final StringRedisTemplate redis;
+  private final RiskPatternProperties patterns;
+  private final RiskReservationProperties reservations;
+  private final RiskSnapshotWireMapper mapper;
+
+  public RedisRiskSnapshotReader(
+      StringRedisTemplate redis,
+      RiskPatternProperties patterns,
+      RiskReservationProperties reservations,
+      RiskSnapshotWireMapper mapper) {
+    this.redis = Objects.requireNonNull(redis, "redis");
+    this.patterns = Objects.requireNonNull(patterns, "patterns");
+    this.reservations = Objects.requireNonNull(reservations, "reservations");
+    this.mapper = Objects.requireNonNull(mapper, "mapper");
+  }
+
+  @Override
+  public RiskSnapshot read(PatternContext context) {
+    RiskSnapshotScriptRequest request =
+        RiskSnapshotScriptRequest.from(context, patterns, reservations);
+    String raw = redis.execute(SCRIPT, request.keys(), request.arguments().toArray());
+    return mapper.map(raw, context.selections()).snapshot();
+  }
+}


## `test(snapshot): verify atomic Redis reads`

diff --git a/src/test/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReaderTest.java b/src/test/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReaderTest.java
new file mode 100644
index 0000000..ca854a3
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/snapshot/RedisRiskSnapshotReaderTest.java
@@ -0,0 +1,57 @@
+package com.sportsbook.risk.snapshot;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.RapidBettingPolicy;
+import com.sportsbook.risk.policy.RepeatedSelectionPolicy;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.policy.SuddenStakePolicy;
+import com.sportsbook.risk.reservation.RiskReservationProperties;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RedisRiskSnapshotReaderTest extends RedisTestSupport {
+  @Test
+  void readsOneAtomicRedisSnapshot() {
+    SelectionId selection = SelectionId.of(new UUID(0, 3));
+    PatternContext context =
+        new PatternContext(
+            UserId.of(new UUID(0, 1)),
+            BetId.of(new UUID(0, 2)),
+            Money.krw(10),
+            List.of(selection),
+            Instant.ofEpochMilli(200_000_000));
+
+    RiskSnapshot snapshot = reader().read(context);
+
+    assertThat(snapshot.limits().require(LimitType.STAKE_DAILY).current()).isZero();
+    assertThat(snapshot.patterns().recentBetCount().valueOrThrow()).isZero();
+    assertThat(snapshot.patterns().recentStakes().valueOrThrow()).isEmpty();
+    assertThat(snapshot.patterns().selectionCount(selection).valueOrThrow()).isZero();
+  }
+
+  private RedisRiskSnapshotReader reader() {
+    RiskPatternProperties patterns =
+        new RiskPatternProperties(
+            new RapidBettingPolicy(true, Duration.ofMinutes(1), 30, PatternAction.SUSPECT),
+            new SuddenStakePolicy(true, 10, 10, PatternAction.SUSPECT),
+            new RepeatedSelectionPolicy(true, Duration.ofHours(24), 5, PatternAction.REVIEW));
+    return new RedisRiskSnapshotReader(
+        redis,
+        patterns,
+        new RiskReservationProperties(null, null),
+        new RiskSnapshotWireMapper(new ObjectMapper()));
+  }
+}


## `feat(check): define diagnostic decisions`

diff --git a/src/main/java/com/sportsbook/risk/service/LimitRejection.java b/src/main/java/com/sportsbook/risk/service/LimitRejection.java
new file mode 100644
index 0000000..70a449f
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/LimitRejection.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.Objects;
+
+/** The first configured capacity exceeded by one diagnostic candidate. */
+public record LimitRejection(
+    String reason, LimitType type, Currency currency, long current, long limit, long requested) {
+  public LimitRejection {
+    if (reason == null || reason.isBlank()) {
+      throw new IllegalArgumentException("reason must not be blank");
+    }
+    if (type == null && !"SINGLE_BET_MAX_EXCEEDED".equals(reason)) {
+      throw new IllegalArgumentException("only the single-bet limit has no rolling type");
+    }
+    if (type != null && !reason.equals(type.name() + "_LIMIT_EXCEEDED")) {
+      throw new IllegalArgumentException("reason does not match the rolling type");
+    }
+    if (type == null || type.currencyScoped()) {
+      Objects.requireNonNull(currency, "currency");
+    } else if (currency != null) {
+      throw new IllegalArgumentException("count rejection must not contain currency");
+    }
+    SafeRedisNumber.requireNonNegative(current, "current");
+    SafeRedisNumber.requireNonNegative(limit, "limit");
+    SafeRedisNumber.requirePositive(requested, "requested");
+    if (current <= limit && requested <= limit - current) {
+      throw new IllegalArgumentException("candidate does not exceed the limit");
+    }
+  }
+
+  public static LimitRejection single(Currency currency, long limit, long requested) {
+    return new LimitRejection("SINGLE_BET_MAX_EXCEEDED", null, currency, 0L, limit, requested);
+  }
+
+  public static LimitRejection rolling(
+      LimitType type, Currency currency, long current, long limit, long requested) {
+    Objects.requireNonNull(type, "type");
+    return new LimitRejection(
+        type.name() + "_LIMIT_EXCEEDED", type, currency, current, limit, requested);
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/service/RiskCheckOutcome.java b/src/main/java/com/sportsbook/risk/service/RiskCheckOutcome.java
new file mode 100644
index 0000000..4fd61a3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/RiskCheckOutcome.java
@@ -0,0 +1,34 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.policy.PatternAction;
+import java.util.List;
+import java.util.Objects;
+
+/** Diagnostic result containing one limit rejection and every ordered pattern signal. */
+public record RiskCheckOutcome(
+    boolean approved, LimitRejection rejection, List<PatternMatch> patterns) {
+  public RiskCheckOutcome {
+    Objects.requireNonNull(patterns, "patterns");
+    patterns = List.copyOf(patterns);
+    boolean blocked = patterns.stream().anyMatch(match -> match.action() == PatternAction.BLOCK);
+    if (approved && (rejection != null || blocked)) {
+      throw new IllegalArgumentException("approved outcome contains a blocking decision");
+    }
+    if (!approved && rejection == null && !blocked) {
+      throw new IllegalArgumentException("rejected outcome has no blocking decision");
+    }
+  }
+
+  public static RiskCheckOutcome approved(List<PatternMatch> patterns) {
+    return new RiskCheckOutcome(true, null, patterns);
+  }
+
+  public static RiskCheckOutcome rejectedByLimit(LimitRejection rejection) {
+    return new RiskCheckOutcome(false, Objects.requireNonNull(rejection, "rejection"), List.of());
+  }
+
+  public static RiskCheckOutcome rejectedByPattern(List<PatternMatch> patterns) {
+    return new RiskCheckOutcome(false, null, patterns);
+  }
+}


## `feat(check): evaluate monetary capacity`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskCheckService.java b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
new file mode 100644
index 0000000..dea63bf
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
@@ -0,0 +1,85 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.event.RiskSignalPublisher;
+import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.pattern.RuleEngine;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.snapshot.LimitSnapshot;
+import com.sportsbook.risk.snapshot.RiskSnapshot;
+import com.sportsbook.risk.snapshot.RiskSnapshotReader;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Timer;
+import java.util.List;
+import java.util.Objects;
+
+/** Read-only policy diagnostics; betting admission uses the atomic reservation boundary. */
+public final class RiskCheckService {
+  private static final List<LimitType> MONETARY_LIMITS =
+      List.of(LimitType.STAKE_DAILY, LimitType.STAKE_WEEKLY, LimitType.STAKE_MONTHLY);
+
+  private final RiskLimitProperties policy;
+  private final RiskSnapshotReader snapshots;
+  private final RuleEngine rules;
+  private final RiskSignalPublisher signals;
+  private final MeterRegistry meters;
+  private final Timer latency;
+
+  public RiskCheckService(
+      RiskLimitProperties policy,
+      RiskSnapshotReader snapshots,
+      RuleEngine rules,
+      RiskSignalPublisher signals,
+      MeterRegistry meters) {
+    this.policy = Objects.requireNonNull(policy, "policy");
+    this.snapshots = Objects.requireNonNull(snapshots, "snapshots");
+    this.rules = Objects.requireNonNull(rules, "rules");
+    this.signals = Objects.requireNonNull(signals, "signals");
+    this.meters = Objects.requireNonNull(meters, "meters");
+    this.latency = Timer.builder("risk.check.latency").register(meters);
+  }
+
+  public RiskCheckOutcome check(RiskCheckCommand command) {
+    Objects.requireNonNull(command, "command");
+    return latency.record(() -> evaluate(command));
+  }
+
+  private RiskCheckOutcome evaluate(RiskCheckCommand command) {
+    Currency currency = command.stake().currency();
+    long requested = command.stake().amount();
+    long singleLimit = policy.singleBetMax(currency);
+    if (requested > singleLimit) {
+      return reject(command, LimitRejection.single(currency, singleLimit, requested));
+    }
+
+    RiskSnapshot snapshot = snapshots.read(PatternContext.from(command));
+    for (LimitType type : MONETARY_LIMITS) {
+      LimitSnapshot.Value value = snapshot.limits().require(type);
+      long current = value.current();
+      long limit = value.effectiveLimit(policy.limit(type, currency));
+      if (exceeds(current, requested, limit)) {
+        return reject(command, LimitRejection.rolling(type, currency, current, limit, requested));
+      }
+    }
+    return RiskCheckOutcome.approved(List.of());
+  }
+
+  private RiskCheckOutcome reject(RiskCheckCommand command, LimitRejection rejection) {
+    meters.counter("risk.limit.violations", "reason", rejection.reason()).increment();
+    if (rejection.type() != null) {
+      signals.publishLimit(
+          command.userId(),
+          rejection.type(),
+          rejection.current(),
+          rejection.limit(),
+          command.stake(),
+          command.now());
+    }
+    return RiskCheckOutcome.rejectedByLimit(rejection);
+  }
+
+  private static boolean exceeds(long current, long requested, long limit) {
+    return current > limit || requested > limit - current;
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotReader.java b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotReader.java
new file mode 100644
index 0000000..db07cac
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/snapshot/RiskSnapshotReader.java
@@ -0,0 +1,8 @@
+package com.sportsbook.risk.snapshot;
+
+import com.sportsbook.risk.pattern.PatternContext;
+
+/** Reads all diagnostic facts for one candidate in a single Redis operation. */
+public interface RiskSnapshotReader {
+  RiskSnapshot read(PatternContext context);
+}


## `feat(check): evaluate selection capacity`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskCheckService.java b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
index dea63bf..2600f67 100644
--- a/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
+++ b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
@@ -62,6 +62,21 @@ public final class RiskCheckService {
         return reject(command, LimitRejection.rolling(type, currency, current, limit, requested));
       }
     }
+    LimitSnapshot.Value selections = snapshot.limits().require(LimitType.SELECTIONS_PER_MINUTE);
+    long selectionCurrent = selections.current();
+    long selectionLimit =
+        selections.effectiveLimit(policy.limit(LimitType.SELECTIONS_PER_MINUTE, currency));
+    long requestedSelections = command.selectionIds().size();
+    if (exceeds(selectionCurrent, requestedSelections, selectionLimit)) {
+      return reject(
+          command,
+          LimitRejection.rolling(
+              LimitType.SELECTIONS_PER_MINUTE,
+              null,
+              selectionCurrent,
+              selectionLimit,
+              requestedSelections));
+    }
     return RiskCheckOutcome.approved(List.of());
   }
 


## `feat(check): evaluate ordered pattern signals`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskCheckService.java b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
index 2600f67..4594cfe 100644
--- a/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
+++ b/src/main/java/com/sportsbook/risk/service/RiskCheckService.java
@@ -4,7 +4,9 @@ import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.event.RiskSignalPublisher;
 import com.sportsbook.risk.pattern.PatternContext;
+import com.sportsbook.risk.pattern.PatternMatch;
 import com.sportsbook.risk.pattern.RuleEngine;
+import com.sportsbook.risk.policy.PatternAction;
 import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.snapshot.LimitSnapshot;
 import com.sportsbook.risk.snapshot.RiskSnapshot;
@@ -53,7 +55,8 @@ public final class RiskCheckService {
       return reject(command, LimitRejection.single(currency, singleLimit, requested));
     }
 
-    RiskSnapshot snapshot = snapshots.read(PatternContext.from(command));
+    PatternContext context = PatternContext.from(command);
+    RiskSnapshot snapshot = snapshots.read(context);
     for (LimitType type : MONETARY_LIMITS) {
       LimitSnapshot.Value value = snapshot.limits().require(type);
       long current = value.current();
@@ -77,7 +80,16 @@ public final class RiskCheckService {
               selectionLimit,
               requestedSelections));
     }
-    return RiskCheckOutcome.approved(List.of());
+    List<PatternMatch> matches = rules.evaluate(context, snapshot.patterns());
+    for (PatternMatch match : matches) {
+      meters
+          .counter("risk.pattern.flags", "rule", match.rule(), "action", match.action().name())
+          .increment();
+      signals.publishPattern(command.userId(), match, command.now());
+    }
+    return matches.stream().anyMatch(match -> match.action() == PatternAction.BLOCK)
+        ? RiskCheckOutcome.rejectedByPattern(matches)
+        : RiskCheckOutcome.approved(matches);
   }
 
   private RiskCheckOutcome reject(RiskCheckCommand command, LimitRejection rejection) {


