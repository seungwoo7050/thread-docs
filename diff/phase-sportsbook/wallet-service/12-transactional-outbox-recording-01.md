# 트랜잭션 아웃박스 기록

## `build(flyway): create an ordered transactional outbox`

diff --git a/src/main/resources/db/migration/V3__transactional_outbox.sql b/src/main/resources/db/migration/V3__transactional_outbox.sql
new file mode 100644
index 0000000..0e3ede1
--- /dev/null
+++ b/src/main/resources/db/migration/V3__transactional_outbox.sql
@@ -0,0 +1,66 @@
+CREATE TABLE outbox_stream (
+    topic VARCHAR(128) NOT NULL,
+    partition_key VARCHAR(128) NOT NULL,
+    last_sequence BIGINT NOT NULL DEFAULT 0,
+    PRIMARY KEY (topic, partition_key),
+    CONSTRAINT ck_outbox_stream_sequence CHECK (last_sequence >= 0)
+);
+
+CREATE TABLE outbox_event (
+    event_id UUID PRIMARY KEY,
+    operation_key VARCHAR(128) NOT NULL,
+    topic VARCHAR(128) NOT NULL,
+    partition_key VARCHAR(128) NOT NULL,
+    schema_name VARCHAR(128) NOT NULL,
+    deduplication_key VARCHAR(128) NOT NULL,
+    stream_sequence BIGINT NOT NULL,
+    payload BYTEA NOT NULL,
+    created_at TIMESTAMPTZ NOT NULL,
+    available_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
+    published_at TIMESTAMPTZ,
+    lease_owner VARCHAR(128),
+    lease_until TIMESTAMPTZ,
+    lease_version BIGINT NOT NULL DEFAULT 0,
+    attempt_count INTEGER NOT NULL DEFAULT 0,
+    last_error VARCHAR(1024),
+    CONSTRAINT fk_outbox_operation
+        FOREIGN KEY (operation_key)
+        REFERENCES wallet_operation(idempotency_key)
+        DEFERRABLE INITIALLY DEFERRED,
+    CONSTRAINT fk_outbox_stream
+        FOREIGN KEY (topic, partition_key)
+        REFERENCES outbox_stream(topic, partition_key),
+    CONSTRAINT uq_outbox_semantic_event UNIQUE (topic, deduplication_key),
+    CONSTRAINT uq_outbox_operation UNIQUE (operation_key),
+    CONSTRAINT uq_outbox_stream_sequence UNIQUE (topic, partition_key, stream_sequence),
+    CONSTRAINT ck_outbox_strings CHECK (
+        btrim(operation_key) <> ''
+        AND btrim(topic) <> ''
+        AND btrim(partition_key) <> ''
+        AND btrim(schema_name) <> ''
+        AND btrim(deduplication_key) <> ''
+    ),
+    CONSTRAINT ck_outbox_payload CHECK (octet_length(payload) > 0),
+    CONSTRAINT ck_outbox_stream_position CHECK (stream_sequence > 0),
+    CONSTRAINT ck_outbox_attempt_count CHECK (attempt_count >= 0),
+    CONSTRAINT ck_outbox_lease_version CHECK (lease_version >= 0),
+    CONSTRAINT ck_outbox_lease_pair CHECK (
+        (lease_owner IS NULL AND lease_until IS NULL)
+        OR (lease_owner IS NOT NULL AND lease_until IS NOT NULL)
+    ),
+    CONSTRAINT ck_outbox_published_lease CHECK (
+        published_at IS NULL OR (lease_owner IS NULL AND lease_until IS NULL)
+    )
+);
+
+CREATE INDEX ix_outbox_claim_due
+    ON outbox_event (available_at, stream_sequence)
+    WHERE published_at IS NULL;
+
+CREATE INDEX ix_outbox_fifo
+    ON outbox_event (topic, partition_key, stream_sequence)
+    WHERE published_at IS NULL;
+
+CREATE INDEX ix_outbox_lease_expiry
+    ON outbox_event (lease_until)
+    WHERE published_at IS NULL AND lease_owner IS NOT NULL;


