# 채널 명령과 권한 규칙

## `feat(channel): 주제와 구성원 응답 지원`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index c20ce24..fda23d6 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -3,9 +3,23 @@
 #include "Connection.hpp"
 #include "Replies.hpp"
 
+#include <sstream>
 #include <utility>
 #include <vector>
 
+namespace {
+    std::string joinWords(const std::vector<std::string>& values, const std::string& separator) {
+        std::ostringstream out;
+        for (std::size_t i = 0; i < values.size(); ++i) {
+            if (i != 0) {
+                out << separator;
+            }
+            out << values[i];
+        }
+        return out.str();
+    }
+}
+
 std::vector<std::string> IrcApplication::splitComma(const std::string& value) {
     std::vector<std::string> parts;
     std::size_t start = 0;
@@ -48,6 +62,32 @@ Channel* IrcApplication::findChannelForCommand(int fd, const std::string& name,
     return &it->second;
 }
 
+void IrcApplication::sendTopicReply(int fd, const Channel& channel) {
+    if (channel.hasTopic()) {
+        sendNumeric(fd, 332, std::vector<std::string>(1, channel.name()), channel.topic());
+    } else {
+        sendNumeric(fd, 331, std::vector<std::string>(1, channel.name()), "No topic is set");
+    }
+}
+
+void IrcApplication::sendNames(int fd, const Channel& channel) {
+    std::vector<std::string> names;
+    const std::vector<int> members = channel.members();
+    for (std::size_t i = 0; i < members.size(); ++i) {
+        const ClientState* client = _clients.find(members[i]);
+        if (client == NULL) {
+            continue;
+        }
+        names.push_back((channel.isOperator(members[i]) ? "@" : "") + client->nick);
+    }
+
+    std::vector<std::string> nameParams;
+    nameParams.push_back("=");
+    nameParams.push_back(channel.name());
+    sendNumeric(fd, 353, nameParams, joinWords(names, " "));
+    sendNumeric(fd, 366, std::vector<std::string>(1, channel.name()), "End of /NAMES list");
+}
+
 int IrcApplication::findNick(const std::string& nickname) const {
     return _clients.findFdByNickname(nickname);
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index aaa44e6..4ccb36f 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -44,6 +44,8 @@ private:
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
+    void sendTopicReply(int fd, const Channel& channel);
+    void sendNames(int fd, const Channel& channel);
     int findNick(const std::string& nickname) const;
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;


## `feat(channel): JOIN 채널 입장 처리`

diff --git a/Makefile b/Makefile
index c858f8b..249a047 100644
--- a/Makefile
+++ b/Makefile
@@ -17,7 +17,7 @@ $(error Unsupported OS for IRC event backend: $(UNAME_S))
 endif
 
 SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/MessagingCommands.cpp \
-	src/ApplicationSupport.cpp src/ClientRegistry.cpp src/RuntimeConfig.cpp \
+	src/ChannelCommands.cpp src/ApplicationSupport.cpp src/ClientRegistry.cpp src/RuntimeConfig.cpp \
 	src/IrcMessage.cpp src/Channel.cpp src/Replies.cpp \
 	src/Connection.cpp src/Server.cpp $(EVENT_SRC)
 OBJS := $(SRCS:.cpp=.o)
diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
new file mode 100644
index 0000000..99298f4
--- /dev/null
+++ b/src/ChannelCommands.cpp
@@ -0,0 +1,46 @@
+#include "IrcApplication.hpp"
+
+#include "IrcMessage.hpp"
+#include "Replies.hpp"
+
+#include <set>
+#include <vector>
+
+void IrcApplication::handleJoin(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "JOIN"), "Not enough parameters");
+        return;
+    }
+
+    if (message.params[0] == "0") {
+        partAllChannels(fd, "Leaving all channels");
+        return;
+    }
+
+    const std::vector<std::string> names = splitComma(message.params[0]);
+    for (std::size_t i = 0; i < names.size(); ++i) {
+        const std::string& name = names[i];
+        if (!Channel::isValidName(name)) {
+            sendNumeric(fd, 403, std::vector<std::string>(1, name), "No such channel");
+            continue;
+        }
+
+        Channel& channel = ensureChannel(name);
+        if (channel.hasMember(fd)) {
+            sendNames(fd, channel);
+            continue;
+        }
+        if (channel.isInviteOnly() && !channel.isInvited(_clients.state(fd).nick)) {
+            sendNumeric(fd, 473, std::vector<std::string>(1, name), "Cannot join channel (+i)");
+            continue;
+        }
+
+        const bool firstMember = channel.empty();
+        channel.addMember(fd, firstMember);
+        channel.clearInvite(_clients.state(fd).nick);
+
+        broadcastToChannel(name, Replies::formatMessage(prefixFor(fd), "JOIN", std::vector<std::string>(1, name)), -1);
+        sendTopicReply(fd, channel);
+        sendNames(fd, channel);
+    }
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 7da3e34..41a30cf 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -52,6 +52,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         sendNumeric(fd, 451, std::vector<std::string>(), "You have not registered");
     } else if (message.command == "PRIVMSG") {
         handlePrivmsg(fd, message);
+    } else if (message.command == "JOIN") {
+        handleJoin(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 4da5b2e..f9bf165 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -42,6 +42,8 @@ private:
 
     void handlePrivmsg(int fd, const IrcMessage& message);
 
+    void handleJoin(int fd, const IrcMessage& message);
+
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
     void sendTopicReply(int fd, const Channel& channel);


## `feat(channel): PART 채널 퇴장 처리`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 99298f4..750afe6 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -44,3 +44,16 @@ void IrcApplication::handleJoin(int fd, const IrcMessage& message) {
         sendNames(fd, channel);
     }
 }
