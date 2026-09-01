# 원시 Kafka 경계와 정확한 DLT 복구

## `feat(avro): decode one exact raw record`

diff --git a/pom.xml b/pom.xml
index 381d40c..6e5a78d 100644
--- a/pom.xml
+++ b/pom.xml
@@ -20,6 +20,7 @@
         <maven.compiler.release>17</maven.compiler.release>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
+        <avro.version>1.12.0</avro.version>
         <compiler.version>3.13.0</compiler.version>
         <surefire.version>3.5.1</surefire.version>
         <junit.jupiter.version>5.11.3</junit.jupiter.version>
@@ -65,6 +66,11 @@
             <artifactId>shared-protocol</artifactId>
             <version>${shared-protocol.version}</version>
         </dependency>
+        <dependency>
+            <groupId>org.apache.avro</groupId>
+            <artifactId>avro</artifactId>
+            <version>${avro.version}</version>
+        </dependency>
         <dependency>
             <groupId>org.junit.jupiter</groupId>
             <artifactId>junit-jupiter</artifactId>
diff --git a/src/main/java/com/sportsbook/settlement/event/StrictAvroDecoder.java b/src/main/java/com/sportsbook/settlement/event/StrictAvroDecoder.java
new file mode 100644
index 0000000..e109652
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/StrictAvroDecoder.java
@@ -0,0 +1,43 @@
+package com.sportsbook.settlement.event;
+
+import java.util.Objects;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificRecordBase;
+
+/** Decodes one exact raw Avro record and rejects every trailing byte. */
+public final class StrictAvroDecoder {
+
+  public <T extends SpecificRecordBase> T decode(byte[] payload, Class<T> type) {
+    Objects.requireNonNull(type, "type");
+    if (payload == null) {
+      throw new DecodeException("Null " + type.getSimpleName() + " Avro payload");
+    }
+    try {
+      T template = type.getDeclaredConstructor().newInstance();
+      SpecificDatumReader<T> reader = new SpecificDatumReader<>(template.getSchema());
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(payload, null);
+      T decoded = reader.read(null, decoder);
+      if (!decoder.isEnd()) {
+        throw new DecodeException("Trailing bytes after " + type.getSimpleName());
+      }
+      return decoded;
+    } catch (DecodeException exception) {
+      throw exception;
+    } catch (Exception exception) {
+      throw new DecodeException("Invalid " + type.getSimpleName() + " Avro payload", exception);
+    }
+  }
+
+  public static final class DecodeException extends RuntimeException {
+
+    public DecodeException(String message) {
+      super(message);
+    }
+
+    public DecodeException(String message, Throwable cause) {
+      super(message, cause);
+    }
+  }
+}


## `test(avro): reject malformed and trailing payloads`

diff --git a/src/test/java/com/sportsbook/settlement/event/StrictAvroDecoderTest.java b/src/test/java/com/sportsbook/settlement/event/StrictAvroDecoderTest.java
new file mode 100644
index 0000000..45795d8
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/StrictAvroDecoderTest.java
@@ -0,0 +1,55 @@
+package com.sportsbook.settlement.event;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.Money;
+import java.io.ByteArrayOutputStream;
+import java.util.Arrays;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.junit.jupiter.api.Test;
+
+class StrictAvroDecoderTest {
+
+  private final StrictAvroDecoder decoder = new StrictAvroDecoder();
+
+  @Test
+  void decodesExactlyOneRawSpecificRecord() {
+    Money source = Money.newBuilder().setAmount(125L).setCurrency("KRW").build();
+
+    Money decoded = decoder.decode(encode(source), Money.class);
+
+    assertThat(decoded.getAmount()).isEqualTo(125L);
+    assertThat(decoded.getCurrency().toString()).isEqualTo("KRW");
+  }
+
+  @Test
+  void rejectsMalformedAndTrailingBytes() {
+    Money source = Money.newBuilder().setAmount(125L).setCurrency("KRW").build();
+    byte[] encoded = encode(source);
+    byte[] trailing = Arrays.copyOf(encoded, encoded.length + 1);
+
+    assertThatThrownBy(() -> decoder.decode(new byte[0], Money.class))
+        .isInstanceOf(StrictAvroDecoder.DecodeException.class);
+    assertThatThrownBy(() -> decoder.decode(null, Money.class))
+        .isInstanceOf(StrictAvroDecoder.DecodeException.class)
+        .hasMessageContaining("Null Money");
+    assertThatThrownBy(() -> decoder.decode(trailing, Money.class))
+        .isInstanceOf(StrictAvroDecoder.DecodeException.class)
+        .hasMessageContaining("Trailing bytes");
+  }
+
+  private static byte[] encode(Money record) {
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<Money>(record.getSchema()).write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (Exception exception) {
+      throw new AssertionError(exception);
+    }
+  }
+}


