# 이중기입 원장 토폴로지

## `feat(ledger): define ledger reasons`

diff --git a/src/main/java/com/sportsbook/wallet/domain/LedgerReason.java b/src/main/java/com/sportsbook/wallet/domain/LedgerReason.java
new file mode 100644
index 0000000..285d3df
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/LedgerReason.java
@@ -0,0 +1,12 @@
+package com.sportsbook.wallet.domain;
+
+/** Business reason recorded on both rows of a ledger pair. */
+public enum LedgerReason {
+  DEPOSIT,
+  WITHDRAW,
+  BET_DEBIT,
+  BET_PAYOUT,
+  BET_REFUND,
+  BET_FORFEIT,
+  BET_ADJUSTMENT
+}


## `feat(id): generate time-ordered UUID v7 identifiers`

diff --git a/src/main/java/com/sportsbook/wallet/infrastructure/id/UuidV7.java b/src/main/java/com/sportsbook/wallet/infrastructure/id/UuidV7.java
new file mode 100644
index 0000000..11ae0dc
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/infrastructure/id/UuidV7.java
@@ -0,0 +1,32 @@
+package com.sportsbook.wallet.infrastructure.id;
+
+import java.security.SecureRandom;
+import java.util.UUID;
+
+/** Generates RFC 9562 UUID version 7 identifiers with a millisecond-ordered prefix. */
+public final class UuidV7 {
+
+  private static final SecureRandom RANDOM = new SecureRandom();
+  private static final long TIMESTAMP_MASK = 0xFFFF_FFFF_FFFFL;
+  private static final int TIMESTAMP_SHIFT = 16;
+  private static final long VERSION_BITS = 0x7000L;
+  private static final int RANDOM_MSB_BOUND = 0x1000;
+  private static final long VARIANT_CLEAR_MASK = 0x3FFF_FFFF_FFFF_FFFFL;
+  private static final long VARIANT_BITS = 0x8000_0000_0000_0000L;
+
+  public static UUID generate() {
+    return generate(System.currentTimeMillis());
+  }
+
+  static UUID generate(long unixMillis) {
+    long mostSignificantBits = (unixMillis & TIMESTAMP_MASK) << TIMESTAMP_SHIFT;
+    mostSignificantBits |= VERSION_BITS;
+    mostSignificantBits |= RANDOM.nextInt(RANDOM_MSB_BOUND);
+
+    long leastSignificantBits = RANDOM.nextLong() & VARIANT_CLEAR_MASK;
+    leastSignificantBits |= VARIANT_BITS;
+    return new UUID(mostSignificantBits, leastSignificantBits);
+  }
+
+  private UuidV7() {}
+}


## `feat(domain): map immutable ledger values`

diff --git a/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
new file mode 100644
index 0000000..f07eb75
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
@@ -0,0 +1,67 @@
+package com.sportsbook.wallet.domain;
+
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.AttributeOverride;
+import jakarta.persistence.AttributeOverrides;
+import jakarta.persistence.Column;
+import jakarta.persistence.Embedded;
+import jakarta.persistence.Entity;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.util.UUID;
+
+/** Immutable journal row. A wallet transfer always persists two matched rows. */
+@Entity
+@Table(name = "ledger_entry")
+public class LedgerEntry {
+
+  @Id
+  @Column(name = "entry_id", nullable = false, updatable = false)
+  private UUID entryId;
+
+  @Column(name = "account_id", nullable = false, updatable = false)
+  private UUID accountId;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "bucket", nullable = false, length = 16, updatable = false)
+  private BalanceBucket bucket;
+
+  @Enumerated(EnumType.STRING)
+  @Column(name = "side", nullable = false, length = 6, updatable = false)
+  private LedgerSide side;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(
+        name = "amount",
+        column = @Column(name = "amount", nullable = false, updatable = false)),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "currency", nullable = false, length = 3, updatable = false))
+  })
+  private EmbeddedMoney money;
+
+  protected LedgerEntry() {}
+
+  public UUID entryId() {
+    return entryId;
+  }
+
+  public UUID accountId() {
+    return accountId;
+  }
+
+  public BalanceBucket bucket() {
+    return bucket;
+  }
+
+  public LedgerSide side() {
+    return side;
+  }
+
+  public Money money() {
+    return money.toMoney();
+  }
+}


