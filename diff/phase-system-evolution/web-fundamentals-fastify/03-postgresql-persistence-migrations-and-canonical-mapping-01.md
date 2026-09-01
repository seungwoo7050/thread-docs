# E03 PostgreSQL 영속성, Migration과 Canonical Mapping

## `Record the fixed E03 restart counterexample`

diff --git a/evidence/E03/baseline-reproducer.mjs b/evidence/E03/baseline-reproducer.mjs
new file mode 100644
index 0000000..1955e05
--- /dev/null
+++ b/evidence/E03/baseline-reproducer.mjs
@@ -0,0 +1,42 @@
+import assert from 'node:assert/strict';
+import { writeFile } from 'node:fs/promises';
+import { buildApp } from './server/app.ts';
+import { fixtureServer } from './test/fixture.ts';
+import scenario from './evidence/E03/scenario.json' with { type: 'json' };
+const start = performance.now();
+const fixture = fixtureServer();
+let app = buildApp(scenario.fixtureOrigin);
+const evidence = {scenario:'A,A,B then close and fresh independent instance', start:scenario.start, codeUnmodified:true, monitors:[], checks:[]};
+try {
+ await new Promise(resolve => fixture.server.listen(4311, '127.0.0.1', resolve));
+ await app.listen({ host:'127.0.0.1',port:4312 });
+ for (const input of scenario.monitors) {
+  const response = await fetch(`${scenario.apiOrigin}/monitors`, {method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify(input)});
+  assert.equal(response.status,201);
+  evidence.monitors.push((await response.json()).data);
+ }
+ for (const expected of scenario.checkSequence) {
+  const response = await fetch(`${scenario.apiOrigin}/monitors/${evidence.monitors[expected.monitor].id}/checks`, {method:'POST'});
+  assert.equal(response.status,200);
+  const check=(await response.json()).data;
+  assert.equal(check.state,expected.state); assert.equal(check.httpStatus,expected.httpStatus);
+  evidence.checks.push(check);
+ }
+ evidence.beforeRestart = (await (await fetch(`${scenario.apiOrigin}/monitors`)).json()).data;
+ assert.equal(evidence.beforeRestart.length,2);
+ await app.close();
+ app=buildApp(scenario.fixtureOrigin);
+ await app.listen({host:'127.0.0.1',port:4312});
+ evidence.afterRestart=(await (await fetch(`${scenario.apiOrigin}/monitors`)).json()).data;
+ evidence.expectedMonitorCount=2;
+ evidence.observedMonitorCount=evidence.afterRestart.length;
+ evidence.result=evidence.afterRestart.length===0 ? 'REPRODUCED: all Monitors and reachable CheckRuns disappeared' : 'NOT_REPRODUCED';
+ evidence.fixtureCalls=Object.fromEntries(fixture.calls);
+ evidence.durationMs=Math.round(performance.now()-start);
+ await writeFile('evidence/E03/baseline.json',JSON.stringify(evidence,null,2)+'\n');
+ console.log(JSON.stringify(evidence,null,2));
+ assert.equal(evidence.afterRestart.length,0,'Baseline expected the documented existing memory behavior');
+} finally {
+ await app.close(); fixture.server.closeAllConnections();
+ await new Promise(resolve => fixture.server.close(resolve));
+}
diff --git a/evidence/E03/baseline.json b/evidence/E03/baseline.json
new file mode 100644
index 0000000..1910d4e
--- /dev/null
+++ b/evidence/E03/baseline.json
@@ -0,0 +1,113 @@
+{
+  "scenario": "A,A,B then close and fresh independent instance",
+  "start": "2d26b6b08e7580eef35a61093f41d7cc60f9de15",
+  "codeUnmodified": true,
+  "monitors": [
+    {
+      "id": "6bafcdc6-5027-46a8-90c0-f5d7046c1302",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:42:41.068Z",
+      "updatedAt": "2026-08-27T23:42:41.068Z",
+      "latestCheck": null
+    },
+    {
+      "id": "ede6a6d9-0b58-4b03-925a-e3b5577c479d",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4311/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:42:41.079Z",
+      "updatedAt": "2026-08-27T23:42:41.079Z",
+      "latestCheck": null
+    }
+  ],
+  "checks": [
+    {
+      "id": "127c88d9-4b44-4ffb-b720-e9728f28a2fc",
+      "monitorId": "6bafcdc6-5027-46a8-90c0-f5d7046c1302",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 2,
+      "failureReason": null,
+      "startedAt": "2026-08-27T23:42:41.081Z",
+      "finishedAt": "2026-08-27T23:42:41.083Z"
+    },
+    {
+      "id": "756fb17f-1fe8-4c61-b2f6-2ef014d59012",
+      "monitorId": "6bafcdc6-5027-46a8-90c0-f5d7046c1302",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 1,
+      "failureReason": null,
+      "startedAt": "2026-08-27T23:42:41.085Z",
+      "finishedAt": "2026-08-27T23:42:41.086Z"
+    },
+    {
+      "id": "d042dc81-d99c-4d2b-8c1e-acf861dae4a4",
+      "monitorId": "ede6a6d9-0b58-4b03-925a-e3b5577c479d",
+      "trigger": "MANUAL",
+      "state": "FAILED",
+      "httpStatus": 503,
+      "latencyMs": 1,
+      "failureReason": "HTTP_STATUS",
+      "startedAt": "2026-08-27T23:42:41.088Z",
+      "finishedAt": "2026-08-27T23:42:41.088Z"
+    }
+  ],
+  "beforeRestart": [
+    {
+      "id": "6bafcdc6-5027-46a8-90c0-f5d7046c1302",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:42:41.068Z",
+      "updatedAt": "2026-08-27T23:42:41.068Z",
+      "latestCheck": {
+        "id": "756fb17f-1fe8-4c61-b2f6-2ef014d59012",
+        "monitorId": "6bafcdc6-5027-46a8-90c0-f5d7046c1302",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:42:41.085Z",
+        "finishedAt": "2026-08-27T23:42:41.086Z"
+      }
+    },
+    {
+      "id": "ede6a6d9-0b58-4b03-925a-e3b5577c479d",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4311/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:42:41.079Z",
+      "updatedAt": "2026-08-27T23:42:41.079Z",
+      "latestCheck": {
+        "id": "d042dc81-d99c-4d2b-8c1e-acf861dae4a4",
+        "monitorId": "ede6a6d9-0b58-4b03-925a-e3b5577c479d",
+        "trigger": "MANUAL",
+        "state": "FAILED",
+        "httpStatus": 503,
+        "latencyMs": 1,
+        "failureReason": "HTTP_STATUS",
+        "startedAt": "2026-08-27T23:42:41.088Z",
+        "finishedAt": "2026-08-27T23:42:41.088Z"
+      }
+    }
+  ],
+  "afterRestart": [],
+  "expectedMonitorCount": 2,
+  "observedMonitorCount": 0,
+  "result": "REPRODUCED: all Monitors and reachable CheckRuns disappeared",
+  "fixtureCalls": {
+    "/ok": 2,
+    "/fail": 1
+  },
+  "durationMs": 68
+}
diff --git a/evidence/E03/scenario.json b/evidence/E03/scenario.json
new file mode 100644
index 0000000..1914234
--- /dev/null
+++ b/evidence/E03/scenario.json
@@ -0,0 +1,117 @@
+{
+  "thread": "E03",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "2d26b6b08e7580eef35a61093f41d7cc60f9de15",
+  "frozenBeforeBaseline": true,
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "monitors": [
+    {
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": true
+    },
+    {
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4311/fail",
+      "interval": 120,
+      "enabled": true
+    }
+  ],
+  "checkSequence": [
+    {
+      "monitor": 0,
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "failureReason": null
+    },
+    {
+      "monitor": 0,
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "failureReason": null
+    },
+    {
+      "monitor": 1,
+      "state": "FAILED",
+      "httpStatus": 503,
+      "failureReason": "HTTP_STATUS"
+    }
+  ],
+  "restart": {
+    "kind": "close first app, create a fresh independent app and connection pool with identical persistence settings",
+    "expectedMonitorCount": 2,
+    "expectedCheckCount": 3,
+    "preserve": [
+      "id",
+      "createdAt",
+      "updatedAt",
+      "startedAt",
+      "finishedAt",
+      "state",
+      "httpStatus",
+      "latencyMs",
+      "failureReason"
+    ]
+  },
+  "update": {
+    "name": "Updated A",
+    "interval": 90
+  },
+  "enabledSequence": [
+    false,
+    true
+  ],
+  "deleteMonitorIndex": 1,
+  "additionalAssertions": {
+    "missingId": "00000000-0000-4000-8000-000000000000",
+    "malformedId": "not-a-uuid",
+    "invalidUpdateOverrides": [
+      {
+        "name": "   "
+      },
+      {
+        "interval": "90"
+      },
+      {
+        "interval": 0
+      },
+      {
+        "enabled": "false"
+      },
+      {
+        "url": "http://127.0.0.1:4314/ok"
+      }
+    ],
+    "mappingFixture": {
+      "id": "00000000-0000-4000-8000-000000000101",
+      "name": "  Canonical 😀  ",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 1,
+      "enabled": false,
+      "timestamp": "2026-01-01T12:34:56.789+09:00",
+      "expectedTimestamp": "2026-01-01T03:34:56.789Z",
+      "latencyMs": 0,
+      "httpStatus": null,
+      "failureReason": "CONNECTION_FAILURE"
+    },
+    "migrationSchema": "e03_migrations",
+    "incompatibleSchema": "e03_incompatible",
+    "incompatibleColumn": "monitors.interval_seconds",
+    "expectedMigrationFiles": [
+      "001_monitors.sql",
+      "002_check_runs.sql"
+    ],
+    "repeatMigrationExpectedApplied": [],
+    "browserLifecycle": "create A and B; run A,A,B; reload and read both histories; edit A; pause A; enable A; delete B; reload and confirm remaining A"
+  },
+  "checkDeadlineMs": 2000,
+  "retries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "parameterSweeps": 0,
+  "baselineStop": "One A,A,B sequence and one new app read. Stop after observed empty Monitor list, without production code changes."
+}


