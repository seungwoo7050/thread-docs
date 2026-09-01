# 이식 가능한 정적 라이브러리 검증

## `build(flags): C99 경고와 builtin 정책을 고정`

diff --git a/Makefile b/Makefile
index fba7298..85b5697 100644
--- a/Makefile
+++ b/Makefile
@@ -1,8 +1,9 @@
 NAME := libft.a
 
 CC := cc
-CFLAGS := -Wall -Wextra -Werror -std=c99 -pedantic
-CPPFLAGS := -I.
+override CFLAGS := -Wall -Wextra -Werror -Wpedantic -std=c99 \
+	-fno-builtin
+override CPPFLAGS := -I.
 DEPFLAGS := -MMD -MP
 AR := ar
 ARFLAGS := rcs
@@ -28,9 +29,11 @@ SRC := \
 	src/list/ft_list_basic.c \
 	src/list/ft_list_lifecycle.c \
 	src/list/ft_list_map.c
+
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
+
 TEST_BIN := tests/bin/test_libft
 TEST_SRC := $(wildcard tests/test_*.c)
 
@@ -42,20 +45,24 @@ WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
 	tests/support/fail_write.c
 WRITE_DEFINES := -Dwrite=test_write
 
-.PHONY: all clean fclean re test write-failure-test
+.PHONY: all bonus clean fclean re test write-failure-test
 
 all: $(NAME)
 
+bonus: all
+
 $(NAME): $(OBJ)
 	$(AR) $(ARFLAGS) $@ $(OBJ)
 
 $(OBJ_DIR)/%.o: %.c libft.h
 	@$(MKDIR) $(dir $@)
-	$(CC) $(CPPFLAGS) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(DEPFLAGS) -c $< -o $@
 
 $(TEST_BIN): $(NAME) $(TEST_SRC) tests/test.h
 	@$(MKDIR) $(dir $@)
-	$(CC) $(CPPFLAGS) $(CFLAGS) $(TEST_SRC) $(NAME) -o $@
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(TEST_SRC) $(NAME) -o $@
 
 test: $(TEST_BIN)
 	./$(TEST_BIN)


## `test(release): archive와 consumer 경계를 검증`

diff --git a/Makefile b/Makefile
index 0728d56..eb42d09 100644
--- a/Makefile
+++ b/Makefile
@@ -52,7 +52,8 @@ WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
 	tests/support/fail_write.c
 WRITE_DEFINES := -Dwrite=test_write
 
-.PHONY: all bonus clean fclean re test failure-test write-failure-test
+.PHONY: all bonus clean fclean re test failure-test write-failure-test \
+	check-archive
 
 all: $(NAME)
 
@@ -101,6 +102,9 @@ $(WRITE_BIN): $(NAME) $(WRITE_OUTPUT_OBJ) $(WRITE_TEST_SRC) \
 write-failure-test: $(WRITE_BIN)
 	./$(WRITE_BIN)
 
+check-archive: $(NAME)
+	CC="$(CC)" sh tests/check_archive.sh $(NAME)
+
 clean:
 	$(RMDIR) build tests/bin
 
