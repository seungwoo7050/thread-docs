## `feat(api): identify limit override targets`

diff --git a/src/main/java/com/sportsbook/risk/api/LimitOverrideTarget.java b/src/main/java/com/sportsbook/risk/api/LimitOverrideTarget.java
new file mode 100644
index 0000000..0716ec4
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/LimitOverrideTarget.java
@@ -0,0 +1,25 @@
+package com.sportsbook.risk.api;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideField;
+import java.util.Objects;
+
+/** Identifies one currency-scoped monetary limit or the currency-neutral selection limit. */
+public record LimitOverrideTarget(LimitType type, Currency currency) {
+  public LimitOverrideTarget {
+    Objects.requireNonNull(type, "type");
+    if (type.currencyScoped() && currency == null) {
+      throw new IllegalArgumentException("currency is required for monetary limits");
+    }
+    if (!type.currencyScoped() && currency != null) {
+      throw new IllegalArgumentException("currency must be omitted for selection limits");
+    }
+  }
+
+  LimitOverrideField field() {
+    return type.currencyScoped()
+        ? LimitOverrideField.monetary(type, currency)
+        : LimitOverrideField.selections();
+  }
+}


## `feat(api): expose effective user limits`

diff --git a/src/main/java/com/sportsbook/risk/api/LimitController.java b/src/main/java/com/sportsbook/risk/api/LimitController.java
new file mode 100644
index 0000000..89328a1
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/LimitController.java
@@ -0,0 +1,56 @@
+package com.sportsbook.risk.api;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideField;
+import com.sportsbook.risk.limit.LimitOverrideStore;
+import com.sportsbook.risk.limit.LimitResolver;
+import java.util.ArrayList;
+import java.util.OptionalLong;
+import java.util.UUID;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Admin-owned effective-limit operations. */
+@RestController
+@RequestMapping("/internal/v1/risk/limits")
+public class LimitController {
+  private final LimitOverrideStore overrides;
+  private final LimitResolver resolver;
+
+  public LimitController(LimitOverrideStore overrides, LimitResolver resolver) {
+    this.overrides = overrides;
+    this.resolver = resolver;
+  }
+
+  @GetMapping("/{userId}")
+  public UserLimitsResponse get(@PathVariable UUID userId) {
+    UserId typedUserId = UserId.of(userId);
+    var entries = new ArrayList<UserLimitsResponse.Entry>();
+    for (LimitType type : LimitType.values()) {
+      if (type.currencyScoped()) {
+        for (Currency currency : Currency.values()) {
+          entries.add(entry(typedUserId, new LimitOverrideTarget(type, currency)));
+        }
+      } else {
+        entries.add(entry(typedUserId, new LimitOverrideTarget(type, null)));
+      }
+    }
+    return new UserLimitsResponse(typedUserId, entries);
+  }
+
+  private UserLimitsResponse.Entry entry(UserId userId, LimitOverrideTarget target) {
+    LimitOverrideField field = target.field();
+    OptionalLong override = overrides.find(userId, field);
+    long value =
+        override.orElseGet(() -> resolver.resolve(userId, target.type(), target.currency()));
+    var source =
+        override.isPresent()
+            ? UserLimitsResponse.Source.OVERRIDE
+            : UserLimitsResponse.Source.POLICY;
+    return new UserLimitsResponse.Entry(target.type(), target.currency(), value, source);
+  }
+}


## `test(api): verify effective limit responses`

diff --git a/src/test/java/com/sportsbook/risk/api/LimitControllerReadTest.java b/src/test/java/com/sportsbook/risk/api/LimitControllerReadTest.java
new file mode 100644
index 0000000..32e1466
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/LimitControllerReadTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.risk.api;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideField;
+import com.sportsbook.risk.limit.LimitOverrideStore;
+import com.sportsbook.risk.limit.LimitResolver;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import java.util.OptionalLong;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+@ExtendWith(MockitoExtension.class)
+class LimitControllerReadTest {
+  private static final UUID USER_VALUE = UUID.fromString("00000000-0000-0000-0000-000000000001");
+  private static final UserId USER = UserId.of(USER_VALUE);
+
+  @Mock private LimitOverrideStore overrides;
+  private LimitResolver resolver;
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    resolver = new LimitResolver(new RiskLimitProperties(null, null, null, null, 0), overrides);
+    mvc = MockMvcBuilders.standaloneSetup(new LimitController(overrides, resolver)).build();
+  }
+
+  @Test
+  void returnsEveryEffectiveLimitAndItsSource() throws Exception {
+    when(overrides.find(any(), any())).thenReturn(OptionalLong.empty());
+    var dailyKrw = LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.KRW);
+    when(overrides.find(eq(USER), eq(dailyKrw))).thenReturn(OptionalLong.of(750L));
+
+    mvc.perform(get("/internal/v1/risk/limits/" + USER_VALUE))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.userId").value(USER_VALUE.toString()))
+        .andExpect(jsonPath("$.limits.length()").value(7))
+        .andExpect(jsonPath("$.limits[0].source").value("OVERRIDE"))
+        .andExpect(jsonPath("$.limits[6].type").value("SELECTIONS_PER_MINUTE"))
+        .andExpect(jsonPath("$.limits[6].currency").doesNotExist());
+  }
+}


