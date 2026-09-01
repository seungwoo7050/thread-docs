# 순서화된 정산 수정, 갭, 충돌 격리

## `feat(database): persist resolution revisions`

diff --git a/src/main/resources/db/migration/V9__resolution_revision_projection.sql b/src/main/resources/db/migration/V9__resolution_revision_projection.sql
new file mode 100644
index 0000000..c8a6c28
--- /dev/null
+++ b/src/main/resources/db/migration/V9__resolution_revision_projection.sql
@@ -0,0 +1,42 @@
+-- Store a verifiable full-snapshot resolution projection across base and revision topics.
+ALTER TABLE bet
+    ADD COLUMN resolution_event_id             UUID,
+    ADD COLUMN resolution_revision_id          UUID,
+    ADD COLUMN resolution_revision_number      BIGINT,
+    ADD COLUMN resolution_payload_sha256       VARCHAR(64),
+    ADD COLUMN source_result_settled_at         TIMESTAMP WITH TIME ZONE;
+
+ALTER TABLE bet
+    ADD CONSTRAINT bet_resolution_hash_valid CHECK (
+        resolution_payload_sha256 IS NULL
+        OR resolution_payload_sha256 ~ '^[0-9a-f]{64}$'
+    ),
+    ADD CONSTRAINT bet_resolution_revision_valid CHECK (
+        (resolution_revision_id IS NULL
+            AND (resolution_revision_number IS NULL OR resolution_revision_number = 0))
+        OR
+        (resolution_revision_id IS NOT NULL AND resolution_revision_number >= 1)
+    ),
+    ADD CONSTRAINT bet_resolution_terminal_only CHECK (
+        status IN ('SETTLED', 'VOIDED')
+        OR (
+            resolution_event_id IS NULL
+            AND resolution_revision_id IS NULL
+            AND resolution_revision_number IS NULL
+            AND resolution_payload_sha256 IS NULL
+            AND source_result_settled_at IS NULL
+        )
+    ),
+    ADD CONSTRAINT bet_void_revision_forbidden CHECK (
+        status <> 'VOIDED' OR resolution_revision_id IS NULL
+    );
+
+-- Null proof columns remain permitted for terminal rows that predate this migration.
+CREATE UNIQUE INDEX uk_bet_resolution_revision
+    ON bet (resolution_revision_id)
+    WHERE resolution_revision_id IS NOT NULL;
+
+COMMENT ON COLUMN bet.resolution_revision_number IS
+    'Logical revision 0 for a base event and 1+ for a full replacement snapshot.';
+COMMENT ON COLUMN bet.source_result_settled_at IS
+    'Source result time used to compare and diagnose corrected settlement projections.';


## `test(database): verify revision projection constraints`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index 3880ce3..4363787 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -62,6 +62,15 @@ class MigrationContractTest {
         .contains("WHERE processed_at IS NULL");
   }
 
