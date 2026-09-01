## `test: reap timeout watchdog timers`

diff --git a/tests/run_with_timeout.sh b/tests/run_with_timeout.sh
index 3e464c7..d30c5ed 100755
--- a/tests/run_with_timeout.sh
+++ b/tests/run_with_timeout.sh
@@ -54,12 +54,42 @@ started_at=$(date +%s)
 child_pid=$!
 
 (
-    sleep "$limit_seconds"
+    timer_pid=
+
+    cleanup_watchdog()
+    {
+        if [ -n "$timer_pid" ]
+        then
+            kill "$timer_pid" 2>/dev/null || true
+            wait "$timer_pid" 2>/dev/null || true
+        fi
+    }
+
+    trap cleanup_watchdog EXIT
+    trap 'exit 129' HUP
+    trap 'exit 130' INT
+    trap 'exit 143' TERM
+
+    sleep "$limit_seconds" &
+    timer_pid=$!
+    if wait "$timer_pid"
+    then
+        timer_pid=
+    else
+        exit 0
+    fi
     if kill -0 "$child_pid" 2>/dev/null
     then
         : > "$timeout_marker"
         kill -TERM "$child_pid" 2>/dev/null || true
-        sleep 2
+        sleep 2 &
+        timer_pid=$!
+        if wait "$timer_pid"
+        then
+            timer_pid=
+        else
+            exit 0
+        fi
         kill -KILL "$child_pid" 2>/dev/null || true
     fi
 ) &


