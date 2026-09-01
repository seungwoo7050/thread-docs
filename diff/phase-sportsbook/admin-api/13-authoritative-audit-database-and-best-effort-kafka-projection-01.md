# 권위 감사 DB와 best-effort Kafka 투영

## `feat(audit): define terminal audit event schema`

diff --git a/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc b/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc
new file mode 100644
index 0000000..67bc452
--- /dev/null
+++ b/src/main/avro/com/sportsbook/admin/event/AdminActionRecorded.avsc
@@ -0,0 +1,26 @@
+{
+  "namespace": "com.sportsbook.admin.event",
+  "type": "record",
+  "name": "AdminActionRecorded",
+  "doc": "Terminal streaming copy of the authoritative admin audit row.",
+  "fields": [
+    {"name": "actionId", "type": "string"},
+    {"name": "actorId", "type": "string"},
+    {"name": "actorRole", "type": "string"},
+    {"name": "action", "type": "string"},
+    {"name": "target", "type": ["null", "string"], "default": null},
+    {"name": "outcome", "type": "string"},
+    {"name": "httpStatus", "type": ["null", "int"], "default": null},
+    {"name": "reason", "type": ["null", "string"], "default": null},
+    {"name": "traceId", "type": ["null", "string"], "default": null},
+    {
+      "name": "startedAt",
+      "type": {"type": "long", "logicalType": "timestamp-millis"}
+    },
+    {
+      "name": "completedAt",
+      "type": ["null", {"type": "long", "logicalType": "timestamp-millis"}],
+      "default": null
+    }
+  ]
+}


## `test(audit): pin terminal event compatibility`

diff --git a/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java b/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java
new file mode 100644
index 0000000..5ba9e35
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java
@@ -0,0 +1,74 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.event.AdminActionRecorded;
+import java.io.ByteArrayInputStream;
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import java.time.Instant;
+import org.apache.avro.SchemaNormalization;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.junit.jupiter.api.Test;
+
+class AdminActionAvroTest {
+
+  @Test
+  void pinsTheTerminalSchemaFingerprint() {
+    assertThat(
+            SchemaNormalization.parsingFingerprint64(AdminActionRecorded.getClassSchema()))
+        .isEqualTo(467411456356349815L);
+  }
+
+  @Test
+  void roundTripsSuccessAndUnknownWithNullableStatus() throws IOException {
+    Instant started = Instant.parse("2026-08-22T01:02:03.004Z");
+    Instant completed = started.plusSeconds(5);
+    AdminActionRecorded success =
+        baseBuilder(started, completed)
+            .setOutcome("SUCCESS")
+            .setHttpStatus(202)
+            .setReason("operator request")
+            .build();
+    AdminActionRecorded unknown =
+        baseBuilder(started, completed)
+            .setOutcome("UNKNOWN")
+            .setHttpStatus(null)
+            .setReason(null)
+            .build();
+
+    assertThat(roundTrip(success)).isEqualTo(success);
+    assertThat(roundTrip(unknown)).isEqualTo(unknown);
+    assertThat(roundTrip(unknown).getHttpStatus()).isNull();
+    assertThat(roundTrip(success).getStartedAt()).isEqualTo(started);
+    assertThat(roundTrip(success).getCompletedAt()).isEqualTo(completed);
+  }
+
+  private static AdminActionRecorded.Builder baseBuilder(Instant started, Instant completed) {
+    return AdminActionRecorded.newBuilder()
+        .setActionId("018f0000-0000-7000-8000-000000000091")
+        .setActorId("operator-1")
+        .setActorRole("ADMIN")
+        .setAction("MARKET_CLOSE")
+        .setTarget("market-1")
+        .setTraceId("0123456789abcdef0123456789abcdef")
+        .setStartedAt(started)
+        .setCompletedAt(completed);
+  }
+
+  private static AdminActionRecorded roundTrip(AdminActionRecorded source) throws IOException {
+    ByteArrayOutputStream bytes = new ByteArrayOutputStream();
+    var writer = new SpecificDatumWriter<AdminActionRecorded>(source.getSchema());
+    var encoder = EncoderFactory.get().binaryEncoder(bytes, null);
+    writer.write(source, encoder);
+    encoder.flush();
+
+    var reader = new SpecificDatumReader<AdminActionRecorded>(source.getSchema());
+    var decoder =
+        DecoderFactory.get().binaryDecoder(new ByteArrayInputStream(bytes.toByteArray()), null);
+    return reader.read(null, decoder);
+  }
+}


