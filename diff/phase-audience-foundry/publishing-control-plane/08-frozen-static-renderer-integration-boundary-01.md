# 동결 정적 렌더러 통합 경계

## `feat(public-sites): define frozen build report`

diff --git a/schemas/public-sites-build-report.schema.json b/schemas/public-sites-build-report.schema.json
new file mode 100644
index 0000000..2983f23
--- /dev/null
+++ b/schemas/public-sites-build-report.schema.json
@@ -0,0 +1,44 @@
+{
+  "$schema": "https://json-schema.org/draft/2020-12/schema",
+  "$id": "https://content-foundry.local/schemas/public-sites-build-report.schema.json",
+  "title": "Verified frozen Public Sites build report",
+  "type": "object",
+  "additionalProperties": false,
+  "required": [
+    "schema_version",
+    "outcome",
+    "source_sha",
+    "renderer_repository",
+    "renderer_sha",
+    "target",
+    "release_id",
+    "bundle_checksum",
+    "build_config_checksum",
+    "article_artifact",
+    "artifact_count",
+    "output_sha256",
+    "verified_at"
+  ],
+  "properties": {
+    "schema_version": { "const": 1 },
+    "outcome": { "const": "succeeded" },
+    "source_sha": { "$ref": "#/$defs/sha" },
+    "renderer_repository": { "const": "seungwoo7050/content-foundry-public-sites" },
+    "renderer_sha": { "const": "1717326cda8262d7f7f56d544b3a9d0215b71d51" },
+    "target": { "const": "site-a" },
+    "release_id": { "type": "string", "pattern": "^REL-[A-Z0-9-]+$" },
+    "bundle_checksum": { "$ref": "#/$defs/digest" },
+    "build_config_checksum": { "$ref": "#/$defs/digest" },
+    "article_artifact": {
+      "type": "string",
+      "pattern": "^article/[a-z0-9]+(?:-[a-z0-9]+)*\\.html$"
+    },
+    "artifact_count": { "type": "integer", "minimum": 1 },
+    "output_sha256": { "$ref": "#/$defs/digest" },
+    "verified_at": { "type": "string", "format": "date-time" }
+  },
+  "$defs": {
+    "sha": { "type": "string", "pattern": "^[0-9a-f]{40}$" },
+    "digest": { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" }
+  }
+}
diff --git a/test/schema.test.js b/test/schema.test.js
index 805da7e..f670932 100644
--- a/test/schema.test.js
+++ b/test/schema.test.js
@@ -20,6 +20,7 @@ async function validators() {
   return {
     auditEvent: ajv.compile(await loadJson("schemas/audit-event.schema.json")),
     article: ajv.compile(await loadJson("schemas/article.schema.json")),
+    buildReport: ajv.compile(await loadJson("schemas/public-sites-build-report.schema.json")),
     receipt: ajv.compile(await loadJson("schemas/publication-receipt.schema.json")),
     site: ajv.compile(await loadJson("schemas/site.schema.json")),
   };
@@ -93,3 +94,28 @@ test("audit and receipt schemas keep approval and publication distinct", async (
   assert.equal(auditEvent(approval), true, JSON.stringify(auditEvent.errors));
   assert.equal(receipt(approval), false);
 });
+
+test("Public Sites build reports pin the frozen renderer", async () => {
+  const { buildReport } = await validators();
+  const report = {
+    schema_version: 1,
+    outcome: "succeeded",
+    source_sha: "a".repeat(40),
+    renderer_repository: "seungwoo7050/content-foundry-public-sites",
+    renderer_sha: "1717326cda8262d7f7f56d544b3a9d0215b71d51",
+    target: "site-a",
+    release_id: "REL-EXAMPLE",
+    bundle_checksum: `sha256:${"b".repeat(64)}`,
+    build_config_checksum: `sha256:${"c".repeat(64)}`,
+    article_artifact: "article/publisher-loop.html",
+    artifact_count: 10,
+    output_sha256: `sha256:${"d".repeat(64)}`,
+    verified_at: "2026-08-29T00:00:00Z",
+  };
+
+  assert.equal(buildReport(report), true, JSON.stringify(buildReport.errors));
+  assert.equal(
+    buildReport({ ...report, renderer_sha: "e".repeat(40) }),
+    false,
+  );
+});


## `feat(public-sites): verify frozen target build`

diff --git a/src/public-sites-build-report.js b/src/public-sites-build-report.js
new file mode 100644
index 0000000..bc005a8
--- /dev/null
+++ b/src/public-sites-build-report.js
@@ -0,0 +1,144 @@
+import { createHash } from "node:crypto";
+import { lstat, mkdir, readFile, readdir, writeFile } from "node:fs/promises";
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+
+import Ajv2020 from "ajv/dist/2020.js";
+
+import { runGit } from "./git.js";
+import { containsSensitiveMaterial } from "./sensitive.js";
+
+export const PUBLIC_SITES_REPOSITORY = "seungwoo7050/content-foundry-public-sites";
+export const PUBLIC_SITES_SHA = "1717326cda8262d7f7f56d544b3a9d0215b71d51";
+
+export class PublicSitesBuildError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicSitesBuildError";
+    this.code = code;
+  }
+}
+
+const schemaPath = path.resolve(
+  path.dirname(fileURLToPath(import.meta.url)),
+  "../schemas/public-sites-build-report.schema.json",
+);
+const ajv = new Ajv2020({
+  allErrors: true,
+  formats: {
+    "date-time": (value) =>
+      /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z$/.test(value) &&
+      !Number.isNaN(Date.parse(value)),
+  },
+  strict: true,
+});
+const validateReport = ajv.compile(JSON.parse(await readFile(schemaPath, "utf8")));
+const sha256 = (bytes) => createHash("sha256").update(bytes).digest("hex");
+
+export function assertFrozenCheckoutState(head, trackedStatus) {
+  if (head !== PUBLIC_SITES_SHA) {
+    throw new PublicSitesBuildError("RENDERER_SHA_MISMATCH", "Public Sites checkout is not the frozen SHA");
+  }
+  if (trackedStatus.trim()) {
+    throw new PublicSitesBuildError("RENDERER_DIRTY", "Public Sites tracked files were modified");
+  }
+}
+
+async function listRegularFiles(root) {
+  const files = [];
+  async function visit(directory) {
+    for (const entry of await readdir(directory, { withFileTypes: true })) {
+      const absolute = path.join(directory, entry.name);
+      const metadata = await lstat(absolute);
+      if (metadata.isSymbolicLink()) {
+        throw new PublicSitesBuildError("UNSAFE_BUILD_OUTPUT", "Build output contains a symbolic link");
+      }
+      if (metadata.isDirectory()) await visit(absolute);
+      else if (metadata.isFile()) files.push(path.relative(root, absolute).split(path.sep).join("/"));
+      else throw new PublicSitesBuildError("UNSAFE_BUILD_OUTPUT", "Build output contains a special file");
+    }
+  }
+  await visit(root);
+  return files.sort((left, right) => Buffer.from(left).compare(Buffer.from(right)));
+}
+
+async function outputDigest(root, files) {
+  const lines = [];
+  for (const relativePath of files) {
+    lines.push(`${sha256(await readFile(path.join(root, relativePath)))}  ${relativePath}\n`);
+  }
+  return `sha256:${sha256(Buffer.from(lines.join(""), "utf8"))}`;
+}
+
+export async function verifyPublicSitesBuild({
+  repositoryRoot,
+  releaseDirectory,
+  outputDirectory,
+  sourceSha,
+  reportPath,
+  now = () => new Date(),
+}) {
+  if (!/^[0-9a-f]{40}$/.test(sourceSha)) {
+    throw new PublicSitesBuildError("INVALID_SOURCE_SHA", "Build report requires a full source SHA");
+  }
+  const head = (await runGit(repositoryRoot, ["rev-parse", "HEAD"])).trim();
+  const trackedStatus = await runGit(repositoryRoot, [
+    "status",
+    "--porcelain=v1",
+    "--untracked-files=no",
+  ]);
+  assertFrozenCheckoutState(head, trackedStatus);
+
+  const release = JSON.parse(await readFile(path.join(releaseDirectory, "release.json"), "utf8"));
+  const articles = await readdir(path.join(releaseDirectory, "articles"), { withFileTypes: true });
+  if (articles.length !== 1 || !articles[0].isFile() || !articles[0].name.endsWith(".json")) {
+    throw new PublicSitesBuildError("UNEXPECTED_RELEASE_SHAPE", "Viability release must contain one article");
+  }
+  const article = JSON.parse(
+    await readFile(path.join(releaseDirectory, "articles", articles[0].name), "utf8"),
+  );
+  const identity = JSON.parse(await readFile(path.join(outputDirectory, "_release.json"), "utf8"));
+  for (const key of ["releaseId", "siteId", "bundleChecksum", "contractVersion"]) {
+    if (identity[key] !== release[key]) {
+      throw new PublicSitesBuildError("BUILD_IDENTITY_MISMATCH", "Build identity does not match release input");
+    }
+  }
+  if (identity.contractVersion !== "4.0.0" || identity.siteId !== "site-a") {
+    throw new PublicSitesBuildError("UNSUPPORTED_BUILD_IDENTITY", "Build target identity is unsupported");
+  }
+  const articleArtifact = `article/${article.slug}.html`;
+  for (const required of ["index.html", articleArtifact, "category/articles.html", "robots.txt", "sitemap.xml"]) {
+    const metadata = await lstat(path.join(outputDirectory, required));
+    if (!metadata.isFile()) {
+      throw new PublicSitesBuildError("MISSING_BUILD_ARTIFACT", "Required build artifact is missing");
+    }
+  }
+  const files = await listRegularFiles(outputDirectory);
+  const report = {
+    schema_version: 1,
+    outcome: "succeeded",
+    source_sha: sourceSha,
+    renderer_repository: PUBLIC_SITES_REPOSITORY,
+    renderer_sha: PUBLIC_SITES_SHA,
+    target: "site-a",
+    release_id: release.releaseId,
+    bundle_checksum: release.bundleChecksum,
+    build_config_checksum: identity.buildConfigChecksum,
+    article_artifact: articleArtifact,
+    artifact_count: files.length,
+    output_sha256: await outputDigest(outputDirectory, files),
+    verified_at: now().toISOString(),
+  };
+  const serialized = `${JSON.stringify(report, null, 2)}\n`;
+  if (!validateReport(report)) {
+    throw new PublicSitesBuildError("INVALID_BUILD_REPORT", "Generated build report violates schema");
+  }
+  if (containsSensitiveMaterial(serialized, report)) {
+    throw new PublicSitesBuildError("SENSITIVE_BUILD_REPORT", "Build report contains sensitive material");
+  }
+  if (reportPath) {
+    await mkdir(path.dirname(reportPath), { recursive: true });
+    await writeFile(reportPath, serialized, { encoding: "utf8", flag: "wx", mode: 0o600 });
+  }
+  return Object.freeze({ report: Object.freeze(report), reportSha256: sha256(serialized) });
+}
diff --git a/test/public-sites-build-report.test.js b/test/public-sites-build-report.test.js
new file mode 100644
index 0000000..ccb028c
--- /dev/null
+++ b/test/public-sites-build-report.test.js
@@ -0,0 +1,57 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { mkdir, mkdtemp, readFile, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import {
+  assertFrozenCheckoutState,
+  PUBLIC_SITES_SHA,
+  verifyPublicSitesBuild,
+} from "../src/public-sites-build-report.js";
+
+const execFileAsync = promisify(execFile);
+
+test("frozen checkout state rejects any SHA or tracked mutation", () => {
+  assert.doesNotThrow(() => assertFrozenCheckoutState(PUBLIC_SITES_SHA, ""));
+  assert.throws(() => assertFrozenCheckoutState("a".repeat(40), ""), {
+    code: "RENDERER_SHA_MISMATCH",
+  });
+  assert.throws(() => assertFrozenCheckoutState(PUBLIC_SITES_SHA, " M tracked.ts\n"), {
+    code: "RENDERER_DIRTY",
+  });
+});
+
+test("build verification fails before trusting a non-frozen repository", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-build-report-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  await writeFile(path.join(root, "README.md"), "test\n");
+  await execFileAsync("git", ["add", "."], { cwd: root });
+  await execFileAsync(
+    "git",
+    ["-c", "user.name=Test", "-c", "user.email=test@example.invalid", "commit", "-q", "-m", "test"],
+    { cwd: root },
+  );
+
+  await assert.rejects(
+    () =>
+      verifyPublicSitesBuild({
+        repositoryRoot: root,
+        releaseDirectory: path.join(root, "missing-release"),
+        outputDirectory: path.join(root, "missing-output"),
+        sourceSha: "b".repeat(40),
+      }),
+    { code: "RENDERER_SHA_MISMATCH" },
+  );
+});
+
+test("report schema fixture remains non-secret", async () => {
+  const schema = JSON.parse(
+    await readFile("schemas/public-sites-build-report.schema.json", "utf8"),
+  );
+  assert.equal(schema.properties.renderer_sha.const, PUBLIC_SITES_SHA);
+  assert.equal(schema.properties.outcome.const, "succeeded");
+});


