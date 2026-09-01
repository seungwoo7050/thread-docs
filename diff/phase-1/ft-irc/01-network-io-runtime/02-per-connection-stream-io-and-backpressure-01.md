# 연결별 스트림 I/O와 백프레셔

## `feat(connection): 스트림 연결 상태 계약 정의`

diff --git a/include/Connection.hpp b/include/Connection.hpp
new file mode 100644
index 0000000..4f88c4f
--- /dev/null
+++ b/include/Connection.hpp
@@ -0,0 +1,70 @@
+#ifndef IRC_CONNECTION_HPP
+#define IRC_CONNECTION_HPP
+
+#include <cstddef>
+#include <string>
+#include <vector>
+
+namespace irc {
+
+class Connection {
+public:
+    struct ReadResult {
+        std::vector<std::string> lines;
+        bool wouldBlock = false;
+        bool peerClosed = false;
+        bool hasError = false;
+        std::string error;
+    };
+
+    struct WriteResult {
+        bool finished = true;
+        bool wouldBlock = false;
+        bool hasError = false;
+        std::string error;
+    };
+
+    Connection(int fd, std::string peerAddress, std::size_t maxLineLength = 512);
+    ~Connection();
+
+    Connection(const Connection&) = delete;
+    Connection& operator=(const Connection&) = delete;
+    Connection(Connection&& other) noexcept;
+    Connection& operator=(Connection&& other) noexcept;
+
+    int fd() const noexcept;
+    const std::string& peerAddress() const noexcept;
+    bool wantsWrite() const noexcept;
+    std::size_t pendingBytes() const noexcept;
+
+    ReadResult readAvailable();
+    WriteResult flushPending();
+
+    void queueRaw(const std::string& bytes);
+    void queueLine(const std::string& line);
+
+    void requestClose(std::string reason = "connection close requested");
+    bool closeRequested() const noexcept;
+    const std::string& closeReason() const noexcept;
+    bool peerClosed() const noexcept;
+
+private:
+    int fd_;
+    std::string peerAddress_;
+    std::string readBuffer_;
+    std::string writeBuffer_;
+    std::size_t writeOffset_;
+    std::size_t maxLineLength_;
+    bool peerClosed_;
+    bool closeRequested_;
+    std::string closeReason_;
+
+    void closeFd() noexcept;
+    bool extractLines(ReadResult& result);
+};
+
+} // namespace irc
+
+using Connection = irc::Connection;
+
+#endif // IRC_CONNECTION_HPP


## `feat(connection): 소켓 소유권과 이동 수명 구현`

diff --git a/src/Connection.cpp b/src/Connection.cpp
new file mode 100644
index 0000000..42a2fb3
--- /dev/null
+++ b/src/Connection.cpp
@@ -0,0 +1,87 @@
+#include "Connection.hpp"
+
+#include <unistd.h>
+
+#include <utility>
+
+namespace irc {
+
+Connection::Connection(int fd, std::string peerAddress, std::size_t maxLineLength)
+    : fd_(fd)
+    , peerAddress_(std::move(peerAddress))
+    , writeOffset_(0)
+    , maxLineLength_(maxLineLength == 0 ? 512 : maxLineLength)
+    , peerClosed_(false)
+    , closeRequested_(false)
+{
+}
+
+Connection::~Connection()
+{
+    closeFd();
+}
+
+Connection::Connection(Connection&& other) noexcept
+    : fd_(other.fd_)
+    , peerAddress_(std::move(other.peerAddress_))
+    , readBuffer_(std::move(other.readBuffer_))
+    , writeBuffer_(std::move(other.writeBuffer_))
+    , writeOffset_(other.writeOffset_)
+    , maxLineLength_(other.maxLineLength_)
+    , peerClosed_(other.peerClosed_)
+    , closeRequested_(other.closeRequested_)
+    , closeReason_(std::move(other.closeReason_))
+{
+    other.fd_ = -1;
+    other.writeOffset_ = 0;
+}
+
+Connection& Connection::operator=(Connection&& other) noexcept
+{
+    if (this != &other) {
+        closeFd();
+        fd_ = other.fd_;
+        peerAddress_ = std::move(other.peerAddress_);
+        readBuffer_ = std::move(other.readBuffer_);
+        writeBuffer_ = std::move(other.writeBuffer_);
+        writeOffset_ = other.writeOffset_;
+        maxLineLength_ = other.maxLineLength_;
+        peerClosed_ = other.peerClosed_;
+        closeRequested_ = other.closeRequested_;
+        closeReason_ = std::move(other.closeReason_);
+
+        other.fd_ = -1;
+        other.writeOffset_ = 0;
+    }
+    return *this;
+}
+
+int Connection::fd() const noexcept
+{
+    return fd_;
+}
+
+const std::string& Connection::peerAddress() const noexcept
+{
+    return peerAddress_;
+}
+
+bool Connection::wantsWrite() const noexcept
+{
+    return writeOffset_ < writeBuffer_.size();
+}
+
+std::size_t Connection::pendingBytes() const noexcept
+{
+    return writeBuffer_.size() - writeOffset_;
+}
+
+void Connection::closeFd() noexcept
+{
+    if (fd_ != -1) {
+        ::close(fd_);
+        fd_ = -1;
+    }
+}
+
+} // namespace irc