## `feat(kafka): validate canonical raw UUID keys`

diff --git a/src/main/java/com/sportsbook/settlement/event/KafkaUuidKeyValidator.java b/src/main/java/com/sportsbook/settlement/event/KafkaUuidKeyValidator.java
new file mode 100644
index 0000000..6429c4f
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/KafkaUuidKeyValidator.java
@@ -0,0 +1,52 @@
+package com.sportsbook.settlement.event;
+
+import java.nio.ByteBuffer;
+import java.nio.charset.CharacterCodingException;
+import java.nio.charset.CodingErrorAction;
+import java.nio.charset.StandardCharsets;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Validates a raw Kafka key as one strict canonical lowercase UUID. */
+public final class KafkaUuidKeyValidator {
+
+  private static final int UUID_TEXT_LENGTH = 36;
+
+  public UUID requireMatching(byte[] rawKey, CharSequence recordField, String fieldName) {
+    Objects.requireNonNull(fieldName, "fieldName");
+    if (rawKey == null || rawKey.length != UUID_TEXT_LENGTH) {
+      throw invalid(fieldName, "key must be 36 raw UTF-8 bytes");
+    }
+    String text = decode(rawKey, fieldName);
+    UUID parsed;
+    try {
+      parsed = UUID.fromString(text);
+    } catch (IllegalArgumentException exception) {
+      throw invalid(fieldName, "key is not a UUID");
+    }
+    if (!parsed.toString().equals(text)) {
+      throw invalid(fieldName, "key is not canonical lowercase UUID text");
+    }
+    if (recordField == null || !text.contentEquals(recordField)) {
+      throw invalid(fieldName, "key does not match record field");
+    }
+    return parsed;
+  }
+
+  private static String decode(byte[] rawKey, String fieldName) {
+    try {
+      return StandardCharsets.UTF_8
+          .newDecoder()
+          .onMalformedInput(CodingErrorAction.REPORT)
+          .onUnmappableCharacter(CodingErrorAction.REPORT)
+          .decode(ByteBuffer.wrap(rawKey))
+          .toString();
+    } catch (CharacterCodingException exception) {
+      throw invalid(fieldName, "key is not strict UTF-8");
+    }
+  }
+
+  private static IllegalArgumentException invalid(String field, String reason) {
+    return new IllegalArgumentException("Invalid Kafka " + field + ": " + reason);
+  }
+}


## `test(kafka): reject noncanonical and mismatched keys`

diff --git a/src/test/java/com/sportsbook/settlement/event/KafkaUuidKeyValidatorTest.java b/src/test/java/com/sportsbook/settlement/event/KafkaUuidKeyValidatorTest.java
new file mode 100644
index 0000000..e04d63b
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/event/KafkaUuidKeyValidatorTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.settlement.event;
+
+import static java.nio.charset.StandardCharsets.UTF_8;
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class KafkaUuidKeyValidatorTest {
+
+  private final KafkaUuidKeyValidator validator = new KafkaUuidKeyValidator();
+
+  @Test
+  void acceptsOnlyAKeyMatchingTheRecordField() {
+    UUID id = UUID.randomUUID();
+
+    assertThat(validator.requireMatching(id.toString().getBytes(UTF_8), id.toString(), "userId"))
+        .isEqualTo(id);
+  }
+
+  @Test
+  void rejectsNoncanonicalMalformedAndMismatchedKeys() {
+    UUID id = UUID.randomUUID();
+    byte[] uppercase = id.toString().toUpperCase(java.util.Locale.ROOT).getBytes(UTF_8);
+    byte[] malformed = id.toString().getBytes(UTF_8);
+    malformed[0] = (byte) 0xff;
+
+    assertInvalid(null, id.toString());
+    assertInvalid(uppercase, id.toString());
+    assertInvalid(malformed, id.toString());
+    assertInvalid(id.toString().getBytes(UTF_8), UUID.randomUUID().toString());
+  }
+
+  private void assertInvalid(byte[] key, String field) {
+    assertThatThrownBy(() -> validator.requireMatching(key, field, "userId"))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageStartingWith("Invalid Kafka userId:");
+  }
+}


