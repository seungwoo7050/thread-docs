# Java·JSON·Avro 계약 소유권

## `build(maven): initialize Java 17 library`

diff --git a/pom.xml b/pom.xml
new file mode 100644
index 0000000..091a71d
--- /dev/null
+++ b/pom.xml
@@ -0,0 +1,45 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<project xmlns="http://maven.apache.org/POM/4.0.0"
+         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
+         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
+                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
+    <modelVersion>4.0.0</modelVersion>
+
+    <groupId>com.sportsbook</groupId>
+    <artifactId>shared-protocol</artifactId>
+    <version>1.0.0-SNAPSHOT</version>
+    <packaging>jar</packaging>
+
+    <name>shared-protocol</name>
+    <description>Shared domain values and event contracts for sportsbook services.</description>
+
+    <properties>
+        <maven.compiler.release>17</maven.compiler.release>
+        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
+        <compiler.plugin.version>3.13.0</compiler.plugin.version>
+        <source.plugin.version>3.3.1</source.plugin.version>
+    </properties>
+
+    <build>
+        <plugins>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-compiler-plugin</artifactId>
+                <version>${compiler.plugin.version}</version>
+            </plugin>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-source-plugin</artifactId>
+                <version>${source.plugin.version}</version>
+                <executions>
+                    <execution>
+                        <id>attach-sources</id>
+                        <goals>
+                            <goal>jar-no-fork</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
+        </plugins>
+    </build>
+</project>


## `feat(money): define monetary amounts`

diff --git a/src/main/java/com/sportsbook/protocol/value/Money.java b/src/main/java/com/sportsbook/protocol/value/Money.java
new file mode 100644
index 0000000..ec7caeb
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/Money.java
@@ -0,0 +1,82 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonAutoDetect;
+import com.fasterxml.jackson.annotation.JsonAutoDetect.Visibility;
+import java.util.Objects;
+
+/**
+ * Money as {@code long} minor units + a {@link Currency} discriminator (ADR-0003).
+ *
+ * <p>Negative amounts are allowed because ledger entries (debit/credit) need both signs;
+ * domain-level "balance must be non-negative" rules live in wallet-service, not here. All
+ * arithmetic uses {@link Math#addExact} / {@link Math#multiplyExact} / {@link Math#subtractExact} /
+ * {@link Math#negateExact} so silent overflow is impossible — an overflowing operation throws
+ * {@link ArithmeticException}.
+ *
+ * <p>{@code @JsonAutoDetect(isGetterVisibility = NONE)}: Jackson otherwise treats {@code isZero()}
+ * / {@code isPositive()} / {@code isNegative()} as boolean bean properties and leaks them into the
+ * JSON payload alongside the record components. Disabling is-getter discovery limits the wire shape
+ * to {@code amount} + {@code currency}.
+ */
+@JsonAutoDetect(isGetterVisibility = Visibility.NONE)
+public record Money(long amount, Currency currency) implements Comparable<Money> {
+
+  public Money {
+    Objects.requireNonNull(currency, "currency");
+  }
+
+  public Money add(Money other) {
+    requireSameCurrency(other);
+    return new Money(Math.addExact(amount, other.amount), currency);
+  }
+
+  public Money subtract(Money other) {
+    requireSameCurrency(other);
+    return new Money(Math.subtractExact(amount, other.amount), currency);
+  }
+
+  public Money multiply(long multiplier) {
+    return new Money(Math.multiplyExact(amount, multiplier), currency);
+  }
+
+  public Money negate() {
+    return new Money(Math.negateExact(amount), currency);
+  }
+
+  @Override
+  public int compareTo(Money other) {
+    requireSameCurrency(other);
+    return Long.compare(amount, other.amount);
+  }
+
+  public boolean isZero() {
+    return amount == 0L;
+  }
+
+  public boolean isPositive() {
+    return amount > 0L;
+  }
+
+  public boolean isNegative() {
+    return amount < 0L;
+  }
+
+  private void requireSameCurrency(Money other) {
+    if (currency != other.currency) {
+      throw new IllegalArgumentException(
+          "Currency mismatch: " + currency + " vs " + other.currency);
+    }
+  }
+
+  public static Money krw(long amount) {
+    return new Money(amount, Currency.KRW);
+  }
+
+  public static Money usd(long amount) {
+    return new Money(amount, Currency.USD);
+  }
+
+  public static Money zero(Currency currency) {
+    return new Money(0L, currency);
+  }
+}


## `feat(events): define wire monetary amounts`

