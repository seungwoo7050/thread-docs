# 계층형 자원 제한과 느린 수신자 격리

## `feat(throttle): 클라이언트별 명령 호출 횟수 제한`

diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
index 7fa483d..bbcd9e4 100644
--- a/src/ClientRegistry.hpp
+++ b/src/ClientRegistry.hpp
@@ -2,6 +2,7 @@
 #define IRC_CLIENT_REGISTRY_HPP
 
 #include <ctime>
+#include <deque>
 #include <map>
 #include <string>
 #include <vector>
@@ -20,6 +21,7 @@ struct ClientState {
     std::time_t connectedAt;
     std::time_t lastActivityAt;
     std::time_t lastPingAt;
+    std::deque<std::time_t> commandWindow;
 
     ClientState();
 };
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 39aa9cc..8167367 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -37,6 +37,9 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
         sendNumeric(fd, 417, std::vector<std::string>(), parseError);
         return;
     }
+    if (!recordCommand(fd, std::time(NULL))) {
+        return;
+    }
     handleMessage(fd, message);
 }
 
@@ -119,3 +122,18 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
         client.lastPingAt = now;
     }
 }
+
+bool IrcApplication::recordCommand(int fd, std::time_t now) {
+    ClientState& client = _clients.state(fd);
+    while (!client.commandWindow.empty() &&
+           now - client.commandWindow.front() >= _runtime.rateLimitWindowSeconds) {
+        client.commandWindow.pop_front();
+    }
+    client.commandWindow.push_back(now);
+    if (_runtime.rateLimitCount != 0 && client.commandWindow.size() > _runtime.rateLimitCount) {
+        sendNumeric(fd, 439, std::vector<std::string>(), "Command rate limit exceeded");
+        requestClose(fd, "command rate limit exceeded");
+        return false;
+    }
+    return true;
+}
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 97a94ff..f0db53d 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -35,6 +35,7 @@ private:
 
     void handleMessage(int fd, const IrcMessage& message);
     void maintainClient(int fd, std::time_t now);
+    bool recordCommand(int fd, std::time_t now);
 
     void handlePass(int fd, const IrcMessage& message);
     void handleNick(int fd, const IrcMessage& message);
diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index fcf2c9e..4752802 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -14,8 +14,8 @@ RuntimeConfig::RuntimeConfig()
 
 void RuntimeConfig::printUsage(const char* programName) {
     std::cerr << "Usage: " << programName << " <port> <password> "
-              << "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N]"
-              << std::endl;
+              << "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N] "
+              << "[--rate-limit=COUNT:SECONDS]" << std::endl;
 }
 
 int RuntimeConfig::parsePort(const char* value) {
@@ -38,6 +38,15 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
             runtime.pingTimeoutSeconds = parsePositiveInt(arg.substr(15), "ping timeout");
         } else if (startsWith(arg, "--registration-timeout=")) {
             runtime.registrationTimeoutSeconds = parsePositiveInt(arg.substr(23), "registration timeout");
+        } else if (startsWith(arg, "--rate-limit=")) {
+            const std::string value = arg.substr(13);
+            const std::string::size_type colon = value.find(':');
+            if (colon == std::string::npos) {
+                throw std::runtime_error("rate limit must use COUNT:SECONDS");
+            }
+            runtime.rateLimitCount = parseSize(value.substr(0, colon), "rate limit count");
+            runtime.rateLimitWindowSeconds =
+                parsePositiveInt(value.substr(colon + 1), "rate limit window");
         } else {
             throw std::runtime_error("unknown option: " + arg);
         }
@@ -45,6 +54,15 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
     return runtime;
 }
 
+std::size_t RuntimeConfig::parseSize(const std::string& value, const std::string& name) {
+    char* end = NULL;
+    const unsigned long parsed = std::strtoul(value.c_str(), &end, 10);
+    if (value.empty() || *end != '\0') {
+        throw std::runtime_error(name + " must be an unsigned integer");
+    }
+    return static_cast<std::size_t>(parsed);
+}
+
 int RuntimeConfig::parsePositiveInt(const std::string& value, const std::string& name) {
     char* end = NULL;
     const long parsed = std::strtol(value.c_str(), &end, 10);
diff --git a/src/RuntimeConfig.hpp b/src/RuntimeConfig.hpp
index bc06944..1ffdde1 100644
--- a/src/RuntimeConfig.hpp
+++ b/src/RuntimeConfig.hpp
@@ -21,6 +21,7 @@ public:
     static RuntimeConfig parseOptions(int argc, char** argv, Server::Config& serverConfig);
 
 private:
+    static std::size_t parseSize(const std::string& value, const std::string& name);
     static int parsePositiveInt(const std::string& value, const std::string& name);
     static bool startsWith(const std::string& value, const std::string& prefix);
 };


