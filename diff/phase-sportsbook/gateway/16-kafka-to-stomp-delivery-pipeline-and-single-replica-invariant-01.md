# Kafka-to-STOMP 전달 파이프라인과 단일 복제본 불변 조건

## `feat(websocket): configure STOMP transport`

diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
new file mode 100644
index 0000000..a800280
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -0,0 +1,46 @@
+package com.sportsbook.gateway.ws;
+
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.messaging.simp.config.MessageBrokerRegistry;
+import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
+import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
+import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
+import org.springframework.web.socket.config.annotation.WebSocketTransportRegistration;
+
+@Configuration
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+@EnableWebSocketMessageBroker
+public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
+
+  private static final int MESSAGE_SIZE_LIMIT = 64 * 1024;
+  private static final int SEND_BUFFER_LIMIT = 512 * 1024;
+  private static final int SEND_TIME_LIMIT = 10_000;
+
+  private final String[] allowedOrigins;
+
+  public WebSocketConfig(@Value("${gateway.ws.allowed-origins}") String[] allowedOrigins) {
+    this.allowedOrigins = allowedOrigins;
+  }
+
+  @Override
+  public void registerStompEndpoints(StompEndpointRegistry registry) {
+    registry.addEndpoint("/ws/v1/odds", "/ws/v1/bets").setAllowedOriginPatterns(allowedOrigins);
+  }
+
+  @Override
+  public void configureMessageBroker(MessageBrokerRegistry registry) {
+    registry.enableSimpleBroker("/topic", "/queue");
+    registry.setApplicationDestinationPrefixes("/app");
+    registry.setUserDestinationPrefix("/user");
+  }
+
+  @Override
+  public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
+    registration
+        .setMessageSizeLimit(MESSAGE_SIZE_LIMIT)
+        .setSendBufferSizeLimit(SEND_BUFFER_LIMIT)
+        .setSendTimeLimit(SEND_TIME_LIMIT);
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 81faa7d..61fe403 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -42,3 +42,5 @@ gateway:
       capacity: ${GATEWAY_RATELIMIT_IP_CAPACITY:60}
       refill-period: ${GATEWAY_RATELIMIT_IP_PERIOD:1m}
     trusted-proxy-cidrs: ${GATEWAY_TRUSTED_PROXY_CIDRS:}
+  ws:
+    allowed-origins: ${GATEWAY_WS_ALLOWED_ORIGINS:http://localhost:[*],http://127.0.0.1:[*]}


## `feat(websocket): track raw WebSocket sessions`

diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
index b1f3853..8ea8096 100644
--- a/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketConfig.java
@@ -21,12 +21,15 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
 
   private final String[] allowedOrigins;
   private final StompAuthChannelInterceptor authentication;
+  private final WebSocketSessionRegistry sessions;
 
   public WebSocketConfig(
       @Value("${gateway.ws.allowed-origins}") String[] allowedOrigins,
-      StompAuthChannelInterceptor authentication) {
+      StompAuthChannelInterceptor authentication,
+      WebSocketSessionRegistry sessions) {
     this.allowedOrigins = allowedOrigins;
     this.authentication = authentication;
+    this.sessions = sessions;
   }
 
   @Override
@@ -51,6 +54,7 @@ public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
     registration
         .setMessageSizeLimit(MESSAGE_SIZE_LIMIT)
         .setSendBufferSizeLimit(SEND_BUFFER_LIMIT)
-        .setSendTimeLimit(SEND_TIME_LIMIT);
+        .setSendTimeLimit(SEND_TIME_LIMIT)
+        .addDecoratorFactory(sessions);
   }
 }
