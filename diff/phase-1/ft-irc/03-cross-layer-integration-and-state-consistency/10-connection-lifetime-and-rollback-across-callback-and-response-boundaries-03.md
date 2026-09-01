## `test(app): 작은 송신 한도에서 상태 정리 검증`

diff --git a/.gitignore b/.gitignore
index 4e22b05..63de748 100644
--- a/.gitignore
+++ b/.gitignore
@@ -5,3 +5,4 @@ ircserv
 *.log
 tests/connection_test
 tests/server_lifetime_test
+tests/application_lifetime_test
diff --git a/Makefile b/Makefile
index 731b739..7846948 100644
--- a/Makefile
+++ b/Makefile
@@ -1,6 +1,7 @@
 NAME := irc-relay-server
 CONNECTION_TEST := tests/connection_test
 SERVER_LIFETIME_TEST := tests/server_lifetime_test
+APPLICATION_LIFETIME_TEST := tests/application_lifetime_test
 
 CXX ?= c++
 CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
@@ -25,7 +26,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all clean connection-test fclean re test unit smoke
+.PHONY: all application-test clean connection-test fclean re test unit smoke
 
 all: $(NAME)
 
@@ -47,14 +48,22 @@ $(SERVER_LIFETIME_TEST): tests/server_lifetime_test.cpp src/Connection.cpp src/S
 unit: $(SERVER_LIFETIME_TEST)
 	./$(SERVER_LIFETIME_TEST)
 
-test: all connection-test unit
+APP_TEST_SRCS := $(filter-out src/main.cpp,$(SRCS))
+
+$(APPLICATION_LIFETIME_TEST): tests/application_lifetime_test.cpp $(APP_TEST_SRCS) src/IrcApplication.hpp src/ClientRegistry.hpp
+	$(CXX) $(CPPFLAGS) -Isrc $(CXXFLAGS) tests/application_lifetime_test.cpp $(APP_TEST_SRCS) -o $@
+
+application-test: $(APPLICATION_LIFETIME_TEST)
+	./$(APPLICATION_LIFETIME_TEST)
+
+test: all connection-test unit application-test
 	bash tests/irc_smoke.sh
 
 smoke: test
 
 clean:
-	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST)
-	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM
+	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST) $(APPLICATION_LIFETIME_TEST)
+	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM $(APPLICATION_LIFETIME_TEST).dSYM
 
 fclean: clean
 	rm -f $(NAME)
