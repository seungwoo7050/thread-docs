## `feat(application): launch wallet service`

diff --git a/pom.xml b/pom.xml
index 32062a2..c5be870 100644
--- a/pom.xml
+++ b/pom.xml
@@ -35,4 +35,13 @@
             <artifactId>spring-boot-starter</artifactId>
         </dependency>
     </dependencies>
+
+    <build>
+        <plugins>
+            <plugin>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-maven-plugin</artifactId>
+            </plugin>
+        </plugins>
+    </build>
 </project>
diff --git a/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java b/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java
new file mode 100644
index 0000000..a6a5dc2
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java
@@ -0,0 +1,12 @@
+package com.sportsbook.wallet;
+
+import org.springframework.boot.SpringApplication;
+import org.springframework.boot.autoconfigure.SpringBootApplication;
+
+@SpringBootApplication
+public class WalletServiceApplication {
+
+  public static void main(String[] args) {
+    SpringApplication.run(WalletServiceApplication.class, args);
+  }
+}


## `build(storage): add JPA PostgreSQL and Flyway`

diff --git a/pom.xml b/pom.xml
index c5be870..738ad12 100644
--- a/pom.xml
+++ b/pom.xml
@@ -32,7 +32,24 @@
         </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
-            <artifactId>spring-boot-starter</artifactId>
+            <artifactId>spring-boot-starter-web</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-validation</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-data-jpa</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.postgresql</groupId>
+            <artifactId>postgresql</artifactId>
+            <scope>runtime</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.flywaydb</groupId>
+            <artifactId>flyway-core</artifactId>
         </dependency>
     </dependencies>
 


## `build(messaging): add Redis Kafka and Avro`

diff --git a/pom.xml b/pom.xml
index 738ad12..a516991 100644
--- a/pom.xml
+++ b/pom.xml
@@ -22,6 +22,7 @@
         <maven.compiler.release>17</maven.compiler.release>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
+        <avro.version>1.12.0</avro.version>
     </properties>
 
     <dependencies>
@@ -51,6 +52,19 @@
             <groupId>org.flywaydb</groupId>
             <artifactId>flyway-core</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-data-redis</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.kafka</groupId>
+            <artifactId>spring-kafka</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.apache.avro</groupId>
+            <artifactId>avro</artifactId>
+            <version>${avro.version}</version>
+        </dependency>
     </dependencies>
 
     <build>


## `build(test): add the containerized test stack`

diff --git a/pom.xml b/pom.xml
index 6fbfd5b..c67686d 100644
--- a/pom.xml
+++ b/pom.xml
@@ -23,8 +23,21 @@
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
         <avro.version>1.12.0</avro.version>
+        <testcontainers.version>1.20.3</testcontainers.version>
     </properties>
 
+    <dependencyManagement>
+        <dependencies>
+            <dependency>
+                <groupId>org.testcontainers</groupId>
+                <artifactId>testcontainers-bom</artifactId>
+                <version>${testcontainers.version}</version>
+                <type>pom</type>
+                <scope>import</scope>
+            </dependency>
+        </dependencies>
+    </dependencyManagement>
+
     <dependencies>
         <dependency>
             <groupId>com.sportsbook</groupId>
@@ -86,6 +99,31 @@
             <groupId>io.micrometer</groupId>
             <artifactId>micrometer-registry-prometheus</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.kafka</groupId>
+            <artifactId>spring-kafka-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.testcontainers</groupId>
+            <artifactId>junit-jupiter</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.testcontainers</groupId>
+            <artifactId>postgresql</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.testcontainers</groupId>
+            <artifactId>kafka</artifactId>
+            <scope>test</scope>
+        </dependency>
     </dependencies>
 
     <build>


## `test(gate): tag wallet persistence semantics`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 9f1ba3e..c5623e8 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -58,6 +58,7 @@ import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.atomic.AtomicBoolean;
 import java.util.function.Supplier;
+import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
@@ -75,6 +76,7 @@ import org.testcontainers.containers.PostgreSQLContainer;
 import org.testcontainers.junit.jupiter.Container;
 import org.testcontainers.junit.jupiter.Testcontainers;
 
+@Tag("wallet-semantic-gate")
 @DataJpaTest(properties = "spring.test.database.replace=NONE")
 @Testcontainers
 @Transactional(propagation = Propagation.NOT_SUPPORTED)


## `test(gate): tag recovery and outbox semantics`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
index 8a7f10c..e37bffd 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
@@ -9,6 +9,7 @@ import com.sportsbook.wallet.outbox.OutboxRetryPolicy;
 import java.time.Duration;
 import java.time.Instant;
 import java.util.concurrent.CompletableFuture;
