# 외부 배당 API 어댑터의 호출 예산·정규화·변경 감지

## `feat(real): bind external provider settings`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
index 5c26ca9..92590b3 100644
--- a/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
+++ b/src/main/java/com/sportsbook/oddsfeed/config/ApplicationConfig.java
@@ -8,7 +8,7 @@ import org.springframework.scheduling.annotation.EnableScheduling;
 
 @Configuration
 @EnableScheduling
-@EnableConfigurationProperties(MockProperties.class)
+@EnableConfigurationProperties({MockProperties.class, RealProperties.class})
 public class ApplicationConfig {
 
   @Bean
diff --git a/src/main/java/com/sportsbook/oddsfeed/config/RealProperties.java b/src/main/java/com/sportsbook/oddsfeed/config/RealProperties.java
new file mode 100644
index 0000000..c115a78
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/RealProperties.java
@@ -0,0 +1,16 @@
+package com.sportsbook.oddsfeed.config;
+
+import java.util.List;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties(prefix = "oddsfeed.real")
+public record RealProperties(
+    String apiKey,
+    String baseUrl,
+    List<String> sportKeys,
+    RateLimit rateLimit,
+    int monthlyQuota,
+    int pollIntervalSeconds) {
+
+  public record RateLimit(int maxRequestsPerMinute) {}
+}


## `test(real): verify external provider settings`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/RealPropertiesTest.java b/src/test/java/com/sportsbook/oddsfeed/config/RealPropertiesTest.java
new file mode 100644
index 0000000..fe63abe
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/RealPropertiesTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.bind.Bindable;
+import org.springframework.boot.context.properties.bind.Binder;
+import org.springframework.boot.context.properties.source.MapConfigurationPropertySource;
+
+class RealPropertiesTest {
+
+  @Test
+  void bindsExternalProviderSettings() {
+    Map<String, String> values =
+        Map.of(
+            "oddsfeed.real.api-key", "secret",
+            "oddsfeed.real.base-url", "https://odds.example",
+            "oddsfeed.real.sport-keys[0]", "soccer_epl",
+            "oddsfeed.real.sport-keys[1]", "basketball_nba",
+            "oddsfeed.real.rate-limit.max-requests-per-minute", "5",
+            "oddsfeed.real.monthly-quota", "500",
+            "oddsfeed.real.poll-interval-seconds", "60");
+
+    RealProperties properties =
+        new Binder(new MapConfigurationPropertySource(values))
+            .bind("oddsfeed.real", Bindable.of(RealProperties.class))
+            .orElseThrow(IllegalStateException::new);
+
+    assertThat(properties.apiKey()).isEqualTo("secret");
+    assertThat(properties.baseUrl()).isEqualTo("https://odds.example");
+    assertThat(properties.sportKeys()).containsExactly("soccer_epl", "basketball_nba");
+    assertThat(properties.rateLimit().maxRequestsPerMinute()).isEqualTo(5);
+    assertThat(properties.monthlyQuota()).isEqualTo(500);
+    assertThat(properties.pollIntervalSeconds()).isEqualTo(60);
+  }
+}


## `feat(real): define external provider defaults`

diff --git a/src/main/resources/application-real.yml b/src/main/resources/application-real.yml
new file mode 100644
index 0000000..f46baa5
--- /dev/null
+++ b/src/main/resources/application-real.yml
@@ -0,0 +1,13 @@
+oddsfeed:
+  provider:
+    mode: real
+  real:
+    api-key: ${THE_ODDS_API_KEY}
+    base-url: https://api.the-odds-api.com/v4
+    sport-keys:
+      - soccer_epl
+      - basketball_nba
+    rate-limit:
+      max-requests-per-minute: 5
+    monthly-quota: 500
+    poll-interval-seconds: 60


## `feat(real): enforce request rate limits`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/RateLimiter.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/RateLimiter.java
new file mode 100644
index 0000000..52787df
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/RateLimiter.java
@@ -0,0 +1,38 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.ArrayDeque;
+import java.util.Deque;
+
+public final class RateLimiter {
+
+  private static final Duration WINDOW = Duration.ofMinutes(1);
+
+  private final int maxRequestsPerWindow;
+  private final Clock clock;
+  private final Deque<Instant> recent = new ArrayDeque<>();
+
+  public RateLimiter(int maxRequestsPerWindow, Clock clock) {
+    this.maxRequestsPerWindow = maxRequestsPerWindow;
+    this.clock = clock;
+  }
+
+  public synchronized boolean tryAcquire() {
+    Instant now = clock.instant();
+    Instant cutoff = now.minus(WINDOW);
+    while (!recent.isEmpty() && recent.peekFirst().isBefore(cutoff)) {
+      recent.pollFirst();
+    }
+    if (recent.size() >= maxRequestsPerWindow) {
+      return false;
+    }
+    recent.addLast(now);
+    return true;
+  }
+
+  synchronized int currentUsage() {
+    return recent.size();
+  }
+}


## `test(real): verify request rate limits`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/RateLimiterTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/RateLimiterTest.java
new file mode 100644
index 0000000..78840b8
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/RateLimiterTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneId;
+import java.time.ZoneOffset;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
+
+class RateLimiterTest {
+
+  @Test
+  void allowsRequestsUpToTheWindowLimit() {
+    RateLimiter limiter =
+        new RateLimiter(3, Clock.fixed(Instant.parse("2026-05-28T10:00:00Z"), ZoneOffset.UTC));
+
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isFalse();
+    assertThat(limiter.currentUsage()).isEqualTo(3);
+  }
+
+  @Test
+  void expiresRequestsOutsideTheWindow() {
+    AtomicReference<Instant> now = new AtomicReference<>(Instant.parse("2026-05-28T10:00:00Z"));
+    Clock clock =
+        new Clock() {
+          @Override
+          public Instant instant() {
+            return now.get();
+          }
+
+          @Override
+          public ZoneId getZone() {
+            return ZoneOffset.UTC;
+          }
+
+          @Override
+          public Clock withZone(ZoneId zone) {
+            return this;
+          }
+        };
+    RateLimiter limiter = new RateLimiter(2, clock);
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isFalse();
+
+    now.set(now.get().plusSeconds(61));
+
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.currentUsage()).isEqualTo(1);
+  }
+}


