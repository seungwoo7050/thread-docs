# 비동기 서버의 결정적 계층 검증과 크로스플랫폼 CI

## `test(smoke): 실제 TCP 등록과 채널 흐름 검증`

diff --git a/Makefile b/Makefile
index 249a047..8733958 100644
--- a/Makefile
+++ b/Makefile
@@ -23,7 +23,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all clean fclean re
+.PHONY: all clean fclean re test smoke
 
 all: $(NAME)
 
@@ -33,6 +33,11 @@ $(NAME): $(OBJS)
 %.o: %.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -MP -c $< -o $@
 
+test: all
+	bash tests/irc_smoke.sh
+
+smoke: test
+
 clean:
 	rm -f $(OBJS) $(DEPS)
 
diff --git a/tests/irc_smoke.sh b/tests/irc_smoke.sh
new file mode 100755
index 0000000..58f5396
--- /dev/null
+++ b/tests/irc_smoke.sh
@@ -0,0 +1,47 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
+PORT="${IRC_SMOKE_PORT:-$(python3 - <<'PY'
+import socket
+s = socket.socket()
+s.bind(("127.0.0.1", 0))
+print(s.getsockname()[1])
+s.close()
+PY
+)}"
+PASSWORD="${IRC_SMOKE_PASSWORD:-educational-secret}"
+LOG_FILE="$(mktemp -t irc_smoke_server.XXXXXX.log)"
+SERVER_PID=""
+
+cleanup() {
+	if [[ -n "${SERVER_PID}" ]] && kill -0 "${SERVER_PID}" 2>/dev/null; then
+		kill "${SERVER_PID}" 2>/dev/null || true
+		wait "${SERVER_PID}" 2>/dev/null || true
+	fi
+	rm -f "${LOG_FILE}"
+}
+trap cleanup EXIT
+
+make -C "${ROOT}" >/dev/null
+"${ROOT}/irc-relay-server" "${PORT}" "${PASSWORD}" >"${LOG_FILE}" 2>&1 &
+SERVER_PID="$!"
+
+python3 - <<PY
+import socket
+import time
+
+host = "127.0.0.1"
+port = int("${PORT}")
+deadline = time.time() + 5
+while time.time() < deadline:
+    try:
+        with socket.create_connection((host, port), timeout=0.2):
+            raise SystemExit(0)
+    except OSError:
+        time.sleep(0.05)
+raise SystemExit("server did not accept connections")
+PY
+
+python3 "${ROOT}/tools/irc_smoke_client.py" 127.0.0.1 "${PORT}" "${PASSWORD}"
+echo "IRC smoke passed on port ${PORT}"
diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
new file mode 100755
index 0000000..c51944b
--- /dev/null
+++ b/tools/irc_smoke_client.py
@@ -0,0 +1,154 @@
+#!/usr/bin/env python3
+"""Small IRC smoke client for the educational reference server."""
+
+import socket
+import sys
+import time
+from typing import List
+
+
+class IrcPeer:
+    def __init__(self, host: str, port: int, label: str):
+        self.label = label
+        self.sock = socket.create_connection((host, port), timeout=3.0)
+        self.sock.settimeout(0.2)
+        self.buffer = b""
+        self.lines: List[str] = []
+
+    def send_line(self, line: str) -> None:
+        self.sock.sendall(line.encode("utf-8") + b"\r\n")
+
+    def send_raw(self, data: bytes) -> None:
+        self.sock.sendall(data)
+
+    def read_available(self, duration: float = 0.05) -> List[str]:
+        deadline = time.time() + duration
+        while time.time() < deadline:
+            try:
+                chunk = self.sock.recv(4096)
+            except socket.timeout:
+                break
+            if not chunk:
+                break
+            self.buffer += chunk
+            while b"\n" in self.buffer:
+                raw, self.buffer = self.buffer.split(b"\n", 1)
+                line = raw.rstrip(b"\r").decode("utf-8", "replace")
+                self.lines.append(line)
+        return list(self.lines)
+
+    def expect(self, needle: str, timeout: float = 2.0) -> str:
+        deadline = time.time() + timeout
+        while time.time() < deadline:
+            for index, line in enumerate(self.lines):
+                if needle in line:
+                    return self.lines.pop(index)
+            self.read_available(0.1)
+        transcript = "\n".join(self.lines[-20:])
+        raise AssertionError(f"{self.label}: expected {needle!r}; recent lines:\n{transcript}")
+
+    def close(self) -> None:
+        try:
+            self.sock.close()
+        except OSError:
+            pass
+
+
+def register(host: str, port: int, password: str, nick: str, realname: str) -> IrcPeer:
+    peer = IrcPeer(host, port, nick)
+    peer.send_line(f"PASS {password}")
+    peer.send_line(f"NICK {nick}")
+    peer.send_line(f"USER {nick} 0 * :{realname}")
+    peer.expect(" 001 ")
+    return peer
+
+
+def main() -> int:
+    if len(sys.argv) != 4:
+        print(f"usage: {sys.argv[0]} <host> <port> <password>", file=sys.stderr)
+        return 2
+
+    host = sys.argv[1]
+    port = int(sys.argv[2])
+    password = sys.argv[3]
+
+    peers: List[IrcPeer] = []
+    try:
+        wrong = IrcPeer(host, port, "wrong-password")
+        wrong.send_line("PASS wrong-password")
+        wrong.expect(" 464 ")
+        wrong.close()
+
+        alice = register(host, port, password, "alice", "Alice Learner")
+        peers.append(alice)
+
+        alice.send_raw(b"PI")
+        time.sleep(0.05)
+        alice.send_raw(b"NG :half-frame\r\n")
+        pong = alice.expect("PONG")
+        if "half-frame" not in pong:
+            raise AssertionError(f"alice: split PING response did not echo token: {pong}")
+
+        dup = register(host, port, password, "dupe", "Dupe One")
+        peers.append(dup)
+        collision = IrcPeer(host, port, "collision")
+        collision.send_line(f"PASS {password}")
+        collision.send_line("NICK dupe")
+        collision.expect(" 433 ")
+        collision.close()
+
+        alice.send_line("JOIN #edu")
+        alice.expect(" JOIN #edu")
+        alice.expect(" 353 ")
+        alice.expect(" 366 ")
+
+        bob = register(host, port, password, "bob", "Bob Learner")
+        carol = register(host, port, password, "carol", "Carol Learner")
+        peers.extend([bob, carol])
+
+        bob.send_line("JOIN #edu")
+        bob.expect(" JOIN #edu")
+        alice.expect(":bob!")
+
+        alice.send_line("TOPIC #edu :Protocol lab")
+        alice.expect(" TOPIC #edu :Protocol lab")
+
+        alice.send_line("PRIVMSG #edu :hello channel")
+        bob.expect(" PRIVMSG #edu :hello channel")
+
+        alice.send_line("INVITE carol #edu")
+        alice.expect(" 341 ")
+        carol.expect(" INVITE carol #edu")
+        carol.send_line("JOIN #edu")
+        carol.expect(" JOIN #edu")
+
+        alice.send_line("MODE #edu +i")
+        alice.expect(" MODE #edu +i")
+        alice.send_line("MODE #edu +o bob")
+        bob.expect(" MODE #edu +o bob")
+
+        bob.send_line("TOPIC #edu :Bob can set topics")
+        bob.expect(" TOPIC #edu :Bob can set topics")
+
+        bob.send_line("KICK #edu carol :practice complete")
+        carol.expect(" KICK #edu carol :practice complete")
+
+        bob.send_line("PART #edu :done")
+        bob.expect(" PART #edu done")
+
+        alice.send_line("MODE #edu")
+        alice.expect(" 324 ")
+
+        alice.send_line("PRIVMSG bob :direct hello")
+        bob.expect(" PRIVMSG bob :direct hello")
+
+        alice.send_line("QUIT :smoke complete")
+        time.sleep(0.05)
+        return 0
+    finally:
+        for peer in peers:
+            peer.close()
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