+import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
@@ -17,6 +18,7 @@ import org.springframework.transaction.annotation.Propagation;
 import org.springframework.transaction.annotation.Transactional;
 import org.testcontainers.junit.jupiter.Testcontainers;
 
+@Tag("wallet-semantic-gate")
 @DataJpaTest(properties = "spring.test.database.replace=NONE")
 @Testcontainers
 @Transactional(propagation = Propagation.NOT_SUPPORTED)
diff --git a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
index df97d75..33deb4f 100644
--- a/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/RecoveryWorkerPersistenceTest.java
@@ -22,6 +22,7 @@ import java.util.UUID;
 import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.CountDownLatch;
 import java.util.concurrent.atomic.AtomicBoolean;
+import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.context.SpringBootTest;
@@ -33,6 +34,7 @@ import org.testcontainers.containers.PostgreSQLContainer;
 import org.testcontainers.junit.jupiter.Container;
 import org.testcontainers.junit.jupiter.Testcontainers;
 
+@Tag("wallet-semantic-gate")
 @SpringBootTest(
     properties = {
       "wallet.integrity.scheduling-enabled=false",


## `test(gate): tag wallet authorization semantics`

diff --git a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
index 7383f5b..a21d37a 100644
--- a/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
+++ b/src/test/java/com/sportsbook/wallet/security/WalletSecurityConfigTest.java
@@ -11,6 +11,7 @@ import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.
 import com.sportsbook.wallet.domain.WalletCaller;
 import java.util.Set;
 import java.util.stream.Stream;
+import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 import org.junit.jupiter.params.ParameterizedTest;
 import org.junit.jupiter.params.provider.Arguments;
@@ -37,6 +38,7 @@ import org.springframework.web.bind.annotation.RequestMapping;
 import org.springframework.web.bind.annotation.RequestMethod;
 import org.springframework.web.bind.annotation.RestController;
 
+@Tag("wallet-semantic-gate")
 @WebMvcTest(controllers = WalletSecurityConfigTest.TestEndpoints.class)
 @Import({WalletSecurityConfig.class, WalletSecurityConfigTest.TestEndpoints.class})
 class WalletSecurityConfigTest {
diff --git a/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java b/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
index 79c5532..2588b96 100644
--- a/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/WalletCreditAuthorizationTest.java
@@ -13,8 +13,10 @@ import com.sportsbook.wallet.service.command.CreditCommand;
 import com.sportsbook.wallet.service.command.CreditReason;
 import java.util.List;
 import java.util.UUID;
+import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 
+@Tag("wallet-semantic-gate")
 class WalletCreditAuthorizationTest {
 
   private static final UUID USER = UUID.fromString("019b76da-a000-7000-8000-000000000027");


## `build(test): align Kafka container archives`

diff --git a/pom.xml b/pom.xml
index eaa092a..e2ab963 100644
--- a/pom.xml
+++ b/pom.xml
@@ -139,6 +139,23 @@
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-maven-plugin</artifactId>
             </plugin>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-surefire-plugin</artifactId>
+                <version>3.2.5</version>
+                <configuration>
+                    <classpathDependencyExcludes>
+                        <classpathDependencyExclude>org.apache.commons:commons-lang3</classpathDependencyExclude>
+                    </classpathDependencyExcludes>
+                    <additionalClasspathDependencies>
+                        <additionalClasspathDependency>
+                            <groupId>org.apache.commons</groupId>
+                            <artifactId>commons-lang3</artifactId>
+                            <version>3.14.0</version>
+                        </additionalClasspathDependency>
+                    </additionalClasspathDependencies>
+                </configuration>
+            </plugin>
             <plugin>
                 <groupId>com.diffplug.spotless</groupId>
                 <artifactId>spotless-maven-plugin</artifactId>


## `test(gate): provision live wallet dependencies`

diff --git a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeFixture.java b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeFixture.java
new file mode 100644
index 0000000..8231a89
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeFixture.java
@@ -0,0 +1,69 @@
+package com.sportsbook.wallet.smoke;
+
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.security.TestInternalApiKeys;
+import com.sportsbook.wallet.web.WalletRequestHeaders;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.http.HttpEntity;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.test.context.DynamicPropertyRegistry;
+import org.springframework.test.context.DynamicPropertySource;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.containers.PostgreSQLContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+import org.testcontainers.kafka.KafkaContainer;
+import org.testcontainers.utility.DockerImageName;
+
+@Testcontainers
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
+abstract class WalletSmokeFixture {
+
+  @Container
+  static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
+
+  @Container
+  static final GenericContainer<?> REDIS =
+      new GenericContainer<>(DockerImageName.parse("redis:7-alpine")).withExposedPorts(6379);
+
+  @Container
+  static final KafkaContainer KAFKA =
+      new KafkaContainer(DockerImageName.parse("apache/kafka:3.8.0"));
+
+  @Autowired TestRestTemplate http;
+
+  @DynamicPropertySource
+  static void runtimeProperties(DynamicPropertyRegistry registry) {
+    registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
+    registry.add("spring.datasource.username", POSTGRES::getUsername);
+    registry.add("spring.datasource.password", POSTGRES::getPassword);
+    registry.add("spring.data.redis.host", REDIS::getHost);
+    registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));
+    registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
+    registry.add("wallet.outbox.scheduling-enabled", () -> "false");
+    registry.add("wallet.recovery.scheduling-enabled", () -> "false");
+    registry.add("wallet.integrity.scheduling-enabled", () -> "false");
+    TestInternalApiKeys.register(registry);
+  }
+
+  ResponseEntity<String> request(
+      HttpMethod method, String path, WalletCaller caller, String operationKey, String body) {
+    HttpHeaders headers = new HttpHeaders();
+    if (caller != null) {
+      headers.set("X-Internal-Service", caller.wireName());
+      headers.set("X-Internal-Api-Key", TestInternalApiKeys.key(caller));
+    }
+    if (operationKey != null) {
+      headers.set(WalletRequestHeaders.IDEMPOTENCY_KEY, operationKey);
+    }
+    if (body != null) {
+      headers.setContentType(MediaType.APPLICATION_JSON);
+    }
+    return http.exchange(path, method, new HttpEntity<>(body, headers), String.class);
+  }
+}


## `test(gate): verify authenticated durable replay`

diff --git a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
new file mode 100644
index 0000000..1ca3b9c
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.wallet.smoke;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.wallet.domain.WalletCaller;
+import java.util.UUID;
+import org.junit.jupiter.api.Tag;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.HttpStatus;
+import org.springframework.jdbc.core.JdbcTemplate;
+
+@Tag("wallet-semantic-gate")
+class WalletSmokeTest extends WalletSmokeFixture {
+  private static final UUID USER_ID = UUID.fromString("019b783d-1000-7000-8000-000000000001");
+  private static final String DEPOSIT_KEY = "smoke:deposit:00000001";
+  private static final String ACCOUNT_PATH = "/internal/v1/wallet/accounts";
+
+  @Autowired ObjectMapper json;
+  @Autowired JdbcTemplate jdbc;
+  @Autowired StringRedisTemplate redis;
+
+  @Test
+  void servesAuthenticatedDurableReplayAcrossPostgresAndRedis() throws Exception {
+    var health = request(HttpMethod.GET, "/actuator/health", null, null, null);
+    var unauthenticated =
+        request(HttpMethod.GET, ACCOUNT_PATH + "/" + USER_ID + "/balance", null, null, null);
+
+    assertThat(health.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(unauthenticated.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
+
+    String account = "{\"userId\":\"" + USER_ID + "\",\"currency\":\"KRW\"}";
+    var opened = request(HttpMethod.POST, ACCOUNT_PATH, WalletCaller.PLATFORM, null, account);
+    assertThat(opened.getStatusCode()).isEqualTo(HttpStatus.OK);
+
+    String deposit =
+        "{\"userId\":\"" + USER_ID + "\",\"amount\":{\"amount\":700,\"currency\":\"KRW\"}}";
+    var first =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/deposit",
+            WalletCaller.PLATFORM,
+            DEPOSIT_KEY,
+            deposit);
+    var replay =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/deposit",
+            WalletCaller.PLATFORM,
+            DEPOSIT_KEY,
+            deposit);
+
+    assertThat(first.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(replay.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(replay.getBody()).isEqualTo(first.getBody());
+    assertThat(json.readTree(first.getBody()).path("reason").textValue()).isEqualTo("DEPOSIT");
+
+    var balance =
+        request(
+            HttpMethod.GET,
+            ACCOUNT_PATH + "/" + USER_ID + "/balance",
+            WalletCaller.GATEWAY,
+            null,
+            null);
+    assertThat(balance.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(json.readTree(balance.getBody()).at("/available/amount").longValue())
+        .isEqualTo(700L);
+    assertThat(count("wallet_operation", DEPOSIT_KEY)).isEqualTo(1);
+    assertThat(count("ledger_entry", DEPOSIT_KEY)).isEqualTo(2);
+    assertThat(redis.opsForValue().get("idempotency:wallet:" + DEPOSIT_KEY)).isEqualTo("1");
+  }
+
+  private int count(String table, String key) {
+    return jdbc.queryForObject(
+        "SELECT COUNT(*) FROM " + table + " WHERE idempotency_key=?", Integer.class, key);
+  }
+}


## `test(gate): publish through a real Kafka broker`

diff --git a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
index 1ca3b9c..0c6cf98 100644
--- a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
+++ b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
@@ -4,7 +4,16 @@ import static org.assertj.core.api.Assertions.assertThat;
 
 import com.fasterxml.jackson.databind.ObjectMapper;
 import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.outbox.KafkaOutboxDispatcher;
+import com.sportsbook.wallet.outbox.WalletEventFactory;
+import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.List;
 import java.util.UUID;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
 import org.junit.jupiter.api.Tag;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
@@ -12,6 +21,8 @@ import org.springframework.data.redis.core.StringRedisTemplate;
 import org.springframework.http.HttpMethod;
 import org.springframework.http.HttpStatus;
 import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
 
 @Tag("wallet-semantic-gate")
 class WalletSmokeTest extends WalletSmokeFixture {
@@ -22,6 +33,8 @@ class WalletSmokeTest extends WalletSmokeFixture {
   @Autowired ObjectMapper json;
   @Autowired JdbcTemplate jdbc;
   @Autowired StringRedisTemplate redis;
+  @Autowired OutboxDeliveryRepository outbox;
+  @Autowired KafkaOutboxDispatcher dispatcher;
 
   @Test
   void servesAuthenticatedDurableReplayAcrossPostgresAndRedis() throws Exception {
@@ -73,8 +86,75 @@ class WalletSmokeTest extends WalletSmokeFixture {
     assertThat(redis.opsForValue().get("idempotency:wallet:" + DEPOSIT_KEY)).isEqualTo("1");
   }
 
+  @Test
+  void publishesCanonicalDebitsThroughKafkaBeforeFencedCompletion() throws Exception {
+    UUID userId = UUID.fromString("019b783d-1000-7000-8000-000000000002");
+    String betId = "019b783d-1000-7000-8000-000000000003";
+    String account = "{\"userId\":\"" + userId + "\",\"currency\":\"KRW\"}";
+    String funds = transaction(userId, 500L);
+    String debit = transaction(userId, 200L);
+    assertThat(
+            request(HttpMethod.POST, ACCOUNT_PATH, WalletCaller.PLATFORM, null, account)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.OK);
+    assertThat(
+            request(
+                    HttpMethod.POST,
+                    "/internal/v1/wallet/transactions/deposit",
+                    WalletCaller.PLATFORM,
+                    "smoke:kafka:deposit",
+                    funds)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.OK);
+    assertThat(
+            request(
+                    HttpMethod.POST,
+                    "/internal/v1/wallet/transactions/debit",
+                    WalletCaller.BETTING,
+                    betId,
+                    debit)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.OK);
+    var message = outbox.claim("smoke-kafka", 1, Duration.ofSeconds(30)).get(0);
+    var consumerFactory =
+        new DefaultKafkaConsumerFactory<String, byte[]>(
+            KafkaTestUtils.consumerProps(KAFKA.getBootstrapServers(), "wallet-smoke", "false"),
+            new StringDeserializer(),
+            new ByteArrayDeserializer());
+    try (var consumer = consumerFactory.createConsumer()) {
+      consumer.subscribe(List.of(WalletEventFactory.DEBITED_TOPIC));
+      dispatcher.dispatch(message).toCompletableFuture().get(10, TimeUnit.SECONDS);
+      var record =
+          KafkaTestUtils.getSingleRecord(
+              consumer, WalletEventFactory.DEBITED_TOPIC, Duration.ofSeconds(10));
+      assertThat(record.key()).isEqualTo(userId.toString());
+      assertThat(record.value()).containsExactly(message.payload());
+      assertThat(record.headers().headers(KafkaOutboxDispatcher.EVENT_ID_HEADER))
+          .extracting(header -> new String(header.value(), StandardCharsets.US_ASCII))
+          .containsExactly(message.lease().eventId().toString());
+      assertThat(outbox.markPublished(message.lease())).isTrue();
+      assertThat(
+              jdbc.queryForObject(
+                  """
+                  SELECT published_at IS NOT NULL AND lease_owner IS NULL AND lease_until IS NULL
+                  FROM outbox_event WHERE event_id=?
+                  """,
+                  Boolean.class,
+                  message.lease().eventId()))
+          .isTrue();
+    }
+  }
+
   private int count(String table, String key) {
     return jdbc.queryForObject(
         "SELECT COUNT(*) FROM " + table + " WHERE idempotency_key=?", Integer.class, key);
   }
+
+  private String transaction(UUID userId, long amount) {
+    return "{\"userId\":\""
+        + userId
+        + "\",\"amount\":{\"amount\":"
+        + amount
+        + ",\"currency\":\"KRW\"}}";
+  }
 }


