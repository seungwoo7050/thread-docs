# 계정 잔액 불변식

## `feat(domain): define balance buckets and ledger sides`

diff --git a/src/main/java/com/sportsbook/wallet/domain/BalanceBucket.java b/src/main/java/com/sportsbook/wallet/domain/BalanceBucket.java
new file mode 100644
index 0000000..ccc733b
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/BalanceBucket.java
@@ -0,0 +1,9 @@
+package com.sportsbook.wallet.domain;
+
+/** Balance bucket addressed by a ledger entry. */
+public enum BalanceBucket {
+  /** Funds available for withdrawal or a new stake. */
+  AVAILABLE,
+  /** Funds held against an unsettled bet. */
+  LOCKED
+}
diff --git a/src/main/java/com/sportsbook/wallet/domain/LedgerSide.java b/src/main/java/com/sportsbook/wallet/domain/LedgerSide.java
new file mode 100644
index 0000000..efe4328
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/LedgerSide.java
@@ -0,0 +1,9 @@
+package com.sportsbook.wallet.domain;
+
+/** Asset-side direction of a double-entry journal row. */
+public enum LedgerSide {
+  /** The addressed bucket increases. */
+  DEBIT,
+  /** The addressed bucket decreases. */
+  CREDIT
+}


## `feat(identity): define system counterparties`

diff --git a/src/main/java/com/sportsbook/wallet/domain/SystemAccountIds.java b/src/main/java/com/sportsbook/wallet/domain/SystemAccountIds.java
new file mode 100644
index 0000000..93a71a2
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/SystemAccountIds.java
@@ -0,0 +1,20 @@
+package com.sportsbook.wallet.domain;
+
+import java.util.Set;
+import java.util.UUID;
+
+/** Stable ledger counterparties that are not user accounts. */
+public final class SystemAccountIds {
+
+  public static final UUID HOUSE = UUID.fromString("00000000-0000-7000-8000-000000000001");
+  public static final UUID EXTERNAL_PAYMENT =
+      UUID.fromString("00000000-0000-7000-8000-000000000002");
+
+  private static final Set<UUID> ALL = Set.of(HOUSE, EXTERNAL_PAYMENT);
+
+  public static boolean isSystemAccount(UUID accountId) {
+    return ALL.contains(accountId);
+  }
+
+  private SystemAccountIds() {}
+}


## `feat(money): persist shared money values`

diff --git a/src/main/java/com/sportsbook/wallet/domain/EmbeddedMoney.java b/src/main/java/com/sportsbook/wallet/domain/EmbeddedMoney.java
new file mode 100644
index 0000000..39bd47e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/EmbeddedMoney.java
@@ -0,0 +1,60 @@
+package com.sportsbook.wallet.domain;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.Column;
+import jakarta.persistence.Embeddable;
+import jakarta.persistence.EnumType;
+import jakarta.persistence.Enumerated;
+import java.util.Objects;
+
+/** Persistence adapter for the shared immutable {@link Money} value. */
+@Embeddable
+public class EmbeddedMoney {
+
+  @Column(nullable = false)
+  private long amount;
+
+  @Enumerated(EnumType.STRING)
+  @Column(nullable = false, length = 3)
+  private Currency currency;
+
+  protected EmbeddedMoney() {}
+
+  public EmbeddedMoney(long amount, Currency currency) {
+    this.amount = amount;
+    this.currency = Objects.requireNonNull(currency, "currency");
+  }
+
+  public static EmbeddedMoney of(Money money) {
+    Objects.requireNonNull(money, "money");
+    return new EmbeddedMoney(money.amount(), money.currency());
+  }
+
+  public Money toMoney() {
+    return new Money(amount, currency);
+  }
+
+  public long amount() {
+    return amount;
+  }
+
+  public Currency currency() {
+    return currency;
+  }
+
+  @Override
+  public boolean equals(Object other) {
+    if (this == other) {
+      return true;
+    }
+    return other instanceof EmbeddedMoney that
+        && amount == that.amount
+        && currency == that.currency;
+  }
+
+  @Override
+  public int hashCode() {
+    return Objects.hash(amount, currency);
+  }
+}


