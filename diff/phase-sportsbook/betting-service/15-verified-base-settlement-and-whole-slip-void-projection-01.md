# 검증된 기본 정산과 전체 슬립 무효화 투영

## `feat(settlement): model whole-slip void reasons`

diff --git a/src/main/java/com/sportsbook/betting/domain/VoidReason.java b/src/main/java/com/sportsbook/betting/domain/VoidReason.java
new file mode 100644
index 0000000..b4dbf3c
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/VoidReason.java
@@ -0,0 +1,8 @@
+package com.sportsbook.betting.domain;
+
+public enum VoidReason {
+  EVENT_CANCELLED,
+  EVENT_POSTPONED,
+  MARKET_VOID,
+  ADMIN_VOID
+}


## `test(settlement): verify void reason symbols`

diff --git a/src/test/java/com/sportsbook/betting/domain/VoidReasonTest.java b/src/test/java/com/sportsbook/betting/domain/VoidReasonTest.java
new file mode 100644
index 0000000..0a9712e
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/domain/VoidReasonTest.java
@@ -0,0 +1,15 @@
+package com.sportsbook.betting.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class VoidReasonTest {
+
+  @Test
+  void mirrorsSharedWireSymbols() {
+    assertThat(VoidReason.values())
+        .extracting(Enum::name)
+        .containsExactly("EVENT_CANCELLED", "EVENT_POSTPONED", "MARKET_VOID", "ADMIN_VOID");
+  }
+}


## `feat(database): add settlement outcome columns`

diff --git a/src/main/resources/db/migration/V3__settlement_outcome.sql b/src/main/resources/db/migration/V3__settlement_outcome.sql
new file mode 100644
index 0000000..9c7a678
--- /dev/null
+++ b/src/main/resources/db/migration/V3__settlement_outcome.sql
@@ -0,0 +1,34 @@
+-- V3: Bet settlement outcome columns (ADR-0006 async settlement, ADR-0013).
+--
+-- betting-service consumes BetSettled (settlement-service publishes it after a
+-- MatchResult) and flips an ACCEPTED slip to SETTLED, recording the adjudicated
+-- result and the actual payout credited to the wallet. max_payout (V1) is the
+-- worst-case figure stored at acceptance for audit; this records the real
+-- settle-time payout. resolved_at is the terminal-transition timestamp, shared
+-- with the VOIDED path added in V4.
+
+ALTER TABLE bet
+    ADD COLUMN settlement_result       VARCHAR(8),
+    ADD COLUMN settled_payout_amount   BIGINT,
+    ADD COLUMN settled_payout_currency VARCHAR(3),
+    ADD COLUMN resolved_at             TIMESTAMP WITH TIME ZONE;
+
+-- ADR-0013 SettlementResult. WON/PUSH pay out, LOST pays zero, VOID is a
+-- per-selection settle-time refund (distinct from a whole-slip VOIDED, V4).
+ALTER TABLE bet ADD CONSTRAINT bet_settlement_result_valid
+    CHECK (settlement_result IS NULL OR settlement_result IN ('WON', 'LOST', 'PUSH', 'VOID'));
+
+-- Payout amount and currency are written together (both null pre-settlement,
+-- both set on SETTLED); the payout never goes negative.
+ALTER TABLE bet ADD CONSTRAINT bet_settled_payout_paired
+    CHECK ((settled_payout_amount IS NULL) = (settled_payout_currency IS NULL));
+ALTER TABLE bet ADD CONSTRAINT bet_settled_payout_nonneg
+    CHECK (settled_payout_amount IS NULL OR settled_payout_amount >= 0);
+-- No FX inside a slip (V1): the payout currency matches the stake currency.
+ALTER TABLE bet ADD CONSTRAINT bet_settled_payout_currency_match
+    CHECK (settled_payout_currency IS NULL OR settled_payout_currency = stake_currency);
+
+COMMENT ON COLUMN bet.settlement_result       IS 'ADR-0013 SettlementResult on SETTLED (WON/LOST/PUSH/VOID). Null until settled.';
+COMMENT ON COLUMN bet.settled_payout_amount   IS 'Actual payout credited at settlement (minor units). Null until settled.';
+COMMENT ON COLUMN bet.settled_payout_currency IS 'Currency of the settle-time payout; equals stake_currency. Null until settled.';
+COMMENT ON COLUMN bet.resolved_at             IS 'Terminal-transition time: settledAt (SETTLED) or voidedAt (VOIDED).';


