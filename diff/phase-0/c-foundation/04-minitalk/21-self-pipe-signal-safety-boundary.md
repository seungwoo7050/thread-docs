# Self-Pipe 기반 시그널 안전 경계

## `refactor(server): 비트 상태 전이 로직 추출`

diff --git a/src/server.c b/src/server.c
index 188fce2..92bc538 100644
--- a/src/server.c
+++ b/src/server.c
@@ -4,6 +4,7 @@
 
 #include <errno.h>
 #include <fcntl.h>
+#include <limits.h>
 #include <stdlib.h>
 #include <string.h>
 #include <sys/select.h>
@@ -20,6 +21,14 @@ typedef struct s_response_request
 	int32_t		status;
 }	t_response_request;
 
+typedef struct s_bit_event
+{
+	pid_t	sender;
+	int		signal;
+}	t_bit_event;
+
+typedef char	t_event_must_fit_pipe_buf[(sizeof(t_bit_event) <= PIPE_BUF) * 2 - 1];
+
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 static volatile sig_atomic_t	g_client_pid;
@@ -87,27 +96,18 @@ static void	queue_response(pid_t client_pid, uint32_t kind,
 		g_response_overflow = 1;
 }
 
-static void	handle_bit(int signal, siginfo_t *info, void *context)
+static void	process_bit(const t_bit_event *event)
 {
 	unsigned char	output;
 	uint32_t		sequence;
-	int				saved_errno;
 
-	saved_errno = errno;
-	(void)context;
-	if (info == NULL || info->si_pid <= 0)
-	{
-		errno = saved_errno;
+	if (event->sender <= 0)
 		return ;
-	}
-	if (g_client_pid == 0 || g_client_pid != info->si_pid)
-	{
-		errno = saved_errno;
+	if (g_client_pid == 0 || g_client_pid != event->sender)
 		return ;
-	}
 	sequence = (uint32_t)g_sequence;
 	g_current_byte <<= 1;
-	if (signal == MT_ONE_SIGNAL)
+	if (event->signal == MT_ONE_SIGNAL)
 		g_current_byte |= 1;
 	g_received_bits++;
 	if (g_received_bits == 8)
@@ -119,7 +119,21 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	}
 	if (g_client_pid != 0)
 		g_sequence++;
-	queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
+	queue_response(event->sender, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
+}
+
+static void	handle_bit(int signal, siginfo_t *info, void *context)
+{
+	t_bit_event	event;
+	int				saved_errno;
+
+	saved_errno = errno;
+	(void)context;
+	event.sender = 0;
+	if (info != NULL)
+		event.sender = info->si_pid;
+	event.signal = signal;
+	process_bit(&event);
 	errno = saved_errno;
 }
 


## `refactor(server): signal 처리를 self-pipe event loop로 제한`

diff --git a/src/server.c b/src/server.c
index 92bc538..c324517 100644
--- a/src/server.c
+++ b/src/server.c
@@ -13,14 +13,6 @@
 #include <sys/un.h>
 #include <unistd.h>
 
