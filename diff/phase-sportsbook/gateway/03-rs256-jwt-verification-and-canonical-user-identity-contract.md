# RS256 JWT 검증과 정규형 사용자 신원 계약

## `feat(security): verify RS256 user tokens`

diff --git a/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java b/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java
new file mode 100644
index 0000000..dc559f5
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java
@@ -0,0 +1,35 @@
+package com.sportsbook.gateway.security;
+
+import java.security.interfaces.RSAPublicKey;
+import java.time.Duration;
+import java.util.Objects;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.jose.jws.SignatureAlgorithm;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.jwt.JwtClaimNames;
+import org.springframework.security.oauth2.jwt.JwtClaimValidator;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.security.oauth2.jwt.JwtTimestampValidator;
+import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
+
+@Configuration(proxyBeanMethods = false)
+@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
+class JwtDecoderConfiguration {
+
+  @Bean
+  JwtDecoder jwtDecoder(JwtSecurityProperties properties) {
+    RSAPublicKey key = new RsaPublicKeyLoader().load(properties.publicKey());
+    NimbusJwtDecoder decoder =
+        NimbusJwtDecoder.withPublicKey(key).signatureAlgorithm(SignatureAlgorithm.RS256).build();
+
+    OAuth2TokenValidator<Jwt> timestamps = new JwtTimestampValidator(Duration.ZERO);
+    OAuth2TokenValidator<Jwt> requiredExpiry =
+        new JwtClaimValidator<>(JwtClaimNames.EXP, Objects::nonNull);
+    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(timestamps, requiredExpiry));
+    return decoder;
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/security/JwtSecurityProperties.java b/src/main/java/com/sportsbook/gateway/security/JwtSecurityProperties.java
new file mode 100644
index 0000000..f63ecfe
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/JwtSecurityProperties.java
@@ -0,0 +1,6 @@
+package com.sportsbook.gateway.security;
+
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@ConfigurationProperties("gateway.security.jwt")
+public record JwtSecurityProperties(String publicKey) {}
diff --git a/src/main/java/com/sportsbook/gateway/security/RsaPublicKeyLoader.java b/src/main/java/com/sportsbook/gateway/security/RsaPublicKeyLoader.java
new file mode 100644
index 0000000..94274a8
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/RsaPublicKeyLoader.java
@@ -0,0 +1,39 @@
+package com.sportsbook.gateway.security;
+
+import java.security.GeneralSecurityException;
+import java.security.KeyFactory;
+import java.security.interfaces.RSAPublicKey;
+import java.security.spec.X509EncodedKeySpec;
+import java.util.Base64;
+
+final class RsaPublicKeyLoader {
+
+  private static final String BEGIN = "-----BEGIN PUBLIC KEY-----";
+  private static final String END = "-----END PUBLIC KEY-----";
+  private static final int MINIMUM_RSA_BITS = 2048;
+
+  RSAPublicKey load(String configuredKey) {
+    if (configuredKey == null || configuredKey.isBlank()) {
+      throw new IllegalStateException("GATEWAY_JWT_PUBLIC_KEY is required");
+    }
+
+    String pem = configuredKey.replace("\\n", "\n").trim();
+    if (!pem.startsWith(BEGIN) || !pem.endsWith(END)) {
+      throw new IllegalStateException("GATEWAY_JWT_PUBLIC_KEY must contain an RSA public key");
+    }
+
+    try {
+      String encoded =
+          pem.substring(BEGIN.length(), pem.length() - END.length()).replaceAll("\\s+", "");
+      byte[] der = Base64.getDecoder().decode(encoded);
+      RSAPublicKey key =
+          (RSAPublicKey) KeyFactory.getInstance("RSA").generatePublic(new X509EncodedKeySpec(der));
+      if (key.getModulus().bitLength() < MINIMUM_RSA_BITS) {
+        throw new IllegalStateException("GATEWAY_JWT_PUBLIC_KEY must be at least 2048 bits");
+      }
+      return key;
+    } catch (GeneralSecurityException | IllegalArgumentException exception) {
+      throw new IllegalStateException("GATEWAY_JWT_PUBLIC_KEY is malformed", exception);
+    }
+  }
+}
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index d936d86..5da8848 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -7,3 +7,8 @@ spring:
 server:
   port: ${GATEWAY_HTTP_PORT:8080}
   shutdown: graceful
+
+gateway:
+  security:
+    jwt:
+      public-key: ${GATEWAY_JWT_PUBLIC_KEY:}


## `test(security): verify token key and lifetime checks`

diff --git a/src/test/java/com/sportsbook/gateway/security/JwtVerificationTest.java b/src/test/java/com/sportsbook/gateway/security/JwtVerificationTest.java
new file mode 100644
index 0000000..7a4b0ad
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/security/JwtVerificationTest.java
@@ -0,0 +1,99 @@
+package com.sportsbook.gateway.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.nimbusds.jose.JWSAlgorithm;
+import com.nimbusds.jose.JWSHeader;
+import com.nimbusds.jose.crypto.MACSigner;
+import com.nimbusds.jose.crypto.RSASSASigner;
+import com.nimbusds.jwt.JWTClaimsSet;
+import com.nimbusds.jwt.PlainJWT;
+import com.nimbusds.jwt.SignedJWT;
+import java.security.KeyPair;
+import java.security.KeyPairGenerator;
+import java.security.interfaces.RSAPrivateKey;
+import java.time.Instant;
+import java.util.Base64;
+import java.util.Date;
+import java.util.UUID;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.security.oauth2.jwt.JwtDecoder;
+import org.springframework.security.oauth2.jwt.JwtException;
+
+class JwtVerificationTest {
+
+  private KeyPair trusted;
+  private JwtDecoder decoder;
+
+  @BeforeEach
+  void setUp() throws Exception {
+    trusted = keyPair();
+    decoder = new JwtDecoderConfiguration().jwtDecoder(new JwtSecurityProperties(pem(trusted)));
+  }
+
+  @Test
+  void requiresWellFormedPublicKey() {
+    RsaPublicKeyLoader loader = new RsaPublicKeyLoader();
+    assertThatThrownBy(() -> loader.load(" ")).isInstanceOf(IllegalStateException.class);
+    String corrupted = pem(trusted).replaceFirst("\n", "\n!@#");
+    assertThatThrownBy(() -> loader.load(corrupted))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageNotContaining(corrupted);
+  }
+
+  @Test
+  void acceptsValidRs256Token() throws Exception {
+    assertThat(decoder.decode(sign(trusted, claims(future()))).getSubject()).isNotBlank();
+  }
+
+  @Test
+  void rejectsWrongSigningKey() throws Exception {
+    String token = sign(keyPair(), claims(future()));
+    assertThatThrownBy(() -> decoder.decode(token)).isInstanceOf(JwtException.class);
+  }
+
+  @Test
+  void requiresUnexpiredLifetime() throws Exception {
+    String expired = sign(trusted, claims(Date.from(Instant.now().minusSeconds(1))));
+    String missing = sign(trusted, claims(null));
+    assertThatThrownBy(() -> decoder.decode(expired)).isInstanceOf(JwtException.class);
+    assertThatThrownBy(() -> decoder.decode(missing)).isInstanceOf(JwtException.class);
+  }
+
+  @Test
+  void rejectsUnsignedAndHmacTokens() throws Exception {
+    String unsigned = new PlainJWT(claims(future())).serialize();
+    SignedJWT hmac = new SignedJWT(new JWSHeader(JWSAlgorithm.HS256), claims(future()));
+    hmac.sign(new MACSigner(new byte[32]));
+    assertThatThrownBy(() -> decoder.decode(unsigned)).isInstanceOf(JwtException.class);
+    assertThatThrownBy(() -> decoder.decode(hmac.serialize())).isInstanceOf(JwtException.class);
+  }
+
+  private static JWTClaimsSet claims(Date expiry) {
+    JWTClaimsSet.Builder builder = new JWTClaimsSet.Builder().subject(UUID.randomUUID().toString());
+    return expiry == null ? builder.build() : builder.expirationTime(expiry).build();
+  }
+
+  private static String sign(KeyPair pair, JWTClaimsSet claims) throws Exception {
+    SignedJWT jwt = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
+    jwt.sign(new RSASSASigner((RSAPrivateKey) pair.getPrivate()));
+    return jwt.serialize();
+  }
+
+  private static KeyPair keyPair() throws Exception {
+    KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
+    generator.initialize(2048);
+    return generator.generateKeyPair();
+  }
+
+  private static Date future() {
+    return Date.from(Instant.now().plusSeconds(300));
+  }
+
+  private static String pem(KeyPair pair) {
+    String body = Base64.getEncoder().encodeToString(pair.getPublic().getEncoded());
+    return "-----BEGIN PUBLIC KEY-----\n" + body + "\n-----END PUBLIC KEY-----";
+  }
+}


## `feat(security): require canonical user claims`

diff --git a/src/main/java/com/sportsbook/gateway/security/GatewayClaimsValidator.java b/src/main/java/com/sportsbook/gateway/security/GatewayClaimsValidator.java
new file mode 100644
index 0000000..c1125c8
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/GatewayClaimsValidator.java
@@ -0,0 +1,50 @@
+package com.sportsbook.gateway.security;
+
+import java.util.HashSet;
+import java.util.List;
+import java.util.UUID;
+import java.util.regex.Pattern;
+import org.springframework.security.oauth2.core.OAuth2Error;
+import org.springframework.security.oauth2.core.OAuth2TokenValidator;
+import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+final class GatewayClaimsValidator implements OAuth2TokenValidator<Jwt> {
+
+  private static final int MAXIMUM_ROLES = 16;
+  private static final Pattern ROLE = Pattern.compile("[A-Z][A-Z0-9_]{0,31}");
+  private static final OAuth2Error INVALID_SUBJECT =
+      new OAuth2Error("invalid_token", "sub must be a canonical UUID", null);
+  private static final OAuth2Error INVALID_ROLES =
+      new OAuth2Error("invalid_token", "roles must be a bounded unique string array", null);
+
+  @Override
+  public OAuth2TokenValidatorResult validate(Jwt jwt) {
+    if (!isCanonicalUuid(jwt.getSubject())) {
+      return OAuth2TokenValidatorResult.failure(INVALID_SUBJECT);
+    }
+    Object rolesClaim = jwt.getClaims().get("roles");
+    if (rolesClaim == null) {
+      return OAuth2TokenValidatorResult.success();
+    }
+    if (!(rolesClaim instanceof List<?> roles)
+        || roles.size() > MAXIMUM_ROLES
+        || roles.stream()
+            .anyMatch(role -> !(role instanceof String value) || !ROLE.matcher(value).matches())
+        || new HashSet<>(roles).size() != roles.size()) {
+      return OAuth2TokenValidatorResult.failure(INVALID_ROLES);
+    }
+    return OAuth2TokenValidatorResult.success();
+  }
+
+  private static boolean isCanonicalUuid(String subject) {
+    if (subject == null || subject.isBlank()) {
+      return false;
+    }
+    try {
+      return UUID.fromString(subject).toString().equals(subject);
+    } catch (IllegalArgumentException exception) {
+      return false;
+    }
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/security/GatewayJwtAuthenticationConverter.java b/src/main/java/com/sportsbook/gateway/security/GatewayJwtAuthenticationConverter.java
new file mode 100644
index 0000000..88dc4bf
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/security/GatewayJwtAuthenticationConverter.java
@@ -0,0 +1,26 @@
+package com.sportsbook.gateway.security;
+
+import java.util.Collection;
+import java.util.List;
+import org.springframework.core.convert.converter.Converter;
+import org.springframework.security.authentication.AbstractAuthenticationToken;
+import org.springframework.security.core.GrantedAuthority;
+import org.springframework.security.core.authority.SimpleGrantedAuthority;
+import org.springframework.security.oauth2.jwt.Jwt;
+import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
+
+final class GatewayJwtAuthenticationConverter
+    implements Converter<Jwt, AbstractAuthenticationToken> {
+
+  @Override
+  public AbstractAuthenticationToken convert(Jwt jwt) {
+    List<String> roles = jwt.getClaimAsStringList("roles");
+    Collection<GrantedAuthority> authorities =
+        roles == null
+            ? List.of()
+            : roles.stream()
+                .<GrantedAuthority>map(role -> new SimpleGrantedAuthority("ROLE_" + role))
+                .toList();
+    return new JwtAuthenticationToken(jwt, authorities, jwt.getSubject());
+  }
+}
diff --git a/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java b/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java
index dc559f5..5177ebf 100644
--- a/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java
+++ b/src/main/java/com/sportsbook/gateway/security/JwtDecoderConfiguration.java
@@ -29,7 +29,9 @@ class JwtDecoderConfiguration {
     OAuth2TokenValidator<Jwt> timestamps = new JwtTimestampValidator(Duration.ZERO);
     OAuth2TokenValidator<Jwt> requiredExpiry =
         new JwtClaimValidator<>(JwtClaimNames.EXP, Objects::nonNull);
-    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(timestamps, requiredExpiry));
+    decoder.setJwtValidator(
+        new DelegatingOAuth2TokenValidator<>(
+            timestamps, requiredExpiry, new GatewayClaimsValidator()));
     return decoder;
   }
 }


## `test(security): reject incomplete user identities`

diff --git a/src/test/java/com/sportsbook/gateway/security/GatewayClaimsValidatorTest.java b/src/test/java/com/sportsbook/gateway/security/GatewayClaimsValidatorTest.java
new file mode 100644
index 0000000..62cf5f1
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/security/GatewayClaimsValidatorTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.gateway.security;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.List;
+import java.util.stream.IntStream;
+import java.util.stream.Stream;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.MethodSource;
+import org.junit.jupiter.params.provider.NullAndEmptySource;
+import org.junit.jupiter.params.provider.ValueSource;
+import org.springframework.security.authentication.AbstractAuthenticationToken;
+import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
+import org.springframework.security.oauth2.jwt.Jwt;
+
+class GatewayClaimsValidatorTest {
+
+  private static final String SUBJECT = "123e4567-e89b-12d3-a456-426614174000";
+  private final GatewayClaimsValidator validator = new GatewayClaimsValidator();
+
+  @ParameterizedTest
+  @NullAndEmptySource
+  @ValueSource(strings = {"not-a-uuid", "1-1-1-1-1", "123E4567-E89B-12D3-A456-426614174000"})
+  void rejectsMissingOrNonCanonicalSubject(String subject) {
+    assertThat(validator.validate(jwt(subject, null)).hasErrors()).isTrue();
+  }
+
+  @ParameterizedTest
+  @MethodSource("invalidRoles")
+  void rejectsMalformedRoles(Object roles) {
+    assertThat(validator.validate(jwt(SUBJECT, roles)).hasErrors()).isTrue();
+  }
+
+  @Test
+  void acceptsIdentityWithoutRoles() {
+    OAuth2TokenValidatorResult result = validator.validate(jwt(SUBJECT, null));
+    assertThat(result.hasErrors()).isFalse();
+  }
+
+  @Test
+  void mapsCanonicalIdentityAndRoles() {
+    Jwt jwt = jwt(SUBJECT, List.of("USER", "BET_OPERATOR"));
+    AbstractAuthenticationToken authentication =
+        new GatewayJwtAuthenticationConverter().convert(jwt);
+
+    assertThat(validator.validate(jwt).hasErrors()).isFalse();
+    assertThat(authentication.getName()).isEqualTo(SUBJECT);
+    assertThat(authentication.getAuthorities())
+        .extracting("authority")
+        .containsExactly("ROLE_USER", "ROLE_BET_OPERATOR");
+  }
+
+  private static Stream<Object> invalidRoles() {
+    return Stream.of(
+        "USER",
+        List.of("USER", "USER"),
+        List.of("user"),
+        List.of("UNSAFE-ROLE"),
+        List.of("A".repeat(33)),
+        List.of(1),
+        IntStream.range(0, 17).mapToObj(index -> "R" + index).toList());
+  }
+
+  private static Jwt jwt(String subject, Object roles) {
+    Jwt.Builder builder =
+        Jwt.withTokenValue("token")
+            .header("alg", "RS256")
+            .issuedAt(Instant.now())
+            .expiresAt(Instant.now().plusSeconds(300));
+    if (subject != null) {
+      builder.subject(subject);
+    }
+    if (roles != null) {
+      builder.claim("roles", roles);
+    }
+    return builder.build();
+  }
+}
