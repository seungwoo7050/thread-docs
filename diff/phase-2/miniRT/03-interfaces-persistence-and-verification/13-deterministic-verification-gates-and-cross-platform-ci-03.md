## `ci: harden cross-platform verification`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index b108d32..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,48 +0,0 @@
-name: C++ 검증
-
-on:
-  push:
-  pull_request:
-
-jobs:
-  release:
-    name: Release · ${{ matrix.os }}
-    runs-on: ${{ matrix.os }}
-    strategy:
-      fail-fast: false
-      matrix:
-        os:
-          - ubuntu-latest
-          - macos-latest
-
-    steps:
-      - uses: actions/checkout@v4
-      - name: 구성
-        run: >
-          cmake -S . -B build
-          -DCMAKE_BUILD_TYPE=Release
-          -DBUILD_TESTING=ON
-      - name: 빌드
-        run: cmake --build build --parallel
-      - name: 회귀 검사
-        run: ctest --test-dir build --output-on-failure
-
-  sanitizers:
-    name: ASan · UBSan
-    runs-on: ubuntu-latest
-
-    steps:
-      - uses: actions/checkout@v4
-      - name: 구성
-        run: >
-          cmake -S . -B build
-          -DCMAKE_BUILD_TYPE=Debug
-          -DBUILD_TESTING=ON
-          -DRAY_ENABLE_SANITIZERS=ON
-      - name: 빌드
-        run: cmake --build build --parallel
-      - name: 메모리·정의되지 않은 동작 검사
-        env:
-          ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
-          UBSAN_OPTIONS: print_stacktrace=1:halt_on_error=1
-        run: ctest --test-dir build --output-on-failure
diff --git a/.github/workflows/cpp-miniRT-ci.yml b/.github/workflows/cpp-miniRT-ci.yml
new file mode 100644
index 0000000..195e9c9
--- /dev/null
+++ b/.github/workflows/cpp-miniRT-ci.yml
@@ -0,0 +1,113 @@
+name: miniRT CI
+
+on:
+  push:
+    branches:
+      - cpp/miniRT
+  pull_request:
+    branches:
+      - cpp/miniRT
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
+    timeout-minutes: 15
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - label: Ubuntu 24.04 / GCC 14
+            artifact_label: ubuntu-gcc
+            os: ubuntu-24.04
+            cxx: g++-14
+          - label: Ubuntu 24.04 / Clang 18
+            artifact_label: ubuntu-clang
+            os: ubuntu-24.04
+            cxx: clang++-18
+          - label: macOS 15 / Apple Clang
+            artifact_label: macos-clang
+            os: macos-15
+            cxx: clang++
+    steps:
+      - name: Check out the project branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v "$CXX_COMMAND"
+          "$CXX_COMMAND" --version
+          cmake --version
+          ctest --version
+          make --version
+
+      - name: Run miniRT functional regression gates
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: make ci CXX="$CXX_COMMAND"
+
+      - name: Upload failed CTest diagnostics
+        if: failure()
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
+        with:
+          name: minirt-${{ matrix.artifact_label }}-ctest-diagnostics
+          path: |
+            build/Testing/Temporary/LastTest.log
+            build/Testing/Temporary/LastTestsFailed.log
+          if-no-files-found: ignore
+          retention-days: 7
+
+  sanitizers:
+    name: Ubuntu 24.04 / Clang 18 / ASan and UBSan
+    runs-on: ubuntu-24.04
+    timeout-minutes: 15
+    env:
+      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1:abort_on_error=1
+      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
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
+          cmake --version
+          ctest --version
+          make --version
+
+      - name: Run combined sanitizer gate
+        run: make sanitize CXX=clang++-18
+
+      - name: Upload failed CTest diagnostics
+        if: failure()
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
+        with:
+          name: minirt-linux-sanitizers-ctest-diagnostics
+          path: |
+            build/sanitize/Testing/Temporary/LastTest.log
+            build/sanitize/Testing/Temporary/LastTestsFailed.log
+          if-no-files-found: ignore
+          retention-days: 7
