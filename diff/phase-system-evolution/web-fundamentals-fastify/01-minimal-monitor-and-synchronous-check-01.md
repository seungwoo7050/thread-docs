# E01 최소 Monitor와 동기 Check

## `메모리 Monitor에서 동기 fixture Check를 실행`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..37b9fa8
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,8 @@
+node_modules/
+.next/
+*.tsbuildinfo
+output/
+playwright-report/
+test-results/
+.env
+.DS_Store
diff --git a/.node-version b/.node-version
new file mode 100644
index 0000000..60ade1a
--- /dev/null
+++ b/.node-version
@@ -0,0 +1 @@
+24.19.0
diff --git a/.npmrc b/.npmrc
new file mode 100644
index 0000000..fe87c9e
--- /dev/null
+++ b/.npmrc
@@ -0,0 +1,4 @@
+engine-strict=true
+save-exact=true
+fund=false
+audit=false
diff --git a/SPEC_REVISION b/SPEC_REVISION
new file mode 100644
index 0000000..a7af57d
--- /dev/null
+++ b/SPEC_REVISION
@@ -0,0 +1 @@
+0a006589477f8ae47bad3faa5510c999cff85ee4
diff --git a/TRACK.md b/TRACK.md
new file mode 100644
index 0000000..2085406
--- /dev/null
+++ b/TRACK.md
@@ -0,0 +1,56 @@
+# Fundamentals / Fastify
+
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+
+E01 establishes Monitor creation and synchronous manual GET checks in process memory.
+Only the configured fixture origin is eligible for outbound requests. The default
+is `http://127.0.0.1:4311`; API and fixture bind to `127.0.0.1`.
+Checks have a fixed 2 second total deadline, perform no retries, do not follow
+redirects, and close the response after observing headers without retaining a body.
+HTTP 200–299 is `SUCCEEDED`; other observed statuses are `FAILED/HTTP_STATUS`.
+A transport failure records `FAILED/CONNECTION_FAILED` or `FAILED/TIMEOUT` with
+`httpStatus: null`. Only the latest terminal result per Monitor is retained.
+`enabled` and `interval` are stored fields; there is no scheduled execution.
+
+## Fixed versions
+
+| Tool | Version |
+| --- | --- |
+| Node.js | 24.19.0 |
+| npm | 11.17.0 |
+| Fastify | 5.12.1 |
+| TypeScript | 5.9.3 |
+| @types/node | 24.13.3 |
+
+Direct dependencies are exact in `package.json`; transitives are pinned in
+`package-lock.json`. Node supplies TypeScript stripping and the unit/functional
+test runner, so no extra runtime loader or test framework is needed.
+Versions were confirmed against the official npm registry. Fastify's
+[server factory reference](https://fastify.dev/docs/latest/Reference/Server/)
+documents its request and listener configuration.
+
+## Run
+
+With the exact Node/npm versions active:
+
+```sh
+npm ci
+npm run fixture
+# A separate terminal:
+npm run dev:api
+```
+
+`API_PORT` changes the API port (default 4312). `FIXTURE_PORT` changes the fixture
+listener (default 4311); set `FIXTURE_ORIGIN` on the API to the same trusted origin.
+Do not expose this development service to a public interface.
+
+```sh
+npm run typecheck
+npm run test:unit
+npm run test:functional
+```
+
+Functional tests own fixed loopback ports 4311 and 4314; stop manual fixture
+processes before running them. Port 4314 is a controlled destination that must
+never receive a check. State disappears on API restart. There is no login,
+database, worker, Redis, Kafka, or production container in E01.
diff --git a/evidence/E01/scenario.json b/evidence/E01/scenario.json
new file mode 100644
index 0000000..9a7745c
--- /dev/null
+++ b/evidence/E01/scenario.json
@@ -0,0 +1,16 @@
+{
+  "thread": "E01",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "ROOT",
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "guardOrigin": "http://127.0.0.1:4314",
+  "monitor": { "name": "Fixture monitor", "path": "/ok", "interval": 60, "enabled": true },
+  "fixedCases": ["GET /ok => SUCCEEDED/200", "GET /fail => FAILED/503", "GET /redirect => FAILED/302 and no second GET", "non-fixture origin rejected; guard request count 0"],
+  "checkDeadlineMs": 2000,
+  "retries": 0,
+  "loadRuns": 0,
+  "parameterSweeps": 0
+}
diff --git a/evidence/E01/verification.json b/evidence/E01/verification.json
new file mode 100644
index 0000000..e9bac48
--- /dev/null
+++ b/evidence/E01/verification.json
@@ -0,0 +1,14 @@
+{
+  "attempt": 1,
+  "runtime": { "node": "24.19.0", "npm": "11.17.0" },
+  "runs": [
+    { "command": "npm run typecheck", "invocation": 1, "exitCode": 0, "elapsedSeconds": 0.97 },
+    { "command": "npm run test:unit", "invocation": 1, "exitCode": 0, "passed": 3, "elapsedSeconds": 0.26 },
+    { "command": "npm run test:functional", "invocation": 1, "exitCode": 0, "passed": 5, "elapsedSeconds": 0.38 }
+  ],
+  "measurement": "/usr/bin/time -p; each command run with fnm exec --using 24.19.0",
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "automaticRetries": 0,
+  "parameterChangesAfterObservation": 0
+}
diff --git a/package-lock.json b/package-lock.json
new file mode 100644
index 0000000..11f0c2b
--- /dev/null
+++ b/package-lock.json
@@ -0,0 +1,686 @@
+{
+  "name": "monitor-fundamentals-fastify",
+  "version": "0.1.0",
+  "lockfileVersion": 3,
+  "requires": true,
+  "packages": {
+    "": {
+      "name": "monitor-fundamentals-fastify",
+      "version": "0.1.0",
+      "dependencies": {
+        "fastify": "5.12.1"
+      },
+      "devDependencies": {
+        "@types/node": "24.13.3",
+        "typescript": "5.9.3"
+      },
+      "engines": {
+        "node": "24.19.0",
+        "npm": "11.17.0"
+      }
+    },
+    "node_modules/@fastify/ajv-compiler": {
+      "version": "4.0.6",
+      "resolved": "https://registry.npmjs.org/@fastify/ajv-compiler/-/ajv-compiler-4.0.6.tgz",
+      "integrity": "sha512-NtuzM0SfaMJbGlnjr9LWQUN5LzgSrbB8tf/wRZNas+4E1O/Nmzl53e7ruT61HDZyRCJGC6FxIogmNZO1c5ETBA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "ajv": "^8.12.0",
+        "ajv-formats": "^3.0.1",
+        "fast-uri": "^4.0.0"
+      }
+    },
+    "node_modules/@fastify/error": {
+      "version": "4.2.0",
+      "resolved": "https://registry.npmjs.org/@fastify/error/-/error-4.2.0.tgz",
+      "integrity": "sha512-RSo3sVDXfHskiBZKBPRgnQTtIqpi/7zhJOEmAxCiBcM7d0uwdGdxLlsCaLzGs8v8NnxIRlfG0N51p5yFaOentQ==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT"
+    },
+    "node_modules/@fastify/fast-json-stringify-compiler": {
+      "version": "5.1.0",
+      "resolved": "https://registry.npmjs.org/@fastify/fast-json-stringify-compiler/-/fast-json-stringify-compiler-5.1.0.tgz",
+      "integrity": "sha512-PxcYtKLbQ8Z+yApiqjK8FwxIwvEj38k2OiLc17u8dkJSlmfi2wHHPaSnaoqBPQqtvF8YVsDgDpP2snDCfFrpfw==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "fast-json-stringify": "^7.0.0"
+      }
+    },
+    "node_modules/@fastify/forwarded": {
+      "version": "3.0.2",
+      "resolved": "https://registry.npmjs.org/@fastify/forwarded/-/forwarded-3.0.2.tgz",
+      "integrity": "sha512-NE8HgKLgYejV9lDpqkEFaDKMLYelJBVfHekhB0UKvX0ghagXRJqg68feg8er1NPXxG4N9i6vPxzt8E+3wHfcmA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT"
+    },
+    "node_modules/@fastify/merge-json-schemas": {
+      "version": "0.2.1",
+      "resolved": "https://registry.npmjs.org/@fastify/merge-json-schemas/-/merge-json-schemas-0.2.1.tgz",
+      "integrity": "sha512-OA3KGBCy6KtIvLf8DINC5880o5iBlDX4SxzLQS8HorJAbqluzLRn80UXU0bxZn7UOFhFgpRJDasfwn9nG4FG4A==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "dequal": "^2.0.3"
+      }
+    },
+    "node_modules/@fastify/proxy-addr": {
+      "version": "5.1.0",
+      "resolved": "https://registry.npmjs.org/@fastify/proxy-addr/-/proxy-addr-5.1.0.tgz",
+      "integrity": "sha512-INS+6gh91cLUjB+PVHfu1UqcB76Sqtpyp7bnL+FYojhjygvOPA9ctiD/JDKsyD9Xgu4hUhCSJBPig/w7duNajw==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "@fastify/forwarded": "^3.0.0",
+        "ipaddr.js": "^2.1.0"
+      }
+    },
+    "node_modules/@pinojs/redact": {
+      "version": "0.4.0",
+      "resolved": "https://registry.npmjs.org/@pinojs/redact/-/redact-0.4.0.tgz",
+      "integrity": "sha512-k2ENnmBugE/rzQfEcdWHcCY+/FM3VLzH9cYEsbdsoqrvzAKRhUZeRNhAZvB8OitQJ1TBed3yqWtdjzS6wJKBwg==",
+      "license": "MIT"
+    },
+    "node_modules/@types/node": {
+      "version": "24.13.3",
+      "resolved": "https://registry.npmjs.org/@types/node/-/node-24.13.3.tgz",
+      "integrity": "sha512-Dh8vAsV36ig5wa9OX4pXvMc9D3Veibfw2wix0CUwYODLD8nkj9UsLjASr49nPg+2eKzxhBV+v7L8pXvT4e639Q==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "undici-types": "~7.18.0"
+      }
+    },
+    "node_modules/abstract-logging": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/abstract-logging/-/abstract-logging-2.0.1.tgz",
+      "integrity": "sha512-2BjRTZxTPvheOvGbBslFSYOUkr+SjPtOnrLP33f+VIWLzezQpZcqVg7ja3L4dBXmzzgwT+a029jRx5PCi3JuiA==",
+      "license": "MIT"
+    },
+    "node_modules/ajv": {
+      "version": "8.20.0",
+      "resolved": "https://registry.npmjs.org/ajv/-/ajv-8.20.0.tgz",
+      "integrity": "sha512-Thbli+OlOj+iMPYFBVBfJ3OmCAnaSyNn4M1vz9T6Gka5Jt9ba/HIR56joy65tY6kx/FCF5VXNB819Y7/GUrBGA==",
+      "license": "MIT",
+      "dependencies": {
+        "fast-deep-equal": "^3.1.3",
+        "fast-uri": "^3.0.1",
+        "json-schema-traverse": "^1.0.0",
+        "require-from-string": "^2.0.2"
+      },
+      "funding": {
+        "type": "github",
+        "url": "https://github.com/sponsors/epoberezkin"
+      }
+    },
+    "node_modules/ajv-formats": {
+      "version": "3.0.1",
+      "resolved": "https://registry.npmjs.org/ajv-formats/-/ajv-formats-3.0.1.tgz",
+      "integrity": "sha512-8iUql50EUR+uUcdRQ3HDqa6EVyo3docL8g5WJ3FNcWmu62IbkGUue/pEyLBW8VGKKucTPgqeks4fIU1DA4yowQ==",
+      "license": "MIT",
+      "dependencies": {
+        "ajv": "^8.0.0"
+      },
+      "peerDependencies": {
+        "ajv": "^8.0.0"
+      },
+      "peerDependenciesMeta": {
+        "ajv": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/ajv/node_modules/fast-uri": {
+      "version": "3.1.6",
+      "resolved": "https://registry.npmjs.org/fast-uri/-/fast-uri-3.1.6.tgz",
+      "integrity": "sha512-7Ical1vFEMr0onbVzEDIreM22I4khW+fzyQPwvAFWBp1iwdshSZRsL4jjRvPG9JP1uiqMHRto+YU6R2/CzDz5Q==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "BSD-3-Clause"
+    },
+    "node_modules/atomic-sleep": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/atomic-sleep/-/atomic-sleep-1.0.0.tgz",
+      "integrity": "sha512-kNOjDqAh7px0XWNI+4QbzoiR/nTkHAWNud2uvnJquD1/x5a7EQZMJT0AczqK0Qn67oY/TTQ1LbUKajZpp3I9tQ==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=8.0.0"
+      }
+    },
+    "node_modules/avvio": {
+      "version": "9.3.0",
+      "resolved": "https://registry.npmjs.org/avvio/-/avvio-9.3.0.tgz",
+      "integrity": "sha512-g2tQ7LE7oOSqDfwEm3M+ZCMTJc7KiZCdJ4UwyZJb5ckTKyYu50OYmvv0mCFXPuYXoM4zkSt8zM9XQ9KCvxA74A==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "@fastify/error": "^4.0.0",
+        "fastq": "^1.17.1"
+      }
+    },
+    "node_modules/cookie": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/cookie/-/cookie-1.1.1.tgz",
+      "integrity": "sha512-ei8Aos7ja0weRpFzJnEA9UHJ/7XQmqglbRwnf2ATjcB9Wq874VKH9kfjjirM6UhU2/E5fFYadylyhFldcqSidQ==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=18"
+      },
+      "funding": {
+        "type": "opencollective",
+        "url": "https://opencollective.com/express"
+      }
+    },
+    "node_modules/dequal": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/dequal/-/dequal-2.0.3.tgz",
+      "integrity": "sha512-0je+qPKHEMohvfRTCEo3CrPG6cAzAYgmzKyxRiYSSDkS6eGJdyVJm7WaYA5ECaAD9wLB2T4EEeymA5aFVcYXCA==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=6"
+      }
+    },
+    "node_modules/fast-decode-uri-component": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/fast-decode-uri-component/-/fast-decode-uri-component-1.0.1.tgz",
+      "integrity": "sha512-WKgKWg5eUxvRZGwW8FvfbaH7AXSh2cL+3j5fMGzUMCxWBJ3dV3a7Wz8y2f/uQ0e3B6WmodD3oS54jTQ9HVTIIg==",
+      "license": "MIT"
+    },
+    "node_modules/fast-deep-equal": {
+      "version": "3.1.3",
+      "resolved": "https://registry.npmjs.org/fast-deep-equal/-/fast-deep-equal-3.1.3.tgz",
+      "integrity": "sha512-f3qQ9oQy9j2AhBe/H9VC91wLmKBCCU/gDOnKNAYG5hswO7BLKj09Hc5HYNz9cGI++xlpDCIgDaitVs03ATR84Q==",
+      "license": "MIT"
+    },
+    "node_modules/fast-json-stringify": {
+      "version": "7.0.1",
+      "resolved": "https://registry.npmjs.org/fast-json-stringify/-/fast-json-stringify-7.0.1.tgz",
+      "integrity": "sha512-eRSayARSbbwlBjpP4vnTTIRD5QPcIrmihPxDeN1DtKnHPg66UuJLx+8hlK1kaFdjvzyQ/dzALoi4vwAQ+T+iZA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "@fastify/merge-json-schemas": "^0.2.0",
+        "ajv": "^8.12.0",
+        "ajv-formats": "^3.0.1",
+        "fast-uri": "^4.0.0",
+        "json-schema-ref-resolver": "^3.0.0",
+        "rfdc": "^1.2.0"
+      }
+    },
+    "node_modules/fast-querystring": {
+      "version": "1.1.2",
+      "resolved": "https://registry.npmjs.org/fast-querystring/-/fast-querystring-1.1.2.tgz",
+      "integrity": "sha512-g6KuKWmFXc0fID8WWH0jit4g0AGBoJhCkJMb1RmbsSEUNvQ+ZC8D6CUZ+GtF8nMzSPXnhiePyyqqipzNNEnHjg==",
+      "license": "MIT",
+      "dependencies": {
+        "fast-decode-uri-component": "^1.0.1"
+      }
+    },
+    "node_modules/fast-uri": {
+      "version": "4.1.3",
+      "resolved": "https://registry.npmjs.org/fast-uri/-/fast-uri-4.1.3.tgz",
+      "integrity": "sha512-7+72G6vLt7jjNas8SmSATx2qeyRIjxeqO3i4IkmDTxlqYZRKANhOe1bnovcp4WZmvsYrp60WyqPyHqgRiX0yXw==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "BSD-3-Clause"
+    },
+    "node_modules/fastify": {
+      "version": "5.12.1",
+      "resolved": "https://registry.npmjs.org/fastify/-/fastify-5.12.1.tgz",
+      "integrity": "sha512-FWi+tQvwxR/PeRX7Z2mhfEF5ozJ3jn9asiiclzKXNSzJRHAYcU924aIOKAdHFJ+YIKieh3cqr1IwCOvTr41B3Q==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "@fastify/ajv-compiler": "^4.0.5",
+        "@fastify/error": "^4.0.0",
+        "@fastify/fast-json-stringify-compiler": "^5.0.0",
+        "@fastify/proxy-addr": "^5.0.0",
+        "abstract-logging": "^2.0.1",
+        "avvio": "^9.0.0",
+        "fast-json-stringify": "^7.0.0",
+        "find-my-way": "^9.6.0",
+        "light-my-request": "^6.0.0",
+        "pino": "^9.14.0 || ^10.1.0",
+        "process-warning": "^5.1.0",
+        "rfdc": "^1.3.1",
+        "secure-json-parse": "^4.0.0",
+        "semver": "^7.6.0",
+        "toad-cache": "^3.7.0"
+      }
+    },
+    "node_modules/fastq": {
+      "version": "1.20.1",
+      "resolved": "https://registry.npmjs.org/fastq/-/fastq-1.20.1.tgz",
+      "integrity": "sha512-GGToxJ/w1x32s/D2EKND7kTil4n8OVk/9mycTc4VDza13lOvpUZTGX3mFSCtV9ksdGBVzvsyAVLM6mHFThxXxw==",
+      "license": "ISC",
+      "dependencies": {
+        "reusify": "^1.0.4"
+      }
+    },
+    "node_modules/find-my-way": {
+      "version": "9.9.0",
+      "resolved": "https://registry.npmjs.org/find-my-way/-/find-my-way-9.9.0.tgz",
+      "integrity": "sha512-sJsgZ1sQH2UDuowPuMKg8az7Qc8F0jnj+SKkFWU/+T0xcFlgV5skgXOGUqmQzOdmW6ALA7AhJINWx3qFBkbLHA==",
+      "license": "MIT",
+      "dependencies": {
+        "fast-deep-equal": "^3.1.3",
+        "fast-querystring": "^1.0.0",
+        "safe-regex2": "^5.0.0"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/ipaddr.js": {
+      "version": "2.5.0",
+      "resolved": "https://registry.npmjs.org/ipaddr.js/-/ipaddr.js-2.5.0.tgz",
+      "integrity": "sha512-aq+t5NAc+cS6rZQQVWC2x98CPqGtKKTMDd4Gaodv0wShnItdKg/51djkGJ1hqH+Oy0ivDftCbSLCQob8zso01w==",
+      "license": "MIT",
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/json-schema-ref-resolver": {
+      "version": "3.0.0",
+      "resolved": "https://registry.npmjs.org/json-schema-ref-resolver/-/json-schema-ref-resolver-3.0.0.tgz",
+      "integrity": "sha512-hOrZIVL5jyYFjzk7+y7n5JDzGlU8rfWDuYyHwGa2WA8/pcmMHezp2xsVwxrebD/Q9t8Nc5DboieySDpCp4WG4A==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "dequal": "^2.0.3"
+      }
+    },
+    "node_modules/json-schema-traverse": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/json-schema-traverse/-/json-schema-traverse-1.0.0.tgz",
+      "integrity": "sha512-NM8/P9n3XjXhIZn1lLhkFaACTOURQXjWhV4BA/RnOv8xvgqtqpAX9IO4mRQxSx1Rlo4tqzeqb0sOlruaOy3dug==",
+      "license": "MIT"
+    },
+    "node_modules/light-my-request": {
+      "version": "6.6.0",
+      "resolved": "https://registry.npmjs.org/light-my-request/-/light-my-request-6.6.0.tgz",
+      "integrity": "sha512-CHYbu8RtboSIoVsHZ6Ye4cj4Aw/yg2oAFimlF7mNvfDV192LR7nDiKtSIfCuLT7KokPSTn/9kfVLm5OGN0A28A==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "BSD-3-Clause",
+      "dependencies": {
+        "cookie": "^1.0.1",
+        "process-warning": "^4.0.0",
+        "set-cookie-parser": "^2.6.0"
+      }
+    },
+    "node_modules/light-my-request/node_modules/process-warning": {
+      "version": "4.0.1",
+      "resolved": "https://registry.npmjs.org/process-warning/-/process-warning-4.0.1.tgz",
+      "integrity": "sha512-3c2LzQ3rY9d0hc1emcsHhfT9Jwz0cChib/QN89oME2R451w5fy3f0afAhERFZAwrbDU43wk12d0ORBpDVME50Q==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT"
+    },
+    "node_modules/on-exit-leak-free": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/on-exit-leak-free/-/on-exit-leak-free-2.1.2.tgz",
+      "integrity": "sha512-0eJJY6hXLGf1udHwfNftBqH+g73EU4B504nZeKpz1sYRKafAghwxEJunB2O7rDZkL4PGfsMVnTXZ2EjibbqcsA==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=14.0.0"
+      }
+    },
+    "node_modules/pino": {
+      "version": "10.3.1",
+      "resolved": "https://registry.npmjs.org/pino/-/pino-10.3.1.tgz",
+      "integrity": "sha512-r34yH/GlQpKZbU1BvFFqOjhISRo1MNx1tWYsYvmj6KIRHSPMT2+yHOEb1SG6NMvRoHRF0a07kCOox/9yakl1vg==",
+      "license": "MIT",
+      "dependencies": {
+        "@pinojs/redact": "^0.4.0",
+        "atomic-sleep": "^1.0.0",
+        "on-exit-leak-free": "^2.1.0",
+        "pino-abstract-transport": "^3.0.0",
+        "pino-std-serializers": "^7.0.0",
+        "process-warning": "^5.0.0",
+        "quick-format-unescaped": "^4.0.3",
+        "real-require": "^0.2.0",
+        "safe-stable-stringify": "^2.3.1",
+        "sonic-boom": "^4.0.1",
+        "thread-stream": "^4.0.0"
+      },
+      "bin": {
+        "pino": "bin.js"
+      }
+    },
+    "node_modules/pino-abstract-transport": {
+      "version": "3.0.0",
+      "resolved": "https://registry.npmjs.org/pino-abstract-transport/-/pino-abstract-transport-3.0.0.tgz",
+      "integrity": "sha512-wlfUczU+n7Hy/Ha5j9a/gZNy7We5+cXp8YL+X+PG8S0KXxw7n/JXA3c46Y0zQznIJ83URJiwy7Lh56WLokNuxg==",
+      "license": "MIT",
+      "dependencies": {
+        "split2": "^4.0.0"
+      }
+    },
+    "node_modules/pino-std-serializers": {
+      "version": "7.1.0",
+      "resolved": "https://registry.npmjs.org/pino-std-serializers/-/pino-std-serializers-7.1.0.tgz",
+      "integrity": "sha512-BndPH67/JxGExRgiX1dX0w1FvZck5Wa4aal9198SrRhZjH3GxKQUKIBnYJTdj2HDN3UQAS06HlfcSbQj2OHmaw==",
+      "license": "MIT"
+    },
+    "node_modules/process-warning": {
+      "version": "5.1.0",
+      "resolved": "https://registry.npmjs.org/process-warning/-/process-warning-5.1.0.tgz",
+      "integrity": "sha512-jQSaVHsPgtyw60e1rQ/A+/ArPEj/S8pS/vFnyGa/gYFXrKk/6RuDkoqVDQ5NI5MmS01698ltlAk0NoDBNLujRw==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT"
+    },
+    "node_modules/quick-format-unescaped": {
+      "version": "4.0.4",
+      "resolved": "https://registry.npmjs.org/quick-format-unescaped/-/quick-format-unescaped-4.0.4.tgz",
+      "integrity": "sha512-tYC1Q1hgyRuHgloV/YXs2w15unPVh8qfu/qCTfhTYamaw7fyhumKa2yGpdSo87vY32rIclj+4fWYQXUMs9EHvg==",
+      "license": "MIT"
+    },
+    "node_modules/real-require": {
+      "version": "0.2.0",
+      "resolved": "https://registry.npmjs.org/real-require/-/real-require-0.2.0.tgz",
+      "integrity": "sha512-57frrGM/OCTLqLOAh0mhVA9VBMHd+9U7Zb2THMGdBUoZVOtGbJzjxsYGDJ3A9AYYCP4hn6y1TVbaOfzWtm5GFg==",
+      "license": "MIT",
+      "engines": {
+        "node": ">= 12.13.0"
+      }
+    },
+    "node_modules/require-from-string": {
+      "version": "2.0.2",
+      "resolved": "https://registry.npmjs.org/require-from-string/-/require-from-string-2.0.2.tgz",
+      "integrity": "sha512-Xf0nWe6RseziFMu+Ap9biiUbmplq6S9/p+7w7YXP/JBHhrUDDUhwa+vANyubuqfZWTveU//DYVGsDG7RKL/vEw==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
+    "node_modules/ret": {
+      "version": "0.5.0",
+      "resolved": "https://registry.npmjs.org/ret/-/ret-0.5.0.tgz",
+      "integrity": "sha512-I1XxrZSQ+oErkRR4jYbAyEEu2I0avBvvMM5JN+6EBprOGRCs63ENqZ3vjavq8fBw2+62G5LF5XelKwuJpcvcxw==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/reusify": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/reusify/-/reusify-1.1.0.tgz",
+      "integrity": "sha512-g6QUff04oZpHs0eG5p83rFLhHeV00ug/Yf9nZM6fLeUrPguBTkTQOdpAWWspMh55TZfVQDPaN3NQJfbVRAxdIw==",
+      "license": "MIT",
+      "engines": {
+        "iojs": ">=1.0.0",
+        "node": ">=0.10.0"
+      }
+    },
+    "node_modules/rfdc": {
+      "version": "1.4.1",
+      "resolved": "https://registry.npmjs.org/rfdc/-/rfdc-1.4.1.tgz",
+      "integrity": "sha512-q1b3N5QkRUWUl7iyylaaj3kOpIT0N2i9MqIEQXP73GVsN9cw3fdx8X63cEmWhJGi2PPCF23Ijp7ktmd39rawIA==",
+      "license": "MIT"
+    },
+    "node_modules/safe-regex2": {
+      "version": "5.1.1",
+      "resolved": "https://registry.npmjs.org/safe-regex2/-/safe-regex2-5.1.1.tgz",
+      "integrity": "sha512-mOSBvHGDZMuIEZMdOz/aCEYDCv0E7nfcNsIhUF+/P+xC7Hyf3FkvymqgPbg9D1EdSGu+uKbJgy09K/RKKc7kJA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "ret": "~0.5.0"
+      },
+      "bin": {
+        "safe-regex2": "bin/safe-regex2.js"
+      }
+    },
+    "node_modules/safe-stable-stringify": {
+      "version": "2.5.0",
+      "resolved": "https://registry.npmjs.org/safe-stable-stringify/-/safe-stable-stringify-2.5.0.tgz",
+      "integrity": "sha512-b3rppTKm9T+PsVCBEOUR46GWI7fdOs00VKZ1+9c1EWDaDMvjQc6tUwuFyIprgGgTcWoVHSKrU8H31ZHA2e0RHA==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/secure-json-parse": {
+      "version": "4.1.0",
+      "resolved": "https://registry.npmjs.org/secure-json-parse/-/secure-json-parse-4.1.0.tgz",
+      "integrity": "sha512-l4KnYfEyqYJxDwlNVyRfO2E4NTHfMKAWdUuA8J0yve2Dz/E/PdBepY03RvyJpssIpRFwJoCD55wA+mEDs6ByWA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/fastify"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/fastify"
+        }
+      ],
+      "license": "BSD-3-Clause"
+    },
+    "node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "license": "ISC",
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/set-cookie-parser": {
+      "version": "2.7.2",
+      "resolved": "https://registry.npmjs.org/set-cookie-parser/-/set-cookie-parser-2.7.2.tgz",
+      "integrity": "sha512-oeM1lpU/UvhTxw+g3cIfxXHyJRc/uidd3yK1P242gzHds0udQBYzs3y8j4gCCW+ZJ7ad0yctld8RYO+bdurlvw==",
+      "license": "MIT"
+    },
+    "node_modules/sonic-boom": {
+      "version": "4.2.1",
+      "resolved": "https://registry.npmjs.org/sonic-boom/-/sonic-boom-4.2.1.tgz",
+      "integrity": "sha512-w6AxtubXa2wTXAUsZMMWERrsIRAdrK0Sc+FUytWvYAhBJLyuI4llrMIC1DtlNSdI99EI86KZum2MMq3EAZlF9Q==",
+      "license": "MIT",
+      "dependencies": {
+        "atomic-sleep": "^1.0.0"
+      }
+    },
+    "node_modules/split2": {
+      "version": "4.2.0",
+      "resolved": "https://registry.npmjs.org/split2/-/split2-4.2.0.tgz",
+      "integrity": "sha512-UcjcJOWknrNkF6PLX83qcHM6KHgVKNkV62Y8a5uYDVv9ydGQVwAHMKqHdJje1VTWpljG0WYpCDhrCdAOYH4TWg==",
+      "license": "ISC",
+      "engines": {
+        "node": ">= 10.x"
+      }
+    },
+    "node_modules/thread-stream": {
+      "version": "4.2.0",
+      "resolved": "https://registry.npmjs.org/thread-stream/-/thread-stream-4.2.0.tgz",
+      "integrity": "sha512-e2zZ96wSChazBsbENf/Pcm/4swHt2cEKQ92rhUjkL9GCKiTDJIaTBenjE/m9DXi0QBmTMDkFDdOomUy20A1tDQ==",
+      "license": "MIT",
+      "dependencies": {
+        "real-require": "^1.0.0"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/thread-stream/node_modules/real-require": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/real-require/-/real-require-1.0.0.tgz",
+      "integrity": "sha512-P4nbQYQfePJxRSmY+v/KINxVucm4NF3p3s7pJveMTtom52FR4YGltUQLB8idDXwDDWW+eYrWDFbuzUnjoWHF7g==",
+      "license": "MIT"
+    },
+    "node_modules/toad-cache": {
+      "version": "3.7.4",
+      "resolved": "https://registry.npmjs.org/toad-cache/-/toad-cache-3.7.4.tgz",
+      "integrity": "sha512-m1TdR/rvT7kgGJZhspNtXdsdYk0fddFpJJFlG5s+UkPFo6lkLoZ3YLOaovPYjq1R75NP5JfeTlSHaOsE09peCg==",
+      "license": "MIT",
+      "engines": {
+        "node": ">=20"
+      }
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
+      "version": "7.18.2",
+      "resolved": "https://registry.npmjs.org/undici-types/-/undici-types-7.18.2.tgz",
+      "integrity": "sha512-AsuCzffGHJybSaRrmr5eHr81mwJU3kjw6M+uprWvCXiNeN9SOGwQ3Jn8jb8m3Z6izVgknn1R0FTCEAP2QrLY/w==",
+      "dev": true,
+      "license": "MIT"
+    }
+  }
+}
diff --git a/package.json b/package.json
new file mode 100644
index 0000000..0271dbd
--- /dev/null
+++ b/package.json
@@ -0,0 +1,19 @@
+{
+  "name": "monitor-fundamentals-fastify",
+  "version": "0.1.0",
+  "private": true,
+  "type": "module",
+  "engines": { "node": "24.19.0", "npm": "11.17.0" },
+  "packageManager": "npm@11.17.0",
+  "scripts": {
+    "dev:api": "node --watch server/main.ts",
+    "start:api": "node server/main.ts",
+    "fixture": "node test/fixture.ts",
+    "typecheck": "tsc --noEmit",
+    "test:unit": "node --test test/unit.test.ts",
+    "test:functional": "node --test test/functional.test.ts",
+    "test": "npm run test:unit && npm run test:functional"
+  },
+  "dependencies": { "fastify": "5.12.1" },
+  "devDependencies": { "@types/node": "24.13.3", "typescript": "5.9.3" }
+}
diff --git a/server/app.ts b/server/app.ts
new file mode 100644
index 0000000..d38efc4
--- /dev/null
+++ b/server/app.ts
@@ -0,0 +1,47 @@
+import { randomUUID } from 'node:crypto';
+import Fastify from 'fastify';
+import { checkMonitor, DEFAULT_FIXTURE_ORIGIN, fixtureUrl } from './check.ts';
+import type { CheckRun, Monitor } from './model.ts';
+
+export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
+  const app = Fastify({ logger: false, bodyLimit: 8_192 });
+  const monitors = new Map<string, Monitor>();
+  const latestChecks = new Map<string, CheckRun>();
+
+  app.get('/health', async () => ({ status: 'ok' }));
+  app.get('/monitors', async () => Array.from(monitors.values(), (monitor) => ({
+    ...monitor, latestCheck: latestChecks.get(monitor.id) ?? null,
+  })));
+
+  app.post<{ Body: { name: string; url: string; interval: number; enabled: boolean } }>(
+    '/monitors', async (request, reply) => {
+      const body = request.body;
+      if (!body || typeof body.name !== 'string' || typeof body.url !== 'string') {
+        return reply.code(400).send({ message: 'Name and URL are required.' });
+      }
+      let url: URL;
+      try {
+        url = fixtureUrl(body.url, fixtureOrigin);
+      } catch {
+        return reply.code(400).send({ message: 'Only the configured fixture origin is allowed.' });
+      }
+      const now = new Date().toISOString();
+      const monitor: Monitor = {
+        id: randomUUID(), name: body.name, url: url.href,
+        interval: body.interval, enabled: body.enabled, createdAt: now, updatedAt: now,
+      };
+      monitors.set(monitor.id, monitor);
+      return reply.code(201).send({ ...monitor, latestCheck: null });
+    },
+  );
+
+  app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request, reply) => {
+    const monitor = monitors.get(request.params.id);
+    if (!monitor) return reply.code(404).send({ message: 'Monitor not found.' });
+    const result = await checkMonitor(monitor, fixtureOrigin);
+    latestChecks.set(monitor.id, result);
+    return result;
+  });
+
+  return app;
+}
diff --git a/server/check.ts b/server/check.ts
new file mode 100644
index 0000000..6282fdb
--- /dev/null
+++ b/server/check.ts
@@ -0,0 +1,57 @@
+import { randomUUID } from 'node:crypto';
+import { request as httpRequest } from 'node:http';
+import { request as httpsRequest } from 'node:https';
+import type { CheckRun, Monitor } from './model.ts';
+
+export const DEFAULT_FIXTURE_ORIGIN = 'http://127.0.0.1:4311';
+export const CHECK_TIMEOUT_MS = 2_000;
+
+export function fixtureUrl(value: string, fixtureOrigin: string): URL {
+  const url = new URL(value);
+  const allowed = new URL(fixtureOrigin);
+  if (
+    !['http:', 'https:'].includes(url.protocol) ||
+    url.origin !== allowed.origin ||
+    url.username || url.password
+  ) {
+    throw new Error('Only the configured fixture origin is allowed.');
+  }
+  return url;
+}
+
+export async function checkMonitor(monitor: Monitor, fixtureOrigin: string): Promise<CheckRun> {
+  // Recheck at the outbound boundary, even though creation also checks the URL.
+  const url = fixtureUrl(monitor.url, fixtureOrigin);
+  const startedAt = new Date().toISOString();
+  const start = performance.now();
+
+  return new Promise((resolve) => {
+    let settled = false;
+    const finish = (httpStatus: number | null, failureReason: CheckRun['failureReason']) => {
+      if (settled) return;
+      settled = true;
+      clearTimeout(deadline);
+      resolve({
+        id: randomUUID(), monitorId: monitor.id, trigger: 'MANUAL',
+        state: failureReason === null ? 'SUCCEEDED' : 'FAILED',
+        httpStatus, latencyMs: Math.round(performance.now() - start), failureReason,
+        startedAt, finishedAt: new Date().toISOString(),
+      });
+    };
+    const request = (url.protocol === 'https:' ? httpsRequest : httpRequest)(url, {
+      method: 'GET', agent: false,
+      headers: { 'user-agent': 'monitor-fixture-check/0.1' },
+    }, (response) => {
+      const status = response.statusCode ?? 0;
+      // E01 observes response headers only: no body is retained and no redirect is followed.
+      response.destroy();
+      finish(status, status >= 200 && status < 300 ? null : 'HTTP_STATUS');
+    });
+    const deadline = setTimeout(() => {
+      finish(null, 'TIMEOUT');
+      request.destroy();
+    }, CHECK_TIMEOUT_MS);
+    request.on('error', () => finish(null, 'CONNECTION_FAILED'));
+    request.end();
+  });
+}
diff --git a/server/main.ts b/server/main.ts
new file mode 100644
index 0000000..5d27ec8
--- /dev/null
+++ b/server/main.ts
@@ -0,0 +1,9 @@
+import { buildApp } from './app.ts';
+import { DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+
+const app = buildApp(process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN);
+await app.listen({ host: '127.0.0.1', port: Number(process.env.API_PORT ?? 4312) });
+console.log('Monitor API listening on loopback.');
+for (const signal of ['SIGINT', 'SIGTERM'] as const) {
+  process.once(signal, async () => { await app.close(); process.exit(0); });
+}
diff --git a/server/model.ts b/server/model.ts
new file mode 100644
index 0000000..bfc24ab
--- /dev/null
+++ b/server/model.ts
@@ -0,0 +1,23 @@
+export type Monitor = {
+  id: string;
+  name: string;
+  url: string;
+  interval: number;
+  enabled: boolean;
+  createdAt: string;
+  updatedAt: string;
+};
+
+export type CheckRun = {
+  id: string;
+  monitorId: string;
+  trigger: 'MANUAL';
+  state: 'SUCCEEDED' | 'FAILED';
+  httpStatus: number | null;
+  latencyMs: number;
+  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILED' | null;
+  startedAt: string;
+  finishedAt: string;
+};
+
+export type MonitorView = Monitor & { latestCheck: CheckRun | null };
diff --git a/test/fixture.ts b/test/fixture.ts
new file mode 100644
index 0000000..5a886c8
--- /dev/null
+++ b/test/fixture.ts
@@ -0,0 +1,30 @@
+import { createServer } from 'node:http';
+import { pathToFileURL } from 'node:url';
+
+export function fixtureServer() {
+  const calls = new Map<string, number>();
+  const server = createServer((request, response) => {
+    const path = request.url ?? '/';
+    calls.set(path, (calls.get(path) ?? 0) + 1);
+    response.setHeader('content-type', 'text/plain');
+    if (path === '/ok') return void response.end('ok\n');
+    if (path === '/fail') { response.statusCode = 503; return void response.end('unavailable\n'); }
+    if (path === '/redirect') {
+      response.writeHead(302, { location: '/ok' });
+      return void response.end('redirect\n');
+    }
+    response.statusCode = 404;
+    response.end('not found\n');
+  });
+  return { server, calls };
+}
+
+if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
+  const { server } = fixtureServer();
+  server.listen(Number(process.env.FIXTURE_PORT ?? 4311), '127.0.0.1', () => {
+    console.log('Controlled fixture listening on loopback.');
+  });
+  for (const signal of ['SIGINT', 'SIGTERM'] as const) {
+    process.once(signal, () => { server.closeAllConnections(); server.close(() => process.exit(0)); });
+  }
+}
diff --git a/test/functional.test.ts b/test/functional.test.ts
new file mode 100644
index 0000000..23224cf
--- /dev/null
+++ b/test/functional.test.ts
@@ -0,0 +1,87 @@
+import { after, before, test } from 'node:test';
+import assert from 'node:assert/strict';
+import { createServer } from 'node:http';
+import { buildApp } from '../server/app.ts';
+import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
+import type { Monitor, MonitorView } from '../server/model.ts';
+import { fixtureServer } from './fixture.ts';
+
+const fixture = fixtureServer();
+const app = buildApp();
+let guardCalls = 0;
+const guard = createServer((_request, response) => { guardCalls++; response.end('guard'); });
+
+before(async () => {
+  await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
+  await new Promise<void>((resolve) => guard.listen(4314, '127.0.0.1', resolve));
+});
+after(async () => {
+  await app.close();
+  fixture.server.closeAllConnections();
+  guard.closeAllConnections();
+  await Promise.all([
+    new Promise<void>((resolve) => fixture.server.close(() => resolve())),
+    new Promise<void>((resolve) => guard.close(() => resolve())),
+  ]);
+});
+
+async function create(path: string): Promise<MonitorView> {
+  const response = await app.inject({ method: 'POST', url: '/monitors', payload: {
+    name: 'Fixture monitor', url: `${DEFAULT_FIXTURE_ORIGIN}${path}`, interval: 60, enabled: true,
+  } });
+  assert.equal(response.statusCode, 201);
+  return response.json<MonitorView>();
+}
+
+test('create a Monitor in memory and synchronously observe GET /ok', async () => {
+  const monitor = await create('/ok');
+  assert.equal(monitor.interval, 60);
+  assert.equal(monitor.enabled, true);
+  assert.equal(monitor.latestCheck, null);
+  const response = await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` });
+  assert.equal(response.statusCode, 200);
+  const result = response.json();
+  assert.equal(result.state, 'SUCCEEDED');
+  assert.equal(result.httpStatus, 200);
+  assert.equal(result.trigger, 'MANUAL');
+  assert.equal(result.failureReason, null);
+  assert.ok(result.latencyMs >= 0);
+  assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
+  assert.equal(fixture.calls.get('/ok'), 1);
+  const list = (await app.inject('/monitors')).json<MonitorView[]>();
+  assert.equal(list.find((item) => item.id === monitor.id)?.latestCheck?.id, result.id);
+});
+
+test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
+  const monitor = await create('/fail');
+  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json();
+  assert.equal(result.state, 'FAILED');
+  assert.equal(result.httpStatus, 503);
+  assert.equal(result.failureReason, 'HTTP_STATUS');
+  assert.equal(fixture.calls.get('/fail'), 1);
+});
+
+test('the check does not follow even a same-origin redirect', async () => {
+  const previousOkCalls = fixture.calls.get('/ok');
+  const monitor = await create('/redirect');
+  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json();
+  assert.equal(result.state, 'FAILED');
+  assert.equal(result.httpStatus, 302);
+  assert.equal(fixture.calls.get('/ok'), previousOkCalls);
+  assert.equal(fixture.calls.get('/redirect'), 1);
+});
+
+test('a non-fixture URL is rejected without contacting the controlled guard', async () => {
+  const monitor = await create('/ok');
+  const unsafe: Monitor = { ...monitor, url: 'http://127.0.0.1:4314/ok' };
+  const response = await app.inject({ method: 'POST', url: '/monitors', payload: unsafe });
+  assert.equal(response.statusCode, 400);
+  await assert.rejects(checkMonitor(unsafe, DEFAULT_FIXTURE_ORIGIN));
+  assert.equal(guardCalls, 0);
+});
+
+test('another application instance starts with empty memory', async () => {
+  const fresh = buildApp();
+  try { assert.deepEqual((await fresh.inject('/monitors')).json(), []); }
+  finally { await fresh.close(); }
+});
diff --git a/test/unit.test.ts b/test/unit.test.ts
new file mode 100644
index 0000000..ee7bbde
--- /dev/null
+++ b/test/unit.test.ts
@@ -0,0 +1,19 @@
+import test from 'node:test';
+import assert from 'node:assert/strict';
+import { fixtureUrl, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
+
+test('a path on the configured fixture origin is allowed', () => {
+  assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
+});
+
+test('a different fixture port or host is refused before networking', () => {
+  for (const url of ['http://127.0.0.1:4314/ok', 'http://localhost:4311/ok']) {
+    assert.throws(() => fixtureUrl(url, DEFAULT_FIXTURE_ORIGIN));
+  }
+});
+
+test('credentials and another protocol cannot bypass the fixture boundary', () => {
+  for (const url of ['http://user:pass@127.0.0.1:4311/ok', 'https://127.0.0.1:4311/ok', 'file:///etc/hosts']) {
+    assert.throws(() => fixtureUrl(url, DEFAULT_FIXTURE_ORIGIN));
+  }
+});
diff --git a/tsconfig.json b/tsconfig.json
new file mode 100644
index 0000000..dc359f7
--- /dev/null
+++ b/tsconfig.json
@@ -0,0 +1,12 @@
+{
+  "compilerOptions": {
+    "target": "ES2022",
+    "module": "NodeNext",
+    "moduleResolution": "NodeNext",
+    "strict": true,
+    "noEmit": true,
+    "allowImportingTsExtensions": true,
+    "skipLibCheck": true
+  },
+  "include": ["server/**/*.ts", "test/**/*.ts"]
+}


