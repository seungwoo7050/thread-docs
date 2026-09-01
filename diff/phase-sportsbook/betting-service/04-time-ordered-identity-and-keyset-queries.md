# 시간 정렬 식별자와 키셋 조회

## `feat(identifier): generate time-ordered bet ids`

diff --git a/src/main/java/com/sportsbook/betting/infrastructure/id/UuidV7.java b/src/main/java/com/sportsbook/betting/infrastructure/id/UuidV7.java
new file mode 100644
index 0000000..79d71db
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/infrastructure/id/UuidV7.java
@@ -0,0 +1,25 @@
+package com.sportsbook.betting.infrastructure.id;
+
+import java.security.SecureRandom;
+import java.util.UUID;
+
+public final class UuidV7 {
+
+  private static final SecureRandom RANDOM = new SecureRandom();
+  private static final long TIMESTAMP_MASK = 0xFFFF_FFFF_FFFFL;
+  private static final long VERSION_MASK = 0x7000L;
+  private static final long VARIANT_MASK = 0x8000_0000_0000_0000L;
+
+  public static UUID generate() {
+    return generate(System.currentTimeMillis());
+  }
+
+  static UUID generate(long unixMillis) {
+    long mostSignificantBits = ((unixMillis & TIMESTAMP_MASK) << 16) | VERSION_MASK;
+    mostSignificantBits |= RANDOM.nextInt(0x1000);
+    long leastSignificantBits = (RANDOM.nextLong() & 0x3FFF_FFFF_FFFF_FFFFL) | VARIANT_MASK;
+    return new UUID(mostSignificantBits, leastSignificantBits);
+  }
+
+  private UuidV7() {}
+}


## `test(identifier): verify UUID version and time`

diff --git a/src/test/java/com/sportsbook/betting/infrastructure/id/UuidV7Test.java b/src/test/java/com/sportsbook/betting/infrastructure/id/UuidV7Test.java
new file mode 100644
index 0000000..582c753
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/infrastructure/id/UuidV7Test.java
@@ -0,0 +1,19 @@
+package com.sportsbook.betting.infrastructure.id;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class UuidV7Test {
+
+  @Test
+  void stampsVersionVariantAndTime() {
+    long timestamp = 1_700_000_000_123L;
+    UUID id = UuidV7.generate(timestamp);
+
+    assertThat(id.version()).isEqualTo(7);
+    assertThat(id.variant()).isEqualTo(2);
+    assertThat(id.getMostSignificantBits() >>> 16).isEqualTo(timestamp);
+  }
+}


## `feat(identifier): generate human bet references`

diff --git a/src/main/java/com/sportsbook/betting/infrastructure/id/BetReferenceGenerator.java b/src/main/java/com/sportsbook/betting/infrastructure/id/BetReferenceGenerator.java
new file mode 100644
index 0000000..c4a942c
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/infrastructure/id/BetReferenceGenerator.java
@@ -0,0 +1,24 @@
+package com.sportsbook.betting.infrastructure.id;
+
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.time.format.DateTimeFormatter;
+import java.util.concurrent.ThreadLocalRandom;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetReferenceGenerator {
+
+  private static final char[] BASE36 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ".toCharArray();
+  private static final DateTimeFormatter DATE =
+      DateTimeFormatter.ofPattern("yyyy-MM-dd").withZone(ZoneOffset.UTC);
+
+  public String next(Instant at) {
+    StringBuilder value = new StringBuilder("B-").append(DATE.format(at)).append('-');
+    ThreadLocalRandom random = ThreadLocalRandom.current();
+    for (int index = 0; index < 8; index++) {
+      value.append(BASE36[random.nextInt(BASE36.length)]);
+    }
+    return value.toString();
+  }
+}


## `test(identifier): verify reference format`

diff --git a/src/test/java/com/sportsbook/betting/infrastructure/id/BetReferenceGeneratorTest.java b/src/test/java/com/sportsbook/betting/infrastructure/id/BetReferenceGeneratorTest.java
new file mode 100644
index 0000000..e09139e
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/infrastructure/id/BetReferenceGeneratorTest.java
@@ -0,0 +1,16 @@
+package com.sportsbook.betting.infrastructure.id;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+
+class BetReferenceGeneratorTest {
+
+  @Test
+  void includesUtcDateAndBase36Suffix() {
+    String reference = new BetReferenceGenerator().next(Instant.parse("2026-08-22T23:30:00Z"));
+
+    assertThat(reference).matches("B-2026-08-22-[0-9A-Z]{8}");
+  }
+}


