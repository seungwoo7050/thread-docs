# 메시지 대상 해석과 팬아웃

## `feat(message): 등록 사용자의 개인 메시지 전달`

diff --git a/Makefile b/Makefile
index 9263916..c858f8b 100644
--- a/Makefile
+++ b/Makefile
@@ -16,7 +16,7 @@ else
 $(error Unsupported OS for IRC event backend: $(UNAME_S))
 endif
 
-SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp \
+SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/MessagingCommands.cpp \
 	src/ApplicationSupport.cpp src/ClientRegistry.cpp src/RuntimeConfig.cpp \
 	src/IrcMessage.cpp src/Channel.cpp src/Replies.cpp \
 	src/Connection.cpp src/Server.cpp $(EVENT_SRC)
diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index 890a95a..608078a 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -5,6 +5,27 @@
 
 #include <vector>
 
+std::vector<std::string> IrcApplication::splitComma(const std::string& value) {
+    std::vector<std::string> parts;
+    std::size_t start = 0;
+    while (start <= value.size()) {
+        const std::size_t comma = value.find(',', start);
+        const std::size_t end = comma == std::string::npos ? value.size() : comma;
+        if (end > start) {
+            parts.push_back(value.substr(start, end - start));
+        }
+        if (comma == std::string::npos) {
+            break;
+        }
+        start = comma + 1;
+    }
+    return parts;
+}
+
+int IrcApplication::findNick(const std::string& nickname) const {
+    return _clients.findFdByNickname(nickname);
+}
+
 std::string IrcApplication::replyTarget(int fd) const {
     const ClientState* client = _clients.find(fd);
     if (client == NULL || client->nick.empty()) {
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index a61f9bb..7da3e34 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -50,6 +50,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleQuit(fd, message);
     } else if (!_clients.state(fd).registered) {
         sendNumeric(fd, 451, std::vector<std::string>(), "You have not registered");
+    } else if (message.command == "PRIVMSG") {
+        handlePrivmsg(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 989e1f2..e82acbd 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -25,6 +25,8 @@ private:
     std::string _serverName;
     ClientRegistry _clients;
 
+    static std::vector<std::string> splitComma(const std::string& value);
+
     void handleMessage(int fd, const IrcMessage& message);
 
     void handlePass(int fd, const IrcMessage& message);
@@ -34,6 +36,9 @@ private:
     void handleQuit(int fd, const IrcMessage& message);
     void maybeRegister(int fd);
 
+    void handlePrivmsg(int fd, const IrcMessage& message);
+
+    int findNick(const std::string& nickname) const;
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;
     std::string prefixFor(const ClientState& client) const;
diff --git a/src/MessagingCommands.cpp b/src/MessagingCommands.cpp
new file mode 100644
index 0000000..8e3e2fe
--- /dev/null
+++ b/src/MessagingCommands.cpp
@@ -0,0 +1,30 @@
+#include "IrcApplication.hpp"
+
+#include "IrcMessage.hpp"
+#include "Replies.hpp"
+
+#include <vector>
+
+void IrcApplication::handlePrivmsg(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 411, std::vector<std::string>(), "No recipient given (PRIVMSG)");
+        return;
+    }
+    if (message.params.size() < 2 || message.params[1].empty()) {
+        sendNumeric(fd, 412, std::vector<std::string>(), "No text to send");
+        return;
+    }
+
+    const std::vector<std::string> targets = splitComma(message.params[0]);
+    for (std::size_t i = 0; i < targets.size(); ++i) {
+        const int targetFd = findNick(targets[i]);
+        if (targetFd == -1) {
+            sendNumeric(fd, 401, std::vector<std::string>(1, targets[i]), "No such nick/channel");
+            continue;
+        }
+        std::vector<std::string> params;
+        params.push_back(targets[i]);
+        params.push_back(message.params[1]);
+        sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params));
+    }
+}


## `feat(channel): 채널 탐색과 대상 해석 지원`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index 608078a..c20ce24 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -3,6 +3,7 @@
 #include "Connection.hpp"
 #include "Replies.hpp"
 
