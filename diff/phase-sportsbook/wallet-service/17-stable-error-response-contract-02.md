## `feat(errors): map direct wallet failures`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
index 871f2b6..b32f415 100644
--- a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
+++ b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
@@ -1,7 +1,12 @@
 package com.sportsbook.wallet.web;
 
+import com.sportsbook.wallet.domain.error.AccountNotFoundException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import com.sportsbook.wallet.domain.error.WalletAccessDeniedException;
+import com.sportsbook.wallet.domain.error.WalletAdjustmentNotFoundException;
 import com.sportsbook.wallet.domain.error.WalletBusyException;
+import com.sportsbook.wallet.domain.error.WalletOperationNotFoundException;
 import com.sportsbook.wallet.domain.error.WalletRejectedException;
 import jakarta.servlet.http.HttpServletRequest;
 import java.net.URI;
@@ -42,6 +47,50 @@ public class WalletExceptionHandler {
         .body(problem);
   }
 
+  @ExceptionHandler(AccountNotFoundException.class)
+  ProblemDetail accountNotFound(AccountNotFoundException exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.ACCOUNT_NOT_FOUND, "The requested wallet account does not exist"),
+        request);
+  }
+
+  @ExceptionHandler(WalletOperationNotFoundException.class)
+  ProblemDetail operationNotFound(
+      WalletOperationNotFoundException exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.OPERATION_NOT_FOUND, "The requested wallet operation does not exist"),
+        request);
+  }
+
+  @ExceptionHandler(WalletAdjustmentNotFoundException.class)
+  ProblemDetail adjustmentNotFound(
+      WalletAdjustmentNotFoundException exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.ADJUSTMENT_NOT_FOUND, "The requested wallet adjustment does not exist"),
+        request);
+  }
+
+  @ExceptionHandler(CurrencyMismatchException.class)
+  ProblemDetail currencyMismatch(CurrencyMismatchException exception, HttpServletRequest request) {
+    ProblemDetail problem =
+        WalletProblems.from(
+            WalletError.CURRENCY_MISMATCH,
+            "The requested currency does not match the wallet account");
+    problem.setProperty("expectedCurrency", exception.expected());
+    return atRequest(problem, request);
+  }
+
+  @ExceptionHandler(WalletAccessDeniedException.class)
+  ProblemDetail accessDenied(WalletAccessDeniedException exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.ACCESS_DENIED, "Authenticated caller cannot perform this wallet operation"),
+        request);
+  }
+
   private ProblemDetail atRequest(ProblemDetail problem, HttpServletRequest request) {
     problem.setInstance(URI.create(request.getRequestURI()));
     return problem;


## `test(errors): map currency and access failures`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerAccessTest.java b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerAccessTest.java
new file mode 100644
index 0000000..ace7a2b
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerAccessTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.wallet.web;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.WalletAccessDeniedException;
+import java.net.URI;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.ProblemDetail;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class WalletExceptionHandlerAccessTest {
+  private final WalletExceptionHandler handler = new WalletExceptionHandler();
+
+  @Test
+  void mapsCurrencyMismatchWithoutReflectingTheActualCurrency() {
+    CurrencyMismatchException failure = new CurrencyMismatchException(Currency.KRW, Currency.USD);
+
+    ProblemDetail problem = handler.currencyMismatch(failure, request("/wallet/account"));
+
+    assertProblem(
+        problem,
+        WalletError.CURRENCY_MISMATCH,
+        "The requested currency does not match the wallet account",
+        "/wallet/account");
+    assertThat(problem.getProperties())
+        .containsOnlyKeys("errorCode", "expectedCurrency")
+        .containsEntry("expectedCurrency", Currency.KRW);
+    assertThat(problem.toString()).doesNotContain(Currency.USD.name(), failure.getMessage());
+  }
+
+  @Test
+  void mapsSemanticAccessDenialsWithoutReflectingCapabilities() {
+    WalletAccessDeniedException failure =
+        new WalletAccessDeniedException(WalletCaller.ADMIN, "secret credit source and reason");
+
+    ProblemDetail problem = handler.accessDenied(failure, request("/wallet/credit"));
+
+    assertProblem(
+        problem,
+        WalletError.ACCESS_DENIED,
+        "Authenticated caller cannot perform this wallet operation",
+        "/wallet/credit");
+    assertThat(problem.getProperties()).containsOnlyKeys("errorCode");
+    assertThat(problem.toString())
+        .doesNotContain(WalletCaller.ADMIN.wireName(), failure.capability(), failure.getMessage());
+  }
+
+  private void assertProblem(
+      ProblemDetail problem, WalletError error, String detail, String instance) {
+    assertThat(problem.getStatus()).isEqualTo(error.httpStatus());
+    assertThat(problem.getType()).isEqualTo(error.type());
+    assertThat(problem.getTitle()).isEqualTo(error.title());
+    assertThat(problem.getDetail()).isEqualTo(detail);
+    assertThat(problem.getInstance()).isEqualTo(URI.create(instance));
+    assertThat(problem.getProperties()).containsEntry("errorCode", error.errorCode());
+  }
+
+  private MockHttpServletRequest request(String path) {
+    return new MockHttpServletRequest("GET", path);
+  }
+}


## `feat(errors): map malformed and unexpected requests`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
index b32f415..24eb0af 100644
--- a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
+++ b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
@@ -13,8 +13,14 @@ import java.net.URI;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.ProblemDetail;
 import org.springframework.http.ResponseEntity;
+import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.web.HttpMediaTypeNotSupportedException;
+import org.springframework.web.bind.MethodArgumentNotValidException;
+import org.springframework.web.bind.ServletRequestBindingException;
 import org.springframework.web.bind.annotation.ExceptionHandler;
 import org.springframework.web.bind.annotation.RestControllerAdvice;
+import org.springframework.web.method.annotation.HandlerMethodValidationException;
+import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
 
 /** Maps durable and retryable wallet failures without exposing request credentials. */
 @RestControllerAdvice
@@ -91,6 +97,30 @@ public class WalletExceptionHandler {
         request);
   }
 
