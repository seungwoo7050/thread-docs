# 채널 상태·구성원 수명 주기·정리

## `feat(channel): 채널 상태 계약 정의`

diff --git a/include/Channel.hpp b/include/Channel.hpp
new file mode 100644
index 0000000..2259993
--- /dev/null
+++ b/include/Channel.hpp
@@ -0,0 +1,55 @@
+#ifndef CHANNEL_HPP
+#define CHANNEL_HPP
+
+#include <set>
+#include <string>
+#include <vector>
+
+class Channel {
+public:
+    Channel();
+    explicit Channel(const std::string& name);
+
+    const std::string& name() const;
+    bool empty() const;
+
+    bool hasMember(int clientId) const;
+    void addMember(int clientId, bool asOperator);
+    void removeMember(int clientId);
+    std::vector<int> members() const;
+
+    bool isOperator(int clientId) const;
+    void setOperator(int clientId, bool enabled);
+
+    bool isInviteOnly() const;
+    void setInviteOnly(bool enabled);
+
+    bool isTopicProtected() const;
+    void setTopicProtected(bool enabled);
+
+    bool hasTopic() const;
+    const std::string& topic() const;
+    void setTopic(const std::string& topic);
+    void clearTopic();
+
+    void invite(const std::string& nickname);
+    bool isInvited(const std::string& nickname) const;
+    void clearInvite(const std::string& nickname);
+
+    std::string modeString() const;
+
+    static bool isValidName(const std::string& name);
+    static std::string canonicalNick(const std::string& nickname);
+
+private:
+    std::string _name;
+    std::set<int> _members;
+    std::set<int> _operators;
+    std::set<std::string> _invited;
+    bool _inviteOnly;
+    bool _topicProtected;
+    bool _hasTopic;
+    std::string _topic;
+};
+
+#endif


## `feat(channel): 구성원과 운영자 상태 관리`

diff --git a/src/Channel.cpp b/src/Channel.cpp
new file mode 100644
index 0000000..0347bf7
--- /dev/null
+++ b/src/Channel.cpp
@@ -0,0 +1,57 @@
+#include "Channel.hpp"
+
+Channel::Channel()
+    : _inviteOnly(false),
+      _topicProtected(true),
+      _hasTopic(false) {
+}
+
+Channel::Channel(const std::string& channelName)
+    : _name(channelName),
+      _inviteOnly(false),
+      _topicProtected(true),
+      _hasTopic(false) {
+}
+
+const std::string& Channel::name() const {
+    return _name;
+}
+
+bool Channel::empty() const {
+    return _members.empty();
+}
+
+bool Channel::hasMember(int clientId) const {
+    return _members.find(clientId) != _members.end();
+}
+
+void Channel::addMember(int clientId, bool asOperator) {
+    _members.insert(clientId);
+    if (asOperator) {
+        _operators.insert(clientId);
+    }
+}
+
+void Channel::removeMember(int clientId) {
+    _members.erase(clientId);
+    _operators.erase(clientId);
+}
+
+std::vector<int> Channel::members() const {
+    return std::vector<int>(_members.begin(), _members.end());
+}
+
+bool Channel::isOperator(int clientId) const {
+    return _operators.find(clientId) != _operators.end();
+}
+
+void Channel::setOperator(int clientId, bool enabled) {
+    if (!_members.count(clientId)) {
+        return;
+    }
+    if (enabled) {
+        _operators.insert(clientId);
+    } else {
+        _operators.erase(clientId);
+    }
+}


## `feat(channel): 주제·초대·모드와 이름 규칙 구현`

diff --git a/src/Channel.cpp b/src/Channel.cpp
index 0347bf7..be6cd1c 100644
--- a/src/Channel.cpp
+++ b/src/Channel.cpp
@@ -1,5 +1,8 @@
 #include "Channel.hpp"
 
+#include <algorithm>
+#include <cctype>
+
 Channel::Channel()
     : _inviteOnly(false),
       _topicProtected(true),
