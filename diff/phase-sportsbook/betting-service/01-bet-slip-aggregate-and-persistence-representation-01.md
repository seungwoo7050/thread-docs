# 베팅 슬립 집계와 영속 표현

## `feat(domain): create pending bet identity`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
new file mode 100644
index 0000000..9b5989b
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -0,0 +1,88 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.protocol.domain.BetStatus;
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import jakarta.persistence.Version;
+import java.time.Instant;
+import java.util.UUID;
+
+@Entity
+@Table(name = "bet")
+public class Bet {
+
+  @Id
+  @Column(name = "bet_id", nullable = false, updatable = false)
+  private UUID betId;
+
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Column(name = "bet_reference", nullable = false, updatable = false, length = 32)
+  private String betReference;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "status", nullable = false, length = 16)
+  private BetStatus status;
+
+  @Column(name = "idempotency_key", nullable = false, updatable = false, length = 128)
+  private String idempotencyKey;
+
+  @Version
+  @Column(name = "version", nullable = false)
+  private long version;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "updated_at", nullable = false)
+  private Instant updatedAt;
+
+  protected Bet() {}
+
+  private Bet(BetDraft draft) {
+    this.betId = draft.betId();
+    this.userId = draft.userId();
+    this.betReference = draft.reference();
+    this.idempotencyKey = draft.idempotencyKey().value();
+    this.status = BetStatus.PENDING;
+    this.createdAt = draft.createdAt();
+    this.updatedAt = draft.createdAt();
+  }
+
+  static Bet from(BetDraft draft) {
+    return new Bet(draft);
+  }
+
+  public UUID betId() {
+    return betId;
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+
+  public String betReference() {
+    return betReference;
+  }
+
+  public BetStatus status() {
+    return status;
+  }
+
+  public String idempotencyKey() {
+    return idempotencyKey;
+  }
+
+  public Instant createdAt() {
+    return createdAt;
+  }
+
+  public Instant updatedAt() {
+    return updatedAt;
+  }
+}


## `test(domain): verify pending bet identity`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
new file mode 100644
index 0000000..c3a8847
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetTest {
+
+  static final Instant NOW = Instant.parse("2026-08-22T00:00:00Z");
+  static final String FINGERPRINT = "a".repeat(64);
+
+  @Test
+  void beginsPendingWithOwnedIdentity() {
+    UUID betId = UUID.randomUUID();
+    Bet bet = Bet.from(draft(betId, new BetSlipType.Single()));
+
+    assertThat(bet.betId()).isEqualTo(betId);
+    assertThat(bet.status()).isEqualTo(BetStatus.PENDING);
+    assertThat(bet.idempotencyKey()).isEqualTo("request-1");
+    assertThat(bet.createdAt()).isEqualTo(NOW);
+  }
+
+  static BetDraft draft(UUID betId, BetSlipType type) {
+    return new BetDraft(
+        betId,
+        UUID.randomUUID(),
+        "B-2026-08-22-00000000",
+        type,
+        Money.krw(1_000),
+        Money.krw(2_000),
+        IdempotencyKey.of("request-1"),
+        FINGERPRINT,
+        NOW);
+  }
+}


## `feat(domain): model persisted bet legs`

