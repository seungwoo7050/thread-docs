## `브라우저에서 Monitor를 만들고 Check 결과를 표시`

diff --git a/.gitignore b/.gitignore
index 37b9fa8..396e6b0 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,6 +1,9 @@
 node_modules/
 .next/
 *.tsbuildinfo
+next-env.d.ts
+AGENTS.md
+CLAUDE.md
 output/
 playwright-report/
 test-results/
diff --git a/TRACK.md b/TRACK.md
index 2085406..ede90d6 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -19,15 +19,27 @@ A transport failure records `FAILED/CONNECTION_FAILED` or `FAILED/TIMEOUT` with
 | Node.js | 24.19.0 |
 | npm | 11.17.0 |
 | Fastify | 5.12.1 |
+| Next.js | 16.3.3 |
+| React / react-dom | 19.2.8 |
+| Playwright | 1.62.1 |
+| Chromium (Playwright revision 1234) | 151.0.7922.34 |
 | TypeScript | 5.9.3 |
 | @types/node | 24.13.3 |
+| @types/react | 19.2.18 |
+| @types/react-dom | 19.2.5 |
 
 Direct dependencies are exact in `package.json`; transitives are pinned in
 `package-lock.json`. Node supplies TypeScript stripping and the unit/functional
 test runner, so no extra runtime loader or test framework is needed.