@@ -55,3 +58,84 @@ void Channel::setOperator(int clientId, bool enabled) {
         _operators.erase(clientId);
     }
 }
+
+bool Channel::isInviteOnly() const {
+    return _inviteOnly;
+}
+
+void Channel::setInviteOnly(bool enabled) {
+    _inviteOnly = enabled;
+}
+
+bool Channel::isTopicProtected() const {
+    return _topicProtected;
+}
+
+void Channel::setTopicProtected(bool enabled) {
+    _topicProtected = enabled;
+}
+
+bool Channel::hasTopic() const {
+    return _hasTopic;
+}
+
+const std::string& Channel::topic() const {
+    return _topic;
+}
+
+void Channel::setTopic(const std::string& topicValue) {
+    _topic = topicValue;
+    _hasTopic = true;
+}
+
+void Channel::clearTopic() {
+    _topic.clear();
+    _hasTopic = false;
+}
+
+void Channel::invite(const std::string& nickname) {
+    _invited.insert(canonicalNick(nickname));
+}
+
+bool Channel::isInvited(const std::string& nickname) const {
+    return _invited.find(canonicalNick(nickname)) != _invited.end();
+}
+
+void Channel::clearInvite(const std::string& nickname) {
+    _invited.erase(canonicalNick(nickname));
+}
+
+std::string Channel::modeString() const {
+    std::string modes = "+";
+    if (_inviteOnly) {
+        modes += "i";
+    }
+    if (_topicProtected) {
+        modes += "t";
+    }
+    if (modes == "+") {
+        modes += "";
+    }
+    return modes;
+}
+
+bool Channel::isValidName(const std::string& name) {
+    if (name.size() < 2 || (name[0] != '#' && name[0] != '&')) {
+        return false;
+    }
+    for (std::size_t i = 0; i < name.size(); ++i) {
+        const unsigned char ch = static_cast<unsigned char>(name[i]);
+        if (std::isspace(ch) || ch == ',' || ch == 7) {
+            return false;
+        }
+    }
+    return true;
+}
+
+std::string Channel::canonicalNick(const std::string& nickname) {
+    std::string lowered = nickname;
+    std::transform(lowered.begin(), lowered.end(), lowered.begin(), [](unsigned char ch) {
+        return static_cast<char>(std::tolower(ch));
+    });
+    return lowered;
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


## `feat(channel): 구성원 정리와 식별자 변경 방송`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index 95c7ef2..611bfec 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -89,6 +89,39 @@ void IrcApplication::sendNames(int fd, const Channel& channel) {
     sendNumeric(fd, 366, std::vector<std::string>(1, channel.name()), "End of /NAMES list");
 }
 
+void IrcApplication::partAllChannels(int fd, const std::string& reason) {
+    std::vector<std::string> channelNames;
+    for (std::map<std::string, Channel>::const_iterator it = _channels.begin(); it != _channels.end(); ++it) {
+        if (it->second.hasMember(fd)) {
+            channelNames.push_back(it->first);
+        }
+    }
+    for (std::size_t i = 0; i < channelNames.size(); ++i) {
+        partChannel(fd, channelNames[i], reason);
+    }
+}
+
+void IrcApplication::partChannel(int fd, const std::string& channelName, const std::string& reason) {
+    std::map<std::string, Channel>::iterator it = _channels.find(channelName);
+    if (it == _channels.end()) {
+        sendNumeric(fd, 403, std::vector<std::string>(1, channelName), "No such channel");
+        return;
+    }
+    if (!it->second.hasMember(fd)) {
+        sendNumeric(fd, 442, std::vector<std::string>(1, channelName), "You're not on that channel");
+        return;
+    }
+
+    std::vector<std::string> params;
+    params.push_back(channelName);
+    if (!reason.empty()) {
+        params.push_back(reason);
+    }
+    broadcastToChannel(channelName, Replies::formatMessage(prefixFor(fd), "PART", params), -1);
+    it->second.removeMember(fd);
+    eraseChannelIfEmpty(channelName);
+}
+
 void IrcApplication::broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg) {
     std::vector<std::string> params;
     params.push_back(channel.name());
@@ -130,6 +163,13 @@ void IrcApplication::broadcastToCommon(int fd, const std::string& line, bool inc
     }
 }
 
