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


## `feat(wordpress): isolate REST runtime credentials`

diff --git a/src/wordpress-rest-transport.js b/src/wordpress-rest-transport.js
new file mode 100644
index 0000000..40b32f0
--- /dev/null
+++ b/src/wordpress-rest-transport.js
@@ -0,0 +1,102 @@
+export class WordPressRuntimeError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "WordPressRuntimeError";
+    this.code = code;
+  }
+}
+
+export function resolveWordPressRuntime(site, environment = process.env) {
+  const mapping = site.wordpress;
+  const baseUrl = environment[mapping.base_url_env];
+  const username = environment[mapping.username_env];
+  const password = environment[mapping.password_env];
+  if (!baseUrl || !username || !password) {
+    throw new WordPressRuntimeError(
+      "WORDPRESS_RUNTIME_MISSING",
+      "WordPress runtime URL and credentials are required",
+    );
+  }
+  let parsed;
+  try {
+    parsed = new URL(baseUrl);
+  } catch {
+    throw new WordPressRuntimeError("WORDPRESS_URL_INVALID", "WordPress runtime URL is invalid");
+  }
+  const loopback = new Set(["127.0.0.1", "localhost", "::1"]).has(parsed.hostname);
+  if (parsed.protocol !== "https:" && !(parsed.protocol === "http:" && loopback)) {
+    throw new WordPressRuntimeError(
+      "WORDPRESS_URL_INSECURE",
+      "WordPress runtime URL must use HTTPS unless it is loopback",
+    );
+  }
+  return Object.freeze({
+    baseUrl: parsed.href.endsWith("/") ? parsed.href : `${parsed.href}/`,
+    username,
+    password,
+  });
+}
+
+function normalizePost(post) {
+  return {
+    id: post.id,
+    status: post.status,
+    slug: post.slug,
+    content: post.content?.raw ?? post.content?.rendered,
+  };
+}
+
+export function createWordPressRestTransport({ baseUrl, username, password, fetchImpl = fetch }) {
+  const authorization = `Basic ${Buffer.from(`${username}:${password}`, "utf8").toString("base64")}`;
+
+  async function request(method, relativePath, body) {
+    const response = await fetchImpl(new URL(relativePath, baseUrl), {
+      method,
+      headers: {
+        accept: "application/json",
+        authorization,
+        ...(body ? { "content-type": "application/json" } : {}),
+      },
+      ...(body ? { body: JSON.stringify(body) } : {}),
+    });
+    if (!response.ok) {
+      throw new WordPressRuntimeError(
+        "WORDPRESS_REQUEST_FAILED",
+        `WordPress request failed with HTTP ${response.status}`,
+      );
+    }
+    try {
+      return await response.json();
+    } catch {
+      throw new WordPressRuntimeError(
+        "WORDPRESS_RESPONSE_INVALID",
+        "WordPress response was not valid JSON",
+      );
+    }
+  }
+
+  return Object.freeze({
+    async findDraftsBySlug(slug) {
+      const query = new URLSearchParams({
+        context: "edit",
+        per_page: "100",
+        slug,
+        status: "draft",
+      });
+      const posts = await request("GET", `wp-json/wp/v2/posts?${query}`);
+      if (!Array.isArray(posts)) {
+        throw new WordPressRuntimeError("WORDPRESS_RESPONSE_INVALID", "WordPress list was invalid");
+      }
+      return posts.map(normalizePost);
+    },
+    async getPost(id) {
+      return normalizePost(await request("GET", `wp-json/wp/v2/posts/${id}?context=edit`));
+    },
+    async createDraft(payload) {
+      return normalizePost(await request("POST", "wp-json/wp/v2/posts", payload));
+    },
+    async updateDraft(id, payload) {
+      return normalizePost(await request("POST", `wp-json/wp/v2/posts/${id}`, payload));
+    },
+  });
+}
diff --git a/test/wordpress-rest-transport.test.js b/test/wordpress-rest-transport.test.js
new file mode 100644
index 0000000..1815a11
--- /dev/null
+++ b/test/wordpress-rest-transport.test.js
@@ -0,0 +1,91 @@
+import assert from "node:assert/strict";
+import test from "node:test";
+
+import {
+  createWordPressRestTransport,
+  resolveWordPressRuntime,
+} from "../src/wordpress-rest-transport.js";
+
+const site = {
+  wordpress: {
+    base_url_env: "PUBLISHER_WP_BASE_URL",
+    username_env: "PUBLISHER_WP_USERNAME",
+    password_env: "PUBLISHER_WP_PASSWORD",
+  },
+};
+
+test("runtime mapping reads values only from named environment entries", () => {
+  const environment = {
+    PUBLISHER_WP_BASE_URL: "http://127.0.0.1:8888",
+    PUBLISHER_WP_USERNAME: ["local", "user"].join("-"),
+    PUBLISHER_WP_PASSWORD: "x".repeat(24),
+  };
+  const runtime = resolveWordPressRuntime(site, environment);
+
+  assert.equal(runtime.baseUrl, "http://127.0.0.1:8888/");
+  assert.equal(runtime.username, environment.PUBLISHER_WP_USERNAME);
+  assert.equal(runtime.password, environment.PUBLISHER_WP_PASSWORD);
+  assert.throws(() => resolveWordPressRuntime(site, {}), {
+    code: "WORDPRESS_RUNTIME_MISSING",
+  });
+  assert.throws(
+    () => resolveWordPressRuntime(site, { ...environment, PUBLISHER_WP_BASE_URL: "http://example.com" }),
+    { code: "WORDPRESS_URL_INSECURE" },
+  );
+});
+
+test("REST transport creates and updates drafts without returning credentials", async () => {
+  const calls = [];
+  const fetchImpl = async (url, options) => {
+    calls.push({ url: url.href, options });
+    return {
+      ok: true,
+      status: 200,
+      async json() {
+        return {
+          id: 42,
+          status: "draft",
+          slug: "wordpress-loop",
+          content: { raw: JSON.parse(options.body).content },
+        };
+      },
+    };
+  };
+  const transport = createWordPressRestTransport({
+    baseUrl: "https://wordpress.example/",
+    username: ["runtime", "user"].join("-"),
+    password: "y".repeat(24),
+    fetchImpl,
+  });
+  const payload = {
+    title: "Draft",
+    slug: "wordpress-loop",
+    content: "<p>Draft</p>",
+    status: "draft",
+  };
+
+  const created = await transport.createDraft(payload);
+  const updated = await transport.updateDraft(42, payload);
+  assert.deepEqual(created, updated);
+  assert.equal(created.status, "draft");
+  assert.equal(calls.length, 2);
+  assert.equal(calls[0].options.headers.authorization.startsWith("Basic "), true);
+  assert.equal(calls[0].options.body, JSON.stringify(payload));
+  assert.doesNotMatch(JSON.stringify(created), /runtime-user|yyyy/);
+});
+
+test("REST failures expose status but never response content", async () => {
+  const transport = createWordPressRestTransport({
+    baseUrl: "https://wordpress.example/",
+    username: "u",
+    password: "z".repeat(24),
+    fetchImpl: async () => ({ ok: false, status: 401 }),
+  });
+
+  await assert.rejects(() => transport.getPost(1), (error) => {
+    assert.equal(error.code, "WORDPRESS_REQUEST_FAILED");
+    assert.match(error.message, /401/);
+    assert.doesNotMatch(error.message, /zzzz/);
+    return true;
+  });
+});


