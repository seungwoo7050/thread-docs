# 불변 소스 승인 수명 주기

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


## `feat(approval): verify immutable Git input`

diff --git a/src/git.js b/src/git.js
new file mode 100644
index 0000000..6be2068
--- /dev/null
+++ b/src/git.js
@@ -0,0 +1,54 @@
+import { execFile } from "node:child_process";
+import { promisify } from "node:util";
+
+const execFileAsync = promisify(execFile);
+const fullSha = /^[0-9a-f]{40}$/;
+const repositoryPath = /^[A-Za-z0-9._/-]+$/;
+
+export class GitStateError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "GitStateError";
+    this.code = code;
+  }
+}
+
+export async function runGit(root, args) {
+  try {
+    const { stdout } = await execFileAsync("git", args, {
+      cwd: root,
+      encoding: "utf8",
+      maxBuffer: 4 * 1024 * 1024,
+    });
+    return stdout;
+  } catch {
+    throw new GitStateError("GIT_COMMAND_FAILED", "Git state could not be verified");
+  }
+}
+
+export async function assertCleanHead(root, requestedSha) {
+  if (!fullSha.test(requestedSha)) {
+    throw new GitStateError("INVALID_SOURCE_SHA", "Source SHA must be a full commit SHA");
+  }
+  const resolved = (await runGit(root, ["rev-parse", "--verify", `${requestedSha}^{commit}`])).trim();
+  const head = (await runGit(root, ["rev-parse", "HEAD"])).trim();
+  if (resolved !== requestedSha || head !== requestedSha) {
+    throw new GitStateError("SOURCE_NOT_HEAD", "Approval source must be the current HEAD commit");
+  }
+  if ((await runGit(root, ["status", "--porcelain=v1", "--untracked-files=all"])).trim()) {
+    throw new GitStateError("DIRTY_WORKTREE", "Approval requires a clean worktree");
+  }
+  return head;
+}
+
+export async function readFileAtCommit(root, sourceSha, filePath) {
+  if (
+    !fullSha.test(sourceSha) ||
+    !repositoryPath.test(filePath) ||
+    filePath.startsWith("/") ||
+    filePath.split("/").includes("..")
+  ) {
+    throw new GitStateError("INVALID_GIT_PATH", "Git object path is invalid");
+  }
+  return runGit(root, ["show", `${sourceSha}:${filePath}`]);
+}
diff --git a/test/git.test.js b/test/git.test.js
new file mode 100644
index 0000000..1e09150
--- /dev/null
+++ b/test/git.test.js
@@ -0,0 +1,64 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { mkdir, mkdtemp, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import { assertCleanHead, readFileAtCommit } from "../src/git.js";
+
+const execFileAsync = promisify(execFile);
+
+async function fixtureRepository(context) {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-git-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  await mkdir(path.join(root, "content/articles"), { recursive: true });
+  await writeFile(path.join(root, "content/articles/example.md"), "committed\n");
+  await execFileAsync("git", ["add", "."], { cwd: root });
+  await execFileAsync(
+    "git",
+    [
+      "-c",
+      "user.name=Publisher Test",
+      "-c",
+      "user.email=publisher@example.invalid",
+      "commit",
+      "-q",
+      "-m",
+      "fixture",
+    ],
+    { cwd: root },
+  );
+  const { stdout } = await execFileAsync("git", ["rev-parse", "HEAD"], { cwd: root });
+  return { root, sha: stdout.trim() };
+}
+
+test("clean HEAD supplies immutable file content", async (context) => {
+  const { root, sha } = await fixtureRepository(context);
+
+  assert.equal(await assertCleanHead(root, sha), sha);
+  assert.equal(await readFileAtCommit(root, sha, "content/articles/example.md"), "committed\n");
+});
+
+test("approval precondition rejects dirty and untracked changes", async (context) => {
+  const { root, sha } = await fixtureRepository(context);
+  await writeFile(path.join(root, "content/articles/example.md"), "changed\n");
+  await assert.rejects(() => assertCleanHead(root, sha), { code: "DIRTY_WORKTREE" });
+
+  await execFileAsync("git", ["restore", "."], { cwd: root });
+  await writeFile(path.join(root, "untracked.txt"), "untracked\n");
+  await assert.rejects(() => assertCleanHead(root, sha), { code: "DIRTY_WORKTREE" });
+});
+
+test("approval precondition requires a full SHA at current HEAD", async (context) => {
+  const { root, sha } = await fixtureRepository(context);
+
+  await assert.rejects(() => assertCleanHead(root, sha.slice(0, 12)), {
+    code: "INVALID_SOURCE_SHA",
+  });
+  await assert.rejects(() => readFileAtCommit(root, sha, "../outside"), {
+    code: "INVALID_GIT_PATH",
+  });
+});


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


## `feat(approval): record explicit decisions`

diff --git a/src/approval.js b/src/approval.js
new file mode 100644
index 0000000..a8adb32
--- /dev/null
+++ b/src/approval.js
@@ -0,0 +1,83 @@
+import { randomUUID } from "node:crypto";
+
+import { writeAuditEvent } from "./audit.js";
+import { validateArticleSource, validateSiteSource } from "./content.js";
+import { assertCleanHead, readFileAtCommit } from "./git.js";
+
+const articlePathPattern = /^content\/articles\/[a-z0-9]+(?:-[a-z0-9]+)*\.md$/;
+const actorPattern = /^[^\p{Cc}]{1,100}$/u;
+
+export class ApprovalError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "ApprovalError";
+    this.code = code;
+  }
+}
+
+export async function recordApprovalDecision({
+  root,
+  articlePath,
+  sourceSha,
+  actor,
+  decision,
+  confirm,
+  now = () => new Date(),
+  uuid = randomUUID,
+}) {
+  if (!articlePathPattern.test(articlePath)) {
+    throw new ApprovalError("INVALID_ARTICLE_PATH", "Approval article path is invalid");
+  }
+  if (!actorPattern.test(actor) || actor.trim() !== actor) {
+    throw new ApprovalError("INVALID_ACTOR", "Approval actor is invalid");
+  }
+  if (!new Set(["approved", "rejected"]).has(decision)) {
+    throw new ApprovalError("INVALID_DECISION", "Approval decision is invalid");
+  }
+  if (typeof confirm !== "function") {
+    throw new ApprovalError(
+      "CONFIRMATION_REQUIRED",
+      "A human confirmation callback is required for an approval decision",
+    );
+  }
+
+  await assertCleanHead(root, sourceSha);
+  const article = validateArticleSource(
+    await readFileAtCommit(root, sourceSha, articlePath),
+    articlePath,
+  );
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  const site = validateSiteSource(
+    await readFileAtCommit(root, sourceSha, sitePath),
+    sitePath,
+  );
+  if (site.id !== article.frontMatter.site) {
+    throw new ApprovalError("SITE_MISMATCH", "Approved article site identity does not match");
+  }
+
+  const accepted = await confirm({
+    articleId: article.frontMatter.id,
+    articleTitle: article.frontMatter.title,
+    decision,
+    siteId: site.id,
+    sourceSha,
+  });
+  if (accepted !== true) {
+    throw new ApprovalError("DECISION_NOT_CONFIRMED", "Approval decision was not confirmed");
+  }
+
+  await assertCleanHead(root, sourceSha);
+  const event = {
+    schema_version: 1,
+    event_id: uuid(),
+    event_type: "approval",
+    article_id: article.frontMatter.id,
+    article_path: articlePath,
+    source_sha: sourceSha,
+    occurred_at: now().toISOString(),
+    decision,
+    actor,
+  };
+  const eventPath = await writeAuditEvent(root, event);
+  return Object.freeze({ event: Object.freeze(event), eventPath });
+}
diff --git a/test/approval.test.js b/test/approval.test.js
new file mode 100644
index 0000000..1473b65
--- /dev/null
+++ b/test/approval.test.js
@@ -0,0 +1,110 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { copyFile, mkdir, mkdtemp, readFile, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import { recordApprovalDecision } from "../src/approval.js";
+
+const execFileAsync = promisify(execFile);
+const articlePath = "content/articles/publisher-loop.md";
+
+async function approvalRepository(context) {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-approval-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  await mkdir(path.join(root, "content/articles"), { recursive: true });
+  await mkdir(path.join(root, "sites"));
+  await copyFile(articlePath, path.join(root, articlePath));
+  await copyFile("sites/public-demo.yml", path.join(root, "sites/public-demo.yml"));
+  await execFileAsync("git", ["add", "."], { cwd: root });
+  await execFileAsync(
+    "git",
+    [
+      "-c",
+      "user.name=Publisher Test",
+      "-c",
+      "user.email=publisher@example.invalid",
+      "commit",
+      "-q",
+      "-m",
+      "fixture",
+    ],
+    { cwd: root },
+  );
+  const { stdout } = await execFileAsync("git", ["rev-parse", "HEAD"], { cwd: root });
+  return { root, sha: stdout.trim() };
+}
+
+test("records explicit approval against the immutable clean HEAD", async (context) => {
+  const { root, sha } = await approvalRepository(context);
+  let prompt;
+  const result = await recordApprovalDecision({
+    root,
+    articlePath,
+    sourceSha: sha,
+    actor: "Local reviewer",
+    decision: "approved",
+    confirm: async (request) => {
+      prompt = request;
+      return true;
+    },
+    now: () => new Date("2026-08-29T01:02:03Z"),
+    uuid: () => "00000000-0000-4000-8000-000000000002",
+  });
+
+  assert.deepEqual(prompt, {
+    articleId: "publisher-loop",
+    articleTitle: "A Verifiable Publishing Loop",
+    decision: "approved",
+    siteId: "public-demo",
+    sourceSha: sha,
+  });
+  assert.equal(result.event.source_sha, sha);
+  assert.equal(result.event.decision, "approved");
+  assert.deepEqual(
+    JSON.parse(await readFile(path.join(root, result.eventPath), "utf8")),
+    result.event,
+  );
+});
+
+test("cannot create an approval without affirmative confirmation", async (context) => {
+  const { root, sha } = await approvalRepository(context);
+  await assert.rejects(
+    () =>
+      recordApprovalDecision({
+        root,
+        articlePath,
+        sourceSha: sha,
+        actor: "Local reviewer",
+        decision: "approved",
+        confirm: async () => false,
+      }),
+    { code: "DECISION_NOT_CONFIRMED" },
+  );
+});
+
+test("dirty content blocks approval before confirmation", async (context) => {
+  const { root, sha } = await approvalRepository(context);
+  await writeFile(path.join(root, articlePath), "changed after review\n");
+  let prompted = false;
+
+  await assert.rejects(
+    () =>
+      recordApprovalDecision({
+        root,
+        articlePath,
+        sourceSha: sha,
+        actor: "Local reviewer",
+        decision: "rejected",
+        confirm: async () => {
+          prompted = true;
+          return true;
+        },
+      }),
+    { code: "DIRTY_WORKTREE" },
+  );
+  assert.equal(prompted, false);
+});