## `Introduce checked PostgreSQL schema migrations`

diff --git a/compose.yaml b/compose.yaml
new file mode 100644
index 0000000..b435051
--- /dev/null
+++ b/compose.yaml
@@ -0,0 +1,20 @@
+name: wse-fundamentals
+services:
+  postgres:
+    image: postgres:17.6-bookworm@sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3
+    environment:
+      POSTGRES_USER: monitor
+      POSTGRES_DB: monitor
+      # Isolated local fixture only; no reusable password or production credential.
+      POSTGRES_HOST_AUTH_METHOD: trust
+    ports:
+      - "127.0.0.1:15431:5432"
+    volumes:
+      - postgres-data:/var/lib/postgresql/data
+    healthcheck:
+      test: ["CMD-SHELL", "pg_isready -U monitor -d monitor"]
+      interval: 1s
+      timeout: 3s
+      retries: 30
+volumes:
+  postgres-data:
diff --git a/package-lock.json b/package-lock.json
index 2a376ca..043e019 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -10,12 +10,14 @@
       "dependencies": {
         "fastify": "5.12.1",
         "next": "16.3.3",
+        "pg": "8.16.3",
         "react": "19.2.8",
         "react-dom": "19.2.8"
       },
       "devDependencies": {
         "@playwright/test": "1.62.1",
         "@types/node": "24.13.3",
+        "@types/pg": "8.15.5",
         "@types/react": "19.2.18",
         "@types/react-dom": "19.2.5",
         "typescript": "5.9.3"
@@ -882,6 +884,18 @@
         "undici-types": "~7.18.0"
       }
     },