-typedef struct s_response_request
-{
-	pid_t		client_pid;
-	uint32_t	kind;
-	uint32_t	token;
-	int32_t		status;
-}	t_response_request;
-
 typedef struct s_bit_event
 {
 	pid_t	sender;
@@ -29,27 +21,27 @@ typedef struct s_bit_event
 
 typedef char	t_event_must_fit_pipe_buf[(sizeof(t_bit_event) <= PIPE_BUF) * 2 - 1];
 
-static volatile sig_atomic_t	g_current_byte;
-static volatile sig_atomic_t	g_received_bits;
-static volatile sig_atomic_t	g_client_pid;
-static volatile sig_atomic_t	g_line_started;
-static volatile sig_atomic_t	g_response_overflow;
-static volatile sig_atomic_t	g_sequence;
-static int					g_response_pipe[2] = {-1, -1};
-static int					g_response_socket = -1;
-static int					g_server_bound;
-static char					g_server_path[MT_RESPONSE_PATH_SIZE];
+static unsigned char			g_current_byte;
+static unsigned int			g_received_bits;
+static pid_t					g_client_pid;
+static int						g_line_started;
+static uint32_t				g_sequence;
+static volatile sig_atomic_t	g_event_overflow;
+static int						g_event_pipe[2] = {-1, -1};
+static int						g_response_socket = -1;
+static int						g_server_bound;
+static char						g_server_path[MT_RESPONSE_PATH_SIZE];
 
 static void	cleanup_server(void)
 {
-	if (g_response_pipe[0] != -1)
-		close(g_response_pipe[0]);
-	if (g_response_pipe[1] != -1)
-		close(g_response_pipe[1]);
+	if (g_event_pipe[0] != -1)
+		close(g_event_pipe[0]);
+	if (g_event_pipe[1] != -1)
+		close(g_event_pipe[1]);
 	if (g_response_socket != -1)
 		close(g_response_socket);
-	g_response_pipe[0] = -1;
-	g_response_pipe[1] = -1;
+	g_event_pipe[0] = -1;
+	g_event_pipe[1] = -1;
 	g_response_socket = -1;
 	if (g_server_bound && g_server_path[0] != '\0')
 		unlink(g_server_path);
@@ -82,46 +74,6 @@ static void	flush_byte(unsigned char output)
 	}
 }
 
-static void	queue_response(pid_t client_pid, uint32_t kind,
-		uint32_t token, int status)
-{
-	t_response_request	request;
-
-	request.client_pid = client_pid;
-	request.kind = kind;
-	request.token = token;
-	request.status = status;
-	if (write(g_response_pipe[1], &request, sizeof(request))
-		!= (ssize_t)sizeof(request))
-		g_response_overflow = 1;
-}
-
-static void	process_bit(const t_bit_event *event)
-{
-	unsigned char	output;
-	uint32_t		sequence;
-
-	if (event->sender <= 0)
-		return ;
-	if (g_client_pid == 0 || g_client_pid != event->sender)
-		return ;
-	sequence = (uint32_t)g_sequence;
-	g_current_byte <<= 1;
-	if (event->signal == MT_ONE_SIGNAL)
-		g_current_byte |= 1;
-	g_received_bits++;
-	if (g_received_bits == 8)
-	{
-		output = (unsigned char)g_current_byte;
-		flush_byte(output);
-		g_current_byte = 0;
-		g_received_bits = 0;
-	}
-	if (g_client_pid != 0)
-		g_sequence++;
-	queue_response(event->sender, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
-}
-
 static void	handle_bit(int signal, siginfo_t *info, void *context)
 {
 	t_bit_event	event;
@@ -133,7 +85,9 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	if (info != NULL)
 		event.sender = info->si_pid;
 	event.signal = signal;
-	process_bit(&event);
+	if (write(g_event_pipe[1], &event, sizeof(event))
+		!= (ssize_t)sizeof(event))
+		g_event_overflow = 1;
 	errno = saved_errno;
 }
 
@@ -150,6 +104,16 @@ static int	set_nonblocking_close_on_exec(int fd)
 	return (0);
 }
 