## `feat(api): validate limit replacements`

diff --git a/src/main/java/com/sportsbook/risk/api/LimitUpdateRequest.java b/src/main/java/com/sportsbook/risk/api/LimitUpdateRequest.java
new file mode 100644
index 0000000..a83b4a1
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/LimitUpdateRequest.java
@@ -0,0 +1,38 @@
+package com.sportsbook.risk.api;
+
+import com.fasterxml.jackson.annotation.JsonIgnore;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import jakarta.validation.constraints.AssertTrue;
+import jakarta.validation.constraints.NotNull;
+import jakarta.validation.constraints.PositiveOrZero;
+
+/** Exact administrative replacement for one user limit. */
+public record LimitUpdateRequest(
+    @NotNull LimitType type, Currency currency, @NotNull @PositiveOrZero Long value) {
+
+  @JsonIgnore
+  @AssertTrue(message = "currency scope does not match the limit type")
+  public boolean hasValidScope() {
+    return type == null || type.currencyScoped() == (currency != null);
+  }
+
+  @JsonIgnore
+  @AssertTrue(message = "limit must be exactly representable")
+  public boolean hasExactValue() {
+    if (value == null || value < 0) {
+      return true;
+    }
+    try {
+      SafeRedisNumber.requireNonNegative(value, "limit");
+      return true;
+    } catch (IllegalArgumentException exception) {
+      return false;
+    }
+  }
+
+  LimitOverrideTarget target() {
+    return new LimitOverrideTarget(type, currency);
+  }
+}


## `feat(api): mutate user limit overrides`

diff --git a/src/main/java/com/sportsbook/risk/api/LimitController.java b/src/main/java/com/sportsbook/risk/api/LimitController.java
index 89328a1..0293a98 100644
--- a/src/main/java/com/sportsbook/risk/api/LimitController.java
+++ b/src/main/java/com/sportsbook/risk/api/LimitController.java
@@ -6,15 +6,21 @@ import com.sportsbook.risk.counter.LimitType;
 import com.sportsbook.risk.limit.LimitOverrideField;
 import com.sportsbook.risk.limit.LimitOverrideStore;
 import com.sportsbook.risk.limit.LimitResolver;
+import jakarta.validation.Valid;
 import java.util.ArrayList;
 import java.util.OptionalLong;
 import java.util.UUID;
+import org.springframework.http.ResponseEntity;
+import org.springframework.web.bind.annotation.DeleteMapping;
 import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PatchMapping;
 import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.RequestBody;
 import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RequestParam;
 import org.springframework.web.bind.annotation.RestController;
 
