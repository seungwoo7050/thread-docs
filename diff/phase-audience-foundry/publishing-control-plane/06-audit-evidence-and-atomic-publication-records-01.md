# 감사 증거와 원자적 게시 기록

## `feat(audit): define publication evidence contracts`

diff --git a/schemas/audit-event.schema.json b/schemas/audit-event.schema.json
new file mode 100644
index 0000000..336fce6
--- /dev/null
+++ b/schemas/audit-event.schema.json
@@ -0,0 +1,90 @@
+{
+  "$schema": "https://json-schema.org/draft/2020-12/schema",
+  "$id": "https://content-foundry.local/schemas/audit-event.schema.json",
+  "title": "Append-only Publisher audit event",
+  "oneOf": [
+    { "$ref": "#/$defs/approval" },
+    { "$ref": "#/$defs/publication" }
+  ],
+  "$defs": {
+    "common": {
+      "type": "object",
+      "required": [
+        "schema_version",
+        "event_id",
+        "event_type",
+        "article_id",
+        "article_path",
+        "source_sha",
+        "occurred_at"
+      ],
+      "properties": {
+        "schema_version": { "const": 1 },
+        "event_id": { "$ref": "#/$defs/uuid" },
+        "event_type": { "enum": ["approval", "publication"] },
+        "article_id": { "$ref": "#/$defs/identifier" },
+        "article_path": {
+          "type": "string",
+          "pattern": "^content/articles/[a-z0-9]+(?:-[a-z0-9]+)*\\.md$"
+        },
+        "source_sha": { "$ref": "#/$defs/sha" },
+        "occurred_at": { "type": "string", "format": "date-time" }
+      }
+    },
+    "approval": {
+      "allOf": [
+        { "$ref": "#/$defs/common" },
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["decision", "actor"],
+          "properties": {
+            "schema_version": true,
+            "event_id": true,
+            "event_type": { "const": "approval" },
+            "article_id": true,
+            "article_path": true,
+            "source_sha": true,
+            "occurred_at": true,
+            "decision": { "enum": ["approved", "rejected"] },
+            "actor": { "type": "string", "minLength": 1, "maxLength": 100 }
+          }
+        }
+      ]
+    },
+    "publication": {
+      "allOf": [
+        { "$ref": "#/$defs/common" },
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["attempt_id", "site_id", "engine", "phase"],
+          "properties": {
+            "schema_version": true,
+            "event_id": true,
+            "event_type": { "const": "publication" },
+            "article_id": true,
+            "article_path": true,
+            "source_sha": true,
+            "occurred_at": true,
+            "attempt_id": { "$ref": "#/$defs/uuid" },
+            "site_id": { "$ref": "#/$defs/identifier" },
+            "engine": { "enum": ["public_sites", "wordpress"] },
+            "phase": { "enum": ["started", "succeeded", "failed"] },
+            "failure_code": { "type": "string", "pattern": "^[A-Z][A-Z0-9_]*$" }
+          }
+        }
+      ]
+    },
+    "identifier": {
+      "type": "string",
+      "pattern": "^[a-z0-9]+(?:-[a-z0-9]+)*$",
+      "maxLength": 80
+    },
+    "sha": { "type": "string", "pattern": "^[0-9a-f]{40}$" },
+    "uuid": {
+      "type": "string",
+      "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
+    }
+  }
+}
diff --git a/schemas/publication-receipt.schema.json b/schemas/publication-receipt.schema.json
new file mode 100644
index 0000000..e923a7c
--- /dev/null
+++ b/schemas/publication-receipt.schema.json
@@ -0,0 +1,70 @@
+{
+  "$schema": "https://json-schema.org/draft/2020-12/schema",
+  "$id": "https://content-foundry.local/schemas/publication-receipt.schema.json",
+  "title": "Successful idempotent publication receipt",
+  "type": "object",
+  "additionalProperties": false,
+  "required": [
+    "schema_version",
+    "receipt_id",
+    "article_id",
+    "article_path",
+    "source_sha",
+    "approval_event_id",
+    "site_id",
+    "engine",
+    "published_at",
+    "result"
+  ],
+  "properties": {
+    "schema_version": { "const": 1 },
+    "receipt_id": { "$ref": "#/$defs/digest" },
+    "article_id": { "$ref": "#/$defs/identifier" },
+    "article_path": {
+      "type": "string",
+      "pattern": "^content/articles/[a-z0-9]+(?:-[a-z0-9]+)*\\.md$"
+    },
+    "source_sha": { "$ref": "#/$defs/sha" },
+    "approval_event_id": { "$ref": "#/$defs/uuid" },
+    "site_id": { "$ref": "#/$defs/identifier" },
+    "engine": { "enum": ["public_sites", "wordpress"] },
+    "published_at": { "type": "string", "format": "date-time" },
+    "result": {
+      "oneOf": [
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["kind", "target", "build_report_sha256"],
+          "properties": {
+            "kind": { "const": "public_sites" },
+            "target": { "type": "string", "minLength": 1, "maxLength": 100 },
+            "build_report_sha256": { "$ref": "#/$defs/digest" }
+          }
+        },
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["kind", "remote_post_id", "remote_status"],
+          "properties": {
+            "kind": { "const": "wordpress" },
+            "remote_post_id": { "type": "integer", "minimum": 1 },
+            "remote_status": { "const": "draft" }
+          }
+        }
+      ]
+    }
+  },
+  "$defs": {
+    "identifier": {
+      "type": "string",
+      "pattern": "^[a-z0-9]+(?:-[a-z0-9]+)*$",
+      "maxLength": 80
+    },
+    "sha": { "type": "string", "pattern": "^[0-9a-f]{40}$" },
+    "digest": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
+    "uuid": {
+      "type": "string",
+      "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
+    }
+  }
+}
diff --git a/test/schema.test.js b/test/schema.test.js
index d8995bc..805da7e 100644
--- a/test/schema.test.js
+++ b/test/schema.test.js
@@ -18,7 +18,9 @@ async function validators() {
     strict: true,
   });
   return {
+    auditEvent: ajv.compile(await loadJson("schemas/audit-event.schema.json")),
     article: ajv.compile(await loadJson("schemas/article.schema.json")),
+    receipt: ajv.compile(await loadJson("schemas/publication-receipt.schema.json")),
     site: ajv.compile(await loadJson("schemas/site.schema.json")),
   };
 }