+static int	set_close_on_exec(int fd)
+{
+	int	flags;
+
+	flags = fcntl(fd, F_GETFD);
+	if (flags == -1 || fcntl(fd, F_SETFD, flags | FD_CLOEXEC) == -1)
+		return (-1);
+	return (0);
+}
+
 static int	remove_stale_socket(const char *path)
 {
 	struct stat	info;
@@ -172,8 +136,9 @@ static int	prepare_response_channel(void)
 {
 	struct sockaddr_un	address;
 
-	if (pipe(g_response_pipe) == -1
-		|| set_nonblocking_close_on_exec(g_response_pipe[1]) == -1)
+	if (pipe(g_event_pipe) == -1
+		|| set_close_on_exec(g_event_pipe[0]) == -1
+		|| set_nonblocking_close_on_exec(g_event_pipe[1]) == -1)
 		return (-1);
 	if (mt_response_path(g_server_path, sizeof(g_server_path), "server",
 			getpid()) == -1 || remove_stale_socket(g_server_path) == -1)
@@ -201,14 +166,14 @@ static int	install_signal_handlers(void)
 {
 	struct sigaction	action;
 
+	memset(&action, 0, sizeof(action));
 	action.sa_sigaction = handle_bit;
 	sigemptyset(&action.sa_mask);
 	sigaddset(&action.sa_mask, MT_ZERO_SIGNAL);
 	sigaddset(&action.sa_mask, MT_ONE_SIGNAL);
 	action.sa_flags = SA_SIGINFO;
-	if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1)
-		return (-1);
-	if (sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
+	if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1
+		|| sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
 		return (-1);
 	return (0);
 }
@@ -310,7 +275,7 @@ static int	handle_session_request(void)
 	status = MT_RESPONSE_OK;
 	if (g_client_pid != 0 && g_client_pid != request.client_pid)
 	{
-		if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
+		if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
 			reset_session(1);
 		else
 			status = MT_RESPONSE_BUSY;
@@ -326,48 +291,92 @@ static int	handle_session_request(void)
 	return (0);
 }
 
-static int	respond_to_bit(void)
+static int	read_event(t_bit_event *event)
 {
-	t_response_request	request;
-	ssize_t				size;
+	unsigned char	*bytes;
+	size_t			offset;
+	ssize_t			size;
 
-	size = read(g_response_pipe[0], &request, sizeof(request));
-	if (size == -1 && errno == EINTR)
+	bytes = (unsigned char *)event;
+	offset = 0;
+	while (offset < sizeof(*event))
+	{
+		size = read(g_event_pipe[0], bytes + offset, sizeof(*event) - offset);
+		if (size == -1 && errno == EINTR)
+			continue ;
+		if (size <= 0)
+			return (-1);
+		offset += (size_t)size;
+	}
+	return (0);
+}
+
+static int	process_bit(const t_bit_event *event)
+{
+	unsigned char	output;
+	uint32_t		sequence;
+
+	if (event->sender <= 0
+		|| (event->signal != MT_ZERO_SIGNAL && event->signal != MT_ONE_SIGNAL))
 		return (0);
-	if (size != (ssize_t)sizeof(request) || g_response_overflow)
-		return (-1);
-	if (send_response(request.client_pid, request.kind, request.token,
-			request.status) == -1 && request.status == MT_RESPONSE_OK
-		&& request.client_pid == (pid_t)g_client_pid)
-		reset_session(1);
+	if (g_client_pid == 0 || g_client_pid != event->sender)
+		return (0);
+	sequence = g_sequence;
+	g_current_byte <<= 1;
+	if (event->signal == MT_ONE_SIGNAL)
+		g_current_byte |= 1;
+	g_received_bits++;
+	if (g_received_bits == 8)
+	{
+		output = g_current_byte;
+		flush_byte(output);
+		g_current_byte = 0;
+		g_received_bits = 0;
+	}
+	if (g_client_pid != 0)
+		g_sequence++;
+	if (send_response(event->sender, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK) == -1)
+	{
+		if (event->sender == g_client_pid)
+			reset_session(1);
+	}
 	return (0);
 }
 
-static int	run_response_loop(void)
+static int	run_event_loop(void)
 {
-	fd_set	read_set;
-	int		max_fd;
-	int		status;
+	t_bit_event	event;
+	fd_set		read_set;
+	int			max_fd;
+	int			status;
 
-	max_fd = g_response_pipe[0];
+	max_fd = g_event_pipe[0];
 	if (g_response_socket > max_fd)
 		max_fd = g_response_socket;
 	while (1)
 	{
+		if (g_event_overflow)
+		{
+			errno = ENOBUFS;
+			return (-1);
+		}
 		FD_ZERO(&read_set);
-		FD_SET(g_response_pipe[0], &read_set);
+		FD_SET(g_event_pipe[0], &read_set);
 		FD_SET(g_response_socket, &read_set);
 		status = pselect(max_fd + 1, &read_set, NULL, NULL, NULL, NULL);
 		if (status == -1 && errno == EINTR)
 			continue ;
-		if (status == -1 || g_response_overflow)
+		if (status == -1)
 			return (-1);
 		if (FD_ISSET(g_response_socket, &read_set)
 			&& handle_session_request() == -1)
 			return (-1);
-		if (FD_ISSET(g_response_pipe[0], &read_set)
-			&& respond_to_bit() == -1)
-			return (-1);
+		if (FD_ISSET(g_event_pipe[0], &read_set))
+		{
+			if (read_event(&event) == -1 || g_event_overflow
+				|| process_bit(&event) == -1)
+				return (-1);
+		}
 	}
 }
 
@@ -388,9 +397,9 @@ int	main(void)
 	}
 	mt_putnbr_fd(getpid(), STDOUT_FILENO);
 	write(STDOUT_FILENO, "\n", 1);
-	if (run_response_loop() == -1)
+	if (run_event_loop() == -1)
 	{
-		mt_putstr_fd("server: response channel failed\n", STDERR_FILENO);
+		mt_putstr_fd("server: signal event channel failed\n", STDERR_FILENO);
 		return (1);
 	}
 	return (0);


## `fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리`

diff --git a/src/server.c b/src/server.c
index 082a3a2..b519dcf 100644
--- a/src/server.c
+++ b/src/server.c
@@ -178,9 +178,15 @@ static int	install_signal_handlers(void)
 	sigemptyset(&action.sa_mask);
 	sigaddset(&action.sa_mask, MT_ZERO_SIGNAL);
 	sigaddset(&action.sa_mask, MT_ONE_SIGNAL);
+	sigaddset(&action.sa_mask, SIGHUP);
+	sigaddset(&action.sa_mask, SIGINT);
+	sigaddset(&action.sa_mask, SIGTERM);
 	action.sa_flags = SA_SIGINFO;
 	if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1
-		|| sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
+		|| sigaction(MT_ONE_SIGNAL, &action, NULL) == -1
+		|| sigaction(SIGHUP, &action, NULL) == -1
+		|| sigaction(SIGINT, &action, NULL) == -1
+		|| sigaction(SIGTERM, &action, NULL) == -1)
 		return (-1);
 	return (0);
 }