+    "node_modules/@types/pg": {
+      "version": "8.15.5",
+      "resolved": "https://registry.npmjs.org/@types/pg/-/pg-8.15.5.tgz",
+      "integrity": "sha512-LF7lF6zWEKxuT3/OR8wAZGzkg4ENGXFNyiV/JeOt9z5B+0ZVwbql9McqX5c/WStFq1GaGso7H1AzP/qSzmlCKQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/node": "*",
+        "pg-protocol": "*",
+        "pg-types": "^2.2.0"
+      }
+    },
     "node_modules/@types/react": {
       "version": "19.2.18",
       "resolved": "https://registry.npmjs.org/@types/react/-/react-19.2.18.tgz",
@@ -1345,6 +1359,95 @@
         "node": ">=14.0.0"
       }
     },
+    "node_modules/pg": {
+      "version": "8.16.3",
+      "resolved": "https://registry.npmjs.org/pg/-/pg-8.16.3.tgz",
+      "integrity": "sha512-enxc1h0jA/aq5oSDMvqyW3q89ra6XIIDZgCX9vkMrnz5DFTw/Ny3Li2lFQ+pt3L6MCgm/5o2o8HW9hiJji+xvw==",
+      "license": "MIT",
+      "dependencies": {
+        "pg-connection-string": "^2.9.1",
+        "pg-pool": "^3.10.1",
+        "pg-protocol": "^1.10.3",
+        "pg-types": "2.2.0",
+        "pgpass": "1.0.5"
+      },
+      "engines": {
+        "node": ">= 16.0.0"
+      },
+      "optionalDependencies": {
+        "pg-cloudflare": "^1.2.7"
+      },
+      "peerDependencies": {
+        "pg-native": ">=3.0.1"
+      },
+      "peerDependenciesMeta": {
+        "pg-native": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/pg-cloudflare": {
+      "version": "1.4.0",
+      "resolved": "https://registry.npmjs.org/pg-cloudflare/-/pg-cloudflare-1.4.0.tgz",
+      "integrity": "sha512-Vo7z/6rrQYxpNRylp4Tlob2elzbh+N/MOQbxFVWCxS7oEx6jF53GTJFxK2WWpKuBRkmiin4Mt+xofFDjx09R0A==",
+      "license": "MIT",
+      "optional": true
+    },
+    "node_modules/pg-connection-string": {
+      "version": "2.14.0",
+      "resolved": "https://registry.npmjs.org/pg-connection-string/-/pg-connection-string-2.14.0.tgz",
+      "integrity": "sha512-XwWDGcLRGCXAR8F/AM5bG7Q+A3Wm2s6QeEjlOKZLlH3UYcguiqCWKyWXVag5TLTIjR7oOJUY8kcADaZgWPyLeg==",
+      "license": "MIT"
+    },
+    "node_modules/pg-int8": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/pg-int8/-/pg-int8-1.0.1.tgz",
+      "integrity": "sha512-WCtabS6t3c8SkpDBUlb1kjOs7l66xsGdKpIPZsg4wR+B3+u9UAum2odSsF9tnvxg80h4ZxLWMy4pRjOsFIqQpw==",
+      "license": "ISC",
+      "engines": {
+        "node": ">=4.0.0"
+      }
+    },
+    "node_modules/pg-pool": {
+      "version": "3.14.0",
+      "resolved": "https://registry.npmjs.org/pg-pool/-/pg-pool-3.14.0.tgz",
+      "integrity": "sha512-gKtPkFdQPU3DksooVLi9LsjZxrsBUZIpa+7aVx+LV5pNh0KzP4Zleud2po+ConrxbuXGBJ6Hfer6hdgpIBpBaw==",
+      "license": "MIT",
+      "peerDependencies": {
+        "pg": ">=8.0"
+      }
+    },
+    "node_modules/pg-protocol": {
+      "version": "1.16.0",
+      "resolved": "https://registry.npmjs.org/pg-protocol/-/pg-protocol-1.16.0.tgz",
+      "integrity": "sha512-sILXutLVjCLjcDuOmvhX5e2Z4cS5qG/6Bu3VkpFwdf/633ElGLpEh9bgmuI5I4sqKqkifQiGyiCcx1HdtrK7tg==",
+      "license": "MIT"
+    },
+    "node_modules/pg-types": {
+      "version": "2.2.0",
+      "resolved": "https://registry.npmjs.org/pg-types/-/pg-types-2.2.0.tgz",
+      "integrity": "sha512-qTAAlrEsl8s4OiEQY69wDvcMIdQN6wdz5ojQiOy6YRMuynxenON0O5oCpJI6lshc6scgAY8qvJ2On/p+CXY0GA==",
+      "license": "MIT",
+      "dependencies": {
+        "pg-int8": "1.0.1",
+        "postgres-array": "~2.0.0",
+        "postgres-bytea": "~1.0.0",
+        "postgres-date": "~1.0.4",
+        "postgres-interval": "^1.1.0"
+      },
+      "engines": {
+        "node": ">=4"
+      }
+    },
+    "node_modules/pgpass": {
+      "version": "1.0.5",
+      "resolved": "https://registry.npmjs.org/pgpass/-/pgpass-1.0.5.tgz",
+      "integrity": "sha512-FdW9r/jQZhSeohs1Z3sI1yxFQNFvMcnmfuj4WBMUTxOrAyLMaTcE1aAMBiTlbMNaXvBCQuVi0R7hd8udDSP7ug==",
+      "license": "MIT",
+      "dependencies": {
+        "split2": "^4.1.0"
+      }
+    },
     "node_modules/picocolors": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/picocolors/-/picocolors-1.1.1.tgz",