+
+void IrcApplication::handlePart(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "PART"), "Not enough parameters");
+        return;
+    }
+
+    const std::string reason = message.params.size() > 1 ? message.params[1] : "";
+    const std::vector<std::string> names = splitComma(message.params[0]);
+    for (std::size_t i = 0; i < names.size(); ++i) {
+        partChannel(fd, names[i], reason);
+    }
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 41a30cf..cf8de44 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -54,6 +54,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handlePrivmsg(fd, message);
     } else if (message.command == "JOIN") {
         handleJoin(fd, message);
+    } else if (message.command == "PART") {
+        handlePart(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index f9bf165..51fdf62 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -43,6 +43,7 @@ private:
     void handlePrivmsg(int fd, const IrcMessage& message);
 
     void handleJoin(int fd, const IrcMessage& message);
+    void handlePart(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);


## `feat(channel): TOPIC 조회와 변경 처리`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 750afe6..947d3f0 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -57,3 +57,31 @@ void IrcApplication::handlePart(int fd, const IrcMessage& message) {
         partChannel(fd, names[i], reason);
     }
 }
+
+void IrcApplication::handleTopic(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "TOPIC"), "Not enough parameters");
+        return;
+    }
+
+    Channel* channel = findChannelForCommand(fd, message.params[0], true);
+    if (!channel) {
+        return;
+    }
+
+    if (message.params.size() == 1) {
+        sendTopicReply(fd, *channel);
+        return;
+    }
+
+    if (channel->isTopicProtected() && !channel->isOperator(fd)) {
+        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        return;
+    }
+
+    channel->setTopic(message.params[1]);
+    std::vector<std::string> params;
+    params.push_back(channel->name());
+    params.push_back(message.params[1]);
+    broadcastToChannel(channel->name(), Replies::formatMessage(prefixFor(fd), "TOPIC", params), -1);
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index cf8de44..3ac3725 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -56,6 +56,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleJoin(fd, message);
     } else if (message.command == "PART") {
         handlePart(fd, message);
+    } else if (message.command == "TOPIC") {
+        handleTopic(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 51fdf62..df4fe1b 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -44,6 +44,7 @@ private:
 
     void handleJoin(int fd, const IrcMessage& message);
     void handlePart(int fd, const IrcMessage& message);
+    void handleTopic(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);


## `feat(channel): KICK 구성원 제거 처리`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 947d3f0..ca76951 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -85,3 +85,42 @@ void IrcApplication::handleTopic(int fd, const IrcMessage& message) {
     params.push_back(message.params[1]);
     broadcastToChannel(channel->name(), Replies::formatMessage(prefixFor(fd), "TOPIC", params), -1);
 }
+
+void IrcApplication::handleKick(int fd, const IrcMessage& message) {
+    if (message.params.size() < 2) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "KICK"), "Not enough parameters");
+        return;
+    }
+
+    Channel* channel = findChannelForCommand(fd, message.params[0], true);
+    if (!channel) {
+        return;
+    }
+    if (!channel->isOperator(fd)) {
+        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        return;
+    }
+
+    const std::string targetNick = message.params[1];
+    const int targetFd = findNick(targetNick);
+    if (targetFd == -1) {
+        sendNumeric(fd, 401, std::vector<std::string>(1, targetNick), "No such nick/channel");
+        return;
+    }
+    if (!channel->hasMember(targetFd)) {
+        std::vector<std::string> params;
+        params.push_back(targetNick);
+        params.push_back(channel->name());
+        sendNumeric(fd, 441, params, "They aren't on that channel");
+        return;
+    }
+
+    const std::string comment = message.params.size() > 2 ? message.params[2] : targetNick;
+    std::vector<std::string> params;
+    params.push_back(channel->name());
+    params.push_back(targetNick);
+    params.push_back(comment);
+    broadcastToChannel(channel->name(), Replies::formatMessage(prefixFor(fd), "KICK", params), -1);
+    channel->removeMember(targetFd);
+    eraseChannelIfEmpty(channel->name());
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 3ac3725..299614b 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -58,6 +58,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handlePart(fd, message);
     } else if (message.command == "TOPIC") {
         handleTopic(fd, message);
+    } else if (message.command == "KICK") {
+        handleKick(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index df4fe1b..bf8ca5c 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -45,6 +45,7 @@ private:
     void handleJoin(int fd, const IrcMessage& message);
     void handlePart(int fd, const IrcMessage& message);
     void handleTopic(int fd, const IrcMessage& message);
+    void handleKick(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);


## `feat(channel): INVITE 초대 상태 처리`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index ca76951..46a362c 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -124,3 +124,47 @@ void IrcApplication::handleKick(int fd, const IrcMessage& message) {
     channel->removeMember(targetFd);
     eraseChannelIfEmpty(channel->name());
 }
+
+void IrcApplication::handleInvite(int fd, const IrcMessage& message) {
+    if (message.params.size() < 2) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "INVITE"), "Not enough parameters");
+        return;
+    }
+
+    const std::string targetNick = message.params[0];
+    const std::string channelName = message.params[1];
+    const int targetFd = findNick(targetNick);
+    if (targetFd == -1) {
+        sendNumeric(fd, 401, std::vector<std::string>(1, targetNick), "No such nick/channel");
+        return;
+    }
+
+    Channel* channel = findChannelForCommand(fd, channelName, true);
+    if (!channel) {
+        return;
+    }
+    if (channel->isInviteOnly() && !channel->isOperator(fd)) {
+        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        return;
+    }
+    if (channel->hasMember(targetFd)) {
+        std::vector<std::string> params;
+        params.push_back(targetNick);
+        params.push_back(channel->name());
+        sendNumeric(fd, 443, params, "is already on channel");
+        return;
+    }
+
+    channel->invite(targetNick);
+
+    std::vector<std::string> invitingParams;
+    invitingParams.push_back(replyTarget(fd));
+    invitingParams.push_back(targetNick);
+    invitingParams.push_back(channel->name());
+    sendRaw(fd, Replies::formatMessage(_serverName, "341", invitingParams));
+
+    std::vector<std::string> inviteParams;
+    inviteParams.push_back(targetNick);
+    inviteParams.push_back(channel->name());
+    sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "INVITE", inviteParams));
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 299614b..35d1598 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -60,6 +60,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleTopic(fd, message);
     } else if (message.command == "KICK") {
         handleKick(fd, message);
+    } else if (message.command == "INVITE") {
+        handleInvite(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index bf8ca5c..22c1678 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -46,6 +46,7 @@ private:
     void handlePart(int fd, const IrcMessage& message);
     void handleTopic(int fd, const IrcMessage& message);
     void handleKick(int fd, const IrcMessage& message);
+    void handleInvite(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);


## `feat(channel): 채널 모드 조회와 i·t 변경`

diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 46a362c..9e3b76d 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -168,3 +168,78 @@ void IrcApplication::handleInvite(int fd, const IrcMessage& message) {
     inviteParams.push_back(channel->name());
     sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "INVITE", inviteParams));
 }