@@ -326,6 +332,9 @@ static int	process_bit(const t_bit_event *event)
 	unsigned char	output;
 	uint32_t		sequence;
 
+	if (event->signal == SIGHUP || event->signal == SIGINT
+		|| event->signal == SIGTERM)
+		return (event->signal);
 	if (event->sender <= 0
 		|| (event->signal != MT_ZERO_SIGNAL && event->signal != MT_ONE_SIGNAL))
 		return (0);
@@ -386,9 +395,11 @@ static int	run_event_loop(void)
 			return (-1);
 		if (FD_ISSET(g_event_pipe[0], &read_set))
 		{
-			if (read_event(&event) == -1 || g_event_overflow
-				|| process_bit(&event) == -1)
+			if (read_event(&event) == -1 || g_event_overflow)
 				return (-1);
+			status = process_bit(&event);
+			if (status != 0)
+				return (status);
 		}
 	}
 }
@@ -413,6 +424,8 @@ static int	write_pid_line(pid_t pid)
 
 int	main(void)
 {
+	int	status;
+
 	if (prepare_response_channel() == -1 || atexit(cleanup_server) != 0)
 	{
 		cleanup_server();
@@ -431,10 +444,13 @@ int	main(void)
 		mt_putstr_fd("server: failed to publish pid\n", STDERR_FILENO);
 		return (1);
 	}
-	if (run_event_loop() == -1)
+	status = run_event_loop();
+	if (status == -1)
 	{
 		mt_putstr_fd("server: signal event channel failed\n", STDERR_FILENO);
 		return (1);
 	}
+	if (status > 0)
+		return (128 + status);
 	return (0);
 }


## `test(runtime): 중단된 client와 server 종료 정리 검증`

