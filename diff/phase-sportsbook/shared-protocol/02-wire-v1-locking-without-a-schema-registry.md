# Schema Registry 없는 Wire V1 고정

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


## `feat(events): define wire monetary amounts`

diff --git a/src/main/avro/.gitkeep b/src/main/avro/.gitkeep
deleted file mode 100644
index 8b13789..0000000
--- a/src/main/avro/.gitkeep
+++ /dev/null
@@ -1 +0,0 @@
-
diff --git a/src/main/avro/com/sportsbook/protocol/event/Money.avsc b/src/main/avro/com/sportsbook/protocol/event/Money.avsc
new file mode 100644
index 0000000..40a65e9
--- /dev/null
+++ b/src/main/avro/com/sportsbook/protocol/event/Money.avsc
@@ -0,0 +1,10 @@
+{
+  "namespace": "com.sportsbook.protocol.event",
+  "type": "record",
+  "name": "Money",
+  "doc": "Monetary amount in long minor units paired with an ISO currency code; mirrors com.sportsbook.protocol.value.Money on the wire.",
+  "fields": [
+    {"name": "amount", "type": "long", "doc": "Minor units (e.g. cents, won)."},
+    {"name": "currency", "type": "string", "doc": "ISO 4217 code such as KRW or USD."}
+  ]
+}


## `test(events): verify wire monetary amounts`

diff --git a/src/test/java/com/sportsbook/protocol/event/AvroRecordTestSupport.java b/src/test/java/com/sportsbook/protocol/event/AvroRecordTestSupport.java
new file mode 100644
index 0000000..810e09e
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/AvroRecordTestSupport.java
@@ -0,0 +1,35 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import java.util.Arrays;
+import org.apache.avro.Schema;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+
+final class AvroRecordTestSupport {
+
+  private AvroRecordTestSupport() {}
+
+  static void assertFields(Schema schema, String... fields) {
+    assertThat(schema.getFields().stream().map(Schema.Field::name))
+        .containsExactlyElementsOf(Arrays.asList(fields));
+  }
+
+  static <T extends SpecificRecord> T roundTrip(T expected, Class<T> type) throws IOException {
+    byte[] payload;
+    try (ByteArrayOutputStream output = new ByteArrayOutputStream()) {
+      var encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<T>(expected.getSchema()).write(expected, encoder);
+      encoder.flush();
+      payload = output.toByteArray();
+    }
+    return new SpecificDatumReader<>(type)
+        .read(null, DecoderFactory.get().binaryDecoder(payload, null));
+  }
+}
diff --git a/src/test/java/com/sportsbook/protocol/event/MoneyRecordTest.java b/src/test/java/com/sportsbook/protocol/event/MoneyRecordTest.java
new file mode 100644
index 0000000..1c9cbe8
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/MoneyRecordTest.java
@@ -0,0 +1,15 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class MoneyRecordTest {
+
+  @Test
+  void moneyRecordPreservesMinorUnitsAndCurrency() throws Exception {
+    Money expected = Money.newBuilder().setAmount(12_345).setCurrency("KRW").build();
+    AvroRecordTestSupport.assertFields(Money.getClassSchema(), "amount", "currency");
+    assertThat(AvroRecordTestSupport.roundTrip(expected, Money.class)).isEqualTo(expected);
+  }
+}


## `build(avro): order shared named schemas`

diff --git a/pom.xml b/pom.xml
index 9bbfbd7..14f8026 100644
--- a/pom.xml
+++ b/pom.xml
@@ -89,6 +89,10 @@
                             <goal>schema</goal>
                         </goals>
                         <configuration>
+                            <imports>
+                                <import>${project.basedir}/src/main/avro/com/sportsbook/protocol/event/Money.avsc</import>
+                                <import>${project.basedir}/src/main/avro/com/sportsbook/protocol/event/BetSettled.avsc</import>
+                            </imports>
                             <sourceDirectory>${project.basedir}/src/main/avro</sourceDirectory>
                             <outputDirectory>${project.build.directory}/generated-sources/avro</outputDirectory>
                             <stringType>String</stringType>


## `test(events): lock wire v1 schema inventory`