@@ -1448,6 +1551,45 @@
         "node": "^10 || ^12 || >=14"
       }
     },
+    "node_modules/postgres-array": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/postgres-array/-/postgres-array-2.0.0.tgz",
+      "integrity": "sha512-VpZrUqU5A69eQyW2c5CA1jtLecCsN2U/bD6VilrFDWq5+5UIEVO7nazS3TEcHf1zuPYO/sqGvUvW62g86RXZuA==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=4"
+      }
+    },
+    "node_modules/postgres-bytea": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/postgres-bytea/-/postgres-bytea-1.0.1.tgz",
+      "integrity": "sha512-5+5HqXnsZPE65IJZSMkZtURARZelel2oXUEO8rH83VS/hxH5vv1uHquPg5wZs8yMAfdv971IU+kcPUczi7NVBQ==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
+    "node_modules/postgres-date": {
+      "version": "1.0.7",
+      "resolved": "https://registry.npmjs.org/postgres-date/-/postgres-date-1.0.7.tgz",
+      "integrity": "sha512-suDmjLVQg78nMK2UZ454hAG+OAW+HQPZ6n++TNDUX+L0+uUlLywnoxJKDou51Zm+zTCjrCl0Nq6J9C5hP9vK/Q==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
+    "node_modules/postgres-interval": {
+      "version": "1.2.0",
+      "resolved": "https://registry.npmjs.org/postgres-interval/-/postgres-interval-1.2.0.tgz",
+      "integrity": "sha512-9ZhXKM/rw350N1ovuWHbGxnGh/SNJ4cnxHiM0rxE4VN41wsg8P8zWn9hv/buK00RP4WvlOyr/RBDiptyxVbkZQ==",
+      "license": "MIT",
+      "dependencies": {
+        "xtend": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/process-warning": {
       "version": "5.1.0",
       "resolved": "https://registry.npmjs.org/process-warning/-/process-warning-5.1.0.tgz",
@@ -1758,6 +1900,15 @@
       "integrity": "sha512-AsuCzffGHJybSaRrmr5eHr81mwJU3kjw6M+uprWvCXiNeN9SOGwQ3Jn8jb8m3Z6izVgknn1R0FTCEAP2QrLY/w==",
       "dev": true,
       "license": "MIT"
+    },
+    "node_modules/xtend": {
+      "version": "4.0.2",
+      "resolved": "https://registry.npmjs.org/xtend/-/xtend-4.0.2.tgz",
+      "integrity": "sha512-LKYU1iAXJXUgAXn9URjiu+MWhyUXHsvfp7mcuYm9dSUKK0/CjtrUwFAxD82/mCWbtLsGjFIad0wIsod4zrTAEQ==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.4"
+      }
     }
   }
 }