@@ -70,3 +72,24 @@ test("a site cannot configure both publication engines", async () => {
 
   assert.equal(site(invalid), false);
 });
+
+test("audit and receipt schemas keep approval and publication distinct", async () => {
+  const { auditEvent, receipt } = await validators();
+  const common = {
+    schema_version: 1,
+    event_id: "00000000-0000-4000-8000-000000000001",
+    article_id: "publisher-loop",
+    article_path: "content/articles/publisher-loop.md",
+    source_sha: "a".repeat(40),
+    occurred_at: "2026-08-29T00:00:00Z",
+  };
+  const approval = {
+    ...common,
+    event_type: "approval",
+    decision: "approved",
+    actor: "Local reviewer",
+  };
+
+  assert.equal(auditEvent(approval), true, JSON.stringify(auditEvent.errors));
+  assert.equal(receipt(approval), false);
+});


## `refactor(security): centralize secret detection`

diff --git a/src/content.js b/src/content.js
index f364143..2fc499c 100644
--- a/src/content.js
+++ b/src/content.js
@@ -6,17 +6,11 @@ import Ajv2020 from "ajv/dist/2020.js";
 import matter from "gray-matter";
 import { parse as parseYaml } from "yaml";
 
+import { containsSensitiveMaterial } from "./sensitive.js";
+
 const moduleDirectory = path.dirname(fileURLToPath(import.meta.url));
 const schemaDirectory = path.resolve(moduleDirectory, "../schemas");
 const identifier = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;
