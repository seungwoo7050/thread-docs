# 시스템 베팅 조합과 금액 의미론

## `feat(domain): calculate system exposure`

diff --git a/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java b/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java
new file mode 100644
index 0000000..96cf524
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java
@@ -0,0 +1,35 @@
+package com.sportsbook.betting.domain;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SystemBetCalculator {
+
+  public int lineCount(BetSlipType type, int legCount) {
+    if (type instanceof BetSlipType.System system) {
+      if (system.totalSelections() != legCount) {
+        throw new IllegalArgumentException("SYSTEM totalSelections must equal leg count");
+      }
+      return Math.toIntExact(binomial(legCount, system.minWins()));
+    }
+    return 1;
+  }
+
+  public Money totalStake(BetSlipType type, Money unitStake, int legCount) {
+    return unitStake.multiply(lineCount(type, legCount));
+  }
+
+  static long binomial(int n, int k) {
+    if (k < 0 || k > n) {
+      return 0;
+    }
+    int smaller = Math.min(k, n - k);
+    long result = 1;
+    for (int index = 0; index < smaller; index++) {
+      result = result * (n - index) / (index + 1);
+    }
+    return result;
+  }
+}


## `test(domain): verify system exposure`

diff --git a/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java b/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java
new file mode 100644
index 0000000..254199a
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import org.junit.jupiter.api.Test;
+
+class SystemBetCalculatorTest {
+
+  private final SystemBetCalculator calculator = new SystemBetCalculator();
+
+  @Test
+  void multipliesUnitStakeByCombinationCount() {
+    BetSlipType.System system = new BetSlipType.System(2, 4);
+
+    assertThat(calculator.lineCount(system, 4)).isEqualTo(6);
+    assertThat(calculator.totalStake(system, Money.krw(1_000), 4))
+        .isEqualTo(Money.krw(6_000));
+  }
+}


## `feat(domain): calculate maximum system payout`

diff --git a/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java b/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java
index 96cf524..754692b 100644
--- a/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java
+++ b/src/main/java/com/sportsbook/betting/domain/SystemBetCalculator.java
@@ -2,6 +2,12 @@ package com.sportsbook.betting.domain;
 
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.math.BigDecimal;
+import java.math.RoundingMode;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.stream.IntStream;
 import org.springframework.stereotype.Component;
 
 @Component
@@ -21,6 +27,45 @@ public class SystemBetCalculator {
     return unitStake.multiply(lineCount(type, legCount));
   }
 