diff --git a/package.json b/package.json
index e2a9214..509ae19 100644
--- a/package.json
+++ b/package.json
@@ -16,17 +16,22 @@
     "test:unit": "node --test test/unit.test.ts",
     "test:functional": "node --test --test-concurrency=1 test/functional.test.ts test/contracts.test.ts",
     "test:e2e": "playwright test",
-    "test": "npm run test:unit && npm run test:functional"
+    "test": "npm run test:unit && npm run test:functional",
+    "db:up": "docker compose --project-name wse-fundamentals up -d --wait",
+    "db:down": "docker compose --project-name wse-fundamentals down",
+    "db:migrate": "node server/migrate.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
     "next": "16.3.3",
+    "pg": "8.16.3",
     "react": "19.2.8",
     "react-dom": "19.2.8"
   },
   "devDependencies": {
     "@playwright/test": "1.62.1",
     "@types/node": "24.13.3",
+    "@types/pg": "8.15.5",
     "@types/react": "19.2.18",
     "@types/react-dom": "19.2.5",
     "typescript": "5.9.3"
diff --git a/server/database.ts b/server/database.ts
new file mode 100644
index 0000000..e3c97d0
--- /dev/null
+++ b/server/database.ts
@@ -0,0 +1,24 @@
+import { Pool } from 'pg';
+
+export type DatabaseConfig = { connectionString: string; schema: string };
+
+export function databaseConfig(): DatabaseConfig {
+  return {
+    connectionString: process.env.DATABASE_URL ?? 'postgresql://monitor@127.0.0.1:15431/monitor',
+    schema: process.env.DATABASE_SCHEMA ?? 'public',
+  };
+}
+
+export function schemaIdentifier(schema: string): string {
+  if (!/^[a-z][a-z0-9_]{0,62}$/.test(schema)) throw new Error('Invalid database schema name.');
+  return `"${schema}"`;
+}
+
+export function databasePool(config: DatabaseConfig): Pool {
+  schemaIdentifier(config.schema);
+  return new Pool({
+    connectionString: config.connectionString,
+    options: `-c search_path=${config.schema} -c timezone=UTC`,
+    connectionTimeoutMillis: 2_000,
+  });
+}
diff --git a/server/migrate.ts b/server/migrate.ts
new file mode 100644
index 0000000..9d907e4
--- /dev/null
+++ b/server/migrate.ts
@@ -0,0 +1,61 @@
+import { createHash } from 'node:crypto';
+import { readFile } from 'node:fs/promises';
+import { pathToFileURL } from 'node:url';
+import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
+import type { DatabaseConfig } from './database.ts';
+
+export async function migrationFiles() {
+  const names = ['001_monitors.sql', '002_check_runs.sql'];
+  return Promise.all(names.map(async (version) => {
+    const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
+    return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
+  }));
+}
+
+export function checkMigrationHistory(
+  applied: { version: string; checksum: string }[],
+  expected: { version: string; checksum: string }[],
+  requireComplete: boolean,
+) {
+  if ((requireComplete && applied.length !== expected.length) || applied.length > expected.length ||
+    applied.some((row, index) => row.version !== expected[index]?.version || row.checksum !== expected[index]?.checksum)) {
+    throw new Error('Database migration history does not match this application. Run the documented migrations on a compatible schema.');
+  }
+}
+
+export async function migrate(config: DatabaseConfig = databaseConfig()): Promise<string[]> {
+  const migrations = await migrationFiles();
+  const pool = databasePool(config);
+  let client;
+  try {
+    client = await pool.connect();
+    // One explicit connection owns the entire DDL transaction. Run migrations serially.
+    await client.query('BEGIN');
+    await client.query(`CREATE SCHEMA IF NOT EXISTS ${schemaIdentifier(config.schema)}`);
+    await client.query(`SET LOCAL search_path TO ${schemaIdentifier(config.schema)}`);
+    await client.query(`CREATE TABLE IF NOT EXISTS schema_migrations (
+      version text PRIMARY KEY, checksum text NOT NULL, applied_at timestamptz(3) NOT NULL DEFAULT current_timestamp
+    )`);
+    const applied = await client.query<{ version: string; checksum: string }>('SELECT version, checksum FROM schema_migrations ORDER BY version');
+    checkMigrationHistory(applied.rows, migrations, false);
+    const added: string[] = [];
+    for (const migration of migrations.slice(applied.rows.length)) {
+      await client.query(migration.sql);
+      await client.query('INSERT INTO schema_migrations (version, checksum) VALUES ($1, $2)', [migration.version, migration.checksum]);
+      added.push(migration.version);
+    }
+    await client.query('COMMIT');
+    return added;
+  } catch (error) {
+    if (client) await client.query('ROLLBACK');
+    throw error;
+  } finally {
+    client?.release();
+    await pool.end();
+  }
+}
+
+if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
+  try { console.log(JSON.stringify({ applied: await migrate() })); }
+  catch { console.error('Database migration failed. Check the database configuration and migration history.'); process.exitCode = 1; }
+}
diff --git a/server/migrations/001_monitors.sql b/server/migrations/001_monitors.sql
new file mode 100644
index 0000000..0eb1ab9
--- /dev/null
+++ b/server/migrations/001_monitors.sql
@@ -0,0 +1,9 @@
+CREATE TABLE monitors (
+  id uuid PRIMARY KEY,
+  name text NOT NULL,
+  url text NOT NULL,
+  interval_seconds integer NOT NULL CHECK (interval_seconds BETWEEN 1 AND 86400),
+  enabled boolean NOT NULL,
+  created_at timestamptz(3) NOT NULL,
+  updated_at timestamptz(3) NOT NULL
+);
diff --git a/server/migrations/002_check_runs.sql b/server/migrations/002_check_runs.sql
new file mode 100644
index 0000000..9836d73
--- /dev/null
+++ b/server/migrations/002_check_runs.sql
@@ -0,0 +1,18 @@
+CREATE TABLE check_runs (
+  id uuid PRIMARY KEY,
+  monitor_id uuid NOT NULL REFERENCES monitors(id) ON DELETE CASCADE,
+  trigger text NOT NULL CHECK (trigger = 'MANUAL'),
+  state text NOT NULL CHECK (state IN ('SUCCEEDED', 'FAILED')),
+  http_status integer,
+  latency_ms integer NOT NULL CHECK (latency_ms >= 0),
+  failure_reason text,
+  started_at timestamptz(3) NOT NULL,
+  finished_at timestamptz(3) NOT NULL,
+  CHECK (finished_at >= started_at),
+  CHECK (
+    (state = 'SUCCEEDED' AND http_status BETWEEN 200 AND 299 AND http_status IS NOT NULL AND failure_reason IS NULL)
+    OR (state = 'FAILED' AND failure_reason = 'HTTP_STATUS' AND http_status IS NOT NULL AND (http_status < 200 OR http_status >= 300))
+    OR (state = 'FAILED' AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE') AND http_status IS NULL)
+  ),
+  CHECK (state != 'FAILED' OR failure_reason IS NOT NULL)
+);


