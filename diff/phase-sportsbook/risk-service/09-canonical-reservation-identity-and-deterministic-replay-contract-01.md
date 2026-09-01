# 정규화 예약 식별성과 결정적 재생 계약

## `feat(reservation): define reservation states`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationState.java b/src/main/java/com/sportsbook/risk/reservation/ReservationState.java
new file mode 100644
index 0000000..a821ade
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationState.java
@@ -0,0 +1,7 @@
+package com.sportsbook.risk.reservation;
+
+/** Nonterminal and committed reservation states exposed to authenticated callers. */
+public enum ReservationState {
+  RESERVED,
+  COMMITTED
+}
diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationTransition.java b/src/main/java/com/sportsbook/risk/reservation/ReservationTransition.java
new file mode 100644
index 0000000..78e0be3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationTransition.java
@@ -0,0 +1,27 @@
+package com.sportsbook.risk.reservation;
+
+/** Exact result of an idempotent commit or release state transition. */
+public enum ReservationTransition {
+  APPLIED(true, false),
+  REPLAYED(true, true),
+  NOT_FOUND(false, false),
+  EXPIRED(false, false),
+  TOMBSTONED(false, false),
+  CONFLICT(false, false);
+
+  private final boolean successful;
+  private final boolean replayed;
+
+  ReservationTransition(boolean successful, boolean replayed) {
+    this.successful = successful;
+    this.replayed = replayed;
+  }
+
+  public boolean successful() {
+    return successful;
+  }
+
+  public boolean replayed() {
+    return replayed;
+  }
+}


## `test(reservation): verify reservation transition semantics`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationTransitionTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationTransitionTest.java
new file mode 100644
index 0000000..f0e7257
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationTransitionTest.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class ReservationTransitionTest {
+  @Test
+  void distinguishesAppliedAndReplaySuccess() {
+    assertThat(ReservationTransition.APPLIED.successful()).isTrue();
+    assertThat(ReservationTransition.APPLIED.replayed()).isFalse();
+    assertThat(ReservationTransition.REPLAYED.successful()).isTrue();
+    assertThat(ReservationTransition.REPLAYED.replayed()).isTrue();
+  }
+
+  @Test
+  void keepsEveryTerminalOrConflictResultUnsuccessful() {
+    assertThat(
+            java.util.List.of(
+                ReservationTransition.NOT_FOUND,
+                ReservationTransition.EXPIRED,
+                ReservationTransition.TOMBSTONED,
+                ReservationTransition.CONFLICT))
+        .allSatisfy(transition -> assertThat(transition.successful()).isFalse());
+    assertThat(ReservationState.values())
+        .containsExactly(ReservationState.RESERVED, ReservationState.COMMITTED);
+  }
+}


## `feat(reservation): define reservation decisions`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationDecision.java b/src/main/java/com/sportsbook/risk/reservation/ReservationDecision.java
new file mode 100644
index 0000000..95c48eb
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationDecision.java
@@ -0,0 +1,85 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.risk.pattern.PatternMatch;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+
+/** Complete atomic admission result retained for deterministic request replay. */
+public record ReservationDecision(
+    Status status,
+    ReservationState state,
+    Instant expiresAt,
+    String token,
+    String rejection,
+    boolean replayed,
+    List<PatternMatch> patterns) {
+  public ReservationDecision {
+    Objects.requireNonNull(status, "status");
+    Objects.requireNonNull(patterns, "patterns");
+    patterns = List.copyOf(patterns);
+    switch (status) {
+      case APPROVED -> {
+        Objects.requireNonNull(state, "state");
+        Objects.requireNonNull(expiresAt, "expiresAt");
+        requireText(token, "token");
+        if (rejection != null) {
+          throw new IllegalArgumentException("approved decision cannot contain a rejection");
+        }
+      }
+      case REJECTED -> {
+        requireAbsent(state, expiresAt, token);
+        requireText(rejection, "rejection");
+      }
+      case CONFLICT -> {
+        requireAbsent(state, expiresAt, token);
+        if (rejection != null || replayed || !patterns.isEmpty()) {
+          throw new IllegalArgumentException("conflict decision cannot contain outcome data");
+        }
+      }
+    }
+  }
+
+  public static ReservationDecision approved(
+      ReservationState state,
+      Instant expiresAt,
+      String token,
+      boolean replayed,
+      List<PatternMatch> patterns) {
+    return new ReservationDecision(
+        Status.APPROVED, state, expiresAt, token, null, replayed, patterns);
+  }
+
+  public static ReservationDecision rejected(
+      String rejection, boolean replayed, List<PatternMatch> patterns) {
+    return new ReservationDecision(
+        Status.REJECTED, null, null, null, rejection, replayed, patterns);
+  }
+
+  public static ReservationDecision conflict() {
+    return new ReservationDecision(Status.CONFLICT, null, null, null, null, false, List.of());
+  }
+
+  public boolean approved() {
+    return status == Status.APPROVED;
+  }
+
+  private static void requireAbsent(Object... values) {
+    if (java.util.Arrays.stream(values).anyMatch(Objects::nonNull)) {
+      throw new IllegalArgumentException("non-approved decision contains reservation state");
+    }
+  }
+
+  private static String requireText(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
+  }
+
+  public enum Status {
+    APPROVED,
+    REJECTED,
+    CONFLICT
+  }
+}