## `feat(audit): map terminal rows to events`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditEventMapper.java b/src/main/java/com/sportsbook/admin/audit/AuditEventMapper.java
new file mode 100644
index 0000000..190f217
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditEventMapper.java
@@ -0,0 +1,24 @@
+package com.sportsbook.admin.audit;
+
+import com.sportsbook.admin.event.AdminActionRecorded;
+
+public final class AuditEventMapper {
+
+  private AuditEventMapper() {}
+
+  public static AdminActionRecorded toEvent(AuditTerminalRecord record) {
+    return AdminActionRecorded.newBuilder()
+        .setActionId(record.actionId().toString())
+        .setActorId(record.actorId())
+        .setActorRole(record.actorRole().name())
+        .setAction(record.action())
+        .setTarget(record.target())
+        .setOutcome(record.outcome().name())
+        .setHttpStatus(record.httpStatus())
+        .setReason(record.reason())
+        .setTraceId(record.traceId())
+        .setStartedAt(record.startedAt())
+        .setCompletedAt(record.completedAt())
+        .build();
+  }
+}


## `test(audit): preserve terminal event fields`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditEventMapperTest.java b/src/test/java/com/sportsbook/admin/audit/AuditEventMapperTest.java
new file mode 100644
index 0000000..4fdc8f4
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditEventMapperTest.java
@@ -0,0 +1,45 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.event.AdminActionRecorded;
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AuditEventMapperTest {
+
+  @Test
+  void preservesEveryTerminalAuditField() {
+    Instant startedAt = Instant.parse("2026-08-22T01:02:03.004Z");
+    Instant completedAt = Instant.parse("2026-08-22T01:02:08.009Z");
+    AuditTerminalRecord record =
+        new AuditTerminalRecord(
+            UUID.fromString("018f0000-0000-7000-8000-000000000092"),
+            "operator-1",
+            AdminRole.TRADER,
+            "MARKET_SUSPEND",
+            "event-1/market-2",
+            AuditOutcome.UNKNOWN,
+            null,
+            "feed timeout",
+            "0123456789abcdef0123456789abcdef",
+            startedAt,
+            completedAt);
+
+    AdminActionRecorded event = AuditEventMapper.toEvent(record);
+
+    assertThat(event.getActionId()).isEqualTo(record.actionId().toString());
+    assertThat(event.getActorId()).isEqualTo(record.actorId());
+    assertThat(event.getActorRole()).isEqualTo("TRADER");
+    assertThat(event.getAction()).isEqualTo("MARKET_SUSPEND");
+    assertThat(event.getTarget()).isEqualTo("event-1/market-2");
+    assertThat(event.getOutcome()).isEqualTo("UNKNOWN");
+    assertThat(event.getHttpStatus()).isNull();
+    assertThat(event.getReason()).isEqualTo("feed timeout");
+    assertThat(event.getTraceId()).isEqualTo(record.traceId());
+    assertThat(event.getStartedAt()).isEqualTo(startedAt);
+    assertThat(event.getCompletedAt()).isEqualTo(completedAt);
+  }
+}


## `feat(audit): serialize Avro event payloads`

diff --git a/src/main/java/com/sportsbook/admin/audit/AvroSerializer.java b/src/main/java/com/sportsbook/admin/audit/AvroSerializer.java
new file mode 100644
index 0000000..71f57e3
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AvroSerializer.java
@@ -0,0 +1,26 @@
+package com.sportsbook.admin.audit;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import java.io.UncheckedIOException;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+
+public final class AvroSerializer {
+
+  private AvroSerializer() {}
+
+  public static byte[] toBytes(SpecificRecord record) {
+    try {
+      ByteArrayOutputStream bytes = new ByteArrayOutputStream();
+      var writer = new SpecificDatumWriter<SpecificRecord>(record.getSchema());
+      var encoder = EncoderFactory.get().binaryEncoder(bytes, null);
+      writer.write(record, encoder);
+      encoder.flush();
+      return bytes.toByteArray();
+    } catch (IOException failure) {
+      throw new UncheckedIOException("Avro serialization failed", failure);
+    }
+  }
+}


## `test(audit): round-trip serialized Avro bytes`

diff --git a/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java b/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java
index 5ba9e35..1408f19 100644
--- a/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AdminActionAvroTest.java
@@ -4,22 +4,18 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.admin.event.AdminActionRecorded;
 import java.io.ByteArrayInputStream;
-import java.io.ByteArrayOutputStream;
 import java.io.IOException;
 import java.time.Instant;
 import org.apache.avro.SchemaNormalization;
 import org.apache.avro.io.DecoderFactory;
-import org.apache.avro.io.EncoderFactory;
 import org.apache.avro.specific.SpecificDatumReader;
