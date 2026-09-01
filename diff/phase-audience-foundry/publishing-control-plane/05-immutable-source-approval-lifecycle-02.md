## `feat(approval): require interactive SHA confirmation`

diff --git a/package.json b/package.json
index 885eb7d..6fca5c8 100644
--- a/package.json
+++ b/package.json
@@ -7,7 +7,11 @@
     "node": ">=22 <25"
   },
   "packageManager": "npm@11.4.2",
+  "bin": {
+    "publisher": "src/cli.js"
+  },
   "scripts": {
+    "approve": "node src/cli.js approve",
     "cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
     "cms:web": "node src/admin-server.js",
     "test": "node --test"
diff --git a/src/cli.js b/src/cli.js
new file mode 100644
index 0000000..5537961
--- /dev/null
+++ b/src/cli.js
@@ -0,0 +1,103 @@
+#!/usr/bin/env node
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+import { createInterface } from "node:readline/promises";
+
+import { recordApprovalDecision } from "./approval.js";
+
+export class CliUsageError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "CliUsageError";
+    this.code = code;
+  }
+}
+
+export function parseOptions(argumentsList) {
+  const options = {};
+  for (let index = 0; index < argumentsList.length; index += 2) {
+    const flag = argumentsList[index];
+    const value = argumentsList[index + 1];
+    if (!flag?.startsWith("--") || value === undefined || value.startsWith("--")) {
+      throw new CliUsageError("INVALID_ARGUMENTS", "Options must be --name value pairs");
+    }
+    const name = flag.slice(2);
+    if (!new Set(["actor", "article", "decision", "source-sha"]).has(name)) {
+      throw new CliUsageError("UNKNOWN_OPTION", "Unknown command option");
+    }
+    if (options[name] !== undefined) {
+      throw new CliUsageError("DUPLICATE_OPTION", "Command option was repeated");
+    }
+    options[name] = value;
+  }
+  for (const required of ["actor", "article", "decision", "source-sha"]) {
+    if (!options[required]) {
+      throw new CliUsageError("MISSING_OPTION", `Missing required option: --${required}`);
+    }
+  }
+  return options;
+}
+
+export function confirmationPhrase(decision, sourceSha) {
+  const verb = decision === "approved" ? "APPROVE" : "REJECT";
+  return `${verb} ${sourceSha}`;
+}
+
+export function requireInteractiveTerminal(input, output) {
+  if (!input.isTTY || !output.isTTY) {
+    throw new CliUsageError(
+      "INTERACTIVE_TERMINAL_REQUIRED",
+      "Approval decisions require an interactive terminal",
+    );
+  }
+}
+
+async function promptForDecision(request, input, output) {
+  requireInteractiveTerminal(input, output);
+  const expected = confirmationPhrase(request.decision, request.sourceSha);
+  output.write(
+    [
+      `Article: ${request.articleId}`,
+      `Site: ${request.siteId}`,
+      `Immutable source: ${request.sourceSha}`,
+      `Type exactly: ${expected}`,
+      "",
+    ].join("\n"),
+  );
+  const prompt = createInterface({ input, output });
+  try {
+    return (await prompt.question("> ")) === expected;
+  } finally {
+    prompt.close();
+  }
+}
+
+export async function main(
+  argv = process.argv.slice(2),
+  { input = process.stdin, output = process.stdout, errorOutput = process.stderr } = {},
+) {
+  try {
+    const [command, ...argumentsList] = argv;
+    if (command !== "approve") {
+      throw new CliUsageError("UNKNOWN_COMMAND", "Expected command: approve");
+    }
+    const options = parseOptions(argumentsList);
+    const result = await recordApprovalDecision({
+      root: process.cwd(),
+      articlePath: options.article,
+      sourceSha: options["source-sha"],
+      actor: options.actor,
+      decision: options.decision,
+      confirm: (request) => promptForDecision(request, input, output),
+    });
+    output.write(`Recorded ${result.event.decision}: ${result.eventPath}\n`);
+    return 0;
+  } catch (error) {
+    errorOutput.write(`ERROR ${error.code || "UNEXPECTED"}: ${error.message}\n`);
+    return 1;
+  }
+}
+
+if (process.argv[1] && fileURLToPath(import.meta.url) === path.resolve(process.argv[1])) {
+  process.exitCode = await main();
+}
diff --git a/test/cli.test.js b/test/cli.test.js
new file mode 100644
index 0000000..f4ceb9b
--- /dev/null
+++ b/test/cli.test.js
@@ -0,0 +1,49 @@
+import assert from "node:assert/strict";
+import test from "node:test";
+
+import {
+  confirmationPhrase,
+  parseOptions,
+  requireInteractiveTerminal,
+} from "../src/cli.js";
+
+test("approval CLI accepts only explicit complete option pairs", () => {
+  assert.deepEqual(
+    parseOptions([
+      "--article",
+      "content/articles/publisher-loop.md",
+      "--source-sha",
+      "a".repeat(40),
+      "--actor",
+      "Local reviewer",
+      "--decision",
+      "approved",
+    ]),
+    {
+      article: "content/articles/publisher-loop.md",
+      "source-sha": "a".repeat(40),
+      actor: "Local reviewer",
+      decision: "approved",
+    },
+  );
+  assert.throws(() => parseOptions(["--yes", "true"]), { code: "UNKNOWN_OPTION" });
+  assert.throws(() => parseOptions(["--actor"]), { code: "INVALID_ARGUMENTS" });
+});
+
+test("confirmation phrase names the decision and full immutable SHA", () => {
+  const sha = "b".repeat(40);
+  assert.equal(confirmationPhrase("approved", sha), `APPROVE ${sha}`);
+  assert.equal(confirmationPhrase("rejected", sha), `REJECT ${sha}`);
+});
+
+test("approval CLI rejects non-interactive confirmation", () => {
+  assert.throws(
+    () => requireInteractiveTerminal({ isTTY: false }, { isTTY: true }),
+    { code: "INTERACTIVE_TERMINAL_REQUIRED" },
+  );
+  assert.throws(
+    () => requireInteractiveTerminal({ isTTY: true }, { isTTY: false }),
+    { code: "INTERACTIVE_TERMINAL_REQUIRED" },
+  );
+  assert.doesNotThrow(() => requireInteractiveTerminal({ isTTY: true }, { isTTY: true }));
+});


