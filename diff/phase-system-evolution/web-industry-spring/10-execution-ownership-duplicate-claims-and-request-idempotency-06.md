## `test(execution): record the bounded E10 repair gate`

diff --git a/evidence/phase-1/E10/repair1/cleanup.json b/evidence/phase-1/E10/repair1/cleanup.json
new file mode 100644
index 0000000..70c77f6
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/cleanup.json
@@ -0,0 +1,70 @@
+{
+  "postgresBefore": {
+    "argv": [
+      "docker",
+      "compose",
+      "--project-name",
+      "wse-industry",
+      "--file",
+      "compose.yaml",
+      "exec",
+      "--no-TTY",
+      "postgres",
+      "psql",
+      "--username",
+      "wse_industry",
+      "--dbname",
+      "monitor",
+      "--tuples-only",
+      "--no-align",
+      "--command",
+      "SELECT coalesce(json_agg(nspname ORDER BY nspname), '[]') FROM pg_namespace WHERE nspname NOT LIKE 'pg_%' AND nspname <> 'information_schema'"
+    ],
+    "exitCode": 0,
+    "nativeSeconds": 0.128686708,
+    "schemas": [
+      "public"
+    ]
+  },
+  "postgresAfter": {
+    "argv": [
+      "docker",
+      "compose",
+      "--project-name",
+      "wse-industry",
+      "--file",
+      "compose.yaml",
+      "exec",
+      "--no-TTY",
+      "postgres",
+      "psql",
+      "--username",
+      "wse_industry",
+      "--dbname",
+      "monitor",
+      "--tuples-only",
+      "--no-align",
+      "--command",
+      "SELECT coalesce(json_agg(nspname ORDER BY nspname), '[]') FROM pg_namespace WHERE nspname NOT LIKE 'pg_%' AND nspname <> 'information_schema'"
+    ],
+    "exitCode": 0,
+    "nativeSeconds": 0.130345,
+    "schemas": [
+      "public"
+    ]
+  },
+  "listenersAfter": {
+    "4321": false,
+    "4322": false,
+    "4323": false,
+    "4324": false,
+    "4325": false
+  },
+  "ownedWorkerPids": [
+    4658,
+    4659
+  ],
+  "allOwnedWorkerExitsAwaited": true,
+  "launcherExitAwaited": true,
+  "publicPreservation": "No operation targeted public data or volumes; the actual test creates/drops only e10_ownership. Schema inventory is observed; no unmeasured public-row equality is claimed."
+}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml b/evidence/phase-1/E10/repair1/gate-artifacts/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml
new file mode 100644
index 0000000..9b675fe
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml
@@ -0,0 +1,142 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.ExecutionOwnershipTest" time="9.056" tests="1" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="catalina.useNaming" value="false"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="org.jboss.logging.provider" value="slf4j"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="APPLICATION_NAME" value="monitor-api"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828143740250_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T14-37-40_199-jvmRun1 surefire-20260828143740250_1tmp surefire_0-20260828143740250_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="ExecutionOwnershipTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="FILE_LOG_CHARSET" value="UTF-8"/>
+    <property name="java.awt.headless" value="true"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828143740250_3.jar"/>
+    <property name="polyglot.engine.WarnInterpreterOnly" value="false"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="catalina.home" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.18421831162643982223"/>
+    <property name="com.zaxxer.hikari.pool_number" value="1"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="PID" value="4624"/>
+    <property name="CONSOLE_LOG_CHARSET" value="UTF-8"/>
+    <property name="catalina.base" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.18421831162643982223"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+    <property name="LOGGED_APPLICATION_NAME" value="[monitor-api] "/>
+  </properties>
+  <testcase name="parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner" classname="dev.evolution.monitor.ExecutionOwnershipTest" time="5.624">
+    <system-out><![CDATA[14:37:40.628 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.ExecutionOwnershipTest]: ExecutionOwnershipTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+14:37:40.718 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.ExecutionOwnershipTest
+
+  .   ____          _            __ _ _
+ /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
+( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
+ \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
+  '  |____| .__|_| |_|_| |_\__, | / / / /
+ =========|_|==============|___/=/_/_/_/
+
+ :: Spring Boot ::               (v3.5.16)
+
+2026-08-28T14:37:41.010+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Starting ExecutionOwnershipTest using Java 21.0.7 with PID 4624 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:37:41.010+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:37:41.301+09:00  INFO 4624 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:37:41.315+09:00  INFO 4624 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:37:41.607+09:00  INFO 4624 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:37:41.617+09:00  INFO 4624 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:37:41.617+09:00  INFO 4624 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:37:41.639+09:00  INFO 4624 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:37:41.639+09:00  INFO 4624 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 620 ms
+2026-08-28T14:37:41.796+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:37:41.813+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@75839695
+2026-08-28T14:37:41.814+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:37:41.832+09:00  INFO 4624 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:37:41.862+09:00  INFO 4624 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e10_ownership"."flyway_schema_history" does not exist yet
+2026-08-28T14:37:41.866+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.014s)
+2026-08-28T14:37:41.884+09:00  INFO 4624 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e10_ownership"."flyway_schema_history" ...
+2026-08-28T14:37:41.921+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": << Empty Schema >>
+2026-08-28T14:37:41.927+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "1 - create monitors"
+2026-08-28T14:37:41.949+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "2 - create check runs"
+2026-08-28T14:37:41.965+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "3 - create users"
+2026-08-28T14:37:41.980+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "4 - require monitor ownership"
+2026-08-28T14:37:41.997+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "5 - index check history"
+2026-08-28T14:37:42.016+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "6 - queue check execution"
+2026-08-28T14:37:42.033+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "7 - execution ownership and manual identity"
+2026-08-28T14:37:42.045+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e10_ownership", now at version v7 (execution time 00:00.037s)
+2026-08-28T14:37:42.112+09:00  INFO 4624 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:37:42.149+09:00  INFO 4624 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:37:42.166+09:00  INFO 4624 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:37:42.312+09:00  INFO 4624 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:37:42.357+09:00  INFO 4624 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:37:42.710+09:00  INFO 4624 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:37:42.735+09:00  INFO 4624 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:42.966+09:00  INFO 4624 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:37:43.484+09:00  INFO 4624 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:37:43.492+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Started ExecutionOwnershipTest in 2.711 seconds (process running for 3.191)
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+]]></system-out>
+    <system-err><![CDATA[Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+]]></system-err>
+  </testcase>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/dev.evolution.monitor.ExecutionOwnershipTest.txt b/evidence/phase-1/E10/repair1/gate-artifacts/dev.evolution.monitor.ExecutionOwnershipTest.txt
new file mode 100644
index 0000000..0eb0b60
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/dev.evolution.monitor.ExecutionOwnershipTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.ExecutionOwnershipTest
+-------------------------------------------------------------------------------
+Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 9.056 s -- in dev.evolution.monitor.ExecutionOwnershipTest
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/e10-ownership.json b/evidence/phase-1/E10/repair1/gate-artifacts/e10-ownership.json
new file mode 100644
index 0000000..c4239cb
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/e10-ownership.json
@@ -0,0 +1,117 @@
+{
+  "fixtureSha256" : "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+  "completed" : [ "same-owner/key parallel requests deduplicated by PostgreSQL after both inserts reached the lock barrier", "changed target conflicts; mutable Monitor fields do not redefine identity; invalid keys cause no write", "two actual workers produced one owner/outbound/result; losing process completion was rejected while RUNNING", "terminal retransmission retains current identity; a different valid key adds one new queued intent" ],
+  "workerEntry" : "two non-web JVMs, test-only startup gates, production CheckWorker.executeNext once",
+  "defaultRequestFactory" : "org.springframework.http.client.JdkClientHttpRequestFactory",
+  "result" : "PASS",
+  "parallel" : {
+    "blockedInsertTransactions" : 2,
+    "sameId" : true,
+    "persistedRows" : 1,
+    "statuses" : [ 202, 202 ],
+    "outboundRequests" : 0
+  },
+  "invalidKeys" : [ {
+    "case" : "missing",
+    "errorCode" : "INVALID_INPUT",
+    "status" : 400,
+    "originalRowUnchanged" : true,
+    "transport" : "default",
+    "persistedRows" : 1,
+    "outboundRequests" : 0
+  }, {
+    "case" : "empty",
+    "errorCode" : "INVALID_INPUT",
+    "status" : 400,
+    "originalRowUnchanged" : true,
+    "transport" : "default",
+    "persistedRows" : 1,
+    "outboundRequests" : 0
+  }, {
+    "case" : "has-space",
+    "errorCode" : "INVALID_INPUT",
+    "status" : 400,
+    "originalRowUnchanged" : true,
+    "transport" : "default",
+    "persistedRows" : 1,
+    "outboundRequests" : 0
+  }, {
+    "case" : "non-ASCII-U+00E9",
+    "errorCode" : "INVALID_INPUT",
+    "status" : 400,
+    "originalRowUnchanged" : true,
+    "transport" : "literal-e9/HTTP1.0",
+    "persistedRows" : 1,
+    "outboundRequests" : 0
+  }, {
+    "case" : "ASCII-length-129",
+    "errorCode" : "INVALID_INPUT",
+    "status" : 400,
+    "originalRowUnchanged" : true,
+    "transport" : "default",
+    "persistedRows" : 1,
+    "outboundRequests" : 0
+  } ],
+  "identityMeaning" : {
+    "persistedRows" : 1,
+    "mutableMonitorReplaySameId" : true,
+    "otherTargetRows" : 0,
+    "invalidInputsRejected" : 5,
+    "otherTargetStatus" : 409
+  },
+  "workersReady" : [ {
+    "processId" : 4658,
+    "ownerId" : "f92c3cc5-5315-429a-a6c1-3b65e96568d6",
+    "checkId" : "c592a461-e36d-450b-8e52-af37a3bb8319"
+  }, {
+    "processId" : 4659,
+    "ownerId" : "da9498df-3a56-4ba8-8e6b-5ccf987ee321",
+    "checkId" : "c592a461-e36d-450b-8e52-af37a3bb8319"
+  } ],
+  "blockedReadyWorkers" : 2,
+  "whileHeld" : {
+    "currentReplayState" : "RUNNING",
+    "ownerAndOutcomeUnchanged" : true,
+    "sameId" : true,
+    "loser" : {
+      "processId" : 4658,
+      "ownerId" : "f92c3cc5-5315-429a-a6c1-3b65e96568d6",
+      "checkId" : "c592a461-e36d-450b-8e52-af37a3bb8319",
+      "wonClaim" : false,
+      "attemptedOwnerId" : "00000000-0000-0000-0000-000000000000",
+      "terminalWriteChangedRow" : false
+    },
+    "persistedRows" : 1,
+    "state" : "RUNNING",
+    "fixtureRequests" : 1
+  },
+  "workersCompleted" : [ {
+    "processId" : 4658,
+    "ownerId" : "f92c3cc5-5315-429a-a6c1-3b65e96568d6",
+    "checkId" : "c592a461-e36d-450b-8e52-af37a3bb8319",
+    "wonClaim" : false,
+    "attemptedOwnerId" : "00000000-0000-0000-0000-000000000000",
+    "terminalWriteChangedRow" : false
+  }, {
+    "processId" : 4659,
+    "ownerId" : "da9498df-3a56-4ba8-8e6b-5ccf987ee321",
+    "checkId" : "c592a461-e36d-450b-8e52-af37a3bb8319",
+    "wonClaim" : true
+  } ],
+  "terminal" : {
+    "rows" : 1,
+    "sameId" : true,
+    "httpStatus" : 200,
+    "state" : "SUCCEEDED",
+    "outboundRequests" : 1
+  },
+  "nextIntent" : {
+    "originalIdentityRetained" : true,
+    "keyLength" : 128,
+    "replaySameId" : true,
+    "newId" : true,
+    "outboundRequests" : 1,
+    "persistedRows" : 2
+  },
+  "allOwnedWorkerExitsAwaited" : true
+}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/one-ready.json b/evidence/phase-1/E10/repair1/gate-artifacts/one-ready.json
new file mode 100644
index 0000000..0696c3c
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/one-ready.json
@@ -0,0 +1 @@
+{"processId":4658,"ownerId":"f92c3cc5-5315-429a-a6c1-3b65e96568d6","checkId":"c592a461-e36d-450b-8e52-af37a3bb8319"}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/one-result.json b/evidence/phase-1/E10/repair1/gate-artifacts/one-result.json
new file mode 100644
index 0000000..20bdf27
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/one-result.json
@@ -0,0 +1 @@
+{"processId":4658,"ownerId":"f92c3cc5-5315-429a-a6c1-3b65e96568d6","checkId":"c592a461-e36d-450b-8e52-af37a3bb8319","wonClaim":false,"attemptedOwnerId":"00000000-0000-0000-0000-000000000000","terminalWriteChangedRow":false}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/one.log b/evidence/phase-1/E10/repair1/gate-artifacts/one.log
new file mode 100644
index 0000000..73842c7
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/one.log
@@ -0,0 +1,29 @@
+2026-08-28T14:37:45.473+09:00  INFO 4658 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : Starting E10WorkerProcess using Java 21.0.7 with PID 4658 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:37:45.476+09:00  INFO 4658 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:37:46.061+09:00  INFO 4658 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:37:46.096+09:00  INFO 4658 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 21 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:37:46.611+09:00  INFO 4658 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:37:46.720+09:00  INFO 4658 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@16a35bd
+2026-08-28T14:37:46.721+09:00  INFO 4658 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:37:46.743+09:00  INFO 4658 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:37:46.790+09:00  INFO 4658 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.025s)
+2026-08-28T14:37:46.822+09:00  INFO 4658 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": 7
+2026-08-28T14:37:46.825+09:00  INFO 4658 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema "e10_ownership" is up to date. No migration necessary.
+2026-08-28T14:37:46.903+09:00  INFO 4658 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:37:46.931+09:00  INFO 4658 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:37:46.948+09:00  INFO 4658 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:37:47.074+09:00  INFO 4658 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:37:47.112+09:00  INFO 4658 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:37:47.515+09:00  INFO 4658 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:37:47.538+09:00  INFO 4658 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:47.754+09:00  INFO 4658 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : Started E10WorkerProcess in 2.589 seconds (process running for 2.797)
+2026-08-28T14:37:49.236+09:00  INFO 4658 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:49.237+09:00  INFO 4658 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
+2026-08-28T14:37:49.239+09:00  INFO 4658 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/two-ready.json b/evidence/phase-1/E10/repair1/gate-artifacts/two-ready.json
new file mode 100644
index 0000000..a30cb38
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/two-ready.json
@@ -0,0 +1 @@
+{"processId":4659,"ownerId":"da9498df-3a56-4ba8-8e6b-5ccf987ee321","checkId":"c592a461-e36d-450b-8e52-af37a3bb8319"}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/two-result.json b/evidence/phase-1/E10/repair1/gate-artifacts/two-result.json
new file mode 100644
index 0000000..cb30c2e
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/two-result.json
@@ -0,0 +1 @@
+{"processId":4659,"ownerId":"da9498df-3a56-4ba8-8e6b-5ccf987ee321","checkId":"c592a461-e36d-450b-8e52-af37a3bb8319","wonClaim":true}
diff --git a/evidence/phase-1/E10/repair1/gate-artifacts/two.log b/evidence/phase-1/E10/repair1/gate-artifacts/two.log
new file mode 100644
index 0000000..7dd1566
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate-artifacts/two.log
@@ -0,0 +1,29 @@
+2026-08-28T14:37:45.490+09:00  INFO 4659 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : Starting E10WorkerProcess using Java 21.0.7 with PID 4659 (/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:37:45.492+09:00  INFO 4659 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:37:46.084+09:00  INFO 4659 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:37:46.120+09:00  INFO 4659 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 23 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:37:46.616+09:00  INFO 4659 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:37:46.720+09:00  INFO 4659 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@4ee25d80
+2026-08-28T14:37:46.721+09:00  INFO 4659 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:37:46.743+09:00  INFO 4659 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:37:46.791+09:00  INFO 4659 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.026s)
+2026-08-28T14:37:47.827+09:00  INFO 4659 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": 7
+2026-08-28T14:37:47.835+09:00  INFO 4659 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema "e10_ownership" is up to date. No migration necessary.
+2026-08-28T14:37:47.963+09:00  INFO 4659 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:37:48.005+09:00  INFO 4659 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:37:48.033+09:00  INFO 4659 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:37:48.231+09:00  INFO 4659 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:37:48.276+09:00  INFO 4659 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:37:48.666+09:00  INFO 4659 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:37:48.684+09:00  INFO 4659 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:48.843+09:00  INFO 4659 --- [monitor-api] [           main] dev.evolution.monitor.E10WorkerProcess   : Started E10WorkerProcess in 3.665 seconds (process running for 3.883)
+2026-08-28T14:37:49.480+09:00  INFO 4659 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:49.481+09:00  INFO 4659 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
+2026-08-28T14:37:49.482+09:00  INFO 4659 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
diff --git a/evidence/phase-1/E10/repair1/gate.log b/evidence/phase-1/E10/repair1/gate.log
new file mode 100644
index 0000000..8fb940f
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/gate.log
@@ -0,0 +1,112 @@
+[INFO] Scanning for projects...
+[INFO] 
+[INFO] ---------------------< dev.evolution:monitor-api >----------------------
+[INFO] Building monitor-api 0.0.1
+[INFO]   from pom.xml
+[INFO] --------------------------------[ jar ]---------------------------------
+[INFO] 
+[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---
+[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed
+[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed
+[INFO] 
+[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---
+[INFO] Copying 1 resource from src/main/resources to target/classes
+[INFO] Copying 7 resources from src/main/resources to target/classes
+[INFO] 
+[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---
+[INFO] Nothing to compile - all classes are up to date.
+[INFO] 
+[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---
+[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources
+[INFO] 
+[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---
+[INFO] Recompiling the module because of changed source code.
+[INFO] Compiling 15 source files with javac [debug parameters release 21] to target/test-classes
+[INFO] 
+[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---
+[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
+[INFO] 
+[INFO] -------------------------------------------------------
+[INFO]  T E S T S
+[INFO] -------------------------------------------------------
+[INFO] Running dev.evolution.monitor.ExecutionOwnershipTest
+14:37:40.628 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.ExecutionOwnershipTest]: ExecutionOwnershipTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+14:37:40.718 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.ExecutionOwnershipTest
+
+  .   ____          _            __ _ _
+ /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
+( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
+ \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
+  '  |____| .__|_| |_|_| |_\__, | / / / /
+ =========|_|==============|___/=/_/_/_/
+
+ :: Spring Boot ::               (v3.5.16)
+
+2026-08-28T14:37:41.010+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Starting ExecutionOwnershipTest using Java 21.0.7 with PID 4624 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:37:41.010+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:37:41.301+09:00  INFO 4624 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:37:41.315+09:00  INFO 4624 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 9 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:37:41.607+09:00  INFO 4624 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:37:41.617+09:00  INFO 4624 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:37:41.617+09:00  INFO 4624 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:37:41.639+09:00  INFO 4624 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:37:41.639+09:00  INFO 4624 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 620 ms
+2026-08-28T14:37:41.796+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:37:41.813+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@75839695
+2026-08-28T14:37:41.814+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:37:41.832+09:00  INFO 4624 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:37:41.862+09:00  INFO 4624 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e10_ownership"."flyway_schema_history" does not exist yet
+2026-08-28T14:37:41.866+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.014s)
+2026-08-28T14:37:41.884+09:00  INFO 4624 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e10_ownership"."flyway_schema_history" ...
+2026-08-28T14:37:41.921+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": << Empty Schema >>
+2026-08-28T14:37:41.927+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "1 - create monitors"
+2026-08-28T14:37:41.949+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "2 - create check runs"
+2026-08-28T14:37:41.965+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "3 - create users"
+2026-08-28T14:37:41.980+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "4 - require monitor ownership"
+2026-08-28T14:37:41.997+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "5 - index check history"
+2026-08-28T14:37:42.016+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "6 - queue check execution"
+2026-08-28T14:37:42.033+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "7 - execution ownership and manual identity"
+2026-08-28T14:37:42.045+09:00  INFO 4624 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e10_ownership", now at version v7 (execution time 00:00.037s)
+2026-08-28T14:37:42.112+09:00  INFO 4624 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:37:42.149+09:00  INFO 4624 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:37:42.166+09:00  INFO 4624 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:37:42.312+09:00  INFO 4624 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:37:42.357+09:00  INFO 4624 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:37:42.710+09:00  INFO 4624 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:37:42.735+09:00  INFO 4624 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:42.966+09:00  INFO 4624 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:37:43.484+09:00  INFO 4624 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:37:43.492+09:00  INFO 4624 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Started ExecutionOwnershipTest in 2.711 seconds (process running for 3.191)
+Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:37:44.396+09:00  INFO 4624 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+2026-08-28T14:37:49.599+09:00  INFO 4624 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T14:37:49.600+09:00  INFO 4624 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T14:37:49.602+09:00  INFO 4624 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:37:49.603+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
+2026-08-28T14:37:49.604+09:00  INFO 4624 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
+[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 9.056 s -- in dev.evolution.monitor.ExecutionOwnershipTest
+[INFO] 
+[INFO] Results:
+[INFO] 
+[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
+[INFO] 
+[INFO] ------------------------------------------------------------------------
+[INFO] BUILD SUCCESS
+[INFO] ------------------------------------------------------------------------
+[INFO] Total time:  12.358 s
+[INFO] Finished at: 2026-08-28T14:37:50+09:00
+[INFO] ------------------------------------------------------------------------
diff --git a/evidence/phase-1/E10/repair1/invocations.jsonl b/evidence/phase-1/E10/repair1/invocations.jsonl
index ad6e1fe..dc2b300 100644
--- a/evidence/phase-1/E10/repair1/invocations.jsonl
+++ b/evidence/phase-1/E10/repair1/invocations.jsonl
@@ -1,2 +1,4 @@
 {"phase":"diagnostic","event":"start","argv":["/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/java","--class-path","/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:","/private/tmp/web-systems-evolution-0a006589-industry-spring/evidence/phase-1/E10/repair1/KeyTransportDiagnostic.java","/private/tmp/web-systems-evolution-0a006589-industry-spring/evidence/phase-1/E10/repair1/diagnostic.json"],"cwd":"/private/tmp/web-systems-evolution-0a006589-industry-spring","startedAt":"2026-08-28T05:33:08.278841+00:00","timeoutSeconds":60,"listenersBefore":{"4324":false}}
 {"phase":"diagnostic","event":"finish","startedAt":"2026-08-28T05:33:08.278841+00:00","finishedAt":"2026-08-28T05:33:10.204682+00:00","elapsedSeconds":1.925166,"exitCode":0,"timedOut":false,"processExitAwaited":true,"listenersAfter":{"4324":false}}
+{"phase":"gate","event":"start","argv":["mvn","-B","-ntp","-f","backend/pom.xml","-Dtest=ExecutionOwnershipTest","test"],"cwd":"/private/tmp/web-systems-evolution-0a006589-industry-spring","startedAt":"2026-08-28T05:37:36.623903+00:00","timeoutSeconds":120,"listenersBefore":{"4321":false,"4322":false,"4323":false,"4324":false,"4325":false}}
+{"phase":"gate","event":"finish","startedAt":"2026-08-28T05:37:36.623903+00:00","finishedAt":"2026-08-28T05:37:50.097536+00:00","elapsedSeconds":13.47297,"exitCode":0,"timedOut":false,"processExitAwaited":true,"listenersAfter":{"4321":false,"4322":false,"4323":false,"4324":false,"4325":false}}
diff --git a/evidence/phase-1/E10/repair1/result.json b/evidence/phase-1/E10/repair1/result.json
new file mode 100644
index 0000000..de0eb60
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/result.json
@@ -0,0 +1,151 @@
+{
+  "recordedAt": "2026-08-28T05:39:49.617504+00:00",
+  "scope": "bounded E10 repair1/2, attempt2; not whole-E10 completion",
+  "branch": "track/industry-spring",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "threadStart": "3cc49f3d2a35055c92d0312fca6167c89dfadec5",
+  "repairStart": "bbacf6924f6a8c6fe87185c5f0b91317ca17b3a1",
+  "testedCandidate": "887ff7842566a7d65618c645a1f9faa6d539aba3",
+  "commitsThroughTestedCandidate": [
+    "bbacf6924f6a8c6fe87185c5f0b91317ca17b3a1",
+    "e299b206f3c56653966a96f479e40d970730ae1e",
+    "3a1adae83fb192337fb9fbc3432a2f5ad13ca2a5",
+    "b9cfba5829db0b3837a1af1df7be9b48e133359e",
+    "887ff7842566a7d65618c645a1f9faa6d539aba3"
+  ],
+  "cause": {
+    "proven": "The selected JdkClientHttpRequestFactory changes the frozen U+00E9 header into wire3f on Temurin21.0.7+6. The real gate reports the same default factory.",
+    "historicalLimit": "The failed attempt did not label the exact invalid input or capture its bytes; that historical omission is retained.",
+    "predeterminedAlternative": "SimpleClientHttpRequestFactory sentc3a9 and was not used as a literal-U+00E9 substitute.",
+    "correction": "Only the frozen U+00E9 request uses a bounded direct e9/HTTP1.0 request to the real authenticated API; production validation is unchanged."
+  },
+  "changedFromPreservedWip": [
+    "backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java",
+    "backend/src/test/java/dev/evolution/monitor/SessionClient.java"
+  ],
+  "otherPreservedWipFilesByteIdentical": 16,
+  "sourceHashes": {
+    "app/monitors/api.ts": "5b10611f09f0e07a4200db4e602fc228cbb0f9ef226893a5a7d94eea48834413",
+    "app/monitors/use-monitor-state.ts": "b9cfb1fcee8262e477d0a479f95312e49a92f4012cbe800961790df8b57170a2",
+    "backend/src/main/java/dev/evolution/monitor/ApiErrors.java": "133e8eab418b633d83c17f6454b09f27a23fdc0fde7346faffb319cd11377538",
+    "backend/src/main/java/dev/evolution/monitor/CheckQueue.java": "3cfd08e2fe4c8e5e76ba743ba02def6dc8aff61540e7a3d0789cfcea9dbc09a9",
+    "backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java": "2479c6a10526b0b89297690b16e5a5e1f613adcbb9a5850269b8f343338c0083",
+    "backend/src/main/java/dev/evolution/monitor/CheckWorker.java": "678d1d8578b7c2c73c9e162f673c1200511f9a0b640658ce0e7e8a4093b5910d",
+    "backend/src/main/java/dev/evolution/monitor/MonitorController.java": "c200762bbf5004c241f8fc8f2ddd84d6a55d799145c6d9ac7c005133d79a0ef6",
+    "backend/src/main/java/dev/evolution/monitor/MonitorStore.java": "5dc1845f37fad4fdafe21b0aebb465c3bd51c19640f9dc4440e4d3fade135122",
+    "backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql": "56e172e958f2cf9b1e336cdf488131843035a5a0bc8aa2abf3823f48c9d78712",
+    "backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java": "3c035c0dfa654c46fc72f8a7dc07323abc40c1c732c3bee8cbd883c466a29d6a",
+    "backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java": "ef277eec07ec2c5a45d2bdf853bbadf522fecb7e85fd352bcf4107a81a2c132a",
+    "backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java": "129969feb15c77cb0eecc8cd05139521915223e11057e5442f857c52a25adb97",
+    "backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java": "0ce73cb2342372b723b15aa1e62f8609502f468b78fc91269389621749afda0a",
+    "backend/src/test/java/dev/evolution/monitor/SessionClient.java": "8d081b46f939152ca0b854eec697fbe27d1a2cbc6d0f38fd68c599b8ef8701bf",
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "4f523656f537d7fa572fc28a2dffba72f3a73dc9fed5172ec597226df351cf31",
+    "scripts/persistence-scenario.mjs": "b8751d7a37b16ab8b48591b203ffd92c74ba7ea052923aa5b760ab138e838818",
+    "tests/browser/ownership.spec.ts": "3dd51622fc3c9738d7463fe3d7edf493577e9ca14d9d2d1fd024ad655ef57719",
+    "tests/browser/server-state.spec.ts": "d22a15dcccf1b8f5ec7ba29bf6b4880d4bd51f77c65306b9872f31012f15748c"
+  },
+  "frozenHashes": {
+    "evidence/phase-1/E10/fixtures.md": "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+    "evidence/phase-1/E10/baseline.json": "1b8f586b5db75289ea0162d741ce4069b3162cff297b67ebeac8c1514122b3f5",
+    "scripts/e10-baseline.mjs": "c0b28e17b497378a7eee2ba598b45e0d7e5890ebd0e88b0387df36d694c2690e"
+  },
+  "allOriginalFailureArtifactsByteIdentical": true,
+  "gateArtifactsSha256": {
+    "backend/target/e10-ownership.json": "2a62868561f1b08ad5d142686db218ff23c2fbf00acff26fcd7f9a56f3080617",
+    "backend/target/surefire-reports/dev.evolution.monitor.ExecutionOwnershipTest.txt": "4bab6107f5bfd022b7f0a73b0d53c314b22b2a9fa9de6ae99636dd3e08e084ec",
+    "backend/target/surefire-reports/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml": "d02a01e87bc13e0155bbd9933a28be2a658ab821ec4b65e67df1ac37c5ee1499",
+    "backend/target/e10-workers/one-ready.json": "98522420417119959b096ee6a7d9c4a4fead724dc3cede9e7681c901c8d45934",
+    "backend/target/e10-workers/one-result.json": "6f75e01f9d30e281b5ca2ce8b93e77fe08854d1929732cd173224f4085586e55",
+    "backend/target/e10-workers/one.log": "e0d5d8e4d5185f0a98e35fc4da344348ce145c1ac19ebc46dbf5eb23104a2ef4",
+    "backend/target/e10-workers/two-ready.json": "04f9c004a88e2d50d81d8a1a80fb047683aaba373038708edea1a76571bde16a",
+    "backend/target/e10-workers/two-result.json": "d06292283fa0426085787f3b71fda3d9c58624e159e2f9659d5b28fef5686c4d",
+    "backend/target/e10-workers/two.log": "b8b7e6a05d80edec2753c50b6e9b5bf8821a1185ec37742ce7033a803394264a"
+  },
+  "verification": {
+    "diagnostic": {
+      "phase": "diagnostic",
+      "event": "finish",
+      "startedAt": "2026-08-28T05:33:08.278841+00:00",
+      "finishedAt": "2026-08-28T05:33:10.204682+00:00",
+      "elapsedSeconds": 1.925166,
+      "exitCode": 0,
+      "timedOut": false,
+      "processExitAwaited": true,
+      "listenersAfter": {
+        "4324": false
+      }
+    },
+    "targetedGate": {
+      "phase": "gate",
+      "event": "finish",
+      "startedAt": "2026-08-28T05:37:36.623903+00:00",
+      "finishedAt": "2026-08-28T05:37:50.097536+00:00",
+      "elapsedSeconds": 13.47297,
+      "exitCode": 0,
+      "timedOut": false,
+      "processExitAwaited": true,
+      "listenersAfter": {
+        "4321": false,
+        "4322": false,
+        "4323": false,
+        "4324": false,
+        "4325": false
+      }
+    },
+    "mavenNativeSeconds": 12.358,
+    "junit": {
+      "name": "dev.evolution.monitor.ExecutionOwnershipTest",
+      "tests": "1",
+      "failures": "0",
+      "errors": "0",
+      "skipped": "0",
+      "time": "9.056"
+    },
+    "invalidCaseCount": 5,
+    "allInvalidCases400": true,
+    "actualWorkerProcesses": 2,
+    "workerOwnerAndEffectProof": "PASS",
+    "cleanup": "PASS"
+  },
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "backendInvocations": 2,
+    "javaTests": 22,
+    "javaPasses": 21,
+    "javaFailures": 1,
+    "typecheckInvocations": 1,
+    "transportDiagnosticInvocations": 1,
+    "transportDiagnosticRequests": 6,
+    "competingWorkerGateInvocations": 1,
+    "competingWorkerProcesses": 2,
+    "browserInvocations": 0,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "freshRepairsUsed": 1,
+    "maxFreshRepairs": 2
+  },
+  "staticCheckNotes": [
+    {
+      "command": "git diff --cached --check",
+      "exitCode": 2,
+      "scope": "Only byte-exact preserved Maven log whitespace and original test-report EOF. Evidence intentionally unchanged."
+    },
+    {
+      "command": "git diff --check -- backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java backend/src/test/java/dev/evolution/monitor/SessionClient.java evidence/phase-1/E10/repair1/correction-plan.md",
+      "exitCode": 0
+    },
+    {
+      "command": "cat backend/src/main/resources/application.properties backend/src/test/resources/application.properties .gitignore",
+      "exitCode": 1,
+      "scope": "Optional test resource file absent; source read only, no application or test invocation."
+    }
+  ],
+  "unrun": [
+    "No repeated baseline",
+    "No full Maven suite",
+    "No browser gate",
+    "No production build/package/load or previous-Thread rerun",
+    "Remaining E10 authoring and root acceptance/tag/index are outside this repair"
+  ]
+}