## `feat(domain): model account identity and reject system UUIDs`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
new file mode 100644
index 0000000..b2d0248
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -0,0 +1,100 @@
+package com.sportsbook.wallet.domain;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import jakarta.persistence.AttributeOverride;
+import jakarta.persistence.AttributeOverrides;
+import jakarta.persistence.Column;
+import jakarta.persistence.Embedded;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import jakarta.persistence.Version;
+import java.math.BigInteger;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+@Entity
+@Table(name = "account")
+public class Account {
+  @Id
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(
+        name = "amount",
+        column = @Column(name = "available_amount", nullable = false)),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "available_currency", nullable = false, length = 3))
+  })
+  private EmbeddedMoney available;
+
+  @Embedded
+  @AttributeOverrides({
+    @AttributeOverride(name = "amount", column = @Column(name = "locked_amount", nullable = false)),
+    @AttributeOverride(
+        name = "currency",
+        column = @Column(name = "locked_currency", nullable = false, length = 3))
+  })
+  private EmbeddedMoney locked;
+
+  @Version
+  @Column(nullable = false)
+  private long version;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "updated_at", nullable = false)
+  private Instant updatedAt;
+
+  @Column(name = "recovery_debt_amount", nullable = false, precision = 38, scale = 0)
+  private BigInteger recoveryDebtAmount;
+
+  protected Account() {}
+
+  private Account(UUID userId, Currency currency, Instant now) {
+    this.userId = Objects.requireNonNull(userId, "userId");
+    if (SystemAccountIds.isSystemAccount(userId)) {
+      throw new IllegalArgumentException("System UUID cannot own an account: " + userId);
+    }
+    Objects.requireNonNull(currency, "currency");
+    this.available = new EmbeddedMoney(0L, currency);
+    this.locked = new EmbeddedMoney(0L, currency);
+    this.recoveryDebtAmount = BigInteger.ZERO;
+    this.createdAt = Objects.requireNonNull(now, "now");
+    this.updatedAt = now;
+  }
+
+  public static Account openFor(UUID userId, Currency currency, Instant now) {
+    return new Account(userId, currency, now);
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+
+  public Money available() {
+    return available.toMoney();
+  }
+
+  public Money locked() {
+    return locked.toMoney();
+  }
+
+  public long version() {
+    return version;
+  }
+
+  public BigInteger recoveryDebtAmount() {
+    return recoveryDebtAmount;
+  }
+
+  public Instant createdAt() {
+    return createdAt;
+  }
+}


## `feat(domain): enforce representable aggregate balances`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
index b2d0248..1a48d4a 100644
--- a/src/main/java/com/sportsbook/wallet/domain/Account.java
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -2,6 +2,7 @@ package com.sportsbook.wallet.domain;
 
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
 import jakarta.persistence.Column;
