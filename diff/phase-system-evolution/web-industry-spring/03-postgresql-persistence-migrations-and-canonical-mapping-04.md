## `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록`

diff --git a/evidence/E03/fixed-persistence.json b/evidence/E03/fixed-persistence.json
new file mode 100644
index 0000000..05ebb0f
--- /dev/null
+++ b/evidence/E03/fixed-persistence.json
@@ -0,0 +1,845 @@
+{
+  "label": "fixed",
+  "requests": [
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": []
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors",
+      "status": 201,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Persisted A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 60,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:04.700918Z"
+          },
+          "latestCheck": null
+        }
+      },
+      "elapsedSeconds": 0.045
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors",
+      "status": 201,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+            "name": "Persisted B",
+            "url": "http://127.0.0.1:4321/fail",
+            "interval": 120,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.732591Z",
+            "updatedAt": "2026-08-28T00:18:04.732591Z"
+          },
+          "latestCheck": null
+        }
+      },
+      "elapsedSeconds": 0.007
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "162964c4-d4ac-4582-a515-b150836703c6",
+          "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 7,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:18:04.752799Z",
+          "finishedAt": "2026-08-28T00:18:04.760545Z"
+        }
+      },
+      "elapsedSeconds": 0.033
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+          "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 0,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:18:04.774072Z",
+          "finishedAt": "2026-08-28T00:18:04.774937Z"
+        }
+      },
+      "elapsedSeconds": 0.01
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+          "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+          "trigger": "MANUAL",
+          "state": "FAILED",
+          "httpStatus": 503,
+          "latencyMs": 0,
+          "failureReason": "HTTP_STATUS",
+          "startedAt": "2026-08-28T00:18:04.784689Z",
+          "finishedAt": "2026-08-28T00:18:04.785689Z"
+        }
+      },
+      "elapsedSeconds": 0.011
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "monitor": {
+              "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "name": "Persisted A",
+              "url": "http://127.0.0.1:4321/ok",
+              "interval": 60,
+              "enabled": true,
+              "createdAt": "2026-08-28T00:18:04.700918Z",
+              "updatedAt": "2026-08-28T00:18:04.700918Z"
+            },
+            "latestCheck": {
+              "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+              "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "trigger": "MANUAL",
+              "state": "SUCCEEDED",
+              "httpStatus": 200,
+              "latencyMs": 0,
+              "failureReason": null,
+              "startedAt": "2026-08-28T00:18:04.774072Z",
+              "finishedAt": "2026-08-28T00:18:04.774937Z"
+            }
+          },
+          {
+            "monitor": {
+              "id": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+              "name": "Persisted B",
+              "url": "http://127.0.0.1:4321/fail",
+              "interval": 120,
+              "enabled": true,
+              "createdAt": "2026-08-28T00:18:04.732591Z",
+              "updatedAt": "2026-08-28T00:18:04.732591Z"
+            },
+            "latestCheck": {
+              "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+              "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+              "trigger": "MANUAL",
+              "state": "FAILED",
+              "httpStatus": 503,
+              "latencyMs": 0,
+              "failureReason": "HTTP_STATUS",
+              "startedAt": "2026-08-28T00:18:04.784689Z",
+              "finishedAt": "2026-08-28T00:18:04.785689Z"
+            }
+          }
+        ]
+      },
+      "elapsedSeconds": 0.011
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          },
+          {
+            "id": "162964c4-d4ac-4582-a515-b150836703c6",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 7,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.752799Z",
+            "finishedAt": "2026-08-28T00:18:04.760545Z"
+          }
+        ]
+      },
+      "elapsedSeconds": 0.017
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks/162964c4-d4ac-4582-a515-b150836703c6",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "162964c4-d4ac-4582-a515-b150836703c6",
+          "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 7,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:18:04.752799Z",
+          "finishedAt": "2026-08-28T00:18:04.760545Z"
+        }
+      },
+      "elapsedSeconds": 0.016
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks/3cc8b09e-9de4-42d7-87b3-938accd43faf",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+          "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 0,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:18:04.774072Z",
+          "finishedAt": "2026-08-28T00:18:04.774937Z"
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Persisted A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 60,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:04.700918Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617/checks",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+            "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+            "trigger": "MANUAL",
+            "state": "FAILED",
+            "httpStatus": 503,
+            "latencyMs": 0,
+            "failureReason": "HTTP_STATUS",
+            "startedAt": "2026-08-28T00:18:04.784689Z",
+            "finishedAt": "2026-08-28T00:18:04.785689Z"
+          }
+        ]
+      },
+      "elapsedSeconds": 0.01
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617/checks/2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+          "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+          "trigger": "MANUAL",
+          "state": "FAILED",
+          "httpStatus": 503,
+          "latencyMs": 0,
+          "failureReason": "HTTP_STATUS",
+          "startedAt": "2026-08-28T00:18:04.784689Z",
+          "finishedAt": "2026-08-28T00:18:04.785689Z"
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+            "name": "Persisted B",
+            "url": "http://127.0.0.1:4321/fail",
+            "interval": 120,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.732591Z",
+            "updatedAt": "2026-08-28T00:18:04.732591Z"
+          },
+          "latestCheck": {
+            "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+            "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+            "trigger": "MANUAL",
+            "state": "FAILED",
+            "httpStatus": 503,
+            "latencyMs": 0,
+            "failureReason": "HTTP_STATUS",
+            "startedAt": "2026-08-28T00:18:04.784689Z",
+            "finishedAt": "2026-08-28T00:18:04.785689Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.01
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors",
+      "status": 400,
+      "wire": {
+        "error": {
+          "code": "INVALID_INPUT",
+          "message": "Name must not contain a NUL character."
+        }
+      },
+      "elapsedSeconds": 0.018
+    },
+    {
+      "method": "PUT",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 400,
+      "wire": {
+        "error": {
+          "code": "INVALID_INPUT",
+          "message": "Name must not contain a NUL character."
+        }
+      },
+      "elapsedSeconds": 0.004
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Persisted A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 60,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:04.700918Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.011
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "monitor": {
+              "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "name": "Persisted A",
+              "url": "http://127.0.0.1:4321/ok",
+              "interval": 60,
+              "enabled": true,
+              "createdAt": "2026-08-28T00:18:04.700918Z",
+              "updatedAt": "2026-08-28T00:18:04.700918Z"
+            },
+            "latestCheck": {
+              "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+              "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "trigger": "MANUAL",
+              "state": "SUCCEEDED",
+              "httpStatus": 200,
+              "latencyMs": 0,
+              "failureReason": null,
+              "startedAt": "2026-08-28T00:18:04.774072Z",
+              "finishedAt": "2026-08-28T00:18:04.774937Z"
+            }
+          },
+          {
+            "monitor": {
+              "id": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+              "name": "Persisted B",
+              "url": "http://127.0.0.1:4321/fail",
+              "interval": 120,
+              "enabled": true,
+              "createdAt": "2026-08-28T00:18:04.732591Z",
+              "updatedAt": "2026-08-28T00:18:04.732591Z"
+            },
+            "latestCheck": {
+              "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+              "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+              "trigger": "MANUAL",
+              "state": "FAILED",
+              "httpStatus": 503,
+              "latencyMs": 0,
+              "failureReason": "HTTP_STATUS",
+              "startedAt": "2026-08-28T00:18:04.784689Z",
+              "finishedAt": "2026-08-28T00:18:04.785689Z"
+            }
+          }
+        ]
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "PUT",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.485463Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.029
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.485463Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "PUT",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": false,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.519763Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": false,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.519763Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.007
+    },
+    {
+      "method": "PUT",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.536333Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "status": 200,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "name": "Updated A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 90,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:18:04.700918Z",
+            "updatedAt": "2026-08-28T00:18:08.536333Z"
+          },
+          "latestCheck": {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          }
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "DELETE",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+      "status": 200,
+      "wire": {
+        "data": null
+      },
+      "elapsedSeconds": 0.008
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+      "status": 404,
+      "wire": {
+        "error": {
+          "code": "NOT_FOUND",
+          "message": "Resource not found."
+        }
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617/checks",
+      "status": 404,
+      "wire": {
+        "error": {
+          "code": "NOT_FOUND",
+          "message": "Resource not found."
+        }
+      },
+      "elapsedSeconds": 0.005
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/9ae30b00-b6ed-4d03-8f7c-2b85fda88617/checks/2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+      "status": 404,
+      "wire": {
+        "error": {
+          "code": "NOT_FOUND",
+          "message": "Resource not found."
+        }
+      },
+      "elapsedSeconds": 0.004
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "monitor": {
+              "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "name": "Updated A",
+              "url": "http://127.0.0.1:4321/ok",
+              "interval": 90,
+              "enabled": true,
+              "createdAt": "2026-08-28T00:18:04.700918Z",
+              "updatedAt": "2026-08-28T00:18:08.536333Z"
+            },
+            "latestCheck": {
+              "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+              "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+              "trigger": "MANUAL",
+              "state": "SUCCEEDED",
+              "httpStatus": 200,
+              "latencyMs": 0,
+              "failureReason": null,
+              "startedAt": "2026-08-28T00:18:04.774072Z",
+              "finishedAt": "2026-08-28T00:18:04.774937Z"
+            }
+          }
+        ]
+      },
+      "elapsedSeconds": 0.009
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors/427e7486-5acb-4c8b-bb71-385bbbc8a726/checks",
+      "status": 200,
+      "wire": {
+        "data": [
+          {
+            "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 0,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.774072Z",
+            "finishedAt": "2026-08-28T00:18:04.774937Z"
+          },
+          {
+            "id": "162964c4-d4ac-4582-a515-b150836703c6",
+            "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+            "trigger": "MANUAL",
+            "state": "SUCCEEDED",
+            "httpStatus": 200,
+            "latencyMs": 7,
+            "failureReason": null,
+            "startedAt": "2026-08-28T00:18:04.752799Z",
+            "finishedAt": "2026-08-28T00:18:04.760545Z"
+          }
+        ]
+      },
+      "elapsedSeconds": 0.007
+    }
+  ],
+  "monitors": [
+    {
+      "id": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4321/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-28T00:18:04.700918Z",
+      "updatedAt": "2026-08-28T00:18:04.700918Z"
+    },
+    {
+      "id": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4321/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-28T00:18:04.732591Z",
+      "updatedAt": "2026-08-28T00:18:04.732591Z"
+    }
+  ],
+  "checks": [
+    {
+      "id": "162964c4-d4ac-4582-a515-b150836703c6",
+      "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 7,
+      "failureReason": null,
+      "startedAt": "2026-08-28T00:18:04.752799Z",
+      "finishedAt": "2026-08-28T00:18:04.760545Z"
+    },
+    {
+      "id": "3cc8b09e-9de4-42d7-87b3-938accd43faf",
+      "monitorId": "427e7486-5acb-4c8b-bb71-385bbbc8a726",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 0,
+      "failureReason": null,
+      "startedAt": "2026-08-28T00:18:04.774072Z",
+      "finishedAt": "2026-08-28T00:18:04.774937Z"
+    },
+    {
+      "id": "2e16a93a-887d-4d86-bf7d-96ff83006bc4",
+      "monitorId": "9ae30b00-b6ed-4d03-8f7c-2b85fda88617",
+      "trigger": "MANUAL",
+      "state": "FAILED",
+      "httpStatus": 503,
+      "latencyMs": 0,
+      "failureReason": "HTTP_STATUS",
+      "startedAt": "2026-08-28T00:18:04.784689Z",
+      "finishedAt": "2026-08-28T00:18:04.785689Z"
+    }
+  ],
+  "processes": [
+    {
+      "role": "fixture",
+      "pid": 80835,
+      "startedAt": "2026-08-28T00:18:00.725Z",
+      "logPath": "output/e03/fixed-fixture.log",
+      "readyAt": "2026-08-28T00:18:00.855Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 0,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:18:08.639Z",
+      "exitAwaited": true
+    },
+    {
+      "role": "api-first",
+      "pid": 80847,
+      "startedAt": "2026-08-28T00:18:00.860Z",
+      "logPath": "output/e03/fixed-api-first.log",
+      "readyAt": "2026-08-28T00:18:04.673Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 143,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:18:04.839Z",
+      "exitAwaited": true,
+      "exitObserved": true
+    },
+    {
+      "role": "api-fresh",
+      "pid": 80865,
+      "startedAt": "2026-08-28T00:18:04.841Z",
+      "logPath": "output/e03/fixed-api-fresh.log",
+      "readyAt": "2026-08-28T00:18:08.345Z",
+      "ownedStartupMarkerObserved": true,
+      "exitCode": 143,
+      "signal": null,
+      "exitedAt": "2026-08-28T00:18:08.637Z",
+      "exitAwaited": true
+    }
+  ],
+  "portChecks": [
+    {
+      "port": 4321,
+      "checkedAt": "2026-08-28T00:18:00.722Z",
+      "result": "free"
+    },
+    {
+      "port": 4322,
+      "checkedAt": "2026-08-28T00:18:00.724Z",
+      "result": "free"
+    },
+    {
+      "port": 4322,
+      "checkedAt": "2026-08-28T00:18:04.840Z",
+      "result": "free"
+    }
+  ],
+  "result": "PASS: all historical records survive a new API process; lifecycle and NUL boundary hold",
+  "elapsedSeconds": 7.921
+}
diff --git a/evidence/E03/generated-sql.txt b/evidence/E03/generated-sql.txt
new file mode 100644
index 0000000..5f83888
--- /dev/null
+++ b/evidence/E03/generated-sql.txt
@@ -0,0 +1,31 @@
+SqlEvent[sql=insert into e03_mapping.monitors (created_at,enabled,interval_seconds,name,updated_at,url,id) values (?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=false]
+SqlEvent[sql=insert into e03_mapping.check_runs (failure_reason,finished_at,http_status,latency_ms,monitor_id,started_at,state,trigger_kind,id) values (?,?,?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=false]
+SqlEvent[sql=insert into e03_mapping.check_runs (failure_reason,finished_at,http_status,latency_ms,monitor_id,started_at,state,trigger_kind,id) values (?,?,?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=insert into e03_mapping.monitors (created_at,enabled,interval_seconds,name,updated_at,url,id) values (?,?,?,?,?,?,?), transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.monitor_id=? order by cre1_0.finished_at desc,cre1_0.id desc, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.id=? and cre1_0.monitor_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.id=? and cre1_0.monitor_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.monitor_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 order by me1_0.created_at,me1_0.id, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.monitor_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.id=? and cre1_0.monitor_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.id=? and cre1_0.monitor_id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.monitor_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=false]
+SqlEvent[sql=update e03_mapping.monitors set created_at=?,enabled=?,interval_seconds=?,name=?,updated_at=?,url=? where id=?, transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e03_mapping.check_runs cre1_0 where cre1_0.monitor_id=? order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=false]
+SqlEvent[sql=delete from e03_mapping.monitors where id=?, transaction=true, readOnly=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.updated_at,me1_0.url from e03_mapping.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=true]
diff --git a/evidence/E03/maven-dependencies.txt b/evidence/E03/maven-dependencies.txt
new file mode 100644
index 0000000..4530cc8
--- /dev/null
+++ b/evidence/E03/maven-dependencies.txt
@@ -0,0 +1,92 @@
+dev.evolution:monitor-api:jar:0.0.1
++- org.springframework.boot:spring-boot-starter-web:jar:3.5.16:compile
+|  +- org.springframework.boot:spring-boot-starter:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot-autoconfigure:jar:3.5.16:compile
+|  |  +- org.springframework.boot:spring-boot-starter-logging:jar:3.5.16:compile
+|  |  |  +- ch.qos.logback:logback-classic:jar:1.5.34:compile
+|  |  |  |  \- ch.qos.logback:logback-core:jar:1.5.34:compile
+|  |  |  +- org.apache.logging.log4j:log4j-to-slf4j:jar:2.24.3:compile
+|  |  |  |  \- org.apache.logging.log4j:log4j-api:jar:2.24.3:compile
+|  |  |  \- org.slf4j:jul-to-slf4j:jar:2.0.18:compile
+|  |  +- jakarta.annotation:jakarta.annotation-api:jar:2.1.1:compile
+|  |  \- org.yaml:snakeyaml:jar:2.4:compile
+|  +- org.springframework.boot:spring-boot-starter-json:jar:3.5.16:compile
+|  |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.21.4:compile
+|  |  |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.21:compile
+|  |  |  \- com.fasterxml.jackson.core:jackson-core:jar:2.21.4:compile
+|  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jdk8:jar:2.21.4:compile
+|  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jsr310:jar:2.21.4:compile
+|  |  \- com.fasterxml.jackson.module:jackson-module-parameter-names:jar:2.21.4:compile
+|  +- org.springframework.boot:spring-boot-starter-tomcat:jar:3.5.16:compile
+|  |  +- org.apache.tomcat.embed:tomcat-embed-core:jar:10.1.55:compile
+|  |  +- org.apache.tomcat.embed:tomcat-embed-el:jar:10.1.55:compile
+|  |  \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:10.1.55:compile
+|  +- org.springframework:spring-web:jar:6.2.19:compile
+|  |  +- org.springframework:spring-beans:jar:6.2.19:compile
+|  |  \- io.micrometer:micrometer-observation:jar:1.15.12:compile
+|  |     \- io.micrometer:micrometer-commons:jar:1.15.12:compile
+|  \- org.springframework:spring-webmvc:jar:6.2.19:compile
+|     +- org.springframework:spring-aop:jar:6.2.19:compile
+|     +- org.springframework:spring-context:jar:6.2.19:compile
+|     \- org.springframework:spring-expression:jar:6.2.19:compile
++- org.springframework.boot:spring-boot-starter-data-jpa:jar:3.5.16:compile
+|  +- org.springframework.boot:spring-boot-starter-jdbc:jar:3.5.16:compile
+|  |  +- com.zaxxer:HikariCP:jar:6.3.3:compile
+|  |  \- org.springframework:spring-jdbc:jar:6.2.19:compile
+|  +- org.hibernate.orm:hibernate-core:jar:6.6.53.Final:compile
+|  |  +- jakarta.persistence:jakarta.persistence-api:jar:3.1.0:compile
+|  |  +- jakarta.transaction:jakarta.transaction-api:jar:2.0.1:compile
+|  |  +- org.jboss.logging:jboss-logging:jar:3.6.3.Final:runtime
+|  |  +- org.hibernate.common:hibernate-commons-annotations:jar:7.0.3.Final:runtime
+|  |  +- io.smallrye:jandex:jar:3.2.0:runtime
+|  |  +- com.fasterxml:classmate:jar:1.7.3:runtime
+|  |  +- net.bytebuddy:byte-buddy:jar:1.17.8:runtime
+|  |  +- org.glassfish.jaxb:jaxb-runtime:jar:4.0.9:runtime
+|  |  |  \- org.glassfish.jaxb:jaxb-core:jar:4.0.9:runtime
+|  |  |     +- org.eclipse.angus:angus-activation:jar:2.0.3:runtime
+|  |  |     +- org.glassfish.jaxb:txw2:jar:4.0.9:runtime
+|  |  |     \- com.sun.istack:istack-commons-runtime:jar:4.1.2:runtime
+|  |  +- jakarta.inject:jakarta.inject-api:jar:2.0.1:runtime
+|  |  \- org.antlr:antlr4-runtime:jar:4.13.2:compile
+|  +- org.springframework.data:spring-data-jpa:jar:3.5.13:compile
+|  |  +- org.springframework.data:spring-data-commons:jar:3.5.13:compile
+|  |  +- org.springframework:spring-orm:jar:6.2.19:compile
+|  |  +- org.springframework:spring-tx:jar:6.2.19:compile
+|  |  \- org.slf4j:slf4j-api:jar:2.0.18:compile
+|  \- org.springframework:spring-aspects:jar:6.2.19:compile
+|     \- org.aspectj:aspectjweaver:jar:1.9.25.1:compile
++- org.flywaydb:flyway-database-postgresql:jar:11.7.2:compile
+|  \- org.flywaydb:flyway-core:jar:11.7.2:compile
+|     \- com.fasterxml.jackson.dataformat:jackson-dataformat-toml:jar:2.21.4:compile
++- org.postgresql:postgresql:jar:42.7.11:runtime
+\- org.springframework.boot:spring-boot-starter-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test-autoconfigure:jar:3.5.16:test
+   +- com.jayway.jsonpath:json-path:jar:2.9.0:test
+   +- jakarta.xml.bind:jakarta.xml.bind-api:jar:4.0.5:runtime
+   |  \- jakarta.activation:jakarta.activation-api:jar:2.1.4:runtime
+   +- net.minidev:json-smart:jar:2.5.2:test
+   |  \- net.minidev:accessors-smart:jar:2.5.2:test
+   |     \- org.ow2.asm:asm:jar:9.7.1:test
+   +- org.assertj:assertj-core:jar:3.27.7:test
+   +- org.awaitility:awaitility:jar:4.2.2:test
+   +- org.hamcrest:hamcrest:jar:3.0:test
+   +- org.junit.jupiter:junit-jupiter:jar:5.12.2:test
+   |  +- org.junit.jupiter:junit-jupiter-api:jar:5.12.2:test
+   |  |  +- org.opentest4j:opentest4j:jar:1.3.0:test
+   |  |  +- org.junit.platform:junit-platform-commons:jar:1.12.2:test
+   |  |  \- org.apiguardian:apiguardian-api:jar:1.1.2:test
+   |  +- org.junit.jupiter:junit-jupiter-params:jar:5.12.2:test
+   |  \- org.junit.jupiter:junit-jupiter-engine:jar:5.12.2:test
+   |     \- org.junit.platform:junit-platform-engine:jar:1.12.2:test
+   +- org.mockito:mockito-core:jar:5.17.0:test
+   |  +- net.bytebuddy:byte-buddy-agent:jar:1.17.8:test
+   |  \- org.objenesis:objenesis:jar:3.3:test
+   +- org.mockito:mockito-junit-jupiter:jar:5.17.0:test
+   +- org.skyscreamer:jsonassert:jar:1.5.3:test
+   |  \- com.vaadin.external.google:android-json:jar:0.0.20131108.vaadin1:test
+   +- org.springframework:spring-core:jar:6.2.19:compile
+   |  \- org.springframework:spring-jcl:jar:6.2.19:compile
+   +- org.springframework:spring-test:jar:6.2.19:test
+   \- org.xmlunit:xmlunit-core:jar:2.10.4:test
diff --git a/evidence/E03/occupied-port-refusal.json b/evidence/E03/occupied-port-refusal.json
new file mode 100644
index 0000000..1f3c2a8
--- /dev/null
+++ b/evidence/E03/occupied-port-refusal.json
@@ -0,0 +1,16 @@
+{
+  "label": "fixed",
+  "requests": [],
+  "monitors": [],
+  "checks": [],
+  "processes": [],
+  "portChecks": [
+    {
+      "port": 4321,
+      "checkedAt": "2026-08-28T00:18:00.408Z",
+      "result": "free"
+    }
+  ],
+  "result": "FAIL: Scenario refuses an occupied or unavailable loopback port: 4322",
+  "elapsedSeconds": 0.006
+}
diff --git a/evidence/E03/runs.jsonl b/evidence/E03/runs.jsonl
index f9f0268..abf496c 100644
--- a/evidence/E03/runs.jsonl
+++ b/evidence/E03/runs.jsonl
@@ -1,2 +1,29 @@
 {"command":"/usr/bin/time -p mvn -B -ntp -f backend/pom.xml -DskipTests package","phase":"unchanged baseline package","elapsedSeconds":2.12,"exitCode":0}
 {"command":"fnm exec --using 24.19.0 node scripts/persistence-scenario.mjs baseline","startedAt":"2026-08-27T23:59:59.179Z","elapsedSeconds":3.016,"exitCode":1,"result":"Expected decisive failure: fresh API returned 0 Monitors instead of 2; scenario stopped and owned processes cleaned up"}
+{"command":"/usr/bin/time -p docker buildx imagetools inspect postgres:17.11-bookworm","elapsedSeconds":2.01,"exitCode":0,"result":"Pinned immutable official PostgreSQL17.11-bookworm digest"}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run db:up","elapsedSeconds":28.68,"exitCode":0,"result":"Own container healthy; internal-only network later found to suppress published port"}
+{"command":"/usr/bin/time -p mvn -B -ntp -f backend/pom.xml -DskipTests package","phase":"E03 compile only","elapsedSeconds":7.38,"exitCode":0}
+{"command":"/usr/bin/time -p mvn -B -ntp -f backend/pom.xml dependency:tree -DoutputFile=target/e03-maven-dependencies.txt","elapsedSeconds":2.19,"exitCode":0}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run verify","phase":"first full attempt","elapsedSeconds":8.11,"exitCode":1,"result":"5 Java tests passed; PostgreSQL fixture setup connection refused; no repaired scenario or browser execution"}
+{"command":"node scripts/database.mjs up","startedAt":"2026-08-28T00:12:53.140Z","elapsedSeconds":0.845,"exitCode":0}
+{"command":"mvn -B -ntp -f backend/pom.xml package","startedAt":"2026-08-28T00:12:53.986Z","elapsedSeconds":6.584,"exitCode":1}
+{"command":"node scripts/database.mjs drop e03_restart","startedAt":"2026-08-28T00:13:00.570Z","elapsedSeconds":0.215,"exitCode":0}
+{"command":"node scripts/database.mjs drop e03_browser","startedAt":"2026-08-28T00:13:00.785Z","elapsedSeconds":0.27,"exitCode":0}
+{"command":"/usr/bin/time -p docker inspect --format '{{json .NetworkSettings.Ports}} {{json .HostConfig.PortBindings}}' wse-industry-postgres-1","elapsedSeconds":0.02,"exitCode":0,"result":"Actual port publication empty on internal-only Docker Desktop network"}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run db:down","phase":"apply own network correction, preserve volume","elapsedSeconds":1.05,"exitCode":0}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run db:up","phase":"corrected dedicated bridge","elapsedSeconds":2.58,"exitCode":0}
+{"command":"/usr/bin/time -p docker inspect --format '{{json .NetworkSettings.Ports}}' wse-industry-postgres-1","elapsedSeconds":0.01,"exitCode":0,"result":"Only127.0.0.1:15432 published"}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run verify","phase":"corrected full verification","elapsedSeconds":46.26,"exitCode":0,"result":"Java24; typecheck; production build; isolation guard; fixed A,A,B process restart and lifecycle; Chromium10; schema cleanup"}
+{"command":"node scripts/database.mjs up","startedAt":"2026-08-28T00:17:38.535Z","elapsedSeconds":0.826,"exitCode":0}
+{"command":"mvn -B -ntp -f backend/pom.xml package","startedAt":"2026-08-28T00:17:39.362Z","elapsedSeconds":10.435,"exitCode":0}
+{"command":"npm run typecheck","startedAt":"2026-08-28T00:17:49.797Z","elapsedSeconds":1.52,"exitCode":0}
+{"command":"npm run build","startedAt":"2026-08-28T00:17:51.317Z","elapsedSeconds":8.977,"exitCode":0}
+{"command":"node scripts/persistence-isolation.mjs","startedAt":"2026-08-28T00:18:00.318Z","elapsedSeconds":0.099,"exitCode":0}
+{"command":"node scripts/persistence-scenario.mjs fixed","phase":"guard fixture, no product requests","startedAt":"2026-08-28T00:18:00.404Z","elapsedSeconds":0.006,"exitCode":1,"result":"Expected occupied-port refusal; zero requests or child processes"}
+{"command":"node scripts/database.mjs reset e03_restart","startedAt":"2026-08-28T00:18:00.417Z","elapsedSeconds":0.266,"exitCode":0}
+{"command":"node scripts/persistence-scenario.mjs fixed","startedAt":"2026-08-28T00:18:00.683Z","elapsedSeconds":7.959,"probeElapsedSeconds":7.921,"exitCode":0}
+{"command":"npm run test:e2e","startedAt":"2026-08-28T00:18:08.642Z","elapsedSeconds":15.124,"exitCode":0}
+{"command":"node scripts/database.mjs drop e03_restart","startedAt":"2026-08-28T00:18:23.767Z","elapsedSeconds":0.431,"exitCode":0}
+{"command":"node scripts/database.mjs drop e03_browser","startedAt":"2026-08-28T00:18:24.198Z","elapsedSeconds":0.324,"exitCode":0}
+{"command":"/usr/bin/time -p docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql --username wse_industry --dbname monitor --tuples-only --no-align --command \"SELECT current_setting('server_version'); SELECT count(*) FROM information_schema.schemata WHERE schema_name IN ('e03_restart','e03_functional','e03_browser','e03_migrations','e03_mapping','e03_missing_column','e03_extra_required');\"","elapsedSeconds":0.52,"exitCode":0,"result":"PostgreSQL17.11; zero remaining owned E03 schemas"}
+{"command":"/usr/bin/time -p fnm exec --using 24.19.0 npm run db:down","phase":"final cleanup, volume preserved","elapsedSeconds":1.47,"exitCode":0}
diff --git a/evidence/E03/verification.md b/evidence/E03/verification.md
index f06123c..0c7e06e 100644
--- a/evidence/E03/verification.md
+++ b/evidence/E03/verification.md
@@ -22,3 +22,100 @@ values, including all generated IDs/timestamps, are in `baseline-persistence.jso
 The baseline uses the existing pinned Java 21.0.7 and Maven 3.9.11, with Node
 24.19.0 selected via fnm. No permission, build, or fixture failure occurred in
 these two invocations. No load, retries, sweep, or profiling was performed.
+
+## Setup and first verification attempt
+
+| Invocation | Actual result | Seconds |
+| --- | --- | ---: |
+| `docker buildx imagetools inspect postgres:17.11-bookworm` | exit 0, official immutable digest resolved | 2.01 |
+| `fnm exec --using 24.19.0 npm run db:up` | exit 0, own database container healthy | 28.68 |
+| `mvn -B -ntp -f backend/pom.xml -DskipTests package` | exit 0, E03 sources/tests compiled; no tests executed | 7.38 |
+| `mvn -B -ntp -f backend/pom.xml dependency:tree -DoutputFile=target/e03-maven-dependencies.txt` | exit 0, exact BOM-resolved dependency tree saved | 2.19 |
+| `fnm exec --using 24.19.0 npm run verify` | exit 1 at PostgreSQL fixture connection setup; later gates not executed | 8.11 |
+| `docker inspect --format '{{json .NetworkSettings.Ports}} {{json .HostConfig.PortBindings}}' wse-industry-postgres-1` | exit 0, actual Ports `{}` despite configured loopback binding | 0.02 |
+
+The first full runner passed the 4 CheckRunner tests and 1 MVC boundary test. Its
+PostgreSQL fixtures failed with connection refused at 127.0.0.1:15432, before
+functional requests, migrations, repaired restart, typecheck/build or browser
+verification. Surefire counted 25 executions including failed class cleanup:
+5 passed and 20 setup/cleanup errors. No fixture value or assertion was changed.
+The runner's schema cleanup commands both succeeded through container-local psql.
+
+The own container was healthy internally but Docker Desktop published no port
+for its internal-only network. Changing only this project's network to a dedicated
+bridge restores the explicitly loopback-bound port; the database/version/port and
+all frozen test inputs stay fixed. No other project's resource is inspected or
+changed.
+
+Before the first repaired restart scenario, the root audit identified a harness
+ownership gap. The script now refuses pre-existing listeners before spawning any
+child, requires each child's own startup log and live PID before HTTP readiness,
+records distinct first/fresh API PIDs, awaits the first exit, and checks the port
+again before restart. A,A,B and every persistence/lifecycle assertion are
+unchanged. The original decisive baseline is retained and never repeated.
+
+## Corrected full verification
+
+The dedicated bridge was applied with `npm run db:down` (exit 0, 1.05 seconds,
+volume preserved), then `npm run db:up` (exit 0, 2.58 seconds). A second own-container
+port inspection exited 0 in 0.01 seconds and reported exactly
+`127.0.0.1:15432 -> 5432/tcp`.
+
+`/usr/bin/time -p fnm exec --using 24.19.0 npm run verify` then exited 0 in
+46.26 seconds. These are the runner's recorded gates, not performance thresholds:
+
+| Gate | Actual result | Runner seconds |
+| --- | --- | ---: |
+| `node scripts/database.mjs up` | own PostgreSQL healthy | 0.826 |
+| `mvn -B -ntp -f backend/pom.xml package` | 24 Java tests passed; package built | 10.435 |
+| `npm run typecheck` | passed | 1.520 |
+| `npm run build` | Next production compilation passed | 8.977 |
+| `node scripts/persistence-isolation.mjs` | occupied API port refused; zero requests/children | 0.099 |
+| `node scripts/database.mjs reset e03_restart` | isolated schema created | 0.266 |
+| `node scripts/persistence-scenario.mjs fixed` | original A,A,B restart/lifecycle passed | 7.959 |
+| `npm run test:e2e` | all 10 Chromium tests passed, one worker, retries 0 | 15.124 |
+| `node scripts/database.mjs drop e03_restart` | cleaned owned schema | 0.431 |
+| `node scripts/database.mjs drop e03_browser` | cleaned owned schema | 0.324 |
+
+Java results: 4 CheckRunner, 1 MVC boundary, 15 real-HTTP functional, and 4 real
+PostgreSQL tests. All original 18 Java and 9 browser cases retain their input and
+assertions, including raw numeric interval 1e309. The added functional tests prove
+NUL rejection without create/update mutation and an actual fixture GET outside a
+store transaction. The extra browser case covers two historical checks, edit,
+pause, reload, activation, reload, deletion and reload.
+
+The PostgreSQL tests passed fresh V1+V2 migrations, repeat with zero migrations,
+independent V1-to-V2 upgrade, closed-context persistence, UTC microsecond mapping,
+integer/boolean/zero/null values, actual transaction proxy participation, flushed
+INSERT rollback, managed UPDATE and parent DELETE with zero remaining children.
+`generated-sql.txt` is the captured ORM SQL with actual transaction/read-only
+flags; it contains no bound user values. Both missing `monitors.name` and added
+`monitors.unmapped_required text NOT NULL` without a default rejected servlet
+application startup before any WebServerInitializedEvent.
+
+The fixed process scenario recorded API PID 80847 starting at
+2026-08-28T00:18:00.860Z, ready at 00:18:04.673Z, and an awaited exit at
+00:18:04.839Z. Port 4322 was checked free again. Fresh API PID 80865 started at
+00:18:04.841Z and was ready at 00:18:08.345Z. Both complete Monitor values and all
+three complete CheckRun values compared equal after restart, including IDs,
+timestamps, status, latency zero and explicit nulls. Lifecycle and both NUL
+create/replacement requests also passed. All owned child exits were awaited.
+Full wire/process evidence is `fixed-persistence.json`.
+
+`occupied-port-refusal.json` records the separate guard fixture's expected exit 1
+in 0.006 seconds, before any product request or child spawn. Its wrapper passed;
+this was not a second execution of the persistence dataset.
+
+Final read-only psql verification reported PostgreSQL
+`17.11 (Debian 17.11-1.pgdg12+2)` and zero remaining owned E03 schemas (exit 0,
+0.52 seconds). The isolated database was then stopped with its volume preserved
+(`npm run db:down`, exit 0, 1.47 seconds).
+The exact resolved dependency tree is in `maven-dependencies.txt`: unchanged
+Boot 3.5.16, Hibernate 6.6.53.Final, Spring Data JPA 3.5.13, HikariCP 6.3.3,
+Flyway 11.7.2 and PostgreSQL JDBC 42.7.11. Frontend/runtime pins are unchanged.
+
+Budget: one decisive unchanged baseline; one setup-failed full runner stopped
+before PostgreSQL/API scenarios; one successful corrected full runner; one
+occupied-port guard; compile/dependency/setup/cleanup commands recorded separately.
+No automatic retries, load, benchmark, sweep, or profiling. No hosted CI run is
+claimed; CI invokes the same runner and destroys only its own disposable project.
