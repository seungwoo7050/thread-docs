## `test(mock): verify deterministic provider sessions`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
index 243b931..252a9fa 100644
--- a/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/mock/MockOddsProviderTest.java
@@ -167,6 +167,31 @@ class MockOddsProviderTest {
             SettlementResult.LOST.name());
   }
 
+  @Test
+  void finalOutcomeDoesNotDependOnOddsTickFrequency() {
+    MockOddsProvider sparse = newProvider(77L);
+    MockOddsProvider busy = newProvider(77L);
+    sparse.seed();
+    busy.seed();
+    var sparseEvent = sparse.listEvents(Sport.FOOTBALL).get(0);
+    var busyEvent =
+        busy.listEvents(Sport.FOOTBALL).stream()
+            .filter(summary -> summary.eventId().equals(sparseEvent.eventId()))
+            .findFirst()
+            .orElseThrow();
+
+    sparse.tick(sparseEvent.scheduledStartAt());
+    busy.tick(busyEvent.scheduledStartAt());
+    for (int second = 1; second <= 20; second++) {
+      busy.tick(busyEvent.scheduledStartAt().plusSeconds(second));
+    }
+    sparse.tick(sparseEvent.scheduledStartAt().plusSeconds(90));
+    busy.tick(busyEvent.scheduledStartAt().plusSeconds(90));
+
+    assertThat(sparse.getMatchResult(sparseEvent.eventId()))
+        .contains(busy.getMatchResult(busyEvent.eventId()).orElseThrow());
+  }
+
   private static MockOddsProvider newProvider(long seed) {
     MockProperties properties =
         new MockProperties(
