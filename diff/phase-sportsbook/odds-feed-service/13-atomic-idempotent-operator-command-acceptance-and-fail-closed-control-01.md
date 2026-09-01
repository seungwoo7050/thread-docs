# 원자적 멱등 운영자 명령 접수와 fail-close 제어

## `feat(commands): define operator market commands`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java
new file mode 100644
index 0000000..50c8f7a
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java
@@ -0,0 +1,13 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import java.util.UUID;
+
+/** The durable result of accepting an operator action. */
+public record OperatorActionSubmission(
+    Outcome outcome, UUID actionId, long sequence, long predecessor, String recordId) {
+
+  public enum Outcome {
+    CREATED,
+    REPLAYED
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorMarketAction.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorMarketAction.java
new file mode 100644
index 0000000..9d090e2
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorMarketAction.java
@@ -0,0 +1,36 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+/** An immutable market transition accepted from an internal operator. */
+public record OperatorMarketAction(
+    UUID actionId,
+    EventId eventId,
+    MarketId marketId,
+    MarketStatus previousStatus,
+    MarketStatus announcedStatus,
+    MarketStatus requestedStatus,
+    String reason,
+    long sequence,
+    long predecessor,
+    Instant occurredAt) {
+
+  public OperatorMarketAction {
+    Objects.requireNonNull(actionId, "actionId");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(marketId, "marketId");
+    Objects.requireNonNull(previousStatus, "previousStatus");
+    Objects.requireNonNull(announcedStatus, "announcedStatus");
+    Objects.requireNonNull(requestedStatus, "requestedStatus");
+    Objects.requireNonNull(reason, "reason");
+    Objects.requireNonNull(occurredAt, "occurredAt");
+    if (sequence <= 0 || predecessor != sequence - 1) {
+      throw new IllegalArgumentException("Invalid operator action sequence");
+    }
+  }
+}


## `test(commands): verify operator command invariants`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorMarketActionTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorMarketActionTest.java
new file mode 100644
index 0000000..efa2bec
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorMarketActionTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class OperatorMarketActionTest {
+
+  private static final UUID ACTION_ID = UUID.randomUUID();
+  private static final EventId EVENT_ID = new EventId(UUID.randomUUID());
+  private static final MarketId MARKET_ID = new MarketId(UUID.randomUUID());
+  private static final Instant OCCURRED_AT = Instant.parse("2026-08-21T05:00:00Z");
+
+  @Test
+  void carriesTheAcceptedTransition() {
+    OperatorMarketAction action = action(2, 1, "incident");
+
+    assertThat(action.actionId()).isEqualTo(ACTION_ID);
+    assertThat(action.eventId()).isEqualTo(EVENT_ID);
+    assertThat(action.marketId()).isEqualTo(MARKET_ID);
+    assertThat(action.previousStatus()).isEqualTo(MarketStatus.OPEN);
+    assertThat(action.announcedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(action.requestedStatus()).isEqualTo(MarketStatus.SUSPENDED);
+    assertThat(action.reason()).isEqualTo("incident");
+    assertThat(action.occurredAt()).isEqualTo(OCCURRED_AT);
+  }
+
+  @Test
+  void rejectsMissingRequiredFields() {
+    assertThatThrownBy(() -> action(1, 0, null)).isInstanceOf(NullPointerException.class);
+  }
+
+  @Test
+  void requiresPositiveConsecutiveSequences() {
+    assertThatThrownBy(() -> action(0, -1, "incident"))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> action(3, 1, "incident")).isInstanceOf(IllegalArgumentException.class);
+  }
+
+  private static OperatorMarketAction action(long sequence, long predecessor, String reason) {
+    return new OperatorMarketAction(
+        ACTION_ID,
+        EVENT_ID,
+        MARKET_ID,
+        MarketStatus.OPEN,
+        MarketStatus.SUSPENDED,
+        MarketStatus.SUSPENDED,
+        reason,
+        sequence,
+        predecessor,
+        OCCURRED_AT);
+  }
+}


## `feat(commands): fingerprint operator requests`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprint.java b/src/main/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprint.java
new file mode 100644
index 0000000..8201b19
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprint.java
@@ -0,0 +1,62 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.UUID;
+
+/** Computes canonical hashes for requests and Redis identities. */
+public final class MarketActionFingerprint {
+
+  private static final String REQUEST_VERSION = "1";
+  private static final String CALLER = "admin-api";
+  private static final String IDEMPOTENCY_DOMAIN = "sportsbook-idempotency-key-v1";
+
+  private MarketActionFingerprint() {}
+
+  public static String request(
+      UUID eventId, UUID marketId, MarketStatus requestedStatus, String reason) {
+    MessageDigest digest = newDigest();
+    updateString(digest, REQUEST_VERSION);
+    updateString(digest, CALLER);
+    updateString(digest, action(requestedStatus));
+    updateString(digest, eventId.toString());
+    updateString(digest, marketId.toString());
+    updateString(digest, requestedStatus.name());
+    updateString(digest, reason);
+    return HexFormat.of().formatHex(digest.digest());
+  }
+
+  public static String idempotencyKey(IdempotencyKey key) {
+    MessageDigest digest = newDigest();
+    updateString(digest, IDEMPOTENCY_DOMAIN);
+    updateString(digest, key.value());
+    return HexFormat.of().formatHex(digest.digest());
+  }
+
+  private static String action(MarketStatus requestedStatus) {
+    return switch (requestedStatus) {
+      case SUSPENDED -> "suspend";
+      case CLOSED -> "close";
+      case OPEN -> "reopen";
+    };
+  }
+
+  private static void updateString(MessageDigest digest, String value) {
+    byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
+    digest.update(ByteBuffer.allocate(Integer.BYTES).putInt(bytes.length).array());
+    digest.update(bytes);
+  }
+
+  private static MessageDigest newDigest() {
+    try {
+      return MessageDigest.getInstance("SHA-256");
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("SHA-256 is unavailable", exception);
+    }
+  }
+}


## `test(commands): verify canonical request fingerprints`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprintTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprintTest.java
new file mode 100644
index 0000000..53b2829
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/MarketActionFingerprintTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.MarketStatus;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class MarketActionFingerprintTest {
+
+  private static final UUID EVENT_ID = UUID.fromString("00000000-0000-0000-0000-000000000001");
+  private static final UUID MARKET_ID = UUID.fromString("00000000-0000-0000-0000-000000000002");
+
+  @Test
+  void locksCanonicalSha256Fingerprint() {
+    assertThat(
+            MarketActionFingerprint.request(
+                EVENT_ID, MARKET_ID, MarketStatus.SUSPENDED, "incident"))
+        .isEqualTo("45e1241da4bf5626acd9ea4c72e71f8aae4c0f19d5195ae6a5a87e6c52c8255f");
+  }
+
+  @Test
+  void actionStatusAndReasonAreFingerprintInputs() {
+    String baseline =
+        MarketActionFingerprint.request(EVENT_ID, MARKET_ID, MarketStatus.SUSPENDED, "incident");
+
+    assertThat(
+            MarketActionFingerprint.request(EVENT_ID, MARKET_ID, MarketStatus.CLOSED, "incident"))
+        .isNotEqualTo(baseline);
+    assertThat(
+            MarketActionFingerprint.request(
+                EVENT_ID, MARKET_ID, MarketStatus.SUSPENDED, "different"))
+        .isNotEqualTo(baseline);
+  }
+}


## `feat(commands): define atomic operator submissions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
new file mode 100644
index 0000000..6096884
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
@@ -0,0 +1,44 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.data.redis.core.script.DefaultRedisScript;
+import org.springframework.data.redis.core.script.RedisScript;
+
+/** Atomic Redis transition used when an operator action is accepted. */
+final class OperatorSubmissionScript {
+
+  static final RedisScript<String> INSTANCE =
+      new DefaultRedisScript<>(
+          """
+          local existing = redis.call('GET', KEYS[1])
+          if existing then
+            if string.sub(existing, 1, 64) == ARGV[1] then
+              return 'REPLAY|' .. string.sub(existing, 66)
+            end
+            return 'CONFLICT'
+          end
+          if redis.call('EXISTS', KEYS[2]) == 1 then
+            return 'CONFLICT'
+          end
+
+          local record = redis.call(
+            'XADD', KEYS[3], '*',
+            'fingerprint', ARGV[1],
+            'actionId', ARGV[2],
+            'eventId', ARGV[3],
+            'marketId', ARGV[4],
+            'requestedStatus', ARGV[5],
+            'previousStatus', 'OPEN',
+            'announcedStatus', ARGV[5],
+            'reason', ARGV[6],
+            'occurredAt', ARGV[7],
+            'sequence', '0',
+            'predecessor', '-1')
+          local metadata = ARGV[1] .. '|' .. ARGV[2] .. '|0|-1|' .. record
+          redis.call('SET', KEYS[1], metadata)
+          redis.call('SET', KEYS[2], KEYS[1])
+          return 'CREATED|' .. ARGV[2] .. '|0|-1|' .. record
+          """,
+          String.class);
+
+  private OperatorSubmissionScript() {}
+}


## `feat(commands): decode submission outcomes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java
index 50c8f7a..20526c2 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionSubmission.java
@@ -6,6 +6,30 @@ import java.util.UUID;
 public record OperatorActionSubmission(
     Outcome outcome, UUID actionId, long sequence, long predecessor, String recordId) {
 
+  private static final int FIELD_COUNT = 5;
+
+  static OperatorActionSubmission fromRedis(String result) {
+    if (result == null) {
+      throw new IllegalStateException("Operator action submission returned no result");
+    }
+    String[] fields = result.split("\\|", -1);
+    if (fields.length != FIELD_COUNT) {
+      throw new IllegalStateException("Malformed operator action submission result");
+    }
+    Outcome outcome =
+        switch (fields[0]) {
+          case "CREATED" -> Outcome.CREATED;
+          case "REPLAY" -> Outcome.REPLAYED;
+          default -> throw new IllegalStateException("Unknown operator action submission result");
+        };
+    return new OperatorActionSubmission(
+        outcome,
+        UUID.fromString(fields[1]),
+        Long.parseLong(fields[2]),
+        Long.parseLong(fields[FIELD_COUNT - 2]),
+        fields[FIELD_COUNT - 1]);
+  }
+
   public enum Outcome {
     CREATED,
     REPLAYED


## `feat(commands): persist idempotent submissions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/IdempotencyConflictException.java b/src/main/java/com/sportsbook/oddsfeed/delivery/IdempotencyConflictException.java
new file mode 100644
index 0000000..286b584
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/IdempotencyConflictException.java
@@ -0,0 +1,13 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.ResponseStatus;
+
+/** The supplied idempotency or action identity is already bound to another request. */
+@ResponseStatus(HttpStatus.CONFLICT)
+public class IdempotencyConflictException extends RuntimeException {
+
+  public IdempotencyConflictException() {
+    super("Idempotency identity is already bound to a different market action");
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
new file mode 100644
index 0000000..ab4b6c6
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -0,0 +1,84 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.config.OperatorDeliveryProperties;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.MarketId;
+import io.micrometer.core.instrument.MeterRegistry;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+/** Persists operator market actions in a durable Redis Stream. */
+@Component
+public class OperatorActionQueue {
+
+  private static final String IDEMPOTENCY_PREFIX = "oddsfeed:operator:idempotency:";
+  private static final String ACTION_PREFIX = "oddsfeed:operator:action:";
+  private static final int MAX_REASON_LENGTH = 256;
+
+  private final StringRedisTemplate redis;
+  private final OperatorDeliveryProperties properties;
+
+  public OperatorActionQueue(
+      StringRedisTemplate redis,
+      OperatorDeliveryProperties properties,
+      CacheProperties cacheProperties,
+      MeterRegistry meterRegistry) {
+    this.redis = redis;
+    this.properties = properties;
+  }
+
+  public OperatorActionSubmission submit(
+      IdempotencyKey idempotencyKey,
+      UUID actionId,
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus requestedStatus,
+      String reason,
+      Instant occurredAt) {
+    String normalizedReason = normalizeReason(reason);
+    String fingerprint =
+        MarketActionFingerprint.request(
+            eventId.value(), marketId.value(), requestedStatus, normalizedReason);
+    String result =
+        redis.execute(
+            OperatorSubmissionScript.INSTANCE,
+            List.of(
+                idempotencyRedisKey(idempotencyKey), actionKey(actionId), properties.streamKey()),
+            fingerprint,
+            actionId.toString(),
+            eventId.value().toString(),
+            marketId.value().toString(),
+            requestedStatus.name(),
+            normalizedReason,
+            Long.toString(occurredAt.toEpochMilli()));
+    if ("CONFLICT".equals(result)) {
+      throw new IdempotencyConflictException();
+    }
+    return OperatorActionSubmission.fromRedis(result);
+  }
+
+  static String idempotencyRedisKey(IdempotencyKey key) {
+    return IDEMPOTENCY_PREFIX + MarketActionFingerprint.idempotencyKey(key);
+  }
+
+  private static String actionKey(UUID actionId) {
+    return ACTION_PREFIX + actionId;
+  }
+
+  private static String normalizeReason(String reason) {
+    if (reason == null) {
+      throw new IllegalArgumentException("Operator action reason is required");
+    }
+    String normalized = reason.trim();
+    if (normalized.isEmpty() || normalized.length() > MAX_REASON_LENGTH) {
+      throw new IllegalArgumentException("Operator action reason must contain 1 to 256 characters");
+    }
+    return normalized;
+  }
+}


## `test(commands): verify idempotent submission replay`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
new file mode 100644
index 0000000..e834ce6
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.config.CacheProperties;
+import com.sportsbook.oddsfeed.config.OperatorDeliveryProperties;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.MarketId;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers(disabledWithoutDocker = true)
+class OperatorActionQueueTest {
+
+  private static final String STREAM = "test:operator-actions";
+  private static final Instant NOW = Instant.parse("2026-08-21T05:00:00Z");
+
+  @Container
+  private static final GenericContainer<?> REDIS =
+      new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
+
+  private LettuceConnectionFactory connectionFactory;
+  private StringRedisTemplate redis;
+
+  @BeforeEach
+  void setUp() {
+    connectionFactory = new LettuceConnectionFactory(REDIS.getHost(), REDIS.getFirstMappedPort());
+    connectionFactory.afterPropertiesSet();
+    redis = new StringRedisTemplate(connectionFactory);
+    redis.afterPropertiesSet();
+    redis.getConnectionFactory().getConnection().serverCommands().flushDb();
+  }
+
+  @AfterEach
+  void tearDown() {
+    connectionFactory.destroy();
+  }
+
+  @Test
+  void sameCanonicalRequestReplaysAcrossAReplacementActionId() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    UUID originalActionId = UUID.randomUUID();
+    IdempotencyKey key = IdempotencyKey.of("stable-key");
+
+    OperatorActionSubmission created =
+        queue.submit(
+            key, originalActionId, eventId, marketId, MarketStatus.SUSPENDED, " incident ", NOW);
+    OperatorActionSubmission replay =
+        queue.submit(
+            key, UUID.randomUUID(), eventId, marketId, MarketStatus.SUSPENDED, "incident", NOW);
+
+    assertThat(created.outcome()).isEqualTo(OperatorActionSubmission.Outcome.CREATED);
+    assertThat(replay.outcome()).isEqualTo(OperatorActionSubmission.Outcome.REPLAYED);
+    assertThat(replay.actionId()).isEqualTo(originalActionId);
+    assertThat(replay.recordId()).isEqualTo(created.recordId());
+    assertThat(redis.opsForStream().size(STREAM)).isOne();
+  }
+
+  private OperatorActionQueue queue() {
+    return new OperatorActionQueue(
+        redis,
+        new OperatorDeliveryProperties(STREAM, "group", "consumer", 20, Duration.ZERO, 10),
+        new CacheProperties(Duration.ofHours(24)),
+        new SimpleMeterRegistry());
+  }
+}