diff --git a/tests/allowed-undefined.txt b/tests/allowed-undefined.txt
new file mode 100644
index 0000000..df1ca04
--- /dev/null
+++ b/tests/allowed-undefined.txt
@@ -0,0 +1,3 @@
+free
+malloc
+write
diff --git a/tests/api-symbols.txt b/tests/api-symbols.txt
new file mode 100644
index 0000000..9929de0
--- /dev/null
+++ b/tests/api-symbols.txt
@@ -0,0 +1,43 @@
+ft_atoi
+ft_bzero
+ft_calloc
+ft_isalnum
+ft_isalpha
+ft_isascii
+ft_isdigit
+ft_isprint
+ft_itoa
+ft_lstadd_back
+ft_lstadd_front
+ft_lstclear
+ft_lstdelone
+ft_lstiter
+ft_lstlast
+ft_lstmap
+ft_lstnew
+ft_lstsize
+ft_memchr
+ft_memcmp
+ft_memcpy
+ft_memmove
+ft_memset
+ft_putchar_fd
+ft_putendl_fd
+ft_putnbr_fd
+ft_putstr_fd
+ft_split
+ft_strchr
+ft_strdup
+ft_striteri
+ft_strjoin
+ft_strlcat
+ft_strlcpy
+ft_strlen
+ft_strmapi
+ft_strncmp
+ft_strnstr
+ft_strrchr
+ft_strtrim
+ft_substr
+ft_tolower
+ft_toupper
diff --git a/tests/archive-members.txt b/tests/archive-members.txt
new file mode 100644
index 0000000..7347f5d
--- /dev/null
+++ b/tests/archive-members.txt
@@ -0,0 +1,17 @@
+ft_allocate.o
+ft_atoi.o
+ft_char.o
+ft_fd_output.o
+ft_itoa.o
+ft_list_basic.o
+ft_list_lifecycle.o
+ft_list_map.o
+ft_memory_copy.o
+ft_memory_fill.o
+ft_memory_move.o
+ft_memory_scan.o
+ft_split.o
+ft_string_bounds.o
+ft_string_build.o
+ft_string_search.o
+ft_string_transform.o
diff --git a/tests/check_archive.sh b/tests/check_archive.sh
new file mode 100644
index 0000000..5a82d8b
--- /dev/null
+++ b/tests/check_archive.sh
@@ -0,0 +1,83 @@
+#!/bin/sh
+
+set -eu
+
+archive=${1:-libft.a}
+output_dir=build/archive-check
+compiler=${CC:-cc}
+project_root=$(pwd -P)
+consumer_dir=$(mktemp -d "${TMPDIR:-/tmp}/libft-consumer.XXXXXX")
+
+cleanup()
+{
+	rm -rf "$consumer_dir"
+}
+
+trap cleanup EXIT HUP INT TERM
+
+case "$archive" in
+	/*)
+		archive_path=$archive
+		;;
+	*)
+		archive_path=$project_root/$archive
+		;;
+esac
+
+mkdir -p "$output_dir"
+
+ar t "$archive" | awk '/\.o$/' | sort > "$output_dir/members.actual"
+sort tests/archive-members.txt > "$output_dir/members.expected"
+cmp "$output_dir/members.expected" "$output_dir/members.actual"
+
+case "$(uname -s)" in
+	Darwin)
+		nm -gU -j "$archive" > "$output_dir/defined.raw"
+		nm -u -j "$archive" > "$output_dir/undefined.raw"
+		sed -E 's/^_//' "$output_dir/defined.raw" \
+			> "$output_dir/defined.normalized"
+		sed -E 's/^_//' "$output_dir/undefined.raw" \
+			> "$output_dir/undefined.normalized"
+		;;
+	Linux)
+		nm -g --defined-only -j "$archive" > "$output_dir/defined.raw"
+		nm -u -j "$archive" > "$output_dir/undefined.raw"
+		cat "$output_dir/defined.raw" > "$output_dir/defined.normalized"
+		cat "$output_dir/undefined.raw" > "$output_dir/undefined.normalized"
+		;;
+	*)
+		echo "unsupported symbol tool platform" >&2
+		exit 1
+		;;
+esac
+
+awk '/^[A-Za-z_][A-Za-z0-9_]*$/' "$output_dir/defined.normalized" \
+	| sort > "$output_dir/symbols.actual"
+sort tests/api-symbols.txt > "$output_dir/symbols.expected"
+cmp "$output_dir/symbols.expected" "$output_dir/symbols.actual"
+
+awk '/^[A-Za-z_][A-Za-z0-9_]*$/' "$output_dir/undefined.normalized" \
+	| sort -u \
+	> "$output_dir/undefined.all"
+comm -23 "$output_dir/undefined.all" "$output_dir/symbols.expected" \
+	> "$output_dir/undefined.external"
+{
+	cat tests/allowed-undefined.txt
+	case "$(uname -s)" in
+		Darwin)
+			printf '%s\n' __error
+			;;
+		Linux)
+			printf '%s\n' __errno_location
+			;;
+	esac
+} | sort > "$output_dir/undefined.expected"
+cmp "$output_dir/undefined.expected" "$output_dir/undefined.external"
+
+cp tests/smoke/consumer.c "$consumer_dir/consumer.c"
+(
+	cd "$consumer_dir"
+	"$compiler" -I"$project_root" -Wall -Wextra -Werror -Wpedantic \
+		-std=c99 -fno-builtin consumer.c "$archive_path" -o consumer
+	./consumer
+)
diff --git a/tests/smoke/consumer.c b/tests/smoke/consumer.c
new file mode 100644
index 0000000..231cf14
--- /dev/null
+++ b/tests/smoke/consumer.c
@@ -0,0 +1,19 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+int	main(void)
+{
+	char	*copy;
+	t_list	*node;
+
+	copy = ft_strdup("foundation");
+	if (copy == NULL || ft_strlen(copy) != 10)
+		return (free(copy), 1);
+	node = ft_lstnew(copy);
+	if (node == NULL)
+		return (free(copy), 1);
+	free(node->content);
+	free(node);
+	return (0);
+}


## `test(sanitize): undefined behavior 검사를 추가`

diff --git a/Makefile b/Makefile
index eb42d09..1220db9 100644
--- a/Makefile
+++ b/Makefile
@@ -52,8 +52,14 @@ WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
 	tests/support/fail_write.c
 WRITE_DEFINES := -Dwrite=test_write
 
+UBSAN_OBJ_DIR := build/ubsan
+UBSAN_OBJ := $(SRC:%.c=$(UBSAN_OBJ_DIR)/%.o)
+UBSAN_DEP := $(UBSAN_OBJ:.o=.d)
+UBSAN_BIN := tests/bin/test_ubsan
+UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
+
 .PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	check-archive
+	ubsan sanitize check-archive
 
 all: $(NAME)
 
@@ -102,6 +108,21 @@ $(WRITE_BIN): $(NAME) $(WRITE_OUTPUT_OBJ) $(WRITE_TEST_SRC) \
 write-failure-test: $(WRITE_BIN)
 	./$(WRITE_BIN)
 
+$(UBSAN_OBJ_DIR)/%.o: %.c libft.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(UBSAN_FLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(UBSAN_BIN): $(UBSAN_OBJ) $(TEST_SRC) tests/test.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(UBSAN_FLAGS) $(TEST_SRC) $(UBSAN_OBJ) -o $@
+
+ubsan: $(UBSAN_BIN)
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 ./$(UBSAN_BIN)
+
+sanitize: ubsan
+
 check-archive: $(NAME)
 	CC="$(CC)" sh tests/check_archive.sh $(NAME)
 
@@ -113,4 +134,4 @@ fclean: clean
 
 re: fclean all
 
--include $(DEP) $(FAIL_DEP) $(WRITE_DEP)
+-include $(DEP) $(FAIL_DEP) $(WRITE_DEP) $(UBSAN_DEP)


## `test(sanitize): address sanitizer 검사를 추가`

diff --git a/Makefile b/Makefile
index 1220db9..2c72ff1 100644
--- a/Makefile
+++ b/Makefile
@@ -52,6 +52,13 @@ WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
 	tests/support/fail_write.c
 WRITE_DEFINES := -Dwrite=test_write
 
+ASAN_OBJ_DIR := build/asan
+ASAN_OBJ := $(SRC:%.c=$(ASAN_OBJ_DIR)/%.o)
+ASAN_DEP := $(ASAN_OBJ:.o=.d)
+ASAN_BIN := tests/bin/test_asan
+ASAN_FLAGS := -fsanitize=address -fno-omit-frame-pointer
+ASAN_OPTIONS ?= detect_leaks=0:halt_on_error=1
+
 UBSAN_OBJ_DIR := build/ubsan
 UBSAN_OBJ := $(SRC:%.c=$(UBSAN_OBJ_DIR)/%.o)
 UBSAN_DEP := $(UBSAN_OBJ:.o=.d)
@@ -59,7 +66,7 @@ UBSAN_BIN := tests/bin/test_ubsan
 UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
 
 .PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	ubsan sanitize check-archive
+	asan ubsan sanitize check-archive
 
 all: $(NAME)
 
@@ -108,6 +115,19 @@ $(WRITE_BIN): $(NAME) $(WRITE_OUTPUT_OBJ) $(WRITE_TEST_SRC) \
 write-failure-test: $(WRITE_BIN)
 	./$(WRITE_BIN)
 
+$(ASAN_OBJ_DIR)/%.o: %.c libft.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(ASAN_FLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(ASAN_BIN): $(ASAN_OBJ) $(TEST_SRC) tests/test.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(ASAN_FLAGS) $(TEST_SRC) $(ASAN_OBJ) -o $@
+
+asan: $(ASAN_BIN)
+	ASAN_OPTIONS=$(ASAN_OPTIONS) ./$(ASAN_BIN)
+
 $(UBSAN_OBJ_DIR)/%.o: %.c libft.h
 	@$(MKDIR) $(dir $@)
 	$(CC) $(CPPFLAGS) $(CFLAGS) \
@@ -134,4 +154,4 @@ fclean: clean
 
 re: fclean all
 
--include $(DEP) $(FAIL_DEP) $(WRITE_DEP) $(UBSAN_DEP)
+-include $(DEP) $(FAIL_DEP) $(WRITE_DEP) $(ASAN_DEP) $(UBSAN_DEP)


## `test(leak): host 누수 검사 경로를 추가`

diff --git a/Makefile b/Makefile
index 2c72ff1..f682ce8 100644
--- a/Makefile
+++ b/Makefile
@@ -66,7 +66,7 @@ UBSAN_BIN := tests/bin/test_ubsan
 UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
 
 .PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	asan ubsan sanitize check-archive
+	asan ubsan sanitize leak check-archive
 
 all: $(NAME)
 
@@ -143,6 +143,17 @@ ubsan: $(UBSAN_BIN)
 
 sanitize: ubsan
 
+leak: $(TEST_BIN)
+	@if command -v leaks >/dev/null 2>&1; then \
+		leaks --atExit -- ./$(TEST_BIN); \
+	elif command -v valgrind >/dev/null 2>&1; then \
+		valgrind --leak-check=full --errors-for-leak-kinds=all \
+			--error-exitcode=1 ./$(TEST_BIN); \
+	else \
+		echo "no supported leak checker found" >&2; \
+		exit 1; \
+	fi
+
 check-archive: $(NAME)
 	CC="$(CC)" sh tests/check_archive.sh $(NAME)
 


## `test(build): Clang과 GCC 호환성을 검증`

diff --git a/Makefile b/Makefile
index f682ce8..3e71d7f 100644
--- a/Makefile
+++ b/Makefile
@@ -66,7 +66,7 @@ UBSAN_BIN := tests/bin/test_ubsan
 UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
 
 .PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	asan ubsan sanitize leak check-archive
+	asan ubsan sanitize leak check-archive check-compilers
 
 all: $(NAME)
 
@@ -157,6 +157,9 @@ leak: $(TEST_BIN)
 check-archive: $(NAME)
 	CC="$(CC)" sh tests/check_archive.sh $(NAME)
 
+check-compilers:
+	sh tests/check_compilers.sh
+
 clean:
 	$(RMDIR) build tests/bin
 
diff --git a/tests/check_compilers.sh b/tests/check_compilers.sh
new file mode 100644
index 0000000..d81576a
--- /dev/null
+++ b/tests/check_compilers.sh
@@ -0,0 +1,87 @@
+#!/bin/sh
+
+set -eu
+
+project_root=$(CDPATH='' cd -- "$(dirname "$0")/.." && pwd -P)
+scratch=$(mktemp -d "${TMPDIR:-/tmp}/libft-compilers.XXXXXX")
+
+cleanup()
+{
+	rm -rf "$scratch"
+}
+
+trap cleanup EXIT HUP INT TERM
+
+find_clang()
+{
+	for candidate in ${CLANG:-} clang cc gcc
+	do
+		if ! command -v "$candidate" >/dev/null 2>&1
+		then
+			continue
+		fi
+		version=$("$candidate" --version 2>/dev/null | sed -n '1p')
+		case "$version" in
+			*clang* | *Clang*)
+				printf '%s\n' "$candidate"
+				return
+				;;
+		esac
+	done
+	return 1
+}
+
+find_gcc()
+{
+	for candidate in ${GCC:-} gcc-15 gcc-14 gcc-13 gcc-12 gcc-11 gcc
+	do
+		if ! command -v "$candidate" >/dev/null 2>&1
+		then
+			continue
+		fi
+		version=$("$candidate" --version 2>/dev/null | sed -n '1p')
+		case "$version" in
+			*clang* | *Clang*)
+				continue
+				;;
+			*gcc* | *GCC* | *"Free Software Foundation"*)
+				printf '%s\n' "$candidate"
+				return
+				;;
+		esac
+	done
+	return 1
+}
+
+run_suite()
+{
+	label=$1
+	compiler=$2
+	work=$scratch/$label
+
+	mkdir -p "$work"
+	cp "$project_root/Makefile" "$project_root/libft.h" "$work/"
+	cp -R "$project_root/src" "$project_root/tests" "$work/"
+	printf 'compiler check: %s (%s)\n' "$label" \
+		"$("$compiler" --version | sed -n '1p')"
+	make -s -C "$work" CC="$compiler" fclean
+	make -s -C "$work" CC="$compiler" all test failure-test \
+		write-failure-test check-archive
+}
+
+clang_compiler=$(find_clang || true)
+gcc_compiler=$(find_gcc || true)
+
+if [ -z "$clang_compiler" ]
+then
+	echo "Clang compiler not found" >&2
+	exit 1
+fi
+if [ -z "$gcc_compiler" ]
+then
+	echo "GNU GCC compiler not found" >&2
+	exit 1
+fi
+
+run_suite clang "$clang_compiler"
+run_suite gcc "$gcc_compiler"


## `test(release): 전체 검증 절차를 연결`

diff --git a/Makefile b/Makefile
index 3e71d7f..bc7095a 100644
--- a/Makefile
+++ b/Makefile
@@ -66,7 +66,7 @@ UBSAN_BIN := tests/bin/test_ubsan
 UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
 
 .PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	asan ubsan sanitize leak check-archive check-compilers
+	asan ubsan sanitize leak check-archive check-compilers check
 
 all: $(NAME)
 
@@ -160,6 +160,19 @@ check-archive: $(NAME)
 check-compilers:
 	sh tests/check_compilers.sh
 
+check:
+	git diff --check
+	$(MAKE) fclean
+	$(MAKE) all
+	$(MAKE) test
+	$(MAKE) failure-test
+	$(MAKE) write-failure-test
+	$(MAKE) sanitize
+	$(MAKE) check-archive
+	$(MAKE) check-compilers
+	$(MAKE) leak
+	$(MAKE) -q all
+
 clean:
 	$(RMDIR) build tests/bin
 


## `build(targets): 결정적 검증 타깃을 노출`

diff --git a/Makefile b/Makefile
index bc7095a..7c770c6 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,8 @@ ARFLAGS := rcs
 RM := rm -f
 RMDIR := rm -rf
 MKDIR := mkdir -p
+LEAKS ?= leaks
+VALGRIND ?= valgrind
 
 SRC := \
 	src/char/ft_char.c \
@@ -65,11 +67,35 @@ UBSAN_DEP := $(UBSAN_OBJ:.o=.d)
 UBSAN_BIN := tests/bin/test_ubsan
 UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
 
-.PHONY: all bonus clean fclean re test failure-test write-failure-test \
-	asan ubsan sanitize leak check-archive check-compilers check
+.DEFAULT_GOAL := all
+
+.PHONY: all help bonus clean fclean re test failure-test write-failure-test \
+	asan ubsan sanitize leak check-archive check-compilers check-core check ci
 
 all: $(NAME)
 
+help:
+	@printf '%s\n' \
+		'Usage: make <target> [VARIABLE=value]' \
+		'' \
+		'Build:' \
+		'  all / bonus          Build libft.a' \
+		'  clean / fclean / re  Remove build output / rebuild' \
+		'' \
+		'Validation:' \
+		'  test                 Run the 43-API regression suite' \
+		'  failure-test         Run allocation failure injection' \
+		'  write-failure-test   Run write failure injection' \
+		'  asan / ubsan         Run sanitizer-specific test binaries' \
+		'  leak                 Run leaks or Valgrind' \
+		'  check-archive        Verify archive, API, and consumer linkage' \
+		'  check-compilers      Run the GCC and Clang compatibility suite' \
+		'  check-core           Run the portable functional gate' \
+		'  check                Add local hygiene, compilers, and leaks' \
+		'  ci                   Extend the strict gate with ASan' \
+		'' \
+		'Overrides: CC, ASAN_OPTIONS, LEAKS, VALGRIND'
+
 bonus: all
 
 $(NAME): $(OBJ)
@@ -144,10 +170,10 @@ ubsan: $(UBSAN_BIN)
 sanitize: ubsan
 
 leak: $(TEST_BIN)
-	@if command -v leaks >/dev/null 2>&1; then \
-		leaks --atExit -- ./$(TEST_BIN); \
-	elif command -v valgrind >/dev/null 2>&1; then \
-		valgrind --leak-check=full --errors-for-leak-kinds=all \
+	@if command -v "$(LEAKS)" >/dev/null 2>&1; then \
+		"$(LEAKS)" --atExit -- ./$(TEST_BIN); \
+	elif command -v "$(VALGRIND)" >/dev/null 2>&1; then \
+		"$(VALGRIND)" --leak-check=full --errors-for-leak-kinds=all \
 			--error-exitcode=1 ./$(TEST_BIN); \
 	else \
 		echo "no supported leak checker found" >&2; \
@@ -160,8 +186,7 @@ check-archive: $(NAME)
 check-compilers:
 	sh tests/check_compilers.sh
 
-check:
-	git diff --check
+check-core:
 	$(MAKE) fclean
 	$(MAKE) all
 	$(MAKE) test
@@ -169,9 +194,24 @@ check:
 	$(MAKE) write-failure-test
 	$(MAKE) sanitize
 	$(MAKE) check-archive
+	$(MAKE) -q all
+
+check:
+	git diff --check
+	$(MAKE) check-core
 	$(MAKE) check-compilers
 	$(MAKE) leak
-	$(MAKE) -q all
+
+ci:
+	@if command -v "$(LEAKS)" >/dev/null 2>&1 \
+		|| command -v "$(VALGRIND)" >/dev/null 2>&1; then :; else \
+		echo "ci requires leaks or Valgrind" >&2; \
+		exit 2; \
+	fi
+	$(MAKE) check-core
+	$(MAKE) check-compilers
+	$(MAKE) leak
+	$(MAKE) asan
 
 clean:
 	$(RMDIR) build tests/bin


## `ci(compilers): 크로스 플랫폼 C 검증을 추가`

diff --git a/.github/workflows/c-libft-ci.yml b/.github/workflows/c-libft-ci.yml
new file mode 100644
index 0000000..0f8df15
--- /dev/null
+++ b/.github/workflows/c-libft-ci.yml
@@ -0,0 +1,78 @@
+name: c/libft CI
+
+on:
+  push:
+    branches:
+      - c/libft
+  pull_request:
+    branches:
+      - c/libft
+
+permissions:
+  contents: read
+
+concurrency:
+  group: ${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+env:
+  LC_ALL: C
+  LANG: C
+  TZ: UTC
+
+jobs:
+  linux-full:
+    name: Linux / GCC 14 and Clang 18 / full
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    env:
+      GCC: gcc-14
+      CLANG: clang-18
+      ASAN_OPTIONS: detect_leaks=0:halt_on_error=1
+      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
+    steps:
+      - name: Check out the branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Install the leak checker
+        run: |
+          sudo apt-get update
+          sudo apt-get install --yes valgrind
+
+      - name: Report toolchain
+        run: |
+          uname -a
+          "$GCC" --version
+          "$CLANG" --version
+          make --version
+          valgrind --version
+
+      - name: Validate test scripts
+        run: shellcheck tests/*.sh
+
+      - name: Run the strict functional gate
+        run: make CC="$GCC" ci
+
+  macos-clang:
+    name: macOS / Apple Clang / core
+    runs-on: macos-15
+    timeout-minutes: 15
+    env:
+      CC: clang
+      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
+    steps:
+      - name: Check out the branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        run: |
+          uname -a
+          "$CC" --version
+          make --version
+
+      - name: Run the portable functional gate
+        run: make CC="$CC" check-core


## `ci(archive): 컴파일러 런타임 아카이브 심볼을 허용`

diff --git a/tests/check_archive.sh b/tests/check_archive.sh
index 5a82d8b..d2f8324 100644
--- a/tests/check_archive.sh
+++ b/tests/check_archive.sh
@@ -59,7 +59,9 @@ cmp "$output_dir/symbols.expected" "$output_dir/symbols.actual"
 awk '/^[A-Za-z_][A-Za-z0-9_]*$/' "$output_dir/undefined.normalized" \
 	| sort -u \
 	> "$output_dir/undefined.all"
+# Stack-protector thunks come from compiler policy, not the libft source ABI.
 comm -23 "$output_dir/undefined.all" "$output_dir/symbols.expected" \
+	| awk '$0 != "_stack_chk_fail" && $0 != "__stack_chk_fail"' \
 	> "$output_dir/undefined.external"
 {
 	cat tests/allowed-undefined.txt
