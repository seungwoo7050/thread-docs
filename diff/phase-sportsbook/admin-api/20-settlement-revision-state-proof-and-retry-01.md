# 정산 개정 상태 증명과 재시도

## `feat(settlement): model revision evidence`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementRevisionView.java b/src/main/java/com/sportsbook/admin/client/SettlementRevisionView.java
new file mode 100644
index 0000000..f848903
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementRevisionView.java
@@ -0,0 +1,40 @@
+package com.sportsbook.admin.client;
+
+import java.time.Instant;
+import java.util.UUID;
+
+public record SettlementRevisionView(
+    UUID revisionId,
+    UUID betId,
+    Long revisionNumber,
+    UUID eventId,
+    UUID sourceCandidateId,
+    State state,
+    Integer attemptCount,
+    Instant nextRetryAt,
+    String lastErrorCode,
+    Instant leaseUntil,
+    WalletStatus walletStatus,
+    Long walletQueueSequence,
+    UUID walletOperationGroupId,
+    Instant walletQueuedAt,
+    Instant walletAppliedAt,
+    Instant walletNextAttemptAt,
+    Instant createdAt,
+    Instant updatedAt,
+    Instant appliedAt) {
+
+  public enum State {
+    PENDING,
+    BLOCKED,
+    EXHAUSTED,
+    APPLIED,
+    REJECTED
+  }
+
+  public enum WalletStatus {
+    BLOCKED,
+    APPLIED,
+    REJECTED
+  }
+}


## `test(settlement): deserialize revision evidence`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionViewTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionViewTest.java
new file mode 100644
index 0000000..3c73a8a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionViewTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementRevisionViewTest {
+
+  private final ObjectMapper json = new ObjectMapper().findAndRegisterModules();
+
+  @Test
+  void deserializesAllNineteenTypedEvidenceFields() throws Exception {
+    SettlementRevisionView view =
+        json.readValue(
+            """
+            {
+              "revisionId":"018f0000-0000-7000-8000-000000000144",
+              "betId":"018f0000-0000-7000-8000-000000000145",
+              "revisionNumber":2,
+              "eventId":"018f0000-0000-7000-8000-000000000146",
+              "sourceCandidateId":"018f0000-0000-7000-8000-000000000147",
+              "state":"BLOCKED","attemptCount":3,
+              "nextRetryAt":"2026-08-22T01:03:00Z","lastErrorCode":"WALLET_BUSY",
+              "leaseUntil":null,"walletStatus":"BLOCKED","walletQueueSequence":17,
+              "walletOperationGroupId":"018f0000-0000-7000-8000-000000000148",
+              "walletQueuedAt":"2026-08-22T01:01:00Z","walletAppliedAt":null,
+              "walletNextAttemptAt":"2026-08-22T01:03:00Z",
+              "createdAt":"2026-08-22T01:00:00Z","updatedAt":"2026-08-22T01:02:00Z",
+              "appliedAt":null
+            }
+            """,
+            SettlementRevisionView.class);
+
+    assertThat(view.revisionId())
+        .isEqualTo(UUID.fromString("018f0000-0000-7000-8000-000000000144"));
+    assertThat(view.revisionNumber()).isEqualTo(2);
+    assertThat(view.state()).isEqualTo(SettlementRevisionView.State.BLOCKED);
+    assertThat(view.attemptCount()).isEqualTo(3);
+    assertThat(view.walletStatus()).isEqualTo(SettlementRevisionView.WalletStatus.BLOCKED);
+    assertThat(view.walletQueueSequence()).isEqualTo(17);
+    assertThat(view.nextRetryAt()).isEqualTo(Instant.parse("2026-08-22T01:03:00Z"));
+    assertThat(view.appliedAt()).isNull();
+  }
+}


## `feat(settlement): verify revision lifecycle proof`