+  @Test
+  void permitsBaseAndHigherResolutionRevisions() {
+    assertThat(migrationText("V9__resolution_revision_projection.sql"))
+        .contains("resolution_revision_number = 0")
+        .contains("resolution_revision_number >= 1")
+        .contains("status <> 'VOIDED' OR resolution_revision_id IS NULL")
+        .contains("WHERE resolution_revision_id IS NOT NULL");
+  }
+
   private String migrationText(String migration) {
     try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {


## `feat(settlement): apply full revision snapshots`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index bce72f2..29b9e0b 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -126,6 +126,9 @@ public class Bet {
   @Column(name = "resolution_event_id")
   private UUID resolutionEventId;
 
+  @Column(name = "resolution_revision_id")
+  private UUID resolutionRevisionId;
+
   @Column(name = "resolution_revision_number")
   private Long resolutionRevisionNumber;
 
@@ -335,6 +338,64 @@ public class Bet {
     return value;
   }
 
+  public RevisionApplyResult applyRevision(
+      UUID eventId,
+      UUID revisionId,
+      long revisionNumber,
+      SettlementResult previousResult,
+      SettlementResult newResult,
+      Money previousPayout,
+      Money newPayout,
+      Instant sourceSettledAt,
+      Instant revisedAt,
+      String payloadHash) {
+    if (status != BetStatus.ACCEPTED && status != BetStatus.SETTLED) {
+      throw new IllegalStateException("Revisions require ACCEPTED or SETTLED status");
+    }
+    if (revisionNumber < 1) {
+      throw new IllegalArgumentException("revisionNumber must be at least 1");
+    }
+    long current = resolutionRevisionNumber == null ? 0 : resolutionRevisionNumber;
+    if (revisionNumber < current) {
+      return RevisionApplyResult.IGNORED;
+    }
+    if (revisionNumber == current) {
+      if (Objects.equals(resolutionRevisionId, revisionId)
+          && Objects.equals(resolutionPayloadSha256, payloadHash)) {
+        return RevisionApplyResult.DUPLICATE;
+      }
+      throw new IllegalStateException("Conflicting equal resolution revision");
+    }
+    boolean gap = revisionNumber > current + 1;
+    if (!gap && status == BetStatus.SETTLED) {
+      if (settlementResult != previousResult || !Objects.equals(settledPayout(), previousPayout)) {
+        throw new IllegalStateException("Revision previous snapshot does not match projection");
+      }
+    }
+    if (newPayout.currency() != stake.currency() || newPayout.isNegative()) {
+      throw new IllegalArgumentException("Revision payout is invalid");
+    }
+    this.status = BetStatus.SETTLED;
+    this.settlementResult = Objects.requireNonNull(newResult, "newResult");
+    this.settledPayout = EmbeddedMoney.of(newPayout);
+    this.voidReason = null;
+    this.resolutionEventId = Objects.requireNonNull(eventId, "eventId");
+    this.resolutionRevisionId = Objects.requireNonNull(revisionId, "revisionId");
+    this.resolutionRevisionNumber = revisionNumber;
+    this.resolutionPayloadSha256 = requireHash(payloadHash);
+    this.sourceResultSettledAt = Objects.requireNonNull(sourceSettledAt, "sourceSettledAt");
+    this.resolvedAt = Objects.requireNonNull(revisedAt, "revisedAt");
+    this.updatedAt = revisedAt;
+    return gap ? RevisionApplyResult.APPLIED_WITH_GAP : RevisionApplyResult.APPLIED;
+  }
+
+  public enum RevisionApplyResult {
+    APPLIED,
+    APPLIED_WITH_GAP,
+    DUPLICATE,
+    IGNORED
+  }
+
   private void requireCompensationInProgress(CompensationAction action) {
     requireStatus(BetStatus.PENDING);
     if (compensationAction != action || compensationState != CompensationState.IN_PROGRESS) {
@@ -509,6 +570,15 @@ public class Bet {
     return resolutionRevisionNumber == null ? -1 : resolutionRevisionNumber;
   }
 
+  public UUID resolutionRevisionId() {
+    return resolutionRevisionId;
+  }
+
+  public boolean hasResolution(UUID eventId, String payloadHash) {
+    return Objects.equals(resolutionEventId, eventId)
+        && Objects.equals(resolutionPayloadSha256, payloadHash);
+  }
+
   public String idempotencyKey() {
     return idempotencyKey;
   }


## `test(settlement): verify revision ordering semantics`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index 127ea83..a10cbd9 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -161,6 +161,105 @@ class BetTest {
         .isInstanceOf(IllegalArgumentException.class)
         .hasMessageContaining("selected leg");
   }
+
+  @Test
+  void revisionCanEstablishProjectionBeforeBase() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+    UUID revisionId = UUID.randomUUID();
+
+    Bet.RevisionApplyResult result =
+        bet.applyRevision(
+            bet.legs().get(0).eventId(),
+            revisionId,
+            1,
+            SettlementResult.LOST,
+            SettlementResult.WON,
+            Money.krw(0),
+            Money.krw(2_000),
+            NOW,
+            NOW.plusSeconds(1),
+            "3".repeat(64));
+
+    assertThat(result).isEqualTo(Bet.RevisionApplyResult.APPLIED);
+    assertThat(bet.status()).isEqualTo(BetStatus.SETTLED);
+    assertThat(bet.resolutionRevisionId()).isEqualTo(revisionId);
+    assertThat(bet.resolutionRevisionNumber()).isEqualTo(1);
+  }
+
+  @Test
+  void detectsGapAndIgnoresOlderRevision() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+    bet.settleBase(
+        bet.legs().get(0).eventId(),
+        SettlementResult.LOST,
+        Money.krw(1_000),
+        Money.krw(0),
+        NOW,
+        "4".repeat(64));
+
+    Bet.RevisionApplyResult gap =
+        bet.applyRevision(
+            bet.legs().get(0).eventId(),
+            UUID.randomUUID(),
+            2,
+            SettlementResult.WON,
+            SettlementResult.PUSH,
+            Money.krw(2_000),
+            Money.krw(1_000),
+            NOW,
+            NOW.plusSeconds(2),
+            "5".repeat(64));
+
+    assertThat(gap).isEqualTo(Bet.RevisionApplyResult.APPLIED_WITH_GAP);
+    assertThat(
+            bet.applyRevision(
+                bet.legs().get(0).eventId(),
+                UUID.randomUUID(),
+                1,
+                SettlementResult.LOST,
+                SettlementResult.WON,
+                Money.krw(0),
+                Money.krw(2_000),
+                NOW,
+                NOW.plusSeconds(1),
+                "6".repeat(64)))
+        .isEqualTo(Bet.RevisionApplyResult.IGNORED);
+  }
+
+  @Test
+  void distinguishesEqualRevisionReplayFromConflict() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+    bet.settleBase(
+        bet.legs().get(0).eventId(),
+        SettlementResult.LOST,
+        Money.krw(1_000),
+        Money.krw(0),
+        NOW,
+        "7".repeat(64));
+    UUID revisionId = UUID.randomUUID();
+
+    assertThat(revise(bet, revisionId, "8".repeat(64))).isEqualTo(Bet.RevisionApplyResult.APPLIED);
+    assertThat(revise(bet, revisionId, "8".repeat(64)))
+        .isEqualTo(Bet.RevisionApplyResult.DUPLICATE);
+    assertThatThrownBy(() -> revise(bet, UUID.randomUUID(), "9".repeat(64)))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("Conflicting equal");
+  }
+
+  static Bet.RevisionApplyResult revise(Bet bet, UUID revisionId, String hash) {
+    return bet.applyRevision(
+        bet.legs().get(0).eventId(),
+        revisionId,
+        1,
+        SettlementResult.LOST,
+        SettlementResult.WON,
+        Money.krw(0),
+        Money.krw(2_000),
+        NOW,
+        NOW.plusSeconds(1),
+        hash);
+  }
+
   static Bet accepted(BetSlipType type, List<BetLeg> legs) {
     Bet bet = Bet.pending(draft(UUID.randomUUID(), type), legs);
     bet.recordRiskReservation(NOW.plusSeconds(120), "9".repeat(64), false, NOW);


## `feat(settlement): consume ordered resolution revisions`

diff --git a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
index 2bf9025..81a4698 100644
--- a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
+++ b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
@@ -5,6 +5,7 @@ import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.domain.VoidReason;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.value.Currency;
@@ -56,6 +57,22 @@ public class BetSettlementService {
         eventId, VoidReason.valueOf(event.getReason().name()), event.getVoidedAt(), payloadHash);
   }
 
