# 안정적 오류 응답 계약

## `feat(errors): define wallet error vocabulary`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletError.java b/src/main/java/com/sportsbook/wallet/web/WalletError.java
new file mode 100644
index 0000000..1d353a7
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/WalletError.java
@@ -0,0 +1,60 @@
+package com.sportsbook.wallet.web;
+
+import java.net.URI;
+
+/** Stable HTTP problem vocabulary for wallet-specific failures. */
+public enum WalletError {
+  INVALID_REQUEST(400, "WALLET_INVALID_REQUEST", "Invalid wallet request", "invalid-request"),
+  AUTHENTICATION_REQUIRED(
+      401, "WALLET_AUTHENTICATION_REQUIRED", "Authentication required", "authentication-required"),
+  ACCESS_DENIED(403, "WALLET_ACCESS_DENIED", "Wallet access denied", "access-denied"),
+  ACCOUNT_NOT_FOUND(404, "WALLET_ACCOUNT_NOT_FOUND", "Account not found", "account-not-found"),
+  OPERATION_NOT_FOUND(
+      404, "WALLET_OPERATION_NOT_FOUND", "Wallet operation not found", "operation-not-found"),
+  ADJUSTMENT_NOT_FOUND(
+      404, "WALLET_ADJUSTMENT_NOT_FOUND", "Wallet adjustment not found", "adjustment-not-found"),
+  IDEMPOTENCY_CONFLICT(
+      409, "WALLET_IDEMPOTENCY_CONFLICT", "Idempotency key conflict", "idempotency-conflict"),
+  CURRENCY_MISMATCH(422, "WALLET_CURRENCY_MISMATCH", "Currency mismatch", "currency-mismatch"),
+  INSUFFICIENT_BALANCE(
+      422, "WALLET_INSUFFICIENT_BALANCE", "Insufficient balance", "insufficient-balance"),
+  AMOUNT_OUT_OF_RANGE(
+      422, "WALLET_AMOUNT_OUT_OF_RANGE", "Amount out of range", "amount-out-of-range"),
+  ACCOUNT_RECOVERY_BLOCKED(
+      423,
+      "WALLET_ACCOUNT_RECOVERY_BLOCKED",
+      "Wallet account blocked for recovery",
+      "account-recovery-blocked"),
+  INTERNAL_ERROR(500, "WALLET_INTERNAL_ERROR", "Internal server error", "internal-error"),
+  WALLET_BUSY(503, "WALLET_BUSY", "Wallet temporarily busy", "busy");
+
+  private static final String TYPE_BASE = "https://sportsbook/errors/wallet/";
+
+  private final int httpStatus;
+  private final String errorCode;
+  private final String title;
+  private final URI type;
+
+  WalletError(int httpStatus, String errorCode, String title, String slug) {
+    this.httpStatus = httpStatus;
+    this.errorCode = errorCode;
+    this.title = title;
+    this.type = URI.create(TYPE_BASE + slug);
+  }
+
+  public int httpStatus() {
+    return httpStatus;
+  }
+
+  public String errorCode() {
+    return errorCode;
+  }
+
+  public String title() {
+    return title;
+  }
+
+  public URI type() {
+    return type;
+  }
+}


## `test(errors): lock wallet error vocabulary`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletErrorTest.java b/src/test/java/com/sportsbook/wallet/web/WalletErrorTest.java
new file mode 100644
index 0000000..1b30300
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletErrorTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.groups.Tuple.tuple;
+
+import org.junit.jupiter.api.Test;
+
+class WalletErrorTest {
+  private static final String BASE = "https://sportsbook/errors/wallet/";
+
+  @Test
+  void locksWalletErrorVocabulary() {
+    assertThat(WalletError.values())
+        .extracting(
+            WalletError::httpStatus,
+            WalletError::errorCode,
+            WalletError::title,
+            error -> error.type().toString())
+        .containsExactly(
+            tuple(
+                400, "WALLET_INVALID_REQUEST", "Invalid wallet request", BASE + "invalid-request"),
+            tuple(
+                401,
+                "WALLET_AUTHENTICATION_REQUIRED",
+                "Authentication required",
+                BASE + "authentication-required"),
+            tuple(403, "WALLET_ACCESS_DENIED", "Wallet access denied", BASE + "access-denied"),
+            tuple(404, "WALLET_ACCOUNT_NOT_FOUND", "Account not found", BASE + "account-not-found"),
+            tuple(
+                404,
+                "WALLET_OPERATION_NOT_FOUND",
+                "Wallet operation not found",
+                BASE + "operation-not-found"),
+            tuple(
+                404,
+                "WALLET_ADJUSTMENT_NOT_FOUND",
+                "Wallet adjustment not found",
+                BASE + "adjustment-not-found"),
+            tuple(
+                409,
+                "WALLET_IDEMPOTENCY_CONFLICT",
+                "Idempotency key conflict",
+                BASE + "idempotency-conflict"),
+            tuple(422, "WALLET_CURRENCY_MISMATCH", "Currency mismatch", BASE + "currency-mismatch"),
+            tuple(
+                422,
+                "WALLET_INSUFFICIENT_BALANCE",
+                "Insufficient balance",
+                BASE + "insufficient-balance"),
+            tuple(
+                422,
+                "WALLET_AMOUNT_OUT_OF_RANGE",
+                "Amount out of range",
+                BASE + "amount-out-of-range"),
+            tuple(
+                423,
+                "WALLET_ACCOUNT_RECOVERY_BLOCKED",
+                "Wallet account blocked for recovery",
+                BASE + "account-recovery-blocked"),
+            tuple(500, "WALLET_INTERNAL_ERROR", "Internal server error", BASE + "internal-error"),
+            tuple(503, "WALLET_BUSY", "Wallet temporarily busy", BASE + "busy"));
+  }
+}


