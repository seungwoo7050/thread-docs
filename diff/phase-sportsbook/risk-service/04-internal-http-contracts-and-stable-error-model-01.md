# 내부 HTTP 계약과 안정된 오류 모델

## `feat(api): validate risk check payloads`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskCheckRequest.java b/src/main/java/com/sportsbook/risk/api/RiskCheckRequest.java
new file mode 100644
index 0000000..1d90efc
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskCheckRequest.java
@@ -0,0 +1,52 @@
+package com.sportsbook.risk.api;
+
+import com.fasterxml.jackson.annotation.JsonIgnore;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import jakarta.validation.constraints.AssertTrue;
+import jakarta.validation.constraints.NotEmpty;
+import jakarta.validation.constraints.NotNull;
+import jakarta.validation.constraints.Size;
+import java.time.Instant;
+import java.util.HashSet;
+import java.util.List;
+
+public record RiskCheckRequest(
+    @NotNull UserId userId,
+    @NotNull BetId betId,
+    @NotNull Money stake,
+    @NotEmpty @Size(max = RiskCheckCommand.MAX_SELECTIONS)
+        List<@NotNull SelectionId> selectionIds) {
+
+  public RiskCheckRequest {
+    selectionIds = selectionIds == null ? null : List.copyOf(selectionIds);
+  }
+
+  @JsonIgnore
+  @AssertTrue(message = "stake amount must be positive and exactly representable")
+  public boolean hasValidStakeAmount() {
+    if (stake == null) {
+      return true;
+    }
+    try {
+      SafeRedisNumber.requirePositive(stake.amount(), "stake.amount");
+      return true;
+    } catch (IllegalArgumentException exception) {
+      return false;
+    }
+  }
+
+  @JsonIgnore
+  @AssertTrue(message = "selectionIds must be unique")
+  public boolean hasUniqueSelections() {
+    return selectionIds == null || new HashSet<>(selectionIds).size() == selectionIds.size();
+  }
+
+  RiskCheckCommand toCommand(Instant now) {
+    return new RiskCheckCommand(userId, betId, stake, selectionIds, now);
+  }
+}


## `feat(api): shape risk check outcomes`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskCheckResponse.java b/src/main/java/com/sportsbook/risk/api/RiskCheckResponse.java
new file mode 100644
index 0000000..fc3be42
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskCheckResponse.java
@@ -0,0 +1,47 @@
+package com.sportsbook.risk.api;
+
+import com.fasterxml.jackson.annotation.JsonInclude;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.service.LimitRejection;
+import com.sportsbook.risk.service.RiskCheckOutcome;
+import java.util.List;
+
+@JsonInclude(JsonInclude.Include.NON_NULL)
+public record RiskCheckResponse(
+    boolean approved, String rejectionReason, LimitInfo limit, List<PatternFlag> patterns) {
+
+  public RiskCheckResponse {
+    patterns = List.copyOf(patterns);
+  }
+
+  static RiskCheckResponse from(RiskCheckOutcome outcome) {
+    LimitRejection rejection = outcome.rejection();
+    return new RiskCheckResponse(
+        outcome.approved(),
+        rejection == null ? null : rejection.reason(),
+        rejection == null ? null : LimitInfo.from(rejection),
+        outcome.patterns().stream().map(PatternFlag::from).toList());
+  }
+
+  @JsonInclude(JsonInclude.Include.NON_NULL)
+  public record LimitInfo(
+      LimitType type, Currency currency, long current, long limit, long requested) {
+    static LimitInfo from(LimitRejection rejection) {
+      return new LimitInfo(
+          rejection.type(),
+          rejection.currency(),
+          rejection.current(),
+          rejection.limit(),
+          rejection.requested());
+    }
+  }
+
+  public record PatternFlag(String rule, PatternAction action, String reason) {
+    static PatternFlag from(PatternMatch match) {
+      return new PatternFlag(match.rule(), match.action(), match.reason());
+    }
+  }
+}


## `test(api): verify risk payload contracts`