## `test(reservation): verify reservation decision invariants`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationDecisionTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationDecisionTest.java
new file mode 100644
index 0000000..269eb12
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationDecisionTest.java
@@ -0,0 +1,58 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.policy.PatternAction;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+class ReservationDecisionTest {
+  private static final PatternMatch FLAG =
+      new PatternMatch("RAPID_BETTING", PatternAction.REVIEW, "matched");
+
+  @Test
+  void representsApprovedAndRejectedReplays() {
+    List<PatternMatch> flags = new ArrayList<>(List.of(FLAG));
+    ReservationDecision approved =
+        ReservationDecision.approved(
+            ReservationState.RESERVED, Instant.EPOCH.plusSeconds(60), "opaque-token", true, flags);
+    flags.clear();
+
+    assertThat(approved.approved()).isTrue();
+    assertThat(approved.token()).isEqualTo("opaque-token");
+    assertThat(approved.patterns()).containsExactly(FLAG);
+    assertThat(approved.replayed()).isTrue();
+    assertThat(ReservationDecision.rejected("STAKE_DAILY", true, List.of()).status())
+        .isEqualTo(ReservationDecision.Status.REJECTED);
+  }
+
+  @Test
+  void rejectsInconsistentOutcomeShapes() {
+    assertThatThrownBy(
+            () ->
+                new ReservationDecision(
+                    ReservationDecision.Status.APPROVED,
+                    ReservationState.RESERVED,
+                    Instant.EPOCH,
+                    "token",
+                    "unexpected",
+                    false,
+                    List.of()))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                new ReservationDecision(
+                    ReservationDecision.Status.REJECTED, null, null, null, " ", false, List.of()))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () ->
+                new ReservationDecision(
+                    ReservationDecision.Status.CONFLICT, null, null, null, null, true, List.of()))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThat(ReservationDecision.conflict().approved()).isFalse();
+  }
+}


## `feat(reservation): fingerprint reservation requests`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationFingerprint.java b/src/main/java/com/sportsbook/risk/reservation/ReservationFingerprint.java
new file mode 100644
index 0000000..e5008f3
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationFingerprint.java
@@ -0,0 +1,45 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Objects;
+
+/** Versioned canonical request binding exposed as an opaque reservation token. */
+public final class ReservationFingerprint {
+  private static final String VERSION = "risk-reservation-v1";
+
+  private ReservationFingerprint() {}
+
+  public static String of(RiskCheckCommand command) {
+    Objects.requireNonNull(command, "command");
+    MessageDigest digest = sha256();
+    add(digest, VERSION);
+    add(digest, command.userId().value().toString());
+    add(digest, command.betId().value().toString());
+    add(digest, Long.toString(command.stake().amount()));
+    add(digest, command.stake().currency().name());
+    command.selectionIds().stream()
+        .map(selection -> selection.value().toString())
+        .sorted()
+        .forEach(value -> add(digest, value));
+    return HexFormat.of().formatHex(digest.digest());
+  }
+
+  private static void add(MessageDigest digest, String value) {
+    byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
+    digest.update(ByteBuffer.allocate(Integer.BYTES).putInt(bytes.length).array());
+    digest.update(bytes);
+  }
+
+  private static MessageDigest sha256() {
+    try {
+      return MessageDigest.getInstance("SHA-256");
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("SHA-256 is unavailable", impossible);
+    }
+  }
+}


