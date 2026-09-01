## `feat(limits): persist user overrides in Redis`

diff --git a/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java b/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java
new file mode 100644
index 0000000..428d373
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java
@@ -0,0 +1,57 @@
+package com.sportsbook.risk.limit;
+
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.Objects;
+import java.util.OptionalLong;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Component;
+
+/** Redis hash implementation of authoritative user-specific limit overrides. */
+@Component
+public final class RedisLimitOverrideStore implements LimitOverrideStore {
+  private static final String PREFIX = "risk:limit:override:";
+
+  private final StringRedisTemplate redis;
+
+  public RedisLimitOverrideStore(StringRedisTemplate redis) {
+    this.redis = Objects.requireNonNull(redis, "redis");
+  }
+
+  @Override
+  public OptionalLong find(UserId userId, LimitOverrideField field) {
+    String value =
+        (String)
+            redis
+                .opsForHash()
+                .get(key(userId), Objects.requireNonNull(field, "field").redisField());
+    if (value == null) {
+      return OptionalLong.empty();
+    }
+    try {
+      return OptionalLong.of(
+          SafeRedisNumber.requireNonNegative(Long.parseLong(value), "stored override"));
+    } catch (NumberFormatException exception) {
+      throw new IllegalStateException("stored override is not an integer", exception);
+    }
+  }
+
+  @Override
+  public void set(UserId userId, LimitOverrideField field, long value) {
+    SafeRedisNumber.requireNonNegative(value, "override");
+    redis
+        .opsForHash()
+        .put(
+            key(userId), Objects.requireNonNull(field, "field").redisField(), Long.toString(value));
+  }
+
+  @Override
+  public void clear(UserId userId, LimitOverrideField field) {
+    redis.opsForHash().delete(key(userId), Objects.requireNonNull(field, "field").redisField());
+  }
+
+  static String key(UserId userId) {
+    Objects.requireNonNull(userId, "userId");
+    return PREFIX + "{" + userId.value() + "}";
+  }
+}


## `test(limits): verify Redis override round trips`

diff --git a/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java b/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java
new file mode 100644
index 0000000..bad5276
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java
@@ -0,0 +1,35 @@
+package com.sportsbook.risk.limit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.support.RedisTestSupport;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RedisLimitOverrideStoreTest extends RedisTestSupport {
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+
+  @Test
+  void roundTripsIsolatedOverrideDimensions() {
+    RedisLimitOverrideStore store = new RedisLimitOverrideStore(redis);
+    LimitOverrideField krw = LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.KRW);
+    LimitOverrideField usd = LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.USD);
+    LimitOverrideField selections = LimitOverrideField.selections();
+
+    assertThat(store.find(USER, krw)).isEmpty();
+    store.set(USER, krw, 1000);
+    store.set(USER, usd, 10);
+    store.set(USER, selections, 7);
+
+    assertThat(store.find(USER, krw)).hasValue(1000);
+    assertThat(store.find(USER, usd)).hasValue(10);
+    assertThat(store.find(USER, selections)).hasValue(7);
+    store.clear(USER, krw);
+    assertThat(store.find(USER, krw)).isEmpty();
+    assertThat(store.find(USER, usd)).hasValue(10);
+  }
+}


## `test(limits): reject corrupt Redis overrides`

diff --git a/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java b/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java
index bad5276..701e538 100644
--- a/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java
+++ b/src/test/java/com/sportsbook/risk/limit/RedisLimitOverrideStoreTest.java
@@ -1,10 +1,12 @@
 package com.sportsbook.risk.limit;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.UserId;
 import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.SafeRedisNumber;
 import com.sportsbook.risk.support.RedisTestSupport;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
@@ -32,4 +34,21 @@ class RedisLimitOverrideStoreTest extends RedisTestSupport {
     assertThat(store.find(USER, krw)).isEmpty();
     assertThat(store.find(USER, usd)).hasValue(10);
   }
