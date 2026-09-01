# 지표·구조화 로그·정상 종료

## `feat(metrics): 서버 실행 지표 조회 기능 추가`

diff --git a/include/Server.hpp b/include/Server.hpp
index c12266b..e01128f 100644
--- a/include/Server.hpp
+++ b/include/Server.hpp
@@ -25,6 +25,13 @@ public:
         std::size_t maxConnections = 256;
     };
 
+    struct Metrics {
+        std::size_t acceptedConnections = 0;
+        std::size_t closedConnections = 0;
+        std::size_t linesReceived = 0;
+        std::size_t outboundQueueDrops = 0;
+    };
+
     using ConnectHandler = std::function<void(Connection&)>;
     using LineHandler = std::function<void(Connection&, const std::string&)>;
     using DisconnectHandler = std::function<void(Connection&, const std::string&)>;
@@ -46,6 +53,7 @@ public:
     int listenFd() const noexcept;
     std::uint16_t port() const noexcept;
     std::size_t connectionCount() const noexcept;
+    const Metrics& metrics() const noexcept;
 
     void setConnectHandler(ConnectHandler handler);
     void setLineHandler(LineHandler handler);
@@ -66,6 +74,7 @@ private:
     std::unordered_map<int, std::unique_ptr<Connection> > connections_;
     bool running_;
     bool stopRequested_;
+    Metrics metrics_;
 
     ConnectHandler onConnect_;
     LineHandler onLine_;
diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index 611bfec..fb3a5af 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -8,6 +8,12 @@
 #include <utility>
 #include <vector>
 
+AppMetrics::AppMetrics()
+    : commandsHandled(0),
+      messagesRelayed(0),
+      rateLimitedClients(0) {
+}
+
 namespace {
     std::string joinWords(const std::vector<std::string>& values, const std::string& separator) {
         std::ostringstream out;
@@ -42,6 +48,23 @@ bool IrcApplication::isChannelTarget(const std::string& target) {
     return !target.empty() && (target[0] == '#' || target[0] == '&');
 }
 
+void IrcApplication::handleMetrics(int fd) {
+    const Server::Metrics& serverMetrics = _server.metrics();
+    std::ostringstream out;
+    out << "connections=" << _server.connectionCount()
+        << " accepted=" << serverMetrics.acceptedConnections
+        << " closed=" << serverMetrics.closedConnections
+        << " rooms=" << _channels.size()
+        << " commands=" << _metrics.commandsHandled
+        << " messages=" << _metrics.messagesRelayed
+        << " queue_drops=" << serverMetrics.outboundQueueDrops
+        << " rate_limited=" << _metrics.rateLimitedClients;
+    std::vector<std::string> params;
+    params.push_back(replyTarget(fd));
+    params.push_back(out.str());
+    sendRaw(fd, Replies::formatMessage(_serverName, "NOTICE", params));
+}
+
 Channel& IrcApplication::ensureChannel(const std::string& name) {
     std::map<std::string, Channel>::iterator it = _channels.find(name);
     if (it == _channels.end()) {
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 8167367..21b74af 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -40,6 +40,7 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
     if (!recordCommand(fd, std::time(NULL))) {
         return;
     }
+    ++_metrics.commandsHandled;
     handleMessage(fd, message);
 }
 
@@ -88,6 +89,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleList(fd, message);
     } else if (message.command == "NAMES") {
         handleNames(fd, message);
+    } else if (message.command == "METRICS") {
+        handleMetrics(fd);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
@@ -131,6 +134,7 @@ bool IrcApplication::recordCommand(int fd, std::time_t now) {
     }
     client.commandWindow.push_back(now);
     if (_runtime.rateLimitCount != 0 && client.commandWindow.size() > _runtime.rateLimitCount) {
+        ++_metrics.rateLimitedClients;
         sendNumeric(fd, 439, std::vector<std::string>(), "Command rate limit exceeded");
         requestClose(fd, "command rate limit exceeded");
         return false;
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index f0db53d..a36d7b0 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -6,6 +6,7 @@
 #include "RuntimeConfig.hpp"
 #include "Server.hpp"
 
+#include <cstddef>
 #include <ctime>
 #include <map>
 #include <string>
@@ -13,6 +14,14 @@
 
 class IrcMessage;
 
+struct AppMetrics {
+    std::size_t commandsHandled;
+    std::size_t messagesRelayed;
+    std::size_t rateLimitedClients;
+
+    AppMetrics();
+};
+
 class IrcApplication {
 public:
     IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime);
@@ -29,6 +38,7 @@ private:
     std::string _serverName;
     ClientRegistry _clients;
     std::map<std::string, Channel> _channels;
+    AppMetrics _metrics;
 
     static std::vector<std::string> splitComma(const std::string& value);
     static bool isChannelTarget(const std::string& target);
@@ -57,6 +67,7 @@ private:
     void handleNames(int fd, const IrcMessage& message);
     void handleChannelMode(int fd, const IrcMessage& message);
 
+    void handleMetrics(int fd);
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
     void sendTopicReply(int fd, const Channel& channel);
diff --git a/src/MessagingCommands.cpp b/src/MessagingCommands.cpp
index 5564cd4..4037bdf 100644
--- a/src/MessagingCommands.cpp
+++ b/src/MessagingCommands.cpp
@@ -28,6 +28,7 @@ void IrcApplication::handlePrivmsg(int fd, const IrcMessage& message) {
             params.push_back(target);
             params.push_back(message.params[1]);
             broadcastToChannel(target, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params), fd);
+            ++_metrics.messagesRelayed;
         } else {
             const int targetFd = findNick(target);
             if (targetFd == -1) {
@@ -38,6 +39,7 @@ void IrcApplication::handlePrivmsg(int fd, const IrcMessage& message) {
             params.push_back(target);
             params.push_back(message.params[1]);
             sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params));
+            ++_metrics.messagesRelayed;
         }
     }
 }
diff --git a/src/Server.cpp b/src/Server.cpp
index 08fc053..2baed67 100644
--- a/src/Server.cpp
+++ b/src/Server.cpp
@@ -182,6 +182,11 @@ std::size_t Server::connectionCount() const noexcept
     return connections_.size();
 }
 