+`typecheck` generates Next route types first; `next-env.d.ts` and `.next` are
+generated build inputs and are not committed.
+Next's automatically written agent helper files are also ignored generated files.
 Versions were confirmed against the official npm registry. Fastify's
 [server factory reference](https://fastify.dev/docs/latest/Reference/Server/)
 documents its request and listener configuration.
+Playwright's pinned [browser manifest](https://github.com/microsoft/playwright/blob/v1.62.1/packages/playwright-core/browsers.json)
+identifies the browser revision; the [web server configuration](https://playwright.dev/docs/test-webserver)
+starts the three real local processes used by E2E.
 
 ## Run
 
@@ -38,8 +50,14 @@ npm ci
 npm run fixture
 # A separate terminal:
 npm run dev:api
+# A third terminal:
+npm run dev:web
 ```
 
+Open `http://127.0.0.1:4313/monitors`. Next proxies same-origin `/api` requests to
+the Fastify loopback API; `API_ORIGIN` configures that trusted proxy target.
+Create a Monitor for `/ok`, `/fail`, or `/redirect`, then choose **Run check**.
+
 `API_PORT` changes the API port (default 4312). `FIXTURE_PORT` changes the fixture
 listener (default 4311); set `FIXTURE_ORIGIN` on the API to the same trusted origin.
 Do not expose this development service to a public interface.
@@ -48,9 +66,15 @@ Do not expose this development service to a public interface.
 npm run typecheck
 npm run test:unit
 npm run test:functional
+npx playwright install chromium
+npm run test:e2e
+npm run build
 ```
 
 Functional tests own fixed loopback ports 4311 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
 never receive a check. State disappears on API restart. There is no login,
 database, worker, Redis, Kafka, or production container in E01.
+The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
+stop manual development processes first. It uses one Chromium worker, no test
+retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
diff --git a/app/layout.tsx b/app/layout.tsx
new file mode 100644
index 0000000..b24eaeb
--- /dev/null
+++ b/app/layout.tsx
@@ -0,0 +1,8 @@
+import type { Metadata } from 'next';
+import './style.css';
+
+export const metadata: Metadata = { title: 'Endpoint Monitor', description: 'Local HTTP fixture monitoring' };
+
+export default function RootLayout({ children }: { children: React.ReactNode }) {
+  return <html lang="en"><body>{children}</body></html>;
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
new file mode 100644
index 0000000..a6eeb96
--- /dev/null
+++ b/app/monitors/page.tsx
@@ -0,0 +1,97 @@
+'use client';
+
+import { useEffect, useState } from 'react';
+import type { FormEvent } from 'react';
+import type { CheckRun, MonitorView } from '../../server/model';
+
+export default function MonitorsPage() {
+  const [monitors, setMonitors] = useState<MonitorView[]>([]);
+  const [error, setError] = useState<string | null>(null);
+  const [creating, setCreating] = useState(false);
+  const [checking, setChecking] = useState<string | null>(null);
+
+  useEffect(() => {
+    fetch('/api/monitors').then(async (response) => {
+      if (!response.ok) throw new Error('Could not load monitors.');
+      setMonitors(await response.json());
+    }).catch((failure: Error) => setError(failure.message));
+  }, []);
+
+  async function createMonitor(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    const fields = new FormData(form);
+    setCreating(true);
+    setError(null);
+    try {
+      const response = await fetch('/api/monitors', {
+        method: 'POST', headers: { 'content-type': 'application/json' },
+        body: JSON.stringify({
+          name: fields.get('name'), url: fields.get('url'),
+          interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
+        }),
+      });
+      const body = await response.json();
+      if (!response.ok) throw new Error(body.message ?? 'Could not create monitor.');
+      setMonitors((current) => [...current, body]);
+      form.reset();
+    } catch (failure) {
+      setError(failure instanceof Error ? failure.message : 'Could not create monitor.');
+    } finally { setCreating(false); }
+  }
+
+  async function runCheck(id: string) {
+    setChecking(id);
+    setError(null);
+    try {
+      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
+      const body = await response.json();
+      if (!response.ok) throw new Error(body.message ?? 'Could not run check.');
+      const result: CheckRun = body;
+      setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
+    } catch (failure) {
+      setError(failure instanceof Error ? failure.message : 'Could not run check.');
+    } finally { setChecking(null); }
+  }
+
+  return <main>
+    <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Monitors</h1>
+      <p>Create an endpoint monitor, run a check, and inspect the response.</p>
+    </header>
+    <aside>Development only. Checks can access the configured fixture origin. State is lost on API restart.</aside>
+    {error && <p role="alert" className="error">{error}</p>}
+    <section aria-labelledby="create-heading" className="panel">
+      <h2 id="create-heading">Create monitor</h2>
+      <form onSubmit={createMonitor}>
+        <label htmlFor="name">Name</label>
+        <input id="name" name="name" required placeholder="Fixture monitor" />
+        <label htmlFor="url">Endpoint URL</label>
+        <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" />
+        <label htmlFor="interval">Interval (seconds)</label>
+        <input id="interval" name="interval" type="number" required min="1" step="1" defaultValue="60" />
+        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked /> Enabled</label>
+        <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
+        <button type="submit" disabled={creating}>{creating ? 'Creating…' : 'Create monitor'}</button>
+      </form>
+    </section>
+    <section aria-labelledby="saved-heading">
+      <h2 id="saved-heading">Your monitors</h2>
+      {monitors.length === 0 && <p>No monitors yet.</p>}
+      {monitors.map((monitor) => <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
+        <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
+        <p className="endpoint">{monitor.url}</p>
+        <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
+        <button onClick={() => runCheck(monitor.id)} disabled={checking !== null}>
+          {checking === monitor.id ? 'Checking…' : 'Run check'}
+        </button>
+        {monitor.latestCheck ? <dl aria-label="Latest check result">
+          <dt>State</dt><dd>{monitor.latestCheck.state}</dd>
+          <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
+          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs} ms</dd>
+          <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
+          <dt>Finished</dt><dd><time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time></dd>
+        </dl> : <p>No checks yet.</p>}
+      </article>)}
+    </section>
+  </main>;
+}
diff --git a/app/page.tsx b/app/page.tsx
new file mode 100644
index 0000000..2d10433
--- /dev/null
+++ b/app/page.tsx
@@ -0,0 +1,3 @@
+import { redirect } from 'next/navigation';
+
+export default function Home() { redirect('/monitors'); }
diff --git a/app/style.css b/app/style.css
new file mode 100644
index 0000000..01a2dfb
--- /dev/null
+++ b/app/style.css
@@ -0,0 +1,24 @@
+:root { color-scheme: light; font-family: system-ui, sans-serif; color: #1c2d3d; background: #f3f6f8; }
+* { box-sizing: border-box; }
+body { margin: 0; }
+main { max-width: 820px; margin: 0 auto; padding: 40px 20px 72px; }
+h1 { font-size: 2.4rem; margin: 10px 0; }
+h2 { font-size: 1.35rem; }
+h3 { font-size: 1.2rem; margin-top: 0; }
+p { line-height: 1.55; }
+.eyebrow { font-size: .75rem; font-weight: 700; letter-spacing: .1em; color: #346471; }
+aside { padding: 14px 18px; background: #e0edf1; border-left: 3px solid #346471; line-height: 1.5; }
+.panel { background: white; border: 1px solid #cbd6dd; border-radius: 8px; margin: 24px 0; padding: 24px; }
+form { display: grid; gap: 9px; }
+input { width: 100%; font: inherit; border: 1px solid #8c9daa; border-radius: 4px; padding: 10px; margin-bottom: 7px; }
+.checkbox { display: flex; gap: 8px; align-items: center; }
+.checkbox input { width: auto; margin: 0; }
+button { font: inherit; color: white; background: #245d68; border: 0; border-radius: 4px; padding: 11px 18px; cursor: pointer; justify-self: start; }
+button:disabled { opacity: .6; cursor: wait; }
+:focus-visible { outline: 3px solid #a85816; outline-offset: 3px; }
+.hint { font-size: .875rem; color: #51626f; margin: 2px 0 10px; }
+.error { color: #8b1c23; background: #ffe9e9; padding: 15px; }
+.endpoint { overflow-wrap: anywhere; font-family: monospace; }
+dl { display: grid; grid-template-columns: 110px minmax(0, 1fr); gap: 10px; margin-bottom: 0; }
+dt { color: #51626f; }
+dd { margin: 0; overflow-wrap: anywhere; }
diff --git a/evidence/E01/verification.json b/evidence/E01/verification.json
index e9bac48..ac67ce6 100644
--- a/evidence/E01/verification.json
+++ b/evidence/E01/verification.json
@@ -4,7 +4,16 @@
   "runs": [
     { "command": "npm run typecheck", "invocation": 1, "exitCode": 0, "elapsedSeconds": 0.97 },
     { "command": "npm run test:unit", "invocation": 1, "exitCode": 0, "passed": 3, "elapsedSeconds": 0.26 },
-    { "command": "npm run test:functional", "invocation": 1, "exitCode": 0, "passed": 5, "elapsedSeconds": 0.38 }
+    { "command": "npm run test:functional", "invocation": 1, "exitCode": 0, "passed": 5, "elapsedSeconds": 0.38 },
+    { "command": "npm run typecheck", "invocation": 2, "exitCode": 0, "elapsedSeconds": 1.23 },
+    { "command": "npm run test:e2e", "invocation": 1, "exitCode": 1, "elapsedSeconds": 15.44,
+      "observed": "Monitor created and SUCCEEDED/200 displayed; auxiliary page-wide zero-alert assertion matched Next's empty route announcer outside main.",
+      "correction": "Scope application-error assertion to main; no fixture, request sequence, result threshold, timeout, or application behavior changed." },
+    { "command": "npm run test:e2e", "invocation": 2, "exitCode": 0, "passed": 1, "elapsedSeconds": 5.02,
+      "browser": "Chromium 151.0.7922.34 (revision 1234)", "screenshot": "output/playwright/E01-success.png (local generated artifact, visually inspected)" },
+    { "command": "npm run typecheck", "invocation": 3, "exitCode": 0, "elapsedSeconds": 1.74,
+      "note": "Now generates Next route types before tsc so a clean checkout is supported." },
+    { "command": "npm run build", "invocation": 1, "exitCode": 0, "elapsedSeconds": 5.66 }
   ],
   "measurement": "/usr/bin/time -p; each command run with fnm exec --using 24.19.0",
   "loadRuns": 0,
diff --git a/next.config.ts b/next.config.ts
new file mode 100644
index 0000000..b8e9929
--- /dev/null
+++ b/next.config.ts
@@ -0,0 +1,10 @@
+import type { NextConfig } from 'next';
+
+const config: NextConfig = {
+  turbopack: { root: process.cwd() },
+  async rewrites() {
+    return [{ source: '/api/:path*', destination: `${process.env.API_ORIGIN ?? 'http://127.0.0.1:4312'}/:path*` }];
+  },
+};
+
+export default config;
diff --git a/package-lock.json b/package-lock.json
index 11f0c2b..2a376ca 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -8,10 +8,16 @@
       "name": "monitor-fundamentals-fastify",
       "version": "0.1.0",
       "dependencies": {
-        "fastify": "5.12.1"
+        "fastify": "5.12.1",
+        "next": "16.3.3",
+        "react": "19.2.8",
+        "react-dom": "19.2.8"
       },
       "devDependencies": {
+        "@playwright/test": "1.62.1",
         "@types/node": "24.13.3",
+        "@types/react": "19.2.18",
+        "@types/react-dom": "19.2.5",
         "typescript": "5.9.3"
       },
       "engines": {
@@ -19,6 +25,16 @@
         "npm": "11.17.0"
       }
     },
+    "node_modules/@emnapi/runtime": {
+      "version": "1.11.3",
+      "resolved": "https://registry.npmjs.org/@emnapi/runtime/-/runtime-1.11.3.tgz",
+      "integrity": "sha512-Xz4Tpyki7XyrpbUK1jR1AhdAdaXyhhY4lZ3neLodmhpuWfy2PAQN5B46sAiU4liOXGLkHypn/qU+jvfWSCYYLA==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "tslib": "^2.4.0"
+      }
+    },
     "node_modules/@fastify/ajv-compiler": {
       "version": "4.0.6",
       "resolved": "https://registry.npmjs.org/@fastify/ajv-compiler/-/ajv-compiler-4.0.6.tgz",
@@ -130,12 +146,732 @@
         "ipaddr.js": "^2.1.0"
       }
     },
+    "node_modules/@img/colour": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/@img/colour/-/colour-1.1.0.tgz",
+      "integrity": "sha512-Td76q7j57o/tLVdgS746cYARfSyxk8iEfRxewL9h4OMzYhbW4TAcppl0mT4eyqXddh6L/jwoM75mo7ixa/pCeQ==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/@img/sharp-darwin-arm64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-darwin-arm64/-/sharp-darwin-arm64-0.35.4.tgz",
+      "integrity": "sha512-Uhfl4V4lhP2nbUVF9+hyH1+luj86f1gUFeo8ALYxFoULoU+G87D43BfeMP8XHsk9boxAnCY/bf2EHwhA7MuGsA==",
+      "cpu": [
+        "arm64"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-darwin-arm64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-darwin-x64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-darwin-x64/-/sharp-darwin-x64-0.35.4.tgz",
+      "integrity": "sha512-hWniXY3bG5qKpkKrAwPe4y+VTPmf086YQAnkxWh7uA1YrlRouWGa0M0Mxj3ZjnXFkv7/TD1bTy9lGUK26vRvWw==",
+      "cpu": [
+        "x64"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-darwin-x64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-freebsd-wasm32": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-freebsd-wasm32/-/sharp-freebsd-wasm32-0.35.4.tgz",
+      "integrity": "sha512-lIsKw/BU+kjB4eZjxrYrZmwOJYi3Ajrv66iAlBmUPyKc3HpnloevB1g3wxGD9P/5BbQ1brBGl65VRRrCvQDEqA==",
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "freebsd"
+      ],
+      "dependencies": {
+        "@img/sharp-wasm32": "0.35.4"
+      },
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-darwin-arm64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-darwin-arm64/-/sharp-libvips-darwin-arm64-1.3.3.tgz",
+      "integrity": "sha512-suTBPTDGrI9WodccaDdwZItTSaBYASlBk1NSfElSHrUfzu3szG6lvIF58+WiFvnfzuK8ZBFS5zE00PxqxnRiPg==",
+      "cpu": [
+        "arm64"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-darwin-x64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-darwin-x64/-/sharp-libvips-darwin-x64-1.3.3.tgz",
+      "integrity": "sha512-FVJZ5mITMobmXIz/hPDTw0EintTW5H3WfrxwLqEqjiIihlu+hVRyGrFQ60xl0Lxn7Bt3zdpevPaQi0HEzqz9fw==",
+      "cpu": [
+        "x64"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-arm": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-arm/-/sharp-libvips-linux-arm-1.3.3.tgz",
+      "integrity": "sha512-3rbU4vqXXc3hY/OiXdl52xZvT0F1yEngWfvqudtPJg/KkyiaQw2DRsFrNzpmLvfavbwOq3qXn36GP8obHRULQA==",
+      "cpu": [
+        "arm"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-arm64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-arm64/-/sharp-libvips-linux-arm64-1.3.3.tgz",
+      "integrity": "sha512-0DaL0A6Xu6sQSQFwe4iVCrKWU2cCTItnRsYsCdxAMm9NF6twAA9BKnoqy4hqz4+azQ0JHuA26qiUKsf1XJ/v5A==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-ppc64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-ppc64/-/sharp-libvips-linux-ppc64-1.3.3.tgz",
+      "integrity": "sha512-cdn1OvUBwsXhbC0zSzJnNzf5MZ/mTrobawDvNXBTxe8VtqKAm0sRuEY2Evzovb/w9JMk4TvRxqt1mekSuJz64w==",
+      "cpu": [
+        "ppc64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-riscv64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-riscv64/-/sharp-libvips-linux-riscv64-1.3.3.tgz",
+      "integrity": "sha512-HjPVx7yKz+0lqdhDlTw1tt90wamBoxhiXpvl1XZpJLiHH4RCJ5yDTqH+VlYPv2fwFs89JFw4c1IexYOcQUi4IQ==",
+      "cpu": [
+        "riscv64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-s390x": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-s390x/-/sharp-libvips-linux-s390x-1.3.3.tgz",
+      "integrity": "sha512-neWLh+3yCNThxnfy3c4BbVBeGgt9aftno+XbT56iK28RgeDs3UOFWviLWlUu0bArYVYJaFDK+RRohbicUNCm8Q==",
+      "cpu": [
+        "s390x"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linux-x64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linux-x64/-/sharp-libvips-linux-x64-1.3.3.tgz",
+      "integrity": "sha512-4vKmvAst9nrowcqquKFAyZJUDolUaIp8uRiN0mWFguJ1IplC9/pitXtlnnlU4aa/eJw3J7i67V+pwUL+wZGdsA==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linuxmusl-arm64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linuxmusl-arm64/-/sharp-libvips-linuxmusl-arm64-1.3.3.tgz",
+      "integrity": "sha512-Y9kQaLMuNoB0bPYOOdcZMaseNrFpPodIWWMrx+CZyydf2xn68j9WYc6sWWRrDwNkzCQjKYfc68L7jKjGlHMibw==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-libvips-linuxmusl-x64": {
+      "version": "1.3.3",
+      "resolved": "https://registry.npmjs.org/@img/sharp-libvips-linuxmusl-x64/-/sharp-libvips-linuxmusl-x64-1.3.3.tgz",
+      "integrity": "sha512-fj8Mv0HHfD1Rr+4I68+3agJynxDWtBFgicTbSOb9Bke6pIwzGcJ+RX/yHjmiEGFMCavY/dxvem7MyNaJF+wDiw==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-linux-arm": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-arm/-/sharp-linux-arm-0.35.4.tgz",
+      "integrity": "sha512-7OAS8gI0EReKGVN2HssHlM6umJgxF5VI3xN0p9FA91p/YO+ou5hiNghLdZ5BEHztwaaK5+bLKRf8x/o2L2nk9A==",
+      "cpu": [
+        "arm"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-arm": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linux-arm64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-arm64/-/sharp-linux-arm64-0.35.4.tgz",
+      "integrity": "sha512-De4jpEnAU8Hd5oT0j1G3uL4ZvTuipVMn7YC6vPaJhy6/7EwEae0SVAoBrUMYQbkLGDm85taVWwuPc1a44LTzCQ==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-arm64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linux-ppc64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-ppc64/-/sharp-linux-ppc64-0.35.4.tgz",
+      "integrity": "sha512-2oYZJeIl4kCcMGk4ouZVjnkCtFrpQFlNEtJ6GbxzhHQchwH0NH/qEb9ykmOl29dqwMq+JhFdZn+1ak2FKhI9fQ==",
+      "cpu": [
+        "ppc64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-ppc64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linux-riscv64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-riscv64/-/sharp-linux-riscv64-0.35.4.tgz",
+      "integrity": "sha512-cPbNChoRURAWdebDIHSenxRpgEdy7JkPydSnUxRm9VvKD7m0/xVaR/8Fzlu81pk5nHEvHH87UZUA7cTtwnbJSA==",
+      "cpu": [
+        "riscv64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-riscv64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linux-s390x": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-s390x/-/sharp-linux-s390x-0.35.4.tgz",
+      "integrity": "sha512-RY0JFY8Fd6RonCBtHz+DvadaPkXDSI1AUn6yWL9TipqkZ1vY8w8evqdgyDFnkm4/K1ve1TvZiaePP5oSd4+WVQ==",
+      "cpu": [
+        "s390x"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-s390x": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linux-x64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linux-x64/-/sharp-linux-x64-0.35.4.tgz",
+      "integrity": "sha512-9qvvEAuk8k89TfWUoX2htWjbAMX8p+NxCppjpcg5k6xMsjhBQPTsoIh36h9Qde4WRuGpJeYnOjdosDn/cnv+OA==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linux-x64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linuxmusl-arm64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linuxmusl-arm64/-/sharp-linuxmusl-arm64-0.35.4.tgz",
+      "integrity": "sha512-KB5jxpfWQTr0nc3xdHtWChdbifHrBGsd2SM62Eyxrl8afikm+f5qGBU75SJIZBT/S1MC8XyacdlXBMSWq6OURA==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linuxmusl-arm64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-linuxmusl-x64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-linuxmusl-x64/-/sharp-linuxmusl-x64-0.35.4.tgz",
+      "integrity": "sha512-f+eZJZIQNEEd26RPSW+76chwOf1XtA2Y/O+5ocVyLliHkeih3e+jhLVBdNTd2rS3IbNXK8+ug93Vf5ZXtF5Lxg==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-libvips-linuxmusl-x64": "1.3.3"
+      }
+    },
+    "node_modules/@img/sharp-wasm32": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-wasm32/-/sharp-wasm32-0.35.4.tgz",
+      "integrity": "sha512-zQnl4Kwp7Q6NHsENtU2T/00Zi+w3AQNwz3+UaTyVBy2FpXrzXzGjndpK61onhZjRtRpQXxCTeqw19bVyXOh7jA==",
+      "license": "Apache-2.0 AND LGPL-3.0-or-later AND MIT",
+      "optional": true,
+      "dependencies": {
+        "@emnapi/runtime": "^1.11.3"
+      },
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-webcontainers-wasm32": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-webcontainers-wasm32/-/sharp-webcontainers-wasm32-0.35.4.tgz",
+      "integrity": "sha512-ESfNkywmCfPNyaZjxooddJQiQ+l/nTpGEOGthxiLnIHXC/CmcBixnfwUleX9mCz9ovrUUvKMap/pm8RYbzfwaA==",
+      "cpu": [
+        "wasm32"
+      ],
+      "license": "Apache-2.0",
+      "optional": true,
+      "dependencies": {
+        "@img/sharp-wasm32": "0.35.4"
+      },
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-win32-arm64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-win32-arm64/-/sharp-win32-arm64-0.35.4.tgz",
+      "integrity": "sha512-iNdlBX9gLVvqe2I3uIJSIKTq6wckP/DYxZtcqxm09x5Gi24DnFBmPAWZmr60ZyYMG0xlzo6goG3670ar+RXvRw==",
+      "cpu": [
+        "arm64"
+      ],
+      "license": "Apache-2.0 AND LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-win32-ia32": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-win32-ia32/-/sharp-win32-ia32-0.35.4.tgz",
+      "integrity": "sha512-kqRsbaa5CS6KHlpxnN7WhE6vAAugXyZButpRdvDWetlv6Qv4N9WTcrWzF7tXfB9T7MsoadqdI8hmwLq6UlLvtw==",
+      "cpu": [
+        "ia32"
+      ],
+      "license": "Apache-2.0 AND LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": "^20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@img/sharp-win32-x64": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/@img/sharp-win32-x64/-/sharp-win32-x64-0.35.4.tgz",
+      "integrity": "sha512-XtmnYhBcrORsJ4XJngyzr/EWP0hRZLAZRFaApdKuviyqF78+ylxh2y06ZmtULAMOnObJ3ucpN0AcwSWnMowTRg==",
+      "cpu": [
+        "x64"
+      ],
+      "license": "Apache-2.0 AND LGPL-3.0-or-later",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      }
+    },
+    "node_modules/@next/env": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/env/-/env-16.3.3.tgz",
+      "integrity": "sha512-U2eYQRwXj+dsqxV79zFqExDdatnNY/ZWc2nsJU1p/OgT7fd3dXwlF6OjYaFQCfMoeTA19PWq+wVmYgimVA+V+g==",
+      "license": "MIT"
+    },
+    "node_modules/@next/swc-darwin-arm64": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-darwin-arm64/-/swc-darwin-arm64-16.3.3.tgz",
+      "integrity": "sha512-8Hiv32QJPwdV6KYJ8meR9SBA061tQqnIKTJDocvOXlEQqib0xMFpzArosuffFUUc0sslbh7QQ8a3Yey1QV8EIw==",
+      "cpu": [
+        "arm64"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-darwin-x64": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-darwin-x64/-/swc-darwin-x64-16.3.3.tgz",
+      "integrity": "sha512-A1lgKgwVchRYmSe467zdwhxT9040dd8lH+o65sL5Jet8fjB4kegw/rDyPIpYVRb6jAqwXFOJpjIXJLxQKLiE3A==",
+      "cpu": [
+        "x64"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-linux-arm64-gnu": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-linux-arm64-gnu/-/swc-linux-arm64-gnu-16.3.3.tgz",
+      "integrity": "sha512-bf0FIssMFueU2dm7vQEWWxk0c8UjKTdW0yzuh0sQsD8pf1+KCLDdaqhYZNMYGmXwEOiHAUzgBKudovIlcvvBjg==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-linux-arm64-musl": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-linux-arm64-musl/-/swc-linux-arm64-musl-16.3.3.tgz",
+      "integrity": "sha512-W7viwCk9JY/cAkdz/A273rd5bb3RgT/IHwR7Upv90tunjBWNtAAhGhoecHh+teRNRSinuAFmE+l7fwZ4YKkrXg==",
+      "cpu": [
+        "arm64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-linux-x64-gnu": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-linux-x64-gnu/-/swc-linux-x64-gnu-16.3.3.tgz",
+      "integrity": "sha512-0W46zw1N3ODpI6n0GeivHvvob1pooozgZVqy65k0mh4/7vr+FbY9+WpHzNVXjHipJf/A3FDheBG19H1s5A25rA==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-linux-x64-musl": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-linux-x64-musl/-/swc-linux-x64-musl-16.3.3.tgz",
+      "integrity": "sha512-H4mBso8ZTMBPtdT0PN0pBx2ayTvQuTuvS6qT13d77yVFJXAPCxkyIhLTmdMaGTJs0krQYI/qpzdHijCeihXhbg==",
+      "cpu": [
+        "x64"
+      ],
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-win32-arm64-msvc": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-win32-arm64-msvc/-/swc-win32-arm64-msvc-16.3.3.tgz",
+      "integrity": "sha512-cTMUJpcEGmeywofCUfhR+rSsoE33+rVPnPEYNTNdLNlsOeEg/vktOsKUSTb28vUGqD2jkm4Zaskcwn7OCI6FQg==",
+      "cpu": [
+        "arm64"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/@next/swc-win32-x64-msvc": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/@next/swc-win32-x64-msvc/-/swc-win32-x64-msvc-16.3.3.tgz",
+      "integrity": "sha512-2VR4cTBzHXaBjnGsuH6GyJjENzQOmHeAh11uY1iUhjm3j5dEUrVJuUj+VL78jaGi/Dik8xS76zEj18BsFhlVZQ==",
+      "cpu": [
+        "x64"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">= 10"
+      }
+    },
     "node_modules/@pinojs/redact": {
       "version": "0.4.0",
       "resolved": "https://registry.npmjs.org/@pinojs/redact/-/redact-0.4.0.tgz",
       "integrity": "sha512-k2ENnmBugE/rzQfEcdWHcCY+/FM3VLzH9cYEsbdsoqrvzAKRhUZeRNhAZvB8OitQJ1TBed3yqWtdjzS6wJKBwg==",
       "license": "MIT"
     },
+    "node_modules/@playwright/test": {
+      "version": "1.62.1",
+      "resolved": "https://registry.npmjs.org/@playwright/test/-/test-1.62.1.tgz",
+      "integrity": "sha512-DTcUc8qii+cpHvtOwggMtBRMjKZHXYWdw8syRYu2vtzuq4Wxphqq4NfCs5Zt44L6mA8rfDfj+PHnxFc/FeK6mQ==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "dependencies": {
+        "playwright": "1.62.1"
+      },
+      "bin": {
+        "playwright": "cli.js"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/@swc/helpers": {
+      "version": "0.5.23",
+      "resolved": "https://registry.npmjs.org/@swc/helpers/-/helpers-0.5.23.tgz",
+      "integrity": "sha512-5lSsMOTXURePglDfvuAQUqkGek9Hg2kksOYay2m0+XR++b2NWYL/4sWyuvVBIs8oKnJaxkdi9whaL/sqN13afw==",
+      "license": "Apache-2.0",
+      "dependencies": {
+        "tslib": "^2.8.0"
+      }
+    },
     "node_modules/@types/node": {
       "version": "24.13.3",
       "resolved": "https://registry.npmjs.org/@types/node/-/node-24.13.3.tgz",
@@ -146,6 +882,26 @@
         "undici-types": "~7.18.0"
       }
     },
+    "node_modules/@types/react": {
+      "version": "19.2.18",
+      "resolved": "https://registry.npmjs.org/@types/react/-/react-19.2.18.tgz",
+      "integrity": "sha512-AnzbBERsrLKtk2XSfTbYRLjQPdy116Sty4q+T+Bp3IC4l6jNBvreVPAHmpq9qhXQM7CXZPjLVmGMw9sy+hxQ3w==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "csstype": "^3.2.2"
+      }
+    },
+    "node_modules/@types/react-dom": {
+      "version": "19.2.5",
+      "resolved": "https://registry.npmjs.org/@types/react-dom/-/react-dom-19.2.5.tgz",
+      "integrity": "sha512-fMPwH9v7r/pp43yUd2/Mbiex5KouJwwR3dzHkhLREUC6764VyDsqxhAxv6OFEYR1RhjOyD1naqba8ECDBe7ZQg==",
+      "dev": true,
+      "license": "MIT",
+      "peerDependencies": {
+        "@types/react": "^19.2.0"
+      }
+    },
     "node_modules/abstract-logging": {
       "version": "2.0.1",
       "resolved": "https://registry.npmjs.org/abstract-logging/-/abstract-logging-2.0.1.tgz",
@@ -230,6 +986,44 @@
         "fastq": "^1.17.1"
       }
     },
+    "node_modules/baseline-browser-mapping": {
+      "version": "2.11.19",
+      "resolved": "https://registry.npmjs.org/baseline-browser-mapping/-/baseline-browser-mapping-2.11.19.tgz",
+      "integrity": "sha512-Grytf1xOxOEMTGRwx6rLGKkTabd4vMg3VrKdj/7joCmV0qgh4QwMMO6xh34YEXQqirAuUdgQGa5orJQQ+69RBw==",
+      "license": "Apache-2.0",
+      "bin": {
+        "baseline-browser-mapping": "dist/cli.cjs"
+      },
+      "engines": {
+        "node": ">=6.0.0"
+      }
+    },
+    "node_modules/caniuse-lite": {
+      "version": "1.0.30001810",
+      "resolved": "https://registry.npmjs.org/caniuse-lite/-/caniuse-lite-1.0.30001810.tgz",
+      "integrity": "sha512-TITQPUkaz+aVk5GL6NhOdwk1aEaNTSDPsGFWrTuhKGtjTF70jL/Oht2W4c6rXUe5fu7Ie19VIahAXHIIiWWNeg==",
+      "funding": [
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/browserslist"
+        },
+        {
+          "type": "tidelift",
+          "url": "https://tidelift.com/funding/github/npm/caniuse-lite"
+        },
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/ai"
+        }
+      ],
+      "license": "CC-BY-4.0"
+    },
+    "node_modules/client-only": {
+      "version": "0.0.1",
+      "resolved": "https://registry.npmjs.org/client-only/-/client-only-0.0.1.tgz",
+      "integrity": "sha512-IV3Ou0jSMzZrd3pZ48nLkT9DA7Ag1pnPzaiQhpW7c3RbcqqzvzzVu+L8gfqMp/8IM2MQtSiqaCxrrcfu8I8rMA==",
+      "license": "MIT"
+    },
     "node_modules/cookie": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/cookie/-/cookie-1.1.1.tgz",
@@ -243,6 +1037,13 @@
         "url": "https://opencollective.com/express"
       }
     },
+    "node_modules/csstype": {
+      "version": "3.2.3",
+      "resolved": "https://registry.npmjs.org/csstype/-/csstype-3.2.3.tgz",
+      "integrity": "sha512-z1HGKcYy2xA8AGQfwrn0PAy+PB7X/GSj3UVJW9qKyn43xWa+gl5nXmU4qqLMRzWVLFC8KusUX8T/0kCiOYpAIQ==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/dequal": {
       "version": "2.0.3",
       "resolved": "https://registry.npmjs.org/dequal/-/dequal-2.0.3.tgz",
@@ -252,6 +1053,16 @@
         "node": ">=6"
       }
     },
+    "node_modules/detect-libc": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/detect-libc/-/detect-libc-2.1.2.tgz",
+      "integrity": "sha512-Btj2BOOO83o3WyH59e8MgXsxEQVcarkUOpEYrubB0urwnN10yQ364rsiByU11nZlqWYZm05i/of7io4mzihBtQ==",
+      "license": "Apache-2.0",
+      "optional": true,
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/fast-decode-uri-component": {
       "version": "1.0.1",
       "resolved": "https://registry.npmjs.org/fast-decode-uri-component/-/fast-decode-uri-component-1.0.1.tgz",
@@ -369,6 +1180,20 @@
         "node": ">=20"
       }
     },
+    "node_modules/fsevents": {
+      "version": "2.3.2",
+      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.2.tgz",
+      "integrity": "sha512-xiqMQR4xAeHTuB9uWm+fFRcIOgKBMiOBP+eXiyT7jsgVCq1bkVygt00oASowB7EdtpOHaaPgKt812P9ab+DDKA==",
+      "hasInstallScript": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": "^8.16.0 || ^10.6.0 || >=11.0.0"
+      }
+    },
     "node_modules/ipaddr.js": {
       "version": "2.5.0",
       "resolved": "https://registry.npmjs.org/ipaddr.js/-/ipaddr.js-2.5.0.tgz",
@@ -440,6 +1265,77 @@
       ],
       "license": "MIT"
     },
+    "node_modules/nanoid": {
+      "version": "3.3.18",
+      "resolved": "https://registry.npmjs.org/nanoid/-/nanoid-3.3.18.tgz",
+      "integrity": "sha512-DTg4MJbGMWkfi6VZFdNt2/caMbQy4Ou+Op/hJQvGEWcnVfoA1QA+xzRKAzw9jD6+GVOOeYr/mIcuDSdug6F6+w==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/ai"
+        }
+      ],
+      "license": "MIT",
+      "bin": {
+        "nanoid": "bin/nanoid.cjs"
+      },
+      "engines": {
+        "node": "^10 || ^12 || ^13.7 || ^14 || >=15.0.1"
+      }
+    },
+    "node_modules/next": {
+      "version": "16.3.3",
+      "resolved": "https://registry.npmjs.org/next/-/next-16.3.3.tgz",
+      "integrity": "sha512-tuRTx1nQ/yVw83cwJBo9F+njGUgMn3UHQycreWHB8XsStvvAh1AthbI8/4IpKnFaF58F+iSiHejYOlMQ/eq83g==",
+      "license": "MIT",
+      "dependencies": {
+        "@next/env": "16.3.3",
+        "@swc/helpers": "0.5.23",
+        "baseline-browser-mapping": "^2.9.19",
+        "caniuse-lite": "^1.0.30001579",
+        "postcss": "8.5.23",
+        "styled-jsx": "5.1.6"
+      },
+      "bin": {
+        "next": "dist/bin/next"
+      },
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "optionalDependencies": {
+        "@next/swc-darwin-arm64": "16.3.3",
+        "@next/swc-darwin-x64": "16.3.3",
+        "@next/swc-linux-arm64-gnu": "16.3.3",
+        "@next/swc-linux-arm64-musl": "16.3.3",
+        "@next/swc-linux-x64-gnu": "16.3.3",
+        "@next/swc-linux-x64-musl": "16.3.3",
+        "@next/swc-win32-arm64-msvc": "16.3.3",
+        "@next/swc-win32-x64-msvc": "16.3.3",
+        "sharp": "^0.35.3"
+      },
+      "peerDependencies": {
+        "@opentelemetry/api": "^1.1.0",
+        "@playwright/test": "^1.51.1",
+        "babel-plugin-react-compiler": "*",
+        "react": "^18.2.0 || 19.0.0-rc-de68d2f4-20241204 || ^19.0.0",
+        "react-dom": "^18.2.0 || 19.0.0-rc-de68d2f4-20241204 || ^19.0.0",
+        "sass": "^1.3.0"
+      },
+      "peerDependenciesMeta": {
+        "@opentelemetry/api": {
+          "optional": true
+        },
+        "@playwright/test": {
+          "optional": true
+        },
+        "babel-plugin-react-compiler": {
+          "optional": true
+        },
+        "sass": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/on-exit-leak-free": {
       "version": "2.1.2",
       "resolved": "https://registry.npmjs.org/on-exit-leak-free/-/on-exit-leak-free-2.1.2.tgz",
@@ -449,6 +1345,12 @@
         "node": ">=14.0.0"
       }
     },
+    "node_modules/picocolors": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/picocolors/-/picocolors-1.1.1.tgz",
+      "integrity": "sha512-xceH2snhtb5M9liqDsmEw56le376mTZkEX/jEb/RxNFyegNul7eNslCXP9FDj/Lcu0X8KEyMceP2ntpaHrDEVA==",
+      "license": "ISC"
+    },
     "node_modules/pino": {
       "version": "10.3.1",
       "resolved": "https://registry.npmjs.org/pino/-/pino-10.3.1.tgz",
@@ -486,6 +1388,66 @@
       "integrity": "sha512-BndPH67/JxGExRgiX1dX0w1FvZck5Wa4aal9198SrRhZjH3GxKQUKIBnYJTdj2HDN3UQAS06HlfcSbQj2OHmaw==",
       "license": "MIT"
     },
+    "node_modules/playwright": {
+      "version": "1.62.1",
+      "resolved": "https://registry.npmjs.org/playwright/-/playwright-1.62.1.tgz",
+      "integrity": "sha512-0M+L3LAD8/nm554LOla9Ayx0j0tmFZ0FBcoQ7F1VuVHpM/XpiC8RcDzBQB8W5+hA8L22THxELzeF+2WcUzvcLg==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "dependencies": {
+        "playwright-core": "1.62.1"
+      },
+      "bin": {
+        "playwright": "cli.js"
+      },
+      "engines": {
+        "node": ">=20"
+      },
+      "optionalDependencies": {
+        "fsevents": "2.3.2"
+      }
+    },
+    "node_modules/playwright-core": {
+      "version": "1.62.1",
+      "resolved": "https://registry.npmjs.org/playwright-core/-/playwright-core-1.62.1.tgz",
+      "integrity": "sha512-wPYSwEBJY9GHraISXqyqtx0na0LpO3XEX7jNDhntbex7tzUS7kLnZsOlFruFJB4Hi/rhDMjXGqHewDZ68nYZVw==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "bin": {
+        "playwright-core": "cli.js"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/postcss": {
+      "version": "8.5.23",
+      "resolved": "https://registry.npmjs.org/postcss/-/postcss-8.5.23.tgz",
+      "integrity": "sha512-g50586zr4bZmwFiTlflMu8E0bDTb5I5gertgwAKmsdUlTQIhZtunzUlD1WSzwcVWPoAVpsrA6vlfCD7oXvRwgg==",
+      "funding": [
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/postcss/"
+        },
+        {
+          "type": "tidelift",
+          "url": "https://tidelift.com/funding/github/npm/postcss"
+        },
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/ai"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "nanoid": "^3.3.16",
+        "picocolors": "^1.1.1",
+        "source-map-js": "^1.2.1"
+      },
+      "engines": {
+        "node": "^10 || ^12 || >=14"
+      }
+    },
     "node_modules/process-warning": {
       "version": "5.1.0",
       "resolved": "https://registry.npmjs.org/process-warning/-/process-warning-5.1.0.tgz",
@@ -508,6 +1470,27 @@
       "integrity": "sha512-tYC1Q1hgyRuHgloV/YXs2w15unPVh8qfu/qCTfhTYamaw7fyhumKa2yGpdSo87vY32rIclj+4fWYQXUMs9EHvg==",
       "license": "MIT"
     },
+    "node_modules/react": {
+      "version": "19.2.8",
+      "resolved": "https://registry.npmjs.org/react/-/react-19.2.8.tgz",
+      "integrity": "sha512-PWaYA1L/q9u2u7xYQi+Y3L3Yfnie7XyLeaJICV1MGD6LprsBxcAqGjYyr0eY3p+QdsA+x/Irkt4Qif8D63+Sbw==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
+    "node_modules/react-dom": {
+      "version": "19.2.8",
+      "resolved": "https://registry.npmjs.org/react-dom/-/react-dom-19.2.8.tgz",
+      "integrity": "sha512-rVprimfGBG3DR+Tq0IQG2DT5PxKth1WIGDmj5yPmlzr4YBe7uyE+Du4oVqTDXZSHGGGXRtTJEGSSePyQCMBglQ==",
+      "license": "MIT",
+      "dependencies": {
+        "scheduler": "^0.27.0"
+      },
+      "peerDependencies": {
+        "react": "^19.2.8"
+      }
+    },
     "node_modules/real-require": {
       "version": "0.2.0",
       "resolved": "https://registry.npmjs.org/real-require/-/real-require-0.2.0.tgz",
@@ -582,6 +1565,12 @@
         "node": ">=10"
       }
     },
+    "node_modules/scheduler": {
+      "version": "0.27.0",
+      "resolved": "https://registry.npmjs.org/scheduler/-/scheduler-0.27.0.tgz",
+      "integrity": "sha512-eNv+WrVbKu1f3vbYJT/xtiF5syA5HPIMtf9IgY/nKg0sWqzAUEvqY/xm7OcZc/qafLx/iO9FgOmeSAp4v5ti/Q==",
+      "license": "MIT"
+    },
     "node_modules/secure-json-parse": {
       "version": "4.1.0",
       "resolved": "https://registry.npmjs.org/secure-json-parse/-/secure-json-parse-4.1.0.tgz",
@@ -616,6 +1605,56 @@
       "integrity": "sha512-oeM1lpU/UvhTxw+g3cIfxXHyJRc/uidd3yK1P242gzHds0udQBYzs3y8j4gCCW+ZJ7ad0yctld8RYO+bdurlvw==",
       "license": "MIT"
     },
+    "node_modules/sharp": {
+      "version": "0.35.4",
+      "resolved": "https://registry.npmjs.org/sharp/-/sharp-0.35.4.tgz",
+      "integrity": "sha512-n++8XWcj+jCOr2IOl7h8LbKnGBDY4aPbmprMONBNFdn0ImXqpGVv5zliDs0V9HbmbCQLpbuo2ej9rAoOQTvMDA==",
+      "license": "Apache-2.0",
+      "optional": true,
+      "dependencies": {
+        "@img/colour": "^1.1.0",
+        "detect-libc": "^2.1.2",
+        "semver": "^7.8.5"
+      },
+      "engines": {
+        "node": ">=20.9.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/libvips"
+      },
+      "optionalDependencies": {
+        "@img/sharp-darwin-arm64": "0.35.4",
+        "@img/sharp-darwin-x64": "0.35.4",
+        "@img/sharp-freebsd-wasm32": "0.35.4",
+        "@img/sharp-libvips-darwin-arm64": "1.3.3",
+        "@img/sharp-libvips-darwin-x64": "1.3.3",
+        "@img/sharp-libvips-linux-arm": "1.3.3",
+        "@img/sharp-libvips-linux-arm64": "1.3.3",
+        "@img/sharp-libvips-linux-ppc64": "1.3.3",
+        "@img/sharp-libvips-linux-riscv64": "1.3.3",
+        "@img/sharp-libvips-linux-s390x": "1.3.3",
+        "@img/sharp-libvips-linux-x64": "1.3.3",
+        "@img/sharp-libvips-linuxmusl-arm64": "1.3.3",
+        "@img/sharp-libvips-linuxmusl-x64": "1.3.3",
+        "@img/sharp-linux-arm": "0.35.4",
+        "@img/sharp-linux-arm64": "0.35.4",
+        "@img/sharp-linux-ppc64": "0.35.4",
+        "@img/sharp-linux-riscv64": "0.35.4",
+        "@img/sharp-linux-s390x": "0.35.4",
+        "@img/sharp-linux-x64": "0.35.4",
+        "@img/sharp-linuxmusl-arm64": "0.35.4",
+        "@img/sharp-linuxmusl-x64": "0.35.4",
+        "@img/sharp-webcontainers-wasm32": "0.35.4",
+        "@img/sharp-win32-arm64": "0.35.4",
+        "@img/sharp-win32-ia32": "0.35.4",
+        "@img/sharp-win32-x64": "0.35.4"
+      },
+      "peerDependenciesMeta": {
+        "@types/node": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/sonic-boom": {
       "version": "4.2.1",
       "resolved": "https://registry.npmjs.org/sonic-boom/-/sonic-boom-4.2.1.tgz",
@@ -625,6 +1664,15 @@
         "atomic-sleep": "^1.0.0"
       }
     },
+    "node_modules/source-map-js": {
+      "version": "1.2.1",
+      "resolved": "https://registry.npmjs.org/source-map-js/-/source-map-js-1.2.1.tgz",
+      "integrity": "sha512-UXWMKhLOwVKb728IUtQPXxfYU+usdybtUrK/8uGE8CQMvrhOpwvzDBwj0QhSL7MQc7vIsISBG8VQ8+IDQxpfQA==",
+      "license": "BSD-3-Clause",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/split2": {
       "version": "4.2.0",
       "resolved": "https://registry.npmjs.org/split2/-/split2-4.2.0.tgz",
@@ -634,6 +1682,29 @@
         "node": ">= 10.x"
       }
     },
+    "node_modules/styled-jsx": {
+      "version": "5.1.6",
+      "resolved": "https://registry.npmjs.org/styled-jsx/-/styled-jsx-5.1.6.tgz",
+      "integrity": "sha512-qSVyDTeMotdvQYoHWLNGwRFJHC+i+ZvdBRYosOFgC+Wg1vx4frN2/RG/NA7SYqqvKNLf39P2LSRA2pu6n0XYZA==",
+      "license": "MIT",
+      "dependencies": {
+        "client-only": "0.0.1"
+      },
+      "engines": {
+        "node": ">= 12.0.0"
+      },
+      "peerDependencies": {
+        "react": ">= 16.8.0 || 17.x.x || ^18.0.0-0 || ^19.0.0-0"
+      },
+      "peerDependenciesMeta": {
+        "@babel/core": {
+          "optional": true
+        },
+        "babel-plugin-macros": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/thread-stream": {
       "version": "4.2.0",
       "resolved": "https://registry.npmjs.org/thread-stream/-/thread-stream-4.2.0.tgz",
@@ -661,6 +1732,12 @@
         "node": ">=20"
       }
     },
+    "node_modules/tslib": {
+      "version": "2.8.1",
+      "resolved": "https://registry.npmjs.org/tslib/-/tslib-2.8.1.tgz",
+      "integrity": "sha512-oJFu94HQb+KVduSUQL7wnpmqnfmLsOA/nAh6b6EH0wCEoK0/mPeXU6c3wKDV83MkOuHPRHtSXKKU99IBazS/2w==",
+      "license": "0BSD"
+    },
     "node_modules/typescript": {
       "version": "5.9.3",
       "resolved": "https://registry.npmjs.org/typescript/-/typescript-5.9.3.tgz",
diff --git a/package.json b/package.json
index 0271dbd..6066d8b 100644
--- a/package.json
+++ b/package.json
@@ -8,12 +8,27 @@
   "scripts": {
     "dev:api": "node --watch server/main.ts",
     "start:api": "node server/main.ts",
+    "dev:web": "NEXT_TELEMETRY_DISABLED=1 next dev --hostname 127.0.0.1 --port 4313",
+    "build": "NEXT_TELEMETRY_DISABLED=1 next build",
+    "start:web": "NEXT_TELEMETRY_DISABLED=1 next start --hostname 127.0.0.1 --port 4313",
     "fixture": "node test/fixture.ts",
-    "typecheck": "tsc --noEmit",
+    "typecheck": "NEXT_TELEMETRY_DISABLED=1 next typegen && tsc --noEmit",
     "test:unit": "node --test test/unit.test.ts",
     "test:functional": "node --test test/functional.test.ts",
+    "test:e2e": "playwright test",
     "test": "npm run test:unit && npm run test:functional"
   },
-  "dependencies": { "fastify": "5.12.1" },
-  "devDependencies": { "@types/node": "24.13.3", "typescript": "5.9.3" }
+  "dependencies": {
+    "fastify": "5.12.1",
+    "next": "16.3.3",
+    "react": "19.2.8",
+    "react-dom": "19.2.8"
+  },
+  "devDependencies": {
+    "@playwright/test": "1.62.1",
+    "@types/node": "24.13.3",
+    "@types/react": "19.2.18",
+    "@types/react-dom": "19.2.5",
+    "typescript": "5.9.3"
+  }
 }
diff --git a/playwright.config.ts b/playwright.config.ts
new file mode 100644
index 0000000..21028f2
--- /dev/null
+++ b/playwright.config.ts
@@ -0,0 +1,19 @@
+import { defineConfig, devices } from '@playwright/test';
+
+export default defineConfig({
+  testDir: './test/browser',
+  outputDir: './output/playwright',
+  fullyParallel: false,
+  workers: 1,
+  retries: 0,
+  timeout: 30_000,
+  reporter: 'list',
+  use: { baseURL: 'http://127.0.0.1:4313', trace: 'retain-on-failure' },
+  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
+  webServer: [
+    { command: 'npm run fixture', url: 'http://127.0.0.1:4311/ok', reuseExistingServer: false, timeout: 30_000 },
+    { command: 'npm run start:api', url: 'http://127.0.0.1:4312/health', reuseExistingServer: false, timeout: 30_000 },
+    { command: 'npm run dev:web', url: 'http://127.0.0.1:4313/monitors', reuseExistingServer: false, timeout: 90_000,
+      env: { NEXT_TELEMETRY_DISABLED: '1' } },
+  ],
+});
diff --git a/test/browser/monitor.spec.ts b/test/browser/monitor.spec.ts
new file mode 100644
index 0000000..98dfe49
--- /dev/null
+++ b/test/browser/monitor.spec.ts
@@ -0,0 +1,22 @@
+import { expect, test } from '@playwright/test';
+
+test('create Fixture monitor, run one manual check, and display the terminal result', async ({ page }) => {
+  await page.goto('/monitors');
+  await page.getByLabel('Name', { exact: true }).fill('Fixture monitor');
+  await page.getByLabel('Endpoint URL').fill('http://127.0.0.1:4311/ok');
+  await page.getByLabel('Interval (seconds)').fill('60');
+  await page.getByLabel('Enabled', { exact: true }).check();
+  await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+
+  const monitor = page.getByRole('article', { name: 'Fixture monitor', exact: true });
+  await expect(monitor).toContainText('No checks yet.');
+  await monitor.getByRole('button', { name: 'Run check', exact: true }).click();
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+  await expect(monitor.getByText('200', { exact: true })).toBeVisible();
+  await expect(monitor).toContainText('Enabled · 60 seconds');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
+  await page.screenshot({ path: 'output/playwright/E01-success.png', fullPage: true });
+
+  await page.reload();
+  await expect(page.getByRole('article', { name: 'Fixture monitor' }).getByText('SUCCEEDED', { exact: true })).toBeVisible();
+});
diff --git a/tsconfig.json b/tsconfig.json
index dc359f7..e82b908 100644
--- a/tsconfig.json
+++ b/tsconfig.json
@@ -1,12 +1,41 @@
 {
   "compilerOptions": {
     "target": "ES2022",
-    "module": "NodeNext",
-    "moduleResolution": "NodeNext",
+    "lib": [
+      "dom",
+      "dom.iterable",
+      "esnext"
+    ],
+    "module": "esnext",
+    "moduleResolution": "bundler",
     "strict": true,
     "noEmit": true,
     "allowImportingTsExtensions": true,
-    "skipLibCheck": true
+    "skipLibCheck": true,
+    "esModuleInterop": true,
+    "resolveJsonModule": true,
+    "isolatedModules": true,
+    "jsx": "react-jsx",
+    "incremental": true,
+    "plugins": [
+      {
+        "name": "next"
+      }
+    ],
+    "allowJs": true
   },
-  "include": ["server/**/*.ts", "test/**/*.ts"]
+  "include": [
+    "next-env.d.ts",
+    "next.config.ts",
+    "playwright.config.ts",
+    "app/**/*.ts",
+    "app/**/*.tsx",
+    "server/**/*.ts",
+    "test/**/*.ts",
+    ".next/types/**/*.ts",
+    ".next/dev/types/**/*.ts"
+  ],
+  "exclude": [
+    "node_modules"
+  ]
 }


