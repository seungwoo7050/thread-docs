# WordPress 초안 멱등성과 식별자 보존

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


## `feat(wordpress): upsert idempotent drafts`

diff --git a/src/adapters/wordpress.js b/src/adapters/wordpress.js
new file mode 100644
index 0000000..79ae718
--- /dev/null
+++ b/src/adapters/wordpress.js
@@ -0,0 +1,116 @@
+import { validateArticleSource, validateSiteSource } from "../content.js";
+import { readFileAtCommit } from "../git.js";
+import { projectMarkdown } from "../public-sites-projection.js";
+
+export class WordPressAdapterError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "WordPressAdapterError";
+    this.code = code;
+  }
+}
+
+function escapeHtml(value) {
+  return value
+    .replaceAll("&", "&amp;")
+    .replaceAll("<", "&lt;")
+    .replaceAll(">", "&gt;")
+    .replaceAll('"', "&quot;")
+    .replaceAll("'", "&#39;");
+}
+
+export function renderWordPressHtml(article, idempotencyKey) {
+  if (!/^[0-9a-f]{64}$/.test(idempotencyKey)) {
+    throw new WordPressAdapterError("INVALID_IDEMPOTENCY_KEY", "WordPress idempotency key is invalid");
+  }
+  const marker = `<!-- content-foundry article=${article.frontMatter.id} key=${idempotencyKey} -->`;
+  const { blocks } = projectMarkdown(article.body);
+  const html = blocks.map((block) => {
+    if (block.type === "heading") {
+      return `<h${block.level} id="${escapeHtml(block.id)}">${escapeHtml(block.text)}</h${block.level}>`;
+    }
+    if (block.type === "list") {
+      const tag = block.ordered ? "ol" : "ul";
+      return `<${tag}>${block.items.map((item) => `<li>${escapeHtml(item)}</li>`).join("")}</${tag}>`;
+    }
+    return `<p>${escapeHtml(block.markdown).replaceAll("\n", "<br>\n")}</p>`;
+  });
+  return `${marker}\n${html.join("\n")}\n`;
+}
+
+function assertDraft(post, articleId) {
+  if (!Number.isInteger(post?.id) || post.id < 1 || post.status !== "draft") {
+    throw new WordPressAdapterError("REMOTE_NOT_DRAFT", "WordPress remote identity is not a draft");
+  }
+  if (post.content && !post.content.includes(`content-foundry article=${articleId} `)) {
+    throw new WordPressAdapterError("REMOTE_IDENTITY_MISMATCH", "WordPress draft belongs to another article");
+  }
+}
+
+export async function upsertWordPressDraft({
+  article,
+  idempotencyKey,
+  transport,
+  priorRemoteId,
+}) {
+  const content = renderWordPressHtml(article, idempotencyKey);
+  const payload = {
+    title: article.frontMatter.title,
+    slug: article.frontMatter.slug,
+    content,
+    status: "draft",
+  };
+  let existing;
+  if (priorRemoteId !== undefined) {
+    existing = await transport.getPost(priorRemoteId);
+    assertDraft(existing, article.frontMatter.id);
+  } else {
+    const candidates = await transport.findDraftsBySlug(article.frontMatter.slug);
+    const marker = `content-foundry article=${article.frontMatter.id} key=${idempotencyKey}`;
+    const matches = candidates.filter((candidate) => candidate.content?.includes(marker));
+    if (matches.length > 1) {
+      throw new WordPressAdapterError("DUPLICATE_REMOTE_DRAFTS", "Multiple WordPress drafts match one publication");
+    }
+    [existing] = matches;
+    if (existing) assertDraft(existing, article.frontMatter.id);
+  }
+
+  const remote = existing
+    ? await transport.updateDraft(existing.id, payload)
+    : await transport.createDraft(payload);
+  assertDraft(remote, article.frontMatter.id);
+  if (existing && remote.id !== existing.id) {
+    throw new WordPressAdapterError("REMOTE_ID_CHANGED", "WordPress update changed the remote identity");
+  }
+  return Object.freeze({
+    kind: "wordpress",
+    remote_post_id: remote.id,
+    remote_status: "draft",
+  });
+}
+
+export async function publishToWordPress({
+  publisherRoot,
+  articlePath,
+  sourceSha,
+  idempotencyKey,
+  transport,
+  priorRemoteId,
+}) {
+  const article = validateArticleSource(
+    await readFileAtCommit(publisherRoot, sourceSha, articlePath),
+    articlePath,
+  );
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  const site = validateSiteSource(
+    await readFileAtCommit(publisherRoot, sourceSha, sitePath),
+    sitePath,
+  );
+  if (site.id !== article.frontMatter.site || site.engine !== "wordpress") {
+    throw new WordPressAdapterError("WORDPRESS_SITE_MISMATCH", "Article does not select a WordPress site");
+  }
+  if (site.wordpress.default_status !== "draft") {
+    throw new WordPressAdapterError("UNSAFE_REMOTE_STATUS", "WordPress site must default to draft");
+  }
+  return upsertWordPressDraft({ article, idempotencyKey, transport, priorRemoteId });
+}
diff --git a/test/wordpress-adapter.test.js b/test/wordpress-adapter.test.js
new file mode 100644
index 0000000..94f6c07
--- /dev/null
+++ b/test/wordpress-adapter.test.js
@@ -0,0 +1,105 @@
+import assert from "node:assert/strict";
+import { readFile } from "node:fs/promises";
+import test from "node:test";
+
+import { validateArticleSource } from "../src/content.js";
+import {
+  renderWordPressHtml,
+  upsertWordPressDraft,
+} from "../src/adapters/wordpress.js";
+
+const articlePath = "content/articles/wordpress-loop.md";
+
+async function articleFixture() {
+  return validateArticleSource(await readFile(articlePath, "utf8"), articlePath);
+}
+
+function memoryTransport() {
+  const posts = [];
+  return {
+    posts,
+    async findDraftsBySlug(slug) {
+      return posts.filter((post) => post.slug === slug && post.status === "draft");
+    },
+    async getPost(id) {
+      return posts.find((post) => post.id === id);
+    },
+    async createDraft(payload) {
+      const post = { id: posts.length + 1, ...payload };
+      posts.push(post);
+      return post;
+    },
+    async updateDraft(id, payload) {
+      const index = posts.findIndex((post) => post.id === id);
+      posts[index] = { id, ...payload };
+      return posts[index];
+    },
+  };
+}
+
+test("create and receipt-loss retry preserve one remote draft ID", async () => {
+  const article = await articleFixture();
+  const transport = memoryTransport();
+  const input = { article, idempotencyKey: "a".repeat(64), transport };
+
+  const created = await upsertWordPressDraft(input);
+  const retried = await upsertWordPressDraft(input);
+
+  assert.equal(created.remote_post_id, 1);
+  assert.equal(retried.remote_post_id, 1);
+  assert.equal(transport.posts.length, 1);
+  assert.equal(transport.posts[0].status, "draft");
+  assert.match(transport.posts[0].content, /content-foundry article=wordpress-loop/);
+});
+
+test("new approved source updates the prior remote identity", async () => {
+  const article = await articleFixture();
+  const transport = memoryTransport();
+  const first = await upsertWordPressDraft({
+    article,
+    idempotencyKey: "b".repeat(64),
+    transport,
+  });
+  const updated = await upsertWordPressDraft({
+    article,
+    idempotencyKey: "c".repeat(64),
+    transport,
+    priorRemoteId: first.remote_post_id,
+  });
+
+  assert.equal(updated.remote_post_id, first.remote_post_id);
+  assert.equal(transport.posts.length, 1);
+  assert.match(transport.posts[0].content, new RegExp("c{64}"));
+});
+
+test("adapter cannot update a non-draft or another article", async () => {
+  const article = await articleFixture();
+  const transport = memoryTransport();
+  transport.posts.push({
+    id: 7,
+    slug: "wordpress-loop",
+    status: "publish",
+    content: "<!-- content-foundry article=wordpress-loop key=old -->",
+  });
+  await assert.rejects(
+    () =>
+      upsertWordPressDraft({
+        article,
+        idempotencyKey: "d".repeat(64),
+        transport,
+        priorRemoteId: 7,
+      }),
+    { code: "REMOTE_NOT_DRAFT" },
+  );
+});
+
+test("WordPress HTML escapes canonical content and pins its marker", async () => {
+  const article = await articleFixture();
+  const html = renderWordPressHtml(
+    { ...article, body: `${article.body}\n\n<script>unsafe()</script>` },
+    "e".repeat(64),
+  );
+  assert.doesNotMatch(html, /<script>/);
+  assert.match(html, /&lt;script&gt;unsafe\(\)&lt;\/script&gt;/);
+  assert.match(html, new RegExp(`key=${"e".repeat(64)}`));
+});