diff --git a/src/test/java/com/sportsbook/risk/api/RiskPayloadTest.java b/src/test/java/com/sportsbook/risk/api/RiskPayloadTest.java
new file mode 100644
index 0000000..0838e1f
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/RiskPayloadTest.java
@@ -0,0 +1,83 @@
+package com.sportsbook.risk.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.pattern.PatternMatch;
+import com.sportsbook.risk.policy.PatternAction;
+import com.sportsbook.risk.policy.SafeRedisNumber;
+import com.sportsbook.risk.service.LimitRejection;
+import com.sportsbook.risk.service.RiskCheckOutcome;
+import jakarta.validation.Validation;
+import jakarta.validation.Validator;
+import java.util.List;
+import java.util.UUID;
+import java.util.stream.IntStream;
+import org.junit.jupiter.api.Test;
+
+class RiskPayloadTest {
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+  private static final BetId BET =
+      BetId.of(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+  private static final SelectionId SELECTION =
+      SelectionId.of(UUID.fromString("00000000-0000-0000-0000-000000000003"));
+  private final ObjectMapper json = new ObjectMapper().findAndRegisterModules();
+  private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
+
+  @Test
+  void readsSharedIdentifiersFromTheirCanonicalJsonValues() throws Exception {
+    String body =
+        """
+        {"userId":"%s","betId":"%s","stake":{"amount":100,"currency":"KRW"},
+         "selectionIds":["%s"]}
+        """
+            .formatted(USER.value(), BET.value(), SELECTION.value());
+
+    RiskCheckRequest request = json.readValue(body, RiskCheckRequest.class);
+
+    assertThat(request)
+        .isEqualTo(new RiskCheckRequest(USER, BET, Money.krw(100), List.of(SELECTION)));
+  }
+
+  @Test
+  void rejectsUnsafeStakeAndInvalidSelectionSets() {
+    List<RiskCheckRequest> invalid =
+        List.of(
+            request(Money.krw(0), List.of(SELECTION)),
+            request(new Money(SafeRedisNumber.MAX_VALUE + 1, Currency.KRW), List.of(SELECTION)),
+            request(Money.krw(1), List.of()),
+            request(Money.krw(1), List.of(SELECTION, SELECTION)),
+            request(
+                Money.krw(1),
+                IntStream.range(0, 16)
+                    .mapToObj(index -> SelectionId.of(new UUID(0, index + 10)))
+                    .toList()));
+
+    assertThat(invalid).allSatisfy(value -> assertThat(validator.validate(value)).isNotEmpty());
+  }
+
+  @Test
+  void serializesLimitAndPatternDecisionsWithTypedFields() throws Exception {
+    var limit = RiskCheckOutcome.rejectedByLimit(LimitRejection.single(Currency.KRW, 1000, 1001));
+    var flag = new PatternMatch("RAPID_BETTING", PatternAction.SUSPECT, "threshold reached");
+
+    assertThat(json.writeValueAsString(RiskCheckResponse.from(limit)))
+        .contains("\"rejectionReason\":\"SINGLE_BET_MAX_EXCEEDED\"")
+        .contains("\"currency\":\"KRW\"")
+        .doesNotContain("\"type\"");
+    assertThat(RiskCheckResponse.from(RiskCheckOutcome.approved(List.of(flag))).patterns())
+        .containsExactly(
+            new RiskCheckResponse.PatternFlag(
+                "RAPID_BETTING", PatternAction.SUSPECT, "threshold reached"));
+  }
+
+  private static RiskCheckRequest request(Money stake, List<SelectionId> selections) {
+    return new RiskCheckRequest(USER, BET, stake, selections);
+  }
+}


## `feat(api): define reservation failures`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskApiException.java b/src/main/java/com/sportsbook/risk/api/RiskApiException.java
new file mode 100644
index 0000000..7e99453
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskApiException.java
@@ -0,0 +1,76 @@
+package com.sportsbook.risk.api;
+
+import static jakarta.servlet.http.HttpServletResponse.SC_CONFLICT;
+import static jakarta.servlet.http.HttpServletResponse.SC_NOT_FOUND;
+
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.error.ProblemDetail;
+import com.sportsbook.protocol.value.BetId;
+import java.net.URI;
+
+/** Stable lifecycle failures raised at the reservation HTTP boundary. */
+final class RiskApiException extends RuntimeException {
+  private final int status;
+  private final URI type;
+  private final String title;
+  private final String errorCode;
+
+  private RiskApiException(int status, URI type, String title, String errorCode, String detail) {
+    super(detail);
+    this.status = status;
+    this.type = type;
+    this.title = title;
+    this.errorCode = errorCode;
+  }
+
+  static RiskApiException duplicate(BetId betId) {
+    ErrorCode code = ErrorCode.DUPLICATE_BET;
+    return new RiskApiException(
+        code.httpStatus(),
+        code.type(),
+        code.title(),
+        code.name(),
+        "Conflicting reservation " + betId.value());
+  }
+
+  static RiskApiException validation(String detail) {
+    ErrorCode code = ErrorCode.VALIDATION_FAILED;
+    return new RiskApiException(code.httpStatus(), code.type(), code.title(), code.name(), detail);
+  }
+
+  static RiskApiException notFound(BetId betId) {
+    return custom(
+        SC_NOT_FOUND,
+        "not-found",
+        "Risk reservation not found",
+        "RISK_RESERVATION_NOT_FOUND",
+        betId);
+  }
+
+  static RiskApiException committed(BetId betId) {
+    return custom(
+        SC_CONFLICT,
+        "committed",
+        "Risk reservation already committed",
+        "RISK_RESERVATION_COMMITTED",
+        betId);
+  }
+
+  private static RiskApiException custom(
+      int status, String suffix, String title, String code, BetId betId) {
+    return new RiskApiException(
+        status,
+        URI.create("https://sportsbook/errors/risk-reservation-" + suffix),
+        title,
+        code,
+        "Reservation " + betId.value() + " cannot complete this transition");
+  }
+
+  int status() {
+    return status;
+  }
+
+  ProblemDetail problem() {
+    return new ProblemDetail(type, title, status, errorCode, getMessage(), null, null);
+  }
+}


## `feat(api): render stable problem details`

diff --git a/src/main/java/com/sportsbook/risk/api/RestExceptionHandler.java b/src/main/java/com/sportsbook/risk/api/RestExceptionHandler.java
new file mode 100644
index 0000000..7271388
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RestExceptionHandler.java
@@ -0,0 +1,66 @@
+package com.sportsbook.risk.api;
+
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.error.ProblemDetail;
+import jakarta.validation.ConstraintViolationException;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.http.MediaType;
+import org.springframework.http.ResponseEntity;
+import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.web.bind.MethodArgumentNotValidException;
+import org.springframework.web.bind.MissingRequestHeaderException;
+import org.springframework.web.bind.annotation.ExceptionHandler;
+import org.springframework.web.bind.annotation.RestControllerAdvice;
+import org.springframework.web.method.annotation.HandlerMethodValidationException;
+import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
+
+/** Renders every controller failure with the shared protocol problem shape. */
+@RestControllerAdvice
+public class RestExceptionHandler {
+  private static final Logger log = LoggerFactory.getLogger(RestExceptionHandler.class);
+
+  @ExceptionHandler(MethodArgumentNotValidException.class)
+  ResponseEntity<ProblemDetail> invalidBody(MethodArgumentNotValidException exception) {
+    String detail =
+        exception.getBindingResult().getAllErrors().stream()
+            .map(error -> error.getDefaultMessage())
+            .filter(message -> message != null && !message.isBlank())
+            .findFirst()
+            .orElse("Request validation failed");
+    return problem(ErrorCode.VALIDATION_FAILED, detail);
+  }
+
+  @ExceptionHandler({HandlerMethodValidationException.class, ConstraintViolationException.class})
+  ResponseEntity<ProblemDetail> invalidMethod(Exception exception) {
+    return problem(ErrorCode.VALIDATION_FAILED, "Request validation failed");
+  }
+
+  @ExceptionHandler({
+    HttpMessageNotReadableException.class,
+    MethodArgumentTypeMismatchException.class,
+    MissingRequestHeaderException.class
+  })
+  ResponseEntity<ProblemDetail> malformed(Exception exception) {
+    return problem(ErrorCode.VALIDATION_FAILED, "Request payload, path, or headers are malformed");
+  }
+
+  @ExceptionHandler(RiskApiException.class)
+  ResponseEntity<ProblemDetail> reservation(RiskApiException exception) {
+    return ResponseEntity.status(exception.status())
+        .contentType(MediaType.APPLICATION_PROBLEM_JSON)
+        .body(exception.problem());
+  }
+
+  @ExceptionHandler(Exception.class)
+  ResponseEntity<ProblemDetail> internal(Exception exception) {
+    log.error("Unhandled internal request failure", exception);
+    return problem(ErrorCode.INTERNAL_ERROR, "The request could not be completed");
+  }
+
+  private static ResponseEntity<ProblemDetail> problem(ErrorCode code, String detail) {
+    return ResponseEntity.status(code.httpStatus())
+        .contentType(MediaType.APPLICATION_PROBLEM_JSON)
+        .body(code.toProblemDetail(detail));
+  }
+}


## `test(api): verify opaque internal failures`

diff --git a/src/test/java/com/sportsbook/risk/api/RestExceptionHandlerTest.java b/src/test/java/com/sportsbook/risk/api/RestExceptionHandlerTest.java
new file mode 100644
index 0000000..e7f83f3
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/RestExceptionHandlerTest.java
@@ -0,0 +1,48 @@
+package com.sportsbook.risk.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+class RestExceptionHandlerTest {
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    mvc =
+        MockMvcBuilders.standaloneSetup(new FailingController())
+            .setControllerAdvice(new RestExceptionHandler())
+            .build();
+  }
+
+  @Test
+  void masksUnexpectedFailuresWithTheSharedProblemShape() throws Exception {
+    String body =
+        mvc.perform(get("/failure"))
+            .andExpect(status().isInternalServerError())
+            .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
+            .andExpect(jsonPath("$.errorCode").value("INTERNAL_ERROR"))
+            .andReturn()
+            .getResponse()
+            .getContentAsString();
+    assertThat(body).doesNotContain("private-detail");
+  }
+
+  @RestController
+  private static class FailingController {
+    @GetMapping("/failure")
+    void fail() {
+      throw new IllegalArgumentException("private-detail");
+    }
+  }
+}


