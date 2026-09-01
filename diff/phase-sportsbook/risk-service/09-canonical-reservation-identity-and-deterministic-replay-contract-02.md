## `feat(reservation): assemble reserve script arguments`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationScriptArguments.java b/src/main/java/com/sportsbook/risk/reservation/ReservationScriptArguments.java
new file mode 100644
index 0000000..c09b5ff
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationScriptArguments.java
@@ -0,0 +1,67 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.ArrayList;
+import java.util.List;
+
+/** Canonical precision-safe argument order consumed by reservation admission. */
+final class ReservationScriptArguments {
+  private ReservationScriptArguments() {}
+
+  static List<String> from(
+      RiskCheckCommand command,
+      RiskLimitProperties limits,
+      RiskPatternProperties patterns,
+      RiskReservationProperties reservations,
+      RiskHistoryProperties history) {
+    var currency = command.stake().currency();
+    List<String> values = new ArrayList<>();
+    values.add("1");
+    values.add(Long.toString(command.now().toEpochMilli()));
+    values.add(Long.toString(reservations.lease().toMillis()));
+    values.add(Long.toString(reservations.retention().toMillis()));
+    values.add(ReservationFingerprint.of(command));
+    values.add(command.userId().value().toString());
+    values.add(command.betId().value().toString());
+    values.add(Long.toString(command.stake().amount()));
+    values.add(currency.name());
+    values.add(Integer.toString(command.selectionIds().size()));
+    values.add(Long.toString(limits.singleBetMax(currency)));
+    values.add(Long.toString(limits.limit(LimitType.STAKE_DAILY, currency)));
+    values.add(Long.toString(limits.limit(LimitType.STAKE_WEEKLY, currency)));
+    values.add(Long.toString(limits.limit(LimitType.STAKE_MONTHLY, currency)));
+    values.add(Long.toString(limits.limit(LimitType.SELECTIONS_PER_MINUTE, currency)));
+    values.add(Long.toString(LimitType.STAKE_DAILY.window().toMillis()));
+    values.add(Long.toString(LimitType.STAKE_WEEKLY.window().toMillis()));
+    values.add(Long.toString(LimitType.STAKE_MONTHLY.window().toMillis()));
+    values.add(Long.toString(LimitType.SELECTIONS_PER_MINUTE.window().toMillis()));
+    values.add(enabled(patterns.rapidBetting().enabled()));
+    values.add(Long.toString(patterns.rapidBetting().window().toMillis()));
+    values.add(Integer.toString(patterns.rapidBetting().maxBets()));
+    values.add(patterns.rapidBetting().action().name());
+    values.add(enabled(patterns.suddenStake().enabled()));
+    values.add(Integer.toString(patterns.suddenStake().multiplier()));
+    values.add(Integer.toString(patterns.suddenStake().lookbackBets()));
+    values.add(patterns.suddenStake().action().name());
+    values.add(enabled(patterns.repeatedSelection().enabled()));
+    values.add(Long.toString(patterns.repeatedSelection().window().toMillis()));
+    values.add(Integer.toString(patterns.repeatedSelection().maxCount()));
+    values.add(patterns.repeatedSelection().action().name());
+    values.add(Long.toString(history.idleRetention().toMillis()));
+    values.add(Integer.toString(history.maxStakeSamples()));
+    command.selectionIds().stream()
+        .map(SelectionId::value)
+        .map(Object::toString)
+        .forEach(values::add);
+    return List.copyOf(values);
+  }
+
+  private static String enabled(boolean value) {
+    return value ? "1" : "0";
+  }
+}


