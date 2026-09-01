# 신뢰된 액터 매핑과 안정된 베팅 HTTP 계약

## `feat(api): define placement wire models`

diff --git a/src/main/java/com/sportsbook/betting/api/BetResponse.java b/src/main/java/com/sportsbook/betting/api/BetResponse.java
new file mode 100644
index 0000000..db6bb21
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/BetResponse.java
@@ -0,0 +1,58 @@
+package com.sportsbook.betting.api;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record BetResponse(
+    UUID betId,
+    String betReference,
+    UUID userId,
+    SlipTypeView slipType,
+    String status,
+    Money stake,
+    Money maxPayout,
+    List<SelectionView> selections,
+    String rejectionReason,
+    Instant createdAt) {
+
+  public record SlipTypeView(String type, Integer minWins, Integer totalSelections) {}
+
+  public record SelectionView(
+      UUID eventId, UUID marketId, UUID selectionId, String oddsAtSubmission) {}
+
+  public static BetResponse from(Bet bet) {
+    BetSlipType type = bet.slipType();
+    SlipTypeView slip =
+        type instanceof BetSlipType.System system
+            ? new SlipTypeView("SYSTEM", system.minWins(), system.totalSelections())
+            : new SlipTypeView(
+                type instanceof BetSlipType.Single ? "SINGLE" : "MULTIPLE", null, null);
+    List<SelectionView> selections =
+        bet.legs().stream()
+            .map(
+                leg ->
+                    new SelectionView(
+                        leg.eventId(),
+                        leg.marketId(),
+                        leg.selectionId(),
+                        leg.oddsAtSubmission().decimal().toPlainString()))
+            .toList();
+    return new BetResponse(
+        bet.betId(),
+        bet.betReference(),
+        bet.userId(),
+        slip,
+        bet.status().name(),
+        bet.stake(),
+        bet.maxPayout(),
+        selections,
+        bet.rejectionReason(),
+        bet.createdAt());
+  }
+}
diff --git a/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java b/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java
new file mode 100644
index 0000000..9de9998
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.betting.api;
+
+import com.sportsbook.protocol.value.Money;
+import jakarta.validation.Valid;
+import jakarta.validation.constraints.NotBlank;
+import jakarta.validation.constraints.NotEmpty;
+import jakarta.validation.constraints.NotNull;
+import java.math.BigDecimal;
+import java.util.List;
+import java.util.UUID;
+
+public record PlaceBetRequest(
+    @NotNull @Valid SlipTypeRequest slipType,
+    @NotEmpty @Valid List<SelectionRequest> selections,
+    @NotNull Money stake) {
+
+  public record SlipTypeRequest(@NotBlank String type, Integer minWins, Integer totalSelections) {}
+
+  public record SelectionRequest(
+      @NotNull UUID eventId,
+      @NotNull UUID marketId,
+      @NotNull UUID selectionId,
+      @NotNull BigDecimal odds) {}
+}


## `test(api): verify trusted placement wire shape`

diff --git a/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java b/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java
new file mode 100644
index 0000000..f86e822
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.betting.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.Arrays;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetWireModelTest {
+
+  @Test
+  void neverAcceptsActorIdentityFromThePlacementBody() {
+    assertThat(
+            Arrays.stream(PlaceBetRequest.class.getRecordComponents())
+                .map(java.lang.reflect.RecordComponent::getName))
+        .doesNotContain("userId");
+  }
+
+  @Test
+  void rendersSystemShapeAndOriginalUnitStake() {
+    BetSlipType type = new BetSlipType.System(2, 3);
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                UUID.randomUUID(),
+                UUID.randomUUID(),
+                "B-2026-08-22-00000000",
+                type,
+                Money.krw(1_000),
+                Money.krw(10_000),
+                IdempotencyKey.of("request-1"),
+                "a".repeat(64),
+                Instant.EPOCH),
+            List.of(leg(), leg(), leg()));
+    bet.recordRiskReservation(Instant.EPOCH.plusSeconds(60), "b".repeat(64), true, Instant.EPOCH);
+    bet.confirmWallet(UUID.randomUUID(), Instant.EPOCH);
+    bet.commitRisk(Instant.EPOCH);
+    bet.accept(Instant.EPOCH);
+
+    BetResponse response = BetResponse.from(bet);
+
+    assertThat(response.slipType()).isEqualTo(new BetResponse.SlipTypeView("SYSTEM", 2, 3));
+    assertThat(response.stake()).isEqualTo(Money.krw(1_000));
+    assertThat(response.selections()).hasSize(3);
+  }
+
+  private static BetLeg leg() {
+    return BetLeg.create(
+        UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"));
+  }
+}


