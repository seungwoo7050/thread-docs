## `feat(settlement): fetch typed admin evidence`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementClient.java b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
new file mode 100644
index 0000000..b481a03
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
@@ -0,0 +1,62 @@
+package com.sportsbook.admin.client;
+
+import java.util.UUID;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.http.HttpStatus;
+import org.springframework.stereotype.Component;
+import org.springframework.web.client.RestClient;
+
+@Component
+public class SettlementClient {
+
+  private final RestClient http;
+  private final DownstreamFailureMapper failures = new DownstreamFailureMapper();
+
+  public SettlementClient(@Qualifier("settlementRestClient") RestClient http) {
+    this.http = http;
+  }
+
+  public SettlementCandidateView getCandidate(UUID candidateId) {
+    var response =
+        failures.execute(
+            () ->
+                http.get()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "admin", "result-candidates")
+                                .pathSegment(candidateId.toString())
+                                .build())
+                    .retrieve()
+                    .toEntity(SettlementCandidateView.class));
+    SettlementCandidateView body =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Settlement candidate GET must return HTTP 200 with a body");
+    return SettlementCandidateView.verify(candidateId, body);
+  }
+
+  public SettlementRevisionView getRevision(UUID revisionId) {
+    var response =
+        failures.execute(
+            () ->
+                http.get()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "admin", "revisions")
+                                .pathSegment(revisionId.toString())
+                                .build())
+                    .retrieve()
+                    .toEntity(SettlementRevisionView.class));
+    SettlementRevisionView body =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Settlement revision GET must return HTTP 200 with a body");
+    return SettlementRevisionProof.verify(revisionId, body);
+  }
+}


## `test(settlement): fetch exact revision evidence`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionQueryTest.java b/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionQueryTest.java
new file mode 100644
index 0000000..3b24bc4
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionQueryTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.http.HttpMethod.GET;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.headerDoesNotExist;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class SettlementClientRevisionQueryTest {
+
+  @Test
+  void fetchesAllRevisionEvidenceWithoutAnIdempotencyHeader() {
+    UUID revision = UUID.fromString("018f0000-0000-7000-8000-000000000158");
+    RestClient.Builder builder =
+        RestClient.builder()
+            .baseUrl("http://settlement.test")
+            .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API)
+            .defaultHeader(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT);
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://settlement.test/internal/admin/revisions/" + revision))
+        .andExpect(method(GET))
+        .andExpect(header(DownstreamHeaders.SERVICE_NAME, "admin-api"))
+        .andExpect(header(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT))
+        .andExpect(headerDoesNotExist("Idempotency-Key"))
+        .andRespond(withSuccess(response(revision), MediaType.APPLICATION_JSON));
+
+    SettlementRevisionView result = new SettlementClient(builder.build()).getRevision(revision);
+
+    assertThat(result.revisionId()).isEqualTo(revision);
+    assertThat(result.state()).isEqualTo(SettlementRevisionView.State.BLOCKED);
+    assertThat(result.walletStatus()).isEqualTo(SettlementRevisionView.WalletStatus.BLOCKED);
+    server.verify();
+  }
+
+  private static String response(UUID revision) {
+    return """
+        {"revisionId":"%s","betId":"018f0000-0000-7000-8000-000000000159",
+         "revisionNumber":2,"eventId":"018f0000-0000-7000-8000-000000000160",
+         "sourceCandidateId":"018f0000-0000-7000-8000-000000000161",
+         "state":"BLOCKED","attemptCount":3,"nextRetryAt":"2026-08-22T01:03:00Z",
+         "lastErrorCode":"WALLET_BUSY","leaseUntil":null,"walletStatus":"BLOCKED",
+         "walletQueueSequence":17,
+         "walletOperationGroupId":null,
+         "walletQueuedAt":"2026-08-22T01:01:00Z","walletAppliedAt":null,
+         "walletNextAttemptAt":"2026-08-22T01:03:00Z",
+         "createdAt":"2026-08-22T01:00:00Z","updatedAt":"2026-08-22T01:02:00Z",
+         "appliedAt":null}
+        """
+        .formatted(revision);
+  }
+}


