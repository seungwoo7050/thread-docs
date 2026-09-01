## `build(deps): add integration test support`

diff --git a/pom.xml b/pom.xml
index 1d5a822..1840e99 100644
--- a/pom.xml
+++ b/pom.xml
@@ -25,6 +25,9 @@
         <bucket4j.version>8.9.0</bucket4j.version>
         <avro.version>1.12.0</avro.version>
         <logstash.version>7.4</logstash.version>
+        <testcontainers.version>1.20.3</testcontainers.version>
+        <wiremock.version>3.9.2</wiremock.version>
+        <surefire.excludedGroups>load</surefire.excludedGroups>
     </properties>
 
     <dependencyManagement>
@@ -36,6 +39,13 @@
                 <type>pom</type>
                 <scope>import</scope>
             </dependency>
+            <dependency>
+                <groupId>org.testcontainers</groupId>
+                <artifactId>testcontainers-bom</artifactId>
+                <version>${testcontainers.version}</version>
+                <type>pom</type>
+                <scope>import</scope>
+            </dependency>
         </dependencies>
     </dependencyManagement>
 
@@ -109,6 +119,42 @@
             <groupId>io.micrometer</groupId>
             <artifactId>micrometer-registry-prometheus</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-test</artifactId>
+            <scope>test</scope>
+        </dependency>
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
+            <artifactId>testcontainers</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.wiremock</groupId>
+            <artifactId>wiremock-standalone</artifactId>
+            <version>${wiremock.version}</version>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.awaitility</groupId>
+            <artifactId>awaitility</artifactId>
+            <scope>test</scope>
+        </dependency>
     </dependencies>
 
     <build>
@@ -127,6 +173,13 @@
                     <skip>true</skip>
                 </configuration>
             </plugin>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-surefire-plugin</artifactId>
+                <configuration>
+                    <excludedGroups>${surefire.excludedGroups}</excludedGroups>
+                </configuration>
+            </plugin>
         </plugins>
     </build>
 </project>


## `build(format): enforce Java formatting`

diff --git a/pom.xml b/pom.xml
index 1840e99..58247e5 100644
--- a/pom.xml
+++ b/pom.xml
@@ -28,6 +28,7 @@
         <testcontainers.version>1.20.3</testcontainers.version>
         <wiremock.version>3.9.2</wiremock.version>
         <surefire.excludedGroups>load</surefire.excludedGroups>
+        <spotless.version>2.43.0</spotless.version>
     </properties>
 
     <dependencyManagement>
@@ -180,6 +181,31 @@
                     <excludedGroups>${surefire.excludedGroups}</excludedGroups>
                 </configuration>
             </plugin>
+            <plugin>
+                <groupId>com.diffplug.spotless</groupId>
+                <artifactId>spotless-maven-plugin</artifactId>
+                <version>${spotless.version}</version>
+                <configuration>
+                    <java>
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
+                        <id>format-check</id>
+                        <phase>verify</phase>
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
         </plugins>
     </build>
 </project>


## `build(checkstyle): enforce static analysis`

diff --git a/config/checkstyle/checkstyle.xml b/config/checkstyle/checkstyle.xml
new file mode 100644
index 0000000..9f34d39
--- /dev/null
+++ b/config/checkstyle/checkstyle.xml
@@ -0,0 +1,22 @@
+<?xml version="1.0"?>
+<!DOCTYPE module PUBLIC
+    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
+    "https://checkstyle.org/dtds/configuration_1_3.dtd">
+
+<module name="Checker">
+    <property name="charset" value="UTF-8"/>
+    <property name="severity" value="error"/>
+
+    <module name="SuppressWarningsFilter"/>
+
+    <module name="TreeWalker">
+        <module name="SuppressWarningsHolder"/>
+        <module name="AvoidStarImport"/>
+        <module name="UnusedImports"/>
+        <module name="RedundantImport"/>
+        <module name="EmptyBlock">
+            <property name="option" value="text"/>
+        </module>
+        <module name="HideUtilityClassConstructor"/>
+    </module>
+</module>
diff --git a/pom.xml b/pom.xml
index 58247e5..8b7762b 100644
--- a/pom.xml
+++ b/pom.xml
@@ -29,6 +29,8 @@
         <wiremock.version>3.9.2</wiremock.version>
         <surefire.excludedGroups>load</surefire.excludedGroups>
         <spotless.version>2.43.0</spotless.version>
+        <checkstyle.plugin.version>3.5.0</checkstyle.plugin.version>
+        <checkstyle.version>10.18.2</checkstyle.version>
     </properties>
 
     <dependencyManagement>
