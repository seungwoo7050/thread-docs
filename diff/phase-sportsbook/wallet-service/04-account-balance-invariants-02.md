## `test(domain): verify deposit and withdrawal arithmetic`

diff --git a/src/test/java/com/sportsbook/wallet/domain/AccountAvailableFundsTest.java b/src/test/java/com/sportsbook/wallet/domain/AccountAvailableFundsTest.java
new file mode 100644
index 0000000..099f850
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/AccountAvailableFundsTest.java
@@ -0,0 +1,63 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.InsufficientBalanceException;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+
+class AccountAvailableFundsTest {
+
+  private static final Instant OPENED_AT = Instant.parse("2026-01-01T00:00:00Z");
+  private static final Instant CHANGED_AT = Instant.parse("2026-01-01T00:00:01Z");
+
+  private Account account;
+
+  @BeforeEach
+  void openAccount() {
+    account = Account.openFor(UUID.randomUUID(), Currency.KRW, OPENED_AT);
+  }
+
+  @Test
+  void depositsAndWithdrawsAvailableFunds() {
+    account.increaseAvailable(Money.krw(100L), CHANGED_AT);
+    account.decreaseAvailable(Money.krw(40L), CHANGED_AT.plusSeconds(1L));
+
+    assertThat(account.available()).isEqualTo(Money.krw(60L));
+    assertThat(account.updatedAt()).isEqualTo(CHANGED_AT.plusSeconds(1L));
+  }
+
+  @Test
+  void validatesPositiveMoneyBeforeMutation() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> account.increaseAvailable(Money.zero(Currency.KRW), CHANGED_AT));
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> account.decreaseAvailable(Money.krw(-1L), CHANGED_AT));
+
+    assertThat(account.available()).isEqualTo(Money.zero(Currency.KRW));
+  }
+
+  @Test
+  void rejectsCurrencyMismatchAndInsufficientFunds() {
+    assertThatThrownBy(() -> account.increaseAvailable(Money.usd(1L), CHANGED_AT))
+        .isInstanceOf(CurrencyMismatchException.class);
+    assertThatThrownBy(() -> account.decreaseAvailable(Money.krw(1L), CHANGED_AT))
+        .isInstanceOf(InsufficientBalanceException.class);
+  }
+
+  @Test
+  void rejectsAnIncreasePastTheAggregateLimit() {
+    account.increaseAvailable(Money.krw(Long.MAX_VALUE), CHANGED_AT);
+
+    assertThatThrownBy(() -> account.increaseAvailable(Money.krw(1L), CHANGED_AT))
+        .isInstanceOf(BalanceLimitExceededException.class);
+  }
+}


## `test(domain): verify locked-bucket invariants`

diff --git a/src/test/java/com/sportsbook/wallet/domain/AccountLockedFundsTest.java b/src/test/java/com/sportsbook/wallet/domain/AccountLockedFundsTest.java
new file mode 100644
index 0000000..c919aac
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/AccountLockedFundsTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.InsufficientBalanceException;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+
+class AccountLockedFundsTest {
+
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+
+  private Account account;
+
+  @BeforeEach
+  void fundAccount() {
+    account = Account.openFor(UUID.randomUUID(), Currency.KRW, NOW);
+    account.increaseAvailable(Money.krw(100L), NOW);
+  }
+
+  @Test
+  void stagesAndRefundsLockedFundsWithoutChangingTheTotal() {
+    account.moveAvailableToLocked(Money.krw(60L), NOW.plusSeconds(1L));
+    account.moveLockedToAvailable(Money.krw(20L), NOW.plusSeconds(2L));
+
+    assertThat(account.available()).isEqualTo(Money.krw(60L));
+    assertThat(account.locked()).isEqualTo(Money.krw(40L));
+    assertThat(account.total()).isEqualTo(Money.krw(100L));
+  }
+
+  @Test
+  void forfeitsOnlyTheLockedBucket() {
+    account.moveAvailableToLocked(Money.krw(80L), NOW);
+    account.forfeitLocked(Money.krw(30L), NOW.plusSeconds(1L));
+
+    assertThat(account.available()).isEqualTo(Money.krw(20L));
+    assertThat(account.locked()).isEqualTo(Money.krw(50L));
+    assertThat(account.total()).isEqualTo(Money.krw(70L));
+  }
+
+  @Test
+  void rejectsMissingLockedFundsAndCurrencyMismatch() {
+    assertThatThrownBy(() -> account.moveLockedToAvailable(Money.krw(1L), NOW))
+        .isInstanceOf(InsufficientBalanceException.class);
+    assertThatThrownBy(() -> account.forfeitLocked(Money.usd(1L), NOW))
+        .isInstanceOf(CurrencyMismatchException.class);
+
+    assertThat(account.total()).isEqualTo(Money.krw(100L));
+  }
+}


## `test(persistence): verify account and ledger schema constraints`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
new file mode 100644
index 0000000..9f25867
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.wallet.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.math.BigDecimal;
+import java.math.BigInteger;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = "spring.test.database.replace=NONE")
+@Testcontainers
+@Transactional(propagation = Propagation.NOT_SUPPORTED)
+class WalletPersistenceTest {
+  @Container
+  static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @Autowired JdbcTemplate jdbc;
+
+  @DynamicPropertySource
+  static void databaseProperties(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+  }
+
+  @Test
+  void migratesFinalConstraintsAndAdjustmentReason() {
+    UUID userId = UUID.fromString("019b76da-a000-7000-8000-000000000010");
+    jdbc.update(
+        """
+        INSERT INTO account (
+            user_id, available_currency, locked_currency, created_at, updated_at
+        ) VALUES (?, 'KRW', 'KRW', now(), now())
+        """,
+        userId);
+    jdbc.update(
+        """
+        INSERT INTO ledger_entry (
+            entry_id, account_id, bucket, side, amount, currency, reason,
+            idempotency_key, operation_group_id, created_at
+        ) VALUES (?, ?, 'AVAILABLE', 'DEBIT', 1, 'KRW', 'BET_ADJUSTMENT', ?, ?, now())
+        """,
+        UUID.randomUUID(),
+        userId,
+        "adjustment:persistence",
+        UUID.randomUUID());
+
+    BigInteger debt = BigInteger.ONE.shiftLeft(Long.SIZE - 1);
+    jdbc.update(
+        "UPDATE account SET recovery_debt_amount=?, recovery_frozen_at=now() WHERE user_id=?",
+        debt,
+        userId);
+    jdbc.update(
+        "UPDATE account SET updated_at=created_at - interval '1 second' WHERE user_id=?", userId);
+
+    assertThat(
+            jdbc.queryForObject(
+                    "SELECT recovery_debt_amount FROM account WHERE user_id=?",
+                    BigDecimal.class,
+                    userId)
+                .toBigIntegerExact())
+        .isEqualTo(debt);
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT reason FROM ledger_entry WHERE idempotency_key=?",
+                String.class,
+                "adjustment:persistence"))
+        .isEqualTo("BET_ADJUSTMENT");
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT updated_at < created_at FROM account WHERE user_id=?",
+                Boolean.class,
+                userId))
+        .isTrue();
+    assertThatThrownBy(
+            () ->
+                jdbc.update(
+                    "UPDATE account SET available_amount=?, locked_amount=? WHERE user_id=?",
+                    Long.MAX_VALUE,
+                    1L,
+                    userId))
+        .isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
+  }
+}
