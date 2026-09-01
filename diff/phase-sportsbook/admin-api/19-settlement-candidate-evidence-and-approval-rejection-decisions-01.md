# 정산 후보 증거와 승인·거절 결정

## `feat(settlement): model candidate evidence`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementCandidateView.java b/src/main/java/com/sportsbook/admin/client/SettlementCandidateView.java
new file mode 100644
index 0000000..c1d978e
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementCandidateView.java
@@ -0,0 +1,46 @@
+package com.sportsbook.admin.client;
+
+import java.time.Instant;
+import java.util.UUID;
+
+public record SettlementCandidateView(
+    UUID candidateId,
+    UUID eventId,
+    Mode mode,
+    Instant settledAt,
+    Instant receivedAt,
+    State state,
+    UUID replacesCandidateId,
+    String decisionReason,
+    Instant decidedAt,
+    Boolean accepted) {
+
+  public static SettlementCandidateView verify(UUID requestedId, SettlementCandidateView view) {
+    if (view == null
+        || !requestedId.equals(view.candidateId())
+        || view.eventId() == null
+        || view.mode() == null
+        || view.settledAt() == null
+        || view.receivedAt() == null
+        || view.state() == null
+        || view.accepted() == null
+        || view.accepted() != (view.state() == State.ACCEPTED)
+        || (view.state() == State.PENDING) != (view.decidedAt() == null)) {
+      throw new DownstreamContractException("complete typed Settlement candidate response");
+    }
+    return view;
+  }
+
+  public enum Mode {
+    COMPLETED,
+    ABANDONED,
+    VOIDED
+  }
+
+  public enum State {
+    PENDING,
+    ACCEPTED,
+    SUPERSEDED,
+    REJECTED
+  }
+}


## `test(settlement): validate candidate lifecycle evidence`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementCandidateViewTest.java b/src/test/java/com/sportsbook/admin/client/SettlementCandidateViewTest.java
new file mode 100644
index 0000000..9b8ecd1
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementCandidateViewTest.java
@@ -0,0 +1,76 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementCandidateViewTest {
+
+  private static final UUID CANDIDATE = UUID.fromString("018f0000-0000-7000-8000-000000000141");
+  private static final UUID EVENT = UUID.fromString("018f0000-0000-7000-8000-000000000142");
+  private static final Instant SETTLED = Instant.parse("2026-08-22T01:00:00Z");
+  private static final Instant RECEIVED = Instant.parse("2026-08-22T01:00:01Z");
+
+  @Test
+  void acceptsPendingAndDecidedCandidateEvidence() {
+    SettlementCandidateView pending =
+        view(SettlementCandidateView.State.PENDING, null, null, false);
+    SettlementCandidateView accepted =
+        view(
+            SettlementCandidateView.State.ACCEPTED,
+            "approved by operator",
+            RECEIVED.plusSeconds(2),
+            true);
+
+    assertThat(SettlementCandidateView.verify(CANDIDATE, pending)).isSameAs(pending);
+    assertThat(SettlementCandidateView.verify(CANDIDATE, accepted)).isSameAs(accepted);
+  }
+
+  @Test
+  void rejectsMismatchedIncompleteAndImpossibleLifecycleEvidence() {
+    UUID other = UUID.fromString("018f0000-0000-7000-8000-000000000143");
+    assertInvalid(
+        new SettlementCandidateView(
+            other,
+            EVENT,
+            SettlementCandidateView.Mode.COMPLETED,
+            SETTLED,
+            RECEIVED,
+            SettlementCandidateView.State.PENDING,
+            null,
+            null,
+            null,
+            false));
+    assertInvalid(view(SettlementCandidateView.State.PENDING, null, RECEIVED, false));
+    assertInvalid(view(SettlementCandidateView.State.REJECTED, "bad result", null, false));
+    assertInvalid(view(SettlementCandidateView.State.PENDING, null, null, true));
+    assertInvalid(
+        view(SettlementCandidateView.State.ACCEPTED, "approved", RECEIVED.plusSeconds(2), false));
+    assertInvalid(
+        new SettlementCandidateView(
+            CANDIDATE, null, null, null, null, null, null, null, null, null));
+  }
+
+  private static void assertInvalid(SettlementCandidateView view) {
+    assertThatThrownBy(() -> SettlementCandidateView.verify(CANDIDATE, view))
+        .isInstanceOf(DownstreamContractException.class);
+  }
+
+  private static SettlementCandidateView view(
+      SettlementCandidateView.State state, String reason, Instant decidedAt, boolean accepted) {
+    return new SettlementCandidateView(
+        CANDIDATE,
+        EVENT,
+        SettlementCandidateView.Mode.COMPLETED,
+        SETTLED,
+        RECEIVED,
+        state,
+        null,
+        reason,
+        decidedAt,
+        accepted);
+  }
+}


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


