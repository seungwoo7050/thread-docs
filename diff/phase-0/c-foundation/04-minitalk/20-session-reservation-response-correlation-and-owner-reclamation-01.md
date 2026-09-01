# 세션 예약·응답 상관·소유자 회수

## `feat(client): ACK 대기 시간 초과 처리`

diff --git a/include/minitalk.h b/include/minitalk.h
index d2489b1..4319c16 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -8,6 +8,7 @@
 # define MT_ZERO_SIGNAL SIGUSR1
 # define MT_ONE_SIGNAL SIGUSR2
 # define MT_ACK_SIGNAL SIGUSR1
+# define MT_ACK_TIMEOUT_SECONDS 1
 
 void	mt_putstr_fd(const char *text, int fd);
 void	mt_putnbr_fd(pid_t number, int fd);
diff --git a/src/client.c b/src/client.c
index 93f66e9..8e12551 100644
--- a/src/client.c
+++ b/src/client.c
@@ -2,22 +2,32 @@
 
 #include <unistd.h>
 
+#define SEND_ERROR 1
+#define SEND_TIMEOUT 2
+
 static volatile sig_atomic_t	g_ack_received;
+static volatile sig_atomic_t	g_timed_out;
 
-static void	handle_ack(int signal)
+static void	handle_client_signal(int signal)
 {
-	(void)signal;
-	g_ack_received = 1;
+	if (signal == MT_ACK_SIGNAL)
+		g_ack_received = 1;
+	else if (signal == SIGALRM)
+		g_timed_out = 1;
 }
 
-static int	install_ack_handler(void)
+static int	install_client_handlers(void)
 {
 	struct sigaction	action;
 
-	action.sa_handler = handle_ack;
+	action.sa_handler = handle_client_signal;
 	sigemptyset(&action.sa_mask);
 	action.sa_flags = 0;
-	return (sigaction(MT_ACK_SIGNAL, &action, NULL));
+	if (sigaction(MT_ACK_SIGNAL, &action, NULL) == -1)
+		return (-1);
+	if (sigaction(SIGALRM, &action, NULL) == -1)
+		return (-1);
+	return (0);
 }
 
 static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
@@ -28,33 +38,51 @@ static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 	if (bit != 0)
 		signal = MT_ONE_SIGNAL;
 	g_ack_received = 0;
+	g_timed_out = 0;
 	if (kill(server_pid, signal) == -1)
-		return (-1);
-	while (!g_ack_received)
+		return (SEND_ERROR);
+	alarm(MT_ACK_TIMEOUT_SECONDS);
+	while (!g_ack_received && !g_timed_out)
 		sigsuspend(old_mask);
+	alarm(0);
+	if (g_timed_out)
+		return (SEND_TIMEOUT);
 	return (0);
 }
 
 static int	send_byte(pid_t server_pid, unsigned char byte,
 		const sigset_t *old_mask)
 {
+	int	status;
 	int	shift;
 
 	shift = 7;
 	while (shift >= 0)
 	{
-		if (send_bit(server_pid, (byte >> shift) & 1, old_mask) == -1)
-			return (-1);
+		status = send_bit(server_pid, (byte >> shift) & 1, old_mask);
+		if (status != 0)
+			return (status);
 		shift--;
 	}
 	return (0);
 }
 
+static int	report_send_status(int status)
+{
+	if (status == SEND_TIMEOUT)
+		mt_putstr_fd("client: timed out waiting for acknowledgement\n",
+			STDERR_FILENO);
+	else
+		mt_putstr_fd("client: failed to send signal\n", STDERR_FILENO);
+	return (1);
+}
+
 int	main(int argc, char **argv)
 {
 	pid_t	server_pid;
 	sigset_t	blocked;
 	sigset_t	old_mask;
+	int		status;
 	size_t	index;
 
 	if (argc != 3)
@@ -67,7 +95,7 @@ int	main(int argc, char **argv)
 		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
 		return (1);
 	}
