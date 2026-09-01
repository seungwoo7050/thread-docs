## `test(ratelimit): verify shared token consumption`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
new file mode 100644
index 0000000..f2a30db
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
@@ -0,0 +1,75 @@
+package com.sportsbook.gateway.ratelimit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.awaitility.Awaitility.await;
+
+import io.lettuce.core.RedisClient;
+import java.time.Duration;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
+import org.testcontainers.containers.GenericContainer;
+import org.testcontainers.junit.jupiter.Container;
+import org.testcontainers.junit.jupiter.Testcontainers;
+import org.testcontainers.utility.DockerImageName;
+
+@Testcontainers(disabledWithoutDocker = true)
+class DistributedRateLimiterTest {
+
+  @Container
+  static final GenericContainer<?> REDIS =
+      new GenericContainer<>(DockerImageName.parse("redis:7-alpine")).withExposedPorts(6379);
+
+  private final RateLimitProperties.Limit limit =
+      new RateLimitProperties.Limit(2, Duration.ofSeconds(2));
+  private RedisClient firstClient;
+  private RedisClient secondClient;
+  private RateLimiterService first;
+  private RateLimiterService second;
+
+  @BeforeEach
+  void createLimiters() {
+    RateLimitProperties policies = new RateLimitProperties(true, limit, limit, List.of());
+    firstClient = client();
+    secondClient = client();
+    first = new RateLimiterService(firstClient, policies);
+    second = new RateLimiterService(secondClient, policies);
+  }
+
+  @AfterEach
+  void closeLimiters() {
+    first.closeConnection();
+    second.closeConnection();
+    firstClient.shutdown();
+    secondClient.shutdown();
+  }
+
+  @Test
+  void sharesCapacityAcrossGatewayInstancesAndRefills() {
+    String key = "gateway:ratelimit:ip:" + UUID.randomUUID();
+
+    RateLimiterService.Result firstUse = first.tryConsume(key, limit);
+    RateLimiterService.Result secondUse = second.tryConsume(key, limit);
+    RateLimiterService.Result exhausted = first.tryConsume(key, limit);
+
+    assertThat(firstUse.allowed()).isTrue();
+    assertThat(secondUse.allowed()).isTrue();
+    assertThat(exhausted.allowed()).isFalse();
+    assertThat(exhausted.retryAfterSeconds()).isPositive();
+
+    await()
+        .atMost(Duration.ofSeconds(2))
+        .pollInterval(Duration.ofMillis(100))
+        .untilAsserted(() -> assertThat(second.tryConsume(key, limit).allowed()).isTrue());
+  }
+
+  private static RedisClient client() {
+    RedisProperties properties = new RedisProperties();
+    properties.setHost(REDIS.getHost());
+    properties.setPort(REDIS.getMappedPort(6379));
+    return new RateLimitRedisConfiguration().rateLimitRedisClient(properties);
+  }
+}


## `test(ratelimit): verify bounded outage recovery`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
index f2a30db..795247f 100644
--- a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
@@ -1,12 +1,28 @@
 package com.sportsbook.gateway.ratelimit;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.awaitility.Awaitility.await;
+import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.times;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
 
 import io.lettuce.core.RedisClient;
+import io.lettuce.core.RedisConnectionException;
+import io.lettuce.core.codec.ByteArrayCodec;
 import java.time.Duration;
 import java.util.List;
 import java.util.UUID;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.stream.IntStream;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
@@ -66,6 +82,82 @@ class DistributedRateLimiterTest {
         .untilAsserted(() -> assertThat(second.tryConsume(key, limit).allowed()).isTrue());
   }
 
