## `refactor(protocol): 이전 signal ACK 경로 제거`

diff --git a/include/minitalk.h b/include/minitalk.h
index defeeb8..b656d0c 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -8,8 +8,6 @@
 
 # define MT_ZERO_SIGNAL SIGUSR1
 # define MT_ONE_SIGNAL SIGUSR2
-# define MT_ACK_SIGNAL SIGUSR1
-# define MT_NACK_SIGNAL SIGUSR2
 # define MT_ACK_TIMEOUT_SECONDS 3
 # define MT_SIGNAL_GAP_US 5000
 # define MT_RESPONSE_MAGIC 0x4d54414bU
diff --git a/src/client.c b/src/client.c
index 6fb6f47..ddd0e60 100644
--- a/src/client.c
+++ b/src/client.c
@@ -17,11 +17,8 @@
 #define SEND_TIMEOUT 2
 #define SEND_REJECTED 3
 
-static volatile sig_atomic_t	g_ack_received;
-static volatile sig_atomic_t	g_timed_out;
-static volatile sig_atomic_t	g_rejected;
-static int						g_response_socket = -1;
-static char						g_client_path[MT_RESPONSE_PATH_SIZE];
+static int	g_response_socket = -1;
+static char	g_client_path[MT_RESPONSE_PATH_SIZE];
 
 static void	cleanup_response_socket(void)
 {
@@ -43,32 +40,6 @@ static void	wait_signal_gap(void)
 		;
 }
 
