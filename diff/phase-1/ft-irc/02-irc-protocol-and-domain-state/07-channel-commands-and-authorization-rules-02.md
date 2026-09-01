## `feat(channel): 채널 운영자 모드 변경`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 9e3b76d..9750654 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -220,6 +220,7 @@ void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
     }
 
     bool adding = true;
+    std::size_t argIndex = 2;
     const std::string modes = message.params[1];
     for (std::size_t i = 0; i < modes.size(); ++i) {
         const char mode = modes[i];
@@ -238,6 +239,22 @@ void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
         } else if (mode == 't') {
             channel->setTopicProtected(adding);
             broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "t", "");
+        } else if (mode == 'o') {
+            if (argIndex >= message.params.size()) {
+                sendNumeric(fd, 461, std::vector<std::string>(1, "MODE"), "Not enough parameters");
+                continue;
+            }
+            const std::string nick = message.params[argIndex++];
+            const int targetFd = findNick(nick);
+            if (targetFd == -1 || !channel->hasMember(targetFd)) {
+                std::vector<std::string> params;
+                params.push_back(nick);
+                params.push_back(channel->name());
+                sendNumeric(fd, 441, params, "They aren't on that channel");
+                continue;
+            }
+            channel->setOperator(targetFd, adding);
+            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "o", _clients.state(targetFd).nick);
         } else {
             sendNumeric(fd, 472, std::vector<std::string>(1, std::string(1, mode)), "is unknown mode char to me");
         }


## `feat(room): 채널 목록 조회 구현`

diff --git a/include/Channel.hpp b/include/Channel.hpp
index 2259993..624163c 100644
--- a/include/Channel.hpp
+++ b/include/Channel.hpp
@@ -3,6 +3,7 @@
 
 #include <set>
 #include <string>
+#include <ctime>
 #include <vector>
 
 class Channel {
@@ -12,6 +13,8 @@ public:
 
     const std::string& name() const;
     bool empty() const;
+    std::size_t memberCount() const;
+    std::time_t createdAt() const;
 
     bool hasMember(int clientId) const;
     void addMember(int clientId, bool asOperator);
@@ -46,6 +49,7 @@ private:
     std::set<int> _members;
     std::set<int> _operators;
     std::set<std::string> _invited;
+    std::time_t _createdAt;
     bool _inviteOnly;
     bool _topicProtected;
     bool _hasTopic;
diff --git a/src/Channel.cpp b/src/Channel.cpp
index be6cd1c..4c159d5 100644
--- a/src/Channel.cpp
+++ b/src/Channel.cpp
@@ -4,13 +4,15 @@
 #include <cctype>
 
 Channel::Channel()
-    : _inviteOnly(false),
+    : _createdAt(std::time(NULL)),
+      _inviteOnly(false),
       _topicProtected(true),
       _hasTopic(false) {
 }
 
 Channel::Channel(const std::string& channelName)
     : _name(channelName),
+      _createdAt(std::time(NULL)),
       _inviteOnly(false),
       _topicProtected(true),
       _hasTopic(false) {
@@ -24,6 +26,14 @@ bool Channel::empty() const {
     return _members.empty();
 }
 
+std::size_t Channel::memberCount() const {
+    return _members.size();
+}
+
+std::time_t Channel::createdAt() const {
+    return _createdAt;
+}
+
 bool Channel::hasMember(int clientId) const {
     return _members.find(clientId) != _members.end();
 }
diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 9750654..f5c6fdc 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -197,6 +197,26 @@ void IrcApplication::handleMode(int fd, const IrcMessage& message) {
     sendNumeric(fd, 501, std::vector<std::string>(), "User modes are not implemented");
 }
 
+void IrcApplication::handleList(int fd, const IrcMessage& message) {
+    std::set<std::string> requested;
+    if (!message.params.empty()) {
+        const std::vector<std::string> names = splitComma(message.params[0]);
+        requested.insert(names.begin(), names.end());
+    }
+
+    sendNumericRaw(fd, 321, std::vector<std::string>{"Channel", "Users", "Name"});
+    for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
+        if (!requested.empty() && requested.find(it->first) == requested.end()) {
+            continue;
+        }
+        std::vector<std::string> params;
+        params.push_back(it->first);
+        params.push_back(std::to_string(it->second.memberCount()));
+        sendNumeric(fd, 322, params, it->second.hasTopic() ? it->second.topic() : "open room");
+    }
+    sendNumeric(fd, 323, std::vector<std::string>(), "End of /LIST");
+}
+
 void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
     Channel* channel = findChannelForCommand(fd, message.params[0], false);
     if (!channel) {
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 5b4a35c..993e7f3 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -64,6 +64,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleInvite(fd, message);
     } else if (message.command == "MODE") {
         handleMode(fd, message);
+    } else if (message.command == "LIST") {
+        handleList(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index ed538ab..aa77c55 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -48,6 +48,7 @@ private:
     void handleKick(int fd, const IrcMessage& message);
     void handleInvite(int fd, const IrcMessage& message);
     void handleMode(int fd, const IrcMessage& message);
+    void handleList(int fd, const IrcMessage& message);
     void handleChannelMode(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);


## `feat(room): 채널 구성원 조회 구현`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index f5c6fdc..1c390cf 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -217,6 +217,25 @@ void IrcApplication::handleList(int fd, const IrcMessage& message) {
     sendNumeric(fd, 323, std::vector<std::string>(), "End of /LIST");
 }
 
+void IrcApplication::handleNames(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
+            sendNames(fd, it->second);
+        }
+        return;
+    }
+
+    const std::vector<std::string> names = splitComma(message.params[0]);
+    for (std::size_t i = 0; i < names.size(); ++i) {
+        std::map<std::string, Channel>::const_iterator found = _channels.find(names[i]);
+        if (found != _channels.end()) {
+            sendNames(fd, found->second);
+        } else {
+            sendNumeric(fd, 366, std::vector<std::string>(1, names[i]), "End of /NAMES list");
+        }
+    }
+}
+
 void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
     Channel* channel = findChannelForCommand(fd, message.params[0], false);
     if (!channel) {
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 993e7f3..c88acf5 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -66,6 +66,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleMode(fd, message);
     } else if (message.command == "LIST") {
         handleList(fd, message);
+    } else if (message.command == "NAMES") {
+        handleNames(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index aa77c55..baca520 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -49,6 +49,7 @@ private:
     void handleInvite(int fd, const IrcMessage& message);
     void handleMode(int fd, const IrcMessage& message);
     void handleList(int fd, const IrcMessage& message);
+    void handleNames(int fd, const IrcMessage& message);
     void handleChannelMode(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);


## `test(client): 채널 목록 응답 검증`

diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index c51944b..3ce6e29 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -101,6 +101,12 @@ def main() -> int:
         alice.expect(" JOIN #edu")
         alice.expect(" 353 ")
         alice.expect(" 366 ")
+        alice.send_line("LIST #edu")
+        alice.expect(" 322 ")
+        alice.expect(" 323 ")
+        alice.send_line("NAMES #edu")
+        alice.expect(" 353 ")
+        alice.expect(" 366 ")
 
         bob = register(host, port, password, "bob", "Bob Learner")
         carol = register(host, port, password, "carol", "Carol Learner")
