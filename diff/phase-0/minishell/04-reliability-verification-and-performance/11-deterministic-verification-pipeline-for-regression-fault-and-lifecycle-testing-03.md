## `build(test): ASan·UBSan 검증 경로 추가`

diff --git a/.gitignore b/.gitignore
index fae90d7..47784b4 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,5 +1,14 @@
 small-shell
 /small-shell-test
+/small-shell-asan
+/small-shell-test-asan
+/small-shell-ubsan
+/small-shell-test-ubsan
+/tests/parser-api-asan
+/tests/parser-api-ubsan
+tests/parser-api-asan.dSYM/
+tests/parser-api-ubsan.dSYM/
+/*.dSYM/
 /tests/timeout-runner
 /tests/parser-api
 *.o
diff --git a/Makefile b/Makefile
index 84dbfd7..a79e410 100644
--- a/Makefile
+++ b/Makefile
@@ -24,6 +24,14 @@ TEST_OBJS := $(SRCS:.c=.test.o)
 TEST_TARGET := small-shell-test
 PARSER_API_TARGET := tests/parser-api
 TIMEOUT_TARGET := tests/timeout-runner
+ASAN_TARGET := small-shell-asan
+ASAN_TEST_TARGET := small-shell-test-asan
+ASAN_PARSER_API_TARGET := tests/parser-api-asan
+UBSAN_TARGET := small-shell-ubsan
+UBSAN_TEST_TARGET := small-shell-test-ubsan
+UBSAN_PARSER_API_TARGET := tests/parser-api-ubsan
+SANITIZER_CFLAGS := $(CFLAGS) -O1 -g -fno-omit-frame-pointer
+SANITIZER_IMAGE ?= gcc:13-bookworm
 
 ifeq ($(USE_READLINE),1)
 CPPFLAGS += -DUSE_READLINE
@@ -51,6 +59,32 @@ $(PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
 $(TIMEOUT_TARGET): tests/timeout_runner.c
 	$(CC) $(CPPFLAGS) $(CFLAGS) -o $@ $<
 
+$(ASAN_TARGET): $(SRCS)
+	$(CC) $(CPPFLAGS) $(SANITIZER_CFLAGS) -fsanitize=address \
+		$(LDFLAGS) -o $@ $(SRCS) $(LDLIBS)
+
+$(ASAN_TEST_TARGET): $(SRCS)
+	$(CC) $(CPPFLAGS) -DSMALL_SHELL_TESTING $(SANITIZER_CFLAGS) \
+		-fsanitize=address $(LDFLAGS) -o $@ $(SRCS) $(LDLIBS)
+
+$(ASAN_PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
+	$(CC) $(CPPFLAGS) $(SANITIZER_CFLAGS) -fsanitize=address \
+		$(LDFLAGS) -o $@ tests/parser_api.c \
+		$(filter-out src/main.c,$(SRCS)) $(LDLIBS)
+
+$(UBSAN_TARGET): $(SRCS)
+	$(CC) $(CPPFLAGS) $(SANITIZER_CFLAGS) -fsanitize=undefined \
+		$(LDFLAGS) -o $@ $(SRCS) $(LDLIBS)
+
+$(UBSAN_TEST_TARGET): $(SRCS)
+	$(CC) $(CPPFLAGS) -DSMALL_SHELL_TESTING $(SANITIZER_CFLAGS) \
+		-fsanitize=undefined $(LDFLAGS) -o $@ $(SRCS) $(LDLIBS)
+
+$(UBSAN_PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
+	$(CC) $(CPPFLAGS) $(SANITIZER_CFLAGS) -fsanitize=undefined \
+		$(LDFLAGS) -o $@ tests/parser_api.c \
+		$(filter-out src/main.c,$(SRCS)) $(LDLIBS)
+
 readline:
 	$(MAKE) USE_READLINE=1
 
@@ -62,8 +96,58 @@ test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./$(PARSER_API_TARGET)
 	./tests/performance.sh
 
+test-asan: $(ASAN_TARGET) $(ASAN_TEST_TARGET) \
+		$(ASAN_PARSER_API_TARGET) $(TIMEOUT_TARGET)
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		SMALL_SHELL_BIN="$(CURDIR)/$(ASAN_TARGET)" ./tests/smoke.sh
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(ASAN_TEST_TARGET)" \
+		./tests/faults.sh
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(ASAN_TEST_TARGET)" \
+		./tests/allocation.sh
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(ASAN_TEST_TARGET)" \
+		./tests/lifecycle.sh
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		./$(ASAN_PARSER_API_TARGET)
+	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
+		SMALL_SHELL_BIN="$(CURDIR)/$(ASAN_TARGET)" \
+		./tests/performance.sh
+
+test-ubsan: $(UBSAN_TARGET) $(UBSAN_TEST_TARGET) \
+		$(UBSAN_PARSER_API_TARGET) $(TIMEOUT_TARGET)
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		SMALL_SHELL_BIN="$(CURDIR)/$(UBSAN_TARGET)" ./tests/smoke.sh
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(UBSAN_TEST_TARGET)" \
+		./tests/faults.sh
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(UBSAN_TEST_TARGET)" \
+		./tests/allocation.sh
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		SMALL_SHELL_TEST_BIN="$(CURDIR)/$(UBSAN_TEST_TARGET)" \
+		./tests/lifecycle.sh
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		./$(UBSAN_PARSER_API_TARGET)
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		SMALL_SHELL_BIN="$(CURDIR)/$(UBSAN_TARGET)" \
+		./tests/performance.sh
+
+test-sanitizers-container:
+	docker run --rm --network none --read-only \
+		--tmpfs /tmp:exec,size=256m -v "$(CURDIR):/source:ro" \
+		$(SANITIZER_IMAGE) \
+		sh /source/tests/container_sanitizers.sh
+
 clean:
 	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
+		$(ASAN_TARGET) $(ASAN_TEST_TARGET) $(ASAN_PARSER_API_TARGET) \
+		$(UBSAN_TARGET) $(UBSAN_TEST_TARGET) $(UBSAN_PARSER_API_TARGET) \
 		$(OBJS) $(TEST_OBJS)
+	rm -rf $(ASAN_TARGET).dSYM $(ASAN_TEST_TARGET).dSYM \
+		$(ASAN_PARSER_API_TARGET).dSYM $(UBSAN_TARGET).dSYM \
+		$(UBSAN_TEST_TARGET).dSYM $(UBSAN_PARSER_API_TARGET).dSYM
 
-.PHONY: all readline test clean
+.PHONY: all readline test test-asan test-ubsan \
+	test-sanitizers-container clean
diff --git a/tests/allocation.sh b/tests/allocation.sh
index ad61087..f4a4134 100755
--- a/tests/allocation.sh
+++ b/tests/allocation.sh
@@ -39,6 +39,8 @@ sweep()
         set +e
         env -i \
             PATH="$PATH" \
+            ASAN_OPTIONS="${ASAN_OPTIONS-}" \
+            UBSAN_OPTIONS="${UBSAN_OPTIONS-}" \
             ALLOC_SWEEP=old \
             HEREDOC_VALUE=expanded \
             SMALL_SHELL_FAIL_ALLOC_SCOPE="$scope" \
@@ -191,6 +193,8 @@ set +e
 printf 'cat <<EOF\nbody\nEOF\necho never\n' >"$TMP/persistent-input.in"
 env -i \
     PATH="$PATH" \
+    ASAN_OPTIONS="${ASAN_OPTIONS-}" \
+    UBSAN_OPTIONS="${UBSAN_OPTIONS-}" \
     SMALL_SHELL_FAIL_ALLOC_SCOPE=input \
     SMALL_SHELL_FAIL_ALLOC=1 \
     SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
@@ -206,6 +210,8 @@ set +e
 printf 'echo hidden\n' >"$TMP/persistent.in"
 env -i \
     PATH="$PATH" \
+    ASAN_OPTIONS="${ASAN_OPTIONS-}" \
+    UBSAN_OPTIONS="${UBSAN_OPTIONS-}" \
     SMALL_SHELL_FAIL_ALLOC_SCOPE=token \
     SMALL_SHELL_FAIL_ALLOC=1 \
     SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
diff --git a/tests/container_sanitizers.sh b/tests/container_sanitizers.sh
new file mode 100644
index 0000000..a0a2f69
--- /dev/null
+++ b/tests/container_sanitizers.sh
@@ -0,0 +1,10 @@
+#!/bin/sh
+set -eu
+
+mkdir /tmp/small-shell
+cp -R /source/. /tmp/small-shell/
+cd /tmp/small-shell
+
+make clean
+make CC=gcc test-asan
+make CC=gcc test-ubsan


## `build: expose deterministic verification targets`

diff --git a/Makefile b/Makefile
index a79e410..9f0c4b1 100644
--- a/Makefile
+++ b/Makefile
@@ -3,6 +3,9 @@ CFLAGS ?= -std=c99 -Wall -Wextra -Wpedantic
 CPPFLAGS ?= -Iinclude
 LDFLAGS ?=
 LDLIBS ?=
+DOCKER ?= docker
+RM ?= rm -f
+RMDIR ?= rm -rf
 
 TARGET := small-shell
 SRCS := \
@@ -33,6 +36,9 @@ UBSAN_PARSER_API_TARGET := tests/parser-api-ubsan
 SANITIZER_CFLAGS := $(CFLAGS) -O1 -g -fno-omit-frame-pointer
 SANITIZER_IMAGE ?= gcc:13-bookworm
 
+.DEFAULT_GOAL := all
+.DELETE_ON_ERROR:
+
 ifeq ($(USE_READLINE),1)
 CPPFLAGS += -DUSE_READLINE
 LDLIBS += -lreadline
@@ -40,6 +46,26 @@ endif
 
 all: $(TARGET)
 
+help:
+	@printf '%s\n' \
+		'Usage: make <target> [VARIABLE=value]' \
+		'' \
+		'Build:' \
+		'  all                        Build small-shell (default)' \
+		'  readline                   Build with GNU Readline support' \
+		'  clean                      Remove generated binaries and objects' \
+		'' \
+		'Validation:' \
+		'  test / check               Run the portable regression suite' \
+		'  test-asan                  Run AddressSanitizer regressions on Linux' \
+		'  test-ubsan                 Run UndefinedBehaviorSanitizer regressions' \
+		'  sanitizers                 Run ASan and UBSan sequentially' \
+		'  test-sanitizers-container  Run sanitizers in the configured image' \
+		'  ci                          Run check and Linux sanitizer gates' \
+		'' \
+		'Overrides: CC, CPPFLAGS, CFLAGS, LDFLAGS, LDLIBS,' \
+		'           DOCKER, SANITIZER_IMAGE, USE_READLINE'
+
 $(TARGET): $(OBJS)
 	$(CC) $(LDFLAGS) -o $@ $(OBJS) $(LDLIBS)
 
@@ -96,6 +122,8 @@ test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./$(PARSER_API_TARGET)
 	./tests/performance.sh
 
+check: test
+
 test-asan: $(ASAN_TARGET) $(ASAN_TEST_TARGET) \
 		$(ASAN_PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
@@ -135,19 +163,27 @@ test-ubsan: $(UBSAN_TARGET) $(UBSAN_TEST_TARGET) \
 		./tests/performance.sh
 
 test-sanitizers-container:
-	docker run --rm --network none --read-only \
+	$(DOCKER) run --rm --network none --read-only \
 		--tmpfs /tmp:exec,size=256m -v "$(CURDIR):/source:ro" \
 		$(SANITIZER_IMAGE) \
 		sh /source/tests/container_sanitizers.sh
 
+sanitizers:
+	$(MAKE) test-asan
+	$(MAKE) test-ubsan
+
+ci:
+	$(MAKE) check
+	$(MAKE) sanitizers
+
 clean:
-	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
+	$(RM) $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
 		$(ASAN_TARGET) $(ASAN_TEST_TARGET) $(ASAN_PARSER_API_TARGET) \
 		$(UBSAN_TARGET) $(UBSAN_TEST_TARGET) $(UBSAN_PARSER_API_TARGET) \
 		$(OBJS) $(TEST_OBJS)
-	rm -rf $(ASAN_TARGET).dSYM $(ASAN_TEST_TARGET).dSYM \
+	$(RMDIR) $(ASAN_TARGET).dSYM $(ASAN_TEST_TARGET).dSYM \
 		$(ASAN_PARSER_API_TARGET).dSYM $(UBSAN_TARGET).dSYM \
 		$(UBSAN_TEST_TARGET).dSYM $(UBSAN_PARSER_API_TARGET).dSYM
 
-.PHONY: all readline test test-asan test-ubsan \
+.PHONY: all help readline test check test-asan test-ubsan sanitizers ci \
 	test-sanitizers-container clean


## `ci: add cross-platform C validation`

diff --git a/.github/workflows/c-minishell-ci.yml b/.github/workflows/c-minishell-ci.yml
new file mode 100644
index 0000000..76fc24d
--- /dev/null
+++ b/.github/workflows/c-minishell-ci.yml
@@ -0,0 +1,109 @@
+name: Minishell C validation
+
+on:
+  push:
+    branches:
+      - c/minishell
+  pull_request:
+    branches:
+      - c/minishell
+
+permissions:
+  contents: read
+
+concurrency:
+  group: c-minishell-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  regression:
+    name: ${{ matrix.os }} / ${{ matrix.compiler }}
+    runs-on: ${{ matrix.os }}
+    timeout-minutes: 15
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - os: ubuntu-24.04
+            compiler: GCC 14
+            cc: gcc-14
+          - os: ubuntu-24.04
+            compiler: Clang 18
+            cc: clang-18
+          - os: macos-15
+            compiler: Apple Clang
+            cc: clang
+    env:
+      CC: ${{ matrix.cc }}
+      CFLAGS: -std=c99 -Wall -Wextra -Wpedantic -Werror
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        shell: bash
+        run: |
+          set -euo pipefail
+          uname -a
+          "$CC" --version
+          make --version
+
+      - name: Build and run shell regressions
+        run: make -j2 check CC="$CC" CFLAGS="$CFLAGS"
+
+  readline:
+    name: Ubuntu / GCC 14 / Readline
+    runs-on: ubuntu-24.04
+    timeout-minutes: 10
+    env:
+      CC: gcc-14
+      CFLAGS: -std=c99 -Wall -Wextra -Wpedantic -Werror
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1
+        with:
+          persist-credentials: false
+
+      - name: Install Readline headers
+        run: |
+          sudo apt-get update
+          sudo apt-get install --yes --no-install-recommends libreadline-dev
+
+      - name: Report toolchain
+        shell: bash
+        run: |
+          set -euo pipefail
+          uname -a
+          "$CC" --version
+          make --version
+
+      - name: Build the Readline variant
+        run: make -j2 readline CC="$CC" CFLAGS="$CFLAGS"
+
+  sanitizers:
+    name: Ubuntu / GCC 14 / ASan and UBSan
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    env:
+      CC: gcc-14
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        shell: bash
+        run: |
+          set -euo pipefail
+          uname -a
+          "$CC" --version
+          make --version
+
+      - name: Run sanitizer regressions
+        run: make sanitizers CC="$CC"