diff --git a/tests/protocol_regressions.sh b/tests/protocol_regressions.sh
index e2c936c..4b10ca2 100755
--- a/tests/protocol_regressions.sh
+++ b/tests/protocol_regressions.sh
@@ -7,6 +7,7 @@ OUT="$TEST_TMP/server.out"
 ERR="$TEST_TMP/server.err"
 SERVER_PID=
 SERVER_PATH=
+CLIENT_PID=
 UNRELATED_PID=
 
 cleanup()
@@ -15,6 +16,11 @@ cleanup()
 		kill "$UNRELATED_PID" 2>/dev/null || true
 		wait "$UNRELATED_PID" 2>/dev/null || true
 	fi
+	if [ -n "$CLIENT_PID" ]; then
+		kill -CONT "$CLIENT_PID" 2>/dev/null || true
+		kill "$CLIENT_PID" 2>/dev/null || true
+		wait "$CLIENT_PID" 2>/dev/null || true
+	fi
 	if [ -n "$SERVER_PID" ]; then
 		kill "$SERVER_PID" 2>/dev/null || true
 		wait "$SERVER_PID" 2>/dev/null || true
@@ -95,8 +101,31 @@ grep -qx 'client: failed to create response channel' "$TEST_TMP/file.err"
 } >"$TEST_TMP/expected"
 diff -u "$TEST_TMP/expected" "$OUT"
 
-kill "$SERVER_PID"
-wait "$SERVER_PID" 2>/dev/null || true
+CLIENT_ERR="$TEST_TMP/client.err"
+LONG_MESSAGE=$(awk 'BEGIN { for (i = 0; i < 16384; i++) printf "z" }')
+"$ROOT/client" "$SERVER_PID" "$LONG_MESSAGE" 2>"$CLIENT_ERR" &
+CLIENT_PID=$!
+sleep 0.05
+kill -STOP "$CLIENT_PID"
+sleep 0.1
+kill -CONT "$CLIENT_PID"
+wait "$CLIENT_PID"
+CLIENT_PATH="$RUNTIME_DIR/client-$CLIENT_PID.sock"
+CLIENT_PID=
+[ ! -e "$CLIENT_PATH" ]
+[ ! -s "$CLIENT_ERR" ]
+
+kill -TERM "$SERVER_PID"
+server_status=0
+wait "$SERVER_PID" || server_status=$?
 SERVER_PID=
-rm -f "$SERVER_PATH"
+[ "$server_status" -eq 143 ]
+[ ! -e "$SERVER_PATH" ]
 SERVER_PATH=
+[ ! -s "$ERR" ]
+{
+	sed -n '1p' "$OUT"
+	printf 'stale\n'
+	printf '%s\n' "$LONG_MESSAGE"
+} >"$TEST_TMP/expected-final"
+diff -u "$TEST_TMP/expected-final" "$OUT"


## `test(server): self-pipe 이벤트 손실 시 fail-stop 검증`

diff --git a/Makefile b/Makefile
index cf635a6..2ad5dbb 100644
--- a/Makefile
+++ b/Makefile
@@ -10,7 +10,8 @@ NAME_STALE_SERVER_EXEC := tests/stale_server_exec
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
-FAULT_CFLAGS := $(CFLAGS) -DMT_WRITE_CALL=mt_test_write -include tests/write_fault.h
+FAULT_CFLAGS := $(CFLAGS) -DMT_WRITE_CALL=mt_test_write \
+	-DMT_EVENT_WRITE=mt_test_event_write -include tests/write_fault.h
 RM := rm -rf
 OBJ_DIR := obj
 FAULT_OBJ_DIR := obj/fault
diff --git a/src/server.c b/src/server.c
index b519dcf..3907913 100644
--- a/src/server.c
+++ b/src/server.c
@@ -13,6 +13,10 @@
 #include <sys/un.h>
 #include <unistd.h>
 
