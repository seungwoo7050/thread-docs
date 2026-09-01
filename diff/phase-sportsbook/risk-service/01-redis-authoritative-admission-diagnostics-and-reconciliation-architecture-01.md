# Redis 권위 상태 기반 승인·진단·재조정 아키텍처

## `docs(project): introduce risk ownership`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..dd16194
--- /dev/null
+++ b/README.md
@@ -0,0 +1,13 @@
+# Risk Service
+
+Risk Service owns sportsbook admission policy. It evaluates user limits and suspicious activity,
+then reserves capacity before a bet can debit funds.
+
+The service is intentionally small at its boundary:
+
+- Redis is the authoritative store for limits, reservations, and recent risk history.
+- Kafka carries accepted bet facts and non-authoritative risk signals.
+- Internal HTTP APIs expose reservation lifecycle, limit administration, and diagnostics.
+- Shared Protocol supplies the value objects and Avro records exchanged with other services.
+
+The implementation targets Java 17 and runs as a Spring Boot service.


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


## `feat(policy): compose diagnostic evaluation`

diff --git a/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java b/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java
new file mode 100644
index 0000000..936a2f5
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/service/RiskPolicyConfiguration.java
@@ -0,0 +1,37 @@
+package com.sportsbook.risk.service;
+
+import com.sportsbook.risk.event.RiskSignalPublisher;
+import com.sportsbook.risk.pattern.RuleEngine;
+import com.sportsbook.risk.pattern.rule.RapidBettingRule;
+import com.sportsbook.risk.pattern.rule.RepeatedSameSelectionRule;
+import com.sportsbook.risk.pattern.rule.SuddenStakeIncreaseRule;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.snapshot.RiskSnapshotReader;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.util.List;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+/** Composes deterministic diagnostic policy collaborators. */
+@Configuration
+public class RiskPolicyConfiguration {
+  @Bean
+  RuleEngine riskRuleEngine(RiskPatternProperties patterns) {
+    return new RuleEngine(
+        List.of(
+            new RapidBettingRule(patterns.rapidBetting()),
+            new SuddenStakeIncreaseRule(patterns.suddenStake()),
+            new RepeatedSameSelectionRule(patterns.repeatedSelection())));
+  }
+
+  @Bean
+  RiskCheckService riskCheckService(
+      RiskLimitProperties limits,
+      RiskSnapshotReader snapshots,
+      RuleEngine rules,
+      RiskSignalPublisher signals,
+      MeterRegistry meters) {
+    return new RiskCheckService(limits, snapshots, rules, signals, meters);
+  }
+}


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


## `feat(reservation): define atomic commit lifecycle`

diff --git a/src/main/resources/scripts/risk-commit.lua b/src/main/resources/scripts/risk-commit.lua
new file mode 100644
index 0000000..fefcb31
--- /dev/null
+++ b/src/main/resources/scripts/risk-commit.lua
@@ -0,0 +1,48 @@
+local maxExact = 9007199254740991
+local now = tonumber(ARGV[2])
+local retention = tonumber(ARGV[3])
+local token = ARGV[4]
+
+local function exact(text, positive)
+  if not text or not string.match(text, "^%d+$") then return nil end
+  local value = tonumber(text)
+  if not value or value > maxExact or (positive and value <= 0) then return nil end
+  return value
+end
+
+local function keyType(key)
+  return redis.call("TYPE", key).ok
+end
+
+if ARGV[1] ~= "1" or #KEYS ~= 2 or #ARGV ~= 12
+  or not exact(ARGV[2], false) or not exact(ARGV[3], true)
+  or not token or #token ~= 64 or not string.match(token, "^[0-9a-f]+$") then
+  return redis.error_reply("invalid commit request")
+end
+for index = 5, 12 do
+  if not exact(ARGV[index], true) then return redis.error_reply("invalid commit policy") end
+end
+
+local lifecycleType = keyType(KEYS[1])
+if lifecycleType == "none" then return "NOT_FOUND" end
+if lifecycleType ~= "hash" then return redis.error_reply("wrong reservation lifecycle type") end
+if redis.call("HGET", KEYS[1], "fingerprint") ~= token then return "CONFLICT" end
+
+local state = redis.call("HGET", KEYS[1], "state")
+if state == "COMMITTED" then return "REPLAYED" end
+if state == "EXPIRED" or state == "RELEASED" or state == "REJECTED" then
+  return "TOMBSTONED"
+end
+if state ~= "RESERVED" then return redis.error_reply("unknown reservation state") end
+
+local expiresAt = exact(redis.call("HGET", KEYS[1], "expiresAt"), false)
+if not expiresAt then return redis.error_reply("corrupt reservation expiry") end
+if expiresAt <= now then
+  redis.call("HSET", KEYS[1], "state", "EXPIRED", "expiredAt", string.format("%.0f", now))
+  redis.call("PEXPIRE", KEYS[1], retention)
+  return "EXPIRED"
+end
+
+redis.call("HSET", KEYS[1], "state", "COMMITTED", "committedAt", string.format("%.0f", now))
+redis.call("PEXPIRE", KEYS[1], retention)
+return "APPLIED"