+  @ExceptionHandler({
+    MethodArgumentNotValidException.class,
+    HandlerMethodValidationException.class,
+    HttpMessageNotReadableException.class,
+    HttpMediaTypeNotSupportedException.class,
+    MethodArgumentTypeMismatchException.class,
+    ServletRequestBindingException.class,
+    IllegalArgumentException.class
+  })
+  ProblemDetail invalidRequest(Exception exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(
+            WalletError.INVALID_REQUEST,
+            "Wallet request is malformed or violates validation constraints"),
+        request);
+  }
+
+  @ExceptionHandler(Exception.class)
+  ProblemDetail internalError(Exception exception, HttpServletRequest request) {
+    return atRequest(
+        WalletProblems.from(WalletError.INTERNAL_ERROR, "Wallet request could not be completed"),
+        request);
+  }
+
   private ProblemDetail atRequest(ProblemDetail problem, HttpServletRequest request) {
     problem.setInstance(URI.create(request.getRequestURI()));
     return problem;


## `test(errors): map malformed wallet requests`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerMalformedMvcTest.java b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerMalformedMvcTest.java
new file mode 100644
index 0000000..81ba9d4
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerMalformedMvcTest.java
@@ -0,0 +1,93 @@
+package com.sportsbook.wallet.web;
+
+import static org.hamcrest.Matchers.aMapWithSize;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import jakarta.validation.Valid;
+import jakarta.validation.constraints.NotBlank;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.ResultActions;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.PathVariable;
+import org.springframework.web.bind.annotation.PostMapping;
+import org.springframework.web.bind.annotation.RequestBody;
+import org.springframework.web.bind.annotation.RequestHeader;
+import org.springframework.web.bind.annotation.RequestParam;
+import org.springframework.web.bind.annotation.RestController;
+
+class WalletExceptionHandlerMalformedMvcTest {
+  private final MockMvc mvc =
+      MockMvcBuilders.standaloneSetup(new ProbeController())
+          .setControllerAdvice(new WalletExceptionHandler())
+          .build();
+
+  @Test
+  void mapsValidationAndUnreadableJsonToFixedProblems() throws Exception {
+    assertInvalid(
+        mvc.perform(
+            post("/probe/body?trace=secret-query")
+                .contentType(MediaType.APPLICATION_JSON)
+                .header("X-Internal-Api-Key", "secret-header")
+                .content("{\"value\":\"\"}")),
+        "/probe/body");
+    assertInvalid(
+        mvc.perform(
+            post("/probe/body")
+                .contentType(MediaType.APPLICATION_JSON)
+                .content("{\"value\":\"secret-body\"")),
+        "/probe/body");
+  }
+
+  @Test
+  void mapsTypeHeaderAndMediaFailuresToFixedProblems() throws Exception {
+    assertInvalid(mvc.perform(get("/probe/not-a-uuid")), "/probe/not-a-uuid");
+    assertInvalid(mvc.perform(get("/probe/header")), "/probe/header");
+    assertInvalid(mvc.perform(get("/probe/validated?value=")), "/probe/validated");
+    assertInvalid(
+        mvc.perform(post("/probe/body").contentType(MediaType.TEXT_PLAIN).content("secret-body")),
+        "/probe/body");
+  }
+
+  private void assertInvalid(ResultActions result, String instance) throws Exception {
+    result
+        .andExpect(status().isBadRequest())
+        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$", aMapWithSize(6)))
+        .andExpect(jsonPath("$.status").value(400))
+        .andExpect(jsonPath("$.type").value(WalletError.INVALID_REQUEST.type().toString()))
+        .andExpect(jsonPath("$.title").value(WalletError.INVALID_REQUEST.title()))
+        .andExpect(
+            jsonPath("$.detail")
+                .value("Wallet request is malformed or violates validation constraints"))
+        .andExpect(jsonPath("$.instance").value(instance))
+        .andExpect(jsonPath("$.errorCode").value(WalletError.INVALID_REQUEST.errorCode()))
+        .andExpect(
+            content()
+                .string(org.hamcrest.Matchers.not(org.hamcrest.Matchers.containsString("secret"))));
+  }
+
+  @RestController
+  static class ProbeController {
+    @PostMapping(path = "/probe/body", consumes = MediaType.APPLICATION_JSON_VALUE)
+    void body(@Valid @RequestBody ProbeBody body) {}
+
+    @GetMapping("/probe/{id}")
+    void type(@PathVariable("id") UUID id) {}
+
+    @GetMapping("/probe/header")
+    void header(@RequestHeader("Idempotency-Key") String key) {}
+
+    @GetMapping("/probe/validated")
+    void validated(@RequestParam @NotBlank String value) {}
+  }
+
+  record ProbeBody(@NotBlank String value) {}
+}


## `test(errors): contain unexpected wallet failures`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerTerminalTest.java b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerTerminalTest.java
new file mode 100644
index 0000000..3a44a62
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerTerminalTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.wallet.web;
+
+import static org.hamcrest.Matchers.aMapWithSize;
+import static org.hamcrest.Matchers.containsString;
+import static org.hamcrest.Matchers.not;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.ResultActions;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+class WalletExceptionHandlerTerminalTest {
+  private final MockMvc mvc =
+      MockMvcBuilders.standaloneSetup(new ProbeController())
+          .setControllerAdvice(new WalletExceptionHandler())
+          .build();
+
+  @Test
+  void mapsIllegalArgumentsToFixedInvalidRequests() throws Exception {
+    assertProblem(
+        mvc.perform(
+            get("/probe/illegal?trace=secret-query").header("X-Diagnostic", "secret-header")),
+        WalletError.INVALID_REQUEST,
+        "Wallet request is malformed or violates validation constraints",
+        "/probe/illegal");
+  }
+
+  @Test
+  void mapsUnexpectedFailuresToSafeInternalErrors() throws Exception {
+    assertProblem(
+        mvc.perform(
+            get("/probe/internal?trace=secret-query").header("X-Diagnostic", "secret-header")),
+        WalletError.INTERNAL_ERROR,
+        "Wallet request could not be completed",
+        "/probe/internal");
+  }
+
+  private void assertProblem(
+      ResultActions result, WalletError error, String detail, String instance) throws Exception {
+    result
+        .andExpect(status().is(error.httpStatus()))
+        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$", aMapWithSize(6)))
+        .andExpect(jsonPath("$.status").value(error.httpStatus()))
+        .andExpect(jsonPath("$.type").value(error.type().toString()))
+        .andExpect(jsonPath("$.title").value(error.title()))
+        .andExpect(jsonPath("$.detail").value(detail))
+        .andExpect(jsonPath("$.instance").value(instance))
+        .andExpect(jsonPath("$.errorCode").value(error.errorCode()))
+        .andExpect(content().string(not(containsString("secret"))));
+  }
+
+  @RestController
+  static class ProbeController {
+    @GetMapping("/probe/illegal")
+    void illegal() {
+      throw new IllegalArgumentException("secret invalid detail");
+    }
+
+    @GetMapping("/probe/internal")
+    void internal() {
+      throw new IllegalStateException("secret internal detail");
+    }
+  }
+}


## `feat(errors): map retryable database outages`

diff --git a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
index 24eb0af..b79b340 100644
--- a/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
+++ b/src/main/java/com/sportsbook/wallet/web/WalletExceptionHandler.java
@@ -8,12 +8,15 @@ import com.sportsbook.wallet.domain.error.WalletAdjustmentNotFoundException;
 import com.sportsbook.wallet.domain.error.WalletBusyException;
 import com.sportsbook.wallet.domain.error.WalletOperationNotFoundException;
 import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import com.sportsbook.wallet.persistence.PostgresFailureTranslator;
 import jakarta.servlet.http.HttpServletRequest;
 import java.net.URI;
+import org.springframework.dao.DataAccessException;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.ProblemDetail;
 import org.springframework.http.ResponseEntity;
 import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.transaction.TransactionException;
 import org.springframework.web.HttpMediaTypeNotSupportedException;
 import org.springframework.web.bind.MethodArgumentNotValidException;
 import org.springframework.web.bind.ServletRequestBindingException;
@@ -43,14 +46,7 @@ public class WalletExceptionHandler {
 
   @ExceptionHandler(WalletBusyException.class)
   ResponseEntity<ProblemDetail> busy(WalletBusyException exception, HttpServletRequest request) {
-    ProblemDetail problem =
-        atRequest(
-            WalletProblems.from(
-                WalletError.WALLET_BUSY, "Retry the wallet request after one second"),
-            request);
-    return ResponseEntity.status(WalletError.WALLET_BUSY.httpStatus())
-        .header(HttpHeaders.RETRY_AFTER, "1")
-        .body(problem);
+    return busyResponse(request);
   }
 
   @ExceptionHandler(AccountNotFoundException.class)
@@ -121,6 +117,27 @@ public class WalletExceptionHandler {
         request);
   }
 
+  @ExceptionHandler({DataAccessException.class, TransactionException.class})
+  ResponseEntity<ProblemDetail> databaseFailure(
+      RuntimeException exception, HttpServletRequest request) {
+    if (PostgresFailureTranslator.isRetryable(exception)) {
+      return busyResponse(request);
+    }
+    return ResponseEntity.status(WalletError.INTERNAL_ERROR.httpStatus())
+        .body(internalError(exception, request));
+  }
+
+  private ResponseEntity<ProblemDetail> busyResponse(HttpServletRequest request) {
+    ProblemDetail problem =
+        atRequest(
+            WalletProblems.from(
+                WalletError.WALLET_BUSY, "Retry the wallet request after one second"),
+            request);
+    return ResponseEntity.status(WalletError.WALLET_BUSY.httpStatus())
+        .header(HttpHeaders.RETRY_AFTER, "1")
+        .body(problem);
+  }
+
   private ProblemDetail atRequest(ProblemDetail problem, HttpServletRequest request) {
     problem.setInstance(URI.create(request.getRequestURI()));
     return problem;


## `test(errors): return stable database outage responses`

diff --git a/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerDatabaseFailureTest.java b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerDatabaseFailureTest.java
new file mode 100644
index 0000000..c9b6933
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/web/WalletExceptionHandlerDatabaseFailureTest.java
@@ -0,0 +1,98 @@
+package com.sportsbook.wallet.web;
+
+import static org.hamcrest.Matchers.aMapWithSize;
+import static org.hamcrest.Matchers.containsString;
+import static org.hamcrest.Matchers.not;
+import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
+import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
+
+import java.sql.SQLException;
+import java.sql.SQLTransientConnectionException;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.dao.DataAccessResourceFailureException;
+import org.springframework.dao.DataIntegrityViolationException;
+import org.springframework.http.HttpHeaders;
+import org.springframework.http.MediaType;
+import org.springframework.test.web.servlet.MockMvc;
+import org.springframework.test.web.servlet.ResultActions;
+import org.springframework.test.web.servlet.setup.MockMvcBuilders;
+import org.springframework.transaction.CannotCreateTransactionException;
+import org.springframework.web.bind.annotation.GetMapping;
+import org.springframework.web.bind.annotation.RestController;
+
+class WalletExceptionHandlerDatabaseFailureTest {
+  private final MockMvc mvc =
+      MockMvcBuilders.standaloneSetup(new ProbeController())
+          .setControllerAdvice(new WalletExceptionHandler())
+          .build();
+
+  @Test
+  void mapsRetryableDatabaseOutagesToBusyResponses() throws Exception {
+    for (String path : List.of("/probe/pool", "/probe/connection")) {
+      assertProblem(
+              mvc.perform(get(path + "?trace=secret-query").header("X-Diagnostic", "secret")),
+              WalletError.WALLET_BUSY,
+              "Retry the wallet request after one second",
+              path)
+          .andExpect(header().string(HttpHeaders.RETRY_AFTER, "1"));
+    }
+  }
+
+  @Test
+  void keepsPermanentDatabaseDefectsInternal() throws Exception {
+    for (String path : List.of("/probe/constraint", "/probe/syntax")) {
+      assertProblem(
+              mvc.perform(get(path + "?trace=secret-query").header("X-Diagnostic", "secret")),
+              WalletError.INTERNAL_ERROR,
+              "Wallet request could not be completed",
+              path)
+          .andExpect(header().doesNotExist(HttpHeaders.RETRY_AFTER));
+    }
+  }
+
+  private ResultActions assertProblem(
+      ResultActions result, WalletError error, String detail, String instance) throws Exception {
+    return result
+        .andExpect(status().is(error.httpStatus()))
+        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
+        .andExpect(jsonPath("$", aMapWithSize(6)))
+        .andExpect(jsonPath("$.status").value(error.httpStatus()))
+        .andExpect(jsonPath("$.type").value(error.type().toString()))
+        .andExpect(jsonPath("$.title").value(error.title()))
+        .andExpect(jsonPath("$.detail").value(detail))
+        .andExpect(jsonPath("$.instance").value(instance))
+        .andExpect(jsonPath("$.errorCode").value(error.errorCode()))
+        .andExpect(content().string(not(containsString("secret"))));
+  }
+
+  @RestController
+  static class ProbeController {
+    @GetMapping("/probe/pool")
+    void pool() {
+      throw new CannotCreateTransactionException(
+          "secret transaction detail", new SQLTransientConnectionException("secret pool detail"));
+    }
+
+    @GetMapping("/probe/connection")
+    void connection() {
+      throw new DataAccessResourceFailureException(
+          "secret connection detail", new SQLException("secret SQL detail", "08006"));
+    }
+
+    @GetMapping("/probe/constraint")
+    void constraint() {
+      throw new DataIntegrityViolationException(
+          "secret duplicate detail", new SQLException("secret constraint detail", "23505"));
+    }
+
+    @GetMapping("/probe/syntax")
+    void syntax() {
+      throw new DataIntegrityViolationException(
+          "secret syntax detail", new SQLException("secret SQL detail", "42601"));
+    }
+  }
+}
