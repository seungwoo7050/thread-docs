## `test(odds): verify slippage boundary`

diff --git a/src/test/java/com/sportsbook/betting/validation/OddsSlippageCheckerTest.java b/src/test/java/com/sportsbook/betting/validation/OddsSlippageCheckerTest.java
new file mode 100644
index 0000000..1b2cb9a
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/validation/OddsSlippageCheckerTest.java
@@ -0,0 +1,34 @@
+package com.sportsbook.betting.validation;
+
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.OddsDriftException;
+import com.sportsbook.betting.policy.BettingPolicyProperties;
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class OddsSlippageCheckerTest {
+
+  @Test
+  void acceptsBoundaryAndRejectsWorsePrice() {
+    BetLeg leg =
+        BetLeg.create(
+            UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.00"));
+    OddsSnapshotReader snapshots = mock(OddsSnapshotReader.class);
+    BettingPolicyProperties policy = new BettingPolicyProperties(0, null, null, null, null);
+    OddsSlippageChecker checker = new OddsSlippageChecker(snapshots, policy);
+
+    when(snapshots.currentOdds(leg)).thenReturn(new BigDecimal("1.94"));
+    assertThatCode(() -> checker.check(List.of(leg))).doesNotThrowAnyException();
+
+    when(snapshots.currentOdds(leg)).thenReturn(new BigDecimal("1.9399"));
+    assertThatThrownBy(() -> checker.check(List.of(leg))).isInstanceOf(OddsDriftException.class);
+  }
+}


## `feat(placement): assemble validated pending bets`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetAssembler.java b/src/main/java/com/sportsbook/betting/placement/BetAssembler.java
new file mode 100644
index 0000000..98f83ea
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/BetAssembler.java
@@ -0,0 +1,70 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.SystemBetCalculator;
+import com.sportsbook.betting.infrastructure.id.BetReferenceGenerator;
+import com.sportsbook.betting.infrastructure.id.UuidV7;
+import com.sportsbook.betting.validation.BetSlipValidator;
+import com.sportsbook.betting.validation.OddsSlippageChecker;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Clock;
+import java.time.Instant;
+import java.util.List;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetAssembler {
+
+  private final BetSlipValidator validator;
+  private final OddsSlippageChecker slippage;
+  private final SystemBetCalculator calculator;
+  private final BetReferenceGenerator references;
+  private final Clock clock;
+
+  public BetAssembler(
+      BetSlipValidator validator,
+      OddsSlippageChecker slippage,
+      SystemBetCalculator calculator,
+      BetReferenceGenerator references,
+      Clock clock) {
+    this.validator = validator;
+    this.slippage = slippage;
+    this.calculator = calculator;
+    this.references = references;
+    this.clock = clock;
+  }
+
+  public Bet assemble(PlaceBetCommand command, String fingerprint) {
+    List<BetLeg> legs =
+        command.selections().stream()
+            .map(
+                input ->
+                    BetLeg.create(
+                        input.eventId(),
+                        input.marketId(),
+                        input.selectionId(),
+                        input.oddsAtSubmission()))
+            .toList();
+    validator.validate(command.slipType(), legs);
+    validator.validateStake(command.unitStake());
+    slippage.check(legs);
+    List<Odds> odds = legs.stream().map(BetLeg::oddsAtSubmission).toList();
+    Money maxPayout = calculator.maxPayout(command.slipType(), command.unitStake(), odds);
+    Instant now = clock.instant();
+    BetDraft draft =
+        new BetDraft(
+            UuidV7.generate(),
+            command.userId(),
+            references.next(now),
+            command.slipType(),
+            command.unitStake(),
+            maxPayout,
+            command.idempotencyKey(),
+            fingerprint,
+            now);
+    return Bet.pending(draft, legs);
+  }
+}


## `test(placement): verify pending bet assembly`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetAssemblerTest.java b/src/test/java/com/sportsbook/betting/placement/BetAssemblerTest.java
new file mode 100644
index 0000000..900e070
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetAssemblerTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.PlacementPhase;
+import com.sportsbook.betting.domain.SystemBetCalculator;
+import com.sportsbook.betting.infrastructure.id.BetReferenceGenerator;
+import com.sportsbook.betting.policy.BettingPolicyProperties;
+import com.sportsbook.betting.validation.BetSlipValidator;
+import com.sportsbook.betting.validation.OddsSlippageChecker;
+import com.sportsbook.betting.validation.OddsSnapshotReader;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetAssemblerTest {
+
+  @Test
+  void validatesAndFreezesOnePendingAggregate() {
+    Instant now = Instant.parse("2026-08-22T00:00:00Z");
+    BettingPolicyProperties policy = new BettingPolicyProperties(0, null, null, null, null);
+    OddsSnapshotReader snapshots =
+        new OddsSnapshotReader(null) {
+          @Override
+          public BigDecimal currentOdds(BetLeg ignored) {
+            return new BigDecimal("2.00");
+          }
+        };
+    BetAssembler assembler =
+        new BetAssembler(
+            new BetSlipValidator(policy),
+            new OddsSlippageChecker(snapshots, policy),
+            new SystemBetCalculator(),
+            new BetReferenceGenerator(),
+            Clock.fixed(now, ZoneOffset.UTC));
+    PlaceBetCommand command =
+        new PlaceBetCommand(
+            UUID.randomUUID(),
+            new BetSlipType.Single(),
+            List.of(
+                new PlaceBetCommand.SelectionInput(
+                    UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"))),
+            Money.krw(1_000),
+            IdempotencyKey.of("request-1"));
+
+    Bet bet = assembler.assemble(command, "a".repeat(64));
+
+    assertThat(bet.placementPhase()).isEqualTo(PlacementPhase.CREATED);
+    assertThat(bet.maxPayout()).isEqualTo(Money.krw(2_000));
+    assertThat(bet.betReference()).startsWith("B-2026-08-22-");
+  }
+}
