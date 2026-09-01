## `test(server): 부분 쓰기와 출력 실패 검증`

diff --git a/.gitignore b/.gitignore
index ca6fb8b..bb944cc 100644
--- a/.gitignore
+++ b/.gitignore
@@ -3,6 +3,7 @@
 server
 client
 obj/
+tests/fault_server
 tests/masked_exec
 tests/parse_pid_test
 tests/response_server
diff --git a/Makefile b/Makefile
index 23b0265..f791ff0 100644
--- a/Makefile
+++ b/Makefile
@@ -4,11 +4,14 @@ NAME_SESSION_SENDER := tests/session_sender
 NAME_MASKED_EXEC := tests/masked_exec
 NAME_RESPONSE_SERVER := tests/response_server
 NAME_PARSE_PID_TEST := tests/parse_pid_test
+NAME_FAULT_SERVER := tests/fault_server
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
+FAULT_CFLAGS := $(CFLAGS) -DMT_WRITE_CALL=mt_test_write -include tests/write_fault.h
 RM := rm -rf
 OBJ_DIR := obj
+FAULT_OBJ_DIR := obj/fault
 
 COMMON_SRC := src/write_utils.c src/parse_pid.c src/response_channel.c
 SERVER_SRC := src/server.c $(COMMON_SRC)
@@ -22,6 +25,8 @@ MASKED_EXEC_OBJ := $(OBJ_DIR)/tests/masked_exec.o
 RESPONSE_SERVER_OBJ := $(OBJ_DIR)/tests/response_server.o $(COMMON_OBJ)
 PARSE_PID_TEST_OBJ := $(OBJ_DIR)/tests/parse_pid_test.o \
 	$(OBJ_DIR)/src/parse_pid.o
+FAULT_SERVER_OBJ := $(SERVER_SRC:%.c=$(FAULT_OBJ_DIR)/%.o) \
+	$(OBJ_DIR)/tests/write_fault.o
 
 .PHONY: all clean fclean re test
 
@@ -45,22 +50,31 @@ $(NAME_RESPONSE_SERVER): $(RESPONSE_SERVER_OBJ)
 $(NAME_PARSE_PID_TEST): $(PARSE_PID_TEST_OBJ)
 	$(CC) $(CFLAGS) $(PARSE_PID_TEST_OBJ) -o $@
 
+$(NAME_FAULT_SERVER): $(FAULT_SERVER_OBJ)
+	$(CC) $(CFLAGS) $(FAULT_SERVER_OBJ) -o $@
+
 $(OBJ_DIR)/%.o: %.c include/minitalk.h
 	mkdir -p $(dir $@)
 	$(CC) $(CFLAGS) -c $< -o $@
 
+$(FAULT_OBJ_DIR)/%.o: %.c include/minitalk.h tests/write_fault.h
+	mkdir -p $(dir $@)
+	$(CC) $(FAULT_CFLAGS) -c $< -o $@
+
 clean:
 	$(RM) obj
 
 fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT) $(NAME_SESSION_SENDER) \
-		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) $(NAME_PARSE_PID_TEST)
+		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) $(NAME_PARSE_PID_TEST) \
+		$(NAME_FAULT_SERVER)
 
 re: fclean all
 
 test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) \
-		$(NAME_PARSE_PID_TEST)
+		$(NAME_PARSE_PID_TEST) $(NAME_FAULT_SERVER)
 	./$(NAME_PARSE_PID_TEST)
 	sh tests/smoke.sh
 	sh tests/session_ownership.sh
 	sh tests/response_validation.sh
+	sh tests/output_failure.sh
diff --git a/src/write_utils.c b/src/write_utils.c
index 703981e..49bc835 100644
--- a/src/write_utils.c
+++ b/src/write_utils.c
@@ -3,6 +3,10 @@
 #include <errno.h>
 #include <unistd.h>
 
