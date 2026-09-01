# 실행 옵션과 오버플로 안전 CLI 파싱

## `feat(config): 기본 실행 인자 해석 모듈 구성`

diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
new file mode 100644
index 0000000..466c267
--- /dev/null
+++ b/src/RuntimeConfig.cpp
@@ -0,0 +1,34 @@
+#include "RuntimeConfig.hpp"
+
+#include <cstdlib>
+#include <iostream>
+#include <stdexcept>
+
+RuntimeConfig::RuntimeConfig()
+    : rateLimitCount(24),
+      rateLimitWindowSeconds(3),
+      idleTimeoutSeconds(120),
+      pingTimeoutSeconds(30),
+      registrationTimeoutSeconds(30) {
+}
+
+void RuntimeConfig::printUsage(const char* programName) {
+    std::cerr << "Usage: " << programName << " <port> <password>" << std::endl;
+}
+
+int RuntimeConfig::parsePort(const char* value) {
+    char* end = NULL;
+    const long port = std::strtol(value, &end, 10);
+    if (!value[0] || *end != '\0' || port <= 0 || port > 65535) {
+        throw std::runtime_error("port must be an integer from 1 to 65535");
+    }
+    return static_cast<int>(port);
+}
+
+RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config& serverConfig) {
+    (void)serverConfig;
+    if (argc > 3) {
+        throw std::runtime_error(std::string("unknown option: ") + argv[3]);
+    }
+    return RuntimeConfig();
+}
diff --git a/src/RuntimeConfig.hpp b/src/RuntimeConfig.hpp
new file mode 100644
index 0000000..d4de818
--- /dev/null
+++ b/src/RuntimeConfig.hpp
@@ -0,0 +1,23 @@
+#ifndef IRC_RUNTIME_CONFIG_HPP
+#define IRC_RUNTIME_CONFIG_HPP
+
+#include "Server.hpp"
+
+#include <cstddef>
+
+class RuntimeConfig {
+public:
+    std::size_t rateLimitCount;
+    int rateLimitWindowSeconds;
+    int idleTimeoutSeconds;
+    int pingTimeoutSeconds;
+    int registrationTimeoutSeconds;
+
+    RuntimeConfig();
+
+    static void printUsage(const char* programName);
+    static int parsePort(const char* value);
+    static RuntimeConfig parseOptions(int argc, char** argv, Server::Config& serverConfig);
+};
+
+#endif // IRC_RUNTIME_CONFIG_HPP


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


