## `test(session): 데이터 없는 활성 예약 경쟁 검증`

diff --git a/tests/session_ownership.sh b/tests/session_ownership.sh
index 767ee70..360b669 100644
--- a/tests/session_ownership.sh
+++ b/tests/session_ownership.sh
@@ -72,7 +72,7 @@ if ! "$ROOT/client" "$SERVER_PID" "line recovered" 2>>"$CLIENT_ERR"; then
 fi
 
 : >"$READY"
-"$ROOT/tests/session_sender" "$SERVER_PID" hold >"$READY" &
+"$ROOT/tests/session_sender" "$SERVER_PID" reserve >"$READY" &
 HOLDER_PID=$!
 tries=0
 while [ "$tries" -lt 50 ]; do
@@ -80,14 +80,14 @@ while [ "$tries" -lt 50 ]; do
 		break
 	fi
 	if ! kill -0 "$HOLDER_PID" 2>/dev/null; then
-		printf 'session holder exited before becoming ready\n' >&2
+		printf 'session reserver exited before becoming ready\n' >&2
 		exit 1
 	fi
 	tries=$((tries + 1))
 	sleep 0.1
 done
 if ! grep -qx 'ready' "$READY"; then
-	printf 'session holder did not become ready\n' >&2
+	printf 'session reserver did not become ready\n' >&2
 	exit 1
 fi
 
@@ -103,8 +103,8 @@ diff -u "$BEFORE_BUSY" "$OUT"
 kill "$HOLDER_PID"
 wait "$HOLDER_PID" 2>/dev/null || true
 HOLDER_PID=
-if ! "$ROOT/client" "$SERVER_PID" "holder recovered" 2>>"$CLIENT_ERR"; then
-	printf 'server did not recover after the live owner exited\n' >&2
+if ! "$ROOT/client" "$SERVER_PID" "reservation recovered" 2>>"$CLIENT_ERR"; then
+	printf 'server did not recover after the live reserved owner exited\n' >&2
 	cat "$CLIENT_ERR" >&2
 	exit 1
 fi
@@ -134,8 +134,7 @@ fi
 	printf 'bit recovered\n'
 	printf 'X\n'
 	printf 'line recovered\n'
-	printf 'X\n'
-	printf 'holder recovered\n'
+	printf 'reservation recovered\n'
 	printf 'masked ack\n'
 } >"$EXPECTED"
 
diff --git a/tests/session_sender.c b/tests/session_sender.c
index 331cba3..5381015 100644
--- a/tests/session_sender.c
+++ b/tests/session_sender.c
@@ -189,15 +189,14 @@ int	main(int argc, char **argv)
 		if (send_bit(server_pid, 0, sequence, server_path) == -1)
 			return (1);
 	}
-	else if (strcmp(argv[2], "partial") == 0
-		|| strcmp(argv[2], "hold") == 0)
+	else if (strcmp(argv[2], "partial") == 0)
 	{
 		if (send_partial(server_pid, &sequence, server_path) == -1)
 			return (1);
 	}
-	else
+	else if (strcmp(argv[2], "reserve") != 0)
 		return (1);
-	if (strcmp(argv[2], "hold") == 0)
+	if (strcmp(argv[2], "reserve") == 0)
 	{
 		write(STDOUT_FILENO, "ready\n", 6);
 		while (1)


## `fix(server): 응답 경로가 사라진 세션 소유자 회수`

diff --git a/src/server.c b/src/server.c
index c690dcc..4fb79f5 100644
--- a/src/server.c
+++ b/src/server.c
@@ -224,6 +224,20 @@ static int	valid_client_socket(const char *path)
 	return (0);
 }
 
