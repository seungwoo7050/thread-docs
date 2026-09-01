## `test(settlement): verify permanent projection conflicts`

diff --git a/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
index beed4a5..c7e4224 100644
--- a/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
@@ -1,9 +1,11 @@
 package com.sportsbook.betting.settlement;
 
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetLeg;
 import com.sportsbook.betting.domain.SystemBetCalculator;
@@ -86,6 +88,33 @@ class BetSettlementServiceTest {
     verify(bet).voidBase(eventId, VoidReason.EVENT_CANCELLED, Instant.EPOCH, "b".repeat(64));
   }
 
+  @Test
+  void classifiesAResolutionActorMismatchAsPermanent() {
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    when(bet.userId()).thenReturn(UUID.randomUUID());
+    when(bets.findLockedByBetId(betId)).thenReturn(Optional.of(bet));
+    BetSettled event =
+        BetSettled.newBuilder()
+            .setBetId(betId.toString())
+            .setUserId(UUID.randomUUID().toString())
+            .setEventId(UUID.randomUUID().toString())
+            .setResult(SettlementResultAvro.WON)
+            .setStake(eventMoney(1_000))
+            .setPayout(eventMoney(2_000))
+            .setSettledAt(Instant.EPOCH)
+            .setResultDetail(java.util.Map.of())
+            .build();
+
+    assertThatThrownBy(
+            () ->
+                new BetSettlementService(bets, new SystemBetCalculator())
+                    .apply(event, "c".repeat(64)))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("actor");
+  }
+
   private static com.sportsbook.protocol.event.Money eventMoney(long amount) {
     return com.sportsbook.protocol.event.Money.newBuilder()
         .setAmount(amount)


## `feat(settlement): count revision projection gaps`

diff --git a/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
index fdae84d..5227b46 100644
--- a/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
+++ b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
@@ -3,9 +3,12 @@ package com.sportsbook.betting.settlement;
 import com.sportsbook.betting.config.BettingTopics;
 import com.sportsbook.betting.config.KafkaMessageValidator;
 import com.sportsbook.betting.config.PermanentKafkaException;
+import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
 import java.security.MessageDigest;
 import java.security.NoSuchAlgorithmException;
 import java.util.HexFormat;
@@ -16,10 +19,14 @@ import org.springframework.stereotype.Component;
 @Component
 public class SettlementResultListener {
 
+  static final String REVISION_GAP_METRIC = "betting.resolution.revision.gaps";
+
   private final BetSettlementService settlement;
+  private final Counter revisionGaps;
 
-  public SettlementResultListener(BetSettlementService settlement) {
+  public SettlementResultListener(BetSettlementService settlement, MeterRegistry meters) {
     this.settlement = settlement;
+    this.revisionGaps = Counter.builder(REVISION_GAP_METRIC).register(meters);
   }
 
   @KafkaListener(
@@ -52,7 +59,10 @@ public class SettlementResultListener {
         BetResolutionRevised event =
             KafkaMessageValidator.decode(record.value(), BetResolutionRevised.class);
         KafkaMessageValidator.requireKey(record.key(), event.getBetId(), "Revision betId");
-        settlement.apply(event, hash);
+        Bet.RevisionApplyResult result = settlement.apply(event, hash);
+        if (result == Bet.RevisionApplyResult.APPLIED_WITH_GAP) {
+          revisionGaps.increment();
+        }
       }
       default -> throw new PermanentKafkaException("Unsupported resolution topic");
     }


## `test(settlement): verify revision gap metric`

diff --git a/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
index eec848f..280cd2f 100644
--- a/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
+++ b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
@@ -1,19 +1,23 @@
 package com.sportsbook.betting.settlement;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.ArgumentMatchers.anyString;
 import static org.mockito.ArgumentMatchers.eq;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
 
 import com.sportsbook.betting.config.BettingTopics;
 import com.sportsbook.betting.config.PermanentKafkaException;
+import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.outbox.AvroSerializer;
 import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import java.nio.charset.StandardCharsets;
 import java.time.Instant;
 import java.util.UUID;
@@ -29,7 +33,7 @@ class SettlementResultListenerTest {
     BetResolutionRevised event = revision();
     ConsumerRecord<byte[], byte[]> record = record(event, event.getBetId());
 
-    new SettlementResultListener(settlement).onResolution(record);
+    new SettlementResultListener(settlement, new SimpleMeterRegistry()).onResolution(record);
 
     verify(settlement).apply(eq(event), anyString());
   }
@@ -40,7 +44,10 @@ class SettlementResultListenerTest {
     BetResolutionRevised event = revision();
     ConsumerRecord<byte[], byte[]> record = record(event, UUID.randomUUID().toString());
 
-    assertThatThrownBy(() -> new SettlementResultListener(settlement).onResolution(record))
+    assertThatThrownBy(
+            () ->
+                new SettlementResultListener(settlement, new SimpleMeterRegistry())
+                    .onResolution(record))
         .isInstanceOf(PermanentKafkaException.class)
         .hasMessageContaining("Kafka key");
     verifyNoInteractions(settlement);
@@ -57,7 +64,10 @@ class SettlementResultListenerTest {
             UUID.randomUUID().toString().getBytes(StandardCharsets.US_ASCII),
             null);
 
-    assertThatThrownBy(() -> new SettlementResultListener(settlement).onResolution(record))
+    assertThatThrownBy(
+            () ->
+                new SettlementResultListener(settlement, new SimpleMeterRegistry())
+                    .onResolution(record))
         .isInstanceOf(PermanentKafkaException.class)
         .hasMessageContaining("payload");
     verifyNoInteractions(settlement);
@@ -78,7 +88,7 @@ class SettlementResultListenerTest {
 
     assertThatThrownBy(
             () ->
-                new SettlementResultListener(settlement)
+                new SettlementResultListener(settlement, new SimpleMeterRegistry())
                     .onResolution(record(BettingTopics.BET_VOIDED, event, event.getEventId())))
         .isInstanceOf(PermanentKafkaException.class)
         .hasMessageContaining("settled VOID");
@@ -100,12 +110,33 @@ class SettlementResultListenerTest {
             .setResultDetail(java.util.Map.of("reason", "MARKET_VOID"))
             .build();
 
-    new SettlementResultListener(settlement)
+    new SettlementResultListener(settlement, new SimpleMeterRegistry())
         .onResolution(record(BettingTopics.BET_SETTLED, event, event.getEventId()));
 
     verify(settlement).apply(eq(event), anyString());
   }
 
+  @Test
+  void countsOnlyAppliedRevisionGaps() throws Exception {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    BetResolutionRevised event = revision();
+    ConsumerRecord<byte[], byte[]> record = record(event, event.getBetId());
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    when(settlement.apply(eq(event), anyString()))
+        .thenReturn(
+            Bet.RevisionApplyResult.APPLIED,
+            Bet.RevisionApplyResult.APPLIED_WITH_GAP,
+            Bet.RevisionApplyResult.DUPLICATE);
+    SettlementResultListener listener = new SettlementResultListener(settlement, meters);
+
+    listener.onResolution(record);
+    assertThat(meters.counter(SettlementResultListener.REVISION_GAP_METRIC).count()).isZero();
+    listener.onResolution(record);
+    listener.onResolution(record);
+
+    assertThat(meters.counter(SettlementResultListener.REVISION_GAP_METRIC).count()).isEqualTo(1);
+  }
+
   private static ConsumerRecord<byte[], byte[]> record(BetResolutionRevised event, String key) {
     return record(BettingTopics.BET_RESOLUTION_REVISED, event, key);
   }


## `feat(settlement): bind revisions to selected events`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 9d5b735..87e5a86 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -355,9 +355,13 @@ public class Bet {
     if (status != BetStatus.ACCEPTED && status != BetStatus.SETTLED) {
       throw new IllegalStateException("Revisions require ACCEPTED or SETTLED status");
     }
+    requireSelectionEvent(eventId);
     if (revisionNumber < 1) {
       throw new IllegalArgumentException("revisionNumber must be at least 1");
     }
+    if (previousPayout.currency() != stake.currency() || previousPayout.isNegative()) {
+      throw new IllegalArgumentException("Revision previous payout is invalid");
+    }
     long current = resolutionRevisionNumber == null ? 0 : resolutionRevisionNumber;
     if (revisionNumber < current) {
       return RevisionApplyResult.IGNORED;


## `test(settlement): reject untrusted revision snapshots`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index 3a7f025..3f0f118 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -256,6 +256,50 @@ class BetTest {
         .hasMessageContaining("Conflicting equal");
   }
 
+  @Test
+  void rejectsRevisionForAnUnselectedEvent() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+
+    assertThatThrownBy(
+            () ->
+                bet.applyRevision(
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    1,
+                    SettlementResult.LOST,
+                    SettlementResult.WON,
+                    Money.krw(0),
+                    Money.krw(2_000),
+                    NOW,
+                    NOW.plusSeconds(1),
+                    "a".repeat(64)))
+        .hasMessageContaining("selected leg");
+  }
+
+  @Test
+  void rejectsInvalidPreviousPayoutEvenAcrossAGap() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+    UUID eventId = bet.legs().get(0).eventId();
+
+    for (Money invalid :
+        List.of(Money.usd(0), new Money(-1, com.sportsbook.protocol.value.Currency.KRW))) {
+      assertThatThrownBy(
+              () ->
+                  bet.applyRevision(
+                      eventId,
+                      UUID.randomUUID(),
+                      2,
+                      SettlementResult.LOST,
+                      SettlementResult.WON,
+                      invalid,
+                      Money.krw(2_000),
+                      NOW,
+                      NOW.plusSeconds(1),
+                      "b".repeat(64)))
+          .hasMessageContaining("previous payout");
+    }
+  }
+
   static Bet.RevisionApplyResult revise(Bet bet, UUID revisionId, String hash) {
     return bet.applyRevision(
         bet.legs().get(0).eventId(),


## `feat(settlement): flush revision identity conflicts`

diff --git a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
index 42ee697..c12996e 100644
--- a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
+++ b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
@@ -12,6 +12,7 @@ import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
 import java.util.UUID;
+import org.springframework.dao.DataIntegrityViolationException;
 import org.springframework.stereotype.Service;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -70,18 +71,23 @@ public class BetSettlementService {
   public Bet.RevisionApplyResult apply(BetResolutionRevised event, String payloadHash) {
     try {
       Bet bet = owned(event.getBetId(), event.getUserId());
-      return bet.applyRevision(
-          canonical(event.getEventId()),
-          canonical(event.getRevisionId()),
-          event.getRevisionNumber(),
-          SettlementResult.valueOf(event.getPreviousResult().name()),
-          SettlementResult.valueOf(event.getNewResult().name()),
-          money(event.getPreviousPayout()),
-          money(event.getNewPayout()),
-          event.getSourceResultSettledAt(),
-          event.getRevisedAt(),
-          payloadHash);
-    } catch (IllegalArgumentException | IllegalStateException failure) {
+      Bet.RevisionApplyResult result =
+          bet.applyRevision(
+              canonical(event.getEventId()),
+              canonical(event.getRevisionId()),
+              event.getRevisionNumber(),
+              SettlementResult.valueOf(event.getPreviousResult().name()),
+              SettlementResult.valueOf(event.getNewResult().name()),
+              money(event.getPreviousPayout()),
+              money(event.getNewPayout()),
+              event.getSourceResultSettledAt(),
+              event.getRevisedAt(),
+              payloadHash);
+      bets.flush();
+      return result;
+    } catch (IllegalArgumentException
+        | IllegalStateException
+        | DataIntegrityViolationException failure) {
       throw permanent("Invalid resolution revision", failure);
     }
   }


## `test(settlement): isolate revision identity conflicts`

diff --git a/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
index c7e4224..37564b2 100644
--- a/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
@@ -13,6 +13,7 @@ import com.sportsbook.betting.domain.VoidReason;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
@@ -22,6 +23,7 @@ import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import org.springframework.dao.DataIntegrityViolationException;
 
 class BetSettlementServiceTest {
 
@@ -115,6 +117,41 @@ class BetSettlementServiceTest {
         .hasMessageContaining("actor");
   }
 
+  @Test
+  void flushesRevisionIdentityBeforeReturningSuccess() {
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    when(bet.userId()).thenReturn(userId);
+    when(bets.findLockedByBetId(betId)).thenReturn(Optional.of(bet));
+    org.mockito.Mockito.doThrow(new DataIntegrityViolationException("duplicate revision"))
+        .when(bets)
+        .flush();
+    BetResolutionRevised event =
+        BetResolutionRevised.newBuilder()
+            .setBetId(betId.toString())
+            .setUserId(userId.toString())
+            .setEventId(UUID.randomUUID().toString())
+            .setRevisionId(UUID.randomUUID().toString())
+            .setRevisionNumber(1L)
+            .setPreviousResult(SettlementResultAvro.LOST)
+            .setNewResult(SettlementResultAvro.WON)
+            .setPreviousPayout(eventMoney(0))
+            .setNewPayout(eventMoney(2_000))
+            .setSourceResultSettledAt(Instant.EPOCH)
+            .setRevisedAt(Instant.EPOCH.plusSeconds(1))
+            .build();
+
+    assertThatThrownBy(
+            () ->
+                new BetSettlementService(bets, new SystemBetCalculator())
+                    .apply(event, "d".repeat(64)))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("revision");
+    verify(bets).flush();
+  }
+
   private static com.sportsbook.protocol.event.Money eventMoney(long amount) {
     return com.sportsbook.protocol.event.Money.newBuilder()
         .setAmount(amount)


## `fix(settlement): validate revision chronology`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index a266166..f28cbc6 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -368,6 +368,11 @@ public class Bet {
     if (revisionNumber < 1) {
       throw new IllegalArgumentException("revisionNumber must be at least 1");
     }
+    sourceSettledAt = Objects.requireNonNull(sourceSettledAt, "sourceSettledAt");
+    revisedAt = Objects.requireNonNull(revisedAt, "revisedAt");
+    if (sourceSettledAt.isAfter(revisedAt)) {
+      throw new IllegalArgumentException("sourceSettledAt must not be after revisedAt");
+    }
     if (previousPayout.currency() != stake.currency() || previousPayout.isNegative()) {
       throw new IllegalArgumentException("Revision previous payout is invalid");
     }
@@ -399,8 +404,8 @@ public class Bet {
     this.resolutionRevisionId = Objects.requireNonNull(revisionId, "revisionId");
     this.resolutionRevisionNumber = revisionNumber;
     this.resolutionPayloadSha256 = requireHash(payloadHash);
-    this.sourceResultSettledAt = Objects.requireNonNull(sourceSettledAt, "sourceSettledAt");
-    this.resolvedAt = Objects.requireNonNull(revisedAt, "revisedAt");
+    this.sourceResultSettledAt = sourceSettledAt;
+    this.resolvedAt = revisedAt;
     this.updatedAt = revisedAt;
     return gap ? RevisionApplyResult.APPLIED_WITH_GAP : RevisionApplyResult.APPLIED;
   }


## `test(settlement): reject impossible revision chronology`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetRevisionChronologyTest.java b/src/test/java/com/sportsbook/betting/domain/BetRevisionChronologyTest.java
new file mode 100644
index 0000000..7a3c542
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/BetRevisionChronologyTest.java
@@ -0,0 +1,81 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.config.PermanentKafkaException;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.settlement.BetSettlementService;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetRevisionChronologyTest {
+
+  private static final Instant NOW = Instant.parse("2026-08-22T00:00:00Z");
+
+  @Test
+  void rejectsImpossibleChronologyBeforeLowerOrDuplicateOutcomes() {
+    Bet bet = BetTest.accepted(new BetSlipType.Single(), List.of(BetTest.leg("2.00")));
+    UUID eventId = bet.legs().get(0).eventId();
+    UUID revisionId = UUID.randomUUID();
+    String revisionHash = "a".repeat(64);
+    bet.applyRevision(
+        eventId,
+        revisionId,
+        2,
+        SettlementResult.LOST,
+        SettlementResult.WON,
+        Money.krw(0),
+        Money.krw(2_000),
+        NOW,
+        NOW.plusSeconds(1),
+        revisionHash);
+    BetRepository bets = mock(BetRepository.class);
+    when(bets.findLockedByBetId(bet.betId())).thenReturn(java.util.Optional.of(bet));
+    BetSettlementService service = new BetSettlementService(bets, new SystemBetCalculator());
+
+    assertPermanentChronologyFailure(
+        service, revised(bet, eventId, UUID.randomUUID(), 1), "b".repeat(64));
+    assertPermanentChronologyFailure(service, revised(bet, eventId, revisionId, 2), revisionHash);
+  }
+
+  private static void assertPermanentChronologyFailure(
+      BetSettlementService service, BetResolutionRevised event, String payloadHash) {
+    assertThatThrownBy(() -> service.apply(event, payloadHash))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessage("Invalid resolution revision")
+        .hasRootCauseMessage("sourceSettledAt must not be after revisedAt");
+  }
+
+  private static BetResolutionRevised revised(
+      Bet bet, UUID eventId, UUID revisionId, long revisionNumber) {
+    return BetResolutionRevised.newBuilder()
+        .setBetId(bet.betId().toString())
+        .setUserId(bet.userId().toString())
+        .setEventId(eventId.toString())
+        .setRevisionId(revisionId.toString())
+        .setRevisionNumber(revisionNumber)
+        .setPreviousResult(SettlementResultAvro.LOST)
+        .setNewResult(SettlementResultAvro.WON)
+        .setPreviousPayout(eventMoney(0))
+        .setNewPayout(eventMoney(2_000))
+        .setSourceResultSettledAt(NOW.plusSeconds(2))
+        .setRevisedAt(NOW.plusSeconds(1))
+        .build();
+  }
+
+  private static com.sportsbook.protocol.event.Money eventMoney(long amount) {
+    return com.sportsbook.protocol.event.Money.newBuilder()
+        .setAmount(amount)
+        .setCurrency("KRW")
+        .build();
+  }
+}
