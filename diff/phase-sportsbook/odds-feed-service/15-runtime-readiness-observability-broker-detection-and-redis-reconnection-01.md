# 런타임 준비도·관측성·브로커 감지와 Redis 재연결

## `build(observability): add service telemetry`

diff --git a/pom.xml b/pom.xml
index 2dcc542..cfc7809 100644
--- a/pom.xml
+++ b/pom.xml
@@ -23,6 +23,7 @@
         <java.version>17</java.version>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
+        <logstash-encoder.version>7.4</logstash-encoder.version>
     </properties>
 
     <dependencies>
@@ -55,5 +56,26 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-webflux</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-actuator</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>net.logstash.logback</groupId>
+            <artifactId>logstash-logback-encoder</artifactId>
+            <version>${logstash-encoder.version}</version>
+        </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-registry-prometheus</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.micrometer</groupId>
+            <artifactId>micrometer-tracing-bridge-otel</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>io.opentelemetry</groupId>
+            <artifactId>opentelemetry-exporter-otlp</artifactId>
+        </dependency>
     </dependencies>
 </project>


## `chore(runtime): configure service management endpoints`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
new file mode 100644
index 0000000..801f69e
--- /dev/null
+++ b/src/main/resources/application.yml
@@ -0,0 +1,28 @@
+server:
+  port: ${SERVER_PORT:8085}
+  shutdown: graceful
+
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,prometheus
+  endpoint:
+    health:
+      probes:
+        enabled: true
+      show-details: when-authorized
+      group:
+        readiness:
+          include: readinessState,redis
+  health:
+    livenessstate:
+      enabled: true
+    readinessstate:
+      enabled: true
+  tracing:
+    sampling:
+      probability: ${OTEL_SAMPLING_PROBABILITY:1.0}
+  otlp:
+    tracing:
+      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318/v1/traces}


## `test(runtime): verify service endpoint defaults`

diff --git a/src/test/java/com/sportsbook/oddsfeed/RuntimeDefaultsTest.java b/src/test/java/com/sportsbook/oddsfeed/RuntimeDefaultsTest.java
new file mode 100644
index 0000000..06bece7
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/RuntimeDefaultsTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.oddsfeed;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.env.YamlPropertySourceLoader;
+import org.springframework.core.env.PropertySource;
+import org.springframework.core.io.ClassPathResource;
+
+class RuntimeDefaultsTest {
+
+  @Test
+  void definesApprovedServiceEndpointDefaults() throws IOException {
+    PropertySource<?> defaults =
+        new YamlPropertySourceLoader()
+            .load("runtime-defaults", new ClassPathResource("application.yml"))
+            .get(0);
+
+    assertThat(defaults.getProperty("server.port")).isEqualTo("${SERVER_PORT:8085}");
+    assertThat(defaults.getProperty("server.shutdown")).isEqualTo("graceful");
+    assertThat(defaults.getProperty("management.endpoints.web.exposure.include"))
+        .isEqualTo("health,prometheus");
+    assertThat(defaults.getProperty("management.endpoint.health.show-details"))
+        .isEqualTo("when-authorized");
+    assertThat(defaults.getProperty("management.endpoint.health.probes.enabled")).isEqualTo(true);
+    String readinessMembers =
+        (String) defaults.getProperty("management.endpoint.health.group.readiness.include");
+    assertThat(readinessMembers.split(",")).contains("readinessState", "redis");
+    assertThat(defaults.getProperty("management.health.livenessstate.enabled")).isEqualTo(true);
+    assertThat(defaults.getProperty("management.health.readinessstate.enabled")).isEqualTo(true);
+    assertThat(defaults.getProperty("management.tracing.sampling.probability"))
+        .isEqualTo("${OTEL_SAMPLING_PROBABILITY:1.0}");
+    assertThat(defaults.getProperty("management.otlp.tracing.endpoint"))
+        .isEqualTo("${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318/v1/traces}");
+  }
+}


## `feat(config): provide scheduling and UTC time`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
new file mode 100644
index 0000000..15b7d4d
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
@@ -0,0 +1,16 @@
+package com.sportsbook.oddsfeed.config;
+
+import java.time.Clock;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.scheduling.annotation.EnableScheduling;
+
+@Configuration
+@EnableScheduling
+public class ApplicationConfig {
+
+  @Bean
+  public Clock systemClock() {
+    return Clock.systemUTC();
+  }
+}