## `test(settlement): fetch exact candidate evidence`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateQueryTest.java b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateQueryTest.java
new file mode 100644
index 0000000..66ac31d
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateQueryTest.java
@@ -0,0 +1,58 @@
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
+class SettlementClientCandidateQueryTest {
+
+  @Test
+  void fetchesCandidateWithOnlySettlementAuthentication() {
+    UUID candidate = UUID.fromString("018f0000-0000-7000-8000-000000000156");
+    RestClient.Builder builder = settlementBuilder();
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(requestTo("http://settlement.test/internal/admin/result-candidates/" + candidate))
+        .andExpect(method(GET))
+        .andExpect(header(DownstreamHeaders.SERVICE_NAME, "admin-api"))
+        .andExpect(header(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT))
+        .andExpect(headerDoesNotExist(DownstreamHeaders.INTERNAL_SERVICE))
+        .andExpect(headerDoesNotExist("Idempotency-Key"))
+        .andRespond(withSuccess(response(candidate), MediaType.APPLICATION_JSON));
+
+    SettlementCandidateView result = new SettlementClient(builder.build()).getCandidate(candidate);
+
+    assertThat(result.candidateId()).isEqualTo(candidate);
+    assertThat(result.state()).isEqualTo(SettlementCandidateView.State.ACCEPTED);
+    assertThat(result.accepted()).isTrue();
+    server.verify();
+  }
+
+  private static RestClient.Builder settlementBuilder() {
+    return RestClient.builder()
+        .baseUrl("http://settlement.test")
+        .defaultHeader(DownstreamHeaders.SERVICE_NAME, DownstreamHeaders.ADMIN_API)
+        .defaultHeader(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT);
+  }
+
+  private static String response(UUID candidate) {
+    return """
+        {"candidateId":"%s","eventId":"018f0000-0000-7000-8000-000000000157",
+         "mode":"COMPLETED","settledAt":"2026-08-22T01:00:00Z",
+         "receivedAt":"2026-08-22T01:00:01Z","state":"ACCEPTED",
+         "replacesCandidateId":null,"decisionReason":"approved",
+         "decidedAt":"2026-08-22T01:00:02Z","accepted":true}
+        """
+        .formatted(candidate);
+  }
+}


