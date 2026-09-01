# 승인 기반 멱등 게시 파이프라인

## `chore(repo): establish publisher baseline`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..0c8e725
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,8 @@
+.DS_Store
+.env
+.env.*
+!.env.example
+node_modules/
+dist/
+.decap/
+.wp-env/
diff --git a/DEVELOPMENT-RULES.md b/DEVELOPMENT-RULES.md
new file mode 100644
index 0000000..bd6daf7
--- /dev/null
+++ b/DEVELOPMENT-RULES.md
@@ -0,0 +1,90 @@
+# Development rules
+
+These rules apply to hand-authored work in Content Foundry Publisher.
+
+## Priority
+
+Apply requirements in this order:
+
+1. safety, security, law, and license obligations
+2. explicit user intent
+3. correctness, integrity, and compatibility
+4. verifiable delivery and rollback boundaries
+5. commit-size conventions
+
+Never weaken correctness or safety to make a commit smaller.
+
+## Plan by reviewable atoms
+
+Before implementation, decompose work into the smallest complete changes that are independently explainable, testable, reviewable, and revertible. Each commit answers one primary review question.
+
+Implementation and the smallest focused tests required to prove it normally belong in the same commit. Formatting, dependency updates, vendored code, generated output, and unrelated cleanup belong in separate commits.
+
+For a planned atom, record when useful:
+
+- purpose and primary review question
+- dependencies and expected files
+- validation and rollback boundary
+- expected meaningful churn
+- reason for any size exception
+
+## Commit size
+
+For hand-authored changes, target 20–80 meaningful lines and one or two primary production files.
+
+Reconsider the boundary above 100 meaningful lines or three primary files. Split by default above 150 meaningful lines. A commit above 200 meaningful lines or five primary files requires a concrete exception explaining why validation and rollback are inseparable.
+
+These are review warnings, not hard CI failures. Do not count lockfiles, generated output, mechanically imported upstream source, or formatting-only churn as hand-authored meaningful lines. Isolate such churn and record its provenance.
+
+The initial repository scaffold is an explicit exception: the policy documents establish one indivisible root baseline before implementation begins.
+
+## Commit messages
+
+Use an imperative Conventional Commit subject with a narrow scope when helpful, for example:
+
+```text
+feat(content): validate article front matter
+fix(wordpress): preserve remote draft identity
+docs(policy): define publication approval gate
+```
+
+The body should state the reason, focused validation, and any non-obvious rollback or size exception. Do not add invented test claims.
+
+## Validation
+
+Run the narrowest relevant check while developing and the repository gate before publishing a completed series. Record what actually ran and distinguish automated evidence from manual acceptance.
+
+Never mark a gate as passing when it was skipped, unavailable, or run against a different commit. Pin release inputs and identify the exact approved source SHA in publication evidence.
+
+## Git discipline
+
+- Preserve unrelated user changes in a dirty worktree.
+- Do not rewrite published history, force-push, reset destructively, or delete branches without explicit authorization.
+- Keep dependency/vendor imports separate from custom behavior.
+- Preserve upstream copyright and license notices.
+- Prefer pinned dependencies over copying or forking upstream projects.
+- Keep repository credentials and runtime secrets outside Git.
+
+## Product invariants
+
+- Content begins as an unapproved draft.
+- Publication requires an explicit human approval tied to an immutable commit.
+- A site selects one publication engine; an article selects a site, not an engine.
+- Approval and publication are distinct, auditable events.
+- A failed publication remains retryable without silently changing approved content.
+- Public Sites integration uses only release-directory input and build-report output.
+- WordPress integration publishes drafts first unless a later, explicit policy authorizes another remote status.
+- Adapters must be idempotent and retain the remote publication identity.
+- Secrets and sensitive recovery material never enter content, logs, fixtures, receipts, or commits.
+
+## Completion evidence
+
+A completed feature reports:
+
+- exact commit SHA and clean/dirty state
+- focused and full checks actually run
+- security, migration, compatibility, and data-integrity implications
+- manual checkpoints still required
+- known risks or deferred work
+
+Documentation and reports must describe the repository as it exists, not as intended.
diff --git a/README.md b/README.md
new file mode 100644
index 0000000..65bf06e
--- /dev/null
+++ b/README.md
@@ -0,0 +1,52 @@
+# Content Foundry Publisher
+
+Content Foundry Publisher is a private, Git-backed editorial workspace for preparing, reviewing, approving, and publishing articles.
+
+This repository starts from a new root history. It does not continue ContentOps history and does not import ContentOps code. ContentOps is retired.
+
+## Product boundary
+
+The product owns:
+
+- Markdown article source and YAML front matter
+- editorial status and review through private Git branches and pull requests
+- an explicit human approval gate before every publication
+- deterministic validation and target-specific publication adapters
+- publication receipts that identify the approved source commit and remote result
+
+The product does not own:
+
+- autonomous approval or unattended publication
+- real-time issue discovery
+- a general-purpose blog hosting platform
+- secrets committed to Git
+- direct imports from the retired ContentOps implementation
+
+AI-assisted drafting may be added later, but generated text is always an unapproved draft. No model may approve or publish content.
+
+## Initial delivery direction
+
+1. Run the editorial workflow locally with Decap CMS local backend.
+2. Store canonical content as Markdown and YAML in this repository.
+3. Validate content before review and again at the approved commit.
+4. Require the user to approve the pull request or equivalent local review checkpoint.
+5. Publish through exactly one adapter selected by the site configuration:
+   - frozen Public Sites release-directory/build-report interface; or
+   - local WordPress through `wp-env`, designed to move later to WordPress.com Business.
+6. Record a non-secret publication receipt without changing the approved article content.
+
+Public Sites remains a separate frozen repository. Integration is permitted only through its public release-directory input and build-report output. Publisher code must not import Public Sites internals or modify its renderer as part of normal content publication.
+
+## Repository state
+
+This initial commit intentionally contains policy and product boundaries only. It makes no claim that an application, CMS, WordPress environment, cloud account, domain, or deployment exists.
+
+Implementation begins in a new Codex session from the fixed decisions in [`docs/PRODUCT-DECISIONS.md`](docs/PRODUCT-DECISIONS.md) and the rules in [`DEVELOPMENT-RULES.md`](DEVELOPMENT-RULES.md).
+
+## Security
+
+Never commit passwords, API tokens, recovery codes, TOTP seeds, cookies, provider credentials, or production identifiers. Account creation, payment, 2FA, and secret entry are interactive user checkpoints.
+
+## Licensing
+
+This private repository does not grant a general license to redistribute its original source. Third-party packages keep their own licenses and notices; see [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md). A future WordPress plugin must live in a clearly separated GPL-compatible package.
diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
new file mode 100644
index 0000000..f24050d
--- /dev/null
+++ b/THIRD_PARTY_NOTICES.md
@@ -0,0 +1,7 @@
+# Third-party notices
+
+No third-party source code is vendored in the initial repository scaffold.
+
+Planned dependencies must be pinned and reviewed before introduction. Their upstream copyright and license notices remain in force. The initial candidates include Decap CMS, unified/remark, Ajv, WordPress `wp-env`, and official provider tooling; listing a candidate here does not declare it installed or approve a version.
+
+When a dependency is added, record its exact package, version or commit, upstream URL, license, integrity mechanism, and any redistribution obligations in the same dependency atom.
diff --git a/docs/PRODUCT-DECISIONS.md b/docs/PRODUCT-DECISIONS.md
new file mode 100644
index 0000000..a232d7e
--- /dev/null
+++ b/docs/PRODUCT-DECISIONS.md
@@ -0,0 +1,71 @@
+# Fixed product decisions
+
+This document is the starting contract for the first implementation session. Changing a fixed decision requires an explicit user decision and a dedicated documentation commit.
+
+## Status
+
+- ContentOps is retired and must not be used as an implementation base.
+- Publisher starts with unrelated Git history in a new private repository.
+- Public Sites is preserved as a frozen, independent renderer.
+- No Cloudflare account, domain, hosted WordPress instance, or production credential is assumed to exist.
+
+## Editorial model
+
+- Canonical articles are Markdown with YAML front matter.
+- Canonical work is stored in private Git.
+- Decap CMS is consumed as a pinned dependency, not copied wholesale or maintained as a product fork.
+- Draft/review state maps to branches and pull requests where practical.
+- The user must explicitly approve content before publication.
+- AI generation is outside the initial critical path and can never supply approval.
+
+## Publication model
+
+- An article references a site identifier.
+- Each site configuration chooses exactly one engine: Public Sites or WordPress.
+- Approval freezes the source commit; publishing consumes that immutable input.
+- Publishing produces a non-secret receipt containing the source SHA, target, timestamp, outcome, and remote identity or build-report identity.
+- Retries use idempotency keys and must not create duplicate remote posts.
+
+## Public Sites boundary
+
+- Pin the frozen Public Sites repository SHA before integration.
+- Treat its release-directory schema and build-report as the only supported interface.
+- Do not import renderer packages, source modules, or internal database concepts.
+- Prove viability with one real Publisher article compiled to a valid release, a target-specific build, and a verified build report before expanding the adapter.
+- If ordinary content requires invasive Public Sites changes, stop and reassess whether to replace the renderer.
+
+## WordPress boundary
+
+- Begin with local WordPress under `wp-env`; external WordPress setup is not a prerequisite.
+- Design the adapter so credentials and base URLs are runtime configuration.
+- Default remote creation to WordPress draft and retain its remote post ID.
+- Treat WordPress.com Business, plugin installation, payment, account login, and 2FA as later interactive user checkpoints.
+- Any WordPress plugin is a separate GPL-compatible package with its own license boundary.
+
+## External hosting
+
+- Cloudflare, DNS, domains, analytics, advertising, consent, and production deployment are deferred.
+- Later production topology may use one registered domain with two distinct origins, but no provider-specific value is invented now.
+- Use official provider APIs, CLIs, and GitHub Actions rather than custom credential transport.
+
+## Initial implementation sequence
+
+1. Establish schemas, repository layout, and deterministic fixtures.
+2. Add content validation and focused tests.
+3. Add Decap local editorial configuration.
+4. Implement approval metadata bound to Git commits.
+5. Prove the frozen Public Sites adapter with a viability spike.
+6. Implement and test the local WordPress draft adapter.
+7. Add idempotent publication orchestration and receipts.
+8. Exercise rejection, retry, partial-failure, and rollback paths.
+9. Complete local end-to-end evidence and documentation.
+10. Stop at interactive checkpoints before any external account or production setup.
+
+## Explicit non-goals for the first phase
+
+- real-time trend or issue discovery
+- autonomous research, drafting, approval, or publishing
+- a new public-site renderer
+- multi-tenant SaaS administration
+- production Cloudflare or WordPress.com deployment
+- importing ContentOps code, migrations, contracts, or operational state


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