-	if (install_ack_handler() == -1)
+	if (install_client_handlers() == -1)
 	{
 		mt_putstr_fd("client: failed to install signal handlers\n",
 			STDERR_FILENO);
@@ -75,6 +103,7 @@ int	main(int argc, char **argv)
 	}
 	sigemptyset(&blocked);
 	sigaddset(&blocked, MT_ACK_SIGNAL);
+	sigaddset(&blocked, SIGALRM);
 	if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
 	{
 		mt_putstr_fd("client: failed to block acknowledgement signal\n",
@@ -84,18 +113,14 @@ int	main(int argc, char **argv)
 	index = 0;
 	while (argv[2][index] != '\0')
 	{
-		if (send_byte(server_pid, (unsigned char)argv[2][index],
-				&old_mask) == -1)
-		{
-			mt_putstr_fd("client: failed to send signal\n", STDERR_FILENO);
-			return (1);
-		}
+		status = send_byte(server_pid, (unsigned char)argv[2][index],
+				&old_mask);
+		if (status != 0)
+			return (report_send_status(status));
 		index++;
 	}
-	if (send_byte(server_pid, '\0', &old_mask) == -1)
-	{
-		mt_putstr_fd("client: failed to finish message\n", STDERR_FILENO);
-		return (1);
-	}
+	status = send_byte(server_pid, '\0', &old_mask);
+	if (status != 0)
+		return (report_send_status(status));
 	return (0);
 }


## `fix(server): 활성 세션에 다른 송신자 거부`

diff --git a/include/minitalk.h b/include/minitalk.h
index 4319c16..1bd6e3a 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -8,6 +8,7 @@
 # define MT_ZERO_SIGNAL SIGUSR1
 # define MT_ONE_SIGNAL SIGUSR2
 # define MT_ACK_SIGNAL SIGUSR1
+# define MT_NACK_SIGNAL SIGUSR2
 # define MT_ACK_TIMEOUT_SECONDS 1
 
 void	mt_putstr_fd(const char *text, int fd);
diff --git a/src/client.c b/src/client.c
index 8e12551..0ef8267 100644
--- a/src/client.c
+++ b/src/client.c
@@ -4,14 +4,18 @@
 
 #define SEND_ERROR 1
 #define SEND_TIMEOUT 2
+#define SEND_REJECTED 3
 
 static volatile sig_atomic_t	g_ack_received;
 static volatile sig_atomic_t	g_timed_out;
+static volatile sig_atomic_t	g_rejected;
 
 static void	handle_client_signal(int signal)
 {
 	if (signal == MT_ACK_SIGNAL)
 		g_ack_received = 1;
+	else if (signal == MT_NACK_SIGNAL)
+		g_rejected = 1;
 	else if (signal == SIGALRM)
 		g_timed_out = 1;
 }
@@ -27,6 +31,8 @@ static int	install_client_handlers(void)
 		return (-1);
 	if (sigaction(SIGALRM, &action, NULL) == -1)
 		return (-1);
+	if (sigaction(MT_NACK_SIGNAL, &action, NULL) == -1)
+		return (-1);
 	return (0);
 }
 
@@ -39,12 +45,15 @@ static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 		signal = MT_ONE_SIGNAL;
 	g_ack_received = 0;
 	g_timed_out = 0;
+	g_rejected = 0;
 	if (kill(server_pid, signal) == -1)
 		return (SEND_ERROR);
 	alarm(MT_ACK_TIMEOUT_SECONDS);
-	while (!g_ack_received && !g_timed_out)
+	while (!g_ack_received && !g_timed_out && !g_rejected)
 		sigsuspend(old_mask);
 	alarm(0);
+	if (g_rejected)
+		return (SEND_REJECTED);
 	if (g_timed_out)
 		return (SEND_TIMEOUT);
 	return (0);
@@ -72,6 +81,9 @@ static int	report_send_status(int status)
 	if (status == SEND_TIMEOUT)
 		mt_putstr_fd("client: timed out waiting for acknowledgement\n",
 			STDERR_FILENO);
