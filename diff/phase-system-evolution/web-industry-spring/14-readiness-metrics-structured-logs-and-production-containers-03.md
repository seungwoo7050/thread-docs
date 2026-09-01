## `test: preserve E24 repair1 compiler failure`

diff --git a/evidence/phase-1/E24/repair1/repair1-affected-java-native.json b/evidence/phase-1/E24/repair1/repair1-affected-java-native.json
new file mode 100644
index 0000000..6f2b0e8
--- /dev/null
+++ b/evidence/phase-1/E24/repair1/repair1-affected-java-native.json
@@ -0,0 +1,22 @@
+{
+  "profile": "phase-1",
+  "thread": "E24",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "invocation": {
+    "command": "mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package",
+    "startedAt": "2026-08-28T09:22:21.048583+00:00",
+    "attempt": 2,
+    "repair": 1,
+    "actor": "author",
+    "permission": "require_escalated network/cache authorization",
+    "forcedDescriptorRefreshes": 1,
+    "elapsedSeconds": 4.051,
+    "exitCode": 1,
+    "signal": null
+  },
+  "source": "output/phase-1/e24/repair1-affected-java.log",
+  "bytes": 3224,
+  "sha256": "6055dc397204b8f4d80df9fe77a31b6115c7c22461c4b4e1c5ecd740e7a34eb3",
+  "encoding": "utf-8",
+  "raw": "[INFO] Scanning for projects...\n[INFO] \n[INFO] ---------------------< dev.evolution:monitor-api >----------------------\n[INFO] Building monitor-api 0.0.1\n[INFO]   from pom.xml\n[INFO] --------------------------------[ jar ]---------------------------------\n[INFO] \n[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---\n[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed\n[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed\n[INFO] \n[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---\n[INFO] Copying 2 resources from src/main/resources to target/classes\n[INFO] Copying 8 resources from src/main/resources to target/classes\n[INFO] \n[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---\n[INFO] Recompiling the module because of changed source code.\n[INFO] Compiling 26 source files with javac [debug parameters release 21] to target/classes\n[INFO] \n[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---\n[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources\n[INFO] \n[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---\n[INFO] Recompiling the module because of changed dependency.\n[INFO] Compiling 20 source files with javac [debug parameters release 21] to target/test-classes\n[INFO] -------------------------------------------------------------\n[ERROR] COMPILATION ERROR : \n[INFO] -------------------------------------------------------------\n[ERROR] /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java:[58,18] incompatible types: try-with-resources not applicable to variable type\n    (io.micrometer.core.instrument.simple.SimpleMeterRegistry cannot be converted to java.lang.AutoCloseable)\n[INFO] 1 error\n[INFO] -------------------------------------------------------------\n[INFO] ------------------------------------------------------------------------\n[INFO] BUILD FAILURE\n[INFO] ------------------------------------------------------------------------\n[INFO] Total time:  3.051 s\n[INFO] Finished at: 2026-08-28T18:22:25+09:00\n[INFO] ------------------------------------------------------------------------\n[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.14.1:testCompile (default-testCompile) on project monitor-api: Compilation failure\n[ERROR] /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java:[58,18] incompatible types: try-with-resources not applicable to variable type\n[ERROR]     (io.micrometer.core.instrument.simple.SimpleMeterRegistry cannot be converted to java.lang.AutoCloseable)\n[ERROR] -> [Help 1]\n[ERROR] \n[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.\n[ERROR] Re-run Maven using the -X switch to enable full debug logging.\n[ERROR] \n[ERROR] For more information about the errors and possible solutions, please read the following articles:\n[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException\n"
+}
diff --git a/evidence/phase-1/E24/repair1/repair1-preservation.json b/evidence/phase-1/E24/repair1/repair1-preservation.json
new file mode 100644
index 0000000..2a74ca3
--- /dev/null
+++ b/evidence/phase-1/E24/repair1/repair1-preservation.json
@@ -0,0 +1,101 @@
+{
+  "profile": "phase-1",
+  "thread": "E24",
+  "attempt": 2,
+  "repair": 1,
+  "status": "FAIL_STOPPED",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "threadStart": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "adoptedWipCommit": "d74a594e0be190ce0db9c75fe973899e15763593",
+  "failure": {
+    "stage": "Maven testCompile",
+    "exitCode": 1,
+    "elapsedSeconds": 4.051,
+    "mavenElapsedSeconds": 3.051,
+    "location": "backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java:58",
+    "nativeDiagnostic": "io.micrometer.core.instrument.simple.SimpleMeterRegistry cannot be converted to java.lang.AutoCloseable",
+    "dependencyResolutionSucceeded": true,
+    "mainCompilationSucceeded": true,
+    "testsReached": false,
+    "retryAfterFailure": false
+  },
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "baselineSeconds": 7.477,
+    "affectedMavenInvocations": 2,
+    "affectedMavenFailures": 2,
+    "affectedMavenWallSeconds": 5.775,
+    "javaTestsExecuted": 0,
+    "imageBuildInvocations": 0,
+    "authorFullContainerRuns": 0,
+    "rootFullContainerRuns": 0,
+    "postgresStopRestoreSequences": 0,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "freshRepairTasksUsed": 1,
+    "freshRepairTasksMaximum": 2,
+    "additionalFreshRepairsRemaining": 1
+  },
+  "frozenInputsUnchanged": {
+    "evidence/phase-1/E24/fixture.json": "47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd",
+    "scripts/e24-seed.mjs": "16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025",
+    "scripts/e24-baseline.mjs": "37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4",
+    "evidence/phase-1/E24/fixtures.md": "5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c"
+  },
+  "adoptedWipFilesStillUnchanged": {
+    ".dockerignore": "037d68e018325d47a656397e8a5f25c75eb3e819e8a8ef30dfec3f39252c5903",
+    ".github/workflows/ci.yml": "6be6d23aa460b59114e4a54d36436166d59dd38bd6f2ea5a1d8b37083256adef",
+    "Dockerfile.api": "772bf2675423c425cf634e86ec4a85cffbe4bdaaecea8b0331af576df1271572",
+    "Dockerfile.frontend": "42da9feb4474ab612ee0aa8621bcd91fd071d37d5e148a1641c84be671770b41",
+    "backend/pom.xml": "9c010b8a0bf11de522b5c0fc66dfffa7ba55ae2212b56ff8206e63c710c1a901",
+    "backend/src/main/java/dev/evolution/monitor/ApiErrors.java": "81c0f40ecee1eeca6c3d55df7c7a6b022e5db2eaddbe4aa696335834d4ad1f0e",
+    "backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java": "00865cb087e10016fc8dba499b673ed1e2e89fc064028a078428a5f55285c0b0",
+    "backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java": "8b65df49a8409afe7bbdfcf5de57f82e968ef338272d5226dbf7dbe142d0eede",
+    "backend/src/main/java/dev/evolution/monitor/CheckQueue.java": "6d68d05bdfcb919160fc6a39fb456befd1308c0e594c72487fbaab73b0a6ffde",
+    "backend/src/main/java/dev/evolution/monitor/CheckWorker.java": "d955d7d0f55ad9671cc46185f02aea4bf6cb36c3c482427f425d2cd2aeca16fc",
+    "backend/src/main/java/dev/evolution/monitor/HttpObservations.java": "5d50f6d91547de7c33f1b228b63dc1c95f7ce6ea1c07d1ec023fe55c5a1ccb40",
+    "backend/src/main/java/dev/evolution/monitor/ProcessObservations.java": "8f31dcf1a2cc3f1943a8671a0b6bf98c441520e2aa94d0dd47552d451eca56a5",
+    "backend/src/main/java/dev/evolution/monitor/WorkerManagement.java": "50662ea34e7678ee625eb1ce361711727a3a5bc1cb34ff9ac268597997fa1eb5",
+    "backend/src/main/java/dev/evolution/monitor/WorkerProcess.java": "fc626881cde513da0eb992f8778c6ee1b17d2bfc77a1a7c3b0052ee4b73e7df6",
+    "backend/src/main/resources/application-worker.properties": "c43808ca3ace52e98d8a1e510680b096da1bf591ccaf65a387dada1d740b2290",
+    "backend/src/main/resources/application.properties": "8f4fe19cc4135428d9a63b9415aa83423b9e40a9f93cadbcefac019065266919",
+    "backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java": "35c193a4b02a088a030213d31381540f5b6e96a63ecfc145ff9c7f8f9469216b",
+    "backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java": "7b7850854e4c4c1dc913616f48cec58d073e6cf1c7b82418a28f50f236e88eb6",
+    "backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java": "3904c2a5851af73494f24ab0cb9ed36b7c5333f9893469a8a2bc8919c8315ad9",
+    "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "247c98f468b8f5a1322f41b4287209f7ecdb316db2b1bfc9f47d16301bb5ad8c",
+    "compose.e24.yaml": "28a9daf0021b0127335044ef31fdb30bf7451cbdf1fcc8f35586d7bc33b09111",
+    "compose.production.yaml": "1f995bb361c0e90dcc562cfe82ff05067137d900b392a270e361efb9ae26de89",
+    "next.config.mjs": "bbf15404bb72ee9b1abe5e7a22e99ab9398c2dba6d8e6e2025593639fcb3d7d7",
+    "package.json": "ef05c9197eda68c152eed320d91570a327b39266af2d0ab189a440d450ef5948",
+    "scripts/e24-fixture.mjs": "20bfd59220394c824a1910183e8857b707164e7d4387957846c8bc155cc08509"
+  },
+  "cleanup": {
+    "mavenProcessExited": true,
+    "noApplicationProcessOrBrowserStartedByThisRepair": true,
+    "noContainerOrSchemaCreatedByThisRepair": true,
+    "postgresStopOrRestoreCommandsThisRepair": 0,
+    "publicDataAndVolumesUntouchedByThisRepair": true
+  },
+  "pending": [
+    "Correct the observed test compilation error only in a separately authorized fresh repair.",
+    "Affected Maven gate has not passed; image build and missing full container observer/gate remain unexecuted.",
+    "Root independent gate, final artifacts and tags remain pending."
+  ],
+  "nativeFiles": [
+    {
+      "source": "output/phase-1/e24/repair1-affected-java.started.json",
+      "bytes": 322,
+      "sha256": "797f0f4a83c3ecc1b1822e0b6e6d8126fca4788d71f3753565c84780035b579f",
+      "encoding": "utf-8",
+      "raw": "{\"command\": \"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\", \"startedAt\": \"2026-08-28T09:22:21.048583+00:00\", \"attempt\": 2, \"repair\": 1, \"actor\": \"author\", \"permission\": \"require_escalated network/cache authorization\", \"forcedDescriptorRefreshes\": 1}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "bytes": 695,
+      "sha256": "b8245c603f27a78f8965e1631bf375f59a4a1a038f9d5343898bfaf28453b327",
+      "encoding": "utf-8",
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:22:21.048583+00:00\",\"attempt\":2,\"repair\":1,\"actor\":\"author\",\"permission\":\"require_escalated network/cache authorization\",\"forcedDescriptorRefreshes\":1,\"elapsedSeconds\":4.051,\"exitCode\":1,\"signal\":null}\n"
+    }
+  ]
+}