diff --git a/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
new file mode 100644
index 0000000..8c80d5a
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/WebSocketSessionRegistry.java
@@ -0,0 +1,46 @@
+package com.sportsbook.gateway.ws;
+
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.concurrent.ConcurrentMap;
+import org.springframework.stereotype.Component;
+import org.springframework.web.socket.CloseStatus;
+import org.springframework.web.socket.WebSocketHandler;
+import org.springframework.web.socket.WebSocketSession;
+import org.springframework.web.socket.handler.WebSocketHandlerDecorator;
+import org.springframework.web.socket.handler.WebSocketHandlerDecoratorFactory;
+
+@Component
+public class WebSocketSessionRegistry implements WebSocketHandlerDecoratorFactory {
+
+  private final ConcurrentMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
+
+  @Override
+  public WebSocketHandler decorate(WebSocketHandler handler) {
+    return new WebSocketHandlerDecorator(handler) {
+      @Override
+      public void afterConnectionEstablished(WebSocketSession session) throws Exception {
+        sessions.put(session.getId(), session);
+        try {
+          super.afterConnectionEstablished(session);
+        } catch (Exception failure) {
+          sessions.remove(session.getId(), session);
+          throw failure;
+        }
+      }
+
+      @Override
+      public void afterConnectionClosed(WebSocketSession session, CloseStatus status)
+          throws Exception {
+        try {
+          super.afterConnectionClosed(session, status);
+        } finally {
+          sessions.remove(session.getId(), session);
+        }
+      }
+    };
+  }
+
+  int size() {
+    return sessions.size();
+  }
+}


## `feat(kafka): consume raw event records`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java
new file mode 100644
index 0000000..799b7c3
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java
@@ -0,0 +1,47 @@
+package com.sportsbook.gateway.kafka;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.springframework.beans.factory.ObjectProvider;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.boot.ssl.SslBundles;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.annotation.EnableKafka;
+import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
+import org.springframework.kafka.core.ConsumerFactory;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.listener.ContainerProperties;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+
+/** Configures event consumers to retain Kafka keys and values as their original bytes. */
+@EnableKafka
+@Configuration(proxyBeanMethods = false)
+public class GatewayKafkaConsumerConfiguration {
+
+  @Bean
+  ConsumerFactory<byte[], byte[]> gatewayConsumerFactory(
+      KafkaProperties properties, SslBundles sslBundles) {
+    Map<String, Object> configuration =
+        new HashMap<>(properties.buildConsumerProperties(sslBundles));
+    configuration.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
+    configuration.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    configuration.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    return new DefaultKafkaConsumerFactory<>(configuration);
+  }
+
+  @Bean
+  ConcurrentKafkaListenerContainerFactory<byte[], byte[]> kafkaListenerContainerFactory(
+      ConsumerFactory<byte[], byte[]> gatewayConsumerFactory,
+      ObjectProvider<DefaultErrorHandler> errorHandler) {
+    ConcurrentKafkaListenerContainerFactory<byte[], byte[]> factory =
+        new ConcurrentKafkaListenerContainerFactory<>();
+    factory.setConsumerFactory(gatewayConsumerFactory);
+    factory.setBatchListener(false);
+    errorHandler.ifAvailable(factory::setCommonErrorHandler);
+    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
+    return factory;
+  }
+}