## `feat(real): persist monthly request quotas`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/QuotaCounter.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/QuotaCounter.java
new file mode 100644
index 0000000..3e93633
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/QuotaCounter.java
@@ -0,0 +1,8 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+public interface QuotaCounter {
+
+  long increment();
+
+  long current();
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounter.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounter.java
new file mode 100644
index 0000000..f05ae3f
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounter.java
@@ -0,0 +1,44 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import java.time.Clock;
+import java.time.Duration;
+import java.time.ZoneOffset;
+import java.time.format.DateTimeFormatter;
+import org.springframework.context.annotation.Profile;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+@Profile("real")
+public class RedisQuotaCounter implements QuotaCounter {
+
+  static final String KEY_PREFIX = "oddsfeed:provider-quota:";
+  static final Duration TTL = Duration.ofDays(35);
+  private static final DateTimeFormatter MONTH = DateTimeFormatter.ofPattern("yyyy-MM");
+
+  private final StringRedisTemplate redis;
+  private final Clock clock;
+
+  public RedisQuotaCounter(StringRedisTemplate redis, Clock clock) {
+    this.redis = redis;
+    this.clock = clock;
+  }
+
+  @Override
+  public long increment() {
+    String key = currentKey();
+    Long value = redis.opsForValue().increment(key);
+    redis.expire(key, TTL);
+    return value == null ? 0 : value;
+  }
+
+  @Override
+  public long current() {
+    String value = redis.opsForValue().get(currentKey());
+    return value == null ? 0 : Long.parseLong(value);
+  }
+
+  String currentKey() {
+    return KEY_PREFIX + MONTH.format(clock.instant().atZone(ZoneOffset.UTC));
+  }
+}


## `test(real): verify monthly quota accounting`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounterTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounterTest.java
new file mode 100644
index 0000000..5da73a2
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/RedisQuotaCounterTest.java
@@ -0,0 +1,76 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.reflect.Proxy;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.data.redis.core.ValueOperations;
+
+class RedisQuotaCounterTest {
+
+  private static final String KEY = RedisQuotaCounter.KEY_PREFIX + "2026-05";
+
+  private final RecordingRedis redis = new RecordingRedis();
+  private final RedisQuotaCounter counter =
+      new RedisQuotaCounter(
+          redis, Clock.fixed(Instant.parse("2026-05-28T10:00:00Z"), ZoneOffset.UTC));
+
+  @Test
+  void incrementsTheCurrentUtcMonthAndRefreshesExpiry() {
+    redis.incremented = 7L;
+
+    assertThat(counter.increment()).isEqualTo(7);
+    assertThat(redis.incrementedKey).isEqualTo(KEY);
+    assertThat(redis.expiredKey).isEqualTo(KEY);
+    assertThat(redis.expiry).isEqualTo(RedisQuotaCounter.TTL);
+  }
+
+  @Test
+  void readsCurrentUsageWithoutMutation() {
+    redis.stored = "19";
+
+    assertThat(counter.current()).isEqualTo(19);
+    assertThat(redis.readKey).isEqualTo(KEY);
+  }
+
+  private static final class RecordingRedis extends StringRedisTemplate {
+    private Long incremented;
+    private String stored;
+    private String incrementedKey;
+    private String readKey;
+    private String expiredKey;
+    private Duration expiry;
+
+    @Override
+    @SuppressWarnings("unchecked")
+    public ValueOperations<String, String> opsForValue() {
+      return (ValueOperations<String, String>)
+          Proxy.newProxyInstance(
+              ValueOperations.class.getClassLoader(),
+              new Class<?>[] {ValueOperations.class},
+              (proxy, method, args) -> {
+                if ("increment".equals(method.getName())) {
+                  incrementedKey = (String) args[0];
+                  return incremented;
+                }
+                if ("get".equals(method.getName())) {
+                  readKey = (String) args[0];
+                  return stored;
+                }
+                return null;
+              });
+    }
+
+    @Override
+    public Boolean expire(String key, Duration timeout) {
+      expiredKey = key;
+      expiry = timeout;
+      return true;
+    }
+  }
+}


## `feat(real): configure provider HTTP access`

diff --git a/src/main/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfig.java b/src/main/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfig.java
new file mode 100644
index 0000000..b407b58
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfig.java
@@ -0,0 +1,23 @@
+package com.sportsbook.oddsfeed.config;
+
+import com.sportsbook.oddsfeed.provider.real.RateLimiter;
+import java.time.Clock;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.context.annotation.Profile;
+import org.springframework.web.reactive.function.client.WebClient;
+
+@Configuration
+@Profile("real")
+public class TheOddsApiClientConfig {
+
+  @Bean
+  public WebClient theOddsWebClient(RealProperties properties) {
+    return WebClient.builder().baseUrl(properties.baseUrl()).build();
+  }
+
+  @Bean
+  public RateLimiter theOddsRateLimiter(RealProperties properties, Clock clock) {
+    return new RateLimiter(properties.rateLimit().maxRequestsPerMinute(), clock);
+  }
+}


## `test(real): verify provider client configuration`

diff --git a/src/test/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfigTest.java b/src/test/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfigTest.java
new file mode 100644
index 0000000..48a5cc7
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/config/TheOddsApiClientConfigTest.java
@@ -0,0 +1,55 @@
+package com.sportsbook.oddsfeed.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.net.URI;
+import java.time.Clock;
+import java.time.Duration;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.concurrent.atomic.AtomicReference;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.reactive.function.client.ClientResponse;
+import reactor.core.publisher.Mono;
+
+class TheOddsApiClientConfigTest {
+
+  private static final RealProperties PROPERTIES =
+      new RealProperties(
+          "key",
+          "https://odds.example",
+          List.of("soccer_epl"),
+          new RealProperties.RateLimit(1),
+          500,
+          60);
+
+  @Test
+  void webClientUsesTheConfiguredBaseUrl() {
+    AtomicReference<URI> requested = new AtomicReference<>();
+    var configured = new TheOddsApiClientConfig().theOddsWebClient(PROPERTIES);
+    var client =
+        configured
+            .mutate()
+            .exchangeFunction(
+                request -> {
+                  requested.set(request.url());
+                  return Mono.just(ClientResponse.create(HttpStatus.NO_CONTENT).build());
+                })
+            .build();
+
+    client.get().uri("/status").retrieve().toBodilessEntity().block(Duration.ofSeconds(1));
+
+    assertThat(requested.get()).isEqualTo(URI.create("https://odds.example/status"));
+  }
+
+  @Test
+  void rateLimiterUsesTheConfiguredBudget() {
+    var clock = Clock.fixed(Instant.parse("2026-05-28T10:00:00Z"), ZoneOffset.UTC);
+    var limiter = new TheOddsApiClientConfig().theOddsRateLimiter(PROPERTIES, clock);
+
+    assertThat(limiter.tryAcquire()).isTrue();
+    assertThat(limiter.tryAcquire()).isFalse();
+  }
+}


## `feat(real): map provider response payloads`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtos.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtos.java
new file mode 100644
index 0000000..8c8f4d1
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtos.java
@@ -0,0 +1,31 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import com.fasterxml.jackson.databind.PropertyNamingStrategies.SnakeCaseStrategy;
+import com.fasterxml.jackson.databind.annotation.JsonNaming;
+import java.math.BigDecimal;
+import java.time.Instant;
+import java.util.List;
+
+public final class TheOddsApiDtos {
+
+  private TheOddsApiDtos() {}
+
+  @JsonNaming(SnakeCaseStrategy.class)
+  public record Event(
+      String id,
+      String sportKey,
+      String sportTitle,
+      Instant commenceTime,
+      String homeTeam,
+      String awayTeam,
+      List<Bookmaker> bookmakers) {}
+
+  @JsonNaming(SnakeCaseStrategy.class)
+  public record Bookmaker(String key, String title, Instant lastUpdate, List<Market> markets) {}
+
+  @JsonNaming(SnakeCaseStrategy.class)
+  public record Market(String key, Instant lastUpdate, List<Outcome> outcomes) {}
+
+  @JsonNaming(SnakeCaseStrategy.class)
+  public record Outcome(String name, BigDecimal price) {}
+}


## `test(real): verify provider payload mapping`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtosTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtosTest.java
new file mode 100644
index 0000000..57f9283
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiDtosTest.java
@@ -0,0 +1,49 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.math.BigDecimal;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class TheOddsApiDtosTest {
+
+  private static final String PAYLOAD =
+      """
+      {
+        "id": "abc123",
+        "sport_key": "soccer_epl",
+        "sport_title": "EPL",
+        "commence_time": "2026-06-01T18:00:00Z",
+        "home_team": "Manchester United",
+        "away_team": "Chelsea",
+        "bookmakers": [{
+          "key": "book",
+          "title": "Book",
+          "last_update": "2026-05-28T10:00:00Z",
+          "markets": [{
+            "key": "h2h",
+            "last_update": "2026-05-28T10:00:00Z",
+            "outcomes": [{"name": "Chelsea", "price": 4.20}]
+          }]
+        }]
+      }
+      """;
+
+  @Test
+  void mapsSnakeCaseEventAndMarketFields() throws Exception {
+    ObjectMapper mapper = new ObjectMapper().findAndRegisterModules();
+
+    TheOddsApiDtos.Event event = mapper.readValue(PAYLOAD, TheOddsApiDtos.Event.class);
+
+    assertThat(event.id()).isEqualTo("abc123");
+    assertThat(event.sportKey()).isEqualTo("soccer_epl");
+    assertThat(event.commenceTime()).isEqualTo(Instant.parse("2026-06-01T18:00:00Z"));
+    assertThat(event.homeTeam()).isEqualTo("Manchester United");
+    var bookmaker = event.bookmakers().get(0);
+    assertThat(bookmaker.lastUpdate()).isEqualTo(Instant.parse("2026-05-28T10:00:00Z"));
+    assertThat(bookmaker.markets().get(0).outcomes().get(0).price())
+        .isEqualByComparingTo(new BigDecimal("4.20"));
+  }
+}


## `feat(real): list provider events within budget`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
new file mode 100644
index 0000000..86d6f5c
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/real/TheOddsApiProvider.java
@@ -0,0 +1,88 @@
+package com.sportsbook.oddsfeed.provider.real;
+
+import com.sportsbook.oddsfeed.config.RealProperties;
+import com.sportsbook.oddsfeed.provider.EventSummary;
+import com.sportsbook.oddsfeed.provider.Sport;
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.context.annotation.Profile;
+import org.springframework.stereotype.Component;
+import org.springframework.web.reactive.function.client.WebClient;
+
+@Component
+@Profile("real")
+public class TheOddsApiProvider {
+
+  private static final Duration FETCH_TIMEOUT = Duration.ofSeconds(10);
+
+  private final WebClient client;
+  private final RealProperties properties;
+  private final RateLimiter rateLimiter;
+  private final QuotaCounter quotaCounter;
+
+  public TheOddsApiProvider(
+      WebClient theOddsWebClient,
+      RealProperties properties,
+      RateLimiter rateLimiter,
+      QuotaCounter quotaCounter) {
+    this.client = theOddsWebClient;
+    this.properties = properties;
+    this.rateLimiter = rateLimiter;
+    this.quotaCounter = quotaCounter;
+  }
+
+  public List<EventSummary> listEvents(Sport sport) {
+    String sportKey = sportKey(sport);
+    if (sportKey == null) {
+      return List.of();
+    }
+    return fetch(sportKey).stream().map(event -> toSummary(event, sport)).toList();
+  }
+
+  private List<TheOddsApiDtos.Event> fetch(String sportKey) {
+    if (!rateLimiter.tryAcquire() || quotaCounter.increment() > properties.monthlyQuota()) {
+      return List.of();
+    }
+    TheOddsApiDtos.Event[] response =
+        client
+            .get()
+            .uri(
+                builder ->
+                    builder
+                        .path("/sports/{key}/odds")
+                        .queryParam("apiKey", properties.apiKey())
+                        .queryParam("regions", "uk")
+                        .queryParam("markets", "h2h")
+                        .queryParam("oddsFormat", "decimal")
+                        .build(sportKey))
+            .retrieve()
+            .bodyToMono(TheOddsApiDtos.Event[].class)
+            .block(FETCH_TIMEOUT);
+    return response == null ? List.of() : List.of(response);
+  }
+
+  private String sportKey(Sport sport) {
+    String preferred =
+        switch (sport) {
+          case FOOTBALL -> "soccer_epl";
+          case BASKETBALL -> "basketball_nba";
+        };
+    return properties.sportKeys().contains(preferred) ? preferred : null;
+  }
+
+  private EventSummary toSummary(TheOddsApiDtos.Event event, Sport sport) {
+    EventId id = new EventId(UUID.nameUUIDFromBytes(event.id().getBytes(StandardCharsets.UTF_8)));
+    return new EventSummary(
+        id,
+        sport,
+        event.sportTitle(),
+        event.homeTeam(),
+        event.awayTeam(),
+        event.commenceTime(),
+        EventLifecycleStatus.SCHEDULED);
+  }
+}