+static int	session_owner_available(void)
+{
+	char	client_path[MT_RESPONSE_PATH_SIZE];
+
+	if (g_client_pid <= 1)
+		return (0);
+	if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
+		return (0);
+	if (mt_response_path(client_path, sizeof(client_path), "client",
+			g_client_pid) == -1 || valid_client_socket(client_path) == -1)
+		return (0);
+	return (1);
+}
+
 static int	send_response(pid_t client_pid, uint32_t kind, uint32_t token,
 		int status)
 {
@@ -307,7 +321,7 @@ static int	handle_session_request(void)
 	status = MT_RESPONSE_OK;
 	if (g_client_pid != 0 && g_client_pid != request.client_pid)
 	{
-		if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
+		if (!session_owner_available())
 		{
 			if (reset_session(1) == -1)
 				return (-1);


## `test(session): 종료 송신자 회수 전 새 세션 복구 검증`

diff --git a/.gitignore b/.gitignore
index 03ea188..cec0747 100644
--- a/.gitignore
+++ b/.gitignore
@@ -11,3 +11,4 @@ tests/response_server
 tests/session_sender
 tests/stale_exec
 tests/stale_server_exec
+tests/unreaped_exec
diff --git a/Makefile b/Makefile
index 7b71b48..08ea459 100644
--- a/Makefile
+++ b/Makefile
@@ -8,6 +8,7 @@ NAME_FAULT_SERVER := tests/fault_server
 NAME_STALE_EXEC := tests/stale_exec
 NAME_STALE_SERVER_EXEC := tests/stale_server_exec
 NAME_HIGH_FD_EXEC := tests/high_fd_exec
+NAME_UNREAPED_EXEC := tests/unreaped_exec
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
@@ -34,6 +35,7 @@ FAULT_SERVER_OBJ := $(SERVER_SRC:%.c=$(FAULT_OBJ_DIR)/%.o) \
 STALE_EXEC_OBJ := $(OBJ_DIR)/tests/stale_exec.o $(COMMON_OBJ)
 STALE_SERVER_EXEC_OBJ := $(OBJ_DIR)/tests/stale_server_exec.o $(COMMON_OBJ)
 HIGH_FD_EXEC_OBJ := $(OBJ_DIR)/tests/high_fd_exec.o
+UNREAPED_EXEC_OBJ := $(OBJ_DIR)/tests/unreaped_exec.o $(COMMON_OBJ)
 
 .PHONY: all clean fclean re test
 
@@ -69,6 +71,9 @@ $(NAME_STALE_SERVER_EXEC): $(STALE_SERVER_EXEC_OBJ)
 $(NAME_HIGH_FD_EXEC): $(HIGH_FD_EXEC_OBJ)
 	$(CC) $(CFLAGS) $(HIGH_FD_EXEC_OBJ) -o $@
 
+$(NAME_UNREAPED_EXEC): $(UNREAPED_EXEC_OBJ)
+	$(CC) $(CFLAGS) $(UNREAPED_EXEC_OBJ) -o $@
+
 $(OBJ_DIR)/%.o: %.c include/minitalk.h
 	mkdir -p $(dir $@)
 	$(CC) $(CFLAGS) -c $< -o $@
@@ -84,13 +89,13 @@ fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT) $(NAME_SESSION_SENDER) \
 		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) $(NAME_PARSE_PID_TEST) \
 		$(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) $(NAME_STALE_SERVER_EXEC)
-	$(RM) $(NAME_HIGH_FD_EXEC)
+	$(RM) $(NAME_HIGH_FD_EXEC) $(NAME_UNREAPED_EXEC)
 
 re: fclean all
 
 test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) \
 		$(NAME_PARSE_PID_TEST) $(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) \
-		$(NAME_STALE_SERVER_EXEC) $(NAME_HIGH_FD_EXEC)
+		$(NAME_STALE_SERVER_EXEC) $(NAME_HIGH_FD_EXEC) $(NAME_UNREAPED_EXEC)
 	./$(NAME_PARSE_PID_TEST)
 	sh tests/smoke.sh
 	sh tests/session_ownership.sh
diff --git a/tests/session_ownership.sh b/tests/session_ownership.sh
index 360b669..be8f0cb 100644
--- a/tests/session_ownership.sh
+++ b/tests/session_ownership.sh
@@ -10,12 +10,18 @@ BUSY_ERR="$TEST_TMP/busy.err"
 EXPECTED_BUSY="$TEST_TMP/expected-busy.err"
 EXPECTED="$TEST_TMP/expected.out"
 READY="$TEST_TMP/ready.out"
+UNREAPED_READY="$TEST_TMP/unreaped-ready.out"
 BEFORE_BUSY="$TEST_TMP/before-busy.out"
 SERVER_PID=
 HOLDER_PID=
