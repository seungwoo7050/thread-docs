## `build(quality): configure formatting and static analysis`

diff --git a/config/checkstyle/checkstyle.xml b/config/checkstyle/checkstyle.xml
new file mode 100644
index 0000000..09e091b
--- /dev/null
+++ b/config/checkstyle/checkstyle.xml
@@ -0,0 +1,14 @@
+<?xml version="1.0"?>
+<!DOCTYPE module PUBLIC
+    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
+    "https://checkstyle.org/dtds/configuration_1_3.dtd">
+<module name="Checker">
+    <property name="charset" value="UTF-8"/>
+    <module name="NewlineAtEndOfFile"/>
+    <module name="TreeWalker">
+        <module name="AvoidStarImport"/>
+        <module name="UnusedImports"/>
+        <module name="NeedBraces"/>
+        <module name="OneStatementPerLine"/>
+    </module>
+</module>
diff --git a/pom.xml b/pom.xml
index 60ea069..7da96a0 100644
--- a/pom.xml
+++ b/pom.xml
@@ -24,6 +24,9 @@
         <surefire.version>3.5.1</surefire.version>
         <junit.version>5.11.3</junit.version>
         <assertj.version>3.26.3</assertj.version>
+        <spotless.version>2.43.0</spotless.version>
+        <checkstyle.plugin.version>3.5.0</checkstyle.plugin.version>
+        <checkstyle.version>10.18.2</checkstyle.version>
     </properties>
 
     <dependencies>
@@ -67,6 +70,41 @@
                 <artifactId>maven-surefire-plugin</artifactId>
                 <version>${surefire.version}</version>
             </plugin>
+            <plugin>
+                <groupId>com.diffplug.spotless</groupId>
+                <artifactId>spotless-maven-plugin</artifactId>
+                <version>${spotless.version}</version>
+                <configuration>
+                    <java>
+                        <googleJavaFormat><version>1.22.0</version></googleJavaFormat>
+                        <removeUnusedImports/>
+                        <trimTrailingWhitespace/>
+                        <endWithNewline/>
+                    </java>
+                </configuration>
+                <executions>
+                    <execution><phase>verify</phase><goals><goal>check</goal></goals></execution>
+                </executions>
+            </plugin>
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
+                    <includeTestSourceDirectory>false</includeTestSourceDirectory>
+                </configuration>
+                <executions>
+                    <execution><phase>verify</phase><goals><goal>check</goal></goals></execution>
+                </executions>
+            </plugin>
         </plugins>
     </build>
 </project>


## `ci(build): verify settlement on Java 17`

diff --git a/.github/workflows/settlement-ci.yml b/.github/workflows/settlement-ci.yml
new file mode 100644
index 0000000..c077106
--- /dev/null
+++ b/.github/workflows/settlement-ci.yml
@@ -0,0 +1,42 @@
+name: settlement verify
+
+on:
+  push:
+    branches: [settlement-service]
+  pull_request:
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    timeout-minutes: 30
+    steps:
+      - name: Checkout settlement service
+        uses: actions/checkout@v4
+        with:
+          path: settlement-service
+      - name: Checkout fixed shared protocol
+        uses: actions/checkout@v4
+        with:
+          ref: f9de6bc1e533761ab4bb1454d8d4ab8175cdf001
+          path: shared-protocol
+      - name: Use Java 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+      - name: Install fixed shared protocol
+        working-directory: shared-protocol
+        run: >-
+          ./mvnw -B
+          -Dmaven.repo.local=${{ runner.temp }}/settlement-m2
+          -DskipTests install
+      - name: Verify settlement service
+        working-directory: settlement-service
+        run: >-
+          ./mvnw -B
+          -Dmaven.repo.local=${{ runner.temp }}/settlement-m2
+          clean verify


## `ci(history): guard settlement archive history`