+  @Transactional
+  public Bet.RevisionApplyResult apply(BetResolutionRevised event, String payloadHash) {
+    Bet bet = owned(event.getBetId(), event.getUserId());
+    return bet.applyRevision(
+        canonical(event.getEventId()),
+        canonical(event.getRevisionId()),
+        event.getRevisionNumber(),
+        SettlementResult.valueOf(event.getPreviousResult().name()),
+        SettlementResult.valueOf(event.getNewResult().name()),
+        money(event.getPreviousPayout()),
+        money(event.getNewPayout()),
+        event.getSourceResultSettledAt(),
+        event.getRevisedAt(),
+        payloadHash);
+  }
+
   private Bet owned(String rawBetId, String rawUserId) {
     UUID betId = canonical(rawBetId);
     UUID userId = canonical(rawUserId);
diff --git a/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
new file mode 100644
index 0000000..4b271c8
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
@@ -0,0 +1,63 @@
+package com.sportsbook.betting.settlement;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SettlementResultListener {
+
+  private final BetSettlementService settlement;
+
+  public SettlementResultListener(BetSettlementService settlement) {
+    this.settlement = settlement;
+  }
+
+  @KafkaListener(
+      topics = {
+        BettingTopics.BET_SETTLED,
+        BettingTopics.BET_VOIDED,
+        BettingTopics.BET_RESOLUTION_REVISED
+      },
+      groupId = "betting-resolution")
+  public void onResolution(ConsumerRecord<String, byte[]> record) throws NoSuchAlgorithmException {
+    String hash = sha256(record.value());
+    switch (record.topic()) {
+      case BettingTopics.BET_SETTLED -> {
+        BetSettled event = AvroSerializer.deserialize(record.value(), BetSettled.class);
+        requireKey(record, event.getEventId());
+        settlement.apply(event, hash);
+      }
+      case BettingTopics.BET_VOIDED -> {
+        BetVoided event = AvroSerializer.deserialize(record.value(), BetVoided.class);
+        requireKey(record, event.getEventId());
+        settlement.apply(event, hash);
+      }
+      case BettingTopics.BET_RESOLUTION_REVISED -> {
+        BetResolutionRevised event =
+            AvroSerializer.deserialize(record.value(), BetResolutionRevised.class);
+        requireKey(record, event.getBetId());
+        settlement.apply(event, hash);
+      }
+      default -> throw new IllegalArgumentException("Unsupported resolution topic");
+    }
+  }
+
+  private static void requireKey(ConsumerRecord<String, byte[]> record, String eventId) {
+    if (!eventId.equals(record.key())) {
+      throw new IllegalArgumentException("Resolution Kafka key does not match eventId");
+    }
+  }
+
+  private static String sha256(byte[] value) throws NoSuchAlgorithmException {
+    return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(value));
+  }
+}