## `test(reservation): verify reservation fingerprints`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationFingerprintTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationFingerprintTest.java
new file mode 100644
index 0000000..043a956
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationFingerprintTest.java
@@ -0,0 +1,57 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationFingerprintTest {
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 2));
+  private static final SelectionId FIRST = SelectionId.of(new UUID(0, 3));
+  private static final SelectionId SECOND = SelectionId.of(new UUID(0, 4));
+
+  @Test
+  void ignoresSelectionOrderAndEvaluationTime() {
+    String token = fingerprint(USER, BET, Money.krw(100), List.of(FIRST, SECOND), Instant.EPOCH);
+
+    assertThat(token).hasSize(64).matches("[0-9a-f]{64}");
+    assertThat(
+            fingerprint(
+                USER, BET, Money.krw(100), List.of(SECOND, FIRST), Instant.EPOCH.plusSeconds(10)))
+        .isEqualTo(token);
+  }
+
+  @Test
+  void changesForEveryRequestIdentityDimension() {
+    String token = fingerprint(USER, BET, Money.krw(100), List.of(FIRST), Instant.EPOCH);
+
+    assertThat(
+            fingerprint(
+                UserId.of(new UUID(0, 9)), BET, Money.krw(100), List.of(FIRST), Instant.EPOCH))
+        .isNotEqualTo(token);
+    assertThat(
+            fingerprint(
+                USER, BetId.of(new UUID(0, 9)), Money.krw(100), List.of(FIRST), Instant.EPOCH))
+        .isNotEqualTo(token);
+    assertThat(fingerprint(USER, BET, Money.krw(101), List.of(FIRST), Instant.EPOCH))
+        .isNotEqualTo(token);
+    assertThat(fingerprint(USER, BET, Money.usd(100), List.of(FIRST), Instant.EPOCH))
+        .isNotEqualTo(token);
+    assertThat(fingerprint(USER, BET, Money.krw(100), List.of(SECOND), Instant.EPOCH))
+        .isNotEqualTo(token);
+  }
+
+  private static String fingerprint(
+      UserId userId, BetId betId, Money stake, List<SelectionId> selections, Instant evaluatedAt) {
+    return ReservationFingerprint.of(
+        new RiskCheckCommand(userId, betId, stake, selections, evaluatedAt));
+  }
+}


## `feat(reservation): namespace active reservation footprints`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationKeys.java b/src/main/java/com/sportsbook/risk/reservation/ReservationKeys.java
new file mode 100644
index 0000000..546138c
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationKeys.java
@@ -0,0 +1,57 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitKeys;
+import java.util.Objects;
+
+/** Redis keys for reservation lifecycle and every active capacity footprint. */
+public final class ReservationKeys {
+  public static final String ACTIVE_COUNT = "risk:reservations:active";
+  private static final String RESERVATION_PREFIX = "risk:reservation:";
+  private static final String USER_PREFIX = "risk:reservations:user:";
+  private static final String EVENT_PREFIX = "risk:event:fingerprint:";
+
+  private ReservationKeys() {}
+
+  public static String lifecycle(BetId betId) {
+    return RESERVATION_PREFIX + required(betId).value();
+  }
+
+  public static String activeBets(UserId userId) {
+    return userBase(userId) + ":bets";
+  }
+
+  public static LimitKeys.Keys activeStakes(UserId userId, Currency currency) {
+    String base =
+        userBase(userId)
+            + ":stakes:"
+            + Objects.requireNonNull(currency, "currency").name().toLowerCase();
+    return new LimitKeys.Keys(base + ":entries", base + ":sum");
+  }
+
+  public static LimitKeys.Keys activeSelections(UserId userId) {
+    String base = userBase(userId) + ":selections";
+    return new LimitKeys.Keys(base + ":entries", base + ":sum");
+  }
+
+  public static String activeSelection(UserId userId, SelectionId selectionId) {
+    Objects.requireNonNull(selectionId, "selectionId");
+    return userBase(userId) + ":selection:" + selectionId.value();
+  }
+
+  public static String acceptedFingerprint(BetId betId) {
+    return EVENT_PREFIX + required(betId).value();
+  }
+
+  private static String userBase(UserId userId) {
+    Objects.requireNonNull(userId, "userId");
+    return USER_PREFIX + "{" + userId.value() + "}";
+  }
+
+  private static BetId required(BetId betId) {
+    return Objects.requireNonNull(betId, "betId");
+  }
+}


