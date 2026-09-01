# 베팅 슬립 구조와 정산 불변식

## `feat(bet): define market and settlement states`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetStatus.java b/src/main/java/com/sportsbook/protocol/domain/BetStatus.java
new file mode 100644
index 0000000..1a9b439
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetStatus.java
@@ -0,0 +1,15 @@
+package com.sportsbook.protocol.domain;
+
+/**
+ * 베팅 조합의 상태입니다(ADR-0013). {@code PENDING}은 입력, 지갑, 위험 한도 등 수락 전 검증을 나타냅니다. {@code REJECTED}는 수락 전
+ * 거절, {@code CANCELLED}는 수락 뒤 사용자 또는 운영자의 취소, {@code VOIDED}는 경기 취소에 따른 자동 환불입니다(ADR-0012). 선택지별 정산
+ * 결과인 {@code SettlementResult.VOID}는 이 상태를 바꾸지 않습니다.
+ */
+public enum BetStatus {
+  PENDING,
+  ACCEPTED,
+  REJECTED,
+  SETTLED,
+  CANCELLED,
+  VOIDED,
+}
diff --git a/src/main/java/com/sportsbook/protocol/domain/MarketType.java b/src/main/java/com/sportsbook/protocol/domain/MarketType.java
new file mode 100644
index 0000000..1ea3c05
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/MarketType.java
@@ -0,0 +1,13 @@
+package com.sportsbook.protocol.domain;
+
+/**
+ * V1 market shapes (ADR-0013). Asian handicap is excluded because ADR-0012 rules out
+ * half-won/half-lost settlement results. Adding a market requires (a) extending this enum, (b)
+ * updating settlement pricing, and (c) adding an OddsProvider mapping in odds-feed-service.
+ */
+public enum MarketType {
+  MATCH_RESULT_1X2,
+  TOTAL_OVER_UNDER,
+  BOTH_TEAMS_TO_SCORE,
+  DOUBLE_CHANCE,
+}
diff --git a/src/main/java/com/sportsbook/protocol/domain/SettlementResult.java b/src/main/java/com/sportsbook/protocol/domain/SettlementResult.java
new file mode 100644
index 0000000..8b3c03d
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/SettlementResult.java
@@ -0,0 +1,15 @@
+package com.sportsbook.protocol.domain;
+
+/**
+ * Settle-time outcome (ADR-0013). {@code HALF_WON} / {@code HALF_LOST} are deferred until Asian
+ * handicap support (ADR-0012). {@code CASHED_OUT} is deferred until cash-out feature. {@code PUSH}
+ * is the stake-refund case (e.g., O/U line exactly hit). {@code VOID} here means per-selection void
+ * at settle time, distinct from {@link BetStatus#VOIDED} which is a whole-slip refund on event
+ * cancellation.
+ */
+public enum SettlementResult {
+  WON,
+  LOST,
+  PUSH,
+  VOID,
+}


## `test(bet): verify state semantics`

