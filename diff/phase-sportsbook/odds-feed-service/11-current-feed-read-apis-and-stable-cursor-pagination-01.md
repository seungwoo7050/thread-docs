# 현재 피드 조회 API와 안정적 커서 페이지네이션

## `feat(catalog): index current event summaries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java b/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java
new file mode 100644
index 0000000..d6e65d9
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/EventCatalog.java
@@ -0,0 +1,40 @@
+package com.sportsbook.oddsfeed.api;
+
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.value.EventId;
+import java.util.Comparator;
+import java.util.List;
+import java.util.Map;
+import java.util.Optional;
+import java.util.concurrent.ConcurrentHashMap;
+import org.springframework.stereotype.Component;
+
+@Component
+public class EventCatalog {
+
+  private final Map<EventId, EventSummary> events = new ConcurrentHashMap<>();
+
+  public void put(EventSummary summary) {
+    events.put(summary.eventId(), summary);
+  }
+
+  public boolean putIfAbsent(EventSummary summary) {
+    return events.putIfAbsent(summary.eventId(), summary) == null;
+  }
+
+  public Optional<EventSummary> get(EventId eventId) {
+    return Optional.ofNullable(events.get(eventId));
+  }
+
+  public List<EventSummary> orderedByKickoff() {
+    return events.values().stream()
+        .sorted(
+            Comparator.comparing(EventSummary::scheduledStartAt)
+                .thenComparing(event -> event.eventId().value()))
+        .toList();
+  }
+
+  public int size() {
+    return events.size();
+  }
+}


## `test(catalog): verify deterministic catalog order`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/EventCatalogTest.java b/src/test/java/com/sportsbook/oddsfeed/api/EventCatalogTest.java
new file mode 100644
index 0000000..0dffb72
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/api/EventCatalogTest.java
@@ -0,0 +1,43 @@
+package com.sportsbook.oddsfeed.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class EventCatalogTest {
+
+  @Test
+  void ordersSnapshotsByKickoffAndIdentity() {
+    EventCatalog catalog = new EventCatalog();
+    Instant kickoff = Instant.parse("2026-06-01T18:00:00Z");
+    EventSummary second = event("00000000-0000-0000-0000-000000000002", kickoff);
+    EventSummary first = event("00000000-0000-0000-0000-000000000001", kickoff);
+    EventSummary earlier = event("00000000-0000-0000-0000-000000000003", kickoff.minusSeconds(60));
+
+    assertThat(catalog.putIfAbsent(second)).isTrue();
+    assertThat(catalog.putIfAbsent(second)).isFalse();
+    catalog.put(first);
+    catalog.put(earlier);
+
+    assertThat(catalog.orderedByKickoff()).containsExactly(earlier, first, second);
+    assertThat(catalog.get(first.eventId())).contains(first);
+    assertThat(catalog.size()).isEqualTo(3);
+  }
+
+  private static EventSummary event(String id, Instant kickoff) {
+    return new EventSummary(
+        new EventId(UUID.fromString(id)),
+        Sport.FOOTBALL,
+        "Premier League",
+        "Home",
+        "Away",
+        kickoff,
+        EventLifecycleStatus.SCHEDULED);
+  }
+}


## `feat(api): read current selection odds`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/OddsReadController.java b/src/main/java/com/sportsbook/oddsfeed/api/OddsReadController.java
new file mode 100644
index 0000000..f98a4b9
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/OddsReadController.java
@@ -0,0 +1,41 @@
+package com.sportsbook.oddsfeed.api;
+
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.math.BigDecimal;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+import org.springframework.web.server.ResponseStatusException;
+
+/** Serves the current cached price for a selection. */
+@RestController
+@RequestMapping("/api/v1/odds")
+public class OddsReadController {
+
+  private final RedisOddsCache cache;
+
+  public OddsReadController(RedisOddsCache cache) {
+    this.cache = cache;
+  }
+
+  @GetMapping("/{eventId}/{marketId}/{selectionId}")
+  public OddsResponse getOdds(
+      @PathVariable("eventId") UUID eventId,
+      @PathVariable("marketId") UUID marketId,
+      @PathVariable("selectionId") UUID selectionId) {
+    Odds odds =
+        cache
+            .getOdds(new EventId(eventId), new MarketId(marketId), new SelectionId(selectionId))
+            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Odds not found"));
+    return new OddsResponse(eventId, marketId, selectionId, odds.decimal());
+  }
+
+  public record OddsResponse(UUID eventId, UUID marketId, UUID selectionId, BigDecimal odds) {}
+}


