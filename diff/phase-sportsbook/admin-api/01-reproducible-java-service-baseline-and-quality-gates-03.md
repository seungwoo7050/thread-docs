## `test(maven): verify wrapper distribution`

diff --git a/src/test/java/com/sportsbook/admin/MavenWrapperTest.java b/src/test/java/com/sportsbook/admin/MavenWrapperTest.java
new file mode 100644
index 0000000..1dbd8ec
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/MavenWrapperTest.java
@@ -0,0 +1,39 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.Properties;
+import org.junit.jupiter.api.Test;
+
+class MavenWrapperTest {
+
+  private static final String MAVEN_3_9_11 =
+      "https://repo.maven.apache.org/maven2/org/apache/maven/"
+          + "apache-maven/3.9.11/apache-maven-3.9.11-bin.zip";
+
+  @Test
+  void pinsTheApprovedMavenDistribution() throws IOException {
+    Properties properties = new Properties();
+    try (var reader = Files.newBufferedReader(Path.of(".mvn/wrapper/maven-wrapper.properties"))) {
+      properties.load(reader);
+    }
+
+    assertThat(properties)
+        .containsEntry("wrapperVersion", "3.3.4")
+        .containsEntry("distributionType", "only-script")
+        .containsEntry("distributionUrl", MAVEN_3_9_11);
+  }
+
+  @Test
+  void shipsExecutableUnixAndWindowsLaunchers() {
+    Path unixLauncher = Path.of("mvnw");
+    Path windowsLauncher = Path.of("mvnw.cmd");
+
+    assertThat(unixLauncher).isRegularFile();
+    assertThat(Files.isExecutable(unixLauncher)).isTrue();
+    assertThat(windowsLauncher).isRegularFile();
+  }
+}


## `build(format): enforce Google Java Format`

diff --git a/pom.xml b/pom.xml
index fc0e9d6..c7fa94f 100644
--- a/pom.xml
+++ b/pom.xml
@@ -55,6 +55,8 @@
              tests do not collide with the app's Jetty / Jackson versions. -->
         <wiremock.version>3.9.2</wiremock.version>
 
+        <spotless.version>2.43.0</spotless.version>
+
         <!-- @Tag("load") tests are skipped by default; override empty to run them:
              mvn test -DexcludedGroups= -Dtest=*LoadTest -->
         <surefire.excludedGroups>load</surefire.excludedGroups>
@@ -276,6 +278,32 @@
                 </configuration>
             </plugin>
 
+            <plugin>
+                <groupId>com.diffplug.spotless</groupId>
+                <artifactId>spotless-maven-plugin</artifactId>
+                <version>${spotless.version}</version>
+                <configuration>
+                    <java>
+                        <includes>
+                            <include>src/main/java/**/*.java</include>
+                            <include>src/test/java/**/*.java</include>
+                        </includes>
+                        <googleJavaFormat>
+                            <version>1.22.0</version>
+                            <style>GOOGLE</style>
+                        </googleJavaFormat>
+                        <removeUnusedImports/>
+                        <trimTrailingWhitespace/>
+                        <endWithNewline/>
+                    </java>
+                </configuration>
+                <executions>
+                    <execution>
+                        <goals><goal>check</goal></goals>
+                    </execution>
+                </executions>
+            </plugin>
+
         </plugins>
     </build>
 </project>


## `test(format): verify formatter scope`

diff --git a/src/test/java/com/sportsbook/admin/FormattingConfigurationTest.java b/src/test/java/com/sportsbook/admin/FormattingConfigurationTest.java
new file mode 100644
index 0000000..8c9baee
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/FormattingConfigurationTest.java
@@ -0,0 +1,25 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+
+class FormattingConfigurationTest {
+
+  @Test
+  void checksProductionAndTestJavaWithThePinnedFormatter() throws IOException {
+    String pom = Files.readString(Path.of("pom.xml"));
+
+    assertThat(pom)
+        .contains("<artifactId>spotless-maven-plugin</artifactId>")
+        .contains("<version>${spotless.version}</version>")
+        .contains("<include>src/main/java/**/*.java</include>")
+        .contains("<include>src/test/java/**/*.java</include>")
+        .contains("<version>1.22.0</version>")
+        .contains("<style>GOOGLE</style>")
+        .contains("<goal>check</goal>");
+  }
+}