@@ -55,6 +56,12 @@ public class Account {
   @Column(name = "recovery_debt_amount", nullable = false, precision = 38, scale = 0)
   private BigInteger recoveryDebtAmount;
 
+  @Column(name = "recovery_frozen_at")
+  private Instant recoveryFrozenAt;
+
+  @Column(name = "next_adjustment_sequence", nullable = false)
+  private long nextAdjustmentSequence;
+
   protected Account() {}
 
   private Account(UUID userId, Currency currency, Instant now) {
@@ -66,6 +73,7 @@ public class Account {
     this.available = new EmbeddedMoney(0L, currency);
     this.locked = new EmbeddedMoney(0L, currency);
     this.recoveryDebtAmount = BigInteger.ZERO;
+    this.nextAdjustmentSequence = 1L;
     this.createdAt = Objects.requireNonNull(now, "now");
     this.updatedAt = now;
   }
@@ -78,6 +86,10 @@ public class Account {
     return userId;
   }
 
+  public Currency currency() {
+    return available.currency();
+  }
+
   public Money available() {
     return available.toMoney();
   }
@@ -86,6 +98,11 @@ public class Account {
     return locked.toMoney();
   }
 
+  public Money total() {
+    requireRepresentableBalance(userId, available.amount(), locked.amount());
+    return new Money(available.amount() + locked.amount(), currency());
+  }
+
   public long version() {
     return version;
   }
@@ -94,7 +111,24 @@ public class Account {
     return recoveryDebtAmount;
   }
 
+  public Instant recoveryFrozenAt() {
+    return recoveryFrozenAt;
+  }
+
+  public long nextAdjustmentSequence() {
+    return nextAdjustmentSequence;
+  }
+
   public Instant createdAt() {
     return createdAt;
   }
+
+  static void requireRepresentableBalance(UUID userId, long availableAmount, long lockedAmount) {
+    if (availableAmount < 0L || lockedAmount < 0L) {
+      throw new IllegalArgumentException("Balance buckets cannot be negative");
+    }
+    if (availableAmount > Long.MAX_VALUE - lockedAmount) {
+      throw new BalanceLimitExceededException(userId, availableAmount, lockedAmount);
+    }
+  }
 }


## `feat(domain): mutate available funds safely`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
index 1a48d4a..3f697de 100644
--- a/src/main/java/com/sportsbook/wallet/domain/Account.java
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -3,6 +3,8 @@ package com.sportsbook.wallet.domain;
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.InsufficientBalanceException;
 import jakarta.persistence.AttributeOverride;
 import jakarta.persistence.AttributeOverrides;
 import jakarta.persistence.Column;
@@ -119,10 +121,37 @@ public class Account {
     return nextAdjustmentSequence;
   }
 
+  public void increaseAvailable(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    long capacity = Long.MAX_VALUE - locked.amount() - available.amount();
+    if (amount.amount() > capacity) {
+      throw new BalanceLimitExceededException(userId, available.amount(), locked.amount());
+    }
+    available = new EmbeddedMoney(available.amount() + amount.amount(), currency());
+    updatedAt = now;
+  }
+
+  public void decreaseAvailable(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    if (available.amount() < amount.amount()) {
+      throw new InsufficientBalanceException(userId, amount, available.toMoney());
+    }
+    available = new EmbeddedMoney(available.amount() - amount.amount(), currency());
+    updatedAt = now;
+  }
+
   public Instant createdAt() {
     return createdAt;
   }
 
+  public Instant updatedAt() {
+    return updatedAt;
+  }
+
   static void requireRepresentableBalance(UUID userId, long availableAmount, long lockedAmount) {
     if (availableAmount < 0L || lockedAmount < 0L) {
       throw new IllegalArgumentException("Balance buckets cannot be negative");
@@ -131,4 +160,17 @@ public class Account {
       throw new BalanceLimitExceededException(userId, availableAmount, lockedAmount);
     }
   }
+
+  private void requireSameCurrency(Money amount) {
+    if (amount.currency() != currency()) {
+      throw new CurrencyMismatchException(currency(), amount.currency());
+    }
+  }
+
+  private static void requirePositive(Money amount) {
+    Objects.requireNonNull(amount, "amount");
+    if (amount.amount() <= 0L) {
+      throw new IllegalArgumentException("Amount must be strictly positive");
+    }
+  }
 }


## `feat(domain): stage refund and forfeit locked funds`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
index 3f697de..51e4bbd 100644
--- a/src/main/java/com/sportsbook/wallet/domain/Account.java
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -144,6 +144,43 @@ public class Account {
     updatedAt = now;
   }
 
+  public void moveAvailableToLocked(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    if (available.amount() < amount.amount()) {
+      throw new InsufficientBalanceException(userId, amount, available.toMoney());
+    }
+    long nextAvailable = available.amount() - amount.amount();
+    long nextLocked = locked.amount() + amount.amount();
+    requireRepresentableBalance(userId, nextAvailable, nextLocked);
+    available = new EmbeddedMoney(nextAvailable, currency());
+    locked = new EmbeddedMoney(nextLocked, currency());
+    updatedAt = now;
+  }
+
+  public void moveLockedToAvailable(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    requireLockedFunds(amount);
+    long nextAvailable = available.amount() + amount.amount();
+    long nextLocked = locked.amount() - amount.amount();
+    requireRepresentableBalance(userId, nextAvailable, nextLocked);
+    available = new EmbeddedMoney(nextAvailable, currency());
+    locked = new EmbeddedMoney(nextLocked, currency());
+    updatedAt = now;
+  }
+
+  public void forfeitLocked(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    requireLockedFunds(amount);
+    locked = new EmbeddedMoney(locked.amount() - amount.amount(), currency());
+    updatedAt = now;
+  }
+
   public Instant createdAt() {
     return createdAt;
   }
@@ -167,6 +204,12 @@ public class Account {
     }
   }
 