## `feat(domain): attach request and operation identity to ledger rows`

diff --git a/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
index f07eb75..b24e6be 100644
--- a/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
+++ b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
@@ -9,12 +9,28 @@ import jakarta.persistence.Entity;
 import jakarta.persistence.EnumType;
 import jakarta.persistence.Enumerated;
 import jakarta.persistence.Id;
+import jakarta.persistence.Index;
 import jakarta.persistence.Table;
+import jakarta.persistence.UniqueConstraint;
+import java.time.Instant;
 import java.util.UUID;
 
 /** Immutable journal row. A wallet transfer always persists two matched rows. */
 @Entity
-@Table(name = "ledger_entry")
+@Table(
+    name = "ledger_entry",
+    uniqueConstraints = {
+      @UniqueConstraint(
+          name = "uk_ledger_entry_idempotency_side",
+          columnNames = {"idempotency_key", "side"}),
+      @UniqueConstraint(
+          name = "uk_ledger_entry_group_side",
+          columnNames = {"operation_group_id", "side"})
+    },
+    indexes = {
+      @Index(name = "ix_ledger_entry_account_created", columnList = "account_id, created_at"),
+      @Index(name = "ix_ledger_entry_idempotency_key", columnList = "idempotency_key")
+    })
 public class LedgerEntry {
 
   @Id
@@ -43,6 +59,19 @@ public class LedgerEntry {
   })
   private EmbeddedMoney money;
 
