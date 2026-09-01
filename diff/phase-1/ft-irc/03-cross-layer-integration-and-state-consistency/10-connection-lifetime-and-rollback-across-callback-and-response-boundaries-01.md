# 콜백·응답 경계의 연결 수명과 롤백

## `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장`

diff --git a/include/Server.hpp b/include/Server.hpp
index e01128f..a155a35 100644
--- a/include/Server.hpp
+++ b/include/Server.hpp
@@ -38,6 +38,7 @@ public:
     using ErrorHandler = std::function<void(const std::string&)>;
 
     explicit Server(Config config);
+    Server(Config config, std::unique_ptr<EventManager> eventManager);
     ~Server();
 
     Server(const Server&) = delete;
@@ -85,10 +86,10 @@ private:
     void acceptReadyClients();
     void rejectReadyClient();
     void handleClientEvent(const Event& event);
-    void refreshInterest(Connection& connection);
+    bool refreshInterest(int fd);
     void closeListenSocket() noexcept;
     void closeAllConnections();
-    void reportError(const std::string& message) const;
+    void reportError(const std::string& message) const noexcept;
 };
 
 } // namespace irc
diff --git a/src/Server.cpp b/src/Server.cpp
index 2baed67..4e9a936 100644
--- a/src/Server.cpp
+++ b/src/Server.cpp
@@ -89,6 +89,18 @@ Server::Server(Config config)
 {
 }
 
+Server::Server(Config config, std::unique_ptr<EventManager> eventManager)
+    : config_(std::move(config))
+    , listenFd_(-1)
+    , eventManager_(std::move(eventManager))
+    , running_(false)
+    , stopRequested_(false)
+{
+    if (!eventManager_) {
+        throw std::invalid_argument("event manager must not be null");
+    }
+}
+
 Server::~Server()
 {
     stop();
@@ -102,9 +114,17 @@ void Server::start()
         return;
     }
 
-    eventManager_ = EventManager::createDefault();
+    if (!eventManager_) {
+        eventManager_ = EventManager::createDefault();
+    }
     createListenSocket();
-    eventManager_->addFd(listenFd_, EventInterest::Read);
+    try {
+        eventManager_->addFd(listenFd_, EventInterest::Read);
+    } catch (...) {
+        closeListenSocket();
+        eventManager_.reset();
+        throw;
+    }
     stopRequested_ = false;
     running_ = true;
 }
@@ -217,8 +237,8 @@ bool Server::sendTo(int fd, const std::string& line)
     if (!queued) {
         ++metrics_.outboundQueueDrops;
     }
-    refreshInterest(*connection);
-    return queued;
+    const bool refreshed = refreshInterest(fd);
+    return queued && refreshed;
 }
 
 bool Server::queueRawTo(int fd, const std::string& bytes)
@@ -231,8 +251,8 @@ bool Server::queueRawTo(int fd, const std::string& bytes)
     if (!queued) {
         ++metrics_.outboundQueueDrops;
     }
-    refreshInterest(*connection);
-    return queued;
+    const bool refreshed = refreshInterest(fd);
+    return queued && refreshed;
 }
 
 void Server::disconnect(int fd, const std::string& reason)
@@ -372,22 +392,31 @@ void Server::acceptReadyClients()
             clientFd = -1;
 
             const int fd = connection->fd();
-            eventManager_->addFd(fd, EventInterest::Read);
-            Connection* connectionPtr = connection.get();
-            connections_[fd] = std::move(connection);
+            const std::pair<std::unordered_map<int, std::unique_ptr<Connection> >::iterator, bool>
+                inserted = connections_.emplace(fd, std::move(connection));
+            if (!inserted.second) {
+                throw std::logic_error("accepted descriptor is already registered");
+            }
+            try {
+                eventManager_->addFd(fd, EventInterest::Read);
+            } catch (...) {
+                connections_.erase(inserted.first);
+                throw;
+            }
             ++metrics_.acceptedConnections;
 
             if (onConnect_) {
                 try {
-                    onConnect_(*connectionPtr);
+                    onConnect_(*inserted.first->second);
                 } catch (const std::exception& exception) {
                     reportError(exception.what());
-                    connectionPtr->requestClose("connect handler error");
+                    Connection* current = findConnection(fd);
+                    if (current != NULL) {
+                        current->requestClose("connect handler error");
+                    }
                 }
             }