-import org.apache.avro.specific.SpecificDatumWriter;
 import org.junit.jupiter.api.Test;
 
 class AdminActionAvroTest {
 
   @Test
   void pinsTheTerminalSchemaFingerprint() {
-    assertThat(
-            SchemaNormalization.parsingFingerprint64(AdminActionRecorded.getClassSchema()))
+    assertThat(SchemaNormalization.parsingFingerprint64(AdminActionRecorded.getClassSchema()))
         .isEqualTo(467411456356349815L);
   }
 
@@ -60,15 +56,10 @@ class AdminActionAvroTest {
   }
 
   private static AdminActionRecorded roundTrip(AdminActionRecorded source) throws IOException {
-    ByteArrayOutputStream bytes = new ByteArrayOutputStream();
-    var writer = new SpecificDatumWriter<AdminActionRecorded>(source.getSchema());
-    var encoder = EncoderFactory.get().binaryEncoder(bytes, null);
-    writer.write(source, encoder);
-    encoder.flush();
-
     var reader = new SpecificDatumReader<AdminActionRecorded>(source.getSchema());
     var decoder =
-        DecoderFactory.get().binaryDecoder(new ByteArrayInputStream(bytes.toByteArray()), null);
+        DecoderFactory.get()
+            .binaryDecoder(new ByteArrayInputStream(AvroSerializer.toBytes(source)), null);
     return reader.read(null, decoder);
   }
 }


## `feat(audit): configure idempotent Kafka delivery`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditKafkaConfiguration.java b/src/main/java/com/sportsbook/admin/audit/AuditKafkaConfiguration.java
new file mode 100644
index 0000000..f48a3d5
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditKafkaConfiguration.java
@@ -0,0 +1,45 @@
+package com.sportsbook.admin.audit;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+@Configuration(proxyBeanMethods = false)
+public class AuditKafkaConfiguration {
+
+  private static final int MAX_IN_FLIGHT_REQUESTS = 5;
+
+  private final String bootstrapServers;
+
+  public AuditKafkaConfiguration(
+      @Value("${spring.kafka.bootstrap-servers}") String bootstrapServers) {
+    this.bootstrapServers = bootstrapServers;
+  }
+
+  @Bean
+  public ProducerFactory<String, byte[]> auditProducerFactory() {
+    Map<String, Object> properties = new HashMap<>();
+    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
+    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
+    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    properties.put(ProducerConfig.ACKS_CONFIG, "all");
+    properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    properties.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, MAX_IN_FLIGHT_REQUESTS);
+    properties.put(ProducerConfig.CLIENT_ID_CONFIG, "admin-api-audit");
+    return new DefaultKafkaProducerFactory<>(properties);
+  }
+
+  @Bean
+  public KafkaTemplate<String, byte[]> auditKafkaTemplate(
+      ProducerFactory<String, byte[]> auditProducerFactory) {
+    return new KafkaTemplate<>(auditProducerFactory);
+  }
+}


## `test(audit): pin Kafka producer guarantees`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditKafkaConfigurationTest.java b/src/test/java/com/sportsbook/admin/audit/AuditKafkaConfigurationTest.java
new file mode 100644
index 0000000..23a9557
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditKafkaConfigurationTest.java
@@ -0,0 +1,31 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+
+class AuditKafkaConfigurationTest {
+
+  @Test
+  void pinsSafeProducerDeliverySettings() {
+    var configuration = new AuditKafkaConfiguration("broker-1:9092");
+    var factory =
+        (DefaultKafkaProducerFactory<String, byte[]>) configuration.auditProducerFactory();
+
+    Map<String, Object> properties = factory.getConfigurationProperties();
+
+    assertThat(properties)
+        .containsEntry(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker-1:9092")
+        .containsEntry(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class)
+        .containsEntry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class)
+        .containsEntry(ProducerConfig.ACKS_CONFIG, "all")
+        .containsEntry(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true)
+        .containsEntry(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5)
+        .containsEntry(ProducerConfig.CLIENT_ID_CONFIG, "admin-api-audit");
+  }
+}


## `feat(audit): publish terminal actions best effort`

