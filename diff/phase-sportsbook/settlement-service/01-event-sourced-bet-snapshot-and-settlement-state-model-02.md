## `feat(selection): apply event outcome snapshots`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
index 3ef3d1f..0994541 100644
--- a/src/main/java/com/sportsbook/settlement/domain/Bet.java
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -1,5 +1,6 @@
 package com.sportsbook.settlement.domain;
 
+import com.sportsbook.protocol.domain.SettlementResult;
 import jakarta.persistence.CascadeType;
 import jakarta.persistence.Column;
 import jakarta.persistence.Embedded;
@@ -13,6 +14,7 @@ import java.time.Instant;
 import java.util.ArrayList;
 import java.util.Collections;
 import java.util.List;
+import java.util.Map;
 import java.util.Objects;
 import java.util.UUID;
 
@@ -115,4 +117,28 @@ public class Bet {
   public List<BetSelection> selections() {
     return Collections.unmodifiableList(selections);
   }
+
+  public boolean applySelectionSnapshot(
+      UUID eventId, Map<UUID, SettlementResult> outcomes, boolean clearMissing, Instant now) {
+    if (status != SettlementStatus.PENDING) {
+      throw new IllegalStateException("Cannot update selections after terminal settlement");
+    }
+    boolean changed = false;
+    for (BetSelection selection : selections) {
+      if (selection.eventId().equals(eventId)) {
+        SettlementResult replacement = outcomes.get(selection.selectionId());
+        if (replacement != null || clearMissing) {
+          changed |= selection.replaceOutcome(replacement);
+        }
+      }
+    }
+    if (changed) {
+      updatedAt = Objects.requireNonNull(now, "now");
+    }
+    return changed;
+  }
+
+  public boolean allSelectionsResolved() {
+    return !selections.isEmpty() && selections.stream().allMatch(s -> s.outcome() != null);
+  }
 }
diff --git a/src/main/java/com/sportsbook/settlement/domain/BetSelection.java b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
index f43b078..865f3ec 100644
--- a/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
+++ b/src/main/java/com/sportsbook/settlement/domain/BetSelection.java
@@ -1,9 +1,12 @@
 package com.sportsbook.settlement.domain;
 
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.value.Odds;
 import com.sportsbook.settlement.infrastructure.id.UuidV7;
 import jakarta.persistence.Column;
 import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
 import jakarta.persistence.FetchType;
 import jakarta.persistence.Id;
 import jakarta.persistence.JoinColumn;
@@ -41,6 +44,10 @@ public class BetSelection {
   @Column(name = "odds", nullable = false, precision = 9, scale = 4, updatable = false)
   private BigDecimal odds;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "outcome", length = 8)
+  private SettlementResult outcome;
+
   protected BetSelection() {}
 
   public BetSelection(UUID eventId, UUID marketId, UUID selectionId, Odds odds) {
@@ -56,15 +63,39 @@ public class BetSelection {
     this.legIndex = index;
   }
 
-  public UUID selectionRowId() { return selectionRowId; }
+  boolean replaceOutcome(SettlementResult replacement) {
+    if (outcome == replacement) {
+      return false;
+    }
+    outcome = replacement;
+    return true;
+  }
 
-  public int legIndex() { return legIndex; }
+  public UUID selectionRowId() {
+    return selectionRowId;
+  }
 
-  public UUID eventId() { return eventId; }
+  public int legIndex() {
+    return legIndex;
+  }
 
-  public UUID marketId() { return marketId; }
+  public UUID eventId() {
+    return eventId;
+  }
 
-  public UUID selectionId() { return selectionId; }
+  public UUID marketId() {
+    return marketId;
+  }
 
-  public Odds odds() { return Odds.ofDecimal(odds); }
+  public UUID selectionId() {
+    return selectionId;
+  }
+
+  public Odds odds() {
+    return Odds.ofDecimal(odds);
+  }
+
+  public SettlementResult outcome() {
+    return outcome;
+  }
 }


## `feat(correction): track bet revision numbers`

