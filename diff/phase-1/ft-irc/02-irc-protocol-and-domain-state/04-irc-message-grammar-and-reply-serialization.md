# IRC 메시지 문법과 응답 직렬화

## `feat(parser): IRC 메시지 값과 직렬화 정의`

diff --git a/include/IrcMessage.hpp b/include/IrcMessage.hpp
new file mode 100644
index 0000000..4c1eb93
--- /dev/null
+++ b/include/IrcMessage.hpp
@@ -0,0 +1,29 @@
+#ifndef IRC_MESSAGE_HPP
+#define IRC_MESSAGE_HPP
+
+#include <string>
+#include <vector>
+
+class IrcMessage {
+public:
+    std::string prefix;
+    std::string command;
+    std::vector<std::string> params;
+    std::string raw;
+
+    IrcMessage();
+    IrcMessage(const std::string& prefix,
+               const std::string& command,
+               const std::vector<std::string>& params);
+
+    bool isCommand(const std::string& name) const;
+    std::string param(std::size_t index, const std::string& fallback = "") const;
+    std::string toLine() const;
+
+    static bool parseLine(const std::string& line, IrcMessage& out, std::string* error);
+    static std::vector<IrcMessage> consumeBuffer(std::string& buffer, std::string* error);
+    static std::string upper(const std::string& value);
+    static std::string trimFrame(const std::string& line);
+};
+
+#endif
diff --git a/src/IrcMessage.cpp b/src/IrcMessage.cpp
new file mode 100644
index 0000000..08ad40f
--- /dev/null
+++ b/src/IrcMessage.cpp
@@ -0,0 +1,54 @@
+#include "IrcMessage.hpp"
+
+#include <algorithm>
+#include <cctype>
+#include <sstream>
+
+IrcMessage::IrcMessage() {
+}
+
+IrcMessage::IrcMessage(const std::string& messagePrefix,
+                       const std::string& messageCommand,
+                       const std::vector<std::string>& messageParams)
+    : prefix(messagePrefix),
+      command(upper(messageCommand)),
+      params(messageParams) {
+}
+
+bool IrcMessage::isCommand(const std::string& name) const {
+    return command == upper(name);
+}
+
+std::string IrcMessage::param(std::size_t index, const std::string& fallback) const {
+    if (index >= params.size()) {
+        return fallback;
+    }
+    return params[index];
+}
+
+std::string IrcMessage::toLine() const {
+    std::ostringstream out;
+    if (!prefix.empty()) {
+        out << ':' << prefix << ' ';
+    }
+    out << command;
+    for (std::size_t i = 0; i < params.size(); ++i) {
+        out << ' ';
+        const std::string& value = params[i];
+        if (value.empty() || value.find(' ') != std::string::npos || value[0] == ':') {
+            out << ':' << value;
+        } else {
+            out << value;
+        }
+    }
+    out << "\r\n";
+    return out.str();
+}
+
+std::string IrcMessage::upper(const std::string& value) {
+    std::string copy = value;
+    std::transform(copy.begin(), copy.end(), copy.begin(), [](unsigned char ch) {
+        return static_cast<char>(std::toupper(ch));
+    });
+    return copy;
+}


## `feat(parser): IRC 한 줄 구문 해석 구현`

diff --git a/src/IrcMessage.cpp b/src/IrcMessage.cpp
index 08ad40f..f59b955 100644
--- a/src/IrcMessage.cpp
+++ b/src/IrcMessage.cpp
@@ -45,6 +45,74 @@ std::string IrcMessage::toLine() const {
     return out.str();
 }
 