-static void	handle_client_signal(int signal)
-{
-	if (signal == MT_ACK_SIGNAL)
-		g_ack_received = 1;
-	else if (signal == MT_NACK_SIGNAL)
-		g_rejected = 1;
-	else if (signal == SIGALRM)
-		g_timed_out = 1;
-}
-
-static int	install_client_handlers(void)
-{
-	struct sigaction	action;
-
-	action.sa_handler = handle_client_signal;
-	sigemptyset(&action.sa_mask);
-	action.sa_flags = 0;
-	if (sigaction(MT_ACK_SIGNAL, &action, NULL) == -1)
-		return (-1);
-	if (sigaction(SIGALRM, &action, NULL) == -1)
-		return (-1);
-	if (sigaction(MT_NACK_SIGNAL, &action, NULL) == -1)
-		return (-1);
-	return (0);
-}
-
 static int	set_nonblocking_close_on_exec(int fd)
 {
 	int	flags;
@@ -100,6 +71,20 @@ static int	remove_stale_socket(const char *path)
 	return (unlink(path));
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
 static int	bind_client_socket(void)
 {
 	struct sockaddr_un	address;
@@ -125,25 +110,11 @@ static int	bind_client_socket(void)
 	return (0);
 }
 
-static int	validate_server_socket(const char *path)
-{
-	struct stat	info;
-
-	if (lstat(path, &info) == -1)
-		return (-1);
-	if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid())
-	{
-		errno = EACCES;
-		return (-1);
-	}
-	return (0);
-}
-
 static int	generate_nonce(uint32_t *nonce)
 {
 	unsigned char	*bytes;
 	size_t			offset;
-	ssize_t			count;
+	ssize_t		count;
 	int				random_fd;
 
 	random_fd = open("/dev/urandom", O_RDONLY);
@@ -269,7 +240,7 @@ static int	request_session(pid_t server_pid, const char *server_path)
 {
 	struct sockaddr_un	address;
 	struct timespec		deadline;
-	t_mt_request			request;
+	t_mt_request		request;
 
 	if (generate_nonce(&request.nonce) == -1
 		|| validate_server_socket(server_path) == -1
@@ -351,13 +322,11 @@ static int	report_send_status(int status)
 
 int	main(int argc, char **argv)
 {
-	pid_t	server_pid;
-	sigset_t	blocked;
-	sigset_t	wait_mask;
-	int		status;
-	size_t	index;
+	pid_t		server_pid;
+	int			status;
+	size_t		index;
 	uint32_t	sequence;
-	char	server_path[MT_RESPONSE_PATH_SIZE];
+	char		server_path[MT_RESPONSE_PATH_SIZE];
 
 	if (argc != 3)
 	{
@@ -366,17 +335,11 @@ int	main(int argc, char **argv)
 	}
 	if (!mt_parse_pid(argv[1], &server_pid) || kill(server_pid, 0) == -1
 		|| mt_response_path(server_path, sizeof(server_path), "server",
-			server_pid) == -1 || validate_server_socket(server_path) == -1)
+			server_pid) == -1)
 	{
 		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
 		return (1);
 	}
-	if (install_client_handlers() == -1)
-	{
-		mt_putstr_fd("client: failed to install signal handlers\n",
-			STDERR_FILENO);
-		return (1);
-	}
 	if (bind_client_socket() == -1 || atexit(cleanup_response_socket) != 0)
 	{
 		cleanup_response_socket();
@@ -384,19 +347,11 @@ int	main(int argc, char **argv)
 			STDERR_FILENO);
 		return (1);
 	}
-	sigemptyset(&blocked);
-	sigaddset(&blocked, MT_ACK_SIGNAL);
-	sigaddset(&blocked, MT_NACK_SIGNAL);
-	sigaddset(&blocked, SIGALRM);
-	if (sigprocmask(SIG_BLOCK, &blocked, &wait_mask) == -1)
+	if (validate_server_socket(server_path) == -1)
 	{
-		mt_putstr_fd("client: failed to block acknowledgement signal\n",
-			STDERR_FILENO);
+		mt_putstr_fd("client: invalid server pid\n", STDERR_FILENO);
 		return (1);
 	}
-	sigdelset(&wait_mask, MT_ACK_SIGNAL);
-	sigdelset(&wait_mask, MT_NACK_SIGNAL);
-	sigdelset(&wait_mask, SIGALRM);
 	status = request_session(server_pid, server_path);
 	if (status != 0)
 		return (report_send_status(status));
diff --git a/src/server.c b/src/server.c
index dd9e8e7..ef6cee3 100644
--- a/src/server.c
+++ b/src/server.c
@@ -26,10 +26,10 @@ static volatile sig_atomic_t	g_client_pid;
 static volatile sig_atomic_t	g_line_started;
 static volatile sig_atomic_t	g_response_overflow;
 static volatile sig_atomic_t	g_sequence;
-static int						g_response_pipe[2] = {-1, -1};
-static int						g_response_socket = -1;
-static int						g_server_bound;
-static char						g_server_path[MT_RESPONSE_PATH_SIZE];
+static int					g_response_pipe[2] = {-1, -1};
+static int					g_response_socket = -1;
+static int					g_server_bound;
+static char					g_server_path[MT_RESPONSE_PATH_SIZE];
 
 static void	cleanup_server(void)
 {
@@ -100,19 +100,11 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 		errno = saved_errno;
 		return ;
 	}
-	if (g_client_pid != 0 && g_client_pid != info->si_pid)
+	if (g_client_pid == 0 || g_client_pid != info->si_pid)
 	{
-		if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
-			reset_session(1);
-		else
-		{
-			kill(info->si_pid, MT_NACK_SIGNAL);
-			errno = saved_errno;
-			return ;
-		}
+		errno = saved_errno;
+		return ;
 	}
-	if (g_client_pid == 0)
-		g_client_pid = info->si_pid;
 	sequence = (uint32_t)g_sequence;
 	g_current_byte <<= 1;
 	if (signal == MT_ONE_SIGNAL)
@@ -125,8 +117,6 @@ static void	handle_bit(int signal, siginfo_t *info, void *context)
 		g_current_byte = 0;
 		g_received_bits = 0;
 	}
-	if (kill(info->si_pid, MT_ACK_SIGNAL) == -1 && errno == ESRCH)
-		reset_session(1);
 	if (g_client_pid != 0)
 		g_sequence++;
 	queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