+#include <utility>
 #include <vector>
 
 std::vector<std::string> IrcApplication::splitComma(const std::string& value) {
@@ -22,6 +23,31 @@ std::vector<std::string> IrcApplication::splitComma(const std::string& value) {
     return parts;
 }
 
+bool IrcApplication::isChannelTarget(const std::string& target) {
+    return !target.empty() && (target[0] == '#' || target[0] == '&');
+}
+
+Channel& IrcApplication::ensureChannel(const std::string& name) {
+    std::map<std::string, Channel>::iterator it = _channels.find(name);
+    if (it == _channels.end()) {
+        it = _channels.insert(std::make_pair(name, Channel(name))).first;
+    }
+    return it->second;
+}
+
+Channel* IrcApplication::findChannelForCommand(int fd, const std::string& name, bool requireMembership) {
+    std::map<std::string, Channel>::iterator it = _channels.find(name);
+    if (it == _channels.end()) {
+        sendNumeric(fd, 403, std::vector<std::string>(1, name), "No such channel");
+        return NULL;
+    }
+    if (requireMembership && !it->second.hasMember(fd)) {
+        sendNumeric(fd, 442, std::vector<std::string>(1, name), "You're not on that channel");
+        return NULL;
+    }
+    return &it->second;
+}
+
 int IrcApplication::findNick(const std::string& nickname) const {
     return _clients.findFdByNickname(nickname);
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index e82acbd..aaa44e6 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -1,10 +1,12 @@
 #ifndef IRC_APPLICATION_HPP
 #define IRC_APPLICATION_HPP
 
+#include "Channel.hpp"
 #include "ClientRegistry.hpp"
 #include "RuntimeConfig.hpp"
 #include "Server.hpp"
 
+#include <map>
 #include <string>
 #include <vector>
 
@@ -24,8 +26,10 @@ private:
     RuntimeConfig _runtime;
     std::string _serverName;
     ClientRegistry _clients;
+    std::map<std::string, Channel> _channels;
 
     static std::vector<std::string> splitComma(const std::string& value);
+    static bool isChannelTarget(const std::string& target);
 
     void handleMessage(int fd, const IrcMessage& message);
 
@@ -38,6 +42,8 @@ private:
 
     void handlePrivmsg(int fd, const IrcMessage& message);
 
+    Channel& ensureChannel(const std::string& name);
+    Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
     int findNick(const std::string& nickname) const;
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;


## `feat(channel): 채널 방송 대상 팬아웃 지원`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index fda23d6..95c7ef2 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -3,6 +3,7 @@
 #include "Connection.hpp"
 #include "Replies.hpp"
 
+#include <set>
 #include <sstream>
 #include <utility>
 #include <vector>
@@ -88,6 +89,47 @@ void IrcApplication::sendNames(int fd, const Channel& channel) {
     sendNumeric(fd, 366, std::vector<std::string>(1, channel.name()), "End of /NAMES list");
 }
 
+void IrcApplication::broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg) {
+    std::vector<std::string> params;
+    params.push_back(channel.name());
+    params.push_back(mode);
+    if (!arg.empty()) {
+        params.push_back(arg);
+    }
+    broadcastToChannel(channel.name(), Replies::formatMessage(prefixFor(fd), "MODE", params), -1);
+}
+
+void IrcApplication::broadcastToChannel(const std::string& channelName, const std::string& line, int exceptFd) {
+    std::map<std::string, Channel>::const_iterator channelIt = _channels.find(channelName);
+    if (channelIt == _channels.end()) {
+        return;
+    }
+
+    const std::vector<int> members = channelIt->second.members();
+    for (std::size_t i = 0; i < members.size(); ++i) {
+        if (members[i] != exceptFd) {
+            sendRaw(members[i], line);
+        }
+    }
+}
+
+void IrcApplication::broadcastToCommon(int fd, const std::string& line, bool includeSelf) {
+    std::set<int> targets;
+    for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
+        if (!it->second.hasMember(fd)) {
+            continue;
+        }
+        const std::vector<int> members = it->second.members();
+        targets.insert(members.begin(), members.end());
+    }
+    if (!includeSelf) {
+        targets.erase(fd);
+    }
+    for (std::set<int>::const_iterator it = targets.begin(); it != targets.end(); ++it) {
+        sendRaw(*it, line);
+    }
+}
+
 int IrcApplication::findNick(const std::string& nickname) const {
     return _clients.findFdByNickname(nickname);
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 4ccb36f..edb76f1 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -46,6 +46,9 @@ private:
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
     void sendTopicReply(int fd, const Channel& channel);
     void sendNames(int fd, const Channel& channel);
+    void broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg);
+    void broadcastToChannel(const std::string& channelName, const std::string& line, int exceptFd);
+    void broadcastToCommon(int fd, const std::string& line, bool includeSelf);
     int findNick(const std::string& nickname) const;
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;


## `feat(message): 채널 대상 메시지 방송`

diff --git a/src/MessagingCommands.cpp b/src/MessagingCommands.cpp
index 8e3e2fe..5564cd4 100644
--- a/src/MessagingCommands.cpp
+++ b/src/MessagingCommands.cpp
@@ -17,14 +17,27 @@ void IrcApplication::handlePrivmsg(int fd, const IrcMessage& message) {
 
     const std::vector<std::string> targets = splitComma(message.params[0]);
     for (std::size_t i = 0; i < targets.size(); ++i) {
-        const int targetFd = findNick(targets[i]);
-        if (targetFd == -1) {
-            sendNumeric(fd, 401, std::vector<std::string>(1, targets[i]), "No such nick/channel");
-            continue;
+        const std::string& target = targets[i];
+        if (isChannelTarget(target)) {
+            std::map<std::string, Channel>::const_iterator channelIt = _channels.find(target);
+            if (channelIt == _channels.end() || !channelIt->second.hasMember(fd)) {
+                sendNumeric(fd, 404, std::vector<std::string>(1, target), "Cannot send to channel");
+                continue;
+            }
+            std::vector<std::string> params;
+            params.push_back(target);
+            params.push_back(message.params[1]);
+            broadcastToChannel(target, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params), fd);
+        } else {
+            const int targetFd = findNick(target);
+            if (targetFd == -1) {
+                sendNumeric(fd, 401, std::vector<std::string>(1, target), "No such nick/channel");
+                continue;
+            }
+            std::vector<std::string> params;
+            params.push_back(target);
+            params.push_back(message.params[1]);
+            sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params));
         }
-        std::vector<std::string> params;
-        params.push_back(targets[i]);
-        params.push_back(message.params[1]);
-        sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "PRIVMSG", params));
     }
 }


## `test(client): 여러 클라이언트의 채널 메시지 전달 검증`

diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index eb0b8bb..d15b34d 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -180,6 +180,16 @@ def main() -> int:
         alice.send_line("METRICS")
         alice.expect(" NOTICE alice :connections=")
 
+        bots: List[IrcPeer] = []
+        for index in range(6):
+            bot = register(host, port, password, f"bot{index}", f"Bot {index}")
+            bot.send_line("JOIN #load")
+            bot.expect(" JOIN #load")
+            bots.append(bot)
+        peers.extend(bots)
+        bots[0].send_line("PRIVMSG #load :load hello")
+        bots[1].expect(" PRIVMSG #load :load hello")
+
         alice.send_line("QUIT :smoke complete")
         time.sleep(0.05)
         return 0