## `feat(wordpress): add credential-free wp-env transport`

diff --git a/src/wp-env-cli-transport.js b/src/wp-env-cli-transport.js
new file mode 100644
index 0000000..237115d
--- /dev/null
+++ b/src/wp-env-cli-transport.js
@@ -0,0 +1,148 @@
+import { execFile } from "node:child_process";
+import path from "node:path";
+import { promisify } from "node:util";
+
+const execFileAsync = promisify(execFile);
+const allowedEnvironment = [
+  "DOCKER_CONTEXT",
+  "DOCKER_HOST",
+  "HOME",
+  "LANG",
+  "LC_ALL",
+  "LOGNAME",
+  "PATH",
+  "SHELL",
+  "TEMP",
+  "TMP",
+  "TMPDIR",
+  "USER",
+];
+
+export class WpEnvCliError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "WpEnvCliError";
+    this.code = code;
+  }
+}
+
+function processEnvironment() {
+  const environment = {};
+  for (const name of allowedEnvironment) {
+    if (process.env[name] !== undefined) environment[name] = process.env[name];
+  }
+  return environment;
+}
+
+async function defaultRunner(executable, args, options) {
+  return execFileAsync(executable, args, {
+    ...options,
+    encoding: "utf8",
+    maxBuffer: 8 * 1024 * 1024,
+  });
+}
+
+function parseLastJson(stdout) {
+  for (const line of stdout.trim().split("\n").reverse()) {
+    try {
+      return JSON.parse(line);
+    } catch {
+      // wp-env may print lifecycle messages before the WP-CLI JSON payload.
+    }
+  }
+  throw new WpEnvCliError("WP_ENV_RESPONSE_INVALID", "wp-env returned no valid JSON payload");
+}
+
+function parseLastInteger(stdout) {
+  for (const line of stdout.trim().split("\n").reverse()) {
+    if (/^[1-9]\d*$/.test(line.trim())) return Number.parseInt(line.trim(), 10);
+  }
+  throw new WpEnvCliError("WP_ENV_RESPONSE_INVALID", "wp-env did not return a post ID");
+}
+
+function normalizePost(post) {
+  const id = Number.parseInt(String(post.ID), 10);
+  if (!Number.isInteger(id) || id < 1) {
+    throw new WpEnvCliError("WP_ENV_RESPONSE_INVALID", "wp-env returned an invalid post ID");
+  }
+  return {
+    id,
+    status: post.post_status,
+    slug: post.post_name,
+    content: post.post_content,
+  };
+}
+
+export function createWpEnvCliTransport({ publisherRoot, runCommand = defaultRunner }) {
+  const executable = path.join(publisherRoot, "node_modules/.bin/wp-env");
+
+  async function runWp(argumentsList) {
+    try {
+      const { stdout } = await runCommand(
+        executable,
+        ["run", "cli", "wp", ...argumentsList],
+        { cwd: publisherRoot, env: processEnvironment() },
+      );
+      return stdout;
+    } catch {
+      throw new WpEnvCliError("WP_ENV_COMMAND_FAILED", "Local WordPress command failed");
+    }
+  }
+
+  async function getPost(id) {
+    const stdout = await runWp([
+      "post",
+      "get",
+      String(id),
+      "--fields=ID,post_status,post_name,post_content",
+      "--format=json",
+    ]);
+    return normalizePost(parseLastJson(stdout));
+  }
+
+  return Object.freeze({
+    async findDraftsBySlug(slug) {
+      const stdout = await runWp([
+        "post",
+        "list",
+        "--post_type=post",
+        "--post_status=draft",
+        `--name=${slug}`,
+        "--fields=ID",
+        "--format=json",
+      ]);
+      const rows = parseLastJson(stdout);
+      if (!Array.isArray(rows)) {
+        throw new WpEnvCliError("WP_ENV_RESPONSE_INVALID", "wp-env returned an invalid post list");
+      }
+      return Promise.all(rows.map(({ ID }) => getPost(ID)));
+    },
+    getPost,
+    async createDraft(payload) {
+      const stdout = await runWp([
+        "post",
+        "create",
+        "--post_type=post",
+        "--post_status=draft",
+        `--post_title=${payload.title}`,
+        `--post_name=${payload.slug}`,
+        `--post_content=${payload.content}`,
+        "--porcelain",
+      ]);
+      const id = parseLastInteger(stdout);
+      return getPost(id);
+    },
+    async updateDraft(id, payload) {
+      await runWp([
+        "post",
+        "update",
+        String(id),
+        "--post_status=draft",
+        `--post_title=${payload.title}`,
+        `--post_name=${payload.slug}`,
+        `--post_content=${payload.content}`,
+      ]);
+      return getPost(id);
+    },
+  });
+}
diff --git a/test/wp-env-cli-transport.test.js b/test/wp-env-cli-transport.test.js
new file mode 100644
index 0000000..9905f3d
--- /dev/null
+++ b/test/wp-env-cli-transport.test.js
@@ -0,0 +1,70 @@
+import assert from "node:assert/strict";
+import test from "node:test";
+
+import { createWpEnvCliTransport } from "../src/wp-env-cli-transport.js";
+
+test("wp-env transport creates drafts and reads the returned remote ID", async () => {
+  const calls = [];
+  const runCommand = async (executable, args, options) => {
+    calls.push({ executable, args, options });
+    if (args.includes("create")) return { stdout: "15\n", stderr: "" };
+    if (args.includes("get")) {
+      return {
+        stdout: 'lifecycle\n{"ID":15,"post_status":"draft","post_name":"wordpress-loop","post_content":"marker"}\n',
+        stderr: "",
+      };
+    }
+    return { stdout: "[]\n", stderr: "" };
+  };
+  const transport = createWpEnvCliTransport({ publisherRoot: "/publisher", runCommand });
+  const created = await transport.createDraft({
+    title: "Draft",
+    slug: "wordpress-loop",
+    content: "marker",
+    status: "draft",
+  });
+
+  assert.equal(created.id, 15);
+  assert.equal(created.status, "draft");
+  assert.match(calls[0].executable, /node_modules\/\.bin\/wp-env$/);
+  assert.equal(calls[0].args.includes("--post_status=draft"), true);
+  assert.equal(Object.keys(calls[0].options.env).some((name) => /TOKEN|PASSWORD|SECRET/.test(name)), false);
+});
+
+test("wp-env transport lists and updates the same draft", async () => {
+  const runCommand = async (_executable, args) => {
+    if (args.includes("list")) return { stdout: '[{"ID":21}]\n', stderr: "" };
+    if (args.includes("get")) {
+      return {
+        stdout: '{"ID":21,"post_status":"draft","post_name":"wordpress-loop","post_content":"marker"}\n',
+        stderr: "",
+      };
+    }
+    return { stdout: "Success\n", stderr: "" };
+  };
+  const transport = createWpEnvCliTransport({ publisherRoot: "/publisher", runCommand });
+
+  const found = await transport.findDraftsBySlug("wordpress-loop");
+  const updated = await transport.updateDraft(21, {
+    title: "Updated",
+    slug: "wordpress-loop",
+    content: "marker",
+    status: "draft",
+  });
+  assert.equal(found[0].id, 21);
+  assert.equal(updated.id, 21);
+});
+
+test("wp-env command failures are redacted", async () => {
+  const transport = createWpEnvCliTransport({
+    publisherRoot: "/publisher",
+    runCommand: async () => {
+      throw new Error("container output should remain private");
+    },
+  });
+  await assert.rejects(() => transport.getPost(1), (error) => {
+    assert.equal(error.code, "WP_ENV_COMMAND_FAILED");
+    assert.doesNotMatch(error.message, /container output/);
+    return true;
+  });
+});


## `fix(wordpress): start one wp-env runtime`

diff --git a/.wp-env.json b/.wp-env.json
index e1758b8..33f0d34 100644
--- a/.wp-env.json
+++ b/.wp-env.json
@@ -3,7 +3,7 @@
   "core": "https://downloads.wordpress.org/release/wordpress-7.1.zip",
   "phpVersion": "8.3",
   "port": 8888,
-  "testsPort": 8889,
+  "testsEnvironment": false,
   "config": {
     "WP_ENVIRONMENT_TYPE": "local"
   }
diff --git a/test/wp-env.test.js b/test/wp-env.test.js
index 22e110a..f45f731 100644
--- a/test/wp-env.test.js
+++ b/test/wp-env.test.js
@@ -10,6 +10,7 @@ test("wp-env pins a provider-free local WordPress runtime", async () => {
   assert.equal(config.core, "https://downloads.wordpress.org/release/wordpress-7.1.zip");
   assert.equal(config.phpVersion, "8.3");
   assert.equal(config.port, 8888);
+  assert.equal(config.testsEnvironment, false);
   assert.equal(config.config.WP_ENVIRONMENT_TYPE, "local");
   assert.equal(JSON.stringify(config).includes("password"), false);
 });


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