## `feat(api): map trusted placement commands`

diff --git a/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java b/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java
index 9de9998..cff21da 100644
--- a/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java
+++ b/src/main/java/com/sportsbook/betting/api/PlaceBetRequest.java
@@ -1,6 +1,11 @@
 package com.sportsbook.betting.api;
 
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.placement.PlaceBetCommand;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
 import jakarta.validation.Valid;
 import jakarta.validation.constraints.NotBlank;
 import jakarta.validation.constraints.NotEmpty;
@@ -14,6 +19,44 @@ public record PlaceBetRequest(
     @NotEmpty @Valid List<SelectionRequest> selections,
     @NotNull Money stake) {
 
+  PlaceBetCommand toCommand(UUID actorId, IdempotencyKey key) {
+    BetSlipType type =
+        switch (slipType.type()) {
+          case "SINGLE" -> {
+            requireNoSystemShape();
+            yield new BetSlipType.Single();
+          }
+          case "MULTIPLE" -> {
+            requireNoSystemShape();
+            yield new BetSlipType.Multiple();
+          }
+          case "SYSTEM" -> {
+            if (slipType.minWins() == null || slipType.totalSelections() == null) {
+              throw new ValidationFailedException("SYSTEM shape is incomplete");
+            }
+            yield new BetSlipType.System(slipType.minWins(), slipType.totalSelections());
+          }
+          default -> throw new ValidationFailedException("Unknown slip type");
+        };
+    List<PlaceBetCommand.SelectionInput> inputs =
+        selections.stream()
+            .map(
+                item ->
+                    new PlaceBetCommand.SelectionInput(
+                        item.eventId(),
+                        item.marketId(),
+                        item.selectionId(),
+                        Odds.ofDecimal(item.odds())))
+            .toList();
+    return new PlaceBetCommand(actorId, type, inputs, stake, key);
+  }
+
+  private void requireNoSystemShape() {
+    if (slipType.minWins() != null || slipType.totalSelections() != null) {
+      throw new ValidationFailedException("Non-SYSTEM shape must omit system fields");
+    }
+  }
+
   public record SlipTypeRequest(@NotBlank String type, Integer minWins, Integer totalSelections) {}
 
   public record SelectionRequest(


## `test(api): verify trusted command mapping`

diff --git a/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java b/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java
index f86e822..07f1c48 100644
--- a/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java
+++ b/src/test/java/com/sportsbook/betting/api/BetWireModelTest.java
@@ -1,10 +1,12 @@
 package com.sportsbook.betting.api;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetDraft;
 import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.error.ValidationFailedException;
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
@@ -25,6 +27,46 @@ class BetWireModelTest {
         .doesNotContain("userId");
   }
 
+  @Test
+  void mapsOnlyTheTrustedActorIntoThePlacementCommand() {
+    UUID actorId = UUID.randomUUID();
+    PlaceBetRequest request =
+        new PlaceBetRequest(
+            new PlaceBetRequest.SlipTypeRequest("SINGLE", null, null),
+            List.of(
+                new PlaceBetRequest.SelectionRequest(
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    new java.math.BigDecimal("2.00"))),
+            Money.krw(1_000));
+
+    var command = request.toCommand(actorId, IdempotencyKey.of("request-2"));
+
+    assertThat(command.userId()).isEqualTo(actorId);
+    assertThat(command.slipType()).isEqualTo(new BetSlipType.Single());
+    assertThat(command.idempotencyKey()).isEqualTo(IdempotencyKey.of("request-2"));
+  }
+
+  @Test
+  void rejectsSystemFieldsOnNonSystemSlips() {
+    PlaceBetRequest request =
+        new PlaceBetRequest(
+            new PlaceBetRequest.SlipTypeRequest("SINGLE", 1, 1),
+            List.of(
+                new PlaceBetRequest.SelectionRequest(
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    new java.math.BigDecimal("2.00"))),
+            Money.krw(1_000));
+
+    assertThatThrownBy(
+            () -> request.toCommand(UUID.randomUUID(), IdempotencyKey.of("request-shape")))
+        .isInstanceOf(ValidationFailedException.class)
+        .hasMessageContaining("omit system fields");
+  }
+
   @Test
   void rendersSystemShapeAndOriginalUnitStake() {
     BetSlipType type = new BetSlipType.System(2, 3);


## `feat(api): expose trusted placement and history routes`

diff --git a/src/main/java/com/sportsbook/betting/api/BetController.java b/src/main/java/com/sportsbook/betting/api/BetController.java
new file mode 100644
index 0000000..dcee3bd
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/BetController.java
@@ -0,0 +1,81 @@
+package com.sportsbook.betting.api;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.placement.BetPlacementService;
+import com.sportsbook.betting.placement.BetQueryService;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.net.URI;
+import java.util.Collections;
+import java.util.List;
+import java.util.UUID;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/internal/v1/bets")
+public class BetController {
+
+  private final BetPlacementService placement;
+  private final BetQueryService queries;
+
+  public BetController(BetPlacementService placement, BetQueryService queries) {
+    this.placement = placement;
+    this.queries = queries;
+  }
+
+  @PostMapping
+  ResponseEntity<BetResponse> place(
+      HttpServletRequest request, @Valid @RequestBody PlaceBetRequest body) {
+    UUID actor = actor(request);
+    Bet bet =
+        placement.place(
+            body.toCommand(actor, IdempotencyKey.of(single(request, "Idempotency-Key"))));
+    HttpStatus status =
+        bet.status() == BetStatus.PENDING ? HttpStatus.ACCEPTED : HttpStatus.CREATED;
+    URI location = URI.create("/api/v1/bets/" + bet.betId());
+    return ResponseEntity.status(status).location(location).body(BetResponse.from(bet));
+  }
+
+  @GetMapping
+  CursorPage<BetResponse> page(
+      HttpServletRequest request,
+      @RequestParam(required = false) UUID cursor,
+      @RequestParam(required = false) Integer limit) {
+    CursorPage<Bet> page = queries.page(actor(request), cursor, limit);
+    return new CursorPage<>(
+        page.items().stream().map(BetResponse::from).toList(), page.nextCursor(), page.hasMore());
+  }
+
+  @GetMapping("/{betId}")
+  BetResponse byId(HttpServletRequest request, @PathVariable UUID betId) {
+    return BetResponse.from(queries.byId(actor(request), betId));
+  }
+
+  private static UUID actor(HttpServletRequest request) {
+    String raw = single(request, "X-User-Id");
+    UUID actor = UUID.fromString(raw);
+    if (!actor.toString().equals(raw)) {
+      throw new ValidationFailedException("X-User-Id must be a canonical lowercase UUID");
+    }
+    return actor;
+  }
+
+  private static String single(HttpServletRequest request, String name) {
+    List<String> values = Collections.list(request.getHeaders(name));
+    if (values.size() != 1 || values.get(0).isBlank()) {
+      throw new ValidationFailedException("Exactly one " + name + " is required");
+    }
+    return values.get(0);
+  }
+}


## `test(api): verify trusted controller boundary`

diff --git a/src/test/java/com/sportsbook/betting/api/BetControllerTest.java b/src/test/java/com/sportsbook/betting/api/BetControllerTest.java
new file mode 100644
index 0000000..d95b873
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/api/BetControllerTest.java
@@ -0,0 +1,97 @@
+package com.sportsbook.betting.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.placement.BetPlacementService;
+import com.sportsbook.betting.placement.BetQueryService;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.value.Money;
+import java.math.BigDecimal;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class BetControllerTest {
+
+  @Test
+  void returnsAcceptedRatherThanSuccessForDurablePendingWork() {
+    BetPlacementService placement = mock(BetPlacementService.class);
+    Bet bet = pendingBet();
+    when(placement.place(any())).thenReturn(bet);
+    BetController controller = new BetController(placement, mock(BetQueryService.class));
+    UUID actorId = bet.userId();
+    MockHttpServletRequest request = request(actorId);
+
+    var response = controller.place(request, body());
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(response.getBody().status()).isEqualTo("PENDING");
+    assertThat(response.getHeaders().getLocation()).hasToString("/api/v1/bets/" + bet.betId());
+  }
+
+  @Test
+  void returnsCreatedWithTheStablePublicBetLocation() {
+    BetPlacementService placement = mock(BetPlacementService.class);
+    Bet bet = pendingBet();
+    when(bet.status()).thenReturn(BetStatus.ACCEPTED);
+    when(placement.place(any())).thenReturn(bet);
+    BetController controller = new BetController(placement, mock(BetQueryService.class));
+
+    var response = controller.place(request(bet.userId()), body());
+
+    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
+    assertThat(response.getHeaders().getLocation()).hasToString("/api/v1/bets/" + bet.betId());
+  }
+
+  @Test
+  void rejectsAmbiguousActorHeaders() {
+    MockHttpServletRequest request = request(UUID.randomUUID());
+    request.addHeader("X-User-Id", UUID.randomUUID().toString());
+
+    assertThatThrownBy(
+            () ->
+                new BetController(mock(BetPlacementService.class), mock(BetQueryService.class))
+                    .byId(request, UUID.randomUUID()))
+        .isInstanceOf(ValidationFailedException.class);
+  }
+
+  private static MockHttpServletRequest request(UUID actorId) {
+    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/internal/v1/bets");
+    request.addHeader("X-User-Id", actorId.toString());
+    request.addHeader("Idempotency-Key", "request-1");
+    return request;
+  }
+
+  private static PlaceBetRequest body() {
+    return new PlaceBetRequest(
+        new PlaceBetRequest.SlipTypeRequest("SINGLE", null, null),
+        List.of(
+            new PlaceBetRequest.SelectionRequest(
+                UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), new BigDecimal("2.00"))),
+        Money.krw(1_000));
+  }
+
+  private static Bet pendingBet() {
+    Bet bet = mock(Bet.class);
+    when(bet.betId()).thenReturn(UUID.randomUUID());
+    when(bet.betReference()).thenReturn("B-2026-08-22-00000000");
+    when(bet.userId()).thenReturn(UUID.randomUUID());
+    when(bet.slipType()).thenReturn(new BetSlipType.Single());
+    when(bet.status()).thenReturn(BetStatus.PENDING);
+    when(bet.stake()).thenReturn(Money.krw(1_000));
+    when(bet.maxPayout()).thenReturn(Money.krw(2_000));
+    when(bet.legs()).thenReturn(List.of());
+    when(bet.createdAt()).thenReturn(Instant.EPOCH);
+    return bet;
+  }
+}


## `feat(api): render stable problem responses`

diff --git a/src/main/java/com/sportsbook/betting/api/BetExceptionHandler.java b/src/main/java/com/sportsbook/betting/api/BetExceptionHandler.java
new file mode 100644
index 0000000..d76947e
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/api/BetExceptionHandler.java
@@ -0,0 +1,49 @@
+package com.sportsbook.betting.api;
+
+import com.sportsbook.betting.error.BetNotFoundException;
+import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.error.ProblemDetail;
+import jakarta.servlet.http.HttpServletRequest;
+import java.net.URI;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.MethodArgumentNotValidException;
+import org.springframework.web.bind.annotation.ExceptionHandler;
+import org.springframework.web.bind.annotation.RestControllerAdvice;
+
+@RestControllerAdvice(assignableTypes = BetController.class)
+public class BetExceptionHandler {
+
+  @ExceptionHandler(BetPlacementException.class)
+  ResponseEntity<ProblemDetail> placement(
+      BetPlacementException failure, HttpServletRequest request) {
+    ErrorCode code = failure.errorCode();
+    return ResponseEntity.status(code.httpStatus())
+        .body(code.toProblemDetail(failure.getMessage(), instance(request), null));
+  }
+
+  @ExceptionHandler(BetNotFoundException.class)
+  ResponseEntity<ProblemDetail> missing(BetNotFoundException failure, HttpServletRequest request) {
+    ProblemDetail body =
+        new ProblemDetail(
+            URI.create("https://sportsbook/errors/bet-not-found"),
+            "Bet not found",
+            404,
+            "BET_NOT_FOUND",
+            failure.getMessage(),
+            instance(request),
+            null);
+    return ResponseEntity.status(404).body(body);
+  }
+
+  @ExceptionHandler({MethodArgumentNotValidException.class, IllegalArgumentException.class})
+  ResponseEntity<ProblemDetail> invalid(Exception failure, HttpServletRequest request) {
+    ErrorCode code = ErrorCode.VALIDATION_FAILED;
+    return ResponseEntity.badRequest()
+        .body(code.toProblemDetail("Request validation failed", instance(request), null));
+  }
+
+  private static URI instance(HttpServletRequest request) {
+    return URI.create(request.getRequestURI());
+  }
+}