diff --git a/src/test/java/com/sportsbook/protocol/event/WireSchemaInventoryTest.java b/src/test/java/com/sportsbook/protocol/event/WireSchemaInventoryTest.java
new file mode 100644
index 0000000..7311cf0
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WireSchemaInventoryTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.protocol.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Set;
+import org.junit.jupiter.api.Test;
+
+class WireSchemaInventoryTest {
+
+  private static final String NAMESPACE = "com.sportsbook.protocol.event.";
+
+  @Test
+  void wireV1ContainsExactRecordSet() throws Exception {
+    assertThat(WireSchemaTestSupport.loadSchemas().keySet())
+        .containsExactlyInAnyOrderElementsOf(
+            Set.of(
+                NAMESPACE + "BetPlacedRequested",
+                NAMESPACE + "BetResolutionRevised",
+                NAMESPACE + "BetSettled",
+                NAMESPACE + "BetVoided",
+                NAMESPACE + "EventLifecycle",
+                NAMESPACE + "MarketStatusChanged",
+                NAMESPACE + "MatchResult",
+                NAMESPACE + "Money",
+                NAMESPACE + "OddsChanged",
+                NAMESPACE + "RiskLimitViolated",
+                NAMESPACE + "RiskPatternSuspected",
+                NAMESPACE + "WalletCredited",
+                NAMESPACE + "WalletDebitFailed",
+                NAMESPACE + "WalletDebited"));
+  }
+}
diff --git a/src/test/java/com/sportsbook/protocol/event/WireSchemaTestSupport.java b/src/test/java/com/sportsbook/protocol/event/WireSchemaTestSupport.java
new file mode 100644
index 0000000..9c72b34
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WireSchemaTestSupport.java
@@ -0,0 +1,46 @@
+package com.sportsbook.protocol.event;
+
+import java.io.File;
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.Comparator;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import org.apache.avro.Schema;
+
+final class WireSchemaTestSupport {
+
+  private static final Path SCHEMA_ROOT = Path.of("src/main/avro");
+
+  private WireSchemaTestSupport() {}
+
+  static Map<String, Schema> loadSchemas() throws IOException {
+    List<Path> files;
+    try (var paths = Files.walk(SCHEMA_ROOT)) {
+      files =
+          paths
+              .filter(path -> path.toString().endsWith(".avsc"))
+              .sorted(
+                  Comparator.comparingInt(WireSchemaTestSupport::schemaOrder)
+                      .thenComparing(Path::toString))
+              .toList();
+    }
+    Schema.Parser parser = new Schema.Parser();
+    Map<String, Schema> schemas = new LinkedHashMap<>();
+    for (Path file : files) {
+      Schema schema = parser.parse(new File(file.toString()));
+      schemas.put(schema.getFullName(), schema);
+    }
+    return schemas;
+  }
+
+  private static int schemaOrder(Path path) {
+    return switch (path.getFileName().toString()) {
+      case "Money.avsc" -> 0;
+      case "BetSettled.avsc" -> 1;
+      default -> 2;
+    };
+  }
+}


## `test(events): lock wire v1 fingerprints`