## `feat(events): consume accepted bet events`

diff --git a/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
new file mode 100644
index 0000000..3ef0783
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/BetPlacedConsumer.java
@@ -0,0 +1,80 @@
+package com.sportsbook.risk.event;
+
+import java.time.Clock;
+import java.util.Objects;
+import java.util.function.Supplier;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.kafka.support.Acknowledgment;
+import org.springframework.kafka.support.KafkaHeaders;
+import org.springframework.messaging.handler.annotation.Header;
+import org.springframework.messaging.handler.annotation.Payload;
+import org.springframework.stereotype.Component;
+
+/** Reconciles accepted bets and durably quarantines only permanent input failures. */
+@Component
+public final class BetPlacedConsumer {
+  private final Supplier<AcceptedBetReconciler> reconciler;
+  private final BetPlacedDeadLetterPublisher deadLetters;
+  private final Clock clock;
+
+  @Autowired
+  public BetPlacedConsumer(
+      ObjectProvider<AcceptedBetReconciler> reconciler, BetPlacedDeadLetterPublisher deadLetters) {
+    this((Supplier<AcceptedBetReconciler>) reconciler::getIfUnique, deadLetters, Clock.systemUTC());
+  }
+
+  BetPlacedConsumer(
+      AcceptedBetReconciler reconciler, BetPlacedDeadLetterPublisher deadLetters, Clock clock) {
+    this(() -> Objects.requireNonNull(reconciler, "reconciler"), deadLetters, clock);
+  }
+
+  private BetPlacedConsumer(
+      Supplier<AcceptedBetReconciler> reconciler,
+      BetPlacedDeadLetterPublisher deadLetters,
+      Clock clock) {
+    this.reconciler = Objects.requireNonNull(reconciler, "reconciler");
+    this.deadLetters = Objects.requireNonNull(deadLetters, "deadLetters");
+    this.clock = Objects.requireNonNull(clock, "clock");
+  }
+
+  @KafkaListener(
+      topics = "${risk.topics.bet-placed:bet.placed.v1}",
+      groupId = "${spring.kafka.consumer.group-id:risk.bet-placed-consumer}")
+  public void onBetPlaced(
+      @Payload(required = false) byte[] payload,
+      @Header(value = KafkaHeaders.RECEIVED_KEY, required = false) String key,
+      Acknowledgment acknowledgment) {
+    Objects.requireNonNull(acknowledgment, "acknowledgment");
+    AcceptedBetEnvelope envelope;
+    try {
+      envelope = AcceptedBetEnvelope.decode(key, payload, clock.instant());
+    } catch (RuntimeException failure) {
+      deadLetter(key, payload, BetPlacedFailureReason.fromDecodeFailure(failure), acknowledgment);
+      return;
+    }
+
+    AcceptedBetReconciliation result =
+        Objects.requireNonNull(requiredReconciler().reconcile(envelope), "reconciliation result");
+    if (result.permanentFailure()) {
+      deadLetter(key, payload, result.failureReason(), acknowledgment);
+      return;
+    }
+    acknowledgment.acknowledge();
+  }
+
+  private AcceptedBetReconciler requiredReconciler() {
+    AcceptedBetReconciler value = reconciler.get();
+    if (value == null) {
+      throw new IllegalStateException("exactly one accepted-bet reconciler is required");
+    }
+    return value;
+  }
+
+  private void deadLetter(
+      String key, byte[] payload, BetPlacedFailureReason reason, Acknowledgment acknowledgment) {
+    deadLetters.publishAndAwait(key, payload, reason);
+    acknowledgment.acknowledge();
+  }
+}


