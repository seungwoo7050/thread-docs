# 신뢰 프록시 판별과 Fail-Open 복구를 포함한 분산 요청 제한

## `feat(ratelimit): define validated bucket policies`

diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitProperties.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitProperties.java
new file mode 100644
index 0000000..1ad3bb6
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitProperties.java
@@ -0,0 +1,33 @@
+package com.sportsbook.gateway.ratelimit;
+
+import jakarta.validation.Valid;
+import jakarta.validation.constraints.AssertTrue;
+import jakarta.validation.constraints.NotNull;
+import jakarta.validation.constraints.Positive;
+import java.time.Duration;
+import java.util.List;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.security.web.util.matcher.IpAddressMatcher;
+import org.springframework.validation.annotation.Validated;
+
+@Validated
+@ConfigurationProperties("gateway.ratelimit")
+public record RateLimitProperties(
+    boolean enabled,
+    @Valid @NotNull Limit user,
+    @Valid @NotNull Limit ip,
+    List<String> trustedProxyCidrs) {
+
+  public RateLimitProperties {
+    trustedProxyCidrs = trustedProxyCidrs == null ? List.of() : List.copyOf(trustedProxyCidrs);
+    trustedProxyCidrs.forEach(IpAddressMatcher::new);
+  }
+
+  public record Limit(@Positive long capacity, @NotNull Duration refillPeriod) {
+
+    @AssertTrue(message = "refill period must be positive")
+    public boolean isRefillPeriodPositive() {
+      return refillPeriod == null || !refillPeriod.isZero() && !refillPeriod.isNegative();
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 5da8848..4b2c54f 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -12,3 +12,12 @@ gateway:
   security:
     jwt:
       public-key: ${GATEWAY_JWT_PUBLIC_KEY:}
+  ratelimit:
+    enabled: ${GATEWAY_RATELIMIT_ENABLED:true}
+    user:
+      capacity: ${GATEWAY_RATELIMIT_USER_CAPACITY:120}
+      refill-period: ${GATEWAY_RATELIMIT_USER_PERIOD:1m}
+    ip:
+      capacity: ${GATEWAY_RATELIMIT_IP_CAPACITY:60}
+      refill-period: ${GATEWAY_RATELIMIT_IP_PERIOD:1m}
+    trusted-proxy-cidrs: ${GATEWAY_TRUSTED_PROXY_CIDRS:}


## `test(ratelimit): reject invalid bucket policies`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitPropertiesTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitPropertiesTest.java
new file mode 100644
index 0000000..ec86c62
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitPropertiesTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.gateway.ratelimit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.AutoConfigurations;
+import org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration;
+import org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+
+class RateLimitPropertiesTest {
+
+  @Test
+  void rejectsZeroAndNegativeCapacity() {
+    assertRejected(valid("gateway.ratelimit.user.capacity=0"));
+    assertRejected(valid("gateway.ratelimit.ip.capacity=-1"));
+  }
+
+  @Test
+  void rejectsMissingAndNonPositiveRefillPeriods() {
+    assertRejected(
+        runner(
+            "gateway.ratelimit.user.capacity=120",
+            "gateway.ratelimit.ip.capacity=60",
+            "gateway.ratelimit.ip.refill-period=1m"));
+    assertRejected(valid("gateway.ratelimit.user.refill-period=0s"));
+    assertRejected(valid("gateway.ratelimit.ip.refill-period=-1s"));
+  }
+
+  @Test
+  void rejectsInvalidTrustedProxyCidr() {
+    assertRejected(valid("gateway.ratelimit.trusted-proxy-cidrs=not-a-cidr"));
+  }
+
+  private static ApplicationContextRunner valid(String... overrides) {
+    ApplicationContextRunner runner =
+        runner(
+            "gateway.ratelimit.enabled=true",
+            "gateway.ratelimit.user.capacity=120",
+            "gateway.ratelimit.user.refill-period=1m",
+            "gateway.ratelimit.ip.capacity=60",
+            "gateway.ratelimit.ip.refill-period=1m");
+    return runner.withPropertyValues(overrides);
+  }
+
+  private static ApplicationContextRunner runner(String... properties) {
+    return new ApplicationContextRunner()
+        .withConfiguration(
+            AutoConfigurations.of(
+                ConfigurationPropertiesAutoConfiguration.class, ValidationAutoConfiguration.class))
+        .withUserConfiguration(PropertyConfiguration.class)
+        .withPropertyValues(properties);
+  }
+
+  private static void assertRejected(ApplicationContextRunner runner) {
+    runner.run(context -> assertThat(context).hasFailed());
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(RateLimitProperties.class)
+  static class PropertyConfiguration {}
+}


## `feat(ratelimit): resolve user and trusted peer buckets`

diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolver.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolver.java
new file mode 100644
index 0000000..792db6b
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolver.java
@@ -0,0 +1,83 @@
+package com.sportsbook.gateway.ratelimit;
+
+import io.netty.util.NetUtil;
+import jakarta.servlet.http.HttpServletRequest;
+import java.net.InetAddress;
+import java.net.UnknownHostException;
+import java.util.ArrayList;
+import java.util.List;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.security.authentication.AnonymousAuthenticationToken;
+import org.springframework.security.core.Authentication;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.web.util.matcher.IpAddressMatcher;
+import org.springframework.stereotype.Component;
+import org.springframework.util.StringUtils;
+
+@Component
+@EnableConfigurationProperties(RateLimitProperties.class)
+public final class RateLimitKeyResolver {
+
+  private static final String USER_PREFIX = "gateway:ratelimit:user:";
+  private static final String IP_PREFIX = "gateway:ratelimit:ip:";
+
+  private final RateLimitProperties properties;
+  private final List<IpAddressMatcher> trustedProxies;
+
+  public RateLimitKeyResolver(RateLimitProperties properties) {
+    this.properties = properties;
+    this.trustedProxies =
+        properties.trustedProxyCidrs().stream().map(IpAddressMatcher::new).toList();
+  }
+
+  public ResolvedKey resolve(HttpServletRequest request) {
+    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
+    if (authentication != null
+        && authentication.isAuthenticated()
+        && !(authentication instanceof AnonymousAuthenticationToken)
+        && StringUtils.hasText(authentication.getName())) {
+      return new ResolvedKey(USER_PREFIX + authentication.getName(), properties.user());
+    }
+    return new ResolvedKey(IP_PREFIX + clientAddress(request), properties.ip());
+  }
+
+  private String clientAddress(HttpServletRequest request) {
+    String peer = request.getRemoteAddr();
+    if (trustedProxies.stream().noneMatch(proxy -> proxy.matches(peer))) {
+      return peer;
+    }
+    String forwarded = request.getHeader("X-Forwarded-For");
+    if (!StringUtils.hasText(forwarded)) {
+      return peer;
+    }
+    List<String> hops = new ArrayList<>();
+    for (String value : forwarded.split(",", -1)) {
+      String hop = normalize(value.trim());
+      if (hop == null) {
+        return peer;
+      }
+      hops.add(hop);
+    }
+    for (int index = hops.size() - 1; index >= 0; index--) {
+      String hop = hops.get(index);
+      if (trustedProxies.stream().noneMatch(proxy -> proxy.matches(hop))) {
+        return hop;
+      }
+    }
+    return peer;
+  }
+
+  private static String normalize(String address) {
+    byte[] bytes = NetUtil.createByteArrayFromIpAddressString(address);
+    if (bytes == null) {
+      return null;
+    }
+    try {
+      return InetAddress.getByAddress(bytes).getHostAddress();
+    } catch (UnknownHostException impossible) {
+      throw new IllegalStateException("Validated IP address could not be parsed", impossible);
+    }
+  }
+
+  public record ResolvedKey(String value, RateLimitProperties.Limit limit) {}
+}


## `test(ratelimit): verify rate limit key isolation`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolverTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolverTest.java
new file mode 100644
index 0000000..863347a
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitKeyResolverTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.gateway.ratelimit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Duration;
+import java.util.List;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.context.SecurityContextHolder;
+
+class RateLimitKeyResolverTest {
+
+  private static final RateLimitProperties PROPERTIES =
+      new RateLimitProperties(
+          true,
+          new RateLimitProperties.Limit(120, Duration.ofMinutes(1)),
+          new RateLimitProperties.Limit(60, Duration.ofMinutes(1)),
+          List.of("10.0.0.0/8"));
+
+  private final RateLimitKeyResolver resolver = new RateLimitKeyResolver(PROPERTIES);
+
+  @AfterEach
+  void clearAuthentication() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void authenticatedUsersReceiveCanonicalNamespacedBuckets() {
+    String subject = "00000000-0000-0000-0000-000000000001";
+    SecurityContextHolder.getContext()
+        .setAuthentication(new UsernamePasswordAuthenticationToken(subject, "unused", List.of()));
+
+    RateLimitKeyResolver.ResolvedKey key = resolver.resolve(request("192.0.2.4", "198.51.100.1"));
+
+    assertThat(key.value()).isEqualTo("gateway:ratelimit:user:" + subject);
+    assertThat(key.limit().capacity()).isEqualTo(120);
+  }
+
+  @Test
+  void untrustedPeersCannotSpoofForwardedAddresses() {
+    RateLimitKeyResolver.ResolvedKey key = resolver.resolve(request("192.0.2.4", "198.51.100.1"));
+
+    assertThat(key.value()).isEqualTo("gateway:ratelimit:ip:192.0.2.4");
+    assertThat(key.limit().capacity()).isEqualTo(60);
+  }
+
+  @Test
+  void trustedPeersWalkTheForwardedChainFromTheRight() {
+    MockHttpServletRequest trusted = request("10.2.3.4", "203.0.113.9, 198.51.100.7, 10.1.2.3");
+    assertThat(resolver.resolve(trusted).value()).isEqualTo("gateway:ratelimit:ip:198.51.100.7");
+
+    MockHttpServletRequest malformed = request("10.2.3.4", "198.51.100.7, invalid, 10.1.2.3");
+    assertThat(resolver.resolve(malformed).value()).isEqualTo("gateway:ratelimit:ip:10.2.3.4");
+  }
+
+  private static MockHttpServletRequest request(String peer, String forwardedFor) {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.setRemoteAddr(peer);
+    request.addHeader("X-Forwarded-For", forwardedFor);
+    return request;
+  }
+}


## `feat(ratelimit): configure bounded Redis access`

diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfiguration.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfiguration.java
new file mode 100644
index 0000000..4b768c4
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfiguration.java
@@ -0,0 +1,47 @@
+package com.sportsbook.gateway.ratelimit;
+
+import io.lettuce.core.ClientOptions;
+import io.lettuce.core.RedisClient;
+import io.lettuce.core.RedisURI;
+import io.lettuce.core.SocketOptions;
+import io.lettuce.core.TimeoutOptions;
+import java.time.Duration;
+import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.util.StringUtils;
+
+@Configuration(proxyBeanMethods = false)
+public class RateLimitRedisConfiguration {
+
+  static final Duration CONNECT_TIMEOUT = Duration.ofMillis(300);
+  static final Duration COMMAND_TIMEOUT = Duration.ofMillis(500);
+
+  @Bean(destroyMethod = "shutdown")
+  RedisClient rateLimitRedisClient(RedisProperties properties) {
+    RedisURI.Builder uri =
+        RedisURI.builder()
+            .withHost(properties.getHost())
+            .withPort(properties.getPort())
+            .withDatabase(properties.getDatabase())
+            .withSsl(properties.getSsl().isEnabled())
+            .withTimeout(COMMAND_TIMEOUT);
+    if (StringUtils.hasText(properties.getPassword())) {
+      if (StringUtils.hasText(properties.getUsername())) {
+        uri.withAuthentication(properties.getUsername(), properties.getPassword().toCharArray());
+      } else {
+        uri.withPassword(properties.getPassword().toCharArray());
+      }
+    }
+
+    RedisClient client = RedisClient.create(uri.build());
+    client.setOptions(
+        ClientOptions.builder()
+            .autoReconnect(true)
+            .disconnectedBehavior(ClientOptions.DisconnectedBehavior.REJECT_COMMANDS)
+            .socketOptions(SocketOptions.builder().connectTimeout(CONNECT_TIMEOUT).build())
+            .timeoutOptions(TimeoutOptions.enabled(COMMAND_TIMEOUT))
+            .build());
+    return client;
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 4b2c54f..b2f045e 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -1,6 +1,15 @@
 spring:
   application:
     name: gateway
+  data:
+    redis:
+      host: ${GATEWAY_REDIS_HOST:localhost}
+      port: ${GATEWAY_REDIS_PORT:6379}
+      username: ${GATEWAY_REDIS_USERNAME:}
+      password: ${GATEWAY_REDIS_PASSWORD:}
+      database: ${GATEWAY_REDIS_DATABASE:0}
+      ssl:
+        enabled: ${GATEWAY_REDIS_SSL:false}
   lifecycle:
     timeout-per-shutdown-phase: 20s
 


## `test(ratelimit): verify Redis connection bounds`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfigurationTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfigurationTest.java
new file mode 100644
index 0000000..9c49ac0
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/RateLimitRedisConfigurationTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.gateway.ratelimit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.junit.jupiter.api.Assertions.assertThrows;
+import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;
+
+import io.lettuce.core.ClientOptions;
+import io.lettuce.core.RedisClient;
+import io.lettuce.core.RedisException;
+import io.lettuce.core.TimeoutOptions;
+import io.lettuce.core.codec.ByteArrayCodec;
+import java.time.Duration;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
+
+class RateLimitRedisConfigurationTest {
+
+  private RedisClient client;
+
+  @AfterEach
+  void shutdownClient() {
+    if (client != null) {
+      client.shutdown();
+    }
+  }
+
+  @Test
+  void configuresReconnectAndFixedConnectionBounds() {
+    client = create("localhost", 6379);
+    ClientOptions options = client.getOptions();
+    TimeoutOptions timeouts = options.getTimeoutOptions();
+
+    assertThat(options.isAutoReconnect()).isTrue();
+    assertThat(options.getDisconnectedBehavior())
+        .isEqualTo(ClientOptions.DisconnectedBehavior.REJECT_COMMANDS);
+    assertThat(options.getSocketOptions().getConnectTimeout()).isEqualTo(Duration.ofMillis(300));
+    assertThat(client.getDefaultTimeout()).isEqualTo(Duration.ofMillis(500));
+    assertThat(timeouts.isTimeoutCommands()).isTrue();
+    assertThat(timeouts.getSource().getTimeUnit().toMillis(timeouts.getSource().getTimeout(null)))
+        .isEqualTo(500);
+  }
+
+  @Test
+  void unreachableRedisFailsWithinTheConnectionBound() {
+    client = create("192.0.2.1", 6379);
+
+    assertTimeoutPreemptively(
+        Duration.ofSeconds(2),
+        () -> assertThrows(RedisException.class, () -> client.connect(ByteArrayCodec.INSTANCE)));
+  }
+
+  private static RedisClient create(String host, int port) {
+    RedisProperties properties = new RedisProperties();
+    properties.setHost(host);
+    properties.setPort(port);
+    return new RateLimitRedisConfiguration().rateLimitRedisClient(properties);
+  }
+}


## `feat(ratelimit): consume distributed token buckets`

diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
new file mode 100644
index 0000000..5085723
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
@@ -0,0 +1,100 @@
+package com.sportsbook.gateway.ratelimit;
+
+import io.github.bucket4j.BucketConfiguration;
+import io.github.bucket4j.ConsumptionProbe;
+import io.github.bucket4j.distributed.BucketProxy;
+import io.github.bucket4j.distributed.ExpirationAfterWriteStrategy;
+import io.github.bucket4j.distributed.proxy.ProxyManager;
+import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager;
+import io.lettuce.core.RedisClient;
+import io.lettuce.core.RedisException;
+import io.lettuce.core.api.StatefulRedisConnection;
+import io.lettuce.core.codec.ByteArrayCodec;
+import jakarta.annotation.PreDestroy;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.concurrent.locks.ReentrantLock;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.stereotype.Service;
+
+@Service
+public final class RateLimiterService {
+
+  private static final Logger log = LoggerFactory.getLogger(RateLimiterService.class);
+  private final RedisClient redisClient;
+  private final Duration expirationGrace;
+  private final ReentrantLock managerLock = new ReentrantLock();
+  private volatile StatefulRedisConnection<byte[], byte[]> connection;
+  private volatile ProxyManager<byte[]> manager;
+
+  public RateLimiterService(RedisClient redisClient, RateLimitProperties properties) {
+    this.redisClient = redisClient;
+    this.expirationGrace =
+        properties.user().refillPeriod().compareTo(properties.ip().refillPeriod()) >= 0
+            ? properties.user().refillPeriod()
+            : properties.ip().refillPeriod();
+  }
+
+  public Result tryConsume(String key, RateLimitProperties.Limit limit) {
+    BucketConfiguration configuration =
+        BucketConfiguration.builder()
+            .addLimit(
+                bandwidth ->
+                    bandwidth
+                        .capacity(limit.capacity())
+                        .refillGreedy(limit.capacity(), limit.refillPeriod()))
+            .build();
+    try {
+      BucketProxy bucket =
+          manager().builder().build(key.getBytes(StandardCharsets.UTF_8), configuration);
+      ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
+      return probe.isConsumed()
+          ? new Result(true, probe.getRemainingTokens(), 0, false)
+          : new Result(false, 0, waitSeconds(probe.getNanosToWaitForRefill()), false);
+    } catch (RedisException failure) {
+      log.warn("Rate limiter fail-open: {}", failure.getClass().getSimpleName());
+      return new Result(true, -1, 0, true);
+    }
+  }
+
+  private static long waitSeconds(long nanos) {
+    return Math.max(1, 1 + (nanos - 1) / 1_000_000_000L);
+  }
+
+  private ProxyManager<byte[]> manager() {
+    if (manager != null) {
+      return manager;
+    }
+    if (!managerLock.tryLock()) {
+      throw new RedisException("Rate limiter initialization is in progress");
+    }
+    try {
+      if (manager == null) {
+        StatefulRedisConnection<byte[], byte[]> opened =
+            redisClient.connect(ByteArrayCodec.INSTANCE);
+        ProxyManager<byte[]> created =
+            LettuceBasedProxyManager.builderFor(opened)
+                .withExpirationStrategy(
+                    ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(
+                        expirationGrace))
+                .build();
+        connection = opened;
+        manager = created;
+      }
+      return manager;
+    } finally {
+      managerLock.unlock();
+    }
+  }
+
+  @PreDestroy
+  void closeConnection() {
+    if (connection != null) {
+      connection.close();
+    }
+  }
+
+  public record Result(
+      boolean allowed, long remainingTokens, long retryAfterSeconds, boolean failOpen) {}
+}