## `test(public-sites): capture viability build`

diff --git a/docs/PUBLIC-SITES-VIABILITY.md b/docs/PUBLIC-SITES-VIABILITY.md
new file mode 100644
index 0000000..51cc359
--- /dev/null
+++ b/docs/PUBLIC-SITES-VIABILITY.md
@@ -0,0 +1,48 @@
+# Public Sites viability evidence
+
+The first real Publisher article passed the frozen Public Sites release boundary
+without changing Public Sites. This is provider-free local evidence, not a public
+deployment or approval to publish externally.
+
+## Immutable inputs
+
+- Publisher source commit: `a62af56dde3f7dc9d6a7bee0ec7f81e83e7b2dc8`
+- Article: `content/articles/publisher-loop.md`
+- Renderer repository: `seungwoo7050/content-foundry-public-sites`
+- Detached renderer commit: `1717326cda8262d7f7f56d544b3a9d0215b71d51`
+- Renderer target: `site-a`
+- Node: `24.19.0`
+- pnpm: `11.22.0`
+
+The Publisher compiler generated a new v4 release from canonical Markdown. It
+did not copy a legacy producer fixture, import renderer modules, or write to the
+archived repository's tracked files.
+
+## Checks actually run
+
+1. Frozen checkout: exact detached SHA, no tracked diff.
+2. Locked install: `pnpm install --frozen-lockfile`.
+3. Workspace prerequisites: six dependency builds completed uncached.
+4. Release preparation and target build: `RELEASE_MODE=preview`, analytics and
+   ads disabled, no provider identifier or credential supplied.
+5. Next.js 16.3.3 generated 14 static routes, including
+   `/article/publisher-loop`, `/category/articles`, and `/about`.
+6. Publisher verification matched `_release.json` to the release input, required
+   the article/category/static metadata artifacts, rejected renderer mutations,
+   and fingerprinted all 64 regular output files.
+
+The standalone frozen `validate:release` command intentionally cannot validate
+v4 without the target's release-mode consumer context. The target build supplied
+that context and performed the successful contract/reference validation.
+
+The captured report is
+`evidence/public-sites-viability-build-report.json`; its exact file SHA-256 is
+`e58ee42735385d0ef3607ebca8883923602773815ac0e17a8b5dc0667e9667b9`.
+
+## Result
+
+The interface is viable for an ordinary title, summary, tags, author, timestamp,
+H1, and Markdown paragraph. The current projection also supports H2-H6 headings
+and ordered/unordered lists, and fails closed on duplicate or unprojectable
+headings. Rich media and other structured blocks are outside this spike and do
+not require a Public Sites modification for the proved MVP article.
diff --git a/evidence/public-sites-viability-build-report.json b/evidence/public-sites-viability-build-report.json
new file mode 100644
index 0000000..764a1bb
--- /dev/null
+++ b/evidence/public-sites-viability-build-report.json
@@ -0,0 +1,15 @@
+{
+  "schema_version": 1,
+  "outcome": "succeeded",
+  "source_sha": "a62af56dde3f7dc9d6a7bee0ec7f81e83e7b2dc8",
+  "renderer_repository": "seungwoo7050/content-foundry-public-sites",
+  "renderer_sha": "1717326cda8262d7f7f56d544b3a9d0215b71d51",
+  "target": "site-a",
+  "release_id": "REL-A62AF56DDE3F",
+  "bundle_checksum": "sha256:3cf437bef85964b38b7356660faf414693ea103ea4b56a9c049e06bc229ca186",
+  "build_config_checksum": "sha256:18483304689d4499c1a862ac5e36944476c32304334b859c7ada448b37a7e871",
+  "article_artifact": "article/publisher-loop.html",
+  "artifact_count": 64,
+  "output_sha256": "sha256:28d854cccc32280182e893c0ae3c01b7d68e5fb855deda33d0c088ee2b9587fc",
+  "verified_at": "2026-08-29T05:00:00.000Z"
+}
diff --git a/test/schema.test.js b/test/schema.test.js
index f670932..469fb87 100644
--- a/test/schema.test.js
+++ b/test/schema.test.js
@@ -118,4 +118,7 @@ test("Public Sites build reports pin the frozen renderer", async () => {
     buildReport({ ...report, renderer_sha: "e".repeat(40) }),
     false,
   );
+
+  const captured = await loadJson("evidence/public-sites-viability-build-report.json");
+  assert.equal(buildReport(captured), true, JSON.stringify(buildReport.errors));
 });


