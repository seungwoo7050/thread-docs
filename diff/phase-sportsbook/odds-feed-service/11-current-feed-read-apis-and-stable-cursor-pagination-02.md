## `test(api): verify cursor navigation`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
index d0bf947..d91b1b5 100644
--- a/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/api/EventReadControllerTest.java
@@ -50,6 +50,53 @@ class EventReadControllerTest {
     assertThat(EventReadController.clampSize(101)).isEqualTo(100);
   }
 
+  @Test
+  void cursorNavigatesToTheFollowingPage() throws Exception {
+    EventSummary first = summary(1, "2026-06-01T18:00:00Z");
+    EventSummary second = summary(2, "2026-06-02T18:00:00Z");
+    EventSummary third = summary(3, "2026-06-03T18:00:00Z");
+    when(catalog.orderedByKickoff()).thenReturn(List.of(first, second, third));
+
+    String body =
+        mockMvc
+            .perform(get("/api/v1/events").param("size", "2"))
+            .andExpect(status().isOk())
+            .andExpect(jsonPath("$.nextCursor").exists())
+            .andReturn()
+            .getResponse()
+            .getContentAsString();
+
+    mockMvc
+        .perform(
+            get("/api/v1/events")
+                .param("size", "2")
+                .param("cursor", extractJsonValue(body, "nextCursor")))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.items.length()").value(1))
+        .andExpect(jsonPath("$.items[0].eventId").value(third.eventId().value().toString()))
+        .andExpect(jsonPath("$.nextCursor").doesNotExist());
+  }
+
+  @Test
+  void rejectsMalformedCursor() throws Exception {
+    when(catalog.orderedByKickoff()).thenReturn(List.of());
+
+    mockMvc
+        .perform(get("/api/v1/events").param("cursor", "not-a-cursor"))
+        .andExpect(status().isBadRequest());
+  }
+
+  @Test
+  void cursorEncodingRoundTrips() {
+    EventSummary summary = summary(11, "2026-06-01T18:00:00Z");
+
+    EventReadController.Cursor cursor =
+        EventReadController.decodeCursor(EventReadController.encodeCursor(summary));
+
+    assertThat(cursor.kickoff()).isEqualTo(summary.scheduledStartAt());
+    assertThat(cursor.eventId()).isEqualTo(summary.eventId().value());
+  }
+
   @Test
   void returnsCurrentEvent() throws Exception {
     EventSummary summary = summary(7, "2026-06-01T18:00:00Z");
@@ -79,4 +126,10 @@ class EventReadControllerTest {
         Instant.parse(kickoff),
         EventLifecycleStatus.SCHEDULED);
   }
+
+  private static String extractJsonValue(String body, String key) {
+    int field = body.indexOf("\"" + key + "\":");
+    int start = body.indexOf('"', field + key.length() + 3) + 1;
+    return body.substring(start, body.indexOf('"', start));
+  }
 }