-const sensitiveKey = /(?:password|secret|token|credential|api[_-]?key)/i;
-const sensitiveValuePatterns = [
-  /-----BEGIN [A-Z ]*PRIVATE KEY-----/,
-  /\bAKIA[0-9A-Z]{16}\b/,
-  /\bghp_[A-Za-z0-9]{36,}\b/,
-  /\bgithub_pat_[A-Za-z0-9_]{40,}\b/,
-  /\bsk-[A-Za-z0-9_-]{20,}\b/,
-];
 
 export class ContentValidationError extends Error {
   constructor(code, message) {
@@ -54,17 +48,8 @@ function parseYamlSafely(source, label) {
   }
 }
 
-function containsSensitiveKey(value) {
-  if (Array.isArray(value)) return value.some(containsSensitiveKey);
-  if (!value || typeof value !== "object") return false;
-  return Object.entries(value).some(
-    ([key, child]) =>
-      (sensitiveKey.test(key) && !key.endsWith("_env")) || containsSensitiveKey(child),
-  );
-}
-
 function rejectSensitiveMaterial(raw, parsed, label) {
-  if (containsSensitiveKey(parsed) || sensitiveValuePatterns.some((pattern) => pattern.test(raw))) {
+  if (containsSensitiveMaterial(raw, parsed)) {
     throw new ContentValidationError(
       "SENSITIVE_MATERIAL",
       `Sensitive material is not permitted in ${label}`,
diff --git a/src/sensitive.js b/src/sensitive.js
new file mode 100644
index 0000000..4e6dd52
--- /dev/null
+++ b/src/sensitive.js
@@ -0,0 +1,23 @@
+const sensitiveKey = /(?:password|secret|token|credential|api[_-]?key)/i;
+const sensitiveValuePatterns = [
+  /-----BEGIN [A-Z ]*PRIVATE KEY-----/,
+  /\bAKIA[0-9A-Z]{16}\b/,
+  /\bghp_[A-Za-z0-9]{36,}\b/,
+  /\bgithub_pat_[A-Za-z0-9_]{40,}\b/,
+  /\bsk-[A-Za-z0-9_-]{20,}\b/,
+];
+
+function containsSensitiveKey(value) {
+  if (Array.isArray(value)) return value.some(containsSensitiveKey);
+  if (!value || typeof value !== "object") return false;
+  return Object.entries(value).some(
+    ([key, child]) =>
+      (sensitiveKey.test(key) && !key.endsWith("_env")) || containsSensitiveKey(child),
+  );
+}
+
+export function containsSensitiveMaterial(raw, parsed) {
+  return (
+    containsSensitiveKey(parsed) || sensitiveValuePatterns.some((pattern) => pattern.test(raw))
+  );
+}


## `feat(audit): write append-only events`

diff --git a/src/audit.js b/src/audit.js
new file mode 100644
index 0000000..b1e7d00
--- /dev/null
+++ b/src/audit.js
@@ -0,0 +1,79 @@
+import { mkdir, readFile, writeFile } from "node:fs/promises";
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+
+import Ajv2020 from "ajv/dist/2020.js";
+
+import { containsSensitiveMaterial } from "./sensitive.js";
+
+const moduleDirectory = path.dirname(fileURLToPath(import.meta.url));
+const schemaDirectory = path.resolve(moduleDirectory, "../schemas");
+
+export class AuditIntegrityError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "AuditIntegrityError";
+    this.code = code;
+  }
+}
+
+async function buildValidators() {
+  const isDateTime = (value) =>
+    typeof value === "string" &&
+    /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z$/.test(value) &&
+    !Number.isNaN(Date.parse(value));
+  const ajv = new Ajv2020({
+    allErrors: true,
+    formats: { "date-time": isDateTime },
+    strict: true,
+  });
+  const load = async (name) =>
+    JSON.parse(await readFile(path.join(schemaDirectory, name), "utf8"));
+  return {
+    event: ajv.compile(await load("audit-event.schema.json")),
+    receipt: ajv.compile(await load("publication-receipt.schema.json")),
+  };
+}
+
+const validators = await buildValidators();
+
+function assertRecord(validate, record, kind) {
+  const serialized = `${JSON.stringify(record, null, 2)}\n`;
+  if (containsSensitiveMaterial(serialized, record)) {
+    throw new AuditIntegrityError(
+      "SENSITIVE_AUDIT_MATERIAL",
+      `Sensitive material is not permitted in ${kind}`,
+    );
+  }
+  if (!validate(record)) {
+    const issues = validate.errors
+      .map(({ instancePath, keyword }) => `${instancePath || "/"}:${keyword}`)
+      .join(", ");
+    throw new AuditIntegrityError("INVALID_AUDIT_RECORD", `${kind} is invalid (${issues})`);
+  }
+  return serialized;
+}
+
+export function validateAuditEvent(event) {
+  return assertRecord(validators.event, event, "audit event");
+}
+
+export function validatePublicationReceipt(receipt) {
+  return assertRecord(validators.receipt, receipt, "publication receipt");
+}
+
+export async function writeAuditEvent(root, event) {
+  const serialized = validateAuditEvent(event);
+  const relativePath = `.publisher/events/${event.event_type}s/${event.event_id}.json`;
+  const absolutePath = path.join(root, relativePath);
+  await mkdir(path.dirname(absolutePath), { recursive: true });
+  try {
+    await writeFile(absolutePath, serialized, { encoding: "utf8", flag: "wx", mode: 0o600 });
+  } catch (error) {
+    if (error.code === "EEXIST") {
+      throw new AuditIntegrityError("EVENT_ALREADY_EXISTS", "Audit event identity already exists");
+    }
+    throw error;
+  }
+  return relativePath;
+}
diff --git a/test/audit.test.js b/test/audit.test.js
new file mode 100644
index 0000000..aa4cc61
--- /dev/null
+++ b/test/audit.test.js
@@ -0,0 +1,39 @@
+import assert from "node:assert/strict";
+import { mkdtemp, readFile, rm } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+
+import { validateAuditEvent, writeAuditEvent } from "../src/audit.js";
+
+const event = Object.freeze({
+  schema_version: 1,
+  event_id: "00000000-0000-4000-8000-000000000001",
+  event_type: "approval",
+  article_id: "publisher-loop",
+  article_path: "content/articles/publisher-loop.md",
+  source_sha: "a".repeat(40),
+  occurred_at: "2026-08-29T00:00:00Z",
+  decision: "approved",
+  actor: "Local reviewer",
+});
+
+test("writes a validated append-only audit event", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-audit-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+
+  const relativePath = await writeAuditEvent(root, event);
+  const stored = JSON.parse(await readFile(path.join(root, relativePath), "utf8"));
+  assert.deepEqual(stored, event);
+  await assert.rejects(() => writeAuditEvent(root, event), { code: "EVENT_ALREADY_EXISTS" });
+});
+
+test("rejects invalid or credential-shaped audit content", () => {
+  assert.throws(() => validateAuditEvent({ ...event, source_sha: "short" }), {
+    code: "INVALID_AUDIT_RECORD",
+  });
+  const token = ["gh", "p_", "a".repeat(40)].join("");
+  assert.throws(() => validateAuditEvent({ ...event, actor: token }), {
+    code: "SENSITIVE_AUDIT_MATERIAL",
+  });
+});


## `feat(audit): couple success to receipt identity`

diff --git a/schemas/audit-event.schema.json b/schemas/audit-event.schema.json
index 336fce6..55bddde 100644
--- a/schemas/audit-event.schema.json
+++ b/schemas/audit-event.schema.json
@@ -59,6 +59,16 @@
           "type": "object",
           "additionalProperties": false,
           "required": ["attempt_id", "site_id", "engine", "phase"],
+          "allOf": [
+            {
+              "if": { "properties": { "phase": { "const": "succeeded" } }, "required": ["phase"] },
+              "then": { "properties": { "receipt_id": true }, "required": ["receipt_id"] }
+            },
+            {
+              "if": { "properties": { "phase": { "const": "failed" } }, "required": ["phase"] },
+              "then": { "properties": { "failure_code": true }, "required": ["failure_code"] }
+            }
+          ],
           "properties": {
             "schema_version": true,
             "event_id": true,
@@ -71,7 +81,8 @@
             "site_id": { "$ref": "#/$defs/identifier" },
             "engine": { "enum": ["public_sites", "wordpress"] },
             "phase": { "enum": ["started", "succeeded", "failed"] },
-            "failure_code": { "type": "string", "pattern": "^[A-Z][A-Z0-9_]*$" }
+            "failure_code": { "type": "string", "pattern": "^[A-Z][A-Z0-9_]*$" },
+            "receipt_id": { "$ref": "#/$defs/digest" }
           }
         }
       ]