-            if (connections_.find(fd) != connections_.end()) {
-                refreshInterest(*connectionPtr);
-            }
+            refreshInterest(fd);
         } catch (const std::exception& exception) {
             if (clientFd != -1) {
                 ::close(clientFd);
@@ -410,8 +439,8 @@ void Server::handleClientEvent(const Event& event)
         return;
     }
 
+    const int fd = found->second->fd();
     Connection* connection = found->second.get();
-    const int fd = connection->fd();
 
     if (event.error) {
         disconnect(fd, eventErrorMessage(event));
@@ -434,10 +463,14 @@ void Server::handleClientEvent(const Event& event)
                     onLine_(*connection, *line);
                 } catch (const std::exception& exception) {
                     reportError(exception.what());
-                    connection->requestClose("line handler error");
+                    Connection* current = findConnection(fd);
+                    if (current != NULL) {
+                        current->requestClose("line handler error");
+                    }
                 }
             }
-            if (connections_.find(fd) == connections_.end()) {
+            connection = findConnection(fd);
+            if (connection == NULL) {
                 return;
             }
             if (connection->closeRequested()) {
@@ -450,7 +483,8 @@ void Server::handleClientEvent(const Event& event)
         }
     }
 
-    if (connections_.find(fd) == connections_.end()) {
+    connection = findConnection(fd);
+    if (connection == NULL) {
         return;
     }
 
@@ -460,6 +494,10 @@ void Server::handleClientEvent(const Event& event)
             disconnect(fd, writeResult.error);
             return;
         }
+        connection = findConnection(fd);
+        if (connection == NULL) {
+            return;
+        }
     }
 
     if (event.hangup && !connection->wantsWrite()) {
@@ -467,23 +505,34 @@ void Server::handleClientEvent(const Event& event)
         return;
     }
 
-    refreshInterest(*connection);
+    refreshInterest(fd);
 }
 
-void Server::refreshInterest(Connection& connection)
+bool Server::refreshInterest(int fd)
 {
-    const int fd = connection.fd();
+    Connection* connection = findConnection(fd);
+    if (connection == NULL) {
+        return false;
+    }
 
-    if (connection.closeRequested() && !connection.wantsWrite()) {
-        disconnect(fd, connection.closeReason());
-        return;
+    if (connection->closeRequested() && !connection->wantsWrite()) {
+        disconnect(fd, connection->closeReason());
+        return false;
     }
 
-    EventInterest interests = connection.closeRequested() ? EventInterest::Write : EventInterest::Read;
-    if (connection.wantsWrite()) {
+    EventInterest interests =
+        connection->closeRequested() ? EventInterest::Write : EventInterest::Read;
+    if (connection->wantsWrite()) {
         interests |= EventInterest::Write;
     }
-    eventManager_->updateFd(fd, interests);
+    try {
+        eventManager_->updateFd(fd, interests);
+    } catch (const std::exception& exception) {
+        reportError(exception.what());
+        disconnect(fd, "event interest update failed");
+        return false;
+    }
+    return true;
 }
 
 void Server::closeListenSocket() noexcept
@@ -516,10 +565,13 @@ void Server::closeAllConnections()
     }
 }
 
-void Server::reportError(const std::string& message) const
+void Server::reportError(const std::string& message) const noexcept
 {
     if (onError_) {
-        onError_(message);
+        try {
+            onError_(message);
+        } catch (...) {
+        }
     }
 }
 


## `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`

diff --git a/.gitignore b/.gitignore
index 2b464f6..4e22b05 100644
--- a/.gitignore
+++ b/.gitignore
@@ -4,3 +4,4 @@ ircserv
 *.d
 *.log
 tests/connection_test
+tests/server_lifetime_test
diff --git a/Makefile b/Makefile
index 7b27359..731b739 100644
--- a/Makefile
+++ b/Makefile
@@ -1,5 +1,6 @@
 NAME := irc-relay-server
 CONNECTION_TEST := tests/connection_test
+SERVER_LIFETIME_TEST := tests/server_lifetime_test
 
 CXX ?= c++
 CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
@@ -24,7 +25,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all clean connection-test fclean re test smoke
+.PHONY: all clean connection-test fclean re test unit smoke
 
 all: $(NAME)
 
@@ -40,14 +41,20 @@ $(CONNECTION_TEST): tests/connection_test.cpp src/Connection.cpp include/Connect
 connection-test: $(CONNECTION_TEST)
 	./$(CONNECTION_TEST)
 
-test: all connection-test
+$(SERVER_LIFETIME_TEST): tests/server_lifetime_test.cpp src/Connection.cpp src/Server.cpp $(EVENT_SRC) include/Server.hpp include/Connection.hpp include/EventManager.hpp src/ConnectionLimits.hpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) tests/server_lifetime_test.cpp src/Connection.cpp src/Server.cpp $(EVENT_SRC) -o $@
+
+unit: $(SERVER_LIFETIME_TEST)
+	./$(SERVER_LIFETIME_TEST)
+
+test: all connection-test unit
 	bash tests/irc_smoke.sh
 
 smoke: test
 
 clean:
