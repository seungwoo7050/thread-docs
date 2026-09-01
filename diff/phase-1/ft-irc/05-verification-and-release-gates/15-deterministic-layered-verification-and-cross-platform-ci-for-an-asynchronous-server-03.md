## `test(connection): 부분 송신과 대기열 경계 검증`

diff --git a/.gitignore b/.gitignore
index bf48a67..2b464f6 100644
--- a/.gitignore
+++ b/.gitignore
@@ -3,3 +3,4 @@ ircserv
 *.o
 *.d
 *.log
+tests/connection_test
diff --git a/Makefile b/Makefile
index 8733958..7b27359 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,5 @@
 NAME := irc-relay-server
+CONNECTION_TEST := tests/connection_test
 
 CXX ?= c++
 CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
@@ -23,7 +24,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all clean fclean re test smoke
+.PHONY: all clean connection-test fclean re test smoke
 
 all: $(NAME)
 
@@ -33,13 +34,20 @@ $(NAME): $(OBJS)
 %.o: %.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -MP -c $< -o $@
 
-test: all
+$(CONNECTION_TEST): tests/connection_test.cpp src/Connection.cpp include/Connection.hpp src/ConnectionLimits.hpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) tests/connection_test.cpp src/Connection.cpp -o $@
+
+connection-test: $(CONNECTION_TEST)
+	./$(CONNECTION_TEST)
+
+test: all connection-test
 	bash tests/irc_smoke.sh
 
 smoke: test
 
 clean:
-	rm -f $(OBJS) $(DEPS)
+	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST)
+	rm -rf $(CONNECTION_TEST).dSYM
 
 fclean: clean
 	rm -f $(NAME)
