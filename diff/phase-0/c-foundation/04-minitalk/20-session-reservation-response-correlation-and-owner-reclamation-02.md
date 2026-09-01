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


## `test(protocol): session sender에 명시적 세션 획득 적용`

diff --git a/tests/session_sender.c b/tests/session_sender.c
index cfea664..5937520 100644
--- a/tests/session_sender.c
+++ b/tests/session_sender.c
@@ -3,14 +3,133 @@
 #include "minitalk.h"
 
 #include <errno.h>
+#include <fcntl.h>
 #include <signal.h>
+#include <stdlib.h>
 #include <string.h>
+#include <sys/select.h>
+#include <sys/socket.h>
+#include <sys/stat.h>
+#include <sys/un.h>
 #include <time.h>
 #include <unistd.h>
 
 static volatile sig_atomic_t	g_ack_received;
 static volatile sig_atomic_t	g_timed_out;
 static volatile sig_atomic_t	g_rejected;
+static int						g_socket = -1;
+static char						g_path[MT_RESPONSE_PATH_SIZE];
+
+static void	cleanup(void)
+{
+	if (g_socket != -1)
+		close(g_socket);
+	if (g_path[0] != '\0')
+		unlink(g_path);
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
+static int	prepare_socket(void)
+{
+	struct sockaddr_un	address;
+	struct stat			info;
+
+	if (mt_response_path(g_path, sizeof(g_path), "client", getpid()) == -1)
+		return (-1);
+	if (lstat(g_path, &info) == 0)
+	{
+		if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid()
+			|| unlink(g_path) == -1)
+			return (-1);
+	}
+	else if (errno != ENOENT)
+		return (-1);
+	g_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
+	if (g_socket == -1 || set_nonblocking(g_socket) == -1)
+		return (-1);
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, g_path, mt_strlen(g_path) + 1);
+	if (bind(g_socket, (struct sockaddr *)&address, sizeof(address)) == -1)
+		return (-1);
+	return (0);
+}
+
+static int	response_matches(const t_mt_response *response,
+		const struct sockaddr_un *source, pid_t server_pid, uint32_t kind,
+		uint32_t token, const char *server_path)
+{
+	return (source->sun_family == AF_UNIX
+		&& source->sun_path[sizeof(source->sun_path) - 1] == '\0'
+		&& strcmp(source->sun_path, server_path) == 0
+		&& response->magic == MT_RESPONSE_MAGIC
+		&& response->server_pid == server_pid
+		&& response->kind == kind
+		&& response->token == token
+		&& response->status == MT_RESPONSE_OK);
+}
+
+static int	wait_for_response(pid_t server_pid, uint32_t kind, uint32_t token,
+		const char *server_path)
+{
+	struct sockaddr_un	source;
+	t_mt_response		response;
+	struct timeval		timeout;
+	fd_set				read_set;
+	socklen_t			source_size;
+	ssize_t				size;
+	int					status;
+
+	while (1)
+	{
+		FD_ZERO(&read_set);
+		FD_SET(g_socket, &read_set);
+		timeout.tv_sec = MT_ACK_TIMEOUT_SECONDS;
+		timeout.tv_usec = 0;
+		status = select(g_socket + 1, &read_set, NULL, NULL, &timeout);
+		if (status == -1 && errno == EINTR)
+			continue ;
+		if (status <= 0)
+			return (-1);
+		memset(&source, 0, sizeof(source));
+		source_size = sizeof(source);
+		size = recvfrom(g_socket, &response, sizeof(response), 0,
+				(struct sockaddr *)&source, &source_size);
+		if (size == (ssize_t)sizeof(response)
+			&& response_matches(&response, &source, server_pid, kind, token,
+				server_path))
+			return (0);
+	}
+}
+
+static int	request_session(pid_t server_pid, const char *server_path)
+{
+	struct sockaddr_un	address;
+	t_mt_request			request;
+
+	memset(&address, 0, sizeof(address));
+	address.sun_family = AF_UNIX;
+	memcpy(address.sun_path, server_path, mt_strlen(server_path) + 1);
+	request.magic = MT_RESPONSE_MAGIC;
+	request.kind = MT_REQUEST_ACQUIRE;
+	request.nonce = (uint32_t)getpid() ^ 0xa5a55a5aU;
+	request.client_pid = getpid();
+	if (sendto(g_socket, &request, sizeof(request), 0,
+			(struct sockaddr *)&address, sizeof(address))
+		!= (ssize_t)sizeof(request))
+		return (-1);
+	return (wait_for_response(server_pid, MT_RESPONSE_READY, request.nonce,
+			server_path));
+}
 
 static void	handle_response(int signal)
 {
@@ -113,11 +232,16 @@ int	main(int argc, char **argv)
 {
 	pid_t		server_pid;
 	sigset_t	wait_mask;
+	char		server_path[MT_RESPONSE_PATH_SIZE];
 
-	if (argc != 3 || !mt_parse_pid(argv[1], &server_pid))
+	if (argc != 3 || !mt_parse_pid(argv[1], &server_pid)
+		|| mt_response_path(server_path, sizeof(server_path), "server",
+			server_pid) == -1 || prepare_socket() == -1 || atexit(cleanup) != 0)
 		return (1);
 	if (install_handlers() == -1 || prepare_masks(&wait_mask) == -1)
 		return (1);
+	if (request_session(server_pid, server_path) == -1)
+		return (1);
 	if (strcmp(argv[2], "bit") == 0)
 	{
 		if (wait_for_ack(server_pid, 0, &wait_mask) == -1)


## `test(protocol): session sender를 sequence ACK로 전환`

diff --git a/tests/session_sender.c b/tests/session_sender.c
index 5937520..8e55b46 100644
--- a/tests/session_sender.c
+++ b/tests/session_sender.c
@@ -157,7 +157,8 @@ static int	install_handlers(void)
 	return (0);
 }
 
-static int	wait_for_ack(pid_t server_pid, int bit, const sigset_t *wait_mask)
+static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
+		const char *server_path)
 {
 	struct timespec	gap;
 	int				signal;
@@ -165,16 +166,9 @@ static int	wait_for_ack(pid_t server_pid, int bit, const sigset_t *wait_mask)
 	signal = MT_ZERO_SIGNAL;
 	if (bit != 0)
 		signal = MT_ONE_SIGNAL;
-	g_ack_received = 0;
-	g_timed_out = 0;
-	g_rejected = 0;
-	if (kill(server_pid, signal) == -1)
-		return (-1);
-	alarm(MT_ACK_TIMEOUT_SECONDS);
-	while (!g_ack_received && !g_timed_out && !g_rejected)
-		sigsuspend(wait_mask);
-	alarm(0);
-	if (!g_ack_received || g_timed_out || g_rejected)
+	if (kill(server_pid, signal) == -1
+		|| wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
+			server_path) == -1)
 		return (-1);
 	gap.tv_sec = 0;
 	gap.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
@@ -187,29 +181,31 @@ static int	wait_for_ack(pid_t server_pid, int bit, const sigset_t *wait_mask)
 }
 
 static int	send_byte(pid_t server_pid, unsigned char byte,
-		const sigset_t *wait_mask)
+		uint32_t *sequence, const char *server_path)
 {
 	int	shift;
 
 	shift = 7;
 	while (shift >= 0)
 	{
-		if (wait_for_ack(server_pid, (byte >> shift) & 1, wait_mask) == -1)
+		if (send_bit(server_pid, (byte >> shift) & 1, *sequence,
+				server_path) == -1)
 			return (-1);
+		(*sequence)++;
 		shift--;
 	}
 	return (0);
 }
 