+	else if (status == SEND_REJECTED)
+		mt_putstr_fd("client: server is busy with another sender\n",
+			STDERR_FILENO);
 	else
 		mt_putstr_fd("client: failed to send signal\n", STDERR_FILENO);
 	return (1);
@@ -103,6 +115,7 @@ int	main(int argc, char **argv)
 	}
 	sigemptyset(&blocked);
 	sigaddset(&blocked, MT_ACK_SIGNAL);
+	sigaddset(&blocked, MT_NACK_SIGNAL);
 	sigaddset(&blocked, SIGALRM);
 	if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
 	{
diff --git a/src/server.c b/src/server.c
index a3df3ce..4109ca3 100644
--- a/src/server.c
+++ b/src/server.c
@@ -4,11 +4,15 @@
 
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
+static volatile sig_atomic_t	g_client_pid;
 
 static void	flush_byte(unsigned char output)
 {
 	if (output == '\0')
+	{
 		write(STDOUT_FILENO, "\n", 1);
+		g_client_pid = 0;
+	}
 	else
 		write(STDOUT_FILENO, &output, 1);
 }
@@ -20,6 +24,13 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 	(void)context;
 	if (info == NULL || info->si_pid <= 0)
 		return ;
+	if (g_client_pid != 0 && g_client_pid != info->si_pid)
+	{
+		kill(info->si_pid, MT_NACK_SIGNAL);
+		return ;
+	}
+	if (g_client_pid == 0)
+		g_client_pid = info->si_pid;
 	g_current_byte <<= 1;
 	if (signal == MT_ONE_SIGNAL)
 		g_current_byte |= 1;


## `fix(server): 종료된 송신자의 세션 복구`

diff --git a/src/server.c b/src/server.c
index 4109ca3..ae178e0 100644
--- a/src/server.c
+++ b/src/server.c
@@ -1,33 +1,59 @@
 #include "minitalk.h"
 
+#include <errno.h>
 #include <unistd.h>
 
 static volatile sig_atomic_t	g_current_byte;
 static volatile sig_atomic_t	g_received_bits;
 static volatile sig_atomic_t	g_client_pid;
+static volatile sig_atomic_t	g_line_started;
+
+static void	reset_session(int close_partial_line)
+{
+	if (close_partial_line && g_line_started)
+		write(STDOUT_FILENO, "\n", 1);
+	g_current_byte = 0;
+	g_received_bits = 0;
+	g_client_pid = 0;
+	g_line_started = 0;
+}
 
 static void	flush_byte(unsigned char output)
 {
 	if (output == '\0')
 	{
 		write(STDOUT_FILENO, "\n", 1);
-		g_client_pid = 0;
+		reset_session(0);
 	}
 	else
+	{
 		write(STDOUT_FILENO, &output, 1);
+		g_line_started = 1;
+	}
 }
 
 static void	handle_bit(int signal, siginfo_t *info, void *context)
 {
 	unsigned char	output;
+	int				saved_errno;
 
+	saved_errno = errno;
 	(void)context;
 	if (info == NULL || info->si_pid <= 0)
+	{
+		errno = saved_errno;
 		return ;
+	}
 	if (g_client_pid != 0 && g_client_pid != info->si_pid)
 	{
-		kill(info->si_pid, MT_NACK_SIGNAL);
-		return ;
+		if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
+			reset_session(1);
+		else
+		{
+			kill(info->si_pid, MT_NACK_SIGNAL);
+			errno = saved_errno;
+			return ;
+		}
 	}
 	if (g_client_pid == 0)
 		g_client_pid = info->si_pid;
@@ -42,7 +68,9 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 		g_current_byte = 0;
 		g_received_bits = 0;
 	}
-	kill(info->si_pid, MT_ACK_SIGNAL);
+	if (kill(info->si_pid, MT_ACK_SIGNAL) == -1 && errno == ESRCH)
+		reset_session(1);
+	errno = saved_errno;
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


## `feat(client): READY 응답을 출처와 nonce로 상관 검증`

diff --git a/src/client.c b/src/client.c
index fb1fea8..742713e 100644
--- a/src/client.c
+++ b/src/client.c
@@ -6,6 +6,7 @@
 #include <fcntl.h>
 #include <stdlib.h>
 #include <string.h>
+#include <sys/select.h>
 #include <sys/socket.h>
 #include <sys/stat.h>
 #include <sys/un.h>
@@ -124,6 +125,173 @@ static int	bind_client_socket(void)
 	return (0);
 }
 
