## `test(security): verify allowlist CIDR boundaries`

diff --git a/src/test/java/com/sportsbook/admin/security/IpAllowlistFilterTest.java b/src/test/java/com/sportsbook/admin/security/IpAllowlistFilterTest.java
new file mode 100644
index 0000000..393ceed
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/IpAllowlistFilterTest.java
@@ -0,0 +1,67 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.sportsbook.admin.error.Rfc7807Writer;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.mock.web.MockHttpServletRequest;
+import org.springframework.mock.web.MockHttpServletResponse;
+
+class IpAllowlistFilterTest {
+
+  private final IpAllowlistFilter filter =
+      new IpAllowlistFilter(
+          new AdminNetworkProperties(
+              List.of("10.0.0.0/24", "2001:db8::/32"), List.of("192.0.2.0/24")),
+          new Rfc7807Writer(new ObjectMapper().findAndRegisterModules()));
+
+  @Test
+  void permitsIpv4AndIpv6AddressesAtTheConfiguredBoundaries() throws Exception {
+    assertThat(filter("/admin/v1/probe", "10.0.0.0", null).getStatus()).isEqualTo(204);
+    assertThat(filter("/admin/v1/probe", "10.0.0.255", null).getStatus()).isEqualTo(204);
+    assertThat(filter("/admin/v1/probe", "2001:db8::ffff", null).getStatus()).isEqualTo(204);
+  }
+
+  @Test
+  void deniesAddressesImmediatelyOutsideTheConfiguredNetworks() throws Exception {
+    MockHttpServletResponse ipv4 = filter("/admin/v1/probe", "10.0.1.0", null);
+    MockHttpServletResponse ipv6 = filter("/admin/v1/probe", "2001:db9::1", null);
+
+    assertThat(ipv4.getStatus()).isEqualTo(403);
+    assertThat(ipv4.getContentAsString()).contains("IP_NOT_ALLOWED");
+    assertThat(ipv6.getStatus()).isEqualTo(403);
+  }
+
+  @Test
+  void trustsForwardingOnlyFromAConfiguredProxy() throws Exception {
+    assertThat(filter("/admin/v1/probe", "192.0.2.5", "10.0.0.8").getStatus()).isEqualTo(204);
+    assertThat(filter("/admin/v1/probe", "198.51.100.5", "10.0.0.8").getStatus()).isEqualTo(403);
+  }
+
+  @Test
+  void doesNotApplyTheAdminBoundaryToAdjacentOrHealthPaths() throws Exception {
+    assertThat(filter("/administrator", "198.51.100.5", null).getStatus()).isEqualTo(204);
+    assertThat(filter("/actuator/health/readiness", "198.51.100.5", null).getStatus())
+        .isEqualTo(204);
+  }
+
+  private MockHttpServletResponse filter(String path, String peer, String forwardedFor)
+      throws Exception {
+    MockHttpServletRequest request = new MockHttpServletRequest("GET", path);
+    request.setServletPath(path);
+    request.setRemoteAddr(peer);
+    if (forwardedFor != null) {
+      request.addHeader("X-Forwarded-For", forwardedFor);
+    }
+    MockHttpServletResponse response = new MockHttpServletResponse();
+
+    filter.doFilter(
+        request,
+        response,
+        (ignoredRequest, filteredResponse) ->
+            ((jakarta.servlet.http.HttpServletResponse) filteredResponse).setStatus(204));
+    return response;
+  }
+}
