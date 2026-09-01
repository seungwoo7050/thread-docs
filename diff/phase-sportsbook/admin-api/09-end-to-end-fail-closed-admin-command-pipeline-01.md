# 실패 폐쇄 관리 명령의 종단간 처리 파이프라인

## `feat(security): enforce authenticated admin requests`

diff --git a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
new file mode 100644
index 0000000..f11885e
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
@@ -0,0 +1,85 @@
+package com.sportsbook.admin.security;
+
+import com.sportsbook.admin.error.Rfc7807Writer;
+import java.util.List;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.http.HttpStatus;
+import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
+import org.springframework.security.config.annotation.web.builders.HttpSecurity;
+import org.springframework.security.config.http.SessionCreationPolicy;
+import org.springframework.security.core.GrantedAuthority;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
+import org.springframework.security.web.SecurityFilterChain;
+
+@Configuration(proxyBeanMethods = false)
+@EnableMethodSecurity
+class SecurityConfig {
+
+  @Bean
+  SecurityFilterChain adminSecurityFilterChain(HttpSecurity http, Rfc7807Writer problems)
+      throws Exception {
+    return http.csrf(csrf -> csrf.disable())
+        .sessionManagement(
+            sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
+        .authorizeHttpRequests(
+            requests ->
+                requests
+                    .requestMatchers(
+                        "/actuator/health/liveness", "/actuator/health/readiness", "/error")
+                    .permitAll()
+                    .requestMatchers("/admin/**")
+                    .authenticated()
+                    .anyRequest()
+                    .denyAll())
+        .exceptionHandling(
+            failures ->
+                failures
+                    .authenticationEntryPoint(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.UNAUTHORIZED,
+                                Rfc7807Writer.UNAUTHORIZED,
+                                "Unauthorized",
+                                "UNAUTHORIZED",
+                                "Authentication is required"))
+                    .accessDeniedHandler(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.FORBIDDEN,
+                                Rfc7807Writer.FORBIDDEN,
+                                "Forbidden",
+                                "FORBIDDEN",
+                                "The operator is not allowed to perform this action")))
+        .oauth2ResourceServer(
+            resourceServer ->
+                resourceServer
+                    .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
+                    .authenticationEntryPoint(
+                        (request, response, failure) ->
+                            problems.write(
+                                request,
+                                response,
+                                HttpStatus.UNAUTHORIZED,
+                                Rfc7807Writer.UNAUTHORIZED,
+                                "Unauthorized",
+                                "UNAUTHORIZED",
+                                "Authentication is required")))
+        .build();
+  }
+
+  private static JwtAuthenticationConverter jwtAuthenticationConverter() {
+    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
+    converter.setJwtGrantedAuthoritiesConverter(
+        jwt ->
+            AdminRole.fromClaim(jwt.getClaims().get("role"))
+                .map(role -> List.<GrantedAuthority>of(new SimpleGrantedAuthority(role.authority())))
+                .orElseGet(List::of));
+    return converter;
+  }
+}


## `feat(security): initialize mutation context after JWT`

diff --git a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
index 978379a..02a4b3d 100644
--- a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
+++ b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
@@ -1,5 +1,6 @@
 package com.sportsbook.admin.security;
 
+import com.sportsbook.admin.context.AdminMutationContextFilter;
 import com.sportsbook.admin.error.Rfc7807Writer;
 import java.util.List;
 import org.springframework.context.annotation.Bean;
@@ -77,6 +78,7 @@ class SecurityConfig {
                                 "UNAUTHORIZED",
                                 "Authentication is required")))
         .addFilterBefore(ipAllowlist, BearerTokenAuthenticationFilter.class)
+        .addFilterAfter(new AdminMutationContextFilter(), BearerTokenAuthenticationFilter.class)
         .build();
   }
 
@@ -85,7 +87,8 @@ class SecurityConfig {
     converter.setJwtGrantedAuthoritiesConverter(
         jwt ->
             AdminRole.fromClaim(jwt.getClaims().get("role"))
-                .map(role -> List.<GrantedAuthority>of(new SimpleGrantedAuthority(role.authority())))
+                .map(
+                    role -> List.<GrantedAuthority>of(new SimpleGrantedAuthority(role.authority())))
                 .orElseGet(List::of));
     return converter;
   }