diff --git a/src/main/java/com/sportsbook/betting/domain/BetLeg.java b/src/main/java/com/sportsbook/betting/domain/BetLeg.java
new file mode 100644
index 0000000..fc97a00
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/BetLeg.java
@@ -0,0 +1,85 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.betting.infrastructure.id.UuidV7;
+import com.sportsbook.protocol.value.Odds;
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.FetchType;
+import jakarta.persistence.Id;
+import jakarta.persistence.JoinColumn;
+import jakarta.persistence.ManyToOne;
+import jakarta.persistence.Table;
+import java.math.BigDecimal;
+import java.util.Objects;
+import java.util.UUID;
+
+@Entity
+@Table(name = "bet_leg")
+public class BetLeg {
+
+  @Id
+  @Column(name = "leg_id", nullable = false, updatable = false)
+  private UUID legId;
+
+  @ManyToOne(fetch = FetchType.LAZY, optional = false)
+  @JoinColumn(name = "bet_id", nullable = false)
+  private Bet bet;
+
+  @Column(name = "leg_index", nullable = false)
+  private int legIndex;
+
+  @Column(name = "event_id", nullable = false)
+  private UUID eventId;
+
+  @Column(name = "market_id", nullable = false)
+  private UUID marketId;
+
+  @Column(name = "selection_id", nullable = false)
+  private UUID selectionId;
+
+  @Column(name = "odds_at_submission", nullable = false, precision = 9, scale = 4)
+  private BigDecimal oddsAtSubmission;
+
+  protected BetLeg() {}
+
+  private BetLeg(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
+    this.legId = UuidV7.generate();
+    this.eventId = Objects.requireNonNull(eventId, "eventId");
+    this.marketId = Objects.requireNonNull(marketId, "marketId");
+    this.selectionId = Objects.requireNonNull(selectionId, "selectionId");
+    this.oddsAtSubmission = Objects.requireNonNull(odds, "odds").decimal();
+  }
+
+  public static BetLeg create(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
+    return new BetLeg(eventId, marketId, selectionId, odds);
+  }
+
+  void assignTo(Bet owner, int index) {
+    this.bet = Objects.requireNonNull(owner, "owner");
+    this.legIndex = index;
+  }
+
+  public UUID legId() {
+    return legId;
+  }
+
+  public int legIndex() {
+    return legIndex;
+  }
+
+  public UUID eventId() {
+    return eventId;
+  }
+
+  public UUID marketId() {
+    return marketId;
+  }
+
+  public UUID selectionId() {
+    return selectionId;
+  }
+
+  public Odds oddsAtSubmission() {
+    return Odds.ofDecimal(oddsAtSubmission);
+  }
+}


## `test(domain): verify retained leg values`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetLegTest.java b/src/test/java/com/sportsbook/betting/domain/BetLegTest.java
new file mode 100644
index 0000000..3e3525b
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/BetLegTest.java
@@ -0,0 +1,25 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Odds;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetLegTest {
+
+  @Test
+  void retainsSelectionAndQuotedOdds() {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+
+    BetLeg leg = BetLeg.create(eventId, marketId, selectionId, Odds.ofDecimal("2.1250"));
+
+    assertThat(leg.legId()).isNotNull();
+    assertThat(leg.eventId()).isEqualTo(eventId);
+    assertThat(leg.marketId()).isEqualTo(marketId);
+    assertThat(leg.selectionId()).isEqualTo(selectionId);
+    assertThat(leg.oddsAtSubmission()).isEqualTo(Odds.ofDecimal("2.125"));
+  }
+}


## `feat(domain): attach structurally valid legs`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index f92bc28..dd70a61 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -5,15 +5,21 @@ import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.value.Money;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
+import jakarta.persistence.CascadeType;
 import jakarta.persistence.Column;
 import jakarta.persistence.Embedded;
 import jakarta.persistence.Entity;
 import jakarta.persistence.EnumType;
 import jakarta.persistence.Enumerated;
 import jakarta.persistence.Id;
+import jakarta.persistence.OneToMany;
+import jakarta.persistence.OrderBy;
 import jakarta.persistence.Table;
 import jakarta.persistence.Version;
 import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.Objects;
 import java.util.UUID;
 
 @Entity
