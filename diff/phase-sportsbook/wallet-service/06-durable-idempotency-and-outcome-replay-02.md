## `feat(locking): acquire transaction-scoped idempotency locks`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java b/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java
new file mode 100644
index 0000000..4999660
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java
@@ -0,0 +1,37 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.Objects;
+import org.springframework.dao.DataAccessException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+
+/** Serializes first writers for one full idempotency key until their transaction completes. */
+@Component
+public class IdempotencyKeyLock {
+
+  private static final String LOCK_NAMESPACE = "wallet:idempotency:";
+  private static final String LOCK_SQL = "select pg_advisory_xact_lock(hashtextextended(?, 0))";
+
+  private final JdbcTemplate jdbc;
+
+  public IdempotencyKeyLock(JdbcTemplate jdbc) {
+    this.jdbc = Objects.requireNonNull(jdbc, "jdbc");
+  }
+
+  public void acquire(IdempotencyKey key) {
+    Objects.requireNonNull(key, "key");
+    if (!TransactionSynchronizationManager.isActualTransactionActive()) {
+      throw new IllegalStateException("Idempotency advisory lock requires an active transaction");
+    }
+    try {
+      jdbc.query(
+          LOCK_SQL,
+          statement -> statement.setString(1, LOCK_NAMESPACE + key.value()),
+          resultSet -> null);
+    } catch (DataAccessException failure) {
+      throw PostgresFailureTranslator.translate(key, failure);
+    }
+  }
+}