+
+void IrcApplication::handleMode(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "MODE"), "Not enough parameters");
+        return;
+    }
+
+    const std::string target = message.params[0];
+    if (isChannelTarget(target)) {
+        handleChannelMode(fd, message);
+        return;
+    }
+
+    const int targetFd = findNick(target);
+    if (targetFd == -1) {
+        sendNumeric(fd, 401, std::vector<std::string>(1, target), "No such nick/channel");
+        return;
+    }
+    if (message.params.size() == 1) {
+        sendNumericRaw(fd, 221, std::vector<std::string>(1, "+"));
+        return;
+    }
+    if (targetFd != fd) {
+        sendNumeric(fd, 502, std::vector<std::string>(), "Cannot change mode for other users");
+        return;
+    }
+    sendNumeric(fd, 501, std::vector<std::string>(), "User modes are not implemented");
+}
+
+void IrcApplication::handleChannelMode(int fd, const IrcMessage& message) {
+    Channel* channel = findChannelForCommand(fd, message.params[0], false);
+    if (!channel) {
+        return;
+    }
+
+    if (message.params.size() == 1) {
+        std::vector<std::string> params;
+        params.push_back(channel->name());
+        params.push_back(channel->modeString());
+        sendNumericRaw(fd, 324, params);
+        return;
+    }
+    if (!channel->hasMember(fd)) {
+        sendNumeric(fd, 442, std::vector<std::string>(1, channel->name()), "You're not on that channel");
+        return;
+    }
+    if (!channel->isOperator(fd)) {
+        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        return;
+    }
+
+    bool adding = true;
+    const std::string modes = message.params[1];
+    for (std::size_t i = 0; i < modes.size(); ++i) {
+        const char mode = modes[i];
+        if (mode == '+') {
+            adding = true;
+            continue;
+        }
+        if (mode == '-') {
+            adding = false;
+            continue;
+        }
+
+        if (mode == 'i') {
+            channel->setInviteOnly(adding);
+            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "i", "");
+        } else if (mode == 't') {
+            channel->setTopicProtected(adding);
+            broadcastMode(fd, *channel, std::string(adding ? "+" : "-") + "t", "");
+        } else {
+            sendNumeric(fd, 472, std::vector<std::string>(1, std::string(1, mode)), "is unknown mode char to me");
+        }
+    }
+}
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index 35d1598..5b4a35c 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -62,6 +62,8 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
         handleKick(fd, message);
     } else if (message.command == "INVITE") {
         handleInvite(fd, message);
+    } else if (message.command == "MODE") {
+        handleMode(fd, message);
     } else {
         sendNumeric(fd, 421, std::vector<std::string>(1, message.command), "Unknown command");
     }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index 22c1678..ed538ab 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -47,6 +47,8 @@ private:
     void handleTopic(int fd, const IrcMessage& message);
     void handleKick(int fd, const IrcMessage& message);
     void handleInvite(int fd, const IrcMessage& message);
+    void handleMode(int fd, const IrcMessage& message);
+    void handleChannelMode(int fd, const IrcMessage& message);
 
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);


