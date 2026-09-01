# 비밀값 안전한 구조화 로깅

## `feat(logging): emit redacted structured events`

diff --git a/src/main/java/com/sportsbook/admin/logging/RedactedEventJsonProvider.java b/src/main/java/com/sportsbook/admin/logging/RedactedEventJsonProvider.java
new file mode 100644
index 0000000..d5bea0c
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/logging/RedactedEventJsonProvider.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.logging;
+
+import ch.qos.logback.classic.spi.ILoggingEvent;
+import ch.qos.logback.classic.spi.ThrowableProxyUtil;
+import com.fasterxml.jackson.core.JsonGenerator;
+import java.io.IOException;
+import java.util.regex.Pattern;
+import net.logstash.logback.composite.AbstractFieldJsonProvider;
+
+public final class RedactedEventJsonProvider extends AbstractFieldJsonProvider<ILoggingEvent> {
+
+  private static final String REDACTED = "[REDACTED]";
+  private static final Pattern LABELLED_SECRET =
+      Pattern.compile(
+          "(?i)((?:[\"']?)(?:authorization|idempotency[-_ ]?key|"
+              + "x[-_ ]?internal[-_ ]?api[-_ ]?key|x[-_ ]?api[-_ ]?key|"
+              + "api[-_ ]?key|password|token)(?:[\"']?)\\s*[:=]\\s*)"
+              + "(?:bearer\\s+)?(?:\"[^\"]*\"|'[^']*'|[^\\r\\n,;]+)");
+  private static final Pattern BEARER_SECRET = Pattern.compile("(?i)\\bbearer\\s+[^\\s,;]+");
+
+  public RedactedEventJsonProvider() {
+    setFieldName("message");
+  }
+
+  @Override
+  public void writeTo(JsonGenerator generator, ILoggingEvent event) throws IOException {
+    generator.writeStringField(getFieldName(), redact(event.getFormattedMessage()));
+    if (event.getThrowableProxy() != null) {
+      generator.writeStringField(
+          "stack_trace", redact(ThrowableProxyUtil.asString(event.getThrowableProxy())));
+    }
+  }
+
+  static String redact(String value) {
+    if (value == null) {
+      return "";
+    }
+    String labelled = LABELLED_SECRET.matcher(value).replaceAll("$1" + REDACTED);
+    return BEARER_SECRET.matcher(labelled).replaceAll("Bearer " + REDACTED);
+  }
+}
diff --git a/src/main/resources/logback-spring.xml b/src/main/resources/logback-spring.xml
new file mode 100644
index 0000000..c03ba58
--- /dev/null
+++ b/src/main/resources/logback-spring.xml
@@ -0,0 +1,30 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<configuration>
+    <springProperty scope="context" name="service" source="spring.application.name"
+                    defaultValue="admin-api"/>
+
+    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
+        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
+            <providers>
+                <timestamp/>
+                <logLevel/>
+                <loggerName/>
+                <provider class="com.sportsbook.admin.logging.RedactedEventJsonProvider"/>
+                <mdc>
+                    <includeMdcKeyName>traceId</includeMdcKeyName>
+                    <includeMdcKeyName>spanId</includeMdcKeyName>
+                    <includeMdcKeyName>adminActionId</includeMdcKeyName>
+                </mdc>
+                <pattern>
+                    <pattern>{"service":"${service}"}</pattern>
+                </pattern>
+            </providers>
+        </encoder>
+    </appender>
+
+    <logger name="com.sportsbook.admin" level="INFO"/>
+    <logger name="org.apache.kafka" level="WARN"/>
+    <root level="INFO">
+        <appender-ref ref="CONSOLE"/>
+    </root>
+</configuration>


## `test(logging): pin structured logger levels`

diff --git a/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java b/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java
new file mode 100644
index 0000000..1112546
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin.logging;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import ch.qos.logback.classic.Level;
+import ch.qos.logback.classic.LoggerContext;
+import org.junit.jupiter.api.Test;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.boot.SpringBootConfiguration;
+import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
+import org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration;
+import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
+import org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration;
+import org.springframework.boot.test.context.SpringBootTest;
+
+@SpringBootTest(
+    classes = StructuredLoggingTest.LoggingApplication.class,
+    properties = "management.endpoint.health.group.readiness.include=readinessState",
+    webEnvironment = SpringBootTest.WebEnvironment.NONE)
+class StructuredLoggingTest {
+
+  @Test
+  void fixesStructuredLoggerLevels() {
+    LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
+
+    assertThat(context.getLogger(Logger.ROOT_LOGGER_NAME).getLevel()).isEqualTo(Level.INFO);
+    assertThat(context.getLogger("com.sportsbook.admin").getLevel()).isEqualTo(Level.INFO);
+    assertThat(context.getLogger("org.apache.kafka").getLevel()).isEqualTo(Level.WARN);
+  }
+
+  @SpringBootConfiguration
+  @EnableAutoConfiguration(
+      exclude = {
+        DataSourceAutoConfiguration.class,
+        HibernateJpaAutoConfiguration.class,
+        FlywayAutoConfiguration.class
+      })
+  static class LoggingApplication {}
+}


