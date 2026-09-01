# 영속 멱등성 네임스페이스와 정확한 결과 재생

## `feat(placement): define immutable placement command`

diff --git a/src/main/java/com/sportsbook/betting/placement/PlaceBetCommand.java b/src/main/java/com/sportsbook/betting/placement/PlaceBetCommand.java
new file mode 100644
index 0000000..ce2becb
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/PlaceBetCommand.java
@@ -0,0 +1,36 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.Objects;
+import java.util.UUID;
+
+public record PlaceBetCommand(
+    UUID userId,
+    BetSlipType slipType,
+    List<SelectionInput> selections,
+    Money unitStake,
+    IdempotencyKey idempotencyKey) {
+
+  public PlaceBetCommand {
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(slipType, "slipType");
+    Objects.requireNonNull(unitStake, "unitStake");
+    Objects.requireNonNull(idempotencyKey, "idempotencyKey");
+    selections = List.copyOf(Objects.requireNonNull(selections, "selections"));
+  }
+
+  public record SelectionInput(
+      UUID eventId, UUID marketId, UUID selectionId, Odds oddsAtSubmission) {
+
+    public SelectionInput {
+      Objects.requireNonNull(eventId, "eventId");
+      Objects.requireNonNull(marketId, "marketId");
+      Objects.requireNonNull(selectionId, "selectionId");
+      Objects.requireNonNull(oddsAtSubmission, "oddsAtSubmission");
+    }
+  }
+}


## `test(placement): verify immutable command input`

