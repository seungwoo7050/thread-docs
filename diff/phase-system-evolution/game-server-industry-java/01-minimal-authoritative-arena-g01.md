# 최소 Authoritative Arena

## `feat: establish single-room Netty arena baseline`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..541f1ec
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,7 @@
+.gradle/
+build/
+evidence/*.log
+evidence/*-result.json
+evidence/runs/
+*.hprof
+hs_err_pid*
diff --git a/TRACK.md b/TRACK.md
new file mode 100644
index 0000000..70a535f
--- /dev/null
+++ b/TRACK.md
@@ -0,0 +1,51 @@
+# Java arena — G01 baseline
+
+Thread: G01. Profile: realtime-core. Spec revision: `5a6e4a2f8fc71d4be18c3279583bfc2558d5c232`.
+
+## Frozen versions
+
+- Runtime/compiler: Eclipse Temurin OpenJDK 21.0.7+6, Java release 21.
+- Build: Gradle wrapper 8.10.2 (wrapper files from the orchestrator platform preflight).
+- Network: Netty 4.1.114.Final, Java NIO selector transport on macOS arm64.
+- JSON: Jackson 2.17.2; test framework: JUnit Jupiter 5.10.5 and Platform 1.10.5.
+- Dependency resolution is frozen in `gradle.lockfile`. No Spring, database or external service.
+
+The wrapper uses the locally installed Temurin path when JAVA_HOME is unset. On another host set JAVA_HOME to Temurin 21.0.7. All dependency versions stay fixed for later Threads unless required by their task. `./track` uses offline Gradle; install dependencies once in a connected environment using the same pinned build with `--write-locks` if necessary, then retain the resulting lock.
+
+## Command contract (frozen before verification)
+
+```sh
+./track build --write-locks  # initial lock generation only
+./track build               # clean compile main/tests and install executable; no tests run
+./track unit-test           # Gradle test, excludes ServerIntegrationTest
+./track integration-test    # real loopback tests, timer/executor cleanup and bounded CLI SIGTERM
+./track scenario-run /absolute/path/to/G01.json /absolute/path/to/result.json
+./track replay-verify /absolute/path/to/replay /absolute/path/to/evidence
+./track server config/server.json
+```
+
+`replay-verify` exits 2 with NOT ACTIVATED until G07. Build does not execute tests or scenarios. Unit/integration tasks re-execute tests every invocation; compilation is reused after the explicit build. Reports are in `build/test-results/{test,integrationTest}` and `build/reports/tests/{test,integrationTest}`. Netty leak detection is PARANOID for all test JVMs. Shell output and command exits are recorded in `evidence/G01-verification.md`; large/generated outputs are ignored.
+
+## Ownership and bounds
+
+Connection lifetime belongs to its non-sharable Netty channel handler. Each accepted channel has one non-sharable `CompleteFrame` handler. It copies at most 16,384 JSON bytes and auto-releases each inbound `ByteBuf`. G01 requires exactly one complete frame per read, deliberately without an incremental parser. Receive allocation is 16,388 bytes and one message per read.
+
+Session registry and Room state belong to one dedicated room-owner thread. Network callbacks submit to its `ArrayBlockingQueue(1024)` and never mutate a Room. Each Room public operation checks the constructing owner thread; unit tests reject mutation from another thread. There is one room and at most eight accepted connections. UUID identifiers are server-generated, distinct from connection objects, and not input authority. Detailed lifecycle and identity matrices remain G03 work.
+
+Each player's pending input storage holds at most 64 intents and rejects overflow with `INPUT_QUEUE_FULL`. An owner tick drains that bounded storage, selects the latest pending direction/TAG, moves players in ASCII ID order, then evaluates one-shot TAG with 64-bit squared distance. Direction persists; TAG does not. No seq, target tick or rate-limit contract is activated. Player data is integer only; unknown position/score fields are ignored.
+
+Both Netty event loops use explicit bounded task and tail queues (1,024 each), not an unbounded executor queue. Room commands use a one-thread `ThreadPoolExecutor` with `AbortPolicy`; overflow causes a terminal `INPUT_QUEUE_FULL` reply attempt. Each connection bounds outstanding writes to 64. The last slot is reserved as a `CONTROL_BACKPRESSURE` terminal reply. No snapshot retention or delta queue exists at G01. Serialized outbound buffers transfer ownership to Netty on `writeAndFlush`; completion decrements an outstanding-buffer metric. Unit tests check actual inbound and outbound reference counts reach zero, including channel disposal. Snapshot cadence/coalescing remain later Threads.
+
+The manual clock advances an explicit 50ms per tick request with no sleeps or system-clock access. The TCP runner waits for INPUT_ACK sent after owner-side enqueue before advancing the clock. The standalone server uses a single 50ms timer thread with one wait and no delayed-task queue; G04 will replace its intentionally basic scheduling with an accumulator and bounded catch-up.
+
+The calling main/test thread coordinates shutdown: stop/join the timer, close listener and client channels, drain the I/O callback boundary, close/clear owner state, shut down/join the owner and both event loops. No event loop blocks on another thread. Clients observe LOBBY/RUNNING/FINISHED from server replies and CLOSED from TCP EOF, while the server records its actual terminal lifecycle. Assertions require zero live channels, pending writes, mailbox tasks and owned threads, terminated executors, stopped timer and locally closed client sockets.
+
+The existing integration suite also starts the actual CLI as one child process on loopback port 0, waits for its SERVER_READY line, performs HELLO/WELCOME through TCP, sends normal SIGTERM, and requires exit 143 within five seconds, a zero-resource shutdown record and successful listener-port rebinding. The optional server config field `shutdown_evidence` names the JSON cleanup file written by the shutdown hook. The process check has no sleep, retry, changing load or canonical-scenario rerun.
+
+## Fixed evidence and scope
+
+The canonical runner reads all clients, setup steps, input boundaries, directions, TAG roles and tick count from the supplied scenario. It resolves role names to actual server-issued identifiers and returns the final view received independently by both TCP clients. It never writes state or substitutes a separate simulation. Scenario SHA-256 is input provenance, not a state hash.
+
+G01 initial budget: build/compile <=8, unit suites <=4, integration suites <=2, canonical scenario <=1; network-fault and load runs exactly zero. Main has its own separately frozen one-build/one-unit/one-integration/one-scenario verification budget. No test sleep, microbenchmark, fuzzing, replay, UDP, reconnect, many-room or distributed implementation is included.
+
+JVM concurrency evidence uses owner-confinement assertions plus real cross-thread Netty handoff, actual thread joins and shutdown assertions. No JVM race detector is installed; no sanitizer result is claimed.
diff --git a/build.gradle b/build.gradle
new file mode 100644
index 0000000..aececfa
--- /dev/null
+++ b/build.gradle
@@ -0,0 +1,39 @@
+plugins {
+    id 'java'
+    id 'application'
+}
+
+repositories { mavenCentral() }
+java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
+application { mainClass = 'arena.ArenaMain' }
+
+dependencies {
+    implementation 'io.netty:netty-transport:4.1.114.Final'
+    implementation 'io.netty:netty-buffer:4.1.114.Final'
+    implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.2'
+    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.5'
+    testRuntimeOnly 'org.junit.platform:junit-platform-launcher:1.10.5'
+}
+
+dependencyLocking { lockAllConfigurations() }
+tasks.register('resolveLockedDependencies') {
+    doLast { configurations.findAll { it.canBeResolved }.each { it.resolve() } }
+}
+tasks.withType(JavaCompile).configureEach {
+    options.encoding = 'UTF-8'
+    options.release = 21
+    options.compilerArgs += ['-Xlint:all', '-Werror']
+}
+tasks.withType(Test).configureEach {
+    useJUnitPlatform()
+    maxParallelForks = 1
+    systemProperty 'io.netty.leakDetection.level', 'paranoid'
+    outputs.upToDateWhen { false }
+    testLogging { events 'passed', 'skipped', 'failed'; exceptionFormat 'full' }
+}
+test { exclude '**/*IntegrationTest.class' }
+tasks.register('integrationTest', Test) {
+    testClassesDirs = sourceSets.test.output.classesDirs
+    classpath = sourceSets.test.runtimeClasspath
+    include '**/*IntegrationTest.class'
+}
diff --git a/config/server.json b/config/server.json
new file mode 100644
index 0000000..37de287
--- /dev/null
+++ b/config/server.json
@@ -0,0 +1 @@
+{"host":"127.0.0.1","port":7777}
diff --git a/evidence/G01-verification.md b/evidence/G01-verification.md
new file mode 100644
index 0000000..efed26a
--- /dev/null
+++ b/evidence/G01-verification.md
@@ -0,0 +1,42 @@
+# G01 initial attempt verification ledger
+
+START: UNBORN. Thread: G01. Spec revision: 5a6e4a2f8fc71d4be18c3279583bfc2558d5c232.
+
+Frozen caps: builds 8, unit suites 4, integration suites 2, canonical scenario 1; fault runs 0; load runs 0.
+
+All commands below ran from the Java worktree. Each stdout/stderr stream was redirected to the listed raw log, and the command exit was captured by the execution tool.
+
+| Run | Exact command | Exit | Raw output |
+|---|---|---:|---|
+| build 1 | `./track build --write-locks` | 0 | `evidence/build-1.log` |
+| unit 1 | `./track unit-test` | 0 | `evidence/unit-1.log`; 7 passed |
+| integration 1 | `./track integration-test` | 0 | `evidence/integration-1.log`; 2 passed |
+| scenario 1 | `./track scenario-run /Users/woopinbell/Desktop/working/workflow/game-server-systems-evolution/index/scenarios/G01.json /private/tmp/game-server-systems-evolution-5a6e4a2f/worktrees/industry-java/evidence/G01-result.json` | 0 | `evidence/scenario-1.log`, `evidence/G01-result.json` |
+| build 2 | `./track build` | 0 | `evidence/build-2.log` |
+| unit 2 | `./track unit-test` | 0 | `evidence/unit-2.log`; 7 passed |
+| integration 2 | `./track integration-test` | 0 | `evidence/integration-2.log`; 3 passed |
+
+After integration 1 the main reviewer requested one additional CLI process-stop assertion within the existing integration suite. The canonical run had already started and completed successfully before that request was handled; it was not repeated. Only the optional standalone-server shutdown evidence path, listener SO_REUSEADDR setting for deterministic post-close rebinding, and the added process test changed after that run. No simulation/clock/scenario logic changed. The final clean build/unit/integration results all passed. Main's separate canonical verification must cover final END.
+
+Preserved suite XML is under `evidence/runs/unit-1/`, `evidence/runs/integration-1/`, `evidence/runs/unit-2/` and `evidence/runs/integration-2/` so later clean builds do not erase it. The final integration pass includes child process SIGTERM exit 143 within five seconds, a parsed/asserted zero-resource shutdown file, client EOF and successful listener-port rebinding. JUnit's temporary child files are removed after the assertion.
+
+Final cumulative usage: **2/8 build/compile invocations, 2/4 unit suites, 2/2 integration suites, 1/1 canonical scenario, 0 network-fault runs, 0 load runs**. All seven verification commands exited 0. The unit/integration Gradle invocations reused UP-TO-DATE compilation. No retry, fixture change, threshold change or skipped assertion occurred. There are no further local integration/canonical runs available in this attempt.
+
+Raw output SHA-256:
+
+```text
+7d40b60fa9ba27b5b68dd393998a989edea7a2a3dd947b2fe7ca9e4759659dd3  build-1.log
+f7d4b31db535026de0e18d7109190073f01ae04d70084bd88a76a50c10ac955c  build-2.log
+c61ea49da53469bbf2588cbae14b582e1015f314545c39e364d75499dfb602b4  unit-1.log
+4daaa9c7e55e7cf6dc35d4864a9a9a35c96c3adf88e8d93badb45bc7408e67fa  unit-2.log
+572a2391e7abd774b3745b80c86ff38e367638cbdcb6d161e051ad94f40fadd7  integration-1.log
+f87c6f8f01aa4074aff6bf9f81ec804852fc3cf30ec33394d7e617cd0d529a86  integration-2.log
+599ee5900f33eff9afa0aa6abfd39e9af099afe761dfbb2048cda44e813f7b27  scenario-1.log
+330180ab532d239951c4054093730b7a25b9190e02c40474938e17ef3f0efbe7  G01-result.json
+```
+
+Canonical observation: 1200 ticks, last tick 1199; alpha `(50000,50000,STOP,score=1)`, bravo `(50000,50000,STOP,score=0)`; all channels, writes, mailbox tasks, owned threads and client sockets cleaned. Pending-input/mailbox/outbound high-water marks were 1/1/2. Scenario SHA-256 was `dae00089308ed65d27c1a196308216bb8b1abac0586a86dbfa83a797eb1dc51a`. State hash and replay inactive.
+
+Environment observations: `/usr/libexec/java_home -V` exited 1 (macOS system runtime not registered); SDKMAN Temurin 21.0.7 installation is available and the wrapper selects it. File inspections, `javap` metadata inspection and copying the platform-only Gradle wrapper are not build/test runs.
+
+`/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/bin/java -version` exited 0 and reported OpenJDK 21.0.7, Temurin-21.0.7+6-LTS. Build output uses Gradle 8.10.2. No sanitizer or JVM race-detector result is claimed; owner rejection and actual Netty/owner/timer shutdown tests supply the bounded concurrency evidence.
diff --git a/gradle.lockfile b/gradle.lockfile
new file mode 100644
index 0000000..cc3fab5
--- /dev/null
+++ b/gradle.lockfile
@@ -0,0 +1,22 @@
+# This is a Gradle generated file for dependency locking.
+# Manual edits can break the build and are not advised.
+# This file is expected to be part of source control.
+com.fasterxml.jackson.core:jackson-annotations:2.17.2=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+com.fasterxml.jackson.core:jackson-core:2.17.2=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+com.fasterxml.jackson.core:jackson-databind:2.17.2=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+com.fasterxml.jackson:jackson-bom:2.17.2=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+io.netty:netty-buffer:4.1.114.Final=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+io.netty:netty-common:4.1.114.Final=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+io.netty:netty-resolver:4.1.114.Final=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+io.netty:netty-transport:4.1.114.Final=compileClasspath,runtimeClasspath,testCompileClasspath,testRuntimeClasspath
+org.apiguardian:apiguardian-api:1.1.2=testCompileClasspath
+org.junit.jupiter:junit-jupiter-api:5.10.5=testCompileClasspath,testRuntimeClasspath
+org.junit.jupiter:junit-jupiter-engine:5.10.5=testRuntimeClasspath
+org.junit.jupiter:junit-jupiter-params:5.10.5=testCompileClasspath,testRuntimeClasspath
+org.junit.jupiter:junit-jupiter:5.10.5=testCompileClasspath,testRuntimeClasspath
+org.junit.platform:junit-platform-commons:1.10.5=testCompileClasspath,testRuntimeClasspath
+org.junit.platform:junit-platform-engine:1.10.5=testRuntimeClasspath
+org.junit.platform:junit-platform-launcher:1.10.5=testRuntimeClasspath
+org.junit:junit-bom:5.10.5=testCompileClasspath,testRuntimeClasspath
+org.opentest4j:opentest4j:1.3.0=testCompileClasspath,testRuntimeClasspath
+empty=annotationProcessor,testAnnotationProcessor
diff --git a/gradle.properties b/gradle.properties
new file mode 100644
index 0000000..7b5cf3d
--- /dev/null
+++ b/gradle.properties
@@ -0,0 +1,4 @@
+org.gradle.daemon=false
+org.gradle.parallel=false
+org.gradle.workers.max=2
+org.gradle.jvmargs=-Xmx512m
diff --git a/gradle/wrapper/gradle-wrapper.jar b/gradle/wrapper/gradle-wrapper.jar
new file mode 100644
index 0000000..a4b76b9
Binary files /dev/null and b/gradle/wrapper/gradle-wrapper.jar differ
diff --git a/gradle/wrapper/gradle-wrapper.properties b/gradle/wrapper/gradle-wrapper.properties
new file mode 100644
index 0000000..df97d72
--- /dev/null
+++ b/gradle/wrapper/gradle-wrapper.properties
@@ -0,0 +1,7 @@
+distributionBase=GRADLE_USER_HOME
+distributionPath=wrapper/dists
+distributionUrl=https\://services.gradle.org/distributions/gradle-8.10.2-bin.zip
+networkTimeout=10000
+validateDistributionUrl=true
+zipStoreBase=GRADLE_USER_HOME
+zipStorePath=wrapper/dists
diff --git a/gradlew b/gradlew
new file mode 100755
index 0000000..f5feea6
--- /dev/null
+++ b/gradlew
@@ -0,0 +1,252 @@
+#!/bin/sh
+
+#
+# Copyright © 2015-2021 the original authors.
+#
+# Licensed under the Apache License, Version 2.0 (the "License");
+# you may not use this file except in compliance with the License.
+# You may obtain a copy of the License at
+#
+#      https://www.apache.org/licenses/LICENSE-2.0
+#
+# Unless required by applicable law or agreed to in writing, software
+# distributed under the License is distributed on an "AS IS" BASIS,
+# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
+# See the License for the specific language governing permissions and
+# limitations under the License.
+#
+# SPDX-License-Identifier: Apache-2.0
+#
+
+##############################################################################
+#
+#   Gradle start up script for POSIX generated by Gradle.
+#
+#   Important for running:
+#
+#   (1) You need a POSIX-compliant shell to run this script. If your /bin/sh is
+#       noncompliant, but you have some other compliant shell such as ksh or
+#       bash, then to run this script, type that shell name before the whole
+#       command line, like:
+#
+#           ksh Gradle
+#
+#       Busybox and similar reduced shells will NOT work, because this script
+#       requires all of these POSIX shell features:
+#         * functions;
+#         * expansions «$var», «${var}», «${var:-default}», «${var+SET}»,
+#           «${var#prefix}», «${var%suffix}», and «$( cmd )»;
+#         * compound commands having a testable exit status, especially «case»;
+#         * various built-in commands including «command», «set», and «ulimit».
+#
+#   Important for patching:
+#
+#   (2) This script targets any POSIX shell, so it avoids extensions provided
+#       by Bash, Ksh, etc; in particular arrays are avoided.
+#
+#       The "traditional" practice of packing multiple parameters into a
+#       space-separated string is a well documented source of bugs and security
+#       problems, so this is (mostly) avoided, by progressively accumulating
+#       options in "$@", and eventually passing that to Java.
+#
+#       Where the inherited environment variables (DEFAULT_JVM_OPTS, JAVA_OPTS,
+#       and GRADLE_OPTS) rely on word-splitting, this is performed explicitly;
+#       see the in-line comments for details.
+#
+#       There are tweaks for specific operating systems such as AIX, CygWin,
+#       Darwin, MinGW, and NonStop.
+#
+#   (3) This script is generated from the Groovy template
+#       https://github.com/gradle/gradle/blob/HEAD/platforms/jvm/plugins-application/src/main/resources/org/gradle/api/internal/plugins/unixStartScript.txt
+#       within the Gradle project.
+#
+#       You can find Gradle at https://github.com/gradle/gradle/.
+#
+##############################################################################
+
+# Attempt to set APP_HOME
+
+# Resolve links: $0 may be a link
+app_path=$0
+
+# Need this for daisy-chained symlinks.
+while
+    APP_HOME=${app_path%"${app_path##*/}"}  # leaves a trailing /; empty if no leading path
+    [ -h "$app_path" ]
+do
+    ls=$( ls -ld "$app_path" )
+    link=${ls#*' -> '}
+    case $link in             #(
+      /*)   app_path=$link ;; #(
+      *)    app_path=$APP_HOME$link ;;
+    esac
+done
+
+# This is normally unused
+# shellcheck disable=SC2034
+APP_BASE_NAME=${0##*/}
+# Discard cd standard output in case $CDPATH is set (https://github.com/gradle/gradle/issues/25036)
+APP_HOME=$( cd -P "${APP_HOME:-./}" > /dev/null && printf '%s
+' "$PWD" ) || exit
+
+# Use the maximum available, or set MAX_FD != -1 to use that value.
+MAX_FD=maximum
+
+warn () {
+    echo "$*"
+} >&2
+
+die () {
+    echo
+    echo "$*"
+    echo
+    exit 1
+} >&2
+
+# OS specific support (must be 'true' or 'false').
+cygwin=false
+msys=false
+darwin=false
+nonstop=false
+case "$( uname )" in                #(
+  CYGWIN* )         cygwin=true  ;; #(
+  Darwin* )         darwin=true  ;; #(
+  MSYS* | MINGW* )  msys=true    ;; #(
+  NONSTOP* )        nonstop=true ;;
+esac
+
+CLASSPATH=$APP_HOME/gradle/wrapper/gradle-wrapper.jar
+
+
+# Determine the Java command to use to start the JVM.
+if [ -n "$JAVA_HOME" ] ; then
+    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
+        # IBM's JDK on AIX uses strange locations for the executables
+        JAVACMD=$JAVA_HOME/jre/sh/java
+    else
+        JAVACMD=$JAVA_HOME/bin/java
+    fi
+    if [ ! -x "$JAVACMD" ] ; then
+        die "ERROR: JAVA_HOME is set to an invalid directory: $JAVA_HOME
+
+Please set the JAVA_HOME variable in your environment to match the
+location of your Java installation."
+    fi
+else
+    JAVACMD=java
+    if ! command -v java >/dev/null 2>&1
+    then
+        die "ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.
+
+Please set the JAVA_HOME variable in your environment to match the
+location of your Java installation."
+    fi
+fi
+
+# Increase the maximum file descriptors if we can.
+if ! "$cygwin" && ! "$darwin" && ! "$nonstop" ; then
+    case $MAX_FD in #(
+      max*)
+        # In POSIX sh, ulimit -H is undefined. That's why the result is checked to see if it worked.
+        # shellcheck disable=SC2039,SC3045
+        MAX_FD=$( ulimit -H -n ) ||
+            warn "Could not query maximum file descriptor limit"
+    esac
+    case $MAX_FD in  #(
+      '' | soft) :;; #(
+      *)
+        # In POSIX sh, ulimit -n is undefined. That's why the result is checked to see if it worked.
+        # shellcheck disable=SC2039,SC3045
+        ulimit -n "$MAX_FD" ||
+            warn "Could not set maximum file descriptor limit to $MAX_FD"
+    esac
+fi
+
+# Collect all arguments for the java command, stacking in reverse order:
+#   * args from the command line
+#   * the main class name
+#   * -classpath
+#   * -D...appname settings
+#   * --module-path (only if needed)
+#   * DEFAULT_JVM_OPTS, JAVA_OPTS, and GRADLE_OPTS environment variables.
+
+# For Cygwin or MSYS, switch paths to Windows format before running java
+if "$cygwin" || "$msys" ; then
+    APP_HOME=$( cygpath --path --mixed "$APP_HOME" )
+    CLASSPATH=$( cygpath --path --mixed "$CLASSPATH" )
+
+    JAVACMD=$( cygpath --unix "$JAVACMD" )
+
+    # Now convert the arguments - kludge to limit ourselves to /bin/sh
+    for arg do
+        if
+            case $arg in                                #(
+              -*)   false ;;                            # don't mess with options #(
+              /?*)  t=${arg#/} t=/${t%%/*}              # looks like a POSIX filepath
+                    [ -e "$t" ] ;;                      #(
+              *)    false ;;
+            esac
+        then
+            arg=$( cygpath --path --ignore --mixed "$arg" )
+        fi
+        # Roll the args list around exactly as many times as the number of
+        # args, so each arg winds up back in the position where it started, but
+        # possibly modified.
+        #
+        # NB: a `for` loop captures its iteration list before it begins, so
+        # changing the positional parameters here affects neither the number of
+        # iterations, nor the values presented in `arg`.
+        shift                   # remove old arg
+        set -- "$@" "$arg"      # push replacement arg
+    done
+fi
+
+
+# Add default JVM options here. You can also use JAVA_OPTS and GRADLE_OPTS to pass JVM options to this script.
+DEFAULT_JVM_OPTS='"-Xmx64m" "-Xms64m"'
+
+# Collect all arguments for the java command:
+#   * DEFAULT_JVM_OPTS, JAVA_OPTS, JAVA_OPTS, and optsEnvironmentVar are not allowed to contain shell fragments,
+#     and any embedded shellness will be escaped.
+#   * For example: A user cannot expect ${Hostname} to be expanded, as it is an environment variable and will be
+#     treated as '${Hostname}' itself on the command line.
+
+set -- \
+        "-Dorg.gradle.appname=$APP_BASE_NAME" \
+        -classpath "$CLASSPATH" \
+        org.gradle.wrapper.GradleWrapperMain \
+        "$@"
+
+# Stop when "xargs" is not available.
+if ! command -v xargs >/dev/null 2>&1
+then
+    die "xargs is not available"
+fi
+
+# Use "xargs" to parse quoted args.
+#
+# With -n1 it outputs one arg per line, with the quotes and backslashes removed.
+#
+# In Bash we could simply go:
+#
+#   readarray ARGS < <( xargs -n1 <<<"$var" ) &&
+#   set -- "${ARGS[@]}" "$@"
+#
+# but POSIX shell has neither arrays nor command substitution, so instead we
+# post-process each arg (as a line of input to sed) to backslash-escape any
+# character that might be a shell metacharacter, then use eval to reverse
+# that process (while maintaining the separation between arguments), and wrap
+# the whole thing up as a single "set" statement.
+#
+# This will of course break if any of these variables contains a newline or
+# an unmatched quote.
+#
+
+eval "set -- $(
+        printf '%s\n' "$DEFAULT_JVM_OPTS $JAVA_OPTS $GRADLE_OPTS" |
+        xargs -n1 |
+        sed ' s~[^-[:alnum:]+,./:=@_]~\\&~g; ' |
+        tr '\n' ' '
+    )" '"$@"'
+
+exec "$JAVACMD" "$@"
diff --git a/gradlew.bat b/gradlew.bat
new file mode 100644
index 0000000..9d21a21
--- /dev/null
+++ b/gradlew.bat
@@ -0,0 +1,94 @@
+@rem
+@rem Copyright 2015 the original author or authors.
+@rem
+@rem Licensed under the Apache License, Version 2.0 (the "License");
+@rem you may not use this file except in compliance with the License.
+@rem You may obtain a copy of the License at
+@rem
+@rem      https://www.apache.org/licenses/LICENSE-2.0
+@rem
+@rem Unless required by applicable law or agreed to in writing, software
+@rem distributed under the License is distributed on an "AS IS" BASIS,
+@rem WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
+@rem See the License for the specific language governing permissions and
+@rem limitations under the License.
+@rem
+@rem SPDX-License-Identifier: Apache-2.0
+@rem
+
+@if "%DEBUG%"=="" @echo off
+@rem ##########################################################################
+@rem
+@rem  Gradle startup script for Windows
+@rem
+@rem ##########################################################################
+
+@rem Set local scope for the variables with windows NT shell
+if "%OS%"=="Windows_NT" setlocal
+
+set DIRNAME=%~dp0
+if "%DIRNAME%"=="" set DIRNAME=.
+@rem This is normally unused
+set APP_BASE_NAME=%~n0
+set APP_HOME=%DIRNAME%
+
+@rem Resolve any "." and ".." in APP_HOME to make it shorter.
+for %%i in ("%APP_HOME%") do set APP_HOME=%%~fi
+
+@rem Add default JVM options here. You can also use JAVA_OPTS and GRADLE_OPTS to pass JVM options to this script.
+set DEFAULT_JVM_OPTS="-Xmx64m" "-Xms64m"
+
+@rem Find java.exe
+if defined JAVA_HOME goto findJavaFromJavaHome
+
+set JAVA_EXE=java.exe
+%JAVA_EXE% -version >NUL 2>&1
+if %ERRORLEVEL% equ 0 goto execute
+
+echo. 1>&2
+echo ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH. 1>&2
+echo. 1>&2
+echo Please set the JAVA_HOME variable in your environment to match the 1>&2
+echo location of your Java installation. 1>&2
+
+goto fail
+
+:findJavaFromJavaHome
+set JAVA_HOME=%JAVA_HOME:"=%
+set JAVA_EXE=%JAVA_HOME%/bin/java.exe
+
+if exist "%JAVA_EXE%" goto execute
+
+echo. 1>&2
+echo ERROR: JAVA_HOME is set to an invalid directory: %JAVA_HOME% 1>&2
+echo. 1>&2
+echo Please set the JAVA_HOME variable in your environment to match the 1>&2
+echo location of your Java installation. 1>&2
+
+goto fail
+
+:execute
+@rem Setup the command line
+
+set CLASSPATH=%APP_HOME%\gradle\wrapper\gradle-wrapper.jar
+
+
+@rem Execute Gradle
+"%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %*
+
+:end
+@rem End local scope for the variables with windows NT shell
+if %ERRORLEVEL% equ 0 goto mainEnd
+
+:fail
+rem Set variable GRADLE_EXIT_CONSOLE if you need the _script_ return code instead of
+rem the _cmd.exe /c_ return code!
+set EXIT_CODE=%ERRORLEVEL%
+if %EXIT_CODE% equ 0 set EXIT_CODE=1
+if not ""=="%GRADLE_EXIT_CONSOLE%" exit %EXIT_CODE%
+exit /b %EXIT_CODE%
+
+:mainEnd
+if "%OS%"=="Windows_NT" endlocal
+
+:omega
diff --git a/settings.gradle b/settings.gradle
new file mode 100644
index 0000000..1f7b0de
--- /dev/null
+++ b/settings.gradle
@@ -0,0 +1 @@
+rootProject.name = 'arena-java'
diff --git a/src/main/java/arena/ArenaMain.java b/src/main/java/arena/ArenaMain.java
new file mode 100644
index 0000000..ab1bbd6
--- /dev/null
+++ b/src/main/java/arena/ArenaMain.java
@@ -0,0 +1,37 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.concurrent.CountDownLatch;
+
+public final class ArenaMain {
+    private ArenaMain() { }
+
+    public static void main(String[] args) throws Exception {
+        if (args.length == 3 && args[0].equals("scenario-run")) {
+            ObjectNode evidence = ScenarioRunner.run(Path.of(args[1]));
+            Path output = Path.of(args[2]);
+            if (output.toAbsolutePath().getParent() != null) Files.createDirectories(output.toAbsolutePath().getParent());
+            Files.write(output, Json.MAPPER.writerWithDefaultPrettyPrinter().writeValueAsBytes(evidence));
+            System.out.println("scenario PASS: " + output);
+            return;
+        }
+        if (args.length == 2 && args[0].equals("server")) {
+            ObjectNode config = Json.read(Files.readAllBytes(Path.of(args[1])));
+            ArenaServer server = new ArenaServer(config.path("host").asText("127.0.0.1"), config.path("port").asInt(7777), false);
+            Runtime.getRuntime().addShutdownHook(new Thread(() -> {
+                server.close();
+                if (config.has("shutdown_evidence")) {
+                    try {
+                        Files.write(Path.of(Json.text(config, "shutdown_evidence")), Json.bytes(server.cleanup()));
+                    } catch (java.io.IOException failure) { throw new java.io.UncheckedIOException(failure); }
+                }
+            }, "arena-shutdown"));
+            System.out.println(Json.message("SERVER_READY").put("port", server.port()));
+            new CountDownLatch(1).await();
+            return;
+        }
+        throw new IllegalArgumentException("scenario-run <scenario> <evidence> | server <config>");
+    }
+}
diff --git a/src/main/java/arena/ArenaServer.java b/src/main/java/arena/ArenaServer.java
new file mode 100644
index 0000000..b354770
--- /dev/null
+++ b/src/main/java/arena/ArenaServer.java
@@ -0,0 +1,359 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.bootstrap.ServerBootstrap;
+import io.netty.channel.Channel;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.ChannelInboundHandlerAdapter;
+import io.netty.channel.ChannelInitializer;
+import io.netty.channel.ChannelOption;
+import io.netty.channel.DefaultSelectStrategyFactory;
+import io.netty.channel.FixedRecvByteBufAllocator;
+import io.netty.channel.nio.NioEventLoopGroup;
+import io.netty.channel.socket.SocketChannel;
+import io.netty.channel.socket.nio.NioServerSocketChannel;
+import io.netty.util.concurrent.DefaultEventExecutorChooserFactory;
+import io.netty.util.concurrent.RejectedExecutionHandlers;
+import io.netty.util.concurrent.ThreadPerTaskExecutor;
+import java.net.InetSocketAddress;
+import java.nio.channels.spi.SelectorProvider;
+import java.util.ArrayList;
+import java.util.HashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Set;
+import java.util.UUID;
+import java.util.concurrent.ArrayBlockingQueue;
+import java.util.concurrent.Callable;
+import java.util.concurrent.ConcurrentHashMap;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.Future;
+import java.util.concurrent.RejectedExecutionException;
+import java.util.concurrent.ThreadFactory;
+import java.util.concurrent.ThreadPoolExecutor;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.TimeoutException;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.concurrent.locks.LockSupport;
+
+/** A single simulation owner, with Netty used only for nonblocking transport. */
+public final class ArenaServer implements AutoCloseable {
+    static final int MAILBOX_LIMIT = 1_024;
+    static final int EVENT_LOOP_QUEUE_LIMIT = 1_024;
+    static final int CONNECTION_LIMIT = 8;
+    static final int OUTBOUND_LIMIT = 64;
+    private static final AtomicInteger SERVER_IDS = new AtomicInteger();
+    private final String threadPrefix = "arena-" + SERVER_IDS.incrementAndGet() + "-";
+    private final Set<Peer> peers = ConcurrentHashMap.newKeySet();
+    private final Set<Thread> ownedThreads = ConcurrentHashMap.newKeySet();
+    private final AtomicInteger connections = new AtomicInteger();
+    private final AtomicInteger pendingWrites = new AtomicInteger();
+    private final AtomicInteger outboundHighWater = new AtomicInteger();
+    private final AtomicInteger mailboxHighWater = new AtomicInteger();
+    private final AtomicBoolean closing = new AtomicBoolean();
+    private final ThreadPoolExecutor owner;
+    private final NioEventLoopGroup acceptLoop;
+    private final NioEventLoopGroup ioLoop;
+    private final boolean manual;
+    private final Channel listener;
+    private final Thread ticker;
+    // The following fields, including the session registry, are exclusively room-owner state.
+    private final Map<Peer, Session> sessions = new HashMap<>();
+    private Room room;
+    private long manualNanos;
+    private volatile List<String> closedLifecycle = List.of();
+    private volatile int closedInputHighWater;
+
+    private static final class Session {
+        final String id = "s-" + UUID.randomUUID();
+        String playerId;
+    }
+
+    private final class Peer {
+        final Channel channel;
+        final AtomicInteger outbound = new AtomicInteger();
+        final AtomicBoolean registered = new AtomicBoolean();
+
+        Peer(Channel channel) { this.channel = channel; }
+
+        void send(ObjectNode message) {
+            send(message, false);
+        }
+
+        void send(ObjectNode message, boolean terminal) {
+            if (!channel.isActive()) return;
+            int count = outbound.incrementAndGet();
+            if (count > OUTBOUND_LIMIT) {
+                outbound.decrementAndGet();
+                channel.close();
+                return;
+            }
+            outboundHighWater.accumulateAndGet(count, Math::max);
+            if (count == OUTBOUND_LIMIT) {
+                message = CompleteFrame.error("CONTROL_BACKPRESSURE", "control message bound reached");
+                terminal = true;
+            }
+            var buffer = CompleteFrame.encode(message);
+            pendingWrites.incrementAndGet();
+            boolean closeAfterWrite = terminal;
+            // Netty takes ownership even when the asynchronous write fails.
+            channel.writeAndFlush(buffer).addListener(result -> {
+                pendingWrites.decrementAndGet();
+                outbound.decrementAndGet();
+                if (!result.isSuccess() || closeAfterWrite) channel.close();
+            });
+        }
+        void error(String code) { send(CompleteFrame.error(code, code)); }
+    }
+
+    public ArenaServer(String host, int port, boolean manual) {
+        this.manual = manual;
+        owner = new ThreadPoolExecutor(1, 1, 0, TimeUnit.MILLISECONDS,
+                new ArrayBlockingQueue<>(MAILBOX_LIMIT), namedFactory("room"), new ThreadPoolExecutor.AbortPolicy());
+        acceptLoop = loop("accept");
+        ioLoop = loop("io");
+        Channel bound = null;
+        try {
+            FixedRecvByteBufAllocator receive = new FixedRecvByteBufAllocator(CompleteFrame.MAX_BYTES + 4);
+            receive.maxMessagesPerRead(1);
+            bound = new ServerBootstrap().group(acceptLoop, ioLoop).channel(NioServerSocketChannel.class)
+                    .option(ChannelOption.SO_BACKLOG, CONNECTION_LIMIT)
+                    .option(ChannelOption.SO_REUSEADDR, true)
+                    .childOption(ChannelOption.TCP_NODELAY, true)
+                    .childOption(ChannelOption.RCVBUF_ALLOCATOR, receive)
+                    .childHandler(new ChannelInitializer<SocketChannel>() {
+                        @Override protected void initChannel(SocketChannel channel) {
+                            Peer peer = new Peer(channel);
+                            channel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
+                                // Per-channel lifetime handler, intentionally not sharable.
+                                @Override public void channelActive(ChannelHandlerContext context) {
+                                    if (closing.get()) { context.close(); return; }
+                                    if (connections.incrementAndGet() > CONNECTION_LIMIT) {
+                                        connections.decrementAndGet();
+                                        context.close();
+                                        return;
+                                    }
+                                    peer.registered.set(true);
+                                    peers.add(peer);
+                                    context.fireChannelActive();
+                                }
+                                @Override public void channelInactive(ChannelHandlerContext context) {
+                                    if (peer.registered.compareAndSet(true, false)) {
+                                        connections.decrementAndGet();
+                                        peers.remove(peer);
+                                        enqueue(peer, () -> disconnect(peer));
+                                    }
+                                    context.fireChannelInactive();
+                                }
+                            });
+                            channel.pipeline().addLast(new CompleteFrame(message -> enqueue(peer, () -> handle(peer, message))));
+                        }
+                    }).bind(host, port).syncUninterruptibly().channel();
+        } catch (RuntimeException failure) {
+            if (bound != null) bound.close().syncUninterruptibly();
+            owner.shutdownNow();
+            acceptLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
+            ioLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
+            throw failure;
+        }
+        listener = bound;
+        if (manual) ticker = null;
+        else {
+            ticker = namedFactory("clock").newThread(() -> {
+                // Baseline timer has exactly one outstanding wait; no delayed-task queue.
+                // Accumulator, drift and catch-up behavior are deliberately deferred to G04.
+                while (!closing.get()) {
+                    LockSupport.parkNanos(TimeUnit.MILLISECONDS.toNanos(50));
+                    if (closing.get()) break;
+                    try { execute(this::tick); }
+                    catch (RejectedExecutionException overload) { break; }
+                }
+            });
+            ticker.start();
+        }
+    }
+
+    private ThreadFactory namedFactory(String role) {
+        return runnable -> {
+            Thread thread = new Thread(runnable, threadPrefix + role);
+            ownedThreads.add(thread);
+            return thread;
+        };
+    }
+
+    private NioEventLoopGroup loop(String role) {
+        return new NioEventLoopGroup(1, new ThreadPerTaskExecutor(namedFactory(role)),
+                DefaultEventExecutorChooserFactory.INSTANCE, SelectorProvider.provider(),
+                DefaultSelectStrategyFactory.INSTANCE, RejectedExecutionHandlers.reject(),
+                ignored -> new ArrayBlockingQueue<>(EVENT_LOOP_QUEUE_LIMIT),
+                ignored -> new ArrayBlockingQueue<>(EVENT_LOOP_QUEUE_LIMIT));
+    }
+
+    public int port() { return ((InetSocketAddress) listener.localAddress()).getPort(); }
+
+    private void execute(Runnable runnable) {
+        owner.execute(runnable);
+        mailboxHighWater.accumulateAndGet(owner.getQueue().size(), Math::max);
+    }
+
+    private void enqueue(Peer peer, Runnable command) {
+        try { execute(command); }
+        catch (RejectedExecutionException full) {
+            peer.send(CompleteFrame.error("INPUT_QUEUE_FULL", "room mailbox bound reached"), true);
+        }
+    }
+
+    private void handle(Peer peer, ObjectNode message) {
+        if (closing.get() || !peer.channel.isActive()) return;
+        try {
+            if (message.path("v").asInt(-1) != 1) { peer.error("PROTOCOL_VERSION_UNSUPPORTED"); return; }
+            String type = Json.text(message, "type");
+            if (type.equals("HELLO")) {
+                Session session = sessions.computeIfAbsent(peer, ignored -> new Session());
+                peer.send(Json.message("WELCOME").put("session_id", session.id));
+                return;
+            }
+            Session session = sessions.get(peer);
+            if (session == null || !session.id.equals(Json.text(message, "session_id"))) {
+                peer.error("SESSION_INVALID"); return;
+            }
+            switch (type) {
+                case "CREATE_ROOM" -> {
+                    if (room != null) { peer.error("ROOM_NOT_JOINABLE"); break; }
+                    room = new Room("r-" + UUID.randomUUID());
+                    peer.send(Json.message("ROOM_CREATED").put("room_id", room.id).put("status", "LOBBY"));
+                }
+                case "JOIN_ROOM" -> {
+                    if (!roomMatches(peer, message)) break;
+                    if (session.playerId != null || room.status() != Room.Status.LOBBY) {
+                        peer.error("ROOM_NOT_JOINABLE"); break;
+                    }
+                    session.playerId = "p-" + UUID.randomUUID();
+                    Room.Player player = room.join(session.playerId);
+                    peer.send(Json.message("ROOM_JOINED").put("room_id", room.id).put("player_id", player.id)
+                            .put("slot", player.slot).put("status", room.status().name()));
+                    if (room.status() == Room.Status.RUNNING) broadcast(room.view("SNAPSHOT"));
+                }
+                case "INPUT" -> {
+                    if (!roomMatches(peer, message)) break;
+                    if (session.playerId == null || !session.playerId.equals(Json.text(message, "player_id"))) {
+                        peer.error("PLAYER_MISMATCH"); break;
+                    }
+                    Room.Direction direction = Room.Direction.valueOf(Json.text(message, "direction"));
+                    String target = message.path("tag_target_player_id").isNull() ? null : Json.text(message, "tag_target_player_id");
+                    String code = room.accept(session.playerId, new Room.Intent(direction, target));
+                    if (code != null) peer.error(code);
+                    else peer.send(Json.message("INPUT_ACK").put("status", "ACCEPTED"));
+                }
+                case "LEAVE_ROOM" -> {
+                    if (!roomMatches(peer, message)) break;
+                    if (session.playerId != null) room.left(session.playerId);
+                    peer.channel.close();
+                }
+                default -> peer.error("MESSAGE_TYPE_UNKNOWN");
+            }
+        } catch (IllegalArgumentException invalid) { peer.error("MESSAGE_INVALID"); }
+    }
+
+    private boolean roomMatches(Peer peer, ObjectNode message) {
+        if (room == null || !room.id.equals(Json.text(message, "room_id"))) {
+            peer.error("ROOM_NOT_FOUND"); return false;
+        }
+        return true;
+    }
+
+    private void disconnect(Peer peer) {
+        Session session = sessions.remove(peer);
+        if (session != null && session.playerId != null && room != null) room.left(session.playerId);
+    }
+
+    private void broadcast(ObjectNode message) {
+        for (var entry : sessions.entrySet()) if (entry.getValue().playerId != null) entry.getKey().send(message);
+    }
+
+    private void tick() {
+        if (room == null || room.status() != Room.Status.RUNNING || closing.get()) return;
+        for (Room.Rejection rejection : room.tick()) {
+            sessions.forEach((peer, session) -> {
+                if (rejection.playerId().equals(session.playerId)) peer.error(rejection.code());
+            });
+        }
+        if (room.status() == Room.Status.FINISHED) {
+            broadcast(room.view("SNAPSHOT"));
+            broadcast(room.view("ROOM_FINISHED"));
+        }
+    }
+
+    /** Manual clock: each requested step represents exactly 50ms; it never reads wall time. */
+    public void advanceTicks(int ticks) {
+        if (!manual || ticks < 0 || ticks > Room.DURATION) throw new IllegalArgumentException("manual tick count");
+        call(() -> {
+            for (int i = 0; i < ticks; i++) { manualNanos += 50_000_000L; tick(); }
+            return null;
+        });
+    }
+
+    private <T> T call(Callable<T> action) {
+        Future<T> future = owner.submit(action);
+        try { return future.get(5, TimeUnit.SECONDS); }
+        catch (InterruptedException interrupted) { Thread.currentThread().interrupt(); throw new IllegalStateException(interrupted); }
+        catch (ExecutionException | TimeoutException failure) { throw new IllegalStateException(failure); }
+    }
+
+    ObjectNode metrics() {
+        if (closing.get()) return cleanup();
+        return call(() -> Json.MAPPER.createObjectNode().put("manual_time_ns", manualNanos)
+                .put("pending_input_high_water", room == null ? 0 : room.inputHighWater())
+                .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get()));
+    }
+
+    public ObjectNode cleanup() {
+        long liveThreads = ownedThreads.stream().filter(Thread::isAlive).count();
+        ObjectNode result = Json.MAPPER.createObjectNode().put("open_channels", peers.size() + (listener.isOpen() ? 1 : 0))
+                .put("connections", connections.get()).put("pending_writes", pendingWrites.get())
+                .put("mailbox_remaining", owner.getQueue().size()).put("live_threads", liveThreads)
+                .put("timer_alive", ticker != null && ticker.isAlive()).put("owner_terminated", owner.isTerminated())
+                .put("event_loops_terminated", acceptLoop.isTerminated() && ioLoop.isTerminated())
+                .put("pending_input_high_water", closedInputHighWater)
+                .put("mailbox_high_water", mailboxHighWater.get()).put("outbound_high_water", outboundHighWater.get());
+        var lifecycle = result.putArray("room_lifecycle");
+        closedLifecycle.forEach(lifecycle::add);
+        return result;
+    }
+
+    @Override public void close() {
+        if (!closing.compareAndSet(false, true)) return;
+        if (ticker != null) {
+            ticker.interrupt();
+            try { ticker.join(5_000); }
+            catch (InterruptedException interrupted) { Thread.currentThread().interrupt(); throw new IllegalStateException(interrupted); }
+            if (ticker.isAlive()) throw new IllegalStateException("clock failed to stop");
+        }
+        listener.close().syncUninterruptibly();
+        List<Peer> closingPeers = new ArrayList<>(peers);
+        for (Peer peer : closingPeers) peer.channel.close().syncUninterruptibly();
+        // Drain channel callbacks before final owner cleanup. No network thread waits for this owner.
+        ioLoop.submit(() -> { }).syncUninterruptibly();
+        call(() -> {
+            sessions.clear();
+            if (room != null) {
+                room.close();
+                closedLifecycle = room.lifecycle();
+                closedInputHighWater = room.inputHighWater();
+            }
+            return null;
+        });
+        owner.shutdown();
+        try {
+            if (!owner.awaitTermination(5, TimeUnit.SECONDS)) throw new IllegalStateException("room owner failed to stop");
+        } catch (InterruptedException interrupted) { Thread.currentThread().interrupt(); throw new IllegalStateException(interrupted); }
+        acceptLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
+        ioLoop.shutdownGracefully(0, 5, TimeUnit.SECONDS).syncUninterruptibly();
+        for (Thread thread : ownedThreads) {
+            try { thread.join(5_000); }
+            catch (InterruptedException interrupted) { Thread.currentThread().interrupt(); throw new IllegalStateException(interrupted); }
+            if (thread.isAlive()) throw new IllegalStateException("owned thread failed to exit: " + thread.getName());
+        }
+    }
+}
diff --git a/src/main/java/arena/CompleteFrame.java b/src/main/java/arena/CompleteFrame.java
new file mode 100644
index 0000000..c6ce052
--- /dev/null
+++ b/src/main/java/arena/CompleteFrame.java
@@ -0,0 +1,50 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.buffer.Unpooled;
+import io.netty.channel.ChannelHandlerContext;
+import io.netty.channel.SimpleChannelInboundHandler;
+import java.io.IOException;
+import java.util.function.Consumer;
+
+/** G01 deliberately assumes one complete frame per read; no cumulation or incremental parser. */
+final class CompleteFrame extends SimpleChannelInboundHandler<ByteBuf> {
+    static final int MAX_BYTES = 16_384;
+    private final Consumer<ObjectNode> receiver;
+
+    // One instance per channel, not @Sharable. Inbound ByteBuf is always auto-released.
+    CompleteFrame(Consumer<ObjectNode> receiver) { this.receiver = receiver; }
+
+    @Override protected void channelRead0(ChannelHandlerContext context, ByteBuf buffer) {
+        if (buffer.readableBytes() < 4) {
+            context.close();
+            return;
+        }
+        long length = buffer.readUnsignedInt();
+        if (length == 0 || length > MAX_BYTES || length != buffer.readableBytes()) {
+            context.writeAndFlush(encode(error("FRAME_SIZE_INVALID", "G01 requires one complete frame")))
+                    .addListener(future -> context.close());
+            return;
+        }
+        byte[] bytes = new byte[(int) length];
+        buffer.readBytes(bytes);
+        try { receiver.accept(Json.read(bytes)); }
+        catch (IOException | IllegalArgumentException invalid) {
+            context.writeAndFlush(encode(error("MESSAGE_INVALID", "JSON object required")));
+        }
+    }
+
+    @Override public void exceptionCaught(ChannelHandlerContext context, Throwable cause) { context.close(); }
+
+    static ObjectNode error(String code, String message) {
+        return Json.message("ERROR").put("code", code).put("message", message);
+    }
+
+    // Ownership transfers to channel.writeAndFlush; caller must release if it never writes.
+    static ByteBuf encode(ObjectNode value) {
+        byte[] payload = Json.bytes(value);
+        if (payload.length == 0 || payload.length > MAX_BYTES) throw new IllegalArgumentException("frame too large");
+        return Unpooled.buffer(4 + payload.length, 4 + payload.length).writeInt(payload.length).writeBytes(payload);
+    }
+}
diff --git a/src/main/java/arena/Json.java b/src/main/java/arena/Json.java
new file mode 100644
index 0000000..3870d71
--- /dev/null
+++ b/src/main/java/arena/Json.java
@@ -0,0 +1,33 @@
+package arena;
+
+import com.fasterxml.jackson.core.JsonProcessingException;
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.ObjectMapper;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+
+final class Json {
+    static final ObjectMapper MAPPER = new ObjectMapper();
+    private Json() { }
+
+    static ObjectNode message(String type) {
+        return MAPPER.createObjectNode().put("v", 1).put("type", type);
+    }
+
+    static ObjectNode read(byte[] bytes) throws IOException {
+        JsonNode node = MAPPER.readTree(bytes);
+        if (!(node instanceof ObjectNode object)) throw new IOException("object required");
+        return object;
+    }
+
+    static byte[] bytes(JsonNode value) {
+        try { return MAPPER.writeValueAsBytes(value); }
+        catch (JsonProcessingException error) { throw new IllegalArgumentException(error); }
+    }
+
+    static String text(ObjectNode object, String field) {
+        JsonNode value = object.get(field);
+        if (value == null || !value.isTextual()) throw new IllegalArgumentException(field + " must be text");
+        return value.textValue();
+    }
+}
diff --git a/src/main/java/arena/Room.java b/src/main/java/arena/Room.java
new file mode 100644
index 0000000..925d2a1
--- /dev/null
+++ b/src/main/java/arena/Room.java
@@ -0,0 +1,152 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ArrayNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.util.ArrayDeque;
+import java.util.ArrayList;
+import java.util.HashSet;
+import java.util.List;
+import java.util.TreeMap;
+
+/** All state is confined to the constructing room-owner thread. */
+final class Room {
+    static final int DURATION = 1_200;
+    static final int INPUT_LIMIT = 64;
+    static final int[][] SPAWNS = {
+        {10_000, 10_000}, {90_000, 90_000}, {10_000, 90_000}, {90_000, 10_000},
+        {50_000, 10_000}, {50_000, 90_000}, {10_000, 50_000}, {90_000, 50_000}
+    };
+    enum Direction { STOP, NORTH, EAST, SOUTH, WEST }
+    enum Status { LOBBY, RUNNING, FINISHED, CLOSED }
+    record Intent(Direction direction, String target) { }
+    record Rejection(String playerId, String code) { }
+
+    static final class Player {
+        final String id;
+        final int slot;
+        final ArrayDeque<Intent> pending = new ArrayDeque<>();
+        int x;
+        int y;
+        int score;
+        int lastTagTick = -20;
+        Direction direction = Direction.STOP;
+        boolean connected = true;
+
+        Player(String id, int slot) {
+            this.id = id; this.slot = slot;
+            x = SPAWNS[slot][0]; y = SPAWNS[slot][1];
+        }
+
+        ObjectNode view() {
+            return Json.MAPPER.createObjectNode().put("player_id", id).put("slot", slot)
+                    .put("x", x).put("y", y).put("direction", direction.name()).put("score", score)
+                    .put("connectivity", connected ? "CONNECTED" : "LEFT");
+        }
+    }
+
+    final String id;
+    private final Thread owner = Thread.currentThread();
+    private final TreeMap<String, Player> players = new TreeMap<>();
+    private final List<String> lifecycle = new ArrayList<>(List.of("LOBBY"));
+    private Status status = Status.LOBBY;
+    private int executedTicks;
+    private int nextSlot;
+    private int inputHighWater;
+
+    Room(String id) { this.id = id; }
+    void assertOwner() {
+        if (Thread.currentThread() != owner) throw new IllegalStateException("room mutation outside owner");
+    }
+    Status status() { assertOwner(); return status; }
+    int executedTicks() { assertOwner(); return executedTicks; }
+    int inputHighWater() { assertOwner(); return inputHighWater; }
+    List<String> lifecycle() { assertOwner(); return List.copyOf(lifecycle); }
+    Player player(String id) { assertOwner(); return players.get(id); }
+
+    Player join(String playerId) {
+        assertOwner();
+        if (status != Status.LOBBY || nextSlot == SPAWNS.length || players.containsKey(playerId))
+            throw new IllegalStateException("ROOM_NOT_JOINABLE");
+        Player player = new Player(playerId, nextSlot++);
+        players.put(playerId, player);
+        if (players.values().stream().filter(p -> p.connected).count() >= 2) transition(Status.RUNNING);
+        return player;
+    }
+
+    String accept(String id, Intent intent) {
+        assertOwner();
+        if (status != Status.RUNNING) return "ROOM_NOT_RUNNING";
+        Player player = players.get(id);
+        if (player == null || !player.connected) return "PLAYER_MISMATCH";
+        if (player.pending.size() == INPUT_LIMIT) return "INPUT_QUEUE_FULL";
+        player.pending.addLast(intent);
+        inputHighWater = Math.max(inputHighWater, player.pending.size());
+        return null;
+    }
+
+    List<Rejection> tick() {
+        assertOwner();
+        if (status != Status.RUNNING) return List.of();
+        TreeMap<String, String> tags = new TreeMap<>();
+        for (Player player : players.values()) {
+            Intent selected = null;
+            while (!player.pending.isEmpty()) selected = player.pending.removeFirst();
+            if (!player.connected) continue;
+            if (selected != null) {
+                player.direction = selected.direction();
+                if (selected.target() != null) tags.put(player.id, selected.target());
+            }
+            switch (player.direction) {
+                case NORTH -> player.y += 400;
+                case EAST -> player.x += 400;
+                case SOUTH -> player.y -= 400;
+                case WEST -> player.x -= 400;
+                case STOP -> { }
+            }
+            player.x = Math.clamp(player.x, 0, 100_000);
+            player.y = Math.clamp(player.y, 0, 100_000);
+        }
+        HashSet<String> tagged = new HashSet<>();
+        List<Rejection> rejected = new ArrayList<>();
+        for (var tag : tags.entrySet()) {
+            Player actor = players.get(tag.getKey());
+            Player target = players.get(tag.getValue());
+            long dx = target == null ? 0 : (long) actor.x - target.x;
+            long dy = target == null ? 0 : (long) actor.y - target.y;
+            if (target == null || actor == target || !actor.connected || !target.connected
+                    || executedTicks - actor.lastTagTick < 20 || tagged.contains(target.id)
+                    || dx * dx + dy * dy > 2_500L * 2_500L) {
+                rejected.add(new Rejection(actor.id, "ACTION_REJECTED"));
+            } else {
+                actor.score++;
+                actor.lastTagTick = executedTicks;
+                tagged.add(target.id);
+            }
+        }
+        if (++executedTicks == DURATION) transition(Status.FINISHED);
+        return rejected;
+    }
+
+    void left(String id) {
+        assertOwner();
+        Player player = players.get(id);
+        if (player != null) { player.connected = false; player.direction = Direction.STOP; player.pending.clear(); }
+    }
+
+    void close() {
+        assertOwner();
+        for (Player player : players.values()) player.pending.clear();
+        if (status != Status.CLOSED) transition(Status.CLOSED);
+    }
+
+    ObjectNode view(String type) {
+        assertOwner();
+        ObjectNode result = Json.message(type).put("room_id", id).put("status", status.name())
+                .put("tick", executedTicks - 1).put("executed_ticks", executedTicks);
+        ArrayNode array = result.putArray("players");
+        players.values().forEach(player -> array.add(player.view()));
+        return result;
+    }
+
+    private void transition(Status next) { status = next; lifecycle.add(next.name()); }
+}
diff --git a/src/main/java/arena/ScenarioRunner.java b/src/main/java/arena/ScenarioRunner.java
new file mode 100644
index 0000000..163d63e
--- /dev/null
+++ b/src/main/java/arena/ScenarioRunner.java
@@ -0,0 +1,138 @@
+package arena;
+
+import com.fasterxml.jackson.databind.JsonNode;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.IOException;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.ArrayList;
+import java.util.HexFormat;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+
+/** Executes scenario intent against the real production TCP path, with a manual clock. */
+final class ScenarioRunner {
+    private ScenarioRunner() { }
+
+    static ObjectNode run(Path inputPath) throws IOException {
+        if (Files.size(inputPath) > 65_536) throw new IOException("scenario too large");
+        byte[] scenarioBytes = Files.readAllBytes(inputPath);
+        ObjectNode scenario = Json.read(scenarioBytes);
+        if (!scenario.path("thread").asText().equals("G01") || scenario.path("contract_version").asInt() != 1
+                || !scenario.path("clock").path("kind").asText().equals("manual")
+                || scenario.path("clock").path("tick_duration_ms").asInt() != 50
+                || scenario.path("ticks").asInt() != Room.DURATION
+                || !scenario.path("shutdown").asText().equals("graceful-after-finished"))
+            throw new IOException("unsupported G01 scenario contract");
+
+        ObjectNode evidence = Json.MAPPER.createObjectNode().put("scenario_id", scenario.path("scenario_id").asText())
+                .put("contract_version", 1).put("thread", "G01").put("seed", scenario.path("seed").asInt())
+                .put("scenario_sha256", sha256(scenarioBytes));
+        Map<String, TcpClient> clients = new LinkedHashMap<>();
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        ObjectNode finalView = null;
+        try {
+            for (JsonNode role : scenario.withArray("clients")) {
+                String name = role.asText();
+                if (clients.containsKey(name) || clients.size() == ArenaServer.CONNECTION_LIMIT)
+                    throw new IOException("invalid client roles");
+                clients.put(name, new TcpClient(server.port()));
+            }
+            String roomId = null;
+            for (JsonNode step : scenario.withArray("setup")) {
+                TcpClient client = requiredClient(clients, step.path("client").asText());
+                switch (step.path("type").asText()) {
+                    case "HELLO" -> client.hello();
+                    case "CREATE_ROOM" -> roomId = client.createRoom();
+                    case "JOIN_ROOM" -> client.join(roomId);
+                    default -> throw new IOException("unsupported setup step");
+                }
+            }
+            int currentTick = 0;
+            int accepted = 0;
+            for (JsonNode step : scenario.withArray("inputs")) {
+                int before = step.path("before_tick").asInt(-1);
+                if (before < currentTick || before >= Room.DURATION) throw new IOException("unordered scenario input");
+                server.advanceTicks(before - currentTick);
+                currentTick = before;
+                TcpClient actor = requiredClient(clients, step.path("client").asText());
+                String target = step.path("tag_target").isNull() ? null
+                        : requiredClient(clients, step.path("tag_target").asText()).playerId;
+                // INPUT_ACK is emitted only after the owner queues the input. This is the barrier.
+                actor.intent(roomId, step.path("direction").asText(), target);
+                accepted++;
+            }
+            server.advanceTicks(scenario.path("ticks").asInt() - currentTick);
+            for (TcpClient client : clients.values()) {
+                ObjectNode observed = client.until("ROOM_FINISHED");
+                if (!observed.path("status").asText().equals("FINISHED")
+                        || observed.path("executed_ticks").asInt() != Room.DURATION)
+                    throw new IOException("server did not finish exact tick count");
+                if (finalView == null) finalView = observed;
+                else if (!finalView.equals(observed)) throw new IOException("clients disagree on final authority");
+            }
+            if (finalView == null) throw new IOException("no final client observation");
+            evidence.put("executed_ticks", finalView.path("executed_ticks").asInt());
+            evidence.put("last_tick", finalView.path("tick").asInt());
+            evidence.put("status", finalView.path("status").asText());
+            evidence.put("accepted_inputs", accepted);
+            evidence.putArray("rejected_inputs");
+            var results = evidence.putArray("players");
+            for (var entry : clients.entrySet()) {
+                JsonNode found = null;
+                for (JsonNode player : finalView.withArray("players"))
+                    if (player.path("player_id").asText().equals(entry.getValue().playerId)) found = player;
+                if (found == null) throw new IOException("missing role in final authority");
+                ObjectNode normalized = ((ObjectNode) found).deepCopy();
+                normalized.remove("player_id");
+                normalized.put("role", entry.getKey());
+                results.add(normalized);
+            }
+            evidence.set("runtime_metrics", server.metrics());
+            server.close();
+            ObjectNode observedLifecycle = evidence.putObject("client_lifecycle");
+            for (var entry : clients.entrySet()) {
+                entry.getValue().expectClosed();
+                var states = observedLifecycle.putArray(entry.getKey());
+                entry.getValue().lifecycle().forEach(states::add);
+            }
+        } finally {
+            server.close();
+            for (TcpClient client : clients.values()) client.close();
+        }
+        ObjectNode cleanup = server.cleanup();
+        assertCleanup(cleanup);
+        if (clients.values().stream().anyMatch(client -> !client.isClosed())) throw new IOException("client socket leak");
+        evidence.set("cleanup", cleanup);
+        evidence.set("lifecycle", cleanup.path("room_lifecycle").deepCopy());
+        evidence.put("client_sockets_closed", true).put("state_hashes", "NOT_ACTIVATED_G07");
+        return evidence;
+    }
+
+    static void assertCleanup(ObjectNode cleanup) throws IOException {
+        List<String> failures = new ArrayList<>();
+        for (String field : List.of("open_channels", "connections", "pending_writes", "mailbox_remaining", "live_threads"))
+            if (cleanup.path(field).asLong(-1) != 0) failures.add(field);
+        if (cleanup.path("timer_alive").asBoolean(true)) failures.add("timer_alive");
+        if (!cleanup.path("owner_terminated").asBoolean()) failures.add("owner_terminated");
+        if (!cleanup.path("event_loops_terminated").asBoolean()) failures.add("event_loops_terminated");
+        if (cleanup.path("pending_input_high_water").asInt() > Room.INPUT_LIMIT) failures.add("input bound");
+        if (cleanup.path("mailbox_high_water").asInt() > ArenaServer.MAILBOX_LIMIT) failures.add("mailbox bound");
+        if (cleanup.path("outbound_high_water").asInt() > ArenaServer.OUTBOUND_LIMIT) failures.add("outbound bound");
+        if (!failures.isEmpty()) throw new IOException("cleanup failure: " + failures);
+    }
+
+    private static TcpClient requiredClient(Map<String, TcpClient> clients, String role) throws IOException {
+        TcpClient client = clients.get(role);
+        if (client == null) throw new IOException("unknown client role");
+        return client;
+    }
+
+    private static String sha256(byte[] bytes) {
+        try { return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(bytes)); }
+        catch (NoSuchAlgorithmException unavailable) { throw new IllegalStateException(unavailable); }
+    }
+}
diff --git a/src/main/java/arena/TcpClient.java b/src/main/java/arena/TcpClient.java
new file mode 100644
index 0000000..f2d8887
--- /dev/null
+++ b/src/main/java/arena/TcpClient.java
@@ -0,0 +1,98 @@
+package arena;
+
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.DataInputStream;
+import java.io.IOException;
+import java.io.UncheckedIOException;
+import java.net.InetSocketAddress;
+import java.net.Socket;
+import java.nio.ByteBuffer;
+import java.util.ArrayList;
+import java.util.List;
+
+/** Blocking CLI client; never used on a Netty event loop or simulation-owner thread. */
+final class TcpClient implements AutoCloseable {
+    private final Socket socket = new Socket();
+    private final DataInputStream input;
+    private final List<String> lifecycle = new ArrayList<>();
+    String sessionId;
+    String playerId;
+
+    TcpClient(int port) throws IOException {
+        try {
+            socket.connect(new InetSocketAddress("127.0.0.1", port), 5_000);
+            socket.setTcpNoDelay(true);
+            socket.setSoTimeout(5_000);
+            input = new DataInputStream(socket.getInputStream());
+        } catch (IOException failure) { socket.close(); throw failure; }
+    }
+
+    void send(ObjectNode message) throws IOException {
+        byte[] payload = Json.bytes(message);
+        if (payload.length > CompleteFrame.MAX_BYTES) throw new IOException("client frame too large");
+        byte[] frame = ByteBuffer.allocate(4 + payload.length).putInt(payload.length).put(payload).array();
+        socket.getOutputStream().write(frame);
+        socket.getOutputStream().flush();
+    }
+
+    ObjectNode until(String type) throws IOException {
+        for (int messages = 0; messages < 64; messages++) {
+            int length = input.readInt();
+            if (length < 1 || length > CompleteFrame.MAX_BYTES) throw new IOException("bad server length");
+            byte[] payload = new byte[length];
+            input.readFully(payload);
+            ObjectNode message = Json.read(payload);
+            String status = message.path("status").asText();
+            if (List.of("LOBBY", "RUNNING", "FINISHED").contains(status)) observe(status);
+            if (message.path("type").asText().equals("ERROR")) throw new IOException("server error: " + message);
+            if (message.path("type").asText().equals(type)) return message;
+        }
+        throw new IOException("response message ceiling exceeded");
+    }
+
+    void hello() throws IOException {
+        send(Json.message("HELLO"));
+        sessionId = Json.text(until("WELCOME"), "session_id");
+    }
+
+    String createRoom() throws IOException {
+        send(auth("CREATE_ROOM", null));
+        return Json.text(until("ROOM_CREATED"), "room_id");
+    }
+
+    void join(String roomId) throws IOException {
+        send(auth("JOIN_ROOM", roomId));
+        playerId = Json.text(until("ROOM_JOINED"), "player_id");
+    }
+
+    ObjectNode auth(String type, String roomId) {
+        ObjectNode request = Json.message(type);
+        if (sessionId != null) request.put("session_id", sessionId);
+        if (roomId != null) request.put("room_id", roomId);
+        if (playerId != null) request.put("player_id", playerId);
+        return request;
+    }
+
+    void intent(String roomId, String direction, String targetId) throws IOException {
+        ObjectNode request = auth("INPUT", roomId).put("direction", direction);
+        if (targetId == null) request.putNull("tag_target_player_id");
+        else request.put("tag_target_player_id", targetId);
+        send(request);
+        until("INPUT_ACK");
+    }
+
+    void expectClosed() throws IOException {
+        if (input.read() != -1) throw new IOException("expected EOF after graceful server close");
+        observe("CLOSED");
+    }
+
+    List<String> lifecycle() { return List.copyOf(lifecycle); }
+    boolean isClosed() { return socket.isClosed(); }
+    private void observe(String state) {
+        if (lifecycle.isEmpty() || !lifecycle.getLast().equals(state)) lifecycle.add(state);
+    }
+    @Override public void close() {
+        try { socket.close(); }
+        catch (IOException failure) { throw new UncheckedIOException(failure); }
+    }
+}
diff --git a/src/test/java/arena/CompleteFrameTest.java b/src/test/java/arena/CompleteFrameTest.java
new file mode 100644
index 0000000..4348f39
--- /dev/null
+++ b/src/test/java/arena/CompleteFrameTest.java
@@ -0,0 +1,38 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import io.netty.buffer.ByteBuf;
+import io.netty.channel.embedded.EmbeddedChannel;
+import java.util.ArrayList;
+import java.util.List;
+import org.junit.jupiter.api.Test;
+
+final class CompleteFrameTest {
+    @Test void completeFrameIsConsumedAndInboundOwnershipReleased() {
+        List<ObjectNode> received = new ArrayList<>();
+        CompleteFrame handler = new CompleteFrame(received::add);
+        assertFalse(handler.isSharable());
+        EmbeddedChannel channel = new EmbeddedChannel(handler);
+        ByteBuf bytes = CompleteFrame.encode(Json.message("HELLO"));
+        channel.writeInbound(bytes);
+        assertEquals(0, bytes.refCnt());
+        assertEquals("HELLO", received.getFirst().path("type").asText());
+        channel.finishAndReleaseAll();
+        assertFalse(channel.isOpen());
+    }
+
+    @Test void outboundBufferOwnershipEndsWhenChannelIsCleaned() {
+        EmbeddedChannel channel = new EmbeddedChannel();
+        ByteBuf bytes = CompleteFrame.encode(Json.message("WELCOME"));
+        channel.writeAndFlush(bytes);
+        assertEquals(1, bytes.refCnt());
+        channel.finishAndReleaseAll();
+        assertEquals(0, bytes.refCnt());
+    }
+
+    @Test void outputFrameAllocationIsBounded() {
+        assertThrows(IllegalArgumentException.class,
+                () -> CompleteFrame.encode(Json.message("ERROR").put("message", "x".repeat(CompleteFrame.MAX_BYTES))));
+    }
+}
diff --git a/src/test/java/arena/RoomTest.java b/src/test/java/arena/RoomTest.java
new file mode 100644
index 0000000..bd409bb
--- /dev/null
+++ b/src/test/java/arena/RoomTest.java
@@ -0,0 +1,66 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.FutureTask;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+
+final class RoomTest {
+    private Room runningRoom() {
+        Room room = new Room("room-unit");
+        room.join("player-a");
+        room.join("player-b");
+        return room;
+    }
+
+    @Test void serverMovementIsIntegerAndClamped() {
+        Room room = runningRoom();
+        assertNull(room.accept("player-a", new Room.Intent(Room.Direction.EAST, null)));
+        room.tick();
+        assertEquals(10_400, room.player("player-a").x);
+        assertEquals(10_000, room.player("player-a").y);
+        for (int i = 0; i < 300; i++) room.tick();
+        assertEquals(100_000, room.player("player-a").x);
+        assertEquals(0, room.player("player-a").score);
+    }
+
+    @Test void tagUsesWideSquaredDistanceAndIsOneShot() {
+        Room room = runningRoom();
+        room.accept("player-a", new Room.Intent(Room.Direction.STOP, "player-b"));
+        assertEquals("ACTION_REJECTED", room.tick().getFirst().code());
+        assertEquals(0, room.player("player-a").score);
+        // Deterministic unit setup of state owned by this thread, no transport shortcut in scenarios.
+        room.player("player-b").x = 10_000;
+        room.player("player-b").y = 10_000;
+        room.accept("player-a", new Room.Intent(Room.Direction.STOP, "player-b"));
+        assertTrue(room.tick().isEmpty());
+        assertEquals(1, room.player("player-a").score);
+        for (int i = 0; i < 40; i++) room.tick();
+        assertEquals(1, room.player("player-a").score, "TAG must not persist with direction");
+    }
+
+    @Test void inputStorageHasHardBoundAndDrains() {
+        Room room = runningRoom();
+        for (int i = 0; i < Room.INPUT_LIMIT; i++)
+            assertNull(room.accept("player-a", new Room.Intent(Room.Direction.STOP, null)));
+        assertEquals("INPUT_QUEUE_FULL", room.accept("player-a", new Room.Intent(Room.Direction.EAST, null)));
+        assertEquals(64, room.player("player-a").pending.size());
+        room.tick();
+        assertTrue(room.player("player-a").pending.isEmpty());
+        assertEquals(64, room.inputHighWater());
+        assertEquals(10_000, room.player("player-a").x);
+    }
+
+    @Test void foreignThreadCannotMutateRoom() throws Exception {
+        Room room = runningRoom();
+        FutureTask<Void> mutation = new FutureTask<>(() -> { room.tick(); return null; });
+        Thread foreign = new Thread(mutation, "unit-foreign-owner");
+        foreign.start();
+        ExecutionException failure = assertThrows(ExecutionException.class, () -> mutation.get(5, TimeUnit.SECONDS));
+        assertInstanceOf(IllegalStateException.class, failure.getCause());
+        foreign.join(5_000);
+        assertFalse(foreign.isAlive());
+        assertEquals(0, room.executedTicks());
+    }
+}
diff --git a/src/test/java/arena/ServerIntegrationTest.java b/src/test/java/arena/ServerIntegrationTest.java
new file mode 100644
index 0000000..cd9334e
--- /dev/null
+++ b/src/test/java/arena/ServerIntegrationTest.java
@@ -0,0 +1,111 @@
+package arena;
+
+import static org.junit.jupiter.api.Assertions.*;
+import com.fasterxml.jackson.databind.node.ObjectNode;
+import java.io.BufferedReader;
+import java.io.InputStreamReader;
+import java.net.InetSocketAddress;
+import java.net.ServerSocket;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.List;
+import java.util.concurrent.FutureTask;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.io.TempDir;
+
+final class ServerIntegrationTest {
+    @TempDir Path temporary;
+
+    @Test void realTcpClientsObserveAuthoritativeSessionAndCleanup() throws Exception {
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, true);
+        TcpClient alpha = new TcpClient(server.port());
+        TcpClient bravo = new TcpClient(server.port());
+        try {
+            alpha.hello(); bravo.hello();
+            String room = alpha.createRoom();
+            alpha.join(room); bravo.join(room);
+            ObjectNode attempt = alpha.auth("INPUT", room).put("direction", "EAST")
+                    .put("position", 999_999).put("score", 999).putNull("tag_target_player_id");
+            alpha.send(attempt);
+            alpha.until("INPUT_ACK");
+            bravo.intent(room, "WEST", null);
+            server.advanceTicks(1_200);
+            ObjectNode first = alpha.until("ROOM_FINISHED");
+            assertEquals(first, bravo.until("ROOM_FINISHED"));
+            assertEquals(1_199, first.path("tick").asInt());
+            assertEquals(1_200, first.path("executed_ticks").asInt());
+            assertEquals("FINISHED", first.path("status").asText());
+            for (var player : first.withArray("players")) {
+                assertEquals(0, player.path("score").asInt());
+                assertEquals(player.path("slot").asInt() == 0 ? 100_000 : 0, player.path("x").asInt());
+            }
+            server.close();
+            alpha.expectClosed(); bravo.expectClosed();
+            assertEquals(List.of("LOBBY", "RUNNING", "FINISHED", "CLOSED"), alpha.lifecycle());
+            ScenarioRunner.assertCleanup(server.cleanup());
+            assertEquals(List.of("LOBBY", "RUNNING", "FINISHED", "CLOSED"),
+                    Json.MAPPER.convertValue(server.cleanup().path("room_lifecycle"), List.class));
+        } finally {
+            alpha.close(); bravo.close(); server.close();
+        }
+        assertTrue(alpha.isClosed()); assertTrue(bravo.isClosed());
+    }
+
+    @Test void realTimerAndBothEventLoopsTerminateWithoutSleep() throws Exception {
+        ArenaServer server = new ArenaServer("127.0.0.1", 0, false);
+        try (TcpClient client = new TcpClient(server.port())) {
+            client.hello();
+            server.close();
+            client.expectClosed();
+        } finally { server.close(); }
+        ScenarioRunner.assertCleanup(server.cleanup());
+        assertFalse(server.cleanup().path("timer_alive").asBoolean());
+    }
+
+    @Test void standaloneProcessHandlesSigtermAndReleasesListener() throws Exception {
+        Path config = temporary.resolve("server.json");
+        Path cleanup = temporary.resolve("shutdown.json");
+        Files.write(config, Json.bytes(Json.MAPPER.createObjectNode().put("host", "127.0.0.1").put("port", 0)
+                .put("shutdown_evidence", cleanup.toString())));
+        Process child = new ProcessBuilder("./track", "server", config.toString())
+                .redirectError(temporary.resolve("server.stderr").toFile()).start();
+        try (BufferedReader output = new BufferedReader(new InputStreamReader(child.getInputStream(), StandardCharsets.UTF_8))) {
+            FutureTask<String> readyLine = new FutureTask<>(output::readLine);
+            Thread reader = new Thread(readyLine, "g01-child-ready");
+            reader.start();
+            String line;
+            try { line = readyLine.get(5, TimeUnit.SECONDS); }
+            catch (Exception failedReadiness) {
+                child.destroyForcibly();
+                assertTrue(child.waitFor(5, TimeUnit.SECONDS), "readiness failure child cleanup");
+                reader.join(5_000);
+                throw failedReadiness;
+            }
+            reader.join(5_000);
+            assertFalse(reader.isAlive());
+            ObjectNode ready = Json.read(line.getBytes(StandardCharsets.UTF_8));
+            assertEquals("SERVER_READY", ready.path("type").asText());
+            int port = ready.path("port").asInt();
+            try (TcpClient client = new TcpClient(port)) {
+                client.hello();
+                child.destroy(); // POSIX SIGTERM, one fixed graceful-stop case.
+                assertTrue(child.waitFor(5, TimeUnit.SECONDS), "child must exit within lifecycle deadline");
+                assertEquals(143, child.exitValue(), "normal JVM SIGTERM exit");
+                client.expectClosed();
+            }
+            ScenarioRunner.assertCleanup(Json.read(Files.readAllBytes(cleanup)));
+            try (ServerSocket released = new ServerSocket()) {
+                released.setReuseAddress(true);
+                released.bind(new InetSocketAddress("127.0.0.1", port));
+                assertTrue(released.isBound());
+            }
+        } finally {
+            if (child.isAlive()) {
+                child.destroyForcibly();
+                assertTrue(child.waitFor(5, TimeUnit.SECONDS), "failed child cleanup");
+            }
+        }
+    }
+}
diff --git a/track b/track
new file mode 100755
index 0000000..b5039cc
--- /dev/null
+++ b/track
@@ -0,0 +1,17 @@
+#!/bin/sh
+set -eu
+cd "$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
+if [ -z "${JAVA_HOME:-}" ] && [ -d "$HOME/.sdkman/candidates/java/21.0.7-tem" ]; then
+    JAVA_HOME="$HOME/.sdkman/candidates/java/21.0.7-tem"
+    export JAVA_HOME
+fi
+command=${1:-help}
+if [ "$#" -gt 0 ]; then shift; fi
+case "$command" in
+    build) exec ./gradlew --offline --no-daemon clean classes testClasses installDist resolveLockedDependencies "$@" ;;
+    unit-test) exec ./gradlew --offline --no-daemon test "$@" ;;
+    integration-test) exec ./gradlew --offline --no-daemon integrationTest "$@" ;;
+    scenario-run|server) exec ./build/install/arena-java/bin/arena-java "$command" "$@" ;;
+    replay-verify) echo 'replay-verify is not activated until G07' >&2; exit 2 ;;
+    *) echo 'usage: ./track build | unit-test | integration-test | scenario-run <scenario.json> <evidence.json> | replay-verify <replay> <evidence> | server <config.json>' >&2; exit 2 ;;
+esac