diff --git a/.github/scripts/verify-history.sh b/.github/scripts/verify-history.sh
new file mode 100644
index 0000000..173c382
--- /dev/null
+++ b/.github/scripts/verify-history.sh
@@ -0,0 +1,81 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+cd "$(git rev-parse --show-toplevel)"
+head_commit=$(git rev-parse HEAD)
+
+subject_pattern='^(feat|fix|test|refactor|perf|build|docs|chore|style|ci)\([a-z0-9-]+\): .+'
+forbidden_subject_pattern='devlog|changelog|provenance'
+forbidden_path_pattern='(^|/)(devlog|changelog|provenance)(/|\.|$)'
+failed=0
+
+report() {
+  printf '%s\n' "$1" >&2
+  failed=1
+}
+
+while read -r commit; do
+  subject=$(git show -s --format=%s "$commit")
+  body=$(git show -s --format=%b "$commit")
+  paths=$(git diff-tree --root --no-commit-id --name-only -r "$commit")
+  short=${commit:0:12}
+  has_main=false
+  has_test=false
+
+  while read -r path; do
+    [[ $path == src/main/* ]] && has_main=true
+    [[ $path == src/test/* ]] && has_test=true
+  done <<<"$paths"
+
+  if [[ ! $subject =~ $subject_pattern ]]; then
+    report "$short has a non-conventional subject: $subject"
+  fi
+  if [[ -n $body ]]; then
+    report "$short has a non-empty commit body"
+  fi
+  if [[ $subject =~ $forbidden_subject_pattern ]] \
+    || grep -Eiq "$forbidden_path_pattern" <<<"$paths"; then
+    report "$short contains forbidden development-log material"
+  fi
+  if [[ $has_main == true && $has_test == true ]]; then
+    report "$short mixes production and test files"
+  fi
+
+  churn=$(
+    git show --numstat --format= "$commit" \
+      | awk '$1 ~ /^[0-9]+$/ && $2 ~ /^[0-9]+$/ { total += $1 + $2 } END { print total + 0 }'
+  )
+  if ((churn > 100)); then
+    exception=false
+    if [[ $subject =~ ^build\(wrapper\): ]]; then
+      exception=true
+      while read -r path; do
+        [[ $path == mvnw || $path == mvnw.cmd || $path == .mvn/wrapper/* ]] || exception=false
+      done <<<"$paths"
+    elif [[ $subject =~ ^build\(flyway\): ]]; then
+      exception=true
+      while read -r path; do
+        [[ $path == src/main/resources/db/migration/* ]] || exception=false
+      done <<<"$paths"
+    elif [[ $subject == "docs(project): document settlement service" \
+      && $commit == "$head_commit" ]]; then
+      exception=true
+      while read -r path; do
+        [[ $path == README.md ]] || exception=false
+      done <<<"$paths"
+    fi
+    if [[ $exception == false ]]; then
+      report "$short exceeds the 100-line review gate: $churn lines"
+    fi
+  fi
+done < <(git rev-list --reverse HEAD)
+
+if [[ -n $(git rev-list --min-parents=2 --max-count=1 HEAD) ]]; then
+  report "archive history contains a merge commit"
+fi
+
+if git ls-tree -r --name-only HEAD | grep -Eiq "$forbidden_path_pattern"; then
+  report "final tree contains forbidden development-log material"
+fi
+
+exit "$failed"
diff --git a/.github/workflows/settlement-ci.yml b/.github/workflows/settlement-ci.yml
index c077106..9e13f71 100644
--- a/.github/workflows/settlement-ci.yml
+++ b/.github/workflows/settlement-ci.yml
@@ -18,6 +18,7 @@ jobs:
         uses: actions/checkout@v4
         with:
           path: settlement-service
+          fetch-depth: 0
       - name: Checkout fixed shared protocol
         uses: actions/checkout@v4
         with:
@@ -34,6 +35,9 @@ jobs:
           ./mvnw -B
           -Dmaven.repo.local=${{ runner.temp }}/settlement-m2
           -DskipTests install
+      - name: Verify archive history
+        working-directory: settlement-service
+        run: bash .github/scripts/verify-history.sh
       - name: Verify settlement service
         working-directory: settlement-service
         run: >-


## `test(ci): verify settlement history guard`

diff --git a/src/test/java/com/sportsbook/settlement/SettlementHistoryGuardTest.java b/src/test/java/com/sportsbook/settlement/SettlementHistoryGuardTest.java
new file mode 100644
index 0000000..707502a
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/SettlementHistoryGuardTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.settlement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+
+class SettlementHistoryGuardTest {
+
+  @Test
+  void rejectsArchiveHistoryRuleViolations() throws Exception {
+    String workflow = Files.readString(Path.of(".github/workflows/settlement-ci.yml"));
+    String guard = Files.readString(Path.of(".github/scripts/verify-history.sh"));
+
+    assertThat(workflow).contains("fetch-depth: 0", "bash .github/scripts/verify-history.sh");
+    assertThat(guard)
+        .contains(
+            "subject_pattern=",
+            "non-empty commit body",
+            "forbidden development-log material",
+            "mixes production and test files",
+            "archive history contains a merge commit",
+            "100-line review gate");
+    assertGuardPasses();
+  }
+
+  private static void assertGuardPasses() throws IOException, InterruptedException {
+    Process process =
+        new ProcessBuilder("bash", ".github/scripts/verify-history.sh")
+            .redirectErrorStream(true)
+            .start();
+    assertThat(process.waitFor(1, TimeUnit.MINUTES)).isTrue();
+    String output = new String(process.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
+    assertThat(process.exitValue()).describedAs(output).isZero();
+  }
+}


## `fix(build): package settlement as a Boot JAR`

diff --git a/pom.xml b/pom.xml
index 4dc2e1d..5f2cc75 100644
--- a/pom.xml
+++ b/pom.xml
@@ -123,6 +123,10 @@
 
     <build>
         <plugins>
+            <plugin>
+                <groupId>org.springframework.boot</groupId>
+                <artifactId>spring-boot-maven-plugin</artifactId>
+            </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-compiler-plugin</artifactId>


## `test(build): require executable settlement packaging`

diff --git a/src/test/java/com/sportsbook/settlement/BuildBaselineTest.java b/src/test/java/com/sportsbook/settlement/BuildBaselineTest.java
index 5ca156b..c3fa670 100644
--- a/src/test/java/com/sportsbook/settlement/BuildBaselineTest.java
+++ b/src/test/java/com/sportsbook/settlement/BuildBaselineTest.java
@@ -3,6 +3,9 @@ package com.sportsbook.settlement;
 import static org.assertj.core.api.Assertions.assertThat;
 
 import com.sportsbook.protocol.value.Money;
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
 import org.junit.jupiter.api.Test;
 
 class BuildBaselineTest {
@@ -12,4 +15,10 @@ class BuildBaselineTest {
     assertThat(Runtime.version().feature()).isGreaterThanOrEqualTo(17);
     assertThat(Money.class.getPackageName()).isEqualTo("com.sportsbook.protocol.value");
   }
+
+  @Test
+  void packagesTheServiceAsAnExecutableBootJar() throws IOException {
+    assertThat(Files.readString(Path.of("pom.xml")))
+        .contains("<artifactId>spring-boot-maven-plugin</artifactId>");
+  }
 }


## `build(release): release settlement 1.0.0`

diff --git a/pom.xml b/pom.xml
index 5f2cc75..12f23a3 100644
--- a/pom.xml
+++ b/pom.xml
@@ -13,7 +13,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>settlement-service</artifactId>
-    <version>0.1.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <name>settlement-service</name>
 
     <properties>


