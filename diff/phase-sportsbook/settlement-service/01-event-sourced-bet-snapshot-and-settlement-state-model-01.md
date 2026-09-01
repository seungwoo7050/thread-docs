# 이벤트 소싱 베팅 스냅샷과 정산 상태 모델

## `docs(readme): define settlement ownership`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..a2a0710
--- /dev/null
+++ b/README.md
@@ -0,0 +1,3 @@
+# Settlement Service
+
+Owns durable bet settlement, lifecycle voiding, corrected-result revisions, and publication of terminal settlement events.


## `build(flyway): add V1 bet read model`

diff --git a/pom.xml b/pom.xml
index 7da96a0..b605e3f 100644
--- a/pom.xml
+++ b/pom.xml
@@ -34,6 +34,19 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-data-jpa</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.flywaydb</groupId>
+            <artifactId>flyway-core</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.postgresql</groupId>
+            <artifactId>postgresql</artifactId>
+            <scope>runtime</scope>
+        </dependency>
         <dependency>
             <groupId>com.sportsbook</groupId>
             <artifactId>shared-protocol</artifactId>
diff --git a/src/main/resources/db/migration/V1__bet_read_model.sql b/src/main/resources/db/migration/V1__bet_read_model.sql
new file mode 100644
index 0000000..9bfcc88
--- /dev/null
+++ b/src/main/resources/db/migration/V1__bet_read_model.sql
@@ -0,0 +1,52 @@
+-- V1: settlement-service read model (ADR-0006 event sourcing).
+--
+-- settlement-service does NOT read the betting-service DB. It rebuilds the bet
+-- snapshot it needs from the BetPlacedRequested event (topic bet.placed.v1) and
+-- tracks each bet's settlement lifecycle here. A bet row is created PENDING on
+-- BetPlacedRequested and transitions to SETTLED / VOIDED once its events resolve
+-- (ADR-0013 status: settlement uses PENDING / SETTLED / VOIDED).
+
+CREATE TABLE bet (
+    bet_id                  UUID                     PRIMARY KEY,
+    user_id                 UUID                     NOT NULL,
+    slip_type               VARCHAR(16)              NOT NULL,   -- SINGLE / MULTIPLE / SYSTEM
+    system_min_wins         INT,                                 -- K, SYSTEM slips only
+    system_total_selections INT,                                 -- N, SYSTEM slips only
+    stake_amount            BIGINT                   NOT NULL,   -- per-line (unit) stake, minor units
+    stake_currency          VARCHAR(3)               NOT NULL,
+    status                  VARCHAR(16)              NOT NULL,   -- PENDING / SETTLED / VOIDED
+    result                  VARCHAR(8),                          -- WON / LOST / PUSH / VOID (when SETTLED)
+    payout_amount           BIGINT,                              -- credited amount (when terminal)
+    payout_currency         VARCHAR(3),
+    requested_at            TIMESTAMP WITH TIME ZONE NOT NULL,   -- carried from the event
+    settled_at              TIMESTAMP WITH TIME ZONE,            -- terminal transition time
+    created_at              TIMESTAMP WITH TIME ZONE NOT NULL,
+    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL
+);
+
+COMMENT ON TABLE  bet            IS 'Event-sourced bet snapshot + settlement lifecycle. Rebuilt from BetPlacedRequested; never read from betting-service DB.';
+COMMENT ON COLUMN bet.slip_type  IS 'BetSlipType tag (ADR-0008). SYSTEM also sets system_min_wins (K) + system_total_selections (N).';
+COMMENT ON COLUMN bet.stake_amount IS 'Per-line (unit) stake. Total committed = unit * line count; settlement re-derives line count from slip_type.';
+COMMENT ON COLUMN bet.status     IS 'Settlement lifecycle: PENDING (awaiting results) -> SETTLED | VOIDED. Distinct from betting-service BetStatus.';
+
+CREATE TABLE bet_selection (
+    selection_row_id UUID         PRIMARY KEY,
+    bet_id           UUID         NOT NULL REFERENCES bet (bet_id),
+    leg_index        INT          NOT NULL,
+    event_id         UUID         NOT NULL,
+    market_id        UUID         NOT NULL,
+    selection_id     UUID         NOT NULL,
+    odds             NUMERIC(9,4) NOT NULL,   -- oddsAtSubmission; V1 uses submission odds for payout
+    outcome          VARCHAR(8),              -- per-selection WON / LOST / PUSH / VOID (set at settle time)
+    CONSTRAINT uq_bet_selection_order UNIQUE (bet_id, leg_index)
+);
+
+COMMENT ON COLUMN bet_selection.odds    IS 'Decimal odds from BetPlacedRequested.oddsAtSubmission (scale 4). V1 has no accepted-odds field; submission odds drive payout.';
+COMMENT ON COLUMN bet_selection.outcome IS 'Per-selection result read from MatchResult.resultDetail at settle time. NULL until that selection''s event resolves.';
+
+-- Settlement scans bets by the event that just resolved ("all bets with a
+-- selection on event X"), so index the join column.
+CREATE INDEX ix_bet_selection_event ON bet_selection (event_id);
+
+-- Find still-open bets quickly (settlement trigger + late-window scans).
+CREATE INDEX ix_bet_pending ON bet (status) WHERE status = 'PENDING';


## `feat(money): persist shared monetary values`

diff --git a/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java b/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java
new file mode 100644
index 0000000..86c2185
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java
@@ -0,0 +1,50 @@
+package com.sportsbook.settlement.domain;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.Column;
+import jakarta.persistence.Embeddable;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import java.util.Objects;
+
+/** Persistence mirror that keeps the shared {@link Money} type free of JPA concerns. */
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
+  public EmbeddedMoney(long amount, Currency currency) {
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
+  @Override
+  public boolean equals(Object candidate) {
+    return this == candidate
+        || candidate instanceof EmbeddedMoney other
+            && amount == other.amount
+            && currency == other.currency;
+  }
+
+  @Override
+  public int hashCode() {
+    return Objects.hash(amount, currency);
+  }
+}


## `feat(state): define settlement lifecycle states`

diff --git a/src/main/java/com/sportsbook/settlement/domain/SettlementStatus.java b/src/main/java/com/sportsbook/settlement/domain/SettlementStatus.java
new file mode 100644
index 0000000..e1afebc
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/domain/SettlementStatus.java
@@ -0,0 +1,12 @@
+package com.sportsbook.settlement.domain;
+
+/** Settlement-owned lifecycle, separate from betting placement state. */
+public enum SettlementStatus {
+  PENDING,
+  SETTLED,
+  VOIDED;
+
+  public boolean isTerminal() {
+    return this != PENDING;
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/domain/SlipKind.java b/src/main/java/com/sportsbook/settlement/domain/SlipKind.java
new file mode 100644
index 0000000..0a5c2ab
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/domain/SlipKind.java
@@ -0,0 +1,30 @@
+package com.sportsbook.settlement.domain;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.util.Objects;
+
+/** Flat persistence discriminator for the shared sealed slip shape. */
+public enum SlipKind {
+  SINGLE,
+  MULTIPLE,
+  SYSTEM;
+
+  public static SlipKind from(BetSlipType type) {
+    Objects.requireNonNull(type, "type");
+    if (type instanceof BetSlipType.Single) {
+      return SINGLE;
+    }
+    if (type instanceof BetSlipType.Multiple) {
+      return MULTIPLE;
+    }
+    return SYSTEM;
+  }
+
+  public BetSlipType toProtocol(Integer minimumWins, Integer totalSelections) {
+    return switch (this) {
+      case SINGLE -> new BetSlipType.Single();
+      case MULTIPLE -> new BetSlipType.Multiple();
+      case SYSTEM -> new BetSlipType.System(minimumWins, totalSelections);
+    };
+  }
+}


## `feat(selection): model ordered selection identity`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
new file mode 100644
index 0000000..7a79689
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -0,0 +1,28 @@
+package com.sportsbook.settlement.domain;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Aggregate root introduced first as the owner of selection identity. */
+@Entity
+@Table(name = "bet")
+public class Bet {
+
+  @Id
+  @Column(name = "bet_id", nullable = false, updatable = false)
+  private UUID betId;
+
+  protected Bet() {}
+
+  Bet(UUID betId) {
+    this.betId = Objects.requireNonNull(betId, "betId");
+  }
+
+  public UUID betId() {
+    return betId;
+  }
+}
diff --git a/src/main/java/com/sportsbook/settlement/domain/BetSelection.java b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
new file mode 100644
index 0000000..f43b078
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
@@ -0,0 +1,70 @@
+package com.sportsbook.settlement.domain;
+
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.settlement.infrastructure.id.UuidV7;
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
+/** Immutable placement identity for one ordered bet leg. */
+@Entity
+@Table(name = "bet_selection")
+public class BetSelection {
+
+  @Id
+  @Column(name = "selection_row_id", nullable = false, updatable = false)
+  private UUID selectionRowId;
+
+  @ManyToOne(fetch = FetchType.LAZY, optional = false)
+  @JoinColumn(name = "bet_id", nullable = false)
+  private Bet bet;
+
+  @Column(name = "leg_index", nullable = false, updatable = false)
+  private int legIndex;
+
+  @Column(name = "event_id", nullable = false, updatable = false)
+  private UUID eventId;
+
+  @Column(name = "market_id", nullable = false, updatable = false)
+  private UUID marketId;
+
+  @Column(name = "selection_id", nullable = false, updatable = false)
+  private UUID selectionId;
+
+  @Column(name = "odds", nullable = false, precision = 9, scale = 4, updatable = false)
+  private BigDecimal odds;
+
+  protected BetSelection() {}
+
+  public BetSelection(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
+    this.selectionRowId = UuidV7.generate();
+    this.eventId = Objects.requireNonNull(eventId, "eventId");
+    this.marketId = Objects.requireNonNull(marketId, "marketId");
+    this.selectionId = Objects.requireNonNull(selectionId, "selectionId");
+    this.odds = Objects.requireNonNull(odds, "odds").decimal();
+  }
+
+  void attach(Bet owner, int index) {
+    this.bet = Objects.requireNonNull(owner, "owner");
+    this.legIndex = index;
+  }
+
+  public UUID selectionRowId() { return selectionRowId; }
+
+  public int legIndex() { return legIndex; }
+
+  public UUID eventId() { return eventId; }
+
+  public UUID marketId() { return marketId; }
+
+  public UUID selectionId() { return selectionId; }
+
+  public Odds odds() { return Odds.ofDecimal(odds); }
+}


## `feat(bet): create pending placement aggregate`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
index 7a79689..3ef3d1f 100644
--- a/src/main/java/com/sportsbook/settlement/domain/Bet.java
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -1,13 +1,22 @@
 package com.sportsbook.settlement.domain;
 
+import jakarta.persistence.CascadeType;
 import jakarta.persistence.Column;
+import jakarta.persistence.Embedded;
 import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
 import jakarta.persistence.Id;
+import jakarta.persistence.OneToMany;
 import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.ArrayList;
+import java.util.Collections;
+import java.util.List;
 import java.util.Objects;
 import java.util.UUID;
 
-/** Aggregate root introduced first as the owner of selection identity. */
+/** Pending read-model aggregate reconstructed without reading betting storage. */
 @Entity
 @Table(name = "bet")
 public class Bet {
@@ -16,13 +25,94 @@ public class Bet {
   @Column(name = "bet_id", nullable = false, updatable = false)
   private UUID betId;
 
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "slip_type", nullable = false, updatable = false)
+  private SlipKind slipKind;
+
+  @Column(name = "system_min_wins", updatable = false)
+  private Integer systemMinimumWins;
+
+  @Column(name = "system_total_selections", updatable = false)
+  private Integer systemTotalSelections;
+
+  @Embedded private EmbeddedMoney stake;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "status", nullable = false)
+  private SettlementStatus status;
+
+  @Column(name = "requested_at", nullable = false, updatable = false)
+  private Instant requestedAt;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "updated_at", nullable = false)
+  private Instant updatedAt;
+
+  @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
+  private List<BetSelection> selections = new ArrayList<>();
+
   protected Bet() {}
 
-  Bet(UUID betId) {
+  private Bet(
+      UUID betId,
+      UUID userId,
+      SlipKind slipKind,
+      Integer minimumWins,
+      Integer totalSelections,
+      EmbeddedMoney stake,
+      Instant requestedAt,
+      Instant now) {
     this.betId = Objects.requireNonNull(betId, "betId");
+    this.userId = Objects.requireNonNull(userId, "userId");
+    this.slipKind = Objects.requireNonNull(slipKind, "slipKind");
+    this.systemMinimumWins = minimumWins;
+    this.systemTotalSelections = totalSelections;
+    this.stake = Objects.requireNonNull(stake, "stake");
+    this.requestedAt = Objects.requireNonNull(requestedAt, "requestedAt");
+    this.createdAt = Objects.requireNonNull(now, "now");
+    this.updatedAt = now;
+    this.status = SettlementStatus.PENDING;
+  }
+
+  public static Bet pending(
+      UUID betId,
+      UUID userId,
+      SlipKind slipKind,
+      Integer minimumWins,
+      Integer totalSelections,
+      EmbeddedMoney stake,
+      Instant requestedAt,
+      List<BetSelection> selections,
+      Instant now) {
+    Bet bet =
+        new Bet(betId, userId, slipKind, minimumWins, totalSelections, stake, requestedAt, now);
+    selections.forEach(
+        selection -> {
+          selection.attach(bet, bet.selections.size());
+          bet.selections.add(selection);
+        });
+    bet.slipKind.toProtocol(minimumWins, totalSelections);
+    return bet;
   }
 
   public UUID betId() {
     return betId;
   }
+
+  public SettlementStatus status() {
+    return status;
+  }
+
+  public com.sportsbook.protocol.domain.BetSlipType slipType() {
+    return slipKind.toProtocol(systemMinimumWins, systemTotalSelections);
+  }
+
+  public List<BetSelection> selections() {
+    return Collections.unmodifiableList(selections);
+  }
 }
diff --git a/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java b/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java
index 86c2185..c99a0e0 100644
--- a/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java
+++ b/src/main/java/com/sportsbook/settlement/domain/EmbeddedMoney.java
@@ -12,11 +12,11 @@ import java.util.Objects;
 @Embeddable
 public class EmbeddedMoney {
 
-  @Column(nullable = false)
+  @Column(name = "stake_amount", nullable = false)
   private long amount;
 
   @Enumerated(EnumType.STRING)
-  @Column(nullable = false, length = 3)
+  @Column(name = "stake_currency", nullable = false, length = 3)
   private Currency currency;
 
   protected EmbeddedMoney() {}


## `feat(bet): record terminal base outcomes`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
index 0994541..8602533 100644
--- a/src/main/java/com/sportsbook/settlement/domain/Bet.java
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -1,6 +1,8 @@
 package com.sportsbook.settlement.domain;
 
 import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
 import jakarta.persistence.CascadeType;
 import jakarta.persistence.Column;
 import jakarta.persistence.Embedded;
@@ -55,6 +57,20 @@ public class Bet {
   @Column(name = "updated_at", nullable = false)
   private Instant updatedAt;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "result")
+  private SettlementResult result;
+
+  @Column(name = "payout_amount")
+  private Long payoutAmount;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "payout_currency")
+  private Currency payoutCurrency;
+
+  @Column(name = "settled_at")
+  private Instant settledAt;
+
   @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
   private List<BetSelection> selections = new ArrayList<>();
 
@@ -106,6 +122,10 @@ public class Bet {
     return betId;
   }
 
+  public UUID userId() {
+    return userId;
+  }
+
   public SettlementStatus status() {
     return status;
   }
@@ -114,6 +134,26 @@ public class Bet {
     return slipKind.toProtocol(systemMinimumWins, systemTotalSelections);
   }
 
+  public Money stake() {
+    return stake.toMoney();
+  }
+
+  public Instant requestedAt() {
+    return requestedAt;
+  }
+
+  public SettlementResult result() {
+    return result;
+  }
+
+  public Money payout() {
+    return payoutAmount == null ? null : new Money(payoutAmount, payoutCurrency);
+  }
+
+  public Instant settledAt() {
+    return settledAt;
+  }
+
   public List<BetSelection> selections() {
     return Collections.unmodifiableList(selections);
   }
@@ -141,4 +181,29 @@ public class Bet {
   public boolean allSelectionsResolved() {
     return !selections.isEmpty() && selections.stream().allMatch(s -> s.outcome() != null);
   }
+
+  public void recordSettled(SettlementResult outcome, Money payout, Instant now) {
+    recordTerminal(SettlementStatus.SETTLED, outcome, payout, now);
+  }
+
+  public void recordVoided(Money refund, Instant now) {
+    recordTerminal(SettlementStatus.VOIDED, SettlementResult.VOID, refund, now);
+  }
+
+  private void recordTerminal(
+      SettlementStatus target, SettlementResult outcome, Money payout, Instant now) {
+    if (status != SettlementStatus.PENDING) {
+      throw new IllegalStateException("Bet is already terminal: " + status);
+    }
+    Objects.requireNonNull(payout, "payout");
+    if (payout.amount() < 0 || payout.currency() != stake.toMoney().currency()) {
+      throw new IllegalArgumentException("Payout must be nonnegative and use the stake currency");
+    }
+    status = target;
+    result = Objects.requireNonNull(outcome, "outcome");
+    payoutAmount = payout.amount();
+    payoutCurrency = payout.currency();
+    settledAt = Objects.requireNonNull(now, "now");
+    updatedAt = now;
+  }
 }


