## `feat(credit): expose authorized credit endpoint`

diff --git a/src/main/java/com/sportsbook/wallet/web/CreditController.java b/src/main/java/com/sportsbook/wallet/web/CreditController.java
new file mode 100644
index 0000000..adffffc
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/CreditController.java
@@ -0,0 +1,38 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.CreditCommand;
+import com.sportsbook.wallet.web.dto.CreditRequest;
+import com.sportsbook.wallet.web.dto.WalletOperationResponse;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import org.springframework.security.core.annotation.AuthenticationPrincipal;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes caller-sensitive wallet credits. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions/credit")
+public class CreditController {
+  private final WalletService wallet;
+
+  public CreditController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping
+  WalletOperationResponse credit(
+      @AuthenticationPrincipal WalletCaller caller,
+      @Valid @RequestBody CreditRequest body,
+      HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    CreditCommand command =
+        new CreditCommand(body.userId(), body.amount(), body.source(), body.reason(), key);
+    return WalletOperationResponse.from(wallet.credit(caller, command));
+  }
+}


## `feat(settlement): expose forfeit endpoint`

diff --git a/src/main/java/com/sportsbook/wallet/web/ForfeitController.java b/src/main/java/com/sportsbook/wallet/web/ForfeitController.java
new file mode 100644
index 0000000..c5b4939
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/ForfeitController.java
@@ -0,0 +1,33 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.service.WalletService;
+import com.sportsbook.wallet.service.command.ForfeitCommand;
+import com.sportsbook.wallet.web.dto.TransactionRequest;
+import com.sportsbook.wallet.web.dto.WalletOperationResponse;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.util.Objects;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes settlement-owned forfeiture of losing locked stakes. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions")
+public class ForfeitController {
+  private final WalletService wallet;
+
+  public ForfeitController(WalletService wallet) {
+    this.wallet = Objects.requireNonNull(wallet, "wallet");
+  }
+
+  @PostMapping("/forfeit")
+  WalletOperationResponse forfeit(
+      @Valid @RequestBody TransactionRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    return WalletOperationResponse.from(
+        wallet.forfeit(new ForfeitCommand(body.userId(), body.amount(), key)));
+  }
+}


## `feat(settlement): expose adjustment endpoints`

diff --git a/src/main/java/com/sportsbook/wallet/web/AdjustmentController.java b/src/main/java/com/sportsbook/wallet/web/AdjustmentController.java
new file mode 100644
index 0000000..7ef87f8
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/web/AdjustmentController.java
@@ -0,0 +1,57 @@
+package com.sportsbook.wallet.web;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.error.WalletAdjustmentNotFoundException;
+import com.sportsbook.wallet.service.WalletAdjustmentService;
+import com.sportsbook.wallet.web.dto.AdjustmentProofResponse;
+import com.sportsbook.wallet.web.dto.AdjustmentRequest;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.validation.Valid;
+import java.net.URI;
+import java.util.Objects;
+import java.util.UUID;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Exposes settlement payout correction requests and durable proofs. */
+@RestController
+@RequestMapping("/internal/v1/wallet/transactions/adjustment")
+public class AdjustmentController {
+  private static final String PROOF_PATH = "/internal/v1/wallet/transactions/adjustment/";
+
+  private final WalletAdjustmentService adjustments;
+
+  public AdjustmentController(WalletAdjustmentService adjustments) {
+    this.adjustments = Objects.requireNonNull(adjustments, "adjustments");
+  }
+
+  @PostMapping
+  ResponseEntity<AdjustmentProofResponse> adjust(
+      @Valid @RequestBody AdjustmentRequest body, HttpServletRequest request) {
+    IdempotencyKey key = WalletRequestHeaders.requireIdempotencyKey(request);
+    WalletAdjustment proof = adjustments.adjust(body.toCommand(key));
+    AdjustmentProofResponse response = AdjustmentProofResponse.from(proof);
+    return switch (proof.status()) {
+      case APPLIED -> ResponseEntity.ok(response);
+      case BLOCKED ->
+          ResponseEntity.accepted()
+              .location(URI.create(PROOF_PATH + proof.revisionId()))
+              .body(response);
+      case REJECTED -> throw new IllegalStateException("Adjustment rejection returned as a proof");
+    };
+  }
+
+  @GetMapping("/{revisionId}")
+  AdjustmentProofResponse proof(@PathVariable UUID revisionId) {
+    return adjustments
+        .findProof(revisionId)
+        .map(AdjustmentProofResponse::from)
+        .orElseThrow(() -> new WalletAdjustmentNotFoundException(revisionId));
+  }
+}