## `feat(logging): emit structured service logs`

diff --git a/src/main/resources/logback-spring.xml b/src/main/resources/logback-spring.xml
new file mode 100644
index 0000000..7d9ba7f
--- /dev/null
+++ b/src/main/resources/logback-spring.xml
@@ -0,0 +1,28 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<configuration>
+    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
+    <springProperty scope="context" name="appName" source="spring.application.name" defaultValue="odds-feed-service"/>
+
+    <springProfile name="!json">
+        <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
+        <root level="INFO">
+            <appender-ref ref="CONSOLE"/>
+        </root>
+    </springProfile>
+
+    <springProfile name="json">
+        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
+            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
+                <includeMdcKeyName>traceId</includeMdcKeyName>
+                <includeMdcKeyName>spanId</includeMdcKeyName>
+                <customFields>{"service":"${appName}"}</customFields>
+            </encoder>
+        </appender>
+        <root level="INFO">
+            <appender-ref ref="JSON"/>
+        </root>
+    </springProfile>
+
+    <logger name="com.sportsbook" level="DEBUG"/>
+    <logger name="org.apache.kafka" level="WARN"/>
+</configuration>


## `feat(kafka): track broker availability`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java b/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java
new file mode 100644
index 0000000..5936b26
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/BrokerAvailability.java
@@ -0,0 +1,22 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import java.util.concurrent.atomic.AtomicBoolean;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class BrokerAvailability {
+
+  private final AtomicBoolean available = new AtomicBoolean();
+
+  public boolean isAvailable() {
+    return available.get();
+  }
+
+  public void markAvailable() {
+    available.set(true);
+  }
+
+  public void markUnavailable() {
+    available.set(false);
+  }
+}


## `test(kafka): verify availability transitions`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java
new file mode 100644
index 0000000..37e3109
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/BrokerAvailabilityTest.java
@@ -0,0 +1,21 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class BrokerAvailabilityTest {
+
+  @Test
+  void startsUnavailableAndTracksAcknowledgements() {
+    BrokerAvailability availability = new BrokerAvailability();
+
+    assertThat(availability.isAvailable()).isFalse();
+
+    availability.markAvailable();
+    assertThat(availability.isAvailable()).isTrue();
+
+    availability.markUnavailable();
+    assertThat(availability.isAvailable()).isFalse();
+  }
+}


## `feat(kafka): probe broker connectivity`

diff --git a/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java
new file mode 100644
index 0000000..11b74d5
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbe.java
@@ -0,0 +1,37 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import org.apache.avro.specific.SpecificRecord;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class KafkaBrokerProbe {
+
+  private final KafkaTemplate<String, SpecificRecord> kafka;
+  private final KafkaTopicsProperties topics;
+  private final BrokerAvailability availability;
+
+  public KafkaBrokerProbe(
+      KafkaTemplate<String, SpecificRecord> kafka,
+      KafkaTopicsProperties topics,
+      BrokerAvailability availability) {
+    this.kafka = kafka;
+    this.topics = topics;
+    this.availability = availability;
+  }
+
+  @Scheduled(fixedDelayString = "${oddsfeed.publish.probe-interval:5000}")
+  public void probe() {
+    try {
+      if (kafka.partitionsFor(topics.oddsChanged()).isEmpty()) {
+        availability.markUnavailable();
+      } else {
+        availability.markAvailable();
+      }
+    } catch (RuntimeException error) {
+      availability.markUnavailable();
+    }
+  }
+}