## `test(api): verify odds read responses`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java
new file mode 100644
index 0000000..c606242
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/api/OddsReadControllerTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.oddsfeed.api;
+
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.oddsfeed.cache.RedisOddsCache;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.test.web.servlet.MockMvc;
+
+@WebMvcTest(
+    controllers = OddsReadController.class,
+    excludeAutoConfiguration = SecurityAutoConfiguration.class)
+class OddsReadControllerTest {
+
+  @Autowired private MockMvc mockMvc;
+  @MockBean private RedisOddsCache cache;
+
+  @Test
+  void returnsCurrentOdds() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    when(cache.getOdds(new EventId(eventId), new MarketId(marketId), new SelectionId(selectionId)))
+        .thenReturn(Optional.of(Odds.ofDecimal("1.85")));
+
+    mockMvc
+        .perform(get("/api/v1/odds/{e}/{m}/{s}", eventId, marketId, selectionId))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.eventId").value(eventId.toString()))
+        .andExpect(jsonPath("$.marketId").value(marketId.toString()))
+        .andExpect(jsonPath("$.selectionId").value(selectionId.toString()))
+        .andExpect(jsonPath("$.odds").value(1.85));
+  }
+
+  @Test
+  void returnsNotFoundForMissingOdds() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+    UUID selectionId = UUID.randomUUID();
+    when(cache.getOdds(new EventId(eventId), new MarketId(marketId), new SelectionId(selectionId)))
+        .thenReturn(Optional.empty());
+
+    mockMvc
+        .perform(get("/api/v1/odds/{e}/{m}/{s}", eventId, marketId, selectionId))
+        .andExpect(status().isNotFound());
+  }
+}


## `feat(api): read current events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
new file mode 100644
index 0000000..2dfaac8
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
@@ -0,0 +1,32 @@
+package com.sportsbook.oddsfeed.api;
+
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.protocol.value.EventId;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+import org.springframework.web.server.ResponseStatusException;
+
+/** Serves current event projections. */
+@RestController
+@RequestMapping("/api/v1/events")
+public class EventReadController {
+
+  private final EventCatalog catalog;
+
+  public EventReadController(EventCatalog catalog) {
+    this.catalog = catalog;
+  }
+
+  @GetMapping("/{eventId}")
+  public ResponseEntity<EventSummary> get(@PathVariable("eventId") UUID eventId) {
+    return catalog
+        .get(new EventId(eventId))
+        .map(ResponseEntity::ok)
+        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Event not found"));
+  }
+}


## `test(api): verify event lookup responses`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
new file mode 100644
index 0000000..f775480
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.oddsfeed.api;
+
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration;
+import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.test.web.servlet.MockMvc;
+
+@WebMvcTest(
+    controllers = EventReadController.class,
+    excludeAutoConfiguration = SecurityAutoConfiguration.class)
+class EventReadControllerTest {
+
+  @Autowired private MockMvc mockMvc;
+  @MockBean private EventCatalog catalog;
+
+  @Test
+  void returnsCurrentEvent() throws Exception {
+    EventSummary summary = summary(7, "2026-06-01T18:00:00Z");
+    when(catalog.get(summary.eventId())).thenReturn(Optional.of(summary));
+
+    mockMvc
+        .perform(get("/api/v1/events/{id}", summary.eventId().value()))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.eventId").value(summary.eventId().value().toString()));
+  }
+
+  @Test
+  void returnsNotFoundForMissingEvent() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    when(catalog.get(new EventId(eventId))).thenReturn(Optional.empty());
+
+    mockMvc.perform(get("/api/v1/events/{id}", eventId)).andExpect(status().isNotFound());
+  }
+
+  private static EventSummary summary(int seed, String kickoff) {
+    return new EventSummary(
+        new EventId(new UUID(0L, seed)),
+        Sport.FOOTBALL,
+        "Premier League",
+        "Home" + seed,
+        "Away" + seed,
+        Instant.parse(kickoff),
+        EventLifecycleStatus.SCHEDULED);
+  }
+}