+void IrcApplication::eraseChannelIfEmpty(const std::string& channelName) {
+    std::map<std::string, Channel>::iterator it = _channels.find(channelName);
+    if (it != _channels.end() && it->second.empty()) {
+        _channels.erase(it);
+    }
+}
+
 int IrcApplication::findNick(const std::string& nickname) const {
     return _clients.findFdByNickname(nickname);
 }
@@ -179,6 +219,44 @@ void IrcApplication::requestClose(int fd, const std::string& reason) {
     }
 }
 
-void IrcApplication::removeClientState(int fd, const std::string&, bool) {
+void IrcApplication::removeClientState(int fd, const std::string& reason, bool notifyPeers) {
+    const ClientState* found = _clients.find(fd);
+    if (found == NULL) {
+        return;
+    }
+
+    const ClientState client = *found;
+    if (notifyPeers && client.registered && !client.nick.empty()) {
+        std::set<int> peers;
+        for (std::map<std::string, Channel>::const_iterator channelIt = _channels.begin(); channelIt != _channels.end(); ++channelIt) {
+            if (!channelIt->second.hasMember(fd)) {
+                continue;
+            }
+            const std::vector<int> members = channelIt->second.members();
+            for (std::size_t i = 0; i < members.size(); ++i) {
+                if (members[i] != fd) {
+                    peers.insert(members[i]);
+                }
+            }
+        }
+        const std::string quitLine = Replies::formatMessage(prefixFor(client), "QUIT", std::vector<std::string>(1, reason));
+        for (std::set<int>::const_iterator peer = peers.begin(); peer != peers.end(); ++peer) {
+            sendRaw(*peer, quitLine);
+        }
+    }
+
+    std::vector<std::string> emptyChannels;
+    for (std::map<std::string, Channel>::iterator channelIt = _channels.begin(); channelIt != _channels.end(); ++channelIt) {
+        if (channelIt->second.hasMember(fd)) {
+            channelIt->second.removeMember(fd);
+            if (channelIt->second.empty()) {
+                emptyChannels.push_back(channelIt->first);
+            }
+        }
+    }
+    for (std::size_t i = 0; i < emptyChannels.size(); ++i) {
+        _channels.erase(emptyChannels[i]);
+    }
+
     _clients.erase(fd);
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index edb76f1..4da5b2e 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -46,9 +46,12 @@ private:
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
     void sendTopicReply(int fd, const Channel& channel);
     void sendNames(int fd, const Channel& channel);
+    void partAllChannels(int fd, const std::string& reason);
+    void partChannel(int fd, const std::string& channelName, const std::string& reason);
     void broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg);
     void broadcastToChannel(const std::string& channelName, const std::string& line, int exceptFd);
     void broadcastToCommon(int fd, const std::string& line, bool includeSelf);
+    void eraseChannelIfEmpty(const std::string& channelName);
     int findNick(const std::string& nickname) const;
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;
diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 2fb141d..3123054 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -62,7 +62,16 @@ void IrcApplication::handleNick(int fd, const IrcMessage& message) {
         return;
     }
 
+    ClientState& client = _clients.state(fd);
+    const bool wasRegistered = client.registered;
+    const std::string oldPrefix = prefixFor(client);
+
     _clients.setNickname(fd, nextNick);
+
+    if (wasRegistered) {
+        broadcastToCommon(fd, Replies::formatMessage(oldPrefix, "NICK", std::vector<std::string>(1, nextNick)), true);
+    }
+
     maybeRegister(fd);
 }
 