## `test(reservation): verify reservation key boundaries`

diff --git a/src/test/java/com/sportsbook/risk/reservation/ReservationKeysTest.java b/src/test/java/com/sportsbook/risk/reservation/ReservationKeysTest.java
new file mode 100644
index 0000000..8c004a7
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/reservation/ReservationKeysTest.java
@@ -0,0 +1,34 @@
+package com.sportsbook.risk.reservation;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ReservationKeysTest {
+  private static final UserId USER = UserId.of(new UUID(0, 1));
+  private static final BetId BET = BetId.of(new UUID(0, 2));
+  private static final SelectionId SELECTION = SelectionId.of(new UUID(0, 3));
+
+  @Test
+  void separatesCurrencyStakeFromNeutralSelectionCapacity() {
+    assertThat(ReservationKeys.activeStakes(USER, Currency.KRW))
+        .isNotEqualTo(ReservationKeys.activeStakes(USER, Currency.USD));
+    assertThat(ReservationKeys.activeSelections(USER).entries()).doesNotContain("krw", "usd");
+    assertThat(ReservationKeys.activeSelection(USER, SELECTION)).doesNotContain("krw", "usd");
+    assertThat(ReservationKeys.activeBets(USER)).contains("{" + USER.value() + "}");
+  }
+
+  @Test
+  void keepsLifecycleAndIngestionIdentityUnambiguous() {
+    assertThat(ReservationKeys.lifecycle(BET)).endsWith(BET.value().toString());
+    assertThat(ReservationKeys.acceptedFingerprint(BET)).endsWith(BET.value().toString());
+    assertThat(ReservationKeys.lifecycle(BET))
+        .isNotEqualTo(ReservationKeys.acceptedFingerprint(BET));
+    assertThat(ReservationKeys.ACTIVE_COUNT).isEqualTo("risk:reservations:active");
+  }
+}


## `feat(reservation): assemble reserve script keys`

diff --git a/src/main/java/com/sportsbook/risk/reservation/ReservationScriptKeys.java b/src/main/java/com/sportsbook/risk/reservation/ReservationScriptKeys.java
new file mode 100644
index 0000000..4510036
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/reservation/ReservationScriptKeys.java
@@ -0,0 +1,46 @@
+package com.sportsbook.risk.reservation;
+
+import com.sportsbook.risk.counter.LimitKeys;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideKeys;
+import com.sportsbook.risk.pattern.HistoryKeys;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Objects;
+
+/** Canonical key order consumed by the reservation admission script. */
+final class ReservationScriptKeys {
+  private ReservationScriptKeys() {}
+
+  static List<String> from(RiskCheckCommand command) {
+    Objects.requireNonNull(command, "command");
+    var userId = command.userId();
+    var currency = command.stake().currency();
+    List<String> keys = new ArrayList<>();
+    keys.add(ReservationKeys.lifecycle(command.betId()));
+    keys.add(ReservationKeys.activeBets(userId));
+    add(keys, ReservationKeys.activeStakes(userId, currency));
+    add(keys, ReservationKeys.activeSelections(userId));
+    keys.add(LimitOverrideKeys.user(userId));
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_DAILY, currency));
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_WEEKLY, currency));
+    add(keys, LimitKeys.monetary(userId, LimitType.STAKE_MONTHLY, currency));
+    add(keys, LimitKeys.selections(userId));
+    keys.add(HistoryKeys.bets(userId));
+    keys.add(HistoryKeys.stakes(userId, currency));
+    keys.add(ReservationKeys.ACTIVE_COUNT);
+    command.selectionIds().stream()
+        .map(selection -> HistoryKeys.selection(userId, selection))
+        .forEach(keys::add);
+    command.selectionIds().stream()
+        .map(selection -> ReservationKeys.activeSelection(userId, selection))
+        .forEach(keys::add);
+    return List.copyOf(keys);
+  }
+
+  private static void add(List<String> keys, LimitKeys.Keys pair) {
+    keys.add(pair.entries());
+    keys.add(pair.sum());
+  }
+}


