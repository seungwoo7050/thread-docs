## `build: expose deterministic verification targets`

diff --git a/Makefile b/Makefile
index 22eed0b..5592b78 100644
--- a/Makefile
+++ b/Makefile
@@ -1,7 +1,14 @@
 NAME := philo
 
-CC := cc
-CFLAGS := -Wall -Wextra -Werror -pthread -Iinclude
+CC ?= cc
+CPPFLAGS ?=
+CPPFLAGS += -Iinclude
+CFLAGS ?= -Wall -Wextra -Werror -pthread
+LDFLAGS ?=
+LDLIBS ?=
+RM ?= rm -f
+RMDIR ?= rm -rf
+MKDIR_P ?= mkdir -p
 TSAN_CC ?= $(CC)
 TSAN_REQUIRED ?= 0
 
@@ -19,32 +26,66 @@ SRCS := \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
 
-.PHONY: all bonus clean fclean re test test-tsan
+.DEFAULT_GOAL := all
+.DELETE_ON_ERROR:
+
+.PHONY: all help bonus clean fclean re test check test-tsan ci
 
 all: $(NAME)
 
+help:
+	@printf '%s\n' \
+		'Usage: make <target> [VARIABLE=value]' \
+		'' \
+		'Build:' \
+		'  all        Build philo (default)' \
+		'  clean      Remove generated objects' \
+		'  fclean     Remove generated objects and philo' \
+		'  re         Rebuild from a clean state' \
+		'' \
+		'Validation:' \
+		'  test       Run smoke and concurrency regressions' \
+		'  check      Run the portable verification gate' \
+		'  test-tsan  Run TSan; skip when unsupported unless TSAN_REQUIRED=1' \
+		'  ci         Run check and require TSan support' \
+		'' \
+		'Overrides: CC, CPPFLAGS, CFLAGS, LDFLAGS, LDLIBS,' \
+		'           TSAN_CC, TSAN_REQUIRED'
+
 $(NAME): $(OBJS)
-	$(CC) $(CFLAGS) $(OBJS) -o $@
+	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) $(LDLIBS) -o $@
 
 $(OBJ_DIR)/%.o: $(SRC_DIR)/%.c include/philo.h
-	@mkdir -p $(dir $@)
-	$(CC) $(CFLAGS) -c $< -o $@
+	@$(MKDIR_P) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
 
 bonus:
 	@printf 'bonus target is unavailable\n'
 	@exit 1
 
 clean:
-	rm -rf $(OBJ_DIR)
+	$(RMDIR) $(OBJ_DIR)
 
 fclean: clean
-	rm -f $(NAME)
+	$(RM) $(NAME)
 
 re: fclean all
 
 test: all
-	./tests/smoke.sh
-	./tests/concurrency.sh
+	CC="$(CC)" ./tests/smoke.sh
+	CC="$(CC)" ./tests/concurrency.sh
+
+check: test
 
 test-tsan:
-	TSAN_CC="$(TSAN_CC)" TSAN_REQUIRED="$(TSAN_REQUIRED)" ./tests/tsan.sh
+	@status=0; \
+	TSAN_CC="$(TSAN_CC)" TSAN_REQUIRED="$(TSAN_REQUIRED)" \
+		./tests/tsan.sh || status=$$?; \
+	if [ "$$status" -eq 77 ] && [ "$(TSAN_REQUIRED)" = 0 ]; then \
+		exit 0; \
+	fi; \
+	exit "$$status"
+
+ci:
+	$(MAKE) check
+	$(MAKE) test-tsan TSAN_REQUIRED=1
diff --git a/tests/concurrency.sh b/tests/concurrency.sh
index c04bd78..08d0092 100755
--- a/tests/concurrency.sh
+++ b/tests/concurrency.sh
@@ -4,6 +4,12 @@ set -eu
 
 ROOT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
 TMP_DIR=$(mktemp -d)
+TEST_CC=${CC:-cc}
+
+cc()
+{
+	command "$TEST_CC" "$@"
+}
 
 cleanup()
 {
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 3e5f6ff..32a5ef1 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -4,6 +4,12 @@ set -eu
 
 ROOT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
 TMP_DIR=$(mktemp -d)
+TEST_CC=${CC:-cc}
+
+cc()
+{
+	command "$TEST_CC" "$@"
+}
 
 cleanup()
 {


## `ci: add cross-platform C validation`

diff --git a/.github/workflows/c-philo-ci.yml b/.github/workflows/c-philo-ci.yml
new file mode 100644
index 0000000..e1e8300
--- /dev/null
+++ b/.github/workflows/c-philo-ci.yml
@@ -0,0 +1,74 @@
+name: Philo C validation
+
+on:
+  push:
+    branches:
+      - c/philo
+  pull_request:
+    branches:
+      - c/philo
+
+permissions:
+  contents: read
+
+concurrency:
+  group: c-philo-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  linux-regression:
+    name: Ubuntu / ${{ matrix.compiler }}
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - compiler: GCC 14
+            cc: gcc-14
+          - compiler: Clang 18
+            cc: clang-18
+    env:
+      CC: ${{ matrix.cc }}
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
+      - name: Build and run concurrency regressions
+        run: make -j2 check CC="$CC"
+
+  macos-tsan:
+    name: macOS 15 / Apple Clang / required TSan
+    runs-on: macos-15
+    timeout-minutes: 25
+    env:
+      CC: clang
+      TSAN_CC: clang
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
+      - name: Run regressions and require ThreadSanitizer
+        run: make ci CC="$CC" TSAN_CC="$TSAN_CC"
