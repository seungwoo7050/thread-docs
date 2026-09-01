## `fix(receipt): reject partial transactions`

diff --git a/src/audit.js b/src/audit.js
index 47a693a..0035ff6 100644
--- a/src/audit.js
+++ b/src/audit.js
@@ -1,4 +1,4 @@
-import { mkdir, readFile, rename, rm, writeFile } from "node:fs/promises";
+import { mkdir, readFile, rename, rm, stat, writeFile } from "node:fs/promises";
 import path from "node:path";
 import { fileURLToPath } from "node:url";
 
@@ -83,10 +83,20 @@ export async function readPublicationSuccess(root, receiptId) {
     throw new AuditIntegrityError("INVALID_RECEIPT_ID", "Publication receipt identity is invalid");
   }
   const relativePath = `.publisher/publications/${receiptId}`;
+  const directory = path.join(root, relativePath);
+  try {
+    await stat(directory);
+  } catch (error) {
+    if (error.code === "ENOENT") return undefined;
+    throw new AuditIntegrityError(
+      "PUBLICATION_TRANSACTION_INVALID",
+      "Publication transaction could not be inspected",
+    );
+  }
   try {
     const [receipt, event] = await Promise.all([
-      readFile(path.join(root, relativePath, "receipt.json"), "utf8").then(JSON.parse),
-      readFile(path.join(root, relativePath, "event.json"), "utf8").then(JSON.parse),
+      readFile(path.join(directory, "receipt.json"), "utf8").then(JSON.parse),
+      readFile(path.join(directory, "event.json"), "utf8").then(JSON.parse),
     ]);
     validatePublicationReceipt(receipt);
     validateAuditEvent(event);
@@ -104,7 +114,6 @@ export async function readPublicationSuccess(root, receiptId) {
     }
     return Object.freeze({ event: Object.freeze(event), receipt: Object.freeze(receipt), relativePath });
   } catch (error) {
-    if (error.code === "ENOENT") return undefined;
     if (error instanceof AuditIntegrityError) throw error;
     throw new AuditIntegrityError(
       "PUBLICATION_TRANSACTION_INVALID",
diff --git a/test/audit.test.js b/test/audit.test.js
index 38f4e7b..d6232e5 100644
--- a/test/audit.test.js
+++ b/test/audit.test.js
@@ -1,5 +1,5 @@
 import assert from "node:assert/strict";
-import { mkdtemp, readFile, rm } from "node:fs/promises";
+import { mkdir, mkdtemp, readFile, rm } from "node:fs/promises";
 import os from "node:os";
 import path from "node:path";
 import test from "node:test";
@@ -84,3 +84,14 @@ test("atomically stores a succeeded event with its receipt", async (context) =>
   const retried = await writePublicationSuccess(root, receipt, succeeded, "retry");
   assert.deepEqual(retried.receipt, receipt);
 });
+
+test("fails closed on a partial publication transaction", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-audit-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const receiptId = "d".repeat(64);
+  await mkdir(path.join(root, ".publisher/publications", receiptId), { recursive: true });
+
+  await assert.rejects(() => readPublicationSuccess(root, receiptId), {
+    code: "PUBLICATION_TRANSACTION_INVALID",
+  });
+});