+static int	validate_server_socket(const char *path)
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
+static int	generate_nonce(uint32_t *nonce)
+{
+	unsigned char	*bytes;
+	size_t			offset;
+	ssize_t			count;
+	int				random_fd;
+
+	random_fd = open("/dev/urandom", O_RDONLY);
+	if (random_fd == -1)
+		return (-1);
+	bytes = (unsigned char *)nonce;
+	offset = 0;
+	while (offset < sizeof(*nonce))
+	{
+		count = read(random_fd, bytes + offset, sizeof(*nonce) - offset);
+		if (count == -1 && errno == EINTR)
+			continue ;
+		if (count <= 0)
+		{
+			close(random_fd);
+			return (-1);
+		}
+		offset += (size_t)count;
+	}
+	if (close(random_fd) == -1)
+		return (-1);
+	if (*nonce == 0)
+		*nonce = 1;
+	return (0);
+}
+
+static struct timespec	time_until(const struct timespec *deadline)
+{
+	struct timespec	now;
+	struct timespec	remaining;
+
+	remaining.tv_sec = 0;
+	remaining.tv_nsec = 0;
+	if (clock_gettime(CLOCK_MONOTONIC, &now) == -1)
+		return (remaining);
+	remaining.tv_sec = deadline->tv_sec - now.tv_sec;
+	remaining.tv_nsec = deadline->tv_nsec - now.tv_nsec;
+	if (remaining.tv_nsec < 0)
+	{
+		remaining.tv_sec--;
+		remaining.tv_nsec += 1000000000L;
+	}
+	if (remaining.tv_sec < 0)
+	{
+		remaining.tv_sec = 0;
+		remaining.tv_nsec = 0;
+	}
+	return (remaining);
+}
+
+static int	valid_source(const struct sockaddr_un *source,
+		const char *server_path)
+{
+	if (source->sun_family != AF_UNIX)
+		return (0);
+	if (source->sun_path[sizeof(source->sun_path) - 1] != '\0')
+		return (0);
+	return (strcmp(source->sun_path, server_path) == 0);
+}
+
+static int	read_response(pid_t server_pid, uint32_t kind, uint32_t token,
+		const char *server_path)
+{
+	unsigned char		payload[sizeof(t_mt_response) + 1];
+	struct sockaddr_un	source;
+	socklen_t			source_size;
+	t_mt_response		response;
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
+		return (SEND_ERROR);
+	if (size != (ssize_t)sizeof(response))
+		return (0);
+	memcpy(&response, payload, sizeof(response));
+	if (!valid_source(&source, server_path)
+		|| response.magic != MT_RESPONSE_MAGIC
+		|| response.server_pid != server_pid || response.kind != kind
+		|| response.token != token)
+		return (0);
+	if (response.status == MT_RESPONSE_BUSY)
+		return (SEND_REJECTED);
+	if (response.status != MT_RESPONSE_OK)
+		return (SEND_ERROR);
+	return (-1);
+}
+
+static int	wait_for_response(pid_t server_pid, uint32_t kind, uint32_t token,
+		const char *server_path, const struct timespec *deadline)
+{
+	struct timespec	remaining;
+	fd_set			read_set;
+	int				status;
+
+	while (1)
+	{
+		status = read_response(server_pid, kind, token, server_path);
+		if (status != 0)
+		{
+			if (status == -1)
+				return (0);
+			return (status);
+		}
+		remaining = time_until(deadline);
+		if (remaining.tv_sec == 0 && remaining.tv_nsec == 0)
+			return (SEND_TIMEOUT);
+		FD_ZERO(&read_set);
+		FD_SET(g_response_socket, &read_set);
+		status = pselect(g_response_socket + 1, &read_set, NULL, NULL,
+				&remaining, NULL);
+		if (status == -1 && errno != EINTR)
+			return (SEND_ERROR);
+	}
+}
+
+static int	request_session(pid_t server_pid, const char *server_path)
+{
+	struct sockaddr_un	address;
+	struct timespec		deadline;
+	t_mt_request			request;
+
+	if (generate_nonce(&request.nonce) == -1
+		|| validate_server_socket(server_path) == -1
+		|| clock_gettime(CLOCK_MONOTONIC, &deadline) == -1)
+		return (SEND_ERROR);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	if (mt_strlen(server_path) >= sizeof(address.sun_path))
+		return (SEND_ERROR);
+	memcpy(address.sun_path, server_path, mt_strlen(server_path) + 1);
+	request.magic = MT_RESPONSE_MAGIC;
+	request.kind = MT_REQUEST_ACQUIRE;
+	request.client_pid = getpid();
+	deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+	if (sendto(g_response_socket, &request, sizeof(request), 0,
+			(struct sockaddr *)&address, sizeof(address))
+		!= (ssize_t)sizeof(request))
+		return (SEND_ERROR);
+	return (wait_for_response(server_pid, MT_RESPONSE_READY, request.nonce,
+			server_path, &deadline));
+}
+
 static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
 {
 	int	signal;
@@ -185,13 +353,16 @@ int	main(int argc, char **argv)
 	sigset_t	wait_mask;
 	int		status;
 	size_t	index;
+	char	server_path[MT_RESPONSE_PATH_SIZE];
 
 	if (argc != 3)
 	{
 		mt_putstr_fd("usage: ./client <server_pid> <message>\n", STDERR_FILENO);
 		return (1);
 	}
-	if (!mt_parse_pid(argv[1], &server_pid) || kill(server_pid, 0) == -1)
+	if (!mt_parse_pid(argv[1], &server_pid) || kill(server_pid, 0) == -1
+		|| mt_response_path(server_path, sizeof(server_path), "server",
+			server_pid) == -1 || validate_server_socket(server_path) == -1)
 	{
 		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
 		return (1);
@@ -222,6 +393,9 @@ int	main(int argc, char **argv)
 	sigdelset(&wait_mask, MT_ACK_SIGNAL);
 	sigdelset(&wait_mask, MT_NACK_SIGNAL);
 	sigdelset(&wait_mask, SIGALRM);
+	status = request_session(server_pid, server_path);
+	if (status != 0)
+		return (report_send_status(status));
 	index = 0;
 	while (argv[2][index] != '\0')
 	{
diff --git a/tests/smoke.sh b/tests/smoke.sh
old mode 100755
new mode 100644
index d43230b..60674de
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -2,15 +2,14 @@
 set -eu
 
 ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
-TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/minitalk-smoke.XXXXXX")
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/signal-message-bus-smoke.XXXXXX")
 OUT="$TEST_TMP/server.out"
 EXPECTED="$TEST_TMP/expected.out"
 SERVER_ERR="$TEST_TMP/server.err"
 CLIENT_ERR="$TEST_TMP/client.err"
 INVALID_ERR="$TEST_TMP/invalid.err"
-TIMEOUT_ERR="$TEST_TMP/timeout.err"
+UNRELATED_ERR="$TEST_TMP/unrelated.err"
 EXPECTED_INVALID_ERR="$TEST_TMP/expected-invalid.err"
-EXPECTED_TIMEOUT_ERR="$TEST_TMP/expected-timeout.err"
 DUMMY_READY="$TEST_TMP/dummy.ready"
 SERVER_PID=
 DUMMY_PID=
@@ -20,15 +19,24 @@ send_checked()
 	label=$1
 	message=$2
 	if ! "$ROOT/client" "$SERVER_PID" "$message" 2>"$CLIENT_ERR"; then
-		printf 'client failed during %s\n' "$label" >&2
+		printf 'normal client failed during %s\n' "$label" >&2
+		cat "$CLIENT_ERR" >&2
 		exit 1
 	fi
 	if [ -s "$CLIENT_ERR" ]; then
-		printf 'client wrote to stderr during %s\n' "$label" >&2
+		printf 'normal client wrote to stderr during %s\n' "$label" >&2
+		cat "$CLIENT_ERR" >&2
 		exit 1
 	fi
 }
 
+server_ready()
+{
+	[ -f "$OUT" ] || return 1
+	[ "$(wc -l <"$OUT")" -ge 1 ] &&
+	[ "$(sed -n '1p' "$OUT")" = "$SERVER_PID" ]
+}
+
 cleanup()
 {
 	if [ -n "$DUMMY_PID" ]; then
@@ -52,7 +60,7 @@ SERVER_PID=$!
 
 tries=0
 while [ "$tries" -lt 50 ]; do
-	if [ -s "$OUT" ] && [ "$(sed -n '1p' "$OUT")" = "$SERVER_PID" ]; then
+	if server_ready; then
 		break
 	fi
 	tries=$((tries + 1))
@@ -63,10 +71,11 @@ if ! kill -0 "$SERVER_PID" 2>/dev/null; then
 	printf 'server exited before smoke test\n' >&2
 	exit 1
 fi
-if [ ! -s "$OUT" ] || [ "$(sed -n '1p' "$OUT")" != "$SERVER_PID" ]; then
+if ! server_ready; then
 	printf 'server did not publish its complete pid line\n' >&2
 	exit 1
 fi
+
 send_checked hello "hello"
 send_checked empty ""
 send_checked utf8 "안녕하세요"
@@ -92,25 +101,23 @@ DUMMY_PID=$!
 tries=0
 while [ "$tries" -lt 50 ] && ! grep -qx 'ready' "$DUMMY_READY"; do
 	if ! kill -0 "$DUMMY_PID" 2>/dev/null; then
-		printf 'non-acking process exited before becoming ready\n' >&2
+		printf 'unrelated process exited before becoming ready\n' >&2
 		exit 1
 	fi
 	tries=$((tries + 1))
 	sleep 0.1
 done
 if ! grep -qx 'ready' "$DUMMY_READY"; then
-	printf 'non-acking process did not become ready\n' >&2
+	printf 'unrelated process did not become ready\n' >&2
 	exit 1
 fi
-if "$ROOT/client" "$DUMMY_PID" "timeout" 2>"$TIMEOUT_ERR"; then
-	printf 'client did not fail when acknowledgement was missing\n' >&2
+if "$ROOT/client" "$DUMMY_PID" "unrelated" 2>"$UNRELATED_ERR"; then
+	printf 'client accepted a process without a server socket\n' >&2
 	exit 1
 fi
-printf 'client: timed out waiting for acknowledgement\n' \
-	>"$EXPECTED_TIMEOUT_ERR"
-diff -u "$EXPECTED_TIMEOUT_ERR" "$TIMEOUT_ERR"
+diff -u "$EXPECTED_INVALID_ERR" "$UNRELATED_ERR"
 if ! kill -0 "$DUMMY_PID" 2>/dev/null; then
-	printf 'non-acking process died during timeout verification\n' >&2
+	printf 'unrelated process died during server identity verification\n' >&2
 	exit 1
 fi
 kill "$DUMMY_PID" 2>/dev/null || true
@@ -118,7 +125,8 @@ wait "$DUMMY_PID" 2>/dev/null || true
 DUMMY_PID=
 
 if [ -s "$SERVER_ERR" ]; then
-	printf 'server wrote unexpected diagnostics\n' >&2
+	printf 'server wrote to stderr\n' >&2
+	cat "$SERVER_ERR" >&2
 	exit 1
 fi
 