## `test(settlement): verify ordered revision consumption`

diff --git a/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
new file mode 100644
index 0000000..43ee507
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
@@ -0,0 +1,71 @@
+package com.sportsbook.betting.settlement;
+
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.anyString;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.verifyNoInteractions;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class SettlementResultListenerTest {
+
+  @Test
+  void dispatchesStrictRevisionBytesWithTheBetKey() throws Exception {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    BetResolutionRevised event = revision();
+    ConsumerRecord<String, byte[]> record = record(event, event.getBetId());
+
+    new SettlementResultListener(settlement).onResolution(record);
+
+    verify(settlement).apply(eq(event), anyString());
+  }
+
+  @Test
+  void rejectsAKeyMismatchBeforeMutatingTheProjection() {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    BetResolutionRevised event = revision();
+    ConsumerRecord<String, byte[]> record = record(event, UUID.randomUUID().toString());
+
+    assertThatThrownBy(() -> new SettlementResultListener(settlement).onResolution(record))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("Kafka key");
+    verifyNoInteractions(settlement);
+  }
+
+  private static ConsumerRecord<String, byte[]> record(BetResolutionRevised event, String key) {
+    return new ConsumerRecord<>(
+        BettingTopics.BET_RESOLUTION_REVISED, 0, 0, key, AvroSerializer.serialize(event));
+  }
+
+  private static BetResolutionRevised revision() {
+    return BetResolutionRevised.newBuilder()
+        .setRevisionId(UUID.randomUUID().toString())
+        .setRevisionNumber(1L)
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(UUID.randomUUID().toString())
+        .setEventId(UUID.randomUUID().toString())
+        .setPreviousResult(SettlementResultAvro.LOST)
+        .setNewResult(SettlementResultAvro.WON)
+        .setPreviousPayout(money(0))
+        .setNewPayout(money(2_000))
+        .setSourceResultSettledAt(Instant.EPOCH)
+        .setRevisedAt(Instant.EPOCH.plusSeconds(1))
+        .build();
+  }
+
+  private static com.sportsbook.protocol.event.Money money(long amount) {
+    return com.sportsbook.protocol.event.Money.newBuilder()
+        .setAmount(amount)
+        .setCurrency("KRW")
+        .build();
+  }
+}


## `feat(settlement): isolate permanent projection conflicts`