## `feat(settlement): verify revision retry receipts`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementRetryReceipt.java b/src/main/java/com/sportsbook/admin/client/SettlementRetryReceipt.java
new file mode 100644
index 0000000..f5bf59f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementRetryReceipt.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.client;
+
+import java.time.Instant;
+import java.util.UUID;
+
+public record SettlementRetryReceipt(
+    UUID idempotencyKey,
+    Outcome outcome,
+    SettlementRevisionView.State revisionState,
+    Integer attemptCount,
+    Instant nextRetryAt) {
+
+  private static final int MAX_ATTEMPTS = 12;
+
+  public static SettlementRetryReceipt verify(UUID requestedKey, SettlementRetryReceipt receipt) {
+    if (receipt == null
+        || !requestedKey.equals(receipt.idempotencyKey())
+        || receipt.outcome() == null
+        || receipt.revisionState() == null
+        || receipt.attemptCount() == null
+        || receipt.attemptCount() < 0
+        || receipt.attemptCount() > MAX_ATTEMPTS
+        || invalidQueuedProof(receipt)) {
+      throw new DownstreamContractException("matching Settlement revision retry receipt");
+    }
+    return receipt;
+  }
+
+  private static boolean invalidQueuedProof(SettlementRetryReceipt receipt) {
+    return receipt.outcome() == Outcome.QUEUED
+        && (receipt.attemptCount() != 0
+            || receipt.nextRetryAt() == null
+            || (receipt.revisionState() != SettlementRevisionView.State.PENDING
+                && receipt.revisionState() != SettlementRevisionView.State.BLOCKED));
+  }
+
+  public enum Outcome {
+    QUEUED,
+    REPLAY
+  }
+}


## `test(settlement): validate revision retry receipts`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRetryReceiptTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRetryReceiptTest.java
new file mode 100644
index 0000000..6d6f190
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRetryReceiptTest.java
@@ -0,0 +1,97 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementRetryReceiptTest {
+
+  private static final UUID KEY = UUID.fromString("018f0000-0000-7000-8000-000000000167");
+  private static final Instant DUE = Instant.parse("2026-08-22T01:05:00Z");
+
+  @Test
+  void acceptsQueuedAndReplayReceipts() {
+    SettlementRetryReceipt pending =
+        receipt(
+            KEY,
+            SettlementRetryReceipt.Outcome.QUEUED,
+            SettlementRevisionView.State.PENDING,
+            0,
+            DUE);
+    SettlementRetryReceipt blocked =
+        receipt(
+            KEY,
+            SettlementRetryReceipt.Outcome.QUEUED,
+            SettlementRevisionView.State.BLOCKED,
+            0,
+            DUE);
+    SettlementRetryReceipt replay =
+        receipt(
+            KEY,
+            SettlementRetryReceipt.Outcome.REPLAY,
+            SettlementRevisionView.State.APPLIED,
+            3,
+            null);
+
+    assertThat(SettlementRetryReceipt.verify(KEY, pending)).isSameAs(pending);
+    assertThat(SettlementRetryReceipt.verify(KEY, blocked)).isSameAs(blocked);
+    assertThat(SettlementRetryReceipt.verify(KEY, replay)).isSameAs(replay);
+  }
+
+  @Test
+  void rejectsMismatchedUnsafeAndImpossibleQueuedProofs() {
+    UUID other = UUID.fromString("018f0000-0000-7000-8000-000000000168");
+    List<SettlementRetryReceipt> invalid =
+        List.of(
+            receipt(
+                other,
+                SettlementRetryReceipt.Outcome.QUEUED,
+                SettlementRevisionView.State.PENDING,
+                0,
+                DUE),
+            receipt(
+                KEY,
+                SettlementRetryReceipt.Outcome.QUEUED,
+                SettlementRevisionView.State.PENDING,
+                1,
+                DUE),
+            receipt(
+                KEY,
+                SettlementRetryReceipt.Outcome.QUEUED,
+                SettlementRevisionView.State.EXHAUSTED,
+                0,
+                DUE),
+            receipt(
+                KEY,
+                SettlementRetryReceipt.Outcome.QUEUED,
+                SettlementRevisionView.State.BLOCKED,
+                0,
+                null),
+            receipt(
+                KEY,
+                SettlementRetryReceipt.Outcome.REPLAY,
+                SettlementRevisionView.State.PENDING,
+                13,
+                null));
+
+    assertThatThrownBy(() -> SettlementRetryReceipt.verify(KEY, null))
+        .isInstanceOf(DownstreamContractException.class);
+    invalid.forEach(
+        receipt ->
+            assertThatThrownBy(() -> SettlementRetryReceipt.verify(KEY, receipt))
+                .isInstanceOf(DownstreamContractException.class));
+  }
+
+  private static SettlementRetryReceipt receipt(
+      UUID key,
+      SettlementRetryReceipt.Outcome outcome,
+      SettlementRevisionView.State state,
+      Integer attemptCount,
+      Instant nextRetryAt) {
+    return new SettlementRetryReceipt(key, outcome, state, attemptCount, nextRetryAt);
+  }
+}