## `test(settlement): map adjustment outcomes`

diff --git a/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerPostTest.java b/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerPostTest.java
new file mode 100644
index 0000000..343c58a
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerPostTest.java
@@ -0,0 +1,97 @@
+package com.sportsbook.wallet.web;
+
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.service.WalletAdjustmentService;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+class AdjustmentControllerPostTest {
+  private static final UUID REVISION_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004b1");
+  private static final UUID BET_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004b2");
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004b3");
+  private static final UUID GROUP_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004b4");
+  private static final IdempotencyKey KEY = IdempotencyKey.of("settlement:revision:" + REVISION_ID);
+  private static final Instant NOW = Instant.parse("2026-08-21T20:00:00Z");
+
+  private final WalletAdjustmentService adjustments = mock(WalletAdjustmentService.class);
+  private final MockMvc mvc =
+      MockMvcBuilders.standaloneSetup(new AdjustmentController(adjustments))
+          .setControllerAdvice(new WalletExceptionHandler())
+          .build();
+
+  @Test
+  void appliesPositiveRevisionsWithNoProofLocation() throws Exception {
+    AdjustmentCommand command = command(700L, 1_000L);
+    when(adjustments.adjust(command)).thenReturn(WalletAdjustment.applied(command, GROUP_ID, NOW));
+
+    mvc.perform(request(700L, 1_000L))
+        .andExpect(status().isOk())
+        .andExpect(header().doesNotExist(HttpHeaders.LOCATION))
+        .andExpect(jsonPath("$.status").value("APPLIED"))
+        .andExpect(jsonPath("$.operationGroupId").value(GROUP_ID.toString()))
+        .andExpect(jsonPath("$.deltaAmount").value(300));
+    verify(adjustments).adjust(command);
+  }
+
+  @Test
+  void queuesNegativeRevisionsAtTheirDurableProofLocation() throws Exception {
+    AdjustmentCommand command = command(1_000L, 700L);
+    when(adjustments.adjust(command)).thenReturn(WalletAdjustment.blocked(command, 4L, NOW));
+
+    mvc.perform(request(1_000L, 700L))
+        .andExpect(status().isAccepted())
+        .andExpect(
+            header()
+                .string(
+                    HttpHeaders.LOCATION,
+                    "/internal/v1/wallet/transactions/adjustment/" + REVISION_ID))
+        .andExpect(jsonPath("$.status").value("BLOCKED"))
+        .andExpect(jsonPath("$.queueSequence").value(4))
+        .andExpect(jsonPath("$.operationGroupId").isEmpty())
+        .andExpect(jsonPath("$.deltaAmount").value(-300));
+    verify(adjustments).adjust(command);
+  }
+
+  private org.springframework.test.web.servlet.request.MockHttpServletRequestBuilder request(
+      long previous, long next) {
+    return post("/internal/v1/wallet/transactions/adjustment")
+        .header(WalletRequestHeaders.IDEMPOTENCY_KEY, KEY.value())
+        .contentType(MediaType.APPLICATION_JSON)
+        .content(body(previous, next));
+  }
+
+  private AdjustmentCommand command(long previous, long next) {
+    return new AdjustmentCommand(
+        REVISION_ID, BET_ID, 2L, USER_ID, Money.krw(previous), Money.krw(next), KEY);
+  }
+
+  private String body(long previous, long next) {
+    return "{\"revisionId\":\""
+        + REVISION_ID
+        + "\",\"betId\":\""
+        + BET_ID
+        + "\",\"revisionNumber\":2,\"userId\":\""
+        + USER_ID
+        + "\",\"previousPayout\":{\"amount\":"
+        + previous
+        + ",\"currency\":\"KRW\"},\"newPayout\":{\"amount\":"
+        + next
+        + ",\"currency\":\"KRW\"}}";
+  }
+}


