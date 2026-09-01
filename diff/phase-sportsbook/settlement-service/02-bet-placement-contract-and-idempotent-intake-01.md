# 베팅 접수 계약과 멱등 입력

## `feat(placement): represent decoded placement snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetPlacement.java b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacement.java
new file mode 100644
index 0000000..083cc7d
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacement.java
@@ -0,0 +1,38 @@
+package com.sportsbook.settlement.readmodel;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Strictly decoded, Avro-free placement snapshot consumed by persistence. */
+public record BetPlacement(
+    UUID betId,
+    UUID userId,
+    BetSlipType slipType,
+    Money unitStake,
+    Instant requestedAt,
+    List<Selection> selections) {
+
+  public BetPlacement {
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(slipType, "slipType");
+    Objects.requireNonNull(unitStake, "unitStake");
+    Objects.requireNonNull(requestedAt, "requestedAt");
+    selections = List.copyOf(Objects.requireNonNull(selections, "selections"));
+  }
+
+  public record Selection(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
+
+    public Selection {
+      Objects.requireNonNull(eventId, "eventId");
+      Objects.requireNonNull(marketId, "marketId");
+      Objects.requireNonNull(selectionId, "selectionId");
+      Objects.requireNonNull(odds, "odds");
+    }
+  }
+}


## `feat(placement): validate placement boundary invariants`

diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementValidator.java b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementValidator.java
new file mode 100644
index 0000000..918e180
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementValidator.java
@@ -0,0 +1,43 @@
+package com.sportsbook.settlement.readmodel;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.HashSet;
+import java.util.Set;
+import java.util.UUID;
+
+/** Enforces the service-owned boundary before a decoded placement reaches storage. */
+public final class BetPlacementValidator {
+
+  private static final int MAX_SELECTIONS = 15;
+
+  public BetPlacement validate(BetPlacement placement) {
+    if (placement.unitStake().amount() <= 0) {
+      throw invalid("unit stake must be positive");
+    }
+    int count = placement.selections().size();
+    if (count < 1 || count > MAX_SELECTIONS) {
+      throw invalid("selection count must be in 1..15");
+    }
+    if (placement.slipType() instanceof BetSlipType.Single && count != 1) {
+      throw invalid("single slip requires exactly one selection");
+    }
+    if (placement.slipType() instanceof BetSlipType.Multiple && count < 2) {
+      throw invalid("multiple slip requires at least two selections");
+    }
+    if (placement.slipType() instanceof BetSlipType.System system
+        && system.totalSelections() != count) {
+      throw invalid("system total selections must match decoded selections");
+    }
+    Set<UUID> selected = new HashSet<>();
+    for (BetPlacement.Selection selection : placement.selections()) {
+      if (!selected.add(selection.selectionId())) {
+        throw invalid("selection identifiers must be unique");
+      }
+    }
+    return placement;
+  }
+
+  private static PlacementContractException invalid(String detail) {
+    return new PlacementContractException("Invalid BetPlacedRequested: " + detail);
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/readmodel/PlacementContractException.java b/src/main/java/com/sportsbook/settlement/readmodel/PlacementContractException.java
new file mode 100644
index 0000000..93845d4
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/readmodel/PlacementContractException.java
@@ -0,0 +1,9 @@
+package com.sportsbook.settlement.readmodel;
+
+/** Permanent placement contract failure that must not be retried. */
+public final class PlacementContractException extends IllegalArgumentException {
+
+  public PlacementContractException(String message) {
+    super(message);
+  }
+}


## `feat(placement): map the fixed Avro placement contract`

diff --git a/src/main/java/com/sportsbook/settlement/event/BetPlacedMapper.java b/src/main/java/com/sportsbook/settlement/event/BetPlacedMapper.java
new file mode 100644
index 0000000..2650158
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/BetPlacedMapper.java
@@ -0,0 +1,78 @@
+package com.sportsbook.settlement.event;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.settlement.readmodel.BetPlacement;
+import com.sportsbook.settlement.readmodel.PlacementContractException;
+import java.math.BigDecimal;
+import java.util.UUID;
+
+/** Maps the fixed Avro placement contract without lossy normalization. */
+public final class BetPlacedMapper {
+
+  public BetPlacement map(BetPlacedRequested event) {
+    try {
+      return new BetPlacement(
+          canonicalUuid(event.getBetId(), "betId"),
+          canonicalUuid(event.getUserId(), "userId"),
+          slipType(event),
+          new Money(
+              event.getStake().getAmount(),
+              Currency.valueOf(event.getStake().getCurrency().toString())),
+          event.getRequestedAt(),
+          event.getSelections().stream().map(this::selection).toList());
+    } catch (PlacementContractException exception) {
+      throw exception;
+    } catch (RuntimeException exception) {
+      throw new PlacementContractException(
+          "Invalid BetPlacedRequested field: " + exception.getMessage());
+    }
+  }
+
+  private BetPlacement.Selection selection(RequestedSelection selection) {
+    String encodedOdds = selection.getOddsAtSubmission().toString();
+    Odds odds = Odds.ofDecimal(new BigDecimal(encodedOdds));
+    if (!odds.decimal().toPlainString().equals(encodedOdds)) {
+      throw new PlacementContractException("oddsAtSubmission must use canonical scale 4");
+    }
+    return new BetPlacement.Selection(
+        canonicalUuid(selection.getEventId(), "eventId"),
+        canonicalUuid(selection.getMarketId(), "marketId"),
+        canonicalUuid(selection.getSelectionId(), "selectionId"),
+        odds);
+  }
+
+  private static BetSlipType slipType(BetPlacedRequested event) {
+    Integer minimumWins = event.getSystemMinWins();
+    Integer totalSelections = event.getSystemTotalSelections();
+    if (event.getSlipType() == BetSlipTypeTag.SYSTEM) {
+      if (minimumWins == null || totalSelections == null) {
+        throw new PlacementContractException("SYSTEM requires K and N");
+      }
+      return new BetSlipType.System(minimumWins, totalSelections);
+    }
+    if (minimumWins != null || totalSelections != null) {
+      throw new PlacementContractException("Non-system slip must omit K and N");
+    }
+    return event.getSlipType() == BetSlipTypeTag.SINGLE
+        ? new BetSlipType.Single()
+        : new BetSlipType.Multiple();
+  }
+
+  private static UUID canonicalUuid(CharSequence encoded, String field) {
+    if (encoded == null) {
+      throw new PlacementContractException(field + " is required");
+    }
+    String text = encoded.toString();
+    UUID parsed = UUID.fromString(text);
+    if (!parsed.toString().equals(text)) {
+      throw new PlacementContractException(field + " must be canonical lowercase UUID text");
+    }
+    return parsed;
+  }
+}


## `test(placement): reject lossy Avro mappings`

diff --git a/src/test/java/com/sportsbook/settlement/event/BetPlacedMapperTest.java b/src/test/java/com/sportsbook/settlement/event/BetPlacedMapperTest.java
new file mode 100644
index 0000000..904e280
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/BetPlacedMapperTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.settlement.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.RequestedSelection;
+import com.sportsbook.settlement.readmodel.BetPlacement;
+import com.sportsbook.settlement.readmodel.PlacementContractException;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetPlacedMapperTest {
+
+  private final BetPlacedMapper mapper = new BetPlacedMapper();
+
+  @Test
+  void mapsCanonicalSystemFieldsAndOriginalUnitStake() {
+    BetPlacedRequested event = event();
+
+    BetPlacement placement = mapper.map(event);
+
+    assertThat(placement.slipType()).isEqualTo(new BetSlipType.System(1, 2));
+    assertThat(placement.unitStake().amount()).isEqualTo(250);
+    assertThat(placement.selections()).extracting(BetPlacement.Selection::selectionId).hasSize(2);
+  }
+
+  @Test
+  void rejectsLossyOddsAndNoncanonicalIdentifiers() {
+    BetPlacedRequested event = event();
+    event.setBetId(event.getBetId().toString().toUpperCase(java.util.Locale.ROOT));
+    assertInvalid(event);
+
+    event = event();
+    event.getSelections().get(0).setOddsAtSubmission("2.00");
+    assertInvalid(event);
+  }
+
+  @Test
+  void rejectsSystemFieldsOnANonSystemSlip() {
+    BetPlacedRequested event = event();
+    event.setSlipType(BetSlipTypeTag.SINGLE);
+
+    assertInvalid(event);
+  }
+
+  private void assertInvalid(BetPlacedRequested event) {
+    assertThatThrownBy(() -> mapper.map(event)).isInstanceOf(PlacementContractException.class);
+  }
+
+  private static BetPlacedRequested event() {
+    List<RequestedSelection> selections =
+        List.of(selection(UUID.randomUUID()), selection(UUID.randomUUID()));
+    return BetPlacedRequested.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(UUID.randomUUID().toString())
+        .setSlipType(BetSlipTypeTag.SYSTEM)
+        .setSystemMinWins(1)
+        .setSystemTotalSelections(2)
+        .setSelections(selections)
+        .setStake(Money.newBuilder().setAmount(250).setCurrency("KRW").build())
+        .setIdempotencyKey("placement-key")
+        .setRequestedAt(Instant.EPOCH)
+        .build();
+  }
+
+  private static RequestedSelection selection(UUID selectionId) {
+    return RequestedSelection.newBuilder()
+        .setEventId(UUID.randomUUID().toString())
+        .setMarketId(UUID.randomUUID().toString())
+        .setSelectionId(selectionId.toString())
+        .setOddsAtSubmission("2.0000")
+        .build();
+  }
+}


## `feat(placement): fingerprint replay semantics`

diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinter.java b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinter.java
new file mode 100644
index 0000000..b98e01e
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinter.java
@@ -0,0 +1,98 @@
+package com.sportsbook.settlement.readmodel;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.BetSelection;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.List;
+import java.util.StringJoiner;
+import java.util.UUID;
+
+/** Builds the same semantic identity from an incoming or persisted placement snapshot. */
+public final class BetPlacementFingerprinter {
+
+  public String fingerprint(BetPlacement placement) {
+    return fingerprint(
+        placement.betId(),
+        placement.userId(),
+        placement.slipType(),
+        placement.unitStake(),
+        placement.requestedAt().toEpochMilli(),
+        placement.selections().stream().map(this::canonical).toList());
+  }
+
+  public String fingerprint(Bet bet) {
+    return fingerprint(
+        bet.betId(),
+        bet.userId(),
+        bet.slipType(),
+        bet.stake(),
+        bet.requestedAt().toEpochMilli(),
+        bet.selections().stream().map(this::canonical).toList());
+  }
+
+  private String fingerprint(
+      UUID betId,
+      UUID userId,
+      BetSlipType slipType,
+      Money stake,
+      long requestedAt,
+      List<String> selections) {
+    StringJoiner fields = new StringJoiner("\0");
+    fields.add(betId.toString()).add(userId.toString()).add(kind(slipType));
+    if (slipType instanceof BetSlipType.System system) {
+      fields.add(Integer.toString(system.minWins()));
+      fields.add(Integer.toString(system.totalSelections()));
+    } else {
+      fields.add("").add("");
+    }
+    fields.add(Long.toString(stake.amount())).add(stake.currency().name());
+    fields.add(Long.toString(requestedAt)).add(Integer.toString(selections.size()));
+    selections.forEach(fields::add);
+    return sha256(fields.toString());
+  }
+
+  private String canonical(BetPlacement.Selection selection) {
+    return canonical(
+        selection.eventId(), selection.marketId(), selection.selectionId(), selection.odds());
+  }
+
+  private String canonical(BetSelection selection) {
+    return canonical(
+        selection.eventId(), selection.marketId(), selection.selectionId(), selection.odds());
+  }
+
+  private String canonical(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
+    return String.join(
+        ":",
+        eventId.toString(),
+        marketId.toString(),
+        selectionId.toString(),
+        Odds.ofDecimal(odds.decimal()).decimal().toPlainString());
+  }
+
+  private static String kind(BetSlipType type) {
+    if (type instanceof BetSlipType.Single) {
+      return "SINGLE";
+    }
+    if (type instanceof BetSlipType.Multiple) {
+      return "MULTIPLE";
+    }
+    return "SYSTEM";
+  }
+
+  private static String sha256(String value) {
+    try {
+      return HexFormat.of()
+          .formatHex(
+              MessageDigest.getInstance("SHA-256").digest(value.getBytes(StandardCharsets.UTF_8)));
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("JDK must provide SHA-256", exception);
+    }
+  }
+}


## `feat(placement): persist exact replays idempotently`

diff --git a/src/main/java/com/sportsbook/settlement/config/TimeConfiguration.java b/src/main/java/com/sportsbook/settlement/config/TimeConfiguration.java
index d90b62a..be8ec60 100644
--- a/src/main/java/com/sportsbook/settlement/config/TimeConfiguration.java
+++ b/src/main/java/com/sportsbook/settlement/config/TimeConfiguration.java
@@ -1,5 +1,7 @@
 package com.sportsbook.settlement.config;
 
+import com.sportsbook.settlement.readmodel.BetPlacementFingerprinter;
+import com.sportsbook.settlement.readmodel.BetPlacementValidator;
 import java.time.Clock;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
@@ -11,4 +13,14 @@ public class TimeConfiguration {
   Clock settlementClock() {
     return Clock.systemUTC();
   }
+
+  @Bean
+  BetPlacementValidator betPlacementValidator() {
+    return new BetPlacementValidator();
+  }
+
+  @Bean
+  BetPlacementFingerprinter betPlacementFingerprinter() {
+    return new BetPlacementFingerprinter();
+  }
 }
diff --git a/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
new file mode 100644
index 0000000..447ffe2
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/readmodel/BetReadModelWriter.java
@@ -0,0 +1,85 @@
+package com.sportsbook.settlement.readmodel;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.BetSelection;
+import com.sportsbook.settlement.domain.EmbeddedMoney;
+import com.sportsbook.settlement.domain.SlipKind;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Clock;
+import java.util.List;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+/** Persists an immutable placement snapshot and isolates conflicting replays. */
+@Service
+public class BetReadModelWriter {
+
+  public enum RecordResult {
+    CREATED,
+    EXACT_REPLAY
+  }
+
+  private final BetRepository repository;
+  private final BetPlacementValidator validator;
+  private final BetPlacementFingerprinter fingerprinter;
+  private final Clock clock;
+
+  public BetReadModelWriter(
+      BetRepository repository,
+      BetPlacementValidator validator,
+      BetPlacementFingerprinter fingerprinter,
+      Clock clock) {
+    this.repository = repository;
+    this.validator = validator;
+    this.fingerprinter = fingerprinter;
+    this.clock = clock;
+  }
+
+  @Transactional
+  public RecordResult record(BetPlacement candidate) {
+    BetPlacement placement = validator.validate(candidate);
+    return repository
+        .findWithSelectionsById(placement.betId())
+        .map(existing -> replay(existing, placement))
+        .orElseGet(() -> create(placement));
+  }
+
+  private RecordResult replay(Bet existing, BetPlacement placement) {
+    if (!fingerprinter.fingerprint(existing).equals(fingerprinter.fingerprint(placement))) {
+      throw new PlacementContractException("Conflicting BetPlacedRequested replay");
+    }
+    return RecordResult.EXACT_REPLAY;
+  }
+
+  private RecordResult create(BetPlacement placement) {
+    List<BetSelection> selections =
+        placement.selections().stream()
+            .map(
+                selection ->
+                    new BetSelection(
+                        selection.eventId(),
+                        selection.marketId(),
+                        selection.selectionId(),
+                        selection.odds()))
+            .toList();
+    Integer minimumWins = null;
+    Integer totalSelections = null;
+    if (placement.slipType() instanceof BetSlipType.System system) {
+      minimumWins = system.minWins();
+      totalSelections = system.totalSelections();
+    }
+    repository.save(
+        Bet.pending(
+            placement.betId(),
+            placement.userId(),
+            SlipKind.from(placement.slipType()),
+            minimumWins,
+            totalSelections,
+            EmbeddedMoney.of(placement.unitStake()),
+            placement.requestedAt(),
+            selections,
+            clock.instant()));
+    return RecordResult.CREATED;
+  }
+}


## `test(placement): distinguish conflicting replays`

diff --git a/src/test/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinterTest.java b/src/test/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinterTest.java
new file mode 100644
index 0000000..f858aec
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/readmodel/BetPlacementFingerprinterTest.java
@@ -0,0 +1,79 @@
+package com.sportsbook.settlement.readmodel;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.BetSelection;
+import com.sportsbook.settlement.domain.EmbeddedMoney;
+import com.sportsbook.settlement.domain.SlipKind;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetPlacementFingerprinterTest {
+
+  private final BetPlacementFingerprinter fingerprinter = new BetPlacementFingerprinter();
+
+  @Test
+  void matchesIncomingAndPersistedSemanticSnapshots() {
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    Instant requestedAt = Instant.parse("2026-01-01T00:00:00.123Z");
+    BetPlacement.Selection decoded = selection();
+    BetPlacement placement =
+        new BetPlacement(
+            betId, userId, new BetSlipType.Single(), Money.krw(100), requestedAt, List.of(decoded));
+    BetSelection stored =
+        new BetSelection(
+            decoded.eventId(), decoded.marketId(), decoded.selectionId(), decoded.odds());
+    Bet bet =
+        Bet.pending(
+            betId,
+            userId,
+            SlipKind.SINGLE,
+            null,
+            null,
+            new EmbeddedMoney(100, Currency.KRW),
+            requestedAt,
+            List.of(stored),
+            Instant.EPOCH);
+
+    assertThat(fingerprinter.fingerprint(placement)).isEqualTo(fingerprinter.fingerprint(bet));
+    assertThat(fingerprinter.fingerprint(placement)).matches("[0-9a-f]{64}");
+  }
+
+  @Test
+  void changesWhenReplaySemanticsChange() {
+    BetPlacement.Selection selected = selection();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    BetPlacement first =
+        new BetPlacement(
+            betId,
+            userId,
+            new BetSlipType.Single(),
+            Money.krw(100),
+            Instant.EPOCH,
+            List.of(selected));
+    BetPlacement changed =
+        new BetPlacement(
+            betId,
+            userId,
+            new BetSlipType.Single(),
+            Money.krw(101),
+            Instant.EPOCH,
+            List.of(selected));
+
+    assertThat(fingerprinter.fingerprint(first)).isNotEqualTo(fingerprinter.fingerprint(changed));
+  }
+
+  private static BetPlacement.Selection selection() {
+    return new BetPlacement.Selection(
+        UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.0000"));
+  }
+}