diff --git a/src/test/java/com/sportsbook/protocol/event/WireSchemaFingerprintTest.java b/src/test/java/com/sportsbook/protocol/event/WireSchemaFingerprintTest.java
new file mode 100644
index 0000000..57a8e1a
--- /dev/null
+++ b/src/test/java/com/sportsbook/protocol/event/WireSchemaFingerprintTest.java
@@ -0,0 +1,106 @@
+package com.sportsbook.protocol.event;
+
+import static java.util.Map.entry;
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Locale;
+import java.util.Map;
+import java.util.Set;
+import java.util.stream.Collectors;
+import org.apache.avro.Schema;
+import org.apache.avro.SchemaNormalization;
+import org.junit.jupiter.api.Test;
+
+class WireSchemaFingerprintTest {
+
+  private static final String NAMESPACE = "com.sportsbook.protocol.event.";
+  private static final Map<String, String> EXPECTED =
+      Map.ofEntries(
+          entry(NAMESPACE + "Money", "0ca10968275dbdf4"),
+          entry(NAMESPACE + "BetSettled", "113bc9d5037a850c"),
+          entry(NAMESPACE + "BetPlacedRequested", "0de97e269b734cc8"),
+          entry(NAMESPACE + "BetResolutionRevised", "b05cdf4b95651059"),
+          entry(NAMESPACE + "BetVoided", "ae914f69f90cd749"),
+          entry(NAMESPACE + "EventLifecycle", "e47d6dbd952bc721"),
+          entry(NAMESPACE + "MarketStatusChanged", "b0de225c89d1303a"),
+          entry(NAMESPACE + "MatchResult", "3f39fbc4bbfea727"),
+          entry(NAMESPACE + "OddsChanged", "4378ea6ef79a8d95"),
+          entry(NAMESPACE + "RiskLimitViolated", "0b63e276e0c81cfb"),
+          entry(NAMESPACE + "RiskPatternSuspected", "540ef62587fd31b0"),
+          entry(NAMESPACE + "WalletCredited", "31bca64c5a3a52a1"),
+          entry(NAMESPACE + "WalletDebitFailed", "10d0317115d8a749"),
+          entry(NAMESPACE + "WalletDebited", "eed98e2d2702dfe7"));
+
+  @Test
+  void wireV1CanonicalFingerprintsAreStable() throws Exception {
+    Map<String, String> actual =
+        WireSchemaTestSupport.loadSchemas().entrySet().stream()
+            .collect(
+                Collectors.toMap(
+                    Map.Entry::getKey,
+                    entry ->
+                        String.format(
+                            Locale.ROOT,
+                            "%016x",
+                            SchemaNormalization.parsingFingerprint64(entry.getValue()))));
+    assertThat(actual).containsExactlyInAnyOrderEntriesOf(EXPECTED);
+  }
+
+  @Test
+  void optionalFieldsHaveExplicitNullDefaults() throws Exception {
+    Set<String> defaulted =
+        WireSchemaTestSupport.loadSchemas().values().stream()
+            .flatMap(schema -> schema.getFields().stream().map(field -> fieldId(schema, field)))
+            .filter(FieldContract::defaulted)
+            .map(FieldContract::id)
+            .collect(Collectors.toSet());
+    assertThat(defaulted)
+        .containsExactlyInAnyOrder(
+            "BetPlacedRequested.systemMinWins",
+            "BetPlacedRequested.systemTotalSelections",
+            "BetSettled.resultDetail",
+            "MarketStatusChanged.reason");
+  }
+
+  @Test
+  void temporalFieldsUseMillisecondInstants() throws Exception {
+    Set<String> timestamps =
+        WireSchemaTestSupport.loadSchemas().values().stream()
+            .flatMap(schema -> schema.getFields().stream().map(field -> fieldId(schema, field)))
+            .filter(FieldContract::timestampMillis)
+            .map(FieldContract::id)
+            .collect(Collectors.toSet());
+    assertThat(timestamps)
+        .containsExactlyInAnyOrder(
+            "BetPlacedRequested.requestedAt",
+            "BetResolutionRevised.sourceResultSettledAt",
+            "BetResolutionRevised.revisedAt",
+            "BetSettled.settledAt",
+            "BetVoided.voidedAt",
+            "EventLifecycle.occurredAt",
+            "EventLifecycle.scheduledStartAt",
+            "MarketStatusChanged.occurredAt",
+            "MatchResult.settledAt",
+            "OddsChanged.changedAt",
+            "RiskLimitViolated.occurredAt",
+            "RiskPatternSuspected.occurredAt",
+            "WalletCredited.occurredAt",
+            "WalletDebitFailed.occurredAt",
+            "WalletDebited.occurredAt");
+  }
+
+  private static FieldContract fieldId(Schema owner, Schema.Field field) {
+    return new FieldContract(owner.getName() + "." + field.name(), field);
+  }
+
+  private record FieldContract(String id, Schema.Field field) {
+    boolean defaulted() {
+      return field.hasDefaultValue();
+    }
+
+    boolean timestampMillis() {
+      return field.schema().getLogicalType() != null
+          && field.schema().getLogicalType().getName().equals("timestamp-millis");
+    }
+  }
+}


## `docs(project): document shared protocol`

diff --git a/README.md b/README.md
index aa6b16e..da79151 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,85 @@
-# Sportsbook Shared Protocol
+# 스포츠북 공통 프로토콜
 
