## `test(idempotency): send the literal frozen non-ASCII header`

diff --git a/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java b/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java
index eb08683..310aaf6 100644
--- a/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java
@@ -93,6 +93,7 @@ class ExecutionOwnershipTest {
                 .digest(Files.readAllBytes(Path.of("../evidence/phase-1/E10/fixtures.md")))));
         evidence.put("completed", completed);
         evidence.put("workerEntry", "two non-web JVMs, test-only startup gates, production CheckWorker.executeNext once");
+        evidence.put("defaultRequestFactory", api.getRestTemplate().getRequestFactory().getClass().getName());
         evidence.put("result", "INCOMPLETE");
         try {
             var alice = new SessionClient(api);
@@ -138,11 +139,22 @@ class ExecutionOwnershipTest {
                     new MonitorController.CreateMonitor("A edited", "http://127.0.0.1:4321/ok", 120, false)), 200);
             assertEquals(id, data(alice.sendCheck(a, KEY), 202).get("id").textValue());
             int rejected = 0;
+            String[] invalidLabels = {"missing", "empty", "has-space", "non-ASCII-U+00E9", "ASCII-length-129"};
+            var invalidResults = new ArrayList<Map<String, Object>>();
+            evidence.put("invalidKeys", invalidResults);
             for (String invalid : new String[]{null, "", "has space", "é", "x".repeat(129)}) {
-                var response = alice.sendCheck(a, invalid);
-                assertEquals(400, response.getStatusCode().value());
-                assertEquals("INVALID_INPUT", response.getBody().at("/error/code").textValue());
-                assertEquals(beforeRejected, row(id));
+                String label = invalidLabels[rejected];
+                var response = "é".equals(invalid) ? alice.sendCheckWithLiteralNonAsciiKey(a, invalid)
+                        : alice.sendCheck(a, invalid);
+                JsonNode afterRejected = row(id);
+                invalidResults.add(Map.of("case", label, "status", response.getStatusCode().value(),
+                        "errorCode", response.getBody().at("/error/code").asText(),
+                        "persistedRows", count(), "originalRowUnchanged", beforeRejected.equals(afterRejected),
+                        "outboundRequests", okCalls.get() + failCalls.get(),
+                        "transport", "é".equals(invalid) ? "literal-e9/HTTP1.0" : "default"));
+                assertEquals(400, response.getStatusCode().value(), label);
+                assertEquals("INVALID_INPUT", response.getBody().at("/error/code").textValue(), label);
+                assertEquals(beforeRejected, afterRejected, label);
                 rejected++;
             }
             assertEquals(1, count());
diff --git a/backend/src/test/java/dev/evolution/monitor/SessionClient.java b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
index ba6ce86..9c6e294 100644
--- a/backend/src/test/java/dev/evolution/monitor/SessionClient.java
+++ b/backend/src/test/java/dev/evolution/monitor/SessionClient.java
@@ -3,8 +3,13 @@ package dev.evolution.monitor;
 import static org.junit.jupiter.api.Assertions.*;
 
 import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.net.InetSocketAddress;
+import java.net.Socket;
+import java.nio.charset.StandardCharsets;
 import java.security.SecureRandom;
 import java.util.Base64;
+import java.util.Locale;
 import java.util.Map;
 import java.util.UUID;
 import org.springframework.boot.test.web.client.TestRestTemplate;
@@ -84,6 +89,40 @@ final class SessionClient {
         return sendWithEvidence("/api/monitors/" + monitorId + "/checks", HttpMethod.POST, null, TRUSTED_ORIGIN, this, key);
     }
 
