# 수락 베팅 Avro 경계와 정확한 노출 계산

## `feat(events): bind risk topic contracts`

diff --git a/src/main/java/com/sportsbook/risk/event/EventTopics.java b/src/main/java/com/sportsbook/risk/event/EventTopics.java
new file mode 100644
index 0000000..58f4e77
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/EventTopics.java
@@ -0,0 +1,25 @@
+package com.sportsbook.risk.event;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+/** Kafka topic ownership for accepted bets, quarantine, and risk signals. */
+@ConfigurationProperties(prefix = "risk.topics")
+public record EventTopics(
+    String betPlaced, String betPlacedDlt, String limitViolated, String patternSuspected) {
+  public EventTopics {
+    betPlaced = valueOrDefault(betPlaced, "bet.placed.v1");
+    betPlacedDlt = valueOrDefault(betPlacedDlt, "bet.placed.v1.DLT");
+    limitViolated = valueOrDefault(limitViolated, "risk.limit.violated");
+    patternSuspected = valueOrDefault(patternSuspected, "risk.pattern.suspected");
+  }
+
+  private static String valueOrDefault(String value, String fallback) {
+    if (value == null) {
+      return fallback;
+    }
+    if (value.isBlank()) {
+      throw new IllegalArgumentException("topic names must not be blank");
+    }
+    return value;
+  }
+}


## `test(events): verify risk topic contracts`