## `feat(reservation): validate reservation wire results`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationWire.java b/src/main/java/com/sportsbook/risk/reservation/ReservationWire.java
new file mode 100644
index 0000000..6e9f2a7
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationWire.java
@@ -0,0 +1,13 @@
+package com.sportsbook.risk.reservation;
+
+/** Precision-safe JSON result emitted by the reservation admission script. */
+record ReservationWire(
+    String version,
+    String expired,
+    String status,
+    String state,
+    String expiresAt,
+    String token,
+    Boolean replayed,
+    String rejection,
+    String patternsJson) {}
diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationWireValidator.java b/src/main/java/com/sportsbook/risk/reservation/ReservationWireValidator.java
new file mode 100644
index 0000000..e5e6249
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationWireValidator.java
@@ -0,0 +1,85 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.time.Instant;
+import java.util.List;
+
+/** Validates mutually exclusive reservation result shapes after JSON decoding. */
+final class ReservationWireValidator {
+  private ReservationWireValidator() {}
+
+  static ReservationDecision decision(ReservationWire wire, List<PatternMatch> patterns) {
+    return switch (wire.status()) {
+      case "APPROVED" -> approved(wire, patterns);
+      case "REJECTED" -> rejected(wire, patterns);
+      case "CONFLICT" -> conflict(wire);
+      default -> throw malformed();
+    };
+  }
+
+  static long exact(String value, String name) {
+    if (value == null || !value.matches("0|[1-9][0-9]*")) {
+      throw malformed();
+    }
+    try {
+      return SafeRedisNumber.requireNonNegative(Long.parseLong(value), name);
+    } catch (IllegalArgumentException failure) {
+      throw malformed(failure);
+    }
+  }
+
+  private static ReservationDecision approved(ReservationWire wire, List<PatternMatch> patterns) {
+    if (wire.rejection() != null || wire.patternsJson() == null || !token(wire.token())) {
+      throw malformed();
+    }
+    ReservationState state;
+    try {
+      state = ReservationState.valueOf(wire.state());
+    } catch (RuntimeException failure) {
+      throw malformed(failure);
+    }
+    return ReservationDecision.approved(
+        state,
+        Instant.ofEpochMilli(exact(wire.expiresAt(), "expiresAt")),
+        wire.token(),
+        wire.replayed(),
+        patterns);
+  }
+
+  private static ReservationDecision rejected(ReservationWire wire, List<PatternMatch> patterns) {
+    if (wire.state() != null
+        || wire.expiresAt() != null
+        || wire.token() != null
+        || wire.rejection() == null
+        || wire.rejection().isBlank()
+        || wire.patternsJson() == null) {
+      throw malformed();
+    }
+    return ReservationDecision.rejected(wire.rejection(), wire.replayed(), patterns);
+  }
+
+  private static ReservationDecision conflict(ReservationWire wire) {
+    if (wire.replayed()
+        || wire.state() != null
+        || wire.expiresAt() != null
+        || wire.token() != null
+        || wire.rejection() != null
+        || wire.patternsJson() != null) {
+      throw malformed();
+    }
+    return ReservationDecision.conflict();
+  }
+
+  private static boolean token(String value) {
+    return value != null && value.matches("[0-9a-f]{64}");
+  }
+
+  static IllegalStateException malformed() {
+    return new IllegalStateException("malformed reservation result");
+  }
+
+  static IllegalStateException malformed(Throwable cause) {
+    return new IllegalStateException("malformed reservation result", cause);
+  }
+}


## `feat(reservation): decode reservation decisions`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationWireMapper.java b/src/main/java/com/sportsbook/risk/reservation/ReservationWireMapper.java
new file mode 100644
index 0000000..bd2a13f
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationWireMapper.java
@@ -0,0 +1,58 @@
+package com.sportsbook.risk.reservation;
+
+import com.fasterxml.jackson.core.type.TypeReference;
+import com.fasterxml.jackson.databind.DeserializationFeature;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.databind.ObjectReader;
+import com.sportsbook.risk.pattern.PatternMatch;
+import java.util.List;
+import java.util.Objects;
+import org.springframework.stereotype.Component;
+
+/** Strictly validates deterministic admission and replay results from Redis. */
+@Component
+public final class ReservationWireMapper {
+  private final ObjectReader reader;
+  private final ObjectMapper mapper;
+
+  public ReservationWireMapper(ObjectMapper mapper) {
+    Objects.requireNonNull(mapper, "mapper");
+    this.mapper = mapper.copy().enable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
+    reader = this.mapper.readerFor(ReservationWire.class);
+  }
+
+  public Decoded map(String raw) {
+    ReservationWire wire = read(raw);
+    if (!"1".equals(wire.version()) || wire.status() == null || wire.replayed() == null) {
+      throw ReservationWireValidator.malformed();
+    }
+    long expired = ReservationWireValidator.exact(wire.expired(), "expired");
+    List<PatternMatch> patterns = patterns(wire.patternsJson());
+    ReservationDecision decision = ReservationWireValidator.decision(wire, patterns);
+    return new Decoded(decision, expired);
+  }
+
+  private ReservationWire read(String raw) {
+    if (raw == null) {
+      throw ReservationWireValidator.malformed();
+    }
+    try {
+      return reader.readValue(raw);
+    } catch (Exception failure) {
+      throw ReservationWireValidator.malformed(failure);
+    }
+  }
+
+  private List<PatternMatch> patterns(String raw) {
+    if (raw == null) {
+      return List.of();
+    }
+    try {
+      return List.copyOf(mapper.readValue(raw, new TypeReference<List<PatternMatch>>() {}));
+    } catch (Exception failure) {
+      throw ReservationWireValidator.malformed(failure);
+    }
+  }
+
+  public record Decoded(ReservationDecision decision, long expired) {}
+}


