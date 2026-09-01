## `fix(runtime): select 범위를 벗어난 descriptor 거부`

diff --git a/src/client.c b/src/client.c
index 8f0a09c..d641a2b 100644
--- a/src/client.c
+++ b/src/client.c
@@ -87,6 +87,7 @@ static int	bind_client_socket(void)
 		return (-1);
 	g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
 	if (g_response_socket == -1
+		|| g_response_socket >= FD_SETSIZE
 		|| set_nonblocking_close_on_exec(g_response_socket) == -1)
 		return (-1);
 	memset(&address, 0, sizeof(address));
diff --git a/src/server.c b/src/server.c
index 3907913..d5c96bc 100644
--- a/src/server.c
+++ b/src/server.c
@@ -143,6 +143,7 @@ static int	prepare_response_channel(void)
 	struct sockaddr_un	address;
 
 	if (pipe(g_event_pipe) == -1
+		|| g_event_pipe[0] >= FD_SETSIZE
 		|| set_close_on_exec(g_event_pipe[0]) == -1
 		|| set_nonblocking_close_on_exec(g_event_pipe[1]) == -1)
 		return (-1);
@@ -151,6 +152,7 @@ static int	prepare_response_channel(void)
 		return (-1);
 	g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
 	if (g_response_socket == -1
+		|| g_response_socket >= FD_SETSIZE
 		|| set_nonblocking_close_on_exec(g_response_socket) == -1)
 		return (-1);
 	memset(&address, 0, sizeof(address));


## `test(runtime): 높은 descriptor 번호의 안전한 실패 검증`

diff --git a/.gitignore b/.gitignore
index 219ba30..03ea188 100644
--- a/.gitignore
+++ b/.gitignore
@@ -4,6 +4,7 @@ server
 client
 obj/
 tests/fault_server
+tests/high_fd_exec
 tests/masked_exec
 tests/parse_pid_test
 tests/response_server
diff --git a/Makefile b/Makefile
index 2ad5dbb..5c9989e 100644
--- a/Makefile
+++ b/Makefile
@@ -7,6 +7,7 @@ NAME_PARSE_PID_TEST := tests/parse_pid_test
 NAME_FAULT_SERVER := tests/fault_server
 NAME_STALE_EXEC := tests/stale_exec
 NAME_STALE_SERVER_EXEC := tests/stale_server_exec
+NAME_HIGH_FD_EXEC := tests/high_fd_exec
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
@@ -32,6 +33,7 @@ FAULT_SERVER_OBJ := $(SERVER_SRC:%.c=$(FAULT_OBJ_DIR)/%.o) \
 	$(OBJ_DIR)/tests/write_fault.o
 STALE_EXEC_OBJ := $(OBJ_DIR)/tests/stale_exec.o $(COMMON_OBJ)
 STALE_SERVER_EXEC_OBJ := $(OBJ_DIR)/tests/stale_server_exec.o $(COMMON_OBJ)
+HIGH_FD_EXEC_OBJ := $(OBJ_DIR)/tests/high_fd_exec.o
 
 .PHONY: all clean fclean re test
 
@@ -64,6 +66,9 @@ $(NAME_STALE_EXEC): $(STALE_EXEC_OBJ)
 $(NAME_STALE_SERVER_EXEC): $(STALE_SERVER_EXEC_OBJ)
 	$(CC) $(CFLAGS) $(STALE_SERVER_EXEC_OBJ) -o $@
 
+$(NAME_HIGH_FD_EXEC): $(HIGH_FD_EXEC_OBJ)
+	$(CC) $(CFLAGS) $(HIGH_FD_EXEC_OBJ) -o $@
+
 $(OBJ_DIR)/%.o: %.c include/minitalk.h
 	mkdir -p $(dir $@)
 	$(CC) $(CFLAGS) -c $< -o $@
@@ -79,15 +84,17 @@ fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT) $(NAME_SESSION_SENDER) \
 		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) $(NAME_PARSE_PID_TEST) \
 		$(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) $(NAME_STALE_SERVER_EXEC)
+	$(RM) $(NAME_HIGH_FD_EXEC)
 
 re: fclean all
 
 test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) \
 		$(NAME_PARSE_PID_TEST) $(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) \
-		$(NAME_STALE_SERVER_EXEC)
+		$(NAME_STALE_SERVER_EXEC) $(NAME_HIGH_FD_EXEC)
 	./$(NAME_PARSE_PID_TEST)
 	sh tests/smoke.sh
 	sh tests/session_ownership.sh
 	sh tests/response_validation.sh
 	sh tests/output_failure.sh
 	sh tests/protocol_regressions.sh
+	sh tests/high_fd.sh
diff --git a/tests/high_fd.sh b/tests/high_fd.sh
new file mode 100755
index 0000000..9407d55
--- /dev/null
+++ b/tests/high_fd.sh
@@ -0,0 +1,53 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/minitalk-high-fd.XXXXXX")
+OUT="$TEST_TMP/server.out"
+ERR="$TEST_TMP/server.err"
+CLIENT_ERR="$TEST_TMP/client.err"
+HIGH_SERVER_OUT="$TEST_TMP/high-server.out"
+HIGH_SERVER_ERR="$TEST_TMP/high-server.err"
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
+"$ROOT/server" >"$OUT" 2>"$ERR" &
+SERVER_PID=$!
+tries=0
+while [ "$tries" -lt 50 ] && ! grep -qx "$SERVER_PID" "$OUT" 2>/dev/null; do
+	if ! kill -0 "$SERVER_PID" 2>/dev/null; then
+		printf 'server exited before high descriptor test\n' >&2
+		exit 1
+	fi
+	tries=$((tries + 1))
+	sleep 0.1
+done
+grep -qx "$SERVER_PID" "$OUT"
+
+client_status=0
+"$ROOT/tests/high_fd_exec" "$ROOT/client" "$SERVER_PID" high-fd \
+	2>"$CLIENT_ERR" || client_status=$?
+[ "$client_status" -eq 1 ]
+grep -qx 'client: failed to create response channel' "$CLIENT_ERR"
+kill -0 "$SERVER_PID"
+[ ! -s "$ERR" ]
+
+server_status=0
+"$ROOT/tests/high_fd_exec" "$ROOT/server" \
+	>"$HIGH_SERVER_OUT" 2>"$HIGH_SERVER_ERR" || server_status=$?
+[ "$server_status" -eq 1 ]
+[ ! -s "$HIGH_SERVER_OUT" ]
+grep -qx 'server: failed to create response channel' "$HIGH_SERVER_ERR"
diff --git a/tests/high_fd_exec.c b/tests/high_fd_exec.c
new file mode 100644
index 0000000..5157db9
--- /dev/null
+++ b/tests/high_fd_exec.c
@@ -0,0 +1,22 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include <fcntl.h>
+#include <sys/select.h>
+#include <unistd.h>
+
+int	main(int argc, char **argv)
+{
+	int	fd;
+
+	if (argc < 2)
+		return (2);
+	fd = -1;
+	while (fd < FD_SETSIZE - 1)
+	{
+		fd = open("/dev/null", O_RDONLY);
+		if (fd == -1)
+			return (1);
+	}
+	execv(argv[1], &argv[1]);
+	return (127);
+}