@@ -206,6 +208,33 @@
                     </execution>
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
+                </configuration>
+                <executions>
+                    <execution>
+                        <id>static-analysis</id>
+                        <phase>verify</phase>
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
         </plugins>
     </build>
 </project>


## `feat(application): start API gateway`

diff --git a/pom.xml b/pom.xml
index 8b7762b..2e98680 100644
--- a/pom.xml
+++ b/pom.xml
@@ -172,9 +172,6 @@
             <plugin>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-maven-plugin</artifactId>
-                <configuration>
-                    <skip>true</skip>
-                </configuration>
             </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
diff --git a/src/main/java/com/sportsbook/gateway/GatewayApplication.java b/src/main/java/com/sportsbook/gateway/GatewayApplication.java
new file mode 100644
index 0000000..8744054
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/GatewayApplication.java
@@ -0,0 +1,15 @@
+package com.sportsbook.gateway;
+
+import org.springframework.boot.SpringApplication;
+import org.springframework.boot.autoconfigure.SpringBootApplication;
+import org.springframework.boot.context.properties.ConfigurationPropertiesScan;
+
+@SuppressWarnings("checkstyle:HideUtilityClassConstructor")
+@SpringBootApplication
+@ConfigurationPropertiesScan
+public class GatewayApplication {
+
+  public static void main(String[] args) {
+    SpringApplication.run(GatewayApplication.class, args);
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
new file mode 100644
index 0000000..d936d86
--- /dev/null
+++ b/src/main/resources/application.yml
@@ -0,0 +1,9 @@
+spring:
+  application:
+    name: gateway
+  lifecycle:
+    timeout-per-shutdown-phase: 20s
+
+server:
+  port: ${GATEWAY_HTTP_PORT:8080}
+  shutdown: graceful


## `test(application): verify gateway entry point`

diff --git a/src/test/java/com/sportsbook/gateway/GatewayApplicationTest.java b/src/test/java/com/sportsbook/gateway/GatewayApplicationTest.java
new file mode 100644
index 0000000..c5097b2
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/GatewayApplicationTest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.gateway;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.reflect.Method;
+import java.lang.reflect.Modifier;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.test.context.SpringBootTest;
+
+@SpringBootTest(
+    properties = {"spring.main.web-application-type=none", "management.tracing.enabled=false"})
+class GatewayApplicationTest {
+
+  @Test
+  void loadsApplicationContext() {}
+
+  @Test
+  void exposesPublicMainEntryPoint() throws NoSuchMethodException {
+    Method main = GatewayApplication.class.getMethod("main", String[].class);
+
+    assertThat(Modifier.isPublic(main.getModifiers())).isTrue();
+    assertThat(Modifier.isStatic(main.getModifiers())).isTrue();
+  }
+}
diff --git a/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
new file mode 100644
index 0000000..fdbd0b1
--- /dev/null
+++ b/src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
@@ -0,0 +1 @@
+mock-maker-subclass


## `ci(gateway): verify Java 17 builds`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..4a3da34
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,50 @@
+name: Gateway CI
+
+on:
+  push:
+    branches:
+      - gateway
+  pull_request:
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    steps:
+      - name: Check out gateway event revision
+        uses: actions/checkout@v4
+        with:
+          repository: ${{ github.repository }}
+          ref: ${{ github.sha }}
+          path: gateway
+          persist-credentials: false
+          fetch-depth: 1
+
+      - name: Check out shared protocol
+        uses: actions/checkout@v4
+        with:
+          repository: ${{ github.repository }}
+          ref: shared-protocol
+          path: shared-protocol
+          persist-credentials: false
+          fetch-depth: 1
+
+      - name: Set up Temurin 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+
+      - name: Verify Docker availability
+        run: docker info
+
+      - name: Install shared protocol
+        working-directory: shared-protocol
+        run: ./mvnw -B -ntp -Dmaven.repo.local="${{ runner.temp }}/gateway-m2" clean install
+
+      - name: Verify gateway
+        working-directory: gateway
+        run: ./mvnw -B -ntp -Dmaven.repo.local="${{ runner.temp }}/gateway-m2" clean verify


## `build(release): release gateway 1.0.0`

diff --git a/pom.xml b/pom.xml
index df2785b..a5f2694 100644
--- a/pom.xml
+++ b/pom.xml
@@ -13,7 +13,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>gateway</artifactId>
-    <version>1.0.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <name>gateway</name>
     <description>Public HTTP and WebSocket boundary for the sportsbook platform.</description>
 
