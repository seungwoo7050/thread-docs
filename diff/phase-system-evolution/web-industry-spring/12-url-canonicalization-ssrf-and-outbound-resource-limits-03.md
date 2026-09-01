## `test(e12): capture outbound completion before observer work`

diff --git a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
index 47b6369..11501ed 100644
--- a/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java
@@ -86,24 +86,29 @@ class CheckRunnerTest {
         try (ServerSocket closedFixture = new ServerSocket()) {
             closedFixture.bind(new InetSocketAddress("127.0.0.1", 4325));
         }
+        var checks = owned(new CheckRunner("http://127.0.0.1:4325", true));
+        var activeExecution = execution();
         long started = System.nanoTime();
-        var result = owned(new CheckRunner("http://127.0.0.1:4325", true)).run(
-                execution(), "http://127.0.0.1:4325/ok");
-        record("closed-local-port", result, started, Map.of("transport", "actual local TCP", "port", 4325));
+        var result = checks.run(activeExecution, "http://127.0.0.1:4325/ok");
+        long duration = elapsed(started);
+        record("closed-local-port", result, duration, Map.of("transport", "actual local TCP", "port", 4325));
         assertNoResponse(result, "CONNECTION_FAILURE");
-        assertTrue(elapsed(started) < 1750);
+        assertTrue(duration < 1750);
     }
 
     @Test
     void headerTimeoutHasNoInventedHttpStatusOnTheWire() throws Exception {
         try (var fixture = new LocalFixture("slow")) {
+            var checks = owned(new CheckRunner("http://127.0.0.1:4325", true));
+            var activeExecution = execution();
             long started = System.nanoTime();
-            var result = owned(new CheckRunner("http://127.0.0.1:4325", true)).run(execution(), "http://127.0.0.1:4325/stall");
-            record("slow-headers", result, started, Map.of("transport", "actual local HTTP", "delayMs", 2000,
+            var result = checks.run(activeExecution, "http://127.0.0.1:4325/stall");
+            long duration = elapsed(started);
+            record("slow-headers", result, duration, Map.of("transport", "actual local HTTP", "delayMs", 2000,
                     "requests", fixture.paths.size(), "responseHeadersSent", false));
             assertNoResponse(result, "TIMEOUT");
             assertEquals(1, fixture.paths.size());
-            assertTrue(elapsed(started) < 1000);
+            assertTrue(duration < 1000);
         }
     }
 
@@ -142,9 +147,11 @@ class CheckRunnerTest {
                 attempt.register(socket);
                 return socket;
             });
+            var activeExecution = execution();
             long started = System.nanoTime();
-            var result = checks.run(execution(), url);
-            record("validated-" + addressText, result, started, Map.of("transport", "connector stub; no live TLS",
+            var result = checks.run(activeExecution, url);
+            long duration = elapsed(started);
+            record("validated-" + addressText, result, duration, Map.of("transport", "connector stub; no live TLS",
                     "dnsCalls", resolutions.get(), "connectorCalls", connections.get(), "connectedAddress", connected.get().getHostAddress(),
                     "logicalHost", "public.e12.test", "bodyBytesRead", socket.readBytes.get() - OK.length(), "socketClosed", socket.closed()));
             assertEquals("SUCCEEDED", result.state());
@@ -155,7 +162,7 @@ class CheckRunnerTest {
             assertEquals(OK.length(), socket.readBytes.get());
             assertTrue(socket.closed());
             assertTrue(socket.request.toString(StandardCharsets.US_ASCII).contains("Host: public.e12.test\r\n"));
-            assertTrue(elapsed(started) < 1750);
+            assertTrue(duration < 1750);
         }
         var parameters = CheckRunner.tlsParameters(URI.create("https://public.e12.test/ok"));
         assertEquals("HTTPS", parameters.getEndpointIdentificationAlgorithm());
@@ -173,9 +180,10 @@ class CheckRunnerTest {
             });
             String literal = "http://" + (address.contains(":") ? "[" + address + "]" : address) + "/ok";
             assertThrows(ResponseStatusException.class, () -> checks.canonicalUrl(literal));
+            var activeExecution = execution();
             long started = System.nanoTime();
-            var result = checks.run(execution(), PUBLIC);
-            record("unsafe-answer-" + address, result, started, Map.of("transport", "numeric resolver stub",
+            var result = checks.run(activeExecution, PUBLIC);
+            record("unsafe-answer-" + address, result, elapsed(started), Map.of("transport", "numeric resolver stub",
                     "unsafeConnectorCalls", connections.get()));
             assertAborted(result);
             assertEquals(0, connections.get());
@@ -186,9 +194,11 @@ class CheckRunnerTest {
                     ? new InetAddress[]{InetAddress.getByName("10.0.0.1")}
                     : new InetAddress[]{InetAddress.getByName("93.184.216.34"), InetAddress.getByName("10.0.0.1")},
                     (url, address, attempt) -> { connections.incrementAndGet(); throw new AssertionError("No mixed-answer I/O"); });
+            var activeExecution = execution();
+            String url = "http://" + hostname + "/ok";
             long started = System.nanoTime();
-            var result = checks.run(execution(), "http://" + hostname + "/ok");
-            record(hostname, result, started, Map.of("transport", "resolver stub", "unsafeConnectorCalls", connections.get()));
+            var result = checks.run(activeExecution, url);
+            record(hostname, result, elapsed(started), Map.of("transport", "resolver stub", "unsafeConnectorCalls", connections.get()));
             assertAborted(result);
             assertEquals(0, connections.get());
         }
@@ -205,9 +215,10 @@ class CheckRunnerTest {
             attempt.register(socket);
             return socket;
         });
+        var activeExecution = execution();
         long started = System.nanoTime();
-        var result = checks.run(execution(), "http://public.e12.test/private");
-        record("redirect-private", result, started, Map.of("transport", "connector stub", "safeConnectorCalls", connections.get(),
+        var result = checks.run(activeExecution, "http://public.e12.test/private");
+        record("redirect-private", result, elapsed(started), Map.of("transport", "connector stub", "safeConnectorCalls", connections.get(),
                 "unsafeConnectorCalls", 0, "socketClosed", socket.closed()));
         assertAborted(result);
         assertEquals(1, connections.get());
@@ -225,8 +236,10 @@ class CheckRunnerTest {
                             sockets.add(socket);
                             return socket;
                         }));
+                var activeExecution = execution();
+                String url = "http://127.0.0.1:4325/" + (mode.equals("redirect") ? "redirect/0" : mode);
                 long started = System.nanoTime();
-                var result = checks.run(execution(), "http://127.0.0.1:4325/" + (mode.equals("redirect") ? "redirect/0" : mode));
+                var result = checks.run(activeExecution, url);
                 long duration = elapsed(started);
                 int reads = sockets.stream().mapToInt(socket -> socket.readBytes.get()).sum();
                 Map<String, Object> details = new LinkedHashMap<>();
@@ -239,7 +252,7 @@ class CheckRunnerTest {
                 if (mode.equals("ok") || mode.equals("body") || mode.equals("informational")) {
                     details.put("bodyBytesRead", reads - fixture.responseHeaders.length());
                 }
-                record("local-" + mode, result, started, details);
+                record("local-" + mode, result, duration, details);
                 assertTrue(duration < 1750, "Fixed total completion bound: " + duration);
                 assertTrue(sockets.stream().allMatch(ObservedSocket::closed));
                 if (mode.equals("trickle")) assertNoResponse(result, "TIMEOUT");
@@ -271,23 +284,34 @@ class CheckRunnerTest {
             }
             return new InetAddress[]{InetAddress.getByName("93.184.216.34")};
         }, (url, address, attempt) -> { connections.incrementAndGet(); throw new AssertionError("No late connection"); });
-        long started = System.nanoTime();
-        CheckRunner.CheckRun result;
+        var activeExecution = execution();
+        Map<String, Object> observation = null;
         try {
-            result = checks.run(execution(), PUBLIC);
+            long started = System.nanoTime();
+            var result = checks.run(activeExecution, PUBLIC);
+            long duration = elapsed(started);
+            observation = record("blocked-DNS", result, duration, Map.of("transport", "explicit resolver barrier",
+                    "resolverThreads", resolutions.get(), "connectorCalls", connections.get()));
+            saveObservations();
             assertNoResponse(result, "TIMEOUT");
-            assertTrue(elapsed(started) < 1750);
+            assertTrue(duration < 1750);
             // A second intent observes finite capacity; it is not an automatic retry of the first intent.
             assertThrows(RejectedExecutionException.class, () -> checks.run(execution(), PUBLIC));
+            observation.put("queuedTasksAccepted", 0);
             assertEquals(1, resolutions.get());
         } finally {
+            long cleanupStarted = System.nanoTime();
             checks.close();
             release.countDown();
             if (actualThread.get() != null) actualThread.get().join(5000);
+            if (observation != null) {
+                observation.put("cleanupMs", elapsed(cleanupStarted));
+                observation.put("resolverThreads", resolutions.get());
+                observation.put("connectorCalls", connections.get());
+                observation.put("actualIoThreadExited", actualThread.get() != null && !actualThread.get().isAlive());
+                saveObservations();
+            }
         }
-        record("blocked-DNS", result, started, Map.of("transport", "explicit resolver barrier",
-                "resolverThreads", resolutions.get(), "queuedTasksAccepted", 0, "connectorCalls", connections.get(),
-                "actualIoThreadExited", !actualThread.get().isAlive()));
         assertFalse(actualThread.get().isAlive());
         assertEquals(0, connections.get());
     }
@@ -306,14 +330,15 @@ class CheckRunnerTest {
         return owned(new CheckRunner("http://127.0.0.1:4321", false, resolver, connector));
     }
     private static long elapsed(long started) { return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started); }
-    private static void record(String label, CheckRunner.CheckRun result, long started, Map<String, ?> details) {
+    private static Map<String, Object> record(String label, CheckRunner.CheckRun result, long duration, Map<String, ?> details) {
         Map<String, Object> row = new LinkedHashMap<>(details);
         row.put("case", label);
         row.put("state", result.state());
         row.put("httpStatus", result.httpStatus());
         row.put("failureReason", result.failureReason());
-        row.put("elapsedMs", elapsed(started));
+        row.put("elapsedMs", duration);
         observations.add(row);
+        return row;
     }
     private static void assertAborted(CheckRunner.CheckRun result) {
         assertEquals("ABORTED", result.state());


