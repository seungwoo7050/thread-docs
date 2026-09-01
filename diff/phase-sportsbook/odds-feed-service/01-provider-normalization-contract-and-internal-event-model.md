# 공급자 정규화 계약과 내부 이벤트 모델

## `build(protocol): use shared protocol 1.0.0`

diff --git a/pom.xml b/pom.xml
index 91196c6..c0fadce 100644
--- a/pom.xml
+++ b/pom.xml
@@ -24,4 +24,12 @@
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <shared-protocol.version>1.0.0</shared-protocol.version>
     </properties>
+
+    <dependencies>
+        <dependency>
+            <groupId>com.sportsbook</groupId>
+            <artifactId>shared-protocol</artifactId>
+            <version>${shared-protocol.version}</version>
+        </dependency>
+    </dependencies>
 </project>


## `feat(provider): define event summaries`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/EventSummary.java b/src/main/java/com/sportsbook/oddsfeed/provider/EventSummary.java
new file mode 100644
index 0000000..4c187e1
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/EventSummary.java
@@ -0,0 +1,26 @@
+package com.sportsbook.oddsfeed.provider;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.Objects;
+
+public record EventSummary(
+    EventId eventId,
+    Sport sport,
+    String competition,
+    String homeTeam,
+    String awayTeam,
+    Instant scheduledStartAt,
+    EventLifecycleStatus status) {
+
+  public EventSummary {
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(sport, "sport");
+    Objects.requireNonNull(competition, "competition");
+    Objects.requireNonNull(homeTeam, "homeTeam");
+    Objects.requireNonNull(awayTeam, "awayTeam");
+    Objects.requireNonNull(scheduledStartAt, "scheduledStartAt");
+    Objects.requireNonNull(status, "status");
+  }
+}
diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/Sport.java b/src/main/java/com/sportsbook/oddsfeed/provider/Sport.java
new file mode 100644
index 0000000..81e35dc
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/Sport.java
@@ -0,0 +1,6 @@
+package com.sportsbook.oddsfeed.provider;
+
+public enum Sport {
+  FOOTBALL,
+  BASKETBALL
+}


## `test(provider): verify event summaries`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/EventSummaryTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/EventSummaryTest.java
new file mode 100644
index 0000000..a60d079
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/EventSummaryTest.java
@@ -0,0 +1,65 @@
+package com.sportsbook.oddsfeed.provider;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class EventSummaryTest {
+
+  @Test
+  void rejectsNullRequiredFields() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    Instant kickoff = Instant.parse("2026-06-01T18:00:00Z");
+    assertThatNullPointerException()
+        .isThrownBy(
+            () ->
+                new EventSummary(
+                    null,
+                    Sport.FOOTBALL,
+                    "Premier League",
+                    "Manchester United",
+                    "Chelsea",
+                    kickoff,
+                    EventLifecycleStatus.SCHEDULED))
+        .withMessageContaining("eventId");
+    assertThatNullPointerException()
+        .isThrownBy(
+            () ->
+                new EventSummary(
+                    eventId,
+                    Sport.FOOTBALL,
+                    "Premier League",
+                    "Manchester United",
+                    "Chelsea",
+                    null,
+                    EventLifecycleStatus.SCHEDULED))
+        .withMessageContaining("scheduledStartAt");
+  }
+
+  @Test
+  void carriesAllFieldsExactly() {
+    EventId eventId = new EventId(UUID.randomUUID());
+    Instant kickoff = Instant.parse("2026-06-01T18:00:00Z");
+    EventSummary summary =
+        new EventSummary(
+            eventId,
+            Sport.FOOTBALL,
+            "Premier League",
+            "Manchester United",
+            "Chelsea",
+            kickoff,
+            EventLifecycleStatus.SCHEDULED);
+    assertThat(summary.eventId()).isEqualTo(eventId);
+    assertThat(summary.sport()).isEqualTo(Sport.FOOTBALL);
+    assertThat(summary.competition()).isEqualTo("Premier League");
+    assertThat(summary.homeTeam()).isEqualTo("Manchester United");
+    assertThat(summary.awayTeam()).isEqualTo("Chelsea");
+    assertThat(summary.scheduledStartAt()).isEqualTo(kickoff);
+    assertThat(summary.status()).isEqualTo(EventLifecycleStatus.SCHEDULED);
+  }
+}


