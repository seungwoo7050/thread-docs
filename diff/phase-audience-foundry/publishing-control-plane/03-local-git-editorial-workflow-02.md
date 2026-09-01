## `feat(editor): configure Decap local workflow`

diff --git a/admin/config.yml b/admin/config.yml
new file mode 100644
index 0000000..483fe56
--- /dev/null
+++ b/admin/config.yml
@@ -0,0 +1,46 @@
+local_backend: true
+backend:
+  name: proxy
+  proxy_url: http://127.0.0.1:8081/api/v1
+  branch: main
+publish_mode: editorial_workflow
+media_folder: content/media
+public_folder: /media
+collections:
+  - name: articles
+    label: Articles
+    label_singular: Article
+    folder: content/articles
+    create: true
+    extension: md
+    format: yaml-frontmatter
+    slug: "{{fields.slug}}"
+    identifier_field: id
+    summary: "{{title}} — {{site}}"
+    fields:
+      - { name: schema_version, label: Schema version, widget: hidden, default: 1 }
+      - { name: id, label: Stable ID, widget: string, pattern: ['^[a-z0-9]+(?:-[a-z0-9]+)*$', 'lowercase kebab-case'] }
+      - { name: site, label: Site, widget: relation, collection: sites, search_fields: [name, id], value_field: id, display_fields: [name, id] }
+      - { name: slug, label: Slug, widget: string, pattern: ['^[a-z0-9]+(?:-[a-z0-9]+)*$', 'lowercase kebab-case'] }
+      - { name: title, label: Title, widget: string }
+      - { name: summary, label: Summary, widget: text }
+      - { name: author, label: Author, widget: string }
+      - { name: created_at, label: Created at, widget: datetime, date_format: YYYY-MM-DD, time_format: HH:mm:ss, format: 'YYYY-MM-DDTHH:mm:ss[Z]' }
+      - { name: tags, label: Tags, widget: list, required: false }
+      - { name: body, label: Body, widget: markdown }
+  - name: sites
+    label: Sites
+    files:
+      - name: public-demo
+        label: Public Sites viability target
+        file: sites/public-demo.yml
+        fields:
+          - { name: schema_version, label: Schema version, widget: hidden, default: 1 }
+          - { name: id, label: Stable ID, widget: hidden }
+          - { name: name, label: Name, widget: string }
+          - { name: engine, label: Engine, widget: hidden }
+          - name: public_sites
+            label: Public Sites
+            widget: object
+            fields:
+              - { name: target, label: Target, widget: string }
diff --git a/admin/index.html b/admin/index.html
new file mode 100644
index 0000000..0285989
--- /dev/null
+++ b/admin/index.html
@@ -0,0 +1,12 @@
+<!doctype html>
+<html lang="en">
+  <head>
+    <meta charset="utf-8" />
+    <meta name="viewport" content="width=device-width, initial-scale=1" />
+    <meta name="robots" content="noindex" />
+    <title>Content Foundry Publisher</title>
+  </head>
+  <body>
+    <script src="/vendor/decap-cms-app.js"></script>
+  </body>
+</html>
diff --git a/package.json b/package.json
index 03c6f6d..885eb7d 100644
--- a/package.json
+++ b/package.json
@@ -8,6 +8,8 @@
   },
   "packageManager": "npm@11.4.2",
   "scripts": {
+    "cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
+    "cms:web": "node src/admin-server.js",
     "test": "node --test"
   },
   "dependencies": {
diff --git a/src/admin-server.js b/src/admin-server.js
new file mode 100644
index 0000000..7fe4498
--- /dev/null
+++ b/src/admin-server.js
@@ -0,0 +1,58 @@
+import { createReadStream } from "node:fs";
+import { stat } from "node:fs/promises";
+import http from "node:http";
+import path from "node:path";
+import { fileURLToPath } from "node:url";
+
+const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), "..");
+const assets = new Map([
+  ["/", ["admin/index.html", "text/html; charset=utf-8"]],
+  ["/index.html", ["admin/index.html", "text/html; charset=utf-8"]],
+  ["/config.yml", ["admin/config.yml", "text/yaml; charset=utf-8"]],
+  [
+    "/vendor/decap-cms-app.js",
+    ["node_modules/decap-cms-app/dist/decap-cms-app.js", "text/javascript; charset=utf-8"],
+  ],
+]);
+
+export function resolveAdminAsset(method, requestUrl) {
+  if (method !== "GET") return undefined;
+  const pathname = new URL(requestUrl, "http://127.0.0.1").pathname;
+  return assets.get(pathname);
+}
+
+export function createAdminServer() {
+  return http.createServer(async (request, response) => {
+    const asset = resolveAdminAsset(request.method, request.url);
+    if (!asset) {
+      response.writeHead(404, { "content-type": "text/plain; charset=utf-8" });
+      response.end("Not found\n");
+      return;
+    }
+    try {
+      const [relativePath, contentType] = asset;
+      const absolutePath = path.join(root, relativePath);
+      const metadata = await stat(absolutePath);
+      response.writeHead(200, {
+        "cache-control": "no-store",
+        "content-length": metadata.size,
+        "content-type": contentType,
+        "x-content-type-options": "nosniff",
+      });
+      createReadStream(absolutePath).pipe(response);
+    } catch {
+      response.writeHead(500, { "content-type": "text/plain; charset=utf-8" });
+      response.end("Local editor asset unavailable\n");
+    }
+  });
+}
+
+if (process.argv[1] && fileURLToPath(import.meta.url) === path.resolve(process.argv[1])) {
+  const port = Number.parseInt(process.env.PUBLISHER_CMS_PORT || "8080", 10);
+  if (!Number.isInteger(port) || port < 1 || port > 65535) {
+    throw new Error("PUBLISHER_CMS_PORT must be a valid TCP port");
+  }
+  createAdminServer().listen(port, "127.0.0.1", () => {
+    process.stdout.write(`Publisher editor: http://127.0.0.1:${port}\n`);
+  });
+}
diff --git a/test/decap.test.js b/test/decap.test.js
new file mode 100644
index 0000000..03418c2
--- /dev/null
+++ b/test/decap.test.js
@@ -0,0 +1,53 @@
+import assert from "node:assert/strict";
+import { readFile } from "node:fs/promises";
+import test from "node:test";
+
+import { parse as parseYaml } from "yaml";
+
+import { resolveAdminAsset } from "../src/admin-server.js";
+
+test("Decap maps editorial workflow to local Git and canonical fields", async () => {
+  const config = parseYaml(await readFile("admin/config.yml", "utf8"), {
+    uniqueKeys: true,
+  });
+  const articles = config.collections.find(({ name }) => name === "articles");
+  const fields = articles.fields.map(({ name }) => name);
+
+  assert.equal(config.local_backend, true);
+  assert.equal(config.backend.name, "proxy");
+  assert.equal(config.backend.proxy_url, "http://127.0.0.1:8081/api/v1");
+  assert.equal(config.backend.branch, "main");
+  assert.equal(config.publish_mode, "editorial_workflow");
+  assert.equal(articles.folder, "content/articles");
+  assert.deepEqual(fields, [
+    "schema_version",
+    "id",
+    "site",
+    "slug",
+    "title",
+    "summary",
+    "author",
+    "created_at",
+    "tags",
+    "body",
+  ]);
+  assert.equal(fields.includes("engine"), false);
+  assert.equal(fields.includes("approval"), false);
+});
+
+test("local editor exposes only reviewed assets", async () => {
+  assert.deepEqual(resolveAdminAsset("GET", "/"), [
+    "admin/index.html",
+    "text/html; charset=utf-8",
+  ]);
+  assert.deepEqual(resolveAdminAsset("GET", "/config.yml"), [
+    "admin/config.yml",
+    "text/yaml; charset=utf-8",
+  ]);
+  assert.equal(resolveAdminAsset("GET", "/README.md"), undefined);
+  assert.equal(resolveAdminAsset("GET", "/../README.md"), undefined);
+  assert.equal(resolveAdminAsset("POST", "/config.yml"), undefined);
+
+  const index = await readFile("admin/index.html", "utf8");
+  assert.match(index, /\/vendor\/decap-cms-app\.js/);
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
