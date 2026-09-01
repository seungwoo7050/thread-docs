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


## `feat(publication): resolve adapters from runtime`

diff --git a/src/runtime-adapters.js b/src/runtime-adapters.js
new file mode 100644
index 0000000..377b6c9
--- /dev/null
+++ b/src/runtime-adapters.js
@@ -0,0 +1,68 @@
+import { mkdir } from "node:fs/promises";
+import path from "node:path";
+
+import { publishToPublicSites } from "./adapters/public-sites.js";
+import { publishToWordPress } from "./adapters/wordpress.js";
+import { validateArticleSource, validateSiteSource } from "./content.js";
+import { readFileAtCommit } from "./git.js";
+import {
+  createWordPressRestTransport,
+  resolveWordPressRuntime,
+} from "./wordpress-rest-transport.js";
+import { createWpEnvCliTransport } from "./wp-env-cli-transport.js";
+
+export class RuntimeAdapterError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "RuntimeAdapterError";
+    this.code = code;
+  }
+}
+
+async function wordpressSiteAtSource(root, sourceSha, articlePath) {
+  const article = validateArticleSource(
+    await readFileAtCommit(root, sourceSha, articlePath),
+    articlePath,
+  );
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  return validateSiteSource(await readFileAtCommit(root, sourceSha, sitePath), sitePath);
+}
+
+export function configuredAdapters(root, environment = process.env) {
+  return Object.freeze({
+    public_sites: async (context) => {
+      const repository = environment.PUBLISHER_PUBLIC_SITES_REPO;
+      if (!repository) {
+        throw new RuntimeAdapterError(
+          "PUBLIC_SITES_REPOSITORY_REQUIRED",
+          "PUBLISHER_PUBLIC_SITES_REPO is required",
+        );
+      }
+      const workRoot = path.join(root, ".publisher/work/public-sites");
+      await mkdir(workRoot, { recursive: true });
+      return publishToPublicSites({
+        ...context,
+        publicSitesRepository: path.resolve(repository),
+        workRoot,
+      });
+    },
+    wordpress: async (context) => {
+      const mode = environment.PUBLISHER_WP_TRANSPORT || "wp-env";
+      let transport;
+      if (mode === "wp-env") {
+        transport = createWpEnvCliTransport({ publisherRoot: root });
+      } else if (mode === "rest") {
+        const site = await wordpressSiteAtSource(root, context.sourceSha, context.articlePath);
+        transport = createWordPressRestTransport({
+          ...resolveWordPressRuntime(site, environment),
+        });
+      } else {
+        throw new RuntimeAdapterError(
+          "WORDPRESS_TRANSPORT_INVALID",
+          "PUBLISHER_WP_TRANSPORT must be wp-env or rest",
+        );
+      }
+      return publishToWordPress({ ...context, transport });
+    },
+  });
+}
diff --git a/test/runtime-adapters.test.js b/test/runtime-adapters.test.js
new file mode 100644
index 0000000..e74b533
--- /dev/null
+++ b/test/runtime-adapters.test.js
@@ -0,0 +1,27 @@
+import assert from "node:assert/strict";
+import { mkdtemp, rm } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+
+import { configuredAdapters } from "../src/runtime-adapters.js";
+
+test("runtime adapter requires the frozen Public Sites checkout path", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-runtime-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const adapters = configuredAdapters(root, {});
+
+  await assert.rejects(() => adapters.public_sites({}), {
+    code: "PUBLIC_SITES_REPOSITORY_REQUIRED",
+  });
+});
+
+test("runtime adapter rejects an invented WordPress transport", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-runtime-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const adapters = configuredAdapters(root, { PUBLISHER_WP_TRANSPORT: "hosted-magic" });
+
+  await assert.rejects(() => adapters.wordpress({}), {
+    code: "WORDPRESS_TRANSPORT_INVALID",
+  });
+});


## `feat(cli): expose gated publication`

diff --git a/package.json b/package.json
index e7efe19..36fb6c1 100644
--- a/package.json
+++ b/package.json
@@ -14,6 +14,7 @@
     "approve": "node src/cli.js approve",
     "cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
     "cms:web": "node src/admin-server.js",
+    "publish": "node src/cli.js publish",
     "test": "node --test",
     "wp:start": "wp-env start",
     "wp:stop": "wp-env stop"
diff --git a/src/cli.js b/src/cli.js
index 5537961..40cb06e 100644
--- a/src/cli.js
+++ b/src/cli.js
@@ -4,6 +4,11 @@ import { fileURLToPath } from "node:url";
 import { createInterface } from "node:readline/promises";
 
 import { recordApprovalDecision } from "./approval.js";