+const Server::Metrics& Server::metrics() const noexcept
+{
+    return metrics_;
+}
+
 void Server::setConnectHandler(ConnectHandler handler)
 {
     onConnect_ = std::move(handler);
@@ -209,6 +214,9 @@ bool Server::sendTo(int fd, const std::string& line)
         return false;
     }
     const bool queued = connection->queueLine(line);
+    if (!queued) {
+        ++metrics_.outboundQueueDrops;
+    }
     refreshInterest(*connection);
     return queued;
 }
@@ -220,6 +228,9 @@ bool Server::queueRawTo(int fd, const std::string& bytes)
         return false;
     }
     const bool queued = connection->queueRaw(bytes);
+    if (!queued) {
+        ++metrics_.outboundQueueDrops;
+    }
     refreshInterest(*connection);
     return queued;
 }
@@ -241,6 +252,7 @@ void Server::disconnect(int fd, const std::string& reason)
 
     std::unique_ptr<Connection> connection = std::move(found->second);
     connections_.erase(found);
+    ++metrics_.closedConnections;
 
     if (onDisconnect_) {
         try {
@@ -363,6 +375,7 @@ void Server::acceptReadyClients()
             eventManager_->addFd(fd, EventInterest::Read);
             Connection* connectionPtr = connection.get();
             connections_[fd] = std::move(connection);
+            ++metrics_.acceptedConnections;
 
             if (onConnect_) {
                 try {
@@ -415,6 +428,7 @@ void Server::handleClientEvent(const Event& event)
         for (std::vector<std::string>::const_iterator line = readResult.lines.begin();
              line != readResult.lines.end();
              ++line) {
+            ++metrics_.linesReceived;
             if (onLine_) {
                 try {
                     onLine_(*connection, *line);


## `test(client): 서버 지표 명령 검증`

diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index 751c842..eb0b8bb 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -177,6 +177,9 @@ def main() -> int:
         flood.expect(" 439 ")
         flood.close()
 
+        alice.send_line("METRICS")
+        alice.expect(" NOTICE alice :connections=")
+
         alice.send_line("QUIT :smoke complete")
         time.sleep(0.05)
         return 0


## `feat(log): 연결 상태와 실행 지표 기록`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index fb3a5af..50372ea 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -3,6 +3,8 @@
 #include "Connection.hpp"
 #include "Replies.hpp"
 
+#include <cctype>
+#include <iostream>
 #include <set>
 #include <sstream>
 #include <utility>
@@ -11,7 +13,10 @@
 AppMetrics::AppMetrics()
     : commandsHandled(0),
       messagesRelayed(0),
-      rateLimitedClients(0) {
+      roomsCreated(0),
+      rateLimitedClients(0),
+      idleTimeouts(0),
+      heartbeatPings(0) {
 }
 
 namespace {
@@ -25,6 +30,17 @@ namespace {
         }
         return out.str();
     }
+
+    std::string logSafe(const std::string& value) {
+        std::string copy = value;
+        for (std::size_t i = 0; i < copy.size(); ++i) {
+            const unsigned char ch = static_cast<unsigned char>(copy[i]);
+            if (std::isspace(ch)) {
+                copy[i] = '_';
+            }
+        }
+        return copy;
+    }
 }
 
 std::vector<std::string> IrcApplication::splitComma(const std::string& value) {
@@ -48,6 +64,14 @@ bool IrcApplication::isChannelTarget(const std::string& target) {
     return !target.empty() && (target[0] == '#' || target[0] == '&');
 }
 
+void logEvent(const std::string& eventName, const std::vector<std::pair<std::string, std::string> >& fields) {
+    std::cerr << "event=" << eventName;
+    for (std::size_t i = 0; i < fields.size(); ++i) {
+        std::cerr << ' ' << fields[i].first << '=' << logSafe(fields[i].second);
+    }
+    std::cerr << std::endl;
+}
+
 void IrcApplication::handleMetrics(int fd) {
     const Server::Metrics& serverMetrics = _server.metrics();
     std::ostringstream out;
@@ -69,6 +93,10 @@ Channel& IrcApplication::ensureChannel(const std::string& name) {
     std::map<std::string, Channel>::iterator it = _channels.find(name);
     if (it == _channels.end()) {
         it = _channels.insert(std::make_pair(name, Channel(name))).first;
+        ++_metrics.roomsCreated;
+        logEvent("room_created", std::vector<std::pair<std::string, std::string> >{
+            std::make_pair("name", name)
+        });
     }
     return it->second;
 }
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index e1fe10f..a760134 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -22,6 +22,10 @@ void IrcApplication::onConnect(Connection& connection) {
     client.connectedAt = now;
     client.lastActivityAt = now;
     _clients.state(client.fd) = client;
+    logEvent("client_connected", std::vector<std::pair<std::string, std::string> >{
+        std::make_pair("fd", std::to_string(client.fd)),
+        std::make_pair("peer", client.host)
+    });
 }
 
 void IrcApplication::onLine(Connection& connection, const std::string& line) {
@@ -46,6 +50,10 @@ void IrcApplication::onLine(Connection& connection, const std::string& line) {
 
 void IrcApplication::onDisconnect(Connection& connection, const std::string& reason) {
     removeClientState(connection.fd(), reason, true);
+    logEvent("client_disconnected", std::vector<std::pair<std::string, std::string> >{
+        std::make_pair("fd", std::to_string(connection.fd())),
+        std::make_pair("reason", reason)
+    });
 }
 
 void IrcApplication::onTick() {
@@ -56,6 +64,23 @@ void IrcApplication::onTick() {
     }
 }
 
+void IrcApplication::logMetrics() const {
+    const Server::Metrics& serverMetrics = _server.metrics();
+    logEvent("server_metrics", std::vector<std::pair<std::string, std::string> >{
+        std::make_pair("accepted", std::to_string(serverMetrics.acceptedConnections)),
+        std::make_pair("closed", std::to_string(serverMetrics.closedConnections)),
+        std::make_pair("lines", std::to_string(serverMetrics.linesReceived)),
+        std::make_pair("queue_drops", std::to_string(serverMetrics.outboundQueueDrops)),
+        std::make_pair("commands", std::to_string(_metrics.commandsHandled)),
+        std::make_pair("messages", std::to_string(_metrics.messagesRelayed)),
+        std::make_pair("rooms", std::to_string(_channels.size())),
+        std::make_pair("rooms_created", std::to_string(_metrics.roomsCreated)),
+        std::make_pair("rate_limited", std::to_string(_metrics.rateLimitedClients)),
+        std::make_pair("idle_timeouts", std::to_string(_metrics.idleTimeouts)),
+        std::make_pair("heartbeats", std::to_string(_metrics.heartbeatPings))
+    });
+}
+
 void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
     if (message.command == "PASS") {
         handlePass(fd, message);
@@ -113,8 +138,13 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
     }
     if (client.awaitingPong &&
         now - client.lastPingAt >= _runtime.pingTimeoutSeconds) {
+        ++_metrics.idleTimeouts;
         sendRaw(fd, Replies::error("Ping timeout"));
         requestClose(fd, "ping timeout");
+        logEvent("client_ping_timeout", std::vector<std::pair<std::string, std::string> >{
+            std::make_pair("fd", std::to_string(fd)),
+            std::make_pair("nick", replyTarget(fd))
+        });
         return;
     }
     if (!client.awaitingPong &&
@@ -123,6 +153,7 @@ void IrcApplication::maintainClient(int fd, std::time_t now) {
         sendRaw(fd, Replies::formatMessage(_serverName, "PING", std::vector<std::string>(1, token)));
         client.awaitingPong = true;
         client.lastPingAt = now;
+        ++_metrics.heartbeatPings;
     }
 }
 
@@ -137,6 +168,10 @@ bool IrcApplication::recordCommand(int fd, std::time_t now) {
         ++_metrics.rateLimitedClients;
         sendNumeric(fd, 439, std::vector<std::string>(), "Command rate limit exceeded");
         requestClose(fd, "command rate limit exceeded");
+        logEvent("client_rate_limited", std::vector<std::pair<std::string, std::string> >{
+            std::make_pair("fd", std::to_string(fd)),
+            std::make_pair("nick", replyTarget(fd))
+        });
         return false;
     }
     return true;
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index a36d7b0..fe9a84d 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -10,6 +10,7 @@
 #include <ctime>
 #include <map>
 #include <string>
+#include <utility>
 #include <vector>
 
 class IrcMessage;
@@ -17,11 +18,19 @@ class IrcMessage;
 struct AppMetrics {
     std::size_t commandsHandled;
     std::size_t messagesRelayed;
+    std::size_t roomsCreated;
     std::size_t rateLimitedClients;
+    std::size_t idleTimeouts;
+    std::size_t heartbeatPings;
 
     AppMetrics();
 };
 
+void logEvent(
+    const std::string& eventName,
+    const std::vector<std::pair<std::string, std::string> >& fields
+);
+
 class IrcApplication {
 public:
     IrcApplication(Server& server, const std::string& password, const RuntimeConfig& runtime);
@@ -30,6 +39,7 @@ public:
     void onLine(Connection& connection, const std::string& line);
     void onDisconnect(Connection& connection, const std::string& reason);
     void onTick();
+    void logMetrics() const;
 
 private:
     Server& _server;
diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 3e90830..07ba9e3 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -4,6 +4,7 @@
 #include "Replies.hpp"
 
 #include <cctype>
+#include <utility>
 #include <vector>
 
 namespace {
@@ -122,4 +123,8 @@ void IrcApplication::maybeRegister(int fd) {
     sendNumeric(fd, 1, std::vector<std::string>(), "Welcome to irc-relay-server, " + client.nick);
     sendNumeric(fd, 2, std::vector<std::string>(), "Your host is " + _serverName);
     sendNumeric(fd, 3, std::vector<std::string>(), "This server is running a C++17 event backend");
+    logEvent("client_registered", std::vector<std::pair<std::string, std::string> >{
+        std::make_pair("fd", std::to_string(fd)),
+        std::make_pair("nick", client.nick)
+    });
 }
diff --git a/src/main.cpp b/src/main.cpp
index 52f0019..856978f 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -9,6 +9,8 @@
 #include <memory>
 #include <stdexcept>
 #include <string>
+#include <utility>
+#include <vector>
 
 namespace {
     volatile std::sig_atomic_t gRunning = 1;
@@ -53,15 +55,21 @@ int main(int argc, char** argv) {
             app->onDisconnect(connection, reason);
         });
         server.setErrorHandler([](const std::string& message) {
-            std::cerr << "irc-relay-server: " << message << std::endl;
+            logEvent("server_error", std::vector<std::pair<std::string, std::string> >{
+                std::make_pair("message", message)
+            });
         });
 
         server.start();
+        logEvent("server_started", std::vector<std::pair<std::string, std::string> >{
+            std::make_pair("port", std::to_string(server.port()))
+        });
         std::cout << "Listening on port " << server.port() << std::endl;
         while (gRunning && server.isRunning()) {
             server.pollOnce();
             app->onTick();
         }
+        app->logMetrics();
         server.setConnectHandler(Server::ConnectHandler());
         server.setLineHandler(Server::LineHandler());
         server.setDisconnectHandler(Server::DisconnectHandler());


## `feat(shutdown): 종료 전 송신 대기열 처리`

diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index a760134..debcad5 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -64,6 +64,15 @@ void IrcApplication::onTick() {
     }
 }
 
+void IrcApplication::shutdown(const std::string& reason) {
+    const std::vector<int> fds = _clients.fds();
+    for (std::size_t i = 0; i < fds.size(); ++i) {
+        sendRaw(fds[i], Replies::error(reason));
+        requestClose(fds[i], reason);
+    }
+    logMetrics();
+}
+
 void IrcApplication::logMetrics() const {
     const Server::Metrics& serverMetrics = _server.metrics();
     logEvent("server_metrics", std::vector<std::pair<std::string, std::string> >{
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index fe9a84d..af3e7f0 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -39,6 +39,7 @@ public:
     void onLine(Connection& connection, const std::string& line);
     void onDisconnect(Connection& connection, const std::string& reason);
     void onTick();
+    void shutdown(const std::string& reason);
     void logMetrics() const;
 
 private:
diff --git a/src/main.cpp b/src/main.cpp
index 856978f..1bd9001 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -14,13 +14,9 @@
 
 namespace {
     volatile std::sig_atomic_t gRunning = 1;
-    Server* gServer = NULL;
 
     void handleSignal(int) {
         gRunning = 0;
-        if (gServer != NULL) {
-            gServer->stop();
-        }
     }
 }
 
@@ -42,8 +38,6 @@ int main(int argc, char** argv) {
         std::unique_ptr<IrcApplication> app;
         Server server(config);
 
-        gServer = &server;
-
         app.reset(new IrcApplication(server, argv[2], runtime));
         server.setConnectHandler([&app](Connection& connection) {
             app->onConnect(connection);
@@ -69,13 +63,15 @@ int main(int argc, char** argv) {
             server.pollOnce();
             app->onTick();
         }
-        app->logMetrics();
+        app->shutdown("Server shutting down");
+        for (int i = 0; i < 8 && server.connectionCount() > 0; ++i) {
+            server.pollOnce(50);
+        }
         server.setConnectHandler(Server::ConnectHandler());
         server.setLineHandler(Server::LineHandler());
         server.setDisconnectHandler(Server::DisconnectHandler());
         server.setErrorHandler(Server::ErrorHandler());
         server.stop();
-        gServer = NULL;
     } catch (const std::exception& error) {
         std::cerr << "irc-relay-server: " << error.what();
         if (errno != 0) {