diff --git a/tests/application_lifetime_test.cpp b/tests/application_lifetime_test.cpp
new file mode 100644
index 0000000..118d15f
--- /dev/null
+++ b/tests/application_lifetime_test.cpp
@@ -0,0 +1,176 @@
+#include "EventManager.hpp"
+#include "IrcApplication.hpp"
+#include "Server.hpp"
+
+#include <arpa/inet.h>
+#include <poll.h>
+#include <sys/socket.h>
+#include <unistd.h>
+
+#include <cerrno>
+#include <cstring>
+#include <iostream>
+#include <memory>
+#include <sstream>
+#include <stdexcept>
+#include <string>
+#include <unordered_map>
+#include <utility>
+#include <vector>
+
+namespace {
+class CapturedStderr {
+public:
+    CapturedStderr() : previous_(std::cerr.rdbuf(stream_.rdbuf())) {}
+    ~CapturedStderr() { std::cerr.rdbuf(previous_); }
+    std::string str() const { return stream_.str(); }
+private:
+    std::ostringstream stream_;
+    std::streambuf* previous_;
+};
+
+class FakeEventManager : public irc::EventManager {
+public:
+    void addFd(int fd, irc::EventInterest interests) override { interests_[fd] = interests; }
+    void updateFd(int fd, irc::EventInterest interests) override {
+        if (interests_.find(fd) == interests_.end()) {
+            throw std::runtime_error("updated an unregistered descriptor");
+        }
+        interests_[fd] = interests;
+    }
+    void removeFd(int fd) override { interests_.erase(fd); }
+    std::vector<irc::Event> wait(int) override {
+        std::vector<irc::Event> ready;
+        ready.swap(events_);
+        return ready;
+    }
+    void queueReadable(int fd) {
+        irc::Event event;
+        event.fd = fd;
+        event.interests = irc::EventInterest::Read;
+        events_.push_back(event);
+    }
+    int clientFd(int listenFd) const {
+        for (std::unordered_map<int, irc::EventInterest>::const_iterator it = interests_.begin();
+             it != interests_.end(); ++it) {
+            if (it->first != listenFd) {
+                return it->first;
+            }
+        }
+        return -1;
+    }
+private:
+    std::unordered_map<int, irc::EventInterest> interests_;
+    std::vector<irc::Event> events_;
+};
+
+class ClientSocket {
+public:
+    explicit ClientSocket(std::uint16_t port) : fd_(::socket(AF_INET, SOCK_STREAM, 0)) {
+        if (fd_ == -1) {
+            throw std::runtime_error(std::string("socket: ") + std::strerror(errno));
+        }
+        sockaddr_in address;
+        std::memset(&address, 0, sizeof(address));
+        address.sin_family = AF_INET;
+        address.sin_port = htons(port);
+        address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+        if (::connect(fd_, reinterpret_cast<sockaddr*>(&address), sizeof(address)) == -1) {
+            throw std::runtime_error(std::string("connect: ") + std::strerror(errno));
+        }
+    }
+    ~ClientSocket() { if (fd_ != -1) { ::close(fd_); } }
+    void sendRegistration() {
+        const std::string bytes = "NICK tiny\r\nUSER tiny 0 * :Tiny User\r\n";
+        std::size_t offset = 0;
+        while (offset < bytes.size()) {
+            const ssize_t count = ::send(fd_, bytes.data() + offset, bytes.size() - offset, 0);
+            if (count > 0) {
+                offset += static_cast<std::size_t>(count);
+            } else if (count == -1 && errno == EINTR) {
+                continue;
+            } else {
+                throw std::runtime_error(std::string("send: ") + std::strerror(errno));
+            }
+        }
+    }
+private:
+    int fd_;
+};
+
+void require(bool condition, const std::string& message) {
+    if (!condition) {
+        throw std::runtime_error(message);
+    }
+}
+
+void waitReadable(int fd) {
+    pollfd descriptor;
+    descriptor.fd = fd;
+    descriptor.events = POLLIN;
+    descriptor.revents = 0;
+    const int result = ::poll(&descriptor, 1, 1000);
+    if (result <= 0 || (descriptor.revents & POLLIN) == 0) {
+        throw std::runtime_error("accepted socket did not become readable");
+    }
+}
+
+void acceptUntil(irc::Server& server, FakeEventManager& events, std::size_t expectedCount) {
+    for (int attempt = 0; attempt < 10; ++attempt) {
+        waitReadable(server.listenFd());
+        events.queueReadable(server.listenFd());
+        server.pollOnce(0);
+        if (server.connectionCount() >= expectedCount) {
+            return;
+        }
+    }
+    throw std::runtime_error("accepted connection was not registered");
+}
+
+void registrationQueueFailureTest() {
+    CapturedStderr captured;
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server::Config config;
+    config.bindAddress = "127.0.0.1";
+    config.port = 0;
+    config.eventTimeoutMs = 0;
+    config.maxPendingBytes = 1;
+    irc::Server server(config, std::move(ownedEvents));
+    RuntimeConfig runtime;
+    IrcApplication app(server, "", runtime);
+    server.setConnectHandler([&app](Connection& connection) { app.onConnect(connection); });
+    server.setLineHandler([&app](Connection& connection, const std::string& line) { app.onLine(connection, line); });
+    server.setDisconnectHandler([&app](Connection& connection, const std::string& reason) { app.onDisconnect(connection, reason); });
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptUntil(server, *events, 1);
+    const int clientFd = events->clientFd(server.listenFd());
+    require(clientFd != -1, "accepted connection was not registered");
+    client.sendRegistration();
+    waitReadable(clientFd);
+    events->queueReadable(clientFd);
+    server.pollOnce(0);
+
+    require(server.connectionCount() == 0, "queue rejection left an application connection behind");
+    require(server.isRunning(), "application queue rejection stopped the server");
+    require(captured.str().find("event=client_registered") == std::string::npos,
+            "registration was recorded after the connection was removed");
+    server.setConnectHandler(Server::ConnectHandler());
+    server.setLineHandler(Server::LineHandler());
+    server.setDisconnectHandler(Server::DisconnectHandler());
+    server.stop();
+}
+} // namespace
+
+int main() {
+    try {
+        registrationQueueFailureTest();
+    } catch (const std::exception& exception) {
+        std::cerr << "application lifetime test failed: " << exception.what() << '\n';
+        return 1;
+    }
+    std::cout << "application lifetime test passed\n";
+    return 0;
+}


