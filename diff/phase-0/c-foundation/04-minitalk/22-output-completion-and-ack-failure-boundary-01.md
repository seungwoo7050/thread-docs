# 출력 완료와 ACK 실패 경계

## `feat(io): 기본 출력 유틸리티 추가`

diff --git a/include/minitalk.h b/include/minitalk.h
new file mode 100644
index 0000000..b0f0894
--- /dev/null
+++ b/include/minitalk.h
@@ -0,0 +1,11 @@
+#ifndef MINITALK_H
+# define MINITALK_H
+
+# include <stddef.h>
+# include <sys/types.h>
+
+void	mt_putstr_fd(const char *text, int fd);
+void	mt_putnbr_fd(pid_t number, int fd);
+size_t	mt_strlen(const char *text);
+
+#endif
diff --git a/src/write_utils.c b/src/write_utils.c
new file mode 100644
index 0000000..7f4ead5
--- /dev/null
+++ b/src/write_utils.c
@@ -0,0 +1,47 @@
+#include "minitalk.h"
+
+#include <unistd.h>
+
+size_t	mt_strlen(const char *text)
+{
+	size_t	length;
+
+	length = 0;
+	while (text != NULL && text[length] != '\0')
+		length++;
+	return (length);
+}
+
+void	mt_putstr_fd(const char *text, int fd)
+{
+	if (text == NULL)
+		return ;
+	write(fd, text, mt_strlen(text));
+}
+
+void	mt_putnbr_fd(pid_t number, int fd)
+{
+	char	buffer[32];
+	int		index;
+	long	value;
+
+	value = (long)number;
+	if (value == 0)
+	{
+		write(fd, "0", 1);
+		return ;
+	}
+	if (value < 0)
+	{
+		write(fd, "-", 1);
+		value = -value;
+	}
+	index = 0;
+	while (value > 0)
+	{
+		buffer[index++] = (char)('0' + (value % 10));
+		value /= 10;
+	}
+	while (index > 0)
+		write(fd, &buffer[--index], 1);
+}


## `feat(server): 시그널 비트를 바이트로 조립`

diff --git a/include/minitalk.h b/include/minitalk.h
index b0f0894..8728807 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -1,9 +1,13 @@
 #ifndef MINITALK_H
 # define MINITALK_H
 
+# include <signal.h>
 # include <stddef.h>
 # include <sys/types.h>
 
+# define MT_ZERO_SIGNAL SIGUSR1
+# define MT_ONE_SIGNAL SIGUSR2
+
 void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
 size_t	mt_strlen(const char *text);
diff --git a/src/server.c b/src/server.c
index 6c5525e..718a3af 100644
--- a/src/server.c
+++ b/src/server.c
@@ -2,8 +2,52 @@
 
 #include <unistd.h>
 