@@ -334,7 +324,9 @@ static int	respond_to_bit(void)
 		return (0);
 	if (size != (ssize_t)sizeof(request) || g_response_overflow)
 		return (-1);
-	(void)send_response(&request);
+	if (send_response(&request) == -1 && request.status == MT_RESPONSE_OK
+		&& request.client_pid == (pid_t)g_client_pid)
+		reset_session(1);
 	return (0);
 }
 
@@ -352,8 +344,7 @@ static int	run_response_loop(void)
 		FD_ZERO(&read_set);
 		FD_SET(g_response_pipe[0], &read_set);
 		FD_SET(g_response_socket, &read_set);
-		status = pselect(max_fd + 1, &read_set, NULL, NULL,
-				NULL, NULL);
+		status = pselect(max_fd + 1, &read_set, NULL, NULL, NULL, NULL);
 		if (status == -1 && errno == EINTR)
 			continue ;
 		if (status == -1 || g_response_overflow)
@@ -378,7 +369,8 @@ int	main(void)
 	}
 	if (install_signal_handlers() == -1)
 	{
-		mt_putstr_fd("server: failed to install signal handlers\n", STDERR_FILENO);
+		mt_putstr_fd("server: failed to install signal handlers\n",
+			STDERR_FILENO);
 		return (1);
 	}
 	mt_putnbr_fd(getpid(), STDOUT_FILENO);
diff --git a/tests/session_sender.c b/tests/session_sender.c
index 8e55b46..9a5a55d 100644
--- a/tests/session_sender.c
+++ b/tests/session_sender.c
@@ -14,11 +14,8 @@
 #include <time.h>
 #include <unistd.h>
 
-static volatile sig_atomic_t	g_ack_received;
-static volatile sig_atomic_t	g_timed_out;
-static volatile sig_atomic_t	g_rejected;
-static int						g_socket = -1;
-static char						g_path[MT_RESPONSE_PATH_SIZE];
+static int	g_socket = -1;
+static char	g_path[MT_RESPONSE_PATH_SIZE];
 
 static void	cleanup(void)
 {
@@ -66,7 +63,8 @@ static int	prepare_socket(void)
 
 static int	response_matches(const t_mt_response *response,
 		const struct sockaddr_un *source, pid_t server_pid, uint32_t kind,
-		uint32_t token, const char *server_path)
+		uint32_t token,
+		const char *server_path)
 {
 	return (source->sun_family == AF_UNIX
 		&& source->sun_path[sizeof(source->sun_path) - 1] == '\0'
@@ -114,7 +112,7 @@ static int	wait_for_response(pid_t server_pid, uint32_t kind, uint32_t token,
 static int	request_session(pid_t server_pid, const char *server_path)
 {
 	struct sockaddr_un	address;
-	t_mt_request			request;
+	t_mt_request		request;
 
 	memset(&address, 0, sizeof(address));
 	address.sun_family = AF_UNIX;
@@ -131,32 +129,6 @@ static int	request_session(pid_t server_pid, const char *server_path)
 			server_path));
 }
 
-static void	handle_response(int signal)
-{
-	if (signal == MT_ACK_SIGNAL)
-		g_ack_received = 1;
-	else if (signal == MT_NACK_SIGNAL)
-		g_rejected = 1;
-	else if (signal == SIGALRM)
-		g_timed_out = 1;
-}
-
-static int	install_handlers(void)
-{
-	struct sigaction	action;
-
-	action.sa_handler = handle_response;
-	sigemptyset(&action.sa_mask);
-	action.sa_flags = 0;
-	if (sigaction(MT_ACK_SIGNAL, &action, NULL) == -1)
-		return (-1);
-	if (sigaction(MT_NACK_SIGNAL, &action, NULL) == -1)
-		return (-1);
-	if (sigaction(SIGALRM, &action, NULL) == -1)
-		return (-1);
-	return (0);
-}
-
 static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
 		const char *server_path)
 {
@@ -167,21 +139,17 @@ static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
 	if (bit != 0)
 		signal = MT_ONE_SIGNAL;
 	if (kill(server_pid, signal) == -1
-		|| wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
-			server_path) == -1)
+		|| wait_for_response(server_pid, MT_RESPONSE_ACK, sequence, server_path) == -1)
 		return (-1);
 	gap.tv_sec = 0;
 	gap.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