## `feat(reservation): replay reservation identities`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index f3ca683..ae6a28d 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -41,7 +41,33 @@ local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
   or typeError(KEYS[5], "zset") or typeError(KEYS[6], "string")
   or typeError(KEYS[7], "hash") or typeError(KEYS[18], "string")
 if errorText then return redis.error_reply(errorText) end
-if redis.call("EXISTS", KEYS[1]) == 1 then return redis.error_reply("reservation already exists") end
+local existing = redis.call("HGET", KEYS[1], "state")
+if existing then
+  if redis.call("HGET", KEYS[1], "fingerprint") ~= fingerprint then
+    return response({status = "CONFLICT", replayed = false})
+  end
+  local patternsJson = redis.call("HGET", KEYS[1], "patternsJson") or "[]"
+  local decoded, patterns = pcall(cjson.decode, patternsJson)
+  if not decoded or type(patterns) ~= "table" or string.sub(patternsJson, 1, 1) ~= "[" then
+    return redis.error_reply("corrupt reservation patterns")
+  end
+  if existing == "RESERVED" or existing == "COMMITTED" then
+    return response({status = "APPROVED", state = existing,
+      expiresAt = redis.call("HGET", KEYS[1], "expiresAt"), token = fingerprint,
+      replayed = true, patternsJson = patternsJson})
+  end
+  if existing == "REJECTED" then
+    local rejection = redis.call("HGET", KEYS[1], "rejection")
+    if not rejection then return redis.error_reply("corrupt reservation rejection") end
+    return response({status = "REJECTED", rejection = rejection,
+      replayed = true, patternsJson = patternsJson})
+  end
+  if existing == "EXPIRED" or existing == "RELEASED" then
+    return response({status = "REJECTED", rejection = "RISK_RESERVATION_" .. existing,
+      replayed = true, patternsJson = patternsJson})
+  end
+  return redis.error_reply("unknown reservation state")
+end
 
 local singleRaw = redis.call("HGET", KEYS[7], "SINGLE_BET_MAX:" .. currency) or ARGV[11]
 local singleLimit = exact(singleRaw, false)


## `test(reservation): verify reservation identity replay`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationReplayScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationReplayScriptTest.java
new file mode 100644
index 0000000..1ecbce4
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationReplayScriptTest.java
@@ -0,0 +1,42 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import org.junit.jupiter.api.Test;
+
+class ReservationReplayScriptTest extends ReservationScriptTestSupport {
+  @Test
+  void replaysTheSameReservedIdentityAndRejectsChangedPayloads() {
+    RiskCheckCommand original = command(20, 100, Currency.KRW);
+
+    ReservationDecision first = reserve(original);
+    ReservationDecision replay = reserve(original);
+    ReservationDecision conflict = reserve(command(20, 101, Currency.KRW));
+
+    assertThat(first.replayed()).isFalse();
+    assertThat(replay.replayed()).isTrue();
+    assertThat(replay.token()).isEqualTo(first.token());
+    assertThat(replay.state()).isEqualTo(ReservationState.RESERVED);
+    assertThat(conflict.status()).isEqualTo(ReservationDecision.Status.CONFLICT);
+    assertThat(redis.opsForValue().get(ReservationKeys.activeStakes(USER, Currency.KRW).sum()))
+        .isEqualTo("100");
+  }
+
+  @Test
+  void replaysStoredBusinessRejections() {
+    RiskCheckCommand command = command(21, 60, Currency.KRW);
+    ReservationScriptRequest request =
+        request(command, limits(50), new RiskPatternProperties(null, null, null));
+
+    ReservationDecision first = execute(request).decision();
+    ReservationDecision replay = execute(request).decision();
+
+    assertThat(first.replayed()).isFalse();
+    assertThat(replay.replayed()).isTrue();
+    assertThat(replay.rejection()).isEqualTo("SINGLE_BET_MAX_EXCEEDED");
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+  }
+}