## `ci: cross-platform verification`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index d1d0a64..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,58 +0,0 @@
-name: C++ build and regression
-
-on:
-  push:
-    branches:
-      - main
-  pull_request:
-
-permissions:
-  contents: read
-
-jobs:
-  verify:
-    name: ${{ matrix.os }} / ${{ matrix.compiler }} / LP64
-    runs-on: ${{ matrix.os }}
-    timeout-minutes: 30
-    strategy:
-      fail-fast: false
-      matrix:
-        include:
-          - os: ubuntu-22.04
-            compiler: gcc
-            command: g++
-            ubsan: true
-            asan: true
-          - os: ubuntu-22.04
-            compiler: clang
-            command: clang++
-            ubsan: true
-            asan: true
-          - os: macos-latest
-            compiler: clang
-            command: clang++
-            ubsan: true
-            asan: false
-
-    steps:
-      - uses: actions/checkout@v4
-
-      - name: Select compiler
-        shell: bash
-        run: |
-          set -euo pipefail
-          cxx_command=${{ matrix.command }}
-          command -v "$cxx_command"
-          "$cxx_command" --version
-          echo "CXX_COMMAND=$cxx_command" >> "$GITHUB_ENV"
-
-      - name: Build, regression, and LP64 checks
-        run: make check-build CXX="$CXX_COMMAND"
-
-      - name: UndefinedBehaviorSanitizer
-        if: matrix.ubsan
-        run: make test-ubsan CXX="$CXX_COMMAND"
-
-      - name: AddressSanitizer
-        if: matrix.asan
-        run: make test-asan CXX="$CXX_COMMAND"
diff --git a/.github/workflows/cpp-foundation-ci.yml b/.github/workflows/cpp-foundation-ci.yml
new file mode 100644
index 0000000..db04f10
--- /dev/null
+++ b/.github/workflows/cpp-foundation-ci.yml
@@ -0,0 +1,151 @@
+name: C++ foundation CI
+
+on:
+  push:
+    branches:
+      - cpp/cpp-foundation
+  pull_request:
+    branches:
+      - cpp/cpp-foundation
+
+permissions:
+  contents: read
+
+concurrency:
+  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
+  cancel-in-progress: true
+
+env:
+  LANG: C
+  LC_ALL: C
+  TZ: UTC
+
+jobs:
+  portability:
+    name: ${{ matrix.label }}
+    runs-on: ${{ matrix.os }}
+    timeout-minutes: 30
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - label: Ubuntu 24.04 / GCC 14
+            os: ubuntu-24.04
+            cxx: g++-14
+            macos: false
+          - label: Ubuntu 24.04 / Clang 18
+            os: ubuntu-24.04
+            cxx: clang++-18
+            macos: false
+          - label: macOS 15 / Apple Clang
+            os: macos-15
+            cxx: clang++
+            macos: true
+    steps:
+      - name: Check out the project branch
+        if: ${{ !matrix.macos }}
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      # runner 2.336.0 can stall in checkout's macOS post action. Materialize
+      # the exact event SHA without persisting credentials or a post hook.
+      - name: Materialize the exact project revision
+        if: ${{ matrix.macos }}
+        env:
+          GH_TOKEN: ${{ github.token }}
+          SOURCE_URL: ${{ github.api_url }}/repos/${{ github.repository }}/tarball/${{ github.sha }}
+        run: |
+          set -euo pipefail
+          archive="$RUNNER_TEMP/cpp-foundation-${GITHUB_SHA}.tar.gz"
+          curl --fail --silent --show-error --location --retry 3 \
+            --header "Accept: application/vnd.github+json" \
+            --header "Authorization: Bearer ${GH_TOKEN}" \
+            --header "X-GitHub-Api-Version: 2022-11-28" \
+            --output "$archive" "$SOURCE_URL"
+          tar -xzf "$archive" --strip-components=1 -C "$GITHUB_WORKSPACE"
+
+      - name: Report toolchain
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v "$CXX_COMMAND"
+          "$CXX_COMMAND" --version
+          make --version
+
+      - name: Build and run functional regression gates
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: |
+          set -euo pipefail
+          make fclean CXX="$CXX_COMMAND"
+          make all CXX="$CXX_COMMAND"
+          make test CXX="$CXX_COMMAND"
+          make check-determinism CXX="$CXX_COMMAND"
+          make check-data-model CXX="$CXX_COMMAND"
+          make -q all CXX="$CXX_COMMAND"
+
+  macos-platform:
+    name: macOS 15 / Apple Clang / platform contracts
+    runs-on: macos-15
+    timeout-minutes: 15
+    steps:
+      - name: Materialize the exact project revision
+        env:
+          GH_TOKEN: ${{ github.token }}
+          SOURCE_URL: ${{ github.api_url }}/repos/${{ github.repository }}/tarball/${{ github.sha }}
+        run: |
+          set -euo pipefail
+          archive="$RUNNER_TEMP/cpp-foundation-${GITHUB_SHA}.tar.gz"
+          curl --fail --silent --show-error --location --retry 3 \
+            --header "Accept: application/vnd.github+json" \
+            --header "Authorization: Bearer ${GH_TOKEN}" \
+            --header "X-GitHub-Api-Version: 2022-11-28" \
+            --output "$archive" "$SOURCE_URL"
+          tar -xzf "$archive" --strip-components=1 -C "$GITHUB_WORKSPACE"
+
+      - name: Report toolchain
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v clang++
+          clang++ --version
+          make --version
+
+      - name: Check macOS release contracts
+        env:
+          # `leaks --atExit` can leave a short-lived Apple helper that races
+          # runner 2.336.0 process cleanup. The foreground gate still must pass.
+          RUNNER_TRACKING_ID: ""
+        run: make check-platform CXX=clang++
+
+  sanitizers:
+    name: Ubuntu 24.04 / Clang 18 / ${{ matrix.label }}
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - label: AddressSanitizer
+            target: test-asan
+          - label: UndefinedBehaviorSanitizer
+            target: test-ubsan
+    steps:
+      - name: Check out the project branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v clang++-18
+          clang++-18 --version
+          make --version
+
+      - name: Run ${{ matrix.label }}
+        run: make "${{ matrix.target }}" CXX=clang++-18
