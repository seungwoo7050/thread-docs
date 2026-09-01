# 내부 지갑 HTTP 계약

## `feat(api): define account representations`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/AccountResponse.java b/src/main/java/com/sportsbook/wallet/web/dto/AccountResponse.java
new file mode 100644
index 0000000..5dd341b
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/AccountResponse.java
@@ -0,0 +1,33 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Public account snapshot without recovery queue internals. */
+public record AccountResponse(
+    UUID userId,
+    Currency currency,
+    Money available,
+    Money locked,
+    boolean outboundFrozen,
+    long version,
+    Instant createdAt,
+    Instant updatedAt) {
+
+  public static AccountResponse from(Account account) {
+    Objects.requireNonNull(account, "account");
+    return new AccountResponse(
+        account.userId(),
+        account.currency(),
+        account.available(),
+        account.locked(),
+        account.isOutboundFrozen(),
+        account.version(),
+        account.createdAt(),
+        account.updatedAt());
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/web/dto/OpenAccountRequest.java b/src/main/java/com/sportsbook/wallet/web/dto/OpenAccountRequest.java
new file mode 100644
index 0000000..d841ccb
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/OpenAccountRequest.java
@@ -0,0 +1,8 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Currency;
+import jakarta.validation.constraints.NotNull;
+import java.util.UUID;
+
+/** Request body for opening an empty wallet account. */
+public record OpenAccountRequest(@NotNull UUID userId, @NotNull Currency currency) {}


## `feat(api): define balance representations`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/BalanceResponse.java b/src/main/java/com/sportsbook/wallet/web/dto/BalanceResponse.java
new file mode 100644
index 0000000..addbd93
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/BalanceResponse.java
@@ -0,0 +1,21 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Public balance view without recovery debt details. */
+public record BalanceResponse(
+    UUID userId, Money available, Money locked, Money total, boolean outboundFrozen) {
+
+  public static BalanceResponse from(Account account) {
+    Objects.requireNonNull(account, "account");
+    return new BalanceResponse(
+        account.userId(),
+        account.available(),
+        account.locked(),
+        account.total(),
+        account.isOutboundFrozen());
+  }
+}


## `feat(api): define transaction representations`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/TransactionRequest.java b/src/main/java/com/sportsbook/wallet/web/dto/TransactionRequest.java
new file mode 100644
index 0000000..98b5108
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/TransactionRequest.java
@@ -0,0 +1,8 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Money;
+import jakarta.validation.constraints.NotNull;
+import java.util.UUID;
+
+/** Shared request body for a single positive wallet transfer. */
+public record TransactionRequest(@NotNull UUID userId, @NotNull Money amount) {}
diff --git a/src/main/java/com/sportsbook/wallet/web/dto/WalletOperationResponse.java b/src/main/java/com/sportsbook/wallet/web/dto/WalletOperationResponse.java
new file mode 100644
index 0000000..fc2ce5c
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/WalletOperationResponse.java
@@ -0,0 +1,19 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.service.WalletOperationResult;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Public view of one authoritative matched ledger transfer. */
+public record WalletOperationResponse(
+    UUID operationGroupId, UUID userId, Money amount, LedgerReason reason, Instant at) {
+
+  public static WalletOperationResponse from(WalletOperationResult result) {
+    Objects.requireNonNull(result, "result");
+    return new WalletOperationResponse(
+        result.operationGroupId(), result.userId(), result.amount(), result.reason(), result.at());
+  }
+}


## `feat(api): parse wallet request identities`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java b/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java
new file mode 100644
index 0000000..4ffffdc
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/WalletRequestHeaders.java
@@ -0,0 +1,47 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.Enumeration;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Parses single-valued wallet request identity headers without normalization. */
+public final class WalletRequestHeaders {
+  public static final String IDEMPOTENCY_KEY = "Idempotency-Key";
+
+  private WalletRequestHeaders() {}
+
+  public static IdempotencyKey requireIdempotencyKey(HttpServletRequest request) {
+    Objects.requireNonNull(request, "request");
+    Enumeration<String> values = request.getHeaders(IDEMPOTENCY_KEY);
+    if (values == null || !values.hasMoreElements()) {
+      throw new IllegalArgumentException("Exactly one Idempotency-Key header is required");
+    }
+    String value = values.nextElement();
+    if (values.hasMoreElements()) {
+      throw new IllegalArgumentException("Exactly one Idempotency-Key header is required");
+    }
+    return IdempotencyKey.of(value);
+  }
+
+  public static IdempotencyKey requireCanonicalDebitKey(HttpServletRequest request) {
+    IdempotencyKey key = requireIdempotencyKey(request);
+    requireCanonicalDebitId(key.value());
+    return key;
+  }
+
+  public static UUID requireCanonicalDebitId(String raw) {
+    Objects.requireNonNull(raw, "raw");
+    UUID parsed;
+    try {
+      parsed = UUID.fromString(raw);
+    } catch (IllegalArgumentException invalid) {
+      throw new IllegalArgumentException("Debit identity must be a canonical UUID");
+    }
+    if (!parsed.toString().equals(raw)) {
+      throw new IllegalArgumentException("Debit identity must be a canonical UUID");
+    }
+    return parsed;
+  }
+}


## `test(api): verify request identity parsing`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletRequestHeadersTest.java b/src/test/java/com/sportsbook/wallet/web/WalletRequestHeadersTest.java
new file mode 100644
index 0000000..605f844
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletRequestHeadersTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.util.List;
+import java.util.Locale;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class WalletRequestHeadersTest {
+  private static final UUID BET_ID = UUID.fromString("019b76da-a000-7000-8000-000000000431");
+
+  @Test
+  void readsExactlyOneIdempotencyKeyWithoutNormalization() {
+    MockHttpServletRequest request = request("deposit: exact identity");
+
+    assertThat(WalletRequestHeaders.requireIdempotencyKey(request))
+        .isEqualTo(IdempotencyKey.of("deposit: exact identity"));
+  }
+
+  @Test
+  void rejectsMissingDuplicateAndMalformedKeys() {
+    MockHttpServletRequest duplicate = request("first");
+    duplicate.addHeader(WalletRequestHeaders.IDEMPOTENCY_KEY, "second");
+
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> WalletRequestHeaders.requireIdempotencyKey(new MockHttpServletRequest()))
+        .withMessage("Exactly one Idempotency-Key header is required");
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> WalletRequestHeaders.requireIdempotencyKey(duplicate))
+        .withMessage("Exactly one Idempotency-Key header is required");
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> WalletRequestHeaders.requireIdempotencyKey(request(" ")))
+        .withMessage("IdempotencyKey must not be blank");
+    assertThatNullPointerException()
+        .isThrownBy(() -> WalletRequestHeaders.requireIdempotencyKey(null));
+  }
+
+  @Test
+  void acceptsOnlyCanonicalLowercaseDebitIdentities() {
+    assertThat(WalletRequestHeaders.requireCanonicalDebitKey(request(BET_ID.toString())))
+        .isEqualTo(IdempotencyKey.of(BET_ID.toString()));
+    assertThat(WalletRequestHeaders.requireCanonicalDebitId(BET_ID.toString())).isEqualTo(BET_ID);
+
+    for (String invalid :
+        List.of(
+            BET_ID.toString().toUpperCase(Locale.ROOT),
+            "1-1-1-1-1",
+            " " + BET_ID,
+            "not-a-debit-id")) {
+      assertThatIllegalArgumentException()
+          .isThrownBy(() -> WalletRequestHeaders.requireCanonicalDebitId(invalid))
+          .withMessage("Debit identity must be a canonical UUID");
+    }
+    assertThatNullPointerException()
+        .isThrownBy(() -> WalletRequestHeaders.requireCanonicalDebitId(null));
+  }
+
+  private MockHttpServletRequest request(String value) {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.addHeader(WalletRequestHeaders.IDEMPOTENCY_KEY, value);
+    return request;
+  }
+}