## `feat(connection): IRC 입력 프레임 추출 구현`

diff --git a/src/Connection.cpp b/src/Connection.cpp
index 42a2fb3..a4e5b5c 100644
--- a/src/Connection.cpp
+++ b/src/Connection.cpp
@@ -84,4 +84,36 @@ void Connection::closeFd() noexcept
     }
 }
 
+bool Connection::extractLines(ReadResult& result)
+{
+    while (true) {
+        const std::string::size_type newline = readBuffer_.find('\n');
+        if (newline == std::string::npos) {
+            if (readBuffer_.size() > maxLineLength_) {
+                result.hasError = true;
+                result.error = "incoming line exceeds maximum length";
+                requestClose(result.error);
+                readBuffer_.clear();
+                return false;
+            }
+            return true;
+        }
+
+        if (newline + 1 > maxLineLength_) {
+            result.hasError = true;
+            result.error = "incoming line exceeds maximum length";
+            requestClose(result.error);
+            readBuffer_.clear();
+            return false;
+        }
+
+        std::string line = readBuffer_.substr(0, newline);
+        readBuffer_.erase(0, newline + 1);
+        if (!line.empty() && line[line.size() - 1] == '\r') {
+            line.erase(line.size() - 1);
+        }
+        result.lines.push_back(line);
+    }
+}
+
 } // namespace irc


## `feat(connection): 논블로킹 수신 상태 처리`

diff --git a/src/Connection.cpp b/src/Connection.cpp
index a4e5b5c..a15d43a 100644
--- a/src/Connection.cpp
+++ b/src/Connection.cpp
@@ -1,10 +1,24 @@
 #include "Connection.hpp"
 
+#include <sys/socket.h>
 #include <unistd.h>
 
+#include <cerrno>
+#include <cstring>
 #include <utility>
 
 namespace irc {
+namespace {
+
+std::string errorMessage(const char* operation)
+{
+    std::string message(operation);
+    message += ": ";
+    message += std::strerror(errno);
+    return message;
+}
+
+} // namespace
 
 Connection::Connection(int fd, std::string peerAddress, std::size_t maxLineLength)
     : fd_(fd)
@@ -76,6 +90,44 @@ std::size_t Connection::pendingBytes() const noexcept
     return writeBuffer_.size() - writeOffset_;
 }
 
+Connection::ReadResult Connection::readAvailable()
+{
+    ReadResult result;
+    char buffer[4096];
+
+    while (true) {
+        const ssize_t count = ::recv(fd_, buffer, sizeof(buffer), 0);
+        if (count > 0) {
+            readBuffer_.append(buffer, static_cast<std::size_t>(count));
+            if (!extractLines(result)) {
+                break;
+            }
+            continue;
+        }
+        if (count == 0) {
+            peerClosed_ = true;
+            result.peerClosed = true;
+            requestClose("peer closed connection");
+            break;
+        }
+
+        if (errno == EINTR) {
+            continue;
+        }
+        if (errno == EAGAIN || errno == EWOULDBLOCK) {
+            result.wouldBlock = true;
+            break;
+        }
+
+        result.hasError = true;
+        result.error = errorMessage("recv");
+        requestClose(result.error);
+        break;
+    }
+
+    return result;
+}
+
 void Connection::closeFd() noexcept
 {
     if (fd_ != -1) {


## `feat(connection): 부분 송신 대기열 처리`

diff --git a/src/Connection.cpp b/src/Connection.cpp
index a15d43a..72aee8b 100644
--- a/src/Connection.cpp
+++ b/src/Connection.cpp
@@ -18,6 +18,15 @@ std::string errorMessage(const char* operation)
     return message;
 }
 
+int sendFlags()
+{
+#ifdef MSG_NOSIGNAL
+    return MSG_NOSIGNAL;
+#else
+    return 0;
+#endif
+}
+
 } // namespace
 
 Connection::Connection(int fd, std::string peerAddress, std::size_t maxLineLength)