+#ifndef MT_WRITE_CALL
+# define MT_WRITE_CALL write
+#endif
+
 size_t	mt_strlen(const char *text)
 {
 	size_t	length;
@@ -23,7 +27,7 @@ int	mt_write_all(int fd, const void *buffer, size_t size)
 	offset = 0;
 	while (offset < size)
 	{
-		written = write(fd, bytes + offset, size - offset);
+		written = MT_WRITE_CALL(fd, bytes + offset, size - offset);
 		if (written == -1 && errno == EINTR)
 			continue ;
 		if (written == -1)
diff --git a/tests/output_failure.sh b/tests/output_failure.sh
new file mode 100755
index 0000000..0473e1c
--- /dev/null
+++ b/tests/output_failure.sh
@@ -0,0 +1,112 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/signal-message-bus-output.XXXXXX")
+SERVER_PID=
+
+cleanup()
+{
+	if [ -n "$SERVER_PID" ]; then
+		kill "$SERVER_PID" 2>/dev/null || true
+		wait "$SERVER_PID" 2>/dev/null || true
+	fi
+	rm -rf "$TEST_TMP"
+}
+
+trap cleanup EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+wait_ready()
+{
+	out=$1
+	tries=0
+	while [ "$tries" -lt 50 ] && ! grep -qx "$SERVER_PID" "$out" 2>/dev/null; do
+		if ! kill -0 "$SERVER_PID" 2>/dev/null; then
+			printf 'fault server exited before publishing its pid\n' >&2
+			exit 1
+		fi
+		tries=$((tries + 1))
+		sleep 0.1
+	done
+	grep -qx "$SERVER_PID" "$out"
+}
+
+STARTUP_OUT="$TEST_TMP/startup.out"
+STARTUP_ERR="$TEST_TMP/startup.err"
+MT_TEST_ZERO_ONCE=1 "$ROOT/tests/fault_server" >"$STARTUP_OUT" 2>"$STARTUP_ERR" &
+STARTUP_PID=$!
+startup_status=0
+wait "$STARTUP_PID" || startup_status=$?
+if [ "$startup_status" -eq 0 ] || [ -s "$STARTUP_OUT" ]; then
+	printf 'zero-length startup write did not fail\n' >&2
+	exit 1
+fi
+grep -qx 'server: failed to publish pid' "$STARTUP_ERR"
+if [ -e "/tmp/signal-message-bus-$(id -u)/server-$STARTUP_PID.sock" ]; then
+	printf 'failed startup left a response socket\n' >&2
+	exit 1
+fi
+
+PARTIAL_OUT="$TEST_TMP/partial.out"
+PARTIAL_ERR="$TEST_TMP/partial.err"
+CLIENT_ERR="$TEST_TMP/client.err"
+MT_TEST_MAX_WRITE=1 MT_TEST_EINTR_ONCE=1 \
+	"$ROOT/tests/fault_server" >"$PARTIAL_OUT" 2>"$PARTIAL_ERR" &
+SERVER_PID=$!
+wait_ready "$PARTIAL_OUT"
+"$ROOT/client" "$SERVER_PID" partial 2>"$CLIENT_ERR"
+kill "$SERVER_PID"
+wait "$SERVER_PID" 2>/dev/null || true
+SERVER_PID=
+if [ -s "$PARTIAL_ERR" ] || [ -s "$CLIENT_ERR" ]; then
+	cat "$PARTIAL_ERR" "$CLIENT_ERR" >&2
+	exit 1
+fi
+{
+	sed -n '1p' "$PARTIAL_OUT"
+	printf 'partial\n'
+} >"$TEST_TMP/partial.expected"
+diff -u "$TEST_TMP/partial.expected" "$PARTIAL_OUT"
+
+BYTE_OUT="$TEST_TMP/byte.out"
+BYTE_ERR="$TEST_TMP/byte.err"
+BYTE_CLIENT_ERR="$TEST_TMP/byte-client.err"
+MT_TEST_FAIL_BYTE=X MT_TEST_FAIL_EPIPE=1 \
+	"$ROOT/tests/fault_server" >"$BYTE_OUT" 2>"$BYTE_ERR" &
+SERVER_PID=$!
+wait_ready "$BYTE_OUT"
+client_status=0
+"$ROOT/client" "$SERVER_PID" X 2>"$BYTE_CLIENT_ERR" || client_status=$?
+server_status=0
+wait "$SERVER_PID" || server_status=$?
+SERVER_PID=
+if [ "$client_status" -eq 0 ] || [ "$server_status" -eq 0 ]; then
+	printf 'output failure was acknowledged as success\n' >&2
+	exit 1
+fi
+[ "$(wc -l <"$BYTE_OUT")" -eq 1 ]
+grep -qx 'server: signal event channel failed' "$BYTE_ERR"
+grep -qx 'client: timed out waiting for acknowledgement' "$BYTE_CLIENT_ERR"
+
+NEWLINE_OUT="$TEST_TMP/newline.out"
+NEWLINE_ERR="$TEST_TMP/newline.err"
+NEWLINE_CLIENT_ERR="$TEST_TMP/newline-client.err"
+MT_TEST_FAIL_NEWLINE_NUMBER=2 MT_TEST_FAIL_EPIPE=1 \
+	"$ROOT/tests/fault_server" >"$NEWLINE_OUT" 2>"$NEWLINE_ERR" &
+SERVER_PID=$!
+wait_ready "$NEWLINE_OUT"
+client_status=0
+"$ROOT/client" "$SERVER_PID" "" 2>"$NEWLINE_CLIENT_ERR" || client_status=$?
+server_status=0
+wait "$SERVER_PID" || server_status=$?
+SERVER_PID=
+if [ "$client_status" -eq 0 ] || [ "$server_status" -eq 0 ]; then
+	printf 'terminating newline failure was acknowledged as success\n' >&2
+	exit 1
+fi
+[ "$(wc -l <"$NEWLINE_OUT")" -eq 1 ]
+grep -qx 'server: signal event channel failed' "$NEWLINE_ERR"
+grep -qx 'client: timed out waiting for acknowledgement' "$NEWLINE_CLIENT_ERR"
diff --git a/tests/write_fault.c b/tests/write_fault.c
new file mode 100644
index 0000000..0f10b67
--- /dev/null
+++ b/tests/write_fault.c
@@ -0,0 +1,103 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "write_fault.h"
+
+#include <errno.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+static int		g_initialized;
+static size_t	g_max_write;
+static int		g_interrupt_once;
+static int		g_interrupted;
+static int		g_zero_once;
+static int		g_zeroed;
+static int		g_fail_byte;
+static int		g_fail_errno;
+static long		g_fail_newline_number;
+static long		g_newline_count;
+
+static long	read_number(const char *name, long fallback)
+{
+	const char	*text;
+	char		*end;
+	long		value;
+
+	text = getenv(name);
+	if (text == NULL || text[0] == '\0')
+		return (fallback);
+	errno = 0;
+	value = strtol(text, &end, 10);
+	if (errno != 0 || *end != '\0')
+		return (fallback);
+	return (value);
+}
+
+static void	initialize(void)
+{
+	const char	*text;
+	long		value;
+
+	g_initialized = 1;
+	value = read_number("MT_TEST_MAX_WRITE", 0);
+	if (value > 0)
+		g_max_write = (size_t)value;
+	g_interrupt_once = read_number("MT_TEST_EINTR_ONCE", 0) == 1;
+	g_zero_once = read_number("MT_TEST_ZERO_ONCE", 0) == 1;
+	text = getenv("MT_TEST_FAIL_BYTE");
+	if (text != NULL && text[0] != '\0' && text[1] == '\0')
+		g_fail_byte = (unsigned char)text[0];
+	g_fail_errno = (int)read_number("MT_TEST_FAIL_ERRNO", EIO);
+	if (read_number("MT_TEST_FAIL_EPIPE", 0) == 1)
+		g_fail_errno = EPIPE;
+	g_fail_newline_number = read_number("MT_TEST_FAIL_NEWLINE_NUMBER", 0);
+}
+
+static int	should_fail(int fd, const unsigned char *bytes, size_t size)
+{
+	size_t	index;
+
+	if (fd != STDOUT_FILENO)
+		return (0);
+	index = 0;
+	while (index < size)
+	{
+		if (g_fail_byte != 0 && bytes[index] == (unsigned char)g_fail_byte)
+			return (1);
+		if (bytes[index] == '\n')
+		{
+			g_newline_count++;
+			if (g_fail_newline_number > 0
+				&& g_newline_count == g_fail_newline_number)
+				return (1);
+		}
+		index++;
+	}
+	return (0);
+}
+
+ssize_t	mt_test_write(int fd, const void *buffer, size_t size)
+{
+	if (!g_initialized)
+		initialize();
+	if (fd == STDOUT_FILENO && g_zero_once && !g_zeroed)
+	{
+		g_zeroed = 1;
+		return (0);
+	}
+	if (fd == STDOUT_FILENO && g_interrupt_once && !g_interrupted)
+	{
+		g_interrupted = 1;
+		errno = EINTR;
+		return (-1);
+	}
+	if (should_fail(fd, (const unsigned char *)buffer, size))
+	{
+		errno = g_fail_errno;
+		return (-1);
+	}
+	if (g_max_write > 0 && size > g_max_write)
+		size = g_max_write;
+	return (write(fd, buffer, size));
+}
diff --git a/tests/write_fault.h b/tests/write_fault.h
new file mode 100644
index 0000000..98af774
--- /dev/null
+++ b/tests/write_fault.h
@@ -0,0 +1,9 @@
+#ifndef WRITE_FAULT_H
+# define WRITE_FAULT_H
+
+# include <stddef.h>
+# include <sys/types.h>
+
+ssize_t	mt_test_write(int fd, const void *buffer, size_t size);
+
+#endif


## `test(server): 회수 줄바꿈 출력 실패 검증`

diff --git a/tests/output_failure.sh b/tests/output_failure.sh
index 0473e1c..f38aa78 100755
--- a/tests/output_failure.sh
+++ b/tests/output_failure.sh
@@ -110,3 +110,26 @@ fi
 [ "$(wc -l <"$NEWLINE_OUT")" -eq 1 ]
 grep -qx 'server: signal event channel failed' "$NEWLINE_ERR"
 grep -qx 'client: timed out waiting for acknowledgement' "$NEWLINE_CLIENT_ERR"
+
+RECOVERY_OUT="$TEST_TMP/recovery.out"
+RECOVERY_ERR="$TEST_TMP/recovery.err"
+RECOVERY_CLIENT_ERR="$TEST_TMP/recovery-client.err"
+MT_TEST_FAIL_NEWLINE_NUMBER=2 MT_TEST_FAIL_EPIPE=1 \
+	"$ROOT/tests/fault_server" >"$RECOVERY_OUT" 2>"$RECOVERY_ERR" &
+SERVER_PID=$!
+wait_ready "$RECOVERY_OUT"
+"$ROOT/tests/session_sender" "$SERVER_PID" partial
+recovery_client_status=0
+"$ROOT/client" "$SERVER_PID" recovered 2>"$RECOVERY_CLIENT_ERR" \
+	|| recovery_client_status=$?
+recovery_server_status=0
+wait "$SERVER_PID" || recovery_server_status=$?
+SERVER_PID=
+if [ "$recovery_client_status" -eq 0 ] || [ "$recovery_server_status" -eq 0 ]; then
+	printf 'abandoned partial-line recovery failure was acknowledged\n' >&2
+	exit 1
+fi
+[ "$(wc -l <"$RECOVERY_OUT")" -eq 1 ]
+grep -qx 'server: signal event channel failed' "$RECOVERY_ERR"
+grep -qx 'client: timed out waiting for acknowledgement' \
+	"$RECOVERY_CLIENT_ERR"