@@ -82,6 +93,7 @@
       "maxLength": 80
     },
     "sha": { "type": "string", "pattern": "^[0-9a-f]{40}$" },
+    "digest": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
     "uuid": {
       "type": "string",
       "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
diff --git a/test/schema.test.js b/test/schema.test.js
index 469fb87..909ab0e 100644
--- a/test/schema.test.js
+++ b/test/schema.test.js
@@ -93,6 +93,22 @@ test("audit and receipt schemas keep approval and publication distinct", async (
 
   assert.equal(auditEvent(approval), true, JSON.stringify(auditEvent.errors));
   assert.equal(receipt(approval), false);
+
+  const publication = {
+    ...common,
+    event_id: "00000000-0000-4000-8000-000000000003",
+    event_type: "publication",
+    attempt_id: "00000000-0000-4000-8000-000000000004",
+    site_id: "public-demo",
+    engine: "public_sites",
+    phase: "succeeded",
+  };
+  assert.equal(auditEvent(publication), false);
+  assert.equal(
+    auditEvent({ ...publication, receipt_id: "b".repeat(64) }),
+    true,
+    JSON.stringify(auditEvent.errors),
+  );
 });
 
 test("Public Sites build reports pin the frozen renderer", async () => {


## `feat(receipt): store success atomically`

diff --git a/src/audit.js b/src/audit.js
index b1e7d00..47a693a 100644
--- a/src/audit.js
+++ b/src/audit.js
@@ -1,4 +1,4 @@
-import { mkdir, readFile, writeFile } from "node:fs/promises";
+import { mkdir, readFile, rename, rm, writeFile } from "node:fs/promises";
 import path from "node:path";
 import { fileURLToPath } from "node:url";
 
@@ -77,3 +77,87 @@ export async function writeAuditEvent(root, event) {
   }
   return relativePath;
 }
+
+export async function readPublicationSuccess(root, receiptId) {
+  if (!/^[0-9a-f]{64}$/.test(receiptId)) {
+    throw new AuditIntegrityError("INVALID_RECEIPT_ID", "Publication receipt identity is invalid");
+  }
+  const relativePath = `.publisher/publications/${receiptId}`;
+  try {
+    const [receipt, event] = await Promise.all([
+      readFile(path.join(root, relativePath, "receipt.json"), "utf8").then(JSON.parse),
+      readFile(path.join(root, relativePath, "event.json"), "utf8").then(JSON.parse),
+    ]);
+    validatePublicationReceipt(receipt);
+    validateAuditEvent(event);
+    if (
+      receipt.receipt_id !== receiptId ||
+      event.phase !== "succeeded" ||
+      event.receipt_id !== receiptId ||
+      event.source_sha !== receipt.source_sha ||
+      event.engine !== receipt.engine
+    ) {
+      throw new AuditIntegrityError(
+        "PUBLICATION_TRANSACTION_MISMATCH",
+        "Publication event and receipt identities do not match",
+      );
+    }
+    return Object.freeze({ event: Object.freeze(event), receipt: Object.freeze(receipt), relativePath });
+  } catch (error) {
+    if (error.code === "ENOENT") return undefined;
+    if (error instanceof AuditIntegrityError) throw error;
+    throw new AuditIntegrityError(
+      "PUBLICATION_TRANSACTION_INVALID",
+      "Publication transaction could not be read",
+    );
+  }
+}
+
+export async function writePublicationSuccess(root, receipt, event, transactionId) {
+  validatePublicationReceipt(receipt);
+  validateAuditEvent(event);
+  if (!/^[A-Za-z0-9-]{1,80}$/.test(transactionId)) {
+    throw new AuditIntegrityError(
+      "INVALID_TRANSACTION_ID",
+      "Publication transaction identity is invalid",
+    );
+  }
+  if (event.phase !== "succeeded" || event.receipt_id !== receipt.receipt_id) {
+    throw new AuditIntegrityError(
+      "PUBLICATION_TRANSACTION_MISMATCH",
+      "Succeeded event must name its receipt",
+    );
+  }
+  const parent = path.join(root, ".publisher/publications");
+  const destination = path.join(parent, receipt.receipt_id);
+  const temporary = path.join(parent, `.tmp-${transactionId}`);
+  await mkdir(parent, { recursive: true });
+  await mkdir(temporary, { mode: 0o700 });
+  try {
+    await Promise.all([
+      writeFile(path.join(temporary, "receipt.json"), validatePublicationReceipt(receipt), {
+        encoding: "utf8",
+        flag: "wx",
+        mode: 0o600,
+      }),
+      writeFile(path.join(temporary, "event.json"), validateAuditEvent(event), {
+        encoding: "utf8",
+        flag: "wx",
+        mode: 0o600,
+      }),
+    ]);
+    await rename(temporary, destination);
+  } catch (error) {
+    await rm(temporary, { force: true, recursive: true });
+    if (new Set(["EEXIST", "ENOTEMPTY"]).has(error.code)) {
+      const existing = await readPublicationSuccess(root, receipt.receipt_id);
+      if (existing && JSON.stringify(existing.receipt) === JSON.stringify(receipt)) return existing;
+      throw new AuditIntegrityError(
+        "RECEIPT_COLLISION",
+        "Publication receipt identity already has different content",
+      );
+    }
+    throw error;
+  }
+  return readPublicationSuccess(root, receipt.receipt_id);
+}
diff --git a/test/audit.test.js b/test/audit.test.js
index aa4cc61..38f4e7b 100644
--- a/test/audit.test.js
+++ b/test/audit.test.js
@@ -4,7 +4,12 @@ import os from "node:os";
 import path from "node:path";
 import test from "node:test";
 
-import { validateAuditEvent, writeAuditEvent } from "../src/audit.js";
+import {
+  readPublicationSuccess,
+  validateAuditEvent,
+  writeAuditEvent,
+  writePublicationSuccess,
+} from "../src/audit.js";
 
 const event = Object.freeze({
   schema_version: 1,
@@ -37,3 +42,45 @@ test("rejects invalid or credential-shaped audit content", () => {
     code: "SENSITIVE_AUDIT_MATERIAL",
   });
 });
+
+test("atomically stores a succeeded event with its receipt", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-audit-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const receiptId = "b".repeat(64);
+  const receipt = {
+    schema_version: 1,
+    receipt_id: receiptId,
+    article_id: "publisher-loop",
+    article_path: "content/articles/publisher-loop.md",
+    source_sha: "a".repeat(40),
+    approval_event_id: event.event_id,
+    site_id: "public-demo",
+    engine: "public_sites",
+    published_at: "2026-08-29T00:01:00Z",
+    result: {
+      kind: "public_sites",
+      target: "site-a",
+      build_report_sha256: "c".repeat(64),
+    },
+  };
+  const succeeded = {
+    schema_version: 1,
+    event_id: "00000000-0000-4000-8000-000000000005",
+    event_type: "publication",
+    article_id: event.article_id,
+    article_path: event.article_path,
+    source_sha: event.source_sha,
+    occurred_at: "2026-08-29T00:01:00Z",
+    attempt_id: "00000000-0000-4000-8000-000000000006",
+    site_id: "public-demo",
+    engine: "public_sites",
+    phase: "succeeded",
+    receipt_id: receiptId,
+  };
+
+  const stored = await writePublicationSuccess(root, receipt, succeeded, "first");
+  assert.deepEqual(stored.receipt, receipt);
+  assert.deepEqual((await readPublicationSuccess(root, receiptId)).event, succeeded);
+  const retried = await writePublicationSuccess(root, receipt, succeeded, "retry");
+  assert.deepEqual(retried.receipt, receipt);
+});