-	while (nanosleep(&gap, &gap) == -1)
-	{
-		if (errno != EINTR)
-			return (-1);
-	}
+	while (nanosleep(&gap, &gap) == -1 && errno == EINTR)
+		;
 	return (0);
 }
 
-static int	send_byte(pid_t server_pid, unsigned char byte,
-		uint32_t *sequence, const char *server_path)
+static int	send_byte(pid_t server_pid, unsigned char byte, uint32_t *sequence,
+		const char *server_path)
 {
 	int	shift;
 
@@ -208,26 +176,9 @@ static int	send_partial(pid_t server_pid, uint32_t *sequence,
 	return (0);
 }
 
-static int	prepare_masks(sigset_t *wait_mask)
-{
-	sigset_t	blocked;
-
-	sigemptyset(&blocked);
-	sigaddset(&blocked, MT_ACK_SIGNAL);
-	sigaddset(&blocked, MT_NACK_SIGNAL);
-	sigaddset(&blocked, SIGALRM);
-	if (sigprocmask(SIG_BLOCK, &blocked, wait_mask) == -1)
-		return (-1);
-	sigdelset(wait_mask, MT_ACK_SIGNAL);
-	sigdelset(wait_mask, MT_NACK_SIGNAL);
-	sigdelset(wait_mask, SIGALRM);
-	return (0);
-}
-
 int	main(int argc, char **argv)
 {
 	pid_t		server_pid;
-	sigset_t	wait_mask;
 	uint32_t	sequence;
 	char		server_path[MT_RESPONSE_PATH_SIZE];
 
@@ -235,8 +186,6 @@ int	main(int argc, char **argv)
 		|| mt_response_path(server_path, sizeof(server_path), "server",
 			server_pid) == -1 || prepare_socket() == -1 || atexit(cleanup) != 0)
 		return (1);
-	if (install_handlers() == -1 || prepare_masks(&wait_mask) == -1)
-		return (1);
 	if (request_session(server_pid, server_path) == -1)
 		return (1);
 	sequence = 0;


## `test(protocol): 응답 출처와 token 검증`

diff --git a/.gitignore b/.gitignore
index 7b06a0d..c7222f6 100644
--- a/.gitignore
+++ b/.gitignore
@@ -4,4 +4,5 @@ server
 client
 obj/
 tests/masked_exec
+tests/response_server
 tests/session_sender
diff --git a/Makefile b/Makefile
index ef06401..45fd582 100644
--- a/Makefile
+++ b/Makefile
@@ -2,6 +2,7 @@ NAME_SERVER := server
 NAME_CLIENT := client
 NAME_SESSION_SENDER := tests/session_sender
 NAME_MASKED_EXEC := tests/masked_exec
+NAME_RESPONSE_SERVER := tests/response_server
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -Iinclude
@@ -17,6 +18,7 @@ SERVER_OBJ := $(OBJ_DIR)/src/server.o $(COMMON_OBJ)
 CLIENT_OBJ := $(OBJ_DIR)/src/client.o $(COMMON_OBJ)
 SESSION_SENDER_OBJ := $(OBJ_DIR)/tests/session_sender.o $(COMMON_OBJ)
 MASKED_EXEC_OBJ := $(OBJ_DIR)/tests/masked_exec.o
+RESPONSE_SERVER_OBJ := $(OBJ_DIR)/tests/response_server.o $(COMMON_OBJ)
 
 .PHONY: all clean fclean re test
 
@@ -34,6 +36,9 @@ $(NAME_SESSION_SENDER): $(SESSION_SENDER_OBJ)
 $(NAME_MASKED_EXEC): $(MASKED_EXEC_OBJ)
 	$(CC) $(CFLAGS) $(MASKED_EXEC_OBJ) -o $@
 
+$(NAME_RESPONSE_SERVER): $(RESPONSE_SERVER_OBJ)
+	$(CC) $(CFLAGS) $(RESPONSE_SERVER_OBJ) -o $@
+
 $(OBJ_DIR)/%.o: %.c include/minitalk.h
 	mkdir -p $(dir $@)
 	$(CC) $(CFLAGS) -c $< -o $@
@@ -43,10 +48,11 @@ clean:
 
 fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT) $(NAME_SESSION_SENDER) \