## `feat(provider): define match outcomes`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/MatchOutcome.java b/src/main/java/com/sportsbook/oddsfeed/provider/MatchOutcome.java
new file mode 100644
index 0000000..3279e1c
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/MatchOutcome.java
@@ -0,0 +1,24 @@
+package com.sportsbook.oddsfeed.provider;
+
+import com.sportsbook.protocol.event.MatchFinalStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.Map;
+import java.util.Objects;
+
+public record MatchOutcome(
+    EventId eventId,
+    String score,
+    MatchFinalStatus finalStatus,
+    Map<String, String> detail,
+    Instant settledAt) {
+
+  public MatchOutcome {
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(score, "score");
+    Objects.requireNonNull(finalStatus, "finalStatus");
+    Objects.requireNonNull(detail, "detail");
+    Objects.requireNonNull(settledAt, "settledAt");
+    detail = Map.copyOf(detail);
+  }
+}


## `test(provider): verify match outcomes`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/MatchOutcomeTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/MatchOutcomeTest.java
new file mode 100644
index 0000000..2219608
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/MatchOutcomeTest.java
@@ -0,0 +1,44 @@
+package com.sportsbook.oddsfeed.provider;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.MatchFinalStatus;
+import com.sportsbook.protocol.value.EventId;
+import java.time.Instant;
+import java.util.HashMap;
+import java.util.Map;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class MatchOutcomeTest {
+
+  @Test
+  void rejectsNullRequiredFields() {
+    assertThatNullPointerException()
+        .isThrownBy(
+            () ->
+                new MatchOutcome(null, "2-1", MatchFinalStatus.COMPLETED, Map.of(), Instant.now()))
+        .withMessageContaining("eventId");
+  }
+
+  @Test
+  void copiesDetailMapDefensively() {
+    Map<String, String> mutable = new HashMap<>();
+    mutable.put("homeGoals", "2");
+    MatchOutcome outcome =
+        new MatchOutcome(
+            new EventId(UUID.randomUUID()),
+            "2-1",
+            MatchFinalStatus.COMPLETED,
+            mutable,
+            Instant.now());
+
+    mutable.put("awayGoals", "1");
+
+    assertThat(outcome.detail()).containsOnlyKeys("homeGoals");
+    assertThatThrownBy(() -> outcome.detail().put("k", "v"))
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
+}


## `feat(provider): define provider events`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/ProviderEvent.java b/src/main/java/com/sportsbook/oddsfeed/provider/ProviderEvent.java
new file mode 100644
index 0000000..bc62b42
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/ProviderEvent.java
@@ -0,0 +1,66 @@
+package com.sportsbook.oddsfeed.provider;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.time.Instant;
+import java.util.Objects;
+
+public sealed interface ProviderEvent
+    permits ProviderEvent.OddsUpdated,
+        ProviderEvent.MarketStatusUpdated,
+        ProviderEvent.LifecycleUpdated {
+
+  EventId eventId();
+
+  Instant occurredAt();
+
+  record OddsUpdated(
+      EventId eventId,
+      MarketId marketId,
+      SelectionId selectionId,
+      Odds previousOdds,
+      Odds newOdds,
+      Instant occurredAt)
+      implements ProviderEvent {
+    public OddsUpdated {
+      Objects.requireNonNull(eventId, "eventId");
+      Objects.requireNonNull(marketId, "marketId");
+      Objects.requireNonNull(selectionId, "selectionId");
+      Objects.requireNonNull(previousOdds, "previousOdds");
+      Objects.requireNonNull(newOdds, "newOdds");
+      Objects.requireNonNull(occurredAt, "occurredAt");
+    }
+  }
+
+  record MarketStatusUpdated(
+      EventId eventId,
+      MarketId marketId,
+      MarketStatus previousStatus,
+      MarketStatus newStatus,
+      String reason,
+      Instant occurredAt)
+      implements ProviderEvent {
+    public MarketStatusUpdated {
+      Objects.requireNonNull(eventId, "eventId");
+      Objects.requireNonNull(marketId, "marketId");
+      Objects.requireNonNull(previousStatus, "previousStatus");
+      Objects.requireNonNull(newStatus, "newStatus");
+      Objects.requireNonNull(occurredAt, "occurredAt");
+    }
+  }
+
+  record LifecycleUpdated(
+      EventId eventId, EventLifecycleStatus status, Instant scheduledStartAt, Instant occurredAt)
+      implements ProviderEvent {
+    public LifecycleUpdated {
+      Objects.requireNonNull(eventId, "eventId");
+      Objects.requireNonNull(status, "status");
+      Objects.requireNonNull(scheduledStartAt, "scheduledStartAt");
+      Objects.requireNonNull(occurredAt, "occurredAt");
+    }
+  }
+}


## `test(provider): verify provider events`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/ProviderEventTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/ProviderEventTest.java
new file mode 100644
index 0000000..0225082
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/ProviderEventTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.oddsfeed.provider;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNullPointerException;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.event.MarketStatus;
+import com.sportsbook.protocol.value.EventId;
+import com.sportsbook.protocol.value.MarketId;
+import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.protocol.value.SelectionId;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class ProviderEventTest {
+
+  private static final EventId EVENT = new EventId(UUID.randomUUID());
+  private static final MarketId MARKET = new MarketId(UUID.randomUUID());
+  private static final SelectionId SELECTION = new SelectionId(UUID.randomUUID());
+  private static final Instant NOW = Instant.parse("2026-05-28T10:00:00Z");
+
+  @Test
+  void sealedHierarchyExposesExactlyThreePermittedTypes() {
+    assertThat(ProviderEvent.class.getPermittedSubclasses())
+        .extracting(Class::getSimpleName)
+        .containsExactlyInAnyOrder("OddsUpdated", "MarketStatusUpdated", "LifecycleUpdated");
+  }
+
+  @Test
+  void oddsUpdatedRejectsNullFields() {
+    Odds o1 = Odds.ofDecimal("1.85");
+    Odds o2 = Odds.ofDecimal("1.90");
+    assertThatNullPointerException()
+        .isThrownBy(() -> new ProviderEvent.OddsUpdated(null, MARKET, SELECTION, o1, o2, NOW))
+        .withMessageContaining("eventId");
+    assertThatNullPointerException()
+        .isThrownBy(() -> new ProviderEvent.OddsUpdated(EVENT, MARKET, SELECTION, null, o2, NOW))
+        .withMessageContaining("previousOdds");
+  }
+
+  @Test
+  void marketStatusUpdatedAllowsNullReason() {
+    ProviderEvent.MarketStatusUpdated event =
+        new ProviderEvent.MarketStatusUpdated(
+            EVENT, MARKET, MarketStatus.OPEN, MarketStatus.SUSPENDED, null, NOW);
+    assertThat(event.reason()).isNull();
+    assertThat(event.previousStatus()).isEqualTo(MarketStatus.OPEN);
+    assertThat(event.newStatus()).isEqualTo(MarketStatus.SUSPENDED);
+  }
+
+  @Test
+  void lifecycleUpdatedExposesScheduledStart() {
+    Instant kickoff = NOW.plusSeconds(3600);
+    ProviderEvent.LifecycleUpdated event =
+        new ProviderEvent.LifecycleUpdated(EVENT, EventLifecycleStatus.SCHEDULED, kickoff, NOW);
+    assertThat(event.scheduledStartAt()).isEqualTo(kickoff);
+    assertThat(event.status()).isEqualTo(EventLifecycleStatus.SCHEDULED);
+  }
+}


## `feat(provider): define odds provider contract`

diff --git a/src/main/java/com/sportsbook/oddsfeed/provider/OddsProvider.java b/src/main/java/com/sportsbook/oddsfeed/provider/OddsProvider.java
new file mode 100644
index 0000000..bf5f819
--- /dev/null
+++ b/src/main/java/com/sportsbook/oddsfeed/provider/OddsProvider.java
@@ -0,0 +1,15 @@
+package com.sportsbook.oddsfeed.provider;
+
+import com.sportsbook.protocol.value.EventId;
+import java.util.List;
+import java.util.Optional;
+import reactor.core.publisher.Flux;
+
+public interface OddsProvider {
+
+  List<EventSummary> listEvents(Sport sport);
+
+  Flux<ProviderEvent> streamEvents(EventId eventId);
+
+  Optional<MatchOutcome> getMatchResult(EventId eventId);
+}


## `test(provider): verify provider contract`

diff --git a/src/test/java/com/sportsbook/oddsfeed/provider/OddsProviderTest.java b/src/test/java/com/sportsbook/oddsfeed/provider/OddsProviderTest.java
new file mode 100644
index 0000000..7dae83d
--- /dev/null
+++ b/src/test/java/com/sportsbook/oddsfeed/provider/OddsProviderTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.oddsfeed.provider;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.EventId;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import reactor.core.publisher.Flux;
+import reactor.test.StepVerifier;
+
+class OddsProviderTest {
+
+  @Test
+  void supportsSnapshotStreamAndResultLookups() {
+    OddsProvider provider =
+        new OddsProvider() {
+          @Override
+          public List<EventSummary> listEvents(Sport sport) {
+            return List.of();
+          }
+
+          @Override
+          public Flux<ProviderEvent> streamEvents(EventId eventId) {
+            return Flux.empty();
+          }
+
+          @Override
+          public Optional<MatchOutcome> getMatchResult(EventId eventId) {
+            return Optional.empty();
+          }
+        };
+
+    EventId eventId = new EventId(UUID.randomUUID());
+    assertThat(provider.listEvents(Sport.FOOTBALL)).isEmpty();
+    assertThat(provider.getMatchResult(eventId)).isEmpty();
+    StepVerifier.create(provider.streamEvents(eventId)).verifyComplete();
+  }
+}
