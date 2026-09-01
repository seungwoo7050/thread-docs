# 클라이언트 등록과 닉네임 수명 주기

## `feat(client): 연결별 등록 상태 저장`

diff --git a/src/ClientRegistry.cpp b/src/ClientRegistry.cpp
new file mode 100644
index 0000000..77706d7
--- /dev/null
+++ b/src/ClientRegistry.cpp
@@ -0,0 +1,44 @@
+#include "ClientRegistry.hpp"
+
+ClientState::ClientState()
+    : fd(-1),
+      passOk(false),
+      hasNick(false),
+      hasUser(false),
+      registered(false),
+      host("localhost") {
+}
+
+ClientState& ClientRegistry::state(int fd) {
+    return _states[fd];
+}
+
+ClientState* ClientRegistry::find(int fd) {
+    std::map<int, ClientState>::iterator it = _states.find(fd);
+    return it == _states.end() ? NULL : &it->second;
+}
+
+const ClientState* ClientRegistry::find(int fd) const {
+    std::map<int, ClientState>::const_iterator it = _states.find(fd);
+    return it == _states.end() ? NULL : &it->second;
+}
+
+bool ClientRegistry::contains(int fd) const {
+    return _states.find(fd) != _states.end();
+}
+
+std::vector<int> ClientRegistry::fds() const {
+    std::vector<int> values;
+    for (std::map<int, ClientState>::const_iterator it = _states.begin(); it != _states.end(); ++it) {
+        values.push_back(it->first);
+    }
+    return values;
+}
+
+void ClientRegistry::erase(int fd) {
+    std::map<int, ClientState>::iterator it = _states.find(fd);
+    if (it == _states.end()) {
+        return;
+    }
+    _states.erase(it);
+}
diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
new file mode 100644
index 0000000..b6143ad
--- /dev/null
+++ b/src/ClientRegistry.hpp
@@ -0,0 +1,35 @@
+#ifndef IRC_CLIENT_REGISTRY_HPP
+#define IRC_CLIENT_REGISTRY_HPP
+
+#include <map>
+#include <string>
+#include <vector>
+
+struct ClientState {
+    int fd;
+    bool passOk;
+    bool hasNick;
+    bool hasUser;
+    bool registered;
+    std::string nick;
+    std::string user;
+    std::string realname;
+    std::string host;
+
+    ClientState();
+};
+
+class ClientRegistry {
+public:
+    ClientState& state(int fd);
+    ClientState* find(int fd);
+    const ClientState* find(int fd) const;
+    bool contains(int fd) const;
+    std::vector<int> fds() const;
+    void erase(int fd);
+
+private:
+    std::map<int, ClientState> _states;
+};
+
+#endif // IRC_CLIENT_REGISTRY_HPP


## `feat(client): 닉네임 색인 관리`

diff --git a/src/ClientRegistry.cpp b/src/ClientRegistry.cpp
index 77706d7..1da3184 100644
--- a/src/ClientRegistry.cpp
+++ b/src/ClientRegistry.cpp
@@ -1,5 +1,7 @@
 #include "ClientRegistry.hpp"
 
+#include "Channel.hpp"
+
 ClientState::ClientState()
     : fd(-1),
       passOk(false),
@@ -35,10 +37,30 @@ std::vector<int> ClientRegistry::fds() const {
     return values;
 }
 
+int ClientRegistry::findFdByNickname(const std::string& nickname) const {
+    const std::map<std::string, int>::const_iterator it =
+        _nicknameIndex.find(Channel::canonicalNick(nickname));
+    return it == _nicknameIndex.end() ? -1 : it->second;
+}
+
+void ClientRegistry::setNickname(int fd, const std::string& nickname) {
+    const std::string canonical = Channel::canonicalNick(nickname);
+    ClientState& client = state(fd);
+    if (!client.nick.empty()) {
+        _nicknameIndex.erase(Channel::canonicalNick(client.nick));
+    }
+    client.nick = nickname;
+    client.hasNick = true;
+    _nicknameIndex[canonical] = fd;
+}
+
 void ClientRegistry::erase(int fd) {
     std::map<int, ClientState>::iterator it = _states.find(fd);
     if (it == _states.end()) {
         return;
     }
+    if (!it->second.nick.empty()) {
+        _nicknameIndex.erase(Channel::canonicalNick(it->second.nick));
+    }
     _states.erase(it);
 }
diff --git a/src/ClientRegistry.hpp b/src/ClientRegistry.hpp
index b6143ad..4fbff28 100644
--- a/src/ClientRegistry.hpp
+++ b/src/ClientRegistry.hpp
@@ -26,10 +26,13 @@ public:
     const ClientState* find(int fd) const;
     bool contains(int fd) const;
     std::vector<int> fds() const;
+    int findFdByNickname(const std::string& nickname) const;
+    void setNickname(int fd, const std::string& nickname);
     void erase(int fd);
 
 private:
     std::map<int, ClientState> _states;
+    std::map<std::string, int> _nicknameIndex;
 };
 
 #endif // IRC_CLIENT_REGISTRY_HPP


## `feat(registration): PASS 인증 상태 처리`

diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
new file mode 100644
index 0000000..0676aae
--- /dev/null
+++ b/src/RegistrationCommands.cpp
@@ -0,0 +1,24 @@
+#include "IrcApplication.hpp"
+
+#include "IrcMessage.hpp"
+
+#include <vector>
+
+void IrcApplication::handlePass(int fd, const IrcMessage& message) {
+    ClientState& client = _clients.state(fd);
+    if (client.registered || client.passOk) {
+        sendNumeric(fd, 462, std::vector<std::string>(), "You may not reregister");
+        return;
+    }
+    if (message.params.empty()) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "PASS"), "Not enough parameters");
+        return;
+    }
+    if (!_password.empty() && message.params[0] != _password) {
+        sendNumeric(fd, 464, std::vector<std::string>(), "Password incorrect");
+        requestClose(fd, "Password incorrect");
+        return;
+    }
+    client.passOk = true;
+    maybeRegister(fd);
+}


## `feat(registration): 닉네임 검증과 색인 갱신`

diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 0676aae..eb855cb 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -2,8 +2,28 @@
 
 #include "IrcMessage.hpp"
 
+#include <cctype>
 #include <vector>
 
+namespace {
+    bool isValidNickname(const std::string& nickname) {
+        if (nickname.empty() || nickname.size() > 30) {
+            return false;
+        }
+        const unsigned char first = static_cast<unsigned char>(nickname[0]);
+        if (std::isdigit(first) || nickname[0] == '#' || nickname[0] == '&' || nickname[0] == ':' || nickname[0] == '-') {
+            return false;
+        }
+        for (std::size_t i = 0; i < nickname.size(); ++i) {
+            const unsigned char ch = static_cast<unsigned char>(nickname[i]);
+            if (std::isspace(ch) || ch == ',' || ch == '*' || ch == '?' || ch == '!' || ch == '@') {
+                return false;
+            }
+        }
+        return true;
+    }
+}
+
 void IrcApplication::handlePass(int fd, const IrcMessage& message) {
     ClientState& client = _clients.state(fd);
     if (client.registered || client.passOk) {
@@ -22,3 +42,25 @@ void IrcApplication::handlePass(int fd, const IrcMessage& message) {
     client.passOk = true;
     maybeRegister(fd);
 }
+
+void IrcApplication::handleNick(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 431, std::vector<std::string>(), "No nickname given");
+        return;
+    }
+
+    const std::string nextNick = message.params[0];
+    if (!isValidNickname(nextNick)) {
+        sendNumeric(fd, 432, std::vector<std::string>(1, nextNick), "Erroneous nickname");
+        return;
+    }
+
+    const int collision = _clients.findFdByNickname(nextNick);
+    if (collision != -1 && collision != fd) {
+        sendNumeric(fd, 433, std::vector<std::string>(1, nextNick), "Nickname is already in use");
+        return;
+    }
+
+    _clients.setNickname(fd, nextNick);
+    maybeRegister(fd);
+}


## `feat(registration): USER 정보와 환영 응답 연결`

diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index eb855cb..3cb3934 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -64,3 +64,30 @@ void IrcApplication::handleNick(int fd, const IrcMessage& message) {
     _clients.setNickname(fd, nextNick);
     maybeRegister(fd);
 }
+
+void IrcApplication::handleUser(int fd, const IrcMessage& message) {
+    ClientState& client = _clients.state(fd);
+    if (client.registered) {
+        sendNumeric(fd, 462, std::vector<std::string>(), "You may not reregister");
+        return;
+    }
+    if (message.params.size() < 4) {
+        sendNumeric(fd, 461, std::vector<std::string>(1, "USER"), "Not enough parameters");
+        return;
+    }
+    client.user = message.params[0];
+    client.realname = message.params[3];
+    client.hasUser = true;
+    maybeRegister(fd);
+}
+
+void IrcApplication::maybeRegister(int fd) {
+    ClientState& client = _clients.state(fd);
+    if (client.registered || !client.passOk || !client.hasNick || !client.hasUser) {
+        return;
+    }
+    client.registered = true;
+    sendNumeric(fd, 1, std::vector<std::string>(), "Welcome to the educational IRC reference, " + client.nick);
+    sendNumeric(fd, 2, std::vector<std::string>(), "Your host is " + _serverName + ", running a C++17 reference server");
+    sendNumeric(fd, 3, std::vector<std::string>(), "This server was created for IRC protocol learning");
+}


## `feat(registration): PING과 QUIT 기본 명령 처리`

diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 3cb3934..2fb141d 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -1,6 +1,7 @@
 #include "IrcApplication.hpp"
 
 #include "IrcMessage.hpp"
+#include "Replies.hpp"
 
 #include <cctype>
 #include <vector>
@@ -81,6 +82,22 @@ void IrcApplication::handleUser(int fd, const IrcMessage& message) {
     maybeRegister(fd);
 }
 
+void IrcApplication::handlePing(int fd, const IrcMessage& message) {
+    if (message.params.empty()) {
+        sendNumeric(fd, 409, std::vector<std::string>(), "No origin specified");
+        return;
+    }
+    std::vector<std::string> params;
+    params.push_back(_serverName);
+    params.push_back(message.params[0]);
+    sendRaw(fd, Replies::formatMessage(_serverName, "PONG", params));
+}
+
+void IrcApplication::handleQuit(int fd, const IrcMessage& message) {
+    const std::string reason = message.params.empty() ? "Client Quit" : message.params[0];
+    requestClose(fd, reason);
+}
+
 void IrcApplication::maybeRegister(int fd) {
     ClientState& client = _clients.state(fd);
     if (client.registered || !client.passOk || !client.hasNick || !client.hasUser) {


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
 