## `feat(kafka): classify permanent and transient failures`

diff --git a/src/main/java/com/sportsbook/settlement/event/MessageFailureClassifier.java b/src/main/java/com/sportsbook/settlement/event/MessageFailureClassifier.java
new file mode 100644
index 0000000..ed7efda
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/MessageFailureClassifier.java
@@ -0,0 +1,27 @@
+package com.sportsbook.settlement.event;
+
+import java.util.Collections;
+import java.util.IdentityHashMap;
+import java.util.Set;
+
+/** Separates poison contracts from failures that must leave the source offset retryable. */
+public final class MessageFailureClassifier {
+
+  public enum Disposition {
+    PERMANENT,
+    TRANSIENT
+  }
+
+  public Disposition classify(Throwable failure) {
+    Set<Throwable> seen = Collections.newSetFromMap(new IdentityHashMap<>());
+    Throwable current = failure;
+    while (current != null && seen.add(current)) {
+      if (current instanceof IllegalArgumentException
+          || current instanceof StrictAvroDecoder.DecodeException) {
+        return Disposition.PERMANENT;
+      }
+      current = current.getCause();
+    }
+    return Disposition.TRANSIENT;
+  }
+}


## `feat(kafka): bound transient listener retries`

diff --git a/src/main/java/com/sportsbook/settlement/event/KafkaRetryPolicy.java b/src/main/java/com/sportsbook/settlement/event/KafkaRetryPolicy.java
new file mode 100644
index 0000000..99a3bf8
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/KafkaRetryPolicy.java
@@ -0,0 +1,32 @@
+package com.sportsbook.settlement.event;
+
+import com.sportsbook.settlement.event.MessageFailureClassifier.Disposition;
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.util.backoff.BackOff;
+import org.springframework.util.backoff.FixedBackOff;
+
+/** Bounded listener retry policy; permanent contracts have no retry cycle. */
+@ConfigurationProperties("settlement.kafka.retry")
+public record KafkaRetryPolicy(int maxAttempts, Duration backoff, Duration dltSendTimeout) {
+
+  public KafkaRetryPolicy {
+    maxAttempts = maxAttempts == 0 ? 3 : maxAttempts;
+    backoff = backoff == null ? Duration.ofSeconds(1) : backoff;
+    dltSendTimeout = dltSendTimeout == null ? Duration.ofSeconds(11) : dltSendTimeout;
+    if (maxAttempts < 1
+        || maxAttempts > 10
+        || backoff.isNegative()
+        || dltSendTimeout.isZero()
+        || dltSendTimeout.isNegative()) {
+      throw new IllegalArgumentException("Invalid settlement Kafka retry policy");
+    }
+  }
+
+  public BackOff backOffFor(Throwable failure, MessageFailureClassifier classifier) {
+    if (classifier.classify(failure) == Disposition.PERMANENT) {
+      return new FixedBackOff(0, 0);
+    }
+    return new FixedBackOff(backoff.toMillis(), maxAttempts - 1L);
+  }
+}


## `feat(kafka): configure bounded raw DLT producer`