diff --git a/src/main/java/com/sportsbook/admin/client/SettlementRevisionProof.java b/src/main/java/com/sportsbook/admin/client/SettlementRevisionProof.java
new file mode 100644
index 0000000..746e95d
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/SettlementRevisionProof.java
@@ -0,0 +1,91 @@
+package com.sportsbook.admin.client;
+
+import java.util.UUID;
+
+public final class SettlementRevisionProof {
+
+  private static final int MAX_ATTEMPTS = 12;
+
+  private SettlementRevisionProof() {}
+
+  public static SettlementRevisionView verify(UUID requestedId, SettlementRevisionView view) {
+    if (view == null
+        || !requestedId.equals(view.revisionId())
+        || view.betId() == null
+        || view.revisionNumber() == null
+        || view.revisionNumber() < 1
+        || view.eventId() == null
+        || view.sourceCandidateId() == null
+        || view.state() == null
+        || view.attemptCount() == null
+        || view.attemptCount() < 0
+        || view.attemptCount() > MAX_ATTEMPTS
+        || view.createdAt() == null
+        || view.updatedAt() == null
+        || view.updatedAt().isBefore(view.createdAt())
+        || (view.state() == SettlementRevisionView.State.APPLIED) != (view.appliedAt() != null)
+        || (view.appliedAt() != null && view.appliedAt().isBefore(view.createdAt()))
+        || invalidSchedule(view)
+        || invalidWalletProof(view)
+        || (view.state() == SettlementRevisionView.State.EXHAUSTED
+            && (view.lastErrorCode() == null || view.walletStatus() != null))
+        || (view.state() == SettlementRevisionView.State.REJECTED
+            && view.walletStatus() == null
+            && view.lastErrorCode() == null)) {
+      throw new DownstreamContractException("complete typed Settlement revision response");
+    }
+    return view;
+  }
+
+  private static boolean invalidSchedule(SettlementRevisionView view) {
+    if (view.leaseUntil() != null) {
+      return (view.state() != SettlementRevisionView.State.PENDING
+              && view.state() != SettlementRevisionView.State.BLOCKED)
+          || view.nextRetryAt() != null;
+    }
+    return switch (view.state()) {
+      case PENDING -> view.attemptCount() >= MAX_ATTEMPTS || view.nextRetryAt() == null;
+      case BLOCKED ->
+          view.nextRetryAt() == null
+              ? view.walletStatus() != SettlementRevisionView.WalletStatus.BLOCKED
+                  || view.lastErrorCode() == null
+              : view.attemptCount() >= MAX_ATTEMPTS;
+      case EXHAUSTED, APPLIED, REJECTED -> view.nextRetryAt() != null;
+    };
+  }
+
+  private static boolean invalidWalletProof(SettlementRevisionView view) {
+    boolean queued = view.walletQueueSequence() != null && view.walletQueuedAt() != null;
+    if ((view.walletQueueSequence() == null) != (view.walletQueuedAt() == null)
+        || (queued && view.walletQueueSequence() <= 0)) {
+      return true;
+    }
+    if (view.walletStatus() == null) {
+      return queued
+          || view.walletOperationGroupId() != null
+          || view.walletAppliedAt() != null
+          || view.walletNextAttemptAt() != null;
+    }
+    return switch (view.walletStatus()) {
+      case BLOCKED ->
+          (view.state() != SettlementRevisionView.State.PENDING
+                  && view.state() != SettlementRevisionView.State.BLOCKED)
+              || !queued
+              || view.walletOperationGroupId() != null
+              || view.walletAppliedAt() != null
+              || view.walletNextAttemptAt() == null;
+      case APPLIED ->
+          view.state() != SettlementRevisionView.State.APPLIED
+              || view.walletOperationGroupId() == null
+              || view.walletAppliedAt() == null
+              || view.walletNextAttemptAt() != null
+              || (queued && view.walletAppliedAt().isBefore(view.walletQueuedAt()));
+      case REJECTED ->
+          view.state() != SettlementRevisionView.State.REJECTED
+              || queued
+              || view.walletOperationGroupId() != null
+              || view.walletAppliedAt() != null
+              || view.walletNextAttemptAt() != null;
+    };
+  }
+}


## `test(settlement): accept valid revision lifecycle proofs`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java
new file mode 100644
index 0000000..8ae5867
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java
@@ -0,0 +1,66 @@
+package com.sportsbook.admin.client;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class SettlementRevisionProofTest {
+
+  private static final UUID REVISION = UUID.fromString("018f0000-0000-7000-8000-000000000151");
+  private static final Instant CREATED = Instant.parse("2026-08-22T01:00:00Z");
+
+  @Test
+  void acceptsPendingAndAppliedLifecycleProofs() {
+    SettlementRevisionView pending =
+        view(REVISION, 1L, SettlementRevisionView.State.PENDING, 0, CREATED, CREATED, null, null);
+    SettlementRevisionView applied =
+        view(
+            REVISION,
+            2L,
+            SettlementRevisionView.State.APPLIED,
+            3,
+            CREATED,
+            CREATED.plusSeconds(5),
+            CREATED.plusSeconds(5),
+            null);
+
+    assertThat(SettlementRevisionProof.verify(REVISION, pending)).isSameAs(pending);
+    assertThat(SettlementRevisionProof.verify(REVISION, applied)).isSameAs(applied);
+  }
+
+  private static SettlementRevisionView view(
+      UUID revisionId,
+      Long revisionNumber,
+      SettlementRevisionView.State state,
+      Integer attemptCount,
+      Instant createdAt,
+      Instant updatedAt,
+      Instant appliedAt,
+      Long walletQueueSequence) {
+    return new SettlementRevisionView(
+        revisionId,
+        UUID.fromString("018f0000-0000-7000-8000-000000000153"),
+        revisionNumber,
+        UUID.fromString("018f0000-0000-7000-8000-000000000154"),
+        UUID.fromString("018f0000-0000-7000-8000-000000000155"),
+        state,
+        attemptCount,
+        state == SettlementRevisionView.State.PENDING
+                || state == SettlementRevisionView.State.BLOCKED
+            ? createdAt
+            : null,
+        null,
+        null,
+        null,
+        walletQueueSequence,
+        null,
+        null,
+        null,
+        null,
+        createdAt,
+        updatedAt,
+        appliedAt);
+  }
+}