## `test(kafka): verify independent broker probes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java
new file mode 100644
index 0000000..9d3c74d
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/kafka/KafkaBrokerProbeTest.java
@@ -0,0 +1,93 @@
+package com.sportsbook.oddsfeed.kafka;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatCode;
+
+import com.sportsbook.oddsfeed.config.KafkaTopicsProperties;
+import java.util.List;
+import java.util.Map;
+import org.apache.avro.specific.SpecificRecord;
+import org.apache.kafka.common.KafkaException;
+import org.apache.kafka.common.Node;
+import org.apache.kafka.common.PartitionInfo;
+import org.junit.jupiter.api.Test;
+import org.springframework.context.annotation.AnnotationConfigApplicationContext;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.EnableScheduling;
+
+class KafkaBrokerProbeTest {
+
+  @Test
+  void updatesAvailabilityWithoutWaitingForARecordSend() {
+    StubKafkaTemplate kafka = new StubKafkaTemplate();
+    BrokerAvailability availability = new BrokerAvailability();
+    KafkaBrokerProbe probe =
+        new KafkaBrokerProbe(
+            kafka,
+            new KafkaTopicsProperties("odds", "market", "lifecycle", "result"),
+            availability);
+
+    probe.probe();
+    assertThat(availability.isAvailable()).isTrue();
+
+    kafka.fail = true;
+    probe.probe();
+    assertThat(availability.isAvailable()).isFalse();
+  }
+
+  @Test
+  void initializesTheDefaultProbeSchedule() {
+    assertThatCode(
+            () -> {
+              try (var context = new AnnotationConfigApplicationContext(ProbeConfiguration.class)) {
+                assertThat(context.getBean(KafkaBrokerProbe.class)).isNotNull();
+              }
+            })
+        .doesNotThrowAnyException();
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableScheduling
+  static class ProbeConfiguration {
+
+    @Bean
+    StubKafkaTemplate kafkaTemplate() {
+      return new StubKafkaTemplate();
+    }
+
+    @Bean
+    KafkaTopicsProperties kafkaTopicsProperties() {
+      return new KafkaTopicsProperties("odds", "market", "lifecycle", "result");
+    }
+
+    @Bean
+    BrokerAvailability brokerAvailability() {
+      return new BrokerAvailability();
+    }
+
+    @Bean
+    KafkaBrokerProbe kafkaBrokerProbe(
+        StubKafkaTemplate kafka, KafkaTopicsProperties topics, BrokerAvailability availability) {
+      return new KafkaBrokerProbe(kafka, topics, availability);
+    }
+  }
+
+  private static final class StubKafkaTemplate extends KafkaTemplate<String, SpecificRecord> {
+    private boolean fail;
+
+    private StubKafkaTemplate() {
+      super(new DefaultKafkaProducerFactory<>(Map.of()));
+    }
+
+    @Override
+    public List<PartitionInfo> partitionsFor(String topic) {
+      if (fail) {
+        throw new KafkaException("broker unavailable");
+      }
+      return List.of(new PartitionInfo(topic, 0, Node.noNode(), null, null));
+    }
+  }
+}


## `feat(delivery): report durable queue health`

diff --git a/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java
new file mode 100644
index 0000000..a394b02
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicator.java
@@ -0,0 +1,33 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import org.springframework.boot.actuate.health.Health;
+import org.springframework.boot.actuate.health.HealthIndicator;
+import org.springframework.stereotype.Component;
+
+@Component
+public class CriticalDeliveryHealthIndicator implements HealthIndicator {
+
+  private final CriticalEventQueue queue;
+  private final OddsFeedPublisher publisher;
+  private final CriticalEventProcessor processor;
+
+  public CriticalDeliveryHealthIndicator(
+      CriticalEventQueue queue, OddsFeedPublisher publisher, CriticalEventProcessor processor) {
+    this.queue = queue;
+    this.publisher = publisher;
+    this.processor = processor;
+  }
+
+  @Override
+  public Health health() {
+    boolean available = queue.isHealthy() && publisher.isHealthy() && processor.isHealthy();
+    Health.Builder health = available ? Health.up() : Health.down();
+    return health
+        .withDetail("redisStream", queue.isHealthy() ? "UP" : "DOWN")
+        .withDetail("kafkaPublisher", publisher.isHealthy() ? "UP" : "DOWN")
+        .withDetail("criticalProcessor", processor.isHealthy() ? "UP" : "DOWN")
+        .withDetail("pendingRecords", queue.pendingCount())
+        .build();
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index df854ab..f051506 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -14,7 +14,7 @@ management:
       show-details: when-authorized
       group:
         readiness:
-          include: readinessState,redis
+          include: readinessState,redis,criticalDelivery
   health:
     livenessstate:
       enabled: true


## `test(delivery): verify queue health details`

diff --git a/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java
new file mode 100644
index 0000000..8b3414e
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/delivery/CriticalDeliveryHealthIndicatorTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.oddsfeed.delivery;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.oddsfeed.config.CriticalDeliveryProperties;
+import com.sportsbook.oddsfeed.publisher.OddsFeedPublisher;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.actuate.health.Status;
+import org.springframework.boot.env.YamlPropertySourceLoader;
+import org.springframework.core.io.ClassPathResource;
+import org.springframework.data.redis.core.StringRedisTemplate;
+
+class CriticalDeliveryHealthIndicatorTest {
+
+  @Test
+  void reportsEveryDeliveryDependencyAndPendingCount() throws java.io.IOException {
+    StubQueue queue = new StubQueue(true, 3);
+    var healthy =
+        new CriticalDeliveryHealthIndicator(queue, new StubPublisher(true), new StubProcessor(true))
+            .health();
+
+    assertThat(healthy.getStatus()).isEqualTo(Status.UP);
+    assertThat(healthy.getDetails())
+        .containsEntry("redisStream", "UP")
+        .containsEntry("kafkaPublisher", "UP")
+        .containsEntry("criticalProcessor", "UP")
+        .containsEntry("pendingRecords", 3L);
+
+    var unavailable =
+        new CriticalDeliveryHealthIndicator(
+                queue, new StubPublisher(false), new StubProcessor(true))
+            .health();
+    assertThat(unavailable.getStatus()).isEqualTo(Status.DOWN);
+    assertThat(unavailable.getDetails()).containsEntry("kafkaPublisher", "DOWN");
+
+    var defaults =
+        new YamlPropertySourceLoader()
+            .load("runtime-defaults", new ClassPathResource("application.yml"))
+            .get(0);
+    assertThat(defaults.getProperty("management.endpoint.health.group.readiness.include"))
+        .isEqualTo("readinessState,redis,criticalDelivery");
+  }
+
+  private static final class StubQueue extends CriticalEventQueue {
+    private final boolean healthy;
+    private final long pending;
+
+    private StubQueue(boolean healthy, long pending) {
+      super(
+          new StringRedisTemplate(),
+          new ObjectMapper(),
+          new CriticalDeliveryProperties("stream", "group", "consumer", 1, Duration.ZERO),
+          new SimpleMeterRegistry());
+      this.healthy = healthy;
+      this.pending = pending;
+    }
+
+    @Override
+    public boolean isHealthy() {
+      return healthy;
+    }
+
+    @Override
+    public long pendingCount() {
+      return pending;
+    }
+  }
+
+  private static final class StubPublisher extends OddsFeedPublisher {
+    private final boolean healthy;
+
+    private StubPublisher(boolean healthy) {
+      super(null, null, null, null);
+      this.healthy = healthy;
+    }
+
+    @Override
+    public boolean isHealthy() {
+      return healthy;
+    }
+  }
+
+  private static final class StubProcessor extends CriticalEventProcessor {
+    private final boolean healthy;
+
+    private StubProcessor(boolean healthy) {
+      super(null, null, null, null);
+      this.healthy = healthy;
+    }
+
+    @Override
+    public boolean isHealthy() {
+      return healthy;
+    }
+  }
+}


## `feat(redis): reconnect after address changes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/RedisClientConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/RedisClientConfig.java
new file mode 100644
index 0000000..f449dfd
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/RedisClientConfig.java
@@ -0,0 +1,21 @@
+package com.sportsbook.oddsfeed.config;
+
+import io.lettuce.core.resource.DnsResolver;
+import io.lettuce.core.resource.DnsResolvers;
+import io.lettuce.core.resource.SocketAddressResolver;
+import org.springframework.boot.autoconfigure.data.redis.ClientResourcesBuilderCustomizer;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+@Configuration(proxyBeanMethods = false)
+public class RedisClientConfig {
+
+  @Bean
+  public ClientResourcesBuilderCustomizer redisDnsResolverCustomizer() {
+    return dnsResolverCustomizer(DnsResolvers.JVM_DEFAULT);
+  }
+
+  static ClientResourcesBuilderCustomizer dnsResolverCustomizer(DnsResolver dnsResolver) {
+    return builder -> builder.socketAddressResolver(SocketAddressResolver.create(dnsResolver));
+  }
+}