+  @Enumerated(EnumType.STRING)
+  @Column(name = "reason", nullable = false, length = 24, updatable = false)
+  private LedgerReason reason;
+
+  @Column(name = "idempotency_key", nullable = false, length = 128, updatable = false)
+  private String idempotencyKey;
+
+  @Column(name = "operation_group_id", nullable = false, updatable = false)
+  private UUID operationGroupId;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
   protected LedgerEntry() {}
 
   public UUID entryId() {
@@ -64,4 +93,20 @@ public class LedgerEntry {
   public Money money() {
     return money.toMoney();
   }
+
+  public LedgerReason reason() {
+    return reason;
+  }
+
+  public String idempotencyKey() {
+    return idempotencyKey;
+  }
+
+  public UUID operationGroupId() {
+    return operationGroupId;
+  }
+
+  public Instant createdAt() {
+    return createdAt;
+  }
 }


## `feat(domain): construct matched debit-credit pairs`

diff --git a/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
index b24e6be..f4eac0b 100644
--- a/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
+++ b/src/main/java/com/sportsbook/wallet/domain/LedgerEntry.java
@@ -1,6 +1,8 @@
 package com.sportsbook.wallet.domain;
 
+import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.infrastructure.id.UuidV7;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
 import jakarta.persistence.Column;
@@ -13,6 +15,7 @@ import jakarta.persistence.Index;
 import jakarta.persistence.Table;
 import jakarta.persistence.UniqueConstraint;
 import java.time.Instant;
+import java.util.Objects;
 import java.util.UUID;
 
 /** Immutable journal row. A wallet transfer always persists two matched rows. */
@@ -74,6 +77,66 @@ public class LedgerEntry {
 
   protected LedgerEntry() {}
 
+  private LedgerEntry(
+      UUID entryId,
+      TransferLeg leg,
+      LedgerSide side,
+      Money money,
+      LedgerReason reason,
+      IdempotencyKey idempotencyKey,
+      UUID operationGroupId,
+      Instant createdAt) {
+    Objects.requireNonNull(money, "money");
+    if (!money.isPositive()) {
+      throw new IllegalArgumentException("Ledger amount must be strictly positive");
+    }
+    this.entryId = Objects.requireNonNull(entryId, "entryId");
+    this.accountId = Objects.requireNonNull(leg, "leg").accountId();
+    this.bucket = leg.bucket();
+    this.side = Objects.requireNonNull(side, "side");
+    this.money = EmbeddedMoney.of(money);
+    this.reason = Objects.requireNonNull(reason, "reason");
+    this.idempotencyKey = Objects.requireNonNull(idempotencyKey, "idempotencyKey").value();
+    this.operationGroupId = Objects.requireNonNull(operationGroupId, "operationGroupId");
+    this.createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  public static Pair pair(
+      TransferLeg destination,
+      TransferLeg source,
+      Money amount,
+      LedgerReason reason,
+      IdempotencyKey idempotencyKey,
+      UUID operationGroupId,
+      Instant now) {
+    Objects.requireNonNull(destination, "destination");
+    Objects.requireNonNull(source, "source");
+    if (destination.equals(source)) {
+      throw new IllegalArgumentException("Ledger transfer legs must differ");
+    }
+    LedgerEntry debit =
+        new LedgerEntry(
+            UuidV7.generate(),
+            destination,
+            LedgerSide.DEBIT,
+            amount,
+            reason,
+            idempotencyKey,
+            operationGroupId,
+            now);
+    LedgerEntry credit =
+        new LedgerEntry(
+            UuidV7.generate(),
+            source,
+            LedgerSide.CREDIT,
+            amount,
+            reason,
+            idempotencyKey,
+            operationGroupId,
+            now);
+    return new Pair(debit, credit);
+  }
+
   public UUID entryId() {
     return entryId;
   }
@@ -109,4 +172,18 @@ public class LedgerEntry {
   public Instant createdAt() {
     return createdAt;
   }
+
+  public record TransferLeg(UUID accountId, BalanceBucket bucket) {
+    public TransferLeg {
+      Objects.requireNonNull(accountId, "accountId");
+      Objects.requireNonNull(bucket, "bucket");
+    }
+  }
+
+  public record Pair(LedgerEntry debit, LedgerEntry credit) {
+    public Pair {
+      Objects.requireNonNull(debit, "debit");
+      Objects.requireNonNull(credit, "credit");
+    }
+  }
 }


## `test(domain): verify ledger pair topology and guards`

diff --git a/src/test/java/com/sportsbook/wallet/domain/LedgerEntryTest.java b/src/test/java/com/sportsbook/wallet/domain/LedgerEntryTest.java
new file mode 100644
index 0000000..f1be00c
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/LedgerEntryTest.java
@@ -0,0 +1,75 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class LedgerEntryTest {
+
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-000000000003");
+  private static final IdempotencyKey KEY = IdempotencyKey.of("deposit:test");
+
+  @Test
+  void constructsACompleteMatchedPair() {
+    UUID groupId = UUID.randomUUID();
+    LedgerEntry.TransferLeg destination =
+        new LedgerEntry.TransferLeg(USER_ID, BalanceBucket.AVAILABLE);
+    LedgerEntry.TransferLeg source =
+        new LedgerEntry.TransferLeg(SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE);
+
+    LedgerEntry.Pair pair =
+        LedgerEntry.pair(
+            destination, source, Money.krw(500L), LedgerReason.DEPOSIT, KEY, groupId, NOW);
+
+    assertThat(pair.debit().accountId()).isEqualTo(USER_ID);
+    assertThat(pair.debit().side()).isEqualTo(LedgerSide.DEBIT);
+    assertThat(pair.credit().accountId()).isEqualTo(SystemAccountIds.EXTERNAL_PAYMENT);
+    assertThat(pair.credit().side()).isEqualTo(LedgerSide.CREDIT);
+    assertThat(pair.debit().money()).isEqualTo(Money.krw(500L));
+    assertThat(pair.credit().money()).isEqualTo(pair.debit().money());
+    assertThat(pair.debit().reason()).isEqualTo(LedgerReason.DEPOSIT);
+    assertThat(pair.debit().idempotencyKey()).isEqualTo(KEY.value());
+    assertThat(pair.debit().operationGroupId()).isEqualTo(groupId);
+    assertThat(pair.credit().operationGroupId()).isEqualTo(groupId);
+    assertThat(pair.debit().createdAt()).isEqualTo(NOW);
+    assertThat(pair.debit().entryId()).isNotEqualTo(pair.credit().entryId());
+  }
+
+  @Test
+  void rejectsTheSameTransferLegOnBothSides() {
+    LedgerEntry.TransferLeg leg = new LedgerEntry.TransferLeg(USER_ID, BalanceBucket.AVAILABLE);
+
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () ->
+                LedgerEntry.pair(
+                    leg, leg, Money.krw(1L), LedgerReason.DEPOSIT, KEY, UUID.randomUUID(), NOW));
+  }
+
+  @Test
+  void rejectsNonPositiveLedgerMoney() {
+    LedgerEntry.TransferLeg destination =
+        new LedgerEntry.TransferLeg(USER_ID, BalanceBucket.AVAILABLE);
+    LedgerEntry.TransferLeg source =
+        new LedgerEntry.TransferLeg(SystemAccountIds.EXTERNAL_PAYMENT, BalanceBucket.AVAILABLE);
+
+    assertThatIllegalArgumentException()
+        .isThrownBy(
+            () ->
+                LedgerEntry.pair(
+                    destination,
+                    source,
+                    Money.zero(Currency.KRW),
+                    LedgerReason.DEPOSIT,
+                    KEY,
+                    UUID.randomUUID(),
+                    NOW));
+  }
+}


## `build(flyway): create the final account and ledger schema`

diff --git a/src/main/resources/db/migration/V1__account_and_ledger.sql b/src/main/resources/db/migration/V1__account_and_ledger.sql
new file mode 100644
index 0000000..216a928
--- /dev/null
+++ b/src/main/resources/db/migration/V1__account_and_ledger.sql
@@ -0,0 +1,64 @@
+CREATE TABLE account (
+    user_id UUID PRIMARY KEY,
+    available_amount BIGINT NOT NULL DEFAULT 0,
+    available_currency VARCHAR(3) NOT NULL,
+    locked_amount BIGINT NOT NULL DEFAULT 0,
+    locked_currency VARCHAR(3) NOT NULL,
+    recovery_debt_amount NUMERIC(38, 0) NOT NULL DEFAULT 0,
+    recovery_frozen_at TIMESTAMPTZ,
+    next_adjustment_sequence BIGINT NOT NULL DEFAULT 1,
+    version BIGINT NOT NULL DEFAULT 0,
+    created_at TIMESTAMPTZ NOT NULL,
+    updated_at TIMESTAMPTZ NOT NULL,
+    CONSTRAINT ck_account_user_uuid CHECK (
+        user_id NOT IN (
+            '00000000-0000-7000-8000-000000000001',
+            '00000000-0000-7000-8000-000000000002'
+        )
+    ),
+    CONSTRAINT ck_account_available_nonnegative CHECK (available_amount >= 0),
+    CONSTRAINT ck_account_locked_nonnegative CHECK (locked_amount >= 0),
+    CONSTRAINT ck_account_currency CHECK (
+        available_currency = locked_currency
+        AND available_currency IN ('KRW', 'USD')
+    ),
+    CONSTRAINT ck_account_aggregate_limit CHECK (
+        available_amount <= 9223372036854775807 - locked_amount
+    ),
+    CONSTRAINT ck_account_recovery_debt CHECK (recovery_debt_amount >= 0),
+    CONSTRAINT ck_account_recovery_freeze CHECK (
+        (recovery_debt_amount = 0 AND recovery_frozen_at IS NULL)
+        OR (recovery_debt_amount > 0 AND recovery_frozen_at IS NOT NULL)
+    ),
+    CONSTRAINT ck_account_adjustment_sequence CHECK (next_adjustment_sequence > 0)
+);
+
+CREATE TABLE ledger_entry (
+    entry_id UUID PRIMARY KEY,
+    account_id UUID NOT NULL,
+    bucket VARCHAR(16) NOT NULL,
+    side VARCHAR(6) NOT NULL,
+    amount BIGINT NOT NULL,
+    currency VARCHAR(3) NOT NULL,
+    reason VARCHAR(24) NOT NULL,
+    idempotency_key VARCHAR(128) NOT NULL,
+    operation_group_id UUID NOT NULL,
+    created_at TIMESTAMPTZ NOT NULL,
+    CONSTRAINT ck_ledger_bucket CHECK (bucket IN ('AVAILABLE', 'LOCKED')),
+    CONSTRAINT ck_ledger_side CHECK (side IN ('DEBIT', 'CREDIT')),
+    CONSTRAINT ck_ledger_amount CHECK (amount > 0),
+    CONSTRAINT ck_ledger_currency CHECK (currency IN ('KRW', 'USD')),
+    CONSTRAINT ck_ledger_reason CHECK (
+        reason IN (
+            'DEPOSIT', 'WITHDRAW', 'BET_DEBIT', 'BET_PAYOUT',
+            'BET_REFUND', 'BET_FORFEIT', 'BET_ADJUSTMENT'
+        )
+    ),
+    CONSTRAINT uk_ledger_entry_idempotency_side UNIQUE (idempotency_key, side),
+    CONSTRAINT uk_ledger_entry_group_side UNIQUE (operation_group_id, side)
+);
+
+CREATE INDEX ix_ledger_entry_account_created
+    ON ledger_entry (account_id, created_at);
+CREATE INDEX ix_ledger_entry_idempotency_key
+    ON ledger_entry (idempotency_key);


## `feat(repository): query durable ledger pairs`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/LedgerEntryRepository.java b/src/main/java/com/sportsbook/wallet/persistence/LedgerEntryRepository.java
new file mode 100644
index 0000000..bbf1b15
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/LedgerEntryRepository.java
@@ -0,0 +1,16 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.domain.LedgerEntry;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+
+/** Append-only ledger queries for durable operation replay and account history. */
+public interface LedgerEntryRepository extends JpaRepository<LedgerEntry, UUID> {
+
+  List<LedgerEntry> findByIdempotencyKey(String idempotencyKey);
+
+  List<LedgerEntry> findByOperationGroupId(UUID operationGroupId);
+
+  List<LedgerEntry> findByAccountIdOrderByCreatedAtAsc(UUID accountId);
+}


## `feat(service): derive results from complete ledger pairs`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
new file mode 100644
index 0000000..67113b7
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletOperationResult.java
@@ -0,0 +1,61 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.domain.LedgerSide;
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import java.time.Instant;
+import java.util.EnumSet;
+import java.util.List;
+import java.util.Set;
+import java.util.UUID;
+import java.util.stream.Collectors;
+
+/** Immutable success response rebuilt from the authoritative matched ledger pair. */
+public record WalletOperationResult(
+    UUID operationGroupId, UUID userId, Money amount, LedgerReason reason, Instant at) {
+
+  public static WalletOperationResult fromExisting(List<LedgerEntry> pair) {
+    if (pair.size() != 2) {
+      throw new IllegalStateException(
+          "Ledger result must contain exactly two entries (got " + pair.size() + ")");
+    }
+
+    LedgerEntry first = pair.get(0);
+    Set<LedgerSide> sides = pair.stream().map(LedgerEntry::side).collect(Collectors.toSet());
+    if (!sides.equals(EnumSet.allOf(LedgerSide.class))) {
+      throw new IllegalStateException("Ledger result must contain one DEBIT and one CREDIT entry");
+    }
+
+    boolean sameTransfer =
+        pair.stream()
+            .allMatch(
+                entry ->
+                    entry.operationGroupId().equals(first.operationGroupId())
+                        && entry.idempotencyKey().equals(first.idempotencyKey())
+                        && entry.money().equals(first.money())
+                        && entry.reason() == first.reason()
+                        && entry.createdAt().equals(first.createdAt()));
+    if (!sameTransfer) {
+      throw new IllegalStateException("Ledger result entries do not describe one matched transfer");
+    }
+
+    Set<UUID> userIds =
+        pair.stream()
+            .map(LedgerEntry::accountId)
+            .filter(accountId -> !SystemAccountIds.isSystemAccount(accountId))
+            .collect(Collectors.toSet());
+    if (userIds.size() != 1) {
+      throw new IllegalStateException(
+          "Ledger result must identify exactly one user account (got " + userIds.size() + ")");
+    }
+
+    return new WalletOperationResult(
+        first.operationGroupId(),
+        userIds.iterator().next(),
+        first.money(),
+        first.reason(),
+        first.createdAt());
+  }
+}