## `test(settlement): expose durable adjustment proofs`

diff --git a/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerLookupTest.java b/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerLookupTest.java
new file mode 100644
index 0000000..c163e10
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/AdjustmentControllerLookupTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.service.WalletAdjustmentService;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.MvcResult;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+class AdjustmentControllerLookupTest {
+  private static final UUID APPLIED_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c1");
+  private static final UUID BLOCKED_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c2");
+  private static final UUID REJECTED_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c3");
+  private static final UUID MISSING_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c4");
+  private static final UUID BET_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c5");
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c6");
+  private static final UUID GROUP_ID = UUID.fromString("019b76da-a000-7000-8000-0000000004c7");
+  private static final Instant NOW = Instant.parse("2026-08-21T21:00:00Z");
+
+  private final WalletAdjustmentService adjustments = mock(WalletAdjustmentService.class);
+  private final MockMvc mvc =
+      MockMvcBuilders.standaloneSetup(new AdjustmentController(adjustments))
+          .setControllerAdvice(new WalletExceptionHandler())
+          .build();
+
+  @Test
+  void returnsEveryDurableProofStatus() throws Exception {
+    AdjustmentCommand applied = command(APPLIED_ID, 700L, 1_000L);
+    AdjustmentCommand blocked = command(BLOCKED_ID, 1_000L, 700L);
+    AdjustmentCommand rejected = command(REJECTED_ID, 1_000L, 700L);
+    when(adjustments.findProof(APPLIED_ID))
+        .thenReturn(Optional.of(WalletAdjustment.applied(applied, GROUP_ID, NOW)));
+    when(adjustments.findProof(BLOCKED_ID))
+        .thenReturn(Optional.of(WalletAdjustment.blocked(blocked, 4L, NOW)));
+    when(adjustments.findProof(REJECTED_ID))
+        .thenReturn(Optional.of(WalletAdjustment.rejected(rejected, NOW)));
+
+    assertStatus(APPLIED_ID, "APPLIED");
+    assertStatus(BLOCKED_ID, "BLOCKED");
+    assertStatus(REJECTED_ID, "REJECTED");
+  }
+
+  @Test
+  void mapsMissingProofsWithoutReflectingTheirIdentity() throws Exception {
+    when(adjustments.findProof(MISSING_ID)).thenReturn(Optional.empty());
+
+    MvcResult result =
+        mvc.perform(get(path(MISSING_ID)))
+            .andExpect(status().isNotFound())
+            .andExpect(jsonPath("$.errorCode").value("WALLET_ADJUSTMENT_NOT_FOUND"))
+            .andExpect(jsonPath("$.detail").value("The requested wallet adjustment does not exist"))
+            .andReturn();
+
+    assertThat(result.getResponse().getContentAsString())
+        .contains("\"instance\":\"" + path(MISSING_ID) + "\"")
+        .doesNotContain("No wallet adjustment exists");
+    verify(adjustments).findProof(MISSING_ID);
+  }
+
+  private void assertStatus(UUID revisionId, String expected) throws Exception {
+    mvc.perform(get(path(revisionId)))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.revisionId").value(revisionId.toString()))
+        .andExpect(jsonPath("$.status").value(expected));
+  }
+
+  private AdjustmentCommand command(UUID revisionId, long previous, long next) {
+    return new AdjustmentCommand(
+        revisionId,
+        BET_ID,
+        2L,
+        USER_ID,
+        Money.krw(previous),
+        Money.krw(next),
+        IdempotencyKey.of("settlement:revision:" + revisionId));
+  }
+
+  private String path(UUID revisionId) {
+    return "/internal/v1/wallet/transactions/adjustment/" + revisionId;
+  }
+}
