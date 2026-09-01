# 재현 가능한 Java 서비스 기반과 품질 게이트

## `build(maven): establish Java 17 baseline`

diff --git a/pom.xml b/pom.xml
new file mode 100644
index 0000000..fc0e9d6
--- /dev/null
+++ b/pom.xml
@@ -0,0 +1,281 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<project xmlns="http://maven.apache.org/POM/4.0.0"
+         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
+         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
+                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
+    <modelVersion>4.0.0</modelVersion>
+
+    <!--
+        ADR-0015 baseline: Java 17 + Spring Boot 3.2.x + Maven. Inherits from
+        spring-boot-starter-parent so dependency/plugin versions stay aligned
+        with the Boot BOM.
+    -->
+    <parent>
+        <groupId>org.springframework.boot</groupId>
+        <artifactId>spring-boot-starter-parent</artifactId>
+        <version>3.2.11</version>
+        <relativePath/>
+    </parent>
+
+    <groupId>com.sportsbook</groupId>
+    <artifactId>admin-api</artifactId>
+    <version>0.1.0-SNAPSHOT</version>
+    <packaging>jar</packaging>
+
+    <name>admin-api</name>
+    <description>
+        Operator-facing REST entry point for the sportsbook system (ADR-0011).
+        A thin authenticated layer: it verifies operator JWTs (RS256, role
+        claim — ADMIN / TRADER / CS / READONLY), enforces an IP allowlist, then
+        delegates each operation (wallet refund, risk limit administration,
+        market controls, and settlement candidate or revision operations) to
+        the owning service's /internal/v1 API with a dedicated caller credential
+        and the required idempotency or action identity. Every action is
+        durably recorded in a local audit_log table and best-effort copied to
+        Kafka admin.action (ADR-0007). No business logic of its own.
+    </description>
+
+    <properties>
+        <java.version>17</java.version>
+        <maven.compiler.release>17</maven.compiler.release>
+        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
+
+        <!-- Aligned with shared-protocol/pom.xml. admin-api consumes the value
+             objects (Money / Currency / UserId / BetId / EventId / MarketId),
+             ErrorCode + ProblemDetail for RFC 7807, and the Avro toolchain.
+             Version 1.0.0 is the fixed portfolio contract. ADR-0014. -->
+        <shared-protocol.version>1.0.0</shared-protocol.version>
+        <avro.version>1.12.0</avro.version>
+        <avro.plugin.version>1.12.0</avro.plugin.version>
+        <testcontainers.version>1.20.3</testcontainers.version>
+        <!-- Pinned because annotationProcessorPaths does not consult the
+             Spring Boot BOM; keep in sync with the BOM-managed version. -->
+        <lombok.version>1.18.30</lombok.version>
+        <!-- Self-contained (shaded) WireMock so the downstream service stubs in
+             tests do not collide with the app's Jetty / Jackson versions. -->
+        <wiremock.version>3.9.2</wiremock.version>
+
+        <!-- @Tag("load") tests are skipped by default; override empty to run them:
+             mvn test -DexcludedGroups= -Dtest=*LoadTest -->
+        <surefire.excludedGroups>load</surefire.excludedGroups>
+    </properties>
+
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
+    <dependencies>
+        <!-- Cross-repo contracts: Money / Currency / UserId / BetId / EventId /
+             MarketId value objects, ErrorCode + ProblemDetail (RFC 7807), and
+             the Avro runtime. Resolved from mavenLocal during dev. -->
+        <dependency>
+            <groupId>com.sportsbook</groupId>
+            <artifactId>shared-protocol</artifactId>
+            <version>${shared-protocol.version}</version>
+        </dependency>
+
+        <!-- Web: the /admin/v1 REST surface + the RestClient used to delegate to
+             each downstream /internal/v1 API. -->
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-web</artifactId>
+        </dependency>
+        <!-- Security: operator authn/authz. spring-security with the OAuth2
+             resource server for RS256 JWT verification (public key via env var)
+             and @PreAuthorize role guards (ADR-0011). -->
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-security</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-validation</artifactId>
+        </dependency>
+        <!-- AOP: weaves the @Audited aspect that durably records every admin
+             action before best-effort streaming (ADR-0011). Also brings
+             aspectjweaver for the @Around advice. -->
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-aop</artifactId>
+        </dependency>
+
+        <!-- Persistence: ADR-0005 — PostgreSQL 16 + Flyway. admin-api touches no
+             business store; the ONLY table it owns is audit_log (ADR-0011). -->
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
+        </dependency>
+
+        <!-- Messaging: ADR-0006 (Kafka) + ADR-0014 (Avro 1.12.x, no Schema
+             Registry in V1). admin-api PUBLISHES the admin.action audit event
+             (AdminActionRecorded, schema owned locally — it is admin-specific
+             and consumed by no other service in V1). -->
+        <dependency>
+            <groupId>org.springframework.kafka</groupId>
+            <artifactId>spring-kafka</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.apache.avro</groupId>
+            <artifactId>avro</artifactId>
+            <version>${avro.version}</version>
+        </dependency>
+
+        <!-- Observability: ADR-0007. JSON structured logs + OTel exporter +
+             Prometheus scrape via Actuator. -->
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-actuator</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>net.logstash.logback</groupId>
+            <artifactId>logstash-logback-encoder</artifactId>
+            <version>7.4</version>
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
+
+        <!-- Lombok: boilerplate trim alongside Java 17 records (ADR-0015). -->
+        <dependency>
+            <groupId>org.projectlombok</groupId>
+            <artifactId>lombok</artifactId>
+            <optional>true</optional>
+        </dependency>
+
+        <!-- Test stack: ADR-0015 — JUnit5 + AssertJ + Mockito + Testcontainers. -->
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-test</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <!-- spring-security-test: JWT mutators (jwt().authorities(...)) + the
+             SecurityMockMvc post-processors for the authz tests. -->
+        <dependency>
+            <groupId>org.springframework.security</groupId>
+            <artifactId>spring-security-test</artifactId>
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
+        <!-- WireMock: stubs each downstream /internal/v1 endpoint so the
+             delegation + header-propagation tests assert what admin-api sends. -->
+        <dependency>
+            <groupId>org.wiremock</groupId>
+            <artifactId>wiremock-standalone</artifactId>
+            <version>${wiremock.version}</version>
+            <scope>test</scope>
+        </dependency>
+    </dependencies>
+
+    <build>
+        <plugins>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-compiler-plugin</artifactId>
+                <configuration>
+                    <annotationProcessorPaths>
+                        <path>
+                            <groupId>org.projectlombok</groupId>
+                            <artifactId>lombok</artifactId>
+                            <version>${lombok.version}</version>
+                        </path>
+                    </annotationProcessorPaths>
+                </configuration>
+            </plugin>
+
+            <!-- ADR-0014: Avro .avsc -> Java generation for the local
+                 AdminActionRecorded audit event. -->
+            <plugin>
+                <groupId>org.apache.avro</groupId>
+                <artifactId>avro-maven-plugin</artifactId>
+                <version>${avro.plugin.version}</version>
+                <executions>
+                    <execution>
+                        <id>schemas</id>
+                        <phase>generate-sources</phase>
+                        <goals>
+                            <goal>schema</goal>
+                        </goals>
+                        <configuration>
+                            <sourceDirectory>${project.basedir}/src/main/avro</sourceDirectory>
+                            <outputDirectory>${project.build.directory}/generated-sources/avro</outputDirectory>
+                            <stringType>String</stringType>
+                        </configuration>
+                    </execution>
+                </executions>
+            </plugin>
+
+            <!-- Skip @Tag("load") tests in the normal build; run the harness
+                 explicitly with `mvn test -DexcludedGroups= -Dtest=*LoadTest`. -->
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-surefire-plugin</artifactId>
+                <configuration>
+                    <excludedGroups>${surefire.excludedGroups}</excludedGroups>
+                </configuration>
+            </plugin>
+
+            <plugin>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-maven-plugin</artifactId>
+                <configuration>
+                    <excludes>
+                        <exclude>
+                            <groupId>org.projectlombok</groupId>
+                            <artifactId>lombok</artifactId>
+                        </exclude>
+                    </excludes>
+                </configuration>
+            </plugin>
+
+        </plugins>
+    </build>
+</project>


## `test(maven): resolve shared protocol 1.0.0`

diff --git a/src/test/java/com/sportsbook/admin/SharedProtocolDependencyTest.java b/src/test/java/com/sportsbook/admin/SharedProtocolDependencyTest.java
new file mode 100644
index 0000000..6b17073
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/SharedProtocolDependencyTest.java
@@ -0,0 +1,15 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import org.junit.jupiter.api.Test;
+
+class SharedProtocolDependencyTest {
+
+  @Test
+  void resolvesTheReleasedMoneyContract() {
+    assertThat(Money.krw(1_000)).isEqualTo(new Money(1_000, Currency.KRW));
+  }
+}