diff --git a/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java b/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java
new file mode 100644
index 0000000..6a35dc0
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/config/RawKafkaProducerConfiguration.java
@@ -0,0 +1,47 @@
+package com.sportsbook.settlement.config;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaOperations;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+/** Dedicated bounded raw-byte producer for exact dead-letter publication. */
+@Configuration
+public class RawKafkaProducerConfiguration {
+
+  public static final String PRODUCER_FACTORY = "settlementRawProducerFactory";
+  public static final String OPERATIONS = "settlementRawKafkaOperations";
+
+  @Bean(PRODUCER_FACTORY)
+  ProducerFactory<byte[], byte[]> settlementRawProducerFactory(KafkaProperties properties) {
+    return new DefaultKafkaProducerFactory<>(producerProperties(properties));
+  }
+
+  @Bean(OPERATIONS)
+  KafkaOperations<byte[], byte[]> settlementRawKafkaOperations(
+      @Qualifier(PRODUCER_FACTORY) ProducerFactory<byte[], byte[]> producerFactory) {
+    return new KafkaTemplate<>(producerFactory);
+  }
+
+  static Map<String, Object> producerProperties(KafkaProperties properties) {
+    Map<String, Object> configured = new HashMap<>(properties.buildProducerProperties());
+    configured.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configured.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configured.put(ProducerConfig.ACKS_CONFIG, "all");
+    configured.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    configured.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
+    configured.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 5_000);
+    configured.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 5_000);
+    configured.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 10_000);
+    configured.put(ProducerConfig.CLIENT_ID_CONFIG, "settlement-service-dlt");
+    return Map.copyOf(configured);
+  }
+}


## `feat(kafka): preserve raw records in exact DLT partitions`

diff --git a/pom.xml b/pom.xml
index 6e5a78d..04f8c98 100644
--- a/pom.xml
+++ b/pom.xml
@@ -71,6 +71,10 @@
             <artifactId>avro</artifactId>
             <version>${avro.version}</version>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.kafka</groupId>
+            <artifactId>spring-kafka</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.junit.jupiter</groupId>
             <artifactId>junit-jupiter</artifactId>
diff --git a/src/main/java/com/sportsbook/settlement/event/ExactDeadLetterRecoverer.java b/src/main/java/com/sportsbook/settlement/event/ExactDeadLetterRecoverer.java
new file mode 100644
index 0000000..653d11b
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/event/ExactDeadLetterRecoverer.java
@@ -0,0 +1,74 @@
+package com.sportsbook.settlement.event;
+
+import java.nio.ByteBuffer;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.Objects;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.TimeoutException;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.header.internals.RecordHeaders;
+import org.springframework.kafka.KafkaException;
+import org.springframework.kafka.core.KafkaOperations;
+
+/** Republishes a poison record to the exact corresponding DLT partition with raw bytes intact. */
+public final class ExactDeadLetterRecoverer {
+
+  static final String ORIGINAL_TOPIC = "settlement-dlt-original-topic";
+  static final String ORIGINAL_PARTITION = "settlement-dlt-original-partition";
+  static final String ORIGINAL_OFFSET = "settlement-dlt-original-offset";
+  static final String ORIGINAL_TIMESTAMP = "settlement-dlt-original-timestamp";
+  static final String CONSUMER_GROUP = "settlement-dlt-consumer-group";
+  static final String EXCEPTION_TYPE = "settlement-dlt-exception-type";
+
+  private final KafkaOperations<byte[], byte[]> kafka;
+  private final String consumerGroup;
+  private final Duration sendTimeout;
+
+  public ExactDeadLetterRecoverer(
+      KafkaOperations<byte[], byte[]> kafka, String consumerGroup, Duration sendTimeout) {
+    this.kafka = Objects.requireNonNull(kafka, "kafka");
+    this.consumerGroup = Objects.requireNonNull(consumerGroup, "consumerGroup");
+    this.sendTimeout = Objects.requireNonNull(sendTimeout, "sendTimeout");
+  }
+
+  public void recover(ConsumerRecord<byte[], byte[]> source, Exception failure) {
+    RecordHeaders headers = new RecordHeaders(source.headers());
+    headers.add(ORIGINAL_TOPIC, utf8(source.topic()));
+    headers.add(ORIGINAL_PARTITION, integer(source.partition()));
+    headers.add(ORIGINAL_OFFSET, longBytes(source.offset()));
+    headers.add(ORIGINAL_TIMESTAMP, longBytes(source.timestamp()));
+    headers.add(CONSUMER_GROUP, utf8(consumerGroup));
+    headers.add(EXCEPTION_TYPE, utf8(failure.getClass().getName()));
+    ProducerRecord<byte[], byte[]> deadLetter =
+        new ProducerRecord<>(
+            source.topic() + ".DLT",
+            source.partition(),
+            null,
+            source.key(),
+            source.value(),
+            headers);
+    try {
+      kafka.send(deadLetter).get(sendTimeout.toMillis(), TimeUnit.MILLISECONDS);
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+      throw new KafkaException("Interrupted while publishing exact DLT record", exception);
+    } catch (ExecutionException | TimeoutException exception) {
+      throw new KafkaException("Failed to publish exact DLT record", exception);
+    }
+  }
+
+  private static byte[] utf8(String value) {
+    return value.getBytes(StandardCharsets.UTF_8);
+  }
+
+  private static byte[] integer(int value) {
+    return ByteBuffer.allocate(Integer.BYTES).putInt(value).array();
+  }
+
+  private static byte[] longBytes(long value) {
+    return ByteBuffer.allocate(Long.BYTES).putLong(value).array();
+  }
+}