diff --git a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
index 81a4698..42ee697 100644
--- a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
+++ b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
@@ -1,5 +1,6 @@
 package com.sportsbook.betting.settlement;
 
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.domain.VoidReason;
@@ -27,50 +28,62 @@ public class BetSettlementService {
 
   @Transactional
   public void apply(BetSettled event, String payloadHash) {
-    Bet bet = owned(event.getBetId(), event.getUserId());
-    UUID eventId = canonical(event.getEventId());
-    if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
-      return;
+    try {
+      Bet bet = owned(event.getBetId(), event.getUserId());
+      UUID eventId = canonical(event.getEventId());
+      if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
+        return;
+      }
+      bet.settleBase(
+          eventId,
+          SettlementResult.valueOf(event.getResult().name()),
+          money(event.getStake()),
+          money(event.getPayout()),
+          event.getSettledAt(),
+          payloadHash);
+    } catch (IllegalArgumentException | IllegalStateException failure) {
+      throw permanent("Invalid settled projection", failure);
     }
-    bet.settleBase(
-        eventId,
-        SettlementResult.valueOf(event.getResult().name()),
-        money(event.getStake()),
-        money(event.getPayout()),
-        event.getSettledAt(),
-        payloadHash);
   }
 
   @Transactional
   public void apply(BetVoided event, String payloadHash) {
-    Bet bet = owned(event.getBetId(), event.getUserId());
-    UUID eventId = canonical(event.getEventId());
-    if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
-      return;
-    }
-    Money refund = money(event.getRefund());
-    Money exposure = calculator.totalStake(bet.slipType(), bet.stake(), bet.legs().size());
-    if (!refund.equals(exposure)) {
-      throw new IllegalArgumentException("Void refund does not match committed exposure");
+    try {
+      Bet bet = owned(event.getBetId(), event.getUserId());
+      UUID eventId = canonical(event.getEventId());
+      if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
+        return;
+      }
+      Money refund = money(event.getRefund());
+      Money exposure = calculator.totalStake(bet.slipType(), bet.stake(), bet.legs().size());
+      if (!refund.equals(exposure)) {
+        throw new IllegalArgumentException("Void refund does not match committed exposure");
+      }
+      bet.voidBase(
+          eventId, VoidReason.valueOf(event.getReason().name()), event.getVoidedAt(), payloadHash);
+    } catch (IllegalArgumentException | IllegalStateException failure) {
+      throw permanent("Invalid void projection", failure);
     }
-    bet.voidBase(
-        eventId, VoidReason.valueOf(event.getReason().name()), event.getVoidedAt(), payloadHash);
   }
 
   @Transactional
   public Bet.RevisionApplyResult apply(BetResolutionRevised event, String payloadHash) {
-    Bet bet = owned(event.getBetId(), event.getUserId());
-    return bet.applyRevision(
-        canonical(event.getEventId()),
-        canonical(event.getRevisionId()),
-        event.getRevisionNumber(),
-        SettlementResult.valueOf(event.getPreviousResult().name()),
-        SettlementResult.valueOf(event.getNewResult().name()),
-        money(event.getPreviousPayout()),
-        money(event.getNewPayout()),
-        event.getSourceResultSettledAt(),
-        event.getRevisedAt(),
-        payloadHash);
+    try {
+      Bet bet = owned(event.getBetId(), event.getUserId());
+      return bet.applyRevision(
+          canonical(event.getEventId()),
+          canonical(event.getRevisionId()),
+          event.getRevisionNumber(),
+          SettlementResult.valueOf(event.getPreviousResult().name()),
+          SettlementResult.valueOf(event.getNewResult().name()),
+          money(event.getPreviousPayout()),
+          money(event.getNewPayout()),
+          event.getSourceResultSettledAt(),
+          event.getRevisedAt(),
+          payloadHash);
+    } catch (IllegalArgumentException | IllegalStateException failure) {
+      throw permanent("Invalid resolution revision", failure);
+    }
   }
 
   private Bet owned(String rawBetId, String rawUserId) {
@@ -78,9 +91,9 @@ public class BetSettlementService {
     UUID userId = canonical(rawUserId);
     Bet bet =
         bets.findLockedByBetId(betId)
-            .orElseThrow(() -> new IllegalStateException("Resolution references unknown bet"));
+            .orElseThrow(() -> new PermanentKafkaException("Resolution references unknown bet"));
     if (!bet.userId().equals(userId)) {
-      throw new IllegalStateException("Resolution actor does not own bet");
+      throw new PermanentKafkaException("Resolution actor does not own bet");
     }
     return bet;
   }
@@ -93,7 +106,7 @@ public class BetSettlementService {
       if (bet.hasResolution(eventId, hash)) {
         return true;
       }
-      throw new IllegalStateException("Conflicting base resolution replay");
+      throw new PermanentKafkaException("Conflicting base resolution replay");
     }
     return false;
   }
@@ -109,4 +122,11 @@ public class BetSettlementService {
   private static Money money(com.sportsbook.protocol.event.Money value) {
     return new Money(value.getAmount(), Currency.valueOf(value.getCurrency()));
   }
+
+  private static PermanentKafkaException permanent(String message, RuntimeException failure) {
+    if (failure instanceof PermanentKafkaException permanent) {
+      return permanent;
+    }
+    return new PermanentKafkaException(message, failure);
+  }
 }


