## `ci: harden cross-platform verification`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index 81776cb..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,51 +0,0 @@
-name: IRC 회귀 검사
-
-on:
-  push:
-  pull_request:
-
-permissions:
-  contents: read
-
-jobs:
-  platform-regression:
-    name: ${{ matrix.os }} 빌드와 회귀
-    runs-on: ${{ matrix.os }}
-    timeout-minutes: 20
-    strategy:
-      fail-fast: false
-      matrix:
-        os:
-          - ubuntu-latest
-          - macos-latest
-
-    steps:
-      - name: 저장소 가져오기
-        uses: actions/checkout@v4
-
-      - name: 경고를 오류로 처리하여 빌드
-        run: make -j2
-
-      - name: 단위 및 네트워크 회귀 실행
-        run: make test
-
-  linux-sanitizers:
-    name: Linux ASan·UBSan
-    runs-on: ubuntu-latest
-    timeout-minutes: 20
-    env:
-      SANITIZER_FLAGS: >-
-        -std=c++17 -Wall -Wextra -Werror -g -O1
-        -fno-omit-frame-pointer -fsanitize=address,undefined
-      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1:abort_on_error=1
-      UBSAN_OPTIONS: print_stacktrace=1:halt_on_error=1
-
-    steps:
-      - name: 저장소 가져오기
-        uses: actions/checkout@v4
-
-      - name: ASan·UBSan 빌드
-        run: make -j2 CXXFLAGS="${SANITIZER_FLAGS}"
-
-      - name: 새니타이저 단위 및 네트워크 회귀 실행
-        run: make test CXXFLAGS="${SANITIZER_FLAGS}"
diff --git a/.github/workflows/cpp-ft-irc-ci.yml b/.github/workflows/cpp-ft-irc-ci.yml
new file mode 100644
index 0000000..0821344
--- /dev/null
+++ b/.github/workflows/cpp-ft-irc-ci.yml
@@ -0,0 +1,87 @@
+name: IRC relay CI
+
+on:
+  push:
+    branches:
+      - cpp/ft_irc
+  pull_request:
+    branches:
+      - cpp/ft_irc
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
+  PYTHONDONTWRITEBYTECODE: "1"
+  TZ: UTC
+
+jobs:
+  portability:
+    name: ${{ matrix.label }}
+    runs-on: ${{ matrix.os }}
+    timeout-minutes: 20
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - label: Ubuntu 24.04 / GCC 14 / epoll
+            os: ubuntu-24.04
+            cxx: g++-14
+          - label: Ubuntu 24.04 / Clang 18 / epoll
+            os: ubuntu-24.04
+            cxx: clang++-18
+          - label: macOS 15 / Apple Clang / kqueue
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
+          python3 --version
+          make --version
+
+      - name: Run IRC functional regression gates
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: make -j2 ci CXX="$CXX_COMMAND"
+
+  sanitizers:
+    name: Ubuntu 24.04 / Clang 18 / ASan and UBSan
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
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
+          python3 --version
+          make --version
+
+      - name: Run combined sanitizer gate
+        run: make -j2 sanitize CXX=clang++-18
