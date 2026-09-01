# 시그널 데이터 채널과 데이터그램 제어 채널의 결합

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


## `feat(client): 메시지 바이트를 시그널로 전송`

diff --git a/Makefile b/Makefile
index c2db69b..3dd1efb 100644
--- a/Makefile
+++ b/Makefile
@@ -6,7 +6,7 @@ CFLAGS := -Wall -Wextra -Werror -Iinclude
 RM := rm -rf
 OBJ_DIR := obj
 
-COMMON_SRC := src/write_utils.c
+COMMON_SRC := src/write_utils.c src/parse_pid.c
 SERVER_SRC := src/server.c $(COMMON_SRC)
 CLIENT_SRC := src/client.c $(COMMON_SRC)
 
diff --git a/include/minitalk.h b/include/minitalk.h
index 8728807..86ab307 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -11,5 +11,6 @@
 void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
 size_t	mt_strlen(const char *text);
+int		mt_parse_pid(const char *text, pid_t *pid);
 
 #endif
diff --git a/src/client.c b/src/client.c
index e095f55..6f6cdda 100644
--- a/src/client.c
+++ b/src/client.c
@@ -2,13 +2,57 @@
 
 #include <unistd.h>
 
+static int	send_bit(pid_t server_pid, int bit)
+{
+	int	signal;
+
+	signal = MT_ZERO_SIGNAL;
+	if (bit != 0)
+		signal = MT_ONE_SIGNAL;
+	if (kill(server_pid, signal) == -1)
+		return (-1);
+	usleep(150);
+	return (0);
+}
+
+static int	send_byte(pid_t server_pid, unsigned char byte)
+{
+	int	shift;
+
+	shift = 7;
+	while (shift >= 0)
+	{
+		if (send_bit(server_pid, (byte >> shift) & 1) == -1)
+			return (-1);
+		shift--;
+	}
+	return (0);
+}
+
 int	main(int argc, char **argv)
 {
-	(void)argv;
+	pid_t	server_pid;
+	size_t	index;
+
 	if (argc != 3)
 	{
 		mt_putstr_fd("usage: ./client <server_pid> <message>\n", STDERR_FILENO);
 		return (1);
 	}
+	if (!mt_parse_pid(argv[1], &server_pid) || kill(server_pid, 0) == -1)
+	{
+		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
+		return (1);
+	}
+	index = 0;
+	while (argv[2][index] != '\0')
+	{
+		if (send_byte(server_pid, (unsigned char)argv[2][index]) == -1)
+		{
+			mt_putstr_fd("client: failed to send signal\n", STDERR_FILENO);
+			return (1);
+		}
+		index++;
+	}
 	return (0);
 }
diff --git a/src/parse_pid.c b/src/parse_pid.c
new file mode 100644
index 0000000..326ad03
--- /dev/null
+++ b/src/parse_pid.c
@@ -0,0 +1,30 @@
+#include "minitalk.h"
+
+static int	mt_is_digit(char character)
+{
+	return (character >= '0' && character <= '9');
+}
+
+int	mt_parse_pid(const char *text, pid_t *pid)
+{
+	long	value;
+	size_t	index;
+
+	if (text == NULL || text[0] == '\0' || pid == NULL)
+		return (0);
+	value = 0;
+	index = 0;
+	while (text[index] != '\0')
+	{
+		if (!mt_is_digit(text[index]))
+			return (0);
+		value = value * 10 + (text[index] - '0');
+		if (value > 999999)
+			return (0);
+		index++;
+	}
+	if (value <= 1)
+		return (0);
+	*pid = (pid_t)value;
+	return (1);
+}


## `feat(protocol): NUL 바이트로 메시지 종료 표시`

diff --git a/src/client.c b/src/client.c
index 6f6cdda..9b5951e 100644
--- a/src/client.c
+++ b/src/client.c
@@ -54,5 +54,10 @@ int	main(int argc, char **argv)
 		}
 		index++;
 	}