## `feat(errors): shape wallet problem details`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletProblems.java b/src/main/java/com/sportsbook/wallet/web/WalletProblems.java
new file mode 100644
index 0000000..d36b9d8
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/WalletProblems.java
@@ -0,0 +1,52 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.wallet.domain.WalletFailureCode;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import java.util.Objects;
+import org.springframework.http.HttpStatusCode;
+import org.springframework.http.ProblemDetail;
+
+/** Builds stable RFC 9457 bodies without consulting mutable wallet state. */
+public final class WalletProblems {
+  public static final String ERROR_CODE = "errorCode";
+
+  private WalletProblems() {}
+
+  public static ProblemDetail from(WalletError error, String detail) {
+    Objects.requireNonNull(error, "error");
+    ProblemDetail problem =
+        ProblemDetail.forStatusAndDetail(
+            HttpStatusCode.valueOf(error.httpStatus()), Objects.requireNonNull(detail, "detail"));
+    problem.setType(error.type());
+    problem.setTitle(error.title());
+    problem.setProperty(ERROR_CODE, error.errorCode());
+    return problem;
+  }
+
+  public static ProblemDetail from(WalletFailureSnapshot failure) {
+    Objects.requireNonNull(failure, "failure");
+    ProblemDetail problem =
+        ProblemDetail.forStatusAndDetail(
+            HttpStatusCode.valueOf(failure.httpStatus()), failure.detail());
+    problem.setType(errorFor(failure.code()).type());
+    problem.setTitle(failure.title());
+    problem.setProperty(ERROR_CODE, failure.code().wireCode());
+    if (failure.balance() != null) {
+      problem.setProperty("balance", failure.balance());
+    }
+    if (failure.expectedCurrency() != null) {
+      problem.setProperty("expectedCurrency", failure.expectedCurrency());
+    }
+    return problem;
+  }
+
+  private static WalletError errorFor(WalletFailureCode code) {
+    return switch (code) {
+      case ACCOUNT_NOT_FOUND -> WalletError.ACCOUNT_NOT_FOUND;
+      case CURRENCY_MISMATCH -> WalletError.CURRENCY_MISMATCH;
+      case INSUFFICIENT_BALANCE -> WalletError.INSUFFICIENT_BALANCE;
+      case ACCOUNT_SUSPENDED -> WalletError.ACCOUNT_RECOVERY_BLOCKED;
+      case AMOUNT_OUT_OF_RANGE -> WalletError.AMOUNT_OUT_OF_RANGE;
+    };
+  }
+}