## `feat(api): expose diagnostic risk checks`

diff --git a/src/main/java/com/sportsbook/risk/api/RiskCheckController.java b/src/main/java/com/sportsbook/risk/api/RiskCheckController.java
new file mode 100644
index 0000000..8b61b92
--- /dev/null
+++ b/src/main/java/com/sportsbook/risk/api/RiskCheckController.java
@@ -0,0 +1,40 @@
+package com.sportsbook.risk.api;
+
+import com.sportsbook.risk.service.RiskCheckCommand;
+import com.sportsbook.risk.service.RiskCheckOutcome;
+import com.sportsbook.risk.service.RiskCheckService;
+import jakarta.validation.Valid;
+import java.time.Clock;
+import java.util.Objects;
+import java.util.function.Function;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+/** Platform-owned diagnostic endpoint; reservation admission remains authoritative. */
+@RestController
+@RequestMapping("/internal/v1/risk")
+public class RiskCheckController {
+  private final Function<RiskCheckCommand, RiskCheckOutcome> check;
+  private final Clock clock;
+
+  @Autowired
+  public RiskCheckController(RiskCheckService service, Clock clock) {
+    this(
+        (Function<RiskCheckCommand, RiskCheckOutcome>)
+            Objects.requireNonNull(service, "service")::check,
+        clock);
+  }
+
+  RiskCheckController(Function<RiskCheckCommand, RiskCheckOutcome> check, Clock clock) {
+    this.check = Objects.requireNonNull(check, "check");
+    this.clock = Objects.requireNonNull(clock, "clock");
+  }
+
+  @PostMapping("/check")
+  public RiskCheckResponse check(@Valid @RequestBody RiskCheckRequest request) {
+    return RiskCheckResponse.from(check.apply(request.toCommand(clock.instant())));
+  }
+}


