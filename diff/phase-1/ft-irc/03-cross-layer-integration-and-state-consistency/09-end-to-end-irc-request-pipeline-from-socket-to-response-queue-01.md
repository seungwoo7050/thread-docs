# 소켓에서 응답 큐까지의 IRC 요청 파이프라인

## `docs(readme): 프로젝트 목표와 초기 개발 규약 정의`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..29348f6
--- /dev/null
+++ b/README.md
@@ -0,0 +1,20 @@
+# irc-relay-server
+
+단일 프로세스에서 여러 IRC 클라이언트의 연결과 메시지를 중계하는 C++17 서버를
+구현한다. 개발 과정에서는 운영체제별 이벤트 API를 공통 경계 뒤에 두고, 연결 수명과
+프로토콜 상태의 소유자를 명확하게 유지한다.
+
+## 초기 개발 규약
+
+- C++17과 `-Wall -Wextra -Werror`를 기본 컴파일 계약으로 사용한다.
+- Linux와 macOS의 논블로킹 이벤트 처리 경로를 함께 고려한다.
+- 소켓, 버퍼, 사용자 및 채널 상태는 수명과 정리 책임이 드러나는 객체가 소유한다.
+- 기능, 수정, 테스트, 문서와 CI 변경은 가능한 한 독립된 커밋으로 남긴다.
+- 각 개발 커밋은 깨끗한 상태에서 컴파일하고, 존재하는 검사를 모두 통과해야 한다.
+- 비밀번호나 생성된 빌드 산출물은 저장소에 기록하지 않는다.
+
+## 예정 범위
+
+첫 개발 범위는 TCP 연결 수락, IRC 프레임 처리, 사용자 등록, 개인·채널 메시지 중계와
+기본 채널 상태 관리다. 서버 간 연동, 영속 저장소, TLS 종료와 전체 IRC 표준 구현은
+초기 범위에 포함하지 않는다.


## `feat(app): IRC 동작 조율 계약 정의`

diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
new file mode 100644
index 0000000..989e1f2
--- /dev/null
+++ b/src/IrcApplication.hpp
@@ -0,0 +1,52 @@
+#ifndef IRC_APPLICATION_HPP
+#define IRC_APPLICATION_HPP
+
+#include "ClientRegistry.hpp"
+#include "RuntimeConfig.hpp"
+#include "Server.hpp"
+
+#include <string>
+#include <vector>
+
+class IrcMessage;
+
+class IrcApplication {
+public:
+    IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime);
+
+    void onConnect(Connection& connection);
+    void onLine(Connection& connection, const std::string& line);
+    void onDisconnect(Connection& connection, const std::string& reason);
+
+private:
+    Server& _server;
+    std::string _password;
+    RuntimeConfig _runtime;
+    std::string _serverName;
+    ClientRegistry _clients;
+
+    void handleMessage(int fd, const IrcMessage& message);
+
+    void handlePass(int fd, const IrcMessage& message);
+    void handleNick(int fd, const IrcMessage& message);
+    void handleUser(int fd, const IrcMessage& message);
+    void handlePing(int fd, const IrcMessage& message);
+    void handleQuit(int fd, const IrcMessage& message);
+    void maybeRegister(int fd);
+
+    std::string replyTarget(int fd) const;
+    std::string prefixFor(int fd) const;
+    std::string prefixFor(const ClientState& client) const;
+    void sendNumeric(
+        int fd,
+        int numericCode,
+        const std::vector<std::string>& params,
+        const std::string& trailing
+    );
+    void sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params);
+    void sendRaw(int fd, const std::string& line);
+    void requestClose(int fd, const std::string& reason);
+    void removeClientState(int fd, const std::string& reason, bool notifyPeers);
+};
+
+#endif // IRC_APPLICATION_HPP


## `feat(app): 연결 수명 콜백 조율`

diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
new file mode 100644
index 0000000..d284729
--- /dev/null
+++ b/src/IrcApplication.cpp
@@ -0,0 +1,38 @@
+#include "IrcApplication.hpp"
+
+#include "Connection.hpp"
+#include "IrcMessage.hpp"
+
+IrcApplication::IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime)
+    : _server(server),
+      _password(password),
+      _runtime(runtime),
+      _serverName("irc.reference.local") {
+}
+
+void IrcApplication::onConnect(Connection& connection) {
+    ClientState client;
+    client.fd = connection.fd();
+    client.host = connection.peerAddress();
+    client.passOk = _password.empty();
+    _clients.state(client.fd) = client;
+}
+
+void IrcApplication::onLine(Connection& connection, const std::string& line) {
+    const int fd = connection.fd();
+    if (!_clients.contains(fd)) {
+        onConnect(connection);
+    }
+
+    IrcMessage message;
+    std::string parseError;
+    if (!IrcMessage::parseLine(line, message, &parseError)) {
+        sendNumeric(fd, 417, std::vector<std::string>(), parseError);
+        return;
+    }
+    handleMessage(fd, message);
+}
+
+void IrcApplication::onDisconnect(Connection& connection, const std::string& reason) {
+    removeClientState(connection.fd(), reason, true);
+}


## `feat(app): 등록 전 명령 분배 구현`

diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index d284729..a61f9bb 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -36,3 +36,21 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
 void IrcApplication::onDisconnect(Connection& connection, const std::string& reason) {
     removeClientState(connection.fd(), reason, true);
 }