diff --git a/src/main/avro/.gitkeep b/src/main/avro/.gitkeep
deleted file mode 100644
index 8b13789..0000000
--- a/src/main/avro/.gitkeep
+++ /dev/null
@@ -1 +0,0 @@
-
diff --git a/src/main/avro/com/sportsbook/protocol/event/Money.avsc b/src/main/avro/com/sportsbook/protocol/event/Money.avsc
new file mode 100644
index 0000000..40a65e9
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/Money.avsc
@@ -0,0 +1,10 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "Money",
+  "doc": "Monetary amount in long minor units paired with an ISO currency code; mirrors com.sportsbook.protocol.value.Money on the wire.",
+  "fields": [
+    {"name": "amount", "type": "long", "doc": "Minor units (e.g. cents, won)."},
+    {"name": "currency", "type": "string", "doc": "ISO 4217 code such as KRW or USD."}
+  ]
+}


## `feat(idempotency): define request keys`

diff --git a/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java b/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java
new file mode 100644
index 0000000..f2997a2
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/value/IdempotencyKey.java
@@ -0,0 +1,57 @@
+package com.sportsbook.protocol.value;
+
+import com.fasterxml.jackson.annotation.JsonCreator;
+import com.fasterxml.jackson.annotation.JsonValue;
+import java.util.Objects;
+import java.util.UUID;
+import java.util.regex.Pattern;
+
+/**
+ * Caller-supplied idempotency key for mutating requests (ADR-0005).
+ *
+ * <p>The first request with a given key succeeds and its outcome is recorded; any retry with the
+ * same key returns the same outcome rather than re-executing the side effect. The protocol layer
+ * here only validates the key's wire shape — the actual dedup happens in each service via a unique
+ * DB constraint (strong) plus a Redis cache (fast path), as described in ADR-0005.
+ *
+ * <p>Constraints:
+ *
+ * <ul>
+ *   <li>Non-blank, length ≤ 128 — fits comfortably in an HTTP header and a typical varchar column.
+ *   <li>Printable ASCII only ({@code 0x20–0x7E}) — avoids encoding ambiguity when the key crosses
+ *       Kafka headers, HTTP headers, and DB columns.
+ * </ul>
+ *
+ * <p>{@code toString} returns the raw value: idempotency keys are not secrets and showing them in
+ * logs is desirable for tracing a retry across services.
+ */
+public record IdempotencyKey(@JsonValue String value) {
+
+  public static final int MAX_LENGTH = 128;
+  private static final Pattern ASCII_PRINTABLE = Pattern.compile("\\A[\\x20-\\x7E]+\\z");
+
+  public IdempotencyKey {
+    Objects.requireNonNull(value, "value");
+    if (value.isBlank()) {
+      throw new IllegalArgumentException("IdempotencyKey must not be blank");
+    }
+    if (value.length() > MAX_LENGTH) {
+      throw new IllegalArgumentException(
+          "IdempotencyKey length " + value.length() + " exceeds max " + MAX_LENGTH);
+    }
+    if (!ASCII_PRINTABLE.matcher(value).matches()) {
+      throw new IllegalArgumentException(
+          "IdempotencyKey must contain only printable ASCII (0x20-0x7E)");
+    }
+  }
+
+  @JsonCreator
+  public static IdempotencyKey of(String value) {
+    return new IdempotencyKey(value);
+  }
+
+  /** Generates a fresh key as a UUID v4 string. Useful for clients that don't supply one. */
+  public static IdempotencyKey random() {
+    return new IdempotencyKey(UUID.randomUUID().toString());
+  }
+}


## `feat(bet): compose self-consistent slips`

