## `build(deps): add protocol dependencies`

diff --git a/pom.xml b/pom.xml
index 091a71d..093f6e7 100644
--- a/pom.xml
+++ b/pom.xml
@@ -16,10 +16,51 @@
     <properties>
         <maven.compiler.release>17</maven.compiler.release>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
+        <spring-boot.version>3.2.11</spring-boot.version>
+        <avro.version>1.12.0</avro.version>
         <compiler.plugin.version>3.13.0</compiler.plugin.version>
+        <surefire.version>3.5.1</surefire.version>
         <source.plugin.version>3.3.1</source.plugin.version>
     </properties>
 
+    <dependencyManagement>
+        <dependencies>
+            <dependency>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-dependencies</artifactId>
+                <version>${spring-boot.version}</version>
+                <type>pom</type>
+                <scope>import</scope>
+            </dependency>
+        </dependencies>
+    </dependencyManagement>
+
+    <dependencies>
+        <dependency>
+            <groupId>com.fasterxml.jackson.core</groupId>
+            <artifactId>jackson-databind</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>com.fasterxml.jackson.datatype</groupId>
+            <artifactId>jackson-datatype-jsr310</artifactId>
+        </dependency>
+        <dependency>
+            <groupId>org.apache.avro</groupId>
+            <artifactId>avro</artifactId>
+            <version>${avro.version}</version>
+        </dependency>
+        <dependency>
+            <groupId>org.junit.jupiter</groupId>
+            <artifactId>junit-jupiter</artifactId>
+            <scope>test</scope>
+        </dependency>
+        <dependency>
+            <groupId>org.assertj</groupId>
+            <artifactId>assertj-core</artifactId>
+            <scope>test</scope>
+        </dependency>
+    </dependencies>
+
     <build>
         <plugins>
             <plugin>
@@ -27,6 +68,11 @@
                 <artifactId>maven-compiler-plugin</artifactId>
                 <version>${compiler.plugin.version}</version>
             </plugin>
+            <plugin>
+                <groupId>org.apache.maven.plugins</groupId>
+                <artifactId>maven-surefire-plugin</artifactId>
+                <version>${surefire.version}</version>
+            </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-source-plugin</artifactId>


## `build(avro): generate protocol records`

diff --git a/pom.xml b/pom.xml
index 093f6e7..2e54430 100644
--- a/pom.xml
+++ b/pom.xml
@@ -21,6 +21,7 @@
         <compiler.plugin.version>3.13.0</compiler.plugin.version>
         <surefire.version>3.5.1</surefire.version>
         <source.plugin.version>3.3.1</source.plugin.version>
+        <avro.plugin.version>1.12.0</avro.plugin.version>
     </properties>
 
     <dependencyManagement>
@@ -73,6 +74,25 @@
                 <artifactId>maven-surefire-plugin</artifactId>
                 <version>${surefire.version}</version>
             </plugin>
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
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-source-plugin</artifactId>
diff --git a/src/main/avro/.gitkeep b/src/main/avro/.gitkeep
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/src/main/avro/.gitkeep
@@ -0,0 +1 @@
+


## `build(format): enforce Java formatting`

diff --git a/pom.xml b/pom.xml
index 2e54430..1415fd3 100644
--- a/pom.xml
+++ b/pom.xml
@@ -22,6 +22,7 @@
         <surefire.version>3.5.1</surefire.version>
         <source.plugin.version>3.3.1</source.plugin.version>
         <avro.plugin.version>1.12.0</avro.plugin.version>
+        <spotless.version>2.43.0</spotless.version>
     </properties>
 
     <dependencyManagement>
@@ -93,6 +94,33 @@
                     </execution>
                 </executions>
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
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-source-plugin</artifactId>


## `build(checkstyle): enforce static analysis`

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
index 1415fd3..9bbfbd7 100644
--- a/pom.xml
+++ b/pom.xml
@@ -23,6 +23,8 @@
         <source.plugin.version>3.3.1</source.plugin.version>
         <avro.plugin.version>1.12.0</avro.plugin.version>
         <spotless.version>2.43.0</spotless.version>
+        <checkstyle.plugin.version>3.5.0</checkstyle.plugin.version>
+        <checkstyle.version>10.18.2</checkstyle.version>
     </properties>
 
     <dependencyManagement>
@@ -121,6 +123,36 @@
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
+                    <sourceDirectories>
+                        <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
+                    </sourceDirectories>
+                </configuration>
+                <executions>
+                    <execution>
+                        <id>checkstyle-validate</id>
+                        <phase>verify</phase>
+                        <goals>
+                            <goal>check</goal>
+                        </goals>
+                    </execution>
+                </executions>
+            </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-source-plugin</artifactId>


## `ci(protocol): verify Java 17 builds`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..31c2d40
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,22 @@
+name: verify
+
+on:
+  push:
+    branches: [shared-protocol]
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
+      - uses: actions/checkout@v4
+      - uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+          cache: maven
+      - run: ./mvnw -B clean verify


## `build(release): release shared protocol 1.0.0`

diff --git a/pom.xml b/pom.xml
index 14f8026..3be36f8 100644
--- a/pom.xml
+++ b/pom.xml
@@ -7,7 +7,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>shared-protocol</artifactId>
-    <version>1.0.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <packaging>jar</packaging>
 
     <name>shared-protocol</name>