+  public Money maxPayout(BetSlipType type, Money unitStake, List<Odds> odds) {
+    BigDecimal sum = BigDecimal.ZERO;
+    for (List<Integer> line : lines(type, odds.size())) {
+      BigDecimal product = BigDecimal.ONE;
+      for (int index : line) {
+        product = product.multiply(odds.get(index).decimal());
+      }
+      sum = sum.add(product);
+    }
+    long amount =
+        BigDecimal.valueOf(unitStake.amount())
+            .multiply(sum)
+            .setScale(0, RoundingMode.FLOOR)
+            .longValueExact();
+    return new Money(amount, unitStake.currency());
+  }
+
+  private static List<List<Integer>> lines(BetSlipType type, int count) {
+    if (type instanceof BetSlipType.System system) {
+      List<List<Integer>> result = new ArrayList<>();
+      collect(0, count, system.minWins(), new ArrayList<>(), result);
+      return result;
+    }
+    return List.of(IntStream.range(0, count).boxed().toList());
+  }
+
+  private static void collect(
+      int start, int count, int size, List<Integer> current, List<List<Integer>> result) {
+    if (current.size() == size) {
+      result.add(List.copyOf(current));
+      return;
+    }
+    for (int index = start; index <= count - (size - current.size()); index++) {
+      current.add(index);
+      collect(index + 1, count, size, current, result);
+      current.remove(current.size() - 1);
+    }
+  }
+
   static long binomial(int n, int k) {
     if (k < 0 || k > n) {
       return 0;


## `test(domain): verify maximum system payout`

diff --git a/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java b/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java
index 254199a..867b10e 100644
--- a/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/SystemBetCalculatorTest.java
@@ -4,6 +4,8 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
 import org.junit.jupiter.api.Test;
 
 class SystemBetCalculatorTest {
@@ -15,7 +17,17 @@ class SystemBetCalculatorTest {
     BetSlipType.System system = new BetSlipType.System(2, 4);
 
     assertThat(calculator.lineCount(system, 4)).isEqualTo(6);
-    assertThat(calculator.totalStake(system, Money.krw(1_000), 4))
-        .isEqualTo(Money.krw(6_000));
+    assertThat(calculator.totalStake(system, Money.krw(1_000), 4)).isEqualTo(Money.krw(6_000));
+  }
+
+  @Test
+  void sumsWinningSystemLinesUsingUnitStake() {
+    Money payout =
+        calculator.maxPayout(
+            new BetSlipType.System(2, 3),
+            Money.krw(100),
+            List.of(Odds.ofDecimal("2.0"), Odds.ofDecimal("3.0"), Odds.ofDecimal("4.0")));
+
+    assertThat(payout).isEqualTo(Money.krw(2_600));
   }
 }


## `feat(placement): reserve exposure with durable proof`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
index 9d1bdef..1446ce4 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -1,12 +1,21 @@
 package com.sportsbook.betting.placement;
 
+import com.sportsbook.betting.client.RiskClient;
+import com.sportsbook.betting.client.RiskClient.Reservation;
 import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.PlacementPhase;
+import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DuplicateBetException;
 import com.sportsbook.betting.error.MarketClosedException;
 import com.sportsbook.betting.error.OddsDriftException;
+import com.sportsbook.betting.error.RiskLimitException;
 import com.sportsbook.betting.error.ValidationFailedException;
 import java.time.Clock;
+import java.util.List;
 import java.util.Optional;
+import java.util.UUID;
 import org.springframework.dao.DataIntegrityViolationException;
 import org.springframework.stereotype.Service;
 
@@ -14,11 +23,20 @@ import org.springframework.stereotype.Service;
 public class BetPlacementService {
 
   private final BetAssembler assembler;
+  private final RiskClient risk;
+  private final SystemBetCalculator calculator;
   private final BetStore store;
   private final Clock clock;
 
-  public BetPlacementService(BetAssembler assembler, BetStore store, Clock clock) {
+  public BetPlacementService(
+      BetAssembler assembler,
+      RiskClient risk,
+      SystemBetCalculator calculator,
+      BetStore store,
+      Clock clock) {
     this.assembler = assembler;
+    this.risk = risk;
+    this.calculator = calculator;
     this.store = store;
     this.clock = clock;
   }
@@ -61,7 +79,47 @@ public class BetPlacementService {
     } catch (DataIntegrityViolationException collision) {
       return replayKnown(key, command, fingerprint, collision);
     }
-    return bet;
+    return advance(bet.betId(), true);
+  }
+
+  private Bet advance(UUID betId, boolean surfaceRejection) {
+    Bet current = store.findById(betId);
+    if (current.placementPhase() == PlacementPhase.CREATED) {
+      reserveRisk(current, surfaceRejection);
+      return store.findById(betId);
+    }
+    return current;
+  }
+
+  private void reserveRisk(Bet bet, boolean surfaceRejection) {
+    try {
+      Reservation reservation =
+          risk.reserve(bet.betId(), bet.userId(), totalExposure(bet), selectionIds(bet.legs()));
+      store.recordRiskReservation(
+          bet.betId(),
+          reservation.expiresAt(),
+          reservation.token(),
+          reservation.alreadyCommitted(),
+          clock.instant());
+    } catch (RiskLimitException | DuplicateBetException | ValidationFailedException rejection) {
+      Bet rejected =
+          store.rejectAtCreation(
+              bet.betId(), rejection.errorCode(), rejection.getMessage(), clock.instant());
+      if (surfaceRejection) {
+        throw rejection;
+      }
+      if (rejected.status() == com.sportsbook.protocol.domain.BetStatus.PENDING) {
+        throw new IllegalStateException("Risk rejection was not persisted");
+      }
+    }
+  }
+
+  private com.sportsbook.protocol.value.Money totalExposure(Bet bet) {
+    return calculator.totalStake(bet.slipType(), bet.stake(), bet.legs().size());
+  }
+
+  private static List<UUID> selectionIds(List<BetLeg> legs) {
+    return legs.stream().map(BetLeg::selectionId).toList();
   }
 
   private Bet replayKnown(


## `feat(wallet): debit full exposure idempotently`

diff --git a/src/main/java/com/sportsbook/betting/client/WalletClient.java b/src/main/java/com/sportsbook/betting/client/WalletClient.java
new file mode 100644
index 0000000..a1190e0
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/client/WalletClient.java
@@ -0,0 +1,85 @@
+package com.sportsbook.betting.client;
+
+import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DependencyUnavailableException;
+import com.sportsbook.betting.error.WalletProofMismatchException;
+import com.sportsbook.betting.error.WalletRejectedException;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.MediaType;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+import org.springframework.web.client.RestClientException;
+
+@Component
+public class WalletClient {
+
+  private static final String DEBIT = "/internal/v1/wallet/transactions/debit";
+  private static final String DEBIT_REASON = "BET_DEBIT";
+  private static final String REFUND_REASON = "BET_REFUND";
+
+  private final RestClient http;
+  private final WalletProblemMapper problems;
+
+  public WalletClient(
+      @Qualifier("walletRestClient") RestClient http, WalletProblemMapper problems) {
+    this.http = http;
+    this.problems = problems;
+  }
+
+  public UUID debit(UUID betId, UUID userId, Money fullExposure) {
+    try {
+      WalletOperationResponse response =
+          http.post()
+              .uri(DEBIT)
+              .header("Idempotency-Key", betId.toString())
+              .contentType(MediaType.APPLICATION_JSON)
+              .body(new WalletDebitRequest(userId, fullExposure))
+              .retrieve()
+              .onStatus(
+                  HttpStatusCode::is4xxClientError,
+                  (request, error) -> {
+                    throw problems.map(problems.read(error));
+                  })
+              .body(WalletOperationResponse.class);
+      return requireDebitProof(response, userId, fullExposure);
+    } catch (BetPlacementException exception) {
+      throw exception;
+    } catch (RestClientException exception) {
+      throw new DependencyUnavailableException("Wallet debit is unavailable", exception);
+    }
+  }
+
+  private static UUID requireOperationId(WalletOperationResponse response) {
+    if (response == null || response.operationGroupId() == null) {
+      throw new DependencyUnavailableException("Wallet returned no operationGroupId");
+    }
+    return response.operationGroupId();
+  }
+
+  private static UUID requireProof(
+      WalletOperationResponse response, UUID userId, Money amount, String reason) {
+    if (response == null
+        || response.operationGroupId() == null
+        || !userId.equals(response.userId())
+        || !amount.equals(response.amount())
+        || !reason.equals(response.reason())
+        || response.at() == null) {
+      String operation = reason.equals(REFUND_REASON) ? "refund" : "debit";
+      throw new WalletProofMismatchException(operation);
+    }
+    return response.operationGroupId();
+  }
+
+  private static UUID requireDebitProof(
+      WalletOperationResponse response, UUID userId, Money amount) {
+    try {
+      return requireProof(response, userId, amount, DEBIT_REASON);
+    } catch (WalletProofMismatchException mismatch) {
+      throw new WalletRejectedException(
+          "WALLET_OPERATION_MISMATCH", "Wallet debit proof did not match this bet");
+    }
+  }
+}


## `feat(outbox): build unit-stake placement events`

diff --git a/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java b/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java
new file mode 100644
index 0000000..3412170
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java
@@ -0,0 +1,64 @@
+package com.sportsbook.betting.outbox;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.infrastructure.id.UuidV7;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.RequestedSelection;
+import java.time.Instant;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetEventFactory {
+
+  public OutboxEvent placedRequested(Bet bet, Instant now) {
+    BetSlipType type = bet.slipType();
+    BetPlacedRequested record =
+        BetPlacedRequested.newBuilder()
+            .setBetId(bet.betId().toString())
+            .setUserId(bet.userId().toString())
+            .setSlipType(tag(type))
+            .setSystemMinWins(type instanceof BetSlipType.System system ? system.minWins() : null)
+            .setSystemTotalSelections(
+                type instanceof BetSlipType.System system ? system.totalSelections() : null)
+            .setSelections(
+                bet.legs().stream()
+                    .map(
+                        leg ->
+                            RequestedSelection.newBuilder()
+                                .setEventId(leg.eventId().toString())
+                                .setMarketId(leg.marketId().toString())
+                                .setSelectionId(leg.selectionId().toString())
+                                .setOddsAtSubmission(
+                                    leg.oddsAtSubmission().decimal().toPlainString())
+                                .build())
+                    .toList())
+            .setStake(
+                com.sportsbook.protocol.event.Money.newBuilder()
+                    .setAmount(bet.stake().amount())
+                    .setCurrency(bet.stake().currency().name())
+                    .build())
+            .setIdempotencyKey(bet.idempotencyKey())
+            .setRequestedAt(bet.createdAt())
+            .build();
+    return OutboxEvent.pending(
+        UuidV7.generate(),
+        BettingTopics.BET_PLACED,
+        bet.userId().toString(),
+        BetPlacedRequested.getClassSchema().getName(),
+        AvroSerializer.serialize(record),
+        now);
+  }
+
+  private static BetSlipTypeTag tag(BetSlipType type) {
+    if (type instanceof BetSlipType.Single) {
+      return BetSlipTypeTag.SINGLE;
+    }
+    if (type instanceof BetSlipType.Multiple) {
+      return BetSlipTypeTag.MULTIPLE;
+    }
+    return BetSlipTypeTag.SYSTEM;
+  }
+}


## `test(outbox): lock system unit-stake contract`

diff --git a/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java b/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java
new file mode 100644
index 0000000..70a1998
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java
@@ -0,0 +1,81 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.MethodSource;
+
+class BetEventFactoryTest {
+
+  @Test
+  void publishesSystemUnitStakeRatherThanExposure() {
+    BetDraft draft =
+        new BetDraft(
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            "B-2026-08-22-00000000",
+            new BetSlipType.System(2, 3),
+            Money.krw(100),
+            Money.krw(2_600),
+            IdempotencyKey.of("request-1"),
+            "a".repeat(64),
+            Instant.EPOCH);
+    Bet bet = Bet.pending(draft, List.of(leg(), leg(), leg()));
+
+    OutboxEvent event = new BetEventFactory().placedRequested(bet, Instant.EPOCH);
+    BetPlacedRequested payload =
+        AvroSerializer.deserialize(event.payload(), BetPlacedRequested.class);
+
+    assertThat(payload.getStake().getAmount()).isEqualTo(100);
+    assertThat(payload.getSelections()).hasSize(3);
+    assertThat(event.partitionKey()).isEqualTo(bet.userId().toString());
+  }
+
+  @ParameterizedTest
+  @MethodSource("nonSystemTypes")
+  void omitsBothSystemFieldsForNonSystemSlips(BetSlipType type) {
+    List<BetLeg> legs = type instanceof BetSlipType.Single ? List.of(leg()) : List.of(leg(), leg());
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                UUID.randomUUID(),
+                UUID.randomUUID(),
+                "B-2026-08-22-00000001",
+                type,
+                Money.krw(100),
+                Money.krw(200),
+                IdempotencyKey.of("request-2"),
+                "b".repeat(64),
+                Instant.EPOCH),
+            legs);
+
+    BetPlacedRequested payload =
+        AvroSerializer.deserialize(
+            new BetEventFactory().placedRequested(bet, Instant.EPOCH).payload(),
+            BetPlacedRequested.class);
+
+    assertThat(payload.getSystemMinWins()).isNull();
+    assertThat(payload.getSystemTotalSelections()).isNull();
+  }
+
+  static List<BetSlipType> nonSystemTypes() {
+    return List.of(new BetSlipType.Single(), new BetSlipType.Multiple());
+  }
+
+  private BetLeg leg() {
+    return BetLeg.create(
+        UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.0"));
+  }
+}