+bool IrcMessage::parseLine(const std::string& line, IrcMessage& out, std::string* error) {
+    const std::string trimmed = trimFrame(line);
+    out = IrcMessage();
+    out.raw = trimmed;
+
+    if (trimmed.empty()) {
+        if (error) {
+            *error = "empty IRC frame";
+        }
+        return false;
+    }
+    if (trimmed.size() > 510) {
+        if (error) {
+            *error = "IRC frame exceeds 510 octets before CRLF";
+        }
+        return false;
+    }
+
+    std::size_t pos = 0;
+    if (trimmed[pos] == ':') {
+        const std::size_t end = trimmed.find(' ');
+        if (end == std::string::npos || end == 1) {
+            if (error) {
+                *error = "message prefix is missing a command";
+            }
+            return false;
+        }
+        out.prefix = trimmed.substr(1, end - 1);
+        pos = end + 1;
+    }
+
+    while (pos < trimmed.size() && trimmed[pos] == ' ') {
+        ++pos;
+    }
+
+    const std::size_t commandStart = pos;
+    while (pos < trimmed.size() && trimmed[pos] != ' ') {
+        ++pos;
+    }
+    if (commandStart == pos) {
+        if (error) {
+            *error = "IRC command is missing";
+        }
+        return false;
+    }
+    out.command = upper(trimmed.substr(commandStart, pos - commandStart));
+
+    while (pos < trimmed.size()) {
+        while (pos < trimmed.size() && trimmed[pos] == ' ') {
+            ++pos;
+        }
+        if (pos >= trimmed.size()) {
+            break;
+        }
+        if (trimmed[pos] == ':') {
+            out.params.push_back(trimmed.substr(pos + 1));
+            break;
+        }
+        const std::size_t paramStart = pos;
+        while (pos < trimmed.size() && trimmed[pos] != ' ') {
+            ++pos;
+        }
+        out.params.push_back(trimmed.substr(paramStart, pos - paramStart));
+    }
+
+    return true;
+}
+
 std::string IrcMessage::upper(const std::string& value) {
     std::string copy = value;
     std::transform(copy.begin(), copy.end(), copy.begin(), [](unsigned char ch) {


## `feat(parser): 버퍼에서 IRC 프레임 소비`

diff --git a/src/IrcMessage.cpp b/src/IrcMessage.cpp
index f59b955..35a77b4 100644
--- a/src/IrcMessage.cpp
+++ b/src/IrcMessage.cpp
@@ -113,6 +113,34 @@ bool IrcMessage::parseLine(const std::string& line, IrcMessage& out, std::string
     return true;
 }
 
+std::vector<IrcMessage> IrcMessage::consumeBuffer(std::string& buffer, std::string* error) {
+    std::vector<IrcMessage> messages;
+    std::size_t newline = buffer.find('\n');
+    while (newline != std::string::npos) {
+        const std::string frame = buffer.substr(0, newline + 1);
+        buffer.erase(0, newline + 1);
+
+        IrcMessage message;
+        std::string localError;
+        if (parseLine(frame, message, &localError)) {
+            messages.push_back(message);
+        } else if (error) {
+            *error = localError;
+        }
+
+        newline = buffer.find('\n');
+    }
+
+    if (buffer.size() > 4096) {
+        if (error) {
+            *error = "discarded oversized partial IRC frame";
+        }
+        buffer.clear();
+    }
+
+    return messages;
+}
+
 std::string IrcMessage::upper(const std::string& value) {
     std::string copy = value;
     std::transform(copy.begin(), copy.end(), copy.begin(), [](unsigned char ch) {
@@ -120,3 +148,11 @@ std::string IrcMessage::upper(const std::string& value) {
     });
     return copy;
 }
+
+std::string IrcMessage::trimFrame(const std::string& line) {
+    std::string trimmed = line;
+    while (!trimmed.empty() && (trimmed[trimmed.size() - 1] == '\n' || trimmed[trimmed.size() - 1] == '\r')) {
+        trimmed.erase(trimmed.size() - 1);
+    }
+    return trimmed;
+}


## `feat(reply): IRC 서버 응답 생성`

diff --git a/include/Replies.hpp b/include/Replies.hpp
new file mode 100644
index 0000000..bd4ad76
--- /dev/null
+++ b/include/Replies.hpp
@@ -0,0 +1,21 @@
+#ifndef REPLIES_HPP
+#define REPLIES_HPP
+
+#include <string>
+#include <vector>
+
+namespace Replies {
+    std::string code(int numeric);
+    std::string formatMessage(const std::string& prefix,
+                              const std::string& command,
+                              const std::vector<std::string>& params);
+    std::string numeric(const std::string& serverName,
+                        const std::string& target,
+                        int numericCode,
+                        const std::vector<std::string>& params,
+                        const std::string& trailing);
+    std::string error(const std::string& message);
+    std::string hostmask(const std::string& nick, const std::string& user, const std::string& host);
+}
+
+#endif
diff --git a/src/Replies.cpp b/src/Replies.cpp
new file mode 100644
index 0000000..f58c6c3
--- /dev/null
+++ b/src/Replies.cpp
@@ -0,0 +1,58 @@
+#include "Replies.hpp"
+
+#include <iomanip>
+#include <sstream>
+
+namespace {
+    bool needsTrailingMarker(const std::string& value) {
+        return value.empty() || value.find(' ') != std::string::npos || value[0] == ':';
+    }
+}
+
+std::string Replies::code(int numeric) {
+    std::ostringstream out;
+    out << std::setfill('0') << std::setw(3) << numeric;
+    return out.str();
+}
+
+std::string Replies::formatMessage(const std::string& prefix,
+                                   const std::string& command,
+                                   const std::vector<std::string>& params) {
+    std::ostringstream out;
+    if (!prefix.empty()) {
+        out << ':' << prefix << ' ';
+    }
+    out << command;
+    for (std::size_t i = 0; i < params.size(); ++i) {
+        out << ' ';
+        if (i + 1 == params.size() && needsTrailingMarker(params[i])) {
+            out << ':' << params[i];
+        } else {
+            out << params[i];
+        }
+    }
+    out << "\r\n";
+    return out.str();
+}
+
+std::string Replies::numeric(const std::string& serverName,
+                             const std::string& target,
+                             int numericCode,
+                             const std::vector<std::string>& params,
+                             const std::string& trailing) {
+    std::vector<std::string> allParams;
+    allParams.push_back(target.empty() ? "*" : target);
+    allParams.insert(allParams.end(), params.begin(), params.end());
+    allParams.push_back(trailing);
+    return formatMessage(serverName, code(numericCode), allParams);
+}
+
+std::string Replies::error(const std::string& message) {
+    return formatMessage("", "ERROR", std::vector<std::string>(1, message));
+}
+
+std::string Replies::hostmask(const std::string& nick, const std::string& user, const std::string& host) {
+    const std::string safeUser = user.empty() ? "unknown" : user;
+    const std::string safeHost = host.empty() ? "localhost" : host;
+    return nick + "!" + safeUser + "@" + safeHost;
+}