## `feat(api): define credit requests`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/CreditRequest.java b/src/main/java/com/sportsbook/wallet/web/dto/CreditRequest.java
new file mode 100644
index 0000000..9083855
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/CreditRequest.java
@@ -0,0 +1,14 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.service.command.CreditReason;
+import jakarta.validation.constraints.NotNull;
+import java.util.UUID;
+
+/** Request body for a caller-authorized wallet credit. */
+public record CreditRequest(
+    @NotNull UUID userId,
+    @NotNull Money amount,
+    @NotNull CreditCommand.Source source,
+    @NotNull CreditReason reason) {}


## `feat(api): define adjustment requests`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentRequest.java b/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentRequest.java
new file mode 100644
index 0000000..e6dee55
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentRequest.java
@@ -0,0 +1,23 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import jakarta.validation.constraints.Min;
+import jakarta.validation.constraints.NotNull;
+import java.util.UUID;
+
+/** Settlement payout revision body whose key is supplied by the authenticated HTTP request. */
+public record AdjustmentRequest(
+    @NotNull UUID revisionId,
+    @NotNull UUID betId,
+    @Min(1) long revisionNumber,
+    @NotNull UUID userId,
+    @NotNull Money previousPayout,
+    @NotNull Money newPayout) {
+
+  public AdjustmentCommand toCommand(IdempotencyKey idempotencyKey) {
+    return new AdjustmentCommand(
+        revisionId, betId, revisionNumber, userId, previousPayout, newPayout, idempotencyKey);
+  }
+}


## `feat(api): define adjustment proofs`

