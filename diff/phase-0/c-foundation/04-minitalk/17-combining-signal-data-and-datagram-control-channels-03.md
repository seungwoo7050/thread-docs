## `perf(protocol): 검증된 ACK 뒤 고정 지연 제거`

diff --git a/include/minitalk.h b/include/minitalk.h
index 05aafb8..6eecede 100644
--- a/include/minitalk.h
+++ b/include/minitalk.h
@@ -9,7 +9,6 @@
 # define MT_ZERO_SIGNAL SIGUSR1
 # define MT_ONE_SIGNAL SIGUSR2
 # define MT_ACK_TIMEOUT_SECONDS 3
-# define MT_SIGNAL_GAP_US 5000
 # define MT_RESPONSE_MAGIC 0x4d54414bU
 # define MT_REQUEST_ACQUIRE 1U
 # define MT_RESPONSE_READY 1U
diff --git a/src/client.c b/src/client.c
index ddd0e60..0423605 100644
--- a/src/client.c
+++ b/src/client.c
@@ -30,15 +30,6 @@ static void	cleanup_response_socket(void)
 	g_client_path[0] = '\0';
 }
 
-static void	wait_signal_gap(void)
-{
-	struct timespec	remaining;
-
-	remaining.tv_sec = 0;
-	remaining.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
-	while (nanosleep(&remaining, &remaining) == -1 && errno == EINTR)
-		;
-}
 
 static int	set_nonblocking_close_on_exec(int fd)
 {
@@ -284,7 +275,6 @@ static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
 			server_path, &deadline);
 	if (status != 0)
 		return (status);
-	wait_signal_gap();
 	return (0);
 }
 
diff --git a/tests/session_sender.c b/tests/session_sender.c
index 9a5a55d..331cba3 100644
--- a/tests/session_sender.c
+++ b/tests/session_sender.c
@@ -132,7 +132,6 @@ static int	request_session(pid_t server_pid, const char *server_path)
 static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
 		const char *server_path)
 {
-	struct timespec	gap;
 	int				signal;
 
 	signal = MT_ZERO_SIGNAL;
@@ -141,10 +140,6 @@ static int	send_bit(pid_t server_pid, int bit, uint32_t sequence,
 	if (kill(server_pid, signal) == -1
 		|| wait_for_response(server_pid, MT_RESPONSE_ACK, sequence, server_path) == -1)
 		return (-1);
-	gap.tv_sec = 0;
-	gap.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
-	while (nanosleep(&gap, &gap) == -1 && errno == EINTR)
-		;
 	return (0);
 }
 