+UNREAPED_PARENT_PID=
 
 cleanup()
 {
+	if [ -n "$UNREAPED_PARENT_PID" ]; then
+		kill "$UNREAPED_PARENT_PID" 2>/dev/null || true
+		wait "$UNREAPED_PARENT_PID" 2>/dev/null || true
+	fi
 	if [ -n "$HOLDER_PID" ]; then
 		kill "$HOLDER_PID" 2>/dev/null || true
 		wait "$HOLDER_PID" 2>/dev/null || true
@@ -109,6 +115,36 @@ if ! "$ROOT/client" "$SERVER_PID" "reservation recovered" 2>>"$CLIENT_ERR"; then
 	exit 1
 fi
 
+: >"$UNREAPED_READY"
+"$ROOT/tests/unreaped_exec" "$ROOT/tests/session_sender" "$SERVER_PID" \
+	>"$UNREAPED_READY" &
+UNREAPED_PARENT_PID=$!
+tries=0
+while [ "$tries" -lt 50 ] && [ ! -s "$UNREAPED_READY" ]; do
+	if ! kill -0 "$UNREAPED_PARENT_PID" 2>/dev/null; then
+		printf 'unreaped owner helper exited before readiness\n' >&2
+		exit 1
+	fi
+	tries=$((tries + 1))
+	sleep 0.1
+done
+UNREAPED_OWNER_PID=$(sed -n '1p' "$UNREAPED_READY")
+case "$UNREAPED_OWNER_PID" in
+	''|*[!0-9]*)
+		printf 'unreaped owner helper did not publish a pid\n' >&2
+		exit 1
+		;;
+esac
+kill -0 "$UNREAPED_OWNER_PID"
+if ! "$ROOT/client" "$SERVER_PID" "unreaped recovered" 2>>"$CLIENT_ERR"; then
+	printf 'server did not recover while the exited owner was unreaped\n' >&2
+	cat "$CLIENT_ERR" >&2
+	exit 1
+fi
+kill "$UNREAPED_PARENT_PID"
+wait "$UNREAPED_PARENT_PID"
+UNREAPED_PARENT_PID=
+
 masked_status=0
 "$ROOT/tests/masked_exec" "$ROOT/client" "$SERVER_PID" "masked ack" \
 	2>>"$CLIENT_ERR" || masked_status=$?
@@ -135,6 +171,8 @@ fi
 	printf 'X\n'
 	printf 'line recovered\n'
 	printf 'reservation recovered\n'
+	printf 'X\n'
+	printf 'unreaped recovered\n'
 	printf 'masked ack\n'
 } >"$EXPECTED"
 
diff --git a/tests/unreaped_exec.c b/tests/unreaped_exec.c
new file mode 100644
index 0000000..c63c595
--- /dev/null
+++ b/tests/unreaped_exec.c
@@ -0,0 +1,102 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "minitalk.h"
+
+#include <errno.h>
+#include <signal.h>
+#include <sys/stat.h>
+#include <sys/wait.h>
+#include <time.h>
+#include <unistd.h>
+
+static volatile sig_atomic_t	g_stop;
+
+static void	handle_stop(int signal)
+{
+	(void)signal;
+	g_stop = 1;
+}
+
+static int	install_stop_handlers(void)
+{
+	struct sigaction	action;
+
+	action.sa_handler = handle_stop;
+	sigemptyset(&action.sa_mask);
+	action.sa_flags = 0;
+	if (sigaction(SIGHUP, &action, NULL) == -1
+		|| sigaction(SIGINT, &action, NULL) == -1
+		|| sigaction(SIGTERM, &action, NULL) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	child_exited_unreaped(pid_t child)
+{
+	siginfo_t	info;
+
+	info.si_pid = 0;
+	if (waitid(P_PID, (id_t)child, &info, WEXITED | WNOHANG | WNOWAIT) == -1)
+		return (-1);
+	return (info.si_pid == child);
+}
+
+static int	wait_for_unreaped_child(pid_t child, const char *client_path)
+{
+	struct timespec	pause_time;
+	struct stat		info;
+	int				tries;
+	int				status;
+
+	tries = 0;
+	while (tries < 50)
+	{
+		status = child_exited_unreaped(child);
+		if (status == -1)
+			return (-1);
+		if (status == 1 && lstat(client_path, &info) == -1 && errno == ENOENT)
+			return (0);
+		pause_time.tv_sec = 0;
+		pause_time.tv_nsec = 100000000L;
+		while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+			;
+		tries++;
+	}
+	return (-1);
+}
+
+int	main(int argc, char **argv)
+{
+	char	client_path[MT_RESPONSE_PATH_SIZE];
+	char	*child_argv[4];
+	pid_t	child;
+
+	if (argc != 3 || install_stop_handlers() == -1)
+		return (2);
+	child = fork();
+	if (child == -1)
+		return (1);
+	if (child == 0)
+	{
+		child_argv[0] = argv[1];
+		child_argv[1] = argv[2];
+		child_argv[2] = "partial";
+		child_argv[3] = NULL;
+		execv(argv[1], child_argv);
+		_exit(127);
+	}
+	if (mt_response_path(client_path, sizeof(client_path), "client", child) == -1
+		|| wait_for_unreaped_child(child, client_path) == -1)
+	{
+		kill(child, SIGKILL);
+		waitpid(child, NULL, 0);
+		return (1);
+	}
+	mt_putnbr_fd(child, STDOUT_FILENO);
+	mt_write_all(STDOUT_FILENO, "\n", 1);
+	while (!g_stop)
+		pause();
+	while (waitpid(child, NULL, 0) == -1 && errno == EINTR)
+		;
+	return (0);
+}