-/** Admin-owned effective-limit operations. */
+/** Admin-owned effective-limit and override operations. */
 @RestController
 @RequestMapping("/internal/v1/risk/limits")
 public class LimitController {
@@ -42,6 +48,28 @@ public class LimitController {
     return new UserLimitsResponse(typedUserId, entries);
   }
 
+  @PatchMapping("/{userId}")
+  public ResponseEntity<Void> update(
+      @PathVariable UUID userId, @Valid @RequestBody LimitUpdateRequest request) {
+    overrides.set(UserId.of(userId), request.target().field(), request.value());
+    return ResponseEntity.noContent().build();
+  }
+
+  @DeleteMapping("/{userId}/{type}")
+  public ResponseEntity<Void> clear(
+      @PathVariable UUID userId,
+      @PathVariable LimitType type,
+      @RequestParam(required = false) Currency currency) {
+    LimitOverrideTarget target;
+    try {
+      target = new LimitOverrideTarget(type, currency);
+    } catch (IllegalArgumentException exception) {
+      throw RiskApiException.validation(exception.getMessage());
+    }
+    overrides.clear(UserId.of(userId), target.field());
+    return ResponseEntity.noContent().build();
+  }
+
   private UserLimitsResponse.Entry entry(UserId userId, LimitOverrideTarget target) {
     LimitOverrideField field = target.field();
     OptionalLong override = overrides.find(userId, field);


## `test(api): verify limit override mutations`

diff --git a/src/test/java/com/sportsbook/risk/api/LimitControllerMutationTest.java b/src/test/java/com/sportsbook/risk/api/LimitControllerMutationTest.java
new file mode 100644
index 0000000..362a412
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/LimitControllerMutationTest.java
@@ -0,0 +1,95 @@
+package com.sportsbook.risk.api;
+
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.patch;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.limit.LimitOverrideField;
+import com.sportsbook.risk.limit.LimitOverrideStore;
+import com.sportsbook.risk.limit.LimitResolver;
+import com.sportsbook.risk.policy.RiskLimitProperties;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.ArgumentMatchers;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+@ExtendWith(MockitoExtension.class)
+class LimitControllerMutationTest {
+  private static final UUID USER_VALUE = UUID.fromString("00000000-0000-0000-0000-000000000001");
+  private static final UserId USER = UserId.of(USER_VALUE);
+
+  @Mock private LimitOverrideStore overrides;
+  private final ObjectMapper json = new ObjectMapper();
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    var resolver = new LimitResolver(new RiskLimitProperties(null, null, null, null, 0), overrides);
+    mvc =
+        MockMvcBuilders.standaloneSetup(new LimitController(overrides, resolver))
+            .setControllerAdvice(new RestExceptionHandler())
+            .build();
+  }
+
+  @Test
+  void setsAndClearsTypedOverrides() throws Exception {
+    var update = new LimitUpdateRequest(LimitType.STAKE_DAILY, Currency.KRW, 750L);
+    mvc.perform(
+            patch("/internal/v1/risk/limits/" + USER_VALUE)
+                .contentType("application/json")
+                .content(json.writeValueAsBytes(update)))
+        .andExpect(status().isNoContent());
+    verify(overrides)
+        .set(USER, LimitOverrideField.monetary(LimitType.STAKE_DAILY, Currency.KRW), 750L);
+
+    mvc.perform(delete("/internal/v1/risk/limits/" + USER_VALUE + "/SELECTIONS_PER_MINUTE"))
+        .andExpect(status().isNoContent());
+    verify(overrides).clear(USER, LimitOverrideField.selections());
+  }
+
+  @Test
+  void rejectsScopeMismatchAndUnsafeValues() throws Exception {
+    var wrongScope = new LimitUpdateRequest(LimitType.STAKE_WEEKLY, null, 100L);
+    mvc.perform(
+            patch("/internal/v1/risk/limits/" + USER_VALUE)
+                .contentType("application/json")
+                .content(json.writeValueAsBytes(wrongScope)))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    String unsafe =
+        "{\"type\":\"STAKE_DAILY\",\"currency\":\"KRW\",\"value\":"
+            + (SafeRedisNumber.MAX_VALUE + 1)
+            + "}";
+    mvc.perform(
+            patch("/internal/v1/risk/limits/" + USER_VALUE)
+                .contentType("application/json")
+                .content(unsafe))
+        .andExpect(status().isBadRequest());
+    verify(overrides, never())
+        .set(ArgumentMatchers.any(), ArgumentMatchers.any(), ArgumentMatchers.anyLong());
+  }
+
+  @Test
+  void rejectsDeleteTargetsWithTheWrongCurrencyScope() throws Exception {
+    mvc.perform(delete("/internal/v1/risk/limits/" + USER_VALUE + "/STAKE_DAILY"))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    mvc.perform(
+            delete("/internal/v1/risk/limits/" + USER_VALUE + "/SELECTIONS_PER_MINUTE")
+                .queryParam("currency", "KRW"))
+        .andExpect(status().isBadRequest());
+  }
+}


## `feat(api): shape reservation decisions`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskReservationResponse.java b/src/main/java/com/sportsbook/risk/api/RiskReservationResponse.java
new file mode 100644
index 0000000..5e29a1f
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskReservationResponse.java
@@ -0,0 +1,34 @@
+package com.sportsbook.risk.api;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import com.sportsbook.risk.reservation.ReservationDecision;
+import com.sportsbook.risk.reservation.ReservationState;
+import java.time.Instant;
+import java.util.List;
+
+/** Admission lease or durable decline returned to betting-service. */
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record RiskReservationResponse(
+    boolean approved,
+    boolean replayed,
+    String rejectionReason,
+    List<RiskCheckResponse.PatternFlag> patterns,
+    ReservationState reservationState,
+    Instant expiresAt,
+    String reservationToken) {
+
+  public RiskReservationResponse {
+    patterns = List.copyOf(patterns);
+  }
+
+  static RiskReservationResponse from(ReservationDecision decision) {
+    return new RiskReservationResponse(
+        decision.approved(),
+        decision.replayed(),
+        decision.rejection(),
+        decision.patterns().stream().map(RiskCheckResponse.PatternFlag::from).toList(),
+        decision.state(),
+        decision.expiresAt(),
+        decision.token());
+  }
+}


## `feat(api): expose reservation admission`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskReservationController.java b/src/main/java/com/sportsbook/risk/api/RiskReservationController.java
new file mode 100644
index 0000000..2fab0b4
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskReservationController.java
@@ -0,0 +1,44 @@
+package com.sportsbook.risk.api;
+
+import com.sportsbook.risk.reservation.ReservationDecision;
+import com.sportsbook.risk.reservation.RiskReservationService;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import jakarta.validation.Valid;
+import java.time.Clock;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Betting-owned atomic admission API. */
+@RestController
+@RequestMapping("/internal/v1/risk/reservations")
+public class RiskReservationController {
+  private final Operations operations;
+  private final Clock clock;
+
+  @Autowired
+  public RiskReservationController(RiskReservationService service, Clock clock) {
+    this(service::reserve, clock);
+  }
+
+  RiskReservationController(Operations operations, Clock clock) {
+    this.operations = operations;
+    this.clock = clock;
+  }
+
+  @PostMapping
+  public RiskReservationResponse reserve(@Valid @RequestBody RiskCheckRequest request) {
+    ReservationDecision decision = operations.reserve(request.toCommand(clock.instant()));
+    if (decision.status() == ReservationDecision.Status.CONFLICT) {
+      throw RiskApiException.duplicate(request.betId());
+    }
+    return RiskReservationResponse.from(decision);
+  }
+
+  @FunctionalInterface
+  interface Operations {
+    ReservationDecision reserve(RiskCheckCommand command);
+  }
+}


## `test(api): verify reservation admission contracts`

diff --git a/src/test/java/com/sportsbook/risk/api/RiskReservationAdmissionTest.java b/src/test/java/com/sportsbook/risk/api/RiskReservationAdmissionTest.java
new file mode 100644
index 0000000..cb5e600
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/RiskReservationAdmissionTest.java
@@ -0,0 +1,98 @@
+package com.sportsbook.risk.api;
+
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.databind.SerializationFeature;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.reservation.ReservationDecision;
+import com.sportsbook.risk.reservation.ReservationState;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+@ExtendWith(MockitoExtension.class)
+class RiskReservationAdmissionTest {
+  private static final Instant NOW = Instant.parse("2026-08-21T10:00:00Z");
+  private static final Instant EXPIRES = NOW.plusSeconds(120);
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+  private static final BetId BET =
+      BetId.of(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+  private static final SelectionId SELECTION =
+      SelectionId.of(UUID.fromString("00000000-0000-0000-0000-000000000003"));
+
+  @Mock private RiskReservationController.Operations operations;
+  private final ObjectMapper json =
+      new ObjectMapper()
+          .findAndRegisterModules()
+          .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    var controller = new RiskReservationController(operations, Clock.fixed(NOW, ZoneOffset.UTC));
+    mvc =
+        MockMvcBuilders.standaloneSetup(controller)
+            .setControllerAdvice(new RestExceptionHandler())
+            .setMessageConverters(new MappingJackson2HttpMessageConverter(json))
+            .build();
+  }
+
+  @Test
+  void returnsTheOpaqueLeaseToken() throws Exception {
+    when(operations.reserve(any()))
+        .thenReturn(
+            ReservationDecision.approved(
+                ReservationState.RESERVED, EXPIRES, "opaque-token", false, List.of()));
+    perform()
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.approved").value(true))
+        .andExpect(jsonPath("$.expiresAt").value(EXPIRES.toString()))
+        .andExpect(jsonPath("$.reservationToken").value("opaque-token"));
+  }
+
+  @Test
+  void returnsDurableDeclinesWithoutAToken() throws Exception {
+    when(operations.reserve(any()))
+        .thenReturn(ReservationDecision.rejected("STAKE_DAILY_LIMIT_EXCEEDED", true, List.of()));
+    perform()
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.approved").value(false))
+        .andExpect(jsonPath("$.replayed").value(true))
+        .andExpect(jsonPath("$.reservationToken").doesNotExist());
+  }
+
+  @Test
+  void mapsChangedPayloadsToDuplicateBet() throws Exception {
+    when(operations.reserve(any())).thenReturn(ReservationDecision.conflict());
+    perform()
+        .andExpect(status().isConflict())
+        .andExpect(jsonPath("$.errorCode").value("DUPLICATE_BET"));
+  }
+
+  private org.springframework.test.web.servlet.ResultActions perform() throws Exception {
+    var request = new RiskCheckRequest(USER, BET, Money.krw(100), List.of(SELECTION));
+    return mvc.perform(
+        post("/internal/v1/risk/reservations")
+            .contentType("application/json")
+            .content(json.writeValueAsBytes(request)));
+  }
+}


