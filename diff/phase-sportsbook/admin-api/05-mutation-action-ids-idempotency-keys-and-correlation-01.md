# 변경 요청의 작업 ID, 멱등성 키, 상관관계

## `feat(context): model UUIDv7 admin action identity`

diff --git a/src/main/java/com/sportsbook/admin/context/AdminContext.java b/src/main/java/com/sportsbook/admin/context/AdminContext.java
new file mode 100644
index 0000000..72adf1f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/context/AdminContext.java
@@ -0,0 +1,16 @@
+package com.sportsbook.admin.context;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.util.Objects;
+import java.util.UUID;
+
+public record AdminContext(String actorId, AdminRole actorRole, UUID actionId, String traceId) {
+
+  public AdminContext {
+    if (actorId == null || actorId.isBlank()) {
+      throw new IllegalArgumentException("actorId must not be blank");
+    }
+    Objects.requireNonNull(actorRole, "actorRole");
+    Objects.requireNonNull(actionId, "actionId");
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/context/Uuid7.java b/src/main/java/com/sportsbook/admin/context/Uuid7.java
new file mode 100644
index 0000000..3289ecb
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/context/Uuid7.java
@@ -0,0 +1,25 @@
+package com.sportsbook.admin.context;
+
+import java.security.SecureRandom;
+import java.util.UUID;
+
+public final class Uuid7 {
+
+  private static final SecureRandom RANDOM = new SecureRandom();
+  private static final int TIMESTAMP_SHIFT = 16;
+  private static final long TIMESTAMP_MASK = 0xFFFFFFFFFFFFL;
+  private static final long VERSION_BITS = 0x7000L;
+  private static final int RANDOM_A_BOUND = 0x1000;
+  private static final long VARIANT_BITS = 0x8000000000000000L;
+  private static final long RANDOM_B_MASK = 0x3FFFFFFFFFFFFFFFL;
+
+  private Uuid7() {}
+
+  public static UUID generate() {
+    long timestamp = System.currentTimeMillis() & TIMESTAMP_MASK;
+    long randomA = RANDOM.nextInt(RANDOM_A_BOUND);
+    long mostSignificant = (timestamp << TIMESTAMP_SHIFT) | VERSION_BITS | randomA;
+    long leastSignificant = VARIANT_BITS | (RANDOM.nextLong() & RANDOM_B_MASK);
+    return new UUID(mostSignificant, leastSignificant);
+  }
+}


## `test(context): verify admin action identities`

diff --git a/src/test/java/com/sportsbook/admin/context/AdminContextIdentityTest.java b/src/test/java/com/sportsbook/admin/context/AdminContextIdentityTest.java
new file mode 100644
index 0000000..1996f78
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/context/AdminContextIdentityTest.java
@@ -0,0 +1,54 @@
+package com.sportsbook.admin.context;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.util.HashSet;
+import java.util.Set;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AdminContextIdentityTest {
+
+  @Test
+  void generatesUniqueRfc9562Version7ActionIds() {
+    Set<UUID> ids = new HashSet<>();
+
+    for (int index = 0; index < 1_000; index++) {
+      UUID id = Uuid7.generate();
+      assertThat(id.version()).isEqualTo(7);
+      assertThat(id.variant()).isEqualTo(2);
+      ids.add(id);
+    }
+
+    assertThat(ids).hasSize(1_000);
+  }
+
+  @Test
+  void embedsTheCurrentUnixMillisecondTimestamp() {
+    long before = System.currentTimeMillis();
+    UUID id = Uuid7.generate();
+    long after = System.currentTimeMillis();
+    long embeddedTimestamp = id.getMostSignificantBits() >>> 16;
+
+    assertThat(embeddedTimestamp).isBetween(before, after);
+  }
+
+  @Test
+  void requiresACompleteAuthenticatedIdentity() {
+    UUID actionId = Uuid7.generate();
+    AdminContext context = new AdminContext("operator-1", AdminRole.ADMIN, actionId, "trace-1");
+
+    assertThat(context.actorId()).isEqualTo("operator-1");
+    assertThat(context.actorRole()).isEqualTo(AdminRole.ADMIN);
+    assertThat(context.actionId()).isEqualTo(actionId);
+    assertThat(context.traceId()).isEqualTo("trace-1");
+    assertThatThrownBy(() -> new AdminContext(" ", AdminRole.ADMIN, actionId, null))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> new AdminContext("operator-1", null, actionId, null))
+        .isInstanceOf(NullPointerException.class);
+    assertThatThrownBy(() -> new AdminContext("operator-1", AdminRole.ADMIN, null, null))
+        .isInstanceOf(NullPointerException.class);
+  }
+}


## `feat(context): resolve one action context per request`

diff --git a/src/main/java/com/sportsbook/admin/config/WebConfig.java b/src/main/java/com/sportsbook/admin/config/WebConfig.java
new file mode 100644
index 0000000..bd2ee5c
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/config/WebConfig.java
@@ -0,0 +1,22 @@
+package com.sportsbook.admin.config;
+
+import com.sportsbook.admin.context.AdminContextArgumentResolver;
+import java.util.List;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.web.method.support.HandlerMethodArgumentResolver;
+import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
+
+@Configuration(proxyBeanMethods = false)
+class WebConfig implements WebMvcConfigurer {
+
+  private final AdminContextArgumentResolver adminContexts;
+
+  WebConfig(AdminContextArgumentResolver adminContexts) {
+    this.adminContexts = adminContexts;
+  }
+
+  @Override
+  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
+    resolvers.add(adminContexts);
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java b/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java
new file mode 100644
index 0000000..9b279cf
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java
@@ -0,0 +1,56 @@
+package com.sportsbook.admin.context;
+
+import com.sportsbook.admin.security.AdminRole;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import org.slf4j.MDC;
+import org.springframework.core.MethodParameter;
+import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.stereotype.Component;
+import org.springframework.web.bind.support.WebDataBinderFactory;
+import org.springframework.web.context.request.NativeWebRequest;
+import org.springframework.web.method.support.HandlerMethodArgumentResolver;
+import org.springframework.web.method.support.ModelAndViewContainer;
+
+@Component
+public final class AdminContextArgumentResolver implements HandlerMethodArgumentResolver {
+
+  public static final String ACTION_ID_HEADER = "X-Admin-Action-Id";
+  private static final String REQUEST_ATTRIBUTE = AdminContext.class.getName();
+
+  @Override
+  public boolean supportsParameter(MethodParameter parameter) {
+    return parameter.getParameterType() == AdminContext.class;
+  }
+
+  @Override
+  public AdminContext resolveArgument(
+      MethodParameter parameter,
+      ModelAndViewContainer container,
+      NativeWebRequest webRequest,
+      WebDataBinderFactory binderFactory) {
+    HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
+    HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
+    if (request == null || response == null) {
+      throw new AuthenticationCredentialsNotFoundException("HTTP request context is required");
+    }
+    Object cached = request.getAttribute(REQUEST_ATTRIBUTE);
+    if (cached instanceof AdminContext context) {
+      return context;
+    }
+    if (!(request.getUserPrincipal() instanceof JwtAuthenticationToken authentication)
+        || !authentication.isAuthenticated()) {
+      throw new AuthenticationCredentialsNotFoundException("Verified operator JWT is required");
+    }
+    AdminRole role =
+        AdminRole.fromClaim(authentication.getToken().getClaims().get("role"))
+            .orElseThrow(
+                () -> new AuthenticationCredentialsNotFoundException("Verified role is required"));
+    AdminContext context =
+        new AdminContext(authentication.getName(), role, Uuid7.generate(), MDC.get("traceId"));
+    request.setAttribute(REQUEST_ATTRIBUTE, context);
+    response.setHeader(ACTION_ID_HEADER, context.actionId().toString());
+    return context;
+  }
+}


## `test(context): reuse the request action context`

diff --git a/src/test/java/com/sportsbook/admin/context/AdminContextArgumentResolverTest.java b/src/test/java/com/sportsbook/admin/context/AdminContextArgumentResolverTest.java
new file mode 100644
index 0000000..4cafb90
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/context/AdminContextArgumentResolverTest.java
@@ -0,0 +1,78 @@
+package com.sportsbook.admin.context;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.admin.security.AdminRole;
+import java.lang.reflect.Method;
+import java.util.List;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.slf4j.MDC;
+import org.springframework.core.MethodParameter;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.web.context.request.ServletWebRequest;
+
+class AdminContextArgumentResolverTest {
+
+  private final AdminContextArgumentResolver resolver = new AdminContextArgumentResolver();
+
+  @AfterEach
+  void clearTraceContext() {
+    MDC.clear();
+  }
+
+  @Test
+  void createsAndCachesOneContextWithTheResponseActionHeader() throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest();
+    MockHttpServletResponse response = new MockHttpServletResponse();
+    request.setUserPrincipal(authentication("operator-1", "TRADER"));
+    MDC.put("traceId", "trace-1");
+    ServletWebRequest webRequest = new ServletWebRequest(request, response);
+
+    AdminContext first = resolver.resolveArgument(parameter(), null, webRequest, null);
+    AdminContext second = resolver.resolveArgument(parameter(), null, webRequest, null);
+
+    assertThat(second).isSameAs(first);
+    assertThat(first.actorId()).isEqualTo("operator-1");
+    assertThat(first.actorRole()).isEqualTo(AdminRole.TRADER);
+    assertThat(first.traceId()).isEqualTo("trace-1");
+    assertThat(first.actionId().version()).isEqualTo(7);
+    assertThat(response.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER))
+        .isEqualTo(first.actionId().toString());
+  }
+
+  @Test
+  void failsClosedWithoutAVerifiedJwt() throws Exception {
+    ServletWebRequest webRequest =
+        new ServletWebRequest(new MockHttpServletRequest(), new MockHttpServletResponse());
+
+    assertThatThrownBy(() -> resolver.resolveArgument(parameter(), null, webRequest, null))
+        .isInstanceOf(AuthenticationCredentialsNotFoundException.class);
+  }
+
+  private static MethodParameter parameter() throws NoSuchMethodException {
+    Method method = Probe.class.getDeclaredMethod("handle", AdminContext.class);
+    return new MethodParameter(method, 0);
+  }
+
+  private static JwtAuthenticationToken authentication(String subject, String role) {
+    Jwt jwt =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .subject(subject)
+            .claim("role", role)
+            .build();
+    return new JwtAuthenticationToken(
+        jwt, List.of(new SimpleGrantedAuthority("ROLE_" + role)), subject);
+  }
+
+  private static final class Probe {
+    void handle(AdminContext context) {}
+  }
+}


