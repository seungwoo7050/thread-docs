## `test(api): verify operator request failures`

diff --git a/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java b/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java
index 738b0d3..8af4a54 100644
--- a/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java
+++ b/src/test/java/com/sportsbook/oddsfeed/api/MarketAdminControllerTest.java
@@ -2,13 +2,16 @@ package com.sportsbook.oddsfeed.api;
 
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.doThrow;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
 import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
 
 import com.sportsbook.oddsfeed.config.InternalSecurityProperties;
+import com.sportsbook.oddsfeed.delivery.IdempotencyConflictException;
 import com.sportsbook.oddsfeed.delivery.OperatorActionQueue;
+import com.sportsbook.oddsfeed.delivery.TerminalMarketReopenException;
 import com.sportsbook.oddsfeed.security.InternalApiKeyAuthenticationFilter;
 import com.sportsbook.oddsfeed.security.SecurityConfig;
 import com.sportsbook.protocol.event.MarketStatus;
@@ -70,16 +73,76 @@ class MarketAdminControllerTest {
     verify(queue).submit(any(), eq(reopenId), any(), any(), eq(MarketStatus.OPEN), any(), any());
   }
 
+  @Test
+  void malformedIdempotencyAndReasonsReturnBadRequest() throws Exception {
+    UUID eventId = UUID.randomUUID();
+    UUID marketId = UUID.randomUUID();
+
+    perform(eventId, marketId, UUID.randomUUID(), "suspend", "incident", " ")
+        .andExpect(status().isBadRequest());
+    perform(eventId, marketId, UUID.randomUUID(), "suspend", " ", "valid-key")
+        .andExpect(status().isBadRequest());
+    perform(eventId, marketId, UUID.randomUUID(), "suspend", "x".repeat(257), "valid-key")
+        .andExpect(status().isBadRequest());
+  }
+
+  @Test
+  void reasonIsNormalizedBeforeSubmission() throws Exception {
+    UUID actionId = UUID.randomUUID();
+
+    perform(
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            actionId,
+            "suspend",
+            "  incident  ",
+            "normalized-key")
+        .andExpect(status().isAccepted());
+
+    verify(queue).submit(any(), eq(actionId), any(), any(), any(), eq("incident"), eq(NOW));
+  }
+
+  @Test
+  void identityConflictReturnsConflict() throws Exception {
+    doThrow(new IdempotencyConflictException())
+        .when(queue)
+        .submit(any(), any(), any(), any(), any(), any(), any());
+
+    perform(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), "close")
+        .andExpect(status().isConflict());
+  }
+
+  @Test
+  void terminalReopenReturnsConflict() throws Exception {
+    doThrow(new TerminalMarketReopenException())
+        .when(queue)
+        .submit(any(), any(), any(), any(), any(), any(), any());
+
+    perform(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), "reopen")
+        .andExpect(status().isConflict());
+  }
+
   private ResultActions perform(UUID eventId, UUID marketId, UUID actionId, String action)
       throws Exception {
+    return perform(eventId, marketId, actionId, action, "incident", "operator-request");
+  }
+
+  private ResultActions perform(
+      UUID eventId,
+      UUID marketId,
+      UUID actionId,
+      String action,
+      String reason,
+      String idempotencyKey)
+      throws Exception {
     String path = "/internal/v1/events/" + eventId + "/markets/" + marketId + "/" + action;
     return mockMvc.perform(
         post(path)
             .header(InternalApiKeyAuthenticationFilter.SERVICE_HEADER, "admin-api")
             .header(InternalApiKeyAuthenticationFilter.API_KEY_HEADER, API_KEY)
-            .header(MarketAdminController.IDEMPOTENCY_HEADER, "operator-request")
+            .header(MarketAdminController.IDEMPOTENCY_HEADER, idempotencyKey)
             .header(MarketAdminController.ACTION_ID_HEADER, actionId)
             .contentType(MediaType.APPLICATION_JSON)
-            .content("{\"reason\":\"incident\"}"));
+            .content("{\"reason\":\"" + reason + "\"}"));
   }
 }