## `feat(operation): resolve fingerprints inside execution transactions`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java
new file mode 100644
index 0000000..fe098cb
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java
@@ -0,0 +1,86 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.persistence.IdempotencyKeyLock;
+import com.sportsbook.wallet.persistence.PostgresFailureTranslator;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
+import java.util.Objects;
+import java.util.Optional;
+import java.util.UUID;
+import java.util.function.Function;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.support.TransactionSynchronizationManager;
+import org.springframework.transaction.support.TransactionTemplate;
+
+/** Executes first writers under a key lock and replays immutable durable outcomes without locks. */
+@Component
+public class WalletOperationExecutor {
+
+  private final WalletOperationRepository operations;
+  private final IdempotencyKeyLock keyLocks;
+  private final TransactionTemplate writeTransaction;
+
+  public WalletOperationExecutor(
+      WalletOperationRepository operations,
+      IdempotencyKeyLock keyLocks,
+      TransactionTemplate writeTransaction) {
+    this.operations = operations;
+    this.keyLocks = keyLocks;
+    this.writeTransaction = writeTransaction;
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  public WalletOperation execute(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      Function<String, WalletOperation> firstWriter) {
+    Objects.requireNonNull(firstWriter, "firstWriter");
+    WalletRequestIdentity request = new WalletRequestIdentity(key, caller, kind, userId, amount);
+    if (TransactionSynchronizationManager.isActualTransactionActive()) {
+      throw new IllegalStateException("Wallet operations require a non-transactional caller");
+    }
+
+    Optional<WalletOperation> replay = findOutcome(key);
+    if (replay.isPresent()) {
+      return request.requireMatching(replay.get());
+    }
+
+    try {
+      return Objects.requireNonNull(
+          writeTransaction.execute(
+              ignored -> {
+                keyLocks.acquire(key);
+                String fingerprint = request.fingerprint();
+                Optional<WalletOperation> winner = findOutcome(key);
+                if (winner.isPresent()) {
+                  return request.requireMatching(winner.get());
+                }
+                WalletOperation created =
+                    Objects.requireNonNull(firstWriter.apply(fingerprint), "firstWriter outcome");
+                request.requireMatching(created);
+                return operations.saveAndFlush(created);
+              }));
+    } catch (RuntimeException failedAttempt) {
+      Optional<WalletOperation> winner = findOutcome(key);
+      if (winner.isEmpty()) {
+        throw PostgresFailureTranslator.translate(key, failedAttempt);
+      }
+      return request.requireMatching(winner.get());
+    }
+  }
+
+  private Optional<WalletOperation> findOutcome(IdempotencyKey key) {
+    try {
+      return operations.findById(key.value());
+    } catch (RuntimeException failure) {
+      throw PostgresFailureTranslator.translate(key, failure);
+    }
+  }
+}


## `feat(operation): reject conflicting request identities`

diff --git a/src/main/java/com/sportsbook/wallet/domain/error/IdempotencyConflictException.java b/src/main/java/com/sportsbook/wallet/domain/error/IdempotencyConflictException.java
new file mode 100644
index 0000000..00a01f3
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/IdempotencyConflictException.java
@@ -0,0 +1,18 @@
+package com.sportsbook.wallet.domain.error;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+
+/** Raised when one idempotency key is retried with a different canonical request. */
+public final class IdempotencyConflictException extends RuntimeException {
+
+  private final String idempotencyKey;
+
+  public IdempotencyConflictException(IdempotencyKey key) {
+    super("Idempotency key was already used by another wallet request: " + key.value());
+    this.idempotencyKey = key.value();
+  }
+
+  public String idempotencyKey() {
+    return idempotencyKey;
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletRequestIdentity.java b/src/main/java/com/sportsbook/wallet/service/WalletRequestIdentity.java
new file mode 100644
index 0000000..268e753
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletRequestIdentity.java
@@ -0,0 +1,39 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Canonical transfer identity used for both first-writer validation and immutable replay. */
+record WalletRequestIdentity(
+    IdempotencyKey key, WalletCaller caller, WalletOperationKind kind, UUID userId, Money amount) {
+
+  WalletRequestIdentity {
+    Objects.requireNonNull(key, "key");
+    Objects.requireNonNull(caller, "caller");
+    Objects.requireNonNull(kind, "kind");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(amount, "amount");
+  }
+
+  String fingerprint() {
+    return OperationFingerprint.transfer(caller, kind, userId, amount).value();
+  }
+
+  WalletOperation requireMatching(WalletOperation operation) {
+    if (!operation.idempotencyKey().equals(key.value())
+        || operation.caller() != caller
+        || operation.kind() != kind
+        || !operation.userId().equals(userId)
+        || !operation.requestAmount().equals(amount)
+        || !operation.requestFingerprint().equals(fingerprint())) {
+      throw new IdempotencyConflictException(key);
+    }
+    return operation;
+  }
+}


## `test(operation): reject every conflicting request field`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletRequestIdentityFieldTest.java b/src/test/java/com/sportsbook/wallet/service/WalletRequestIdentityFieldTest.java
new file mode 100644
index 0000000..2626eb2
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/service/WalletRequestIdentityFieldTest.java
@@ -0,0 +1,76 @@
+package com.sportsbook.wallet.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class WalletRequestIdentityFieldTest {
+
+  private static final IdempotencyKey KEY = IdempotencyKey.of("identity:baseline");
+  private static final UUID USER = UUID.fromString("019b76da-a000-7000-8000-000000000030");
+  private static final Money AMOUNT = Money.krw(100L);
+
+  @Test
+  void rejectsEveryConflictingRequestFieldAndStoredFingerprint() {
+    WalletRequestIdentity baseline =
+        identity(KEY, WalletCaller.PLATFORM, WalletOperationKind.DEPOSIT, USER, AMOUNT);
+    WalletOperation matching = outcome(KEY, baseline.fingerprint());
+    List<WalletRequestIdentity> conflicts =
+        List.of(
+            identity(
+                IdempotencyKey.of("identity:other"),
+                WalletCaller.PLATFORM,
+                WalletOperationKind.DEPOSIT,
+                USER,
+                AMOUNT),
+            identity(KEY, WalletCaller.BETTING, WalletOperationKind.DEPOSIT, USER, AMOUNT),
+            identity(KEY, WalletCaller.PLATFORM, WalletOperationKind.WITHDRAW, USER, AMOUNT),
+            identity(
+                KEY,
+                WalletCaller.PLATFORM,
+                WalletOperationKind.DEPOSIT,
+                UUID.fromString("019b76da-a000-7000-8000-000000000031"),
+                AMOUNT),
+            identity(
+                KEY, WalletCaller.PLATFORM, WalletOperationKind.DEPOSIT, USER, Money.krw(101L)));
+
+    assertThat(baseline.requireMatching(matching)).isSameAs(matching);
+    conflicts.forEach(
+        conflict ->
+            assertThatThrownBy(() -> conflict.requireMatching(matching))
+                .isInstanceOf(IdempotencyConflictException.class));
+    assertThatThrownBy(() -> baseline.requireMatching(outcome(KEY, "f".repeat(64))))
+        .isInstanceOf(IdempotencyConflictException.class);
+  }
+
+  private static WalletRequestIdentity identity(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount) {
+    return new WalletRequestIdentity(key, caller, kind, userId, amount);
+  }
+
+  private static WalletOperation outcome(IdempotencyKey key, String fingerprint) {
+    return WalletOperation.succeeded(
+        key,
+        WalletCaller.PLATFORM,
+        WalletOperationKind.DEPOSIT,
+        USER,
+        AMOUNT,
+        fingerprint,
+        UUID.fromString("019b76da-a000-7000-8000-000000000032"),
+        Instant.parse("2026-01-01T00:00:00Z"));
+  }
+}


## `feat(service): replay durable rejection snapshots`

diff --git a/src/main/java/com/sportsbook/wallet/domain/error/WalletRejectedException.java b/src/main/java/com/sportsbook/wallet/domain/error/WalletRejectedException.java
new file mode 100644
index 0000000..f96bb80
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/WalletRejectedException.java
@@ -0,0 +1,19 @@
+package com.sportsbook.wallet.domain.error;
+
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import java.util.Objects;
+
+/** Replays the immutable business failure committed for an idempotent wallet request. */
+public final class WalletRejectedException extends RuntimeException {
+
+  private final WalletFailureSnapshot failure;
+
+  public WalletRejectedException(WalletFailureSnapshot failure) {
+    super(Objects.requireNonNull(failure, "failure").detail());
+    this.failure = failure;
+  }
+
+  public WalletFailureSnapshot failure() {
+    return failure;
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOutcomeResolver.java b/src/main/java/com/sportsbook/wallet/service/WalletOutcomeResolver.java
new file mode 100644
index 0000000..2bff9b6
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOutcomeResolver.java
@@ -0,0 +1,31 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationStatus;
+import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import com.sportsbook.wallet.persistence.LedgerEntryRepository;
+import java.util.Objects;
+import org.springframework.stereotype.Component;
+
+/** Converts a durable operation row into its exact success or business-rejection response. */
+@Component
+public class WalletOutcomeResolver {
+
+  private final LedgerEntryRepository ledger;
+
+  public WalletOutcomeResolver(LedgerEntryRepository ledger) {
+    this.ledger = ledger;
+  }
+
+  public WalletOperationResult resolve(WalletOperation operation) {
+    Objects.requireNonNull(operation, "operation");
+    if (operation.status() == WalletOperationStatus.REJECTED) {
+      throw new WalletRejectedException(operation.failure());
+    }
+    if (operation.status() != WalletOperationStatus.SUCCEEDED) {
+      throw new IllegalStateException("Blocked wallet operation has no final response");
+    }
+    return WalletOperationResult.fromExisting(
+        ledger.findByOperationGroupId(operation.operationGroupId()));
+  }
+}


## `test(operation): replay durable outcomes before advisory locking`

diff --git a/src/test/java/com/sportsbook/wallet/service/WalletOperationExecutorTest.java b/src/test/java/com/sportsbook/wallet/service/WalletOperationExecutorTest.java
new file mode 100644
index 0000000..b7ceee4
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/service/WalletOperationExecutorTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.wallet.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.persistence.IdempotencyKeyLock;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import java.util.function.Function;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.InjectMocks;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.transaction.support.TransactionTemplate;
+
+@ExtendWith(MockitoExtension.class)
+class WalletOperationExecutorTest {
+  @Mock WalletOperationRepository operations;
+  @Mock IdempotencyKeyLock keyLocks;
+  @Mock TransactionTemplate writeTransaction;
+  @InjectMocks WalletOperationExecutor executor;
+
+  @Test
+  void replaysTheDurableOutcomeBeforeTakingTheAdvisoryLock() {
+    IdempotencyKey key = IdempotencyKey.of("operation:durable-replay");
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000020");
+    Money amount = Money.krw(50L);
+    WalletRequestIdentity request =
+        new WalletRequestIdentity(
+            key, WalletCaller.PLATFORM, WalletOperationKind.DEPOSIT, userId, amount);
+    WalletOperation outcome =
+        WalletOperation.succeeded(
+            key,
+            request.caller(),
+            request.kind(),
+            request.userId(),
+            request.amount(),
+            request.fingerprint(),
+            UUID.fromString("019b76da-a000-7000-8000-000000000021"),
+            Instant.parse("2026-01-03T00:00:00Z"));
+    @SuppressWarnings("unchecked")
+    Function<String, WalletOperation> firstWriter = mock(Function.class);
+    when(operations.findById(key.value())).thenReturn(Optional.of(outcome));
+
+    assertThat(executor.execute(key, request.caller(), request.kind(), userId, amount, firstWriter))
+        .isSameAs(outcome);
+
+    verify(keyLocks, never()).acquire(key);
+    verify(writeTransaction, never()).execute(org.mockito.ArgumentMatchers.any());
+    verify(firstWriter, never()).apply(org.mockito.ArgumentMatchers.anyString());
+  }
+}


## `feat(cache): treat Redis idempotency data as a fallible hint`

diff --git a/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java b/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java
new file mode 100644
index 0000000..140eb17
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.time.Duration;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.dao.DataAccessException;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+/** Best-effort existence marker; PostgreSQL remains the only owner of request outcomes. */
+@Component
+public class IdempotencyCache {
+
+  static final Duration TTL = Duration.ofHours(24L);
+  static final String KEY_PREFIX = "idempotency:wallet:";
+
+  private static final Logger log = LoggerFactory.getLogger(IdempotencyCache.class);
+
+  private final StringRedisTemplate redis;
+
+  public IdempotencyCache(StringRedisTemplate redis) {
+    this.redis = redis;
+  }
+
+  public boolean mightContain(IdempotencyKey key) {
+    try {
+      return Boolean.TRUE.equals(redis.hasKey(redisKey(key)));
+    } catch (DataAccessException unavailable) {
+      log.warn("Redis idempotency lookup failed; using PostgreSQL", unavailable);
+      return false;
+    }
+  }
+
+  public void mark(IdempotencyKey key) {
+    try {
+      redis.opsForValue().set(redisKey(key), "1", TTL);
+    } catch (DataAccessException unavailable) {
+      log.warn("Redis idempotency marker failed after PostgreSQL outcome", unavailable);
+    }
+  }
+
+  private static String redisKey(IdempotencyKey key) {
+    return KEY_PREFIX + key.value();
+  }
+}


## `feat(service): cache committed outcomes without owning correctness`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java
index fe098cb..6db6193 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java
@@ -23,14 +23,17 @@ public class WalletOperationExecutor {
   private final WalletOperationRepository operations;
   private final IdempotencyKeyLock keyLocks;
   private final TransactionTemplate writeTransaction;
+  private final IdempotencyCache cache;
 
   public WalletOperationExecutor(
       WalletOperationRepository operations,
       IdempotencyKeyLock keyLocks,
-      TransactionTemplate writeTransaction) {
+      TransactionTemplate writeTransaction,
+      IdempotencyCache cache) {
     this.operations = operations;
     this.keyLocks = keyLocks;
     this.writeTransaction = writeTransaction;
+    this.cache = cache;
   }
 
   @SuppressWarnings("checkstyle:ParameterNumber")
@@ -47,32 +50,58 @@ public class WalletOperationExecutor {
       throw new IllegalStateException("Wallet operations require a non-transactional caller");
     }
 
+    boolean hinted = cacheHint(key);
     Optional<WalletOperation> replay = findOutcome(key);
     if (replay.isPresent()) {
+      if (!hinted) {
+        cacheMark(key);
+      }
       return request.requireMatching(replay.get());
     }
 
     try {
-      return Objects.requireNonNull(
-          writeTransaction.execute(
-              ignored -> {
-                keyLocks.acquire(key);
-                String fingerprint = request.fingerprint();
-                Optional<WalletOperation> winner = findOutcome(key);
-                if (winner.isPresent()) {
-                  return request.requireMatching(winner.get());
-                }
-                WalletOperation created =
-                    Objects.requireNonNull(firstWriter.apply(fingerprint), "firstWriter outcome");
-                request.requireMatching(created);
-                return operations.saveAndFlush(created);
-              }));
+      WalletOperation outcome =
+          Objects.requireNonNull(
+              writeTransaction.execute(
+                  ignored -> {
+                    keyLocks.acquire(key);
+                    String fingerprint = request.fingerprint();
+                    Optional<WalletOperation> winner = findOutcome(key);
+                    if (winner.isPresent()) {
+                      return request.requireMatching(winner.get());
+                    }
+                    WalletOperation created =
+                        Objects.requireNonNull(
+                            firstWriter.apply(fingerprint), "firstWriter outcome");
+                    request.requireMatching(created);
+                    return operations.saveAndFlush(created);
+                  }));
+      cacheMark(key);
+      return outcome;
     } catch (RuntimeException failedAttempt) {
       Optional<WalletOperation> winner = findOutcome(key);
       if (winner.isEmpty()) {
         throw PostgresFailureTranslator.translate(key, failedAttempt);
       }
-      return request.requireMatching(winner.get());
+      WalletOperation outcome = request.requireMatching(winner.get());
+      cacheMark(key);
+      return outcome;
+    }
+  }
+
+  private boolean cacheHint(IdempotencyKey key) {
+    try {
+      return cache.mightContain(key);
+    } catch (RuntimeException ignored) {
+      return false;
+    }
+  }
+
+  private void cacheMark(IdempotencyKey key) {
+    try {
+      cache.mark(key);
+    } catch (RuntimeException ignored) {
+      // Cache state cannot change a committed PostgreSQL outcome.
     }
   }
 