## `feat(settlement): retry paused revisions`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementClient.java b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
index 3eb1404..403687d 100644
--- a/src/main/java/com/sportsbook/admin/client/SettlementClient.java
+++ b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
@@ -109,4 +109,27 @@ public class SettlementClient {
     return SettlementCandidateReceipt.verify(
         idempotencyKey, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, receipt);
   }
+
+  public SettlementRetryReceipt retryRevision(UUID revisionId, UUID idempotencyKey) {
+    var response =
+        failures.execute(
+            () ->
+                http.post()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "admin", "revisions")
+                                .pathSegment(revisionId.toString(), "retry")
+                                .build())
+                    .header("Idempotency-Key", idempotencyKey.toString())
+                    .retrieve()
+                    .toEntity(SettlementRetryReceipt.class));
+    SettlementRetryReceipt receipt =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.ACCEPTED,
+            ignored -> true,
+            "Settlement revision retry must return HTTP 202 with a receipt");
+    return SettlementRetryReceipt.verify(idempotencyKey, receipt);
+  }
 }


## `test(settlement): send exact revision retry`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionRetryTest.java b/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionRetryTest.java
new file mode 100644
index 0000000..ed34d32
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementClientRevisionRetryTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.http.HttpMethod.POST;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.content;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.header;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.headerDoesNotExist;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
+import static org.springframework.test.web.client.response.MockRestResponseCreators.withStatus;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.client.MockRestServiceServer;
+import org.springframework.web.client.RestClient;
+
+class SettlementClientRevisionRetryTest {
+
+  @Test
+  void postsAnEmptyRetryAndRequiresAnAcceptedReceipt() {
+    UUID revision = UUID.fromString("018f0000-0000-7000-8000-000000000169");
+    UUID key = UUID.fromString("018f0000-0000-7000-8000-000000000170");
+    RestClient.Builder builder = settlementBuilder();
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://settlement.test/internal/admin/revisions/" + revision + "/retry"))
+        .andExpect(method(POST))
+        .andExpect(header(DownstreamHeaders.SERVICE_NAME, "admin-api"))
+        .andExpect(header(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT))
+        .andExpect(header("Idempotency-Key", key.toString()))
+        .andExpect(headerDoesNotExist(DownstreamHeaders.INTERNAL_SERVICE))
+        .andExpect(content().string(""))
+        .andRespond(
+            withStatus(HttpStatus.ACCEPTED)
+                .contentType(MediaType.APPLICATION_JSON)
+                .body(
+                    """
+                    {"idempotencyKey":"%s","outcome":"QUEUED","revisionState":"PENDING",
+                     "attemptCount":0,"nextRetryAt":"2026-08-22T01:05:00Z"}
+                    """
+                        .formatted(key)));
+
+    SettlementRetryReceipt receipt =
+        new SettlementClient(builder.build()).retryRevision(revision, key);
+
+    assertThat(receipt.outcome()).isEqualTo(SettlementRetryReceipt.Outcome.QUEUED);
+    assertThat(receipt.revisionState()).isEqualTo(SettlementRevisionView.State.PENDING);
+    server.verify();
+  }
+
+  private static RestClient.Builder settlementBuilder() {
+    return RestClient.builder()
+        .baseUrl("http://settlement.test")
+        .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API)
+        .defaultHeader(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT);
+  }
+}