## `feat(context): create mutation identities early`

diff --git a/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java b/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java
index 9b279cf..867551f 100644
--- a/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java
+++ b/src/main/java/com/sportsbook/admin/context/AdminContextArgumentResolver.java
@@ -6,6 +6,7 @@ import jakarta.servlet.http.HttpServletResponse;
 import org.slf4j.MDC;
 import org.springframework.core.MethodParameter;
 import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
+import org.springframework.security.core.context.SecurityContextHolder;
 import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
 import org.springframework.stereotype.Component;
 import org.springframework.web.bind.support.WebDataBinderFactory;
@@ -35,12 +36,21 @@ public final class AdminContextArgumentResolver implements HandlerMethodArgument
     if (request == null || response == null) {
       throw new AuthenticationCredentialsNotFoundException("HTTP request context is required");
     }
+    return initialize(request, response);
+  }
+
+  static AdminContext initialize(HttpServletRequest request, HttpServletResponse response) {
     Object cached = request.getAttribute(REQUEST_ATTRIBUTE);
     if (cached instanceof AdminContext context) {
       return context;
     }
-    if (!(request.getUserPrincipal() instanceof JwtAuthenticationToken authentication)
-        || !authentication.isAuthenticated()) {
+    Object principal = request.getUserPrincipal();
+    Object securityAuthentication = SecurityContextHolder.getContext().getAuthentication();
+    JwtAuthenticationToken authentication =
+        principal instanceof JwtAuthenticationToken jwt
+            ? jwt
+            : securityAuthentication instanceof JwtAuthenticationToken jwt ? jwt : null;
+    if (authentication == null || !authentication.isAuthenticated()) {
       throw new AuthenticationCredentialsNotFoundException("Verified operator JWT is required");
     }
     AdminRole role =
diff --git a/src/main/java/com/sportsbook/admin/context/AdminMutationContextFilter.java b/src/main/java/com/sportsbook/admin/context/AdminMutationContextFilter.java
new file mode 100644
index 0000000..af869d2
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/context/AdminMutationContextFilter.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.context;
+
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.util.Set;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+public final class AdminMutationContextFilter extends OncePerRequestFilter {
+
+  private static final Set<String> MUTATIONS = Set.of("POST", "PATCH", "DELETE");
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    return !request.getRequestURI().startsWith("/admin/v1/")
+        || !MUTATIONS.contains(request.getMethod());
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    if (SecurityContextHolder.getContext().getAuthentication()
+            instanceof JwtAuthenticationToken authentication
+        && authentication.isAuthenticated()) {
+      AdminContextArgumentResolver.initialize(request, response);
+    }
+    chain.doFilter(request, response);
+  }
+}


## `test(context): create identities before binding`

diff --git a/src/test/java/com/sportsbook/admin/context/AdminMutationContextFilterTest.java b/src/test/java/com/sportsbook/admin/context/AdminMutationContextFilterTest.java
new file mode 100644
index 0000000..a25205d
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/context/AdminMutationContextFilterTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.admin.context;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.AfterEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockFilterChain;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.core.context.SecurityContextHolder;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+
+class AdminMutationContextFilterTest {
+
+  private final AdminMutationContextFilter filter = new AdminMutationContextFilter();
+
+  @AfterEach
+  void clearSecurityContext() {
+    SecurityContextHolder.clearContext();
+  }
+
+  @Test
+  void createsOneUuid7IdentityBeforeAControllerBindsTheMutation() throws Exception {
+    SecurityContextHolder.getContext().setAuthentication(authentication());
+    MockHttpServletRequest request =
+        new MockHttpServletRequest("POST", "/admin/v1/wallet/user-1/refund");
+    request.setRequestURI("/admin/v1/wallet/user-1/refund");
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(request, response, new MockFilterChain());
+
+    UUID actionId =
+        UUID.fromString(response.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER));
+    assertThat(actionId.version()).isEqualTo(7);
+    assertThat(AdminContextArgumentResolver.initialize(request, response).actionId())
+        .isEqualTo(actionId);
+  }
+
+  @Test
+  void leavesReadsAndUnauthenticatedMutationsWithoutAnIdentity() throws Exception {
+    MockHttpServletRequest read = new MockHttpServletRequest("GET", "/admin/v1/audit-logs");
+    read.setRequestURI("/admin/v1/audit-logs");
+    MockHttpServletResponse readResponse = new MockHttpServletResponse();
+    filter.doFilter(read, readResponse, new MockFilterChain());
+
+    MockHttpServletRequest mutation = new MockHttpServletRequest("POST", "/admin/v1/test");
+    mutation.setRequestURI("/admin/v1/test");
+    MockHttpServletResponse mutationResponse = new MockHttpServletResponse();
+    filter.doFilter(mutation, mutationResponse, new MockFilterChain());
+
+    assertThat(readResponse.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER)).isNull();
+    assertThat(mutationResponse.getHeader(AdminContextArgumentResolver.ACTION_ID_HEADER)).isNull();
+  }
+
+  private static JwtAuthenticationToken authentication() {
+    Jwt jwt =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .subject("operator-1")
+            .claim("role", "ADMIN")
+            .build();
+    return new JwtAuthenticationToken(
+        jwt, List.of(new SimpleGrantedAuthority("ROLE_ADMIN")), "operator-1");
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