## `test(flyway): verify ordered outbox constraints`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/OutboxMigrationTest.java b/src/test/java/com/sportsbook/wallet/outbox/OutboxMigrationTest.java
new file mode 100644
index 0000000..743b317
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/OutboxMigrationTest.java
@@ -0,0 +1,71 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import java.util.stream.Collectors;
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
+class OutboxMigrationTest {
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
+  void definesCommittedStreamPositionsAndOneEventPerOperation() {
+    Map<String, String> constraints =
+        jdbc
+            .query(
+                """
+                SELECT conname, pg_get_constraintdef(oid) AS definition
+                FROM pg_constraint
+                WHERE conrelid IN ('outbox_stream'::regclass, 'outbox_event'::regclass)
+                """,
+                (result, row) -> Map.entry(result.getString(1), result.getString(2)))
+            .stream()
+            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
+
+    assertThat(constraints.get("outbox_stream_pkey"))
+        .contains("PRIMARY KEY (topic, partition_key)");
+    assertThat(constraints.get("fk_outbox_stream"))
+        .contains("FOREIGN KEY (topic, partition_key)")
+        .contains("outbox_stream(topic, partition_key)");
+    assertThat(constraints.get("uq_outbox_stream_sequence"))
+        .contains("UNIQUE (topic, partition_key, stream_sequence)");
+    assertThat(constraints.get("uq_outbox_operation")).contains("UNIQUE (operation_key)");
+    assertThat(constraints.get("fk_outbox_operation")).contains("DEFERRABLE INITIALLY DEFERRED");
+    assertThat(constraints).doesNotContainKey("ck_outbox_timestamps");
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT column_default FROM information_schema.columns "
+                    + "WHERE table_name='outbox_event' AND column_name='available_at'",
+                String.class))
+        .contains("clock_timestamp()");
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT indexdef FROM pg_indexes WHERE indexname='ix_outbox_fifo'", String.class))
+        .contains("topic, partition_key, stream_sequence")
+        .contains("published_at IS NULL");
+  }
+}