+    ResponseEntity<JsonNode> sendCheckWithLiteralNonAsciiKey(String monitorId, String key) throws Exception {
+        assertEquals("é", key, "This transport is only for the frozen E10 non-ASCII input");
+        assertTrue(cookie != null && csrfHeader != null && csrfToken != null, "Existing session/CSRF proof is required");
+        var uri = api.getRestTemplate().getUriTemplateHandler().expand("/api/monitors/" + monitorId + "/checks");
+        assertEquals("http", uri.getScheme());
+        assertEquals(4322, uri.getPort());
+        assertTrue(uri.getHost().equals("localhost") || uri.getHost().equals("127.0.0.1"));
+        // The selected JDK transport changes é to ?. Simple transport sends UTF-8 instead.
+        // Write the one literal Latin-1 header byte directly; HTTP/1.0 plus close avoids chunk framing.
+        try (var socket = new Socket()) {
+            socket.connect(new InetSocketAddress("127.0.0.1", 4322), 5000);
+            socket.setSoTimeout(5000);
+            String prefix = "POST " + uri.getRawPath() + " HTTP/1.0\r\nHost: 127.0.0.1:4322\r\n"
+                    + "Origin: " + TRUSTED_ORIGIN + "\r\nCookie: " + cookie + "\r\n"
+                    + csrfHeader + ": " + csrfToken + "\r\nIdempotency-Key: ";
+            var output = socket.getOutputStream();
+            output.write(prefix.getBytes(StandardCharsets.US_ASCII));
+            output.write(key.getBytes(StandardCharsets.ISO_8859_1));
+            output.write("\r\nContent-Length: 0\r\nConnection: close\r\n\r\n".getBytes(StandardCharsets.US_ASCII));
+            output.flush();
+            byte[] response = socket.getInputStream().readNBytes(65537);
+            assertTrue(response.length <= 65536, "Literal-key response must remain bounded");
+            String wire = new String(response, StandardCharsets.ISO_8859_1);
+            int headerEnd = wire.indexOf("\r\n\r\n");
+            assertTrue(headerEnd > 0, "The real API must return complete HTTP headers");
+            String headers = wire.substring(0, headerEnd);
+            assertFalse(headers.toLowerCase(Locale.ROOT).contains("\r\ntransfer-encoding:"),
+                    "HTTP/1.0 literal-key response must not use transfer encoding");
+            int status = Integer.parseInt(headers.split("\r\n", 2)[0].split(" ", 3)[1]);
+            JsonNode body = new ObjectMapper().readTree(response, headerEnd + 4, response.length - headerEnd - 4);
+            return ResponseEntity.status(status).body(body);
+        }
+    }
+
     private ResponseEntity<JsonNode> sendWithEvidence(String path, HttpMethod method, Object body,
             String origin, SessionClient proof, String key) {
         var headers = new HttpHeaders();
diff --git a/evidence/phase-1/E10/repair1/correction-plan.md b/evidence/phase-1/E10/repair1/correction-plan.md
new file mode 100644
index 0000000..80fe6ef
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/correction-plan.md
@@ -0,0 +1,39 @@
+# Repair1 correction selected after the single diagnostic
+
+The frozen loopback diagnostic ran once and completed all six captures. The
+selected factory was `JdkClientHttpRequestFactory` on21.0.7+6-LTS. Missing, empty,
+spaced and129-character values were preserved. U+00E9 became wire byte`3f` (`?`),
+which is within the unchanged production ASCII33–126 rule. The one predetermined
+Simple factory alternative sent`c3a9`, so it is not used as a substitute for the
+literal U+00E9 header value.
+
+Only the frozen U+00E9 request now uses a narrowly scoped SessionClient adapter.
+It writes byte`e9` directly to the existing API at127.0.0.1:4322 with the existing
+session, trusted Origin and CSRF proof entirely in memory. An HTTP/1.0 request
+with Connection:close permits a bounded response read without a general HTTP
+framing implementation. The response is the real API response, parsed as JSON;
+the unchanged gate still requires400/INVALID_INPUT and no row/outbound effect.
+This adapter only accepts the frozen `é` input and refuses any other origin or
+port. Its5-second connect/read and64KiB response bounds are test transport safety
+bounds, not a change to the product's2-second outbound timeout.
+
+The remaining four invalid inputs and every other E10 request retain the original
+transport. The gate records each fixed case label, actual status/code, durable
+row count, outbound count and original-row equality before asserting, so a new
+failure cannot again leave the invalid input unlabelled. The actual gate's
+default request-factory class is also recorded for comparison with the diagnostic.
+
+No production source, worker entry/gate, barrier, migration, browser, fixed input
+or original acceptance assertion is changed by this correction. The18 inherited
+WIP files were adopted byte-for-byte in the preceding commits; the only repair
+source edits are SessionClient.java and ExecutionOwnershipTest.java.
+
+Read-only PostgreSQL schema inventory before the gate returned only`public`
+(native command0.128686708 seconds). The owned e10_ownership schema is absent.
+All attempted runtime verification remains limited to one diagnostic followed by
+the one pre-recorded ExecutionOwnershipTest command. Any gate failure stops this
+repair without another edit/retry.
+
+The historical failure did not record its exact input or bytes. Its unlabelled
+assertion is preserved as such; the diagnostic proves the transport conversion
+in the frozen matrix, not a retroactively logged historical request.
diff --git a/evidence/phase-1/E10/repair1/diagnostic.json b/evidence/phase-1/E10/repair1/diagnostic.json
new file mode 100644
index 0000000..dfa795d
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/diagnostic.json
@@ -0,0 +1,55 @@
+{
+  "observations" : [ {
+    "case" : "missing",
+    "requestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+    "inputCodePoints" : null,
+    "wireCodePoints" : null,
+    "wireValueHex" : null,
+    "inputPreserved" : true,
+    "captureStatus" : 204
+  }, {
+    "case" : "empty",
+    "requestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+    "inputCodePoints" : [ ],
+    "wireCodePoints" : [ ],
+    "wireValueHex" : "",
+    "inputPreserved" : true,
+    "captureStatus" : 204
+  }, {
+    "case" : "has-space",
+    "requestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+    "inputCodePoints" : [ 104, 97, 115, 32, 115, 112, 97, 99, 101 ],
+    "wireCodePoints" : [ 104, 97, 115, 32, 115, 112, 97, 99, 101 ],
+    "wireValueHex" : "686173207370616365",
+    "inputPreserved" : true,
+    "captureStatus" : 204
+  }, {
+    "case" : "non-ASCII-U+00E9",
+    "requestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+    "inputCodePoints" : [ 233 ],
+    "wireCodePoints" : [ 63 ],
+    "wireValueHex" : "3f",
+    "inputPreserved" : false,
+    "captureStatus" : 204
+  }, {
+    "case" : "ASCII-length-129",
+    "requestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+    "inputCodePoints" : [ 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120 ],
+    "wireCodePoints" : [ 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120, 120 ],
+    "wireValueHex" : "787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878787878",
+    "inputPreserved" : true,
+    "captureStatus" : 204
+  }, {
+    "case" : "non-ASCII-U+00E9-simple",
+    "requestFactory" : "org.springframework.http.client.SimpleClientHttpRequestFactory",
+    "inputCodePoints" : [ 233 ],
+    "wireCodePoints" : [ 195, 169 ],
+    "wireValueHex" : "c3a9",
+    "inputPreserved" : false,
+    "captureStatus" : 204
+  } ],
+  "result" : "OBSERVED",
+  "defaultRequestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+  "javaRuntimeVersion" : "21.0.7+6-LTS",
+  "elapsedSeconds" : 0.628406875
+}
diff --git a/evidence/phase-1/E10/repair1/diagnostic.log b/evidence/phase-1/E10/repair1/diagnostic.log
new file mode 100644
index 0000000..e69de29
diff --git a/evidence/phase-1/E10/repair1/invocations.jsonl b/evidence/phase-1/E10/repair1/invocations.jsonl
new file mode 100644
index 0000000..ad6e1fe
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/invocations.jsonl
@@ -0,0 +1,2 @@
+{"phase":"diagnostic","event":"start","argv":["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/java","--class-path","/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:","/private/tmp/web-systems-evolution-0a006589-industry-spring/evidence/phase-1/E10/repair1/KeyTransportDiagnostic.java","/private/tmp/web-systems-evolution-0a006589-industry-spring/evidence/phase-1/E10/repair1/diagnostic.json"],"cwd":"/private/tmp/web-systems-evolution-0a006589-industry-spring","startedAt":"2026-08-28T05:33:08.278841+00:00","timeoutSeconds":60,"listenersBefore":{"4324":false}}
+{"phase":"diagnostic","event":"finish","startedAt":"2026-08-28T05:33:08.278841+00:00","finishedAt":"2026-08-28T05:33:10.204682+00:00","elapsedSeconds":1.925166,"exitCode":0,"timedOut":false,"processExitAwaited":true,"listenersAfter":{"4324":false}}