## `feat(kafka): bound dead-letter publication`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java
new file mode 100644
index 0000000..a05ce59
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java
@@ -0,0 +1,55 @@
+package com.sportsbook.gateway.kafka;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.boot.ssl.SslBundles;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+
+/** Publishes failed source records to a bounded, same-partition dead-letter destination. */
+@Configuration(proxyBeanMethods = false)
+public class GatewayDeadLetterConfiguration {
+
+  @Bean
+  ProducerFactory<byte[], byte[]> gatewayProducerFactory(
+      KafkaProperties properties, SslBundles sslBundles) {
+    Map<String, Object> configuration =
+        new HashMap<>(properties.buildProducerProperties(sslBundles));
+    configuration.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configuration.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configuration.put(ProducerConfig.ACKS_CONFIG, "all");
+    configuration.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    return new DefaultKafkaProducerFactory<>(configuration);
+  }
+
+  @Bean
+  KafkaTemplate<byte[], byte[]> gatewayKafkaTemplate(
+      ProducerFactory<byte[], byte[]> gatewayProducerFactory) {
+    return new KafkaTemplate<>(gatewayProducerFactory);
+  }
+
+  @Bean
+  DeadLetterPublishingRecoverer gatewayDeadLetterRecoverer(
+      KafkaTemplate<byte[], byte[]> gatewayKafkaTemplate,
+      GatewayTopicProperties topics,
+      GatewayKafkaProperties properties) {
+    DeadLetterPublishingRecoverer recoverer =
+        new DeadLetterPublishingRecoverer(
+            gatewayKafkaTemplate,
+            (record, exception) ->
+                topics.deadLetterDestination(record.topic(), record.partition()));
+    recoverer.setFailIfSendResultIsError(true);
+    recoverer.setWaitForSendResultTimeout(properties.dltWaitTimeout());
+    recoverer.setTimeoutBuffer(properties.dltTimeoutBuffer().toMillis());
+    recoverer.setVerifyPartition(false);
+    recoverer.setStripPreviousExceptionHeaders(true);
+    return recoverer;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
new file mode 100644
index 0000000..7cf8a37
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.kafka;
+
+import java.time.Duration;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.boot.context.properties.bind.DefaultValue;
+
+/** Bounded publication settings for failed raw event records. */
+@ConfigurationProperties(prefix = "gateway.kafka")
+public record GatewayKafkaProperties(
+    @DefaultValue("11s") Duration dltWaitTimeout,
+    @DefaultValue("1s") Duration dltTimeoutBuffer) {
+
+  public GatewayKafkaProperties {
+    requirePositive(dltWaitTimeout, "dlt-wait-timeout");
+    requirePositive(dltTimeoutBuffer, "dlt-timeout-buffer");
+  }
+
+  private static void requirePositive(Duration value, String property) {
+    if (value == null || value.isZero() || value.isNegative()) {
+      throw new IllegalArgumentException("gateway.kafka." + property + " must be positive");
+    }
+  }
+}


## `feat(events): decode strict Avro records`

diff --git a/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java b/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java
new file mode 100644
index 0000000..69f3e56
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java
@@ -0,0 +1,36 @@
+package com.sportsbook.gateway.events;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import java.io.IOException;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.DatumReader;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificRecord;
+
+/** Decodes one complete binary Avro record using its shared protocol generated type. */
+public final class StrictAvroDecoder {
+
+  public static <T extends SpecificRecord> T decode(byte[] payload, Class<T> type) {
+    if (payload == null) {
+      throw new GatewayEventContractException("Kafka event payload must not be null");
+    }
+    try {
+      DatumReader<T> reader = new SpecificDatumReader<>(type);
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(payload, null);
+      T decoded = reader.read(null, decoder);
+      if (!decoder.isEnd()) {
+        throw new GatewayEventContractException(
+            "Avro record has trailing bytes: " + type.getName());
+      }
+      return decoded;
+    } catch (GatewayEventContractException failure) {
+      throw failure;
+    } catch (IOException | RuntimeException failure) {
+      throw new GatewayEventContractException(
+          "Failed to decode Avro record " + type.getName(), failure);
+    }
+  }
+
+  private StrictAvroDecoder() {}
+}


## `feat(websocket): publish odds updates`

diff --git a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
new file mode 100644
index 0000000..b43e087
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
@@ -0,0 +1,22 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.protocol.event.OddsChanged;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.messaging.simp.SimpMessagingTemplate;
+import org.springframework.stereotype.Component;
+
+/** Hands validated Kafka events to the local STOMP broker. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class GatewayPushPublisher {
+
+  private final SimpMessagingTemplate messaging;
+
+  public GatewayPushPublisher(SimpMessagingTemplate messaging) {
+    this.messaging = messaging;
+  }
+
+  public void publishOdds(OddsChanged event) {
+    messaging.convertAndSend("/topic/odds/" + event.getEventId(), OddsUpdate.from(event));
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java b/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java
new file mode 100644
index 0000000..6ae506d
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/OddsStreamListener.java
@@ -0,0 +1,31 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.gateway.events.GatewayEventContract;
+import com.sportsbook.protocol.event.OddsChanged;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+/** Publishes validated odds events to their public event stream. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class OddsStreamListener {
+
+  private final GatewayPushPublisher publisher;
+
+  public OddsStreamListener(GatewayPushPublisher publisher) {
+    this.publisher = publisher;
+  }
+
+  @KafkaListener(
+      id = "gateway-odds-listener",
+      topics = "${gateway.topics.odds-changed}",
+      groupId = "gateway-odds",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onOddsChanged(ConsumerRecord<byte[], byte[]> record) {
+    OddsChanged event = GatewayEventContract.oddsChanged(record);
+    publisher.publishOdds(event);
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java b/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java
new file mode 100644
index 0000000..e3765e5
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/OddsUpdate.java
@@ -0,0 +1,24 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.protocol.event.OddsChanged;
+import java.time.Instant;
+
+/** Public odds projection delivered on an event-specific STOMP topic. */
+public record OddsUpdate(
+    String eventId,
+    String marketId,
+    String selectionId,
+    String previousOdds,
+    String newOdds,
+    Instant changedAt) {
+
+  static OddsUpdate from(OddsChanged event) {
+    return new OddsUpdate(
+        event.getEventId(),
+        event.getMarketId(),
+        event.getSelectionId(),
+        event.getPreviousOdds(),
+        event.getNewOdds(),
+        event.getChangedAt());
+  }
+}


## `feat(websocket): publish terminal bet updates`

diff --git a/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
new file mode 100644
index 0000000..2cad337
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ws/BetStatusStreamListener.java
@@ -0,0 +1,43 @@
+package com.sportsbook.gateway.ws;
+
+import com.sportsbook.gateway.events.GatewayEventContract;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+/** Publishes validated terminal bet events to their owning user. */
+@Component
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+public class BetStatusStreamListener {
+
+  private final GatewayPushPublisher publisher;
+
+  public BetStatusStreamListener(GatewayPushPublisher publisher) {
+    this.publisher = publisher;
+  }
+
+  @KafkaListener(
+      id = "gateway-settled-listener",
+      topics = "${gateway.topics.bet-settled}",
+      groupId = "gateway-bets",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onBetSettled(ConsumerRecord<byte[], byte[]> record) {
+    BetSettled event = GatewayEventContract.betSettled(record);
+    publisher.publishBet(event.getUserId(), BetStatusUpdate.settled(event));
+  }
+
+  @KafkaListener(
+      id = "gateway-voided-listener",
+      topics = "${gateway.topics.bet-voided}",
+      groupId = "gateway-bets",
+      containerFactory = "kafkaListenerContainerFactory",
+      autoStartup = "${spring.kafka.listener.auto-startup:true}")
+  public void onBetVoided(ConsumerRecord<byte[], byte[]> record) {
+    BetVoided event = GatewayEventContract.betVoided(record);
+    publisher.publishBet(event.getUserId(), BetStatusUpdate.voided(event));
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
index b43e087..f2794c7 100644
--- a/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
+++ b/src/main/java/com/sportsbook/gateway/ws/GatewayPushPublisher.java
@@ -19,4 +19,8 @@ public class GatewayPushPublisher {
   public void publishOdds(OddsChanged event) {
     messaging.convertAndSend("/topic/odds/" + event.getEventId(), OddsUpdate.from(event));
   }
+
+  public void publishBet(String userId, BetStatusUpdate update) {
+    messaging.convertAndSendToUser(userId, "/queue/bets", update);
+  }
 }