diff --git a/src/main/java/com/sportsbook/settlement/domain/Bet.java b/src/main/java/com/sportsbook/settlement/domain/Bet.java
index 736591b..16722c6 100644
--- a/src/main/java/com/sportsbook/settlement/domain/Bet.java
+++ b/src/main/java/com/sportsbook/settlement/domain/Bet.java
@@ -72,6 +72,9 @@ public class Bet {
   @Column(name = "settled_at")
   private Instant settledAt;
 
+  @Column(name = "revision_number", nullable = false)
+  private long revisionNumber;
+
   @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
   @OrderBy("legIndex ASC")
   private List<BetSelection> selections = new ArrayList<>();
@@ -156,6 +159,10 @@ public class Bet {
     return settledAt;
   }
 
+  public long revisionNumber() {
+    return revisionNumber;
+  }
+
   public List<BetSelection> selections() {
     return Collections.unmodifiableList(selections);
   }
@@ -192,6 +199,23 @@ public class Bet {
     recordTerminal(SettlementStatus.VOIDED, SettlementResult.VOID, refund, now);
   }
 
+  public long recordRevision(SettlementResult replacement, Money payout, Instant now) {
+    if (status != SettlementStatus.SETTLED) {
+      throw new IllegalStateException("Only settled bets can be revised");
+    }
+    Objects.requireNonNull(payout, "payout");
+    if (payout.isNegative() || payout.currency() != stake.toMoney().currency()) {
+      throw new IllegalArgumentException("Revision payout must use the stake currency");
+    }
+    revisionNumber = Math.incrementExact(revisionNumber);
+    result = Objects.requireNonNull(replacement, "replacement");
+    payoutAmount = payout.amount();
+    payoutCurrency = payout.currency();
+    settledAt = Objects.requireNonNull(now, "now");
+    updatedAt = now;
+    return revisionNumber;
+  }
+
   private void recordTerminal(
       SettlementStatus target, SettlementResult outcome, Money payout, Instant now) {
     if (status != SettlementStatus.PENDING) {


## `fix(persistence): validate mapped schema at startup`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index a14aa8e..a94dcf4 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -1,4 +1,7 @@
 spring:
+  jpa:
+    hibernate:
+      ddl-auto: validate
   kafka:
     consumer:
       group-id: settlement-service


## `test(bet): distinguish result void from whole slip void`

diff --git a/src/test/java/com/sportsbook/settlement/domain/BetTerminalOutcomeTest.java b/src/test/java/com/sportsbook/settlement/domain/BetTerminalOutcomeTest.java
new file mode 100644
index 0000000..813d93a
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/domain/BetTerminalOutcomeTest.java
@@ -0,0 +1,67 @@
+package com.sportsbook.settlement.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+import static org.assertj.core.api.Assertions.assertThatIllegalStateException;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetTerminalOutcomeTest {
+
+  @Test
+  void recordsResultVoidAsSettledRatherThanLifecycleVoided() {
+    Bet bet = pending();
+    Instant now = Instant.parse("2026-01-01T00:00:00Z");
+
+    bet.recordSettled(SettlementResult.VOID, Money.krw(100), now);
+
+    assertThat(bet.status()).isEqualTo(SettlementStatus.SETTLED);
+    assertThat(bet.result()).isEqualTo(SettlementResult.VOID);
+    assertThat(bet.payout()).isEqualTo(Money.krw(100));
+    assertThat(bet.settledAt()).isEqualTo(now);
+  }
+
+  @Test
+  void reservesVoidedStateForWholeSlipVoid() {
+    Bet bet = pending();
+
+    bet.recordVoided(Money.krw(100), Instant.EPOCH);
+
+    assertThat(bet.status()).isEqualTo(SettlementStatus.VOIDED);
+    assertThatIllegalStateException()
+        .isThrownBy(() -> bet.recordSettled(SettlementResult.WON, Money.krw(200), Instant.EPOCH));
+  }
+
+  @Test
+  void rejectsNegativeOrCrossCurrencyPayouts() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () -> pending().recordSettled(SettlementResult.WON, Money.krw(-1), Instant.EPOCH));
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () -> pending().recordSettled(SettlementResult.WON, Money.usd(1), Instant.EPOCH));
+  }
+
+  private static Bet pending() {
+    BetSelection leg =
+        new BetSelection(
+            UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.0000"));
+    return Bet.pending(
+        UUID.randomUUID(),
+        UUID.randomUUID(),
+        SlipKind.SINGLE,
+        null,
+        null,
+        new EmbeddedMoney(100, Currency.KRW),
+        Instant.EPOCH,
+        List.of(leg),
+        Instant.EPOCH);
+  }
+}
