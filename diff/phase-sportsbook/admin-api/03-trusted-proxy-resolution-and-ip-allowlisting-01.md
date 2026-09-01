# 신뢰 프록시 해석과 IP 허용 목록

## `feat(security): parse CIDR address ranges`

diff --git a/src/main/java/com/sportsbook/admin/security/CidrBlock.java b/src/main/java/com/sportsbook/admin/security/CidrBlock.java
new file mode 100644
index 0000000..57dfcd6
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/CidrBlock.java
@@ -0,0 +1,74 @@
+package com.sportsbook.admin.security;
+
+import java.net.InetAddress;
+import java.net.UnknownHostException;
+import java.util.Optional;
+import org.springframework.util.StringUtils;
+
+final class CidrBlock {
+
+  private final byte[] network;
+  private final int prefixLength;
+
+  private CidrBlock(byte[] network, int prefixLength) {
+    this.network = network.clone();
+    this.prefixLength = prefixLength;
+  }
+
+  static CidrBlock parse(String value) {
+    if (!StringUtils.hasText(value)) {
+      throw new IllegalArgumentException("CIDR must not be blank");
+    }
+    String[] parts = value.trim().split("/", -1);
+    if (parts.length != 2) {
+      throw new IllegalArgumentException("CIDR must contain one prefix length");
+    }
+    InetAddress address =
+        parseAddress(parts[0]).orElseThrow(() -> new IllegalArgumentException("Invalid CIDR IP"));
+    int prefix;
+    try {
+      prefix = Integer.parseInt(parts[1]);
+    } catch (NumberFormatException invalidPrefix) {
+      throw new IllegalArgumentException("Invalid CIDR prefix", invalidPrefix);
+    }
+    int maximumPrefix = address.getAddress().length * Byte.SIZE;
+    if (prefix < 0 || prefix > maximumPrefix) {
+      throw new IllegalArgumentException("CIDR prefix is out of range");
+    }
+    return new CidrBlock(address.getAddress(), prefix);
+  }
+
+  static Optional<InetAddress> parseAddress(String value) {
+    if (!StringUtils.hasText(value)) {
+      return Optional.empty();
+    }
+    String candidate = value.trim();
+    if (!candidate.matches("[0-9.]+") && !candidate.matches("[0-9A-Fa-f:]+")) {
+      return Optional.empty();
+    }
+    try {
+      return Optional.of(InetAddress.getByName(candidate));
+    } catch (UnknownHostException invalidAddress) {
+      return Optional.empty();
+    }
+  }
+
+  boolean contains(InetAddress address) {
+    byte[] candidate = address.getAddress();
+    if (candidate.length != network.length) {
+      return false;
+    }
+    int completeBytes = prefixLength / Byte.SIZE;
+    int remainingBits = prefixLength % Byte.SIZE;
+    for (int index = 0; index < completeBytes; index++) {
+      if (candidate[index] != network[index]) {
+        return false;
+      }
+    }
+    if (remainingBits == 0) {
+      return true;
+    }
+    int mask = -1 << (Byte.SIZE - remainingBits);
+    return (candidate[completeBytes] & mask) == (network[completeBytes] & mask);
+  }
+}


## `test(security): parse canonical IP literals`

diff --git a/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java b/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java
new file mode 100644
index 0000000..271cbc9
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java
@@ -0,0 +1,17 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.net.InetAddress;
+import org.junit.jupiter.api.Test;
+
+class CidrBlockLiteralTest {
+
+  @Test
+  void parsesCanonicalIpv4AndIpv6Literals() throws Exception {
+    assertThat(CidrBlock.parseAddress("127.0.0.1"))
+        .contains(InetAddress.getByAddress(new byte[] {127, 0, 0, 1}));
+    assertThat(CidrBlock.parseAddress("2001:db8::1"))
+        .contains(InetAddress.getByName("2001:db8::1"));
+  }
+}


## `feat(security): parse only literal IP addresses`