+	if (send_byte(server_pid, '\0') == -1)
+	{
+		mt_putstr_fd("client: failed to finish message\n", STDERR_FILENO);
+		return (1);
+	}
 	return (0);
 }
diff --git a/src/server.c b/src/server.c
index 718a3af..0628209 100644
--- a/src/server.c
+++ b/src/server.c
@@ -5,6 +5,14 @@
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 
+static void	flush_byte(unsigned char output)
+{
+	if (output == '\0')
+		write(STDOUT_FILENO, "\n", 1);
+	else
+		write(STDOUT_FILENO, &output, 1);
+}
+
 static void	handle_bit(int signal, siginfo_t *info, void *context)
 {
 	unsigned char	output;
@@ -19,7 +27,7 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	if (g_received_bits == 8)
 	{
 		output = (unsigned char)g_current_byte;
-		write(STDOUT_FILENO, &output, 1);
+		flush_byte(output);
 		g_current_byte = 0;
 		g_received_bits = 0;
 	}


## `feat(protocol): 비트 처리마다 ACK 전송`

diff --git a/include/minitalk.h b/include/minitalk.h
index 86ab307..d2489b1 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -7,6 +7,7 @@
 
 # define MT_ZERO_SIGNAL SIGUSR1
 # define MT_ONE_SIGNAL SIGUSR2
+# define MT_ACK_SIGNAL SIGUSR1
 
 void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
diff --git a/src/client.c b/src/client.c
index 9b5951e..93f66e9 100644
--- a/src/client.c
+++ b/src/client.c
@@ -2,27 +2,48 @@
 
 #include <unistd.h>
 
-static int	send_bit(pid_t server_pid, int bit)
+static volatile sig_atomic_t	g_ack_received;
+
+static void	handle_ack(int signal)
+{
+	(void)signal;
+	g_ack_received = 1;
+}
+
+static int	install_ack_handler(void)
+{
+	struct sigaction	action;
+
+	action.sa_handler = handle_ack;
+	sigemptyset(&action.sa_mask);
+	action.sa_flags = 0;
+	return (sigaction(MT_ACK_SIGNAL, &action, NULL));
+}
+
+static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 {
 	int	signal;
 
 	signal = MT_ZERO_SIGNAL;
 	if (bit != 0)
 		signal = MT_ONE_SIGNAL;
+	g_ack_received = 0;
 	if (kill(server_pid, signal) == -1)
 		return (-1);
-	usleep(150);
+	while (!g_ack_received)
+		sigsuspend(old_mask);
 	return (0);
 }
 
-static int	send_byte(pid_t server_pid, unsigned char byte)
+static int	send_byte(pid_t server_pid, unsigned char byte,
+		const sigset_t *old_mask)
 {
 	int	shift;
 
 	shift = 7;
 	while (shift >= 0)
 	{
-		if (send_bit(server_pid, (byte >> shift) & 1) == -1)
+		if (send_bit(server_pid, (byte >> shift) & 1, old_mask) == -1)
 			return (-1);
 		shift--;
 	}
@@ -32,6 +53,8 @@ static int	send_byte(pid_t server_pid, unsigned char byte)
 int	main(int argc, char **argv)
 {
 	pid_t	server_pid;
+	sigset_t	blocked;
+	sigset_t	old_mask;
 	size_t	index;
 
 	if (argc != 3)
@@ -44,17 +67,32 @@ int	main(int argc, char **argv)
 		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
 		return (1);
 	}
+	if (install_ack_handler() == -1)
+	{
+		mt_putstr_fd("client: failed to install signal handlers\n",
+			STDERR_FILENO);
+		return (1);
+	}
+	sigemptyset(&blocked);
+	sigaddset(&blocked, MT_ACK_SIGNAL);
+	if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
+	{
+		mt_putstr_fd("client: failed to block acknowledgement signal\n",
+			STDERR_FILENO);
+		return (1);
+	}
 	index = 0;
 	while (argv[2][index] != '\0')
 	{
-		if (send_byte(server_pid, (unsigned char)argv[2][index]) == -1)
+		if (send_byte(server_pid, (unsigned char)argv[2][index],
+				&old_mask) == -1)
 		{
 			mt_putstr_fd("client: failed to send signal\n", STDERR_FILENO);
 			return (1);
 		}
 		index++;
 	}
-	if (send_byte(server_pid, '\0') == -1)
+	if (send_byte(server_pid, '\0', &old_mask) == -1)
 	{
 		mt_putstr_fd("client: failed to finish message\n", STDERR_FILENO);
 		return (1);
diff --git a/src/server.c b/src/server.c
index 0628209..a3df3ce 100644
--- a/src/server.c
+++ b/src/server.c
@@ -31,6 +31,7 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 		g_current_byte = 0;
 		g_received_bits = 0;
 	}
+	kill(info->si_pid, MT_ACK_SIGNAL);
 }
 
 static int	install_signal_handlers(void)


## `feat(protocol): 응답 메시지 wire 형식 정의`

diff --git a/include/minitalk.h b/include/minitalk.h
index add255d..777b445 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -2,6 +2,7 @@
 # define MINITALK_H
 
 # include <signal.h>
+# include <stdint.h>
 # include <stddef.h>
 # include <sys/types.h>
 
@@ -11,6 +12,30 @@
 # define MT_NACK_SIGNAL SIGUSR2
 # define MT_ACK_TIMEOUT_SECONDS 3
 # define MT_SIGNAL_GAP_US 5000
+# define MT_RESPONSE_MAGIC 0x4d54414bU
+# define MT_REQUEST_ACQUIRE 1U
+# define MT_RESPONSE_READY 1U
+# define MT_RESPONSE_ACK 2U
+# define MT_RESPONSE_OK 0
+# define MT_RESPONSE_BUSY 1
+# define MT_RESPONSE_PATH_SIZE 104
+
+typedef struct s_mt_request
+{
+	uint32_t	magic;
+	uint32_t	kind;
+	uint32_t	nonce;
+	pid_t		client_pid;
+}	t_mt_request;
+
+typedef struct s_mt_response
+{
+	uint32_t	magic;
+	uint32_t	kind;
+	uint32_t	token;
+	int32_t		status;
+	pid_t		server_pid;
+}	t_mt_response;
 
 void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);


