# E07 CheckRun 이력, Cursor와 URL 상태

## `test(history): freeze E07 pagination counterexample`

diff --git a/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java b/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java
new file mode 100644
index 0000000..4966b9b
--- /dev/null
+++ b/backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java
@@ -0,0 +1,99 @@
+package dev.evolution.monitor;
+
+import static org.junit.jupiter.api.Assertions.*;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterAll;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.ResponseEntity;
+import org.springframework.test.annotation.DirtiesContext;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
+@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
+class HistoryPaginationTest {
+    private static final String PASSWORD = SessionClient.password();
+    @Autowired TestRestTemplate api;
+    @Autowired ObjectMapper json;
+
+    @DynamicPropertySource
+    static void database(DynamicPropertyRegistry properties) {
+        TestDatabase.configure(properties, "e07_history");
+    }
+
+    @BeforeAll
+    static void accounts(@Autowired UserAccounts accounts) {
+        accounts.bootstrap(PASSWORD, SessionClient.password());
+    }
+
+    @AfterAll
+    static void cleanup() { TestDatabase.drop("e07_history"); }
+
+    @Test
+    void fixedSevenTiesHaveBoundedStableContinuation() throws Exception {
+        var alice = new SessionClient(api);
+        assertEquals(200, alice.login("alice-e04", PASSWORD).getStatusCode().value());
+        String monitor = data(alice.mutate("/api/monitors", HttpMethod.POST, Map.of(
+                "name", "History fixture", "url", "http://127.0.0.1:4321/ok", "interval", 60, "enabled", true)), 201)
+                .at("/monitor/id").textValue();
+        seed(monitor, 1, 7);
+        var first = alice.get(path(monitor) + "?limit=3");
+        JsonNode rows = data(first, 200);
+        if (Boolean.getBoolean("e07.baseline")) {
+            Files.writeString(Path.of("target/e07-baseline.json"), json.writerWithDefaultPrettyPrinter()
+                    .writeValueAsString(Map.of("start", "d2abc8a44e38a05660e49ce664de6f38f8831edd",
+                            "request", "GET history?limit=3", "historyRequests", 1, "status", 200,
+                            "rowCount", rows.size(), "ids", ids(rows),
+                            "nextCursorPresent", first.getHeaders().containsKey("X-Next-Cursor"))) + "\n");
+        }
+        assertEquals(3, rows.size(), "The explicit first page must be bounded to three rows");
+        assertEquals(List.of(id(7), id(6), id(5)), ids(rows));
+        assertNotNull(first.getHeaders().getFirst("X-Next-Cursor"));
+    }
+
+    private static void seed(String monitor, int from, int through) throws Exception {
+        try (var connection = TestDatabase.connect(); var insert = connection.prepareStatement("""
+                INSERT INTO e07_history.check_runs
+                (id, monitor_id, trigger_kind, state, http_status, latency_ms, failure_reason, started_at, finished_at)
+                VALUES (?, ?, 'MANUAL', ?, ?, 1, ?, ?::timestamptz, ?::timestamptz)
+                """)) {
+            for (int number = from; number <= through; number++) {
+                boolean success = number % 2 != 0 || number == 8;
+                String time = number == 8 ? "2026-08-28T00:00:01.000Z" : "2026-08-28T00:00:00.000Z";
+                insert.setObject(1, UUID.fromString(id(number)));
+                insert.setObject(2, UUID.fromString(monitor));
+                insert.setString(3, success ? "SUCCEEDED" : "FAILED");
+                insert.setInt(4, success ? 200 : 503);
+                insert.setString(5, success ? null : "HTTP_STATUS");
+                insert.setString(6, time);
+                insert.setString(7, time);
+                assertEquals(1, insert.executeUpdate());
+            }
+        }
+    }
+
+    private static String id(int number) { return "00000000-0000-4000-8000-%012d".formatted(number); }
+    private static String path(String monitor) { return "/api/monitors/" + monitor + "/checks"; }
+    private static List<String> ids(JsonNode rows) {
+        var ids = new java.util.ArrayList<String>();
+        rows.forEach(row -> ids.add(row.get("id").textValue()));
+        return ids;
+    }
+    private static JsonNode data(ResponseEntity<JsonNode> response, int status) {
+        assertEquals(status, response.getStatusCode().value());
+        assertTrue(response.getBody().has("data"));
+        return response.getBody().get("data");
+    }
+}
diff --git a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
index 75d9e29..093f3e8 100644
--- a/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
+++ b/backend/src/test/java/dev/evolution/monitor/TestDatabase.java
@@ -12,7 +12,7 @@ final class TestDatabase {
     private static final Set<String> SCHEMAS = Set.of("e04_functional", "e04_migrations", "e04_mapping",
             "e04_missing_column", "e04_extra_required", "e04_session", "e04_users",
             "e04_missing_user_column", "e04_extra_user_required", "e05_ownership", "e05_fresh",
-            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk");
+            "e05_upgrade", "e05_missing_owner", "e05_nullable_owner", "e05_missing_owner_fk", "e07_history");
 
     static Connection connect() throws SQLException {
         return DriverManager.getConnection(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://127.0.0.1:15432/monitor"),
diff --git a/evidence/E07/baseline.json b/evidence/E07/baseline.json
new file mode 100644
index 0000000..6d5b227
--- /dev/null
+++ b/evidence/E07/baseline.json
@@ -0,0 +1,9 @@
+{
+  "ids" : [ "00000000-0000-4000-8000-000000000007", "00000000-0000-4000-8000-000000000006", "00000000-0000-4000-8000-000000000005", "00000000-0000-4000-8000-000000000004", "00000000-0000-4000-8000-000000000003", "00000000-0000-4000-8000-000000000002", "00000000-0000-4000-8000-000000000001" ],
+  "nextCursorPresent" : false,
+  "request" : "GET history?limit=3",
+  "historyRequests" : 1,
+  "status" : 200,
+  "start" : "d2abc8a44e38a05660e49ce664de6f38f8831edd",
+  "rowCount" : 7
+}
diff --git a/evidence/E07/fixtures.md b/evidence/E07/fixtures.md
new file mode 100644
index 0000000..7f39b45
--- /dev/null
+++ b/evidence/E07/fixtures.md
@@ -0,0 +1,79 @@
+# E07 frozen history pagination and URL scenario
+
+Frozen before baseline execution or product edits.
+START: `d2abc8a44e38a05660e49ce664de6f38f8831edd`.
+Spec: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Branch: `track/industry-spring`; attempt1.
+
+## Fixed API contract
+
+- Existing `GET /api/monitors/{id}/checks` retains `{data: CheckRun[]}`.
+- `limit` defaults to20; accepted integer sizes are1–100. Invalid/nonpositive/
+  noninteger/over-maximum size returns400 / INVALID_INPUT.
+- Optional `state` is exactly `SUCCEEDED` or `FAILED`; unknown/empty values are400.
+- Order is `(finished_at DESC, id DESC)`. SQL reads at most `limit + 1` rows.
+- Optional `X-Next-Cursor` is present only when another page exists. It encodes a
+  versioned continuation tuple and binds Monitor, filter and page size. Malformed
+  cursors or condition mismatch return400 / INVALID_INPUT. Cursor is pagination
+  state, never authentication; every query retains the verified owner predicate.
+- Continue strictly below the last returned `(finished_at, id)` tuple. No offset
+  pagination and no requirement to include a later, newer insertion in an already
+  started continuation. Owner/foreign/nonexistent and direct CheckRun semantics
+  remain unchanged.
+
+## Fixed rows and backend sequence
+
+Use one Alice-owned `History fixture` Monitor, URL
+`http://127.0.0.1:4321/ok`, interval60, enabled true. Seven immutable CheckRuns have
+UUIDs `00000000-0000-4000-8000-000000000001` through
+`00000000-0000-4000-8000-000000000007`. Both timestamps are
+`2026-08-28T00:00:00.000Z` for every row. Odd IDs are SUCCEEDED/HTTP200/null reason;
+even IDs are FAILED/HTTP503/HTTP_STATUS. All are MANUAL with latency1ms.
+
+1. At unchanged START, issue exactly one bounded-history GET with `limit=3`.
+   Record actual array length and cursor presence; required first page is IDs7,6,5
+   with a next cursor. Stop after the decisive unbounded/missing-cursor result.
+2. Post-change: first page3, all continuations3; collect exactly7 unique originals
+   in descending ID order. FAILED filter returns exactly6,4,2.
+3. Default20 and maximum100 are accepted. Sizes0,-1,1.5,101 and nonnumeric text,
+   invalid state, malformed/version-invalid cursor and changed Monitor/filter/size
+   conditions return400. Foreign/nonexistent Monitor reads remain404.
+4. Reset to the same seven rows. Read first page3, then insert exactly one eighth
+   SUCCEEDED row ending000000000008 at `2026-08-28T00:00:01.000Z` (both timestamps).
+   Continue the original cursor: IDs4,3,2 then1, with no8, gaps or duplicates.
+5. Inspect generated owner/filter/keyset/order/SQL-limit clauses and actual index
+   definitions. Add only basic history ordering/filter indexes through a new
+   migration; preserve every earlier migration byte and historical row.
+
+## Fixed browser URL and stale-response sequence
+
+Recreate the seven rows in the existing isolated browser schema. Keep `/monitors`
+and use URL query keys `history` (Monitor ID), `state`, `limit`, `cursor`. Explicit
+browser page size is3; missing API limit still defaults to20.
+
+1. Deep-link to first history page3; display7,6,5. The actual browser client reads
+   the cursor response header. Insert the same newer eighth row between pages.
+2. Next displays4,3,2; next displays1. Back/forward restores those URL conditions
+   and rows. Reload the cursor page and retain1.
+3. Change filter to FAILED: clear cursor, preserve size3, display6,4,2.
+4. Intercept exactly one SUCCEEDED history read, forward it to the real API, and
+   hold its response at an explicit promise barrier. While held, change to All.
+   The newer All response must display8,7,6 with its matching URL/filter.
+5. Release the older SUCCEEDED response (8,7,5). It must not overwrite the active
+   All rows, URL or request status. Back/forward and reload still reconstruct the
+   selected filter/page. No sleep, outcome-driven delay or polling request loop.
+
+## Boundaries and budget
+
+Use PostgreSQL15432/project `wse-industry`, new owned `e07_*` Java schemas and the
+existing `e04_browser` harness, API4322/UI4323/fixture4321. Refuse occupied servers,
+await only owned processes/held routes, and remove only disposable schemas.
+Preserve public data and volumes. Runtime credentials/CSRF/session material never
+enter evidence; existing capture restrictions remain.
+
+One unchanged baseline. Zero load runs, automatic retries or parameter sweeps.
+Run decisive authoring tests and static/build checks; record all invocations and
+failures. Root owns final affected regression verification. No new dependency,
+generic cache/cursor framework, planner deep dive, E08 or later features. Legacy
+reload setup may explicitly close URL history only where needed to preserve its
+original product inputs and assertions.