## `feat(outbox): model pending deduplicated messages`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/PendingOutboxMessage.java b/src/main/java/com/sportsbook/wallet/outbox/PendingOutboxMessage.java
new file mode 100644
index 0000000..a8f2cfb
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/PendingOutboxMessage.java
@@ -0,0 +1,61 @@
+package com.sportsbook.wallet.outbox;
+
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record PendingOutboxMessage(
+    UUID eventId,
+    String operationKey,
+    String topic,
+    String partitionKey,
+    String schemaName,
+    String deduplicationKey,
+    byte[] payload,
+    Instant createdAt) {
+
+  public PendingOutboxMessage {
+    eventId = Objects.requireNonNull(eventId, "eventId");
+    operationKey = required(operationKey, "operationKey");
+    topic = required(topic, "topic");
+    partitionKey = required(partitionKey, "partitionKey");
+    schemaName = required(schemaName, "schemaName");
+    deduplicationKey = required(deduplicationKey, "deduplicationKey");
+    payload = Objects.requireNonNull(payload, "payload").clone();
+    if (payload.length == 0) {
+      throw new IllegalArgumentException("payload must not be empty");
+    }
+    createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  public static PendingOutboxMessage create(
+      String operationKey,
+      String topic,
+      String partitionKey,
+      String schemaName,
+      String deduplicationKey,
+      byte[] payload,
+      Instant now) {
+    return new PendingOutboxMessage(
+        UUID.randomUUID(),
+        operationKey,
+        topic,
+        partitionKey,
+        schemaName,
+        deduplicationKey,
+        payload,
+        now);
+  }
+
+  @Override
+  public byte[] payload() {
+    return payload.clone();
+  }
+
+  private static String required(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
+  }
+}


## `test(outbox): verify immutable message payloads`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/PendingOutboxMessageTest.java b/src/test/java/com/sportsbook/wallet/outbox/PendingOutboxMessageTest.java
new file mode 100644
index 0000000..7c3ca74
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/PendingOutboxMessageTest.java
@@ -0,0 +1,43 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class PendingOutboxMessageTest {
+
+  @Test
+  void copiesThePayloadAtConstruction() {
+    byte[] payload = {1, 2, 3};
+
+    PendingOutboxMessage message = message(payload);
+    payload[0] = 9;
+
+    assertThat(message.payload()).containsExactly(1, 2, 3);
+  }
+
+  @Test
+  void neverExposesItsPayloadArray() {
+    PendingOutboxMessage message = message(new byte[] {1, 2, 3});
+
+    byte[] exposed = message.payload();
+    exposed[1] = 9;
+
+    assertThat(message.payload()).containsExactly(1, 2, 3);
+  }
+
+  private PendingOutboxMessage message(byte[] payload) {
+    Instant now = Instant.parse("2026-08-21T00:00:00Z");
+    return new PendingOutboxMessage(
+        UUID.fromString("0198ca71-8000-7000-8000-000000000001"),
+        "operation-1",
+        "wallet.debited.v1",
+        "bet-1",
+        "WalletDebited",
+        "debit:bet-1",
+        payload,
+        now);
+  }
+}


## `feat(outbox): map durable stream positions`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxEvent.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxEvent.java
new file mode 100644
index 0000000..3a3a2b3
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxEvent.java
@@ -0,0 +1,97 @@
+package com.sportsbook.wallet.outbox;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.UUID;
+
+@Entity
+@Table(name = "outbox_event")
+public class OutboxEvent {
+
+  @Id
+  @Column(name = "event_id", nullable = false, updatable = false)
+  private UUID eventId;
+
+  @Column(name = "operation_key", nullable = false, updatable = false)
+  private String operationKey;
+
+  @Column(nullable = false, updatable = false)
+  private String topic;
+
+  @Column(name = "partition_key", nullable = false, updatable = false)
+  private String partitionKey;
+
+  @Column(name = "schema_name", nullable = false, updatable = false)
+  private String schemaName;
+
+  @Column(name = "deduplication_key", nullable = false, updatable = false)
+  private String deduplicationKey;
+
+  @Column(name = "stream_sequence", nullable = false, updatable = false)
+  private long streamSequence;
+
+  @Column(nullable = false, updatable = false)
+  private byte[] payload;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "available_at", nullable = false, insertable = false, updatable = false)
+  private Instant availableAt;
+
+  protected OutboxEvent() {}
+
+  private OutboxEvent(PendingOutboxMessage message, long streamSequence) {
+    if (streamSequence < 1L) {
+      throw new IllegalArgumentException("streamSequence must be positive");
+    }
+    eventId = message.eventId();
+    operationKey = message.operationKey();
+    topic = message.topic();
+    partitionKey = message.partitionKey();
+    schemaName = message.schemaName();
+    deduplicationKey = message.deduplicationKey();
+    this.streamSequence = streamSequence;
+    payload = message.payload();
+    createdAt = message.createdAt();
+  }
+
+  public static OutboxEvent pending(PendingOutboxMessage message, long streamSequence) {
+    return new OutboxEvent(message, streamSequence);
+  }
+
+  public UUID eventId() {
+    return eventId;
+  }
+
+  public String operationKey() {
+    return operationKey;
+  }
+
+  public String topic() {
+    return topic;
+  }
+
+  public String partitionKey() {
+    return partitionKey;
+  }
+
+  public String schemaName() {
+    return schemaName;
+  }
+
+  public String deduplicationKey() {
+    return deduplicationKey;
+  }
+
+  public long streamSequence() {
+    return streamSequence;
+  }
+
+  public byte[] payload() {
+    return payload.clone();
+  }
+}


## `test(outbox): verify durable stream positions`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/OutboxEventTest.java b/src/test/java/com/sportsbook/wallet/outbox/OutboxEventTest.java
new file mode 100644
index 0000000..7b03b52
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/OutboxEventTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import jakarta.persistence.Column;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+import org.springframework.test.util.ReflectionTestUtils;
+
+class OutboxEventTest {
+  private static final Instant NOW = Instant.parse("2026-01-05T00:00:00Z");
+
+  @Test
+  void mapsAnImmutableMessageAtItsCommittedStreamPosition() throws NoSuchFieldException {
+    byte[] payload = {1, 2, 3};
+    PendingOutboxMessage message =
+        PendingOutboxMessage.create(
+            "operation-1", "wallet.debited.v1", "user-1", "WalletDebited", "bet-1", payload, NOW);
+
+    OutboxEvent event = OutboxEvent.pending(message, 7L);
+    payload[0] = 9;
+    byte[] exposed = event.payload();
+    exposed[1] = 9;
+
+    assertThat(event.operationKey()).isEqualTo("operation-1");
+    assertThat(event.topic()).isEqualTo("wallet.debited.v1");
+    assertThat(event.partitionKey()).isEqualTo("user-1");
+    assertThat(event.schemaName()).isEqualTo("WalletDebited");
+    assertThat(event.deduplicationKey()).isEqualTo("bet-1");
+    assertThat(event.streamSequence()).isEqualTo(7L);
+    assertThat(event.payload()).containsExactly(1, 2, 3);
+    Column deadline = OutboxEvent.class.getDeclaredField("availableAt").getAnnotation(Column.class);
+    assertThat(deadline.insertable()).isFalse();
+    assertThat(deadline.updatable()).isFalse();
+    assertThat(ReflectionTestUtils.getField(event, "availableAt")).isNull();
+    assertThatThrownBy(() -> OutboxEvent.pending(message, 0L))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("positive");
+  }
+}