## `build(checkstyle): enforce semantic Java rules`

diff --git a/config/checkstyle/checkstyle.xml b/config/checkstyle/checkstyle.xml
new file mode 100644
index 0000000..7bd8971
--- /dev/null
+++ b/config/checkstyle/checkstyle.xml
@@ -0,0 +1,43 @@
+<?xml version="1.0"?>
+<!DOCTYPE module PUBLIC
+    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
+    "https://checkstyle.org/dtds/configuration_1_3.dtd">
+
+<!--
+    Minimal semantic rule set. Spotless + google-java-format owns layout;
+    Checkstyle owns rules that aren't pure formatting (magic numbers, parameter
+    counts, unused imports, etc.). Grows as real violations show up during dev.
+-->
+<module name="Checker">
+    <property name="severity" value="error"/>
+    <property name="charset" value="UTF-8"/>
+
+    <module name="SuppressWarningsFilter"/>
+
+    <module name="TreeWalker">
+        <module name="SuppressWarningsHolder"/>
+
+        <module name="MagicNumber">
+            <property name="ignoreNumbers" value="-1, 0, 1, 2, 100"/>
+            <property name="ignoreHashCodeMethod" value="true"/>
+            <property name="ignoreAnnotation" value="true"/>
+            <property name="ignoreFieldDeclaration" value="true"/>
+        </module>
+
+        <!-- Records' generated constructors are not methods (no METHOD_DEF
+             token) so this rule only catches regular methods, never record
+             headers. Test fixture builders can exceed via @SuppressWarnings. -->
+        <module name="ParameterNumber">
+            <property name="max" value="10"/>
+            <property name="ignoreOverriddenMethods" value="true"/>
+            <property name="tokens" value="METHOD_DEF"/>
+        </module>
+
+        <module name="UnusedImports"/>
+        <module name="RedundantImport"/>
+        <module name="EmptyBlock">
+            <property name="option" value="text"/>
+        </module>
+        <module name="HideUtilityClassConstructor"/>
+    </module>
+</module>
diff --git a/pom.xml b/pom.xml
index c7fa94f..1d3c6f2 100644
--- a/pom.xml
+++ b/pom.xml
@@ -56,6 +56,8 @@
         <wiremock.version>3.9.2</wiremock.version>
 
         <spotless.version>2.43.0</spotless.version>
+        <checkstyle.plugin.version>3.5.0</checkstyle.plugin.version>
+        <checkstyle.version>10.18.2</checkstyle.version>
 
         <!-- @Tag("load") tests are skipped by default; override empty to run them:
              mvn test -DexcludedGroups= -Dtest=*LoadTest -->
@@ -304,6 +306,35 @@
                 </executions>
             </plugin>
 
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-checkstyle-plugin</artifactId>
+                <version>${checkstyle.plugin.version}</version>
+                <dependencies>
+                    <dependency>
+                        <groupId>com.puppycrawl.tools</groupId>
+                        <artifactId>checkstyle</artifactId>
+                        <version>${checkstyle.version}</version>
+                    </dependency>
+                </dependencies>
+                <configuration>
+                    <configLocation>config/checkstyle/checkstyle.xml</configLocation>
+                    <consoleOutput>true</consoleOutput>
+                    <failOnViolation>true</failOnViolation>
+                    <includeTestSourceDirectory>false</includeTestSourceDirectory>
+                    <sourceDirectories>
+                        <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
+                    </sourceDirectories>
+                </configuration>
+                <executions>
+                    <execution>
+                        <id>checkstyle-validate</id>
+                        <phase>verify</phase>
+                        <goals><goal>check</goal></goals>
+                    </execution>
+                </executions>
+            </plugin>
+
         </plugins>
     </build>
 </project>


## `test(checkstyle): verify semantic rule set`

