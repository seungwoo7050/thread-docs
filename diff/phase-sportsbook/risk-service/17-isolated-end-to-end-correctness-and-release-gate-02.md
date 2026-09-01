## `ci(risk): verify Java 17 correctness gates`

diff --git a/.github/workflows/verify.yml b/.github/workflows/verify.yml
new file mode 100644
index 0000000..44379ed
--- /dev/null
+++ b/.github/workflows/verify.yml
@@ -0,0 +1,45 @@
+name: verify
+
+on:
+  pull_request:
+  push:
+    branches: [risk-service]
+
+permissions:
+  contents: read
+
+jobs:
+  java-17:
+    runs-on: ubuntu-latest
+    env:
+      RISK_MAVEN_REPO: ${{ runner.temp }}/sportsbook-m2
+    steps:
+      - name: Check out risk service
+        uses: actions/checkout@v4
+
+      - name: Check out shared protocol
+        uses: actions/checkout@v4
+        with:
+          repository: ${{ github.repository }}
+          ref: shared-protocol
+          path: shared-protocol
+
+      - name: Set up Temurin 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+
+      - name: Install shared protocol
+        working-directory: shared-protocol
+        run: ./mvnw -B -Dmaven.repo.local="${RISK_MAVEN_REPO}" clean install
+
+      - name: Verify risk service
+        run: ./mvnw -B -Dmaven.repo.local="${RISK_MAVEN_REPO}" clean verify
+
+      - name: Run risk correctness gate
+        run: |
+          export INTERNAL_BETTING_SERVICE_API_KEY=$(openssl rand -hex 32)
+          export INTERNAL_ADMIN_API_KEY=$(openssl rand -hex 32)
+          export INTERNAL_PLATFORM_API_KEY=$(openssl rand -hex 32)
+          bash load-test/run-gate.sh


## `build(release): release risk service 1.0.0`

diff --git a/pom.xml b/pom.xml
index fe980f3..f28eea7 100644
--- a/pom.xml
+++ b/pom.xml
@@ -6,7 +6,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>risk-service</artifactId>
-    <version>1.0.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <packaging>jar</packaging>
 
     <name>risk-service</name>