+#ifndef MT_EVENT_WRITE
+# define MT_EVENT_WRITE write
+#endif
+
 typedef struct s_bit_event
 {
 	pid_t	sender;
@@ -87,7 +91,7 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	if (info != NULL)
 		event.sender = info->si_pid;
 	event.signal = signal;
-	if (write(g_event_pipe[1], &event, sizeof(event))
+	if (MT_EVENT_WRITE(g_event_pipe[1], &event, sizeof(event))
 		!= (ssize_t)sizeof(event))
 		g_event_overflow = 1;
 	errno = saved_errno;
diff --git a/tests/protocol_regressions.sh b/tests/protocol_regressions.sh
index 5f43394..612954a 100755
--- a/tests/protocol_regressions.sh
+++ b/tests/protocol_regressions.sh
@@ -148,3 +148,23 @@ SERVER_PATH=
 	printf '%s\n' "$LONG_MESSAGE"
 } >"$TEST_TMP/expected-final"
 diff -u "$TEST_TMP/expected-final" "$OUT"
+
+OVERFLOW_OUT="$TEST_TMP/overflow.out"
+OVERFLOW_ERR="$TEST_TMP/overflow.err"
+OVERFLOW_CLIENT_ERR="$TEST_TMP/overflow-client.err"
+MT_TEST_EVENT_EAGAIN=1 "$ROOT/tests/fault_server" \
+	>"$OVERFLOW_OUT" 2>"$OVERFLOW_ERR" &
+SERVER_PID=$!
+wait_ready "$OVERFLOW_OUT"
+overflow_client_status=0
+"$ROOT/client" "$SERVER_PID" overflow 2>"$OVERFLOW_CLIENT_ERR" \
+	|| overflow_client_status=$?
+overflow_server_status=0
+wait "$SERVER_PID" || overflow_server_status=$?
+OVERFLOW_PATH="$RUNTIME_DIR/server-$SERVER_PID.sock"
+SERVER_PID=
+[ "$overflow_client_status" -ne 0 ]
+[ "$overflow_server_status" -ne 0 ]
+grep -qx 'server: signal event channel failed' "$OVERFLOW_ERR"
+grep -qx 'client: timed out waiting for acknowledgement' "$OVERFLOW_CLIENT_ERR"
+[ ! -e "$OVERFLOW_PATH" ]
diff --git a/tests/write_fault.c b/tests/write_fault.c
index 0f10b67..c4dd543 100644
--- a/tests/write_fault.c
+++ b/tests/write_fault.c
@@ -101,3 +101,16 @@ ssize_t	mt_test_write(int fd, const void *buffer, size_t size)
 		size = g_max_write;
 	return (write(fd, buffer, size));
 }
+
+ssize_t	mt_test_event_write(int fd, const void *buffer, size_t size)
+{
+	static int	failed;
+
+	if (!failed && read_number("MT_TEST_EVENT_EAGAIN", 0) == 1)
+	{
+		failed = 1;
+		errno = EAGAIN;
+		return (-1);
+	}
+	return (write(fd, buffer, size));
+}
diff --git a/tests/write_fault.h b/tests/write_fault.h
index 98af774..d809246 100644
--- a/tests/write_fault.h
+++ b/tests/write_fault.h
@@ -5,5 +5,6 @@
 # include <sys/types.h>
 
 ssize_t	mt_test_write(int fd, const void *buffer, size_t size);
+ssize_t	mt_test_event_write(int fd, const void *buffer, size_t size);
 
 #endif


## `fix(server): 상속된 이벤트 시그널 마스크 해제`

diff --git a/src/server.c b/src/server.c
index d5c96bc..c690dcc 100644
--- a/src/server.c
+++ b/src/server.c
@@ -197,6 +197,19 @@ static int	install_signal_handlers(void)
 	return (0);
 }
 
+static int	unblock_event_signals(void)
+{
+	sigset_t	event_signals;
+
+	sigemptyset(&event_signals);
+	sigaddset(&event_signals, MT_ZERO_SIGNAL);
+	sigaddset(&event_signals, MT_ONE_SIGNAL);
+	sigaddset(&event_signals, SIGHUP);
+	sigaddset(&event_signals, SIGINT);
+	sigaddset(&event_signals, SIGTERM);
+	return (sigprocmask(SIG_UNBLOCK, &event_signals, NULL));
+}
+
 static int	valid_client_socket(const char *path)
 {
 	struct stat	info;
@@ -439,7 +452,7 @@ int	main(void)
 			STDERR_FILENO);
 		return (1);
 	}