## `feat(events): execute accepted projections`

diff --git a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
index 69be2ef..eec0d7b 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RedisRiskReservationStore.java
@@ -7,6 +7,7 @@ import com.sportsbook.risk.policy.RiskLimitProperties;
 import com.sportsbook.risk.policy.RiskPatternProperties;
 import com.sportsbook.risk.service.RiskCheckCommand;
 import java.time.Instant;
+import java.util.List;
 import java.util.Objects;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.data.redis.core.script.RedisScript;
@@ -19,6 +20,8 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
       RedisLuaScriptLoader.stringScript("risk-reserve.lua");
   private static final RedisScript<String> COMMIT =
       RedisLuaScriptLoader.stringScript("risk-commit.lua");
+  private static final RedisScript<String> PROJECT_ACCEPTED =
+      RedisLuaScriptLoader.stringScript("risk-project-accepted.lua");
   private static final RedisScript<String> RELEASE =
       RedisLuaScriptLoader.stringScript("risk-release.lua");
 
@@ -56,19 +59,27 @@ public final class RedisRiskReservationStore implements RiskReservationStore {
   public ReservationTransition commit(BetId betId, String token, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.commit(betId, token, now, reservations, patterns, history);
-    return executeTransition(COMMIT, request, "commit");
+    return executeTransition(COMMIT, request.keys(), request.arguments(), "commit");
+  }
+
+  @Override
+  public ReservationTransition projectAccepted(RiskCheckCommand command, String fingerprint) {
+    AcceptedProjectionRequest request =
+        AcceptedProjectionRequest.from(command, fingerprint, reservations, patterns, history);
+    return executeTransition(
+        PROJECT_ACCEPTED, request.keys(), request.arguments(), "accepted projection");
   }
 
   @Override
   public ReservationTransition release(BetId betId, Instant now) {
     ReservationTransitionRequest request =
         ReservationTransitionRequest.release(betId, now, reservations);
-    return executeTransition(RELEASE, request, "release");
+    return executeTransition(RELEASE, request.keys(), request.arguments(), "release");
   }
 
   private ReservationTransition executeTransition(
-      RedisScript<String> script, ReservationTransitionRequest request, String operation) {
-    String raw = redis.execute(script, request.keys(), request.arguments().toArray());
+      RedisScript<String> script, List<String> keys, List<String> arguments, String operation) {
+    String raw = redis.execute(script, keys, arguments.toArray());
     if (raw == null) {
       throw new IllegalStateException("Redis " + operation + " script returned no result");
     }
diff --git a/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java b/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
index 4d91954..fb652c1 100644
--- a/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
+++ b/src/main/java/com/sportsbook/risk/reservation/RiskReservationStore.java
@@ -10,5 +10,7 @@ public interface RiskReservationStore {
 
   ReservationTransition commit(BetId betId, String token, Instant now);
 
+  ReservationTransition projectAccepted(RiskCheckCommand command, String fingerprint);
+
   ReservationTransition release(BetId betId, Instant now);
 }


