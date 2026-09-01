# 재현 가능한 Standalone 컨테이너 패키징과 런타임 검증

## `chore(runtime): 지원 Node.js와 npm 버전 고정`

diff --git a/.node-version b/.node-version
new file mode 100644
index 0000000..ca5c350
--- /dev/null
+++ b/.node-version
@@ -0,0 +1 @@
+24.18.0
diff --git a/.nvmrc b/.nvmrc
new file mode 100644
index 0000000..ca5c350
--- /dev/null
+++ b/.nvmrc
@@ -0,0 +1 @@
+24.18.0
diff --git a/package-lock.json b/package-lock.json
index d8078b1..23de53f 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -29,6 +29,10 @@
         "tsx": "^4.23.0",
         "typescript": "^5",
         "vitest": "^3.2.4"
+      },
+      "engines": {
+        "node": "24.18.0",
+        "npm": "11.16.0"
       }
     },
     "node_modules/@adobe/css-tools": {
diff --git a/package.json b/package.json
index fde3f1b..cffeede 100644
--- a/package.json
+++ b/package.json
@@ -2,6 +2,11 @@
   "name": "woopinbell-portfolio",
   "version": "0.1.0",
   "private": true,
+  "packageManager": "npm@11.16.0",
+  "engines": {
+    "node": "24.18.0",
+    "npm": "11.16.0"
+  },
   "scripts": {
     "dev": "next dev --webpack -p 3100",
     "prebuild": "npm run content:check",


## `fix(build): production build에 webpack compiler 고정`

diff --git a/package.json b/package.json
index 7478f7c..d496daa 100644
--- a/package.json
+++ b/package.json
@@ -10,7 +10,7 @@
   "scripts": {
     "dev": "next dev --webpack -p 3100",
     "prebuild": "npm run content:check && npm run content:ready",
-    "build": "next build",
+    "build": "next build --webpack",
     "start": "next start -p 3100",
     "start:e2e": "next start -p 3200",
     "lint": "eslint .",


## `build: standalone server 산출물 생성`

diff --git a/next.config.ts b/next.config.ts
index ca6c939..ead555b 100644
--- a/next.config.ts
+++ b/next.config.ts
@@ -2,6 +2,7 @@ import type { NextConfig } from "next";
 
 const nextConfig: NextConfig = {
   devIndicators: false,
+  output: "standalone",
 };
 
 export default nextConfig;


## `test(build): standalone 산출물 완전성 검증`

diff --git a/package.json b/package.json
index d496daa..bb92b96 100644
--- a/package.json
+++ b/package.json
@@ -11,6 +11,7 @@
     "dev": "next dev --webpack -p 3100",
     "prebuild": "npm run content:check && npm run content:ready",
     "build": "next build --webpack",
+    "build:verify": "node scripts/verify-build-output.mjs",
     "start": "next start -p 3100",
     "start:e2e": "next start -p 3200",
     "lint": "eslint .",
diff --git a/scripts/verify-build-output.mjs b/scripts/verify-build-output.mjs
new file mode 100644
index 0000000..b4a02e0
--- /dev/null
+++ b/scripts/verify-build-output.mjs
@@ -0,0 +1,15 @@
+import { existsSync } from "node:fs";
+import { resolve } from "node:path";
+
+const requiredArtifacts = [
+  ".next/standalone/server.js",
+  ".next/static"
+];
+
+const missing = requiredArtifacts.filter((artifact) => !existsSync(resolve(artifact)));
+
+if (missing.length > 0) {
+  throw new Error(`Standalone build output is incomplete:\n${missing.map((artifact) => `- ${artifact}`).join("\n")}`);
+}
+
+console.log(`verified ${requiredArtifacts.length} portfolio build artifacts`);


## `build(docker): public 자산을 포함한 비루트 standalone image 추가`

diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..09c98bf
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,12 @@
+.git
+.github
+.next
+node_modules
+coverage
+playwright-report
+test-results
+output
+.env
+.env.*
+*.log
+README.md
diff --git a/Dockerfile b/Dockerfile
new file mode 100644
index 0000000..c8217c5
--- /dev/null
+++ b/Dockerfile
@@ -0,0 +1,29 @@
+FROM node:24.18.0-bookworm-slim AS dependencies
+
+RUN npm install --global npm@11.16.0
+WORKDIR /app
+COPY package.json package-lock.json ./
+RUN npm ci
+
+FROM dependencies AS builder
+
+ARG PORTFOLIO_CONTENT_MODE=template
+ARG SITE_URL
+ENV PORTFOLIO_CONTENT_MODE=$PORTFOLIO_CONTENT_MODE
+ENV SITE_URL=$SITE_URL
+COPY . .
+RUN npm run build && npm run build:verify
+
+FROM node:24.18.0-bookworm-slim AS runner
+
+ENV NODE_ENV=production
+ENV HOSTNAME=0.0.0.0
+ENV PORT=3100
+WORKDIR /app
+USER node
+COPY --from=builder --chown=node:node /app/.next/standalone ./
+COPY --from=builder --chown=node:node /app/.next/static ./.next/static
+COPY --from=builder --chown=node:node /app/public ./public
+
+EXPOSE 3100
+CMD ["node", "server.js"]


## `test(docker): runtime route와 public 자산 검증 자동화`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 8031e9e..af4d349 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -62,3 +62,6 @@ jobs:
 
       - name: Run production Lighthouse budgets
         run: npm run lighthouse:audit
+
+      - name: Verify container runtime and public assets
+        run: npm run test:container
diff --git a/package.json b/package.json
index 63b4949..babb5cc 100644
--- a/package.json
+++ b/package.json
@@ -24,6 +24,7 @@
     "content:check": "node --import tsx scripts/validate-content.ts",
     "content:ready": "node --import tsx scripts/validate-content-readiness.ts",
     "test": "vitest run",
+    "test:container": "node scripts/verify-container-runtime.mjs",
     "test:watch": "vitest",
     "test:e2e": "playwright test",
     "test:e2e:ci": "npm run build && playwright test --config=playwright.production.config.ts --grep-invert @visual",
diff --git a/scripts/verify-container-runtime.mjs b/scripts/verify-container-runtime.mjs
new file mode 100644
index 0000000..cc65ddd
--- /dev/null
+++ b/scripts/verify-container-runtime.mjs
@@ -0,0 +1,203 @@
+import { spawn } from "node:child_process";
+import { randomBytes } from "node:crypto";
+import { readdir, readFile } from "node:fs/promises";
+import path from "node:path";
+import process from "node:process";
+import { setTimeout as delay } from "node:timers/promises";
+
+const suffix = `${process.pid}-${randomBytes(4).toString("hex")}`;
+const imageName = `portfolio-runtime-test:${suffix}`;
+const containerName = `portfolio-runtime-test-${suffix}`;
+const contentDirectory = path.join(process.cwd(), "src/content");
+const mimeByExtension = new Map([
+  [".avif", "image/avif"],
+  [".jpeg", "image/jpeg"],
+  [".jpg", "image/jpeg"],
+  [".pdf", "application/pdf"],
+  [".png", "image/png"],
+  [".svg", "image/svg+xml"],
+  [".webp", "image/webp"],
+]);
+
+async function docker(args, options = {}) {
+  return new Promise((resolve, reject) => {
+    const child = spawn("docker", args, {
+      cwd: process.cwd(),
+      stdio: options.capture ? ["ignore", "pipe", "pipe"] : "inherit",
+    });
+    let stderr = "";
+    let stdout = "";
+
+    if (options.capture) {
+      child.stdout.setEncoding("utf8");
+      child.stderr.setEncoding("utf8");
+      child.stdout.on("data", (chunk) => {
+        stdout += chunk;
+      });
+      child.stderr.on("data", (chunk) => {
+        stderr += chunk;
+      });
+    }
+
+    child.on("error", reject);
+    child.on("close", (exitCode) => {
+      if (exitCode !== 0) {
+        reject(
+          new Error(
+            `docker ${args.join(" ")} exited ${exitCode}${stderr ? `: ${stderr.trim()}` : ""}`,
+          ),
+        );
+        return;
+      }
+      resolve(stdout.trim());
+    });
+  });
+}
+
+function collectAssetPaths(value, assets) {
+  if (typeof value === "string") {
+    if (/^\/(content|template)\//.test(value)) {
+      assets.add(value);
+    }
+    return;
+  }
+
+  if (Array.isArray(value)) {
+    for (const item of value) collectAssetPaths(item, assets);
+    return;
+  }
+
+  if (value && typeof value === "object") {
+    for (const item of Object.values(value)) collectAssetPaths(item, assets);
+  }
+}
+
+async function discoverAssets() {
+  const assets = new Set();
+
+  for (const filename of (await readdir(contentDirectory)).sort()) {
+    if (!filename.endsWith(".json")) continue;
+    collectAssetPaths(
+      JSON.parse(await readFile(path.join(contentDirectory, filename), "utf8")),
+      assets,
+    );
+  }
+
+  return [...assets].sort();
+}
+
+async function waitUntilReady(baseUrl) {
+  let lastError;
+
+  for (let attempt = 0; attempt < 60; attempt += 1) {
+    try {
+      const response = await fetch(baseUrl);
+      if (response.ok) return;
+      lastError = new Error(`readiness returned ${response.status}`);
+    } catch (error) {
+      lastError = error;
+    }
+    await delay(1_000);
+  }
+
+  throw new Error(`container did not become ready: ${lastError}`);
+}
+
+async function verifyResponse(baseUrl, pathname, expectedMime) {
+  const response = await fetch(new URL(pathname, baseUrl));
+  const body = new Uint8Array(await response.arrayBuffer());
+  const contentType = (response.headers.get("content-type") ?? "").toLowerCase();
+
+  if (response.status !== 200) {
+    throw new Error(`${pathname}: expected 200, received ${response.status}`);
+  }
+  if (body.byteLength === 0) {
+    throw new Error(`${pathname}: response body is empty`);
+  }
+  if (expectedMime && !contentType.includes(expectedMime)) {
+    throw new Error(
+      `${pathname}: expected ${expectedMime}, received ${contentType || "none"}`,
+    );
+  }
+}
+
+let containerStarted = false;
+let failed = false;
+
+try {
+  const assets = await discoverAssets();
+  if (assets.length === 0) {
+    throw new Error("content JSON did not reference a public asset");
+  }
+
+  await docker(["build", "--tag", imageName, "."]);
+  await docker([
+    "run",
+    "--detach",
+    "--name",
+    containerName,
+    "--publish",
+    "127.0.0.1::3100",
+    imageName,
+  ]);
+  containerStarted = true;
+
+  const publishedPort = await docker(
+    ["port", containerName, "3100/tcp"],
+    { capture: true },
+  );
+  const port = publishedPort.match(/:(\d+)\s*$/)?.[1];
+  if (!port) throw new Error(`cannot parse published port: ${publishedPort}`);
+
+  const baseUrl = `http://127.0.0.1:${port}`;
+  await waitUntilReady(baseUrl);
+
+  const user = await docker(
+    ["inspect", "--format", "{{.Config.User}}", containerName],
+    { capture: true },
+  );
+  if (user !== "node") {
+    throw new Error(`container must run as node, received ${user || "root"}`);
+  }
+
+  await verifyResponse(baseUrl, "/", "text/html");
+  await verifyResponse(
+    baseUrl,
+    "/projects/example-project?view=classic",
+    "text/html",
+  );
+
+  for (const asset of assets) {
+    const extension = path.extname(new URL(asset, baseUrl).pathname).toLowerCase();
+    const expectedMime = mimeByExtension.get(extension);
+    if (!expectedMime) {
+      throw new Error(`${asset}: unsupported MIME contract for ${extension}`);
+    }
+    await verifyResponse(baseUrl, asset, expectedMime);
+  }
+
+  console.log(
+    `verified non-root container, 2 routes, and ${assets.length} content assets`,
+  );
+} catch (error) {
+  failed = true;
+  if (containerStarted) {
+    try {
+      await docker(["logs", containerName]);
+    } catch {}
+  }
+  throw error;
+} finally {
+  if (containerStarted) {
+    try {
+      await docker(["rm", "--force", containerName], { capture: true });
+    } catch (error) {
+      if (!failed) throw error;
+    }
+  }
+  try {
+    await docker(["image", "rm", "--force", imageName], { capture: true });
+  } catch (error) {
+    if (!failed) throw error;
+  }
+}