diff --git a/src/main/java/com/sportsbook/admin/security/CidrBlock.java b/src/main/java/com/sportsbook/admin/security/CidrBlock.java
index 57dfcd6..16439f8 100644
--- a/src/main/java/com/sportsbook/admin/security/CidrBlock.java
+++ b/src/main/java/com/sportsbook/admin/security/CidrBlock.java
@@ -7,6 +7,10 @@ import org.springframework.util.StringUtils;
 
 final class CidrBlock {
 
+  private static final int IPV4_OCTETS = 4;
+  private static final int MAX_IPV4_OCTET = 255;
+  private static final int IPV6_BYTES = 16;
+
   private final byte[] network;
   private final int prefixLength;
 
@@ -43,12 +47,42 @@ final class CidrBlock {
       return Optional.empty();
     }
     String candidate = value.trim();
-    if (!candidate.matches("[0-9.]+") && !candidate.matches("[0-9A-Fa-f:]+")) {
+    return candidate.indexOf(':') >= 0 ? parseIpv6(candidate) : parseIpv4(candidate);
+  }
+
+  private static Optional<InetAddress> parseIpv4(String candidate) {
+    String[] octets = candidate.split("\\.", -1);
+    if (octets.length != IPV4_OCTETS) {
+      return Optional.empty();
+    }
+    byte[] address = new byte[IPV4_OCTETS];
+    for (int index = 0; index < octets.length; index++) {
+      String octet = octets[index];
+      if (!octet.matches("0|[1-9][0-9]{0,2}")) {
+        return Optional.empty();
+      }
+      int parsed = Integer.parseInt(octet);
+      if (parsed > MAX_IPV4_OCTET) {
+        return Optional.empty();
+      }
+      address[index] = (byte) parsed;
+    }
+    try {
+      return Optional.of(InetAddress.getByAddress(address));
+    } catch (UnknownHostException impossibleLength) {
+      throw new IllegalStateException(
+          "IPv4 addresses always contain four octets", impossibleLength);
+    }
+  }
+
+  private static Optional<InetAddress> parseIpv6(String candidate) {
+    if (!candidate.matches("[0-9A-Fa-f:]+")) {
       return Optional.empty();
     }
     try {
-      return Optional.of(InetAddress.getByName(candidate));
-    } catch (UnknownHostException invalidAddress) {
+      InetAddress parsed = InetAddress.getByName(candidate);
+      return parsed.getAddress().length == IPV6_BYTES ? Optional.of(parsed) : Optional.empty();
+    } catch (UnknownHostException invalidLiteral) {
       return Optional.empty();
     }
   }


## `test(security): reject ambiguous IP forms`

diff --git a/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java b/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java
index 271cbc9..7901219 100644
--- a/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java
+++ b/src/test/java/com/sportsbook/admin/security/CidrBlockLiteralTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.admin.security;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import java.net.InetAddress;
 import org.junit.jupiter.api.Test;
@@ -14,4 +15,17 @@ class CidrBlockLiteralTest {
     assertThat(CidrBlock.parseAddress("2001:db8::1"))
         .contains(InetAddress.getByName("2001:db8::1"));
   }
+
+  @Test
+  void rejectsAlternateNumericFormsAndHostnamesWithoutResolution() {
+    assertThat(CidrBlock.parseAddress("2130706433")).isEmpty();
+    assertThat(CidrBlock.parseAddress("127.1")).isEmpty();
+    assertThat(CidrBlock.parseAddress("0177.0.0.1")).isEmpty();
+    assertThat(CidrBlock.parseAddress("deadbeef")).isEmpty();
+    assertThat(CidrBlock.parseAddress("localhost")).isEmpty();
+    assertThat(CidrBlock.parseAddress("example.com")).isEmpty();
+
+    assertThatThrownBy(() -> CidrBlock.parse("2130706433/8"))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
 }


## `feat(security): resolve trusted client addresses`

diff --git a/src/main/java/com/sportsbook/admin/security/TrustedProxyResolver.java b/src/main/java/com/sportsbook/admin/security/TrustedProxyResolver.java
new file mode 100644
index 0000000..83f7055
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/TrustedProxyResolver.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.security;
+
+import jakarta.servlet.http.HttpServletRequest;
+import java.net.InetAddress;
+import java.util.ArrayList;
+import java.util.Collections;
+import java.util.List;
+import java.util.Optional;
+
+final class TrustedProxyResolver {
+
+  private static final String FORWARDED_FOR = "X-Forwarded-For";
+  private final List<CidrBlock> trustedProxies;
+
+  TrustedProxyResolver(List<String> trustedProxyCidrs) {
+    trustedProxies = trustedProxyCidrs.stream().map(CidrBlock::parse).toList();
+  }
+
+  Optional<InetAddress> resolve(HttpServletRequest request) {
+    Optional<InetAddress> parsedPeer = CidrBlock.parseAddress(request.getRemoteAddr());
+    if (parsedPeer.isEmpty()) {
+      return Optional.empty();
+    }
+    InetAddress peer = parsedPeer.orElseThrow();
+    if (!isTrusted(peer)) {
+      return Optional.of(peer);
+    }
+
+    List<InetAddress> hops = forwardedHops(request);
+    if (hops.isEmpty()) {
+      return Optional.empty();
+    }
+    for (int index = hops.size() - 1; index >= 0; index--) {
+      InetAddress hop = hops.get(index);
+      if (!isTrusted(hop)) {
+        return Optional.of(hop);
+      }
+    }
+    return Optional.empty();
+  }
+
+  private List<InetAddress> forwardedHops(HttpServletRequest request) {
+    List<String> values = Collections.list(request.getHeaders(FORWARDED_FOR));
+    List<InetAddress> hops = new ArrayList<>();
+    for (String value : values) {
+      for (String hop : value.split(",", -1)) {
+        Optional<InetAddress> parsed = CidrBlock.parseAddress(hop);
+        if (parsed.isEmpty()) {
+          return List.of();
+        }
+        hops.add(parsed.orElseThrow());
+      }
+    }
+    return List.copyOf(hops);
+  }
+
+  private boolean isTrusted(InetAddress address) {
+    return trustedProxies.stream().anyMatch(cidr -> cidr.contains(address));
+  }
+}


## `test(security): reject forwarded-address spoofing`

diff --git a/src/test/java/com/sportsbook/admin/security/TrustedProxyResolverTest.java b/src/test/java/com/sportsbook/admin/security/TrustedProxyResolverTest.java
new file mode 100644
index 0000000..92a46da
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/TrustedProxyResolverTest.java
@@ -0,0 +1,62 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.net.InetAddress;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+
+class TrustedProxyResolverTest {
+
+  private final TrustedProxyResolver resolver =
+      new TrustedProxyResolver(List.of("10.0.0.0/8", "fd00::/8"));
+
+  @Test
+  void ignoresForwardedHeadersFromAnUntrustedPeer() throws Exception {
+    MockHttpServletRequest request = requestFrom("203.0.113.9");
+    request.addHeader("X-Forwarded-For", "10.1.2.3");
+
+    assertThat(resolver.resolve(request)).contains(InetAddress.getByName("203.0.113.9"));
+  }
+
+  @Test
+  void walksATrustedProxyChainFromRightToLeft() throws Exception {
+    MockHttpServletRequest request = requestFrom("10.0.0.5");
+    request.addHeader("X-Forwarded-For", "198.51.100.7, 10.0.0.4");
+
+    assertThat(resolver.resolve(request)).contains(InetAddress.getByName("198.51.100.7"));
+  }
+
+  @Test
+  void rejectsATrustedPeerWhenAnyForwardedHopIsMalformed() {
+    MockHttpServletRequest request = requestFrom("10.0.0.5");
+    request.addHeader("X-Forwarded-For", "198.51.100.7, attacker.example");
+
+    assertThat(resolver.resolve(request)).isEmpty();
+  }
+
+  @Test
+  void rejectsAChainWithNoResolvableClient() {
+    MockHttpServletRequest request = requestFrom("10.0.0.5");
+    request.addHeader("X-Forwarded-For", "10.0.0.2, 10.0.0.4");
+
+    assertThat(resolver.resolve(request)).isEmpty();
+    assertThat(resolver.resolve(requestFrom("10.0.0.5"))).isEmpty();
+  }
+
+  @Test
+  void supportsTrustedIpv6ProxiesAndRejectsANonnumericPeer() throws Exception {
+    MockHttpServletRequest ipv6 = requestFrom("fd00::5");
+    ipv6.addHeader("X-Forwarded-For", "2001:db8::42");
+
+    assertThat(resolver.resolve(ipv6)).contains(InetAddress.getByName("2001:db8::42"));
+    assertThat(resolver.resolve(requestFrom("proxy.internal"))).isEmpty();
+  }
+
+  private static MockHttpServletRequest requestFrom(String remoteAddress) {
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/admin/v1/probe");
+    request.setRemoteAddr(remoteAddress);
+    return request;
+  }
+}


## `feat(security): validate network boundary settings`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminNetworkProperties.java b/src/main/java/com/sportsbook/admin/security/AdminNetworkProperties.java
new file mode 100644
index 0000000..15ef121
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminNetworkProperties.java
@@ -0,0 +1,30 @@
+package com.sportsbook.admin.security;
+
+import java.util.List;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("admin.security")
+public record AdminNetworkProperties(List<String> ipAllowlist, List<String> trustedProxyCidrs) {
+
+  public AdminNetworkProperties {
+    ipAllowlist = validated(ipAllowlist, true, "ADMIN_IP_ALLOWLIST");
+    trustedProxyCidrs = validated(trustedProxyCidrs, false, "ADMIN_TRUSTED_PROXY_CIDRS");
+  }
+
+  private static List<String> validated(
+      List<String> configured, boolean required, String settingName) {
+    List<String> values =
+        configured == null
+            ? List.of()
+            : configured.stream().filter(value -> value != null && !value.isBlank()).toList();
+    if (required && values.isEmpty()) {
+      throw new IllegalArgumentException(settingName + " must contain at least one CIDR");
+    }
+    try {
+      values.forEach(CidrBlock::parse);
+    } catch (IllegalArgumentException invalidCidr) {
+      throw new IllegalArgumentException(settingName + " contains an invalid CIDR", invalidCidr);
+    }
+    return List.copyOf(values);
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index db42335..7d15dcc 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -59,3 +59,5 @@ admin:
     jwt:
       public-key: ${ADMIN_JWT_PUBLIC_KEY:}
       issuer: ${ADMIN_JWT_ISSUER:}
+    ip-allowlist: ${ADMIN_IP_ALLOWLIST:127.0.0.1/32,::1/128}
+    trusted-proxy-cidrs: ${ADMIN_TRUSTED_PROXY_CIDRS:}


## `test(security): reject invalid network boundaries`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminNetworkPropertiesTest.java b/src/test/java/com/sportsbook/admin/security/AdminNetworkPropertiesTest.java
new file mode 100644
index 0000000..571277f
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminNetworkPropertiesTest.java
@@ -0,0 +1,57 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+
+class AdminNetworkPropertiesTest {
+
+  private final ApplicationContextRunner contextRunner =
+      new ApplicationContextRunner().withUserConfiguration(NetworkConfiguration.class);
+
+  @Test
+  void bindsValidatedIpv4AndIpv6Boundaries() {
+    contextRunner
+        .withPropertyValues(
+            "admin.security.ip-allowlist=127.0.0.1/32,::1/128",
+            "admin.security.trusted-proxy-cidrs=10.0.0.0/8,fd00::/8")
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              AdminNetworkProperties properties = context.getBean(AdminNetworkProperties.class);
+              assertThat(properties.ipAllowlist()).containsExactly("127.0.0.1/32", "::1/128");
+              assertThat(properties.trustedProxyCidrs()).containsExactly("10.0.0.0/8", "fd00::/8");
+            });
+  }
+
+  @Test
+  void rejectsAMissingAllowlistAndMalformedCidrs() {
+    contextRunner.run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues("admin.security.ip-allowlist=10.0.0.0/33")
+        .run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues(
+            "admin.security.ip-allowlist=127.0.0.1/32",
+            "admin.security.trusted-proxy-cidrs=proxy.internal/24")
+        .run(context -> assertThat(context).hasFailed());
+  }
+
+  @Test
+  void makesConfiguredListsImmutable() {
+    AdminNetworkProperties properties =
+        new AdminNetworkProperties(List.of("127.0.0.1/32"), List.of());
+
+    assertThatThrownBy(() -> properties.ipAllowlist().add("10.0.0.0/8"))
+        .isInstanceOf(UnsupportedOperationException.class);
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(AdminNetworkProperties.class)
+  static class NetworkConfiguration {}
+}


## `feat(security): enforce the admin IP allowlist`

diff --git a/src/main/java/com/sportsbook/admin/security/IpAllowlistFilter.java b/src/main/java/com/sportsbook/admin/security/IpAllowlistFilter.java
new file mode 100644
index 0000000..5b59604
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/IpAllowlistFilter.java
@@ -0,0 +1,60 @@
+package com.sportsbook.admin.security;
+
+import com.sportsbook.admin.error.Rfc7807Writer;
+import jakarta.servlet.FilterChain;
+import jakarta.servlet.ServletException;
+import jakarta.servlet.http.HttpServletRequest;
+import jakarta.servlet.http.HttpServletResponse;
+import java.io.IOException;
+import java.net.InetAddress;
+import java.util.List;
+import java.util.Optional;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.http.HttpStatus;
+import org.springframework.web.filter.OncePerRequestFilter;
+
+final class IpAllowlistFilter extends OncePerRequestFilter {
+
+  private static final Logger LOG = LoggerFactory.getLogger(IpAllowlistFilter.class);
+
+  private final TrustedProxyResolver clientAddresses;
+  private final List<CidrBlock> allowedClients;
+  private final Rfc7807Writer problems;
+
+  IpAllowlistFilter(AdminNetworkProperties properties, Rfc7807Writer problems) {
+    clientAddresses = new TrustedProxyResolver(properties.trustedProxyCidrs());
+    allowedClients = properties.ipAllowlist().stream().map(CidrBlock::parse).toList();
+    this.problems = problems;
+  }
+
+  @Override
+  protected boolean shouldNotFilter(HttpServletRequest request) {
+    String path = request.getServletPath();
+    return !(path.equals("/admin") || path.startsWith("/admin/"));
+  }
+
+  @Override
+  protected void doFilterInternal(
+      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
+      throws ServletException, IOException {
+    Optional<InetAddress> clientAddress = clientAddresses.resolve(request);
+    boolean allowed =
+        clientAddress.isPresent()
+            && allowedClients.stream().anyMatch(cidr -> cidr.contains(clientAddress.orElseThrow()));
+    if (allowed) {
+      chain.doFilter(request, response);
+      return;
+    }
+
+    LOG.warn("admin_ip_allowlist_denied path={}", request.getRequestURI());
+    problems.write(
+        request,
+        response,
+        HttpStatus.FORBIDDEN,
+        Rfc7807Writer.IP_NOT_ALLOWED,
+        "Forbidden",
+        "IP_NOT_ALLOWED",
+        "The client address is not allowed");
+  }
+}
diff --git a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
index 273beee..978379a 100644
--- a/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
+++ b/src/main/java/com/sportsbook/admin/security/SecurityConfig.java
@@ -11,6 +11,7 @@ import org.springframework.security.config.http.SessionCreationPolicy;
 import org.springframework.security.core.GrantedAuthority;
 import org.springframework.security.core.authority.SimpleGrantedAuthority;
 import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
+import org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter;
 import org.springframework.security.web.SecurityFilterChain;
 
 @Configuration(proxyBeanMethods = false)
@@ -18,8 +19,10 @@ import org.springframework.security.web.SecurityFilterChain;
 class SecurityConfig {
 
   @Bean
-  SecurityFilterChain adminSecurityFilterChain(HttpSecurity http, Rfc7807Writer problems)
+  SecurityFilterChain adminSecurityFilterChain(
+      HttpSecurity http, Rfc7807Writer problems, AdminNetworkProperties networkProperties)
       throws Exception {
+    IpAllowlistFilter ipAllowlist = new IpAllowlistFilter(networkProperties, problems);
     return http.csrf(csrf -> csrf.disable())
         .sessionManagement(
             sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
@@ -73,6 +76,7 @@ class SecurityConfig {
                                 "Unauthorized",
                                 "UNAUTHORIZED",
                                 "Authentication is required")))
+        .addFilterBefore(ipAllowlist, BearerTokenAuthenticationFilter.class)
         .build();
   }
 