-Shared Java contracts for the sportsbook services.
+스포츠북 서비스가 동일한 값 객체, 오류 형식과 비동기 이벤트 계약을 사용하도록
+제공하는 Java 17 라이브러리입니다. 실행 애플리케이션이나 서비스별 업무 정책은
+포함하지 않습니다.
+
+## 기술 구성
+
+- Java 17
+- Maven Wrapper 3.9.11
+- Apache Avro 1.12
+- Jackson
+- JUnit 5와 AssertJ
+- Spotless와 Checkstyle
+
+## 빌드
+
+```sh
+./mvnw clean verify
+```
+
+다른 서비스를 빌드하기 전에 공통 artifact를 로컬 Maven 저장소에 설치합니다.
+
+```sh
+./mvnw clean install
+```
+
+Maven 좌표는 다음과 같습니다.
+
+```xml
+<dependency>
+  <groupId>com.sportsbook</groupId>
+  <artifactId>shared-protocol</artifactId>
+  <version>1.0.0</version>
+</dependency>
+```
+
+## Java 계약
+
+- `Currency`와 overflow-safe `Money`
+- 정규화된 decimal `Odds`와 American/fractional 표시 변환
+- `BetId`, `UserId`, `EventId`, `MarketId`, `SelectionId`
+- HTTP와 Kafka 경계에서 공유하는 `IdempotencyKey`
+- 단일·다중·시스템 베팅을 표현하는 `BetSlipType`
+- 구조적 불변식을 보장하는 `BetSelection`과 `BetSlip`
+- 공통 `ErrorCode`와 framework-neutral `ProblemDetail`
+
+금액 잔액, 위험 한도, 배당 허용 오차, 정산 계산 같은 업무 정책은 해당 서비스를
+소유자로 둡니다. 이 라이브러리는 여러 서비스가 공유해야 하는 데이터 모양과 기본
+불변식만 책임집니다.
+
+## Avro 계약
+
+wire v1에는 다음 14개 top-level record가 있습니다.
+
+| 영역 | record |
+| --- | --- |
+| 지갑 | `WalletDebited`, `WalletCredited`, `WalletDebitFailed` |
+| 위험 | `RiskLimitViolated`, `RiskPatternSuspected` |
+| 경기와 마켓 | `EventLifecycle`, `MarketStatusChanged`, `OddsChanged`, `MatchResult` |
+| 베팅 | `BetPlacedRequested`, `BetSettled`, `BetVoided`, `BetResolutionRevised` |
+| 공통 값 | Avro `Money` |
+
+generated Java source는 `target/generated-sources/avro`에 만들어집니다. generated
+파일을 직접 수정하지 말고 `src/main/avro`의 schema를 변경해야 합니다.
+
+### 정산 결과 정정
+
+`BetResolutionRevised`는 이미 완료된 정산 결과가 정정될 때 사용합니다.
+
+- topic: `bet.resolution.revised.v1`
+- Kafka key: `betId`
+- 최초 `BetSettled`는 논리 revision 0
+- 정정 revision은 bet별로 1부터 단조 증가
+- 이전 결과와 payout, 새 결과와 payout을 모두 포함하는 full snapshot
+- lifecycle에 의한 VOID 정정은 이 계약의 범위가 아님
+
+producer는 wallet adjustment가 확인된 뒤 durable revision과 outbox를 함께
+커밋해야 합니다. consumer는 `revisionNumber`로 중복과 역순 전달을 처리합니다.
+
+## 문서
+
+- [계약 소유권과 표현 경계](architecture/contract-ownership-and-representation-boundaries.md)
+- [이벤트 흐름과 consumer map](architecture/event-flow-and-consumer-map.md)
+- [이벤트 schema 변경 규칙](architecture/event-schema-evolution.md)
diff --git a/architecture/contract-ownership-and-representation-boundaries.md b/architecture/contract-ownership-and-representation-boundaries.md
new file mode 100644
index 0000000..d59b0ae
--- /dev/null
+++ b/architecture/contract-ownership-and-representation-boundaries.md
@@ -0,0 +1,82 @@
+# 계약 소유권과 표현 경계
+
+## 세 가지 표현
+
+동일한 업무 개념이라도 Java domain, JSON API, Avro event는 서로 다른 경계를
+가집니다.
+
+| 표현 | 목적 | 주요 제약 |
+| --- | --- | --- |
+| Java | 서비스 내부 타입 안전성 | 생성자에서 null과 구조적 불변식 검증 |
+| JSON | 동기 HTTP API | Jackson annotation과 record component가 wire shape 결정 |
+| Avro | Kafka 비동기 전달 | schema의 name, namespace, field order와 type이 계약 |
+
+표현을 서로 자동 치환 가능한 것으로 취급하지 않습니다. 각 adapter가 명시적으로
+변환하고, 경계별 테스트가 변환 결과를 검증합니다.
+
+## 금액
+
+`com.sportsbook.protocol.value.Money`는 Java 계산용 값 객체입니다.
+
+- long minor unit을 사용합니다.
+- currency가 다른 금액의 연산과 비교를 거부합니다.
+- add, subtract, multiply, negate에서 overflow를 조용히 허용하지 않습니다.
+- ledger entry 표현을 위해 음수 금액을 허용합니다.
+
+`com.sportsbook.protocol.event.Money`는 Avro record입니다. 생성된 클래스이므로
+Java 값 객체의 메서드나 검증을 갖지 않습니다. producer와 consumer adapter가
+currency와 업무별 금액 범위를 검증합니다.
+
+## 식별자와 idempotency
+
+Java API는 UUID를 그대로 주고받는 대신 typed ID를 사용합니다. 동일한 UUID라도
+`EventId`와 `MarketId`, `BetId`와 `UserId`는 서로 대입할 수 없습니다.
+JSON에서는 각 typed ID가 canonical UUID string으로 직렬화됩니다.
+
+`IdempotencyKey`는 다음 wire 조건만 보장합니다.
+
+- non-blank
+- 최대 128자
+- printable ASCII
+
+요청 결과 저장, payload fingerprint 비교, 중복 side effect 방지는 각 서비스가
+durable store와 unique constraint로 구현합니다.
+
+## 베팅 조합
+
+`BetSlip`은 모든 consumer가 안전하게 해석할 수 있는 구조만 허용합니다.
+
+- SINGLE은 selection 1개
+- MULTIPLE은 selection 2개 이상
+- SYSTEM은 `totalSelections`와 실제 selection 수가 동일
+- stake는 양수
+- SETTLED 상태는 result와 settled time을 포함
+- WON, PUSH, VOID는 payout을 포함
+- LOST는 payout을 포함하지 않음
+- 입력 selection list는 defensive copy
+
+same-event 정책, market 조합 제한, 최대 selection 수, 배당 drift 허용치는
+betting-service가 소유합니다.
+
+## 오류
+
+`ErrorCode`는 서비스 경계를 넘어 의미가 동일해야 하는 오류만 포함합니다.
+서비스 내부 전용 오류는 해당 서비스에 둡니다.
+
+`ProblemDetail`은 Spring Web 타입에 의존하지 않는 JSON record입니다. HTTP
+adapter는 이를 status와 response body로 변환하고, background consumer도 같은
+error code를 사용할 수 있습니다. optional detail, instance, correlationId는 null일
+때 JSON에서 생략됩니다.
+
+## 소유권
+
+| 계약 | 공통 라이브러리 책임 | 서비스 책임 |
+| --- | --- | --- |
+| Money | wire shape와 안전한 Java 연산 | balance, ledger, payout 정책 |
+| Odds | 값 정규화와 표시 변환 | 가격 source와 drift 허용치 |
+| BetSlip | 구조적 자기 일관성 | 승인, 위험 확인, wallet saga |
+| Event schema | 직렬화 형식 | topic 설정, publish, consume, idempotency |
+| ErrorCode | 공통 식별자와 HTTP 의미 | logging, retry, 사용자 메시지 |
+
+공통 라이브러리에 서비스 repository, Spring component, Kafka client 또는 DB model을
+추가하지 않습니다.
diff --git a/architecture/event-flow-and-consumer-map.md b/architecture/event-flow-and-consumer-map.md
new file mode 100644
index 0000000..aff006a
--- /dev/null
+++ b/architecture/event-flow-and-consumer-map.md
@@ -0,0 +1,79 @@
+# 이벤트 흐름과 consumer map
+
+## 동기 경로와 비동기 경로
+
+베팅 접수는 risk와 wallet에 동기 요청을 보내 결과를 확인합니다. 동기 결과와 별도로
+각 서비스는 경로에 맞는 저장과 전달 정책을 사용해 비동기 이벤트를 전파합니다.
+Avro record가 존재한다고 해서 해당 명령 자체가 Kafka로 처리된다는 뜻은 아닙니다.
+
+## Topic map
+
+아래 표는 sportsbook 1.0에서 구현해야 하는 cross-service topology입니다.
+shared-protocol 자체가 producer나 listener를 제공한다는 뜻은 아닙니다.
+
+| topic | record | producer | key | 1.0 대상 consumer |
+| --- | --- | --- | --- | --- |
+| `wallet.debited.v1` | `WalletDebited` | wallet | userId | 현재 first-party consumer 없음 |
+| `wallet.credited.v1` | `WalletCredited` | wallet | userId | 현재 first-party consumer 없음 |
+| `wallet.debit-failed.v1` | `WalletDebitFailed` | wallet | userId | 현재 first-party consumer 없음 |
+| `risk.limit.violated` | `RiskLimitViolated` | risk | userId | 현재 first-party consumer 없음 |
+| `risk.pattern.suspected` | `RiskPatternSuspected` | risk | userId | 현재 first-party consumer 없음 |
+| `odds.changed` | `OddsChanged` | odds-feed | eventId | gateway |
+| `market.status.changed` | `MarketStatusChanged` | odds-feed | eventId | 현재 first-party consumer 없음 |
+| `event.lifecycle` | `EventLifecycle` | odds-feed | eventId | settlement |
+| `match.result` | `MatchResult` | odds-feed | eventId | settlement |
+| `bet.placed.v1` | `BetPlacedRequested` | betting | userId | risk, settlement |
+| `bet.settled.v1` | `BetSettled` | settlement | eventId | betting, gateway |
+| `bet.voided.v1` | `BetVoided` | settlement | eventId | betting, gateway |
+| `bet.resolution.revised.v1` | `BetResolutionRevised` | settlement | betId | betting, gateway |
+
+Topic 이름은 service configuration과 orchestration topic manifest에서 동일하게
+관리합니다.
+
+## 전달 보장
+
+Kafka 전달은 중복과 경로별 보장 차이를 전제로 처리합니다.
+
+- betting, wallet, settlement의 업무 이벤트는 durable outbox를 통해 publish합니다.
+- risk 알림은 best-effort이며 odds-feed critical event는 Redis Stream에서 Kafka
+  publish를 확인한 뒤 ack합니다.
+- consumer가 존재하는 경로는 event identity 또는 업무 idempotency key로 중복
+  side effect를 막습니다.
+- side effect와 offset commit 사이의 장애를 정상적인 redelivery로 처리합니다.
+- exactly-once라는 가정에 의존하지 않습니다.
+
+Topic 내부에서는 같은 partition key에 대한 순서만 기대할 수 있습니다. 서로 다른
+topic 사이에는 전체 순서가 없습니다. lifecycle이 placement보다 먼저 도착하거나
+revision이 최초 projection보다 먼저 도착하는 경우도 consumer 상태 전이에서
+처리해야 합니다.
+
+## 정산과 정정
+
+`BetSettled`와 `BetVoided`는 최초 terminal projection을 만듭니다.
+`BetResolutionRevised`는 SETTLED 결과 정정에만 사용합니다.
+
+정정 producer의 순서는 다음과 같습니다.
+
+1. corrected match result를 승인합니다.
+2. 기존 payout과 새 payout의 차이를 계산합니다.
+3. wallet adjustment를 idempotent operation으로 완료합니다.
+4. revision과 outbox를 동일 transaction으로 저장합니다.
+5. `bet.resolution.revised.v1`에 betId key로 publish합니다.
+
+consumer는 저장된 revision number를 기준으로 다음과 같이 동작합니다.
+
+- 더 작은 revision은 무시
+- 같은 revision과 같은 payload는 no-op
+- 같은 revision과 다른 payload는 conflict로 격리
+- 더 큰 revision은 full snapshot으로 projection 교체
+- revision이 먼저 도착한 뒤 최초 settlement가 도착하면 revision 0 이벤트를 무시
+
+gateway가 client에 revision을 전달할 때도 revision id와 number를 포함해야 client가
+역순 WebSocket message를 구분할 수 있습니다.
+
+## Redis와 Kafka
+
+odds-feed의 Redis projection은 betting-service의 동기 가격 조회에 사용됩니다.
+Kafka odds event는 gateway push와 다른 비동기 consumer를 위한 것입니다. Redis
+write 성공과 Kafka publish 성공은 같은 보장을 제공하지 않으므로, critical-event
+queue와 readiness가 delivery gap을 드러내야 합니다.
diff --git a/architecture/event-schema-evolution.md b/architecture/event-schema-evolution.md
new file mode 100644
index 0000000..a2aa537
--- /dev/null
+++ b/architecture/event-schema-evolution.md
@@ -0,0 +1,72 @@
+# 이벤트 schema 변경 규칙
+
+## Runtime 조건
+
+서비스는 Schema Registry나 payload 안의 writer schema id 없이 Avro binary를
+사용합니다. consumer는 자신이 가진 generated class schema로 payload를 읽습니다.
+따라서 일반적인 Avro 호환성 판정만으로 mixed deployment의 안전성을 보장할 수
+없습니다.
+
+wire v1의 기존 record는 다음 요소를 고정합니다.
+
+- record name과 namespace
+- field name과 순서
+- primitive, collection, union과 named type
+- enum symbol과 순서
+- default 존재 여부와 값
+- timestamp logical type
+
+기존 record에 optional field를 추가하는 방식도 현재 codec에서는 안전한 rolling
+변경으로 간주하지 않습니다. 새로운 의미가 필요하면 별도 record와 topic을
+추가합니다.
+
+## Named schema
+
+`Money`와 `SettlementResultAvro`는 여러 event가 공유하는 named type입니다.
+`SettlementResultAvro`는 `BetSettled.avsc`에서 정의됩니다. 같은 fullname의 enum을
+다른 schema에서 다시 정의하면 parser와 generated source가 충돌합니다.
+
+Avro Maven Plugin은 다음 import 순서를 사용합니다.
+
+1. `Money.avsc`
+2. `BetSettled.avsc`
+3. 나머지 schema
+
+`BetResolutionRevised`는 두 named type을 이름으로 재사용합니다.
+
+## Contract test
+
+빌드는 다음 경계를 검증합니다.
+
+- top-level record inventory가 정확히 14개인지 확인
+- 각 generated SpecificRecord의 field 순서 확인
+- required와 explicit null default 구분
+- timestamp-millis logical type 확인
+- named type fullname과 enum 재사용 확인
+- SpecificDatumWriter/Reader binary round trip
+- CRC-64-AVRO parsing canonical fingerprint 고정
+
+Canonical fingerprint는 field의 구조적 계약을 빠르게 감지하지만 doc, default,
+logical type의 모든 의미를 보존하지 않습니다. 그래서 default와 logical type을
+별도 assertion으로 검증합니다.
+
+## 변경 절차
+
+호환 가능한 기능 추가는 다음 순서를 따릅니다.
+
+1. 새 schema와 전용 topic/key 계약을 정의합니다.
+2. schema inventory, field, named type, binary round-trip 테스트를 추가합니다.
+3. orchestration에서 topic을 생성합니다.
+4. consumer를 먼저 배포하고 unknown message를 받지 않는 상태를 확인합니다.
+5. producer를 활성화합니다.
+6. duplicate, retry, out-of-order와 replay 불변성을 검증합니다.
+
+기존 topic을 대체해야 한다면 새 generation topic을 만들고 consumer 전환이 완료된
+뒤 producer를 이동합니다. topic 이름만 유지한 채 incompatible payload로 바꾸지
+않습니다.
+
+## Generated source
+
+`target/generated-sources/avro`는 build output입니다. source control에 넣거나 직접
+수정하지 않습니다. schema 변경 후에는 반드시 clean build로 stale generated class가
+남지 않음을 확인합니다.
