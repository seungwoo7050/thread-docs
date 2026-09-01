## `test: record E12 destination and resource evidence`

diff --git a/evidence/phase-1/E12/baseline.json b/evidence/phase-1/E12/baseline.json
new file mode 100644
index 0000000..e02d282
--- /dev/null
+++ b/evidence/phase-1/E12/baseline.json
@@ -0,0 +1,43 @@
+{
+  "start": "2e86db44f1f4f5dd605da18579647418deeefb01",
+  "baselineHead": "806c527a8d2d55b1b61bf959e9b3abe55e19c142",
+  "productMatchesStart": true,
+  "result": "REPRODUCED",
+  "hashes": {
+    "scenario.json": "18e66712591a57bcbfb265db7527754ef486fd9b8d9347fce7ea9433ddeb1d9a",
+    "fixture.ts": "ec1d9ec4596cd674ae53af79798b7e3ff89b960483ec34508c6e191d923d730b",
+    "baseline.mjs": "3b190e4a2a9eab0044b375ad3514269331014e68bebd92ceea2443a031fc0360"
+  },
+  "budget": {
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0
+  },
+  "observation": {
+    "method": "POST",
+    "path": "/monitors",
+    "monitor": {
+      "name": "Public destination fixture",
+      "url": "http://public.e12.test/ok",
+      "interval": 60,
+      "enabled": true
+    },
+    "status": 400,
+    "body": {
+      "error": {
+        "code": "INVALID_INPUT",
+        "message": "An absolute HTTP(S) URL on the configured fixture origin is required."
+      }
+    },
+    "monitorRows": 0,
+    "workerProcesses": 0,
+    "endpointRequests": 0,
+    "transport": "Fastify injection only; no worker or outbound HTTP client is started."
+  },
+  "decisiveFailure": "The fixture-only allowlist rejects the fixed general HTTP destination before persistence.",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 724
+}
diff --git a/evidence/phase-1/E12/cleanup.json b/evidence/phase-1/E12/cleanup.json
new file mode 100644
index 0000000..a8f417a
--- /dev/null
+++ b/evidence/phase-1/E12/cleanup.json
@@ -0,0 +1,14 @@
+{
+  "schemas": [
+    "public"
+  ],
+  "ownedSchemasRemoved": true,
+  "portsFree": [
+    4311,
+    4312,
+    4313,
+    4314
+  ],
+  "ownedWorkersAwaitedByTests": true,
+  "publicSchemaPreserved": true
+}
diff --git a/evidence/phase-1/E12/connect-timeout.json b/evidence/phase-1/E12/connect-timeout.json
new file mode 100644
index 0000000..c8779af
--- /dev/null
+++ b/evidence/phase-1/E12/connect-timeout.json
@@ -0,0 +1,34 @@
+{
+  "result": "PASS",
+  "timedOut": {
+    "id": "1b591899-3968-4a67-aa61-c6d70b2f74b7",
+    "monitorId": "12000000-0000-4000-8000-000000000001",
+    "trigger": "MANUAL",
+    "startedAt": "2026-08-28T05:45:51.609Z",
+    "finishedAt": "2026-08-28T05:45:52.109Z",
+    "state": "FAILED",
+    "httpStatus": null,
+    "latencyMs": 500,
+    "failureReason": "TIMEOUT"
+  },
+  "durationMs": 500,
+  "closedRequests": 1,
+  "outcomes": [
+    {
+      "status": 404,
+      "disposition": "permanent",
+      "calls": 1
+    },
+    {
+      "status": 429,
+      "disposition": "retryable",
+      "calls": 1
+    },
+    {
+      "status": 503,
+      "disposition": "retryable",
+      "calls": 1
+    }
+  ],
+  "automaticRetries": 0
+}
diff --git a/evidence/phase-1/E12/destinations.json b/evidence/phase-1/E12/destinations.json
new file mode 100644
index 0000000..b8c0ed4
--- /dev/null
+++ b/evidence/phase-1/E12/destinations.json
@@ -0,0 +1,321 @@
+{
+  "result": "PASS",
+  "decisions": [
+    {
+      "input": {
+        "url": "http://public.e12.test/ok",
+        "addresses": [
+          "93.184.216.34"
+        ],
+        "allowed": true
+      },
+      "result": {
+        "id": "14f35b2d-1564-40c4-ad83-0d2b3828dd34",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.602Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 3,
+        "failureReason": null
+      },
+      "resolutions": 1,
+      "connectorCalls": 1,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "https://public.e12.test/ok",
+        "addresses": [
+          "2606:4700:4700::1111"
+        ],
+        "allowed": true
+      },
+      "result": {
+        "id": "4dd7cc51-2907-4bdf-930f-477772eac9f8",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null
+      },
+      "resolutions": 1,
+      "connectorCalls": 1,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://user:fixture@public.e12.test/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "44d30145-01f2-446d-bcab-45ff7b2d8edb",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "file:///fixture",
+        "allowed": false
+      },
+      "result": {
+        "id": "2d3d516a-8213-48d2-b892-0ddc7464c36a",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://localhost/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "abe52118-5011-4bed-85c2-99eef63a7506",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://127.0.0.1/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "21236d3a-f974-4087-9a94-c47b0914b791",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.605Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://[::1]/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "8ae26b66-1dce-458b-9494-e1d1e496e77a",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.605Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://10.0.0.1/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "92daa92d-6cb5-4c9f-873b-edfaab87440a",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://[fc00::1]/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "acae080d-bd51-4699-bf0a-dc789d1a8ad2",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://169.254.169.254/ok",
+        "allowed": false,
+        "neverContact": true
+      },
+      "result": {
+        "id": "f54b7412-83d0-44fe-87f4-857a7e8dfe7c",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://[fe80::1]/ok",
+        "allowed": false,
+        "neverContact": true
+      },
+      "result": {
+        "id": "876c7ad6-6df7-4c10-8181-c7fc168fc273",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://[::ffff:127.0.0.1]/ok",
+        "allowed": false
+      },
+      "result": {
+        "id": "9b13d645-c4ba-4cdf-b992-a34f3e829e6d",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 0,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://private.e12.test/ok",
+        "addresses": [
+          "10.0.0.1"
+        ],
+        "allowed": false
+      },
+      "result": {
+        "id": "b3405364-649a-4ea6-8343-6a0338960fd8",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 1,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    },
+    {
+      "input": {
+        "url": "http://mixed.e12.test/ok",
+        "addresses": [
+          "93.184.216.34",
+          "10.0.0.1"
+        ],
+        "allowed": false
+      },
+      "result": {
+        "id": "f621f819-303c-4172-bba0-00feb684721f",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:51.606Z",
+        "finishedAt": "2026-08-28T05:45:51.606Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION"
+      },
+      "resolutions": 1,
+      "connectorCalls": 0,
+      "unsafeConnectorCalls": 0
+    }
+  ],
+  "redirect": {
+    "id": "296795d9-969e-4abc-b956-c828f14ec150",
+    "monitorId": "12000000-0000-4000-8000-000000000001",
+    "trigger": "MANUAL",
+    "startedAt": "2026-08-28T05:45:51.606Z",
+    "finishedAt": "2026-08-28T05:45:51.606Z",
+    "state": "ABORTED",
+    "httpStatus": null,
+    "latencyMs": null,
+    "failureReason": "UNSAFE_DESTINATION"
+  },
+  "initialRedirectConnections": 1,
+  "rebinding": {
+    "resolverCalls": 1,
+    "connectedAddresses": [
+      "93.184.216.34"
+    ],
+    "unsafeConnectorCalls": 0
+  },
+  "productionIgnoresFixtureOverride": true,
+  "actualNetworkConnections": 0
+}
diff --git a/evidence/phase-1/E12/resources.json b/evidence/phase-1/E12/resources.json
new file mode 100644
index 0000000..b3ba0d9
--- /dev/null
+++ b/evidence/phase-1/E12/resources.json
@@ -0,0 +1,179 @@
+{
+  "result": "PASS",
+  "baselineNowStatus": 201,
+  "productionLocalStatus": 400,
+  "results": [
+    {
+      "input": {
+        "path": "/slow",
+        "state": "FAILED",
+        "httpStatus": null,
+        "reason": "TIMEOUT",
+        "requests": 1
+      },
+      "check": {
+        "id": "6324e9e3-3d3c-465b-93e5-92e28d96b582",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:52.795Z",
+        "finishedAt": "2026-08-28T05:45:53.298Z",
+        "state": "FAILED",
+        "httpStatus": null,
+        "latencyMs": 503,
+        "failureReason": "TIMEOUT"
+      },
+      "durationMs": 503,
+      "paths": [
+        "/slow"
+      ],
+      "consumedBodyBytes": 0,
+      "bodyObservation": "real IncomingMessage data observer",
+      "openSocketsAfter": 0
+    },
+    {
+      "input": {
+        "path": "/large",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "reason": null,
+        "requests": 1
+      },
+      "check": {
+        "id": "8aeb99b8-f768-471a-92f1-0609c496e830",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:53.300Z",
+        "finishedAt": "2026-08-28T05:45:53.304Z",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 3,
+        "failureReason": null
+      },
+      "durationMs": 4,
+      "paths": [
+        "/large"
+      ],
+      "consumedBodyBytes": 0,
+      "bodyObservation": "real IncomingMessage data observer",
+      "openSocketsAfter": 0
+    },
+    {
+      "input": {
+        "path": "/redirect/0",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "reason": "REDIRECT_LIMIT",
+        "requests": 4
+      },
+      "check": {
+        "id": "4c4f61b4-7c68-4f59-aa04-4d29cbebb12d",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:53.304Z",
+        "finishedAt": "2026-08-28T05:45:53.310Z",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "REDIRECT_LIMIT"
+      },
+      "durationMs": 6,
+      "paths": [
+        "/redirect/0",
+        "/redirect/1",
+        "/redirect/2",
+        "/redirect/3"
+      ],
+      "consumedBodyBytes": 0,
+      "bodyObservation": "real IncomingMessage data observer",
+      "openSocketsAfter": 0
+    },
+    {
+      "input": {
+        "path": "/private",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "reason": "UNSAFE_DESTINATION",
+        "requests": 1,
+        "guardRequests": 0
+      },
+      "check": {
+        "id": "4e0db145-7535-4ade-9230-f8ffe39e2fda",
+        "monitorId": "d6e431dc-9977-4801-8b64-ea2b3deb7eab",
+        "trigger": "MANUAL",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": "UNSAFE_DESTINATION",
+        "startedAt": "2026-08-28T05:45:53.476Z",
+        "finishedAt": "2026-08-28T05:45:53.485Z"
+      },
+      "durationMs": 452,
+      "paths": [
+        "/private"
+      ],
+      "consumedBodyBytes": 0,
+      "bodyObservation": "worker cancels redirect before any body consumption; source boundary",
+      "openSocketsAfter": 0
+    },
+    {
+      "input": {
+        "path": "/timed/0",
+        "state": "FAILED",
+        "httpStatus": null,
+        "reason": "TIMEOUT",
+        "requests": 4
+      },
+      "check": {
+        "id": "1092404b-0cb5-46dd-8ce3-1d3abe631e14",
+        "monitorId": "12000000-0000-4000-8000-000000000001",
+        "trigger": "MANUAL",
+        "startedAt": "2026-08-28T05:45:53.762Z",
+        "finishedAt": "2026-08-28T05:45:55.263Z",
+        "state": "FAILED",
+        "httpStatus": null,
+        "latencyMs": 1501,
+        "failureReason": "TIMEOUT"
+      },
+      "durationMs": 1501,
+      "paths": [
+        "/timed/0",
+        "/timed/1",
+        "/timed/2",
+        "/timed/3"
+      ],
+      "consumedBodyBytes": 0,
+      "bodyObservation": "real IncomingMessage data observer",
+      "openSocketsAfter": 0
+    }
+  ],
+  "workerEvidence": {
+    "workerPid": 9696,
+    "observerPid": 9683,
+    "queued": {
+      "id": "4e0db145-7535-4ade-9230-f8ffe39e2fda",
+      "monitorId": "d6e431dc-9977-4801-8b64-ea2b3deb7eab",
+      "trigger": "MANUAL",
+      "state": "QUEUED",
+      "httpStatus": null,
+      "latencyMs": null,
+      "failureReason": null,
+      "startedAt": null,
+      "finishedAt": null
+    },
+    "terminal": {
+      "id": "4e0db145-7535-4ade-9230-f8ffe39e2fda",
+      "monitorId": "d6e431dc-9977-4801-8b64-ea2b3deb7eab",
+      "trigger": "MANUAL",
+      "state": "ABORTED",
+      "httpStatus": null,
+      "latencyMs": null,
+      "failureReason": "UNSAFE_DESTINATION",
+      "startedAt": "2026-08-28T05:45:53.476Z",
+      "finishedAt": "2026-08-28T05:45:53.485Z"
+    },
+    "sameIdentityReplay": true,
+    "rows": 1
+  },
+  "guardRequests": 0,
+  "automaticRetries": 0
+}
diff --git a/evidence/phase-1/E12/verification.json b/evidence/phase-1/E12/verification.json
new file mode 100644
index 0000000..5336299
--- /dev/null
+++ b/evidence/phase-1/E12/verification.json
@@ -0,0 +1,129 @@
+{
+  "thread": "E12",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "attempt": 1,
+  "start": "2e86db44f1f4f5dd605da18579647418deeefb01",
+  "freezeCommit": "806c527a8d2d55b1b61bf959e9b3abe55e19c142",
+  "implementationCommit": "c62ceeadf98b19bde7afd834e00259f3cf687254",
+  "status": "AUTHOR_GATES_PASS_AWAITING_ROOT_INDEPENDENT_VERIFICATION",
+  "frozenHashes": {
+    "scenario.json": "18e66712591a57bcbfb265db7527754ef486fd9b8d9347fce7ea9433ddeb1d9a",
+    "fixture.ts": "ec1d9ec4596cd674ae53af79798b7e3ff89b960483ec34508c6e191d923d730b",
+    "baseline.mjs": "3b190e4a2a9eab0044b375ad3514269331014e68bebd92ceea2443a031fc0360",
+    "rootResultSemanticsSupplement": "3846e8660512ed0a13bceb783c9e6553fcfa614ab61a1012ac9c1b99a811baf9"
+  },
+  "invocations": [
+    {
+      "command": "fnm exec --using 24.19.0 node evidence/phase-1/E12/baseline.mjs",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "REPRODUCED",
+      "durationMs": 724,
+      "toolWallSeconds": 0.945006875,
+      "artifact": "baseline.json",
+      "observation": "Unchanged START rejects the exact public-shaped Monitor with400 INVALID_INPUT, persists0 rows and starts no worker/outbound request."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "PASS",
+      "realSeconds": 2.68
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:outbound",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "PASS",
+      "passed": 3,
+      "nativeDurationMs": 3868.758125,
+      "realSeconds": 4.02,
+      "artifacts": ["destinations.json", "connect-timeout.json", "resources.json"]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "PASS",
+      "passed": 21,
+      "nativeDurationMs": 1133.163542,
+      "realSeconds": 1.28
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "PASS",
+      "passed": 15,
+      "nativeDurationMs": 10157.061542,
+      "realSeconds": 10.31
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "ordinal": 1,
+      "exitCode": 0,
+      "result": "PASS",
+      "passed": 10,
+      "nativeDurationMs": 8005.183792,
+      "realSeconds": 8.16
+    }
+  ],
+  "resultBoundaries": {
+    "actualPublicOrMetadataRequests": 0,
+    "dnsAndPublicConnectorProof": "Deterministic resolver and connector stubs; production connector uses the validated numeric address while preserving Host/TLS authority.",
+    "bodyProof": "Real IncomingMessage data observers consumed0 bytes for slow/large/redirect/total cases. Large65537-byte200 remains SUCCEEDED. The separate worker private-redirect case uses the inspected same header-close boundary, not a cross-process byte counter. No zero-TCP-buffering claim.",
+    "policyWorker": "Actual separate CLI worker9696 persisted the accepted CheckRun as ABORTED/null/UNSAFE_DESTINATION; same key returned that same terminal row, guard requests0.",
+    "fixedResources": "connect500/read500/total1500ms, redirects3, body ceiling65536; no parameter changes after baseline.",
+    "observedResourceDurationsMs": { "slow": 503, "large": 4, "redirectLimit": 6, "workerPrivateRedirect": 452, "total": 1501 },
+    "timingThreshold": "Frozen1750ms completion ceiling includes250ms scheduling allowance; not a performance benchmark."
+  },
+  "consumerAdaptations": [
+    "The same historical /redirect fixture now expects final200 and one /ok hit. Its fixture and historical evidence bytes are unchanged.",
+    "Direct unsafe checkMonitor now asserts ABORTED/null/UNSAFE_DESTINATION; the API still rejects unsafe input with400 and the guard remains untouched.",
+    "Real local test consumers pass the exact fixture origin explicitly. Worker/browser API bootstrap sets NODE_ENV=test plus WSE_TEST_FIXTURE_ORIGIN; production defaults and production environment ignore it.",
+    "The fresh migration chain expectation appends009; old migration checksums, records, pins and legacy-null assertions remain unchanged."
+  ],
+  "nonAcceptanceInspectionIssues": [
+    { "kind": "read-path miss", "count": 1, "detail": "A batched rg named absent .env.example and exited2; its other source results were read. No test or mutation occurred." },
+    { "kind": "documentation retrieval", "count": 1, "detail": "The web tool returned Internal Error for the exact v24.19.0 HTTP documentation URL. Official24.x/IANA references and the installed pinned runtime Agent source were inspected; no package or endpoint probe was run." },
+    { "kind": "output truncation", "count": 1, "detail": "The initial combined documentation output exceeded the tool output budget. Relevant IANA and pinned runtime source details were subsequently inspected." }
+  ],
+  "cleanup": {
+    "artifact": "cleanup.json",
+    "schemas": ["public"],
+    "ownedSchemas": ["e12_baseline", "e12_outbound"],
+    "legacySchemas": "Only the declared existing unit/API/PG regression helpers were used.",
+    "portsFree": [4311, 4312, 4313, 4314],
+    "ownedWorkersAwaitedByTests": true,
+    "publicSchemaAndVolumesPreserved": true
+  },
+  "preservation": {
+    "allEarlierEvidenceUnchanged": true,
+    "migrations001Through008Unchanged": true,
+    "packageLockAndDependencyPinsUnchanged": true,
+    "originalE12FreezeUnchanged": true,
+    "mainSpecIndexTagsUnchangedByAgent": true
+  },
+  "budget": {
+    "baselineInvocations": 1,
+    "typecheckInvocations": 1,
+    "outboundInvocations": 1,
+    "unitInvocations": 1,
+    "functionalInvocations": 1,
+    "integrationInvocations": 1,
+    "ownershipInvocations": 0,
+    "productionBuildInvocations": 0,
+    "browserInvocations": 0,
+    "executionInvocations": 0,
+    "e11RecoveryInvocations": 0,
+    "acceptanceFailures": 0,
+    "freshRepairsUsed": 0,
+    "maximumFreshRepairs": 2,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "permissionDenials": 0
+  },
+  "unresolved": ["Root independent final production/browser/worker acceptance and candidate verification remain outside this author report."]
+}