diff --git a/src/test/java/com/sportsbook/admin/CheckstyleConfigurationTest.java b/src/test/java/com/sportsbook/admin/CheckstyleConfigurationTest.java
new file mode 100644
index 0000000..42efdca
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/CheckstyleConfigurationTest.java
@@ -0,0 +1,29 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+
+class CheckstyleConfigurationTest {
+
+  @Test
+  void enablesTheApprovedSemanticRuleSetForProductionSources() throws IOException {
+    String rules = Files.readString(Path.of("config/checkstyle/checkstyle.xml"));
+    String pom = Files.readString(Path.of("pom.xml"));
+
+    assertThat(rules)
+        .contains("<module name=\"MagicNumber\">")
+        .contains("<module name=\"ParameterNumber\">")
+        .contains("<module name=\"UnusedImports\"/>")
+        .contains("<module name=\"RedundantImport\"/>")
+        .contains("<module name=\"EmptyBlock\">")
+        .contains("<module name=\"HideUtilityClassConstructor\"/>");
+    assertThat(pom)
+        .contains("<artifactId>maven-checkstyle-plugin</artifactId>")
+        .contains("<includeTestSourceDirectory>false</includeTestSourceDirectory>")
+        .contains("<phase>verify</phase>");
+  }
+}


## `feat(app): bootstrap admin API`

diff --git a/src/main/java/com/sportsbook/admin/AdminApiApplication.java b/src/main/java/com/sportsbook/admin/AdminApiApplication.java
new file mode 100644
index 0000000..38267cc
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/AdminApiApplication.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin;
+
+import org.springframework.boot.SpringApplication;
+import org.springframework.boot.autoconfigure.SpringBootApplication;
+import org.springframework.boot.context.properties.ConfigurationPropertiesScan;
+
+/**
+ * admin-api entry point — the operator-facing REST front door (ADR-0011).
+ *
+ * <p>admin-api is a thin authenticated layer with no business logic of its own. It verifies
+ * operator JWTs (RS256, role claim — ADMIN / TRADER / CS / READONLY), enforces an IP allowlist,
+ * then delegates each operation to the owning service's {@code /internal/v1} API over HTTP with a
+ * dedicated caller credential and the required idempotency or action identity. Every action is
+ * recorded in the authoritative local {@code audit_log}; Kafka {@code admin.action} receives a
+ * best-effort copy for streaming consumers (ADR-0007).
+ *
+ * <p>{@code @ConfigurationPropertiesScan} binds the {@code admin.*} property records (downstream
+ * endpoints, security, audit) without an explicit {@code @EnableConfigurationProperties}.
+ */
+// @SpringBootApplication is meta-annotated with @Configuration, so Spring instantiates this class
+// as a bean; a private constructor would break that. Suppress the utility-class rule explicitly.
+@SuppressWarnings("checkstyle:HideUtilityClassConstructor")
+@SpringBootApplication
+@ConfigurationPropertiesScan
+public class AdminApiApplication {
+
+  public static void main(String[] args) {
+    SpringApplication.run(AdminApiApplication.class, args);
+  }
+}


## `test(app): verify application context`

diff --git a/src/test/java/com/sportsbook/admin/AdminApiApplicationTest.java b/src/test/java/com/sportsbook/admin/AdminApiApplicationTest.java
new file mode 100644
index 0000000..18a068a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/AdminApiApplicationTest.java
@@ -0,0 +1,42 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.audit.AuditLogRepository;
+import com.sportsbook.admin.audit.AuditWriteRepository;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.context.ApplicationContext;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+
+@SpringBootTest(
+    webEnvironment = SpringBootTest.WebEnvironment.MOCK,
+    properties = {
+      "spring.autoconfigure.exclude="
+          + "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,"
+          + "org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration",
+      "management.endpoint.health.validate-group-membership=false",
+      "admin.security.jwt.public-key=test-key",
+      "admin.downstream.credentials.wallet-api-key=wallet-admin-test-key-000000000001",
+      "admin.downstream.credentials.risk-api-key=risk-admin-test-key-00000000000002",
+      "admin.downstream.credentials.odds-feed-api-key=odds-admin-test-key-00000000000003",
+      "admin.downstream.credentials.settlement-api-key=settlement-admin-test-key-000000004"
+    })
+class AdminApiApplicationTest {
+
+  @Autowired private ApplicationContext applicationContext;
+
+  @MockBean private JwtDecoder jwtDecoder;
+
+  @MockBean private AuditLogRepository auditLogs;
+
+  @MockBean private AuditWriteRepository auditWrites;
+
+  @Test
+  void startsTheAdminApplicationContext() {
+    assertThat(applicationContext.getBean(AdminApiApplication.class)).isNotNull();
+  }
+}
diff --git a/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
new file mode 100644
index 0000000..fdbd0b1
--- /dev/null
+++ b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
@@ -0,0 +1 @@
+mock-maker-subclass


