# 정규 콘텐츠 계약과 입력 검증

## `feat(content): define canonical article and site`

diff --git a/content/articles/publisher-loop.md b/content/articles/publisher-loop.md
new file mode 100644
index 0000000..7fb1649
--- /dev/null
+++ b/content/articles/publisher-loop.md
@@ -0,0 +1,18 @@
+---
+schema_version: 1
+id: publisher-loop
+site: public-demo
+slug: publisher-loop
+title: A Verifiable Publishing Loop
+summary: A deterministic article used to prove the Publisher release boundary.
+author: Content Foundry
+created_at: 2026-08-29T00:00:00Z
+tags:
+  - publishing
+  - verification
+---
+
+# A Verifiable Publishing Loop
+
+Canonical Markdown stays reviewable in private Git. Publication starts only after
+a person approves the exact source commit, and a receipt records the result.
diff --git a/schemas/article.schema.json b/schemas/article.schema.json
new file mode 100644
index 0000000..cc1b543
--- /dev/null
+++ b/schemas/article.schema.json
@@ -0,0 +1,40 @@
+{
+  "$schema": "https://json-schema.org/draft/2020-12/schema",
+  "$id": "https://content-foundry.local/schemas/article.schema.json",
+  "title": "Canonical article front matter",
+  "type": "object",
+  "additionalProperties": false,
+  "required": [
+    "schema_version",
+    "id",
+    "site",
+    "slug",
+    "title",
+    "summary",
+    "author",
+    "created_at"
+  ],
+  "properties": {
+    "schema_version": { "const": 1 },
+    "id": { "$ref": "#/$defs/identifier" },
+    "site": { "$ref": "#/$defs/identifier" },
+    "slug": { "$ref": "#/$defs/identifier" },
+    "title": { "type": "string", "minLength": 1, "maxLength": 160 },
+    "summary": { "type": "string", "minLength": 1, "maxLength": 300 },
+    "author": { "type": "string", "minLength": 1, "maxLength": 100 },
+    "created_at": { "type": "string", "format": "date-time" },
+    "tags": {
+      "type": "array",
+      "maxItems": 10,
+      "uniqueItems": true,
+      "items": { "type": "string", "minLength": 1, "maxLength": 40 }
+    }
+  },
+  "$defs": {
+    "identifier": {
+      "type": "string",
+      "pattern": "^[a-z0-9]+(?:-[a-z0-9]+)*$",
+      "maxLength": 80
+    }
+  }
+}
diff --git a/schemas/site.schema.json b/schemas/site.schema.json
new file mode 100644
index 0000000..cd1a05c
--- /dev/null
+++ b/schemas/site.schema.json
@@ -0,0 +1,77 @@
+{
+  "$schema": "https://json-schema.org/draft/2020-12/schema",
+  "$id": "https://content-foundry.local/schemas/site.schema.json",
+  "title": "Publication site",
+  "oneOf": [
+    { "$ref": "#/$defs/publicSites" },
+    { "$ref": "#/$defs/wordpress" }
+  ],
+  "$defs": {
+    "base": {
+      "type": "object",
+      "required": ["schema_version", "id", "name", "engine"],
+      "properties": {
+        "schema_version": { "const": 1 },
+        "id": {
+          "type": "string",
+          "pattern": "^[a-z0-9]+(?:-[a-z0-9]+)*$",
+          "maxLength": 80
+        },
+        "name": { "type": "string", "minLength": 1, "maxLength": 100 },
+        "engine": { "enum": ["public_sites", "wordpress"] }
+      }
+    },
+    "publicSites": {
+      "allOf": [
+        { "$ref": "#/$defs/base" },
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["public_sites"],
+          "properties": {
+            "schema_version": true,
+            "id": true,
+            "name": true,
+            "engine": { "const": "public_sites" },
+            "public_sites": {
+              "type": "object",
+              "additionalProperties": false,
+              "required": ["target"],
+              "properties": {
+                "target": { "type": "string", "minLength": 1, "maxLength": 100 }
+              }
+            }
+          }
+        }
+      ]
+    },
+    "wordpress": {
+      "allOf": [
+        { "$ref": "#/$defs/base" },
+        {
+          "type": "object",
+          "additionalProperties": false,
+          "required": ["wordpress"],
+          "properties": {
+            "schema_version": true,
+            "id": true,
+            "name": true,
+            "engine": { "const": "wordpress" },
+            "wordpress": {
+              "type": "object",
+              "additionalProperties": false,
+              "required": ["base_url_env", "username_env", "password_env", "default_status"],
+              "properties": {
+                "base_url_env": { "$ref": "#/$defs/envName" },
+                "username_env": { "$ref": "#/$defs/envName" },
+                "password_env": { "$ref": "#/$defs/envName" },
+                "default_status": { "const": "draft" }
+              }
+            }
+          }
+        }
+      ]
+    },
+    "envName": { "type": "string", "pattern": "^[A-Z][A-Z0-9_]*$" }
+  }
+}
diff --git a/sites/public-demo.yml b/sites/public-demo.yml
new file mode 100644
index 0000000..f5e966f
--- /dev/null
+++ b/sites/public-demo.yml
@@ -0,0 +1,6 @@
+schema_version: 1
+id: public-demo
+name: Public Sites viability target
+engine: public_sites
+public_sites:
+  target: publisher-viability
diff --git a/test/schema.test.js b/test/schema.test.js
new file mode 100644
index 0000000..d8995bc
--- /dev/null
+++ b/test/schema.test.js
@@ -0,0 +1,72 @@
+import assert from "node:assert/strict";
+import { readFile } from "node:fs/promises";
+import test from "node:test";
+
+import Ajv2020 from "ajv/dist/2020.js";
+import matter from "gray-matter";
+import { parse as parseYaml } from "yaml";
+
+const loadJson = async (path) => JSON.parse(await readFile(path, "utf8"));
+const loadYaml = async (path) => parseYaml(await readFile(path, "utf8"), {
+  uniqueKeys: true,
+});
+
+async function validators() {
+  const ajv = new Ajv2020({
+    allErrors: true,
+    formats: { "date-time": true },
+    strict: true,
+  });
+  return {
+    article: ajv.compile(await loadJson("schemas/article.schema.json")),
+    site: ajv.compile(await loadJson("schemas/site.schema.json")),
+  };
+}
+
+test("deterministic article and site fixtures satisfy their schemas", async () => {
+  const { article, site } = await validators();
+  const source = await readFile("content/articles/publisher-loop.md", "utf8");
+  const parsed = matter(source, {
+    engines: { yaml: (value) => parseYaml(value, { uniqueKeys: true }) },
+    language: "yaml",
+  });
+
+  assert.equal(article(parsed.data), true, JSON.stringify(article.errors));
+  assert.match(parsed.content, /Canonical Markdown/);
+  assert.equal(
+    site(await loadYaml("sites/public-demo.yml")),
+    true,
+    JSON.stringify(site.errors),
+  );
+});
+
+test("article selects a site and cannot select an engine", async () => {
+  const { article } = await validators();
+  const invalid = {
+    schema_version: 1,
+    id: "wrong-model",
+    site: "public-demo",
+    engine: "wordpress",
+    slug: "wrong-model",
+    title: "Wrong model",
+    summary: "Articles do not own publication engines.",
+    author: "Content Foundry",
+    created_at: "2026-08-29T00:00:00Z",
+  };
+
+  assert.equal(article(invalid), false);
+  assert.match(JSON.stringify(article.errors), /additionalProperties/);
+});
+
+test("a site cannot configure both publication engines", async () => {
+  const { site } = await validators();
+  const invalid = await loadYaml("sites/public-demo.yml");
+  invalid.wordpress = {
+    base_url_env: "PUBLISHER_WP_BASE_URL",
+    username_env: "PUBLISHER_WP_USERNAME",
+    password_env: "PUBLISHER_WP_PASSWORD",
+    default_status: "draft",
+  };
+
+  assert.equal(site(invalid), false);
+});


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