## `test(logging): redact structured secrets`

diff --git a/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java b/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java
index 1112546..a1e68f4 100644
--- a/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java
+++ b/src/test/java/com/sportsbook/admin/logging/StructuredLoggingTest.java
@@ -4,22 +4,88 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import ch.qos.logback.classic.Level;
 import ch.qos.logback.classic.LoggerContext;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.util.List;
 import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
 import org.slf4j.Logger;
 import org.slf4j.LoggerFactory;
+import org.slf4j.MDC;
 import org.springframework.boot.SpringBootConfiguration;
 import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
 import org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration;
 import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
 import org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration;
 import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.system.CapturedOutput;
+import org.springframework.boot.test.system.OutputCaptureExtension;
 
+@ExtendWith(OutputCaptureExtension.class)
 @SpringBootTest(
     classes = StructuredLoggingTest.LoggingApplication.class,
     properties = "management.endpoint.health.group.readiness.include=readinessState",
     webEnvironment = SpringBootTest.WebEnvironment.NONE)
 class StructuredLoggingTest {
 
+  private static final Logger log = LoggerFactory.getLogger(StructuredLoggingTest.class);
+  private static final ObjectMapper JSON = new ObjectMapper();
+
+  @Test
+  void redactsHeadersCredentialsAndStackTraces(CapturedOutput output) throws Exception {
+    List<String> secrets =
+        List.of(
+            "authorization.fixture.secret",
+            "idempotency.fixture.secret",
+            "api.header.fixture.secret",
+            "internal.api.fixture.secret",
+            "api.label.fixture.secret",
+            "password.fixture.secret",
+            "token.fixture.secret",
+            "bare.bearer.fixture.secret",
+            "mdc.fixture.secret",
+            "stack.fixture.secret");
+    MDC.put("traceId", "fixture-trace-id");
+    MDC.put("spanId", "fixture-span-id");
+    MDC.put("adminActionId", "018f0000-0000-7000-8000-000000000093");
+    MDC.put("authorization", secrets.get(8));
+    try {
+      log.info(
+          "audit-marker Authorization: Bearer {}, Idempotency-Key={}, X-API-Key={}, "
+              + "X-Internal-Api-Key={}, apiKey={}, password={}, token={}, standalone Bearer {}",
+          secrets.get(0),
+          secrets.get(1),
+          secrets.get(2),
+          secrets.get(3),
+          secrets.get(4),
+          secrets.get(5),
+          secrets.get(6),
+          secrets.get(7),
+          new IllegalStateException("X-API-Key: " + secrets.get(9)));
+    } finally {
+      MDC.clear();
+    }
+
+    JsonNode event = event(output, "audit-marker");
+    assertThat(event.fieldNames())
+        .toIterable()
+        .containsExactlyInAnyOrder(
+            "@timestamp",
+            "level",
+            "logger_name",
+            "message",
+            "stack_trace",
+            "traceId",
+            "spanId",
+            "adminActionId",
+            "service");
+    assertThat(event.path("service").asText()).isEqualTo("admin-api");
+    assertThat(event.path("message").asText()).contains("[REDACTED]");
+    assertThat(event.path("stack_trace").asText()).contains("IllegalStateException", "[REDACTED]");
+    assertThat(event.has("authorization")).isFalse();
+    assertThat(secrets).allSatisfy(secret -> assertThat(event.toString()).doesNotContain(secret));
+  }
+
   @Test
   void fixesStructuredLoggerLevels() {
     LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
@@ -29,6 +95,12 @@ class StructuredLoggingTest {
     assertThat(context.getLogger("org.apache.kafka").getLevel()).isEqualTo(Level.WARN);
   }
 
+  private static JsonNode event(CapturedOutput output, String marker) throws Exception {
+    String line =
+        output.getOut().lines().filter(value -> value.contains(marker)).findFirst().orElseThrow();
+    return JSON.readTree(line);
+  }
+
   @SpringBootConfiguration
   @EnableAutoConfiguration(
       exclude = {