+static volatile sig_atomic_t	g_current_byte;
+static volatile sig_atomic_t	g_received_bits;
+
+static void	handle_bit(int signal, siginfo_t *info, void *context)
+{
+	unsigned char	output;
+
+	(void)context;
+	if (info == NULL || info->si_pid <= 0)
+		return ;
+	g_current_byte <<= 1;
+	if (signal == MT_ONE_SIGNAL)
+		g_current_byte |= 1;
+	g_received_bits++;
+	if (g_received_bits == 8)
+	{
+		output = (unsigned char)g_current_byte;
+		write(STDOUT_FILENO, &output, 1);
+		g_current_byte = 0;
+		g_received_bits = 0;
+	}
+}
+
+static int	install_signal_handlers(void)
+{
+	struct sigaction	action;
+
+	action.sa_sigaction = handle_bit;
+	sigemptyset(&action.sa_mask);
+	sigaddset(&action.sa_mask, MT_ZERO_SIGNAL);
+	sigaddset(&action.sa_mask, MT_ONE_SIGNAL);
+	action.sa_flags = SA_SIGINFO;
+	if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1)
+		return (-1);
+	if (sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
+		return (-1);
+	return (0);
+}
+
 int	main(void)
 {
+	if (install_signal_handlers() == -1)
+	{
+		mt_putstr_fd("server: failed to install signal handlers\n", STDERR_FILENO);
+		return (1);
+	}
 	mt_putnbr_fd(getpid(), STDOUT_FILENO);
 	write(STDOUT_FILENO, "\n", 1);
 	while (1)


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


## `fix(io): 중단·부분 쓰기를 끝까지 처리`

diff --git a/include/minitalk.h b/include/minitalk.h
index b656d0c..05aafb8 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -39,6 +39,7 @@ void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
 size_t	mt_strlen(const char *text);
 int		mt_parse_pid(const char *text, pid_t *pid);
+int		mt_write_all(int fd, const void *buffer, size_t size);
 int		mt_runtime_dir(char *buffer, size_t size);
 int		mt_response_path(char *buffer, size_t size, const char *role,
 			pid_t pid);
diff --git a/src/write_utils.c b/src/write_utils.c
index 7f4ead5..703981e 100644
--- a/src/write_utils.c
+++ b/src/write_utils.c
@@ -1,5 +1,6 @@
 #include "minitalk.h"
 
+#include <errno.h>
 #include <unistd.h>
 
 size_t	mt_strlen(const char *text)
@@ -12,11 +13,36 @@ size_t	mt_strlen(const char *text)
 	return (length);
 }
 
+int	mt_write_all(int fd, const void *buffer, size_t size)
+{
+	const unsigned char	*bytes;
+	size_t				offset;
+	ssize_t				written;
+
+	bytes = (const unsigned char *)buffer;
+	offset = 0;
+	while (offset < size)
+	{
+		written = write(fd, bytes + offset, size - offset);
+		if (written == -1 && errno == EINTR)
+			continue ;
+		if (written == -1)
+			return (-1);
+		if (written == 0)
+		{
+			errno = EIO;
+			return (-1);
+		}
+		offset += (size_t)written;
+	}
+	return (0);
+}
+
 void	mt_putstr_fd(const char *text, int fd)
 {
 	if (text == NULL)
 		return ;
-	write(fd, text, mt_strlen(text));
+	mt_write_all(fd, text, mt_strlen(text));
 }
 
 void	mt_putnbr_fd(pid_t number, int fd)
@@ -28,12 +54,12 @@ void	mt_putnbr_fd(pid_t number, int fd)
 	value = (long)number;
 	if (value == 0)
 	{
-		write(fd, "0", 1);
+		mt_write_all(fd, "0", 1);
 		return ;
 	}
 	if (value < 0)
 	{
-		write(fd, "-", 1);
+		mt_write_all(fd, "-", 1);
 		value = -value;
 	}
 	index = 0;
@@ -43,5 +69,5 @@ void	mt_putnbr_fd(pid_t number, int fd)
 		value /= 10;
 	}
 	while (index > 0)
-		write(fd, &buffer[--index], 1);
+		mt_write_all(fd, &buffer[--index], 1);
 }


## `fix(server): stdout 실패 뒤 ACK 전송 차단`

diff --git a/src/server.c b/src/server.c
index c324517..082a3a2 100644
--- a/src/server.c
+++ b/src/server.c
@@ -49,29 +49,31 @@ static void	cleanup_server(void)
 	g_server_path[0] = '\0';
 }
 