## `test(database): lock settlement schema checksum`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index eeae7d1..0b4fe8d 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -23,13 +23,19 @@ class MigrationContractTest {
         .isEqualTo("28161d23320d94a41d17b64a1dd0e2c9513fdfa74ac10ea1fb86bc4edf2c3d39");
   }
 
+  @Test
+  void preservesSettlementOutcomeSchema() {
+    assertThat(sha256("V3__settlement_outcome.sql"))
+        .isEqualTo("a57b6a695e8a94624d1e62fe4719e5ce384bc6cec00236d11b68b1a2e21b9589");
+  }
+
   private String sha256(String migration) {
-    try (InputStream input =
-        getClass().getResourceAsStream("/db/migration/" + migration)) {
+    try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {
         throw new IllegalStateException("Missing migration " + migration);
       }
-      return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(input.readAllBytes()));
+      return HexFormat.of()
+          .formatHex(MessageDigest.getInstance("SHA-256").digest(input.readAllBytes()));
     } catch (IOException | NoSuchAlgorithmException exception) {
       throw new IllegalStateException(exception);
     }


## `feat(database): add whole-slip void reason`

diff --git a/src/main/resources/db/migration/V4__bet_void_reason.sql b/src/main/resources/db/migration/V4__bet_void_reason.sql
new file mode 100644
index 0000000..293c4b9
--- /dev/null
+++ b/src/main/resources/db/migration/V4__bet_void_reason.sql
@@ -0,0 +1,15 @@
+-- V4: Whole-slip void reason (ADR-0012: cancelled / postponed events void the bet).
+--
+-- betting-service consumes BetVoided (settlement-service publishes it when an
+-- event is cancelled/postponed, or a market / admin void applies) and flips an
+-- ACCEPTED slip to VOIDED; wallet-service handles the full stake refund. The
+-- void reuses resolved_at (V3) as its terminal-transition timestamp.
+
+ALTER TABLE bet ADD COLUMN void_reason VARCHAR(24);
+
+ALTER TABLE bet ADD CONSTRAINT bet_void_reason_valid
+    CHECK (void_reason IS NULL OR void_reason IN (
+        'EVENT_CANCELLED', 'EVENT_POSTPONED', 'MARKET_VOID', 'ADMIN_VOID'
+    ));
+
+COMMENT ON COLUMN bet.void_reason IS 'Why a VOIDED slip was voided (ADR-0012). Null unless status = VOIDED.';


## `test(database): lock void schema checksum`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index 0b4fe8d..20bebbf 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -29,6 +29,12 @@ class MigrationContractTest {
         .isEqualTo("a57b6a695e8a94624d1e62fe4719e5ce384bc6cec00236d11b68b1a2e21b9589");
   }
 
+  @Test
+  void preservesWholeSlipVoidSchema() {
+    assertThat(sha256("V4__bet_void_reason.sql"))
+        .isEqualTo("4e42907201bdbfe211f505ac9a8fbe4321a493b55a56747f48af078eb98f3ca8");
+  }
+
   private String sha256(String migration) {
     try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {


## `feat(settlement): project verified base resolutions`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 3769131..bce72f2 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -2,6 +2,7 @@ package com.sportsbook.betting.domain;
 
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.value.Money;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
@@ -102,6 +103,38 @@ public class Bet {
   @Column(name = "compensation_operation_id")
   private UUID compensationOperationId;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "settlement_result", length = 8)
+  private SettlementResult settlementResult;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(name = "amount", column = @Column(name = "settled_payout_amount")),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "settled_payout_currency", length = 3))
+  })
+  private EmbeddedMoney settledPayout;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "void_reason", length = 24)
+  private VoidReason voidReason;
+
+  @Column(name = "resolved_at")
+  private Instant resolvedAt;
+
+  @Column(name = "resolution_event_id")
+  private UUID resolutionEventId;
+
+  @Column(name = "resolution_revision_number")
+  private Long resolutionRevisionNumber;
+
+  @Column(name = "resolution_payload_sha256", length = 64)
+  private String resolutionPayloadSha256;
+
+  @Column(name = "source_result_settled_at")
+  private Instant sourceResultSettledAt;
+
   @Enumerated(EnumType.STRING)
   @Column(name = "status", nullable = false, length = 16)
   private BetStatus status;
