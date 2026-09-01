## `feat(audit): stream completed action rows`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditService.java b/src/main/java/com/sportsbook/admin/audit/AuditService.java
index 635469a..43a0aa6 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditService.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditService.java
@@ -8,9 +8,11 @@ import org.springframework.stereotype.Service;
 public class AuditService {
 
   private final AuditWriteRepository writes;
+  private final AdminActionPublisher publisher;
 
-  public AuditService(AuditWriteRepository writes) {
+  public AuditService(AuditWriteRepository writes, AdminActionPublisher publisher) {
     this.writes = writes;
+    this.publisher = publisher;
   }
 
   public void begin(AdminContext context, String action, String target, String reason) {
@@ -22,10 +24,11 @@ public class AuditService {
     }
   }
 
-  public AuditTerminalRecord complete(
-      UUID actionId, AuditOutcome outcome, Integer httpStatus) {
+  public AuditTerminalRecord complete(UUID actionId, AuditOutcome outcome, Integer httpStatus) {
     try {
-      return writes.complete(actionId, outcome, httpStatus);
+      AuditTerminalRecord terminal = writes.complete(actionId, outcome, httpStatus);
+      publisher.publish(terminal);
+      return terminal;
     } catch (RuntimeException failure) {
       throw new AuditPersistenceException(
           actionId, AuditPersistenceException.Phase.COMPLETE, failure);


## `test(audit): stream only finalized action rows`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java b/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java
index 37534ac..38936e5 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditServiceFailureTest.java
@@ -13,13 +13,12 @@ import org.springframework.dao.DataAccessResourceFailureException;
 
 class AuditServiceFailureTest {
 
-  private static final UUID ACTION_ID =
-      UUID.fromString("018f0000-0000-7000-8000-000000000041");
+  private static final UUID ACTION_ID = UUID.fromString("018f0000-0000-7000-8000-000000000041");
 
   @Test
   void identifiesBeginPersistenceFailures() {
     AuditWriteRepository writes = mock(AuditWriteRepository.class);
-    AuditService service = new AuditService(writes);
+    AuditService service = new AuditService(writes, mock(AdminActionPublisher.class));
     AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, ACTION_ID, "trace-1");
     doThrow(new DataAccessResourceFailureException("database unavailable"))
         .when(writes)
@@ -38,7 +37,7 @@ class AuditServiceFailureTest {
   @Test
   void identifiesTerminalFinalizationFailures() {
     AuditWriteRepository writes = mock(AuditWriteRepository.class);
-    AuditService service = new AuditService(writes);
+    AuditService service = new AuditService(writes, mock(AdminActionPublisher.class));
     doThrow(new IllegalStateException("lost STARTED claim"))
         .when(writes)
         .complete(ACTION_ID, AuditOutcome.UNKNOWN, null);
diff --git a/src/test/java/com/sportsbook/admin/audit/AuditServicePublishingTest.java b/src/test/java/com/sportsbook/admin/audit/AuditServicePublishingTest.java
new file mode 100644
index 0000000..dd02095
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditServicePublishingTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+
+class AuditServicePublishingTest {
+
+  private static final UUID ACTION_ID = UUID.fromString("018f0000-0000-7000-8000-000000000095");
+
+  @Test
+  void publishesOnlyAfterTheGuardedTerminalUpdate() {
+    AuditWriteRepository writes = mock(AuditWriteRepository.class);
+    AdminActionPublisher publisher = mock(AdminActionPublisher.class);
+    AuditTerminalRecord terminal = terminal();
+    when(writes.complete(ACTION_ID, AuditOutcome.SUCCESS, 202)).thenReturn(terminal);
+    AuditService service = new AuditService(writes, publisher);
+
+    AuditTerminalRecord result = service.complete(ACTION_ID, AuditOutcome.SUCCESS, 202);
+
+    assertThat(result).isSameAs(terminal);
+    InOrder lifecycle = inOrder(writes, publisher);
+    lifecycle.verify(writes).complete(ACTION_ID, AuditOutcome.SUCCESS, 202);
+    lifecycle.verify(publisher).publish(terminal);
+  }
+
+  private static AuditTerminalRecord terminal() {
+    Instant started = Instant.parse("2026-08-22T01:02:03Z");
+    return new AuditTerminalRecord(
+        ACTION_ID,
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


## `test(audit): deliver terminal events to Kafka`

diff --git a/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherBrokerTest.java b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherBrokerTest.java
new file mode 100644
index 0000000..855b6ea
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherBrokerTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.event.AdminActionRecorded;
+import com.sportsbook.admin.security.AdminRole;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.io.IOException;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Map;
+import java.util.UUID;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;
+
+@SpringJUnitConfig(AdminActionPublisherBrokerTest.TestConfiguration.class)
+@EmbeddedKafka(topics = "admin.action", partitions = 1)
+class AdminActionPublisherBrokerTest {
+
+  @Autowired private EmbeddedKafkaBroker broker;
+
+  @Test
+  void sendsAnAvroTerminalEventKeyedByActor() throws IOException {
+    var configuration = new AuditKafkaConfiguration(broker.getBrokersAsString());
+    var producerFactory = configuration.auditProducerFactory();
+    var template = configuration.auditKafkaTemplate(producerFactory);
+    var publisher = new AdminActionPublisher(template, new SimpleMeterRegistry(), "admin.action");
+
+    try {
+      publisher.publish(terminal());
+      template.flush();
+
+      ConsumerRecord<String, byte[]> record = consumeOne();
+      AdminActionRecorded event = decode(record.value());
+      assertThat(record.key()).isEqualTo("operator-1");
+      assertThat(event.getActionId()).isEqualTo("018f0000-0000-7000-8000-000000000093");
+      assertThat(event.getOutcome()).isEqualTo("SUCCESS");
+      assertThat(event.getHttpStatus()).isEqualTo(202);
+    } finally {
+      template.destroy();
+      ((DefaultKafkaProducerFactory<String, byte[]>) producerFactory).destroy();
+    }
+  }
+
+  private ConsumerRecord<String, byte[]> consumeOne() {
+    Map<String, Object> properties =
+        KafkaTestUtils.consumerProps("audit-publisher-test", "false", broker);
+    properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
+    try (Consumer<String, byte[]> consumer =
+        new DefaultKafkaConsumerFactory<>(
+                properties, new StringDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      consumer.subscribe(java.util.List.of("admin.action"));
+      return KafkaTestUtils.getSingleRecord(consumer, "admin.action", Duration.ofSeconds(10));
+    }
+  }
+
+  private static AdminActionRecorded decode(byte[] value) throws IOException {
+    var reader = new SpecificDatumReader<AdminActionRecorded>(AdminActionRecorded.class);
+    return reader.read(null, DecoderFactory.get().binaryDecoder(value, null));
+  }
+
+  private static AuditTerminalRecord terminal() {
+    Instant started = Instant.parse("2026-08-22T01:02:03Z");
+    return new AuditTerminalRecord(
+        UUID.fromString("018f0000-0000-7000-8000-000000000093"),
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
+
+  @Configuration(proxyBeanMethods = false)
+  static class TestConfiguration {}
+}


## `test(audit): provide real Kafka broker fixture`

diff --git a/src/test/java/com/sportsbook/admin/audit/RealKafkaAuditFixture.java b/src/test/java/com/sportsbook/admin/audit/RealKafkaAuditFixture.java
new file mode 100644
index 0000000..9e7e10b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/RealKafkaAuditFixture.java
@@ -0,0 +1,90 @@
+package com.sportsbook.admin.audit;
+
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Duration;
+import java.util.List;
+import java.util.Map;
+import java.util.UUID;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.admin.Admin;
+import org.apache.kafka.clients.admin.AdminClientConfig;
+import org.apache.kafka.clients.admin.NewTopic;
+import org.apache.kafka.clients.consumer.Consumer;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeAll;
+import org.junit.jupiter.api.BeforeEach;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+import org.testcontainers.kafka.KafkaContainer;
+import org.testcontainers.utility.DockerImageName;
+
+@Testcontainers
+abstract class RealKafkaAuditFixture {
+
+  static final String TOPIC = "admin.action";
+
+  @Container
+  static final KafkaContainer KAFKA =
+      new KafkaContainer(DockerImageName.parse("apache/kafka:3.8.0"))
+          .withEnv("KAFKA_AUTO_CREATE_TOPICS_ENABLE", "false");
+
+  private ProducerFactory<String, byte[]> producerFactory;
+  private KafkaTemplate<String, byte[]> kafka;
+  private AdminActionPublisher publisher;
+
+  @BeforeAll
+  static void createTopic() throws Exception {
+    Map<String, Object> settings =
+        Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA.getBootstrapServers());
+    try (Admin admin = Admin.create(settings)) {
+      admin
+          .createTopics(List.of(new NewTopic(TOPIC, 3, (short) 1)))
+          .all()
+          .get(10, TimeUnit.SECONDS);
+    }
+  }
+
+  @BeforeEach
+  void createPublisher() {
+    AuditKafkaConfiguration configuration =
+        new AuditKafkaConfiguration(KAFKA.getBootstrapServers());
+    producerFactory = configuration.auditProducerFactory();
+    kafka = configuration.auditKafkaTemplate(producerFactory);
+    publisher = new AdminActionPublisher(kafka, new SimpleMeterRegistry(), TOPIC);
+  }
+
+  @AfterEach
+  void destroyPublisher() {
+    kafka.destroy();
+    ((DefaultKafkaProducerFactory<String, byte[]>) producerFactory).destroy();
+  }
+
+  void publish(AuditTerminalRecord record) {
+    publisher.publish(record);
+    kafka.flush();
+  }
+
+  ConsumerRecord<String, byte[]> consumeOne() {
+    Map<String, Object> properties =
+        KafkaTestUtils.consumerProps(
+            KAFKA.getBootstrapServers(), "real-audit-" + UUID.randomUUID(), "false");
+    properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
+    properties.put(ConsumerConfig.ALLOW_AUTO_CREATE_TOPICS_CONFIG, false);
+    try (Consumer<String, byte[]> consumer =
+        new DefaultKafkaConsumerFactory<>(
+                properties, new StringDeserializer(), new ByteArrayDeserializer())
+            .createConsumer()) {
+      consumer.subscribe(List.of(TOPIC));
+      return KafkaTestUtils.getSingleRecord(consumer, TOPIC, Duration.ofSeconds(10));
+    }
+  }
+}


## `test(audit): publish terminal events through real Kafka`

diff --git a/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherRealKafkaTest.java b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherRealKafkaTest.java
new file mode 100644
index 0000000..4f8cb1b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AdminActionPublisherRealKafkaTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.event.AdminActionRecorded;
+import com.sportsbook.admin.security.AdminRole;
+import java.io.IOException;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.avro.SchemaNormalization;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.junit.jupiter.api.Test;
+
+class AdminActionPublisherRealKafkaTest extends RealKafkaAuditFixture {
+
+  @Test
+  void publishesACompleteRawAvroTerminalEvent() throws IOException {
+    Instant started = Instant.parse("2026-08-22T01:02:03.123Z");
+    Instant completed = Instant.parse("2026-08-22T01:02:04.456Z");
+    AuditTerminalRecord terminal =
+        new AuditTerminalRecord(
+            UUID.fromString("018f0000-0000-7000-8000-000000000093"),
+            "operator-real-kafka",
+            AdminRole.TRADER,
+            "MARKET_CLOSE",
+            "event-1/market-4",
+            AuditOutcome.UNKNOWN,
+            null,
+            "downstream outcome unknown",
+            "7f6d55f9e7e2482190f9ed6647e9d62b",
+            started,
+            completed);
+
+    publish(terminal);
+
+    var record = consumeOne();
+    AdminActionRecorded event = decode(record.value());
+    assertThat(record.topic()).isEqualTo(TOPIC);
+    assertThat(record.key()).isEqualTo(terminal.actorId());
+    assertThat(event.getActionId()).isEqualTo(terminal.actionId().toString());
+    assertThat(event.getActorId()).isEqualTo(terminal.actorId());
+    assertThat(event.getActorRole()).isEqualTo(terminal.actorRole().name());
+    assertThat(event.getAction()).isEqualTo(terminal.action());
+    assertThat(event.getTarget()).isEqualTo(terminal.target());
+    assertThat(event.getOutcome()).isEqualTo(terminal.outcome().name());
+    assertThat(event.getHttpStatus()).isNull();
+    assertThat(event.getReason()).isEqualTo(terminal.reason());
+    assertThat(event.getTraceId()).isEqualTo(terminal.traceId());
+    assertThat(event.getStartedAt()).isEqualTo(started);
+    assertThat(event.getCompletedAt()).isEqualTo(completed);
+    assertThat(SchemaNormalization.parsingFingerprint64(event.getSchema()))
+        .isEqualTo(467411456356349815L);
+  }
+
+  private static AdminActionRecorded decode(byte[] value) throws IOException {
+    var reader = new SpecificDatumReader<AdminActionRecorded>(AdminActionRecorded.class);
+    return reader.read(null, DecoderFactory.get().binaryDecoder(value, null));
+  }
+}


## `feat(ops): make readiness database-only`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 7bdd5e7..719937c 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -37,6 +37,11 @@ management:
     health:
       probes:
         enabled: true
+      group:
+        liveness:
+          include: livenessState
+        readiness:
+          include: readinessState,db
       show-details: never
       show-components: never
   health:


## `test(ops): fail readiness with the database`

diff --git a/src/test/java/com/sportsbook/admin/ops/DatabaseReadinessTest.java b/src/test/java/com/sportsbook/admin/ops/DatabaseReadinessTest.java
new file mode 100644
index 0000000..282d193
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/DatabaseReadinessTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.ops;
+
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.admin.security.TestJwtKeys;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.springframework.test.web.servlet.MockMvc;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@SpringBootTest(
+    properties = {
+      "spring.kafka.bootstrap-servers=127.0.0.1:1",
+      "admin.audit.stale-scan-interval=PT1H",
+      "admin.downstream.credentials.wallet-api-key=wallet-admin-test-key-000000000001",
+      "admin.downstream.credentials.risk-api-key=risk-admin-test-key-00000000000002",
+      "admin.downstream.credentials.odds-feed-api-key=odds-admin-test-key-00000000000003",
+      "admin.downstream.credentials.settlement-api-key=settlement-admin-test-key-000000004"
+    })
+@AutoConfigureMockMvc
+@Testcontainers
+class DatabaseReadinessTest {
+
+  @Container
+  private static final PostgreSQLContainer<?> POSTGRES =
+      new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @DynamicPropertySource
+  static void dependencies(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+    registry.add("admin.security.jwt.public-key", TestJwtKeys::publicKeyPem);
+  }
+
+  @Autowired private MockMvc mvc;
+
+  @Test
+  void staysReadyWithoutKafkaAndFailsWhenPostgresqlStops() throws Exception {
+    mvc.perform(get("/actuator/health/readiness"))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.status").value("UP"))
+        .andExpect(jsonPath("$.components").doesNotExist());
+
+    POSTGRES.stop();
+
+    mvc.perform(get("/actuator/health/readiness"))
+        .andExpect(status().isServiceUnavailable())
+        .andExpect(jsonPath("$.status").value("DOWN"))
+        .andExpect(jsonPath("$.components").doesNotExist());
+  }
+}
diff --git a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
index 1b0fa7a..9aea4fb 100644
--- a/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
+++ b/src/test/java/com/sportsbook/admin/security/SecurityChainTest.java
@@ -16,6 +16,7 @@ import com.sportsbook.admin.context.AdminContextArgumentResolver;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.actuate.health.HealthIndicator;
 import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
 import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
 import org.springframework.boot.test.context.SpringBootTest;
@@ -63,6 +64,9 @@ class SecurityChainTest {
 
   @MockBean private AuditWriteRepository auditWrites;
 
+  @MockBean(name = "db")
+  private HealthIndicator databaseHealth;
+
   @Test
   void rejectsAnonymousRequestsWithAnRfc7807Response() throws Exception {
     mvc.perform(get(ADMIN_ONLY))
@@ -92,12 +96,9 @@ class SecurityChainTest {
         .andExpect(content().contentTypeCompatibleWith("text/plain"));
 
     mvc.perform(
-            get("/actuator/metrics")
-                .header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+            get("/actuator/metrics").header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
         .andExpect(status().isForbidden());
-    mvc.perform(
-            get("/actuator/env")
-                .header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
+    mvc.perform(get("/actuator/env").header(AUTHORIZATION, TestJwtKeys.bearer("admin-1", "ADMIN")))
         .andExpect(status().isForbidden());
   }
 