## `feat(api): require one idempotency header`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminRequestException.java b/src/main/java/com/sportsbook/admin/api/AdminRequestException.java
new file mode 100644
index 0000000..b318c33
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AdminRequestException.java
@@ -0,0 +1,12 @@
+package com.sportsbook.admin.api;
+
+public final class AdminRequestException extends IllegalArgumentException {
+
+  public AdminRequestException(String message) {
+    super(message);
+  }
+
+  public AdminRequestException(String message, Throwable cause) {
+    super(message, cause);
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java b/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java
new file mode 100644
index 0000000..141e26f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/api/AdminRequestHeaders.java
@@ -0,0 +1,43 @@
+package com.sportsbook.admin.api;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import jakarta.servlet.http.HttpServletRequest;
+import java.util.Enumeration;
+import java.util.UUID;
+
+public final class AdminRequestHeaders {
+
+  public static final String IDEMPOTENCY_KEY = "Idempotency-Key";
+
+  private AdminRequestHeaders() {}
+
+  public static IdempotencyKey requireIdempotencyKey(HttpServletRequest request) {
+    String value = requireSingleValue(request);
+    try {
+      return IdempotencyKey.of(value);
+    } catch (IllegalArgumentException invalid) {
+      throw new AdminRequestException("Idempotency-Key is invalid", invalid);
+    }
+  }
+
+  public static UUID requireUuidIdempotencyKey(HttpServletRequest request) {
+    String value = requireSingleValue(request);
+    try {
+      return UUID.fromString(value);
+    } catch (IllegalArgumentException invalid) {
+      throw new AdminRequestException("Idempotency-Key must be a UUID", invalid);
+    }
+  }
+
+  private static String requireSingleValue(HttpServletRequest request) {
+    Enumeration<String> values = request.getHeaders(IDEMPOTENCY_KEY);
+    if (values == null || !values.hasMoreElements()) {
+      throw new AdminRequestException("Exactly one Idempotency-Key header is required");
+    }
+    String value = values.nextElement();
+    if (values.hasMoreElements()) {
+      throw new AdminRequestException("Exactly one Idempotency-Key header is required");
+    }
+    return value;
+  }
+}


## `feat(api): render invalid admin requests`

diff --git a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
index cef41df..9a7f9d9 100644
--- a/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
+++ b/src/main/java/com/sportsbook/admin/api/AdminExceptionHandler.java
@@ -5,6 +5,7 @@ import com.sportsbook.admin.client.DownstreamContractException;
 import com.sportsbook.admin.client.DownstreamStatusException;
 import com.sportsbook.admin.client.DownstreamUnavailableException;
 import com.sportsbook.admin.context.AdminContextArgumentResolver;
+import com.sportsbook.protocol.error.ErrorCode;
 import com.sportsbook.protocol.error.ProblemDetail;
 import jakarta.servlet.http.HttpServletRequest;
 import java.net.URI;
@@ -13,8 +14,12 @@ import org.springframework.http.HttpHeaders;
 import org.springframework.http.HttpStatus;
 import org.springframework.http.MediaType;
 import org.springframework.http.ResponseEntity;
+import org.springframework.http.converter.HttpMessageNotReadableException;
+import org.springframework.web.bind.MethodArgumentNotValidException;
+import org.springframework.web.bind.ServletRequestBindingException;
 import org.springframework.web.bind.annotation.ExceptionHandler;
 import org.springframework.web.bind.annotation.RestControllerAdvice;
+import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
 
 @RestControllerAdvice
 public final class AdminExceptionHandler {
@@ -31,9 +36,7 @@ public final class AdminExceptionHandler {
   ResponseEntity<byte[]> relayDownstream(DownstreamStatusException failure) {
     HttpHeaders headers = new HttpHeaders();
     headers.setContentType(
-        failure.contentType() == null
-            ? MediaType.APPLICATION_PROBLEM_JSON
-            : failure.contentType());
+        failure.contentType() == null ? MediaType.APPLICATION_PROBLEM_JSON : failure.contentType());
     headers.setCacheControl("no-store");
     return new ResponseEntity<>(failure.body(), headers, failure.status());
   }
@@ -68,6 +71,25 @@ public final class AdminExceptionHandler {
         request);
   }
 
+  @ExceptionHandler({
+    AdminRequestException.class,
+    MethodArgumentNotValidException.class,
+    HttpMessageNotReadableException.class,
+    MethodArgumentTypeMismatchException.class,
+    ServletRequestBindingException.class
+  })
+  ResponseEntity<ProblemDetail> invalidRequest(Exception failure, HttpServletRequest request) {
+    ProblemDetail body =
+        ErrorCode.VALIDATION_FAILED.toProblemDetail(
+            "The admin request is invalid",
+            URI.create(request.getRequestURI()),
+            MDC.get("traceId"));
+    return ResponseEntity.badRequest()
+        .contentType(MediaType.APPLICATION_PROBLEM_JSON)
+        .cacheControl(org.springframework.http.CacheControl.noStore())
+        .body(body);
+  }
+
   @ExceptionHandler(AuditPersistenceException.class)
   ResponseEntity<ProblemDetail> auditPersistence(
       AuditPersistenceException failure, HttpServletRequest request) {
@@ -91,17 +113,12 @@ public final class AdminExceptionHandler {
     HttpHeaders headers = new HttpHeaders();
     headers.setContentType(MediaType.APPLICATION_PROBLEM_JSON);
     headers.setCacheControl("no-store");
-    headers.set(
-        AdminContextArgumentResolver.ACTION_ID_HEADER, failure.actionId().toString());
+    headers.set(AdminContextArgumentResolver.ACTION_ID_HEADER, failure.actionId().toString());
     return new ResponseEntity<>(body, headers, HttpStatus.SERVICE_UNAVAILABLE);
   }
 
   private static ResponseEntity<ProblemDetail> problem(
-      HttpStatus status,
-      URI type,
-      String code,
-      String detail,
-      HttpServletRequest request) {
+      HttpStatus status, URI type, String code, String detail, HttpServletRequest request) {
     ProblemDetail body =
         new ProblemDetail(
             type,


## `feat(client): validate isolated downstream credentials`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java b/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java
new file mode 100644
index 0000000..5d33f10
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamCredentials.java
@@ -0,0 +1,33 @@
+package com.sportsbook.admin.client;
+
+import jakarta.validation.constraints.NotBlank;
+import jakarta.validation.constraints.Size;
+import java.util.HashSet;
+import java.util.Objects;
+import java.util.stream.Stream;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.validation.annotation.Validated;
+
+@Validated
+@ConfigurationProperties("admin.downstream.credentials")
+public record DownstreamCredentials(
+    @NotBlank @Size(min = 32) String walletApiKey,
+    @NotBlank @Size(min = 32) String riskApiKey,
+    @NotBlank @Size(min = 32) String oddsFeedApiKey,
+    @NotBlank @Size(min = 32) String settlementApiKey) {
+
+  public DownstreamCredentials {
+    var configured =
+        Stream.of(walletApiKey, riskApiKey, oddsFeedApiKey, settlementApiKey)
+            .filter(Objects::nonNull)
+            .toList();
+    if (new HashSet<>(configured).size() != configured.size()) {
+      throw new IllegalArgumentException("Admin downstream API keys must be distinct");
+    }
+  }
+
+  @Override
+  public String toString() {
+    return "DownstreamCredentials[REDACTED]";
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 7d15dcc..3ed9980 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -61,3 +61,9 @@ admin:
       issuer: ${ADMIN_JWT_ISSUER:}
     ip-allowlist: ${ADMIN_IP_ALLOWLIST:127.0.0.1/32,::1/128}
     trusted-proxy-cidrs: ${ADMIN_TRUSTED_PROXY_CIDRS:}
+  downstream:
+    credentials:
+      wallet-api-key: ${ADMIN_WALLET_API_KEY:}
+      risk-api-key: ${ADMIN_RISK_API_KEY:}
+      odds-feed-api-key: ${ADMIN_ODDS_FEED_API_KEY:}
+      settlement-api-key: ${ADMIN_SETTLEMENT_API_KEY:}


## `feat(client): classify ambiguous downstream failures`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
index b327f31..f69c89e 100644
--- a/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamFailureMapper.java
@@ -1,9 +1,12 @@
 package com.sportsbook.admin.client;
 
+import java.net.SocketTimeoutException;
 import java.util.function.Supplier;
 import org.springframework.http.HttpHeaders;
 import org.springframework.http.MediaType;
 import org.springframework.web.client.HttpClientErrorException;
+import org.springframework.web.client.HttpServerErrorException;
+import org.springframework.web.client.ResourceAccessException;
 
 public final class DownstreamFailureMapper {
 
@@ -15,6 +18,17 @@ public final class DownstreamFailureMapper {
       MediaType contentType = headers == null ? null : headers.getContentType();
       throw new DownstreamStatusException(
           rejection.getStatusCode(), contentType, rejection.getResponseBodyAsByteArray());
+    } catch (HttpServerErrorException serverError) {
+      throw new DownstreamUnavailableException(
+          DownstreamUnavailableException.Reason.SERVER_ERROR,
+          serverError.getStatusCode(),
+          serverError);
+    } catch (ResourceAccessException transportFailure) {
+      DownstreamUnavailableException.Reason reason =
+          hasCause(transportFailure, SocketTimeoutException.class)
+              ? DownstreamUnavailableException.Reason.TIMEOUT
+              : DownstreamUnavailableException.Reason.TRANSPORT;
+      throw new DownstreamUnavailableException(reason, null, transportFailure);
     }
   }
 
@@ -25,4 +39,13 @@ public final class DownstreamFailureMapper {
           return null;
         });
   }
+
+  private static boolean hasCause(Throwable failure, Class<? extends Throwable> type) {
+    for (Throwable cause = failure; cause != null; cause = cause.getCause()) {
+      if (type.isInstance(cause)) {
+        return true;
+      }
+    }
+    return false;
+  }
 }
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java b/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java
new file mode 100644
index 0000000..732b821
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamUnavailableException.java
@@ -0,0 +1,29 @@
+package com.sportsbook.admin.client;
+
+import org.springframework.http.HttpStatusCode;
+
+public final class DownstreamUnavailableException extends RuntimeException {
+
+  public enum Reason {
+    SERVER_ERROR,
+    TIMEOUT,
+    TRANSPORT
+  }
+
+  private final Reason reason;
+  private final HttpStatusCode status;
+
+  DownstreamUnavailableException(Reason reason, HttpStatusCode status, Throwable cause) {
+    super("Downstream outcome is unknown: " + reason, cause);
+    this.reason = reason;
+    this.status = status;
+  }
+
+  public Reason reason() {
+    return reason;
+  }
+
+  public HttpStatusCode status() {
+    return status;
+  }
+}


## `feat(client): reject malformed success responses`

diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamContract.java b/src/main/java/com/sportsbook/admin/client/DownstreamContract.java
new file mode 100644
index 0000000..c88eb74
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamContract.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin.client;
+
+import java.util.function.Predicate;
+import org.springframework.http.HttpStatus;
+import org.springframework.http.ResponseEntity;
+
+public final class DownstreamContract {
+
+  private DownstreamContract() {}
+
+  public static <T> T requireBody(
+      ResponseEntity<T> response, HttpStatus expectedStatus, Predicate<T> proof, String contract) {
+    T body = response.getBody();
+    if (response.getStatusCode().value() != expectedStatus.value()
+        || body == null
+        || !proof.test(body)) {
+      throw new DownstreamContractException(contract);
+    }
+    return body;
+  }
+
+  public static void requireEmpty(
+      ResponseEntity<byte[]> response, HttpStatus expectedStatus, String contract) {
+    byte[] body = response.getBody();
+    if (response.getStatusCode().value() != expectedStatus.value()
+        || (body != null && body.length != 0)) {
+      throw new DownstreamContractException(contract);
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
new file mode 100644
index 0000000..2f3192b
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/client/DownstreamContractException.java
@@ -0,0 +1,8 @@
+package com.sportsbook.admin.client;
+
+public final class DownstreamContractException extends RuntimeException {
+
+  DownstreamContractException(String contract) {
+    super("Downstream success response violated contract: " + contract);
+  }
+}


## `feat(audit): declare audited action metadata`

diff --git a/src/main/java/com/sportsbook/admin/audit/Audited.java b/src/main/java/com/sportsbook/admin/audit/Audited.java
new file mode 100644
index 0000000..b6fa085
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/audit/Audited.java
@@ -0,0 +1,17 @@
+package com.sportsbook.admin.audit;
+
+import java.lang.annotation.ElementType;
+import java.lang.annotation.Retention;
+import java.lang.annotation.RetentionPolicy;
+import java.lang.annotation.Target;
+
+@Target(ElementType.METHOD)
+@Retention(RetentionPolicy.RUNTIME)
+public @interface Audited {
+
+  AdminAction action();
+
+  String target() default "";
+
+  String reason() default "";
+}


