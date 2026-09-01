# 시그널 비트 인코딩과 NUL 프레이밍

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


## `test(smoke): 프로세스 간 메시지 전달 검증`

diff --git a/Makefile b/Makefile
index 3dd1efb..45f38c1 100644
--- a/Makefile
+++ b/Makefile
@@ -13,7 +13,7 @@ CLIENT_SRC := src/client.c $(COMMON_SRC)
 SERVER_OBJ := $(SERVER_SRC:%.c=$(OBJ_DIR)/%.o)
 CLIENT_OBJ := $(CLIENT_SRC:%.c=$(OBJ_DIR)/%.o)
 
-.PHONY: all clean fclean re
+.PHONY: all clean fclean re test
 
 all: $(NAME_SERVER) $(NAME_CLIENT)
 
@@ -34,3 +34,6 @@ fclean: clean
 	$(RM) $(NAME_SERVER) $(NAME_CLIENT)
 
 re: fclean all
+
+test: all
+	sh tests/smoke.sh
diff --git a/tests/smoke.sh b/tests/smoke.sh
new file mode 100755
index 0000000..e06d5ea
--- /dev/null
+++ b/tests/smoke.sh
@@ -0,0 +1,60 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TEST_TMP=$(mktemp -d "${TMPDIR:-/tmp}/minitalk-smoke.XXXXXX")
+OUT="$TEST_TMP/server.out"
+EXPECTED="$TEST_TMP/expected.out"
+SERVER_ERR="$TEST_TMP/server.err"
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
+"$ROOT/server" >"$OUT" 2>"$SERVER_ERR" &
+SERVER_PID=$!
+
+tries=0
+while [ "$tries" -lt 50 ]; do
+	if [ -s "$OUT" ] && [ "$(sed -n '1p' "$OUT")" = "$SERVER_PID" ]; then
+		break
+	fi
+	tries=$((tries + 1))
+	sleep 0.1
+done
+
+if ! kill -0 "$SERVER_PID" 2>/dev/null; then
+	printf 'server exited before smoke test\n' >&2
+	exit 1
+fi
+if [ ! -s "$OUT" ] || [ "$(sed -n '1p' "$OUT")" != "$SERVER_PID" ]; then
+	printf 'server did not publish its complete pid line\n' >&2
+	exit 1
+fi
+if ! "$ROOT/client" "$SERVER_PID" "hello" 2>"$CLIENT_ERR"; then
+	printf 'client failed to deliver hello\n' >&2
+	exit 1
+fi
+if [ -s "$CLIENT_ERR" ] || [ -s "$SERVER_ERR" ]; then
+	printf 'smoke test wrote unexpected diagnostics\n' >&2
+	exit 1
+fi
+
+{
+	printf '%s\n' "$SERVER_PID"
+	printf 'hello\n'
+} >"$EXPECTED"
+
+diff -u "$EXPECTED" "$OUT"


## `test(smoke): 빈 문자열과 장문 전송 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index e06d5ea..58ac9de 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -9,6 +9,20 @@ SERVER_ERR="$TEST_TMP/server.err"
 CLIENT_ERR="$TEST_TMP/client.err"
 SERVER_PID=
 
+send_checked()
+{
+	label=$1
+	message=$2
+	if ! "$ROOT/client" "$SERVER_PID" "$message" 2>"$CLIENT_ERR"; then
+		printf 'client failed during %s\n' "$label" >&2
+		exit 1
+	fi
+	if [ -s "$CLIENT_ERR" ]; then
+		printf 'client wrote to stderr during %s\n' "$label" >&2
+		exit 1
+	fi
+}
+
 cleanup()
 {
 	if [ -n "$SERVER_PID" ]; then
@@ -43,18 +57,25 @@ if [ ! -s "$OUT" ] || [ "$(sed -n '1p' "$OUT")" != "$SERVER_PID" ]; then
 	printf 'server did not publish its complete pid line\n' >&2
 	exit 1
 fi
-if ! "$ROOT/client" "$SERVER_PID" "hello" 2>"$CLIENT_ERR"; then
-	printf 'client failed to deliver hello\n' >&2
-	exit 1
-fi
-if [ -s "$CLIENT_ERR" ] || [ -s "$SERVER_ERR" ]; then
-	printf 'smoke test wrote unexpected diagnostics\n' >&2
+send_checked hello "hello"
+send_checked empty ""
+send_checked utf8 "안녕하세요"
+LONG_MESSAGE=$(awk 'BEGIN { for (i = 0; i < 1024; i++) printf "x" }')
+send_checked long "$LONG_MESSAGE"
+send_checked final "last message"
+
+if [ -s "$SERVER_ERR" ]; then
+	printf 'server wrote unexpected diagnostics\n' >&2
 	exit 1
 fi
 
 {
 	printf '%s\n' "$SERVER_PID"
 	printf 'hello\n'
+	printf '\n'
+	printf '안녕하세요\n'
+	printf '%s\n' "$LONG_MESSAGE"
+	printf 'last message\n'
 } >"$EXPECTED"
 
 diff -u "$EXPECTED" "$OUT"