-static void	reset_session(int close_partial_line)
+static int	reset_session(int close_partial_line)
 {
-	if (close_partial_line && g_line_started)
-		write(STDOUT_FILENO, "\n", 1);
+	if (close_partial_line && g_line_started
+		&& mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
+		return (-1);
 	g_current_byte = 0;
 	g_received_bits = 0;
 	g_client_pid = 0;
 	g_line_started = 0;
 	g_sequence = 0;
+	return (0);
 }
 
-static void	flush_byte(unsigned char output)
+static int	flush_byte(unsigned char output)
 {
 	if (output == '\0')
 	{
-		write(STDOUT_FILENO, "\n", 1);
-		reset_session(0);
-	}
-	else
-	{
-		write(STDOUT_FILENO, &output, 1);
-		g_line_started = 1;
+		if (mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
+			return (-1);
+		return (reset_session(0));
 	}
+	if (mt_write_all(STDOUT_FILENO, &output, 1) == -1)
+		return (-1);
+	g_line_started = 1;
+	return (0);
 }
 
 static void	handle_bit(int signal, siginfo_t *info, void *context)
@@ -166,6 +168,11 @@ static int	install_signal_handlers(void)
 {
 	struct sigaction	action;
 
+	memset(&action, 0, sizeof(action));
+	action.sa_handler = SIG_IGN;
+	sigemptyset(&action.sa_mask);
+	if (sigaction(SIGPIPE, &action, NULL) == -1)
+		return (-1);
 	memset(&action, 0, sizeof(action));
 	action.sa_sigaction = handle_bit;
 	sigemptyset(&action.sa_mask);
@@ -276,7 +283,10 @@ static int	handle_session_request(void)
 	if (g_client_pid != 0 && g_client_pid != request.client_pid)
 	{
 		if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
-			reset_session(1);
+		{
+			if (reset_session(1) == -1)
+				return (-1);
+		}
 		else
 			status = MT_RESPONSE_BUSY;
 	}
@@ -286,8 +296,8 @@ static int	handle_session_request(void)
 		new_owner = 1;
 	}
 	if (send_response(request.client_pid, MT_RESPONSE_READY, request.nonce,
-			status) == -1 && new_owner)
-		reset_session(0);
+			status) == -1 && new_owner && reset_session(0) == -1)
+		return (-1);
 	return (0);
 }
 
@@ -329,16 +339,19 @@ static int	process_bit(const t_bit_event *event)
 	if (g_received_bits == 8)
 	{
 		output = g_current_byte;
-		flush_byte(output);
+		if (flush_byte(output) == -1)
+			return (-1);
 		g_current_byte = 0;
 		g_received_bits = 0;
 	}
 	if (g_client_pid != 0)
 		g_sequence++;
-	if (send_response(event->sender, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK) == -1)
+	if (send_response(event->sender, MT_RESPONSE_ACK, sequence,
+			MT_RESPONSE_OK) == -1)
 	{
-		if (event->sender == g_client_pid)
-			reset_session(1);
+		if (event->sender == g_client_pid
+			&& reset_session(1) == -1)
+			return (-1);
 	}
 	return (0);
 }
@@ -380,6 +393,24 @@ static int	run_event_loop(void)
 	}
 }
 
+static int	write_pid_line(pid_t pid)
+{
+	char	buffer[32];
+	size_t	index;
+	long	value;
+
+	index = sizeof(buffer);
+	buffer[--index] = '\n';
+	value = (long)pid;
+	while (value > 0)
+	{
+		buffer[--index] = (char)('0' + value % 10);
+		value /= 10;
+	}
+	return (mt_write_all(STDOUT_FILENO, buffer + index,
+		sizeof(buffer) - index));
+}
+
 int	main(void)
 {
 	if (prepare_response_channel() == -1 || atexit(cleanup_server) != 0)
@@ -395,8 +426,11 @@ int	main(void)
 			STDERR_FILENO);
 		return (1);
 	}
-	mt_putnbr_fd(getpid(), STDOUT_FILENO);
-	write(STDOUT_FILENO, "\n", 1);
+	if (write_pid_line(getpid()) == -1)
+	{
+		mt_putstr_fd("server: failed to publish pid\n", STDERR_FILENO);
+		return (1);
+	}
 	if (run_event_loop() == -1)
 	{
 		mt_putstr_fd("server: signal event channel failed\n", STDERR_FILENO);


