# JWT 운영자 신뢰와 역할 기반 접근 경계

## `feat(security): define exact operator roles`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminRole.java b/src/main/java/com/sportsbook/admin/security/AdminRole.java
new file mode 100644
index 0000000..ff346c1
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminRole.java
@@ -0,0 +1,25 @@
+package com.sportsbook.admin.security;
+
+import java.util.Optional;
+
+public enum AdminRole {
+  ADMIN,
+  TRADER,
+  CS,
+  READONLY;
+
+  public String authority() {
+    return "ROLE_" + name();
+  }
+
+  public static Optional<AdminRole> fromClaim(Object claim) {
+    if (!(claim instanceof String value)) {
+      return Optional.empty();
+    }
+    try {
+      return Optional.of(valueOf(value));
+    } catch (IllegalArgumentException unknownRole) {
+      return Optional.empty();
+    }
+  }
+}


## `test(security): reject inexact operator roles`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminRoleTest.java b/src/test/java/com/sportsbook/admin/security/AdminRoleTest.java
new file mode 100644
index 0000000..0c8520e
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminRoleTest.java
@@ -0,0 +1,32 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+class AdminRoleTest {
+
+  @Test
+  void mapsOnlyExactUppercaseRoleClaims() {
+    assertThat(AdminRole.fromClaim("ADMIN")).contains(AdminRole.ADMIN);
+    assertThat(AdminRole.fromClaim("TRADER")).contains(AdminRole.TRADER);
+    assertThat(AdminRole.fromClaim("CS")).contains(AdminRole.CS);
+    assertThat(AdminRole.fromClaim("READONLY")).contains(AdminRole.READONLY);
+  }
+
+  @Test
+  void rejectsMissingMalformedAndUnknownRoleClaims() {
+    assertThat(AdminRole.fromClaim(null)).isEmpty();
+    assertThat(AdminRole.fromClaim("")).isEmpty();
+    assertThat(AdminRole.fromClaim("admin")).isEmpty();
+    assertThat(AdminRole.fromClaim(" ADMIN ")).isEmpty();
+    assertThat(AdminRole.fromClaim("OWNER")).isEmpty();
+    assertThat(AdminRole.fromClaim(List.of("ADMIN"))).isEmpty();
+  }
+
+  @Test
+  void exposesSpringRoleAuthorities() {
+    assertThat(AdminRole.TRADER.authority()).isEqualTo("ROLE_TRADER");
+  }
+}