## `test(errors): verify wallet problem shapes`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletProblemsTest.java b/src/test/java/com/sportsbook/wallet/web/WalletProblemsTest.java
new file mode 100644
index 0000000..3b5f12b
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletProblemsTest.java
@@ -0,0 +1,94 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+import static org.junit.jupiter.params.provider.Arguments.arguments;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletFailureCode;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import java.util.Map;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.Arguments;
+import org.junit.jupiter.params.provider.MethodSource;
+import org.springframework.http.ProblemDetail;
+
+class WalletProblemsTest {
+
+  @Test
+  void shapesGenericWalletProblems() {
+    ProblemDetail problem =
+        WalletProblems.from(WalletError.ACCESS_DENIED, "Caller cannot use route");
+
+    assertThat(problem.getStatus()).isEqualTo(403);
+    assertThat(problem.getType()).isEqualTo(WalletError.ACCESS_DENIED.type());
+    assertThat(problem.getTitle()).isEqualTo("Wallet access denied");
+    assertThat(problem.getDetail()).isEqualTo("Caller cannot use route");
+    assertThat(problem.getInstance()).isNull();
+    assertThat(problem.getProperties())
+        .containsExactly(Map.entry(WalletProblems.ERROR_CODE, "WALLET_ACCESS_DENIED"));
+  }
+
+  @ParameterizedTest
+  @MethodSource("failureTypes")
+  void mapsEveryDurableFailureType(WalletFailureCode code, WalletError error) {
+    WalletFailureSnapshot failure = WalletFailureSnapshot.of(code, "stored detail");
+
+    ProblemDetail problem = WalletProblems.from(failure);
+
+    assertThat(problem.getStatus()).isEqualTo(code.httpStatus());
+    assertThat(problem.getType()).isEqualTo(error.type());
+    assertThat(problem.getTitle()).isEqualTo(code.title());
+    assertThat(problem.getDetail()).isEqualTo("stored detail");
+    assertThat(problem.getProperties())
+        .containsExactly(Map.entry(WalletProblems.ERROR_CODE, code.wireCode()));
+  }
+
+  @Test
+  void addsOnlyPersistedBalanceFacts() {
+    Money balance = Money.krw(700L);
+    ProblemDetail problem =
+        WalletProblems.from(
+            WalletFailureSnapshot.withBalance(
+                WalletFailureCode.INSUFFICIENT_BALANCE, "stored balance", balance));
+
+    assertThat(problem.getProperties())
+        .containsEntry(WalletProblems.ERROR_CODE, "WALLET_INSUFFICIENT_BALANCE")
+        .containsEntry("balance", balance)
+        .doesNotContainKey("expectedCurrency");
+  }
+
+  @Test
+  void addsOnlyPersistedExpectedCurrencyFacts() {
+    ProblemDetail problem =
+        WalletProblems.from(
+            WalletFailureSnapshot.currencyMismatch("stored currency", Currency.USD));
+
+    assertThat(problem.getProperties())
+        .containsEntry(WalletProblems.ERROR_CODE, "WALLET_CURRENCY_MISMATCH")
+        .containsEntry("expectedCurrency", Currency.USD)
+        .doesNotContainKey("balance");
+  }
+
+  @Test
+  void rejectsMissingProblemInputs() {
+    assertThatNullPointerException()
+        .isThrownBy(() -> WalletProblems.from((WalletError) null, "detail"));
+    assertThatNullPointerException()
+        .isThrownBy(() -> WalletProblems.from(WalletError.INVALID_REQUEST, null));
+    assertThatNullPointerException()
+        .isThrownBy(() -> WalletProblems.from((WalletFailureSnapshot) null));
+  }
+
+  private static Stream<Arguments> failureTypes() {
+    return Stream.of(
+        arguments(WalletFailureCode.ACCOUNT_NOT_FOUND, WalletError.ACCOUNT_NOT_FOUND),
+        arguments(WalletFailureCode.CURRENCY_MISMATCH, WalletError.CURRENCY_MISMATCH),
+        arguments(WalletFailureCode.INSUFFICIENT_BALANCE, WalletError.INSUFFICIENT_BALANCE),
+        arguments(WalletFailureCode.ACCOUNT_SUSPENDED, WalletError.ACCOUNT_RECOVERY_BLOCKED),
+        arguments(WalletFailureCode.AMOUNT_OUT_OF_RANGE, WalletError.AMOUNT_OUT_OF_RANGE));
+  }
+}


## `feat(errors): identify missing wallet proofs`