diff --git a/src/test/java/com/sportsbook/protocol/domain/DomainEnumsTest.java b/src/test/java/com/sportsbook/protocol/domain/DomainEnumsTest.java
new file mode 100644
index 0000000..42f90f4
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/domain/DomainEnumsTest.java
@@ -0,0 +1,85 @@
+package com.sportsbook.protocol.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import org.junit.jupiter.api.Test;
+
+/**
+ * Pins ADR-0013 value sets. Any addition/removal forces a test update and surfaces the change in
+ * code review — preventing silent vocabulary drift across services.
+ */
+class DomainEnumsTest {
+
+  private final ObjectMapper mapper = new ObjectMapper();
+
+  @Test
+  void marketTypeV1SetIsExactlyFourMarkets() {
+    assertThat(MarketType.values())
+        .containsExactlyInAnyOrder(
+            MarketType.MATCH_RESULT_1X2,
+            MarketType.TOTAL_OVER_UNDER,
+            MarketType.BOTH_TEAMS_TO_SCORE,
+            MarketType.DOUBLE_CHANCE);
+  }
+
+  @Test
+  void betStatusV1SetCoversFullLifecycle() {
+    assertThat(BetStatus.values())
+        .containsExactlyInAnyOrder(
+            BetStatus.PENDING,
+            BetStatus.ACCEPTED,
+            BetStatus.REJECTED,
+            BetStatus.SETTLED,
+            BetStatus.CANCELLED,
+            BetStatus.VOIDED);
+  }
+
+  @Test
+  void settlementResultV1SetExcludesHalfWonAndCashedOut() {
+    assertThat(SettlementResult.values())
+        .containsExactlyInAnyOrder(
+            SettlementResult.WON,
+            SettlementResult.LOST,
+            SettlementResult.PUSH,
+            SettlementResult.VOID);
+  }
+
+  @Test
+  void marketTypeValueOfRoundTrips() {
+    assertThat(MarketType.valueOf("MATCH_RESULT_1X2")).isEqualTo(MarketType.MATCH_RESULT_1X2);
+    assertThat(MarketType.valueOf("DOUBLE_CHANCE")).isEqualTo(MarketType.DOUBLE_CHANCE);
+  }
+
+  @Test
+  void betStatusValueOfRoundTrips() {
+    assertThat(BetStatus.valueOf("PENDING")).isEqualTo(BetStatus.PENDING);
+    assertThat(BetStatus.valueOf("VOIDED")).isEqualTo(BetStatus.VOIDED);
+  }
+
+  @Test
+  void settlementResultValueOfRoundTrips() {
+    assertThat(SettlementResult.valueOf("WON")).isEqualTo(SettlementResult.WON);
+    assertThat(SettlementResult.valueOf("PUSH")).isEqualTo(SettlementResult.PUSH);
+  }
+
+  @Test
+  void marketTypeJsonSerializesAsEnumName() throws Exception {
+    assertThat(mapper.writeValueAsString(MarketType.MATCH_RESULT_1X2))
+        .isEqualTo("\"MATCH_RESULT_1X2\"");
+    assertThat(mapper.readValue("\"DOUBLE_CHANCE\"", MarketType.class))
+        .isEqualTo(MarketType.DOUBLE_CHANCE);
+  }
+
+  @Test
+  void betStatusJsonRoundTrips() throws Exception {
+    String json = mapper.writeValueAsString(BetStatus.ACCEPTED);
+    assertThat(mapper.readValue(json, BetStatus.class)).isEqualTo(BetStatus.ACCEPTED);
+  }
+
+  @Test
+  void settlementResultJsonRoundTrips() throws Exception {
+    String json = mapper.writeValueAsString(SettlementResult.VOID);
+    assertThat(mapper.readValue(json, SettlementResult.class)).isEqualTo(SettlementResult.VOID);
+  }
+}


## `feat(bet): classify bet slips`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java b/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java
new file mode 100644
index 0000000..ba40754
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetSlipType.java
@@ -0,0 +1,60 @@
+package com.sportsbook.protocol.domain;
+
+import com.fasterxml.jackson.annotation.JsonSubTypes;
+import com.fasterxml.jackson.annotation.JsonTypeInfo;
+
+/**
+ * Bet slip shape per ADR-0008: Single / Multiple / System(K-of-N). System is generalized (minWins,
+ * totalSelections) rather than enumerating named variants — Trixie / Yankee / Lucky 15 etc. stay as
+ * frontend labels so adding a new K-of-N flavor needs zero protocol change.
+ *
+ * <p>Jackson polymorphism via {@code @JsonTypeInfo} + {@code @JsonSubTypes} produces a tagged shape
+ * on the wire: {@code {"type":"SINGLE"}}, {@code {"type":"MULTIPLE"}}, {@code
+ * {"type":"SYSTEM","minWins":2,"totalSelections":3}}.
+ */
+@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
+@JsonSubTypes({
+  @JsonSubTypes.Type(value = BetSlipType.Single.class, name = "SINGLE"),
+  @JsonSubTypes.Type(value = BetSlipType.Multiple.class, name = "MULTIPLE"),
+  @JsonSubTypes.Type(value = BetSlipType.System.class, name = "SYSTEM"),
+})
+public sealed interface BetSlipType
+    permits BetSlipType.Single, BetSlipType.Multiple, BetSlipType.System {
+
+  /** 1 selection per slip. */
+  record Single() implements BetSlipType {}
+
+  /** All N selections must win (parlay / accumulator). */
+  record Multiple() implements BetSlipType {}
+
+  /**
+   * K-of-N partial-win slip. {@link #MAX_TOTAL_SELECTIONS} bounds the structurally possible shapes
+   * (also keeps the C(total, min) combination explosion tractable). The runtime policy bound
+   * ({@code betting.policy.max-selections} in application.yml) may be tighter.
+   */
+  record System(int minWins, int totalSelections) implements BetSlipType {
+
+    public static final int MIN_TOTAL_SELECTIONS = 2;
+    public static final int MAX_TOTAL_SELECTIONS = 15;
+
+    public System {
+      if (totalSelections < MIN_TOTAL_SELECTIONS || totalSelections > MAX_TOTAL_SELECTIONS) {
+        throw new IllegalArgumentException(
+            "System.totalSelections ("
+                + totalSelections
+                + ") must be in "
+                + MIN_TOTAL_SELECTIONS
+                + ".."
+                + MAX_TOTAL_SELECTIONS);
+      }
+      if (minWins < 1 || minWins > totalSelections) {
+        throw new IllegalArgumentException(
+            "System.minWins ("
+                + minWins
+                + ") must be in 1..totalSelections ("
+                + totalSelections
+                + ")");
+      }
+    }
+  }
+}