## `chore(config): define runtime defaults`

diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
new file mode 100644
index 0000000..906266d
--- /dev/null
+++ b/src/main/resources/application.yml
@@ -0,0 +1,55 @@
+spring:
+  application:
+    name: admin-api
+  datasource:
+    url: ${ADMIN_DB_URL:jdbc:postgresql://localhost:5432/admin}
+    username: ${ADMIN_DB_USER:admin}
+    password: ${ADMIN_DB_PASSWORD:admin}
+    hikari:
+      maximum-pool-size: 10
+      minimum-idle: 2
+      pool-name: admin-hikari
+  jpa:
+    hibernate:
+      ddl-auto: validate
+    open-in-view: false
+    properties:
+      hibernate:
+        jdbc:
+          time_zone: UTC
+  flyway:
+    enabled: true
+    locations: classpath:db/migration
+    baseline-on-migrate: false
+  kafka:
+    bootstrap-servers: ${ADMIN_KAFKA_BOOTSTRAP:localhost:9092}
+
+server:
+  port: ${ADMIN_HTTP_PORT:8090}
+  shutdown: graceful
+
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,info,prometheus,metrics
+  endpoint:
+    health:
+      probes:
+        enabled: true
+      show-details: never
+      show-components: never
+  health:
+    livenessstate:
+      enabled: true
+    readinessstate:
+      enabled: true
+  metrics:
+    tags:
+      service: admin-api
+
+logging:
+  level:
+    root: INFO
+    com.sportsbook.admin: INFO
+    org.hibernate.SQL: WARN


## `test(config): verify runtime binding`

diff --git a/src/test/java/com/sportsbook/admin/ApplicationConfigurationTest.java b/src/test/java/com/sportsbook/admin/ApplicationConfigurationTest.java
new file mode 100644
index 0000000..63ac8dc
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ApplicationConfigurationTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.env.YamlPropertySourceLoader;
+import org.springframework.core.env.MapPropertySource;
+import org.springframework.core.io.ClassPathResource;
+import org.springframework.mock.env.MockEnvironment;
+
+class ApplicationConfigurationTest {
+
+  @Test
+  void bindsRuntimeEnvironmentIntoTheServiceConfiguration() throws IOException {
+    MockEnvironment environment = new MockEnvironment();
+    environment
+        .getPropertySources()
+        .addFirst(
+            new MapPropertySource(
+                "test-runtime",
+                Map.of(
+                    "ADMIN_HTTP_PORT", "9190",
+                    "ADMIN_DB_URL", "jdbc:postgresql://db/admin_test",
+                    "ADMIN_KAFKA_BOOTSTRAP", "kafka:29092")));
+    new YamlPropertySourceLoader()
+        .load("admin", new ClassPathResource("application.yml"))
+        .forEach(environment.getPropertySources()::addLast);
+
+    assertThat(environment.getProperty("spring.application.name")).isEqualTo("admin-api");
+    assertThat(environment.getProperty("server.port", Integer.class)).isEqualTo(9190);
+    assertThat(environment.getProperty("spring.datasource.url"))
+        .isEqualTo("jdbc:postgresql://db/admin_test");
+    assertThat(environment.getProperty("spring.kafka.bootstrap-servers")).isEqualTo("kafka:29092");
+    assertThat(environment.getProperty("spring.jpa.hibernate.ddl-auto")).isEqualTo("validate");
+    assertThat(environment.getProperty("spring.flyway.baseline-on-migrate", Boolean.class))
+        .isFalse();
+  }
+}
