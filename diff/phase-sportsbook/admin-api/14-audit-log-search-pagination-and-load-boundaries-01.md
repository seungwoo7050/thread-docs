# 감사 로그 검색, 페이지네이션, 부하 경계

## `feat(audit): expose the audit read repository`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java b/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java
new file mode 100644
index 0000000..a69c359
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java
@@ -0,0 +1,8 @@
+package com.sportsbook.admin.audit;
+
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
+
+public interface AuditLogRepository
+    extends JpaRepository<AuditLogEntity, UUID>, JpaSpecificationExecutor<AuditLogEntity> {}


## `feat(audit): query actions by actor and time`

diff --git a/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java b/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java
index a69c359..91d4022 100644
--- a/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java
+++ b/src/main/java/com/sportsbook/admin/audit/AuditLogRepository.java
@@ -1,8 +1,24 @@
 package com.sportsbook.admin.audit;
 
+import java.time.Instant;
 import java.util.UUID;
+import org.springframework.data.domain.Page;
+import org.springframework.data.domain.Pageable;
 import org.springframework.data.jpa.repository.JpaRepository;
 import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
 
 public interface AuditLogRepository
-    extends JpaRepository<AuditLogEntity, UUID>, JpaSpecificationExecutor<AuditLogEntity> {}
+    extends JpaRepository<AuditLogEntity, UUID>, JpaSpecificationExecutor<AuditLogEntity> {
+
+  @Query(
+      "SELECT audit FROM AuditLogEntity audit "
+          + "WHERE audit.startedAt >= :from AND audit.startedAt < :to "
+          + "AND (:actor IS NULL OR audit.actorId = :actor)")
+  Page<AuditLogEntity> search(
+      @Param("from") Instant from,
+      @Param("to") Instant to,
+      @Param("actor") String actor,
+      Pageable pageable);
+}


## `feat(audit): expose lifecycle read models`