## `test(client): 명령 호출 횟수 제한 검증`

diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index 1ff4b2a..751c842 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -29,6 +29,8 @@ class IrcPeer:
                 chunk = self.sock.recv(4096)
             except socket.timeout:
                 break
+            except OSError:
+                break
             if not chunk:
                 break
             self.buffer += chunk
@@ -169,6 +171,12 @@ def main() -> int:
         idle.send_line("PING :still-alive")
         idle.expect(" PONG ")
 
+        flood = register(host, port, password, "flood", "Flood Tester")
+        for index in range(25):
+            flood.send_line(f"PING :burst-{index}")
+        flood.expect(" 439 ")
+        flood.close()
+
         alice.send_line("QUIT :smoke complete")
         time.sleep(0.05)
         return 0


## `feat(buffer): 송신 대기열 크기 제한`

diff --git a/include/Connection.hpp b/include/Connection.hpp
index 4f88c4f..2a68e81 100644
--- a/include/Connection.hpp
+++ b/include/Connection.hpp
@@ -24,7 +24,10 @@ public:
         std::string error;
     };
 
-    Connection(int fd, std::string peerAddress, std::size_t maxLineLength = 512);
+    Connection(int fd,
+               std::string peerAddress,
+               std::size_t maxLineLength = 512,
+               std::size_t maxPendingBytes = 1048576);
     ~Connection();
 
     Connection(const Connection&) = delete;
@@ -40,8 +43,8 @@ public:
     ReadResult readAvailable();
     WriteResult flushPending();
 
-    void queueRaw(const std::string& bytes);
-    void queueLine(const std::string& line);
+    bool queueRaw(const std::string& bytes);
+    bool queueLine(const std::string& line);
 
     void requestClose(std::string reason = "connection close requested");
     bool closeRequested() const noexcept;
@@ -55,12 +58,14 @@ private:
     std::string writeBuffer_;
     std::size_t writeOffset_;
     std::size_t maxLineLength_;
+    std::size_t maxPendingBytes_;
     bool peerClosed_;
     bool closeRequested_;
     std::string closeReason_;
 
     void closeFd() noexcept;
     bool extractLines(ReadResult& result);
+    bool canAppendPending(std::size_t byteCount) const noexcept;
 };
 
 } // namespace irc
diff --git a/include/Server.hpp b/include/Server.hpp
index 8798cc7..4cb1376 100644
--- a/include/Server.hpp
+++ b/include/Server.hpp
@@ -21,6 +21,7 @@ public:
         int backlog = 128;
         int eventTimeoutMs = 1000;
         std::size_t maxLineLength = 512;
+        std::size_t maxPendingBytes = 1048576;
     };
 
     using ConnectHandler = std::function<void(Connection&)>;
diff --git a/src/Connection.cpp b/src/Connection.cpp
index 72aee8b..b542eeb 100644
--- a/src/Connection.cpp
+++ b/src/Connection.cpp
@@ -29,11 +29,15 @@ int sendFlags()
 
 } // namespace
 
-Connection::Connection(int fd, std::string peerAddress, std::size_t maxLineLength)
+Connection::Connection(int fd,
+                       std::string peerAddress,
+                       std::size_t maxLineLength,
+                       std::size_t maxPendingBytes)
     : fd_(fd)
     , peerAddress_(std::move(peerAddress))
     , writeOffset_(0)
     , maxLineLength_(maxLineLength == 0 ? 512 : maxLineLength)