diff --git a/src/test/java/com/sportsbook/betting/placement/PlaceBetCommandTest.java b/src/test/java/com/sportsbook/betting/placement/PlaceBetCommandTest.java
new file mode 100644
index 0000000..bb79f39
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/PlaceBetCommandTest.java
@@ -0,0 +1,30 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.util.ArrayList;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class PlaceBetCommandTest {
+
+  @Test
+  void defensivelyCopiesSelections() {
+    var mutable = new ArrayList<PlaceBetCommand.SelectionInput>();
+    PlaceBetCommand command =
+        new PlaceBetCommand(
+            UUID.randomUUID(),
+            new BetSlipType.Single(),
+            mutable,
+            Money.krw(1_000),
+            IdempotencyKey.of("request-1"));
+
+    assertThat(command.selections()).isEmpty();
+    assertThatThrownBy(() -> command.selections().clear())
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
+}


## `feat(placement): fingerprint canonical requests`

diff --git a/src/main/java/com/sportsbook/betting/placement/RequestFingerprint.java b/src/main/java/com/sportsbook/betting/placement/RequestFingerprint.java
new file mode 100644
index 0000000..5acf9cc
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/RequestFingerprint.java
@@ -0,0 +1,50 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+
+public final class RequestFingerprint {
+
+  public static String of(PlaceBetCommand command) {
+    StringBuilder canonical =
+        new StringBuilder()
+            .append(command.userId())
+            .append('|')
+            .append(slipType(command.slipType()))
+            .append('|')
+            .append(command.unitStake().amount())
+            .append('|')
+            .append(command.unitStake().currency());
+    for (PlaceBetCommand.SelectionInput selection : command.selections()) {
+      canonical
+          .append('|')
+          .append(selection.eventId())
+          .append('|')
+          .append(selection.marketId())
+          .append('|')
+          .append(selection.selectionId())
+          .append('|')
+          .append(selection.oddsAtSubmission().decimal().stripTrailingZeros().toPlainString());
+    }
+    try {
+      return HexFormat.of()
+          .formatHex(
+              MessageDigest.getInstance("SHA-256")
+                  .digest(canonical.toString().getBytes(StandardCharsets.UTF_8)));
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("SHA-256 unavailable", exception);
+    }
+  }
+
+  private static String slipType(BetSlipType type) {
+    if (type instanceof BetSlipType.System system) {
+      return "SYSTEM:" + system.minWins() + ':' + system.totalSelections();
+    }
+    return type instanceof BetSlipType.Single ? "SINGLE" : "MULTIPLE";
+  }
+
+  private RequestFingerprint() {}
+}


## `test(placement): verify canonical request fingerprints`

diff --git a/src/test/java/com/sportsbook/betting/placement/RequestFingerprintTest.java b/src/test/java/com/sportsbook/betting/placement/RequestFingerprintTest.java
new file mode 100644
index 0000000..fb40b7e
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/RequestFingerprintTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RequestFingerprintTest {
+
+  @Test
+  void isStableButSeparatesSystemShape() {
+    UUID user = UUID.randomUUID();
+    var selection =
+        new PlaceBetCommand.SelectionInput(
+            UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.5000"));
+    PlaceBetCommand first = command(user, new BetSlipType.System(2, 3), selection);
+    PlaceBetCommand same = command(user, new BetSlipType.System(2, 3), selection);
+    PlaceBetCommand different = command(user, new BetSlipType.System(1, 3), selection);
+
+    assertThat(RequestFingerprint.of(first)).matches("[0-9a-f]{64}");
+    assertThat(RequestFingerprint.of(first)).isEqualTo(RequestFingerprint.of(same));
+    assertThat(RequestFingerprint.of(first)).isNotEqualTo(RequestFingerprint.of(different));
+  }
+
+  private PlaceBetCommand command(
+      UUID user, BetSlipType type, PlaceBetCommand.SelectionInput input) {
+    return new PlaceBetCommand(
+        user, type, List.of(input), Money.krw(1_000), IdempotencyKey.of("key"));
+  }
+}


## `feat(placement): classify durable request outcomes`

diff --git a/src/main/java/com/sportsbook/betting/placement/PlacementOutcome.java b/src/main/java/com/sportsbook/betting/placement/PlacementOutcome.java
new file mode 100644
index 0000000..0ba5ef9
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/PlacementOutcome.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.placement;
+
+public enum PlacementOutcome {
+  BET,
+  REJECTION;
+
+  public boolean hasBet() {
+    return this == BET;
+  }
+}


## `test(placement): verify durable outcome vocabulary`

diff --git a/src/test/java/com/sportsbook/betting/placement/PlacementOutcomeTest.java b/src/test/java/com/sportsbook/betting/placement/PlacementOutcomeTest.java
new file mode 100644
index 0000000..7438386
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/PlacementOutcomeTest.java
@@ -0,0 +1,14 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class PlacementOutcomeTest {
+
+  @Test
+  void distinguishesBetPointerFromPreflightVerdict() {
+    assertThat(PlacementOutcome.BET.hasBet()).isTrue();
+    assertThat(PlacementOutcome.REJECTION.hasBet()).isFalse();
+  }
+}


## `feat(error): replay durable lookup outcomes`

diff --git a/src/main/java/com/sportsbook/betting/error/BetNotFoundException.java b/src/main/java/com/sportsbook/betting/error/BetNotFoundException.java
new file mode 100644
index 0000000..52f7ca9
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/BetNotFoundException.java
@@ -0,0 +1,8 @@
+package com.sportsbook.betting.error;
+
+public class BetNotFoundException extends RuntimeException {
+
+  public BetNotFoundException(String message) {
+    super(message);
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/error/PersistedRejectionException.java b/src/main/java/com/sportsbook/betting/error/PersistedRejectionException.java
new file mode 100644
index 0000000..4a63e5d
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/error/PersistedRejectionException.java
@@ -0,0 +1,10 @@
+package com.sportsbook.betting.error;
+
+import com.sportsbook.protocol.error.ErrorCode;
+
+public class PersistedRejectionException extends BetPlacementException {
+
+  public PersistedRejectionException(ErrorCode errorCode, String detail) {
+    super(errorCode, detail);
+  }
+}


## `feat(placement): own idempotency request namespace`

diff --git a/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java b/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java
new file mode 100644
index 0000000..edfb2bd
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java
@@ -0,0 +1,91 @@
+package com.sportsbook.betting.placement;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+@Entity
+@Table(name = "placement_request")
+public class PlacementRequest {
+
+  @Id
+  @Column(name = "idempotency_key", nullable = false, updatable = false, length = 128)
+  private String idempotencyKey;
+
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Column(name = "request_fingerprint", updatable = false, length = 64)
+  private String requestFingerprint;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "outcome", nullable = false, updatable = false, length = 16)
+  private PlacementOutcome outcome;
+
+  @Column(name = "bet_id", updatable = false)
+  private UUID betId;
+
+  @Column(name = "error_code", updatable = false, length = 64)
+  private String errorCode;
+
+  @Column(name = "error_detail", updatable = false, length = 1024)
+  private String errorDetail;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  protected PlacementRequest() {}
+
+  private PlacementRequest(
+      String key,
+      UUID userId,
+      String fingerprint,
+      PlacementOutcome outcome,
+      UUID betId,
+      String errorCode,
+      String errorDetail,
+      Instant createdAt) {
+    this.idempotencyKey = Objects.requireNonNull(key, "key");
+    this.userId = Objects.requireNonNull(userId, "userId");
+    this.requestFingerprint = Objects.requireNonNull(fingerprint, "fingerprint");
+    this.outcome = Objects.requireNonNull(outcome, "outcome");
+    this.betId = betId;
+    this.errorCode = errorCode;
+    this.errorDetail = errorDetail;
+    this.createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  public String idempotencyKey() {
+    return idempotencyKey;
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+
+  public String requestFingerprint() {
+    return requestFingerprint;
+  }
+
+  public PlacementOutcome outcome() {
+    return outcome;
+  }
+
+  public UUID betId() {
+    return betId;
+  }
+
+  public String errorCode() {
+    return errorCode;
+  }
+
+  public String errorDetail() {
+    return errorDetail;
+  }
+}


## `feat(placement): create durable request outcomes`

diff --git a/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java b/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java
index edfb2bd..1cbe47a 100644
--- a/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java
+++ b/src/main/java/com/sportsbook/betting/placement/PlacementRequest.java
@@ -1,5 +1,7 @@
 package com.sportsbook.betting.placement;
 
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.protocol.error.ErrorCode;
 import jakarta.persistence.Column;
 import jakarta.persistence.Entity;
 import jakarta.persistence.EnumType;
@@ -61,6 +63,24 @@ public class PlacementRequest {
     this.createdAt = Objects.requireNonNull(createdAt, "createdAt");
   }
 
+  public static PlacementRequest forBet(Bet bet) {
+    return new PlacementRequest(
+        bet.idempotencyKey(),
+        bet.userId(),
+        bet.requestFingerprint(),
+        PlacementOutcome.BET,
+        bet.betId(),
+        null,
+        null,
+        bet.createdAt());
+  }
+
+  public static PlacementRequest rejected(
+      String key, UUID userId, String fingerprint, ErrorCode code, String detail, Instant at) {
+    return new PlacementRequest(
+        key, userId, fingerprint, PlacementOutcome.REJECTION, null, code.name(), detail, at);
+  }
+
   public String idempotencyKey() {
     return idempotencyKey;
   }


## `feat(placement): claim durable request keys`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetStore.java b/src/main/java/com/sportsbook/betting/placement/BetStore.java
new file mode 100644
index 0000000..78e3f2d
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/BetStore.java
@@ -0,0 +1,51 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.PlacementRequestRepository;
+import com.sportsbook.protocol.error.ErrorCode;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+@Component
+public class BetStore {
+
+  private final BetRepository bets;
+  private final PlacementRequestRepository requests;
+
+  public BetStore(BetRepository bets, PlacementRequestRepository requests) {
+    this.bets = bets;
+    this.requests = requests;
+  }
+
+  @Transactional(readOnly = true)
+  public Optional<PlacementRequest> findPlacementRequest(String idempotencyKey) {
+    return requests.findById(idempotencyKey);
+  }
+
+  @Transactional(readOnly = true)
+  public Optional<Bet> findByIdempotencyKey(String idempotencyKey) {
+    return bets.findByIdempotencyKey(idempotencyKey);
+  }
+
+  @Transactional(readOnly = true)
+  public Bet findById(UUID betId) {
+    return bets.findWithLegsByBetId(betId)
+        .orElseThrow(() -> new IllegalStateException("Bet not found during placement: " + betId));
+  }
+
+  @Transactional
+  public void savePending(Bet bet) {
+    bets.saveAndFlush(bet);
+    requests.saveAndFlush(PlacementRequest.forBet(bet));
+  }
+
+  @Transactional
+  public void savePreflightRejection(
+      String key, UUID userId, String fingerprint, ErrorCode code, String detail, Instant now) {
+    requests.saveAndFlush(PlacementRequest.rejected(key, userId, fingerprint, code, detail, now));
+  }
+}


## `test(placement): verify atomic request claim`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java b/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java
new file mode 100644
index 0000000..4bfb32c
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java
@@ -0,0 +1,54 @@
+package com.sportsbook.betting.placement;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.PlacementRequestRepository;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+
+class BetStoreTest {
+
+  @Test
+  void claimsBetBeforePublishingItsRequestPointer() {
+    BetRepository bets = mock(BetRepository.class);
+    PlacementRequestRepository requests = mock(PlacementRequestRepository.class);
+    Bet bet = pendingBet();
+
+    new BetStore(bets, requests).savePending(bet);
+
+    InOrder order = inOrder(bets, requests);
+    order.verify(bets).saveAndFlush(bet);
+    order.verify(requests).saveAndFlush(any(PlacementRequest.class));
+  }
+
+  private static Bet pendingBet() {
+    UUID betId = UUID.randomUUID();
+    BetDraft draft =
+        new BetDraft(
+            betId,
+            UUID.randomUUID(),
+            "B-2026-08-22-00000000",
+            new BetSlipType.Single(),
+            Money.krw(1_000),
+            Money.krw(2_000),
+            IdempotencyKey.of("request-1"),
+            "a".repeat(64),
+            Instant.EPOCH);
+    BetLeg leg =
+        BetLeg.create(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"));
+    return Bet.pending(draft, List.of(leg));
+  }
+}


## `feat(placement): persist request ownership`

diff --git a/src/main/java/com/sportsbook/betting/persistence/PlacementRequestRepository.java b/src/main/java/com/sportsbook/betting/persistence/PlacementRequestRepository.java
new file mode 100644
index 0000000..262e1c1
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/persistence/PlacementRequestRepository.java
@@ -0,0 +1,6 @@
+package com.sportsbook.betting.persistence;
+
+import com.sportsbook.betting.placement.PlacementRequest;
+import org.springframework.data.jpa.repository.JpaRepository;
+
+public interface PlacementRequestRepository extends JpaRepository<PlacementRequest, String> {}


## `feat(placement): replay durable outcomes safely`

diff --git a/src/main/java/com/sportsbook/betting/placement/PlacementReplay.java b/src/main/java/com/sportsbook/betting/placement/PlacementReplay.java
new file mode 100644
index 0000000..b4465d9
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/PlacementReplay.java
@@ -0,0 +1,55 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.DuplicateBetException;
+import com.sportsbook.betting.error.PersistedRejectionException;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.error.ErrorCode;
+import java.util.UUID;
+import java.util.function.Function;
+
+final class PlacementReplay {
+
+  private PlacementReplay() {}
+
+  static Bet bet(Bet existing, UUID actorId, String fingerprint) {
+    validateIdentity(existing.userId(), existing.requestFingerprint(), actorId, fingerprint);
+    if (existing.status() == BetStatus.REJECTED) {
+      throw persisted(existing.rejectionReason(), existing.rejectionDetail());
+    }
+    return existing;
+  }
+
+  static Bet request(
+      PlacementRequest request, UUID actorId, String fingerprint, Function<UUID, Bet> betLoader) {
+    validateIdentity(request.userId(), request.requestFingerprint(), actorId, fingerprint);
+    if (!request.outcome().hasBet()) {
+      throw persisted(request.errorCode(), request.errorDetail());
+    }
+    return bet(betLoader.apply(request.betId()), actorId, fingerprint);
+  }
+
+  private static void validateIdentity(
+      UUID savedActor, String savedFingerprint, UUID actorId, String fingerprint) {
+    if (!savedActor.equals(actorId)) {
+      throw new DuplicateBetException("Idempotency-Key cannot be reused by this actor");
+    }
+    if (savedFingerprint != null
+        && !savedFingerprint.equals(fingerprint)
+        && !savedFingerprint.startsWith("legacy-")) {
+      throw new DuplicateBetException(
+          "Idempotency-Key cannot be reused with a different request payload");
+    }
+  }
+
+  private static PersistedRejectionException persisted(String rawCode, String detail) {
+    ErrorCode code;
+    try {
+      code = ErrorCode.valueOf(rawCode);
+    } catch (RuntimeException invalidCode) {
+      code = ErrorCode.INTERNAL_ERROR;
+    }
+    String replayDetail = detail == null ? rawCode : detail;
+    return new PersistedRejectionException(code, replayDetail);
+  }
+}


