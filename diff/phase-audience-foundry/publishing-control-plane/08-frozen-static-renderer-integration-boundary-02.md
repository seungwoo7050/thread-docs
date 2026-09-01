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


## `refactor(public-sites): migrate renderer identity`

diff --git a/schemas/public-sites-build-report.schema.json b/schemas/public-sites-build-report.schema.json
index 2983f23..4a0f26c 100644
--- a/schemas/public-sites-build-report.schema.json
+++ b/schemas/public-sites-build-report.schema.json
@@ -23,7 +23,12 @@
     "schema_version": { "const": 1 },
     "outcome": { "const": "succeeded" },
     "source_sha": { "$ref": "#/$defs/sha" },
-    "renderer_repository": { "const": "seungwoo7050/content-foundry-public-sites" },
+    "renderer_repository": {
+      "enum": [
+        "seungwoo7050/audience-foundry-publishing-static-renderer",
+        "seungwoo7050/content-foundry-public-sites"
+      ]
+    },
     "renderer_sha": { "const": "1717326cda8262d7f7f56d544b3a9d0215b71d51" },
     "target": { "const": "site-a" },
     "release_id": { "type": "string", "pattern": "^REL-[A-Z0-9-]+$" },
diff --git a/src/public-sites-build-report.js b/src/public-sites-build-report.js
index bc005a8..58b21a8 100644
--- a/src/public-sites-build-report.js
+++ b/src/public-sites-build-report.js
@@ -8,7 +8,10 @@ import Ajv2020 from "ajv/dist/2020.js";
 import { runGit } from "./git.js";
 import { containsSensitiveMaterial } from "./sensitive.js";
 
-export const PUBLIC_SITES_REPOSITORY = "seungwoo7050/content-foundry-public-sites";
+export const PUBLIC_SITES_REPOSITORY =
+  "seungwoo7050/audience-foundry-publishing-static-renderer";
+export const PUBLIC_SITES_LEGACY_REPOSITORY =
+  "seungwoo7050/content-foundry-public-sites";
 export const PUBLIC_SITES_SHA = "1717326cda8262d7f7f56d544b3a9d0215b71d51";
 
 export class PublicSitesBuildError extends Error {
diff --git a/test/public-sites-build-report.test.js b/test/public-sites-build-report.test.js
index ccb028c..8ae61e6 100644
--- a/test/public-sites-build-report.test.js
+++ b/test/public-sites-build-report.test.js
@@ -8,6 +8,8 @@ import { promisify } from "node:util";
 
 import {
   assertFrozenCheckoutState,
+  PUBLIC_SITES_LEGACY_REPOSITORY,
+  PUBLIC_SITES_REPOSITORY,
   PUBLIC_SITES_SHA,
   verifyPublicSitesBuild,
 } from "../src/public-sites-build-report.js";
@@ -53,5 +55,9 @@ test("report schema fixture remains non-secret", async () => {
     await readFile("schemas/public-sites-build-report.schema.json", "utf8"),
   );
   assert.equal(schema.properties.renderer_sha.const, PUBLIC_SITES_SHA);
+  assert.deepEqual(schema.properties.renderer_repository.enum, [
+    PUBLIC_SITES_REPOSITORY,
+    PUBLIC_SITES_LEGACY_REPOSITORY,
+  ]);
   assert.equal(schema.properties.outcome.const, "succeeded");
 });
diff --git a/test/schema.test.js b/test/schema.test.js
index 909ab0e..c24f499 100644
--- a/test/schema.test.js
+++ b/test/schema.test.js
@@ -117,7 +117,7 @@ test("Public Sites build reports pin the frozen renderer", async () => {
     schema_version: 1,
     outcome: "succeeded",
     source_sha: "a".repeat(40),
-    renderer_repository: "seungwoo7050/content-foundry-public-sites",
+    renderer_repository: "seungwoo7050/audience-foundry-publishing-static-renderer",
     renderer_sha: "1717326cda8262d7f7f56d544b3a9d0215b71d51",
     target: "site-a",
     release_id: "REL-EXAMPLE",