## `feat(bet): define bet selections`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetSelection.java b/src/main/java/com/sportsbook/protocol/domain/BetSelection.java
new file mode 100644
index 0000000..58f54a3
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetSelection.java
@@ -0,0 +1,29 @@
+package com.sportsbook.protocol.domain;
+
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.util.Objects;
+
+/**
+ * One pick on a bet slip. {@code oddsAtPlacement} is the price the user saw at submit;
+ * betting-service compares it against current odds with the slippage tolerance (ADR-0008, 3%).
+ * {@code marketType} is denormalized onto the selection so settlement can resolve the right pricing
+ * logic without joining back to a Market entity.
+ */
+public record BetSelection(
+    EventId eventId,
+    MarketId marketId,
+    MarketType marketType,
+    SelectionId selectionId,
+    Odds oddsAtPlacement) {
+
+  public BetSelection {
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(marketId, "marketId");
+    Objects.requireNonNull(marketType, "marketType");
+    Objects.requireNonNull(selectionId, "selectionId");
+    Objects.requireNonNull(oddsAtPlacement, "oddsAtPlacement");
+  }
+}


## `test(bet): verify selection invariants`

diff --git a/src/test/java/com/sportsbook/protocol/domain/BetSelectionTest.java b/src/test/java/com/sportsbook/protocol/domain/BetSelectionTest.java
new file mode 100644
index 0000000..de8115a
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/domain/BetSelectionTest.java
@@ -0,0 +1,42 @@
+package com.sportsbook.protocol.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetSelectionTest {
+
+  private final EventId eventId = EventId.of(UUID.randomUUID());
+  private final MarketId marketId = MarketId.of(UUID.randomUUID());
+  private final SelectionId selectionId = SelectionId.of(UUID.randomUUID());
+  private final Odds odds = Odds.ofDecimal("1.85");
+
+  @Test
+  void selectionPreservesPlacementSnapshot() {
+    BetSelection selection =
+        new BetSelection(eventId, marketId, MarketType.MATCH_RESULT_1X2, selectionId, odds);
+    assertThat(selection.eventId()).isEqualTo(eventId);
+    assertThat(selection.marketId()).isEqualTo(marketId);
+    assertThat(selection.marketType()).isEqualTo(MarketType.MATCH_RESULT_1X2);
+    assertThat(selection.selectionId()).isEqualTo(selectionId);
+    assertThat(selection.oddsAtPlacement()).isEqualTo(odds);
+  }
+
+  @Test
+  void requiredFieldsRejectNulls() {
+    assertThatNullPointerException()
+        .isThrownBy(
+            () -> new BetSelection(null, marketId, MarketType.MATCH_RESULT_1X2, selectionId, odds));
+    assertThatNullPointerException()
+        .isThrownBy(
+            () ->
+                new BetSelection(
+                    eventId, marketId, MarketType.MATCH_RESULT_1X2, selectionId, null));
+  }
+}