## `test(api): verify diagnostic risk checks`

diff --git a/src/test/java/com/sportsbook/risk/api/RiskCheckControllerTest.java b/src/test/java/com/sportsbook/risk/api/RiskCheckControllerTest.java
new file mode 100644
index 0000000..dd2cebb
--- /dev/null
+++ b/src/test/java/com/sportsbook/risk/api/RiskCheckControllerTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.risk.api;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.never;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.protocol.value.BetId;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.SelectionId;
+import com.sportsbook.protocol.value.UserId;
+import com.sportsbook.risk.counter.LimitType;
+import com.sportsbook.risk.service.LimitRejection;
+import com.sportsbook.risk.service.RiskCheckCommand;
+import com.sportsbook.risk.service.RiskCheckOutcome;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import java.util.function.Function;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.extension.ExtendWith;
+import org.mockito.ArgumentCaptor;
+import org.mockito.Mock;
+import org.mockito.junit.jupiter.MockitoExtension;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+
+@ExtendWith(MockitoExtension.class)
+class RiskCheckControllerTest {
+  private static final Instant NOW = Instant.parse("2026-08-21T10:00:00Z");
+  private static final UserId USER =
+      UserId.of(UUID.fromString("00000000-0000-0000-0000-000000000001"));
+  private static final BetId BET =
+      BetId.of(UUID.fromString("00000000-0000-0000-0000-000000000002"));
+  private static final SelectionId SELECTION =
+      SelectionId.of(UUID.fromString("00000000-0000-0000-0000-000000000003"));
+  @Mock private Function<RiskCheckCommand, RiskCheckOutcome> check;
+  private final ObjectMapper json = new ObjectMapper().findAndRegisterModules();
+  private MockMvc mvc;
+
+  @BeforeEach
+  void setUp() {
+    var controller = new RiskCheckController(check, Clock.fixed(NOW, ZoneOffset.UTC));
+    mvc =
+        MockMvcBuilders.standaloneSetup(controller)
+            .setControllerAdvice(new RestExceptionHandler())
+            .build();
+  }
+
+  @Test
+  void mapsTypedCandidateAtTheInjectedTime() throws Exception {
+    when(check.apply(any())).thenReturn(RiskCheckOutcome.approved(List.of()));
+    mvc.perform(post("/internal/v1/risk/check").contentType("application/json").content(body(100)))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.approved").value(true));
+
+    var command = ArgumentCaptor.forClass(RiskCheckCommand.class);
+    verify(check).apply(command.capture());
+    assertThat(command.getValue())
+        .isEqualTo(new RiskCheckCommand(USER, BET, Money.krw(100), List.of(SELECTION), NOW));
+  }
+
+  @Test
+  void rendersTheFirstLimitRejection() throws Exception {
+    var rejection = LimitRejection.rolling(LimitType.STAKE_DAILY, Currency.KRW, 900, 1000, 101);
+    when(check.apply(any())).thenReturn(RiskCheckOutcome.rejectedByLimit(rejection));
+
+    mvc.perform(post("/internal/v1/risk/check").contentType("application/json").content(body(101)))
+        .andExpect(status().isOk())
+        .andExpect(jsonPath("$.rejectionReason").value("STAKE_DAILY_LIMIT_EXCEEDED"))
+        .andExpect(jsonPath("$.limit.type").value("STAKE_DAILY"));
+  }
+
+  @Test
+  void rejectsDuplicateSelectionsBeforeCallingTheService() throws Exception {
+    var request = new RiskCheckRequest(USER, BET, Money.krw(100), List.of(SELECTION, SELECTION));
+    mvc.perform(
+            post("/internal/v1/risk/check")
+                .contentType("application/json")
+                .content(json.writeValueAsBytes(request)))
+        .andExpect(status().isBadRequest())
+        .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"));
+    verify(check, never()).apply(any());
+  }
+
+  private byte[] body(long amount) throws Exception {
+    return json.writeValueAsBytes(
+        new RiskCheckRequest(USER, BET, Money.krw(amount), List.of(SELECTION)));
+  }
+}