## `fix(app): 응답 실패 뒤 명령 처리를 중단`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 8199fe8..251cbe4 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -222,23 +222,31 @@ void IrcApplication::handleList(int fd, const IrcMessage& message) {
         requested.insert(names.begin(), names.end());
     }
 
-    sendNumericRaw(fd, 321, std::vector<std::string>{"Channel", "Users", "Name"});
-    for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
+    if (!sendNumericRaw(fd, 321, std::vector<std::string>{"Channel", "Users", "Name"})) {
+        return;
+    }
+    for (std::map<std::string, Channel>::const_iterator it = _channels.begin();
+         it != _channels.end(); ++it) {
         if (!requested.empty() && requested.find(it->first) == requested.end()) {
             continue;
         }
         std::vector<std::string> params;
         params.push_back(it->first);
         params.push_back(std::to_string(it->second.memberCount()));
-        sendNumeric(fd, 322, params, it->second.hasTopic() ? it->second.topic() : "open room");
+        if (!sendNumeric(fd, 322, params, it->second.hasTopic() ? it->second.topic() : "open room")) {
+            return;
+        }
     }
     sendNumeric(fd, 323, std::vector<std::string>(), "End of /LIST");
 }
 
 void IrcApplication::handleNames(int fd, const IrcMessage& message) {
     if (message.params.empty()) {
-        for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
-            sendNames(fd, it->second);
+        for (std::map<std::string, Channel>::const_iterator it = _channels.begin();
+             it != _channels.end(); ++it) {
+            if (!sendNames(fd, it->second)) {
+                return;
+            }
         }
         return;
     }
@@ -247,9 +255,12 @@ void IrcApplication::handleNames(int fd, const IrcMessage& message) {
     for (std::size_t i = 0; i < names.size(); ++i) {
         std::map<std::string, Channel>::const_iterator found = _channels.find(names[i]);
         if (found != _channels.end()) {
-            sendNames(fd, found->second);
-        } else {
-            sendNumeric(fd, 366, std::vector<std::string>(1, names[i]), "End of /NAMES list");
+            if (!sendNames(fd, found->second)) {
+                return;
+            }
+        } else if (!sendNumeric(fd, 366, std::vector<std::string>(1, names[i]),
+                                "End of /NAMES list")) {
+            return;
         }
     }
 }
@@ -260,19 +271,20 @@ void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
         return;
     }
 
+    const std::string channelName = channel->name();
     if (message.params.size() == 1) {
         std::vector<std::string> params;
-        params.push_back(channel->name());
+        params.push_back(channelName);
         params.push_back(channel->modeString());
         sendNumericRaw(fd, 324, params);
         return;
     }
     if (!channel->hasMember(fd)) {
-        sendNumeric(fd, 442, std::vector<std::string>(1, channel->name()), "You're not on that channel");
+        sendNumeric(fd, 442, std::vector<std::string>(1, channelName), "You're not on that channel");
         return;
     }
     if (!channel->isOperator(fd)) {
-        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        sendNumeric(fd, 482, std::vector<std::string>(1, channelName), "You're not channel operator");
         return;
     }
 