## `feat(api): query actor scoped bet history`

diff --git a/src/main/java/com/sportsbook/betting/api/CursorPage.java b/src/main/java/com/sportsbook/betting/api/CursorPage.java
new file mode 100644
index 0000000..5f332ae
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/CursorPage.java
@@ -0,0 +1,5 @@
+package com.sportsbook.betting.api;
+
+import java.util.List;
+
+public record CursorPage<T>(List<T> items, String nextCursor, boolean hasMore) {}
diff --git a/src/main/java/com/sportsbook/betting/placement/BetQueryService.java b/src/main/java/com/sportsbook/betting/placement/BetQueryService.java
new file mode 100644
index 0000000..f13c988
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/BetQueryService.java
@@ -0,0 +1,49 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.api.CursorPage;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.BetNotFoundException;
+import com.sportsbook.betting.persistence.BetRepository;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.domain.PageRequest;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class BetQueryService {
+
+  private static final int DEFAULT_LIMIT = 20;
+  private static final int MAX_LIMIT = 100;
+
+  private final BetRepository bets;
+
+  public BetQueryService(BetRepository bets) {
+    this.bets = bets;
+  }
+
+  @Transactional(readOnly = true)
+  public Bet byId(UUID actorId, UUID betId) {
+    return bets.findWithLegsByBetId(betId)
+        .filter(bet -> bet.userId().equals(actorId))
+        .orElseThrow(() -> new BetNotFoundException("No bet with id " + betId));
+  }
+
+  @Transactional(readOnly = true)
+  public CursorPage<Bet> page(UUID actorId, UUID cursor, Integer requestedLimit) {
+    int limit =
+        requestedLimit == null || requestedLimit <= 0
+            ? DEFAULT_LIMIT
+            : Math.min(requestedLimit, MAX_LIMIT);
+    PageRequest probe = PageRequest.of(0, limit + 1);
+    List<Bet> rows =
+        cursor == null
+            ? bets.findByUserIdOrderByBetIdDesc(actorId, probe)
+            : bets.findByUserIdAndBetIdLessThanOrderByBetIdDesc(actorId, cursor, probe);
+    boolean hasMore = rows.size() > limit;
+    List<Bet> items = hasMore ? rows.subList(0, limit) : rows;
+    String next =
+        hasMore && !items.isEmpty() ? items.get(items.size() - 1).betId().toString() : null;
+    return new CursorPage<>(List.copyOf(items), next, hasMore);
+  }
+}


## `test(api): verify actor scoped keyset queries`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetQueryServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetQueryServiceTest.java
new file mode 100644
index 0000000..85df2ef
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetQueryServiceTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.BetNotFoundException;
+import com.sportsbook.betting.persistence.BetRepository;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.PageRequest;
+
+class BetQueryServiceTest {
+
+  @Test
+  void hidesBetsOwnedByAnotherActor() {
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    when(bet.userId()).thenReturn(UUID.randomUUID());
+    when(bets.findWithLegsByBetId(betId)).thenReturn(Optional.of(bet));
+
+    assertThatThrownBy(() -> new BetQueryService(bets).byId(UUID.randomUUID(), betId))
+        .isInstanceOf(BetNotFoundException.class);
+  }
+
+  @Test
+  void usesBoundedKeysetPaginationAndReturnsTheLastVisibleCursor() {
+    BetRepository bets = mock(BetRepository.class);
+    UUID actorId = UUID.randomUUID();
+    Bet first = mock(Bet.class);
+    Bet second = mock(Bet.class);
+    UUID firstId = UUID.randomUUID();
+    UUID secondId = UUID.randomUUID();
+    when(first.betId()).thenReturn(firstId);
+    when(second.betId()).thenReturn(secondId);
+    when(bets.findByUserIdOrderByBetIdDesc(actorId, PageRequest.of(0, 2)))
+        .thenReturn(List.of(first, second));
+
+    var page = new BetQueryService(bets).page(actorId, null, 1);
+
+    assertThat(page.items()).containsExactly(first);
+    assertThat(page.nextCursor()).isEqualTo(firstId.toString());
+    assertThat(page.hasMore()).isTrue();
+  }
+}
