## `test(placement): verify durable replay fencing`

diff --git a/src/test/java/com/sportsbook/betting/placement/PlacementReplayTest.java b/src/test/java/com/sportsbook/betting/placement/PlacementReplayTest.java
new file mode 100644
index 0000000..bc2badc
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/PlacementReplayTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+
+import com.sportsbook.betting.error.DuplicateBetException;
+import com.sportsbook.betting.error.PersistedRejectionException;
+import com.sportsbook.protocol.error.ErrorCode;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class PlacementReplayTest {
+
+  @Test
+  void rethrowsTheOriginalDefinitiveVerdict() {
+    UUID actorId = UUID.randomUUID();
+    PlacementRequest request =
+        PlacementRequest.rejected(
+            "request-1",
+            actorId,
+            "a".repeat(64),
+            ErrorCode.VALIDATION_FAILED,
+            "invalid slip",
+            Instant.EPOCH);
+
+    PersistedRejectionException failure =
+        catchThrowableOfType(
+            () -> PlacementReplay.request(request, actorId, "a".repeat(64), ignored -> null),
+            PersistedRejectionException.class);
+
+    assertThat(failure.errorCode()).isEqualTo(ErrorCode.VALIDATION_FAILED);
+    assertThat(failure).hasMessage("invalid slip");
+  }
+
+  @Test
+  void rejectsCrossActorKeyReplayBeforeLoadingTheBet() {
+    PlacementRequest request =
+        PlacementRequest.rejected(
+            "request-1",
+            UUID.randomUUID(),
+            "a".repeat(64),
+            ErrorCode.VALIDATION_FAILED,
+            "invalid slip",
+            Instant.EPOCH);
+
+    assertThat(
+            catchThrowableOfType(
+                    () ->
+                        PlacementReplay.request(
+                            request, UUID.randomUUID(), "a".repeat(64), ignored -> null),
+                    DuplicateBetException.class)
+                .errorCode())
+        .isEqualTo(ErrorCode.DUPLICATE_BET);
+  }
+}


## `feat(placement): claim idempotent placement outcomes`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
new file mode 100644
index 0000000..9d1bdef
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -0,0 +1,88 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.MarketClosedException;
+import com.sportsbook.betting.error.OddsDriftException;
+import com.sportsbook.betting.error.ValidationFailedException;
+import java.time.Clock;
+import java.util.Optional;
+import org.springframework.dao.DataIntegrityViolationException;
+import org.springframework.stereotype.Service;
+
+@Service
+public class BetPlacementService {
+
+  private final BetAssembler assembler;
+  private final BetStore store;
+  private final Clock clock;
+
+  public BetPlacementService(BetAssembler assembler, BetStore store, Clock clock) {
+    this.assembler = assembler;
+    this.store = store;
+    this.clock = clock;
+  }
+
+  public Bet place(PlaceBetCommand command) {
+    String key = command.idempotencyKey().value();
+    String fingerprint = RequestFingerprint.of(command);
+    Optional<PlacementRequest> request = store.findPlacementRequest(key);
+    if (request.isPresent()) {
+      return PlacementReplay.request(request.get(), command.userId(), fingerprint, store::findById);
+    }
+    Optional<Bet> legacyBet = store.findByIdempotencyKey(key);
+    if (legacyBet.isPresent()) {
+      return PlacementReplay.bet(legacyBet.get(), command.userId(), fingerprint);
+    }
+
+    Bet bet;
+    try {
+      bet = assembler.assemble(command, fingerprint);
+    } catch (BetPlacementException rejection) {
+      if (!isDurablePreflight(rejection)) {
+        throw rejection;
+      }
+      try {
+        store.savePreflightRejection(
+            key,
+            command.userId(),
+            fingerprint,
+            rejection.errorCode(),
+            rejection.getMessage(),
+            clock.instant());
+      } catch (DataIntegrityViolationException collision) {
+        return replayKnown(key, command, fingerprint, collision);
+      }
+      throw rejection;
+    }
+
+    try {
+      store.savePending(bet);
+    } catch (DataIntegrityViolationException collision) {
+      return replayKnown(key, command, fingerprint, collision);
+    }
+    return bet;
+  }
+
+  private Bet replayKnown(
+      String key,
+      PlaceBetCommand command,
+      String fingerprint,
+      DataIntegrityViolationException collision) {
+    Optional<PlacementRequest> request = store.findPlacementRequest(key);
+    if (request.isPresent()) {
+      return PlacementReplay.request(request.get(), command.userId(), fingerprint, store::findById);
+    }
+    Optional<Bet> legacyBet = store.findByIdempotencyKey(key);
+    if (legacyBet.isPresent()) {
+      return PlacementReplay.bet(legacyBet.get(), command.userId(), fingerprint);
+    }
+    throw collision;
+  }
+
+  private static boolean isDurablePreflight(BetPlacementException rejection) {
+    return rejection instanceof ValidationFailedException
+        || rejection instanceof MarketClosedException
+        || rejection instanceof OddsDriftException;
+  }
+}