@@ -292,28 +304,43 @@ void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
 
         if (mode == 'i') {
             channel->setInviteOnly(adding);
-            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "i", "");
+            if (!broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "i", "")) {
+                return;
+            }
         } else if (mode == 't') {
             channel->setTopicProtected(adding);
-            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "t", "");
+            if (!broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "t", "")) {
+                return;
+            }
         } else if (mode == 'o') {
             if (argIndex >= message.params.size()) {
-                sendNumeric(fd, 461, std::vector<std::string>(1, "MODE"), "Not enough parameters");
+                if (!sendNumeric(fd, 461, std::vector<std::string>(1, "MODE"),
+                                 "Not enough parameters")) {
+                    return;
+                }
                 continue;
             }
             const std::string nick = message.params[argIndex++];
             const int targetFd = findNick(nick);
-            if (targetFd == -1 || !channel->hasMember(targetFd)) {
+            const ClientState* targetClient = _clients.find(targetFd);
+            if (targetClient == NULL || !channel->hasMember(targetFd)) {
                 std::vector<std::string> params;
                 params.push_back(nick);
-                params.push_back(channel->name());
-                sendNumeric(fd, 441, params, "They aren't on that channel");
+                params.push_back(channelName);
+                if (!sendNumeric(fd, 441, params, "They aren't on that channel")) {
+                    return;
+                }
                 continue;
             }
+            const std::string targetNick = targetClient->nick;
             channel->setOperator(targetFd, adding);
-            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "o", _clients.state(targetFd).nick);
-        } else {
-            sendNumeric(fd, 472, std::vector<std::string>(1, std::string(1, mode)), "is unknown mode char to me");
+            if (!broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "o",
+                               targetNick)) {
+                return;
+            }
+        } else if (!sendNumeric(fd, 472, std::vector<std::string>(1, std::string(1, mode)),
+                                "is unknown mode char to me")) {
+            return;
         }
     }
 }


## `test(app): 연결 정리 뒤 모드 변경 중단 검증`

diff --git a/tests/application_lifetime_test.cpp b/tests/application_lifetime_test.cpp
index 118d15f..bd20391 100644
--- a/tests/application_lifetime_test.cpp
+++ b/tests/application_lifetime_test.cpp
@@ -1,8 +1,16 @@
+#include "Channel.hpp"
+#include "ClientRegistry.hpp"
 #include "EventManager.hpp"
-#include "IrcApplication.hpp"
+#include "IrcMessage.hpp"
+#include "RuntimeConfig.hpp"
 #include "Server.hpp"
 
+#define private public
+#include "IrcApplication.hpp"
+#undef private
+
 #include <arpa/inet.h>
+#include <algorithm>
 #include <poll.h>
 #include <sys/socket.h>
 #include <unistd.h>
@@ -33,6 +41,10 @@ class FakeEventManager : public irc::EventManager {
 public:
     void addFd(int fd, irc::EventInterest interests) override { interests_[fd] = interests; }
     void updateFd(int fd, irc::EventInterest interests) override {
+        if (fd == failUpdateFd_) {
+            failUpdateFd_ = -1;
+            throw std::runtime_error("injected update failure");
+        }
         if (interests_.find(fd) == interests_.end()) {
             throw std::runtime_error("updated an unregistered descriptor");
         }
@@ -59,9 +71,32 @@ public:
         }
         return -1;
     }