## `feat(server): 획득 요청을 검증해 세션 소유권 예약`

diff --git a/src/server.c b/src/server.c
index 23114a6..a43ff7e 100644
--- a/src/server.c
+++ b/src/server.c
@@ -6,11 +6,20 @@
 #include <fcntl.h>
 #include <stdlib.h>
 #include <string.h>
+#include <sys/select.h>
 #include <sys/socket.h>
 #include <sys/stat.h>
 #include <sys/un.h>
 #include <unistd.h>
 
+typedef struct s_response_request
+{
+	pid_t		client_pid;
+	uint32_t	kind;
+	uint32_t	token;
+	int32_t		status;
+}	t_response_request;
+
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 static volatile sig_atomic_t	g_client_pid;
@@ -168,6 +177,142 @@ static int	install_signal_handlers(void)
 	return (0);
 }
 
+static int	valid_client_socket(const char *path)
+{
+	struct stat	info;
+
+	if (lstat(path, &info) == -1)
+		return (-1);
+	if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid())
+	{
+		errno = EACCES;
+		return (-1);
+	}
+	return (0);
+}
+
+static int	send_response(const t_response_request *request)
+{
+	struct sockaddr_un	address;
+	t_mt_response		response;
+	char				client_path[MT_RESPONSE_PATH_SIZE];
+
+	if (mt_response_path(client_path, sizeof(client_path), "client",
+			request->client_pid) == -1 || valid_client_socket(client_path) == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	if (mt_strlen(client_path) >= sizeof(address.sun_path))
+	{
+		errno = ENAMETOOLONG;
+		return (-1);
+	}
+	memcpy(address.sun_path, client_path, mt_strlen(client_path) + 1);
+	response.magic = MT_RESPONSE_MAGIC;
+	response.kind = request->kind;
+	response.token = request->token;
+	response.status = request->status;
+	response.server_pid = getpid();
+	if (sendto(g_response_socket, &response, sizeof(response), 0,
+			(struct sockaddr *)&address, sizeof(address))
+		!= (ssize_t)sizeof(response))
+		return (-1);
+	return (0);
+}
+
+static int	valid_request_source(const struct sockaddr_un *source,
+		const t_mt_request *request)
+{
+	char	client_path[MT_RESPONSE_PATH_SIZE];
+
+	if (request->magic != MT_RESPONSE_MAGIC
+		|| request->kind != MT_REQUEST_ACQUIRE || request->client_pid <= 1
+		|| source->sun_family != AF_UNIX
+		|| source->sun_path[sizeof(source->sun_path) - 1] != '\0'
+		|| mt_response_path(client_path, sizeof(client_path), "client",
+			request->client_pid) == -1
+		|| strcmp(source->sun_path, client_path) != 0
+		|| valid_client_socket(client_path) == -1
+		|| kill(request->client_pid, 0) == -1)
+		return (0);
+	return (1);
+}
+
+static int	read_session_request(t_mt_request *request)
+{
+	unsigned char		payload[sizeof(*request) + 1];
+	struct sockaddr_un	source;
+	socklen_t			source_size;
+	ssize_t				size;
+
+	memset(&source, 0, sizeof(source));
+	source_size = sizeof(source);
+	size = recvfrom(g_response_socket, payload, sizeof(payload), 0,
+			(struct sockaddr *)&source, &source_size);
+	if (size == -1 && (errno == EAGAIN || errno == EWOULDBLOCK
+			|| errno == EINTR))
+		return (0);
+	if (size == -1)
+		return (-1);
+	if (size != (ssize_t)sizeof(*request))
+		return (0);
+	memcpy(request, payload, sizeof(*request));
+	return (valid_request_source(&source, request));
+}
+
+static int	handle_session_request(void)
+{
+	t_response_request	response;
+	t_mt_request		request;
+	int					new_owner;
+	int					status;
+
+	status = read_session_request(&request);
+	if (status <= 0)
+		return (status);
+	new_owner = 0;
+	response.status = MT_RESPONSE_OK;
+	if (g_client_pid != 0 && g_client_pid != request.client_pid)
+	{
+		if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
+			reset_session(1);
+		else
+			response.status = MT_RESPONSE_BUSY;
+	}
+	if (response.status == MT_RESPONSE_OK && g_client_pid == 0)
+	{
+		g_client_pid = request.client_pid;
+		new_owner = 1;
+	}
+	response.client_pid = request.client_pid;
+	response.kind = MT_RESPONSE_READY;
+	response.token = request.nonce;
+	if (send_response(&response) == -1 && new_owner)
+		reset_session(0);
+	return (0);
+}
+
+static int	run_response_loop(void)
+{
+	fd_set	read_set;
+	int		status;
+
+	while (1)
+	{
+		FD_ZERO(&read_set);
+		FD_SET(g_response_socket, &read_set);
+		status = pselect(g_response_socket + 1, &read_set, NULL, NULL,
+				NULL, NULL);
+		if (status == -1 && errno == EINTR)
+			continue ;
+		if (status == -1)
+			return (-1);
+		if (FD_ISSET(g_response_socket, &read_set)
+			&& handle_session_request() == -1)
+			return (-1);
+	}
+}
+
 int	main(void)
 {
 	if (prepare_response_channel() == -1 || atexit(cleanup_server) != 0)
@@ -184,7 +329,10 @@ int	main(void)
 	}
 	mt_putnbr_fd(getpid(), STDOUT_FILENO);
 	write(STDOUT_FILENO, "\n", 1);
-	while (1)
-		pause();
+	if (run_response_loop() == -1)
+	{
+		mt_putstr_fd("server: response channel failed\n", STDERR_FILENO);
+		return (1);
+	}
 	return (0);
 }


