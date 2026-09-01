# 정적 릴리스 투영과 결정적 컴파일

## `fix(site): select frozen site-a target`

diff --git a/sites/public-demo.yml b/sites/public-demo.yml
index f5e966f..dd3f70e 100644
--- a/sites/public-demo.yml
+++ b/sites/public-demo.yml
@@ -3,4 +3,4 @@ id: public-demo
 name: Public Sites viability target
 engine: public_sites
 public_sites:
-  target: publisher-viability
+  target: site-a


## `feat(public-sites): project canonical article`

diff --git a/src/public-sites-projection.js b/src/public-sites-projection.js
new file mode 100644
index 0000000..08b73f7
--- /dev/null
+++ b/src/public-sites-projection.js
@@ -0,0 +1,189 @@
+import { createHash } from "node:crypto";
+
+export class PublicSitesProjectionError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicSitesProjectionError";
+    this.code = code;
+  }
+}
+
+function stablePublicId(prefix, value) {
+  return `${prefix}-${createHash("sha256").update(value).digest("hex").slice(0, 12).toUpperCase()}`;
+}
+
+function headingId(value) {
+  const id = value
+    .normalize("NFKD")
+    .toLowerCase()
+    .replace(/[^a-z0-9]+/g, "-")
+    .replace(/^-|-$/g, "");
+  if (!id) throw new PublicSitesProjectionError("UNSUPPORTED_HEADING", "Heading needs an ASCII identifier");
+  return id;
+}
+
+export function projectMarkdown(body) {
+  const blocks = [];
+  const toc = [];
+  for (const section of body.trim().split(/\n\s*\n/)) {
+    const lines = section.split("\n");
+    const heading = lines.length === 1 ? lines[0].match(/^(#{1,6})\s+(.+)$/) : null;
+    if (heading) {
+      const level = heading[1].length;
+      if (level === 1) continue;
+      const text = heading[2].trim();
+      const id = headingId(text);
+      if (toc.some((entry) => entry.id === id)) {
+        throw new PublicSitesProjectionError("DUPLICATE_HEADING", "Heading identifiers must be unique");
+      }
+      const entry = { id, text, level };
+      blocks.push({ type: "heading", ...entry });
+      toc.push(entry);
+      continue;
+    }
+    if (lines.every((line) => /^[-*]\s+\S/.test(line))) {
+      blocks.push({
+        type: "list",
+        ordered: false,
+        items: lines.map((line) => line.replace(/^[-*]\s+/, "")),
+      });
+      continue;
+    }
+    if (lines.every((line) => /^\d+\.\s+\S/.test(line))) {
+      blocks.push({
+        type: "list",
+        ordered: true,
+        items: lines.map((line) => line.replace(/^\d+\.\s+/, "")),
+      });
+      continue;
+    }
+    blocks.push({ type: "paragraph", markdown: section });
+  }
+  if (blocks.length === 0) {
+    throw new PublicSitesProjectionError("EMPTY_PROJECTION", "Article produced no public content blocks");
+  }
+  return { blocks, toc };
+}
+
+export function projectPublicSitesRecords({ article, site, sourceSha }) {
+  if (site.engine !== "public_sites" || site.public_sites?.target !== "site-a") {
+    throw new PublicSitesProjectionError("UNSUPPORTED_TARGET", "Site does not select the frozen site-a target");
+  }
+  if (!/^[0-9a-f]{40}$/.test(sourceSha)) {
+    throw new PublicSitesProjectionError("INVALID_SOURCE_SHA", "Projection requires a full source SHA");
+  }
+  const frontMatter = article.frontMatter;
+  const publicArticleId = stablePublicId("ART", frontMatter.id);
+  const { blocks, toc } = projectMarkdown(article.body);
+  const articlePath = `/article/${frontMatter.slug}`;
+  const category = {
+    id: "articles",
+    slug: "articles",
+    label: "Articles",
+    description: "Published articles",
+  };
+  const tags = (frontMatter.tags || []).map((tag) => ({
+    id: tag,
+    slug: tag,
+    label: tag,
+    description: "",
+  }));
+  const author = { displayName: frontMatter.author, profileId: "publisher-author" };
+  const theme = "minimal-knowledge-base";
+  const skin = "calm-blue";
+
+  return Object.freeze({
+    articleId: publicArticleId,
+    records: Object.freeze({
+      release: {
+        contractVersion: "4.0.0",
+        releaseId: `REL-${sourceSha.slice(0, 12).toUpperCase()}`,
+        siteId: site.id,
+        createdAt: frontMatter.created_at,
+        contentRevision: Number.parseInt(sourceSha.slice(0, 8), 16),
+        siteConfigRevision: Number.parseInt(sourceSha.slice(8, 16), 16),
+        articleCount: 1,
+        pageCount: 1,
+        defaultTheme: theme,
+        defaultSkin: skin,
+        bundleChecksum: `sha256:${"0".repeat(64)}`,
+      },
+      site: {
+        id: site.id,
+        origin: "https://publisher.invalid",
+        locale: "en-US",
+        timeZone: "Asia/Seoul",
+        name: site.name,
+        shortName: site.name,
+        description: "A provider-free Content Foundry Publisher preview.",
+        defaultTheme: theme,
+        defaultSkin: skin,
+        author,
+        analytics: { provider: "disabled", publicMeasurementId: null },
+        ads: { provider: "disabled", enabled: false, publicClientId: null },
+        search: { enabled: true },
+        featureFlags: {},
+      },
+      taxonomy: { categories: [category], tags },
+      navigation: {
+        items: [
+          { id: "home", label: "Home", path: "/", children: [] },
+          { id: "articles", label: "Articles", path: "/category/articles", children: [] },
+        ],
+      },
+      presentation: {
+        home: {
+          featuredArticleIds: [publicArticleId],
+          currentArticleIds: [],
+          evergreenArticleIds: [],
+        },
+        categoryHighlights: [{ categoryId: "articles", articleIds: [publicArticleId] }],
+        brand: { logoMediaId: null, faviconMediaId: null, socialImageMediaId: null },
+      },
+      article: {
+        id: publicArticleId,
+        revision: 1,
+        slug: frontMatter.slug,
+        title: frontMatter.title,
+        summary: frontMatter.summary,
+        status: "published",
+        categoryId: "articles",
+        tagIds: frontMatter.tags || [],
+        author,
+        publishedAt: frontMatter.created_at,
+        updatedAt: frontMatter.created_at,
+        content: blocks,
+        toc,
+        faq: [],
+        sourceDisclosures: [],
+        relatedArticleIds: [],
+        heroMediaId: null,
+        seo: {
+          title: frontMatter.title,
+          description: frontMatter.summary,
+          canonicalPath: articlePath,
+          index: false,
+          follow: true,
+        },
+        advertising: { enabled: false },
+        updateTriggers: [],
+      },
+      about: {
+        id: "about",
+        path: "/about",
+        title: "About",
+        summary: "About this local provider-free preview.",
+        content: [{ type: "paragraph", markdown: "This preview was compiled from reviewed canonical Markdown." }],
+        seo: {
+          title: `About ${site.name}`,
+          description: "About this local provider-free preview.",
+          canonicalPath: "/about",
+          index: false,
+          follow: true,
+        },
+      },
+      mediaManifest: { items: [] },
+      redirects: { items: [] },
+    }),
+  });
+}
diff --git a/test/public-sites-projection.test.js b/test/public-sites-projection.test.js
new file mode 100644
index 0000000..dddcad6
--- /dev/null
+++ b/test/public-sites-projection.test.js
@@ -0,0 +1,58 @@
+import assert from "node:assert/strict";
+import test from "node:test";
+
+import { loadArticleWithSite } from "../src/content.js";
+import {
+  projectMarkdown,
+  projectPublicSitesRecords,
+} from "../src/public-sites-projection.js";
+
+test("projects the real Publisher article without copying renderer fixtures", async () => {
+  const loaded = await loadArticleWithSite(
+    process.cwd(),
+    "content/articles/publisher-loop.md",
+  );
+  const sourceSha = "a".repeat(40);
+  const projected = projectPublicSitesRecords({ ...loaded, sourceSha });
+  const { records } = projected;
+
+  assert.match(projected.articleId, /^ART-[A-F0-9]{12}$/);
+  assert.equal(records.release.releaseId, `REL-${"A".repeat(12)}`);
+  assert.equal(records.release.articleCount, 1);
+  assert.equal(records.site.origin, "https://publisher.invalid");
+  assert.equal(records.site.analytics.provider, "disabled");
+  assert.equal(records.site.ads.enabled, false);
+  assert.equal(records.article.title, loaded.article.frontMatter.title);
+  assert.match(records.article.content[0].markdown, /Canonical Markdown/);
+  assert.equal(records.article.seo.index, false);
+  assert.deepEqual(records.presentation.home.featuredArticleIds, [projected.articleId]);
+});
+
+test("projects headings and lists deterministically", () => {
+  assert.deepEqual(projectMarkdown("# Title\n\n## Steps\n\n1. First\n2. Second"), {
+    blocks: [
+      { type: "heading", id: "steps", text: "Steps", level: 2 },
+      { type: "list", ordered: true, items: ["First", "Second"] },
+    ],
+    toc: [{ id: "steps", text: "Steps", level: 2 }],
+  });
+  assert.throws(() => projectMarkdown("## Same\n\n## Same"), {
+    code: "DUPLICATE_HEADING",
+  });
+});
+
+test("fails closed for a non-Public-Sites target", async () => {
+  const loaded = await loadArticleWithSite(
+    process.cwd(),
+    "content/articles/publisher-loop.md",
+  );
+  assert.throws(
+    () =>
+      projectPublicSitesRecords({
+        ...loaded,
+        site: { ...loaded.site, public_sites: { target: "unknown" } },
+        sourceSha: "b".repeat(40),
+      }),
+    { code: "UNSUPPORTED_TARGET" },
+  );
+});


## `feat(public-sites): compile release directory`

diff --git a/src/public-sites-release.js b/src/public-sites-release.js
new file mode 100644
index 0000000..24d7ad5
--- /dev/null
+++ b/src/public-sites-release.js
@@ -0,0 +1,136 @@
+import { createHash, randomUUID } from "node:crypto";
+import { mkdir, rename, rm, stat, writeFile } from "node:fs/promises";
+import path from "node:path";
+
+import { validateArticleSource, validateSiteSource } from "./content.js";
+import { readFileAtCommit } from "./git.js";
+import { projectPublicSitesRecords } from "./public-sites-projection.js";
+import { containsSensitiveMaterial } from "./sensitive.js";
+
+export class PublicSitesReleaseError extends Error {
+  constructor(code, message) {
+    super(message);
+    this.name = "PublicSitesReleaseError";
+    this.code = code;
+  }
+}
+
+function canonicalizeJson(value) {
+  if (value === null || typeof value === "boolean" || typeof value === "string") {
+    return JSON.stringify(value);
+  }
+  if (typeof value === "number") {
+    if (!Number.isFinite(value)) throw new PublicSitesReleaseError("INVALID_NUMBER", "Release number is not finite");
+    return JSON.stringify(value);
+  }
+  if (Array.isArray(value)) return `[${value.map(canonicalizeJson).join(",")}]`;
+  if (typeof value === "object") {
+    return `{${Object.keys(value)
+      .sort()
+      .map((key) => `${JSON.stringify(key)}:${canonicalizeJson(value[key])}`)
+      .join(",")}}`;
+  }
+  throw new PublicSitesReleaseError("INVALID_JSON_VALUE", "Release contains a non-JSON value");
+}
+
+const jsonBytes = (value) => Buffer.from(`${JSON.stringify(value, null, 2)}\n`, "utf8");
+const sha256 = (bytes) => createHash("sha256").update(bytes).digest("hex");
+
+function payloadFiles(projection) {
+  const { articleId, records } = projection;
+  return new Map([
+    [`articles/${articleId}.json`, jsonBytes(records.article)],
+    ["media/media-manifest.json", jsonBytes(records.mediaManifest)],
+    ["navigation.json", jsonBytes(records.navigation)],
+    ["pages/about.json", jsonBytes(records.about)],
+    ["presentation.json", jsonBytes(records.presentation)],
+    ["redirects.json", jsonBytes(records.redirects)],
+    ["site.json", jsonBytes(records.site)],
+    ["taxonomy.json", jsonBytes(records.taxonomy)],
+  ]);
+}
+
+async function pathExists(target) {
+  try {
+    await stat(target);
+    return true;
+  } catch (error) {
+    if (error.code === "ENOENT") return false;
+    throw error;
+  }
+}
+
+export async function writePublicSitesRelease(outputDirectory, projection) {
+  const output = path.resolve(outputDirectory);
+  if (await pathExists(output)) {
+    throw new PublicSitesReleaseError("OUTPUT_EXISTS", "Release output already exists");
+  }
+  const parent = path.dirname(output);
+  await mkdir(parent, { recursive: true });
+  const temporary = path.join(parent, `.${path.basename(output)}.${randomUUID()}.tmp`);
+  const files = payloadFiles(projection);
+  const checksumManifest = Buffer.from(
+    [...files.entries()]
+      .sort(([left], [right]) => Buffer.from(left).compare(Buffer.from(right)))
+      .map(([relativePath, bytes]) => `${sha256(bytes)}  ${relativePath}\n`)
+      .join(""),
+    "utf8",
+  );
+  const releaseWithZeroChecksum = {
+    ...projection.records.release,
+    bundleChecksum: `sha256:${"0".repeat(64)}`,
+  };
+  const bundleChecksum = `sha256:${sha256(
+    Buffer.concat([
+      Buffer.from(`${canonicalizeJson(releaseWithZeroChecksum)}\n`, "utf8"),
+      checksumManifest,
+    ]),
+  )}`;
+  const release = { ...releaseWithZeroChecksum, bundleChecksum };
+  const serialized = Buffer.concat([...files.values(), checksumManifest, jsonBytes(release)]);
+  if (containsSensitiveMaterial(serialized.toString("utf8"), release)) {
+    throw new PublicSitesReleaseError("SENSITIVE_RELEASE", "Sensitive material is not permitted in a release");
+  }
+
+  try {
+    for (const [relativePath, bytes] of files) {
+      const target = path.join(temporary, relativePath);
+      await mkdir(path.dirname(target), { recursive: true });
+      await writeFile(target, bytes, { flag: "wx", mode: 0o600 });
+    }
+    await writeFile(path.join(temporary, "checksums.txt"), checksumManifest, {
+      flag: "wx",
+      mode: 0o600,
+    });
+    await writeFile(path.join(temporary, "release.json"), jsonBytes(release), {
+      flag: "wx",
+      mode: 0o600,
+    });
+    await rename(temporary, output);
+  } catch (error) {
+    await rm(temporary, { force: true, recursive: true });
+    throw error;
+  }
+  return Object.freeze({
+    articleId: projection.articleId,
+    bundleChecksum,
+    releaseDirectory: output,
+    releaseId: release.releaseId,
+  });
+}
+
+export async function compilePublicSitesRelease({ root, articlePath, sourceSha, outputDirectory }) {
+  const article = validateArticleSource(
+    await readFileAtCommit(root, sourceSha, articlePath),
+    articlePath,
+  );
+  const sitePath = `sites/${article.frontMatter.site}.yml`;
+  const site = validateSiteSource(await readFileAtCommit(root, sourceSha, sitePath), sitePath);
+  if (site.id !== article.frontMatter.site) {
+    throw new PublicSitesReleaseError("SITE_MISMATCH", "Release site identity does not match article");
+  }
+  return writePublicSitesRelease(
+    outputDirectory,
+    projectPublicSitesRecords({ article, site, sourceSha }),
+  );
+}
diff --git a/test/public-sites-release.test.js b/test/public-sites-release.test.js
new file mode 100644
index 0000000..315892f
--- /dev/null
+++ b/test/public-sites-release.test.js
@@ -0,0 +1,59 @@
+import assert from "node:assert/strict";
+import { createHash } from "node:crypto";
+import { mkdtemp, readFile, readdir, rm } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+import test from "node:test";
+
+import { loadArticleWithSite } from "../src/content.js";
+import { projectPublicSitesRecords } from "../src/public-sites-projection.js";
+import { writePublicSitesRelease } from "../src/public-sites-release.js";
+
+async function fileSnapshot(root) {
+  const paths = [];
+  async function visit(directory) {
+    for (const entry of await readdir(directory, { withFileTypes: true })) {
+      const absolute = path.join(directory, entry.name);
+      if (entry.isDirectory()) await visit(absolute);
+      else paths.push(path.relative(root, absolute));
+    }
+  }
+  await visit(root);
+  paths.sort();
+  return Promise.all(paths.map(async (name) => [name, await readFile(path.join(root, name), "utf8")]));
+}
+
+test("writes deterministic release bytes and complete checksum manifest", async (context) => {
+  const temporary = await mkdtemp(path.join(os.tmpdir(), "publisher-release-"));
+  context.after(() => rm(temporary, { force: true, recursive: true }));
+  const loaded = await loadArticleWithSite(process.cwd(), "content/articles/publisher-loop.md");
+  const projection = projectPublicSitesRecords({ ...loaded, sourceSha: "c".repeat(40) });
+  const first = path.join(temporary, "first");
+  const second = path.join(temporary, "second");
+
+  const result = await writePublicSitesRelease(first, projection);
+  await writePublicSitesRelease(second, projection);
+  assert.deepEqual(await fileSnapshot(first), await fileSnapshot(second));
+  assert.match(result.bundleChecksum, /^sha256:[0-9a-f]{64}$/);
+
+  const manifest = await readFile(path.join(first, "checksums.txt"), "utf8");
+  for (const line of manifest.trimEnd().split("\n")) {
+    const [digest, relativePath] = line.split("  ");
+    const bytes = await readFile(path.join(first, relativePath));
+    assert.equal(createHash("sha256").update(bytes).digest("hex"), digest);
+  }
+  assert.doesNotMatch(manifest, /release\.json|checksums\.txt/);
+});
+
+test("refuses to overwrite a release directory", async (context) => {
+  const temporary = await mkdtemp(path.join(os.tmpdir(), "publisher-release-"));
+  context.after(() => rm(temporary, { force: true, recursive: true }));
+  const loaded = await loadArticleWithSite(process.cwd(), "content/articles/publisher-loop.md");
+  const projection = projectPublicSitesRecords({ ...loaded, sourceSha: "d".repeat(40) });
+  const output = path.join(temporary, "release");
+  await writePublicSitesRelease(output, projection);
+
+  await assert.rejects(() => writePublicSitesRelease(output, projection), {
+    code: "OUTPUT_EXISTS",
+  });
+});


## `fix(public-sites): separate site and target identity`

diff --git a/src/public-sites-projection.js b/src/public-sites-projection.js
index 08b73f7..1c96951 100644
--- a/src/public-sites-projection.js
+++ b/src/public-sites-projection.js
@@ -91,6 +91,7 @@ export function projectPublicSitesRecords({ article, site, sourceSha }) {
   const author = { displayName: frontMatter.author, profileId: "publisher-author" };
   const theme = "minimal-knowledge-base";
   const skin = "calm-blue";
+  const publicSiteId = site.public_sites.target;
 
   return Object.freeze({
     articleId: publicArticleId,
@@ -98,7 +99,7 @@ export function projectPublicSitesRecords({ article, site, sourceSha }) {
       release: {
         contractVersion: "4.0.0",
         releaseId: `REL-${sourceSha.slice(0, 12).toUpperCase()}`,
-        siteId: site.id,
+        siteId: publicSiteId,
         createdAt: frontMatter.created_at,
         contentRevision: Number.parseInt(sourceSha.slice(0, 8), 16),
         siteConfigRevision: Number.parseInt(sourceSha.slice(8, 16), 16),
@@ -109,7 +110,7 @@ export function projectPublicSitesRecords({ article, site, sourceSha }) {
         bundleChecksum: `sha256:${"0".repeat(64)}`,
       },
       site: {
-        id: site.id,
+        id: publicSiteId,
         origin: "https://publisher.invalid",
         locale: "en-US",
         timeZone: "Asia/Seoul",
diff --git a/test/public-sites-projection.test.js b/test/public-sites-projection.test.js
index dddcad6..bb4956d 100644
--- a/test/public-sites-projection.test.js
+++ b/test/public-sites-projection.test.js
@@ -19,6 +19,8 @@ test("projects the real Publisher article without copying renderer fixtures", as
   assert.match(projected.articleId, /^ART-[A-F0-9]{12}$/);
   assert.equal(records.release.releaseId, `REL-${"A".repeat(12)}`);
   assert.equal(records.release.articleCount, 1);
+  assert.equal(records.release.siteId, "site-a");
+  assert.equal(records.site.id, "site-a");
   assert.equal(records.site.origin, "https://publisher.invalid");
   assert.equal(records.site.analytics.provider, "disabled");
   assert.equal(records.site.ads.enabled, false);