@@ -257,6 +290,51 @@ public class Bet {
     this.updatedAt = Objects.requireNonNull(now, "now");
   }
 
+  public void settleBase(
+      UUID eventId,
+      SettlementResult result,
+      Money eventStake,
+      Money payout,
+      Instant settledAt,
+      String payloadHash) {
+    requireStatus(BetStatus.ACCEPTED);
+    requireSelectionEvent(eventId);
+    if (!stake.toMoney().equals(eventStake)) {
+      throw new IllegalArgumentException("Settlement stake does not match original unit stake");
+    }
+    if (payout.currency() != stake.currency() || payout.isNegative()) {
+      throw new IllegalArgumentException("Settlement payout is invalid");
+    }
+    this.status = BetStatus.SETTLED;
+    this.settlementResult = Objects.requireNonNull(result, "result");
+    this.settledPayout = EmbeddedMoney.of(payout);
+    recordBaseResolution(eventId, settledAt, payloadHash);
+  }
+
+  public void voidBase(UUID eventId, VoidReason reason, Instant voidedAt, String payloadHash) {
+    requireStatus(BetStatus.ACCEPTED);
+    requireSelectionEvent(eventId);
+    this.status = BetStatus.VOIDED;
+    this.voidReason = Objects.requireNonNull(reason, "reason");
+    recordBaseResolution(eventId, voidedAt, payloadHash);
+  }
+
+  private void recordBaseResolution(UUID eventId, Instant at, String payloadHash) {
+    this.resolutionEventId = Objects.requireNonNull(eventId, "eventId");
+    this.resolutionRevisionNumber = 0L;
+    this.resolutionPayloadSha256 = requireHash(payloadHash);
+    this.sourceResultSettledAt = Objects.requireNonNull(at, "at");
+    this.resolvedAt = at;
+    this.updatedAt = at;
+  }
+
+  private static String requireHash(String value) {
+    if (value == null || !value.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("payload hash must be lowercase SHA-256");
+    }
+    return value;
+  }
+
   private void requireCompensationInProgress(CompensationAction action) {
     requireStatus(BetStatus.PENDING);
     if (compensationAction != action || compensationState != CompensationState.IN_PROGRESS) {
@@ -411,6 +489,26 @@ public class Bet {
     return compensationOperationId;
   }
 
+  public SettlementResult settlementResult() {
+    return settlementResult;
+  }
+
+  public Money settledPayout() {
+    return settledPayout == null ? null : settledPayout.toMoney();
+  }
+
+  public VoidReason voidReason() {
+    return voidReason;
+  }
+
+  public Instant resolvedAt() {
+    return resolvedAt;
+  }
+
+  public long resolutionRevisionNumber() {
+    return resolutionRevisionNumber == null ? -1 : resolutionRevisionNumber;
+  }
+
   public String idempotencyKey() {
     return idempotencyKey;
   }


## `test(settlement): verify base resolution projection`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index 1b49ce0..127ea83 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -1,9 +1,11 @@
 package com.sportsbook.betting.domain;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
 import java.time.Instant;
@@ -114,6 +116,60 @@ class BetTest {
     assertThat(bet.status()).isEqualTo(BetStatus.REJECTED);
   }
 
+  @Test
+  void settlesAgainstOriginalSystemUnitStake() {
+    Bet bet = accepted(new BetSlipType.System(2, 3), List.of(leg("2"), leg("3"), leg("4")));
+    UUID eventId = bet.legs().get(0).eventId();
+
+    bet.settleBase(
+        eventId,
+        SettlementResult.WON,
+        Money.krw(1_000),
+        Money.krw(2_600),
+        NOW.plusSeconds(10),
+        "1".repeat(64));
+
+    assertThat(bet.status()).isEqualTo(BetStatus.SETTLED);
+    assertThat(bet.settlementResult()).isEqualTo(SettlementResult.WON);
+    assertThat(bet.settledPayout()).isEqualTo(Money.krw(2_600));
+    assertThat(bet.resolutionRevisionNumber()).isZero();
+  }
+
+  @Test
+  void projectsWholeSlipVoidSeparately() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+
+    bet.voidBase(bet.legs().get(0).eventId(), VoidReason.EVENT_CANCELLED, NOW, "2".repeat(64));
+
+    assertThat(bet.status()).isEqualTo(BetStatus.VOIDED);
+    assertThat(bet.voidReason()).isEqualTo(VoidReason.EVENT_CANCELLED);
+  }
+
+  @Test
+  void rejectsBaseResolutionForAnUnselectedEvent() {
+    Bet bet = accepted(new BetSlipType.Single(), List.of(leg("2")));
+
+    assertThatThrownBy(
+            () ->
+                bet.settleBase(
+                    UUID.randomUUID(),
+                    SettlementResult.WON,
+                    Money.krw(1_000),
+                    Money.krw(2_000),
+                    NOW,
+                    "a".repeat(64)))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("selected leg");
+  }
+  static Bet accepted(BetSlipType type, List<BetLeg> legs) {
+    Bet bet = Bet.pending(draft(UUID.randomUUID(), type), legs);
+    bet.recordRiskReservation(NOW.plusSeconds(120), "9".repeat(64), false, NOW);
+    bet.confirmWallet(UUID.randomUUID(), NOW);
+    bet.commitRisk(NOW);
+    bet.accept(NOW);
+    return bet;
+  }
+
   @Test
   void rejectsSlipShapeMismatch() {
     org.assertj.core.api.Assertions.assertThatThrownBy(


## `feat(settlement): project base resolution snapshots`

diff --git a/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
new file mode 100644
index 0000000..2bf9025
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/settlement/BetSettlementService.java
@@ -0,0 +1,95 @@
+package com.sportsbook.betting.settlement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.SystemBetCalculator;
+import com.sportsbook.betting.domain.VoidReason;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class BetSettlementService {
+
+  private final BetRepository bets;
+  private final SystemBetCalculator calculator;
+
+  public BetSettlementService(BetRepository bets, SystemBetCalculator calculator) {
+    this.bets = bets;
+    this.calculator = calculator;
+  }
+
+  @Transactional
+  public void apply(BetSettled event, String payloadHash) {
+    Bet bet = owned(event.getBetId(), event.getUserId());
+    UUID eventId = canonical(event.getEventId());
+    if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
+      return;
+    }
+    bet.settleBase(
+        eventId,
+        SettlementResult.valueOf(event.getResult().name()),
+        money(event.getStake()),
+        money(event.getPayout()),
+        event.getSettledAt(),
+        payloadHash);
+  }
+
+  @Transactional
+  public void apply(BetVoided event, String payloadHash) {
+    Bet bet = owned(event.getBetId(), event.getUserId());
+    UUID eventId = canonical(event.getEventId());
+    if (duplicateOrSuperseded(bet, eventId, payloadHash)) {
+      return;
+    }
+    Money refund = money(event.getRefund());
+    Money exposure = calculator.totalStake(bet.slipType(), bet.stake(), bet.legs().size());
+    if (!refund.equals(exposure)) {
+      throw new IllegalArgumentException("Void refund does not match committed exposure");
+    }
+    bet.voidBase(
+        eventId, VoidReason.valueOf(event.getReason().name()), event.getVoidedAt(), payloadHash);
+  }
+
+  private Bet owned(String rawBetId, String rawUserId) {
+    UUID betId = canonical(rawBetId);
+    UUID userId = canonical(rawUserId);
+    Bet bet =
+        bets.findLockedByBetId(betId)
+            .orElseThrow(() -> new IllegalStateException("Resolution references unknown bet"));
+    if (!bet.userId().equals(userId)) {
+      throw new IllegalStateException("Resolution actor does not own bet");
+    }
+    return bet;
+  }
+
+  private static boolean duplicateOrSuperseded(Bet bet, UUID eventId, String hash) {
+    if (bet.resolutionRevisionNumber() > 0) {
+      return true;
+    }
+    if (bet.resolutionRevisionNumber() == 0) {
+      if (bet.hasResolution(eventId, hash)) {
+        return true;
+      }
+      throw new IllegalStateException("Conflicting base resolution replay");
+    }
+    return false;
+  }
+
+  private static UUID canonical(String value) {
+    UUID parsed = UUID.fromString(value);
+    if (!parsed.toString().equals(value)) {
+      throw new IllegalArgumentException("Resolution identifier must be canonical UUID");
+    }
+    return parsed;
+  }
+
+  private static Money money(com.sportsbook.protocol.event.Money value) {
+    return new Money(value.getAmount(), Currency.valueOf(value.getCurrency()));
+  }
+}


## `test(settlement): verify base resolution contract`

diff --git a/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
new file mode 100644
index 0000000..beed4a5
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/settlement/BetSettlementServiceTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.betting.settlement;
+
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.SystemBetCalculator;
+import com.sportsbook.betting.domain.VoidReason;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetSettlementServiceTest {
+
+  @Test
+  void projectsSettledSystemEventUsingItsOriginalUnitStake() {
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    UUID eventId = UUID.randomUUID();
+    when(bet.userId()).thenReturn(userId);
+    when(bet.resolutionRevisionNumber()).thenReturn(-1L);
+    when(bets.findLockedByBetId(betId)).thenReturn(Optional.of(bet));
+    BetSettled event =
+        BetSettled.newBuilder()
+            .setBetId(betId.toString())
+            .setUserId(userId.toString())
+            .setEventId(eventId.toString())
+            .setResult(SettlementResultAvro.WON)
+            .setStake(eventMoney(1_000))
+            .setPayout(eventMoney(2_600))
+            .setSettledAt(Instant.EPOCH)
+            .setResultDetail(java.util.Map.of())
+            .build();
+
+    new BetSettlementService(bets, new SystemBetCalculator()).apply(event, "a".repeat(64));
+
+    verify(bet)
+        .settleBase(
+            eventId,
+            SettlementResult.WON,
+            Money.krw(1_000),
+            Money.krw(2_600),
+            Instant.EPOCH,
+            "a".repeat(64));
+  }
+
+  @Test
+  void validatesWholeSlipVoidAgainstCommittedSystemExposure() {
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    UUID eventId = UUID.randomUUID();
+    when(bet.userId()).thenReturn(userId);
+    when(bet.resolutionRevisionNumber()).thenReturn(-1L);
+    when(bet.slipType()).thenReturn(new BetSlipType.System(2, 3));
+    when(bet.stake()).thenReturn(Money.krw(1_000));
+    when(bet.legs())
+        .thenReturn(List.of(mock(BetLeg.class), mock(BetLeg.class), mock(BetLeg.class)));
+    when(bets.findLockedByBetId(betId)).thenReturn(Optional.of(bet));
+    BetVoided event =
+        BetVoided.newBuilder()
+            .setBetId(betId.toString())
+            .setUserId(userId.toString())
+            .setEventId(eventId.toString())
+            .setReason(com.sportsbook.protocol.event.VoidReason.EVENT_CANCELLED)
+            .setRefund(eventMoney(3_000))
+            .setVoidedAt(Instant.EPOCH)
+            .build();
+
+    new BetSettlementService(bets, new SystemBetCalculator()).apply(event, "b".repeat(64));
+
+    verify(bet).voidBase(eventId, VoidReason.EVENT_CANCELLED, Instant.EPOCH, "b".repeat(64));
+  }
+
+  private static com.sportsbook.protocol.event.Money eventMoney(long amount) {
+    return com.sportsbook.protocol.event.Money.newBuilder()
+        .setAmount(amount)
+        .setCurrency("KRW")
+        .build();
+  }
+}