## `feat(protocol): 비트 ACK를 sequence 응답으로 큐잉`

diff --git a/src/client.c b/src/client.c
index 742713e..918a333 100644
--- a/src/client.c
+++ b/src/client.c
@@ -292,9 +292,12 @@ static int	request_session(pid_t server_pid, const char *server_path)
 			server_path, &deadline));
 }
 
-static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
+static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
+		uint32_t sequence, const char *server_path)
 {
-	int	signal;
+	struct timespec	deadline;
+	int				signal;
+	int				status;
 
 	signal = MT_ZERO_SIGNAL;
 	if (bit != 0)
@@ -312,12 +315,19 @@ static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 		return (SEND_REJECTED);
 	if (g_timed_out)
 		return (SEND_TIMEOUT);
+	if (clock_gettime(CLOCK_MONOTONIC, &deadline) == -1)
+		return (SEND_ERROR);
+	deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+	status = wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
+			server_path, &deadline);
+	if (status != 0)
+		return (status);
 	wait_signal_gap();
 	return (0);
 }
 
 static int	send_byte(pid_t server_pid, unsigned char byte,
-		const sigset_t *old_mask)
+		const sigset_t *old_mask, uint32_t *sequence, const char *server_path)
 {
 	int	status;
 	int	shift;
@@ -325,9 +335,11 @@ static int	send_byte(pid_t server_pid, unsigned char byte,
 	shift = 7;
 	while (shift >= 0)
 	{
-		status = send_bit(server_pid, (byte >> shift) & 1, old_mask);
+		status = send_bit(server_pid, (byte >> shift) & 1, old_mask,
+				*sequence, server_path);
 		if (status != 0)
 			return (status);
+		(*sequence)++;
 		shift--;
 	}
 	return (0);
@@ -353,6 +365,7 @@ int	main(int argc, char **argv)
 	sigset_t	wait_mask;
 	int		status;
 	size_t	index;
+	uint32_t	sequence;
 	char	server_path[MT_RESPONSE_PATH_SIZE];
 
 	if (argc != 3)
@@ -397,15 +410,16 @@ int	main(int argc, char **argv)
 	if (status != 0)
 		return (report_send_status(status));
 	index = 0;
+	sequence = 0;
 	while (argv[2][index] != '\0')
 	{
 		status = send_byte(server_pid, (unsigned char)argv[2][index],
-				&wait_mask);
+				&wait_mask, &sequence, server_path);
 		if (status != 0)
 			return (report_send_status(status));
 		index++;
 	}
-	status = send_byte(server_pid, '\0', &wait_mask);
+	status = send_byte(server_pid, '\0', &wait_mask, &sequence, server_path);
 	if (status != 0)
 		return (report_send_status(status));
 	return (0);
diff --git a/src/server.c b/src/server.c
index a43ff7e..dd9e8e7 100644
--- a/src/server.c
+++ b/src/server.c
@@ -24,14 +24,23 @@ static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 static volatile sig_atomic_t	g_client_pid;
 static volatile sig_atomic_t	g_line_started;
+static volatile sig_atomic_t	g_response_overflow;
+static volatile sig_atomic_t	g_sequence;
+static int						g_response_pipe[2] = {-1, -1};
 static int						g_response_socket = -1;
 static int						g_server_bound;
 static char						g_server_path[MT_RESPONSE_PATH_SIZE];
 
 static void	cleanup_server(void)
 {
+	if (g_response_pipe[0] != -1)
+		close(g_response_pipe[0]);
+	if (g_response_pipe[1] != -1)
+		close(g_response_pipe[1]);
 	if (g_response_socket != -1)
 		close(g_response_socket);
+	g_response_pipe[0] = -1;
+	g_response_pipe[1] = -1;
 	g_response_socket = -1;
 	if (g_server_bound && g_server_path[0] != '\0')
 		unlink(g_server_path);
@@ -47,6 +56,7 @@ static void	reset_session(int close_partial_line)
 	g_received_bits = 0;
 	g_client_pid = 0;
 	g_line_started = 0;
+	g_sequence = 0;
 }
 
 static void	flush_byte(unsigned char output)
@@ -63,9 +73,24 @@ static void	flush_byte(unsigned char output)
 	}
 }
 
