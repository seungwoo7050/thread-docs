# 등록 제한·유휴 감지·토큰 검증 하트비트

## `feat(registration): 등록 대기 시간 제한`

diff --git a/src/ClientRegistry.cpp b/src/ClientRegistry.cpp
index 1da3184..a6b7ab7 100644
--- a/src/ClientRegistry.cpp
+++ b/src/ClientRegistry.cpp
@@ -8,7 +8,8 @@ ClientState::ClientState()
       hasNick(false),
       hasUser(false),
       registered(false),
-      host("localhost") {
+      host("localhost"),
+      connectedAt(0) {
 }
 
 ClientState& ClientRegistry::state(int fd) {
diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
index 4fbff28..d1847d9 100644
--- a/src/ClientRegistry.hpp
+++ b/src/ClientRegistry.hpp
@@ -1,6 +1,7 @@
 #ifndef IRC_CLIENT_REGISTRY_HPP
 #define IRC_CLIENT_REGISTRY_HPP
 
+#include <ctime>
 #include <map>
 #include <string>
 #include <vector>
@@ -15,6 +16,7 @@ struct ClientState {
     std::string user;
     std::string realname;
     std::string host;
+    std::time_t connectedAt;
 
     ClientState();
 };
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index c88acf5..c90c93f 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -3,6 +3,8 @@
 #include "Connection.hpp"
 #include "IrcMessage.hpp"
 
+#include <ctime>
+
 IrcApplication::IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime)
     : _server(server),
       _password(password),
@@ -11,10 +13,12 @@ IrcApplication::IrcApplication(Server& server, const std::string& password, cons
 }
 
 void IrcApplication::onConnect(Connection& connection) {
+    const std::time_t now = std::time(NULL);
     ClientState client;
     client.fd = connection.fd();
     client.host = connection.peerAddress();
     client.passOk = _password.empty();
+    client.connectedAt = now;
     _clients.state(client.fd) = client;
 }
 
@@ -37,6 +41,14 @@ void IrcApplication::onDisconnect(Connection& connection, const std::string& rea
     removeClientState(connection.fd(), reason, true);
 }
 
+void IrcApplication::onTick() {
+    const std::time_t now = std::time(NULL);
+    const std::vector<int> fds = _clients.fds();
+    for (std::size_t i = 0; i < fds.size(); ++i) {
+        maintainClient(fds[i], now);
+    }
+}
+
 void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
     if (message.command == "PASS") {
         handlePass(fd, message);
@@ -72,3 +84,16 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
 }
+
+void IrcApplication::maintainClient(int fd, std::time_t now) {
+    ClientState* found = _clients.find(fd);
+    if (found == NULL) {
+        return;
+    }
+    ClientState& client = *found;
+    if (!client.registered &&
+        now - client.connectedAt >= _runtime.registrationTimeoutSeconds) {
+        sendNumeric(fd, 451, std::vector<std::string>(), "Registration timeout");
+        requestClose(fd, "registration timeout");
+    }
+}
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index baca520..4eedb1d 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -6,6 +6,7 @@
 #include "RuntimeConfig.hpp"
 #include "Server.hpp"
 
+#include <ctime>
 #include <map>
 #include <string>
 #include <vector>
@@ -19,6 +20,7 @@ public:
     void onConnect(Connection& connection);
     void onLine(Connection& connection, const std::string& line);
     void onDisconnect(Connection& connection, const std::string& reason);
+    void onTick();
 
 private:
     Server& _server;
@@ -32,6 +34,7 @@ private:
     static bool isChannelTarget(const std::string& target);
 
     void handleMessage(int fd, const IrcMessage& message);
+    void maintainClient(int fd, std::time_t now);
 
     void handlePass(int fd, const IrcMessage& message);
     void handleNick(int fd, const IrcMessage& message);
diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index 466c267..114bae9 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -13,7 +13,8 @@ RuntimeConfig::RuntimeConfig()
 }
 
 void RuntimeConfig::printUsage(const char* programName) {
-    std::cerr << "Usage: " << programName << " <port> <password>" << std::endl;
+    std::cerr << "Usage: " << programName << " <port> <password> "
+              << "[--registration-timeout=N]" << std::endl;
 }
 
 int RuntimeConfig::parsePort(const char* value) {
@@ -27,8 +28,27 @@ int RuntimeConfig::parsePort(const char* value) {
 
 RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config& serverConfig) {
     (void)serverConfig;
-    if (argc > 3) {
-        throw std::runtime_error(std::string("unknown option: ") + argv[3]);
+    RuntimeConfig runtime;
+    for (int i = 3; i < argc; ++i) {
+        const std::string arg(argv[i]);
+        if (startsWith(arg, "--registration-timeout=")) {
+            runtime.registrationTimeoutSeconds = parsePositiveInt(arg.substr(23), "registration timeout");
+        } else {
+            throw std::runtime_error("unknown option: " + arg);
+        }
     }
-    return RuntimeConfig();
+    return runtime;
+}
+
+int RuntimeConfig::parsePositiveInt(const std::string& value, const std::string& name) {
+    char* end = NULL;
+    const long parsed = std::strtol(value.c_str(), &end, 10);
+    if (value.empty() || *end != '\0' || parsed <= 0 || parsed > 86400) {
+        throw std::runtime_error(name + " must be a positive integer");
+    }
+    return static_cast<int>(parsed);
+}
+
+bool RuntimeConfig::startsWith(const std::string& value, const std::string& prefix) {
+    return value.size() >= prefix.size() && value.compare(0, prefix.size(), prefix) == 0;
 }
diff --git a/src/RuntimeConfig.hpp b/src/RuntimeConfig.hpp
index d4de818..bc06944 100644
--- a/src/RuntimeConfig.hpp
+++ b/src/RuntimeConfig.hpp
@@ -4,6 +4,7 @@
 #include "Server.hpp"
 
 #include <cstddef>
+#include <string>
 
 class RuntimeConfig {
 public:
@@ -18,6 +19,10 @@ public:
     static void printUsage(const char* programName);
     static int parsePort(const char* value);
     static RuntimeConfig parseOptions(int argc, char** argv, Server::Config& serverConfig);
+
+private:
+    static int parsePositiveInt(const std::string& value, const std::string& name);
+    static bool startsWith(const std::string& value, const std::string& prefix);
 };
 
 #endif // IRC_RUNTIME_CONFIG_HPP
diff --git a/src/main.cpp b/src/main.cpp
index cfeb210..52f0019 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -60,6 +60,7 @@ int main(int argc, char** argv) {
         std::cout << "Listening on port " << server.port() << std::endl;
         while (gRunning && server.isRunning()) {
             server.pollOnce();
+            app->onTick();
         }
         server.setConnectHandler(Server::ConnectHandler());
         server.setLineHandler(Server::LineHandler());


## `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`

diff --git a/src/ClientRegistry.cpp b/src/ClientRegistry.cpp
index a6b7ab7..fd8355a 100644
--- a/src/ClientRegistry.cpp
+++ b/src/ClientRegistry.cpp
@@ -8,8 +8,11 @@ ClientState::ClientState()
       hasNick(false),
       hasUser(false),
       registered(false),
+      awaitingPong(false),
       host("localhost"),
-      connectedAt(0) {
+      connectedAt(0),
+      lastActivityAt(0),
+      lastPingAt(0) {
 }
 
 ClientState& ClientRegistry::state(int fd) {
diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
index d1847d9..7fa483d 100644
--- a/src/ClientRegistry.hpp
+++ b/src/ClientRegistry.hpp
@@ -12,11 +12,14 @@ struct ClientState {
     bool hasNick;
     bool hasUser;
     bool registered;
+    bool awaitingPong;
     std::string nick;
     std::string user;
     std::string realname;
     std::string host;
     std::time_t connectedAt;
+    std::time_t lastActivityAt;
+    std::time_t lastPingAt;
 
     ClientState();
 };
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index c90c93f..39aa9cc 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -2,6 +2,7 @@
 
 #include "Connection.hpp"
 #include "IrcMessage.hpp"
+#include "Replies.hpp"
 
 #include <ctime>
 
@@ -19,6 +20,7 @@ void IrcApplication::onConnect(Connection& connection) {
     client.host = connection.peerAddress();
     client.passOk = _password.empty();
     client.connectedAt = now;
+    client.lastActivityAt = now;
     _clients.state(client.fd) = client;
 }
 
@@ -27,6 +29,7 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
     if (!_clients.contains(fd)) {
         onConnect(connection);
     }
+    _clients.state(fd).lastActivityAt = std::time(NULL);
 
     IrcMessage message;
     std::string parseError;
@@ -58,6 +61,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleUser(fd, message);
     } else if (message.command == "PING") {
         handlePing(fd, message);
+    } else if (message.command == "PONG") {
+        handlePong(fd, message);
     } else if (message.command == "QUIT") {
         handleQuit(fd, message);
     } else if (!_clients.state(fd).registered) {
@@ -95,5 +100,22 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
         now - client.connectedAt >= _runtime.registrationTimeoutSeconds) {
         sendNumeric(fd, 451, std::vector<std::string>(), "Registration timeout");
         requestClose(fd, "registration timeout");
+        return;
+    }
+    if (_runtime.idleTimeoutSeconds <= 0) {
+        return;
+    }
+    if (client.awaitingPong &&
+        now - client.lastPingAt >= _runtime.pingTimeoutSeconds) {
+        sendRaw(fd, Replies::error("Ping timeout"));
+        requestClose(fd, "ping timeout");
+        return;
+    }
+    if (!client.awaitingPong &&
+        now - client.lastActivityAt >= _runtime.idleTimeoutSeconds) {
+        const std::string token = "heartbeat-" + std::to_string(fd) + "-" + std::to_string(now);
+        sendRaw(fd, Replies::formatMessage(_serverName, "PING", std::vector<std::string>(1, token)));
+        client.awaitingPong = true;
+        client.lastPingAt = now;
     }
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 4eedb1d..97a94ff 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -40,6 +40,7 @@ private:
     void handleNick(int fd, const IrcMessage& message);
     void handleUser(int fd, const IrcMessage& message);
     void handlePing(int fd, const IrcMessage& message);
+    void handlePong(int fd, const IrcMessage& message);
     void handleQuit(int fd, const IrcMessage& message);
     void maybeRegister(int fd);
 
diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 3123054..a88e094 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -102,6 +102,12 @@ void IrcApplication::handlePing(int fd, const IrcMessage& message) {
     sendRaw(fd, Replies::formatMessage(_serverName, "PONG", params));
 }
 
+void IrcApplication::handlePong(int fd, const IrcMessage&) {
+    ClientState& client = _clients.state(fd);
+    client.awaitingPong = false;
+    client.lastPingAt = 0;
+}
+
 void IrcApplication::handleQuit(int fd, const IrcMessage& message) {
     const std::string reason = message.params.empty() ? "Client Quit" : message.params[0];
     requestClose(fd, reason);
diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index 114bae9..fcf2c9e 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -14,7 +14,8 @@ RuntimeConfig::RuntimeConfig()
 
 void RuntimeConfig::printUsage(const char* programName) {
     std::cerr << "Usage: " << programName << " <port> <password> "
-              << "[--registration-timeout=N]" << std::endl;
+              << "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N]"
+              << std::endl;
 }
 
 int RuntimeConfig::parsePort(const char* value) {
@@ -31,7 +32,11 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
     RuntimeConfig runtime;
     for (int i = 3; i < argc; ++i) {
         const std::string arg(argv[i]);
-        if (startsWith(arg, "--registration-timeout=")) {
+        if (startsWith(arg, "--idle-timeout=")) {
+            runtime.idleTimeoutSeconds = parsePositiveInt(arg.substr(15), "idle timeout");
+        } else if (startsWith(arg, "--ping-timeout=")) {
+            runtime.pingTimeoutSeconds = parsePositiveInt(arg.substr(15), "ping timeout");
+        } else if (startsWith(arg, "--registration-timeout=")) {
             runtime.registrationTimeoutSeconds = parsePositiveInt(arg.substr(23), "registration timeout");
         } else {
             throw std::runtime_error("unknown option: " + arg);


## `test(client): 서버 PING에 응답하는 검사 클라이언트 구현`

diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index 3ce6e29..c044ecd 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -1,5 +1,5 @@
 #!/usr/bin/env python3
-"""Small IRC smoke client for the educational reference server."""
+"""Small IRC smoke client for irc-relay-server."""
 
 import socket
 import sys
@@ -8,8 +8,9 @@ from typing import List
 
 
 class IrcPeer:
-    def __init__(self, host: str, port: int, label: str):
+    def __init__(self, host: str, port: int, label: str, auto_pong: bool = True):
         self.label = label
+        self.auto_pong = auto_pong
         self.sock = socket.create_connection((host, port), timeout=3.0)
         self.sock.settimeout(0.2)
         self.buffer = b""
@@ -35,6 +36,7 @@ class IrcPeer:
                 raw, self.buffer = self.buffer.split(b"\n", 1)
                 line = raw.rstrip(b"\r").decode("utf-8", "replace")
                 self.lines.append(line)
+                self._auto_reply_to_ping(line)
         return list(self.lines)
 
     def expect(self, needle: str, timeout: float = 2.0) -> str:
@@ -53,6 +55,19 @@ class IrcPeer:
         except OSError:
             pass
 
+    def _auto_reply_to_ping(self, line: str) -> None:
+        if not self.auto_pong:
+            return
+        token = None
+        if " PING " in line:
+            token = line.split(" PING ", 1)[1]
+        elif line.startswith("PING "):
+            token = line.split(" ", 1)[1]
+        if token is None:
+            return
+        token = token.lstrip(":")
+        self.send_line(f"PONG :{token}")
+
 
 def register(host: str, port: int, password: str, nick: str, realname: str) -> IrcPeer:
     peer = IrcPeer(host, port, nick)


## `test(client): 유휴 연결의 PING·PONG 흐름 검증`

diff --git a/tests/irc_smoke.sh b/tests/irc_smoke.sh
index 58f5396..39f50a0 100755
--- a/tests/irc_smoke.sh
+++ b/tests/irc_smoke.sh
@@ -24,7 +24,7 @@ cleanup() {
 trap cleanup EXIT
 
 make -C "${ROOT}" >/dev/null
-"${ROOT}/irc-relay-server" "${PORT}" "${PASSWORD}" >"${LOG_FILE}" 2>&1 &
+"${ROOT}/irc-relay-server" "${PORT}" "${PASSWORD}" --idle-timeout=1 --ping-timeout=2 --registration-timeout=5 >"${LOG_FILE}" 2>&1 &
 SERVER_PID="$!"
 
 python3 - <<PY
diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index c044ecd..1ff4b2a 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -163,6 +163,12 @@ def main() -> int:
         alice.send_line("PRIVMSG bob :direct hello")
         bob.expect(" PRIVMSG bob :direct hello")
 
+        idle = register(host, port, password, "idle", "Idle Tester")
+        peers.append(idle)
+        idle.expect(" PING ", timeout=5.0)
+        idle.send_line("PING :still-alive")
+        idle.expect(" PONG ")
+
         alice.send_line("QUIT :smoke complete")
         time.sleep(0.05)
         return 0


## `refactor(command): 명령 처리 시각 기록 통합`

diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index debcad5..c6ec80a 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -33,7 +33,8 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
     if (!_clients.contains(fd)) {
         onConnect(connection);
     }
-    _clients.state(fd).lastActivityAt = std::time(NULL);
+    const std::time_t now = std::time(NULL);
+    _clients.state(fd).lastActivityAt = now;
 
     IrcMessage message;
     std::string parseError;
@@ -41,7 +42,7 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
         sendNumeric(fd, 417, std::vector<std::string>(), parseError);
         return;
     }
-    if (!recordCommand(fd, std::time(NULL))) {
+    if (!recordCommand(fd, now)) {
         return;
     }
     ++_metrics.commandsHandled;


## `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`

diff --git a/src/ClientRegistry.cpp b/src/ClientRegistry.cpp
index fd8355a..63d75c1 100644
--- a/src/ClientRegistry.cpp
+++ b/src/ClientRegistry.cpp
@@ -9,10 +9,7 @@ ClientState::ClientState()
       hasUser(false),
       registered(false),
       awaitingPong(false),
-      host("localhost"),
-      connectedAt(0),
-      lastActivityAt(0),
-      lastPingAt(0) {
+      host("localhost") {
 }
 
 ClientState& ClientRegistry::state(int fd) {
diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
index bbcd9e4..a49e217 100644
--- a/src/ClientRegistry.hpp
+++ b/src/ClientRegistry.hpp
@@ -1,12 +1,15 @@
 #ifndef IRC_CLIENT_REGISTRY_HPP
 #define IRC_CLIENT_REGISTRY_HPP
 
-#include <ctime>
+#include <chrono>
 #include <deque>
 #include <map>
 #include <string>
 #include <vector>
 
+typedef std::chrono::steady_clock MonotonicClock;
+typedef MonotonicClock::time_point MonotonicTime;
+
 struct ClientState {
     int fd;
     bool passOk;
@@ -18,10 +21,11 @@ struct ClientState {
     std::string user;
     std::string realname;
     std::string host;
-    std::time_t connectedAt;
-    std::time_t lastActivityAt;
-    std::time_t lastPingAt;
-    std::deque<std::time_t> commandWindow;
+    std::string pendingPongToken;
+    MonotonicTime connectedAt;
+    MonotonicTime lastActivityAt;
+    MonotonicTime lastPingAt;
+    std::deque<MonotonicTime> commandWindow;
 
     ClientState();
 };
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index c6ec80a..f9a54a7 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -4,17 +4,18 @@
 #include "IrcMessage.hpp"
 #include "Replies.hpp"
 
-#include <ctime>
+#include <chrono>
 
 IrcApplication::IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime)
     : _server(server),
       _password(password),
       _runtime(runtime),
-      _serverName("irc.relay.local") {
+      _serverName("irc.relay.local"),
+      _nextHeartbeatToken(0) {
 }
 
 void IrcApplication::onConnect(Connection& connection) {
-    const std::time_t now = std::time(NULL);
+    const MonotonicTime now = MonotonicClock::now();
     ClientState client;
     client.fd = connection.fd();
     client.host = connection.peerAddress();
@@ -33,7 +34,7 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
     if (!_clients.contains(fd)) {
         onConnect(connection);
     }
-    const std::time_t now = std::time(NULL);
+    const MonotonicTime now = MonotonicClock::now();
     _clients.state(fd).lastActivityAt = now;
 
     IrcMessage message;
@@ -58,7 +59,7 @@ void IrcApplication::onDisconnect(Connection& connection, const std::string& rea
 }
 
 void IrcApplication::onTick() {
-    const std::time_t now = std::time(NULL);
+    const MonotonicTime now = MonotonicClock::now();
     const std::vector<int> fds = _clients.fds();
     for (std::size_t i = 0; i < fds.size(); ++i) {
         maintainClient(fds[i], now);
@@ -131,14 +132,14 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
     }
 }
 
-void IrcApplication::maintainClient(int fd, std::time_t now) {
+void IrcApplication::maintainClient(int fd, const MonotonicTime& now) {
     ClientState* found = _clients.find(fd);
     if (found == NULL) {
         return;
     }
     ClientState& client = *found;
     if (!client.registered &&
-        now - client.connectedAt >= _runtime.registrationTimeoutSeconds) {
+        now - client.connectedAt >= std::chrono::seconds(_runtime.registrationTimeoutSeconds)) {
         sendNumeric(fd, 451, std::vector<std::string>(), "Registration timeout");
         requestClose(fd, "registration timeout");
         return;
@@ -147,7 +148,7 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
         return;
     }
     if (client.awaitingPong &&
-        now - client.lastPingAt >= _runtime.pingTimeoutSeconds) {
+        now - client.lastPingAt >= std::chrono::seconds(_runtime.pingTimeoutSeconds)) {
         ++_metrics.idleTimeouts;
         sendRaw(fd, Replies::error("Ping timeout"));
         requestClose(fd, "ping timeout");
@@ -158,19 +159,22 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
         return;
     }
     if (!client.awaitingPong &&
-        now - client.lastActivityAt >= _runtime.idleTimeoutSeconds) {
-        const std::string token = "heartbeat-" + std::to_string(fd) + "-" + std::to_string(now);
+        now - client.lastActivityAt >= std::chrono::seconds(_runtime.idleTimeoutSeconds)) {
+        const std::string token =
+            "heartbeat-" + std::to_string(fd) + "-" + std::to_string(++_nextHeartbeatToken);
         sendRaw(fd, Replies::formatMessage(_serverName, "PING", std::vector<std::string>(1, token)));
         client.awaitingPong = true;
+        client.pendingPongToken = token;
         client.lastPingAt = now;
         ++_metrics.heartbeatPings;
     }
 }
 
-bool IrcApplication::recordCommand(int fd, std::time_t now) {
+bool IrcApplication::recordCommand(int fd, const MonotonicTime& now) {
     ClientState& client = _clients.state(fd);
     while (!client.commandWindow.empty() &&
-           now - client.commandWindow.front() >= _runtime.rateLimitWindowSeconds) {
+           now - client.commandWindow.front() >=
+               std::chrono::seconds(_runtime.rateLimitWindowSeconds)) {
         client.commandWindow.pop_front();
     }
     client.commandWindow.push_back(now);
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index af3e7f0..ccd87bc 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -7,7 +7,7 @@
 #include "Server.hpp"
 
 #include <cstddef>
-#include <ctime>
+#include <cstdint>
 #include <map>
 #include <string>
 #include <utility>
@@ -50,13 +50,14 @@ private:
     ClientRegistry _clients;
     std::map<std::string, Channel> _channels;
     AppMetrics _metrics;
+    std::uint64_t _nextHeartbeatToken;
 
     static std::vector<std::string> splitComma(const std::string& value);
     static bool isChannelTarget(const std::string& target);
 
     void handleMessage(int fd, const IrcMessage& message);
-    void maintainClient(int fd, std::time_t now);
-    bool recordCommand(int fd, std::time_t now);
+    void maintainClient(int fd, const MonotonicTime& now);
+    bool recordCommand(int fd, const MonotonicTime& now);
 
     void handlePass(int fd, const IrcMessage& message);
     void handleNick(int fd, const IrcMessage& message);
diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 07ba9e3..3104228 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -103,10 +103,16 @@ void IrcApplication::handlePing(int fd, const IrcMessage& message) {
     sendRaw(fd, Replies::formatMessage(_serverName, "PONG", params));
 }
 
-void IrcApplication::handlePong(int fd, const IrcMessage&) {
+void IrcApplication::handlePong(int fd, const IrcMessage& message) {
     ClientState& client = _clients.state(fd);
+    if (!client.awaitingPong ||
+        message.params.size() != 1 ||
+        message.params[0] != client.pendingPongToken) {
+        return;
+    }
     client.awaitingPong = false;
-    client.lastPingAt = 0;
+    client.pendingPongToken.clear();
+    client.lastPingAt = MonotonicTime();
 }
 
 void IrcApplication::handleQuit(int fd, const IrcMessage& message) {