+import { publishApprovedArticle } from "./publication.js";
+import { configuredAdapters } from "./runtime-adapters.js";
+
+const approvalOptions = ["actor", "article", "decision", "source-sha"];
+const publicationOptions = ["article", "source-sha"];
 
 export class CliUsageError extends Error {
   constructor(code, message) {
@@ -13,7 +18,11 @@ export class CliUsageError extends Error {
   }
 }
 
-export function parseOptions(argumentsList) {
+export function parseOptions(
+  argumentsList,
+  allowedOptions = approvalOptions,
+  requiredOptions = approvalOptions,
+) {
   const options = {};
   for (let index = 0; index < argumentsList.length; index += 2) {
     const flag = argumentsList[index];
@@ -22,7 +31,7 @@ export function parseOptions(argumentsList) {
       throw new CliUsageError("INVALID_ARGUMENTS", "Options must be --name value pairs");
     }
     const name = flag.slice(2);
-    if (!new Set(["actor", "article", "decision", "source-sha"]).has(name)) {
+    if (!new Set(allowedOptions).has(name)) {
       throw new CliUsageError("UNKNOWN_OPTION", "Unknown command option");
     }
     if (options[name] !== undefined) {
@@ -30,7 +39,7 @@ export function parseOptions(argumentsList) {
     }
     options[name] = value;
   }
-  for (const required of ["actor", "article", "decision", "source-sha"]) {
+  for (const required of requiredOptions) {
     if (!options[required]) {
       throw new CliUsageError("MISSING_OPTION", `Missing required option: --${required}`);
     }
@@ -38,6 +47,9 @@ export function parseOptions(argumentsList) {
   return options;
 }
 
+export const parsePublicationOptions = (argumentsList) =>
+  parseOptions(argumentsList, publicationOptions, publicationOptions);
+
 export function confirmationPhrase(decision, sourceSha) {
   const verb = decision === "approved" ? "APPROVE" : "REJECT";
   return `${verb} ${sourceSha}`;
@@ -78,20 +90,34 @@ export async function main(
 ) {
   try {
     const [command, ...argumentsList] = argv;
-    if (command !== "approve") {
-      throw new CliUsageError("UNKNOWN_COMMAND", "Expected command: approve");
+    if (command === "approve") {
+      const options = parseOptions(argumentsList);
+      const result = await recordApprovalDecision({
+        root: process.cwd(),
+        articlePath: options.article,
+        sourceSha: options["source-sha"],
+        actor: options.actor,
+        decision: options.decision,
+        confirm: (request) => promptForDecision(request, input, output),
+      });
+      output.write(`Recorded ${result.event.decision}: ${result.eventPath}\n`);
+      return 0;
+    }
+    if (command === "publish") {
+      const options = parsePublicationOptions(argumentsList);
+      const root = process.cwd();
+      const result = await publishApprovedArticle({
+        root,
+        articlePath: options.article,
+        sourceSha: options["source-sha"],
+        adapters: configuredAdapters(root),
+      });
+      output.write(
+        `${result.idempotent ? "Reused" : "Created"} publication receipt: ${result.receipt.receipt_id}\n`,
+      );
+      return 0;
     }
-    const options = parseOptions(argumentsList);
-    const result = await recordApprovalDecision({
-      root: process.cwd(),
-      articlePath: options.article,
-      sourceSha: options["source-sha"],
-      actor: options.actor,
-      decision: options.decision,
-      confirm: (request) => promptForDecision(request, input, output),
-    });
-    output.write(`Recorded ${result.event.decision}: ${result.eventPath}\n`);
-    return 0;
+    throw new CliUsageError("UNKNOWN_COMMAND", "Expected command: approve or publish");
   } catch (error) {
     errorOutput.write(`ERROR ${error.code || "UNEXPECTED"}: ${error.message}\n`);
     return 1;
diff --git a/test/cli.test.js b/test/cli.test.js
index f4ceb9b..dda503f 100644
--- a/test/cli.test.js
+++ b/test/cli.test.js
@@ -4,6 +4,7 @@ import test from "node:test";
 import {
   confirmationPhrase,
   parseOptions,
+  parsePublicationOptions,
   requireInteractiveTerminal,
 } from "../src/cli.js";
 
@@ -47,3 +48,31 @@ test("approval CLI rejects non-interactive confirmation", () => {
   );
   assert.doesNotThrow(() => requireInteractiveTerminal({ isTTY: true }, { isTTY: true }));
 });
+
+test("publication CLI accepts only article and immutable SHA", () => {
+  const sha = "c".repeat(40);
+  assert.deepEqual(
+    parsePublicationOptions([
+      "--article",
+      "content/articles/wordpress-loop.md",
+      "--source-sha",
+      sha,
+    ]),
+    {
+      article: "content/articles/wordpress-loop.md",
+      "source-sha": sha,
+    },
+  );
+  assert.throws(
+    () =>
+      parsePublicationOptions([
+        "--article",
+        "content/articles/wordpress-loop.md",
+        "--source-sha",
+        sha,
+        "--engine",
+        "wordpress",
+      ]),
+    { code: "UNKNOWN_OPTION" },
+  );
+});
