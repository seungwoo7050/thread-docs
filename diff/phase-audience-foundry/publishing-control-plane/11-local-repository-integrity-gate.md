# 로컬 저장소 무결성 게이트

## `feat(content): validate canonical inputs`

diff --git a/src/content.js b/src/content.js
new file mode 100644
index 0000000..f364143
--- /dev/null
+++ b/src/content.js
@@ -0,0 +1,135 @@
+import { readFile } from "node:fs/promises";
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+
+import Ajv2020 from "ajv/dist/2020.js";
+import matter from "gray-matter";
+import { parse as parseYaml } from "yaml";
+
+const moduleDirectory = path.dirname(fileURLToPath(import.meta.url));
+const schemaDirectory = path.resolve(moduleDirectory, "../schemas");
+const identifier = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;
+const sensitiveKey = /(?:password|secret|token|credential|api[_-]?key)/i;
+const sensitiveValuePatterns = [
+  /-----BEGIN [A-Z ]*PRIVATE KEY-----/,
+  /\bAKIA[0-9A-Z]{16}\b/,
+  /\bghp_[A-Za-z0-9]{36,}\b/,
+  /\bgithub_pat_[A-Za-z0-9_]{40,}\b/,
+  /\bsk-[A-Za-z0-9_-]{20,}\b/,
+];
+
+export class ContentValidationError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "ContentValidationError";
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
+    article: ajv.compile(await load("article.schema.json")),
+    site: ajv.compile(await load("site.schema.json")),
+  };
+}
+
+const validators = await buildValidators();
+
+function parseYamlSafely(source, label) {
+  try {
+    return parseYaml(source, { maxAliasCount: 0, strict: true, uniqueKeys: true });
+  } catch {
+    throw new ContentValidationError("INVALID_YAML", `Invalid YAML in ${label}`);
+  }
+}
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
+function rejectSensitiveMaterial(raw, parsed, label) {
+  if (containsSensitiveKey(parsed) || sensitiveValuePatterns.some((pattern) => pattern.test(raw))) {
+    throw new ContentValidationError(
+      "SENSITIVE_MATERIAL",
+      `Sensitive material is not permitted in ${label}`,
+    );
+  }
+}
+
+function assertSchema(validate, value, label) {
+  if (validate(value)) return;
+  const issues = validate.errors
+    .map(({ instancePath, keyword }) => `${instancePath || "/"}:${keyword}`)
+    .join(", ");
+  throw new ContentValidationError("SCHEMA_INVALID", `${label} violates schema (${issues})`);
+}
+
+export function validateArticleSource(source, articlePath) {
+  let parsed;
+  try {
+    parsed = matter(source, {
+      engines: { yaml: (value) => parseYamlSafely(value, articlePath) },
+      language: "yaml",
+    });
+  } catch (error) {
+    if (error instanceof ContentValidationError) throw error;
+    throw new ContentValidationError("INVALID_FRONT_MATTER", `Invalid front matter in ${articlePath}`);
+  }
+  rejectSensitiveMaterial(source, parsed.data, articlePath);
+  assertSchema(validators.article, parsed.data, articlePath);
+  if (!parsed.content.trim()) {
+    throw new ContentValidationError("EMPTY_BODY", `${articlePath} has no Markdown body`);
+  }
+  return Object.freeze({ frontMatter: Object.freeze(parsed.data), body: parsed.content });
+}
+
+export function validateSiteSource(source, sitePath) {
+  const parsed = parseYamlSafely(source, sitePath);
+  rejectSensitiveMaterial(source, parsed, sitePath);
+  assertSchema(validators.site, parsed, sitePath);
+  return Object.freeze(parsed);
+}
+
+export async function loadArticleWithSite(root, articlePath) {
+  if (!/^content\/articles\/[a-z0-9]+(?:-[a-z0-9]+)*\.md$/.test(articlePath)) {
+    throw new ContentValidationError("INVALID_PATH", "Article path is outside the canonical layout");
+  }
+  const article = validateArticleSource(
+    await readFile(path.join(root, articlePath), "utf8"),
+    articlePath,
+  );
+  if (!identifier.test(article.frontMatter.site)) {
+    throw new ContentValidationError("INVALID_SITE", "Article site identifier is invalid");
+  }
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  let siteSource;
+  try {
+    siteSource = await readFile(path.join(root, sitePath), "utf8");
+  } catch (error) {
+    if (error.code === "ENOENT") {
+      throw new ContentValidationError("SITE_NOT_FOUND", `Configured site does not exist: ${sitePath}`);
+    }
+    throw error;
+  }
+  const site = validateSiteSource(siteSource, sitePath);
+  if (site.id !== article.frontMatter.site) {
+    throw new ContentValidationError("SITE_MISMATCH", `Site file identity does not match ${sitePath}`);
+  }
+  return Object.freeze({ article, articlePath, site, sitePath });
+}
diff --git a/test/content.test.js b/test/content.test.js
new file mode 100644
index 0000000..e3e5e86
--- /dev/null
+++ b/test/content.test.js
@@ -0,0 +1,94 @@
+import assert from "node:assert/strict";
+import { mkdir, mkdtemp, readFile, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+
+import {
+  ContentValidationError,
+  loadArticleWithSite,
+  validateArticleSource,
+  validateSiteSource,
+} from "../src/content.js";
+
+const fixturePath = "content/articles/publisher-loop.md";
+
+test("loads canonical Markdown and resolves its single site engine", async () => {
+  const loaded = await loadArticleWithSite(process.cwd(), fixturePath);
+
+  assert.equal(loaded.article.frontMatter.id, "publisher-loop");
+  assert.equal(loaded.site.id, "public-demo");
+  assert.equal(loaded.site.engine, "public_sites");
+  assert.match(loaded.article.body, /approves/);
+});
+
+test("rejects duplicate YAML keys without echoing their values", async () => {
+  const source = await readFile(fixturePath, "utf8");
+  const duplicate = source.replace("schema_version: 1", "schema_version: 1\nschema_version: 2");
+
+  assert.throws(
+    () => validateArticleSource(duplicate, fixturePath),
+    (error) =>
+      error instanceof ContentValidationError &&
+      error.code === "INVALID_YAML" &&
+      !error.message.includes("schema_version: 2"),
+  );
+});
+
+test("rejects empty bodies and article-owned engines", async () => {
+  const source = await readFile(fixturePath, "utf8");
+  const empty = source.replace(/---\n\n# A[\s\S]*$/, "---\n");
+  const engine = source.replace("site: public-demo", "site: public-demo\nengine: wordpress");
+
+  assert.throws(() => validateArticleSource(empty, fixturePath), { code: "EMPTY_BODY" });
+  assert.throws(() => validateArticleSource(engine, fixturePath), { code: "SCHEMA_INVALID" });
+});
+
+test("rejects credential-shaped fields and values with redacted errors", async () => {
+  const source = await readFile(fixturePath, "utf8");
+  const forbiddenField = source.replace("author: Content Foundry", "api_key: redacted");
+  const token = ["gh", "p_", "a".repeat(40)].join("");
+  const forbiddenValue = source.replace("author: Content Foundry", `author: ${token}`);
+
+  for (const candidate of [forbiddenField, forbiddenValue]) {
+    assert.throws(
+      () => validateArticleSource(candidate, fixturePath),
+      (error) => error.code === "SENSITIVE_MATERIAL" && !error.message.includes(token),
+    );
+  }
+});
+
+test("site configuration accepts environment names but no credential values", () => {
+  const valid = `
+schema_version: 1
+id: wordpress-local
+name: Local WordPress
+engine: wordpress
+wordpress:
+  base_url_env: PUBLISHER_WP_BASE_URL
+  username_env: PUBLISHER_WP_USERNAME
+  password_env: PUBLISHER_WP_PASSWORD
+  default_status: draft
+`;
+  assert.equal(validateSiteSource(valid, "sites/wordpress-local.yml").engine, "wordpress");
+  assert.throws(
+    () => validateSiteSource(`${valid}\npassword: redacted\n`, "sites/wordpress-local.yml"),
+    { code: "SENSITIVE_MATERIAL" },
+  );
+});
+
+test("rejects traversal and unresolved site references", async (context) => {
+  await assert.rejects(() => loadArticleWithSite(process.cwd(), "../README.md"), {
+    code: "INVALID_PATH",
+  });
+  const source = await readFile(fixturePath, "utf8");
+  const missing = source.replace("site: public-demo", "site: missing-site");
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-content-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await mkdir(path.join(root, "content/articles"), { recursive: true });
+  await writeFile(path.join(root, fixturePath), missing);
+
+  await assert.rejects(() => loadArticleWithSite(root, fixturePath), {
+    code: "SITE_NOT_FOUND",
+  });
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


## `test(gate): validate repository locally`

diff --git a/package.json b/package.json
index 36fb6c1..615451b 100644
--- a/package.json
+++ b/package.json
@@ -12,6 +12,7 @@
   },
   "scripts": {
     "approve": "node src/cli.js approve",
+    "check": "node src/repository-check.js && node --test",
     "cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
     "cms:web": "node src/admin-server.js",
     "publish": "node src/cli.js publish",
diff --git a/src/repository-check.js b/src/repository-check.js
new file mode 100644
index 0000000..e6373ca
--- /dev/null
+++ b/src/repository-check.js
@@ -0,0 +1,102 @@
+import { readFile, readdir } from "node:fs/promises";
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+
+import { parse as parseYaml } from "yaml";
+
+import { loadArticleWithSite } from "./content.js";
+import { runGit } from "./git.js";
+import { containsSensitiveMaterial } from "./sensitive.js";
+
+export class RepositoryCheckError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "RepositoryCheckError";
+    this.code = code;
+  }
+}
+
+export async function validateCanonicalRepository(root) {
+  const entries = await readdir(path.join(root, "content/articles"), { withFileTypes: true });
+  const articlePaths = entries
+    .filter((entry) => entry.isFile() && entry.name.endsWith(".md"))
+    .map((entry) => `content/articles/${entry.name}`)
+    .sort();
+  if (articlePaths.length === 0) {
+    throw new RepositoryCheckError("NO_ARTICLES", "Canonical repository has no articles");
+  }
+  const ids = new Set();
+  const slugs = new Set();
+  for (const articlePath of articlePaths) {
+    const { article } = await loadArticleWithSite(root, articlePath);
+    for (const [label, value, values] of [
+      ["id", article.frontMatter.id, ids],
+      ["slug", article.frontMatter.slug, slugs],
+    ]) {
+      if (values.has(value)) {
+        throw new RepositoryCheckError(
+          "DUPLICATE_ARTICLE_IDENTITY",
+          `Canonical article ${label} is duplicated`,
+        );
+      }
+      values.add(value);
+    }
+  }
+  return Object.freeze({ articleCount: articlePaths.length });
+}
+
+function structuredValue(filePath, source) {
+  if (!/^(?:content\/|sites\/|\.publisher\/|evidence\/|admin\/)/.test(filePath)) {
+    return {};
+  }
+  try {
+    if (filePath.endsWith(".json")) return JSON.parse(source);
+    if (/\.ya?ml$/.test(filePath)) return parseYaml(source, { uniqueKeys: true });
+  } catch {
+    // The owning parser reports malformed structured files separately.
+  }
+  return {};
+}
+
+export async function scanRepositorySecrets(root) {
+  const listed = await runGit(root, [
+    "ls-files",
+    "--cached",
+    "--others",
+    "--exclude-standard",
+    "-z",
+  ]);
+  const files = listed.split("\0").filter(Boolean);
+  for (const filePath of files) {
+    const bytes = await readFile(path.join(root, filePath));
+    if (bytes.includes(0)) continue;
+    const source = bytes.toString("utf8");
+    if (containsSensitiveMaterial(source, structuredValue(filePath, source))) {
+      throw new RepositoryCheckError(
+        "SENSITIVE_REPOSITORY_MATERIAL",
+        `Sensitive material is not permitted in repository file: ${filePath}`,
+      );
+    }
+  }
+  return Object.freeze({ fileCount: files.length });
+}
+
+export async function runRepositoryGate(root = process.cwd()) {
+  const [canonical, security] = await Promise.all([
+    validateCanonicalRepository(root),
+    scanRepositorySecrets(root),
+  ]);
+  return Object.freeze({ ...canonical, ...security });
+}
+
+if (process.argv[1] && fileURLToPath(import.meta.url) === path.resolve(process.argv[1])) {
+  try {
+    const result = await runRepositoryGate();
+    process.stdout.write(
+      `Validated ${result.articleCount} articles; scanned ${result.fileCount} repository files\n`,
+    );
+  } catch (error) {
+    process.stderr.write(`ERROR ${error.code || "UNEXPECTED"}: ${error.message}\n`);
+    process.exitCode = 1;
+  }
+}
diff --git a/test/repository-check.test.js b/test/repository-check.test.js
new file mode 100644
index 0000000..aa2367c
--- /dev/null
+++ b/test/repository-check.test.js
@@ -0,0 +1,35 @@
+import assert from "node:assert/strict";
+import { execFile } from "node:child_process";
+import { mkdtemp, rm, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+import { promisify } from "node:util";
+
+import {
+  runRepositoryGate,
+  scanRepositorySecrets,
+} from "../src/repository-check.js";
+
+const execFileAsync = promisify(execFile);
+
+test("repository gate validates both canonical engine fixtures", async () => {
+  const result = await runRepositoryGate(process.cwd());
+
+  assert.equal(result.articleCount, 2);
+  assert.ok(result.fileCount > 20);
+});
+
+test("repository scan catches an untracked credential without echoing it", async (context) => {
+  const root = await mkdtemp(path.join(os.tmpdir(), "publisher-secret-scan-"));
+  context.after(() => rm(root, { force: true, recursive: true }));
+  await execFileAsync("git", ["init", "-q"], { cwd: root });
+  const token = ["gh", "p_", "a".repeat(40)].join("");
+  await writeFile(path.join(root, "untracked.txt"), `${token}\n`);
+
+  await assert.rejects(() => scanRepositorySecrets(root), (error) => {
+    assert.equal(error.code, "SENSITIVE_REPOSITORY_MATERIAL");
+    assert.doesNotMatch(error.message, new RegExp(token));
+    return true;
+  });
+});