+static void	queue_response(pid_t client_pid, uint32_t kind,
+		uint32_t token, int status)
+{
+	t_response_request	request;
+
+	request.client_pid = client_pid;
+	request.kind = kind;
+	request.token = token;
+	request.status = status;
+	if (write(g_response_pipe[1], &request, sizeof(request))
+		!= (ssize_t)sizeof(request))
+		g_response_overflow = 1;
+}
+
 static void	handle_bit(int signal, siginfo_t *info, void *context)
 {
 	unsigned char	output;
+	uint32_t		sequence;
 	int				saved_errno;
 
 	saved_errno = errno;
@@ -88,6 +113,7 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	}
 	if (g_client_pid == 0)
 		g_client_pid = info->si_pid;
+	sequence = (uint32_t)g_sequence;
 	g_current_byte <<= 1;
 	if (signal == MT_ONE_SIGNAL)
 		g_current_byte |= 1;
@@ -101,6 +127,9 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	}
 	if (kill(info->si_pid, MT_ACK_SIGNAL) == -1 && errno == ESRCH)
 		reset_session(1);
+	if (g_client_pid != 0)
+		g_sequence++;
+	queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
 	errno = saved_errno;
 }
 
@@ -139,6 +168,9 @@ static int	prepare_response_channel(void)
 {
 	struct sockaddr_un	address;
 
+	if (pipe(g_response_pipe) == -1
+		|| set_nonblocking_close_on_exec(g_response_pipe[1]) == -1)
+		return (-1);
 	if (mt_response_path(g_server_path, sizeof(g_server_path), "server",
 			getpid()) == -1 || remove_stale_socket(g_server_path) == -1)
 		return (-1);
@@ -292,24 +324,46 @@ static int	handle_session_request(void)
 	return (0);
 }
 