## `feat(settlement): expose lifecycle queries`

diff --git a/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java b/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java
new file mode 100644
index 0000000..7754980
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/SettlementQueryController.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.admin.client.SettlementCandidateView;
+import com.sportsbook.admin.client.SettlementClient;
+import com.sportsbook.admin.client.SettlementRevisionView;
+import java.util.UUID;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+@RestController
+@RequestMapping("/admin/v1/settlements")
+public class SettlementQueryController {
+
+  private final SettlementClient settlements;
+
+  public SettlementQueryController(SettlementClient settlements) {
+    this.settlements = settlements;
+  }
+
+  @GetMapping("/result-candidates/{candidateId}")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public SettlementCandidateView getCandidate(@PathVariable UUID candidateId) {
+    return settlements.getCandidate(candidateId);
+  }
+
+  @GetMapping("/revisions/{revisionId}")
+  @PreAuthorize("hasAnyRole('ADMIN','TRADER','CS','READONLY')")
+  public SettlementRevisionView getRevision(@PathVariable UUID revisionId) {
+    return settlements.getRevision(revisionId);
+  }
+}


## `test(settlement): guard lifecycle queries`

diff --git a/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java b/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java
new file mode 100644
index 0000000..823a24b
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/api/SettlementQueryControllerTest.java
@@ -0,0 +1,50 @@
+package com.sportsbook.admin.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+
+import com.sportsbook.admin.audit.Audited;
+import com.sportsbook.admin.client.SettlementClient;
+import java.lang.reflect.Method;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.access.prepost.PreAuthorize;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RequestMapping;
+
+class SettlementQueryControllerTest {
+
+  private static final UUID TARGET = UUID.fromString("018f0000-0000-7000-8000-000000000171");
+
+  @Test
+  void exposesCandidateEvidenceToEveryAdminRoleWithoutMutationAudit() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+
+    new SettlementQueryController(settlements).getCandidate(TARGET);
+
+    verify(settlements).getCandidate(TARGET);
+    assertReadContract("getCandidate", "/result-candidates/{candidateId}");
+  }
+
+  @Test
+  void exposesRevisionEvidenceToEveryAdminRoleWithoutMutationAudit() throws NoSuchMethodException {
+    SettlementClient settlements = mock(SettlementClient.class);
+
+    new SettlementQueryController(settlements).getRevision(TARGET);
+
+    verify(settlements).getRevision(TARGET);
+    assertReadContract("getRevision", "/revisions/{revisionId}");
+  }
+
+  private static void assertReadContract(String methodName, String path)
+      throws NoSuchMethodException {
+    assertThat(SettlementQueryController.class.getAnnotation(RequestMapping.class).value())
+        .containsExactly("/admin/v1/settlements");
+    Method method = SettlementQueryController.class.getMethod(methodName, UUID.class);
+    assertThat(method.getAnnotation(GetMapping.class).value()).containsExactly(path);
+    assertThat(method.getAnnotation(PreAuthorize.class).value())
+        .isEqualTo("hasAnyRole('ADMIN','TRADER','CS','READONLY')");
+    assertThat(method.getAnnotation(Audited.class)).isNull();
+  }
+}