diff --git a/src/main/java/com/sportsbook/protocol/domain/BetSlip.java b/src/main/java/com/sportsbook/protocol/domain/BetSlip.java
new file mode 100644
index 0000000..d0fa921
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/domain/BetSlip.java
@@ -0,0 +1,104 @@
+package com.sportsbook.protocol.domain;
+
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.UserId;
+import java.time.Instant;
+import java.util.List;
+import java.util.Objects;
+
+/**
+ * shared-protocol holds only structural invariants — "is this BetSlip self-consistent as a data
+ * shape?". Domain validation (ADR-0008 L1 Same Market, L2 Same Event, L4/L5 policy bounds, odds
+ * slippage tolerance) lives in betting-service's BetSlipValidator. The invariants here prevent the
+ * wire from carrying nonsensical shapes (Single with 3 selections, SETTLED without a result, etc.)
+ * which protect every consumer downstream.
+ */
+public record BetSlip(
+    BetId id,
+    UserId userId,
+    BetSlipType type,
+    List<BetSelection> selections,
+    Money stake,
+    BetStatus status,
+    Instant placedAt,
+    SettlementResult settlementResult,
+    Instant settledAt,
+    Money payout) {
+
+  public static final int MULTIPLE_MIN_SELECTIONS = 2;
+
+  public BetSlip {
+    Objects.requireNonNull(id, "id");
+    Objects.requireNonNull(userId, "userId");
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(selections, "selections");
+    Objects.requireNonNull(stake, "stake");
+    Objects.requireNonNull(status, "status");
+    Objects.requireNonNull(placedAt, "placedAt");
+
+    if (selections.isEmpty()) {
+      throw new IllegalArgumentException("BetSlip must have at least one selection");
+    }
+    if (!stake.isPositive()) {
+      throw new IllegalArgumentException("BetSlip stake must be positive (got " + stake + ")");
+    }
+
+    if (type instanceof BetSlipType.Single && selections.size() != 1) {
+      throw new IllegalArgumentException(
+          "Single slip must have exactly 1 selection (got " + selections.size() + ")");
+    }
+    if (type instanceof BetSlipType.Multiple && selections.size() < MULTIPLE_MIN_SELECTIONS) {
+      throw new IllegalArgumentException(
+          "Multiple slip must have at least "
+              + MULTIPLE_MIN_SELECTIONS
+              + " selections (got "
+              + selections.size()
+              + ")");
+    }
+    if (type instanceof BetSlipType.System sys && selections.size() != sys.totalSelections()) {
+      throw new IllegalArgumentException(
+          "System slip selections.size ("
+              + selections.size()
+              + ") must equal type.totalSelections ("
+              + sys.totalSelections()
+              + ")");
+    }
+
+    if (status == BetStatus.SETTLED) {
+      if (settlementResult == null) {
+        throw new IllegalArgumentException("SETTLED slip must have settlementResult");
+      }
+      if (settledAt == null) {
+        throw new IllegalArgumentException("SETTLED slip must have settledAt");
+      }
+    } else {
+      if (settlementResult != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have settlementResult (status=" + status + ")");
+      }
+      if (settledAt != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have settledAt (status=" + status + ")");
+      }
+      if (payout != null) {
+        throw new IllegalArgumentException(
+            "Non-SETTLED slip must not have payout (status=" + status + ")");
+      }
+    }
+
+    if (settlementResult == SettlementResult.WON
+        || settlementResult == SettlementResult.PUSH
+        || settlementResult == SettlementResult.VOID) {
+      if (payout == null) {
+        throw new IllegalArgumentException(
+            settlementResult + " slip must have payout (winnings or refund)");
+      }
+    } else if (settlementResult == SettlementResult.LOST && payout != null) {
+      throw new IllegalArgumentException("LOST slip must not have payout");
+    }
+
+    // Defensive copy: prevent external mutation of the selections list after construction.
+    selections = List.copyOf(selections);
+  }
+}


## `feat(errors): define problem details`

diff --git a/src/main/java/com/sportsbook/protocol/error/ErrorCode.java b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
index 636b76b..bc98c06 100644
--- a/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
+++ b/src/main/java/com/sportsbook/protocol/error/ErrorCode.java
@@ -36,4 +36,16 @@ public enum ErrorCode {
   public String title() {
     return title;
   }
+
+  public ProblemDetail toProblemDetail() {
+    return toProblemDetail(null, null, null);
+  }
+
+  public ProblemDetail toProblemDetail(String detail) {
+    return toProblemDetail(detail, null, null);
+  }
+
+  public ProblemDetail toProblemDetail(String detail, URI instance, String correlationId) {
+    return new ProblemDetail(type, title, httpStatus, name(), detail, instance, correlationId);
+  }
 }
diff --git a/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java b/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java
new file mode 100644
index 0000000..c3c58a1
--- /dev/null
+++ b/src/main/java/com/sportsbook/protocol/error/ProblemDetail.java
@@ -0,0 +1,33 @@
+package com.sportsbook.protocol.error;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import java.net.URI;
+import java.util.Objects;
+
+/**
+ * RFC 7807 ProblemDetail with two sportsbook-specific extensions: {@code errorCode} (stable
+ * machine-readable identifier — the {@link ErrorCode} enum name) and {@code correlationId}
+ * (Micrometer/OTel trace id).
+ *
+ * <p>Self-defined rather than reusing {@code org.springframework.http.ProblemDetail} so this
+ * library stays framework-neutral. Spring's class pulls in spring-web transitively which is
+ * unwanted for non-web consumers (background workers, batch jobs).
+ *
+ * <p>{@code @JsonInclude(NON_NULL)} keeps the wire compact when optional fields are absent.
+ */
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record ProblemDetail(
+    URI type,
+    String title,
+    int status,
+    String errorCode,
+    String detail,
+    URI instance,
+    String correlationId) {
+
+  public ProblemDetail {
+    Objects.requireNonNull(type, "type");
+    Objects.requireNonNull(title, "title");
+    Objects.requireNonNull(errorCode, "errorCode");
+  }
+}


