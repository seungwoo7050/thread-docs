## `E01 검증 게이트와 실행 증거 고정`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..010b39a
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,32 @@
+name: E01 verification
+on: [push, pull_request]
+permissions:
+  contents: read
+jobs:
+  unit-functional-browser:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 15
+    env:
+      NEXT_TELEMETRY_DISABLED: '1'
+    steps:
+      - uses: actions/checkout@v4.2.2
+      - uses: actions/setup-java@v4.7.1
+        with:
+          distribution: temurin
+          java-version: '21.0.7+6'
+      - uses: actions/setup-node@v4.4.0
+        with:
+          node-version-file: .node-version
+          cache: npm
+      - name: Install pinned Maven
+        shell: bash
+        run: |
+          curl --fail --silent --show-error --output "$RUNNER_TEMP/maven.tar.gz" https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.11/apache-maven-3.9.11-bin.tar.gz
+          echo "bcfe4fe305c962ace56ac7b5fc7a08b87d5abd8b7e89027ab251069faebee516b0ded8961445d6d91ec1985dfe30f8153268843c89aa392733d1a3ec956c9978  $RUNNER_TEMP/maven.tar.gz" | sha512sum --check
+          tar -xzf "$RUNNER_TEMP/maven.tar.gz" -C "$RUNNER_TEMP"
+          echo "$RUNNER_TEMP/apache-maven-3.9.11/bin" >> "$GITHUB_PATH"
+      - run: npm install --global npm@11.17.0
+      - run: npm ci
+      - run: npx playwright install --with-deps chromium
+      - name: Unit, functional, type, build and minimum browser gates
+        run: npm run verify
diff --git a/TRACK.md b/TRACK.md
new file mode 100644
index 0000000..a7be6ee
--- /dev/null
+++ b/TRACK.md
@@ -0,0 +1,81 @@
+# Industry / Spring
+
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
+
+E01 is a local development product: Next.js/React renders the Monitor form and terminal Check result; Spring MVC owns an in-memory Monitor map and latest Check result. A manual check is synchronous. There are no accounts, database, scheduler, workers, cache, broker, or production containers.
+
+## Pinned toolchain
+
+| Component | Version | Pin |
+| --- | --- | --- |
+| Java (Eclipse Temurin) | 21.0.7+6 | `.java-version`, CI; Maven enforces 21.0.7 |
+| Maven | 3.9.11 | `.maven-version`, Maven enforcer, CI archive plus SHA-512 |
+| Spring Boot | 3.5.16 | `backend/pom.xml` parent/BOM, including exact transitive versions |
+| Node.js | 24.19.0 | `.node-version`, package engines, CI |
+| npm | 11.17.0 | packageManager, engines, CI |
+| Next.js | 16.3.3 | `package.json`, `package-lock.json` |
+| React / React DOM | 19.2.8 | `package.json`, `package-lock.json` |
+| TypeScript | 5.9.3 | `package.json`, `package-lock.json` |
+| Playwright | 1.62.1 | `package.json`, lock; Chromium revision 1234 / 151.0.7922.34 |
+| Node / React / React DOM types | 24.10.1 / 19.2.18 / 19.2.5 | `package.json`, lock |
+| Maven Enforcer | 3.6.2 | `backend/pom.xml` |
+
+Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. There is no container build yet; CI and local runtime files are the current execution contract.
+
+## Run locally
+
+Run all commands from the repository root using the pinned runtimes. Maven artifacts are isolated in `.m2/repository` by `.mvn/maven.config`.
+
+```sh
+fnm use 24.19.0
+npm ci
+```
+
+Start each process in a separate terminal:
+
+```sh
+npm run fixture
+npm run api:dev
+npm run dev
+```
+
+Open [Monitors](http://127.0.0.1:4323/monitors). Create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to observe `SUCCEEDED` and `HTTP 200`. `/fail` yields `FAILED` and `HTTP 503`.
+
+All three defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap.
+
+## Check boundary
+
+- Only HTTP URLs matching the configured fixture hostname and explicit port are accepted, both on Monitor creation and at the outbound boundary. Credentials and fragments are rejected. A hostname alias is not treated as the configured host.
+- Checks send one GET with no request body, use no proxy, and never follow redirects. `/redirect` therefore records `FAILED / 302`, with no request to `/ok`.
+- Connect timeout is 1 second and response-header read timeout is 2 seconds. No response body is materialized or retained. This is a controlled fixture implementation, not general Internet monitoring or general SSRF defense.
+- `200..299` is `SUCCEEDED`. Other observed HTTP statuses are `FAILED / HTTP_STATUS`. No HTTP response produces a null status and `TIMEOUT` or `CONNECTION_FAILURE`; no synthetic status is invented.
+- Latest results survive page reload, but all state disappears with the API process. Interval and enabled are stored, without automatic scheduling. Basic HTTP errors are shown in the UI; no uniform runtime error contract is claimed in E01.
+
+## Verification
+
+```sh
+npx playwright install chromium
+npm run verify
+```
+
+`verify` runs Maven unit and real-HTTP functional tests and packages the API, then TypeScript checking, a Next production compilation, and two real Chromium browser tests against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`. E01's committed evidence is in `evidence/E01`.
+
+The CI workflow installs the exact toolchain and runs the same gates. No hosted CI run is claimed by local verification. The browser gate starts and stops its own processes and refuses existing servers. There are no load tests, benchmarks, or parameter sweeps.
+
+Individual commands:
+
+```sh
+npm run test:api
+npm run typecheck
+npm run build
+npm run api:package
+npm run test:e2e
+```
+
+## Official version references
+
+- [Spring Boot 3.5 system requirements](https://docs.spring.io/spring-boot/3.5/system-requirements.html)
+- [Spring Boot 3.5.16 parent POM](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-parent/3.5.16/spring-boot-starter-parent-3.5.16.pom)
+- [Next.js security releases](https://nextjs.org/blog)
+- [Playwright 1.62.1 browser manifest](https://github.com/microsoft/playwright/blob/v1.62.1/packages/playwright-core/browsers.json)
+- [Maven 3.9.11 distribution checksum](https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.11/apache-maven-3.9.11-bin.tar.gz.sha512)
diff --git a/evidence/E01/maven-dependencies.txt b/evidence/E01/maven-dependencies.txt
new file mode 100644
index 0000000..3de4fb1
--- /dev/null
+++ b/evidence/E01/maven-dependencies.txt
@@ -0,0 +1,64 @@
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
+\- org.springframework.boot:spring-boot-starter-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test:jar:3.5.16:test
+   +- org.springframework.boot:spring-boot-test-autoconfigure:jar:3.5.16:test
+   +- com.jayway.jsonpath:json-path:jar:2.9.0:test
+   |  \- org.slf4j:slf4j-api:jar:2.0.18:compile
+   +- jakarta.xml.bind:jakarta.xml.bind-api:jar:4.0.5:test
+   |  \- jakarta.activation:jakarta.activation-api:jar:2.1.4:test
+   +- net.minidev:json-smart:jar:2.5.2:test
+   |  \- net.minidev:accessors-smart:jar:2.5.2:test
+   |     \- org.ow2.asm:asm:jar:9.7.1:test
+   +- org.assertj:assertj-core:jar:3.27.7:test
+   |  \- net.bytebuddy:byte-buddy:jar:1.17.8:test
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
diff --git a/evidence/E01/runs.jsonl b/evidence/E01/runs.jsonl
new file mode 100644
index 0000000..c0473db
--- /dev/null
+++ b/evidence/E01/runs.jsonl
@@ -0,0 +1,4 @@
+{"command":"mvn -B -ntp -f backend/pom.xml package","startedAt":"2026-08-27T23:10:57.929Z","elapsedSeconds":23.674,"exitCode":0}
+{"command":"npm run typecheck","startedAt":"2026-08-27T23:11:21.604Z","elapsedSeconds":1.402,"exitCode":0}
+{"command":"npm run build","startedAt":"2026-08-27T23:11:23.006Z","elapsedSeconds":12.545,"exitCode":0}
+{"command":"npm run test:e2e","startedAt":"2026-08-27T23:11:35.552Z","elapsedSeconds":10.806,"exitCode":0}
diff --git a/evidence/E01/verification.md b/evidence/E01/verification.md
new file mode 100644
index 0000000..ecdf6f4
--- /dev/null
+++ b/evidence/E01/verification.md
@@ -0,0 +1,38 @@
+# E01 verification evidence
+
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
+Attempt: 1; initial orphan establishment, not a before/after failure claim.
+
+## Fixed inputs
+
+- Fixture `127.0.0.1:4321`, API `127.0.0.1:4322`, Next UI `127.0.0.1:4323`.
+- Create `Fixture monitor`, `/ok`, interval 60, enabled true; manually check once and display `SUCCEEDED / 200`. Reload confirms latest result is retained by the API.
+- Separate `/fail` check returns and displays `FAILED / 503 / HTTP_STATUS`.
+- Fixture `/redirect` returns 302 to `/ok`; test asserts that `/ok` request count does not increase. `/redirect-outside` points to controlled loopback trap port 4324; trap request count remains zero.
+- Direct non-fixture Monitor URL on port 4324 is rejected with 400 before any outbound request. No public monitoring destination is used.
+- Unit boundary cases are status 199, 200, 299, 302, 503 and fixed invalid scheme/hostname/port/credential/fragment/syntax inputs.
+
+## Executed verification
+
+One invocation: `fnm exec --using=24.19.0 npm run verify` on 2026-08-27 23:10 UTC (2026-08-28 KST). The command invokes the following steps once, in order. `runs.jsonl` preserves the generated command records.
+
+| Command | Outcome | Wall seconds |
+| --- | --- | ---: |
+| `mvn -B -ntp -f backend/pom.xml package` | PASS: 2 unit + 5 real-HTTP functional tests; executable jar built | 23.674 |
+| `npm run typecheck` | PASS | 1.402 |
+| `npm run build` | PASS: Next.js 16.3.3 production compilation | 12.545 |
+| `npm run test:e2e` | PASS: 2 actual Chromium browser tests; 1 worker | 10.806 |
+
+Browser case durations were 739 ms and 476 ms; suite wall duration was 10.0 seconds, including process startup. These are execution evidence, not latency requirements or performance claims. The browser gate uses the packaged Spring API and the Next development server. Hosted CI was configured but not executed here.
+
+Runtime observed: Eclipse Temurin 21.0.7+6-LTS, Maven 3.9.11, Node v24.19.0, npm 11.17.0, Playwright 1.62.1, Chromium revision 1234 / 151.0.7922.34. `npm install` resolved the committed lock and reported 0 vulnerabilities across 32 audited packages. Java transitive coordinates are captured in `maven-dependencies.txt` from the pinned POM/BOM.
+
+Invocation budget: 1 full verification; 1 unit/functional suite; 1 initial type check; 1 frontend build; 1 browser suite (2 cases); 0 failed verification cases, 0 automatic retries; 0 load/benchmark/profiler runs; 0 parameter sweeps. An additional static type check is recorded below after following Next's bundled guidance to ignore generated `next-env.d.ts` and run `next typegen` before `tsc`, so a fresh checkout can generate its type inputs. Product code and browser assertions were unchanged.
+
+Additional command: `/usr/bin/time -p fnm exec --using=24.19.0 npm run typecheck` passed in 1.32 seconds with `.next` and `next-env.d.ts` moved aside first. Thus total static type-check invocations are 2. The new script generates its type inputs before `tsc` and does not depend on preexisting build output.
+
+Auxiliary evidence collection: npm metadata and Maven dependency-tree resolution initially encountered sandbox DNS failures; their normal approved retries succeeded. A non-test package-export inspection command failed before the manifest was read directly. Dependency-tree collection took 9.136 seconds on its successful invocation; it did not run tests. None of these changed fixture parameters or test assertions.
+
+The official action manifests for checkout 4.2.2, setup-java 4.7.1, and setup-node 4.4.0 were successfully fetched to verify the workflow's pinned action tags exist.
+
+The implementation deliberately retains only the latest terminal result per Monitor in process memory. No persistence or production readiness claim is made.