diff --git a/src/main/java/com/sportsbook/admin/audit/AdminActionPublisher.java b/src/main/java/com/sportsbook/admin/audit/AdminActionPublisher.java
new file mode 100644
index 0000000..3440983
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AdminActionPublisher.java
@@ -0,0 +1,53 @@
+package com.sportsbook.admin.audit;
+
+import io.micrometer.core.instrument.MeterRegistry;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class AdminActionPublisher {
+
+  private static final Logger log = LoggerFactory.getLogger(AdminActionPublisher.class);
+  private static final String FAILURE_METRIC = "admin.audit.publish.failure";
+
+  private final KafkaTemplate<String, byte[]> kafka;
+  private final MeterRegistry meters;
+  private final String topic;
+
+  public AdminActionPublisher(
+      KafkaTemplate<String, byte[]> auditKafkaTemplate,
+      MeterRegistry meters,
+      @Value("${admin.audit.topic:admin.action}") String topic) {
+    this.kafka = auditKafkaTemplate;
+    this.meters = meters;
+    this.topic = topic;
+  }
+
+  public void publish(AuditTerminalRecord record) {
+    try {
+      byte[] value = AvroSerializer.toBytes(AuditEventMapper.toEvent(record));
+      kafka
+          .send(topic, record.actorId(), value)
+          .whenComplete(
+              (ignored, failure) -> {
+                if (failure != null) {
+                  recordFailure(record, failure);
+                }
+              });
+    } catch (RuntimeException failure) {
+      recordFailure(record, failure);
+    }
+  }
+
+  private void recordFailure(AuditTerminalRecord record, Throwable failure) {
+    meters.counter(FAILURE_METRIC).increment();
+    log.error(
+        "Failed to publish terminal audit action {} ({})",
+        record.actionId(),
+        record.action(),
+        failure);
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index cee451c..7bdd5e7 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -74,6 +74,7 @@ admin:
       odds-feed-api-key: ${ADMIN_ODDS_FEED_API_KEY:}
       settlement-api-key: ${ADMIN_SETTLEMENT_API_KEY:}
   audit:
+    topic: ${ADMIN_AUDIT_TOPIC:admin.action}
     stale-after: ${ADMIN_AUDIT_STALE_AFTER:5m}
     stale-scan-interval: ${ADMIN_AUDIT_STALE_SCAN_INTERVAL:PT30S}
     stale-batch-size: ${ADMIN_AUDIT_STALE_BATCH_SIZE:100}


## `test(audit): contain Kafka publish failures`

diff --git a/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherFailureTest.java b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherFailureTest.java
new file mode 100644
index 0000000..8d8b3b9
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherFailureTest.java
@@ -0,0 +1,70 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.security.AdminRole;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import org.junit.jupiter.api.Test;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.support.SendResult;
+
+class AdminActionPublisherFailureTest {
+
+  private final SimpleMeterRegistry meters = new SimpleMeterRegistry();
+
+  @Test
+  void containsASynchronousBrokerFailure() {
+    KafkaTemplate<String, byte[]> kafka = kafkaTemplate();
+    when(kafka.send(eq("admin.action"), eq("operator-1"), any(byte[].class)))
+        .thenThrow(new IllegalStateException("broker unavailable"));
+    var publisher = new AdminActionPublisher(kafka, meters, "admin.action");
+
+    assertThatCode(() -> publisher.publish(terminal())).doesNotThrowAnyException();
+
+    assertThat(meters.counter("admin.audit.publish.failure").count()).isEqualTo(1);
+    verify(kafka).send(eq("admin.action"), eq("operator-1"), any(byte[].class));
+  }
+
+  @Test
+  void containsAnAsynchronousBrokerFailure() {
+    KafkaTemplate<String, byte[]> kafka = kafkaTemplate();
+    CompletableFuture<SendResult<String, byte[]>> failed = new CompletableFuture<>();
+    failed.completeExceptionally(new IllegalStateException("ack lost"));
+    when(kafka.send(eq("admin.action"), eq("operator-1"), any(byte[].class))).thenReturn(failed);
+    var publisher = new AdminActionPublisher(kafka, meters, "admin.action");
+
+    assertThatCode(() -> publisher.publish(terminal())).doesNotThrowAnyException();
+
+    assertThat(meters.counter("admin.audit.publish.failure").count()).isEqualTo(1);
+  }
+
+  @SuppressWarnings("unchecked")
+  private static KafkaTemplate<String, byte[]> kafkaTemplate() {
+    return mock(KafkaTemplate.class);
+  }
+
+  private static AuditTerminalRecord terminal() {
+    Instant started = Instant.parse("2026-08-22T01:02:03Z");
+    return new AuditTerminalRecord(
+        UUID.fromString("018f0000-0000-7000-8000-000000000094"),
+        "operator-1",
+        AdminRole.ADMIN,
+        "MARKET_CLOSE",
+        "market-1",
+        AuditOutcome.SUCCESS,
+        202,
+        "operator request",
+        "trace-1",
+        started,
+        started.plusSeconds(1));
+  }
+}