## `feat(publication): enforce committed approval gate`

diff --git a/src/approval-gate.js b/src/approval-gate.js
new file mode 100644
index 0000000..5408237
--- /dev/null
+++ b/src/approval-gate.js
@@ -0,0 +1,95 @@
+import { validateAuditEvent } from "./audit.js";
+import { validateArticleSource, validateSiteSource } from "./content.js";
+import { runGit } from "./git.js";
+
+export class ApprovalGateError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "ApprovalGateError";
+    this.code = code;
+  }
+}
+
+function statusPath(line) {
+  const value = line.slice(3);
+  const renamed = value.includes(" -> ") ? value.split(" -> ").at(-1) : value;
+  return renamed.replace(/^"|"$/g, "");
+}
+
+export async function assertPublicationWorktree(root) {
+  const status = await runGit(root, ["status", "--porcelain=v1", "--untracked-files=all"]);
+  const unsafe = status
+    .split("\n")
+    .filter(Boolean)
+    .map(statusPath)
+    .filter(
+      (filePath) =>
+        !filePath.startsWith(".publisher/events/publications/") &&
+        !filePath.startsWith(".publisher/publications/"),
+    );
+  if (unsafe.length > 0) {
+    throw new ApprovalGateError(
+      "UNCOMMITTED_PUBLICATION_STATE",
+      "Publication requires committed content and approval metadata",
+    );
+  }
+}
+
+export async function requireCommittedApproval({ root, articlePath, sourceSha }) {
+  await assertPublicationWorktree(root);
+  if (!/^[0-9a-f]{40}$/.test(sourceSha)) {
+    throw new ApprovalGateError("INVALID_SOURCE_SHA", "Publication source SHA is invalid");
+  }
+  const revisions = new Set((await runGit(root, ["rev-list", "HEAD"])).trim().split("\n"));
+  if (!revisions.has(sourceSha)) {
+    throw new ApprovalGateError(
+      "SOURCE_NOT_REACHABLE",
+      "Publication source is not reachable from the audited HEAD",
+    );
+  }
+  const listed = await runGit(root, [
+    "ls-tree",
+    "-r",
+    "--name-only",
+    "HEAD",
+    "--",
+    ".publisher/events/approvals",
+  ]);
+  const matching = [];
+  for (const eventPath of listed.split("\n").filter(Boolean)) {
+    let event;
+    try {
+      event = JSON.parse(await runGit(root, ["show", `HEAD:${eventPath}`]));
+    } catch {
+      throw new ApprovalGateError("INVALID_APPROVAL_EVENT", "Committed approval event is invalid");
+    }
+    try {
+      validateAuditEvent(event);
+    } catch {
+      throw new ApprovalGateError("INVALID_APPROVAL_EVENT", "Committed approval event is invalid");
+    }
+    if (event.article_path === articlePath && event.source_sha === sourceSha) matching.push(event);
+  }
+  matching.sort(
+    (left, right) =>
+      left.occurred_at.localeCompare(right.occurred_at) || left.event_id.localeCompare(right.event_id),
+  );
+  const approval = matching.at(-1);
+  if (!approval) {
+    throw new ApprovalGateError("APPROVAL_REQUIRED", "No committed approval exists for this source SHA");
+  }
+  if (approval.decision !== "approved") {
+    throw new ApprovalGateError("APPROVAL_REJECTED", "Latest committed decision rejects this source SHA");
+  }
+
+  const article = validateArticleSource(
+    await runGit(root, ["show", `${sourceSha}:${articlePath}`]),
+    articlePath,
+  );
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  const site = validateSiteSource(await runGit(root, ["show", `${sourceSha}:${sitePath}`]), sitePath);
+  if (article.frontMatter.id !== approval.article_id || site.id !== article.frontMatter.site) {
+    throw new ApprovalGateError("APPROVAL_IDENTITY_MISMATCH", "Approval identity does not match source content");
+  }
+  return Object.freeze({ approval: Object.freeze(approval), article, site, sitePath });
+}
diff --git a/test/approval-gate.test.js b/test/approval-gate.test.js
new file mode 100644
index 0000000..fcb4bd9
--- /dev/null
+++ b/test/approval-gate.test.js
@@ -0,0 +1,107 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { copyFile, mkdir, mkdtemp, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import { requireCommittedApproval } from "../src/approval-gate.js";
+import { writeAuditEvent } from "../src/audit.js";
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
+async function gateRepository(context) {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-gate-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  await mkdir(path.join(root, "content/articles"), { recursive: true });
+  await mkdir(path.join(root, "sites"));
+  await copyFile(articlePath, path.join(root, articlePath));
+  await copyFile("sites/wordpress-local.yml", path.join(root, "sites/wordpress-local.yml"));
+  await commit(root, "source");
+  const { stdout } = await execFileAsync("git", ["rev-parse", "HEAD"], { cwd: root });
+  return { root, sourceSha: stdout.trim() };
+}
+
+function decision(sourceSha, overrides = {}) {
+  return {
+    schema_version: 1,
+    event_id: "00000000-0000-4000-8000-000000000010",
+    event_type: "approval",
+    article_id: "wordpress-loop",
+    article_path: articlePath,
+    source_sha: sourceSha,
+    occurred_at: "2026-08-29T01:00:00Z",
+    decision: "approved",
+    actor: "Local reviewer",
+    ...overrides,
+  };
+}
+
+test("publication gate requires approval committed after the source", async (context) => {
+  const { root, sourceSha } = await gateRepository(context);
+  await assert.rejects(() => requireCommittedApproval({ root, articlePath, sourceSha }), {
+    code: "APPROVAL_REQUIRED",
+  });
+
+  await writeAuditEvent(root, decision(sourceSha));
+  await assert.rejects(() => requireCommittedApproval({ root, articlePath, sourceSha }), {
+    code: "UNCOMMITTED_PUBLICATION_STATE",
+  });
+  await commit(root, "approve");
+
+  const gated = await requireCommittedApproval({ root, articlePath, sourceSha });
+  assert.equal(gated.approval.decision, "approved");
+  assert.equal(gated.article.frontMatter.id, "wordpress-loop");
+  assert.equal(gated.site.engine, "wordpress");
+});
+
+test("a later committed rejection blocks the same immutable source", async (context) => {
+  const { root, sourceSha } = await gateRepository(context);
+  await writeAuditEvent(root, decision(sourceSha));
+  await commit(root, "approve");
+  await writeAuditEvent(
+    root,
+    decision(sourceSha, {
+      event_id: "00000000-0000-4000-8000-000000000011",
+      occurred_at: "2026-08-29T02:00:00Z",
+      decision: "rejected",
+    }),
+  );
+  await commit(root, "reject");
+
+  await assert.rejects(() => requireCommittedApproval({ root, articlePath, sourceSha }), {
+    code: "APPROVAL_REJECTED",
+  });
+});
+
+test("uncommitted article changes block publication", async (context) => {
+  const { root, sourceSha } = await gateRepository(context);
+  await writeAuditEvent(root, decision(sourceSha));
+  await commit(root, "approve");
+  await writeFile(path.join(root, articlePath), "changed outside review\n");
+
+  await assert.rejects(() => requireCommittedApproval({ root, articlePath, sourceSha }), {
+    code: "UNCOMMITTED_PUBLICATION_STATE",
+  });
+});
