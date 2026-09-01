# Unix 데이터그램 엔드포인트의 안전한 수명

## `feat(runtime): 안전한 응답 endpoint 경로 생성`

diff --git a/Makefile b/Makefile
index 0ea8eea..ef06401 100644
--- a/Makefile
+++ b/Makefile
@@ -8,7 +8,7 @@ CFLAGS := -Wall -Wextra -Werror -Iinclude
 RM := rm -rf
 OBJ_DIR := obj
 
-COMMON_SRC := src/write_utils.c src/parse_pid.c
+COMMON_SRC := src/write_utils.c src/parse_pid.c src/response_channel.c
 SERVER_SRC := src/server.c $(COMMON_SRC)
 CLIENT_SRC := src/client.c $(COMMON_SRC)
 
diff --git a/include/minitalk.h b/include/minitalk.h
index 777b445..defeeb8 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -41,5 +41,8 @@ void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
 size_t	mt_strlen(const char *text);
 int		mt_parse_pid(const char *text, pid_t *pid);
+int		mt_runtime_dir(char *buffer, size_t size);
+int		mt_response_path(char *buffer, size_t size, const char *role,
+			pid_t pid);
 
 #endif
diff --git a/src/response_channel.c b/src/response_channel.c
new file mode 100644
index 0000000..6085926
--- /dev/null
+++ b/src/response_channel.c
@@ -0,0 +1,69 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "minitalk.h"
+
+#include <errno.h>
+#include <stdio.h>
+#include <string.h>
+#include <sys/stat.h>
+#include <unistd.h>
+
+static int	validate_runtime_dir(const char *path)
+{
+	struct stat	info;
+
+	if (lstat(path, &info) == -1)
+		return (-1);
+	if (!S_ISDIR(info.st_mode) || info.st_uid != getuid()
+		|| (info.st_mode & 077) != 0)
+	{
+		errno = EACCES;
+		return (-1);
+	}
+	return (0);
+}
+
+int	mt_runtime_dir(char *buffer, size_t size)
+{
+	int	length;
+
+	if (buffer == NULL || size == 0)
+	{
+		errno = EINVAL;
+		return (-1);
+	}
+	length = snprintf(buffer, size, "/tmp/signal-message-bus-%lu",
+		(unsigned long)getuid());
+	if (length < 0 || (size_t)length >= size)
+	{
+		errno = ENAMETOOLONG;
+		return (-1);
+	}
+	if (mkdir(buffer, 0700) == -1 && errno != EEXIST)
+		return (-1);
+	return (validate_runtime_dir(buffer));
+}
+
+int	mt_response_path(char *buffer, size_t size, const char *role, pid_t pid)
+{
+	char	directory[MT_RESPONSE_PATH_SIZE];
+	int		length;
+
+	if (buffer == NULL || role == NULL || pid <= 1
+		|| mt_runtime_dir(directory, sizeof(directory)) == -1)
+		return (-1);
+	if (strcmp(role, "server") != 0 && strcmp(role, "client") != 0
+		&& strcmp(role, "forger") != 0)
+	{
+		errno = EINVAL;
+		return (-1);
+	}
+	length = snprintf(buffer, size, "%s/%s-%ld.sock", directory, role,
+		(long)pid);
+	if (length < 0 || (size_t)length >= size)
+	{
+		errno = ENAMETOOLONG;
+		return (-1);
+	}
+	return (0);
+}


## `feat(client): datagram 응답 endpoint 수명주기 관리`

diff --git a/src/client.c b/src/client.c
index 8f3aef7..fb1fea8 100644
--- a/src/client.c
+++ b/src/client.c
@@ -1,6 +1,14 @@
+#define _POSIX_C_SOURCE 200809L
+
 #include "minitalk.h"
 
 #include <errno.h>
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <sys/socket.h>
+#include <sys/stat.h>
+#include <sys/un.h>
 #include <time.h>
 #include <unistd.h>
 
