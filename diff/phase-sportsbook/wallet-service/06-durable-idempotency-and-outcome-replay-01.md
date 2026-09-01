# 영속 멱등성과 결과 재생

## `feat(operation): define operation kinds and terminal states`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletOperationKind.java b/src/main/java/com/sportsbook/wallet/domain/WalletOperationKind.java
new file mode 100644
index 0000000..2b0425b
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletOperationKind.java
@@ -0,0 +1,16 @@
+package com.sportsbook.wallet.domain;
+
+/** Stable request identity for every money-moving wallet operation. */
+public enum WalletOperationKind {
+  DEPOSIT,
+  WITHDRAW,
+  BET_DEBIT,
+  BET_PAYOUT,
+  BET_REFUND,
+  BET_FORFEIT,
+  BET_ADJUSTMENT;
+
+  public LedgerReason ledgerReason() {
+    return LedgerReason.valueOf(name());
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletOperationStatus.java b/src/main/java/com/sportsbook/wallet/domain/WalletOperationStatus.java
new file mode 100644
index 0000000..22faf11
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletOperationStatus.java
@@ -0,0 +1,8 @@
+package com.sportsbook.wallet.domain;
+
+/** Durable outcome state. Only a blocked adjustment may transition after its first commit. */
+public enum WalletOperationStatus {
+  SUCCEEDED,
+  REJECTED,
+  BLOCKED_FUNDS
+}


## `feat(operation): encode canonical request fields`

diff --git a/src/main/java/com/sportsbook/wallet/service/CanonicalRequestEncoder.java b/src/main/java/com/sportsbook/wallet/service/CanonicalRequestEncoder.java
new file mode 100644
index 0000000..d667812
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/CanonicalRequestEncoder.java
@@ -0,0 +1,63 @@
+package com.sportsbook.wallet.service;
+
+import java.io.ByteArrayOutputStream;
+import java.io.DataOutputStream;
+import java.io.IOException;
+import java.nio.charset.StandardCharsets;
+import java.util.UUID;
+
+/** Versioned TLV writer used only for wallet request fingerprints. */
+final class CanonicalRequestEncoder {
+  private static final byte[] MAGIC =
+      "sportsbook.wallet.operation".getBytes(StandardCharsets.US_ASCII);
+  private static final int VERSION = 1;
+
+  private final ByteArrayOutputStream bytes = new ByteArrayOutputStream();
+  private final DataOutputStream output = new DataOutputStream(bytes);
+
+  CanonicalRequestEncoder() {
+    try {
+      output.write(MAGIC);
+      output.writeByte(0);
+      output.writeByte(VERSION);
+    } catch (IOException impossible) {
+      throw new IllegalStateException(impossible);
+    }
+  }
+
+  CanonicalRequestEncoder text(int tag, String value) {
+    return field(tag, value.getBytes(StandardCharsets.UTF_8));
+  }
+
+  CanonicalRequestEncoder uuid(int tag, UUID value) {
+    try {
+      byte[] encoded =
+          java.nio.ByteBuffer.allocate(Long.BYTES * 2)
+              .putLong(value.getMostSignificantBits())
+              .putLong(value.getLeastSignificantBits())
+              .array();
+      return field(tag, encoded);
+    } catch (RuntimeException failure) {
+      throw new IllegalArgumentException("Invalid UUID fingerprint field", failure);
+    }
+  }
+
+  CanonicalRequestEncoder number(int tag, long value) {
+    return field(tag, java.nio.ByteBuffer.allocate(Long.BYTES).putLong(value).array());
+  }
+
+  byte[] toByteArray() {
+    return bytes.toByteArray();
+  }
+
+  private CanonicalRequestEncoder field(int tag, byte[] value) {
+    try {
+      output.writeByte(tag);
+      output.writeInt(value.length);
+      output.write(value);
+      return this;
+    } catch (IOException impossible) {
+      throw new IllegalStateException(impossible);
+    }
+  }
+}


## `feat(operation): fingerprint canonical wallet requests`

diff --git a/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java b/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java
new file mode 100644
index 0000000..619e36f
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java
@@ -0,0 +1,87 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Objects;
+import java.util.UUID;
+
+/** SHA-256 of a versioned binary representation of semantic request identity. */
+public record OperationFingerprint(String value) {
+  private static final int HASH_LENGTH = 64;
+  private static final int USER_TAG = 3;
+  private static final int AMOUNT_TAG = 4;
+  private static final int CURRENCY_TAG = 5;
+  private static final int REVISION_TAG = 6;
+  private static final int BET_TAG = 7;
+  private static final int REVISION_NUMBER_TAG = 8;
+  private static final int PREVIOUS_PAYOUT_TAG = 9;
+  private static final int NEW_PAYOUT_TAG = 10;
+
+  public OperationFingerprint {
+    Objects.requireNonNull(value, "value");
+    if (value.length() != HASH_LENGTH || !value.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("Operation fingerprint must be lower-case SHA-256 hex");
+    }
+  }
+
+  public static OperationFingerprint transfer(
+      WalletCaller caller, WalletOperationKind kind, UUID userId, Money amount) {
+    return digest(base(caller, kind, userId, amount).toByteArray());
+  }
+
+  public static OperationFingerprint adjustment(
+      WalletCaller caller,
+      UUID userId,
+      Money previousPayout,
+      Money newPayout,
+      UUID revisionId,
+      UUID betId,
+      long revisionNumber) {
+    Objects.requireNonNull(previousPayout, "previousPayout");
+    Objects.requireNonNull(newPayout, "newPayout");
+    if (previousPayout.currency() != newPayout.currency()) {
+      throw new IllegalArgumentException("Adjustment payout currencies must match");
+    }
+    long delta = Math.subtractExact(newPayout.amount(), previousPayout.amount());
+    if (delta == Long.MIN_VALUE) {
+      throw new ArithmeticException("Adjustment delta is not representable");
+    }
+    CanonicalRequestEncoder encoded =
+        base(
+            caller,
+            WalletOperationKind.BET_ADJUSTMENT,
+            userId,
+            new Money(Math.abs(delta), previousPayout.currency()));
+    encoded
+        .uuid(REVISION_TAG, Objects.requireNonNull(revisionId, "revisionId"))
+        .uuid(BET_TAG, Objects.requireNonNull(betId, "betId"))
+        .number(REVISION_NUMBER_TAG, revisionNumber)
+        .number(PREVIOUS_PAYOUT_TAG, previousPayout.amount())
+        .number(NEW_PAYOUT_TAG, newPayout.amount());
+    return digest(encoded.toByteArray());
+  }
+
+  private static CanonicalRequestEncoder base(
+      WalletCaller caller, WalletOperationKind kind, UUID userId, Money amount) {
+    Objects.requireNonNull(amount, "amount");
+    return new CanonicalRequestEncoder()
+        .text(1, Objects.requireNonNull(caller, "caller").name())
+        .text(2, Objects.requireNonNull(kind, "kind").name())
+        .uuid(USER_TAG, Objects.requireNonNull(userId, "userId"))
+        .number(AMOUNT_TAG, amount.amount())
+        .text(CURRENCY_TAG, amount.currency().name());
+  }
+
+  private static OperationFingerprint digest(byte[] canonical) {
+    try {
+      byte[] hash = MessageDigest.getInstance("SHA-256").digest(canonical);
+      return new OperationFingerprint(HexFormat.of().formatHex(hash));
+    } catch (NoSuchAlgorithmException impossible) {
+      throw new IllegalStateException("JVM lacks SHA-256", impossible);
+    }
+  }
+}


## `test(operation): lock request fingerprint vectors`

diff --git a/src/test/java/com/sportsbook/wallet/service/OperationFingerprintTest.java b/src/test/java/com/sportsbook/wallet/service/OperationFingerprintTest.java
new file mode 100644
index 0000000..de55fa8
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/service/OperationFingerprintTest.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class OperationFingerprintTest {
+  private static final UUID USER = UUID.fromString("00000000-0000-7000-8000-000000000123");
+
+  @Test
+  void locksTheTransferVector() {
+    OperationFingerprint result =
+        OperationFingerprint.transfer(
+            WalletCaller.PLATFORM, WalletOperationKind.DEPOSIT, USER, Money.krw(1_000L));
+
+    assertThat(result.value())
+        .isEqualTo("d544e877620dfa1a88ed3affc6b00ae586064c9ec6633f9cf27c17b49fda49c5");
+  }
+
+  @Test
+  void locksTheAdjustmentVector() {
+    OperationFingerprint result =
+        OperationFingerprint.adjustment(
+            WalletCaller.SETTLEMENT,
+            USER,
+            Money.krw(1_000L),
+            Money.krw(700L),
+            UUID.fromString("00000000-0000-7000-8000-000000000456"),
+            UUID.fromString("00000000-0000-7000-8000-000000000789"),
+            2L);
+
+    assertThat(result.value())
+        .isEqualTo("5c95e5e161c733bca2c4f80de2118c1d6fbf037ad14ecb25948e55cbabd4e40d");
+  }
+
+  @Test
+  void rejectsNonCanonicalDigestText() {
+    assertThatIllegalArgumentException().isThrownBy(() -> new OperationFingerprint("a".repeat(63)));
+    assertThatIllegalArgumentException().isThrownBy(() -> new OperationFingerprint("A".repeat(64)));
+  }
+}


## `feat(errors): define durable wallet failures`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletFailureCode.java b/src/main/java/com/sportsbook/wallet/domain/WalletFailureCode.java
new file mode 100644
index 0000000..85c7218
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletFailureCode.java
@@ -0,0 +1,32 @@
+package com.sportsbook.wallet.domain;
+
+/** Stable business failures that may be committed as an exact replay outcome. */
+public enum WalletFailureCode {
+  ACCOUNT_NOT_FOUND(404, "WALLET_ACCOUNT_NOT_FOUND", "Account not found"),
+  CURRENCY_MISMATCH(422, "WALLET_CURRENCY_MISMATCH", "Currency mismatch"),
+  INSUFFICIENT_BALANCE(422, "WALLET_INSUFFICIENT_BALANCE", "Insufficient balance"),
+  ACCOUNT_SUSPENDED(423, "WALLET_ACCOUNT_SUSPENDED", "Wallet account suspended"),
+  AMOUNT_OUT_OF_RANGE(422, "WALLET_AMOUNT_OUT_OF_RANGE", "Amount out of range");
+
+  private final int httpStatus;
+  private final String wireCode;
+  private final String title;
+
+  WalletFailureCode(int httpStatus, String wireCode, String title) {
+    this.httpStatus = httpStatus;
+    this.wireCode = wireCode;
+    this.title = title;
+  }
+
+  public int httpStatus() {
+    return httpStatus;
+  }
+
+  public String wireCode() {
+    return wireCode;
+  }
+
+  public String title() {
+    return title;
+  }
+}


## `feat(operation): construct durable operation outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java b/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
index fdc1ae4..debd01d 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
@@ -1,5 +1,7 @@
 package com.sportsbook.wallet.domain;
 
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
 import jakarta.persistence.Column;
@@ -11,6 +13,7 @@ import jakarta.persistence.Id;
 import jakarta.persistence.Table;
 import jakarta.persistence.Version;
 import java.time.Instant;
+import java.util.Objects;
 import java.util.UUID;
 
 /** Authoritative immutable request identity and durable outcome for one idempotency key. */
@@ -70,4 +73,99 @@ public class WalletOperation {
   private long version;
 
   protected WalletOperation() {}
+
+  private WalletOperation(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      Instant now) {
+    if (!amount.isPositive()) {
+      throw new IllegalArgumentException("Operation amount must be strictly positive");
+    }
+    Objects.requireNonNull(caller, "caller");
+    Objects.requireNonNull(kind, "kind");
+    this.idempotencyKey = Objects.requireNonNull(key, "key").value();
+    this.caller = caller;
+    this.kind = kind;
+    this.userId = Objects.requireNonNull(userId, "userId");
+    this.requestAmount = EmbeddedMoney.of(amount);
+    this.requestFingerprint = Objects.requireNonNull(fingerprint, "fingerprint");
+    this.requestedAt = Objects.requireNonNull(now, "now");
+    this.updatedAt = now;
+  }
+
+  public static WalletOperation succeeded(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      UUID operationGroupId,
+      Instant now) {
+    WalletOperation operation =
+        new WalletOperation(key, caller, kind, userId, amount, fingerprint, now);
+    operation.status = WalletOperationStatus.SUCCEEDED;
+    operation.operationGroupId = Objects.requireNonNull(operationGroupId, "operationGroupId");
+    operation.completedAt = now;
+    return operation;
+  }
+
+  public static WalletOperation rejected(
+      IdempotencyKey key,
+      WalletCaller caller,
+      WalletOperationKind kind,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      WalletFailureSnapshot failure,
+      Instant now) {
+    WalletOperation operation =
+        new WalletOperation(key, caller, kind, userId, amount, fingerprint, now);
+    operation.status = WalletOperationStatus.REJECTED;
+    operation.failure = Objects.requireNonNull(failure, "failure");
+    operation.completedAt = now;
+    return operation;
+  }
+
+  public static WalletOperation blockedFunds(
+      IdempotencyKey key,
+      WalletCaller caller,
+      UUID userId,
+      Money amount,
+      String fingerprint,
+      Instant now) {
+    WalletOperation operation =
+        new WalletOperation(
+            key, caller, WalletOperationKind.BET_ADJUSTMENT, userId, amount, fingerprint, now);
+    operation.status = WalletOperationStatus.BLOCKED_FUNDS;
+    return operation;
+  }
+
+  public WalletOperationStatus status() {
+    return status;
+  }
+
+  public UUID operationGroupId() {
+    return operationGroupId;
+  }
+
+  public WalletFailureSnapshot failure() {
+    return failure;
+  }
+
+  public Instant requestedAt() {
+    return requestedAt;
+  }
+
+  public Instant updatedAt() {
+    return updatedAt;
+  }
+
+  public Instant completedAt() {
+    return completedAt;
+  }
 }


## `feat(operation): complete blocked outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java b/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
index 14d6340..95e72d7 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletOperation.java
@@ -146,6 +146,18 @@ public class WalletOperation {
     return operation;
   }
 
+  public void completeBlocked(UUID groupId, Instant now) {
+    if (status != WalletOperationStatus.BLOCKED_FUNDS) {
+      throw new IllegalStateException("Only blocked operations can be completed");
+    }
+    Objects.requireNonNull(now, "now");
+    UUID completedGroupId = Objects.requireNonNull(groupId, "groupId");
+    status = WalletOperationStatus.SUCCEEDED;
+    operationGroupId = completedGroupId;
+    updatedAt = now;
+    completedAt = now;
+  }
+
   public WalletOperationStatus status() {
     return status;
   }


## `build(flyway): create authoritative wallet outcomes`

diff --git a/src/main/resources/db/migration/V2__wallet_operation.sql b/src/main/resources/db/migration/V2__wallet_operation.sql
new file mode 100644
index 0000000..5a8a800
--- /dev/null
+++ b/src/main/resources/db/migration/V2__wallet_operation.sql
@@ -0,0 +1,82 @@
+CREATE TABLE wallet_operation (
+    idempotency_key VARCHAR(128) PRIMARY KEY,
+    caller_id VARCHAR(16) NOT NULL,
+    operation_kind VARCHAR(32) NOT NULL,
+    user_id UUID NOT NULL,
+    request_amount BIGINT NOT NULL,
+    request_currency VARCHAR(3) NOT NULL,
+    request_fingerprint VARCHAR(64) NOT NULL,
+    status VARCHAR(24) NOT NULL,
+    operation_group_id UUID UNIQUE,
+    failure_code VARCHAR(32),
+    failure_http_status SMALLINT,
+    failure_title VARCHAR(128),
+    failure_detail VARCHAR(1024),
+    failure_balance_amount BIGINT,
+    failure_balance_currency VARCHAR(3),
+    failure_expected_currency VARCHAR(3),
+    requested_at TIMESTAMPTZ NOT NULL,
+    updated_at TIMESTAMPTZ NOT NULL,
+    completed_at TIMESTAMPTZ,
+    version BIGINT NOT NULL DEFAULT 0,
+    CONSTRAINT ck_wallet_operation_caller CHECK (
+        caller_id IN ('PLATFORM', 'BETTING', 'SETTLEMENT', 'ADMIN')
+    ),
+    CONSTRAINT ck_wallet_operation_kind CHECK (
+        operation_kind IN (
+            'DEPOSIT', 'WITHDRAW', 'BET_DEBIT', 'BET_PAYOUT',
+            'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        )
+    ),
+    CONSTRAINT ck_wallet_operation_caller_kind CHECK (
+        (caller_id = 'PLATFORM' AND operation_kind IN ('DEPOSIT', 'WITHDRAW'))
+        OR (caller_id = 'BETTING' AND operation_kind IN ('BET_DEBIT', 'BET_REFUND'))
+        OR (caller_id = 'SETTLEMENT' AND operation_kind IN (
+            'BET_PAYOUT', 'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        ))
+        OR (caller_id = 'ADMIN' AND operation_kind = 'BET_REFUND')
+    ),
+    CONSTRAINT ck_wallet_operation_request CHECK (
+        request_amount > 0
+        AND request_currency IN ('KRW', 'USD')
+        AND request_fingerprint ~ '^[0-9a-f]{64}$'
+    ),
+    CONSTRAINT ck_wallet_operation_status CHECK (
+        status IN ('SUCCEEDED', 'REJECTED', 'BLOCKED_FUNDS')
+    ),
+    CONSTRAINT ck_wallet_operation_outcome CHECK (
+        (
+            status = 'SUCCEEDED'
+            AND operation_group_id IS NOT NULL
+            AND completed_at IS NOT NULL
+            AND failure_code IS NULL
+            AND failure_http_status IS NULL
+            AND failure_title IS NULL
+            AND failure_detail IS NULL
+        )
+        OR (
+            status = 'REJECTED'
+            AND operation_group_id IS NULL
+            AND completed_at IS NOT NULL
+            AND failure_code IS NOT NULL
+            AND failure_http_status BETWEEN 400 AND 499
+            AND failure_title IS NOT NULL
+            AND failure_detail IS NOT NULL
+        )
+        OR (
+            status = 'BLOCKED_FUNDS'
+            AND operation_kind = 'BET_ADJUSTMENT'
+            AND operation_group_id IS NULL
+            AND completed_at IS NULL
+            AND failure_code IS NULL
+            AND failure_http_status IS NULL
+            AND failure_title IS NULL
+            AND failure_detail IS NULL
+        )
+    ),
+    CONSTRAINT uk_wallet_operation_key_group
+        UNIQUE (idempotency_key, operation_group_id)
+);
+
+CREATE INDEX ix_wallet_operation_user_requested
+    ON wallet_operation (user_id, requested_at);


## `feat(repository): persist wallet outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java b/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java
new file mode 100644
index 0000000..9d29de8
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java
@@ -0,0 +1,7 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.domain.WalletOperation;
+import org.springframework.data.jpa.repository.JpaRepository;
+
+/** Durable operation outcomes. Reads intentionally remain lock-free because rows are terminal. */
+public interface WalletOperationRepository extends JpaRepository<WalletOperation, String> {}