## `test(settlement): reject invalid revision lifecycle proofs`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java
index 8ae5867..7a3ef53 100644
--- a/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionProofTest.java
@@ -1,8 +1,10 @@
 package com.sportsbook.admin.client;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import java.time.Instant;
+import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
 
@@ -30,6 +32,73 @@ class SettlementRevisionProofTest {
     assertThat(SettlementRevisionProof.verify(REVISION, applied)).isSameAs(applied);
   }
 
+  @Test
+  void rejectsMismatchedUnsafeAndImpossibleLifecycleProofs() {
+    UUID other = UUID.fromString("018f0000-0000-7000-8000-000000000152");
+    List<SettlementRevisionView> invalid =
+        List.of(
+            view(other, 1L, SettlementRevisionView.State.PENDING, 0, CREATED, CREATED, null, null),
+            view(
+                REVISION,
+                0L,
+                SettlementRevisionView.State.PENDING,
+                0,
+                CREATED,
+                CREATED,
+                null,
+                null),
+            view(
+                REVISION,
+                1L,
+                SettlementRevisionView.State.PENDING,
+                13,
+                CREATED,
+                CREATED,
+                null,
+                null),
+            view(
+                REVISION,
+                1L,
+                SettlementRevisionView.State.PENDING,
+                0,
+                CREATED,
+                CREATED.minusSeconds(1),
+                null,
+                null),
+            view(
+                REVISION,
+                1L,
+                SettlementRevisionView.State.APPLIED,
+                1,
+                CREATED,
+                CREATED,
+                null,
+                null),
+            view(
+                REVISION,
+                1L,
+                SettlementRevisionView.State.PENDING,
+                0,
+                CREATED,
+                CREATED,
+                CREATED,
+                null),
+            view(
+                REVISION,
+                1L,
+                SettlementRevisionView.State.BLOCKED,
+                1,
+                CREATED,
+                CREATED,
+                null,
+                -1L));
+
+    invalid.forEach(
+        view ->
+            assertThatThrownBy(() -> SettlementRevisionProof.verify(REVISION, view))
+                .isInstanceOf(DownstreamContractException.class));
+  }
+
   private static SettlementRevisionView view(
       UUID revisionId,
       Long revisionNumber,


## `test(settlement): provide revision wallet proof fixture`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
new file mode 100644
index 0000000..a9764c3
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
@@ -0,0 +1,53 @@
+package com.sportsbook.admin.client;
+
+import java.time.Instant;
+import java.util.UUID;
+
+class SettlementRevisionWalletProofTest {
+
+  private static final UUID REVISION = UUID.fromString("018f0000-0000-7000-8000-000000000182");
+  private static final UUID OPERATION = UUID.fromString("018f0000-0000-7000-8000-000000000183");
+  private static final Instant CREATED = Instant.parse("2026-08-22T01:00:00Z");
+  private static final Instant QUEUED = CREATED.plusSeconds(1);
+  private static final Instant DUE = CREATED.plusSeconds(2);
+  private static final Instant UPDATED = CREATED.plusSeconds(3);
+
+  private static SettlementRevisionView view(
+      SettlementRevisionView.State state,
+      SettlementRevisionView.WalletStatus walletStatus,
+      Long queueSequence,
+      UUID operationGroup,
+      Instant queuedAt,
+      Instant walletAppliedAt,
+      Instant walletNextAttemptAt) {
+    boolean active =
+        state == SettlementRevisionView.State.PENDING
+            || state == SettlementRevisionView.State.BLOCKED;
+    String error =
+        state == SettlementRevisionView.State.EXHAUSTED
+                || state == SettlementRevisionView.State.REJECTED
+                || walletStatus == SettlementRevisionView.WalletStatus.BLOCKED
+            ? "WALLET_ERROR"
+            : null;
+    return new SettlementRevisionView(
+        REVISION,
+        UUID.fromString("018f0000-0000-7000-8000-000000000184"),
+        2L,
+        UUID.fromString("018f0000-0000-7000-8000-000000000185"),
+        UUID.fromString("018f0000-0000-7000-8000-000000000186"),
+        state,
+        3,
+        active ? DUE : null,
+        error,
+        null,
+        walletStatus,
+        queueSequence,
+        operationGroup,
+        queuedAt,
+        walletAppliedAt,
+        walletNextAttemptAt,
+        CREATED,
+        UPDATED,
+        state == SettlementRevisionView.State.APPLIED ? UPDATED : null);
+  }
+}


## `test(settlement): accept valid revision wallet proofs`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
index a9764c3..2aed627 100644
--- a/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
@@ -1,7 +1,11 @@
 package com.sportsbook.admin.client;
 
+import static org.assertj.core.api.Assertions.assertThat;
+
 import java.time.Instant;
+import java.util.List;
 import java.util.UUID;
+import org.junit.jupiter.api.Test;
 
 class SettlementRevisionWalletProofTest {
 
@@ -12,6 +16,41 @@ class SettlementRevisionWalletProofTest {
   private static final Instant DUE = CREATED.plusSeconds(2);
   private static final Instant UPDATED = CREATED.plusSeconds(3);
 
+  @Test
+  void acceptsBlockedAppliedRejectedAndExhaustedWalletProofs() {
+    List<SettlementRevisionView> valid =
+        List.of(
+            view(
+                SettlementRevisionView.State.BLOCKED,
+                SettlementRevisionView.WalletStatus.BLOCKED,
+                17L,
+                null,
+                QUEUED,
+                null,
+                DUE),
+            view(
+                SettlementRevisionView.State.APPLIED,
+                SettlementRevisionView.WalletStatus.APPLIED,
+                17L,
+                OPERATION,
+                QUEUED,
+                UPDATED,
+                null),
+            view(
+                SettlementRevisionView.State.REJECTED,
+                SettlementRevisionView.WalletStatus.REJECTED,
+                null,
+                null,
+                null,
+                null,
+                null),
+            view(SettlementRevisionView.State.EXHAUSTED, null, null, null, null, null, null));
+
+    valid.forEach(
+        evidence ->
+            assertThat(SettlementRevisionProof.verify(REVISION, evidence)).isSameAs(evidence));
+  }
+
   private static SettlementRevisionView view(
       SettlementRevisionView.State state,
       SettlementRevisionView.WalletStatus walletStatus,


## `test(settlement): reject invalid revision wallet proofs`

diff --git a/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
index 2aed627..c9d9c99 100644
--- a/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
+++ b/src/test/java/com/sportsbook/admin/client/SettlementRevisionWalletProofTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.admin.client;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import java.time.Instant;
 import java.util.List;
@@ -51,6 +52,57 @@ class SettlementRevisionWalletProofTest {
             assertThat(SettlementRevisionProof.verify(REVISION, evidence)).isSameAs(evidence));
   }
 
+  @Test
+  void rejectsIncompleteAndContradictoryWalletProofs() {
+    List<SettlementRevisionView> invalid =
+        List.of(
+            view(
+                SettlementRevisionView.State.BLOCKED,
+                SettlementRevisionView.WalletStatus.BLOCKED,
+                17L,
+                OPERATION,
+                QUEUED,
+                null,
+                DUE),
+            view(
+                SettlementRevisionView.State.BLOCKED,
+                SettlementRevisionView.WalletStatus.BLOCKED,
+                0L,
+                null,
+                QUEUED,
+                null,
+                DUE),
+            view(
+                SettlementRevisionView.State.APPLIED,
+                SettlementRevisionView.WalletStatus.APPLIED,
+                null,
+                null,
+                null,
+                UPDATED,
+                null),
+            view(
+                SettlementRevisionView.State.REJECTED,
+                SettlementRevisionView.WalletStatus.REJECTED,
+                17L,
+                null,
+                QUEUED,
+                null,
+                null),
+            view(
+                SettlementRevisionView.State.EXHAUSTED,
+                SettlementRevisionView.WalletStatus.BLOCKED,
+                17L,
+                null,
+                QUEUED,
+                null,
+                DUE));
+
+    invalid.forEach(
+        evidence ->
+            assertThatThrownBy(() -> SettlementRevisionProof.verify(REVISION, evidence))
+                .isInstanceOf(DownstreamContractException.class));
+  }
+
   private static SettlementRevisionView view(
       SettlementRevisionView.State state,
       SettlementRevisionView.WalletStatus walletStatus,