-static int	send_partial(pid_t server_pid, const sigset_t *wait_mask)
+static int	send_partial(pid_t server_pid, uint32_t *sequence,
+		const char *server_path)
 {
-	if (send_byte(server_pid, 'X', wait_mask) == -1)
+	if (send_byte(server_pid, 'X', sequence, server_path) == -1
+		|| send_bit(server_pid, 0, (*sequence)++, server_path) == -1
+		|| send_bit(server_pid, 1, (*sequence)++, server_path) == -1
+		|| send_bit(server_pid, 0, (*sequence)++, server_path) == -1)
 		return (-1);
-	if (wait_for_ack(server_pid, 0, wait_mask) == -1)
-		return (-1);
-	if (wait_for_ack(server_pid, 1, wait_mask) == -1)
-		return (-1);
-	return (wait_for_ack(server_pid, 0, wait_mask));
+	return (0);
 }
 
 static int	prepare_masks(sigset_t *wait_mask)
@@ -232,6 +228,7 @@ int	main(int argc, char **argv)
 {
 	pid_t		server_pid;
 	sigset_t	wait_mask;
+	uint32_t	sequence;
 	char		server_path[MT_RESPONSE_PATH_SIZE];
 
 	if (argc != 3 || !mt_parse_pid(argv[1], &server_pid)
@@ -242,15 +239,16 @@ int	main(int argc, char **argv)
 		return (1);
 	if (request_session(server_pid, server_path) == -1)
 		return (1);
+	sequence = 0;
 	if (strcmp(argv[2], "bit") == 0)
 	{
-		if (wait_for_ack(server_pid, 0, &wait_mask) == -1)
+		if (send_bit(server_pid, 0, sequence, server_path) == -1)
 			return (1);
 	}
 	else if (strcmp(argv[2], "partial") == 0
 		|| strcmp(argv[2], "hold") == 0)
 	{
-		if (send_partial(server_pid, &wait_mask) == -1)
+		if (send_partial(server_pid, &sequence, server_path) == -1)
 			return (1);
 	}
 	else