-	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST)
-	rm -rf $(CONNECTION_TEST).dSYM
+	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST)
+	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM
 
 fclean: clean
 	rm -f $(NAME)
diff --git a/tests/server_lifetime_test.cpp b/tests/server_lifetime_test.cpp
new file mode 100644
index 0000000..40b09d9
--- /dev/null
+++ b/tests/server_lifetime_test.cpp
@@ -0,0 +1,319 @@
+#include "EventManager.hpp"
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
+#include <stdexcept>
+#include <string>
+#include <unordered_map>
+#include <utility>
+#include <vector>
+
+namespace {
+
+class FakeEventManager : public irc::EventManager {
+public:
+    void addFd(int fd, irc::EventInterest interests) override
+    {
+        ++addCalls_;
+        if (failNextAdd_) {
+            failNextAdd_ = false;
+            throw std::runtime_error("injected add failure");
+        }
+        interests_[fd] = interests;
+    }
+
+    void updateFd(int fd, irc::EventInterest interests) override
+    {
+        if (failNextUpdate_) {
+            failNextUpdate_ = false;
+            throw std::runtime_error("injected update failure");
+        }
+        if (interests_.find(fd) == interests_.end()) {
+            throw std::runtime_error("updated an unregistered descriptor");
+        }
+        interests_[fd] = interests;
+    }
+
+    void removeFd(int fd) override
+    {
+        interests_.erase(fd);
+    }
+
+    std::vector<irc::Event> wait(int) override
+    {
+        std::vector<irc::Event> result;
+        result.swap(events_);
+        return result;
+    }
+
+    void failNextAdd()
+    {
+        failNextAdd_ = true;
+    }
+
+    void failNextUpdate()
+    {
+        failNextUpdate_ = true;
+    }
+
+    std::size_t addCallCount() const
+    {
+        return addCalls_;
+    }
+
+    void queueReadable(int fd)
+    {
+        irc::Event event;
+        event.fd = fd;
+        event.interests = irc::EventInterest::Read;
+        events_.push_back(event);
+    }
+
+    bool contains(int fd) const
+    {
+        return interests_.find(fd) != interests_.end();
+    }
+
+    int clientFd(int listenFd) const
+    {
+        for (std::unordered_map<int, irc::EventInterest>::const_iterator it = interests_.begin();
+             it != interests_.end();
+             ++it) {
+            if (it->first != listenFd) {
+                return it->first;
+            }
+        }
+        return -1;
+    }
+
+private:
+    std::unordered_map<int, irc::EventInterest> interests_;
+    std::vector<irc::Event> events_;
+    std::size_t addCalls_ = 0;
+    bool failNextAdd_ = false;
+    bool failNextUpdate_ = false;
+};
+
+class ClientSocket {
+public:
+    explicit ClientSocket(std::uint16_t port)
+        : fd_(::socket(AF_INET, SOCK_STREAM, 0))
+    {
+        if (fd_ == -1) {
+            throw std::runtime_error(std::string("socket: ") + std::strerror(errno));
+        }
+
+        sockaddr_in address;
+        std::memset(&address, 0, sizeof(address));
+        address.sin_family = AF_INET;
+        address.sin_port = htons(port);
+        address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+        if (::connect(fd_, reinterpret_cast<sockaddr*>(&address), sizeof(address)) == -1) {
+            const std::string message = std::string("connect: ") + std::strerror(errno);
+            ::close(fd_);
+            fd_ = -1;
+            throw std::runtime_error(message);
+        }
+    }
+
+    ~ClientSocket()
+    {
+        if (fd_ != -1) {
+            ::close(fd_);
+        }
+    }
+
+    void sendLine(const std::string& line)
+    {
+        const std::string bytes = line + "\r\n";
+        std::size_t offset = 0;
+        while (offset < bytes.size()) {
+            const ssize_t count = ::send(fd_, bytes.data() + offset, bytes.size() - offset, 0);
+            if (count > 0) {
+                offset += static_cast<std::size_t>(count);
+                continue;
+            }
+            if (count == -1 && errno == EINTR) {
+                continue;
+            }
+            throw std::runtime_error(std::string("send: ") + std::strerror(errno));
+        }
+    }
+
+private:
+    int fd_;
+};
+
+void require(bool condition, const std::string& message)
+{
+    if (!condition) {
+        throw std::runtime_error(message);
+    }
+}
+
+irc::Server::Config loopbackConfig()
+{
+    irc::Server::Config config;
+    config.bindAddress = "127.0.0.1";
+    config.port = 0;
+    config.eventTimeoutMs = 0;
+    return config;
+}
+
+void acceptOne(irc::Server& server, FakeEventManager& events)
+{
+    const std::size_t initialAddCalls = events.addCallCount();
+    for (int attempt = 0; attempt < 100; ++attempt) {
+        events.queueReadable(server.listenFd());
+        server.pollOnce(0);
+        if (events.addCallCount() > initialAddCalls) {
+            return;
+        }
+        ::usleep(1000);
+    }
+    throw std::runtime_error("client connection was not accepted");
+}
+
+void waitReadable(int fd)
+{
+    pollfd descriptor;
+    descriptor.fd = fd;
+    descriptor.events = POLLIN;
+    descriptor.revents = 0;
+    while (true) {
+        const int count = ::poll(&descriptor, 1, 1000);
+        if (count > 0 && (descriptor.revents & POLLIN) != 0) {
+            return;
+        }
+        if (count == -1 && errno == EINTR) {
+            continue;
+        }
+        throw std::runtime_error("accepted socket did not become readable");
+    }
+}
+
+void registrationRollbackTest()
+{
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server server(loopbackConfig(), std::move(ownedEvents));
+    server.start();
+
+    events->failNextAdd();
+    ClientSocket client(server.port());
+    acceptOne(server, *events);
+
+    require(server.connectionCount() == 0, "failed registration left a connection behind");
+    require(events->clientFd(server.listenFd()) == -1,
+            "failed registration left an event descriptor behind");
+}
+
+void connectCallbackLifetimeTest()
+{
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server server(loopbackConfig(), std::move(ownedEvents));
+    server.setErrorHandler([](const std::string&) { throw std::runtime_error("error handler"); });
+    server.setConnectHandler([&server](irc::Connection& connection) {
+        server.disconnect(connection.fd(), "callback disconnect");
+        throw std::runtime_error("connect callback");
+    });
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptOne(server, *events);
+
+    require(server.connectionCount() == 0,
+            "connect callback accessed a connection after removing it");
+}
+
+void lineCallbackLifetimeTest()
+{
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server server(loopbackConfig(), std::move(ownedEvents));
+    server.setLineHandler([&server](irc::Connection& connection, const std::string&) {
+        server.disconnect(connection.fd(), "callback disconnect");
+        throw std::runtime_error("line callback");
+    });
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptOne(server, *events);
+    const int clientFd = events->clientFd(server.listenFd());
+    require(clientFd != -1, "accepted connection was not registered");
+
+    client.sendLine("PING :lifetime");
+    waitReadable(clientFd);
+    events->queueReadable(clientFd);
+    server.pollOnce(0);
+
+    require(server.connectionCount() == 0,
+            "line callback accessed a connection after removing it");
+}
+
+void interestUpdateRollbackTest()
+{
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server server(loopbackConfig(), std::move(ownedEvents));
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptOne(server, *events);
+    const int clientFd = events->clientFd(server.listenFd());
+    require(clientFd != -1, "accepted connection was not registered");
+
+    events->failNextUpdate();
+    require(!server.sendTo(clientFd, "NOTICE * :queued"),
+            "send succeeded after its write interest update failed");
+    require(server.connectionCount() == 0, "update failure left a connection behind");
+    require(!events->contains(clientFd), "update failure left an event descriptor behind");
+}
+
+void queueLimitCloseTest()
+{
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server::Config config = loopbackConfig();
+    config.maxPendingBytes = 4;
+    irc::Server server(config, std::move(ownedEvents));
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptOne(server, *events);
+    const int clientFd = events->clientFd(server.listenFd());
+    require(clientFd != -1, "accepted connection was not registered");
+
+    require(!server.sendTo(clientFd, "payload"), "oversized output was accepted");
+    require(server.connectionCount() == 0,
+            "queue rejection did not finish the requested connection close");
+    require(!events->contains(clientFd), "queue rejection left an event descriptor behind");
+}
+
+} // namespace
+
+int main()
+{
+    try {
+        registrationRollbackTest();
+        connectCallbackLifetimeTest();
+        lineCallbackLifetimeTest();
+        interestUpdateRollbackTest();
+        queueLimitCloseTest();
+    } catch (const std::exception& exception) {
+        std::cerr << "server lifetime test failed: " << exception.what() << '\n';
+        return 1;
+    }
+
+    std::cout << "server lifetime test passed\n";
+    return 0;
+}


