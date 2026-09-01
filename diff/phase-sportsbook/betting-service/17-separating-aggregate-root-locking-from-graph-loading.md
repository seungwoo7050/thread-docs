# 집계 루트 잠금과 그래프 로딩 분리

## `feat(persistence): load owned bets with evidence`

diff --git a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
new file mode 100644
index 0000000..800a93b
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
@@ -0,0 +1,36 @@
+package com.sportsbook.betting.persistence;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.protocol.domain.BetStatus;
+import jakarta.persistence.LockModeType;
+import java.time.Instant;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.data.domain.Pageable;
+import org.springframework.data.jpa.repository.EntityGraph;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Lock;
+
+public interface BetRepository extends JpaRepository<Bet, UUID> {
+
+  @EntityGraph(attributePaths = "legs")
+  Optional<Bet> findByIdempotencyKey(String idempotencyKey);
+
+  @EntityGraph(attributePaths = "legs")
+  Optional<Bet> findWithLegsByBetId(UUID betId);
+
+  @Lock(LockModeType.PESSIMISTIC_WRITE)
+  @EntityGraph(attributePaths = "legs")
+  Optional<Bet> findLockedByBetId(UUID betId);
+
+  @EntityGraph(attributePaths = "legs")
+  List<Bet> findByStatusAndCreatedAtBefore(BetStatus status, Instant threshold, Pageable pageable);
+
+  @EntityGraph(attributePaths = "legs")
+  List<Bet> findByUserIdOrderByBetIdDesc(UUID userId, Pageable pageable);
+
+  @EntityGraph(attributePaths = "legs")
+  List<Bet> findByUserIdAndBetIdLessThanOrderByBetIdDesc(
+      UUID userId, UUID cursor, Pageable pageable);
+}


## `test(persistence): verify transition lock contract`

diff --git a/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java
new file mode 100644
index 0000000..67a3db9
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java
@@ -0,0 +1,20 @@
+package com.sportsbook.betting.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import jakarta.persistence.LockModeType;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.jpa.repository.EntityGraph;
+import org.springframework.data.jpa.repository.Lock;
+
+class BetRepositoryContractTest {
+
+  @Test
+  void locksAggregateBeforeSagaOrResolutionTransition() throws Exception {
+    var method = BetRepository.class.getMethod("findLockedByBetId", UUID.class);
+
+    assertThat(method.getAnnotation(Lock.class).value()).isEqualTo(LockModeType.PESSIMISTIC_WRITE);
+    assertThat(method.getAnnotation(EntityGraph.class).attributePaths()).containsExactly("legs");
+  }
+}


## `fix(persistence): lock bet roots before loading legs`

diff --git a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
index 252100e..3b4730a 100644
--- a/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
+++ b/src/main/java/com/sportsbook/betting/persistence/BetRepository.java
@@ -22,9 +22,18 @@ public interface BetRepository extends JpaRepository<Bet, UUID> {
   @EntityGraph(attributePaths = "legs")
   Optional<Bet> findWithLegsByBetId(UUID betId);
 
+  default Optional<Bet> findLockedByBetId(UUID betId) {
+    return findLockedRootByBetId(betId)
+        .map(
+            bet -> {
+              bet.legs().size();
+              return bet;
+            });
+  }
+
   @Lock(LockModeType.PESSIMISTIC_WRITE)
-  @EntityGraph(attributePaths = "legs")
-  Optional<Bet> findLockedByBetId(UUID betId);
+  @Query("select bet from Bet bet where bet.betId = :betId")
+  Optional<Bet> findLockedRootByBetId(@Param("betId") UUID betId);
 
   @Transactional
   @Query(


## `test(persistence): separate root locking from graph loading`

diff --git a/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java
index 67a3db9..da63a34 100644
--- a/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/BetRepositoryContractTest.java
@@ -11,10 +11,12 @@ import org.springframework.data.jpa.repository.Lock;
 class BetRepositoryContractTest {
 
   @Test
-  void locksAggregateBeforeSagaOrResolutionTransition() throws Exception {
-    var method = BetRepository.class.getMethod("findLockedByBetId", UUID.class);
+  void locksTheRootBeforeLoadingTheAggregateGraph() throws Exception {
+    var aggregate = BetRepository.class.getMethod("findLockedByBetId", UUID.class);
+    var root = BetRepository.class.getMethod("findLockedRootByBetId", UUID.class);
 
-    assertThat(method.getAnnotation(Lock.class).value()).isEqualTo(LockModeType.PESSIMISTIC_WRITE);
-    assertThat(method.getAnnotation(EntityGraph.class).attributePaths()).containsExactly("legs");
+    assertThat(aggregate.isDefault()).isTrue();
+    assertThat(root.getAnnotation(Lock.class).value()).isEqualTo(LockModeType.PESSIMISTIC_WRITE);
+    assertThat(root.getAnnotation(EntityGraph.class)).isNull();
   }
 }