## `feat(settlement): verify candidate decisions`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementCandidateReceipt.java b/src/main/java/com/sportsbook/admin/client/SettlementCandidateReceipt.java
new file mode 100644
index 0000000..d28fc81
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementCandidateReceipt.java
@@ -0,0 +1,22 @@
+package com.sportsbook.admin.client;
+
+import java.util.UUID;
+
+public record SettlementCandidateReceipt(UUID idempotencyKey, Outcome outcome, Boolean replay) {
+
+  public static SettlementCandidateReceipt verify(
+      UUID requestedKey, Outcome expectedOutcome, SettlementCandidateReceipt receipt) {
+    if (receipt == null
+        || !requestedKey.equals(receipt.idempotencyKey())
+        || receipt.outcome() != expectedOutcome
+        || receipt.replay() == null) {
+      throw new DownstreamContractException("matching Settlement candidate receipt");
+    }
+    return receipt;
+  }
+
+  public enum Outcome {
+    CANDIDATE_APPROVED,
+    CANDIDATE_REJECTED
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/SettlementRejectionPayload.java b/src/main/java/com/sportsbook/admin/client/SettlementRejectionPayload.java
new file mode 100644
index 0000000..5b51b2b
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementRejectionPayload.java
@@ -0,0 +1,18 @@
+package com.sportsbook.admin.client;
+
+import java.util.Objects;
+
+public record SettlementRejectionPayload(String reason) {
+
+  private static final int MAX_REASON_LENGTH = 256;
+
+  public SettlementRejectionPayload {
+    Objects.requireNonNull(reason, "reason");
+    reason = reason.trim();
+    if (reason.isEmpty()
+        || reason.length() > MAX_REASON_LENGTH
+        || reason.codePoints().anyMatch(Character::isISOControl)) {
+      throw new IllegalArgumentException("Rejection reason must be 1 to 256 printable characters");
+    }
+  }
+}


## `test(settlement): validate candidate decisions`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementCandidateReceiptTest.java b/src/test/java/com/sportsbook/admin/client/SettlementCandidateReceiptTest.java
new file mode 100644
index 0000000..60a0599
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementCandidateReceiptTest.java
@@ -0,0 +1,55 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementCandidateReceiptTest {
+
+  private static final UUID KEY = UUID.fromString("018f0000-0000-7000-8000-000000000161");
+
+  @Test
+  void acceptsMatchingApprovalAndRejectionReceipts() {
+    SettlementCandidateReceipt approval =
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, false);
+    SettlementCandidateReceipt rejection =
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, true);
+
+    assertThat(
+            SettlementCandidateReceipt.verify(
+                KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, approval))
+        .isSameAs(approval);
+    assertThat(
+            SettlementCandidateReceipt.verify(
+                KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, rejection))
+        .isSameAs(rejection);
+  }
+
+  @Test
+  void rejectsMissingMismatchedAndIncompleteReceipts() {
+    UUID other = UUID.fromString("018f0000-0000-7000-8000-000000000162");
+
+    assertInvalid(null);
+    assertInvalid(
+        new SettlementCandidateReceipt(
+            other, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, false));
+    assertInvalid(
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_REJECTED, false));
+    assertInvalid(
+        new SettlementCandidateReceipt(
+            KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, null));
+  }
+
+  private static void assertInvalid(SettlementCandidateReceipt receipt) {
+    assertThatThrownBy(
+            () ->
+                SettlementCandidateReceipt.verify(
+                    KEY, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, receipt))
+        .isInstanceOf(DownstreamContractException.class);
+  }
+}
diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRejectionPayloadTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRejectionPayloadTest.java
new file mode 100644
index 0000000..8c017d2
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRejectionPayloadTest.java
@@ -0,0 +1,27 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import org.junit.jupiter.api.Test;
+
+class SettlementRejectionPayloadTest {
+
+  @Test
+  void trimsAReasonWithinTheProviderLimit() {
+    assertThat(new SettlementRejectionPayload("  bad result  ").reason()).isEqualTo("bad result");
+    assertThat(new SettlementRejectionPayload("r".repeat(256)).reason()).hasSize(256);
+  }
+
+  @Test
+  void rejectsMissingBlankOversizedAndControlReasons() {
+    assertThatThrownBy(() -> new SettlementRejectionPayload(null))
+        .isInstanceOf(NullPointerException.class);
+    assertThatThrownBy(() -> new SettlementRejectionPayload("   "))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new SettlementRejectionPayload("r".repeat(257)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new SettlementRejectionPayload("bad\nresult"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(settlement): approve result candidates`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementClient.java b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
index b481a03..73124bd 100644
--- a/src/main/java/com/sportsbook/admin/client/SettlementClient.java
+++ b/src/main/java/com/sportsbook/admin/client/SettlementClient.java
@@ -59,4 +59,28 @@ public class SettlementClient {
             "Settlement revision GET must return HTTP 200 with a body");
     return SettlementRevisionProof.verify(revisionId, body);
   }
+
+  public SettlementCandidateReceipt approveCandidate(UUID candidateId, UUID idempotencyKey) {
+    var response =
+        failures.execute(
+            () ->
+                http.post()
+                    .uri(
+                        builder ->
+                            builder
+                                .pathSegment("internal", "admin", "result-candidates")
+                                .pathSegment(candidateId.toString(), "approve")
+                                .build())
+                    .header("Idempotency-Key", idempotencyKey.toString())
+                    .retrieve()
+                    .toEntity(SettlementCandidateReceipt.class));
+    SettlementCandidateReceipt receipt =
+        DownstreamContract.requireBody(
+            response,
+            HttpStatus.OK,
+            ignored -> true,
+            "Settlement candidate approval must return HTTP 200 with a receipt");
+    return SettlementCandidateReceipt.verify(
+        idempotencyKey, SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED, receipt);
+  }
 }


## `test(settlement): send exact candidate approval`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateApprovalTest.java b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateApprovalTest.java
new file mode 100644
index 0000000..e3af0dd
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementClientCandidateApprovalTest.java
@@ -0,0 +1,61 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.http.HttpMethod.POST;
+import static org.springframework.test.web.client.match.MockRestRequestMatchers.content;
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
+class SettlementClientCandidateApprovalTest {
+
+  @Test
+  void postsAnEmptyApprovalWithTheExactSettlementHeaders() {
+    UUID candidate = UUID.fromString("018f0000-0000-7000-8000-000000000163");
+    UUID key = UUID.fromString("018f0000-0000-7000-8000-000000000164");
+    RestClient.Builder builder = settlementBuilder();
+    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
+    server
+        .expect(
+            requestTo(
+                "http://settlement.test/internal/admin/result-candidates/"
+                    + candidate
+                    + "/approve"))
+        .andExpect(method(POST))
+        .andExpect(header(DownstreamHeaders.SERVICE_NAME, "admin-api"))
+        .andExpect(header(DownstreamHeaders.API_KEY, ClientIsolationFixture.SETTLEMENT))
+        .andExpect(header("Idempotency-Key", key.toString()))
+        .andExpect(headerDoesNotExist(DownstreamHeaders.INTERNAL_SERVICE))
+        .andExpect(content().string(""))
+        .andRespond(
+            withSuccess(
+                """
+                {"idempotencyKey":"%s","outcome":"CANDIDATE_APPROVED","replay":false}
+                """
+                    .formatted(key),
+                MediaType.APPLICATION_JSON));
+
+    SettlementCandidateReceipt receipt =
+        new SettlementClient(builder.build()).approveCandidate(candidate, key);
+
+    assertThat(receipt.idempotencyKey()).isEqualTo(key);
+    assertThat(receipt.outcome()).isEqualTo(SettlementCandidateReceipt.Outcome.CANDIDATE_APPROVED);
+    assertThat(receipt.replay()).isFalse();
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


