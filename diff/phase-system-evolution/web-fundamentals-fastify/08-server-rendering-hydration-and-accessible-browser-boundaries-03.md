## `test: verify E08 production rendering and browser boundaries`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index 3db77ca..b9e79ee 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -35,10 +35,8 @@ jobs:
         run: npm run test:functional
       - name: Install pinned Chromium
         run: npx playwright install --with-deps chromium
-      - name: Browser E2E
+      - name: Production build and browser E2E
         run: npm run test:e2e
-      - name: Next build
-        run: npm run build
       - name: Stop isolated PostgreSQL
         if: always()
         run: npm run db:down
diff --git a/TRACK.md b/TRACK.md
index 388e036..6d3b353 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -31,6 +31,7 @@ A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
 | PostgreSQL Docker image | 17.6-bookworm |
 | pg | 8.16.3 |
 | @types/pg | 8.15.5 |
+| axe-core (E08 test only) | 4.10.3 |
 
 Direct dependencies are exact in `package.json`; transitives are pinned in
 `package-lock.json`. Node supplies TypeScript stripping and the unit/functional
@@ -466,3 +467,66 @@ open the bare list before explicitly opening history. Its current test consumer
 therefore removes only E07's four history query parameters before those real
 reloads and restores that wrapper in `finally`. The frozen harness, fixtures,
 mutation/barrier/failure assertions and E07's normal URL reload are unchanged.