diff --git a/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentProofResponse.java b/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentProofResponse.java
new file mode 100644
index 0000000..80aa952
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/dto/AdjustmentProofResponse.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.web.dto;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.AdjustmentStatus;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Public settlement proof without internal retry or persistence metadata. */
+public record AdjustmentProofResponse(
+    UUID revisionId,
+    UUID betId,
+    long revisionNumber,
+    UUID userId,
+    Money previousPayout,
+    Money newPayout,
+    long deltaAmount,
+    Currency currency,
+    AdjustmentStatus status,
+    Long queueSequence,
+    UUID operationGroupId,
+    Instant queuedAt,
+    Instant appliedAt,
+    Instant nextAttemptAt) {
+
+  public static AdjustmentProofResponse from(WalletAdjustment proof) {
+    Objects.requireNonNull(proof, "proof");
+    return new AdjustmentProofResponse(
+        proof.revisionId(),
+        proof.betId(),
+        proof.revisionNumber(),
+        proof.userId(),
+        proof.previousPayout(),
+        proof.newPayout(),
+        proof.deltaAmount(),
+        proof.currency(),
+        proof.status(),
+        proof.queueSequence(),
+        proof.operationGroupId(),
+        proof.queuedAt(),
+        proof.appliedAt(),
+        proof.nextAttemptAt());
+  }
+}


## `feat(accounts): expose account HTTP endpoints`

diff --git a/src/main/java/com/sportsbook/wallet/web/AccountController.java b/src/main/java/com/sportsbook/wallet/web/AccountController.java
new file mode 100644
index 0000000..30e0b7c
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/AccountController.java
@@ -0,0 +1,38 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.OpenAccountCommand;
+import com.sportsbook.wallet.web.dto.AccountResponse;
+import com.sportsbook.wallet.web.dto.BalanceResponse;
+import com.sportsbook.wallet.web.dto.OpenAccountRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes account creation and read-only balance snapshots. */
+@RestController
+@RequestMapping("/internal/v1/wallet/accounts")
+public class AccountController {
+  private final WalletService wallet;
+
+  public AccountController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping
+  AccountResponse openAccount(@Valid @RequestBody OpenAccountRequest request) {
+    return AccountResponse.from(
+        wallet.openAccount(new OpenAccountCommand(request.userId(), request.currency())));
+  }
+
+  @GetMapping("/{userId}/balance")
+  BalanceResponse balance(@PathVariable UUID userId) {
+    return BalanceResponse.from(wallet.requireAccount(userId));
+  }
+}


## `feat(platform): expose deposit and withdrawal endpoints`

diff --git a/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java b/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java
new file mode 100644
index 0000000..3320843
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/PlatformTransactionController.java
@@ -0,0 +1,42 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.DepositCommand;
+import com.sportsbook.wallet.service.command.WithdrawCommand;
+import com.sportsbook.wallet.web.dto.TransactionRequest;
+import com.sportsbook.wallet.web.dto.WalletOperationResponse;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes platform-owned external payment transfers. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions")
+public class PlatformTransactionController {
+  private final WalletService wallet;
+
+  public PlatformTransactionController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping("/deposit")
+  WalletOperationResponse deposit(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    return WalletOperationResponse.from(
+        wallet.deposit(new DepositCommand(body.userId(), body.amount(), key)));
+  }
+
+  @PostMapping("/withdraw")
+  WalletOperationResponse withdraw(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    return WalletOperationResponse.from(
+        wallet.withdraw(new WithdrawCommand(body.userId(), body.amount(), key)));
+  }
+}


## `feat(betting): expose debit endpoints`

diff --git a/src/main/java/com/sportsbook/wallet/web/BettingTransactionController.java b/src/main/java/com/sportsbook/wallet/web/BettingTransactionController.java
new file mode 100644
index 0000000..841e5f4
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/BettingTransactionController.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.domain.error.WalletOperationNotFoundException;
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.DebitCommand;
+import com.sportsbook.wallet.web.dto.TransactionRequest;
+import com.sportsbook.wallet.web.dto.WalletOperationResponse;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes betting-owned stake reservation and durable debit lookup. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions/debit")
+public class BettingTransactionController {
+  private final WalletService wallet;
+
+  public BettingTransactionController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping
+  WalletOperationResponse debit(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireCanonicalDebitKey(request);
+    return WalletOperationResponse.from(
+        wallet.debit(new DebitCommand(body.userId(), body.amount(), key)));
+  }
+
+  @GetMapping("/{betId}")
+  WalletOperationResponse debitOutcome(@PathVariable String betId) {
+    UUID canonicalBetId = WalletRequestHeaders.requireCanonicalDebitId(betId);
+    return wallet
+        .findDebit(canonicalBetId)
+        .map(WalletOperationResponse::from)
+        .orElseThrow(() -> new WalletOperationNotFoundException(canonicalBetId));
+  }
+}