## `feat(bet): compose self-consistent slips`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetSlip.java b/src/main/java/com/sportsbook/protocol/domain/BetSlip.java
new file mode 100644
index 0000000..d0fa921
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetSlip.java
@@ -0,0 +1,104 @@
+package com.sportsbook.protocol.domain;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.UserId;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+
+/**
+ * shared-protocol holds only structural invariants — "is this BetSlip self-consistent as a data
+ * shape?". Domain validation (ADR-0008 L1 Same Market, L2 Same Event, L4/L5 policy bounds, odds
+ * slippage tolerance) lives in betting-service's BetSlipValidator. The invariants here prevent the
+ * wire from carrying nonsensical shapes (Single with 3 selections, SETTLED without a result, etc.)
+ * which protect every consumer downstream.
+ */
+public record BetSlip(
+    BetId id,
+    UserId userId,
+    BetSlipType type,
+    List<BetSelection> selections,
+    Money stake,
+    BetStatus status,
+    Instant placedAt,
+    SettlementResult settlementResult,
+    Instant settledAt,
+    Money payout) {
+
+  public static final int MULTIPLE_MIN_SELECTIONS = 2;
+
+  public BetSlip {
+    Objects.requireNonNull(id, "id");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(selections, "selections");
+    Objects.requireNonNull(stake, "stake");
+    Objects.requireNonNull(status, "status");
+    Objects.requireNonNull(placedAt, "placedAt");
+
+    if (selections.isEmpty()) {
+      throw new IllegalArgumentException("BetSlip must have at least one selection");
+    }
+    if (!stake.isPositive()) {
+      throw new IllegalArgumentException("BetSlip stake must be positive (got " + stake + ")");
+    }
+
+    if (type instanceof BetSlipType.Single && selections.size() != 1) {
+      throw new IllegalArgumentException(
+          "Single slip must have exactly 1 selection (got " + selections.size() + ")");
+    }
+    if (type instanceof BetSlipType.Multiple && selections.size() < MULTIPLE_MIN_SELECTIONS) {
+      throw new IllegalArgumentException(
+          "Multiple slip must have at least "
+              + MULTIPLE_MIN_SELECTIONS
+              + " selections (got "
+              + selections.size()
+              + ")");
+    }
+    if (type instanceof BetSlipType.System sys && selections.size() != sys.totalSelections()) {
+      throw new IllegalArgumentException(
+          "System slip selections.size ("
+              + selections.size()
+              + ") must equal type.totalSelections ("
+              + sys.totalSelections()
+              + ")");
+    }
+
+    if (status == BetStatus.SETTLED) {
+      if (settlementResult == null) {
+        throw new IllegalArgumentException("SETTLED slip must have settlementResult");
+      }
+      if (settledAt == null) {
+        throw new IllegalArgumentException("SETTLED slip must have settledAt");
+      }
+    } else {
+      if (settlementResult != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have settlementResult (status=" + status + ")");
+      }
+      if (settledAt != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have settledAt (status=" + status + ")");
+      }
+      if (payout != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have payout (status=" + status + ")");
+      }
+    }
+
+    if (settlementResult == SettlementResult.WON
+        || settlementResult == SettlementResult.PUSH
+        || settlementResult == SettlementResult.VOID) {
+      if (payout == null) {
+        throw new IllegalArgumentException(
+            settlementResult + " slip must have payout (winnings or refund)");
+      }
+    } else if (settlementResult == SettlementResult.LOST && payout != null) {
+      throw new IllegalArgumentException("LOST slip must not have payout");
+    }
+
+    // Defensive copy: prevent external mutation of the selections list after construction.
+    selections = List.copyOf(selections);
+  }
+}


## `test(bet): verify structural slip invariants`