## `feat(kafka): wire retry recovery handler`

diff --git a/src/main/java/com/sportsbook/settlement/config/KafkaRecoveryConfiguration.java b/src/main/java/com/sportsbook/settlement/config/KafkaRecoveryConfiguration.java
new file mode 100644
index 0000000..f472fc5
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/config/KafkaRecoveryConfiguration.java
@@ -0,0 +1,68 @@
+package com.sportsbook.settlement.config;
+
+import com.sportsbook.settlement.event.ExactDeadLetterRecoverer;
+import com.sportsbook.settlement.event.KafkaRetryPolicy;
+import com.sportsbook.settlement.event.MessageFailureClassifier;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.KafkaException;
+import org.springframework.kafka.core.KafkaOperations;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+
+/** Connects failure classification, bounded retries, and exact dead-letter recovery. */
+@Configuration
+public class KafkaRecoveryConfiguration {
+
+  static final String CONSUMER_GROUP = "settlement-service";
+
+  @Bean
+  MessageFailureClassifier messageFailureClassifier() {
+    return new MessageFailureClassifier();
+  }
+
+  @Bean
+  ExactDeadLetterRecoverer exactDeadLetterRecoverer(
+      @Qualifier(RawKafkaProducerConfiguration.OPERATIONS)
+          KafkaOperations<byte[], byte[]> operations,
+      KafkaRetryPolicy policy) {
+    return new ExactDeadLetterRecoverer(operations, CONSUMER_GROUP, policy.dltSendTimeout());
+  }
+
+  @Bean
+  DefaultErrorHandler settlementKafkaErrorHandler(
+      ExactDeadLetterRecoverer recoverer,
+      KafkaRetryPolicy policy,
+      MessageFailureClassifier classifier) {
+    DefaultErrorHandler handler =
+        new DefaultErrorHandler(
+            (record, failure) -> recover(record, failure, recoverer, classifier),
+            policy.backOffFor(new IllegalStateException("initial"), classifier));
+    handler.setBackOffFunction((record, failure) -> policy.backOffFor(failure, classifier));
+    handler.setCommitRecovered(true);
+    handler.setAckAfterHandle(true);
+    handler.setResetStateOnRecoveryFailure(true);
+    return handler;
+  }
+
+  static void recover(
+      ConsumerRecord<?, ?> record,
+      Exception failure,
+      ExactDeadLetterRecoverer recoverer,
+      MessageFailureClassifier classifier) {
+    if (classifier.classify(failure) != MessageFailureClassifier.Disposition.PERMANENT) {
+      throw new KafkaException("Transient listener failure remains uncommitted", failure);
+    }
+    recoverer.recover(rawRecord(record), failure);
+  }
+
+  @SuppressWarnings("unchecked")
+  static ConsumerRecord<byte[], byte[]> rawRecord(ConsumerRecord<?, ?> record) {
+    if ((record.key() != null && !(record.key() instanceof byte[]))
+        || (record.value() != null && !(record.value() instanceof byte[]))) {
+      throw new KafkaException("Settlement listeners require raw byte keys and values");
+    }
+    return (ConsumerRecord<byte[], byte[]>) record;
+  }
+}


