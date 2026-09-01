## `feat(client): 비트 ACK를 sequence로 상관 검증`

diff --git a/src/client.c b/src/client.c
index 918a333..6fb6f47 100644
--- a/src/client.c
+++ b/src/client.c
@@ -292,8 +292,8 @@ static int	request_session(pid_t server_pid, const char *server_path)
 			server_path, &deadline));
 }
 
-static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
-		uint32_t sequence, const char *server_path)
+static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
+		const char *server_path)
 {
 	struct timespec	deadline;
 	int				signal;
@@ -302,22 +302,13 @@ static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
 	signal = MT_ZERO_SIGNAL;
 	if (bit != 0)
 		signal = MT_ONE_SIGNAL;
-	g_ack_received = 0;
-	g_timed_out = 0;
-	g_rejected = 0;
-	if (kill(server_pid, signal) == -1)
+	if (validate_server_socket(server_path) == -1)
 		return (SEND_ERROR);
-	alarm(MT_ACK_TIMEOUT_SECONDS);
-	while (!g_ack_received && !g_timed_out && !g_rejected)
-		sigsuspend(old_mask);
-	alarm(0);
-	if (g_rejected)
-		return (SEND_REJECTED);
-	if (g_timed_out)
-		return (SEND_TIMEOUT);
 	if (clock_gettime(CLOCK_MONOTONIC, &deadline) == -1)
 		return (SEND_ERROR);
 	deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+	if (kill(server_pid, signal) == -1)
+		return (SEND_ERROR);
 	status = wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
 			server_path, &deadline);
 	if (status != 0)
@@ -327,7 +318,7 @@ static int	send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
 }
 
 static int	send_byte(pid_t server_pid, unsigned char byte,
-		const sigset_t *old_mask, uint32_t *sequence, const char *server_path)
+		uint32_t *sequence, const char *server_path)
 {
 	int	status;
 	int	shift;
@@ -335,8 +326,8 @@ static int	send_byte(pid_t server_pid, unsigned char byte,
 	shift = 7;
 	while (shift >= 0)
 	{
-		status = send_bit(server_pid, (byte >> shift) & 1, old_mask,
-				*sequence, server_path);
+		status = send_bit(server_pid, (byte >> shift) & 1, *sequence,
+				server_path);
 		if (status != 0)
 			return (status);
 		(*sequence)++;
@@ -414,12 +405,12 @@ int	main(int argc, char **argv)
 	while (argv[2][index] != '\0')
 	{
 		status = send_byte(server_pid, (unsigned char)argv[2][index],
-				&wait_mask, &sequence, server_path);
+				&sequence, server_path);
 		if (status != 0)
 			return (report_send_status(status));
 		index++;
 	}
-	status = send_byte(server_pid, '\0', &wait_mask, &sequence, server_path);
+	status = send_byte(server_pid, '\0', &sequence, server_path);
 	if (status != 0)
 		return (report_send_status(status));
 	return (0);


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