## `test(placement): verify idempotent outcome replay`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
new file mode 100644
index 0000000..27a3e0c
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
@@ -0,0 +1,86 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.catchThrowableOfType;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.error.PersistedRejectionException;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetPlacementServiceTest {
+
+  @Test
+  void replaysOwnedVerdictBeforeRepeatingValidationOrSideEffects() {
+    BetAssembler assembler = mock(BetAssembler.class);
+    BetStore store = mock(BetStore.class);
+    PlaceBetCommand command = command();
+    String fingerprint = RequestFingerprint.of(command);
+    PlacementRequest request =
+        PlacementRequest.rejected(
+            "request-1",
+            command.userId(),
+            fingerprint,
+            ErrorCode.VALIDATION_FAILED,
+            "saved verdict",
+            Instant.EPOCH);
+    when(store.findPlacementRequest("request-1")).thenReturn(Optional.of(request));
+    BetPlacementService service =
+        new BetPlacementService(assembler, store, Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
+
+    catchThrowableOfType(() -> service.place(command), PersistedRejectionException.class);
+
+    verifyNoInteractions(assembler);
+  }
+
+  @Test
+  void persistsDuplicateSelectionValidationBeforeCreatingABet() {
+    BetAssembler assembler = mock(BetAssembler.class);
+    BetStore store = mock(BetStore.class);
+    PlaceBetCommand command = command();
+    when(assembler.assemble(eq(command), any(String.class)))
+        .thenThrow(
+            new com.sportsbook.betting.error.ValidationFailedException(
+                "Duplicate selection is not allowed"));
+
+    BetPlacementService service =
+        new BetPlacementService(assembler, store, Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
+    catchThrowableOfType(
+        () -> service.place(command),
+        com.sportsbook.betting.error.ValidationFailedException.class);
+
+    org.mockito.Mockito.verify(store)
+        .savePreflightRejection(
+            eq("request-1"),
+            eq(command.userId()),
+            any(String.class),
+            eq(ErrorCode.VALIDATION_FAILED),
+            eq("Duplicate selection is not allowed"),
+            eq(Instant.EPOCH));
+    org.mockito.Mockito.verify(store, org.mockito.Mockito.never()).savePending(any());
+  }
+
+  private static PlaceBetCommand command() {
+    return new PlaceBetCommand(
+        UUID.randomUUID(),
+        new BetSlipType.Single(),
+        List.of(
+            new PlaceBetCommand.SelectionInput(
+                UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"))),
+        Money.krw(1_000),
+        IdempotencyKey.of("request-1"));
+  }
+}
diff --git a/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
new file mode 100644
index 0000000..fdbd0b1
--- /dev/null
+++ b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
@@ -0,0 +1 @@
+mock-maker-subclass


## `feat(placement): cache completed idempotency markers`

diff --git a/src/main/java/com/sportsbook/betting/placement/IdempotencyCache.java b/src/main/java/com/sportsbook/betting/placement/IdempotencyCache.java
new file mode 100644
index 0000000..dd57921
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/IdempotencyCache.java
@@ -0,0 +1,32 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.time.Duration;
+import java.util.UUID;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class IdempotencyCache {
+
+  private static final Logger LOG = LoggerFactory.getLogger(IdempotencyCache.class);
+  static final Duration TTL = Duration.ofHours(24);
+  static final String PREFIX = "idempotency:betting:";
+
+  private final StringRedisTemplate redis;
+
+  public IdempotencyCache(StringRedisTemplate redis) {
+    this.redis = redis;
+  }
+
+  public void markProcessed(IdempotencyKey key, UUID betId) {
+    try {
+      redis.opsForValue().set(PREFIX + key.value(), betId.toString(), TTL);
+    } catch (DataAccessException exception) {
+      LOG.warn("Could not update idempotency cache for bet {}", betId);
+    }
+  }
+}


## `test(placement): verify completed marker cache`

diff --git a/src/test/java/com/sportsbook/betting/placement/IdempotencyCacheTest.java b/src/test/java/com/sportsbook/betting/placement/IdempotencyCacheTest.java
new file mode 100644
index 0000000..86c7443
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/IdempotencyCacheTest.java
@@ -0,0 +1,27 @@
+package com.sportsbook.betting.placement;
+
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.ValueOperations;
+
+class IdempotencyCacheTest {
+
+  @Test
+  void marksOnlyCompletedBetIdentity() {
+    StringRedisTemplate redis = mock(StringRedisTemplate.class);
+    @SuppressWarnings("unchecked")
+    ValueOperations<String, String> values = mock(ValueOperations.class);
+    when(redis.opsForValue()).thenReturn(values);
+    UUID betId = UUID.randomUUID();
+
+    new IdempotencyCache(redis).markProcessed(IdempotencyKey.of("request-1"), betId);
+
+    verify(values).set("idempotency:betting:request-1", betId.toString(), IdempotencyCache.TTL);
+  }
+}