+
+void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
+    if (message.command == "PASS") {
+        handlePass(fd, message);
+    } else if (message.command == "NICK") {
+        handleNick(fd, message);
+    } else if (message.command == "USER") {
+        handleUser(fd, message);
+    } else if (message.command == "PING") {
+        handlePing(fd, message);
+    } else if (message.command == "QUIT") {
+        handleQuit(fd, message);
+    } else if (!_clients.state(fd).registered) {
+        sendNumeric(fd, 451, std::vector<std::string>(), "You have not registered");
+    } else {
+        sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
+    }
+}


## `feat(app): 응답과 클라이언트 정리 지원 구현`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
new file mode 100644
index 0000000..890a95a
--- /dev/null
+++ b/src/ApplicationSupport.cpp
@@ -0,0 +1,55 @@
+#include "IrcApplication.hpp"
+
+#include "Connection.hpp"
+#include "Replies.hpp"
+
+#include <vector>
+
+std::string IrcApplication::replyTarget(int fd) const {
+    const ClientState* client = _clients.find(fd);
+    if (client == NULL || client->nick.empty()) {
+        return "*";
+    }
+    return client->nick;
+}
+
+std::string IrcApplication::prefixFor(int fd) const {
+    const ClientState* client = _clients.find(fd);
+    if (client == NULL) {
+        return _serverName;
+    }
+    return prefixFor(*client);
+}
+
+std::string IrcApplication::prefixFor(const ClientState& client) const {
+    if (client.nick.empty()) {
+        return _serverName;
+    }
+    return Replies::hostmask(client.nick, client.user, client.host);
+}
+
+void IrcApplication::sendNumeric(int fd, int numericCode, const std::vector<std::string>& params, const std::string& trailing) {
+    sendRaw(fd, Replies::numeric(_serverName, replyTarget(fd), numericCode, params, trailing));
+}
+
+void IrcApplication::sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params) {
+    std::vector<std::string> allParams;
+    allParams.push_back(replyTarget(fd));
+    allParams.insert(allParams.end(), params.begin(), params.end());
+    sendRaw(fd, Replies::formatMessage(_serverName, Replies::code(numericCode), allParams));
+}
+
+void IrcApplication::sendRaw(int fd, const std::string& line) {
+    _server.sendTo(fd, line);
+}
+
+void IrcApplication::requestClose(int fd, const std::string& reason) {
+    Connection* connection = _server.findConnection(fd);
+    if (connection != NULL) {
+        connection->requestClose(reason);
+    }
+}
+
+void IrcApplication::removeClientState(int fd, const std::string&, bool) {
+    _clients.erase(fd);
+}


## `feat(server): IRC 애플리케이션 실행 진입점 구성`

diff --git a/src/main.cpp b/src/main.cpp
new file mode 100644
index 0000000..cfeb210
--- /dev/null
+++ b/src/main.cpp
@@ -0,0 +1,80 @@
+#include "IrcApplication.hpp"
+#include "RuntimeConfig.hpp"
+#include "Server.hpp"
+
+#include <cerrno>
+#include <csignal>
+#include <cstring>
+#include <iostream>
+#include <memory>
+#include <stdexcept>
+#include <string>
+
+namespace {
+    volatile std::sig_atomic_t gRunning = 1;
+    Server* gServer = NULL;
+
+    void handleSignal(int) {
+        gRunning = 0;
+        if (gServer != NULL) {
+            gServer->stop();
+        }
+    }
+}
+
+int main(int argc, char** argv) {
+    if (argc < 3) {
+        RuntimeConfig::printUsage(argv[0]);
+        return 1;
+    }
+
+    try {
+        std::signal(SIGINT, handleSignal);
+        std::signal(SIGTERM, handleSignal);
+        std::signal(SIGPIPE, SIG_IGN);
+
+        Server::Config config;
+        config.port = static_cast<unsigned short>(RuntimeConfig::parsePort(argv[1]));
+        RuntimeConfig runtime = RuntimeConfig::parseOptions(argc, argv, config);
+
+        std::unique_ptr<IrcApplication> app;
+        Server server(config);
+
+        gServer = &server;
+
+        app.reset(new IrcApplication(server, argv[2], runtime));
+        server.setConnectHandler([&app](Connection& connection) {
+            app->onConnect(connection);
+        });
+        server.setLineHandler([&app](Connection& connection, const std::string& line) {
+            app->onLine(connection, line);
+        });
+        server.setDisconnectHandler([&app](Connection& connection, const std::string& reason) {
+            app->onDisconnect(connection, reason);
+        });
+        server.setErrorHandler([](const std::string& message) {
+            std::cerr << "irc-relay-server: " << message << std::endl;
+        });
+
+        server.start();
+        std::cout << "Listening on port " << server.port() << std::endl;
+        while (gRunning && server.isRunning()) {
+            server.pollOnce();
+        }
+        server.setConnectHandler(Server::ConnectHandler());
+        server.setLineHandler(Server::LineHandler());
+        server.setDisconnectHandler(Server::DisconnectHandler());
+        server.setErrorHandler(Server::ErrorHandler());
+        server.stop();
+        gServer = NULL;
+    } catch (const std::exception& error) {
+        std::cerr << "irc-relay-server: " << error.what();
+        if (errno != 0) {
+            std::cerr << ": " << std::strerror(errno);
+        }
+        std::cerr << std::endl;
+        return 1;
+    }
+
+    return 0;
+}


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