-	if (install_signal_handlers() == -1)
+	if (install_signal_handlers() == -1 || unblock_event_signals() == -1)
 	{
 		mt_putstr_fd("server: failed to install signal handlers\n",
 			STDERR_FILENO);


## `test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증`

diff --git a/Makefile b/Makefile
index 5c9989e..7b71b48 100644
--- a/Makefile
+++ b/Makefile
@@ -98,3 +98,4 @@ test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER) \
 	sh tests/output_failure.sh
 	sh tests/protocol_regressions.sh
 	sh tests/high_fd.sh
+	sh tests/inherited_mask.sh
diff --git a/tests/inherited_mask.sh b/tests/inherited_mask.sh
new file mode 100755
index 0000000..d71a79d
--- /dev/null
+++ b/tests/inherited_mask.sh
@@ -0,0 +1,73 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/minitalk-inherited-mask.XXXXXX")
+OUT="$TEST_TMP/server.out"
+EXPECTED="$TEST_TMP/expected.out"
+SERVER_ERR="$TEST_TMP/server.err"
+CLIENT_ERR="$TEST_TMP/client.err"
+WRAPPER_PID=
+SERVER_PID=
+SERVER_PATH=
+
+cleanup()
+{
+	if [ -n "$SERVER_PID" ]; then
+		kill -TERM "$SERVER_PID" 2>/dev/null || true
+	fi
+	if [ -n "$WRAPPER_PID" ]; then
+		wait "$WRAPPER_PID" 2>/dev/null || true
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
+"$ROOT/tests/masked_exec" "$ROOT/server" >"$OUT" 2>"$SERVER_ERR" &
+WRAPPER_PID=$!
+tries=0
+while [ "$tries" -lt 50 ]; do
+	SERVER_PID=$(sed -n '1p' "$OUT" 2>/dev/null || true)
+	if [ -n "$SERVER_PID" ]; then
+		break
+	fi
+	if ! kill -0 "$WRAPPER_PID" 2>/dev/null; then
+		printf 'masked server wrapper exited before readiness\n' >&2
+		exit 1
+	fi
+	tries=$((tries + 1))
+	sleep 0.1
+done
+case "$SERVER_PID" in
+	''|*[!0-9]*)
+		printf 'masked server did not publish a pid\n' >&2
+		exit 1
+		;;
+esac
+SERVER_PATH="/tmp/signal-message-bus-$(id -u)/server-$SERVER_PID.sock"
+
+"$ROOT/client" "$SERVER_PID" "inherited mask" 2>"$CLIENT_ERR"
+[ ! -s "$CLIENT_ERR" ]
+[ ! -s "$SERVER_ERR" ]
+{
+	printf '%s\n' "$SERVER_PID"
+	printf 'inherited mask\n'
+} >"$EXPECTED"
+diff -u "$EXPECTED" "$OUT"
+
+kill -TERM "$SERVER_PID"
+server_status=0
+wait "$WRAPPER_PID" || server_status=$?
+WRAPPER_PID=
+SERVER_PID=
+[ "$server_status" -eq 143 ]
+[ ! -e "$SERVER_PATH" ]
+SERVER_PATH=
+[ ! -s "$SERVER_ERR" ]
diff --git a/tests/masked_exec.c b/tests/masked_exec.c
index 4c85d63..2ab5342 100644
--- a/tests/masked_exec.c
+++ b/tests/masked_exec.c
@@ -64,6 +64,9 @@ int	main(int argc, char **argv)
 	sigemptyset(&blocked);
 	sigaddset(&blocked, MT_ZERO_SIGNAL);
 	sigaddset(&blocked, MT_ONE_SIGNAL);
+	sigaddset(&blocked, SIGHUP);
+	sigaddset(&blocked, SIGINT);
+	sigaddset(&blocked, SIGTERM);
 	if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
 		return (1);
 	child = fork();
