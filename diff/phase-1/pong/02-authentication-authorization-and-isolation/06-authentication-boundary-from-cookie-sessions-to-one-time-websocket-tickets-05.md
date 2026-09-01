## `fix(realtime): WebSocket transport payload 상한 설정`

diff --git a/apps/api/src/app.ts b/apps/api/src/app.ts
index 1f2a824..941db2f 100644
--- a/apps/api/src/app.ts
+++ b/apps/api/src/app.ts
@@ -122,7 +122,7 @@ export function buildApp({
   });
   app.register(cookie);
   app.register(async (realtime) => {
-    await realtime.register(websocket);
+    await realtime.register(websocket, { options: { maxPayload: PRE_AUTH_MESSAGE_MAX_BYTES } });
     realtime.get("/ws", { websocket: true }, (socket, request) => {
       const pendingPayloads: string[] = [];
       let pendingBytes = 0;
diff --git a/apps/api/src/ws-ticket.test.ts b/apps/api/src/ws-ticket.test.ts
index cc357a4..c8f139c 100644
--- a/apps/api/src/ws-ticket.test.ts
+++ b/apps/api/src/ws-ticket.test.ts
@@ -145,7 +145,7 @@ describe("one-time websocket tickets", () => {
     try {
       const closed = closeDetails(socket);
       socket.send(Buffer.alloc(8 * 1024 + 1));
-      expect(await closed).toEqual({ code: 1009, reason: "pre-auth payload too large" });
+      expect(await closed).toEqual({ code: 1009, reason: "" });
     } finally {
       releaseAuthentication();
     }
