# 외부 의존성을 제외한 상태 프로브와 운영 지표

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


## `feat(observability): expose service health and metrics`

diff --git a/pom.xml b/pom.xml
index 2e98680..df2785b 100644
--- a/pom.xml
+++ b/pom.xml
@@ -172,6 +172,18 @@
             <plugin>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-maven-plugin</artifactId>
+                <executions>
+                    <execution>
+                        <goals>
+                            <goal>build-info</goal>
+                        </goals>
+                        <configuration>
+                            <excludeInfoProperties>
+                                <excludeInfoProperty>time</excludeInfoProperty>
+                            </excludeInfoProperties>
+                        </configuration>
+                    </execution>
+                </executions>
             </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
index 5085723..14aa47e 100644
--- a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
@@ -10,12 +10,16 @@ import io.lettuce.core.RedisClient;
 import io.lettuce.core.RedisException;
 import io.lettuce.core.api.StatefulRedisConnection;
 import io.lettuce.core.codec.ByteArrayCodec;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Metrics;
 import jakarta.annotation.PreDestroy;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
 import java.util.concurrent.locks.ReentrantLock;
 import org.slf4j.Logger;
 import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.stereotype.Service;
 
 @Service
@@ -24,18 +28,26 @@ public final class RateLimiterService {
   private static final Logger log = LoggerFactory.getLogger(RateLimiterService.class);
   private final RedisClient redisClient;
   private final Duration expirationGrace;
+  private final Counter failOpenCounter;
   private final ReentrantLock managerLock = new ReentrantLock();
   private volatile StatefulRedisConnection<byte[], byte[]> connection;
   private volatile ProxyManager<byte[]> manager;
 
-  public RateLimiterService(RedisClient redisClient, RateLimitProperties properties) {
+  @Autowired
+  public RateLimiterService(
+      RedisClient redisClient, RateLimitProperties properties, MeterRegistry meterRegistry) {
     this.redisClient = redisClient;
+    this.failOpenCounter = meterRegistry.counter("gateway.ratelimit.fail.open");
     this.expirationGrace =
         properties.user().refillPeriod().compareTo(properties.ip().refillPeriod()) >= 0
             ? properties.user().refillPeriod()
             : properties.ip().refillPeriod();
   }
 
+  public RateLimiterService(RedisClient redisClient, RateLimitProperties properties) {
+    this(redisClient, properties, Metrics.globalRegistry);
+  }
+
   public Result tryConsume(String key, RateLimitProperties.Limit limit) {
     BucketConfiguration configuration =
         BucketConfiguration.builder()
@@ -53,6 +65,7 @@ public final class RateLimiterService {
           ? new Result(true, probe.getRemainingTokens(), 0, false)
           : new Result(false, 0, waitSeconds(probe.getNanosToWaitForRefill()), false);
     } catch (RedisException failure) {
+      failOpenCounter.increment();
       log.warn("Rate limiter fail-open: {}", failure.getClass().getSimpleName());
       return new Result(true, -1, 0, true);
     }
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 730568d..04f69e7 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -41,6 +41,27 @@ server:
   port: ${GATEWAY_HTTP_PORT:8080}
   shutdown: graceful
 
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,info,prometheus
+  endpoint:
+    health:
+      probes:
+        enabled: true
+      show-components: never
+      show-details: never
+      group:
+        liveness:
+          include: livenessState
+        readiness:
+          include: readinessState
+  metrics:
+    use-global-registry: true
+    tags:
+      service: ${spring.application.name}
+
 gateway:
   downstream:
     betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}


## `test(observability): verify operational endpoints`

diff --git a/src/test/java/com/sportsbook/gateway/observability/OperationalEndpointsTest.java b/src/test/java/com/sportsbook/gateway/observability/OperationalEndpointsTest.java
new file mode 100644
index 0000000..f8e7d93
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/observability/OperationalEndpointsTest.java
@@ -0,0 +1,83 @@
+package com.sportsbook.gateway.observability;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import java.util.Set;
+import java.util.stream.Collectors;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.actuate.endpoint.web.ExposableWebEndpoint;
+import org.springframework.boot.actuate.endpoint.web.WebEndpointsSupplier;
+import org.springframework.boot.actuate.health.HealthEndpointGroups;
+import org.springframework.boot.info.BuildProperties;
+import org.springframework.boot.test.autoconfigure.actuate.observability.AutoConfigureObservability;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+
+@AutoConfigureObservability(metrics = true, tracing = false)
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
+    properties = {
+      "gateway.ratelimit.enabled=false",
+      "gateway.downstream.wallet.api-key=fixture-wallet-key-32-characters-long"
+    })
+class OperationalEndpointsTest {
+
+  @Autowired TestRestTemplate http;
+  @Autowired WebEndpointsSupplier endpoints;
+  @Autowired HealthEndpointGroups healthGroups;
+  @Autowired BuildProperties buildProperties;
+  @MockBean JwtDecoder decoder;
+
+  @Test
+  void exposesOnlyThePublicOperationalInventory() {
+    Set<String> exposed =
+        endpoints.getEndpoints().stream()
+            .map(ExposableWebEndpoint::getEndpointId)
+            .map(Object::toString)
+            .collect(Collectors.toSet());
+
+    assertThat(exposed).containsExactlyInAnyOrder("health", "info", "prometheus");
+  }
+
+  @Test
+  void reportsOnlyAvailabilityStateForProbes() {
+    for (String group : new String[] {"liveness", "readiness"}) {
+      ResponseEntity<JsonNode> response =
+          http.getForEntity("/actuator/health/" + group, JsonNode.class);
+
+      assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
+      assertThat(response.getBody()).isNotNull();
+      assertThat(response.getBody().path("status").asText()).isEqualTo("UP");
+      assertThat(response.getBody().fieldNames()).toIterable().containsExactly("status");
+    }
+    assertThat(healthGroups.get("liveness").isMember("livenessState")).isTrue();
+    assertThat(healthGroups.get("liveness").isMember("readinessState")).isFalse();
+    assertThat(healthGroups.get("readiness").isMember("readinessState")).isTrue();
+    assertThat(healthGroups.get("readiness").isMember("livenessState")).isFalse();
+    for (String dependency : new String[] {"redis", "kafka", "betting", "wallet", "oddsFeed"}) {
+      assertThat(healthGroups.get("liveness").isMember(dependency)).isFalse();
+      assertThat(healthGroups.get("readiness").isMember(dependency)).isFalse();
+    }
+  }
+
+  @Test
+  void publishesBuildIdentityAndPrometheusMetricsWithoutSourceMetadata() {
+    ResponseEntity<JsonNode> info = http.getForEntity("/actuator/info", JsonNode.class);
+    JsonNode build = info.getBody().path("build");
+    assertThat(build.path("group").asText()).isEqualTo("com.sportsbook");
+    assertThat(build.path("artifact").asText()).isEqualTo("gateway");
+    assertThat(build.path("version").asText()).isEqualTo(buildProperties.getVersion());
+    assertThat(info.getBody().has("git")).isFalse();
+
+    ResponseEntity<String> metrics = http.getForEntity("/actuator/prometheus", String.class);
+    assertThat(metrics.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(metrics.getBody())
+        .containsPattern("gateway_ratelimit_fail_open_total\\{service=\"gateway\",?} 0\\.0");
+  }
+}
diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
index 27b8d70..bb6a7f4 100644
--- a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
@@ -15,6 +15,7 @@ import com.sportsbook.gateway.error.GatewayProblemWriter;
 import io.lettuce.core.RedisClient;
 import io.lettuce.core.RedisConnectionException;
 import io.lettuce.core.codec.ByteArrayCodec;
+import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
 import io.micrometer.tracing.Tracer;
 import jakarta.servlet.DispatcherType;
 import java.time.Duration;
@@ -170,6 +171,22 @@ class DistributedRateLimiterTest {
         .isInstanceOf(IllegalArgumentException.class);
   }
 
+  @Test
+  void countsFailOpenResultsWithoutDynamicTags() {
+    RedisClient unavailable = mock(RedisClient.class);
+    when(unavailable.connect(any(ByteArrayCodec.class)))
+        .thenThrow(new RedisConnectionException("fixture outage"));
+    SimpleMeterRegistry meters = new SimpleMeterRegistry();
+    RateLimitProperties policies = new RateLimitProperties(true, limit, limit, List.of());
+    RateLimiterService limiter = new RateLimiterService(unavailable, policies, meters);
+
+    limiter.tryConsume("first", limit);
+    limiter.tryConsume("second", limit);
+
+    assertThat(meters.get("gateway.ratelimit.fail.open").counter().count()).isEqualTo(2);
+    assertThat(meters.get("gateway.ratelimit.fail.open").counter().getId().getTags()).isEmpty();
+  }
+
   @Test
   void returnsProblemResponseAfterSeparateIpAndUserBucketsAreExhausted() throws Exception {
     RateLimitFilter filter = filter(true);