diff --git a/tests/connection_test.cpp b/tests/connection_test.cpp
new file mode 100644
index 0000000..88ab4c0
--- /dev/null
+++ b/tests/connection_test.cpp
@@ -0,0 +1,224 @@
+#include "Connection.hpp"
+#include "../src/ConnectionLimits.hpp"
+
+#include <cerrno>
+#include <cstdlib>
+#include <iostream>
+#include <limits>
+#include <string>
+#include <utility>
+#include <vector>
+
+namespace {
+
+struct SendStep {
+    enum Kind {
+        Bytes,
+        Error,
+        Zero
+    };
+
+    Kind kind;
+    std::size_t byteCount;
+    int errorNumber;
+
+    static SendStep bytes(std::size_t byteCount)
+    {
+        return SendStep{Bytes, byteCount, 0};
+    }
+
+    static SendStep error(int errorNumber)
+    {
+        return SendStep{Error, 0, errorNumber};
+    }
+
+    static SendStep zero()
+    {
+        return SendStep{Zero, 0, 0};
+    }
+};
+
+class ScriptedSender {
+public:
+    explicit ScriptedSender(std::vector<SendStep> steps)
+        : steps_(std::move(steps))
+        , index_(0)
+    {
+    }
+
+    ssize_t send(int, const void* data, std::size_t size, int)
+    {
+        if (index_ >= steps_.size()) {
+            std::cerr << "unexpected send call" << std::endl;
+            std::abort();
+        }
+
+        const SendStep step = steps_[index_++];
+        if (step.kind == SendStep::Error) {
+            errno = step.errorNumber;
+            return -1;
+        }
+        if (step.kind == SendStep::Zero) {
+            return 0;
+        }
+        if (step.byteCount > size) {
+            return static_cast<ssize_t>(step.byteCount);
+        }
+
+        bytes_.append(static_cast<const char*>(data), step.byteCount);
+        return static_cast<ssize_t>(step.byteCount);
+    }
+
+    const std::string& bytes() const
+    {
+        return bytes_;
+    }
+
+    std::size_t calls() const
+    {
+        return index_;
+    }
+
+private:
+    std::vector<SendStep> steps_;
+    std::size_t index_;
+    std::string bytes_;
+};
+
+void require(bool condition, const char* message)
+{
+    if (!condition) {
+        std::cerr << "FAIL: " << message << std::endl;
+        std::exit(1);
+    }
+}
+
+Connection::SendOperation operationFor(ScriptedSender& sender)
+{
+    return [&sender](int fd, const void* data, std::size_t size, int flags) {
+        return sender.send(fd, data, size, flags);
+    };
+}
+
+void testLimitArithmetic()
+{
+    const std::size_t maximum = std::numeric_limits<std::size_t>::max();
+
+    require(irc::detail::canAppendPending(0, maximum, maximum),
+            "an empty queue must accept its exact maximum");
+    require(!irc::detail::canAppendPending(1, maximum, maximum),
+            "limit checks must not wrap when the appended size overflows addition");
+    require(!irc::detail::canAppendPending(maximum, 1, maximum),
+            "a full queue must reject one additional byte");
+    require(!irc::detail::canAppendPending(maximum, 0, maximum - 1),
+            "an already-invalid queue size must be rejected");
+}
+
+void testHardLimitAndLineTerminator()
+{
+    ScriptedSender sender({SendStep::bytes(5)});
+    Connection connection(-1, "test", 512, 5, operationFor(sender));
+
+    require(connection.queueLine("abc\n"), "line plus CRLF must fit the exact limit");
+    require(connection.pendingBytes() == 5, "line terminators must count toward the limit");
+    require(!connection.queueRaw("x"), "one byte beyond the hard limit must be rejected");
+    require(connection.closeRequested(), "limit overflow must request connection closure");
+    require(connection.pendingBytes() == 5, "rejected bytes must not mutate the queue");
+
+    const Connection::WriteResult result = connection.flushPending();
+    require(result.finished && !result.hasError, "an exact-limit queue must still flush");
+    require(sender.bytes() == "abc\r\n", "queueLine must normalize its terminator");
+}
+
+void testInterruptedAndPartialSendState()
+{
+    ScriptedSender sender({
+        SendStep::error(EINTR),
+        SendStep::bytes(2),
+        SendStep::error(EAGAIN),
+        SendStep::bytes(3),
+        SendStep::error(EWOULDBLOCK),
+        SendStep::bytes(7),
+    });
+    Connection connection(-1, "test", 512, 10, operationFor(sender));
+
+    require(connection.queueRaw("abcdef"), "initial payload must be queued");
+
+    Connection::WriteResult result = connection.flushPending();
+    require(!result.finished && result.wouldBlock && !result.hasError,
+            "EINTR must retry and EAGAIN must preserve pending output");
+    require(connection.pendingBytes() == 4, "only successfully sent bytes may be consumed");
+    require(sender.bytes() == "ab", "the first partial send must not duplicate data");
+
+    require(connection.queueRaw("ghijkl"),
+            "freed capacity after a partial send must be reusable up to the exact limit");
+    require(connection.pendingBytes() == 10, "pending bytes must exclude the sent prefix");
+    require(!connection.queueRaw("m"), "the refilled hard limit must reject another byte");
+    require(connection.pendingBytes() == 10, "rejection must preserve the refilled queue");
+
+    result = connection.flushPending();
+    require(!result.finished && result.wouldBlock && !result.hasError,
+            "a later partial send must also retain its offset");
+    require(connection.pendingBytes() == 7, "the second partial send must advance once");
+
+    result = connection.flushPending();
+    require(result.finished && !result.wouldBlock && !result.hasError,
+            "the remaining output must finish on the next writable event");
+    require(connection.pendingBytes() == 0 && !connection.wantsWrite(),
+            "a completed queue must reset its storage state");
+    require(sender.bytes() == "abcdefghijkl",
+            "retries must deliver each byte exactly once and in order");
+    require(sender.calls() == 6, "the scripted syscall sequence must be exhausted");
+}
+
+void testZeroAndTerminalErrorPreservePendingBytes()
+{
+    ScriptedSender zeroSender({SendStep::zero(), SendStep::bytes(3)});
+    Connection zeroConnection(-1, "test", 512, 8, operationFor(zeroSender));
+    require(zeroConnection.queueRaw("abc"), "zero-send payload must be queued");
+
+    Connection::WriteResult result = zeroConnection.flushPending();
+    require(!result.finished && result.wouldBlock && !result.hasError,
+            "a zero-byte send must retain the queue for a later writable event");
+    require(zeroConnection.pendingBytes() == 3, "zero-byte send must consume nothing");
+
+    result = zeroConnection.flushPending();
+    require(result.finished && zeroSender.bytes() == "abc",
+            "data retained after a zero-byte send must be delivered later");
+
+    ScriptedSender errorSender({SendStep::error(EPIPE)});
+    Connection errorConnection(-1, "test", 512, 8, operationFor(errorSender));
+    require(errorConnection.queueRaw("xyz"), "terminal-error payload must be queued");
+
+    result = errorConnection.flushPending();
+    require(result.hasError && !result.finished, "a terminal send error must be reported");
+    require(errorConnection.closeRequested(), "a terminal send error must request closure");
+    require(errorConnection.pendingBytes() == 3, "a failed send must not consume queued bytes");
+}
+
+void testInvalidSendCountCannotAdvanceOffset()
+{
+    ScriptedSender sender({SendStep::bytes(4)});
+    Connection connection(-1, "test", 512, 8, operationFor(sender));
+    require(connection.queueRaw("abc"), "invalid-count payload must be queued");
+
+    const Connection::WriteResult result = connection.flushPending();
+    require(result.hasError && !result.finished,
+            "a byte count larger than the request must be rejected");
+    require(connection.closeRequested(), "an invalid byte count must request closure");
+    require(connection.pendingBytes() == 3, "an invalid byte count must not corrupt the offset");
+}
+
+} // namespace
+
+int main()
+{
+    testLimitArithmetic();
+    testHardLimitAndLineTerminator();
+    testInterruptedAndPartialSendState();
+    testZeroAndTerminalErrorPreservePendingBytes();
+    testInvalidSendCountCannotAdvanceOffset();
+
+    std::cout << "Connection tests passed" << std::endl;
+    return 0;
+}


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


