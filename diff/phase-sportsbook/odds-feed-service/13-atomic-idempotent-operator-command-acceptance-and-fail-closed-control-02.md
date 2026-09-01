## `test(commands): verify submission conflicts`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index e834ce6..c449fc1 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.oddsfeed.delivery;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.oddsfeed.config.OperatorDeliveryProperties;
@@ -70,6 +71,46 @@ class OperatorActionQueueTest {
     assertThat(redis.opsForStream().size(STREAM)).isOne();
   }
 
+  @Test
+  void changedRequestOrReusedActionIdReturnsConflict() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    UUID actionId = UUID.randomUUID();
+    queue.submit(
+        IdempotencyKey.of("stable-key"),
+        actionId,
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "incident",
+        NOW);
+
+    assertThatThrownBy(
+            () ->
+                queue.submit(
+                    IdempotencyKey.of("stable-key"),
+                    UUID.randomUUID(),
+                    eventId,
+                    marketId,
+                    MarketStatus.SUSPENDED,
+                    "different",
+                    NOW))
+        .isInstanceOf(IdempotencyConflictException.class);
+    assertThatThrownBy(
+            () ->
+                queue.submit(
+                    IdempotencyKey.of("another-key"),
+                    actionId,
+                    eventId,
+                    marketId,
+                    MarketStatus.SUSPENDED,
+                    "incident",
+                    NOW))
+        .isInstanceOf(IdempotencyConflictException.class);
+    assertThat(redis.opsForStream().size(STREAM)).isOne();
+  }
+
   private OperatorActionQueue queue() {
     return new OperatorActionQueue(
         redis,


## `feat(commands): fail close restrictive submissions`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index ab4b6c6..816f9c9 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -1,5 +1,6 @@
 package com.sportsbook.oddsfeed.delivery;
 
+import com.sportsbook.oddsfeed.cache.CacheKeys;
 import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.oddsfeed.config.OperatorDeliveryProperties;
 import com.sportsbook.protocol.event.MarketStatus;
@@ -23,6 +24,7 @@ public class OperatorActionQueue {
 
   private final StringRedisTemplate redis;
   private final OperatorDeliveryProperties properties;
+  private final long marketTtlMillis;
 
   public OperatorActionQueue(
       StringRedisTemplate redis,
@@ -31,6 +33,7 @@ public class OperatorActionQueue {
       MeterRegistry meterRegistry) {
     this.redis = redis;
     this.properties = properties;
+    this.marketTtlMillis = cacheProperties.ttl().toMillis();
   }
 
   public OperatorActionSubmission submit(
@@ -49,14 +52,24 @@ public class OperatorActionQueue {
         redis.execute(
             OperatorSubmissionScript.INSTANCE,
             List.of(
-                idempotencyRedisKey(idempotencyKey), actionKey(actionId), properties.streamKey()),
+                idempotencyRedisKey(idempotencyKey),
+                actionKey(actionId),
+                CacheKeys.market(eventId, marketId),
+                CacheKeys.providerMarket(eventId, marketId),
+                CacheKeys.marketOverride(eventId, marketId),
+                CacheKeys.marketFeedHold(eventId, marketId),
+                CacheKeys.eventTerminal(eventId),
+                CacheKeys.marketTerminal(eventId, marketId),
+                CacheKeys.eventMarkets(eventId),
+                properties.streamKey()),
             fingerprint,
             actionId.toString(),
             eventId.value().toString(),
             marketId.value().toString(),
             requestedStatus.name(),
             normalizedReason,
-            Long.toString(occurredAt.toEpochMilli()));
+            Long.toString(occurredAt.toEpochMilli()),
+            Long.toString(marketTtlMillis));
     if ("CONFLICT".equals(result)) {
       throw new IdempotencyConflictException();
     }
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
index 6096884..c52b187 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
@@ -20,15 +20,42 @@ final class OperatorSubmissionScript {
             return 'CONFLICT'
           end
 
+          local requested = ARGV[5]
+          local provider = redis.call('GET', KEYS[4]) or 'OPEN'
+          local terminal = redis.call('EXISTS', KEYS[7]) == 1
+            or redis.call('EXISTS', KEYS[8]) == 1
+          if provider == 'CLOSED' then
+            redis.call('SET', KEYS[8], 'MARKET_CLOSED', 'NX')
+            terminal = true
+          end
+
+          local previous = redis.call('GET', KEYS[3]) or 'OPEN'
+          local announced = requested
+          if terminal then
+            previous = 'CLOSED'
+            announced = 'CLOSED'
+          elseif requested == 'OPEN' then
+            announced = provider
+          end
+
+          if requested ~= 'OPEN' then
+            redis.call('SET', KEYS[5], requested)
+            redis.call('PSETEX', KEYS[3], ARGV[8], announced)
+            redis.call('HSET', KEYS[9], ARGV[4], announced)
+          else
+            redis.call('HSETNX', KEYS[9], ARGV[4], previous)
+          end
+          redis.call('PEXPIRE', KEYS[9], ARGV[8])
+
           local record = redis.call(
-            'XADD', KEYS[3], '*',
+            'XADD', KEYS[10], '*',
             'fingerprint', ARGV[1],
             'actionId', ARGV[2],
             'eventId', ARGV[3],
             'marketId', ARGV[4],
-            'requestedStatus', ARGV[5],
-            'previousStatus', 'OPEN',
-            'announcedStatus', ARGV[5],
+            'requestedStatus', requested,
+            'previousStatus', previous,
+            'announcedStatus', announced,
             'reason', ARGV[6],
             'occurredAt', ARGV[7],
             'sequence', '0',


## `test(commands): verify immediate restrictive projection`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index c449fc1..149a5c5 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -3,6 +3,7 @@ package com.sportsbook.oddsfeed.delivery;
 import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
+import com.sportsbook.oddsfeed.cache.CacheKeys;
 import com.sportsbook.oddsfeed.config.CacheProperties;
 import com.sportsbook.oddsfeed.config.OperatorDeliveryProperties;
 import com.sportsbook.protocol.event.MarketStatus;
@@ -111,6 +112,55 @@ class OperatorActionQueueTest {
     assertThat(redis.opsForStream().size(STREAM)).isOne();
   }
 
+  @Test
+  void restrictiveSubmissionProjectsClosedStateBeforeReturn() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.market(eventId, marketId), MarketStatus.OPEN.name());
+
+    queue.submit(
+        IdempotencyKey.of("close-key"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.CLOSED,
+        " incident ",
+        NOW);
+
+    assertThat(redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.opsForHash().get(CacheKeys.eventMarkets(eventId), marketId.value().toString()))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.opsForStream().size(STREAM)).isOne();
+  }
+
+  @Test
+  void terminalRestrictionRemainsEffectivelyClosed() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.eventTerminal(eventId), "FINISHED");
+
+    queue.submit(
+        IdempotencyKey.of("terminal-key"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.SUSPENDED,
+        "late suspension",
+        NOW);
+
+    assertThat(redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.CLOSED.name());
+    assertThat(redis.opsForHash().get(CacheKeys.eventMarkets(eventId), marketId.value().toString()))
+        .isEqualTo(MarketStatus.CLOSED.name());
+  }
+
   private OperatorActionQueue queue() {
     return new OperatorActionQueue(
         redis,


## `feat(commands): define terminal reopen conflicts`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/TerminalMarketReopenException.java b/src/main/java/com/sportsbook/oddsfeed/delivery/TerminalMarketReopenException.java
new file mode 100644
index 0000000..9331fbf
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/TerminalMarketReopenException.java
@@ -0,0 +1,13 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.ResponseStatus;
+
+/** Raised when an operator attempts to reopen a terminal event or market. */
+@ResponseStatus(HttpStatus.CONFLICT)
+public class TerminalMarketReopenException extends RuntimeException {
+
+  public TerminalMarketReopenException() {
+    super("A terminal event or market cannot be reopened");
+  }
+}


## `feat(commands): defer operator reopens`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
index 816f9c9..55badf6 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueue.java
@@ -73,6 +73,9 @@ public class OperatorActionQueue {
     if ("CONFLICT".equals(result)) {
       throw new IdempotencyConflictException();
     }
+    if ("TERMINAL".equals(result)) {
+      throw new TerminalMarketReopenException();
+    }
     return OperatorActionSubmission.fromRedis(result);
   }
 
diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
index c52b187..0663815 100644
--- a/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/OperatorSubmissionScript.java
@@ -28,6 +28,9 @@ final class OperatorSubmissionScript {
             redis.call('SET', KEYS[8], 'MARKET_CLOSED', 'NX')
             terminal = true
           end
+          if requested == 'OPEN' and terminal then
+            return 'TERMINAL'
+          end
 
           local previous = redis.call('GET', KEYS[3]) or 'OPEN'
           local announced = requested
@@ -35,7 +38,11 @@ final class OperatorSubmissionScript {
             previous = 'CLOSED'
             announced = 'CLOSED'
           elseif requested == 'OPEN' then
-            announced = provider
+            if redis.call('EXISTS', KEYS[6]) == 1 then
+              announced = 'SUSPENDED'
+            else
+              announced = provider
+            end
           end
 
           if requested ~= 'OPEN' then


## `test(commands): verify deferred reopen safety`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
index 149a5c5..0e125fb 100644
--- a/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/OperatorActionQueueTest.java
@@ -17,6 +17,7 @@ import java.util.UUID;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.Range;
 import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
 import org.springframework.data.redis.core.StringRedisTemplate;
 import org.testcontainers.containers.GenericContainer;
@@ -161,6 +162,72 @@ class OperatorActionQueueTest {
         .isEqualTo(MarketStatus.CLOSED.name());
   }
 
+  @Test
+  void terminalEventOrMarketRejectsReopenWithoutEnqueueing() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.eventTerminal(eventId), "FINISHED");
+
+    assertThatThrownBy(
+            () ->
+                queue.submit(
+                    IdempotencyKey.of("event-terminal"),
+                    UUID.randomUUID(),
+                    eventId,
+                    marketId,
+                    MarketStatus.OPEN,
+                    "review",
+                    NOW))
+        .isInstanceOf(TerminalMarketReopenException.class);
+
+    redis.delete(CacheKeys.eventTerminal(eventId));
+    redis.opsForValue().set(CacheKeys.marketTerminal(eventId, marketId), "MARKET_CLOSED");
+    assertThatThrownBy(
+            () ->
+                queue.submit(
+                    IdempotencyKey.of("market-terminal"),
+                    UUID.randomUUID(),
+                    eventId,
+                    marketId,
+                    MarketStatus.OPEN,
+                    "review",
+                    NOW))
+        .isInstanceOf(TerminalMarketReopenException.class);
+    assertThat(redis.hasKey(STREAM)).isFalse();
+  }
+
+  @Test
+  void reopenRetainsOverrideUntilAcknowledgedDelivery() {
+    OperatorActionQueue queue = queue();
+    EventId eventId = new EventId(UUID.randomUUID());
+    MarketId marketId = new MarketId(UUID.randomUUID());
+    redis.opsForValue().set(CacheKeys.providerMarket(eventId, marketId), MarketStatus.OPEN.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketOverride(eventId, marketId), MarketStatus.SUSPENDED.name());
+    redis.opsForValue().set(CacheKeys.market(eventId, marketId), MarketStatus.SUSPENDED.name());
+    redis
+        .opsForValue()
+        .set(CacheKeys.marketFeedHold(eventId, marketId), Long.toString(NOW.toEpochMilli()));
+
+    queue.submit(
+        IdempotencyKey.of("reopen-key"),
+        UUID.randomUUID(),
+        eventId,
+        marketId,
+        MarketStatus.OPEN,
+        "review complete",
+        NOW);
+
+    assertThat(redis.opsForValue().get(CacheKeys.marketOverride(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+    assertThat(redis.opsForValue().get(CacheKeys.market(eventId, marketId)))
+        .isEqualTo(MarketStatus.SUSPENDED.name());
+    var record = redis.opsForStream().range(STREAM, Range.unbounded()).get(0);
+    assertThat(record.getValue()).containsEntry("announcedStatus", MarketStatus.SUSPENDED.name());
+  }
+
   private OperatorActionQueue queue() {
     return new OperatorActionQueue(
         redis,


## `feat(api): accept durable market controls`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/MarketAdminController.java b/src/main/java/com/sportsbook/oddsfeed/api/MarketAdminController.java
new file mode 100644
index 0000000..b59f509
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/MarketAdminController.java
@@ -0,0 +1,82 @@
+package com.sportsbook.oddsfeed.api;
+
+import com.sportsbook.oddsfeed.delivery.OperatorActionQueue;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.MarketId;
+import jakarta.validation.Valid;
+import jakarta.validation.constraints.NotBlank;
+import jakarta.validation.constraints.Size;
+import java.time.Clock;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestHeader;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.ResponseStatus;
+import org.springframework.web.bind.annotation.RestController;
+import org.springframework.web.server.ResponseStatusException;
+
+/** Durably accepts authenticated operator market controls. */
+@RestController
+@RequestMapping("/internal/v1/events/{eventId}/markets/{marketId}")
+public class MarketAdminController {
+
+  public static final String IDEMPOTENCY_HEADER = "Idempotency-Key";
+  public static final String ACTION_ID_HEADER = "X-Admin-Action-Id";
+
+  private final OperatorActionQueue queue;
+  private final Clock clock;
+
+  public MarketAdminController(OperatorActionQueue queue, Clock clock) {
+    this.queue = queue;
+    this.clock = clock;
+  }
+
+  @PostMapping("/{action:suspend|close|reopen}")
+  @ResponseStatus(HttpStatus.ACCEPTED)
+  public void transition(
+      @PathVariable("eventId") UUID eventUuid,
+      @PathVariable("marketId") UUID marketUuid,
+      @PathVariable("action") String action,
+      @RequestHeader(IDEMPOTENCY_HEADER) String rawIdempotencyKey,
+      @RequestHeader(ACTION_ID_HEADER) UUID actionId,
+      @Valid @RequestBody MarketStatusChangeRequest body) {
+    IdempotencyKey idempotencyKey;
+    try {
+      idempotencyKey = IdempotencyKey.of(rawIdempotencyKey);
+    } catch (IllegalArgumentException exception) {
+      throw new ResponseStatusException(
+          HttpStatus.BAD_REQUEST, "Invalid Idempotency-Key", exception);
+    }
+    queue.submit(
+        idempotencyKey,
+        actionId,
+        new EventId(eventUuid),
+        new MarketId(marketUuid),
+        requestedStatus(action),
+        body.reason(),
+        clock.instant());
+  }
+
+  private static MarketStatus requestedStatus(String action) {
+    return switch (action) {
+      case "suspend" -> MarketStatus.SUSPENDED;
+      case "close" -> MarketStatus.CLOSED;
+      case "reopen" -> MarketStatus.OPEN;
+      default -> throw new IllegalArgumentException("Unsupported operator action");
+    };
+  }
+
+  public record MarketStatusChangeRequest(@NotBlank @Size(max = 256) String reason) {
+
+    public MarketStatusChangeRequest {
+      if (reason != null) {
+        reason = reason.trim();
+      }
+    }
+  }
+}


## `test(api): verify authenticated market controls`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java
new file mode 100644
index 0000000..738b0d3
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java
@@ -0,0 +1,85 @@
+package com.sportsbook.oddsfeed.api;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import com.sportsbook.oddsfeed.delivery.OperatorActionQueue;
+import com.sportsbook.oddsfeed.security.InternalApiKeyAuthenticationFilter;
+import com.sportsbook.oddsfeed.security.SecurityConfig;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import java.time.Clock;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.annotation.Import;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.ResultActions;
+
+@WebMvcTest(controllers = MarketAdminController.class)
+@Import(SecurityConfig.class)
+@EnableConfigurationProperties(InternalSecurityProperties.class)
+class MarketAdminControllerTest {
+
+  private static final String API_KEY = "test-internal-api-key-0123456789abcdef";
+  private static final Instant NOW = Instant.parse("2026-05-28T10:00:00Z");
+
+  @Autowired private MockMvc mockMvc;
+  @MockBean private OperatorActionQueue queue;
+  @MockBean private Clock clock;
+
+  @BeforeEach
+  void setUpClock() {
+    when(clock.instant()).thenReturn(NOW);
+  }
+
+  @Test
+  void authenticatedControlsReturnOnlyAfterDurableSubmission() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+    UUID suspendId = UUID.randomUUID();
+    UUID closeId = UUID.randomUUID();
+    UUID reopenId = UUID.randomUUID();
+
+    perform(eventId, marketId, suspendId, "suspend").andExpect(status().isAccepted());
+    perform(eventId, marketId, closeId, "close").andExpect(status().isAccepted());
+    perform(eventId, marketId, reopenId, "reopen").andExpect(status().isAccepted());
+
+    verify(queue)
+        .submit(
+            any(),
+            eq(suspendId),
+            eq(new EventId(eventId)),
+            eq(new MarketId(marketId)),
+            eq(MarketStatus.SUSPENDED),
+            eq("incident"),
+            eq(NOW));
+    verify(queue).submit(any(), eq(closeId), any(), any(), eq(MarketStatus.CLOSED), any(), any());
+    verify(queue).submit(any(), eq(reopenId), any(), any(), eq(MarketStatus.OPEN), any(), any());
+  }
+
+  private ResultActions perform(UUID eventId, UUID marketId, UUID actionId, String action)
+      throws Exception {
+    String path = "/internal/v1/events/" + eventId + "/markets/" + marketId + "/" + action;
+    return mockMvc.perform(
+        post(path)
+            .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "admin-api")
+            .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, API_KEY)
+            .header(MarketAdminController.IDEMPOTENCY_HEADER, "operator-request")
+            .header(MarketAdminController.ACTION_ID_HEADER, actionId)
+            .contentType(MediaType.APPLICATION_JSON)
+            .content("{\"reason\":\"incident\"}"));
+  }
+}