diff --git a/src/test/java/com/sportsbook/protocol/domain/BetSlipStructureTest.java b/src/test/java/com/sportsbook/protocol/domain/BetSlipStructureTest.java
new file mode 100644
index 0000000..3fa9a12
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/domain/BetSlipStructureTest.java
@@ -0,0 +1,78 @@
+package com.sportsbook.protocol.domain;
+
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetSlipStructureTest {
+
+  @Test
+  void slipTypesAcceptTheirRequiredSelectionCounts() {
+    pending(new BetSlipType.Single(), List.of(selection()));
+    pending(new BetSlipType.Multiple(), List.of(selection(), selection()));
+    pending(new BetSlipType.System(2, 3), List.of(selection(), selection(), selection()));
+  }
+
+  @Test
+  void emptyAndMismatchedSelectionsAreRejected() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> pending(new BetSlipType.Single(), List.of()));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> pending(new BetSlipType.Single(), List.of(selection(), selection())));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> pending(new BetSlipType.Multiple(), List.of(selection())));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> pending(new BetSlipType.System(2, 3), List.of(selection(), selection())));
+  }
+
+  @Test
+  void stakeMustBePositive() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () ->
+                new BetSlip(
+                    BetId.of(UUID.randomUUID()),
+                    UserId.of(UUID.randomUUID()),
+                    new BetSlipType.Single(),
+                    List.of(selection()),
+                    Money.krw(0),
+                    BetStatus.PENDING,
+                    Instant.EPOCH,
+                    null,
+                    null,
+                    null));
+  }
+
+  private BetSlip pending(BetSlipType type, List<BetSelection> selections) {
+    return new BetSlip(
+        BetId.of(UUID.randomUUID()),
+        UserId.of(UUID.randomUUID()),
+        type,
+        selections,
+        Money.krw(10_000),
+        BetStatus.PENDING,
+        Instant.EPOCH,
+        null,
+        null,
+        null);
+  }
+
+  private BetSelection selection() {
+    return new BetSelection(
+        EventId.of(UUID.randomUUID()),
+        MarketId.of(UUID.randomUUID()),
+        MarketType.MATCH_RESULT_1X2,
+        SelectionId.of(UUID.randomUUID()),
+        Odds.ofDecimal("1.85"));
+  }
+}


## `test(bet): verify settlement slip invariants`

diff --git a/src/test/java/com/sportsbook/protocol/domain/BetSlipSettlementTest.java b/src/test/java/com/sportsbook/protocol/domain/BetSlipSettlementTest.java
new file mode 100644
index 0000000..5893176
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/domain/BetSlipSettlementTest.java
@@ -0,0 +1,75 @@
+package com.sportsbook.protocol.domain;
+
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetSlipSettlementTest {
+
+  @Test
+  void wonPushAndVoidRequirePayoutSnapshots() {
+    settled(SettlementResult.WON, Money.krw(18_500));
+    settled(SettlementResult.PUSH, Money.krw(10_000));
+    settled(SettlementResult.VOID, Money.krw(10_000));
+    assertThatIllegalArgumentException().isThrownBy(() -> settled(SettlementResult.WON, null));
+  }
+
+  @Test
+  void lostOutcomeMustNotCarryPayout() {
+    settled(SettlementResult.LOST, null);
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> settled(SettlementResult.LOST, Money.krw(0)));
+  }
+
+  @Test
+  void settledSlipRequiresResultAndTimestamp() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> create(BetStatus.SETTLED, null, Instant.EPOCH, null));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> create(BetStatus.SETTLED, SettlementResult.LOST, null, null));
+  }
+
+  @Test
+  void activeSlipCannotCarrySettlementFields() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () ->
+                create(BetStatus.PENDING, SettlementResult.WON, Instant.EPOCH, Money.krw(18_500)));
+  }
+
+  private BetSlip settled(SettlementResult result, Money payout) {
+    return create(BetStatus.SETTLED, result, Instant.EPOCH, payout);
+  }
+
+  private BetSlip create(
+      BetStatus status, SettlementResult result, Instant settledAt, Money payout) {
+    BetSelection selection =
+        new BetSelection(
+            EventId.of(UUID.randomUUID()),
+            MarketId.of(UUID.randomUUID()),
+            MarketType.MATCH_RESULT_1X2,
+            SelectionId.of(UUID.randomUUID()),
+            Odds.ofDecimal("1.85"));
+    return new BetSlip(
+        BetId.of(UUID.randomUUID()),
+        UserId.of(UUID.randomUUID()),
+        new BetSlipType.Single(),
+        List.of(selection),
+        Money.krw(10_000),
+        status,
+        Instant.EPOCH,
+        result,
+        settledAt,
+        payout);
+  }
+}


