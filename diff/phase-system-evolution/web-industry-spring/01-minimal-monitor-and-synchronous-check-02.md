## `브라우저에서 Monitor 생성과 Check 결과 표시`

diff --git a/.gitignore b/.gitignore
index 43bf81a..f83bf47 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,5 +1,6 @@
 node_modules/
 .next/
+next-env.d.ts
 .m2/
 .npm-cache/
 backend/target/
diff --git a/.node-version b/.node-version
new file mode 100644
index 0000000..60ade1a
--- /dev/null
+++ b/.node-version
@@ -0,0 +1 @@
+24.19.0
diff --git a/.npmrc b/.npmrc
new file mode 100644
index 0000000..3b4bfa0
--- /dev/null
+++ b/.npmrc
@@ -0,0 +1,3 @@
+save-exact=true
+engine-strict=true
+fetch-retries=0
diff --git a/AGENTS.md b/AGENTS.md
new file mode 100644
index 0000000..643577d
--- /dev/null
+++ b/AGENTS.md
@@ -0,0 +1,9 @@
+<!-- BEGIN:nextjs-agent-rules -->
+
+# This is NOT the Next.js you know
+
+This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.
+
+This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.
+
+<!-- END:nextjs-agent-rules -->
diff --git a/CLAUDE.md b/CLAUDE.md
new file mode 100644
index 0000000..43c994c
--- /dev/null
+++ b/CLAUDE.md
@@ -0,0 +1 @@
+@AGENTS.md
diff --git a/app/layout.tsx b/app/layout.tsx
new file mode 100644
index 0000000..bc03a8e
--- /dev/null
+++ b/app/layout.tsx
@@ -0,0 +1,8 @@
+import type { Metadata } from 'next';
+import './style.css';
+
+export const metadata: Metadata = { title: 'Endpoint Monitor', description: 'Local fixture monitoring' };
+
+export default function Layout({ children }: Readonly<{ children: React.ReactNode }>) {
+  return <html lang="en"><body>{children}</body></html>;
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
new file mode 100644
index 0000000..2bd0959
--- /dev/null
+++ b/app/monitors/page.tsx
@@ -0,0 +1,95 @@
+'use client';
+
+import { useEffect, useState, type FormEvent } from 'react';
+
+type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
+type CheckRun = {
+  id: string; monitorId: string; state: 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
+  latencyMs: number; failureReason: string | null; finishedAt: string;
+};
+type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
+
+export default function Monitors() {
+  const [monitors, setMonitors] = useState<MonitorView[]>([]);
+  const [error, setError] = useState('');
+  const [busy, setBusy] = useState(false);
+
+  useEffect(() => {
+    fetch('/api/monitors').then(async response => {
+      if (!response.ok) throw new Error('Could not load monitors.');
+      setMonitors(await response.json());
+    }).catch(error => setError(error.message));
+  }, []);
+
+  async function create(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    const fields = new FormData(form);
+    setBusy(true);
+    setError('');
+    try {
+      const response = await fetch('/api/monitors', {
+        method: 'POST', headers: { 'Content-Type': 'application/json' },
+        body: JSON.stringify({ name: fields.get('name'), url: fields.get('url'),
+          interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on' }),
+      });
+      if (!response.ok) throw new Error(`Monitor could not be created (HTTP ${response.status}). Use the configured local fixture.`);
+      const created: MonitorView = await response.json();
+      setMonitors(current => [...current, created]);
+      form.reset();
+    } catch (error) {
+      setError(error instanceof Error ? error.message : 'Monitor could not be created.');
+    } finally {
+      setBusy(false);
+    }
+  }
+
+  async function check(id: string) {
+    setBusy(true);
+    setError('');
+    try {
+      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
+      if (!response.ok) throw new Error(`Check could not be completed (HTTP ${response.status}).`);
+      const latestCheck: CheckRun = await response.json();
+      setMonitors(current => current.map(row => row.monitor.id === id ? { ...row, latestCheck } : row));
+    } catch (error) {
+      setError(error instanceof Error ? error.message : 'Check could not be completed.');
+    } finally {
+      setBusy(false);
+    }
+  }
+
+  return <main>
+    <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
+      <p>Create a monitor and check its HTTP response.</p></header>
+    <aside>Development fixture only. Data is held in memory and disappears when the API restarts.</aside>
+    <section aria-labelledby="create-title">
+      <h2 id="create-title">Create monitor</h2>
+      <form onSubmit={create}>
+        <label>Name<input name="name" required defaultValue="Fixture monitor" /></label>
+        <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
+        <label>Interval (seconds)<input name="interval" type="number" min="1" required defaultValue="60" /></label>
+        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
+        <p className="hint">Interval and enabled are stored; E01 runs checks manually.</p>
+        <button disabled={busy}>Create monitor</button>
+      </form>
+    </section>
+    {error && <p role="alert" className="error">{error}</p>}
+    <section aria-labelledby="monitors-title"><h2 id="monitors-title">Monitors</h2>
+      {monitors.length === 0 && <p>No monitors yet.</p>}
+      {monitors.map(({ monitor, latestCheck }) => <article key={monitor.id} aria-label={monitor.name}>
+        <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
+        <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
+        <button disabled={busy} onClick={() => check(monitor.id)}>Run check</button>
+        <div aria-live="polite" className="result">
+          {latestCheck ? <>
+            <strong>{latestCheck.state}</strong>
+            <span>HTTP {latestCheck.httpStatus ?? '—'}</span>
+            <span>{latestCheck.latencyMs} ms</span>
+            {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
+          </> : <span>No checks yet.</span>}
+        </div>
+      </article>)}
+    </section>
+  </main>;
+}
diff --git a/app/page.tsx b/app/page.tsx
new file mode 100644
index 0000000..2cd2755
--- /dev/null
+++ b/app/page.tsx
@@ -0,0 +1,5 @@
+import { redirect } from 'next/navigation';
+
+export default function Home() {
+  redirect('/monitors');
+}
diff --git a/app/style.css b/app/style.css
new file mode 100644
index 0000000..b77730e
--- /dev/null
+++ b/app/style.css
@@ -0,0 +1,24 @@
+:root { color-scheme: light; font-family: system-ui, sans-serif; background: #f6f7f9; color: #172235; }
+* { box-sizing: border-box; }
+body { margin: 0; }
+main { max-width: 820px; margin: 0 auto; padding: 48px 24px; }
+h1 { margin: 8px 0; font-size: 2.2rem; }
+h2 { font-size: 1.2rem; }
+h3 { margin: 0; }
+.eyebrow { color: #456080; font-size: .75rem; font-weight: 700; letter-spacing: .12em; }
+aside { padding: 16px; margin: 24px 0; background: #e8edf4; border-radius: 8px; }
+section { margin: 32px 0; }
+form, article { background: white; border: 1px solid #d7dfe9; padding: 24px; border-radius: 8px; }
+form { display: grid; gap: 16px; }
+label { display: grid; gap: 6px; font-size: .9rem; font-weight: 600; }
+input { width: 100%; padding: 10px 12px; border: 1px solid #91a2b9; border-radius: 4px; font: inherit; }
+.checkbox { display: flex; align-items: center; }
+.checkbox input { width: auto; }
+button { width: fit-content; padding: 10px 18px; color: white; background: #245a97; border: 0; border-radius: 5px; font: inherit; cursor: pointer; }
+button:disabled { opacity: .55; cursor: wait; }
+button:focus-visible, input:focus-visible { outline: 3px solid #ae741b; outline-offset: 3px; }
+.hint { margin: 0; color: #52617a; font-size: .85rem; }
+.error { color: #9b1c24; }
+article { margin: 16px 0; }
+.url { overflow-wrap: anywhere; color: #52617a; }
+.result { display: flex; flex-wrap: wrap; gap: 16px; margin-top: 16px; padding-top: 16px; border-top: 1px solid #e3e8ef; }
diff --git a/next.config.mjs b/next.config.mjs
new file mode 100644
index 0000000..7a1e01e
--- /dev/null
+++ b/next.config.mjs
@@ -0,0 +1,8 @@
+/** @type {import('next').NextConfig} */
+const config = {
+  poweredByHeader: false,
+  async rewrites() {
+    return [{ source: '/api/:path*', destination: `${process.env.API_ORIGIN ?? 'http://127.0.0.1:4322'}/api/:path*` }];
+  },
+};
+export default config;
diff --git a/package-lock.json b/package-lock.json
new file mode 100644
index 0000000..f1908cb
--- /dev/null
+++ b/package-lock.json
@@ -0,0 +1,1143 @@
+{
+  "name": "industry-spring-monitor",
+  "version": "0.0.1",
+  "lockfileVersion": 3,
+  "requires": true,
+  "packages": {
+    "": {
+      "name": "industry-spring-monitor",
+      "version": "0.0.1",
+      "dependencies": {
+        "next": "16.3.3",
+        "react": "19.2.8",
+        "react-dom": "19.2.8"
+      },
+      "devDependencies": {
+        "@playwright/test": "1.62.1",
+        "@types/node": "24.10.1",
+        "@types/react": "19.2.18",
+        "@types/react-dom": "19.2.5",
+        "typescript": "5.9.3"
+      },
+      "engines": {
+        "node": "24.19.0",
+        "npm": "11.17.0"
+      }
+    },
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
+    "node_modules/@types/node": {
+      "version": "24.10.1",
+      "resolved": "https://registry.npmjs.org/@types/node/-/node-24.10.1.tgz",
+      "integrity": "sha512-GNWcUTRBgIRJD5zj+Tq0fKOJ5XZajIiBroOF0yvj2bSU1WvNdYS/dn9UxwsujGW4JX06dnHyjV2y9rRaybH0iQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "undici-types": "~7.16.0"
+      }
+    },
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
+    "node_modules/csstype": {
+      "version": "3.2.3",
+      "resolved": "https://registry.npmjs.org/csstype/-/csstype-3.2.3.tgz",
+      "integrity": "sha512-z1HGKcYy2xA8AGQfwrn0PAy+PB7X/GSj3UVJW9qKyn43xWa+gl5nXmU4qqLMRzWVLFC8KusUX8T/0kCiOYpAIQ==",
+      "dev": true,
+      "license": "MIT"
+    },
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
+    "node_modules/picocolors": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/picocolors/-/picocolors-1.1.1.tgz",
+      "integrity": "sha512-xceH2snhtb5M9liqDsmEw56le376mTZkEX/jEb/RxNFyegNul7eNslCXP9FDj/Lcu0X8KEyMceP2ntpaHrDEVA==",
+      "license": "ISC"
+    },
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
+    "node_modules/scheduler": {
+      "version": "0.27.0",
+      "resolved": "https://registry.npmjs.org/scheduler/-/scheduler-0.27.0.tgz",
+      "integrity": "sha512-eNv+WrVbKu1f3vbYJT/xtiF5syA5HPIMtf9IgY/nKg0sWqzAUEvqY/xm7OcZc/qafLx/iO9FgOmeSAp4v5ti/Q==",
+      "license": "MIT"
+    },
+    "node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "license": "ISC",
+      "optional": true,
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
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
+    "node_modules/source-map-js": {
+      "version": "1.2.1",
+      "resolved": "https://registry.npmjs.org/source-map-js/-/source-map-js-1.2.1.tgz",
+      "integrity": "sha512-UXWMKhLOwVKb728IUtQPXxfYU+usdybtUrK/8uGE8CQMvrhOpwvzDBwj0QhSL7MQc7vIsISBG8VQ8+IDQxpfQA==",
+      "license": "BSD-3-Clause",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
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
+    "node_modules/tslib": {
+      "version": "2.8.1",
+      "resolved": "https://registry.npmjs.org/tslib/-/tslib-2.8.1.tgz",
+      "integrity": "sha512-oJFu94HQb+KVduSUQL7wnpmqnfmLsOA/nAh6b6EH0wCEoK0/mPeXU6c3wKDV83MkOuHPRHtSXKKU99IBazS/2w==",
+      "license": "0BSD"
+    },
+    "node_modules/typescript": {
+      "version": "5.9.3",
+      "resolved": "https://registry.npmjs.org/typescript/-/typescript-5.9.3.tgz",
+      "integrity": "sha512-jl1vZzPDinLr9eUt3J/t7V6FgNEw9QjvBPdysz9KfQDD41fQrC2Y4vKQdiaUpFT4bXlb1RHhLpp8wtm6M5TgSw==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "bin": {
+        "tsc": "bin/tsc",
+        "tsserver": "bin/tsserver"
+      },
+      "engines": {
+        "node": ">=14.17"
+      }
+    },
+    "node_modules/undici-types": {
+      "version": "7.16.0",
+      "resolved": "https://registry.npmjs.org/undici-types/-/undici-types-7.16.0.tgz",
+      "integrity": "sha512-Zz+aZWSj8LE6zoxD+xrjh4VfkIG8Ya6LvYkZqtUQGJPZjYl53ypCaUwWqo7eI0x66KBGeRo+mlBEkMSeSZ38Nw==",
+      "dev": true,
+      "license": "MIT"
+    }
+  }
+}
diff --git a/package.json b/package.json
new file mode 100644
index 0000000..99ff394
--- /dev/null
+++ b/package.json
@@ -0,0 +1,31 @@
+{
+  "name": "industry-spring-monitor",
+  "version": "0.0.1",
+  "private": true,
+  "packageManager": "npm@11.17.0",
+  "engines": { "node": "24.19.0", "npm": "11.17.0" },
+  "scripts": {
+    "dev": "next dev --hostname 127.0.0.1 --port 4323",
+    "build": "next build --webpack",
+    "start": "next start --hostname 127.0.0.1 --port 4323",
+    "fixture": "node scripts/fixture.mjs",
+    "api:dev": "mvn -B -ntp -f backend/pom.xml spring-boot:run",
+    "api:package": "mvn -B -ntp -f backend/pom.xml -DskipTests package",
+    "test:api": "mvn -B -ntp -f backend/pom.xml test",
+    "typecheck": "next typegen && tsc --noEmit",
+    "test:e2e": "playwright test",
+    "verify": "node scripts/verify.mjs"
+  },
+  "dependencies": {
+    "next": "16.3.3",
+    "react": "19.2.8",
+    "react-dom": "19.2.8"
+  },
+  "devDependencies": {
+    "@playwright/test": "1.62.1",
+    "@types/node": "24.10.1",
+    "@types/react": "19.2.18",
+    "@types/react-dom": "19.2.5",
+    "typescript": "5.9.3"
+  }
+}
diff --git a/playwright.config.ts b/playwright.config.ts
new file mode 100644
index 0000000..befb9d7
--- /dev/null
+++ b/playwright.config.ts
@@ -0,0 +1,16 @@
+import { defineConfig, devices } from '@playwright/test';
+
+export default defineConfig({
+  testDir: './tests/browser',
+  fullyParallel: false,
+  workers: 1,
+  retries: 0,
+  timeout: 30_000,
+  reporter: [['list'], ['json', { outputFile: 'test-results/browser.json' }]],
+  use: { ...devices['Desktop Chrome'], baseURL: 'http://127.0.0.1:4323', trace: 'retain-on-failure' },
+  webServer: [
+    { command: 'node scripts/fixture.mjs', url: 'http://127.0.0.1:4321/ok', reuseExistingServer: false },
+    { command: 'java -jar backend/target/monitor-api-0.0.1.jar', url: 'http://127.0.0.1:4322/api/monitors', reuseExistingServer: false },
+    { command: 'npm run dev', url: 'http://127.0.0.1:4323/monitors', reuseExistingServer: false, timeout: 90_000 },
+  ],
+});
diff --git a/scripts/fixture.mjs b/scripts/fixture.mjs
new file mode 100644
index 0000000..d485b24
--- /dev/null
+++ b/scripts/fixture.mjs
@@ -0,0 +1,14 @@
+import { createServer } from 'node:http';
+
+const port = Number(process.env.FIXTURE_PORT ?? 4321);
+const server = createServer((request, response) => {
+  response.setHeader('Content-Type', 'text/plain');
+  switch (request.url) {
+    case '/ok': response.writeHead(200).end('ok\n'); break;
+    case '/fail': response.writeHead(503).end('unavailable\n'); break;
+    case '/redirect': response.writeHead(302, { Location: '/ok' }).end(); break;
+    default: response.writeHead(404).end('not found\n');
+  }
+});
+server.listen(port, '127.0.0.1', () => console.log(`Fixture http://127.0.0.1:${port}`));
+for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => server.close());
diff --git a/scripts/verify.mjs b/scripts/verify.mjs
new file mode 100644
index 0000000..534a89d
--- /dev/null
+++ b/scripts/verify.mjs
@@ -0,0 +1,19 @@
+import { spawnSync } from 'node:child_process';
+import { appendFileSync, mkdirSync } from 'node:fs';
+
+// No retries and no performance scenarios. Every invocation is recorded, including failures.
+mkdirSync('output/verification', { recursive: true });
+const commands = [
+  ['mvn', ['-B', '-ntp', '-f', 'backend/pom.xml', 'package']],
+  ['npm', ['run', 'typecheck']],
+  ['npm', ['run', 'build']],
+  ['npm', ['run', 'test:e2e']],
+];
+for (const [command, args] of commands) {
+  const started = Date.now();
+  const result = spawnSync(command, args, { stdio: 'inherit', env: { ...process.env, NEXT_TELEMETRY_DISABLED: '1' } });
+  const entry = { command: [command, ...args].join(' '), startedAt: new Date(started).toISOString(),
+    elapsedSeconds: (Date.now() - started) / 1000, exitCode: result.status, error: result.error?.message };
+  appendFileSync('output/verification/runs.jsonl', `${JSON.stringify(entry)}\n`);
+  if (result.status !== 0) process.exit(result.status ?? 1);
+}
diff --git a/tests/browser/monitor.spec.ts b/tests/browser/monitor.spec.ts
new file mode 100644
index 0000000..c28f8b2
--- /dev/null
+++ b/tests/browser/monitor.spec.ts
@@ -0,0 +1,29 @@
+import { test, expect } from '@playwright/test';
+
+test('create fixture monitor and display successful synchronous check', async ({ page }) => {
+  await page.goto('/monitors');
+  await page.getByLabel('Name', { exact: true }).fill('Fixture monitor');
+  await page.getByLabel('URL', { exact: true }).fill('http://127.0.0.1:4321/ok');
+  await page.getByLabel('Interval (seconds)').fill('60');
+  await page.getByLabel('Enabled', { exact: true }).check();
+  await page.getByRole('button', { name: 'Create monitor' }).click();
+  const monitor = page.getByRole('article', { name: 'Fixture monitor', exact: true });
+  await expect(monitor).toBeVisible();
+  await monitor.getByRole('button', { name: 'Run check' }).click();
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+  await expect(monitor.getByText('HTTP 200', { exact: true })).toBeVisible();
+  await page.reload();
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+});
+
+test('display HTTP failure as a completed check result', async ({ page }) => {
+  await page.goto('/monitors');
+  await page.getByLabel('Name', { exact: true }).fill('Unavailable fixture');
+  await page.getByLabel('URL', { exact: true }).fill('http://127.0.0.1:4321/fail');
+  await page.getByRole('button', { name: 'Create monitor' }).click();
+  const monitor = page.getByRole('article', { name: 'Unavailable fixture', exact: true });
+  await monitor.getByRole('button', { name: 'Run check' }).click();
+  await expect(monitor.getByText('FAILED', { exact: true })).toBeVisible();
+  await expect(monitor.getByText('HTTP 503', { exact: true })).toBeVisible();
+  await expect(monitor.getByText('HTTP_STATUS', { exact: true })).toBeVisible();
+});
diff --git a/tsconfig.json b/tsconfig.json
new file mode 100644
index 0000000..3169e2f
--- /dev/null
+++ b/tsconfig.json
@@ -0,0 +1,38 @@
+{
+  "compilerOptions": {
+    "target": "ES2022",
+    "lib": [
+      "dom",
+      "dom.iterable",
+      "esnext"
+    ],
+    "allowJs": false,
+    "skipLibCheck": true,
+    "strict": true,
+    "noEmit": true,
+    "esModuleInterop": true,
+    "module": "esnext",
+    "moduleResolution": "bundler",
+    "resolveJsonModule": true,
+    "isolatedModules": true,
+    "jsx": "react-jsx",
+    "incremental": true,
+    "plugins": [
+      {
+        "name": "next"
+      }
+    ]
+  },
+  "include": [
+    "next-env.d.ts",
+    "app/**/*.ts",
+    "app/**/*.tsx",
+    "tests/**/*.ts",
+    "playwright.config.ts",
+    ".next/types/**/*.ts",
+    ".next/dev/types/**/*.ts"
+  ],
+  "exclude": [
+    "node_modules"
+  ]
+}