diff --git a/src/main/java/com/sportsbook/wallet/domain/error/WalletAdjustmentNotFoundException.java b/src/main/java/com/sportsbook/wallet/domain/error/WalletAdjustmentNotFoundException.java
new file mode 100644
index 0000000..58ed94d
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/WalletAdjustmentNotFoundException.java
@@ -0,0 +1,20 @@
+package com.sportsbook.wallet.domain.error;
+
+import java.util.Objects;
+import java.util.UUID;
+
+/** Raised when a settlement revision has no durable wallet adjustment proof. */
+public final class WalletAdjustmentNotFoundException extends RuntimeException {
+  private final UUID revisionId;
+
+  public WalletAdjustmentNotFoundException(UUID revisionId) {
+    super(
+        "No wallet adjustment exists for revision "
+            + Objects.requireNonNull(revisionId, "revisionId"));
+    this.revisionId = revisionId;
+  }
+
+  public UUID revisionId() {
+    return revisionId;
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/domain/error/WalletOperationNotFoundException.java b/src/main/java/com/sportsbook/wallet/domain/error/WalletOperationNotFoundException.java
new file mode 100644
index 0000000..ba01663
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/WalletOperationNotFoundException.java
@@ -0,0 +1,18 @@
+package com.sportsbook.wallet.domain.error;
+
+import java.util.Objects;
+import java.util.UUID;
+
+/** Raised when a bet has no durable wallet debit proof. */
+public final class WalletOperationNotFoundException extends RuntimeException {
+  private final UUID betId;
+
+  public WalletOperationNotFoundException(UUID betId) {
+    super("No wallet debit exists for bet " + Objects.requireNonNull(betId, "betId"));
+    this.betId = betId;
+  }
+
+  public UUID betId() {
+    return betId;
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


## `test(errors): replay persisted wallet failures`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerReplayTest.java b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerReplayTest.java
new file mode 100644
index 0000000..bb01c73
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerReplayTest.java
@@ -0,0 +1,82 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletFailureCode;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import java.net.URI;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.ProblemDetail;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class WalletExceptionHandlerReplayTest {
+  private final WalletExceptionHandler handler = new WalletExceptionHandler();
+
+  @Test
+  void replaysOnlyPersistedRejectionFacts() {
+    Money storedBalance = Money.krw(700L);
+    WalletFailureSnapshot stored =
+        WalletFailureSnapshot.withBalance(
+            WalletFailureCode.INSUFFICIENT_BALANCE, "stored balance at rejection", storedBalance);
+    MockHttpServletRequest request = request("/internal/v1/wallet/transactions/debit");
+    request.setQueryString("apiKey=must-not-become-instance");
+
+    ProblemDetail first = handler.rejected(new WalletRejectedException(stored), request);
+    ProblemDetail replay = handler.rejected(new WalletRejectedException(stored), request);
+
+    assertThat(first.getStatus()).isEqualTo(422);
+    assertThat(first.getTitle()).isEqualTo("Insufficient balance");
+    assertThat(first.getDetail()).isEqualTo("stored balance at rejection");
+    assertThat(first.getType())
+        .isEqualTo(URI.create("https://sportsbook/errors/wallet/insufficient-balance"));
+    assertThat(first.getInstance()).isEqualTo(URI.create(request.getRequestURI()));
+    assertThat(first.getProperties())
+        .containsEntry("errorCode", "WALLET_INSUFFICIENT_BALANCE")
+        .containsEntry("balance", storedBalance)
+        .doesNotContainKey("expectedCurrency");
+    assertThat(replay.getProperties()).isEqualTo(first.getProperties());
+    assertThat(replay.getDetail()).isEqualTo(first.getDetail());
+  }
+
+  @Test
+  void preservesPersistedCurrencyMismatchFacts() {
+    WalletFailureSnapshot stored =
+        WalletFailureSnapshot.currencyMismatch("stored currency mismatch", Currency.KRW);
+
+    ProblemDetail replay =
+        handler.rejected(
+            new WalletRejectedException(stored),
+            request("/internal/v1/wallet/transactions/credit"));
+
+    assertThat(replay.getStatus()).isEqualTo(422);
+    assertThat(replay.getProperties())
+        .containsEntry("errorCode", "WALLET_CURRENCY_MISMATCH")
+        .containsEntry("expectedCurrency", Currency.KRW)
+        .doesNotContainKey("balance");
+  }
+
+  @Test
+  void reportsConflictsWithoutReflectingTheirIdentity() {
+    IdempotencyKey key = IdempotencyKey.of("secret:conflicting-operation-key");
+    IdempotencyConflictException conflict = new IdempotencyConflictException(key);
+
+    ProblemDetail problem =
+        handler.idempotencyConflict(conflict, request("/internal/v1/wallet/transactions/deposit"));
+
+    assertThat(problem.getStatus()).isEqualTo(409);
+    assertThat(problem.getTitle()).isEqualTo("Idempotency key conflict");
+    assertThat(problem.getDetail())
+        .isEqualTo("Idempotency key belongs to a different wallet request");
+    assertThat(problem.getProperties()).containsEntry("errorCode", "WALLET_IDEMPOTENCY_CONFLICT");
+    assertThat(problem.toString()).doesNotContain(key.value(), conflict.getMessage());
+  }
+
+  private MockHttpServletRequest request(String path) {
+    return new MockHttpServletRequest("POST", path);
+  }
+}