diff --git a/src/test/java/com/sportsbook/risk/event/EventTopicsTest.java b/src/test/java/com/sportsbook/risk/event/EventTopicsTest.java
new file mode 100644
index 0000000..167b5d0
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/EventTopicsTest.java
@@ -0,0 +1,28 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class EventTopicsTest {
+  @Test
+  void suppliesTheExactOwnedTopicDefaults() {
+    EventTopics topics = new EventTopics(null, null, null, null);
+
+    assertThat(topics.betPlaced()).isEqualTo("bet.placed.v1");
+    assertThat(topics.betPlacedDlt()).isEqualTo("bet.placed.v1.DLT");
+    assertThat(topics.limitViolated()).isEqualTo("risk.limit.violated");
+    assertThat(topics.patternSuspected()).isEqualTo("risk.pattern.suspected");
+  }
+
+  @Test
+  void acceptsExplicitNamesButRejectsBlanks() {
+    EventTopics topics = new EventTopics("placed", "dlt", "limit", "pattern");
+
+    assertThat(topics.betPlaced()).isEqualTo("placed");
+    assertThat(topics.betPlacedDlt()).isEqualTo("dlt");
+    assertThatThrownBy(() -> new EventTopics(" ", "dlt", "limit", "pattern"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(events): encode shared Avro records`

diff --git a/src/main/java/com/sportsbook/risk/event/AvroCodec.java b/src/main/java/com/sportsbook/risk/event/AvroCodec.java
new file mode 100644
index 0000000..8524dc4
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AvroCodec.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.event;
+
+import java.io.ByteArrayOutputStream;
+import java.util.Objects;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecordBase;
+
+/** Plain Avro binary codec for the one-schema-per-topic wire contract. */
+public final class AvroCodec {
+  private AvroCodec() {}
+
+  public static <T extends SpecificRecordBase> byte[] encode(T record) {
+    Objects.requireNonNull(record, "record");
+    try (ByteArrayOutputStream output = new ByteArrayOutputStream()) {
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<T>(record.getSchema()).write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (Exception exception) {
+      throw new IllegalStateException(
+          "failed to encode " + record.getClass().getSimpleName(), exception);
+    }
+  }
+
+  public static <T extends SpecificRecordBase> T decode(byte[] payload, Class<T> type) {
+    Objects.requireNonNull(payload, "payload");
+    Objects.requireNonNull(type, "type");
+    try {
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(payload, null);
+      T value = new SpecificDatumReader<>(type).read(null, decoder);
+      if (!decoder.isEnd()) {
+        throw new IllegalArgumentException("Avro payload contains trailing bytes");
+      }
+      return value;
+    } catch (Exception exception) {
+      throw new IllegalStateException("failed to decode " + type.getSimpleName(), exception);
+    }
+  }
+}


## `test(events): verify Avro codec round trips`

diff --git a/src/test/java/com/sportsbook/risk/event/AvroCodecTest.java b/src/test/java/com/sportsbook/risk/event/AvroCodecTest.java
new file mode 100644
index 0000000..0d75987
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AvroCodecTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.protocol.event.RiskLimitType;
+import com.sportsbook.protocol.event.RiskLimitViolated;
+import com.sportsbook.protocol.event.RiskPatternSuspected;
+import com.sportsbook.protocol.event.RiskPatternType;
+import java.time.Instant;
+import java.util.Arrays;
+import java.util.List;
+import java.util.Map;
+import org.apache.avro.specific.SpecificRecordBase;
+import org.junit.jupiter.api.Test;
+
+class AvroCodecTest {
+  @Test
+  void roundTripsOwnedInputAndSignalRecords() {
+    BetPlacedRequested placed =
+        new BetPlacedRequested(
+            "bet",
+            "user",
+            BetSlipTypeTag.SINGLE,
+            null,
+            null,
+            List.of(new RequestedSelection("event", "market", "selection", "2.00")),
+            new Money(100L, "KRW"),
+            "request",
+            Instant.EPOCH);
+    RiskLimitViolated limit =
+        new RiskLimitViolated(
+            "user", RiskLimitType.STAKE_DAILY, 90L, 100L, new Money(20L, "KRW"), Instant.EPOCH);
+    RiskPatternSuspected pattern =
+        new RiskPatternSuspected(
+            "user", RiskPatternType.RAPID_BETTING, Map.of("action", "REVIEW"), Instant.EPOCH);
+
+    assertRoundTrip(placed, BetPlacedRequested.class);
+    assertRoundTrip(limit, RiskLimitViolated.class);
+    assertRoundTrip(pattern, RiskPatternSuspected.class);
+  }
+
+  @Test
+  void rejectsTrailingWireData() {
+    RiskPatternSuspected record =
+        new RiskPatternSuspected("user", RiskPatternType.RAPID_BETTING, Map.of(), Instant.EPOCH);
+    byte[] encoded = AvroCodec.encode(record);
+    byte[] trailing = Arrays.copyOf(encoded, encoded.length + 1);
+
+    assertThatThrownBy(() -> AvroCodec.decode(trailing, RiskPatternSuspected.class))
+        .isInstanceOf(IllegalStateException.class)
+        .hasRootCauseMessage("Avro payload contains trailing bytes");
+  }
+
+  private static <T extends SpecificRecordBase> void assertRoundTrip(T record, Class<T> type) {
+    assertThat(AvroCodec.decode(AvroCodec.encode(record), type)).isEqualTo(record);
+  }
+}


## `feat(events): calculate accepted bet exposure`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java
new file mode 100644
index 0000000..5bf6b90
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetExposure.java
@@ -0,0 +1,82 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.util.Objects;
+
+/** Exact monetary exposure represented by an accepted bet event. */
+public record AcceptedBetExposure(long totalAmount) {
+
+  public AcceptedBetExposure {
+    SafeRedisNumber.requirePositive(totalAmount, "totalAmount");
+  }
+
+  public static AcceptedBetExposure from(BetPlacedRequested event) {
+    Objects.requireNonNull(event, "event");
+    Objects.requireNonNull(event.getSelections(), "selections");
+    int selectionCount = event.getSelections().size();
+    if (selectionCount < 1 || selectionCount > RiskCheckCommand.MAX_SELECTIONS) {
+      throw new IllegalArgumentException("selection count must be between 1 and 15");
+    }
+
+    BetSlipTypeTag slipType = Objects.requireNonNull(event.getSlipType(), "slipType");
+    long unitAmount =
+        SafeRedisNumber.requirePositive(
+            Objects.requireNonNull(event.getStake(), "stake").getAmount(), "stake.amount");
+    long total =
+        switch (slipType) {
+          case SINGLE -> unitAmount(selectionCount, event);
+          case MULTIPLE -> multipleAmount(selectionCount, unitAmount, event);
+          case SYSTEM -> systemAmount(selectionCount, unitAmount, event);
+        };
+    return new AcceptedBetExposure(total);
+  }
+
+  private static long unitAmount(int actual, BetPlacedRequested event) {
+    requireNoSystemShape(event);
+    if (actual != 1) {
+      throw new IllegalArgumentException("SINGLE must contain exactly one selection");
+    }
+    return event.getStake().getAmount();
+  }
+
+  private static long multipleAmount(int count, long unitAmount, BetPlacedRequested event) {
+    requireNoSystemShape(event);
+    if (count < 2) {
+      throw new IllegalArgumentException("MULTIPLE must contain at least two selections");
+    }
+    return unitAmount;
+  }
+
+  private static long systemAmount(int count, long unitAmount, BetPlacedRequested event) {
+    Integer minimum = event.getSystemMinWins();
+    Integer total = event.getSystemTotalSelections();
+    if (minimum == null || total == null) {
+      throw new IllegalArgumentException("SYSTEM requires minWins and totalSelections");
+    }
+    if (total != count || total < 2 || total > RiskCheckCommand.MAX_SELECTIONS) {
+      throw new IllegalArgumentException("SYSTEM totalSelections must match 2..15 selections");
+    }
+    if (minimum < 1 || minimum > total) {
+      throw new IllegalArgumentException("SYSTEM minWins must be between 1 and totalSelections");
+    }
+    return SafeRedisNumber.multiply(unitAmount, combinations(total, minimum), "totalAmount");
+  }
+
+  private static void requireNoSystemShape(BetPlacedRequested event) {
+    if (event.getSystemMinWins() != null || event.getSystemTotalSelections() != null) {
+      throw new IllegalArgumentException("non-SYSTEM slip must not contain SYSTEM fields");
+    }
+  }
+
+  private static long combinations(int total, int choose) {
+    int smaller = Math.min(choose, total - choose);
+    long result = 1L;
+    for (int index = 1; index <= smaller; index++) {
+      result = Math.multiplyExact(result, total - smaller + index) / index;
+    }
+    return result;
+  }
+}


## `test(events): verify accepted bet exposure`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java
new file mode 100644
index 0000000..249e320
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetExposureTest.java
@@ -0,0 +1,90 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.List;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class AcceptedBetExposureTest {
+  @Test
+  void singleAndMultipleKeepTheSubmittedAmount() {
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 1, 75L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(75L);
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 3, 90L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(90L);
+  }
+
+  @Test
+  void systemMultipliesTheUnitAmountByExactCombinations() {
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 5, 5, 100L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(1_000L);
+    assertThat(AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 3, 5, 5, 100L)))
+        .extracting(AcceptedBetExposure::totalAmount)
+        .isEqualTo(1_000L);
+  }
+
+  @Test
+  void rejectsSlipShapeMismatches() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 2, 1L)))
+        .hasMessageContaining("exactly one");
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 1, 1L)))
+        .hasMessageContaining("at least two");
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 4, 3, 1L)))
+        .hasMessageContaining("totalSelections");
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 4, 3, 3, 1L)))
+        .hasMessageContaining("minWins");
+  }
+
+  @Test
+  void rejectsSystemFieldsOnUnitTotalSlips() {
+    assertThatThrownBy(() -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, 1, 2, 2, 1L)))
+        .hasMessageContaining("non-SYSTEM");
+  }
+
+  @Test
+  void rejectsUnsafeUnitAndCalculatedAmounts() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SINGLE, null, null, 1, 0L)))
+        .hasMessageContaining("positive");
+    long unsafeUnit = SafeRedisNumber.MAX_VALUE / 3L + 1L;
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.SYSTEM, 2, 3, 3, unsafeUnit)))
+        .hasMessageContaining("totalAmount");
+  }
+
+  @Test
+  void capsEverySlipAtFifteenSelections() {
+    assertThatThrownBy(
+            () -> AcceptedBetExposure.from(event(BetSlipTypeTag.MULTIPLE, null, null, 16, 1L)))
+        .hasMessageContaining("between 1 and 15");
+  }
+
+  private static BetPlacedRequested event(
+      BetSlipTypeTag type, Integer minimum, Integer total, int selectionCount, long amount) {
+    List<RequestedSelection> selections =
+        IntStream.range(0, selectionCount).mapToObj(ignored -> new RequestedSelection()).toList();
+    return BetPlacedRequested.newBuilder()
+        .setBetId("10000000-0000-4000-8000-000000000001")
+        .setUserId("20000000-0000-4000-8000-000000000001")
+        .setSlipType(type)
+        .setSystemMinWins(minimum)
+        .setSystemTotalSelections(total)
+        .setSelections(selections)
+        .setStake(new Money(amount, "KRW"))
+        .setIdempotencyKey("accepted-exposure-test")
+        .setRequestedAt(java.time.Instant.EPOCH)
+        .build();
+  }
+}


## `feat(events): validate accepted event envelopes`

diff --git a/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java b/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java
new file mode 100644
index 0000000..b99a55e
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/event/AcceptedBetEnvelope.java
@@ -0,0 +1,88 @@
+package com.sportsbook.risk.event;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.reservation.ReservationFingerprint;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Typed, canonical boundary for one accepted-bet Kafka event. */
+public record AcceptedBetEnvelope(RiskCheckCommand command, Instant requestedAt) {
+
+  public AcceptedBetEnvelope {
+    Objects.requireNonNull(command, "command");
+    Objects.requireNonNull(requestedAt, "requestedAt");
+  }
+
+  public static AcceptedBetEnvelope decode(String kafkaKey, byte[] payload, Instant observedAt) {
+    BetPlacedRequested event = AvroCodec.decode(payload, BetPlacedRequested.class);
+    return from(kafkaKey, event, observedAt);
+  }
+
+  static AcceptedBetEnvelope from(String kafkaKey, BetPlacedRequested event, Instant observedAt) {
+    Objects.requireNonNull(event, "event");
+    Objects.requireNonNull(observedAt, "observedAt");
+    AcceptedBetExposure exposure = AcceptedBetExposure.from(event);
+
+    UserId userId = UserId.of(uuid(event.getUserId(), "userId"));
+    if (!userId.value().toString().equals(kafkaKey)) {
+      throw new IllegalArgumentException("Kafka key must equal userId");
+    }
+    BetId betId = BetId.of(uuid(event.getBetId(), "betId"));
+    List<SelectionId> selectionIds =
+        event.getSelections().stream().map(AcceptedBetEnvelope::validateSelection).toList();
+    if (selectionIds.stream().distinct().count() != selectionIds.size()) {
+      throw new IllegalArgumentException("selectionIds must be unique");
+    }
+
+    com.sportsbook.protocol.event.Money wireStake =
+        Objects.requireNonNull(event.getStake(), "stake");
+    Currency currency = Currency.valueOf(required(wireStake.getCurrency(), "stake.currency"));
+    new IdempotencyKey(required(event.getIdempotencyKey(), "idempotencyKey"));
+    Instant requestedAt = Objects.requireNonNull(event.getRequestedAt(), "requestedAt");
+    RiskCheckCommand command =
+        new RiskCheckCommand(
+            userId, betId, new Money(exposure.totalAmount(), currency), selectionIds, observedAt);
+    return new AcceptedBetEnvelope(command, requestedAt);
+  }
+
+  public String reservationFingerprint() {
+    return ReservationFingerprint.of(command);
+  }
+
+  private static SelectionId validateSelection(RequestedSelection selection) {
+    Objects.requireNonNull(selection, "selection");
+    EventId.of(uuid(selection.getEventId(), "eventId"));
+    MarketId.of(uuid(selection.getMarketId(), "marketId"));
+    Odds.ofDecimal(required(selection.getOddsAtSubmission(), "oddsAtSubmission"));
+    return SelectionId.of(uuid(selection.getSelectionId(), "selectionId"));
+  }
+
+  private static UUID uuid(CharSequence value, String name) {
+    String text = required(value, name);
+    UUID uuid = UUID.fromString(text);
+    if (!uuid.toString().equals(text)) {
+      throw new IllegalArgumentException(name + " must be a canonical UUID");
+    }
+    return uuid;
+  }
+
+  private static String required(CharSequence value, String name) {
+    if (value == null || value.isEmpty()) {
+      throw new IllegalArgumentException(name + " is required");
+    }
+    return value.toString();
+  }
+}


## `test(events): verify accepted event envelopes`

diff --git a/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java b/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java
new file mode 100644
index 0000000..851a30e
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/event/AcceptedBetEnvelopeTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.risk.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.protocol.value.Currency;
+import java.time.Instant;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+class AcceptedBetEnvelopeTest {
+  private static final Instant REQUESTED_AT = Instant.parse("2026-08-21T06:00:00Z");
+  private static final Instant OBSERVED_AT = REQUESTED_AT.plusSeconds(1);
+  private static final String USER = "10000000-0000-4000-8000-000000000001";
+
+  @Test
+  void decodesTypedSystemExposureAndRecomputesTheReservationFingerprint() {
+    AcceptedBetEnvelope envelope = decode(validEvent(), USER);
+
+    assertThat(envelope.command().userId().value().toString()).isEqualTo(USER);
+    assertThat(envelope.command().stake().amount()).isEqualTo(300L);
+    assertThat(envelope.command().stake().currency()).isEqualTo(Currency.USD);
+    assertThat(envelope.command().selectionIds()).hasSize(3).doesNotHaveDuplicates();
+    assertThat(envelope.command().now()).isEqualTo(OBSERVED_AT);
+    assertThat(envelope.requestedAt()).isEqualTo(REQUESTED_AT);
+    assertThat(envelope.reservationFingerprint())
+        .isEqualTo("606ff15f92f1e6fc874679165bbbc550258e21e1c51354b249d9bd57376b4e22");
+  }
+
+  @Test
+  void rejectsMalformedTypedIdsAndMismatchedKafkaKeys() {
+    BetPlacedRequested malformed = validEvent();
+    malformed.setBetId("not-a-uuid");
+    assertThatThrownBy(() -> decode(malformed, USER)).isInstanceOf(IllegalArgumentException.class);
+
+    assertThatThrownBy(() -> decode(validEvent(), "10000000-0000-4000-8000-000000000002"))
+        .hasMessageContaining("Kafka key");
+  }
+
+  @Test
+  void rejectsDuplicateTypedSelections() {
+    BetPlacedRequested event = validEvent();
+    event.setSelections(List.of(selection(1), selection(1), selection(2)));
+    assertThatThrownBy(() -> decode(event, USER)).hasMessageContaining("unique");
+  }
+
+  @Test
+  void rejectsUnsupportedCurrenciesAndInvalidSlipShapes() {
+    BetPlacedRequested currency = validEvent();
+    currency.setStake(new Money(100L, "EUR"));
+    assertThatThrownBy(() -> decode(currency, USER)).isInstanceOf(IllegalArgumentException.class);
+
+    BetPlacedRequested shape = validEvent();
+    shape.setSystemTotalSelections(4);
+    assertThatThrownBy(() -> decode(shape, USER)).hasMessageContaining("totalSelections");
+  }
+
+  @Test
+  void validatesDiscardedSelectionAndIdempotencyFields() {
+    BetPlacedRequested event = validEvent();
+    event.getSelections().get(0).setEventId("bad-event-id");
+    assertThatThrownBy(() -> decode(event, USER)).isInstanceOf(IllegalArgumentException.class);
+
+    BetPlacedRequested key = validEvent();
+    key.setIdempotencyKey("\n");
+    assertThatThrownBy(() -> decode(key, USER)).isInstanceOf(IllegalArgumentException.class);
+  }
+
+  private static AcceptedBetEnvelope decode(BetPlacedRequested event, String kafkaKey) {
+    return AcceptedBetEnvelope.decode(kafkaKey, AvroCodec.encode(event), OBSERVED_AT);
+  }
+
+  private static BetPlacedRequested validEvent() {
+    return BetPlacedRequested.newBuilder()
+        .setBetId("20000000-0000-4000-8000-000000000001")
+        .setUserId(USER)
+        .setSlipType(BetSlipTypeTag.SYSTEM)
+        .setSystemMinWins(2)
+        .setSystemTotalSelections(3)
+        .setSelections(List.of(selection(1), selection(2), selection(3)))
+        .setStake(new Money(100L, "USD"))
+        .setIdempotencyKey("accepted-envelope-test")
+        .setRequestedAt(REQUESTED_AT)
+        .build();
+  }
+
+  private static RequestedSelection selection(int suffix) {
+    String tail = String.format("%012d", suffix);
+    return new RequestedSelection(
+        "30000000-0000-4000-8000-" + tail,
+        "40000000-0000-4000-8000-" + tail,
+        "50000000-0000-4000-8000-" + tail,
+        "1.8500");
+  }
+}