## `feat(security): validate JWT settings`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminJwtProperties.java b/src/main/java/com/sportsbook/admin/security/AdminJwtProperties.java
new file mode 100644
index 0000000..71524f6
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminJwtProperties.java
@@ -0,0 +1,15 @@
+package com.sportsbook.admin.security;
+
+import jakarta.validation.constraints.NotBlank;
+import java.util.Optional;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+import org.springframework.validation.annotation.Validated;
+
+@Validated
+@ConfigurationProperties("admin.security.jwt")
+public record AdminJwtProperties(@NotBlank String publicKey, String issuer) {
+
+  public Optional<String> expectedIssuer() {
+    return Optional.ofNullable(issuer).filter(value -> !value.isBlank());
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 906266d..db42335 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -53,3 +53,9 @@ logging:
     root: INFO
     com.sportsbook.admin: INFO
     org.hibernate.SQL: WARN
+
+admin:
+  security:
+    jwt:
+      public-key: ${ADMIN_JWT_PUBLIC_KEY:}
+      issuer: ${ADMIN_JWT_ISSUER:}


## `test(security): reject invalid JWT settings`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminJwtPropertiesTest.java b/src/test/java/com/sportsbook/admin/security/AdminJwtPropertiesTest.java
new file mode 100644
index 0000000..320dff4
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminJwtPropertiesTest.java
@@ -0,0 +1,51 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.boot.test.context.runner.ApplicationContextRunner;
+import org.springframework.context.annotation.Configuration;
+
+class AdminJwtPropertiesTest {
+
+  private final ApplicationContextRunner contextRunner =
+      new ApplicationContextRunner().withUserConfiguration(JwtPropertiesConfiguration.class);
+
+  @Test
+  void requiresAPublicKey() {
+    contextRunner.run(context -> assertThat(context).hasFailed());
+    contextRunner
+        .withPropertyValues("admin.security.jwt.public-key=   ")
+        .run(context -> assertThat(context).hasFailed());
+  }
+
+  @Test
+  void bindsARequiredKeyAndOptionalExactIssuer() {
+    contextRunner
+        .withPropertyValues(
+            "admin.security.jwt.public-key=test-public-key",
+            "admin.security.jwt.issuer=https://iam.example.test")
+        .run(
+            context -> {
+              assertThat(context).hasNotFailed();
+              AdminJwtProperties properties = context.getBean(AdminJwtProperties.class);
+              assertThat(properties.publicKey()).isEqualTo("test-public-key");
+              assertThat(properties.expectedIssuer()).contains("https://iam.example.test");
+            });
+  }
+
+  @Test
+  void treatsABlankIssuerAsUnconfigured() {
+    contextRunner
+        .withPropertyValues(
+            "admin.security.jwt.public-key=test-public-key", "admin.security.jwt.issuer=")
+        .run(
+            context ->
+                assertThat(context.getBean(AdminJwtProperties.class).expectedIssuer()).isEmpty());
+  }
+
+  @Configuration(proxyBeanMethods = false)
+  @EnableConfigurationProperties(AdminJwtProperties.class)
+  static class JwtPropertiesConfiguration {}
+}


## `feat(security): require strong RSA public keys`

diff --git a/src/main/java/com/sportsbook/admin/security/RsaPublicKeyParser.java b/src/main/java/com/sportsbook/admin/security/RsaPublicKeyParser.java
new file mode 100644
index 0000000..cb6409f
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/RsaPublicKeyParser.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.security;
+
+import java.security.GeneralSecurityException;
+import java.security.KeyFactory;
+import java.security.interfaces.RSAPublicKey;
+import java.security.spec.X509EncodedKeySpec;
+import java.util.Base64;
+
+final class RsaPublicKeyParser {
+
+  private static final String BEGIN = "-----BEGIN PUBLIC KEY-----";
+  private static final String END = "-----END PUBLIC KEY-----";
+  private static final int MINIMUM_RSA_BITS = 2048;
+
+  RSAPublicKey parse(String configuredKey) {
+    if (configuredKey == null || configuredKey.isBlank()) {
+      throw new IllegalStateException("ADMIN_JWT_PUBLIC_KEY is required");
+    }
+
+    String pem = configuredKey.replace("\\n", "\n").trim();
+    if (!pem.startsWith(BEGIN) || !pem.endsWith(END)) {
+      throw new IllegalStateException("ADMIN_JWT_PUBLIC_KEY must be an SPKI public key");
+    }
+
+    try {
+      String encoded =
+          pem.substring(BEGIN.length(), pem.length() - END.length()).replaceAll("\\s+", "");
+      byte[] der = Base64.getDecoder().decode(encoded);
+      var parsed = KeyFactory.getInstance("RSA").generatePublic(new X509EncodedKeySpec(der));
+      if (!(parsed instanceof RSAPublicKey rsaKey)) {
+        throw new IllegalStateException("ADMIN_JWT_PUBLIC_KEY must be RSA");
+      }
+      if (rsaKey.getModulus().bitLength() < MINIMUM_RSA_BITS) {
+        throw new IllegalStateException("ADMIN_JWT_PUBLIC_KEY must be at least 2048 bits");
+      }
+      return rsaKey;
+    } catch (GeneralSecurityException | IllegalArgumentException exception) {
+      throw new IllegalStateException("ADMIN_JWT_PUBLIC_KEY is malformed", exception);
+    }
+  }
+}


## `test(security): verify RSA key strength and parsing`

diff --git a/src/test/java/com/sportsbook/admin/security/RsaPublicKeyParserTest.java b/src/test/java/com/sportsbook/admin/security/RsaPublicKeyParserTest.java
new file mode 100644
index 0000000..164cc1a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/RsaPublicKeyParserTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.security.KeyPair;
+import java.security.KeyPairGenerator;
+import java.security.PublicKey;
+import java.security.interfaces.RSAPublicKey;
+import java.util.Base64;
+import org.junit.jupiter.api.Test;
+
+class RsaPublicKeyParserTest {
+
+  private final RsaPublicKeyParser parser = new RsaPublicKeyParser();
+
+  @Test
+  void acceptsA2048BitSpkiKeyWithRealOrEscapedNewlines() throws Exception {
+    RSAPublicKey key = (RSAPublicKey) rsaKeyPair(2048).getPublic();
+    String pem = pem(key);
+
+    assertThat(parser.parse(pem)).isEqualTo(key);
+    assertThat(parser.parse(pem.replace("\n", "\\n"))).isEqualTo(key);
+  }
+
+  @Test
+  void rejectsMissingMalformedAndWrongPemMarkersWithoutEchoingTheKey() {
+    String secret = "-----BEGIN RSA PUBLIC KEY-----\nsensitive\n-----END RSA PUBLIC KEY-----";
+
+    assertThatThrownBy(() -> parser.parse(null)).isInstanceOf(IllegalStateException.class);
+    assertThatThrownBy(() -> parser.parse("not-a-key"))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageNotContaining("not-a-key");
+    assertThatThrownBy(() -> parser.parse(secret))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageNotContaining(secret);
+  }
+
+  @Test
+  void rejectsRsaKeysSmallerThan2048Bits() throws Exception {
+    String weakKey = pem(rsaKeyPair(1024).getPublic());
+
+    assertThatThrownBy(() -> parser.parse(weakKey))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("at least 2048 bits")
+        .hasMessageNotContaining(weakKey);
+  }
+
+  private static KeyPair rsaKeyPair(int bits) throws Exception {
+    KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
+    generator.initialize(bits);
+    return generator.generateKeyPair();
+  }
+
+  private static String pem(PublicKey key) {
+    String body = Base64.getEncoder().encodeToString(key.getEncoded());
+    return "-----BEGIN PUBLIC KEY-----\n" + body + "\n-----END PUBLIC KEY-----";
+  }
+}


## `feat(security): require bounded JWT lifetime`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminJwtTimestampValidator.java b/src/main/java/com/sportsbook/admin/security/AdminJwtTimestampValidator.java
new file mode 100644
index 0000000..c2f8fd2
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminJwtTimestampValidator.java
@@ -0,0 +1,34 @@
+package com.sportsbook.admin.security;
+
+import java.time.Clock;
+import java.time.Duration;
+import org.springframework.security.oauth2.core.OAuth2Error;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtTimestampValidator;
+
+final class AdminJwtTimestampValidator implements OAuth2TokenValidator<Jwt> {
+
+  private static final OAuth2Error MISSING_EXPIRY =
+      new OAuth2Error("invalid_token", "The exp claim is required", null);
+
+  private final JwtTimestampValidator timestamps;
+
+  AdminJwtTimestampValidator() {
+    this(Clock.systemUTC());
+  }
+
+  AdminJwtTimestampValidator(Clock clock) {
+    timestamps = new JwtTimestampValidator(Duration.ZERO);
+    timestamps.setClock(clock);
+  }
+
+  @Override
+  public OAuth2TokenValidatorResult validate(Jwt jwt) {
+    if (jwt.getExpiresAt() == null) {
+      return OAuth2TokenValidatorResult.failure(MISSING_EXPIRY);
+    }
+    return timestamps.validate(jwt);
+  }
+}


## `test(security): reject expired and future JWTs`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminJwtTimestampValidatorTest.java b/src/test/java/com/sportsbook/admin/security/AdminJwtTimestampValidatorTest.java
new file mode 100644
index 0000000..b776532
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminJwtTimestampValidatorTest.java
@@ -0,0 +1,52 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+class AdminJwtTimestampValidatorTest {
+
+  private static final Instant NOW = Instant.parse("2026-08-23T00:00:00Z");
+  private final AdminJwtTimestampValidator validator =
+      new AdminJwtTimestampValidator(Clock.fixed(NOW, ZoneOffset.UTC));
+
+  @Test
+  void acceptsOnlyATokenThatIsCurrentlyValid() {
+    Jwt valid = token(NOW.minusSeconds(1), NOW.plusSeconds(60), NOW.minusSeconds(1));
+
+    assertThat(validator.validate(valid).hasErrors()).isFalse();
+  }
+
+  @Test
+  void rejectsMissingOrExpiredExpiryWithoutClockSkew() {
+    Jwt missingExpiry = token(NOW.minusSeconds(1), null, NOW.minusSeconds(1));
+    Jwt expired = token(NOW.minusSeconds(60), NOW.minusSeconds(1), NOW.minusSeconds(60));
+
+    assertThat(validator.validate(missingExpiry).hasErrors()).isTrue();
+    assertThat(validator.validate(expired).hasErrors()).isTrue();
+  }
+
+  @Test
+  void rejectsTokensThatAreNotYetValid() {
+    Jwt future = token(NOW, NOW.plusSeconds(120), NOW.plusSeconds(1));
+
+    assertThat(validator.validate(future).hasErrors()).isTrue();
+  }
+
+  private static Jwt token(Instant issuedAt, Instant expiresAt, Instant notBefore) {
+    Jwt.Builder builder =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .subject("operator-1")
+            .issuedAt(issuedAt)
+            .notBefore(notBefore);
+    if (expiresAt != null) {
+      builder.expiresAt(expiresAt);
+    }
+    return builder.build();
+  }
+}


## `feat(security): validate operator subjects`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminJwtSubjectValidator.java b/src/main/java/com/sportsbook/admin/security/AdminJwtSubjectValidator.java
new file mode 100644
index 0000000..d0d87a5
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminJwtSubjectValidator.java
@@ -0,0 +1,26 @@
+package com.sportsbook.admin.security;
+
+import org.springframework.security.oauth2.core.OAuth2Error;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+final class AdminJwtSubjectValidator implements OAuth2TokenValidator<Jwt> {
+
+  private static final int MAXIMUM_SUBJECT_LENGTH = 128;
+  private static final OAuth2Error INVALID_SUBJECT =
+      new OAuth2Error("invalid_token", "The sub claim is invalid", null);
+
+  @Override
+  public OAuth2TokenValidatorResult validate(Jwt jwt) {
+    String subject = jwt.getSubject();
+    if (subject == null
+        || subject.isBlank()
+        || !subject.equals(subject.trim())
+        || subject.codePointCount(0, subject.length()) > MAXIMUM_SUBJECT_LENGTH
+        || subject.codePoints().anyMatch(Character::isISOControl)) {
+      return OAuth2TokenValidatorResult.failure(INVALID_SUBJECT);
+    }
+    return OAuth2TokenValidatorResult.success();
+  }
+}


## `test(security): reject missing and unbounded subjects`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminJwtSubjectValidatorTest.java b/src/test/java/com/sportsbook/admin/security/AdminJwtSubjectValidatorTest.java
new file mode 100644
index 0000000..dccdd61
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminJwtSubjectValidatorTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+class AdminJwtSubjectValidatorTest {
+
+  private final AdminJwtSubjectValidator validator = new AdminJwtSubjectValidator();
+
+  @Test
+  void acceptsBoundedNonBlankSubjects() {
+    assertThat(validator.validate(token("a")).hasErrors()).isFalse();
+    assertThat(validator.validate(token("x".repeat(128))).hasErrors()).isFalse();
+  }
+
+  @Test
+  void rejectsMissingBlankOrOversizedSubjects() {
+    assertThat(validator.validate(token(null)).hasErrors()).isTrue();
+    assertThat(validator.validate(token("")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("   ")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("x".repeat(129))).hasErrors()).isTrue();
+  }
+
+  @Test
+  void rejectsWhitespaceEdgesAndControlCharacters() {
+    assertThat(validator.validate(token(" operator")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("operator ")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("operator\nadmin")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("operator\u0000admin")).hasErrors()).isTrue();
+  }
+
+  private static Jwt token(String subject) {
+    Jwt.Builder builder = Jwt.withTokenValue("token").header("alg", "RS256").claim("role", "ADMIN");
+    if (subject != null) {
+      builder.subject(subject);
+    }
+    return builder.build();
+  }
+}


## `feat(security): validate operator role claims`

diff --git a/src/main/java/com/sportsbook/admin/security/AdminJwtRoleValidator.java b/src/main/java/com/sportsbook/admin/security/AdminJwtRoleValidator.java
new file mode 100644
index 0000000..1dda451
--- /dev/null
+++ b/src/main/java/com/sportsbook/admin/security/AdminJwtRoleValidator.java
@@ -0,0 +1,19 @@
+package com.sportsbook.admin.security;
+
+import org.springframework.security.oauth2.core.OAuth2Error;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+final class AdminJwtRoleValidator implements OAuth2TokenValidator<Jwt> {
+
+  private static final OAuth2Error INVALID_ROLE =
+      new OAuth2Error("invalid_token", "The role claim is invalid", null);
+
+  @Override
+  public OAuth2TokenValidatorResult validate(Jwt jwt) {
+    return AdminRole.fromClaim(jwt.getClaims().get("role")).isPresent()
+        ? OAuth2TokenValidatorResult.success()
+        : OAuth2TokenValidatorResult.failure(INVALID_ROLE);
+  }
+}


## `test(security): reject missing and unknown roles`

diff --git a/src/test/java/com/sportsbook/admin/security/AdminJwtRoleValidatorTest.java b/src/test/java/com/sportsbook/admin/security/AdminJwtRoleValidatorTest.java
new file mode 100644
index 0000000..90c35e4
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/security/AdminJwtRoleValidatorTest.java
@@ -0,0 +1,37 @@
+package com.sportsbook.admin.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.List;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+class AdminJwtRoleValidatorTest {
+
+  private final AdminJwtRoleValidator validator = new AdminJwtRoleValidator();
+
+  @Test
+  void acceptsEachExactOperatorRole() {
+    for (AdminRole role : AdminRole.values()) {
+      assertThat(validator.validate(token(role.name())).hasErrors()).isFalse();
+    }
+  }
+
+  @Test
+  void rejectsMissingUnknownAndNonStringRoles() {
+    assertThat(validator.validate(token(null)).hasErrors()).isTrue();
+    assertThat(validator.validate(token("")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("admin")).hasErrors()).isTrue();
+    assertThat(validator.validate(token(" ADMIN ")).hasErrors()).isTrue();
+    assertThat(validator.validate(token("OWNER")).hasErrors()).isTrue();
+    assertThat(validator.validate(token(List.of("ADMIN"))).hasErrors()).isTrue();
+  }
+
+  private static Jwt token(Object role) {
+    Jwt.Builder builder = Jwt.withTokenValue("token").header("alg", "RS256").subject("operator-1");
+    if (role != null) {
+      builder.claim("role", role);
+    }
+    return builder.build();
+  }
+}