+    , maxPendingBytes_(maxPendingBytes == 0 ? 1048576 : maxPendingBytes)
     , peerClosed_(false)
     , closeRequested_(false)
 {
@@ -51,6 +55,7 @@ Connection::Connection(Connection&& other) noexcept
     , writeBuffer_(std::move(other.writeBuffer_))
     , writeOffset_(other.writeOffset_)
     , maxLineLength_(other.maxLineLength_)
+    , maxPendingBytes_(other.maxPendingBytes_)
     , peerClosed_(other.peerClosed_)
     , closeRequested_(other.closeRequested_)
     , closeReason_(std::move(other.closeReason_))
@@ -69,6 +74,7 @@ Connection& Connection::operator=(Connection&& other) noexcept
         writeBuffer_ = std::move(other.writeBuffer_);
         writeOffset_ = other.writeOffset_;
         maxLineLength_ = other.maxLineLength_;
+        maxPendingBytes_ = other.maxPendingBytes_;
         peerClosed_ = other.peerClosed_;
         closeRequested_ = other.closeRequested_;
         closeReason_ = std::move(other.closeReason_);
@@ -183,19 +189,29 @@ Connection::WriteResult Connection::flushPending()
     return result;
 }
 
-void Connection::queueRaw(const std::string& bytes)
+bool Connection::queueRaw(const std::string& bytes)
 {
+    if (!canAppendPending(bytes.size())) {
+        requestClose("outbound queue limit exceeded");
+        return false;
+    }
     writeBuffer_.append(bytes);
+    return true;
 }
 
-void Connection::queueLine(const std::string& line)
+bool Connection::queueLine(const std::string& line)
 {
     std::size_t end = line.size();
     while (end > 0 && (line[end - 1] == '\r' || line[end - 1] == '\n')) {
         --end;
     }
+    if (!canAppendPending(end + 2)) {
+        requestClose("outbound queue limit exceeded");
+        return false;
+    }
     writeBuffer_.append(line, 0, end);
     writeBuffer_.append("\r\n");
+    return true;
 }
 
 void Connection::requestClose(std::string reason)
@@ -259,4 +275,9 @@ bool Connection::extractLines(ReadResult& result)
     }
 }
 
+bool Connection::canAppendPending(std::size_t byteCount) const noexcept
+{
+    return pendingBytes() + byteCount <= maxPendingBytes_;
+}
+
 } // namespace irc
diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index 4752802..98e2e3c 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -15,7 +15,7 @@ RuntimeConfig::RuntimeConfig()
 void RuntimeConfig::printUsage(const char* programName) {
     std::cerr << "Usage: " << programName << " <port> <password> "
               << "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N] "
-              << "[--rate-limit=COUNT:SECONDS]" << std::endl;
+              << "[--rate-limit=COUNT:SECONDS] [--max-pending-bytes=N]" << std::endl;
 }
 
 int RuntimeConfig::parsePort(const char* value) {
@@ -28,7 +28,6 @@ int RuntimeConfig::parsePort(const char* value) {
 }
 
 RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config& serverConfig) {
-    (void)serverConfig;
     RuntimeConfig runtime;
     for (int i = 3; i < argc; ++i) {
         const std::string arg(argv[i]);
@@ -47,6 +46,8 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
             runtime.rateLimitCount = parseSize(value.substr(0, colon), "rate limit count");
             runtime.rateLimitWindowSeconds =
                 parsePositiveInt(value.substr(colon + 1), "rate limit window");
+        } else if (startsWith(arg, "--max-pending-bytes=")) {
+            serverConfig.maxPendingBytes = parseSize(arg.substr(20), "max pending bytes");
         } else {
             throw std::runtime_error("unknown option: " + arg);
         }
diff --git a/src/Server.cpp b/src/Server.cpp
index 83d0f3a..0475d0a 100644
--- a/src/Server.cpp
+++ b/src/Server.cpp
@@ -208,9 +208,9 @@ bool Server::sendTo(int fd, const std::string& line)
     if (connection == NULL) {
         return false;
     }
-    connection->queueLine(line);
+    const bool queued = connection->queueLine(line);
     refreshInterest(*connection);
-    return true;
+    return queued;
 }
 
 bool Server::queueRawTo(int fd, const std::string& bytes)
@@ -219,9 +219,9 @@ bool Server::queueRawTo(int fd, const std::string& bytes)
     if (connection == NULL) {
         return false;
     }
-    connection->queueRaw(bytes);
+    const bool queued = connection->queueRaw(bytes);
     refreshInterest(*connection);
-    return true;
+    return queued;
 }
 
 void Server::disconnect(int fd, const std::string& reason)
@@ -349,7 +349,8 @@ void Server::acceptReadyClients()
             std::unique_ptr<Connection> connection(new Connection(
                 clientFd,
                 formatPeerAddress(peerStorage),
-                config_.maxLineLength));
+                config_.maxLineLength,
+                config_.maxPendingBytes));
             clientFd = -1;
 
             const int fd = connection->fd();


