# 사이트 주도 게시 라우팅

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


## `feat(wordpress): configure local draft target`

diff --git a/.wp-env.json b/.wp-env.json
new file mode 100644
index 0000000..e1758b8
--- /dev/null
+++ b/.wp-env.json
@@ -0,0 +1,10 @@
+{
+  "$schema": "https://schemas.wp.org/trunk/wp-env.json",
+  "core": "https://downloads.wordpress.org/release/wordpress-7.1.zip",
+  "phpVersion": "8.3",
+  "port": 8888,
+  "testsPort": 8889,
+  "config": {
+    "WP_ENVIRONMENT_TYPE": "local"
+  }
+}
diff --git a/admin/config.yml b/admin/config.yml
index 483fe56..2cabf5e 100644
--- a/admin/config.yml
+++ b/admin/config.yml
@@ -44,3 +44,19 @@ collections:
             widget: object
             fields:
               - { name: target, label: Target, widget: string }
+      - name: wordpress-local
+        label: Local WordPress draft target
+        file: sites/wordpress-local.yml
+        fields:
+          - { name: schema_version, label: Schema version, widget: hidden, default: 1 }
+          - { name: id, label: Stable ID, widget: hidden }
+          - { name: name, label: Name, widget: string }
+          - { name: engine, label: Engine, widget: hidden }
+          - name: wordpress
+            label: WordPress runtime mapping
+            widget: object
+            fields:
+              - { name: base_url_env, label: Base URL environment name, widget: string }
+              - { name: username_env, label: Username environment name, widget: string }
+              - { name: password_env, label: Password environment name, widget: string }
+              - { name: default_status, label: Default remote status, widget: hidden, default: draft }
diff --git a/content/articles/wordpress-loop.md b/content/articles/wordpress-loop.md
new file mode 100644
index 0000000..3064227
--- /dev/null
+++ b/content/articles/wordpress-loop.md
@@ -0,0 +1,18 @@
+---
+schema_version: 1
+id: wordpress-loop
+site: wordpress-local
+slug: wordpress-loop
+title: A Retryable WordPress Draft
+summary: A deterministic article used to prove create, update, and retry without duplicates.
+author: Content Foundry
+created_at: 2026-08-29T00:00:00Z
+tags:
+  - publishing
+  - wordpress
+---
+
+# A Retryable WordPress Draft
+
+The local adapter creates a draft once, preserves its remote post ID, and updates
+that same draft when a retry follows a partial failure.
diff --git a/package.json b/package.json
index 0d53a12..e7efe19 100644
--- a/package.json
+++ b/package.json
@@ -14,7 +14,9 @@
     "approve": "node src/cli.js approve",
     "cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
     "cms:web": "node src/admin-server.js",
-    "test": "node --test"
+    "test": "node --test",
+    "wp:start": "wp-env start",
+    "wp:stop": "wp-env stop"
   },
   "dependencies": {
     "ajv": "8.20.0",
diff --git a/sites/wordpress-local.yml b/sites/wordpress-local.yml
new file mode 100644
index 0000000..b92b4ea
--- /dev/null
+++ b/sites/wordpress-local.yml
@@ -0,0 +1,9 @@
+schema_version: 1
+id: wordpress-local
+name: Local WordPress draft target
+engine: wordpress
+wordpress:
+  base_url_env: PUBLISHER_WP_BASE_URL
+  username_env: PUBLISHER_WP_USERNAME
+  password_env: PUBLISHER_WP_PASSWORD
+  default_status: draft
diff --git a/test/decap.test.js b/test/decap.test.js
index 03418c2..a9834d2 100644
--- a/test/decap.test.js
+++ b/test/decap.test.js
@@ -33,6 +33,12 @@ test("Decap maps editorial workflow to local Git and canonical fields", async ()
   ]);
   assert.equal(fields.includes("engine"), false);
   assert.equal(fields.includes("approval"), false);
+  assert.deepEqual(
+    config.collections
+      .find(({ name }) => name === "sites")
+      .files.map(({ name }) => name),
+    ["public-demo", "wordpress-local"],
+  );
 });
 
 test("local editor exposes only reviewed assets", async () => {
diff --git a/test/wp-env.test.js b/test/wp-env.test.js
new file mode 100644
index 0000000..22e110a
--- /dev/null
+++ b/test/wp-env.test.js
@@ -0,0 +1,33 @@
+import assert from "node:assert/strict";
+import { readFile } from "node:fs/promises";
+import test from "node:test";
+
+import { loadArticleWithSite } from "../src/content.js";
+
+test("wp-env pins a provider-free local WordPress runtime", async () => {
+  const config = JSON.parse(await readFile(".wp-env.json", "utf8"));
+
+  assert.equal(config.core, "https://downloads.wordpress.org/release/wordpress-7.1.zip");
+  assert.equal(config.phpVersion, "8.3");
+  assert.equal(config.port, 8888);
+  assert.equal(config.config.WP_ENVIRONMENT_TYPE, "local");
+  assert.equal(JSON.stringify(config).includes("password"), false);
+});
+
+test("WordPress fixture selects its site and draft policy", async () => {
+  const loaded = await loadArticleWithSite(
+    process.cwd(),
+    "content/articles/wordpress-loop.md",
+  );
+
+  assert.equal(loaded.site.engine, "wordpress");
+  assert.equal(loaded.site.wordpress.default_status, "draft");
+  assert.deepEqual(
+    [
+      loaded.site.wordpress.base_url_env,
+      loaded.site.wordpress.username_env,
+      loaded.site.wordpress.password_env,
+    ],
+    ["PUBLISHER_WP_BASE_URL", "PUBLISHER_WP_USERNAME", "PUBLISHER_WP_PASSWORD"],
+  );
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