-		$(NAME_MASKED_EXEC)
+		$(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER)
 
 re: fclean all
 
-test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC)
+test: all $(NAME_SESSION_SENDER) $(NAME_MASKED_EXEC) $(NAME_RESPONSE_SERVER)
 	sh tests/smoke.sh
 	sh tests/session_ownership.sh
+	sh tests/response_validation.sh
diff --git a/tests/response_server.c b/tests/response_server.c
new file mode 100644
index 0000000..c048f41
--- /dev/null
+++ b/tests/response_server.c
@@ -0,0 +1,279 @@
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
+#include <time.h>
+#include <unistd.h>
+
+typedef struct s_bit_event
+{
+	pid_t	sender;
+	int		signal;
+}	t_bit_event;
+
+static int	g_pipe[2] = {-1, -1};
+static int	g_server_socket = -1;
+static int	g_forger_socket = -1;
+static char	g_server_path[MT_RESPONSE_PATH_SIZE];
+static char	g_forger_path[MT_RESPONSE_PATH_SIZE];
+
+static void	cleanup(void)
+{
+	if (g_pipe[0] != -1)
+		close(g_pipe[0]);
+	if (g_pipe[1] != -1)
+		close(g_pipe[1]);
+	if (g_server_socket != -1)
+		close(g_server_socket);
+	if (g_forger_socket != -1)
+		close(g_forger_socket);
+	if (g_server_path[0] != '\0')
+		unlink(g_server_path);
+	if (g_forger_path[0] != '\0')
+		unlink(g_forger_path);
+}
+
+static int	set_nonblocking(int fd)
+{
+	int	flags;
+
+	flags = fcntl(fd, F_GETFL);
+	if (flags == -1 || fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	bind_path(int fd, const char *path)
+{
+	struct sockaddr_un	address;
+	struct stat			info;
+
+	if (lstat(path, &info) == 0)
+	{
+		if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid()
+			|| unlink(path) == -1)
+			return (-1);
+	}
+	else if (errno != ENOENT)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, path, mt_strlen(path) + 1);
+	return (bind(fd, (struct sockaddr *)&address, sizeof(address)));
+}
+
+static int	prepare(void)
+{
+	struct sigaction	action;
+
+	if (pipe(g_pipe) == -1 || set_nonblocking(g_pipe[1]) == -1)
+		return (-1);
+	g_server_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
+	g_forger_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (g_server_socket == -1 || g_forger_socket == -1
+		|| mt_response_path(g_server_path, sizeof(g_server_path), "server",
+			getpid()) == -1
+		|| mt_response_path(g_forger_path, sizeof(g_forger_path), "forger",
+			getpid()) == -1 || bind_path(g_server_socket, g_server_path) == -1
+		|| bind_path(g_forger_socket, g_forger_path) == -1)
+		return (-1);
+	memset(&action, 0, sizeof(action));
+	sigemptyset(&action.sa_mask);
+	sigaddset(&action.sa_mask, MT_ZERO_SIGNAL);
+	sigaddset(&action.sa_mask, MT_ONE_SIGNAL);
+	action.sa_flags = SA_SIGINFO;
+	return (0);
+}
+
+static void	handle_bit(int signal, siginfo_t *info, void *context)
+{
+	t_bit_event	event;
+	int				saved_errno;
+
+	saved_errno = errno;
+	(void)context;
+	if (info != NULL)
+	{
+		event.sender = info->si_pid;
+		event.signal = signal;
+		write(g_pipe[1], &event, sizeof(event));
+	}
+	errno = saved_errno;
+}
+
+static int	install_handlers(void)
+{
+	struct sigaction	action;
+
+	memset(&action, 0, sizeof(action));
+	action.sa_sigaction = handle_bit;
+	sigemptyset(&action.sa_mask);
+	sigaddset(&action.sa_mask, MT_ZERO_SIGNAL);
+	sigaddset(&action.sa_mask, MT_ONE_SIGNAL);
+	action.sa_flags = SA_SIGINFO;
+	if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1
+		|| sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	send_response(int socket_fd, pid_t client_pid,
+		t_mt_response *response)
+{
+	struct sockaddr_un	address;
+	char				path[MT_RESPONSE_PATH_SIZE];
+
+	if (mt_response_path(path, sizeof(path), "client", client_pid) == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, path, mt_strlen(path) + 1);
+	if (sendto(socket_fd, response, sizeof(*response), 0,
+			(struct sockaddr *)&address, sizeof(address))
+		!= (ssize_t)sizeof(*response))
+		return (-1);
+	return (0);
+}
+
+static int	reply_with_invalid_events(pid_t client_pid, uint32_t kind,
+		uint32_t token)
+{
+	t_mt_response	response;
+	struct timespec	pause_time;
+
+	response.magic = MT_RESPONSE_MAGIC;
+	response.kind = kind;
+	response.token = token;
+	response.status = MT_RESPONSE_OK;
+	response.server_pid = getpid();
+	if (send_response(g_forger_socket, client_pid, &response) == -1)
+		return (-1);
+	response.token = token + 1;
+	if (send_response(g_server_socket, client_pid, &response) == -1)
+		return (-1);
+	response.token = token;
+	response.magic = 0;
+	if (send_response(g_server_socket, client_pid, &response) == -1)
+		return (-1);
+	response.magic = MT_RESPONSE_MAGIC;
+	response.server_pid = getpid() + 1;
+	if (send_response(g_server_socket, client_pid, &response) == -1)
+		return (-1);
+	pause_time.tv_sec = 0;
+	pause_time.tv_nsec = 20000000L;
+	while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+		;
+	response.server_pid = getpid();
+	return (send_response(g_server_socket, client_pid, &response));
+}
+
+static int	receive_session_request(t_mt_request *request)
+{
+	unsigned char		payload[sizeof(*request) + 1];
+	struct sockaddr_un	source;
+	char				client_path[MT_RESPONSE_PATH_SIZE];
+	socklen_t			source_size;
+	ssize_t				size;
+
+	memset(&source, 0, sizeof(source));
+	source_size = sizeof(source);
+	size = recvfrom(g_server_socket, payload, sizeof(payload), 0,
+			(struct sockaddr *)&source, &source_size);
+	if (size != (ssize_t)sizeof(*request))
+		return (-1);
+	memcpy(request, payload, sizeof(*request));
+	if (request->magic != MT_RESPONSE_MAGIC
+		|| request->kind != MT_REQUEST_ACQUIRE
+		|| request->client_pid <= 1
+		|| mt_response_path(client_path, sizeof(client_path), "client",
+			request->client_pid) == -1
+		|| source.sun_family != AF_UNIX
+		|| strcmp(source.sun_path, client_path) != 0)
+		return (-1);
+	return (0);
+}
+
+static void	wait_for_client_cleanup(pid_t client_pid)
+{
+	struct timespec	pause_time;
+	struct stat		info;
+	char				path[MT_RESPONSE_PATH_SIZE];
+	int					tries;
+
+	if (mt_response_path(path, sizeof(path), "client", client_pid) == -1)
+		return ;
+	tries = 0;
+	while (tries < 50 && lstat(path, &info) == 0)
+	{
+		pause_time.tv_sec = 0;
+		pause_time.tv_nsec = 10000000L;
+		while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+			;
+		tries++;
+	}
+}
+
+int	main(void)
+{
+	t_bit_event		event;
+	t_mt_request	request;
+	t_mt_response	response;
+	uint32_t		sequence;
+	unsigned char	byte;
+	int				bits;
+	ssize_t			size;
+
+	if (prepare() == -1 || install_handlers() == -1 || atexit(cleanup) != 0)
+		return (1);
+	mt_putnbr_fd(getpid(), STDOUT_FILENO);
+	write(STDOUT_FILENO, "\n", 1);
+	if (receive_session_request(&request) == -1
+		|| reply_with_invalid_events(request.client_pid, MT_RESPONSE_READY,
+			request.nonce) == -1)
+		return (1);
+	sequence = 0;
+	byte = 0;
+	bits = 0;
+	while (1)
+	{
+		size = read(g_pipe[0], &event, sizeof(event));
+		if (size == -1 && errno == EINTR)
+			continue ;
+		if (size != (ssize_t)sizeof(event))
+			return (1);
+		byte <<= 1;
+		if (event.signal == MT_ONE_SIGNAL)
+			byte |= 1;
+		bits++;
+		response.magic = MT_RESPONSE_MAGIC;
+		response.kind = MT_RESPONSE_ACK;
+		response.token = sequence;
+		response.status = MT_RESPONSE_OK;
+		response.server_pid = getpid();
+		if ((sequence == 0 && reply_with_invalid_events(event.sender,
+				MT_RESPONSE_ACK, sequence) == -1) || (sequence != 0
+				&& send_response(g_server_socket, event.sender, &response) == -1))
+			return (1);
+		sequence++;
+		if (bits == 8)
+		{
+			if (byte == '\0')
+			{
+				write(STDOUT_FILENO, "\n", 1);
+				wait_for_client_cleanup(event.sender);
+				return (0);
+			}
+			write(STDOUT_FILENO, &byte, 1);
+			byte = 0;
+			bits = 0;
+		}
+	}
+	return (1);
+}
diff --git a/tests/response_validation.sh b/tests/response_validation.sh
new file mode 100755
index 0000000..6ae2ca0
--- /dev/null
+++ b/tests/response_validation.sh
@@ -0,0 +1,53 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/signal-message-bus-response.XXXXXX")
+OUT="$TEST_TMP/server.out"
+ERR="$TEST_TMP/server.err"
+CLIENT_ERR="$TEST_TMP/client.err"
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
+"$ROOT/tests/response_server" >"$OUT" 2>"$ERR" &
+SERVER_PID=$!
+tries=0
+while [ "$tries" -lt 50 ] && ! grep -qx "$SERVER_PID" "$OUT" 2>/dev/null; do
+	if ! kill -0 "$SERVER_PID" 2>/dev/null; then
+		printf 'response server exited before becoming ready\n' >&2
+		exit 1
+	fi
+	tries=$((tries + 1))
+	sleep 0.1
+done
+if ! grep -qx "$SERVER_PID" "$OUT"; then
+	printf 'response server did not become ready\n' >&2
+	exit 1
+fi
+
+"$ROOT/client" "$SERVER_PID" probe 2>"$CLIENT_ERR"
+wait "$SERVER_PID"
+SERVER_PID=
+
+if [ -s "$CLIENT_ERR" ] || [ -s "$ERR" ]; then
+	cat "$CLIENT_ERR" "$ERR" >&2
+	exit 1
+fi
+{
+	printf '%s\n' "$(sed -n '1p' "$OUT")"
+	printf 'probe\n'
+} >"$TEST_TMP/expected"
+diff -u "$TEST_TMP/expected" "$OUT"