+  private void requireLockedFunds(Money amount) {
+    if (locked.amount() < amount.amount()) {
+      throw new InsufficientBalanceException(userId, amount, locked.toMoney());
+    }
+  }
+
   private static void requirePositive(Money amount) {
     Objects.requireNonNull(amount, "amount");
     if (amount.amount() <= 0L) {


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


## `test(domain): verify account identity and reserved UUID guards`

diff --git a/src/test/java/com/sportsbook/wallet/domain/AccountIdentityTest.java b/src/test/java/com/sportsbook/wallet/domain/AccountIdentityTest.java
new file mode 100644
index 0000000..647bac3
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/AccountIdentityTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AccountIdentityTest {
+
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+
+  @Test
+  void opensAnEmptyUserAccount() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000001");
+
+    Account account = Account.openFor(userId, Currency.KRW, NOW);
+
+    assertThat(account.userId()).isEqualTo(userId);
+    assertThat(account.available()).isEqualTo(Money.zero(Currency.KRW));
+    assertThat(account.locked()).isEqualTo(Money.zero(Currency.KRW));
+    assertThat(account.createdAt()).isEqualTo(NOW);
+  }
+
+  @Test
+  void rejectsReservedSystemCounterparties() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> Account.openFor(SystemAccountIds.HOUSE, Currency.KRW, NOW));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> Account.openFor(SystemAccountIds.EXTERNAL_PAYMENT, Currency.KRW, NOW));
+  }
+}


## `test(domain): verify Long.MAX aggregate boundaries`

diff --git a/src/test/java/com/sportsbook/wallet/domain/AccountBalanceLimitTest.java b/src/test/java/com/sportsbook/wallet/domain/AccountBalanceLimitTest.java
new file mode 100644
index 0000000..9496357
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/AccountBalanceLimitTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AccountBalanceLimitTest {
+
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-000000000002");
+
+  @Test
+  void acceptsEverySplitAtTheAggregateLimit() {
+    assertThatCode(() -> Account.requireRepresentableBalance(USER_ID, Long.MAX_VALUE, 0L))
+        .doesNotThrowAnyException();
+    assertThatCode(() -> Account.requireRepresentableBalance(USER_ID, Long.MAX_VALUE - 1L, 1L))
+        .doesNotThrowAnyException();
+  }
+
+  @Test
+  void rejectsAnAggregateOneUnitPastTheLimit() {
+    assertThatThrownBy(() -> Account.requireRepresentableBalance(USER_ID, Long.MAX_VALUE, 1L))
+        .isInstanceOf(BalanceLimitExceededException.class);
+    assertThatThrownBy(() -> Account.requireRepresentableBalance(USER_ID, 1L, Long.MAX_VALUE))
+        .isInstanceOf(BalanceLimitExceededException.class);
+  }
+}