+
+  @Test
+  void failsClosedForUnsafeWritesAndCorruptStoredValues() {
+    RedisLimitOverrideStore store = new RedisLimitOverrideStore(redis);
+    LimitOverrideField field = LimitOverrideField.monetary(LimitType.STAKE_MONTHLY, Currency.KRW);
+
+    assertThatThrownBy(() -> store.set(USER, field, -1))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> store.set(USER, field, SafeRedisNumber.MAX_VALUE + 1))
+        .isInstanceOf(IllegalArgumentException.class);
+    redis.opsForHash().put(RedisLimitOverrideStore.key(USER), field.redisField(), "corrupt");
+    assertThatThrownBy(() -> store.find(USER, field))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessage("stored override is not an integer");
+    redis.opsForHash().put(RedisLimitOverrideStore.key(USER), field.redisField(), "-1");
+    assertThatThrownBy(() -> store.find(USER, field)).isInstanceOf(IllegalArgumentException.class);
+  }
 }


## `refactor(limits): share override key contract`

diff --git a/src/main/java/com/sportsbook/risk/limit/LimitOverrideKeys.java b/src/main/java/com/sportsbook/risk/limit/LimitOverrideKeys.java
new file mode 100644
index 0000000..7985cf6
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/limit/LimitOverrideKeys.java
@@ -0,0 +1,16 @@
+package com.sportsbook.risk.limit;
+
+import com.sportsbook.protocol.value.UserId;
+import java.util.Objects;
+
+/** Canonical Redis hash key for one user's administrative risk overrides. */
+public final class LimitOverrideKeys {
+  private static final String PREFIX = "risk:limit:override:";
+
+  private LimitOverrideKeys() {}
+
+  public static String user(UserId userId) {
+    Objects.requireNonNull(userId, "userId");
+    return PREFIX + "{" + userId.value() + "}";
+  }
+}
diff --git a/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java b/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java
index 428d373..794a0f0 100644
--- a/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java
+++ b/src/main/java/com/sportsbook/risk/limit/RedisLimitOverrideStore.java
@@ -10,8 +10,6 @@ import org.springframework.stereotype.Component;
 /** Redis hash implementation of authoritative user-specific limit overrides. */
 @Component
 public final class RedisLimitOverrideStore implements LimitOverrideStore {
-  private static final String PREFIX = "risk:limit:override:";
-
   private final StringRedisTemplate redis;
 
   public RedisLimitOverrideStore(StringRedisTemplate redis) {
@@ -24,7 +22,9 @@ public final class RedisLimitOverrideStore implements LimitOverrideStore {
         (String)
             redis
                 .opsForHash()
-                .get(key(userId), Objects.requireNonNull(field, "field").redisField());
+                .get(
+                    LimitOverrideKeys.user(userId),
+                    Objects.requireNonNull(field, "field").redisField());
     if (value == null) {
       return OptionalLong.empty();
     }
@@ -42,16 +42,20 @@ public final class RedisLimitOverrideStore implements LimitOverrideStore {
     redis
         .opsForHash()
         .put(
-            key(userId), Objects.requireNonNull(field, "field").redisField(), Long.toString(value));
+            LimitOverrideKeys.user(userId),
+            Objects.requireNonNull(field, "field").redisField(),
+            Long.toString(value));
   }
 
   @Override
   public void clear(UserId userId, LimitOverrideField field) {
-    redis.opsForHash().delete(key(userId), Objects.requireNonNull(field, "field").redisField());
+    redis
+        .opsForHash()
+        .delete(
+            LimitOverrideKeys.user(userId), Objects.requireNonNull(field, "field").redisField());
   }
 
   static String key(UserId userId) {
-    Objects.requireNonNull(userId, "userId");
-    return PREFIX + "{" + userId.value() + "}";
+    return LimitOverrideKeys.user(userId);
   }
 }
