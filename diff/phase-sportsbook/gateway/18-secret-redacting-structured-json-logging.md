# 비밀정보를 정화하는 구조화 JSON 로깅

## `build(deps): add observability support`

diff --git a/pom.xml b/pom.xml
index 612d83b..1d5a822 100644
--- a/pom.xml
+++ b/pom.xml
@@ -24,6 +24,7 @@
         <spring-cloud.version>2023.0.3</spring-cloud.version>
         <bucket4j.version>8.9.0</bucket4j.version>
         <avro.version>1.12.0</avro.version>
+        <logstash.version>7.4</logstash.version>
     </properties>
 
     <dependencyManagement>
@@ -91,6 +92,23 @@
             <artifactId>avro</artifactId>
             <version>${avro.version}</version>
         </dependency>
+        <dependency>
+            <groupId>net.logstash.logback</groupId>
+            <artifactId>logstash-logback-encoder</artifactId>
+            <version>${logstash.version}</version>
+        </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-tracing-bridge-otel</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.opentelemetry</groupId>
+            <artifactId>opentelemetry-exporter-otlp</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-registry-prometheus</artifactId>
+        </dependency>
     </dependencies>
 
     <build>


## `feat(logging): emit redacted structured logs`

diff --git a/src/main/java/com/sportsbook/gateway/logging/RedactedEventJsonProvider.java b/src/main/java/com/sportsbook/gateway/logging/RedactedEventJsonProvider.java
new file mode 100644
index 0000000..79cda2c
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/logging/RedactedEventJsonProvider.java
@@ -0,0 +1,41 @@
+package com.sportsbook.gateway.logging;
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
+  private static final String STACK_TRACE = "stack_trace";
+  private static final Pattern LABELLED_SECRET =
+      Pattern.compile(
+          "(?i)((?:[\\\"']?)(?:authorization|(?:x[-_ ]?)?internal[-_ ]?api[-_ ]?key|"
+              + "api[-_ ]?key|password|token)(?:[\\\"']?)\\s*[:=]\\s*)"
+              + "(?:bearer\\s+)?(?:\\\"[^\\\"]*\\\"|'[^']*'|[^\\r\\n,;]+)");
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
+          STACK_TRACE, redact(ThrowableProxyUtil.asString(event.getThrowableProxy())));
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
index 0000000..5e4ac23
--- /dev/null
+++ b/src/main/resources/logback-spring.xml
@@ -0,0 +1,29 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<configuration>
+    <springProperty scope="context" name="service" source="spring.application.name"
+                    defaultValue="gateway"/>
+
+    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
+        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
+            <providers>
+                <timestamp/>
+                <logLevel/>
+                <loggerName/>
+                <provider class="com.sportsbook.gateway.logging.RedactedEventJsonProvider"/>
+                <mdc>
+                    <includeMdcKeyName>traceId</includeMdcKeyName>
+                    <includeMdcKeyName>spanId</includeMdcKeyName>
+                </mdc>
+                <pattern>
+                    <pattern>{"service":"${service}"}</pattern>
+                </pattern>
+            </providers>
+        </encoder>
+    </appender>
+
+    <logger name="com.sportsbook.gateway" level="INFO"/>
+    <logger name="org.apache.kafka" level="WARN"/>
+    <root level="INFO">
+        <appender-ref ref="CONSOLE"/>
+    </root>
+</configuration>


## `test(logging): verify JSON redaction and context`

diff --git a/src/test/java/com/sportsbook/gateway/logging/StructuredLoggingTest.java b/src/test/java/com/sportsbook/gateway/logging/StructuredLoggingTest.java
new file mode 100644
index 0000000..2233307
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/logging/StructuredLoggingTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.gateway.logging;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import ch.qos.logback.classic.Level;
+import ch.qos.logback.classic.LoggerContext;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.slf4j.MDC;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.system.CapturedOutput;
+import org.springframework.boot.test.system.OutputCaptureExtension;
+
+@ExtendWith(OutputCaptureExtension.class)
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
+class StructuredLoggingTest {
+
+  private static final Logger log = LoggerFactory.getLogger(StructuredLoggingTest.class);
+  private static final ObjectMapper JSON = new ObjectMapper();
+
+  @Test
+  void redactsSecretsAndEmitsOnlyAllowedContext(CapturedOutput output) throws Exception {
+    List<String> secrets =
+        List.of(
+            "basic.fixture.secret",
+            "internal.fixture.secret",
+            "api.fixture.secret",
+            "password.fixture.secret",
+            "token.fixture.secret",
+            "bare.fixture.secret",
+            "mdc.fixture.secret",
+            "stack.fixture.secret");
+    MDC.put("traceId", "fixture-trace-id");
+    MDC.put("spanId", "fixture-span-id");
+    MDC.put("authorization", secrets.get(6));
+    try {
+      log.info(
+          "audit-marker authorization: Basic {}, x-internal-api-key={}, apiKey={}, password={},"
+              + " token={}, standalone Bearer {}",
+          secrets.get(0),
+          secrets.get(1),
+          secrets.get(2),
+          secrets.get(3),
+          secrets.get(4),
+          secrets.get(5),
+          new IllegalStateException("authorization: Basic " + secrets.get(7)));
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
+            "service");
+    assertThat(event.path("service").asText()).isEqualTo("gateway");
+    assertThat(event.path("traceId").asText()).isEqualTo("fixture-trace-id");
+    assertThat(event.path("spanId").asText()).isEqualTo("fixture-span-id");
+    assertThat(event.has("authorization")).isFalse();
+    assertThat(event.path("message").asText()).contains("[REDACTED]");
+    assertThat(event.path("stack_trace").asText()).contains("IllegalStateException", "[REDACTED]");
+    assertThat(secrets).allSatisfy(secret -> assertThat(event.toString()).doesNotContain(secret));
+  }
+
+  @Test
+  void omitsStackTraceWithoutFailure(CapturedOutput output) throws Exception {
+    log.info("plain-marker");
+
+    assertThat(event(output, "plain-marker").has("stack_trace")).isFalse();
+  }
+
+  @Test
+  void fixesApplicationAndDependencyLogLevels() {
+    LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
+
+    assertThat(context.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME).getLevel())
+        .isEqualTo(Level.INFO);
+    assertThat(context.getLogger("com.sportsbook.gateway").getLevel()).isEqualTo(Level.INFO);
+    assertThat(context.getLogger("org.apache.kafka").getLevel()).isEqualTo(Level.WARN);
+  }
+
+  private static JsonNode event(CapturedOutput output, String marker) throws Exception {
+    String line =
+        output.getOut().lines().filter(value -> value.contains(marker)).findFirst().orElseThrow();
+    return JSON.readTree(line);
+  }
+}