+static int	respond_to_bit(void)
+{
+	t_response_request	request;
+	ssize_t				size;
+
+	size = read(g_response_pipe[0], &request, sizeof(request));
+	if (size == -1 && errno == EINTR)
+		return (0);
+	if (size != (ssize_t)sizeof(request) || g_response_overflow)
+		return (-1);
+	(void)send_response(&request);
+	return (0);
+}
+
 static int	run_response_loop(void)
 {
 	fd_set	read_set;
+	int		max_fd;
 	int		status;
 
+	max_fd = g_response_pipe[0];
+	if (g_response_socket > max_fd)
+		max_fd = g_response_socket;
 	while (1)
 	{
 		FD_ZERO(&read_set);
+		FD_SET(g_response_pipe[0], &read_set);
 		FD_SET(g_response_socket, &read_set);
-		status = pselect(g_response_socket + 1, &read_set, NULL, NULL,
+		status = pselect(max_fd + 1, &read_set, NULL, NULL,
 				NULL, NULL);
 		if (status == -1 && errno == EINTR)
 			continue ;
-		if (status == -1)
+		if (status == -1 || g_response_overflow)
 			return (-1);
 		if (FD_ISSET(g_response_socket, &read_set)
 			&& handle_session_request() == -1)
 			return (-1);
+		if (FD_ISSET(g_response_pipe[0], &read_set)
+			&& respond_to_bit() == -1)
+			return (-1);
 	}
 }
 