+
+## E08 authenticated rendering and browser accessibility
+
+`app/monitors/page.tsx` is now a Server Component. Its server-only reader forwards
+only this request's raw Cookie and optional Origin to the trusted Fastify API,
+preserving session validation, ambiguous-cookie rejection, ownership and error
+categories. Request-time `headers()` and explicit `cache: 'no-store'` reads keep
+authenticated HTML/RSC out of shared caches. No cookie, session hash, password or
+CSRF value is passed as a client prop. Session loss between the collection and
+selected-history reads discards both and redirects to `/login` before rendering.
+These boundaries follow the pinned Next [headers](https://nextjs.org/docs/app/api-reference/functions/headers)
+and [fetch](https://nextjs.org/docs/app/api-reference/functions/fetch) contracts.
+
+The existing E06 state owner initializes directly from that public payload;
+hydration does not issue a second collection/history GET or replace data with a
+loading shell. Native URL history selection and explicit mutation refreshes
+remain. Static headings and notices stay on the server; only the login form and
+mutable Monitor workspace are client islands. Browser APIs run in handlers or
+effects. Login/logout replace the document to discard the prior client route
+cache at an account boundary.
+
+Native labeled inputs/buttons remain keyboard operable. Expand controls expose
+their state, loading history announces status, and the history table has a
+caption, column scopes and a labeled keyboard-focusable scroll region. The three
+history states use a native listbox so arrow/Home navigation does not depend on
+opening an operating-system popup. No custom keyboard handler is needed. The only
+new dependency is exact test-only `axe-core@4.10.3`; its browser engine supplies
+the required automated checks without another integration wrapper. All default
+rules run with no exclusions; serious/critical violations must be zero, and
+lower-severity/incomplete findings are retained. The [pinned axe API](https://github.com/dequelabs/axe-core/blob/v4.10.3/doc/API.md)
+documents the scan and result categories.
+
+`npm run test:e2e` now builds Next and runs **all** browser tests against
+`next start`, including E08's frozen Chromium SSR/hydration/privacy/keyboard/axe
+scenario. CI uses that same command. The standalone command
+`node --import ./test/e08-rsc-preload.mjs evidence/E08/run.mjs verification` owns only `e08_browser` and ports
+4311–4313, refuses an occupied 4311–4314, and awaits process/schema cleanup.
+Use both commands under `fnm exec --using 24.19.0`; PostgreSQL must already be up.
+Safe results go to `output/e08`; no raw HTML/RSC, runtime logs, screenshots,
+traces, videos or storage-state are retained. The committed frozen inputs and
+one unchanged-START production failure are in `evidence/E08`.
+
+The frozen raw RSC observer omits Next's framework `_rsc` discriminator. Its
+caller follows only the pinned Next [documented canonicalization](https://nextjs.org/docs/app/guides/cdn-caching)
+once: same origin/path, unchanged user query, only `_rsc` added. Login and all
+other redirects stay unfollowed. No product validation default is disabled.
+
+The E07 delayed-read test now opens its same `limit=3` request through browser
+selection because direct URLs render that history on the server. Its fixed
+rows, held response, FAILED filter, release and exact-ID assertions are unchanged;
+the ordinary URL/back/forward/reload test is unchanged.
+
+E02's browser test consumers also follow the new initial-read boundary. The
+NOT_FOUND case seeds its exact old mock row in the exclusively owned browser
+schema and retains the check-response failure. The initial INTERNAL_ERROR case
+temporarily renames that schema's Monitor table after startup and restores it in
+`finally`, exercising an actual failed SSR read without changing rows or public
+data. Its category/no-article assertions remain; all frozen error/prose unit
+cases are unchanged.
+
+E04's Bob-login consumer reads the actual response body before releasing that
+single response to the browser; document replacement otherwise evicts Chromium's
+response body. The same login status, username and resulting session are checked.
diff --git a/evidence/E08/browser.json b/evidence/E08/browser.json
new file mode 100644
index 0000000..4fbc5b8
--- /dev/null
+++ b/evidence/E08/browser.json
@@ -0,0 +1,225 @@
+{
+  "rscContinuations": [
+    {
+      "status": 307,
+      "sameOriginAndPath": true,
+      "unchangedUserQuery": true,
+      "onlyFrameworkRscAdded": true,
+      "followCount": 1
+    },
+    {
+      "status": 307,
+      "sameOriginAndPath": true,
+      "unchangedUserQuery": true,
+      "onlyFrameworkRscAdded": true,
+      "followCount": 1
+    },
+    {
+      "status": 307,
+      "sameOriginAndPath": true,
+      "unchangedUserQuery": true,
+      "onlyFrameworkRscAdded": true,
+      "followCount": 1
+    }
+  ],
+  "stage": "complete",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 1,
+    "historyIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "monitorNameInHtml": true,
+    "historyIdsInHtml": true,
+    "cacheControl": "private, no-cache, no-store, max-age=0, must-revalidate",
+    "structure": {
+      "main": 1,
+      "headings": [
+        {
+          "level": 1,
+          "text": "Monitors"
+        },
+        {
+          "level": 2,
+          "text": "Create monitor"
+        },
+        {
+          "level": 2,
+          "text": "Your monitors"
+        },
+        {
+          "level": 3,
+          "text": "E08 Monitor A"
+        },
+        {
+          "level": 4,
+          "text": "Check history"
+        }
+      ]
+    }
+  },
+  "hydration": {
+    "identicalInitialMainText": true,
+    "identicalHeadingStructure": true,
+    "initialDataReads": 0,
+    "keyboardHandlerAttached": true,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "privacy": [
+    {
+      "identity": "alice",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": false,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "bob",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "anonymous",
+      "htmlStatus": 307,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    }
+  ],
+  "keyboard": {
+    "pointerActions": 0,
+    "createdMonitors": 1,
+    "tabCounts": [
+      2,
+      5,
+      1
+    ],
+    "openedDetailHistory": true,
+    "selectedSuccessAndAll": true
+  },
+  "accessibility": [
+    {
+      "screen": "login",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Sign in"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 34
+    },
+    {
+      "screen": "list",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Monitors"
+          },
+          {
+            "level": 2,
+            "text": "Create monitor"
+          },
+          {
+            "level": 2,
+            "text": "Your monitors"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor A"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor B"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 35
+    },
+    {
+      "screen": "detail",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Monitors"
+          },
+          {
+            "level": 2,
+            "text": "Create monitor"
+          },
+          {
+            "level": 2,
+            "text": "Your monitors"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor A"
+          },
+          {
+            "level": 4,
+            "text": "Check history"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor B"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [
+        {
+          "id": "color-contrast",
+          "impact": "serious",
+          "nodes": [
+            [
+              "#history-state-08000000-0000-4000-9000-000000000001"
+            ]
+          ]
+        }
+      ],
+      "passedRules": 42
+    }
+  ],
+  "final": {
+    "aliceMonitors": 2,
+    "bobMonitors": 0,
+    "unchangedCheckIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "secretValuesInHtmlOrRsc": 0,
+    "browserStorageEntries": 0,
+    "httpOnlyCookieHidden": true,
+    "serverLogSecrets": "checked by standalone production runner",
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "result": "PASS"
+}
diff --git a/evidence/E08/cleanup.json b/evidence/E08/cleanup.json
new file mode 100644
index 0000000..843c459
--- /dev/null
+++ b/evidence/E08/cleanup.json
@@ -0,0 +1,12 @@
+{
+  "ports": [
+    4311,
+    4312,
+    4313,
+    4314
+  ],
+  "portsFree": true,
+  "residualTestSchemas": [],
+  "databaseReachable": true,
+  "publicSchemaNotMutatedByAudit": true
+}
diff --git a/evidence/E08/production.json b/evidence/E08/production.json
new file mode 100644
index 0000000..6e3fae6
--- /dev/null
+++ b/evidence/E08/production.json
@@ -0,0 +1,267 @@
+{
+  "mode": "verification",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "productionBuild": true,
+  "hashes": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a"
+  },
+  "commands": [
+    {
+      "command": "npm run build (under fnm Node24.19.0)",
+      "exitCode": 0,
+      "durationMs": 3412
+    },
+    {
+      "command": "node test/fixture.ts",
+      "ready": true,
+      "readinessDurationMs": 80
+    },
+    {
+      "command": "node server/main.ts",
+      "ready": true,
+      "readinessDurationMs": 193
+    },
+    {
+      "command": "node node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 4313",
+      "ready": true,
+      "readinessDurationMs": 160
+    }
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "PASS",
+  "portsInitiallyFree": true,
+  "stage": "complete",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 1,
+    "historyIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "monitorNameInHtml": true,
+    "historyIdsInHtml": true,
+    "cacheControl": "private, no-cache, no-store, max-age=0, must-revalidate",
+    "structure": {
+      "main": 1,
+      "headings": [
+        {
+          "level": 1,
+          "text": "Monitors"
+        },
+        {
+          "level": 2,
+          "text": "Create monitor"
+        },
+        {
+          "level": 2,
+          "text": "Your monitors"
+        },
+        {
+          "level": 3,
+          "text": "E08 Monitor A"
+        },
+        {
+          "level": 4,
+          "text": "Check history"
+        }
+      ]
+    }
+  },
+  "hydration": {
+    "identicalInitialMainText": true,
+    "identicalHeadingStructure": true,
+    "initialDataReads": 0,
+    "keyboardHandlerAttached": true,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "privacy": [
+    {
+      "identity": "alice",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": false,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "bob",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "anonymous",
+      "htmlStatus": 307,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    }
+  ],
+  "keyboard": {
+    "pointerActions": 0,
+    "createdMonitors": 1,
+    "tabCounts": [
+      2,
+      5,
+      1
+    ],
+    "openedDetailHistory": true,
+    "selectedSuccessAndAll": true
+  },
+  "accessibility": [
+    {
+      "screen": "login",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Sign in"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 34
+    },
+    {
+      "screen": "list",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Monitors"
+          },
+          {
+            "level": 2,
+            "text": "Create monitor"
+          },
+          {
+            "level": 2,
+            "text": "Your monitors"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor A"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor B"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 35
+    },
+    {
+      "screen": "detail",
+      "tool": "axe-core@4.10.3",
+      "structure": {
+        "main": 1,
+        "headings": [
+          {
+            "level": 1,
+            "text": "Monitors"
+          },
+          {
+            "level": 2,
+            "text": "Create monitor"
+          },
+          {
+            "level": 2,
+            "text": "Your monitors"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor A"
+          },
+          {
+            "level": 4,
+            "text": "Check history"
+          },
+          {
+            "level": 3,
+            "text": "E08 Monitor B"
+          }
+        ]
+      },
+      "violations": [],
+      "incomplete": [
+        {
+          "id": "color-contrast",
+          "impact": "serious",
+          "nodes": [
+            [
+              "#history-state-08000000-0000-4000-9000-000000000001"
+            ]
+          ]
+        }
+      ],
+      "passedRules": 42
+    }
+  ],
+  "final": {
+    "aliceMonitors": 2,
+    "bobMonitors": 0,
+    "unchangedCheckIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "secretValuesInHtmlOrRsc": 0,
+    "browserStorageEntries": 0,
+    "httpOnlyCookieHidden": true,
+    "serverLogSecrets": 0,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "cleanup": {
+    "children": [
+      {
+        "name": "production-next",
+        "code": 143,
+        "signal": null,
+        "forced": false,
+        "durationMs": 6
+      },
+      {
+        "name": "api",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 7
+      },
+      {
+        "name": "fixture",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 3
+      }
+    ],
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 6665
+}
diff --git a/evidence/E08/rsc-diagnostic.json b/evidence/E08/rsc-diagnostic.json
new file mode 100644
index 0000000..7224a3b
--- /dev/null
+++ b/evidence/E08/rsc-diagnostic.json
@@ -0,0 +1,122 @@
+{
+  "mode": "verification",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "productionBuild": true,
+  "hashes": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a"
+  },
+  "commands": [
+    {
+      "command": "npm run build (under fnm Node24.19.0)",
+      "exitCode": 0,
+      "durationMs": 2897
+    },
+    {
+      "command": "node test/fixture.ts",
+      "ready": true,
+      "readinessDurationMs": 71
+    },
+    {
+      "command": "node server/main.ts",
+      "ready": true,
+      "readinessDurationMs": 200
+    },
+    {
+      "command": "node node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 4313",
+      "ready": true,
+      "readinessDurationMs": 169
+    }
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "FAILED_BEFORE_SCENARIO",
+  "portsInitiallyFree": true,
+  "stage": "privacy",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 1,
+    "historyIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "monitorNameInHtml": true,
+    "historyIdsInHtml": true,
+    "cacheControl": "private, no-cache, no-store, max-age=0, must-revalidate",
+    "structure": {
+      "main": 1,
+      "headings": [
+        {
+          "level": 1,
+          "text": "Monitors"
+        },
+        {
+          "level": 2,
+          "text": "Create monitor"
+        },
+        {
+          "level": 2,
+          "text": "Your monitors"
+        },
+        {
+          "level": 3,
+          "text": "E08 Monitor A"
+        },
+        {
+          "level": 4,
+          "text": "Check history"
+        }
+      ]
+    }
+  },
+  "hydration": {
+    "identicalInitialMainText": true,
+    "identicalHeadingStructure": true,
+    "initialDataReads": 0,
+    "keyboardHandlerAttached": true,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "privacy": [],
+  "failure": {
+    "stage": "privacy",
+    "message": "The expression evaluated to a falsy value:\n\n  assert.ok(privateMarkers.every((marker) => htmlBody.includes(marker) && rscBody.includes(marker)))\n"
+  },
+  "cleanup": {
+    "children": [
+      {
+        "name": "production-next",
+        "code": 143,
+        "signal": null,
+        "forced": false,
+        "durationMs": 8
+      },
+      {
+        "name": "api",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 11
+      },
+      {
+        "name": "fixture",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 5
+      }
+    ],
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 5340
+}
diff --git a/evidence/E08/verification-attempt-1.json b/evidence/E08/verification-attempt-1.json
new file mode 100644
index 0000000..ca70c4b
--- /dev/null
+++ b/evidence/E08/verification-attempt-1.json
@@ -0,0 +1,122 @@
+{
+  "mode": "verification",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "productionBuild": true,
+  "hashes": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a"
+  },
+  "commands": [
+    {
+      "command": "npm run build (under fnm Node24.19.0)",
+      "exitCode": 0,
+      "durationMs": 3848
+    },
+    {
+      "command": "node test/fixture.ts",
+      "ready": true,
+      "readinessDurationMs": 74
+    },
+    {
+      "command": "node server/main.ts",
+      "ready": true,
+      "readinessDurationMs": 211
+    },
+    {
+      "command": "node node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 4313",
+      "ready": true,
+      "readinessDurationMs": 177
+    }
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "FAILED_BEFORE_SCENARIO",
+  "portsInitiallyFree": true,
+  "stage": "privacy",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 1,
+    "historyIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "monitorNameInHtml": true,
+    "historyIdsInHtml": true,
+    "cacheControl": "private, no-cache, no-store, max-age=0, must-revalidate",
+    "structure": {
+      "main": 1,
+      "headings": [
+        {
+          "level": 1,
+          "text": "Monitors"
+        },
+        {
+          "level": 2,
+          "text": "Create monitor"
+        },
+        {
+          "level": 2,
+          "text": "Your monitors"
+        },
+        {
+          "level": 3,
+          "text": "E08 Monitor A"
+        },
+        {
+          "level": 4,
+          "text": "Check history"
+        }
+      ]
+    }
+  },
+  "hydration": {
+    "identicalInitialMainText": true,
+    "identicalHeadingStructure": true,
+    "initialDataReads": 0,
+    "keyboardHandlerAttached": true,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "privacy": [],
+  "failure": {
+    "stage": "privacy",
+    "message": "The expression evaluated to a falsy value:\n\n  assert.ok(privateMarkers.every((marker) => htmlBody.includes(marker) && rscBody.includes(marker)))\n"
+  },
+  "cleanup": {
+    "children": [
+      {
+        "name": "production-next",
+        "code": 143,
+        "signal": null,
+        "forced": false,
+        "durationMs": 5
+      },
+      {
+        "name": "api",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 7
+      },
+      {
+        "name": "fixture",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 5
+      }
+    ],
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 6482
+}
diff --git a/evidence/E08/verification-attempt-2.json b/evidence/E08/verification-attempt-2.json
new file mode 100644
index 0000000..bb9d619
--- /dev/null
+++ b/evidence/E08/verification-attempt-2.json
@@ -0,0 +1,147 @@
+{
+  "mode": "verification",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "productionBuild": true,
+  "hashes": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a"
+  },
+  "commands": [
+    {
+      "command": "npm run build (under fnm Node24.19.0)",
+      "exitCode": 0,
+      "durationMs": 3046
+    },
+    {
+      "command": "node test/fixture.ts",
+      "ready": true,
+      "readinessDurationMs": 74
+    },
+    {
+      "command": "node server/main.ts",
+      "ready": true,
+      "readinessDurationMs": 202
+    },
+    {
+      "command": "node node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 4313",
+      "ready": true,
+      "readinessDurationMs": 150
+    }
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "FAILED_BEFORE_SCENARIO",
+  "portsInitiallyFree": true,
+  "stage": "keyboard",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 1,
+    "historyIds": [
+      "08000000-0000-4000-a000-000000000002",
+      "08000000-0000-4000-a000-000000000001"
+    ],
+    "monitorNameInHtml": true,
+    "historyIdsInHtml": true,
+    "cacheControl": "private, no-cache, no-store, max-age=0, must-revalidate",
+    "structure": {
+      "main": 1,
+      "headings": [
+        {
+          "level": 1,
+          "text": "Monitors"
+        },
+        {
+          "level": 2,
+          "text": "Create monitor"
+        },
+        {
+          "level": 2,
+          "text": "Your monitors"
+        },
+        {
+          "level": 3,
+          "text": "E08 Monitor A"
+        },
+        {
+          "level": 4,
+          "text": "Check history"
+        }
+      ]
+    }
+  },
+  "hydration": {
+    "identicalInitialMainText": true,
+    "identicalHeadingStructure": true,
+    "initialDataReads": 0,
+    "keyboardHandlerAttached": true,
+    "errors": {
+      "hydration": 0,
+      "page": 0,
+      "console": 0
+    }
+  },
+  "privacy": [
+    {
+      "identity": "alice",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": false,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "bob",
+      "htmlStatus": 200,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    },
+    {
+      "identity": "anonymous",
+      "htmlStatus": 307,
+      "rscStatus": 200,
+      "noForeignRecordData": true,
+      "noStore": true,
+      "rscNoStore": true
+    }
+  ],
+  "failure": {
+    "stage": "keyboard",
+    "message": "expect(locator).toHaveValue(expected) failed\n\nLocator:  getByRole('article', { name: 'E08 Monitor A', exact: true }).getByRole('region', { name: 'Check history for E08 Monitor A' }).getByLabel('History state', { exact: true })\nExpected: \"SUCCEEDED\"\nReceived: \"\"\nTimeout:  5000ms\n\nCall log:\n  - Expect \"to.have.value\" with timeout 5000ms\n  - waiting for getByRole('article', { name: 'E08 Monitor A', exact: true }).getByRole('region', { name: 'Check history for E08 Monitor A' }).getByLabel('History state', { exact: true })\n    14 × locator resolved to <select id=\"history-state-08000000-0000-4000-9000-000000000001\">…</select>\n       - unexpected value \"\"\n"
+  },
+  "cleanup": {
+    "children": [
+      {
+        "name": "production-next",
+        "code": 143,
+        "signal": null,
+        "forced": false,
+        "durationMs": 8
+      },
+      {
+        "name": "api",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 7
+      },
+      {
+        "name": "fixture",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 4
+      }
+    ],
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 10898
+}
diff --git a/evidence/E08/verification.json b/evidence/E08/verification.json
new file mode 100644
index 0000000..3dd1d58
--- /dev/null
+++ b/evidence/E08/verification.json
@@ -0,0 +1,244 @@
+{
+  "thread": "E08",
+  "attempt": 1,
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "result": "PASS",
+  "runtime": {
+    "node": "24.19.0",
+    "npm": "11.17.0",
+    "next": "16.3.3",
+    "playwright": "1.62.1",
+    "chromium": "151.0.7922.34",
+    "axeCore": "4.10.3",
+    "productionServer": "next start"
+  },
+  "frozen": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a",
+    "unchangedAfterBaseline": true,
+    "baseline": "baseline.json",
+    "decisiveBaseline": "At unchanged START, authenticated JavaScript-disabled production HTML had no Monitor A or either CheckRun. Stopped at this first failure."
+  },
+  "commands": [
+    {
+      "command": "fnm exec --using 24.19.0 node evidence/E08/run.mjs baseline",
+      "invocation": 1,
+      "exitCode": 0,
+      "runnerDurationMs": 5189,
+      "buildDurationMs": 3362,
+      "result": "REPRODUCED"
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm install --save-dev --save-exact axe-core@4.10.3 --ignore-scripts --no-audit --no-fund",
+      "invocation": 1,
+      "exitCode": 0,
+      "toolDurationMs": 847,
+      "npmDurationMs": 759,
+      "reason": "Only new dependency: pinned test-only engine required for the automated accessibility scan. No existing package pin changed."
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 1,
+      "exitCode": 0,
+      "durationMs": null,
+      "durationNote": "Wall duration was not captured."
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:unit",
+      "invocation": 1,
+      "exitCode": 0,
+      "testDurationMs": 2595.233291,
+      "passed": 18,
+      "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node evidence/E08/run.mjs verification",
+      "invocation": 1,
+      "exitCode": 1,
+      "wallDurationMs": 6850,
+      "runnerDurationMs": 6482,
+      "buildDurationMs": 3848,
+      "failure": "Initial SSR and hydration passed; the raw RSC request received Next's 307 canonicalization with an empty body, failing the Alice data assertion.",
+      "report": "verification-attempt-1.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node output/e08/diagnose-rsc.mjs",
+      "invocation": 1,
+      "exitCode": 1,
+      "wallDurationMs": 5840,
+      "runnerDurationMs": 5340,
+      "buildDurationMs": 2897,
+      "failure": "One diagnostic replay of the same frozen scenario: HTML200 contained all markers, raw RSC307 had zero body bytes. Only status/type/marker booleans were observed; no raw bodies or secrets retained.",
+      "report": "rsc-diagnostic.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "invocation": 1,
+      "exitCode": 0,
+      "wallDurationMs": 9670,
+      "testDurationMs": 9275.505166,
+      "passed": 15,
+      "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "invocation": 1,
+      "exitCode": 0,
+      "wallDurationMs": 8920,
+      "testDurationMs": 8562.152583,
+      "passed": 10,
+      "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 2,
+      "exitCode": 0,
+      "wallDurationMs": 2940
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "invocation": 2,
+      "exitCode": 0,
+      "wallDurationMs": 2680,
+      "testDurationMs": 2245.718084,
+      "passed": 19,
+      "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node --import ./test/e08-rsc-preload.mjs evidence/E08/run.mjs verification",
+      "invocation": 2,
+      "exitCode": 1,
+      "wallDurationMs": 11640,
+      "runnerDurationMs": 10898,
+      "buildDurationMs": 3046,
+      "frameworkRscContinuations": 3,
+      "failure": "SSR, hydration and all three identities' HTML/RSC privacy passed; native popup select did not change value on frozen ArrowDown/Enter input.",
+      "report": "verification-attempt-2.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node --import ./test/e08-rsc-preload.mjs evidence/E08/run.mjs verification",
+      "invocation": 3,
+      "exitCode": 0,
+      "wallDurationMs": 7160,
+      "runnerDurationMs": 6665,
+      "buildDurationMs": 3412,
+      "frameworkRscContinuations": 3,
+      "result": "PASS",
+      "report": "production.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 1,
+      "exitCode": 1,
+      "wallDurationMs": 57300,
+      "testDurationMs": 43100,
+      "build": "PASS",
+      "passed": 1,
+      "failed": 1,
+      "notRun": 9,
+      "failure": "E02's old browser collection-response mock could not provide the initial row once the collection was read on the server."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 2,
+      "exitCode": 1,
+      "wallDurationMs": 13590,
+      "testDurationMs": 9600,
+      "build": "PASS",
+      "passed": 6,
+      "failed": 1,
+      "notRun": 4,
+      "failure": "E04 Bob login returned200, but document replacement evicted the response body before the legacy post-click response.json assertion."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 3,
+      "exitCode": 1,
+      "wallDurationMs": 14200,
+      "testDurationMs": 9700,
+      "build": "PASS",
+      "passed": 6,
+      "failed": 1,
+      "notRun": 4,
+      "failure": "E04's eager waitForResponse.then(response.json) still lost the browser body during document replacement."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 4,
+      "exitCode": 0,
+      "wallDurationMs": 22620,
+      "testDurationMs": 18700,
+      "build": "PASS",
+      "passed": 11,
+      "failed": 0,
+      "notRun": 0,
+      "frameworkRscContinuations": 3,
+      "report": "browser.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 3,
+      "exitCode": 0,
+      "wallDurationMs": 2410
+    }
+  ],
+  "changesAfterFailures": [
+    "Root-approved RSC caller adapter follows at most one307 only for the exact primary RSC route, identical origin/path/user query, and one added _rsc discriminator. No login, foreign-origin, foreign-path or changed-query redirect is followed. Next's default and frozen observer remain unchanged.",
+    "Native history select size=3 retains the original labels, values, URL semantics and frozen ArrowDown/Enter/Home assertions. No custom keyboard handler or keyboard observer adapter.",
+    "E07 delayed-response consumer opens the same limit3 history request via browser selection; frozen rows, barrier, FAILED selection, exact IDs and the separate normal URL/back/forward/reload case are unchanged.",
+    "E02 consumers assert one worker. NOT_FOUND inserts only its exact existing row in the owned e03_browser schema, retaining the check failure; initial INTERNAL_ERROR renames only that schema's monitors table after startup, restoring in finally. Categories, no-success/no-article assertions and frozen error/prose unit cases remain.",
+    "E04 Bob-login consumer fetches the intercepted actual login response once with maxRetries0/maxRedirects0, reads its username before fulfill releases it, and preserves status200, identity, cookie/navigation and session assertions. No substitute body, additional login request or product change."
+  ],
+  "evidenceNotes": [
+    "Failed standalone reports retain the frozen runner's initial FAILED_BEFORE_SCENARIO label; their actual failure.stage is privacy or keyboard, after earlier assertions ran.",
+    "Alice's noForeignRecordData=false means the observer found Alice's expected own record. Bob and anonymous both report true and contain no Alice record data.",
+    "All three axe screens have zero violations of any severity. Detail has one color-contrast incomplete result on the native history listbox (impact serious); it is retained and no manual contrast verdict is claimed.",
+    "Standalone production.json checks captured server logs for secret values. The shared browser suite delegates that log check to the standalone result rather than claiming access to Playwright webServer logs."
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "unchangedStartBaselines": 1,
+    "standaloneVerificationInvocations": 3,
+    "diagnosticScenarioReplays": 1,
+    "productionBrowserSuiteInvocations": 4,
+    "fixedScenarioRunsInsideBrowserSuite": 1,
+    "productionBuildInvocations": 9,
+    "additionalFrameworkRscRequests": 9,
+    "typecheckInvocations": 3,
+    "unitInvocations": 2,
+    "apiInvocations": 1,
+    "postgresIntegrationInvocations": 1,
+    "freshRepairAttemptsAfterFinalReport": 0,
+    "testRetryConfiguration": "Playwright workers1/retries0/maxFailures1; assertion waiting is not a scenario rerun."
+  },
+  "scope": {
+    "newDependency": "axe-core@4.10.3 (test-only)",
+    "existingPinsUnchanged": true,
+    "priorEvidenceAndMigrationsUnchanged": true,
+    "branchSpecUnchanged": true,
+    "mainMutatedByThisAgent": false,
+    "authorizedRootMainWorkflowCommit": "589cabcb57112ebc782ef10128089014515d7fbc",
+    "futureThreadImplementation": false,
+    "rawHtmlOrRscRetained": false,
+    "rawRuntimeLogsRetained": false,
+    "screenshotsTracesVideosOrStorageStateRetained": false
+  },
+  "cleanup": {
+    "portsFree": [4311, 4312, 4313, 4314],
+    "residualTestSchemas": [],
+    "ownedStandaloneSchemaDropped": "e08_browser",
+    "browserSchemaDropped": "e03_browser",
+    "standaloneForcedProcessKills": 0,
+    "postgresLeftRunning": true,
+    "publicAndVolumesNotModified": true,
+    "auditReport": "cleanup.json"
+  },
+  "preparationNote": "An initial sandboxed docker-compose status read was denied, then rerun with normal escalation. No scenario started in the denied command. Source/path read errors did not start tests or change fixtures.",
+  "unresolved": "No blocking acceptance failure. Native-listbox contrast remains an explicitly reported axe incomplete result."
+}
diff --git a/package-lock.json b/package-lock.json
index 043e019..9c13988 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -20,6 +20,7 @@
         "@types/pg": "8.15.5",
         "@types/react": "19.2.18",
         "@types/react-dom": "19.2.5",
+        "axe-core": "4.10.3",
         "typescript": "5.9.3"
       },
       "engines": {
@@ -1000,6 +1001,16 @@
         "fastq": "^1.17.1"
       }
     },
+    "node_modules/axe-core": {
+      "version": "4.10.3",
+      "resolved": "https://registry.npmjs.org/axe-core/-/axe-core-4.10.3.tgz",
+      "integrity": "sha512-Xm7bpRXnDSX2YE2YFfBk2FnF0ep6tmG7xPh8iHee8MIcrgq762Nkce856dYtJYLkuIoYZvGfTs/PbZhideTcEg==",
+      "dev": true,
+      "license": "MPL-2.0",
+      "engines": {
+        "node": ">=4"
+      }
+    },
     "node_modules/baseline-browser-mapping": {
       "version": "2.11.19",
       "resolved": "https://registry.npmjs.org/baseline-browser-mapping/-/baseline-browser-mapping-2.11.19.tgz",
diff --git a/package.json b/package.json
index dcbf7dc..a57aea8 100644
--- a/package.json
+++ b/package.json
@@ -15,7 +15,7 @@
     "typecheck": "NEXT_TELEMETRY_DISABLED=1 next typegen && tsc --noEmit",
     "test:unit": "node --test test/unit.test.ts",
     "test:functional": "node --test --test-concurrency=1 test/functional.test.ts test/contracts.test.ts",
-    "test:e2e": "playwright test",
+    "test:e2e": "npm run build && playwright test",
     "test": "npm run test:unit && npm run test:functional",
     "db:up": "docker compose --project-name wse-fundamentals up -d --wait",
     "db:down": "docker compose --project-name wse-fundamentals down",
@@ -35,6 +35,7 @@
     "@types/pg": "8.15.5",
     "@types/react": "19.2.18",
     "@types/react-dom": "19.2.5",
+    "axe-core": "4.10.3",
     "typescript": "5.9.3"
   }
 }
diff --git a/playwright.config.ts b/playwright.config.ts
index f9d061c..af740dc 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -17,7 +17,7 @@ export default defineConfig({
     { command: 'npm run fixture', url: 'http://127.0.0.1:4311/ok', reuseExistingServer: false, timeout: 30_000 },
     { command: 'node test/prepare-browser-db.ts && npm run start:api', url: 'http://127.0.0.1:4312/health', reuseExistingServer: false, timeout: 30_000,
       env: { DATABASE_SCHEMA: 'e03_browser' } },
-    { command: 'npm run dev:web', url: 'http://127.0.0.1:4313/monitors', reuseExistingServer: false, timeout: 90_000,
+    { command: 'npm run start:web', url: 'http://127.0.0.1:4313/login', reuseExistingServer: false, timeout: 90_000,
       env: { NEXT_TELEMETRY_DISABLED: '1' } },
   ],
 });
diff --git a/test/browser/contracts.spec.ts b/test/browser/contracts.spec.ts
index 6a2cb54..23194d0 100644
--- a/test/browser/contracts.spec.ts
+++ b/test/browser/contracts.spec.ts
@@ -1,6 +1,9 @@
 import { expect } from '@playwright/test';
 import { test } from './session';
 import scenario from '../../evidence/E02/scenario.json' with { type: 'json' };
+import { databasePool, schemaIdentifier } from '../../server/database.ts';
+import { monitorToValues } from '../../server/mapping.ts';
+import { testDatabaseConfig } from '../database.ts';
 
 test('INVALID_INPUT displays the same category for both server prose variants and never creates success UI', async ({ page }) => {
   let failure = scenario.browserErrors[0];
@@ -26,35 +29,62 @@ test('INVALID_INPUT displays the same category for both server prose variants an
   }
 });
 
-test('NOT_FOUND from a manual check leaves the Monitor without a successful result', async ({ page }) => {
+test('NOT_FOUND from a manual check leaves the Monitor without a successful result', async ({ page, users }) => {
   const failure = scenario.browserErrors.find(({ code }) => code === 'NOT_FOUND')!;
   const monitor = {
     ...scenario.monitor, id: scenario.browserMonitorId,
     createdAt: scenario.browserMonitorTimestamp, updatedAt: scenario.browserMonitorTimestamp, latestCheck: null,
   };
-  await page.route('**/api/monitors', (route) => route.fulfill({ json: { data: [monitor] } }));
-  await page.route(`**/api/monitors/${monitor.id}/checks`, (route) => route.fulfill({
-    status: failure.status, json: { error: { code: failure.code, message: failure.message } },
-  }));
-  await page.goto('/monitors');
-  const article = page.getByRole('article', { name: scenario.monitor.name, exact: true });
-  await article.getByRole('button', { name: 'Run check', exact: true }).click();
-  const alert = page.getByRole('main').getByRole('alert');
-  await expect(alert).toHaveText('The requested monitor or resource was not found.');
-  await expect(alert).toHaveAttribute('data-error-code', 'NOT_FOUND');
-  await expect(article).toContainText('No checks yet.');
-  await expect(article.getByText('SUCCEEDED', { exact: true })).toHaveCount(0);
-  await expect(article.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+  expect(test.info().config.workers).toBe(1);
+  const config = testDatabaseConfig('e03_browser');
+  const schema = schemaIdentifier(config.schema);
+  const pool = databasePool(config);
+  let inserted = false;
+  try {
+    // E08's initial read is on the server. Seed the exact former mocked row in
+    // this suite's exclusively owned schema; keep the check failure unchanged.
+    const result = await pool.query(`INSERT INTO ${schema}.monitors
+      (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+      SELECT $1, $2, $3, $4, $5, $6, $7, id FROM ${schema}.users WHERE username = $8`, [...monitorToValues(monitor), users[0].username]);
+    inserted = result.rowCount === 1;
+    expect(inserted).toBe(true);
+    await page.route(`**/api/monitors/${monitor.id}/checks`, (route) => route.fulfill({
+      status: failure.status, json: { error: { code: failure.code, message: failure.message } },
+    }));
+    await page.goto('/monitors');
+    const article = page.getByRole('article', { name: scenario.monitor.name, exact: true });
+    await article.getByRole('button', { name: 'Run check', exact: true }).click();
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveText('The requested monitor or resource was not found.');
+    await expect(alert).toHaveAttribute('data-error-code', 'NOT_FOUND');
+    await expect(article).toContainText('No checks yet.');
+    await expect(article.getByText('SUCCEEDED', { exact: true })).toHaveCount(0);
+    await expect(article.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+  } finally {
+    if (inserted) await pool.query(`DELETE FROM ${schema}.monitors WHERE id = $1`, [monitor.id]);
+    await pool.end();
+  }
 });
 
 test('INTERNAL_ERROR during loading displays the service failure category without server details', async ({ page }) => {
-  const failure = scenario.browserErrors.find(({ code }) => code === 'INTERNAL_ERROR')!;
-  await page.route('**/api/monitors', (route) => route.fulfill({
-    status: failure.status, json: { error: { code: failure.code, message: failure.message } },
-  }));
-  await page.goto('/monitors');
-  const alert = page.getByRole('main').getByRole('alert');
-  await expect(alert).toHaveText('The monitoring service could not complete the request.');
-  await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
-  await expect(page.getByRole('article')).toHaveCount(0);
+  expect(test.info().config.workers).toBe(1);
+  const config = testDatabaseConfig('e03_browser');
+  const schema = schemaIdentifier(config.schema);
+  const pool = databasePool(config);
+  let renamed = false;
+  try {
+    // The API is already running. Fail only this owned schema's real SSR read,
+    // preserving all rows and restoring the table before the following test.
+    await pool.query(`ALTER TABLE ${schema}.monitors RENAME TO e08_unavailable_monitors`);
+    renamed = true;
+    await page.goto('/monitors');
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveText('The monitoring service could not complete the request.');
+    await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(page.getByRole('article')).toHaveCount(0);
+    await expect(page.getByRole('main')).not.toContainText('e08_unavailable_monitors');
+  } finally {
+    if (renamed) await pool.query(`ALTER TABLE ${schema}.e08_unavailable_monitors RENAME TO monitors`);
+    await pool.end();
+  }
 });
diff --git a/test/browser/history.spec.ts b/test/browser/history.spec.ts
index 77777fb..5c0e7ca 100644
--- a/test/browser/history.spec.ts
+++ b/test/browser/history.spec.ts
@@ -89,7 +89,10 @@ test('E07 delayed history response cannot replace a newer URL filter', async ({
         await route.fulfill({ response });
       } else await route.continue();
     });
-    await page.goto(`/monitors?${new URLSearchParams({ monitor: scenario.monitor.id, limit: '3' })}`);
+    // E08 renders direct-URL history on the server. Start the same held browser
+    // GET through client selection; its fixture, filter and barrier stay fixed.
+    await page.goto('/monitors?limit=3');
+    await page.getByRole('article', { name: scenario.monitor.name, exact: true }).getByRole('button', { name: 'View history', exact: true }).click();
     await held;
     const history = page.getByRole('region', { name: `Check history for ${scenario.monitor.name}` });
     await expect(history.getByText('Loading history…', { exact: true })).toBeVisible();
diff --git a/test/browser/lifecycle.spec.ts b/test/browser/lifecycle.spec.ts
index ac630d5..a43cf82 100644
--- a/test/browser/lifecycle.spec.ts
+++ b/test/browser/lifecycle.spec.ts
@@ -72,7 +72,6 @@ test('E03 persist A,A,B history, edit, pause, enable and delete through the real
   expect(removed.status()).toBe(404);
   expect((await removed.json()).error.code).toBe('NOT_FOUND');
   await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
-  await page.screenshot({ path: 'output/playwright/E03-lifecycle.png', fullPage: true });
 });
 
 test.describe('E04 real browser session lifecycle', () => {
@@ -122,17 +121,25 @@ test.describe('E04 real browser session lifecycle', () => {
     await page.goto('/monitors');
     await expect(page).toHaveURL(`${authScenario.browserOrigin}/login`);
 
+    let bobLoginUsername: string | undefined;
+    await page.route('**/api/auth/login', async (route) => {
+      // Read the actual response before document replacement evicts its browser body.
+      const response = await route.fetch({ maxRedirects: 0, maxRetries: 0 });
+      bobLoginUsername = (await response.json()).data.user.username;
+      await route.fulfill({ response });
+    });
     const bobResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/login') && response.request().method() === 'POST');
     await submitCredentials(page, users[1]);
     const response = await bobResponse;
     expect(response.status()).toBe(200);
-    expect((await response.json()).data.user.username).toBe(users[1].username);
+    expect(bobLoginUsername).toBe(users[1].username);
     await expect(page).toHaveURL(`${authScenario.browserOrigin}/monitors`);
     await expect(page.getByRole('button', { name: 'Sign out', exact: true })).toBeVisible();
     const bobSession = await page.request.get('/api/auth/session');
     expect(bobSession.status()).toBe(200);
     expect((await bobSession.json()).data.user.username).toBe(users[1].username);
     await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
+    await page.unroute('**/api/auth/login');
     await mkdir('output/e04', { recursive: true });
     await writeFile('output/e04/browser.json', JSON.stringify({
       wrongLoginStatus: wrongStatus, aliceLoginStatus: aliceStatus, reloadAuthenticated: true,
diff --git a/test/browser/monitor.spec.ts b/test/browser/monitor.spec.ts
index 4e5973f..d391042 100644
--- a/test/browser/monitor.spec.ts
+++ b/test/browser/monitor.spec.ts
@@ -16,7 +16,6 @@ test('create Fixture monitor, run one manual check, and display the terminal res
   await expect(monitor.getByText('200', { exact: true })).toBeVisible();
   await expect(monitor).toContainText('Enabled · 60 seconds');
   await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
-  await page.screenshot({ path: 'output/playwright/E01-success.png', fullPage: true });
 
   await page.reload();
   await expect(page.getByRole('article', { name: 'Fixture monitor' }).getByText('SUCCEEDED', { exact: true })).toBeVisible();
diff --git a/test/browser/rendering.spec.ts b/test/browser/rendering.spec.ts
new file mode 100644
index 0000000..3c035dd
--- /dev/null
+++ b/test/browser/rendering.spec.ts
@@ -0,0 +1,22 @@
+import { test, expect } from '@playwright/test';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { databasePool } from '../../server/database.ts';
+import { testDatabaseConfig } from '../database.ts';
+import { insertRenderingFixture, removeRenderingFixture } from '../../evidence/E08/fixture.mjs';
+import { runRenderingScenario } from '../../evidence/E08/browser-scenario.mjs';
+import { e08RscTransport } from '../e08-rsc-transport.mjs';
+
+test('E08 production SSR, hydration, owner privacy and keyboard accessibility', async ({ browser }) => {
+  const pool = databasePool(testDatabaseConfig('e03_browser'));
+  const report: { result?: string; rscContinuations: object[] } = { rscContinuations: [] };
+  try {
+    const users = await insertRenderingFixture(pool);
+    await runRenderingScenario(e08RscTransport(browser, (continuation: object) => report.rscContinuations.push(continuation)), pool, users, 'verification', report);
+    expect(report.result).toBe('PASS');
+  } finally {
+    await removeRenderingFixture(pool);
+    await pool.end();
+    await mkdir('output/e08', { recursive: true });
+    await writeFile('output/e08/browser.json', JSON.stringify(report, null, 2) + '\n');
+  }
+});
diff --git a/test/e08-rsc-preload.mjs b/test/e08-rsc-preload.mjs
new file mode 100644
index 0000000..1360f18
--- /dev/null
+++ b/test/e08-rsc-preload.mjs
@@ -0,0 +1,8 @@
+import { chromium } from '@playwright/test';
+import { e08RscTransport } from './e08-rsc-transport.mjs';
+
+// Used only by the standalone frozen E08 runner through node --import.
+const launch = chromium.launch.bind(chromium);
+chromium.launch = async (...args) => e08RscTransport(await launch(...args), (continuation) => {
+  console.log(JSON.stringify({ e08RscContinuation: continuation }));
+});
diff --git a/test/e08-rsc-transport.mjs b/test/e08-rsc-transport.mjs
new file mode 100644
index 0000000..635bc34
--- /dev/null
+++ b/test/e08-rsc-transport.mjs
@@ -0,0 +1,29 @@
+import scenario from '../evidence/E08/scenario.json' with { type: 'json' };
+
+// The frozen observer requests raw RSC without Next's _rsc discriminator.
+// Follow its documented canonicalization once, without following login or
+// changing any user query. This adapts transport only, not an assertion.
+export function e08RscTransport(browser, record) {
+  return {
+    version: () => browser.version(),
+    close: () => browser.close(),
+    async newContext(options) {
+      const context = await browser.newContext(options);
+      const get = context.request.get.bind(context.request);
+      context.request.get = async (url, options) => {
+        const response = await get(url, options);
+        if (url !== scenario.primaryRoute || options?.headers?.rsc !== '1' || response.status() !== 307) return response;
+        const source = new URL(url, scenario.browserOrigin);
+        const destination = new URL(response.headers().location ?? '', source);
+        const query = new URLSearchParams(destination.search);
+        const discriminator = query.getAll('_rsc');
+        query.delete('_rsc');
+        if (destination.origin !== source.origin || destination.pathname !== source.pathname || destination.hash !== source.hash ||
+          discriminator.length !== 1 || query.toString() !== source.searchParams.toString()) return response;
+        record({ status: 307, sameOriginAndPath: true, unchangedUserQuery: true, onlyFrameworkRscAdded: true, followCount: 1 });
+        return get(destination.href, { ...options, maxRedirects: 0 });
+      };
+      return context;
+    },
+  };
+}
diff --git a/test/unit.test.ts b/test/unit.test.ts
index 2defd10..7a0bd7d 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -11,6 +11,43 @@ import { loginInput } from '../server/contracts.ts';
 import { csrfTokenForSession, validCsrfToken, SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
 import { DEFAULT_HISTORY_LIMIT, MAX_HISTORY_LIMIT, MAX_HISTORY_CURSOR_LENGTH, historyCursor, historyQuery } from '../server/history.ts';
 import historyScenario from '../evidence/E07/scenario.json' with { type: 'json' };
+import { emptyMonitors, historyLocation } from '../app/monitors/initial-state.ts';
+import { e08RscTransport } from './e08-rsc-transport.mjs';
+import renderingScenario from '../evidence/E08/scenario.json' with { type: 'json' };
+
+test('E08 RSC caller follows only one same-route framework canonicalization', async () => {
+  const route = renderingScenario.primaryRoute;
+  for (const [location, count] of [
+    [`${route}&_rsc`, 2], ['/login', 1], [`https://example.invalid${route}&_rsc`, 1],
+    [`${route}&state=FAILED&_rsc`, 1], [`${route}&_rsc&_rsc=extra`, 1],
+  ] as const) {
+    const calls: string[] = [];
+    const context = { request: { get: async (url: string, _options?: object) => {
+      calls.push(url);
+      return { status: () => 307, headers: () => ({ location }) };
+    } } };
+    const browser = { version: () => '', close: async () => {}, newContext: async () => context };
+    const client = await e08RscTransport(browser, () => {}).newContext({});
+    await client.request.get(route, { headers: { rsc: '1' }, maxRedirects: 0 });
+    assert.equal(calls.length, count);
+  }
+});
+
+test('E08 server and browser history keys match without dropping invalid query parameters', () => {
+  const browser = new URLSearchParams('state=FAILED&limit=20&monitor=example&state=SUCCEEDED&extra=1');
+  const server = new URLSearchParams('state=FAILED&state=SUCCEEDED&limit=20&monitor=example&extra=1');
+  assert.deepEqual(historyLocation(browser), historyLocation(server));
+  assert.deepEqual(historyLocation(browser), { id: 'example', search: 'extra=1&limit=20&state=FAILED&state=SUCCEEDED' });
+  assert.equal(browser.get('monitor'), 'example');
+});
+
+test('E08 unauthenticated initial payloads never share mutable state between requests', () => {
+  const first = emptyMonitors();
+  const second = emptyMonitors();
+  assert.notEqual(first.monitors, second.monitors);
+  assert.notEqual(first.histories, second.histories);
+  assert.deepEqual(Object.keys(first).sort(), ['authenticated', 'error', 'histories', 'monitors']);
+});
 
 test('a path on the configured fixture origin is allowed', () => {
   assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