@@ -79,6 +85,10 @@ public class Bet {
   @Column(name = "updated_at", nullable = false)
   private Instant updatedAt;
 
+  @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
+  @OrderBy("legIndex ASC")
+  private List<BetLeg> legs = new ArrayList<>();
+
   protected Bet() {}
 
   private Bet(BetDraft draft) {
@@ -103,6 +113,36 @@ public class Bet {
     return new Bet(draft);
   }
 
+  public static Bet pending(BetDraft draft, List<BetLeg> legs) {
+    Objects.requireNonNull(legs, "legs");
+    requireStructure(draft.slipType(), legs.size());
+    Bet bet = new Bet(draft);
+    for (int index = 0; index < legs.size(); index++) {
+      BetLeg leg = Objects.requireNonNull(legs.get(index), "leg");
+      leg.assignTo(bet, index);
+      bet.legs.add(leg);
+    }
+    return bet;
+  }
+
+  private static void requireStructure(BetSlipType type, int legCount) {
+    if (type instanceof BetSlipType.Single && legCount != 1) {
+      throw new IllegalArgumentException("SINGLE requires exactly one leg");
+    }
+    if (type instanceof BetSlipType.Multiple && legCount < 2) {
+      throw new IllegalArgumentException("MULTIPLE requires at least two legs");
+    }
+    if (type instanceof BetSlipType.System system && system.totalSelections() != legCount) {
+      throw new IllegalArgumentException("SYSTEM totalSelections must equal leg count");
+    }
+  }
+
+  private void requireSelectionEvent(UUID eventId) {
+    if (legs.stream().noneMatch(leg -> leg.eventId().equals(eventId))) {
+      throw new IllegalArgumentException("Resolution event must belong to a selected leg");
+    }
+  }
+
   public UUID betId() {
     return betId;
   }
@@ -150,4 +190,8 @@ public class Bet {
   public Instant updatedAt() {
     return updatedAt;
   }
+
+  public List<BetLeg> legs() {
+    return List.copyOf(legs);
+  }
 }


## `test(domain): verify aggregate leg structure`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index c4eddea..2942cdf 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -7,6 +7,7 @@ import com.sportsbook.protocol.domain.BetStatus;
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
 import java.time.Instant;
+import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 
@@ -36,6 +37,35 @@ class BetTest {
     assertThat(bet.requestFingerprint()).isEqualTo(FINGERPRINT);
   }
 
+  @Test
+  void assignsLegOrderWhenCreatingAggregate() {
+    Bet bet =
+        Bet.pending(
+            draft(UUID.randomUUID(), new BetSlipType.Multiple()),
+            List.of(leg("2.00"), leg("3.00")));
+
+    assertThat(bet.legs()).extracting(BetLeg::legIndex).containsExactly(0, 1);
+  }
+
+  @Test
+  void rejectsSlipShapeMismatch() {
+    org.assertj.core.api.Assertions.assertThatThrownBy(
+            () ->
+                Bet.pending(
+                    draft(UUID.randomUUID(), new BetSlipType.Single()),
+                    List.of(leg("2.00"), leg("3.00"))))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("exactly one");
+  }
+
+  static BetLeg leg(String odds) {
+    return BetLeg.create(
+        UUID.randomUUID(),
+        UUID.randomUUID(),
+        UUID.randomUUID(),
+        com.sportsbook.protocol.value.Odds.ofDecimal(odds));
+  }
+
   static BetDraft draft(UUID betId, BetSlipType type) {
     return new BetDraft(
         betId,


## `feat(domain): persist monetary values`

diff --git a/src/main/java/com/sportsbook/betting/domain/EmbeddedMoney.java b/src/main/java/com/sportsbook/betting/domain/EmbeddedMoney.java
new file mode 100644
index 0000000..b3841db
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/EmbeddedMoney.java
@@ -0,0 +1,44 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.Column;
+import jakarta.persistence.Embeddable;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import java.util.Objects;
+
+@Embeddable
+public class EmbeddedMoney {
+
+  @Column(nullable = false)
+  private long amount;
+
+  @Enumerated(EnumType.STRING)
+  @Column(nullable = false, length = 3)
+  private Currency currency;
+
+  protected EmbeddedMoney() {}
+
+  private EmbeddedMoney(long amount, Currency currency) {
+    this.amount = amount;
+    this.currency = Objects.requireNonNull(currency, "currency");
+  }
+
+  public static EmbeddedMoney of(Money money) {
+    Objects.requireNonNull(money, "money");
+    return new EmbeddedMoney(money.amount(), money.currency());
+  }
+
+  public Money toMoney() {
+    return new Money(amount, currency);
+  }
+
+  public long amount() {
+    return amount;
+  }
+
+  public Currency currency() {
+    return currency;
+  }
+}


## `test(domain): verify monetary round trip`

diff --git a/src/test/java/com/sportsbook/betting/domain/EmbeddedMoneyTest.java b/src/test/java/com/sportsbook/betting/domain/EmbeddedMoneyTest.java
new file mode 100644
index 0000000..661c3f1
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/EmbeddedMoneyTest.java
@@ -0,0 +1,20 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Money;
+import org.junit.jupiter.api.Test;
+
+class EmbeddedMoneyTest {
+
+  @Test
+  void roundTripsSharedMoney() {
+    Money source = Money.krw(12_500);
+
+    EmbeddedMoney persisted = EmbeddedMoney.of(source);
+
+    assertThat(persisted.amount()).isEqualTo(12_500);
+    assertThat(persisted.currency()).isEqualTo(source.currency());
+    assertThat(persisted.toMoney()).isEqualTo(source);
+  }
+}


## `feat(domain): persist slip discriminators`

diff --git a/src/main/java/com/sportsbook/betting/domain/SlipKind.java b/src/main/java/com/sportsbook/betting/domain/SlipKind.java
new file mode 100644
index 0000000..6e565c2
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/SlipKind.java
@@ -0,0 +1,24 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.Objects;
+
+public enum SlipKind {
+  SINGLE,
+  MULTIPLE,
+  SYSTEM;
+
+  public static SlipKind of(BetSlipType slipType) {
+    Objects.requireNonNull(slipType, "slipType");
+    if (slipType instanceof BetSlipType.Single) {
+      return SINGLE;
+    }
+    if (slipType instanceof BetSlipType.Multiple) {
+      return MULTIPLE;
+    }
+    if (slipType instanceof BetSlipType.System) {
+      return SYSTEM;
+    }
+    throw new IllegalArgumentException("Unsupported slip type");
+  }
+}


## `test(domain): verify slip discriminator mapping`

diff --git a/src/test/java/com/sportsbook/betting/domain/SlipKindTest.java b/src/test/java/com/sportsbook/betting/domain/SlipKindTest.java
new file mode 100644
index 0000000..eaa52ae
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/SlipKindTest.java
@@ -0,0 +1,16 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import org.junit.jupiter.api.Test;
+
+class SlipKindTest {
+
+  @Test
+  void mapsEverySharedSlipVariant() {
+    assertThat(SlipKind.of(new BetSlipType.Single())).isEqualTo(SlipKind.SINGLE);
+    assertThat(SlipKind.of(new BetSlipType.Multiple())).isEqualTo(SlipKind.MULTIPLE);
+    assertThat(SlipKind.of(new BetSlipType.System(2, 3))).isEqualTo(SlipKind.SYSTEM);
+  }
+}


## `feat(domain): retain slip and wager values`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 9b5989b..f92bc28 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -1,7 +1,12 @@
 package com.sportsbook.betting.domain;
 
 import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.AttributeOverride;
+import jakarta.persistence.AttributeOverrides;
 import jakarta.persistence.Column;
+import jakarta.persistence.Embedded;
 import jakarta.persistence.Entity;
 import jakarta.persistence.EnumType;
 import jakarta.persistence.Enumerated;
@@ -25,6 +30,38 @@ public class Bet {
   @Column(name = "bet_reference", nullable = false, updatable = false, length = 32)
   private String betReference;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "slip_type", nullable = false, updatable = false, length = 16)
+  private SlipKind slipKind;
+
+  @Column(name = "system_min_wins")
+  private Integer systemMinWins;
+
+  @Column(name = "system_total_selections")
+  private Integer systemTotalSelections;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(name = "amount", column = @Column(name = "stake_amount", nullable = false)),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "stake_currency", nullable = false, length = 3))
+  })
+  private EmbeddedMoney stake;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(
+        name = "amount", column = @Column(name = "max_payout_amount", nullable = false)),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "max_payout_currency", nullable = false, length = 3))
+  })
+  private EmbeddedMoney maxPayout;
+
+  @Column(name = "request_fingerprint", updatable = false, length = 64)
+  private String requestFingerprint;
+
   @Enumerated(EnumType.STRING)
   @Column(name = "status", nullable = false, length = 16)
   private BetStatus status;
