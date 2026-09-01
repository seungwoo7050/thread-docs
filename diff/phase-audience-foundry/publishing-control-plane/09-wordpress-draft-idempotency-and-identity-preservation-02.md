## `feat(publication): orchestrate idempotent receipts`

diff --git a/.gitignore b/.gitignore
index 0c8e725..c996f79 100644
--- a/.gitignore
+++ b/.gitignore
@@ -6,3 +6,5 @@ node_modules/
 dist/
 .decap/
 .wp-env/
+.publisher/locks/
+.publisher/work/
diff --git a/src/publication-receipts.js b/src/publication-receipts.js
new file mode 100644
index 0000000..9e7bbce
--- /dev/null
+++ b/src/publication-receipts.js
@@ -0,0 +1,67 @@
+import { createHash } from "node:crypto";
+import { readdir } from "node:fs/promises";
+import path from "node:path";
+
+import { readPublicationSuccess } from "./audit.js";
+
+export class PublicationReceiptError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicationReceiptError";
+    this.code = code;
+  }
+}
+
+export function publicationId({ articleId, siteId, engine, sourceSha }) {
+  return createHash("sha256")
+    .update(["publisher-v1", articleId, siteId, engine, sourceSha].join("\0"))
+    .digest("hex");
+}
+
+export async function priorWordPressRemoteId(root, currentReceiptId, articleId, siteId) {
+  const directory = path.join(root, ".publisher/publications");
+  let names;
+  try {
+    names = await readdir(directory);
+  } catch (error) {
+    if (error.code === "ENOENT") return undefined;
+    throw error;
+  }
+  const receipts = [];
+  for (const name of names.filter((value) => /^[0-9a-f]{64}$/.test(value))) {
+    if (name === currentReceiptId) continue;
+    const transaction = await readPublicationSuccess(root, name);
+    const receipt = transaction?.receipt;
+    if (
+      receipt?.article_id === articleId &&
+      receipt.site_id === siteId &&
+      receipt.engine === "wordpress" &&
+      receipt.result.kind === "wordpress"
+    ) {
+      receipts.push(receipt);
+    }
+  }
+  receipts.sort((left, right) => left.published_at.localeCompare(right.published_at));
+  return receipts.at(-1)?.result.remote_post_id;
+}
+
+export function resultForReceipt(engine, result) {
+  if (engine === "public_sites" && result?.kind === "public_sites") {
+    return {
+      kind: "public_sites",
+      target: result.target,
+      build_report_sha256: result.build_report_sha256,
+    };
+  }
+  if (engine === "wordpress" && result?.kind === "wordpress") {
+    return {
+      kind: "wordpress",
+      remote_post_id: result.remote_post_id,
+      remote_status: result.remote_status,
+    };
+  }
+  throw new PublicationReceiptError(
+    "ADAPTER_RESULT_MISMATCH",
+    "Adapter result does not match site engine",
+  );
+}
diff --git a/src/publication.js b/src/publication.js
new file mode 100644
index 0000000..83e559c
--- /dev/null
+++ b/src/publication.js
@@ -0,0 +1,168 @@
+import { randomUUID } from "node:crypto";
+import { mkdir, rm } from "node:fs/promises";
+import path from "node:path";
+
+import { requireCommittedApproval } from "./approval-gate.js";
+import {
+  readPublicationSuccess,
+  writeAuditEvent,
+  writePublicationSuccess,
+} from "./audit.js";
+import {
+  priorWordPressRemoteId,
+  publicationId,
+  resultForReceipt,
+} from "./publication-receipts.js";
+
+export { publicationId } from "./publication-receipts.js";
+
+export class PublicationError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicationError";
+    this.code = code;
+  }
+}
+
+function assertReceiptIdentity(receipt, expected) {
+  if (
+    receipt.receipt_id !== expected.receiptId ||
+    receipt.article_id !== expected.articleId ||
+    receipt.article_path !== expected.articlePath ||
+    receipt.source_sha !== expected.sourceSha ||
+    receipt.site_id !== expected.siteId ||
+    receipt.engine !== expected.engine
+  ) {
+    throw new PublicationError(
+      "RECEIPT_IDENTITY_MISMATCH",
+      "Existing publication receipt does not match requested input",
+    );
+  }
+}
+
+function failureCode(error) {
+  return /^[A-Z][A-Z0-9_]*$/.test(error?.code || "") ? error.code : "PUBLICATION_FAILED";
+}
+
+export async function publishApprovedArticle({
+  root,
+  articlePath,
+  sourceSha,
+  adapters,
+  adapterContext = {},
+  now = () => new Date(),
+  uuid = randomUUID,
+}) {
+  const gated = await requireCommittedApproval({ root, articlePath, sourceSha });
+  const engine = gated.site.engine;
+  const adapter = adapters[engine];
+  if (typeof adapter !== "function") {
+    throw new PublicationError("ADAPTER_UNAVAILABLE", "Configured publication adapter is unavailable");
+  }
+  const receiptId = publicationId({
+    articleId: gated.article.frontMatter.id,
+    siteId: gated.site.id,
+    engine,
+    sourceSha,
+  });
+  const expected = {
+    receiptId,
+    articleId: gated.article.frontMatter.id,
+    articlePath,
+    sourceSha,
+    siteId: gated.site.id,
+    engine,
+  };
+  const existing = await readPublicationSuccess(root, receiptId);
+  if (existing) {
+    assertReceiptIdentity(existing.receipt, expected);
+    return Object.freeze({ idempotent: true, ...existing });
+  }
+
+  const lockParent = path.join(root, ".publisher/locks");
+  const lockPath = path.join(lockParent, receiptId);
+  await mkdir(lockParent, { recursive: true });
+  try {
+    await mkdir(lockPath);
+  } catch (error) {
+    if (error.code === "EEXIST") {
+      throw new PublicationError("PUBLICATION_IN_PROGRESS", "This publication is already in progress");
+    }
+    throw error;
+  }
+  try {
+    const raced = await readPublicationSuccess(root, receiptId);
+    if (raced) {
+      assertReceiptIdentity(raced.receipt, expected);
+      return Object.freeze({ idempotent: true, ...raced });
+    }
+    const attemptId = uuid();
+    const common = {
+      schema_version: 1,
+      event_type: "publication",
+      article_id: expected.articleId,
+      article_path: articlePath,
+      source_sha: sourceSha,
+      attempt_id: attemptId,
+      site_id: expected.siteId,
+      engine,
+    };
+    await writeAuditEvent(root, {
+      ...common,
+      event_id: uuid(),
+      occurred_at: now().toISOString(),
+      phase: "started",
+    });
+    try {
+      const priorRemoteId =
+        engine === "wordpress"
+          ? await priorWordPressRemoteId(root, receiptId, expected.articleId, expected.siteId)
+          : undefined;
+      const rawResult = await adapter({
+        ...adapterContext,
+        publisherRoot: root,
+        articlePath,
+        sourceSha,
+        idempotencyKey: receiptId,
+        priorRemoteId,
+      });
+      const publishedAt = now().toISOString();
+      const receipt = {
+        schema_version: 1,
+        receipt_id: receiptId,
+        article_id: expected.articleId,
+        article_path: articlePath,
+        source_sha: sourceSha,
+        approval_event_id: gated.approval.event_id,
+        site_id: expected.siteId,
+        engine,
+        published_at: publishedAt,
+        result: resultForReceipt(engine, rawResult),
+      };
+      const transaction = await writePublicationSuccess(
+        root,
+        receipt,
+        {
+          ...common,
+          event_id: uuid(),
+          occurred_at: publishedAt,
+          phase: "succeeded",
+          receipt_id: receiptId,
+        },
+        uuid(),
+      );
+      return Object.freeze({ idempotent: false, ...transaction });
+    } catch (error) {
+      await writeAuditEvent(root, {
+        ...common,
+        event_id: uuid(),
+        occurred_at: now().toISOString(),
+        phase: "failed",
+        failure_code: failureCode(error),
+      });
+      throw error;
+    }
+  } finally {
+    await rm(lockPath, { force: true, recursive: true });
+  }
+}
diff --git a/test/publication.test.js b/test/publication.test.js
new file mode 100644
index 0000000..5941789
--- /dev/null
+++ b/test/publication.test.js
@@ -0,0 +1,148 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { copyFile, mkdir, mkdtemp, readdir, rm } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import { writeAuditEvent } from "../src/audit.js";
+import { publicationId, publishApprovedArticle } from "../src/publication.js";
+
+const execFileAsync = promisify(execFile);
+const articlePath = "content/articles/wordpress-loop.md";
+
+async function commit(root, message) {
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
+      message,
+    ],
+    { cwd: root },
+  );
+}
+
+async function publicationRepository(context, { approved = true } = {}) {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-orchestration-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  await mkdir(path.join(root, "content/articles"), { recursive: true });
+  await mkdir(path.join(root, "sites"));
+  await copyFile(articlePath, path.join(root, articlePath));
+  await copyFile("sites/wordpress-local.yml", path.join(root, "sites/wordpress-local.yml"));
+  await commit(root, "source");
+  const { stdout } = await execFileAsync("git", ["rev-parse", "HEAD"], { cwd: root });
+  const sourceSha = stdout.trim();
+  if (approved) {
+    await writeAuditEvent(root, {
+      schema_version: 1,
+      event_id: "00000000-0000-4000-8000-000000000020",
+      event_type: "approval",
+      article_id: "wordpress-loop",
+      article_path: articlePath,
+      source_sha: sourceSha,
+      occurred_at: "2026-08-29T01:00:00Z",
+      decision: "approved",
+      actor: "Local reviewer",
+    });
+    await commit(root, "approve");
+  }
+  return { root, sourceSha };
+}
+
+function deterministicRuntime() {
+  let sequence = 30;
+  return {
+    now: () => new Date(`2026-08-29T01:00:${String(sequence++).padStart(2, "0")}Z`),
+    uuid: () => `00000000-0000-4000-8000-${String(sequence++).padStart(12, "0")}`,
+  };
+}
+
+test("approval is mandatory before any adapter call", async (context) => {
+  const { root, sourceSha } = await publicationRepository(context, { approved: false });
+  let called = false;
+
+  await assert.rejects(
+    () =>
+      publishApprovedArticle({
+        root,
+        articlePath,
+        sourceSha,
+        adapters: { wordpress: async () => (called = true) },
+      }),
+    { code: "APPROVAL_REQUIRED" },
+  );
+  assert.equal(called, false);
+});
+
+test("successful retry returns one receipt without a second adapter call", async (context) => {
+  const { root, sourceSha } = await publicationRepository(context);
+  let calls = 0;
+  const input = {
+    root,
+    articlePath,
+    sourceSha,
+    adapters: {
+      wordpress: async () => {
+        calls += 1;
+        return { kind: "wordpress", remote_post_id: 77, remote_status: "draft" };
+      },
+    },
+    ...deterministicRuntime(),
+  };
+
+  const first = await publishApprovedArticle(input);
+  const retried = await publishApprovedArticle(input);
+  const expectedId = publicationId({
+    articleId: "wordpress-loop",
+    siteId: "wordpress-local",
+    engine: "wordpress",
+    sourceSha,
+  });
+
+  assert.equal(first.idempotent, false);
+  assert.equal(retried.idempotent, true);
+  assert.equal(first.receipt.receipt_id, expectedId);
+  assert.equal(retried.receipt.result.remote_post_id, 77);
+  assert.equal(retried.receipt.result.remote_status, "draft");
+  assert.equal(calls, 1);
+  assert.deepEqual(
+    (await readdir(path.join(root, ".publisher/publications", expectedId))).sort(),
+    ["event.json", "receipt.json"],
+  );
+});
+
+test("failed publication records failure and remains retryable", async (context) => {
+  const { root, sourceSha } = await publicationRepository(context);
+  let calls = 0;
+  const runtime = deterministicRuntime();
+  const input = {
+    root,
+    articlePath,
+    sourceSha,
+    adapters: {
+      wordpress: async () => {
+        calls += 1;
+        if (calls === 1) throw Object.assign(new Error("private detail"), { code: "TEST_FAILURE" });
+        return { kind: "wordpress", remote_post_id: 88, remote_status: "draft" };
+      },
+    },
+    ...runtime,
+  };
+
+  await assert.rejects(() => publishApprovedArticle(input), { code: "TEST_FAILURE" });
+  const retried = await publishApprovedArticle(input);
+  const eventFiles = await readdir(path.join(root, ".publisher/events/publications"));
+
+  assert.equal(retried.receipt.result.remote_post_id, 88);
+  assert.equal(calls, 2);
+  assert.equal(eventFiles.length, 3);
+});