## `feat(events): bind accepted identities to admission`

diff --git a/src/main/resources/scripts/risk-reserve.lua b/src/main/resources/scripts/risk-reserve.lua
index 22ab10f..77b5271 100644
--- a/src/main/resources/scripts/risk-reserve.lua
+++ b/src/main/resources/scripts/risk-reserve.lua
@@ -65,11 +65,16 @@ for index = 1, selectionCount do
   end
   seen[selectionId] = true; table.insert(selections, selectionId)
 end
+local acceptedKey = "risk:event:fingerprint:" .. betId
 local errorText = typeError(KEYS[1], "hash") or typeError(KEYS[2], "zset")
   or typeError(KEYS[3], "zset") or typeError(KEYS[4], "string")
   or typeError(KEYS[5], "zset") or typeError(KEYS[6], "string")
   or typeError(KEYS[7], "hash") or typeError(KEYS[18], "string")
+  or typeError(acceptedKey, "string")
 if errorText then return redis.error_reply(errorText) end
+if redis.call("EXISTS", acceptedKey) == 1 then
+  return response({status = "CONFLICT", replayed = false})
+end
 
 local activeBase, cleanups, stakeDecrements =
   "risk:reservations:user:{" .. userId .. "}", {}, {}


## `test(events): prevent accepted identity readmission`

diff --git a/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java b/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java
new file mode 100644
index 0000000..2474f7d
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/AcceptedReservationBoundaryScriptTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.risk.reservation;
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
+import com.sportsbook.risk.counter.RedisLuaScriptLoader;
+import com.sportsbook.risk.pattern.RiskHistoryProperties;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.RiskPatternProperties;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.script.RedisScript;
+
+class AcceptedReservationBoundaryScriptTest extends RedisTestSupport {
+  private static final RedisScript<String> PROJECT =
+      RedisLuaScriptLoader.stringScript("risk-project-accepted.lua");
+  private static final RedisScript<String> RESERVE =
+      RedisLuaScriptLoader.stringScript("risk-reserve.lua");
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 10));
+  private static final RiskCheckCommand COMMAND =
+      new RiskCheckCommand(
+          USER,
+          BET,
+          Money.krw(50),
+          List.of(SelectionId.of(new UUID(0, 20))),
+          Instant.ofEpochMilli(2_000_000));
+
+  @Test
+  void acceptedProjectionPreventsReservationReadmission() {
+    var reservations = new RiskReservationProperties(null, null);
+    var patterns = new RiskPatternProperties(null, null, null);
+    var history = new RiskHistoryProperties(null, 0);
+    String fingerprint = ReservationFingerprint.of(COMMAND);
+    AcceptedProjectionRequest projection =
+        AcceptedProjectionRequest.from(COMMAND, fingerprint, reservations, patterns, history);
+    ReservationScriptRequest admission =
+        ReservationScriptRequest.from(
+            COMMAND,
+            new RiskLimitProperties(null, null, null, null, 0),
+            patterns,
+            reservations,
+            history);
+
+    assertThat(redis.execute(PROJECT, projection.keys(), projection.arguments().toArray()))
+        .isEqualTo("APPLIED");
+    ReservationDecision decision =
+        new ReservationWireMapper(new ObjectMapper())
+            .map(redis.execute(RESERVE, admission.keys(), admission.arguments().toArray()))
+            .decision();
+
+    assertThat(decision.status()).isEqualTo(ReservationDecision.Status.CONFLICT);
+    assertThat(redis.hasKey(ReservationKeys.lifecycle(BET))).isFalse();
+    assertThat(redis.hasKey(ReservationKeys.activeBets(USER))).isFalse();
+    assertThat(
+            redis
+                .opsForValue()
+                .get(LimitKeys.monetary(USER, LimitType.STAKE_DAILY, Currency.KRW).sum()))
+        .isEqualTo("50");
+  }
+}