diff --git a/src/main/java/com/sportsbook/admin/api/AuditLogView.java b/src/main/java/com/sportsbook/admin/api/AuditLogView.java
new file mode 100644
index 0000000..a4e925b
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AuditLogView.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AuditLogEntity;
+import java.time.Instant;
+import java.util.UUID;
+
+public record AuditLogView(
+    UUID actionId,
+    String actorId,
+    String actorRole,
+    String action,
+    String target,
+    String outcome,
+    Integer httpStatus,
+    String reason,
+    String traceId,
+    Instant startedAt,
+    Instant completedAt) {
+
+  public static AuditLogView from(AuditLogEntity entity) {
+    return new AuditLogView(
+        entity.getActionId(),
+        entity.getActorId(),
+        entity.getActorRole().name(),
+        entity.getAction(),
+        entity.getTarget(),
+        entity.getOutcome().name(),
+        entity.getHttpStatus(),
+        entity.getReason(),
+        entity.getTraceId(),
+        entity.getStartedAt(),
+        entity.getCompletedAt());
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/api/OffsetPage.java b/src/main/java/com/sportsbook/admin/api/OffsetPage.java
new file mode 100644
index 0000000..4f24c48
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/OffsetPage.java
@@ -0,0 +1,16 @@
+package com.sportsbook.admin.api;
+
+import java.util.List;
+import org.springframework.data.domain.Page;
+
+public record OffsetPage<T>(List<T> items, int page, int size, long totalElements, int totalPages) {
+
+  public static <T> OffsetPage<T> from(Page<?> source, List<T> items) {
+    return new OffsetPage<>(
+        List.copyOf(items),
+        source.getNumber(),
+        source.getSize(),
+        source.getTotalElements(),
+        source.getTotalPages());
+  }
+}


## `test(audit): preserve lifecycle query fields`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditLogViewTest.java b/src/test/java/com/sportsbook/admin/audit/AuditLogViewTest.java
new file mode 100644
index 0000000..f6a1b0f
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/audit/AuditLogViewTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.admin.audit;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.admin.api.AuditLogView;
+import com.sportsbook.admin.api.OffsetPage;
+import com.sportsbook.admin.security.AdminRole;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.PageImpl;
+import org.springframework.data.domain.PageRequest;
+
+class AuditLogViewTest {
+
+  @Test
+  void representsAnUnfinishedActionWithoutInventingTerminalFields() {
+    Instant started = Instant.parse("2026-08-22T01:02:03Z");
+    AuditLogEntity entity =
+        new AuditLogEntity(
+            UUID.fromString("018f0000-0000-7000-8000-000000000096"),
+            "operator-1",
+            AdminRole.READONLY,
+            "AUDIT_SEARCH",
+            null,
+            AuditOutcome.STARTED,
+            null,
+            null,
+            "trace-1",
+            started,
+            null);
+
+    AuditLogView view = AuditLogView.from(entity);
+
+    assertThat(view.actorRole()).isEqualTo("READONLY");
+    assertThat(view.outcome()).isEqualTo("STARTED");
+    assertThat(view.httpStatus()).isNull();
+    assertThat(view.startedAt()).isEqualTo(started);
+    assertThat(view.completedAt()).isNull();
+  }
+
+  @Test
+  void copiesPageMetadataAndItems() {
+    AuditLogView item = AuditLogView.from(terminal());
+    var source = new PageImpl<>(List.of(terminal()), PageRequest.of(2, 1), 7);
+
+    OffsetPage<AuditLogView> page = OffsetPage.from(source, List.of(item));
+
+    assertThat(page.items()).containsExactly(item);
+    assertThat(page.page()).isEqualTo(2);
+    assertThat(page.size()).isEqualTo(1);
+    assertThat(page.totalElements()).isEqualTo(7);
+    assertThat(page.totalPages()).isEqualTo(7);
+  }
+
+  private static AuditLogEntity terminal() {
+    Instant started = Instant.parse("2026-08-22T01:02:03Z");
+    return new AuditLogEntity(
+        UUID.fromString("018f0000-0000-7000-8000-000000000097"),
+        "operator-1",
+        AdminRole.ADMIN,
+        "MARKET_CLOSE",
+        "market-1",
+        AuditOutcome.SUCCESS,
+        202,
+        "operator request",
+        "trace-1",
+        started,
+        started.plusSeconds(1));
+  }
+}


## `feat(audit): look up actions by identifier`

diff --git a/src/main/java/com/sportsbook/admin/api/AuditLogController.java b/src/main/java/com/sportsbook/admin/api/AuditLogController.java
index 75cf479..f297751 100644
--- a/src/main/java/com/sportsbook/admin/api/AuditLogController.java
+++ b/src/main/java/com/sportsbook/admin/api/AuditLogController.java
@@ -4,6 +4,7 @@ import com.sportsbook.admin.audit.AuditLogEntity;
 import com.sportsbook.admin.audit.AuditLogRepository;
 import java.time.Instant;
 import java.util.List;
+import java.util.UUID;
 import org.springframework.data.domain.Page;
 import org.springframework.data.domain.PageRequest;
 import org.springframework.data.domain.Sort;
@@ -11,6 +12,7 @@ import org.springframework.format.annotation.DateTimeFormat;
 import org.springframework.http.HttpStatus;
 import org.springframework.security.access.prepost.PreAuthorize;
 import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
 import org.springframework.web.bind.annotation.RequestMapping;
 import org.springframework.web.bind.annotation.RequestParam;
 import org.springframework.web.bind.annotation.RestController;
@@ -47,8 +49,7 @@ public class AuditLogController {
     int normalizedPage = Math.max(0, page);
     int normalizedSize = Math.min(size < 1 ? DEFAULT_PAGE_SIZE : size, MAX_PAGE_SIZE);
     Sort newestFirst =
-        Sort.by(Sort.Direction.DESC, "startedAt")
-            .and(Sort.by(Sort.Direction.DESC, "actionId"));
+        Sort.by(Sort.Direction.DESC, "startedAt").and(Sort.by(Sort.Direction.DESC, "actionId"));
     Page<AuditLogEntity> result =
         repository.search(
             lower,
@@ -59,6 +60,16 @@ public class AuditLogController {
     return OffsetPage.from(result, items);
   }
 
+  @GetMapping("/{actionId}")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public AuditLogView findByActionId(@PathVariable UUID actionId) {
+    return repository
+        .findById(actionId)
+        .map(AuditLogView::from)
+        .orElseThrow(
+            () -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Audit action not found"));
+  }
+
   private static String normalizeActor(String actor) {
     return actor == null || actor.isBlank() ? null : actor.trim();
   }


## `test(audit): resolve action identifiers exactly`

diff --git a/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java b/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java
index 94d2f2a..c8fbcc3 100644
--- a/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java
+++ b/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java
@@ -9,8 +9,13 @@ import static org.mockito.Mockito.never;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.admin.audit.AuditLogEntity;
 import com.sportsbook.admin.audit.AuditLogRepository;
+import com.sportsbook.admin.audit.AuditOutcome;
+import com.sportsbook.admin.security.AdminRole;
 import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
 import org.junit.jupiter.api.Test;
 import org.mockito.ArgumentCaptor;
 import org.springframework.data.domain.Page;
@@ -32,8 +37,12 @@ class AuditLogControllerTest {
     OffsetPage<AuditLogView> result = controller.search(from, to, "  ", -4, 500);
 
     ArgumentCaptor<Pageable> page = ArgumentCaptor.forClass(Pageable.class);
-    verify(repository).search(org.mockito.ArgumentMatchers.eq(from),
-        org.mockito.ArgumentMatchers.eq(to), isNull(), page.capture());
+    verify(repository)
+        .search(
+            org.mockito.ArgumentMatchers.eq(from),
+            org.mockito.ArgumentMatchers.eq(to),
+            isNull(),
+            page.capture());
     assertThat(page.getValue().getPageNumber()).isZero();
     assertThat(page.getValue().getPageSize()).isEqualTo(200);
     assertThat(page.getValue().getSort().getOrderFor("startedAt").isDescending()).isTrue();
@@ -58,11 +67,42 @@ class AuditLogControllerTest {
   void permitsEveryOperatorRole() throws NoSuchMethodException {
     PreAuthorize guard =
         AuditLogController.class
-            .getMethod(
-                "search", Instant.class, Instant.class, String.class, int.class, int.class)
+            .getMethod("search", Instant.class, Instant.class, String.class, int.class, int.class)
             .getAnnotation(PreAuthorize.class);
 
-    assertThat(guard.value())
-        .isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+    assertThat(guard.value()).isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+  }
+
+  @Test
+  void returnsTheExactActionRequested() {
+    AuditLogRepository repository = mock(AuditLogRepository.class);
+    AuditLogEntity entity = mock(AuditLogEntity.class);
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000098");
+    when(entity.getActionId()).thenReturn(actionId);
+    when(entity.getActorId()).thenReturn("operator-1");
+    when(entity.getActorRole()).thenReturn(AdminRole.CS);
+    when(entity.getAction()).thenReturn("WALLET_REFUND");
+    when(entity.getOutcome()).thenReturn(AuditOutcome.FAILED);
+    when(entity.getHttpStatus()).thenReturn(409);
+    when(repository.findById(actionId)).thenReturn(Optional.of(entity));
+
+    AuditLogView view = new AuditLogController(repository).findByActionId(actionId);
+
+    assertThat(view.actionId()).isEqualTo(actionId);
+    assertThat(view.actorRole()).isEqualTo("CS");
+    assertThat(view.outcome()).isEqualTo("FAILED");
+    assertThat(view.httpStatus()).isEqualTo(409);
+  }
+
+  @Test
+  void returnsNotFoundForAnUnknownAction() {
+    AuditLogRepository repository = mock(AuditLogRepository.class);
+    UUID actionId = UUID.fromString("018f0000-0000-7000-8000-000000000099");
+    when(repository.findById(actionId)).thenReturn(Optional.empty());
+
+    assertThatThrownBy(() -> new AuditLogController(repository).findByActionId(actionId))
+        .isInstanceOfSatisfying(
+            ResponseStatusException.class,
+            failure -> assertThat(failure.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND));
   }
 }


## `feat(audit): expose filtered audit search`

diff --git a/src/main/java/com/sportsbook/admin/api/AuditLogController.java b/src/main/java/com/sportsbook/admin/api/AuditLogController.java
new file mode 100644
index 0000000..75cf479
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AuditLogController.java
@@ -0,0 +1,65 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.audit.AuditLogEntity;
+import com.sportsbook.admin.audit.AuditLogRepository;
+import java.time.Instant;
+import java.util.List;
+import org.springframework.data.domain.Page;
+import org.springframework.data.domain.PageRequest;
+import org.springframework.data.domain.Sort;
+import org.springframework.format.annotation.DateTimeFormat;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
+import org.springframework.web.bind.annotation.RestController;
+import org.springframework.web.server.ResponseStatusException;
+
+@RestController
+@RequestMapping("/admin/v1/audit-logs")
+public class AuditLogController {
+
+  private static final int DEFAULT_PAGE_SIZE = 20;
+  private static final int MAX_PAGE_SIZE = 200;
+
+  private final AuditLogRepository repository;
+
+  public AuditLogController(AuditLogRepository repository) {
+    this.repository = repository;
+  }
+
+  @GetMapping
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public OffsetPage<AuditLogView> search(
+      @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
+          Instant from,
+      @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
+          Instant to,
+      @RequestParam(required = false) String actor,
+      @RequestParam(defaultValue = "0") int page,
+      @RequestParam(defaultValue = "20") int size) {
+    Instant lower = from == null ? Instant.EPOCH : from;
+    Instant upper = to == null ? Instant.now() : to;
+    if (!lower.isBefore(upper)) {
+      throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "from must be before to");
+    }
+    int normalizedPage = Math.max(0, page);
+    int normalizedSize = Math.min(size < 1 ? DEFAULT_PAGE_SIZE : size, MAX_PAGE_SIZE);
+    Sort newestFirst =
+        Sort.by(Sort.Direction.DESC, "startedAt")
+            .and(Sort.by(Sort.Direction.DESC, "actionId"));
+    Page<AuditLogEntity> result =
+        repository.search(
+            lower,
+            upper,
+            normalizeActor(actor),
+            PageRequest.of(normalizedPage, normalizedSize, newestFirst));
+    List<AuditLogView> items = result.stream().map(AuditLogView::from).toList();
+    return OffsetPage.from(result, items);
+  }
+
+  private static String normalizeActor(String actor) {
+    return actor == null || actor.isBlank() ? null : actor.trim();
+  }
+}


## `test(audit): page filtered actions newest first`

diff --git a/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java b/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java
index 8899275..cce0d22 100644
--- a/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java
+++ b/src/test/java/com/sportsbook/admin/audit/AuditLogRepositoryTest.java
@@ -10,6 +10,8 @@ import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
 import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
+import org.springframework.data.domain.PageRequest;
+import org.springframework.data.domain.Sort;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 import org.testcontainers.containers.PostgreSQLContainer;
@@ -60,4 +62,51 @@ class AuditLogRepositoryTest {
     assertThat(found.getStartedAt()).isEqualTo(startedAt);
     assertThat(found.getCompletedAt()).isNull();
   }
+
+  @Test
+  void filtersByActorAndTimeBeforePagingNewestFirst() {
+    Instant origin = Instant.parse("2026-08-23T01:00:00Z");
+    repository.saveAllAndFlush(
+        java.util.List.of(
+            terminal(21, "operator-1", origin.plusSeconds(60)),
+            terminal(22, "operator-2", origin.plusSeconds(120)),
+            terminal(23, "operator-1", origin.plusSeconds(180)),
+            terminal(24, "operator-1", origin.plusSeconds(240))));
+    entities.clear();
+    PageRequest newestFirst = PageRequest.of(0, 1, Sort.by(Sort.Direction.DESC, "startedAt"));
+
+    var first = repository.search(origin, origin.plusSeconds(240), "operator-1", newestFirst);
+    var second =
+        repository.search(origin, origin.plusSeconds(240), "operator-1", newestFirst.next());
+    var literalInjection =
+        repository.search(origin, origin.plusSeconds(240), "operator-1' OR '1'='1", newestFirst);
+
+    assertThat(first.getTotalElements()).isEqualTo(2);
+    assertThat(first.getContent())
+        .extracting(AuditLogEntity::getActionId)
+        .containsExactly(actionId(23));
+    assertThat(second.getContent())
+        .extracting(AuditLogEntity::getActionId)
+        .containsExactly(actionId(21));
+    assertThat(literalInjection).isEmpty();
+  }
+
+  private static AuditLogEntity terminal(int suffix, String actorId, Instant startedAt) {
+    return new AuditLogEntity(
+        actionId(suffix),
+        actorId,
+        AdminRole.ADMIN,
+        "MARKET_CLOSE",
+        "market-1",
+        AuditOutcome.SUCCESS,
+        202,
+        "operator request",
+        "trace-1",
+        startedAt,
+        startedAt.plusSeconds(1));
+  }
+
+  private static UUID actionId(int suffix) {
+    return UUID.fromString("018f0000-0000-7000-8000-0000000000" + suffix);
+  }
 }


## `test(audit): bound search requests by contract`

diff --git a/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java b/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java
new file mode 100644
index 0000000..94d2f2a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/AuditLogControllerTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.isNull;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.admin.audit.AuditLogRepository;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.data.domain.Page;
+import org.springframework.data.domain.Pageable;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.server.ResponseStatusException;
+
+class AuditLogControllerTest {
+
+  @Test
+  void boundsPagingAndTreatsBlankActorAsAbsent() {
+    AuditLogRepository repository = mock(AuditLogRepository.class);
+    Instant from = Instant.parse("2026-08-22T00:00:00Z");
+    Instant to = Instant.parse("2026-08-23T00:00:00Z");
+    when(repository.search(any(), any(), isNull(), any())).thenReturn(Page.empty());
+    var controller = new AuditLogController(repository);
+
+    OffsetPage<AuditLogView> result = controller.search(from, to, "  ", -4, 500);
+
+    ArgumentCaptor<Pageable> page = ArgumentCaptor.forClass(Pageable.class);
+    verify(repository).search(org.mockito.ArgumentMatchers.eq(from),
+        org.mockito.ArgumentMatchers.eq(to), isNull(), page.capture());
+    assertThat(page.getValue().getPageNumber()).isZero();
+    assertThat(page.getValue().getPageSize()).isEqualTo(200);
+    assertThat(page.getValue().getSort().getOrderFor("startedAt").isDescending()).isTrue();
+    assertThat(result.items()).isEmpty();
+  }
+
+  @Test
+  void rejectsAnEmptyTimeWindowBeforeQuerying() {
+    AuditLogRepository repository = mock(AuditLogRepository.class);
+    var controller = new AuditLogController(repository);
+    Instant boundary = Instant.parse("2026-08-23T00:00:00Z");
+
+    assertThatThrownBy(() -> controller.search(boundary, boundary, null, 0, 20))
+        .isInstanceOfSatisfying(
+            ResponseStatusException.class,
+            failure -> assertThat(failure.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST));
+
+    verify(repository, never()).search(any(), any(), any(), any());
+  }
+
+  @Test
+  void permitsEveryOperatorRole() throws NoSuchMethodException {
+    PreAuthorize guard =
+        AuditLogController.class
+            .getMethod(
+                "search", Instant.class, Instant.class, String.class, int.class, int.class)
+            .getAnnotation(PreAuthorize.class);
+
+    assertThat(guard.value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+  }
+}