## `feat(server): 최대 연결 수 제한`

diff --git a/include/Server.hpp b/include/Server.hpp
index 4cb1376..c12266b 100644
--- a/include/Server.hpp
+++ b/include/Server.hpp
@@ -22,6 +22,7 @@ public:
         int eventTimeoutMs = 1000;
         std::size_t maxLineLength = 512;
         std::size_t maxPendingBytes = 1048576;
+        std::size_t maxConnections = 256;
     };
 
     using ConnectHandler = std::function<void(Connection&)>;
@@ -73,6 +74,7 @@ private:
 
     void createListenSocket();
     void acceptReadyClients();
+    void rejectReadyClient();
     void handleClientEvent(const Event& event);
     void refreshInterest(Connection& connection);
     void closeListenSocket() noexcept;
diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index 98e2e3c..982e41a 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -15,7 +15,8 @@ RuntimeConfig::RuntimeConfig()
 void RuntimeConfig::printUsage(const char* programName) {
     std::cerr << "Usage: " << programName << " <port> <password> "
               << "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N] "
-              << "[--rate-limit=COUNT:SECONDS] [--max-pending-bytes=N]" << std::endl;
+              << "[--rate-limit=COUNT:SECONDS] [--max-pending-bytes=N] [--max-connections=N]"
+              << std::endl;
 }
 
 int RuntimeConfig::parsePort(const char* value) {
@@ -48,6 +49,8 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
                 parsePositiveInt(value.substr(colon + 1), "rate limit window");
         } else if (startsWith(arg, "--max-pending-bytes=")) {
             serverConfig.maxPendingBytes = parseSize(arg.substr(20), "max pending bytes");
+        } else if (startsWith(arg, "--max-connections=")) {
+            serverConfig.maxConnections = parseSize(arg.substr(18), "max connections");
         } else {
             throw std::runtime_error("unknown option: " + arg);
         }
diff --git a/src/Server.cpp b/src/Server.cpp
index 0475d0a..08fc053 100644
--- a/src/Server.cpp
+++ b/src/Server.cpp
@@ -341,6 +341,12 @@ void Server::acceptReadyClients()
             return;
         }
 
+        if (config_.maxConnections != 0 && connections_.size() >= config_.maxConnections) {
+            rejectReadyClient();
+            ::close(clientFd);
+            continue;
+        }
+
         try {
             setCloseOnExec(clientFd);
             setNonBlocking(clientFd);
@@ -378,6 +384,11 @@ void Server::acceptReadyClients()
     }
 }
 
+void Server::rejectReadyClient()
+{
+    reportError("connection rejected: max connection count reached");
+}
+
 void Server::handleClientEvent(const Event& event)
 {
     std::unordered_map<int, std::unique_ptr<Connection> >::iterator found =


## `test(smoke): 서버 보호 옵션으로 실행 검증`

diff --git a/tests/irc_smoke.sh b/tests/irc_smoke.sh
index 39f50a0..1d7d470 100755
--- a/tests/irc_smoke.sh
+++ b/tests/irc_smoke.sh
@@ -10,8 +10,8 @@ print(s.getsockname()[1])
 s.close()
 PY
 )}"
-PASSWORD="${IRC_SMOKE_PASSWORD:-educational-secret}"
-LOG_FILE="$(mktemp -t irc_smoke_server.XXXXXX.log)"
+PASSWORD="${IRC_SMOKE_PASSWORD:-relay-secret}"
+LOG_FILE="$(mktemp "${TMPDIR:-/tmp}/irc_smoke_server.XXXXXX")"
 SERVER_PID=""
 
 cleanup() {
@@ -24,7 +24,13 @@ cleanup() {
 trap cleanup EXIT
 
 make -C "${ROOT}" >/dev/null
-"${ROOT}/irc-relay-server" "${PORT}" "${PASSWORD}" --idle-timeout=1 --ping-timeout=2 --registration-timeout=5 >"${LOG_FILE}" 2>&1 &
+"${ROOT}/irc-relay-server" "${PORT}" "${PASSWORD}" \
+	--idle-timeout=2 \
+	--ping-timeout=2 \
+	--registration-timeout=5 \
+	--rate-limit=24:3 \
+	--max-pending-bytes=1048576 \
+	>"${LOG_FILE}" 2>&1 &
 SERVER_PID="$!"
 
 python3 - <<PY