## `feat(public-sites): add frozen build adapter`

diff --git a/src/adapters/public-sites.js b/src/adapters/public-sites.js
new file mode 100644
index 0000000..81872e1
--- /dev/null
+++ b/src/adapters/public-sites.js
@@ -0,0 +1,166 @@
+import { execFile } from "node:child_process";
+import { randomUUID } from "node:crypto";
+import { mkdir } from "node:fs/promises";
+import path from "node:path";
+import { promisify } from "node:util";
+
+import {
+  assertFrozenCheckoutState,
+  verifyPublicSitesBuild,
+} from "../public-sites-build-report.js";
+import { compilePublicSitesRelease } from "../public-sites-release.js";
+import { runGit } from "../git.js";
+
+const execFileAsync = promisify(execFile);
+const inheritedEnvironment = [
+  "COREPACK_HOME",
+  "FNM_DIR",
+  "HOME",
+  "LANG",
+  "LC_ALL",
+  "LOGNAME",
+  "PATH",
+  "PNPM_HOME",
+  "SHELL",
+  "TEMP",
+  "TMP",
+  "TMPDIR",
+  "USER",
+  "XDG_CACHE_HOME",
+];
+
+export class PublicSitesAdapterError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicSitesAdapterError";
+    this.code = code;
+  }
+}
+
+function buildEnvironment(releaseDirectory) {
+  const environment = {};
+  for (const name of inheritedEnvironment) {
+    if (process.env[name] !== undefined) environment[name] = process.env[name];
+  }
+  return {
+    ...environment,
+    CI: "1",
+    CONTENT_RELEASE_DIR: releaseDirectory,
+    ENABLE_ADS: "false",
+    ENABLE_ANALYTICS: "false",
+    RELEASE_MODE: "preview",
+    SITE_ORIGIN: "",
+  };
+}
+
+async function runProcess(command, args, options) {
+  await execFileAsync(command, args, {
+    ...options,
+    encoding: "utf8",
+    maxBuffer: 8 * 1024 * 1024,
+  });
+}
+
+async function inspectFrozenCheckout(repositoryRoot) {
+  const head = (await runGit(repositoryRoot, ["rev-parse", "HEAD"])).trim();
+  const trackedStatus = await runGit(repositoryRoot, [
+    "status",
+    "--porcelain=v1",
+    "--untracked-files=no",
+  ]);
+  assertFrozenCheckoutState(head, trackedStatus);
+}
+
+function assertExternalWorkRoot(repositoryRoot, workRoot) {
+  const relative = path.relative(path.resolve(repositoryRoot), path.resolve(workRoot));
+  if (relative === "" || (!relative.startsWith("..") && !path.isAbsolute(relative))) {
+    throw new PublicSitesAdapterError(
+      "WORK_ROOT_INSIDE_RENDERER",
+      "Publication work must remain outside the frozen renderer checkout",
+    );
+  }
+}
+
+export function createPublicSitesAdapter({
+  compileRelease = compilePublicSitesRelease,
+  inspectCheckout = inspectFrozenCheckout,
+  runCommand = runProcess,
+  verifyBuild = verifyPublicSitesBuild,
+  uuid = randomUUID,
+} = {}) {
+  return async function publishPublicSites({
+    publisherRoot,
+    publicSitesRepository,
+    workRoot,
+    articlePath,
+    sourceSha,
+    now = () => new Date(),
+  }) {
+    assertExternalWorkRoot(publicSitesRepository, workRoot);
+    await inspectCheckout(publicSitesRepository);
+    const attemptRoot = path.join(workRoot, uuid());
+    await mkdir(attemptRoot, { recursive: false });
+    const releaseDirectory = path.join(attemptRoot, "release");
+    const reportPath = path.join(attemptRoot, "build-report.json");
+    await compileRelease({
+      root: publisherRoot,
+      articlePath,
+      sourceSha,
+      outputDirectory: releaseDirectory,
+    });
+    const environment = buildEnvironment(releaseDirectory);
+    try {
+      await runCommand(
+        "fnm",
+        [
+          "exec",
+          "--using=24.19.0",
+          "--",
+          "corepack",
+          "pnpm",
+          "exec",
+          "turbo",
+          "run",
+          "build",
+          "--filter=@content-foundry/site-a^...",
+        ],
+        { cwd: publicSitesRepository, env: environment },
+      );
+      await runCommand(
+        "fnm",
+        [
+          "exec",
+          "--using=24.19.0",
+          "--",
+          "corepack",
+          "pnpm",
+          "--filter",
+          "@content-foundry/site-a",
+          "build",
+        ],
+        { cwd: publicSitesRepository, env: environment },
+      );
+    } catch {
+      throw new PublicSitesAdapterError(
+        "PUBLIC_SITES_BUILD_FAILED",
+        "Frozen Public Sites target build failed",
+      );
+    }
+    const verified = await verifyBuild({
+      repositoryRoot: publicSitesRepository,
+      releaseDirectory,
+      outputDirectory: path.join(publicSitesRepository, "apps/site-a/out"),
+      sourceSha,
+      reportPath,
+      now,
+    });
+    return Object.freeze({
+      kind: "public_sites",
+      target: "site-a",
+      build_report_sha256: verified.reportSha256,
+      buildReportPath: reportPath,
+    });
+  };
+}
+
+export const publishToPublicSites = createPublicSitesAdapter();
diff --git a/test/public-sites-adapter.test.js b/test/public-sites-adapter.test.js
new file mode 100644
index 0000000..1f9bb6c
--- /dev/null
+++ b/test/public-sites-adapter.test.js
@@ -0,0 +1,94 @@
+import assert from "node:assert/strict";
+import { mkdir, mkdtemp, rm } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+
+import { createPublicSitesAdapter } from "../src/adapters/public-sites.js";
+
+test("adapter runs the frozen target with provider-free environment", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-public-adapter-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const renderer = path.join(root, "renderer");
+  const work = path.join(root, "work");
+  await mkdir(renderer);
+  await mkdir(work);
+  const calls = [];
+  const adapter = createPublicSitesAdapter({
+    compileRelease: async ({ outputDirectory }) => mkdir(outputDirectory),
+    inspectCheckout: async () => {},
+    runCommand: async (command, args, options) => calls.push({ command, args, options }),
+    verifyBuild: async ({ reportPath }) => {
+      assert.match(reportPath, /build-report\.json$/);
+      return { reportSha256: "a".repeat(64) };
+    },
+    uuid: () => "attempt-one",
+  });
+
+  const result = await adapter({
+    publisherRoot: root,
+    publicSitesRepository: renderer,
+    workRoot: work,
+    articlePath: "content/articles/publisher-loop.md",
+    sourceSha: "b".repeat(40),
+  });
+
+  assert.equal(calls.length, 2);
+  assert.equal(calls[0].command, "fnm");
+  assert.match(calls[0].args.join(" "), /site-a\^\.\.\./);
+  assert.match(calls[1].args.join(" "), /@content-foundry\/site-a build/);
+  assert.equal(calls[1].options.env.RELEASE_MODE, "preview");
+  assert.equal(calls[1].options.env.ENABLE_ADS, "false");
+  assert.equal(calls[1].options.env.ENABLE_ANALYTICS, "false");
+  assert.equal(calls[1].options.env.SITE_ORIGIN, "");
+  assert.equal(result.build_report_sha256, "a".repeat(64));
+});
+
+test("failed target build is retryable without exposing command output", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-public-adapter-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  const renderer = path.join(root, "renderer");
+  const work = path.join(root, "work");
+  await mkdir(renderer);
+  await mkdir(work);
+  let attempt = 0;
+  const adapter = createPublicSitesAdapter({
+    compileRelease: async ({ outputDirectory }) => mkdir(outputDirectory),
+    inspectCheckout: async () => {},
+    runCommand: async () => {
+      if (attempt++ === 0) throw new Error("provider output must stay private");
+    },
+    verifyBuild: async () => ({ reportSha256: "c".repeat(64) }),
+    uuid: () => `attempt-${attempt}`,
+  });
+  const input = {
+    publisherRoot: root,
+    publicSitesRepository: renderer,
+    workRoot: work,
+    articlePath: "content/articles/publisher-loop.md",
+    sourceSha: "d".repeat(40),
+  };
+
+  await assert.rejects(() => adapter(input), (error) => {
+    assert.equal(error.code, "PUBLIC_SITES_BUILD_FAILED");
+    assert.doesNotMatch(error.message, /provider output/);
+    return true;
+  });
+  const retried = await adapter(input);
+  assert.equal(retried.build_report_sha256, "c".repeat(64));
+});
+
+test("adapter never writes publication work inside the renderer", async () => {
+  const adapter = createPublicSitesAdapter();
+  await assert.rejects(
+    () =>
+      adapter({
+        publisherRoot: "/publisher",
+        publicSitesRepository: "/renderer",
+        workRoot: "/renderer/output/work",
+        articlePath: "content/articles/publisher-loop.md",
+        sourceSha: "e".repeat(40),
+      }),
+    { code: "WORK_ROOT_INSIDE_RENDERER" },
+  );
+});


