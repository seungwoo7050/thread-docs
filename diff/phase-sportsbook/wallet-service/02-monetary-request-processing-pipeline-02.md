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


## `feat(service): execute durable transfer outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
new file mode 100644
index 0000000..56d1a6e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -0,0 +1,97 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.error.AccountNotFoundException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.persistence.AccountRepository;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.temporal.ChronoUnit;
+import java.util.UUID;
+import java.util.function.BiFunction;
+import org.springframework.stereotype.Component;
+
+/** Runs account mutation, matched ledger pair, and authoritative outcome in one transaction. */
+@Component
+public class WalletTransferExecutor {
+
+  private final AccountRepository accounts;
+  private final WalletOperationExecutor operations;
+  private final WalletTransferWriter transfers;
+  private final WalletOutcomeResolver outcomes;
+  private final Clock clock;
+
+  public WalletTransferExecutor(
+      AccountRepository accounts,
+      WalletOperationExecutor operations,
+      WalletTransferWriter transfers,
+      WalletOutcomeResolver outcomes,
+      Clock clock) {
+    this.accounts = accounts;
+    this.operations = operations;
+    this.transfers = transfers;
+    this.outcomes = outcomes;
+    this.clock = clock;
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  public WalletOperationResult execute(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    WalletOperation operation =
+        operations.execute(
+            key,
+            caller,
+            kind,
+            userId,
+            amount,
+            fingerprint -> firstWrite(key, caller, kind, userId, amount, fingerprint, mutation));
+    return outcomes.resolve(operation);
+  }
+
+  @SuppressWarnings("checkstyle:ParameterNumber")
+  private WalletOperation firstWrite(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      BiFunction<Account, Instant, WalletTransferPlan> mutation) {
+    Instant now = clock.instant().truncatedTo(ChronoUnit.MICROS);
+    try {
+      Account account = lockAccount(userId, amount);
+      WalletTransferPlan plan = mutation.apply(account, now);
+      WalletOperationResult result =
+          transfers.write(
+              plan.destination(), plan.source(), amount, plan.reason(), key, userId, now);
+      return WalletOperation.succeeded(
+          key, caller, kind, userId, amount, fingerprint, result.operationGroupId(), now);
+    } catch (RuntimeException businessOrInfrastructure) {
+      WalletFailureSnapshot failure =
+          WalletFailureMapper.snapshot(businessOrInfrastructure, amount);
+      return WalletOperation.rejected(key, caller, kind, userId, amount, fingerprint, failure, now);
+    }
+  }
+
+  private Account lockAccount(UUID userId, Money amount) {
+    Account account =
+        accounts
+            .findByUserIdForUpdate(userId)
+            .orElseThrow(() -> new AccountNotFoundException(userId));
+    if (account.currency() != amount.currency()) {
+      throw new CurrencyMismatchException(account.currency(), amount.currency());
+    }
+    return account;
+  }
+}


## `feat(errors): map durable and retryable failures`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
new file mode 100644
index 0000000..871f2b6
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
@@ -0,0 +1,49 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import com.sportsbook.wallet.domain.error.WalletBusyException;
+import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import jakarta.servlet.http.HttpServletRequest;
+import java.net.URI;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.ProblemDetail;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.ExceptionHandler;
+import org.springframework.web.bind.annotation.RestControllerAdvice;
+
+/** Maps durable and retryable wallet failures without exposing request credentials. */
+@RestControllerAdvice
+public class WalletExceptionHandler {
+
+  @ExceptionHandler(WalletRejectedException.class)
+  ProblemDetail rejected(WalletRejectedException exception, HttpServletRequest request) {
+    return atRequest(WalletProblems.from(exception.failure()), request);
+  }
+
+  @ExceptionHandler(IdempotencyConflictException.class)
+  ProblemDetail idempotencyConflict(
+      IdempotencyConflictException exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.IDEMPOTENCY_CONFLICT,
+            "Idempotency key belongs to a different wallet request"),
+        request);
+  }
+
+  @ExceptionHandler(WalletBusyException.class)
+  ResponseEntity<ProblemDetail> busy(WalletBusyException exception, HttpServletRequest request) {
+    ProblemDetail problem =
+        atRequest(
+            WalletProblems.from(
+                WalletError.WALLET_BUSY, "Retry the wallet request after one second"),
+            request);
+    return ResponseEntity.status(WalletError.WALLET_BUSY.httpStatus())
+        .header(HttpHeaders.RETRY_AFTER, "1")
+        .body(problem);
+  }
+
+  private ProblemDetail atRequest(ProblemDetail problem, HttpServletRequest request) {
+    problem.setInstance(URI.create(request.getRequestURI()));
+    return problem;
+  }
+}