@@ -128,6 +137,88 @@ Connection::ReadResult Connection::readAvailable()
     return result;
 }
 
+Connection::WriteResult Connection::flushPending()
+{
+    WriteResult result;
+
+    while (wantsWrite()) {
+        const char* data = writeBuffer_.data() + writeOffset_;
+        const std::size_t size = writeBuffer_.size() - writeOffset_;
+        const ssize_t count = ::send(fd_, data, size, sendFlags());
+
+        if (count > 0) {
+            writeOffset_ += static_cast<std::size_t>(count);
+            continue;
+        }
+        if (count == 0) {
+            result.wouldBlock = true;
+            break;
+        }
+        if (errno == EINTR) {
+            continue;
+        }
+        if (errno == EAGAIN || errno == EWOULDBLOCK) {
+            result.wouldBlock = true;
+            break;
+        }
+
+        result.hasError = true;
+        result.error = errorMessage("send");
+        requestClose(result.error);
+        break;
+    }
+
+    if (!wantsWrite()) {
+        writeBuffer_.clear();
+        writeOffset_ = 0;
+        result.finished = true;
+    } else {
+        result.finished = false;
+        if (writeOffset_ > 16384) {
+            writeBuffer_.erase(0, writeOffset_);
+            writeOffset_ = 0;
+        }
+    }
+
+    return result;
+}
+
+void Connection::queueRaw(const std::string& bytes)
+{
+    writeBuffer_.append(bytes);
+}
+
+void Connection::queueLine(const std::string& line)
+{
+    std::size_t end = line.size();
+    while (end > 0 && (line[end - 1] == '\r' || line[end - 1] == '\n')) {
+        --end;
+    }
+    writeBuffer_.append(line, 0, end);
+    writeBuffer_.append("\r\n");
+}
+
+void Connection::requestClose(std::string reason)
+{
+    closeRequested_ = true;
+    closeReason_ = std::move(reason);
+}
+
+bool Connection::closeRequested() const noexcept
+{
+    return closeRequested_;
+}
+
+const std::string& Connection::closeReason() const noexcept
+{
+    return closeReason_;
+}
+
+bool Connection::peerClosed() const noexcept
+{
+    return peerClosed_;
+}
+
 void Connection::closeFd() noexcept
 {
     if (fd_ != -1) {


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


## `fix(connection): 송신 대기열 계산과 재시도 상태 보호`

diff --git a/include/Connection.hpp b/include/Connection.hpp
index 2a68e81..2b5b4e9 100644
--- a/include/Connection.hpp
+++ b/include/Connection.hpp
@@ -1,7 +1,10 @@
 #ifndef IRC_CONNECTION_HPP
 #define IRC_CONNECTION_HPP
 
+#include <sys/types.h>
+
 #include <cstddef>
+#include <functional>
 #include <string>
 #include <vector>
 
@@ -9,6 +12,8 @@ namespace irc {
 
 class Connection {
 public:
+    using SendOperation = std::function<ssize_t(int, const void*, std::size_t, int)>;
+
     struct ReadResult {
         std::vector<std::string> lines;
         bool wouldBlock = false;
@@ -27,7 +32,8 @@ public:
     Connection(int fd,
                std::string peerAddress,
                std::size_t maxLineLength = 512,
-               std::size_t maxPendingBytes = 1048576);
+               std::size_t maxPendingBytes = 1048576,
+               SendOperation sendOperation = SendOperation());
     ~Connection();
 
     Connection(const Connection&) = delete;
@@ -59,6 +65,7 @@ private:
     std::size_t writeOffset_;
     std::size_t maxLineLength_;
     std::size_t maxPendingBytes_;
+    SendOperation sendOperation_;
     bool peerClosed_;
     bool closeRequested_;
     std::string closeReason_;
diff --git a/src/Connection.cpp b/src/Connection.cpp
index b542eeb..6376217 100644
--- a/src/Connection.cpp
+++ b/src/Connection.cpp
@@ -1,4 +1,5 @@
 #include "Connection.hpp"
+#include "ConnectionLimits.hpp"
 
 #include <sys/socket.h>
 #include <unistd.h>
@@ -27,17 +28,24 @@ int sendFlags()
 #endif
 }
 
+ssize_t sendBytes(int fd, const void* data, std::size_t size, int flags)
+{
+    return ::send(fd, data, size, flags);
+}
+
 } // namespace
 
 Connection::Connection(int fd,
                        std::string peerAddress,
                        std::size_t maxLineLength,
-                       std::size_t maxPendingBytes)
+                       std::size_t maxPendingBytes,
+                       SendOperation sendOperation)
     : fd_(fd)
     , peerAddress_(std::move(peerAddress))
     , writeOffset_(0)
     , maxLineLength_(maxLineLength == 0 ? 512 : maxLineLength)
     , maxPendingBytes_(maxPendingBytes == 0 ? 1048576 : maxPendingBytes)
+    , sendOperation_(sendOperation ? std::move(sendOperation) : SendOperation(sendBytes))
     , peerClosed_(false)
     , closeRequested_(false)
 {
@@ -56,6 +64,7 @@ Connection::Connection(Connection&& other) noexcept
     , writeOffset_(other.writeOffset_)
     , maxLineLength_(other.maxLineLength_)
     , maxPendingBytes_(other.maxPendingBytes_)
+    , sendOperation_(std::move(other.sendOperation_))
     , peerClosed_(other.peerClosed_)
     , closeRequested_(other.closeRequested_)
     , closeReason_(std::move(other.closeReason_))
@@ -75,6 +84,7 @@ Connection& Connection::operator=(Connection&& other) noexcept
         writeOffset_ = other.writeOffset_;
         maxLineLength_ = other.maxLineLength_;
         maxPendingBytes_ = other.maxPendingBytes_;
+        sendOperation_ = std::move(other.sendOperation_);
         peerClosed_ = other.peerClosed_;
         closeRequested_ = other.closeRequested_;
         closeReason_ = std::move(other.closeReason_);
@@ -150,10 +160,19 @@ Connection::WriteResult Connection::flushPending()
     while (wantsWrite()) {
         const char* data = writeBuffer_.data() + writeOffset_;
         const std::size_t size = writeBuffer_.size() - writeOffset_;
-        const ssize_t count = ::send(fd_, data, size, sendFlags());
+        const ssize_t count = sendOperation_
+            ? sendOperation_(fd_, data, size, sendFlags())
+            : sendBytes(fd_, data, size, sendFlags());
 
         if (count > 0) {
-            writeOffset_ += static_cast<std::size_t>(count);
+            const std::size_t sent = static_cast<std::size_t>(count);
+            if (sent > size) {
+                result.hasError = true;
+                result.error = "send returned more bytes than requested";
+                requestClose(result.error);
+                break;
+            }
+            writeOffset_ += sent;
             continue;
         }
         if (count == 0) {
@@ -205,7 +224,9 @@ bool Connection::queueLine(const std::string& line)
     while (end > 0 && (line[end - 1] == '\r' || line[end - 1] == '\n')) {
         --end;
     }
-    if (!canAppendPending(end + 2)) {
+    const std::size_t pending = pendingBytes();
+    if (!detail::canAppendPending(pending, end, maxPendingBytes_)
+        || !detail::canAppendPending(pending + end, 2, maxPendingBytes_)) {
         requestClose("outbound queue limit exceeded");
         return false;
     }
@@ -277,7 +298,7 @@ bool Connection::extractLines(ReadResult& result)
 
 bool Connection::canAppendPending(std::size_t byteCount) const noexcept
 {
-    return pendingBytes() + byteCount <= maxPendingBytes_;
+    return detail::canAppendPending(pendingBytes(), byteCount, maxPendingBytes_);
 }
 
 } // namespace irc
diff --git a/src/ConnectionLimits.hpp b/src/ConnectionLimits.hpp
new file mode 100644
index 0000000..8b85afc
--- /dev/null
+++ b/src/ConnectionLimits.hpp
@@ -0,0 +1,19 @@
+#ifndef IRC_CONNECTION_LIMITS_HPP
+#define IRC_CONNECTION_LIMITS_HPP
+
+#include <cstddef>
+
+namespace irc {
+namespace detail {
+
+inline bool canAppendPending(std::size_t pending,
+                             std::size_t byteCount,
+                             std::size_t limit) noexcept
+{
+    return pending <= limit && byteCount <= limit - pending;
+}
+
+} // namespace detail
+} // namespace irc
+
+#endif // IRC_CONNECTION_LIMITS_HPP


