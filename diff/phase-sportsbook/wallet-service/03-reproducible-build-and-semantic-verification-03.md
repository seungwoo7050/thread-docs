## `test(gate): recover blocked adjustments end to end`

diff --git a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
index 0c6cf98..1b4ad9b 100644
--- a/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
+++ b/src/test/java/com/sportsbook/wallet/smoke/WalletSmokeTest.java
@@ -7,6 +7,7 @@ import com.sportsbook.wallet.domain.WalletCaller;
 import com.sportsbook.wallet.outbox.KafkaOutboxDispatcher;
 import com.sportsbook.wallet.outbox.WalletEventFactory;
 import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
+import com.sportsbook.wallet.service.RecoveryWorker;
 import java.nio.charset.StandardCharsets;
 import java.time.Duration;
 import java.util.List;
@@ -35,6 +36,7 @@ class WalletSmokeTest extends WalletSmokeFixture {
   @Autowired StringRedisTemplate redis;
   @Autowired OutboxDeliveryRepository outbox;
   @Autowired KafkaOutboxDispatcher dispatcher;
+  @Autowired RecoveryWorker recovery;
 
   @Test
   void servesAuthenticatedDurableReplayAcrossPostgresAndRedis() throws Exception {
@@ -145,6 +147,76 @@ class WalletSmokeTest extends WalletSmokeFixture {
     }
   }
 
+  @Test
+  void wakesAndRecoversBlockedAdjustments() throws Exception {
+    UUID userId = UUID.fromString("019b783d-1000-7000-8000-000000000004");
+    UUID revisionId = UUID.fromString("019b783d-1000-7000-8000-000000000005");
+    UUID betId = UUID.fromString("019b783d-1000-7000-8000-000000000006");
+    String key = "settlement:revision:" + revisionId;
+    String proofPath = "/internal/v1/wallet/transactions/adjustment/" + revisionId;
+    String account = "{\"userId\":\"" + userId + "\",\"currency\":\"KRW\"}";
+    String adjustment =
+        """
+        {"revisionId":"%s","betId":"%s","revisionNumber":1,"userId":"%s",
+        "previousPayout":{"amount":500,"currency":"KRW"},
+        "newPayout":{"amount":200,"currency":"KRW"}}
+        """
+            .formatted(revisionId, betId, userId);
+    assertThat(
+            request(HttpMethod.POST, ACCOUNT_PATH, WalletCaller.PLATFORM, null, account)
+                .getStatusCode())
+        .isEqualTo(HttpStatus.OK);
+
+    var blocked =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/adjustment",
+            WalletCaller.SETTLEMENT,
+            key,
+            adjustment);
+    assertThat(blocked.getStatusCode()).isEqualTo(HttpStatus.ACCEPTED);
+    assertThat(blocked.getHeaders().getLocation()).hasToString(proofPath);
+    assertThat(json.readTree(blocked.getBody()).path("status").textValue()).isEqualTo("BLOCKED");
+    assertThat(count("ledger_entry", key)).isZero();
+
+    var funded =
+        request(
+            HttpMethod.POST,
+            "/internal/v1/wallet/transactions/deposit",
+            WalletCaller.PLATFORM,
+            "smoke:recovery:deposit",
+            transaction(userId, 300L));
+    assertThat(funded.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(recovery.recoverOne()).isEqualTo(RecoveryWorker.Result.APPLIED);
+
+    var proof = request(HttpMethod.GET, proofPath, WalletCaller.SETTLEMENT, null, null);
+    var recovered = json.readTree(proof.getBody());
+    assertThat(proof.getStatusCode()).isEqualTo(HttpStatus.OK);
+    assertThat(recovered.path("status").textValue()).isEqualTo("APPLIED");
+    assertThat(recovered.path("deltaAmount").longValue()).isEqualTo(-300L);
+    assertThat(recovered.path("operationGroupId").isTextual()).isTrue();
+    assertThat(recovered.path("appliedAt").isTextual()).isTrue();
+    assertThat(recovered.path("nextAttemptAt").isNull()).isTrue();
+    assertThat(
+            jdbc.queryForObject(
+                """
+                SELECT recovery_debt_amount=0 AND recovery_frozen_at IS NULL
+                FROM account WHERE user_id=?
+                """,
+                Boolean.class,
+                userId))
+        .isTrue();
+    assertThat(
+            jdbc.queryForObject(
+                """
+                SELECT COUNT(*) FROM ledger_entry
+                WHERE idempotency_key=? AND reason='BET_ADJUSTMENT'
+                """,
+                Integer.class,
+                key))
+        .isEqualTo(2);
+  }
+
   private int count(String table, String key) {
     return jdbc.queryForObject(
         "SELECT COUNT(*) FROM " + table + " WHERE idempotency_key=?", Integer.class, key);


## `build(gates): expose deterministic semantic verification`

diff --git a/pom.xml b/pom.xml
index e2ab963..02e5531 100644
--- a/pom.xml
+++ b/pom.xml
@@ -208,4 +208,22 @@
             </plugin>
         </plugins>
     </build>
+
+    <profiles>
+        <profile>
+            <id>semantic-gates</id>
+            <build>
+                <plugins>
+                    <plugin>
+                        <groupId>org.apache.maven.plugins</groupId>
+                        <artifactId>maven-surefire-plugin</artifactId>
+                        <configuration>
+                            <groups>wallet-semantic-gate</groups>
+                            <failIfNoTests>true</failIfNoTests>
+                        </configuration>
+                    </plugin>
+                </plugins>
+            </build>
+        </profile>
+    </profiles>
 </project>


## `ci(wallet): verify Java 17 builds`

diff --git a/.github/workflows/wallet-ci.yml b/.github/workflows/wallet-ci.yml
new file mode 100644
index 0000000..fd6de50
--- /dev/null
+++ b/.github/workflows/wallet-ci.yml
@@ -0,0 +1,45 @@
+name: Wallet CI
+
+on:
+  push:
+    branches:
+      - wallet-service
+  pull_request:
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    steps:
+      - name: Check out wallet service
+        uses: actions/checkout@v4
+        with:
+          path: wallet-service
+
+      - name: Check out shared protocol
+        uses: actions/checkout@v4
+        with:
+          repository: ${{ github.repository }}
+          ref: shared-protocol
+          path: shared-protocol
+
+      - name: Set up Java 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+
+      - name: Install shared protocol
+        working-directory: shared-protocol
+        run: ./mvnw -Dmaven.repo.local="${{ runner.temp }}/wallet-m2" -B -ntp clean install
+
+      - name: Verify wallet service
+        working-directory: wallet-service
+        run: ./mvnw -Dmaven.repo.local="${{ runner.temp }}/wallet-m2" -B -ntp clean verify
+
+      - name: Run wallet semantic gates
+        working-directory: wallet-service
+        run: ./mvnw -Dmaven.repo.local="${{ runner.temp }}/wallet-m2" -B -ntp -Psemantic-gates clean verify


## `build(release): release wallet service 1.0.0`

diff --git a/pom.xml b/pom.xml
index 02e5531..fcca924 100644
--- a/pom.xml
+++ b/pom.xml
@@ -13,7 +13,7 @@
 
     <groupId>com.sportsbook</groupId>
     <artifactId>wallet-service</artifactId>
-    <version>1.0.0-SNAPSHOT</version>
+    <version>1.0.0</version>
     <name>wallet-service</name>
     <description>Authoritative account and double-entry ledger service.</description>
 