## `feat(api): paginate event summaries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/CursorPage.java b/src/main/java/com/sportsbook/oddsfeed/api/CursorPage.java
new file mode 100644
index 0000000..9bafb8f
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/api/CursorPage.java
@@ -0,0 +1,6 @@
+package com.sportsbook.oddsfeed.api;
+
+import java.util.List;
+
+/** A page with an optional cursor for the following page. */
+public record CursorPage<T>(List<T> items, String nextCursor) {}
diff --git a/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
index 2dfaac8..88c1080 100644
--- a/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
+++ b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
@@ -2,12 +2,14 @@ package com.sportsbook.oddsfeed.api;
 
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.protocol.value.EventId;
+import java.util.List;
 import java.util.UUID;
 import org.springframework.http.HttpStatus;
 import org.springframework.http.ResponseEntity;
 import org.springframework.web.bind.annotation.GetMapping;
 import org.springframework.web.bind.annotation.PathVariable;
 import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
 import org.springframework.web.bind.annotation.RestController;
 import org.springframework.web.server.ResponseStatusException;
 
@@ -16,12 +18,24 @@ import org.springframework.web.server.ResponseStatusException;
 @RequestMapping("/api/v1/events")
 public class EventReadController {
 
+  private static final int DEFAULT_PAGE_SIZE = 20;
+  private static final int MAX_PAGE_SIZE = 100;
+
   private final EventCatalog catalog;
 
   public EventReadController(EventCatalog catalog) {
     this.catalog = catalog;
   }
 
+  @GetMapping
+  public CursorPage<EventSummary> list(
+      @RequestParam(value = "size", defaultValue = "20") int requestedSize) {
+    int size = clampSize(requestedSize);
+    List<EventSummary> events = catalog.orderedByKickoff();
+    int endIndex = Math.min(size, events.size());
+    return new CursorPage<>(events.subList(0, endIndex), null);
+  }
+
   @GetMapping("/{eventId}")
   public ResponseEntity<EventSummary> get(@PathVariable("eventId") UUID eventId) {
     return catalog
@@ -29,4 +43,11 @@ public class EventReadController {
         .map(ResponseEntity::ok)
         .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Event not found"));
   }
+
+  static int clampSize(int requestedSize) {
+    if (requestedSize <= 0) {
+      return DEFAULT_PAGE_SIZE;
+    }
+    return Math.min(requestedSize, MAX_PAGE_SIZE);
+  }
 }


## `test(api): verify event page boundaries`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
index f775480..d0bf947 100644
--- a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
@@ -1,5 +1,6 @@
 package com.sportsbook.oddsfeed.api;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.mockito.Mockito.when;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
@@ -10,6 +11,7 @@ import com.sportsbook.oddsfeed.provider.Sport;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.EventId;
 import java.time.Instant;
+import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
@@ -27,6 +29,27 @@ class EventReadControllerTest {
   @Autowired private MockMvc mockMvc;
   @MockBean private EventCatalog catalog;
 
+  @Test
+  void listsEventsWithinTheRequestedPageSize() throws Exception {
+    EventSummary first = summary(1, "2026-06-01T18:00:00Z");
+    EventSummary second = summary(2, "2026-06-02T18:00:00Z");
+    when(catalog.orderedByKickoff()).thenReturn(List.of(first, second));
+
+    mockMvc
+        .perform(get("/api/v1/events").param("size", "1"))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.items.length()").value(1))
+        .andExpect(jsonPath("$.items[0].eventId").value(first.eventId().value().toString()));
+  }
+
+  @Test
+  void pageSizeUsesDefaultAndMaximumBoundaries() {
+    assertThat(EventReadController.clampSize(0)).isEqualTo(20);
+    assertThat(EventReadController.clampSize(-1)).isEqualTo(20);
+    assertThat(EventReadController.clampSize(40)).isEqualTo(40);
+    assertThat(EventReadController.clampSize(101)).isEqualTo(100);
+  }
+
   @Test
   void returnsCurrentEvent() throws Exception {
     EventSummary summary = summary(7, "2026-06-01T18:00:00Z");


## `feat(api): encode stable event cursors`

diff --git a/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
index 88c1080..7aab529 100644
--- a/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
+++ b/src/main/java/com/sportsbook/oddsfeed/api/EventReadController.java
@@ -2,6 +2,9 @@ package com.sportsbook.oddsfeed.api;
 
 import com.sportsbook.oddsfeed.provider.EventSummary;
 import com.sportsbook.protocol.value.EventId;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.Base64;
 import java.util.List;
 import java.util.UUID;
 import org.springframework.http.HttpStatus;
@@ -29,11 +32,15 @@ public class EventReadController {
 
   @GetMapping
   public CursorPage<EventSummary> list(
+      @RequestParam(value = "cursor", required = false) String cursor,
       @RequestParam(value = "size", defaultValue = "20") int requestedSize) {
     int size = clampSize(requestedSize);
     List<EventSummary> events = catalog.orderedByKickoff();
-    int endIndex = Math.min(size, events.size());
-    return new CursorPage<>(events.subList(0, endIndex), null);
+    int startIndex = cursor == null ? 0 : indexAfter(events, decodeCursor(cursor));
+    int endIndex = Math.min(startIndex + size, events.size());
+    List<EventSummary> page = events.subList(startIndex, endIndex);
+    String nextCursor = endIndex < events.size() ? encodeCursor(page.get(page.size() - 1)) : null;
+    return new CursorPage<>(page, nextCursor);
   }
 
   @GetMapping("/{eventId}")
@@ -50,4 +57,40 @@ public class EventReadController {
     }
     return Math.min(requestedSize, MAX_PAGE_SIZE);
   }
+
+  static String encodeCursor(EventSummary summary) {
+    String value = summary.scheduledStartAt() + "|" + summary.eventId().value();
+    return Base64.getUrlEncoder()
+        .withoutPadding()
+        .encodeToString(value.getBytes(StandardCharsets.UTF_8));
+  }
+
+  static Cursor decodeCursor(String encoded) {
+    try {
+      String value = new String(Base64.getUrlDecoder().decode(encoded), StandardCharsets.UTF_8);
+      int separator = value.indexOf('|');
+      if (separator < 0) {
+        throw new IllegalArgumentException("Missing cursor separator");
+      }
+      return new Cursor(
+          Instant.parse(value.substring(0, separator)),
+          UUID.fromString(value.substring(separator + 1)));
+    } catch (IllegalArgumentException | java.time.format.DateTimeParseException exception) {
+      throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Invalid cursor", exception);
+    }
+  }
+
+  private static int indexAfter(List<EventSummary> events, Cursor cursor) {
+    for (int index = 0; index < events.size(); index++) {
+      EventSummary event = events.get(index);
+      int kickoffOrder = event.scheduledStartAt().compareTo(cursor.kickoff());
+      if (kickoffOrder > 0
+          || kickoffOrder == 0 && event.eventId().value().compareTo(cursor.eventId()) > 0) {
+        return index;
+      }
+    }
+    return events.size();
+  }
+
+  record Cursor(Instant kickoff, UUID eventId) {}
 }