+    std::vector<int> clientFds(int listenFd) const {
+        std::vector<int> result;
+        for (std::unordered_map<int, irc::EventInterest>::const_iterator it = interests_.begin();
+             it != interests_.end(); ++it) {
+            if (it->first != listenFd) {
+                result.push_back(it->first);
+            }
+        }
+        std::sort(result.begin(), result.end());
+        return result;
+    }
+    void failNextUpdateFor(int fd) { failUpdateFd_ = fd; }
+    std::size_t clientCount(int listenFd) const {
+        std::size_t count = 0;
+        for (std::unordered_map<int, irc::EventInterest>::const_iterator it = interests_.begin();
+             it != interests_.end(); ++it) {
+            if (it->first != listenFd) {
+                ++count;
+            }
+        }
+        return count;
+    }
 private:
     std::unordered_map<int, irc::EventInterest> interests_;
     std::vector<irc::Event> events_;
+    int failUpdateFd_ = -1;
 };
 
 class ClientSocket {
@@ -162,11 +197,80 @@ void registrationQueueFailureTest() {
     server.setDisconnectHandler(Server::DisconnectHandler());
     server.stop();
 }
+
+void modeStopsAfterSenderCleanupTest() {
+    CapturedStderr captured;
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server::Config config;
+    config.bindAddress = "127.0.0.1";
+    config.port = 0;
+    config.eventTimeoutMs = 0;
+    irc::Server server(config, std::move(ownedEvents));
+    RuntimeConfig runtime;
+    IrcApplication app(server, "", runtime);
+    server.setConnectHandler([&app](Connection& connection) { app.onConnect(connection); });
+    server.setDisconnectHandler([&app](Connection& connection, const std::string& reason) {
+        app.onDisconnect(connection, reason);
+    });
+    server.start();
+
+    ClientSocket first(server.port());
+    acceptUntil(server, *events, 1);
+    ClientSocket second(server.port());
+    acceptUntil(server, *events, 2);
+    const std::vector<int> fds = events->clientFds(server.listenFd());
+    require(fds.size() == 2, "two application connections were not registered");
+    const int senderFd = fds[0];
+    const int peerFd = fds[1];
+
+    app._clients.setNickname(senderFd, "operator");
+    ClientState* sender = app._clients.find(senderFd);
+    require(sender != NULL, "sender state was not created");
+    sender->registered = true;
+    sender->user = "operator";
+    app._clients.setNickname(peerFd, "peer");
+    ClientState* peer = app._clients.find(peerFd);
+    require(peer != NULL, "peer state was not created");
+    peer->registered = true;
+    peer->user = "peer";
+
+    Channel room("#room");
+    room.addMember(senderFd, true);
+    room.addMember(peerFd, false);
+    room.setTopicProtected(false);
+    app._channels.insert(std::make_pair(std::string("#room"), room));
+
+    events->failNextUpdateFor(senderFd);
+    IrcMessage mode;
+    mode.command = "MODE";
+    mode.params.push_back("#room");
+    mode.params.push_back("+it");
+    app.handleChannelMode(senderFd, mode);
+
+    std::map<std::string, Channel>::const_iterator current = app._channels.find("#room");
+    const bool channelPresent = current != app._channels.end();
+    const bool inviteOnly = channelPresent && current->second.isInviteOnly();
+    const bool topicProtected = channelPresent && current->second.isTopicProtected();
+    const bool senderRemoved = !app._clients.contains(senderFd);
+    const bool peerPresent = app._clients.contains(peerFd);
+    server.setConnectHandler(Server::ConnectHandler());
+    server.setDisconnectHandler(Server::DisconnectHandler());
+    server.stop();
+
+    require(channelPresent, "peer channel state was removed");
+    require(inviteOnly, "first channel mode was not applied");
+    require(!topicProtected, "later channel mode was applied after the sender was removed");
+    require(senderRemoved, "sender state survived the injected update failure");
+    require(peerPresent, "unrelated peer state was removed");
+}
+
 } // namespace
 
 int main() {
     try {
         registrationQueueFailureTest();
+        modeStopsAfterSenderCleanupTest();
     } catch (const std::exception& exception) {
         std::cerr << "application lifetime test failed: " << exception.what() << '\n';
         return 1;