@@ -11,6 +19,18 @@
 static volatile sig_atomic_t	g_ack_received;
 static volatile sig_atomic_t	g_timed_out;
 static volatile sig_atomic_t	g_rejected;
+static int						g_response_socket = -1;
+static char						g_client_path[MT_RESPONSE_PATH_SIZE];
+
+static void	cleanup_response_socket(void)
+{
+	if (g_response_socket != -1)
+		close(g_response_socket);
+	g_response_socket = -1;
+	if (g_client_path[0] != '\0')
+		unlink(g_client_path);
+	g_client_path[0] = '\0';
+}
 
 static void	wait_signal_gap(void)
 {
@@ -48,6 +68,62 @@ static int	install_client_handlers(void)
 	return (0);
 }
 
+static int	set_nonblocking_close_on_exec(int fd)
+{
+	int	flags;
+
+	flags = fcntl(fd, F_GETFL);
+	if (flags == -1 || fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1)
+		return (-1);
+	flags = fcntl(fd, F_GETFD);
+	if (flags == -1 || fcntl(fd, F_SETFD, flags | FD_CLOEXEC) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	remove_stale_socket(const char *path)
+{
+	struct stat	info;
+
+	if (lstat(path, &info) == -1)
+	{
+		if (errno == ENOENT)
+			return (0);
+		return (-1);
+	}
+	if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid())
+	{
+		errno = EACCES;
+		return (-1);
+	}
+	return (unlink(path));
+}
+
+static int	bind_client_socket(void)
+{
+	struct sockaddr_un	address;
+
+	if (mt_response_path(g_client_path, sizeof(g_client_path), "client",
+			getpid()) == -1 || remove_stale_socket(g_client_path) == -1)
+		return (-1);
+	g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (g_response_socket == -1
+		|| set_nonblocking_close_on_exec(g_response_socket) == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	if (mt_strlen(g_client_path) >= sizeof(address.sun_path))
+	{
+		errno = ENAMETOOLONG;
+		return (-1);
+	}
+	memcpy(address.sun_path, g_client_path, mt_strlen(g_client_path) + 1);
+	if (bind(g_response_socket, (struct sockaddr *)&address,
+			sizeof(address)) == -1)
+		return (-1);
+	return (0);
+}
+
 static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 {
 	int	signal;
@@ -126,6 +202,13 @@ int	main(int argc, char **argv)
 			STDERR_FILENO);
 		return (1);
 	}
+	if (bind_client_socket() == -1 || atexit(cleanup_response_socket) != 0)
+	{
+		cleanup_response_socket();
+		mt_putstr_fd("client: failed to create response channel\n",
+			STDERR_FILENO);
+		return (1);
+	}
 	sigemptyset(&blocked);
 	sigaddset(&blocked, MT_ACK_SIGNAL);
 	sigaddset(&blocked, MT_NACK_SIGNAL);


## `feat(server): datagram 응답 endpoint 수명주기 관리`

diff --git a/src/server.c b/src/server.c
index ae178e0..23114a6 100644
--- a/src/server.c
+++ b/src/server.c
@@ -1,12 +1,34 @@
+#define _POSIX_C_SOURCE 200809L
+
 #include "minitalk.h"
 
 #include <errno.h>
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <sys/socket.h>
+#include <sys/stat.h>
+#include <sys/un.h>
 #include <unistd.h>
 
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 static volatile sig_atomic_t	g_client_pid;
 static volatile sig_atomic_t	g_line_started;
+static int						g_response_socket = -1;
+static int						g_server_bound;
+static char						g_server_path[MT_RESPONSE_PATH_SIZE];
+
+static void	cleanup_server(void)
+{
+	if (g_response_socket != -1)
+		close(g_response_socket);
+	g_response_socket = -1;
+	if (g_server_bound && g_server_path[0] != '\0')
+		unlink(g_server_path);
+	g_server_bound = 0;
+	g_server_path[0] = '\0';
+}
 
 static void	reset_session(int close_partial_line)
 {
@@ -73,6 +95,63 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	errno = saved_errno;
 }
 
+static int	set_nonblocking_close_on_exec(int fd)
+{
+	int	flags;
+
+	flags = fcntl(fd, F_GETFL);
+	if (flags == -1 || fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1)
+		return (-1);
+	flags = fcntl(fd, F_GETFD);
+	if (flags == -1 || fcntl(fd, F_SETFD, flags | FD_CLOEXEC) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	remove_stale_socket(const char *path)
+{
+	struct stat	info;
+
+	if (lstat(path, &info) == -1)
+	{
+		if (errno == ENOENT)
+			return (0);
+		return (-1);
+	}
+	if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid())
+	{
+		errno = EACCES;
+		return (-1);
+	}
+	return (unlink(path));
+}
+
+static int	prepare_response_channel(void)
+{
+	struct sockaddr_un	address;
+
+	if (mt_response_path(g_server_path, sizeof(g_server_path), "server",
+			getpid()) == -1 || remove_stale_socket(g_server_path) == -1)
+		return (-1);
+	g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (g_response_socket == -1
+		|| set_nonblocking_close_on_exec(g_response_socket) == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	if (mt_strlen(g_server_path) >= sizeof(address.sun_path))
+	{
+		errno = ENAMETOOLONG;
+		return (-1);
+	}
+	memcpy(address.sun_path, g_server_path, mt_strlen(g_server_path) + 1);
+	if (bind(g_response_socket, (struct sockaddr *)&address,
+			sizeof(address)) == -1)
+		return (-1);
+	g_server_bound = 1;
+	return (0);
+}
+
 static int	install_signal_handlers(void)
 {
 	struct sigaction	action;
@@ -91,6 +170,13 @@ static int	install_signal_handlers(void)
 
 int	main(void)
 {
+	if (prepare_response_channel() == -1 || atexit(cleanup_server) != 0)
+	{
+		cleanup_server();
+		mt_putstr_fd("server: failed to create response channel\n",
+			STDERR_FILENO);
+		return (1);
+	}
 	if (install_signal_handlers() == -1)
 	{
 		mt_putstr_fd("server: failed to install signal handlers\n", STDERR_FILENO);


## `fix(client): bind한 응답 경로만 정리`

diff --git a/src/client.c b/src/client.c
index 0423605..8f0a09c 100644
--- a/src/client.c
+++ b/src/client.c
@@ -19,14 +19,16 @@
 
 static int	g_response_socket = -1;
 static char	g_client_path[MT_RESPONSE_PATH_SIZE];
+static int	g_client_bound;
 
 static void	cleanup_response_socket(void)
 {
 	if (g_response_socket != -1)
 		close(g_response_socket);
 	g_response_socket = -1;
-	if (g_client_path[0] != '\0')
+	if (g_client_bound && g_client_path[0] != '\0')
 		unlink(g_client_path);
+	g_client_bound = 0;
 	g_client_path[0] = '\0';
 }
 
@@ -98,6 +100,7 @@ static int	bind_client_socket(void)
 	if (bind(g_response_socket, (struct sockaddr *)&address,
 			sizeof(address)) == -1)
 		return (-1);
+	g_client_bound = 1;
 	return (0);
 }
 


## `test(runtime): stale 응답 endpoint 처리 검증`

diff --git a/.gitignore b/.gitignore
index bb944cc..219ba30 100644
--- a/.gitignore
+++ b/.gitignore
@@ -8,3 +8,5 @@ tests/masked_exec
 tests/parse_pid_test
 tests/response_server
 tests/session_sender
+tests/stale_exec
+tests/stale_server_exec
diff --git a/Makefile b/Makefile
index f791ff0..cf635a6 100644
--- a/Makefile
+++ b/Makefile
@@ -5,6 +5,8 @@ NAME_MASKED_EXEC := tests/masked_exec
 NAME_RESPONSE_SERVER := tests/response_server
 NAME_PARSE_PID_TEST := tests/parse_pid_test
 NAME_FAULT_SERVER := tests/fault_server
+NAME_STALE_EXEC := tests/stale_exec
+NAME_STALE_SERVER_EXEC := tests/stale_server_exec
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
@@ -27,6 +29,8 @@ PARSE_PID_TEST_OBJ := $(OBJ_DIR)/tests/parse_pid_test.o \
 	$(OBJ_DIR)/src/parse_pid.o
 FAULT_SERVER_OBJ := $(SERVER_SRC:%.c=$(FAULT_OBJ_DIR)/%.o) \
 	$(OBJ_DIR)/tests/write_fault.o
+STALE_EXEC_OBJ := $(OBJ_DIR)/tests/stale_exec.o $(COMMON_OBJ)
+STALE_SERVER_EXEC_OBJ := $(OBJ_DIR)/tests/stale_server_exec.o $(COMMON_OBJ)
 
 .PHONY: all clean fclean re test
 
@@ -53,6 +57,12 @@ $(NAME_PARSE_PID_TEST): $(PARSE_PID_TEST_OBJ)
 $(NAME_FAULT_SERVER): $(FAULT_SERVER_OBJ)
 	$(CC) $(CFLAGS) $(FAULT_SERVER_OBJ) -o $@
 
+$(NAME_STALE_EXEC): $(STALE_EXEC_OBJ)
+	$(CC) $(CFLAGS) $(STALE_EXEC_OBJ) -o $@
+
+$(NAME_STALE_SERVER_EXEC): $(STALE_SERVER_EXEC_OBJ)
+	$(CC) $(CFLAGS) $(STALE_SERVER_EXEC_OBJ) -o $@
+
 $(OBJ_DIR)/%.o: %.c include/minitalk.h
 	mkdir -p $(dir $@)
 	$(CC) $(CFLAGS) -c $< -o $@
@@ -67,14 +77,16 @@ clean:
 fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT) $(NAME_SESSION_SENDER) \
 		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) $(NAME_PARSE_PID_TEST) \
-		$(NAME_FAULT_SERVER)
+		$(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) $(NAME_STALE_SERVER_EXEC)
 
 re: fclean all
 
 test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) \
-		$(NAME_PARSE_PID_TEST) $(NAME_FAULT_SERVER)
+		$(NAME_PARSE_PID_TEST) $(NAME_FAULT_SERVER) $(NAME_STALE_EXEC) \
+		$(NAME_STALE_SERVER_EXEC)
 	./$(NAME_PARSE_PID_TEST)
 	sh tests/smoke.sh
 	sh tests/session_ownership.sh
 	sh tests/response_validation.sh
 	sh tests/output_failure.sh
+	sh tests/protocol_regressions.sh
diff --git a/tests/protocol_regressions.sh b/tests/protocol_regressions.sh
new file mode 100755
index 0000000..e2c936c
--- /dev/null
+++ b/tests/protocol_regressions.sh
@@ -0,0 +1,102 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/minitalk-protocol.XXXXXX")
+OUT="$TEST_TMP/server.out"
+ERR="$TEST_TMP/server.err"
+SERVER_PID=
+SERVER_PATH=
+UNRELATED_PID=
+
+cleanup()
+{
+	if [ -n "$UNRELATED_PID" ]; then
+		kill "$UNRELATED_PID" 2>/dev/null || true
+		wait "$UNRELATED_PID" 2>/dev/null || true
+	fi
+	if [ -n "$SERVER_PID" ]; then
+		kill "$SERVER_PID" 2>/dev/null || true
+		wait "$SERVER_PID" 2>/dev/null || true
+	fi
+	if [ -n "$SERVER_PATH" ]; then
+		rm -f "$SERVER_PATH"
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
+	file=$1
+	tries=0
+	while [ "$tries" -lt 50 ] && ! grep -qx "$SERVER_PID" "$file" 2>/dev/null; do
+		if ! kill -0 "$SERVER_PID" 2>/dev/null; then
+			printf 'server exited before becoming ready\n' >&2
+			exit 1
+		fi
+		tries=$((tries + 1))
+		sleep 0.1
+	done
+	grep -qx "$SERVER_PID" "$file"
+}
+
+UNRELATED_ERR="$TEST_TMP/unrelated.err"
+sleep 30 &
+UNRELATED_PID=$!
+if "$ROOT/client" "$UNRELATED_PID" unrelated 2>"$UNRELATED_ERR"; then
+	printf 'client accepted a process without an active server\n' >&2
+	exit 1
+fi
+grep -qx 'client: invalid server pid' "$UNRELATED_ERR"
+kill -0 "$UNRELATED_PID"
+kill "$UNRELATED_PID"
+wait "$UNRELATED_PID" 2>/dev/null || true
+UNRELATED_PID=
+
+STALE_UNRELATED_ERR="$TEST_TMP/stale-unrelated.err"
+"$ROOT/tests/stale_server_exec" "$ROOT/client" unrelated \
+	2>"$STALE_UNRELATED_ERR"
+grep -qx 'client: failed to send signal' "$STALE_UNRELATED_ERR"
+
+STALE_SERVER_OUT="$TEST_TMP/stale-server.out"
+STALE_SERVER_ERR="$TEST_TMP/stale-server.err"
+"$ROOT/tests/stale_server_exec" "$ROOT/server" \
+	>"$STALE_SERVER_OUT" 2>"$STALE_SERVER_ERR"
+[ ! -s "$STALE_SERVER_OUT" ]
+grep -qx 'server: failed to create response channel' "$STALE_SERVER_ERR"
+
+RUNTIME_DIR="/tmp/signal-message-bus-$(id -u)"
+"$ROOT/server" >"$OUT" 2>"$ERR" &
+SERVER_PID=$!
+SERVER_PATH="$RUNTIME_DIR/server-$SERVER_PID.sock"
+wait_ready "$OUT"
+if [ "$(uname -s)" = Darwin ]; then
+	runtime_mode=$(stat -f '%Lp' "$RUNTIME_DIR")
+else
+	runtime_mode=$(stat -c '%a' "$RUNTIME_DIR")
+fi
+[ "$runtime_mode" = 700 ]
+
+"$ROOT/tests/stale_exec" "$ROOT/client" "$SERVER_PID" stale socket \
+	2>"$TEST_TMP/stale.err"
+[ ! -s "$TEST_TMP/stale.err" ]
+"$ROOT/tests/stale_exec" "$ROOT/client" "$SERVER_PID" blocked file \
+	2>"$TEST_TMP/file.err"
+grep -qx 'client: failed to create response channel' "$TEST_TMP/file.err"
+[ ! -s "$ERR" ]
+{
+	sed -n '1p' "$OUT"
+	printf 'stale\n'
+} >"$TEST_TMP/expected"
+diff -u "$TEST_TMP/expected" "$OUT"
+
+kill "$SERVER_PID"
+wait "$SERVER_PID" 2>/dev/null || true
+SERVER_PID=
+rm -f "$SERVER_PATH"
+SERVER_PATH=
diff --git a/tests/stale_exec.c b/tests/stale_exec.c
new file mode 100644
index 0000000..b54987d
--- /dev/null
+++ b/tests/stale_exec.c
@@ -0,0 +1,114 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "minitalk.h"
+
+#include <errno.h>
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <sys/socket.h>
+#include <sys/stat.h>
+#include <sys/un.h>
+#include <sys/wait.h>
+#include <unistd.h>
+
+static int	create_stale_socket(const char *path)
+{
+	struct sockaddr_un	address;
+	int					socket_fd;
+
+	socket_fd = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (socket_fd == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, path, mt_strlen(path) + 1);
+	if (bind(socket_fd, (struct sockaddr *)&address, sizeof(address)) == -1)
+	{
+		close(socket_fd);
+		return (-1);
+	}
+	return (close(socket_fd));
+}
+
+static int	create_stale_entry(const char *path, const char *mode)
+{
+	int	fd;
+
+	unlink(path);
+	if (strcmp(mode, "socket") == 0)
+		return (create_stale_socket(path));
+	if (strcmp(mode, "file") != 0)
+		return (-1);
+	fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0600);
+	if (fd == -1)
+		return (-1);
+	return (close(fd));
+}
+
+static int	child_status(pid_t child)
+{
+	int	status;
+
+	while (waitpid(child, &status, 0) == -1)
+	{
+		if (errno != EINTR)
+			return (-1);
+	}
+	if (WIFEXITED(status))
+		return (WEXITSTATUS(status));
+	return (-1);
+}
+
+int	main(int argc, char **argv)
+{
+	int		gate[2];
+	pid_t	child;
+	char	path[MT_RESPONSE_PATH_SIZE];
+	char	token;
+	char	*child_argv[4];
+	int		status;
+
+	if (argc != 5 || pipe(gate) == -1)
+		return (2);
+	child = fork();
+	if (child == -1)
+		return (1);
+	if (child == 0)
+	{
+		close(gate[1]);
+		if (read(gate[0], &token, 1) != 1)
+			_exit(126);
+		close(gate[0]);
+		child_argv[0] = argv[1];
+		child_argv[1] = argv[2];
+		child_argv[2] = argv[3];
+		child_argv[3] = NULL;
+		execv(argv[1], child_argv);
+		_exit(127);
+	}
+	close(gate[0]);
+	if (mt_response_path(path, sizeof(path), "client", child) == -1
+		|| create_stale_entry(path, argv[4]) == -1
+		|| write(gate[1], "x", 1) != 1)
+	{
+		close(gate[1]);
+		kill(child, SIGKILL);
+		waitpid(child, NULL, 0);
+		unlink(path);
+		return (1);
+	}
+	close(gate[1]);
+	status = child_status(child);
+	if (strcmp(argv[4], "socket") == 0)
+	{
+		if (status != 0 || lstat(path, &(struct stat){0}) == 0
+			|| errno != ENOENT)
+			return (1);
+		return (0);
+	}
+	if (status == 0 || lstat(path, &(struct stat){0}) == -1)
+		return (1);
+	unlink(path);
+	return (0);
+}
diff --git a/tests/stale_server_exec.c b/tests/stale_server_exec.c
new file mode 100644
index 0000000..d44ccae
--- /dev/null
+++ b/tests/stale_server_exec.c
@@ -0,0 +1,208 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "minitalk.h"
+
+#include <errno.h>
+#include <fcntl.h>
+#include <signal.h>
+#include <stdio.h>
+#include <string.h>
+#include <sys/socket.h>
+#include <sys/un.h>
+#include <stdlib.h>
+#include <sys/stat.h>
+#include <sys/wait.h>
+#include <time.h>
+#include <unistd.h>
+
+static int	wait_for_child(pid_t child)
+{
+	struct timespec	pause_time;
+	int				status;
+	int				tries;
+	pid_t			result;
+
+	tries = 0;
+	while (tries < 50)
+	{
+		result = waitpid(child, &status, WNOHANG);
+		if (result == child)
+		{
+			if (WIFEXITED(status))
+				return (WEXITSTATUS(status));
+			return (-1);
+		}
+		if (result == -1 && errno != EINTR)
+			break ;
+		pause_time.tv_sec = 0;
+		pause_time.tv_nsec = 100000000L;
+		while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+			;
+		tries++;
+	}
+	kill(child, SIGKILL);
+	waitpid(child, NULL, 0);
+	return (-1);
+}
+
+static int bind_stale_socket(const char *path)
+{
+	struct sockaddr_un address;
+	int socket_fd;
+
+	socket_fd = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (socket_fd == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, path, mt_strlen(path) + 1);
+	if (bind(socket_fd, (struct sockaddr *)&address, sizeof(address)) == -1)
+	{
+		close(socket_fd);
+		return (-1);
+	}
+	return (close(socket_fd));
+}
+
+static int run_client(const char *client_path, pid_t target)
+{
+	char pid_text[32];
+	char *client_argv[4];
+	pid_t client;
+	int status;
+
+	if (snprintf(pid_text, sizeof(pid_text), "%ld", (long)target) < 0)
+		return (-1);
+	client = fork();
+	if (client == -1)
+		return (-1);
+	if (client == 0)
+	{
+		client_argv[0] = (char *)client_path;
+		client_argv[1] = pid_text;
+		client_argv[2] = "unrelated";
+		client_argv[3] = NULL;
+		execv(client_path, client_argv);
+		_exit(127);
+	}
+	if (waitpid(client, &status, 0) != client || !WIFEXITED(status))
+		return (-1);
+	return (WEXITSTATUS(status));
+}
+
+static int test_unrelated_process(const char *client_path)
+{
+	struct timespec pause_time;
+	struct stat info;
+	char path[MT_RESPONSE_PATH_SIZE];
+	char token;
+	char *sleep_argv[3];
+	pid_t child;
+	int gate[2];
+	int status;
+
+	if (pipe(gate) == -1)
+		return (1);
+	child = fork();
+	if (child == -1)
+		return (1);
+	if (child == 0)
+	{
+		close(gate[1]);
+		if (read(gate[0], &token, 1) != 1)
+			_exit(126);
+		close(gate[0]);
+		sleep_argv[0] = "sleep";
+		sleep_argv[1] = "30";
+		sleep_argv[2] = NULL;
+		execv("/bin/sleep", sleep_argv);
+		_exit(127);
+	}
+	close(gate[0]);
+	if (mt_response_path(path, sizeof(path), "server", child) == -1)
+		return (1);
+	unlink(path);
+	if (bind_stale_socket(path) == -1 || write(gate[1], "x", 1) != 1)
+	{
+		close(gate[1]);
+		kill(child, SIGKILL);
+		waitpid(child, NULL, 0);
+		unlink(path);
+		return (1);
+	}
+	close(gate[1]);
+	pause_time.tv_sec = 0;
+	pause_time.tv_nsec = 100000000L;
+	while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+		;
+	status = run_client(client_path, child);
+	if (status != 1 || kill(child, 0) == -1 || lstat(path, &info) == -1
+		|| !S_ISSOCK(info.st_mode) || info.st_uid != getuid())
+	{
+		kill(child, SIGKILL);
+		waitpid(child, NULL, 0);
+		unlink(path);
+		return (1);
+	}
+	kill(child, SIGTERM);
+	waitpid(child, NULL, 0);
+	unlink(path);
+	return (0);
+}
+
+int	main(int argc, char **argv)
+{
+	struct stat	info;
+	int			gate[2];
+	int			file_fd;
+	int			status;
+	pid_t		child;
+	char		path[MT_RESPONSE_PATH_SIZE];
+	char		token;
+	char		*child_argv[2];
+
+	if (argc == 3 && strcmp(argv[2], "unrelated") == 0)
+		return (test_unrelated_process(argv[1]));
+	if (argc != 2 || pipe(gate) == -1)
+		return (2);
+	child = fork();
+	if (child == -1)
+		return (1);
+	if (child == 0)
+	{
+		close(gate[1]);
+		if (read(gate[0], &token, 1) != 1)
+			_exit(126);
+		close(gate[0]);
+		child_argv[0] = argv[1];
+		child_argv[1] = NULL;
+		execv(argv[1], child_argv);
+		_exit(127);
+	}
+	close(gate[0]);
+	if (mt_response_path(path, sizeof(path), "server", child) == -1)
+		return (1);
+	unlink(path);
+	file_fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0600);
+	if (file_fd == -1 || close(file_fd) == -1
+		|| write(gate[1], "x", 1) != 1)
+	{
+		close(gate[1]);
+		kill(child, SIGKILL);
+		waitpid(child, NULL, 0);
+		unlink(path);
+		return (1);
+	}
+	close(gate[1]);
+	status = wait_for_child(child);
+	if (lstat(path, &info) == -1 || !S_ISREG(info.st_mode)
+		|| info.st_uid != getuid())
+	{
+		unlink(path);
+		return (1);
+	}
+	unlink(path);
+	if (status != 1)
+		return (1);
+	return (0);
+}