## `feat(repository): persist and list outbox messages`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/OutboxEventRepository.java b/src/main/java/com/sportsbook/wallet/persistence/OutboxEventRepository.java
new file mode 100644
index 0000000..d1bd2c4
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/OutboxEventRepository.java
@@ -0,0 +1,11 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.outbox.OutboxEvent;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+
+public interface OutboxEventRepository extends JpaRepository<OutboxEvent, UUID> {
+
+  List<OutboxEvent> findAllByOrderByTopicAscPartitionKeyAscStreamSequenceAsc();
+}


## `test(repository): persist and deduplicate outbox messages`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxEventRepositoryTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxEventRepositoryTest.java
new file mode 100644
index 0000000..4fa5584
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxEventRepositoryTest.java
@@ -0,0 +1,96 @@
+package com.sportsbook.wallet.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.outbox.OutboxEvent;
+import com.sportsbook.wallet.outbox.PendingOutboxMessage;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.dao.DataIntegrityViolationException;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = "spring.test.database.replace=NONE")
+@Testcontainers
+class OutboxEventRepositoryTest {
+  private static final Instant NOW = Instant.parse("2999-01-06T00:00:00Z");
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-000000000030");
+
+  @Container
+  static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @Autowired JdbcTemplate jdbc;
+  @Autowired OutboxEventRepository events;
+  @Autowired WalletOperationRepository operations;
+
+  @DynamicPropertySource
+  static void databaseProperties(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+  }
+
+  @Test
+  void persistsOrderedRowsAndRejectsSemanticDuplicates() {
+    operations.saveAllAndFlush(java.util.List.of(operation("outbox:one"), operation("outbox:two")));
+    jdbc.update(
+        "INSERT INTO outbox_stream(topic, partition_key) VALUES (?, ?)",
+        "wallet.debited.v1",
+        USER_ID.toString());
+    PendingOutboxMessage first = message("outbox:one", "bet-1");
+    events.saveAndFlush(OutboxEvent.pending(first, 1L));
+
+    assertThat(events.findAllByOrderByTopicAscPartitionKeyAscStreamSequenceAsc())
+        .singleElement()
+        .satisfies(
+            stored -> {
+              assertThat(stored.eventId()).isEqualTo(first.eventId());
+              assertThat(stored.streamSequence()).isEqualTo(1L);
+            });
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT available_at < created_at FROM outbox_event WHERE event_id=?",
+                Boolean.class,
+                first.eventId()))
+        .isTrue();
+    assertThatThrownBy(
+            () -> events.saveAndFlush(OutboxEvent.pending(message("outbox:two", "bet-1"), 2L)))
+        .isInstanceOf(DataIntegrityViolationException.class);
+  }
+
+  private static PendingOutboxMessage message(String operationKey, String deduplicationKey) {
+    return PendingOutboxMessage.create(
+        operationKey,
+        "wallet.debited.v1",
+        USER_ID.toString(),
+        "WalletDebited",
+        deduplicationKey,
+        new byte[] {1},
+        NOW);
+  }
+
+  private static WalletOperation operation(String key) {
+    return WalletOperation.succeeded(
+        IdempotencyKey.of(key),
+        WalletCaller.PLATFORM,
+        WalletOperationKind.DEPOSIT,
+        USER_ID,
+        Money.krw(1L),
+        "a".repeat(64),
+        UUID.randomUUID(),
+        NOW);
+  }
+}


