## `test(idempotency): freeze bounded E10 repair diagnostic`

diff --git a/evidence/phase-1/E10/repair1/KeyTransportDiagnostic.java b/evidence/phase-1/E10/repair1/KeyTransportDiagnostic.java
new file mode 100644
index 0000000..e983687
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/KeyTransportDiagnostic.java
@@ -0,0 +1,115 @@
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.io.ByteArrayOutputStream;
+import java.net.InetAddress;
+import java.net.InetSocketAddress;
+import java.net.ServerSocket;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Objects;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.springframework.boot.autoconfigure.http.client.HttpClientAutoConfiguration;
+import org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration;
+import org.springframework.boot.test.web.client.TestRestTemplate;
+import org.springframework.boot.web.client.RestTemplateBuilder;
+import org.springframework.context.annotation.AnnotationConfigApplicationContext;
+import org.springframework.http.HttpEntity;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.HttpMethod;
+import org.springframework.http.client.SimpleClientHttpRequestFactory;
+
+// One frozen, unauthenticated loopback capture; this is not an API acceptance test.
+class KeyTransportDiagnostic {
+    public static void main(String[] args) throws Exception {
+        long started = System.nanoTime();
+        var evidence = new LinkedHashMap<String, Object>();
+        var observations = new ArrayList<Map<String, Object>>();
+        evidence.put("observations", observations);
+        evidence.put("result", "INCOMPLETE");
+        try (var context = new AnnotationConfigApplicationContext();
+                var listener = new ServerSocket();
+                var executor = Executors.newVirtualThreadPerTaskExecutor()) {
+            context.register(HttpClientAutoConfiguration.class, RestTemplateAutoConfiguration.class);
+            context.refresh();
+            var api = new TestRestTemplate(context.getBean(RestTemplateBuilder.class));
+            evidence.put("defaultRequestFactory", api.getRestTemplate().getRequestFactory().getClass().getName());
+            evidence.put("javaRuntimeVersion", System.getProperty("java.runtime.version"));
+            listener.setReuseAddress(false);
+            listener.bind(new InetSocketAddress(InetAddress.getByName("127.0.0.1"), 4324));
+            listener.setSoTimeout(5000);
+            String[] inputs = {null, "", "has space", "é", "x".repeat(129), "é"};
+            String[] labels = {"missing", "empty", "has-space", "non-ASCII-U+00E9", "ASCII-length-129", "non-ASCII-U+00E9-simple"};
+            for (int index = 0; index < inputs.length; index++) {
+                if (index == 5) api.getRestTemplate().setRequestFactory(new SimpleClientHttpRequestFactory());
+                var observed = executor.submit(() -> capture(listener));
+                var headers = new HttpHeaders();
+                headers.setOrigin("http://127.0.0.1:4323");
+                if (inputs[index] != null) headers.set("Idempotency-Key", inputs[index]);
+                var response = api.exchange("http://127.0.0.1:4324/api/monitors/00000000-0000-0000-0000-000000000010/checks",
+                        HttpMethod.POST, new HttpEntity<>(null, headers), Void.class);
+                String wireValue = observed.get(5, TimeUnit.SECONDS);
+                var entry = new LinkedHashMap<String, Object>();
+                entry.put("case", labels[index]);
+                entry.put("requestFactory", api.getRestTemplate().getRequestFactory().getClass().getName());
+                entry.put("inputCodePoints", points(inputs[index]));
+                entry.put("wireCodePoints", points(wireValue));
+                entry.put("wireValueHex", wireValue == null ? null
+                        : HexFormat.of().formatHex(wireValue.getBytes(StandardCharsets.ISO_8859_1)));
+                entry.put("inputPreserved", Objects.equals(inputs[index], wireValue));
+                entry.put("captureStatus", response.getStatusCode().value());
+                observations.add(entry);
+                if (response.getStatusCode().value() != 204) throw new IllegalStateException("Capture response was not204");
+            }
+            evidence.put("result", "OBSERVED");
+        } finally {
+            evidence.put("elapsedSeconds", (System.nanoTime() - started) / 1_000_000_000.0);
+            Files.writeString(Path.of(args[0]), new ObjectMapper().writerWithDefaultPrettyPrinter()
+                    .writeValueAsString(evidence) + "\n");
+        }
+    }
+
+    private static List<Integer> points(String value) {
+        return value == null ? null : value.codePoints().boxed().toList();
+    }
+
+    private static String capture(ServerSocket listener) throws Exception {
+        try (var socket = listener.accept()) {
+            socket.setSoTimeout(5000);
+            var bytes = new ByteArrayOutputStream();
+            int suffix = 0;
+            while (suffix != 0x0d0a0d0a) {
+                int next = socket.getInputStream().read();
+                if (next < 0 || bytes.size() >= 16384) throw new IllegalStateException("Incomplete bounded diagnostic request");
+                bytes.write(next);
+                suffix = (suffix << 8) | next;
+            }
+            String key = null;
+            boolean seen = false;
+            for (String line : bytes.toString(StandardCharsets.ISO_8859_1).split("\r\n")) {
+                int colon = line.indexOf(':');
+                if (colon < 0) continue;
+                String name = line.substring(0, colon);
+                if (name.equalsIgnoreCase("Cookie") || name.equalsIgnoreCase("Authorization")
+                        || name.equalsIgnoreCase("X-CSRF-TOKEN")) {
+                    throw new IllegalStateException("Sensitive header forbidden in this diagnostic");
+                }
+                if (name.equalsIgnoreCase("Idempotency-Key")) {
+                    if (seen) throw new IllegalStateException("Unexpected duplicate diagnostic header");
+                    seen = true;
+                    key = line.substring(colon + 1);
+                    if (key.startsWith(" ")) key = key.substring(1);
+                }
+            }
+            socket.getOutputStream().write(("HTTP/1.1 204 No Content\r\nContent-Length: 0\r\n"
+                    + "Connection: close\r\n\r\n").getBytes(StandardCharsets.US_ASCII));
+            socket.getOutputStream().flush();
+            return key;
+        }
+    }
+}
diff --git a/evidence/phase-1/E10/repair1/attempt1/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml b/evidence/phase-1/E10/repair1/attempt1/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml
new file mode 100644
index 0000000..0193b6a
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/attempt1/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml
@@ -0,0 +1,154 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.ExecutionOwnershipTest" time="4.31" tests="1" errors="0" skipped="0" failures="1" flakes="0">
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
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828141827370_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T14-18-27_330-jvmRun1 surefire-20260828141827370_1tmp surefire_0-20260828141827370_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="ExecutionOwnershipTest,OwnershipAuthorizationTest,MonitorFunctionalTest,CheckQueueTest,HistoryIndexMigrationTest"/>
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
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828141827370_3.jar"/>
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
+    <property name="catalina.home" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.10047164489170671883"/>
+    <property name="com.zaxxer.hikari.pool_number" value="1"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="PID" value="90398"/>
+    <property name="CONSOLE_LOG_CHARSET" value="UTF-8"/>
+    <property name="catalina.base" value="/private/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/tomcat.4322.10047164489170671883"/>
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
+  <testcase name="parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner" classname="dev.evolution.monitor.ExecutionOwnershipTest" time="0.983">
+    <failure message="expected: &lt;400&gt; but was: &lt;202&gt;" type="org.opentest4j.AssertionFailedError"><![CDATA[org.opentest4j.AssertionFailedError: expected: <400> but was: <202>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
+	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
+	at dev.evolution.monitor.ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner(ExecutionOwnershipTest.java:143)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+]]></failure>
+    <system-out><![CDATA[14:18:27.771 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.ExecutionOwnershipTest]: ExecutionOwnershipTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+14:18:27.867 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.ExecutionOwnershipTest
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
+2026-08-28T14:18:28.162+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Starting ExecutionOwnershipTest using Java 21.0.7 with PID 90398 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:18:28.163+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:18:28.535+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:18:28.554+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 12 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:18:28.877+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:18:28.889+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:18:28.889+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:18:28.912+09:00  INFO 90398 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:18:28.912+09:00  INFO 90398 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 739 ms
+2026-08-28T14:18:29.057+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:18:29.075+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@15c16f19
+2026-08-28T14:18:29.076+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:18:29.094+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:29.125+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e10_ownership"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:29.127+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.016s)
+2026-08-28T14:18:29.139+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e10_ownership"."flyway_schema_history" ...
+2026-08-28T14:18:29.187+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": << Empty Schema >>
+2026-08-28T14:18:29.191+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "1 - create monitors"
+2026-08-28T14:18:29.212+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "2 - create check runs"
+2026-08-28T14:18:29.226+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "3 - create users"
+2026-08-28T14:18:29.237+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "4 - require monitor ownership"
+2026-08-28T14:18:29.251+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "5 - index check history"
+2026-08-28T14:18:29.262+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "6 - queue check execution"
+2026-08-28T14:18:29.277+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:29.289+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e10_ownership", now at version v7 (execution time 00:00.031s)
+2026-08-28T14:18:29.341+09:00  INFO 90398 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:18:29.377+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:18:29.395+09:00  INFO 90398 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:18:29.522+09:00  INFO 90398 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:18:29.580+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:18:29.969+09:00  INFO 90398 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:18:29.992+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:30.081+09:00  INFO 90398 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:18:30.497+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:18:30.505+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Started ExecutionOwnershipTest in 2.577 seconds (process running for 3.075)
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
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
diff --git a/evidence/phase-1/E10/repair1/attempt1/backend-1-partial.json b/evidence/phase-1/E10/repair1/attempt1/backend-1-partial.json
new file mode 100644
index 0000000..e3ec01d
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/attempt1/backend-1-partial.json
@@ -0,0 +1,14 @@
+{
+  "fixtureSha256" : "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+  "completed" : [ "same-owner/key parallel requests deduplicated by PostgreSQL after both inserts reached the lock barrier" ],
+  "workerEntry" : "two non-web JVMs, test-only startup gates, production CheckWorker.executeNext once",
+  "result" : "INCOMPLETE",
+  "parallel" : {
+    "blockedInsertTransactions" : 2,
+    "sameId" : true,
+    "persistedRows" : 1,
+    "statuses" : [ 202, 202 ],
+    "outboundRequests" : 0
+  },
+  "allOwnedWorkerExitsAwaited" : true
+}
diff --git a/evidence/phase-1/E10/repair1/attempt1/backend-1.log b/evidence/phase-1/E10/repair1/attempt1/backend-1.log
new file mode 100644
index 0000000..1c97cd6
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/attempt1/backend-1.log
@@ -0,0 +1,366 @@
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
+[INFO] Recompiling the module because of changed source code.
+[INFO] Compiling 21 source files with javac [debug parameters release 21] to target/classes
+[INFO] 
+[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---
+[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources
+[INFO] 
+[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---
+[INFO] Recompiling the module because of changed dependency.
+[INFO] Compiling 15 source files with javac [debug parameters release 21] to target/test-classes
+[INFO] 
+[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---
+[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
+[INFO] 
+[INFO] -------------------------------------------------------
+[INFO]  T E S T S
+[INFO] -------------------------------------------------------
+[INFO] Running dev.evolution.monitor.ExecutionOwnershipTest
+14:18:27.771 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.ExecutionOwnershipTest]: ExecutionOwnershipTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+14:18:27.867 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.ExecutionOwnershipTest
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
+2026-08-28T14:18:28.162+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Starting ExecutionOwnershipTest using Java 21.0.7 with PID 90398 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:18:28.163+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:18:28.535+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:18:28.554+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 12 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:18:28.877+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:18:28.889+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:18:28.889+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:18:28.912+09:00  INFO 90398 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:18:28.912+09:00  INFO 90398 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 739 ms
+2026-08-28T14:18:29.057+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
+2026-08-28T14:18:29.075+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@15c16f19
+2026-08-28T14:18:29.076+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
+2026-08-28T14:18:29.094+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:29.125+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e10_ownership"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:29.127+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.016s)
+2026-08-28T14:18:29.139+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e10_ownership"."flyway_schema_history" ...
+2026-08-28T14:18:29.187+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e10_ownership": << Empty Schema >>
+2026-08-28T14:18:29.191+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "1 - create monitors"
+2026-08-28T14:18:29.212+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "2 - create check runs"
+2026-08-28T14:18:29.226+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "3 - create users"
+2026-08-28T14:18:29.237+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "4 - require monitor ownership"
+2026-08-28T14:18:29.251+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "5 - index check history"
+2026-08-28T14:18:29.262+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "6 - queue check execution"
+2026-08-28T14:18:29.277+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e10_ownership" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:29.289+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e10_ownership", now at version v7 (execution time 00:00.031s)
+2026-08-28T14:18:29.341+09:00  INFO 90398 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:18:29.377+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.6.53.Final
+2026-08-28T14:18:29.395+09:00  INFO 90398 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:18:29.522+09:00  INFO 90398 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:18:29.580+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:18:29.969+09:00  INFO 90398 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:18:29.992+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:30.081+09:00  INFO 90398 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:18:30.497+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:18:30.505+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.ExecutionOwnershipTest       : Started ExecutionOwnershipTest in 2.577 seconds (process running for 3.075)
+Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:18:31.374+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+2026-08-28T14:18:32.000+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T14:18:32.001+09:00  INFO 90398 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T14:18:32.003+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:32.004+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
+2026-08-28T14:18:32.006+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
+[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 4.310 s <<< FAILURE! -- in dev.evolution.monitor.ExecutionOwnershipTest
+[ERROR] dev.evolution.monitor.ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner -- Time elapsed: 0.983 s <<< FAILURE!
+org.opentest4j.AssertionFailedError: expected: <400> but was: <202>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
+	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
+	at dev.evolution.monitor.ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner(ExecutionOwnershipTest.java:143)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+
+[INFO] Running dev.evolution.monitor.CheckQueueTest
+2026-08-28T14:18:32.014+09:00  INFO 90398 --- [monitor-api] [           main] t.c.s.AnnotationConfigContextLoaderUtils : Could not detect default configuration classes for test class [dev.evolution.monitor.CheckQueueTest]: CheckQueueTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+2026-08-28T14:18:32.017+09:00  INFO 90398 --- [monitor-api] [           main] .b.t.c.SpringBootTestContextBootstrapper : Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.CheckQueueTest
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
+2026-08-28T14:18:32.050+09:00  INFO 90398 --- [monitor-api] [           main] dev.evolution.monitor.CheckQueueTest     : Starting CheckQueueTest using Java 21.0.7 with PID 90398 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:18:32.050+09:00  INFO 90398 --- [monitor-api] [           main] dev.evolution.monitor.CheckQueueTest     : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:18:32.223+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:18:32.228+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 2 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:18:32.279+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:18:32.280+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:18:32.280+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:18:32.288+09:00  INFO 90398 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:18:32.288+09:00  INFO 90398 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 236 ms
+2026-08-28T14:18:32.316+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Starting...
+2026-08-28T14:18:32.325+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-2 - Added connection org.postgresql.jdbc.PgConnection@228ed68c
+2026-08-28T14:18:32.325+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Start completed.
+2026-08-28T14:18:32.326+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:32.343+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e09_scheduler"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:32.345+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.006s)
+2026-08-28T14:18:32.360+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e09_scheduler"."flyway_schema_history" ...
+2026-08-28T14:18:32.382+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e09_scheduler": << Empty Schema >>
+2026-08-28T14:18:32.386+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "1 - create monitors"
+2026-08-28T14:18:32.401+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "2 - create check runs"
+2026-08-28T14:18:32.417+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "3 - create users"
+2026-08-28T14:18:32.429+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "4 - require monitor ownership"
+2026-08-28T14:18:32.442+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "5 - index check history"
+2026-08-28T14:18:32.450+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "6 - queue check execution"
+2026-08-28T14:18:32.460+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e09_scheduler" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:32.468+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e09_scheduler", now at version v7 (execution time 00:00.033s)
+2026-08-28T14:18:32.478+09:00  INFO 90398 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:18:32.479+09:00  INFO 90398 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:18:32.483+09:00  INFO 90398 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:18:32.488+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-2)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:18:32.512+09:00  INFO 90398 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:18:32.519+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:32.654+09:00  INFO 90398 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:18:32.878+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:18:32.887+09:00  INFO 90398 --- [monitor-api] [           main] dev.evolution.monitor.CheckQueueTest     : Started CheckQueueTest in 0.868 seconds (process running for 5.457)
+2026-08-28T14:18:33.092+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:18:33.092+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:18:33.093+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
+2026-08-28T14:18:33.419+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T14:18:33.419+09:00  INFO 90398 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T14:18:33.422+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:33.422+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Shutdown initiated...
+2026-08-28T14:18:33.426+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Shutdown completed.
+[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.420 s -- in dev.evolution.monitor.CheckQueueTest
+[INFO] Running dev.evolution.monitor.MonitorFunctionalTest
+2026-08-28T14:18:33.440+09:00  INFO 90398 --- [monitor-api] [           main] t.c.s.AnnotationConfigContextLoaderUtils : Could not detect default configuration classes for test class [dev.evolution.monitor.MonitorFunctionalTest]: MonitorFunctionalTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+2026-08-28T14:18:33.444+09:00  INFO 90398 --- [monitor-api] [           main] .b.t.c.SpringBootTestContextBootstrapper : Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.MonitorFunctionalTest
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
+2026-08-28T14:18:33.489+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Starting MonitorFunctionalTest using Java 21.0.7 with PID 90398 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:18:33.489+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:18:33.600+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:18:33.603+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 2 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:18:33.649+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:18:33.650+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:18:33.650+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:18:33.661+09:00  INFO 90398 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:18:33.661+09:00  INFO 90398 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 170 ms
+2026-08-28T14:18:33.691+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Starting...
+2026-08-28T14:18:33.699+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-3 - Added connection org.postgresql.jdbc.PgConnection@13068335
+2026-08-28T14:18:33.699+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Start completed.
+2026-08-28T14:18:33.702+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:33.722+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e04_functional"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:33.723+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.009s)
+2026-08-28T14:18:33.735+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e04_functional"."flyway_schema_history" ...
+2026-08-28T14:18:33.762+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e04_functional": << Empty Schema >>
+2026-08-28T14:18:33.768+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "1 - create monitors"
+2026-08-28T14:18:33.789+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "2 - create check runs"
+2026-08-28T14:18:33.807+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "3 - create users"
+2026-08-28T14:18:33.822+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "4 - require monitor ownership"
+2026-08-28T14:18:33.844+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "5 - index check history"
+2026-08-28T14:18:33.864+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "6 - queue check execution"
+2026-08-28T14:18:33.886+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e04_functional" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:33.897+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e04_functional", now at version v7 (execution time 00:00.043s)
+2026-08-28T14:18:33.911+09:00  INFO 90398 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:18:33.912+09:00  INFO 90398 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:18:33.917+09:00  INFO 90398 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:18:33.923+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-3)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:18:33.947+09:00  INFO 90398 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:18:33.956+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:33.981+09:00  INFO 90398 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:18:34.028+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:18:34.029+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.MonitorFunctionalTest        : Started MonitorFunctionalTest in 0.576 seconds (process running for 6.599)
+2026-08-28T14:18:34.195+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:18:34.195+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:18:34.195+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+2026-08-28T14:18:35.525+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T14:18:35.527+09:00  INFO 90398 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T14:18:35.531+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:35.532+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Shutdown initiated...
+2026-08-28T14:18:35.549+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Shutdown completed.
+[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.115 s -- in dev.evolution.monitor.MonitorFunctionalTest
+[INFO] Running dev.evolution.monitor.HistoryIndexMigrationTest
+2026-08-28T14:18:35.599+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:35.615+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e07_index_upgrade"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:35.617+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.011s)
+2026-08-28T14:18:35.632+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e07_index_upgrade"."flyway_schema_history" ...
+2026-08-28T14:18:35.760+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": << Empty Schema >>
+2026-08-28T14:18:35.771+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "1 - create monitors"
+2026-08-28T14:18:35.802+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "2 - create check runs"
+2026-08-28T14:18:35.815+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "3 - create users"
+2026-08-28T14:18:35.826+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "4 - require monitor ownership"
+2026-08-28T14:18:35.835+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 4 migrations to schema "e07_index_upgrade", now at version v4 (execution time 00:00.034s)
+2026-08-28T14:18:35.958+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:35.966+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:35.976+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 4
+2026-08-28T14:18:35.977+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "5 - index check history"
+2026-08-28T14:18:35.986+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema "e07_index_upgrade", now at version v5 (execution time 00:00.003s)
+2026-08-28T14:18:36.002+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.051+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.062+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 5
+2026-08-28T14:18:36.063+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema "e07_index_upgrade" is up to date. No migration necessary.
+2026-08-28T14:18:36.108+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:36.117+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.128+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 5
+2026-08-28T14:18:36.131+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "6 - queue check execution"
+2026-08-28T14:18:36.144+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema "e07_index_upgrade", now at version v6 (execution time 00:00.006s)
+2026-08-28T14:18:36.157+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.167+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 6
+2026-08-28T14:18:36.168+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema "e07_index_upgrade" is up to date. No migration necessary.
+2026-08-28T14:18:36.215+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:36.227+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.238+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 6
+2026-08-28T14:18:36.241+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e07_index_upgrade" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:36.253+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema "e07_index_upgrade", now at version v7 (execution time 00:00.004s)
+2026-08-28T14:18:36.275+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.286+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e07_index_upgrade": 7
+2026-08-28T14:18:36.286+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Schema "e07_index_upgrade" is up to date. No migration necessary.
+[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.760 s -- in dev.evolution.monitor.HistoryIndexMigrationTest
+[INFO] Running dev.evolution.monitor.OwnershipAuthorizationTest
+2026-08-28T14:18:36.321+09:00  INFO 90398 --- [monitor-api] [           main] t.c.s.AnnotationConfigContextLoaderUtils : Could not detect default configuration classes for test class [dev.evolution.monitor.OwnershipAuthorizationTest]: OwnershipAuthorizationTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+2026-08-28T14:18:36.324+09:00  INFO 90398 --- [monitor-api] [           main] .b.t.c.SpringBootTestContextBootstrapper : Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.OwnershipAuthorizationTest
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
+2026-08-28T14:18:36.362+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.OwnershipAuthorizationTest   : Starting OwnershipAuthorizationTest using Java 21.0.7 with PID 90398 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)
+2026-08-28T14:18:36.362+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.OwnershipAuthorizationTest   : No active profile set, falling back to 1 default profile: "default"
+2026-08-28T14:18:36.426+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
+2026-08-28T14:18:36.429+09:00  INFO 90398 --- [monitor-api] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 2 ms. Found 0 JPA repository interfaces.
+2026-08-28T14:18:36.459+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 4322 (http)
+2026-08-28T14:18:36.460+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
+2026-08-28T14:18:36.460+09:00  INFO 90398 --- [monitor-api] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.55]
+2026-08-28T14:18:36.468+09:00  INFO 90398 --- [monitor-api] [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
+2026-08-28T14:18:36.468+09:00  INFO 90398 --- [monitor-api] [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 105 ms
+2026-08-28T14:18:36.484+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-4 - Starting...
+2026-08-28T14:18:36.490+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-4 - Added connection org.postgresql.jdbc.PgConnection@3316aded
+2026-08-28T14:18:36.490+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-4 - Start completed.
+2026-08-28T14:18:36.491+09:00  INFO 90398 --- [monitor-api] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)
+2026-08-28T14:18:36.500+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "e05_ownership"."flyway_schema_history" does not exist yet
+2026-08-28T14:18:36.501+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbValidate     : Successfully validated 7 migrations (execution time 00:00.004s)
+2026-08-28T14:18:36.509+09:00  INFO 90398 --- [monitor-api] [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "e05_ownership"."flyway_schema_history" ...
+2026-08-28T14:18:36.525+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Current version of schema "e05_ownership": << Empty Schema >>
+2026-08-28T14:18:36.528+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "1 - create monitors"
+2026-08-28T14:18:36.542+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "2 - create check runs"
+2026-08-28T14:18:36.559+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "3 - create users"
+2026-08-28T14:18:36.570+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "4 - require monitor ownership"
+2026-08-28T14:18:36.594+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "5 - index check history"
+2026-08-28T14:18:36.609+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "6 - queue check execution"
+2026-08-28T14:18:36.632+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "e05_ownership" to version "7 - execution ownership and manual identity"
+2026-08-28T14:18:36.644+09:00  INFO 90398 --- [monitor-api] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 7 migrations to schema "e05_ownership", now at version v7 (execution time 00:00.040s)
+2026-08-28T14:18:36.654+09:00  INFO 90398 --- [monitor-api] [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
+2026-08-28T14:18:36.655+09:00  INFO 90398 --- [monitor-api] [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
+2026-08-28T14:18:36.659+09:00  INFO 90398 --- [monitor-api] [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
+2026-08-28T14:18:36.663+09:00  INFO 90398 --- [monitor-api] [           main] org.hibernate.orm.connections.pooling    : HHH10001005: Database info:
+	Database JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-4)']
+	Database driver: undefined/unknown
+	Database version: 17.11
+	Autocommit mode: undefined/unknown
+	Isolation level: undefined/unknown
+	Minimum pool size: undefined/unknown
+	Maximum pool size: undefined/unknown
+2026-08-28T14:18:36.684+09:00  INFO 90398 --- [monitor-api] [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
+2026-08-28T14:18:36.691+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:36.721+09:00  INFO 90398 --- [monitor-api] [           main] r$InitializeUserDetailsManagerConfigurer : Global AuthenticationManager configured with UserDetailsService bean with name userAccounts
+2026-08-28T14:18:36.762+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 4322 (http) with context path '/'
+2026-08-28T14:18:36.763+09:00  INFO 90398 --- [monitor-api] [           main] d.e.monitor.OwnershipAuthorizationTest   : Started OwnershipAuthorizationTest in 0.428 seconds (process running for 9.333)
+2026-08-28T14:18:36.940+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
+2026-08-28T14:18:36.941+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
+2026-08-28T14:18:36.941+09:00  INFO 90398 --- [monitor-api] [0.1-4322-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
+2026-08-28T14:18:39.481+09:00  INFO 90398 --- [monitor-api] [           main] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
+2026-08-28T14:18:39.483+09:00  INFO 90398 --- [monitor-api] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
+2026-08-28T14:18:39.485+09:00  INFO 90398 --- [monitor-api] [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
+2026-08-28T14:18:39.485+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-4 - Shutdown initiated...
+2026-08-28T14:18:39.490+09:00  INFO 90398 --- [monitor-api] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-4 - Shutdown completed.
+[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.174 s -- in dev.evolution.monitor.OwnershipAuthorizationTest
+[INFO] 
+[INFO] Results:
+[INFO] 
+[ERROR] Failures: 
+[ERROR]   ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner:143 expected: <400> but was: <202>
+[INFO] 
+[ERROR] Tests run: 21, Failures: 1, Errors: 0, Skipped: 0
+[INFO] 
+[INFO] ------------------------------------------------------------------------
+[INFO] BUILD FAILURE
+[INFO] ------------------------------------------------------------------------
+[INFO] Total time:  14.767 s
+[INFO] Finished at: 2026-08-28T14:18:39+09:00
+[INFO] ------------------------------------------------------------------------
+[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:3.5.6:test (default-test) on project monitor-api: There are test failures.
+[ERROR] 
+[ERROR] See /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire-reports for the individual test results.
+[ERROR] See dump files (if any exist) [date].dump, [date]-jvmRun[N].dump and [date].dumpstream.
+[ERROR] -> [Help 1]
+[ERROR] 
+[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
+[ERROR] Re-run Maven using the -X switch to enable full debug logging.
+[ERROR] 
+[ERROR] For more information about the errors and possible solutions, please read the following articles:
+[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
diff --git a/evidence/phase-1/E10/repair1/attempt1/dev.evolution.monitor.ExecutionOwnershipTest.txt b/evidence/phase-1/E10/repair1/attempt1/dev.evolution.monitor.ExecutionOwnershipTest.txt
new file mode 100644
index 0000000..74db031
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/attempt1/dev.evolution.monitor.ExecutionOwnershipTest.txt
@@ -0,0 +1,17 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.ExecutionOwnershipTest
+-------------------------------------------------------------------------------
+Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 4.310 s <<< FAILURE! -- in dev.evolution.monitor.ExecutionOwnershipTest
+dev.evolution.monitor.ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner -- Time elapsed: 0.983 s <<< FAILURE!
+org.opentest4j.AssertionFailedError: expected: <400> but was: <202>
+	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
+	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
+	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
+	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
+	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
+	at dev.evolution.monitor.ExecutionOwnershipTest.parallelIdentityAndTwoRealWorkersRetainOneIntentAndOneExecutionOwner(ExecutionOwnershipTest.java:143)
+	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
+
diff --git a/evidence/phase-1/E10/repair1/attempt1/invocations.jsonl b/evidence/phase-1/E10/repair1/attempt1/invocations.jsonl
new file mode 100644
index 0000000..86d7682
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/attempt1/invocations.jsonl
@@ -0,0 +1,3 @@
+{"command":"node scripts/e10-baseline.mjs","startedAt":"2026-08-28T05:06:41.343Z","elapsedSeconds":8.036,"exitCode":1}
+{"name":"backend-1","command":"mvn -B -ntp -f backend/pom.xml -Dtest=ExecutionOwnershipTest,OwnershipAuthorizationTest,MonitorFunctionalTest,CheckQueueTest,HistoryIndexMigrationTest package","startedAt":"2026-08-28T05:18:23.987Z","elapsedSeconds":15.919,"exitCode":1}
+{"name":"typecheck-1","command":"npm run typecheck","startedAt":"2026-08-28T05:18:38.344Z","elapsedSeconds":1.988,"exitCode":0}
diff --git a/evidence/phase-1/E10/repair1/diagnostic-plan.md b/evidence/phase-1/E10/repair1/diagnostic-plan.md
new file mode 100644
index 0000000..72fe50e
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/diagnostic-plan.md
@@ -0,0 +1,46 @@
+# E10 repair1 frozen transport diagnostic
+
+Recorded before execution. Repair start is
+`bbacf6924f6a8c6fe87185c5f0b91317ca17b3a1`; Thread START, SPEC_REVISION,
+all five invalid inputs and every acceptance condition remain unchanged.
+
+One Java source launch uses the failed test's preserved classpath and Temurin
+21.0.7+6. It constructs Boot's standard HTTP-client/RestTemplate auto-configuration
+and TestRestTemplate. Source inspection found no application/test overrides of
+these builders or request factories. It records the selected factory class.
+No application, database, authentication, worker or browser is started.
+
+Exactly six sequential empty-body POSTs target a raw listener at
+`127.0.0.1:4324`; occupied bind is refused. The path is a diagnostic-only nested
+Check path with a fixed all-zero test UUID. The first five requests use the
+selected factory, with the frozen sequence: missing key, empty key, `has space`,
+U+00E9 (`é`), and 129 `x` characters. The sixth uses exactly one predetermined
+alternative, Spring's `SimpleClientHttpRequestFactory`, with the same U+00E9
+input. This is one bounded comparison, not a factory or parameter sweep.
+
+The listener captures only the public Idempotency-Key value bytes, rejects any
+credential/CSRF-bearing headers, returns204, and closes each connection. No
+request headers, private bodies, credentials, cookies or tokens are serialized.
+Each accept/read and future wait is bounded at5 seconds; the launcher has a
+60-second process bound. Listener, accepted sockets and executor close on exit.
+
+Hypothesis, not yet a result: the default selected JDK HTTP/1.1 transport encodes
+U+00E9 as ASCII `?`. Read-only inspection of this exact JDK's Http1Request.headers
+shows `String.getBytes(StandardCharsets.US_ASCII)`. The diagnostic records the
+actual bytes for every input without substituting an expected result.
+
+Only if the observed transport identifies the defect will the test transport
+receive the minimum correction. The one subsequent acceptance command is:
+
+```sh
+mvn -B -ntp -f backend/pom.xml -Dtest=ExecutionOwnershipTest test
+```
+
+It retains the fixed real authenticated API, PostgreSQL barrier, both real worker
+JVMs, owner/terminal checks and2-second outbound timeout. No baseline, full suite,
+browser, build, load, retry or previous-Thread scenario is authorized here. Any
+acceptance failure stops this repair; no second gate or self-retry is allowed.
+
+The evidence launcher records its exact argv, start/finish times, monotonic
+duration, exit and pre/post loopback listener checks. Both the failed attempt's
+original invocation ledger and this repair's ledger are retained separately.
diff --git a/evidence/phase-1/E10/repair1/preflight.json b/evidence/phase-1/E10/repair1/preflight.json
new file mode 100644
index 0000000..1f339d6
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/preflight.json
@@ -0,0 +1,53 @@
+{
+  "recordedAt": "2026-08-28T05:30:03.090729+00:00",
+  "branch": "track/industry-spring",
+  "threadStart": "3cc49f3d2a35055c92d0312fca6167c89dfadec5",
+  "repairStart": "bbacf6924f6a8c6fe87185c5f0b91317ca17b3a1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "originalAuthorStoppedByRoot": true,
+  "wip": {
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
+    "backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java": "77b54595f77fe2420813bb494a6884f38dd8dbc37c1cec06bc015dba749d5da4",
+    "backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java": "129969feb15c77cb0eecc8cd05139521915223e11057e5442f857c52a25adb97",
+    "backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java": "0ce73cb2342372b723b15aa1e62f8609502f468b78fc91269389621749afda0a",
+    "backend/src/test/java/dev/evolution/monitor/SessionClient.java": "0841aaa9c8103022e272ce8e1d34dc72ffc389db484eb58b4b336882256f7212",
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "4f523656f537d7fa572fc28a2dffba72f3a73dc9fed5172ec597226df351cf31",
+    "scripts/persistence-scenario.mjs": "b8751d7a37b16ab8b48591b203ffd92c74ba7ea052923aa5b760ab138e838818",
+    "tests/browser/ownership.spec.ts": "3dd51622fc3c9738d7463fe3d7edf493577e9ca14d9d2d1fd024ad655ef57719",
+    "tests/browser/server-state.spec.ts": "d22a15dcccf1b8f5ec7ba29bf6b4880d4bd51f77c65306b9872f31012f15748c"
+  },
+  "frozen": {
+    "evidence/phase-1/E10/fixtures.md": "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+    "evidence/phase-1/E10/baseline.json": "1b8f586b5db75289ea0162d741ce4069b3162cff297b67ebeac8c1514122b3f5",
+    "scripts/e10-baseline.mjs": "c0b28e17b497378a7eee2ba598b45e0d7e5890ebd0e88b0387df36d694c2690e"
+  },
+  "preparationErrors": [
+    {
+      "command": "javap -classpath .m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar -p -c org.springframework.boot.web.client.ClientHttpRequestFactoryBuilder",
+      "exitCode": 1,
+      "reason": "Incorrect package in a read-only class lookup; corrected lookup used org.springframework.boot.http.client.ClientHttpRequestFactoryBuilder; no application/test ran."
+    }
+  ],
+  "originalBudget": {
+    "baselineInvocations": 1,
+    "backendInvocations": 1,
+    "typecheckInvocations": 1,
+    "javaTests": 21,
+    "javaPasses": 20,
+    "javaFailures": 1,
+    "workerProcessInvocations": 0,
+    "browserInvocations": 0,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  }
+}
diff --git a/evidence/phase-1/E10/repair1/preserved-manifest.json b/evidence/phase-1/E10/repair1/preserved-manifest.json
new file mode 100644
index 0000000..e8c0f71
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/preserved-manifest.json
@@ -0,0 +1,33 @@
+{
+  "head": "bbacf6924f6a8c6fe87185c5f0b91317ca17b3a1",
+  "start": "3cc49f3d2a35055c92d0312fca6167c89dfadec5",
+  "profile": "phase-1",
+  "spec_revision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "files": {
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
+    "backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java": "77b54595f77fe2420813bb494a6884f38dd8dbc37c1cec06bc015dba749d5da4",
+    "backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java": "129969feb15c77cb0eecc8cd05139521915223e11057e5442f857c52a25adb97",
+    "backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java": "0ce73cb2342372b723b15aa1e62f8609502f468b78fc91269389621749afda0a",
+    "backend/src/test/java/dev/evolution/monitor/SessionClient.java": "0841aaa9c8103022e272ce8e1d34dc72ffc389db484eb58b4b336882256f7212",
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "4f523656f537d7fa572fc28a2dffba72f3a73dc9fed5172ec597226df351cf31",
+    "scripts/persistence-scenario.mjs": "b8751d7a37b16ab8b48591b203ffd92c74ba7ea052923aa5b760ab138e838818",
+    "tests/browser/ownership.spec.ts": "3dd51622fc3c9738d7463fe3d7edf493577e9ca14d9d2d1fd024ad655ef57719",
+    "tests/browser/server-state.spec.ts": "d22a15dcccf1b8f5ec7ba29bf6b4880d4bd51f77c65306b9872f31012f15748c"
+  },
+  "actual_failed_outputs": {
+    "output/phase-1/e10/backend-1.log": "471d9d4e212fccb2a474f513a385ed8bb65fa0f9e096b05b8a0527f2c6961a66",
+    "output/phase-1/e10/backend-1-partial.json": "4384ac541c9b79c5b3b4b02d96ebf258f9d4dc9662c717c51f1ec32f611147f8",
+    "output/phase-1/e10/invocations.jsonl": "2e6c7c1a8b7e9e97975fbc9d5169ead423dc3a39166c51c94e10790313bb041d",
+    "backend/target/surefire-reports/dev.evolution.monitor.ExecutionOwnershipTest.txt": "086d957eedbecb4b1b57114652455778e6f796ff723aa56b5e518bd79f0429fa",
+    "backend/target/surefire-reports/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml": "7ac4c871d1b4e9d6f8f5e007296a8de92f8aa56c55b4a732c582facacfc9fe49"
+  }
+}
diff --git a/evidence/phase-1/E10/repair1/run-repair1.py b/evidence/phase-1/E10/repair1/run-repair1.py
new file mode 100644
index 0000000..4d23d5f
--- /dev/null
+++ b/evidence/phase-1/E10/repair1/run-repair1.py
@@ -0,0 +1,85 @@
+"""Execute each of the two authorized E10 repair1 invocations at most once."""
+from datetime import datetime, timezone
+from pathlib import Path
+import json
+import os
+import signal
+import socket
+import subprocess
+import sys
+import time
+import xml.etree.ElementTree as ET
+
+repair = Path(__file__).resolve().parent
+root = repair.parents[3]
+phase = sys.argv[1]
+if phase not in {"diagnostic", "gate"}:
+    raise SystemExit("Only the frozen diagnostic and targeted gate are authorized")
+ledger = repair / "invocations.jsonl"
+previous = [json.loads(line) for line in ledger.read_text().splitlines()] if ledger.exists() else []
+if any(entry["phase"] == phase for entry in previous):
+    raise SystemExit("Refusing to repeat an already recorded repair invocation")
+java_home = Path("/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem")
+env = os.environ.copy()
+env.update(JAVA_HOME=str(java_home), PATH=str(java_home / "bin") + ":" + env["PATH"],
+           DB_URL="jdbc:postgresql://127.0.0.1:15432/monitor", DB_USER="wse_industry",
+           DB_PASSWORD="", API_PORT="4322", FIXTURE_ORIGIN="http://127.0.0.1:4321")
+for name in ["E04_ALICE_PASSWORD", "E04_BOB_PASSWORD"]:
+    env.pop(name, None)
+if phase == "diagnostic":
+    report = ET.parse(repair / "attempt1/TEST-dev.evolution.monitor.ExecutionOwnershipTest.xml").getroot()
+    classpath = report.find("./properties/property[@name='java.class.path']").get("value")
+    command = [str(java_home / "bin/java"), "--class-path", classpath,
+               str(repair / "KeyTransportDiagnostic.java"), str(repair / "diagnostic.json")]
+    timeout = 60
+    ports = [4324]
+else:
+    diagnostic = json.loads((repair / "diagnostic.json").read_text())
+    if diagnostic["result"] != "OBSERVED":
+        raise SystemExit("The single diagnostic did not complete; do not run the gate")
+    command = ["mvn", "-B", "-ntp", "-f", "backend/pom.xml", "-Dtest=ExecutionOwnershipTest", "test"]
+    timeout = 120
+    ports = [4321, 4322, 4323, 4324, 4325]
+
+def listeners():
+    result = {}
+    for port in ports:
+        with socket.socket() as connection:
+            connection.settimeout(0.25)
+            result[str(port)] = connection.connect_ex(("127.0.0.1", port)) == 0
+    return result
+
+def record(entry):
+    with ledger.open("a") as output:
+        output.write(json.dumps(entry, separators=(",", ":")) + "\n")
+
+before = listeners()
+if any(before.values()):
+    raise SystemExit("Refusing occupied owned loopback ports")
+start = datetime.now(timezone.utc).isoformat()
+record({"phase": phase, "event": "start", "argv": command, "cwd": str(root),
+        "startedAt": start, "timeoutSeconds": timeout, "listenersBefore": before})
+started = time.monotonic()
+timed_out = False
+with (repair / (phase + ".log")).open("w") as output:
+    process = subprocess.Popen(command, cwd=root, env=env, stdout=output,
+                               stderr=subprocess.STDOUT, start_new_session=True)
+    try:
+        exit_code = process.wait(timeout=timeout)
+    except subprocess.TimeoutExpired:
+        timed_out = True
+        os.killpg(process.pid, signal.SIGTERM)
+        try:
+            process.wait(timeout=5)
+        except subprocess.TimeoutExpired:
+            os.killpg(process.pid, signal.SIGKILL)
+            process.wait(timeout=5)
+        exit_code = 124
+entry = {"phase": phase, "event": "finish", "startedAt": start,
+         "finishedAt": datetime.now(timezone.utc).isoformat(),
+         "elapsedSeconds": round(time.monotonic() - started, 6), "exitCode": exit_code,
+         "timedOut": timed_out, "processExitAwaited": process.poll() is not None,
+         "listenersAfter": listeners()}
+record(entry)
+print(json.dumps(entry))
+raise SystemExit(exit_code)