+  @Test
+  void warmedConnectionFailsOpenWithinTheBoundAndRecovers() {
+    String key = "gateway:ratelimit:ip:" + UUID.randomUUID();
+    assertThat(first.tryConsume(key, limit).failOpen()).isFalse();
+
+    REDIS.getDockerClient().pauseContainerCmd(REDIS.getContainerId()).exec();
+    try {
+      RateLimiterService.Result degraded =
+          assertTimeoutPreemptively(Duration.ofSeconds(2), () -> first.tryConsume(key, limit));
+      assertThat(degraded.allowed()).isTrue();
+      assertThat(degraded.failOpen()).isTrue();
+      assertThat(degraded.remainingTokens()).isEqualTo(-1);
+    } finally {
+      REDIS.getDockerClient().unpauseContainerCmd(REDIS.getContainerId()).exec();
+    }
+
+    await()
+        .atMost(Duration.ofSeconds(5))
+        .untilAsserted(() -> assertThat(first.tryConsume(key, limit).failOpen()).isFalse());
+  }
+
+  @Test
+  void coldConnectionAttemptIsSingleFlightAndRecovers() throws Exception {
+    RedisClient delegate = client();
+    RedisClient blocked = mock(RedisClient.class);
+    CountDownLatch entered = new CountDownLatch(1);
+    CountDownLatch release = new CountDownLatch(1);
+    AtomicInteger attempts = new AtomicInteger();
+    when(blocked.connect(any(ByteArrayCodec.class)))
+        .thenAnswer(
+            invocation -> {
+              if (attempts.incrementAndGet() == 1) {
+                entered.countDown();
+                release.await();
+                throw new RedisConnectionException("fixture outage");
+              }
+              ByteArrayCodec codec = invocation.getArgument(0);
+              return delegate.connect(codec);
+            });
+    RateLimitProperties policies = new RateLimitProperties(true, limit, limit, List.of());
+    RateLimiterService limiter = new RateLimiterService(blocked, policies);
+    ExecutorService workers = Executors.newFixedThreadPool(5);
+    String key = "gateway:ratelimit:ip:" + UUID.randomUUID();
+    try {
+      Future<RateLimiterService.Result> leader =
+          workers.submit(() -> limiter.tryConsume(key, limit));
+      assertThat(entered.await(1, TimeUnit.SECONDS)).isTrue();
+      List<Future<RateLimiterService.Result>> followers =
+          IntStream.range(0, 4)
+              .mapToObj(ignored -> workers.submit(() -> limiter.tryConsume(key, limit)))
+              .toList();
+      for (Future<RateLimiterService.Result> follower : followers) {
+        assertThat(follower.get(1, TimeUnit.SECONDS))
+            .isEqualTo(new RateLimiterService.Result(true, -1, 0, true));
+      }
+      verify(blocked, times(1)).connect(any(ByteArrayCodec.class));
+      release.countDown();
+      assertThat(leader.get(1, TimeUnit.SECONDS))
+          .isEqualTo(new RateLimiterService.Result(true, -1, 0, true));
+      assertThat(limiter.tryConsume(key + ":recovered", limit).failOpen()).isFalse();
+      verify(blocked, times(2)).connect(any(ByteArrayCodec.class));
+    } finally {
+      release.countDown();
+      workers.shutdownNow();
+      limiter.closeConnection();
+      delegate.shutdown();
+    }
+  }
+
+  @Test
+  void invalidLimitIsNeverConvertedToFailOpen() {
+    RateLimitProperties.Limit invalid = new RateLimitProperties.Limit(0, Duration.ofSeconds(1));
+    assertThatThrownBy(() -> first.tryConsume("invalid", invalid))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
   private static RedisClient client() {
     RedisProperties properties = new RedisProperties();
     properties.setHost(REDIS.getHost());


## `feat(ratelimit): enforce request rate limits`

diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitFilter.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitFilter.java
new file mode 100644
index 0000000..667517a
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitFilter.java
@@ -0,0 +1,65 @@
+package com.sportsbook.gateway.ratelimit;
+
+import com.sportsbook.gateway.error.GatewayErrorCode;
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import org.springframework.http.HttpHeaders;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+public final class RateLimitFilter extends OncePerRequestFilter {
+
+  static final String REMAINING_HEADER = "X-RateLimit-Remaining";
+
+  private final RateLimitProperties properties;
+  private final RateLimitKeyResolver keys;
+  private final RateLimiterService limiter;
+  private final GatewayProblemWriter problems;
+
+  public RateLimitFilter(
+      RateLimitProperties properties,
+      RateLimitKeyResolver keys,
+      RateLimiterService limiter,
+      GatewayProblemWriter problems) {
+    this.properties = properties;
+    this.keys = keys;
+    this.limiter = limiter;
+    this.problems = problems;
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    String path = request.getRequestURI();
+    return !properties.enabled()
+        || path.equals("/error")
+        || path.equals("/actuator")
+        || path.startsWith("/actuator/");
+  }
+
+  @Override
+  protected boolean shouldNotFilterErrorDispatch() {
+    return true;
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    RateLimitKeyResolver.ResolvedKey key = keys.resolve(request);
+    RateLimiterService.Result result = limiter.tryConsume(key.value(), key.limit());
+    if (result.allowed()) {
+      if (!result.failOpen()) {
+        response.setHeader(REMAINING_HEADER, Long.toString(result.remainingTokens()));
+      }
+      chain.doFilter(request, response);
+      return;
+    }
+
+    response.setHeader(HttpHeaders.RETRY_AFTER, Long.toString(result.retryAfterSeconds()));
+    response.setHeader(REMAINING_HEADER, "0");
+    problems.write(request, response, GatewayErrorCode.GATEWAY_RATE_LIMITED);
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitHttpConfiguration.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitHttpConfiguration.java
new file mode 100644
index 0000000..148fa29
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimitHttpConfiguration.java
@@ -0,0 +1,23 @@
+package com.sportsbook.gateway.ratelimit;
+
+import com.sportsbook.gateway.error.GatewayProblemWriter;
+import org.springframework.boot.autoconfigure.security.SecurityProperties;
+import org.springframework.boot.web.servlet.FilterRegistrationBean;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+@Configuration(proxyBeanMethods = false)
+public class RateLimitHttpConfiguration {
+
+  @Bean
+  FilterRegistrationBean<RateLimitFilter> rateLimitFilter(
+      RateLimitProperties properties,
+      RateLimitKeyResolver keys,
+      RateLimiterService limiter,
+      GatewayProblemWriter problems) {
+    FilterRegistrationBean<RateLimitFilter> registration = new FilterRegistrationBean<>();
+    registration.setFilter(new RateLimitFilter(properties, keys, limiter, problems));
+    registration.setOrder(SecurityProperties.DEFAULT_FILTER_ORDER + 10);
+    return registration;
+  }
+}


## `test(ratelimit): verify HTTP limit responses`

diff --git a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
index 795247f..27b8d70 100644
--- a/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
+++ b/src/test/java/com/sportsbook/gateway/ratelimit/DistributedRateLimiterTest.java
@@ -10,9 +10,13 @@ import static org.mockito.Mockito.times;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.gateway.error.GatewayProblemWriter;
 import io.lettuce.core.RedisClient;
 import io.lettuce.core.RedisConnectionException;
 import io.lettuce.core.codec.ByteArrayCodec;
+import io.micrometer.tracing.Tracer;
+import jakarta.servlet.DispatcherType;
 import java.time.Duration;
 import java.util.List;
 import java.util.UUID;
@@ -21,12 +25,19 @@ import java.util.concurrent.ExecutorService;
 import java.util.concurrent.Executors;
 import java.util.concurrent.Future;
 import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicBoolean;
 import java.util.concurrent.atomic.AtomicInteger;
 import java.util.stream.IntStream;
 import org.junit.jupiter.api.AfterEach;
 import org.junit.jupiter.api.BeforeEach;
 import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.support.DefaultListableBeanFactory;
 import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
+import org.springframework.http.HttpHeaders;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
+import org.springframework.security.core.context.SecurityContextHolder;
 import org.testcontainers.containers.GenericContainer;
 import org.testcontainers.junit.jupiter.Container;
 import org.testcontainers.junit.jupiter.Testcontainers;
@@ -57,6 +68,7 @@ class DistributedRateLimiterTest {
 
   @AfterEach
   void closeLimiters() {
+    SecurityContextHolder.clearContext();
     first.closeConnection();
     second.closeConnection();
     firstClient.shutdown();
@@ -158,6 +170,62 @@ class DistributedRateLimiterTest {
         .isInstanceOf(IllegalArgumentException.class);
   }
 
+  @Test
+  void returnsProblemResponseAfterSeparateIpAndUserBucketsAreExhausted() throws Exception {
+    RateLimitFilter filter = filter(true);
+    String peer = "198.51.100.44";
+    assertThat(
+            exchange(filter, request(peer)).response().getHeader(RateLimitFilter.REMAINING_HEADER))
+        .isEqualTo("0");
+
+    Exchange denied = exchange(filter, request(peer));
+    assertThat(denied.response().getStatus()).isEqualTo(429);
+    assertThat(denied.response().getHeader(HttpHeaders.RETRY_AFTER)).isNotBlank();
+    assertThat(denied.response().getContentAsString()).contains("GATEWAY_RATE_LIMITED");
+
+    String user = UUID.randomUUID().toString();
+    SecurityContextHolder.getContext()
+        .setAuthentication(new UsernamePasswordAuthenticationToken(user, "unused", List.of()));
+    assertThat(exchange(filter, request(peer)).response().getStatus()).isEqualTo(200);
+  }
+
+  @Test
+  void disabledAndErrorDispatchesNeverConsumeTokens() throws Exception {
+    Exchange disabled = exchange(filter(false), request("198.51.100.45"));
+    assertThat(disabled.passed()).isTrue();
+    assertThat(disabled.response().getHeader(RateLimitFilter.REMAINING_HEADER)).isNull();
+
+    MockHttpServletRequest error = request("198.51.100.46");
+    error.setDispatcherType(DispatcherType.ERROR);
+    assertThat(exchange(filter(true), error).passed()).isTrue();
+  }
+
+  private RateLimitFilter filter(boolean enabled) {
+    RateLimitProperties.Limit single = new RateLimitProperties.Limit(1, Duration.ofMinutes(1));
+    RateLimitProperties policies = new RateLimitProperties(enabled, single, single, List.of());
+    GatewayProblemWriter writer =
+        new GatewayProblemWriter(
+            new ObjectMapper(), new DefaultListableBeanFactory().getBeanProvider(Tracer.class));
+    return new RateLimitFilter(policies, new RateLimitKeyResolver(policies), first, writer);
+  }
+
+  private static Exchange exchange(RateLimitFilter filter, MockHttpServletRequest request)
+      throws Exception {
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    AtomicBoolean passed = new AtomicBoolean();
+    filter.doFilter(request, response, (filtered, result) -> passed.set(true));
+    return new Exchange(response, passed.get());
+  }
+
+  private static MockHttpServletRequest request(String peer) {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    request.setRemoteAddr(peer);
+    request.setRequestURI("/api/v1/events");
+    return request;
+  }
+
+  private record Exchange(MockHttpServletResponse response, boolean passed) {}
+
   private static RedisClient client() {
     RedisProperties properties = new RedisProperties();
     properties.setHost(REDIS.getHost());


## `feat(observability): expose service health and metrics`

diff --git a/pom.xml b/pom.xml
index 2e98680..df2785b 100644
--- a/pom.xml
+++ b/pom.xml
@@ -172,6 +172,18 @@
             <plugin>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-maven-plugin</artifactId>
+                <executions>
+                    <execution>
+                        <goals>
+                            <goal>build-info</goal>
+                        </goals>
+                        <configuration>
+                            <excludeInfoProperties>
+                                <excludeInfoProperty>time</excludeInfoProperty>
+                            </excludeInfoProperties>
+                        </configuration>
+                    </execution>
+                </executions>
             </plugin>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
diff --git a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
index 5085723..14aa47e 100644
--- a/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
+++ b/src/main/java/com/sportsbook/gateway/ratelimit/RateLimiterService.java
@@ -10,12 +10,16 @@ import io.lettuce.core.RedisClient;
 import io.lettuce.core.RedisException;
 import io.lettuce.core.api.StatefulRedisConnection;
 import io.lettuce.core.codec.ByteArrayCodec;
+import io.micrometer.core.instrument.Counter;
+import io.micrometer.core.instrument.MeterRegistry;
+import io.micrometer.core.instrument.Metrics;
 import jakarta.annotation.PreDestroy;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
 import java.util.concurrent.locks.ReentrantLock;
 import org.slf4j.Logger;
 import org.slf4j.LoggerFactory;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.stereotype.Service;
 
 @Service
@@ -24,18 +28,26 @@ public final class RateLimiterService {
   private static final Logger log = LoggerFactory.getLogger(RateLimiterService.class);
   private final RedisClient redisClient;
   private final Duration expirationGrace;
+  private final Counter failOpenCounter;
   private final ReentrantLock managerLock = new ReentrantLock();
   private volatile StatefulRedisConnection<byte[], byte[]> connection;
   private volatile ProxyManager<byte[]> manager;
 
-  public RateLimiterService(RedisClient redisClient, RateLimitProperties properties) {
+  @Autowired
+  public RateLimiterService(
+      RedisClient redisClient, RateLimitProperties properties, MeterRegistry meterRegistry) {
     this.redisClient = redisClient;
+    this.failOpenCounter = meterRegistry.counter("gateway.ratelimit.fail.open");
     this.expirationGrace =
         properties.user().refillPeriod().compareTo(properties.ip().refillPeriod()) >= 0
             ? properties.user().refillPeriod()
             : properties.ip().refillPeriod();
   }
 
+  public RateLimiterService(RedisClient redisClient, RateLimitProperties properties) {
+    this(redisClient, properties, Metrics.globalRegistry);
+  }
+
   public Result tryConsume(String key, RateLimitProperties.Limit limit) {
     BucketConfiguration configuration =
         BucketConfiguration.builder()
@@ -53,6 +65,7 @@ public final class RateLimiterService {
           ? new Result(true, probe.getRemainingTokens(), 0, false)
           : new Result(false, 0, waitSeconds(probe.getNanosToWaitForRefill()), false);
     } catch (RedisException failure) {
+      failOpenCounter.increment();
       log.warn("Rate limiter fail-open: {}", failure.getClass().getSimpleName());
       return new Result(true, -1, 0, true);
     }
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 730568d..04f69e7 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -41,6 +41,27 @@ server:
   port: ${GATEWAY_HTTP_PORT:8080}
   shutdown: graceful
 
+management:
+  endpoints:
+    web:
+      exposure:
+        include: health,info,prometheus
+  endpoint:
+    health:
+      probes:
+        enabled: true
+      show-components: never
+      show-details: never
+      group:
+        liveness:
+          include: livenessState
+        readiness:
+          include: readinessState
+  metrics:
+    use-global-registry: true
+    tags:
+      service: ${spring.application.name}
+
 gateway:
   downstream:
     betting-uri: ${GATEWAY_BETTING_URI:http://localhost:8082}