@@ -48,6 +85,14 @@ public class Bet {
     this.betId = draft.betId();
     this.userId = draft.userId();
     this.betReference = draft.reference();
+    this.slipKind = SlipKind.of(draft.slipType());
+    if (draft.slipType() instanceof BetSlipType.System system) {
+      this.systemMinWins = system.minWins();
+      this.systemTotalSelections = system.totalSelections();
+    }
+    this.stake = EmbeddedMoney.of(draft.stake());
+    this.maxPayout = EmbeddedMoney.of(draft.maxPayout());
+    this.requestFingerprint = draft.requestFingerprint();
     this.idempotencyKey = draft.idempotencyKey().value();
     this.status = BetStatus.PENDING;
     this.createdAt = draft.createdAt();
@@ -74,6 +119,26 @@ public class Bet {
     return status;
   }
 
+  public BetSlipType slipType() {
+    return switch (slipKind) {
+      case SINGLE -> new BetSlipType.Single();
+      case MULTIPLE -> new BetSlipType.Multiple();
+      case SYSTEM -> new BetSlipType.System(systemMinWins, systemTotalSelections);
+    };
+  }
+
+  public Money stake() {
+    return stake.toMoney();
+  }
+
+  public Money maxPayout() {
+    return maxPayout.toMoney();
+  }
+
+  public String requestFingerprint() {
+    return requestFingerprint;
+  }
+
   public String idempotencyKey() {
     return idempotencyKey;
   }


## `feat(domain): validate pending bet drafts`

diff --git a/src/main/java/com/sportsbook/betting/domain/BetDraft.java b/src/main/java/com/sportsbook/betting/domain/BetDraft.java
new file mode 100644
index 0000000..3859a7d
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/BetDraft.java
@@ -0,0 +1,44 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+import java.util.regex.Pattern;
+
+public record BetDraft(
+    UUID betId,
+    UUID userId,
+    String reference,
+    BetSlipType slipType,
+    Money stake,
+    Money maxPayout,
+    IdempotencyKey idempotencyKey,
+    String requestFingerprint,
+    Instant createdAt) {
+
+  private static final Pattern SHA_256 = Pattern.compile("[0-9a-f]{64}");
+
+  public BetDraft {
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(slipType, "slipType");
+    Objects.requireNonNull(stake, "stake");
+    Objects.requireNonNull(maxPayout, "maxPayout");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    Objects.requireNonNull(createdAt, "createdAt");
+    if (reference == null || reference.isBlank()) {
+      throw new IllegalArgumentException("reference must not be blank");
+    }
+    if (!SHA_256
+        .matcher(Objects.requireNonNull(requestFingerprint, "requestFingerprint"))
+        .matches()) {
+      throw new IllegalArgumentException("requestFingerprint must be lowercase SHA-256");
+    }
+    if (stake.currency() != maxPayout.currency()) {
+      throw new IllegalArgumentException("stake and max payout currencies differ");
+    }
+  }
+}


