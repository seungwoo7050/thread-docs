## `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인`

diff --git a/src/ApplicationSupport.cpp b/src/ApplicationSupport.cpp
index 50372ea..461e853 100644
--- a/src/ApplicationSupport.cpp
+++ b/src/ApplicationSupport.cpp
@@ -114,15 +114,16 @@ Channel* IrcApplication::findChannelForCommand(int fd, const std::string& name,
     return &it->second;
 }
 
-void IrcApplication::sendTopicReply(int fd, const Channel& channel) {
+bool IrcApplication::sendTopicReply(int fd, const Channel& channel) {
+    const std::string channelName = channel.name();
     if (channel.hasTopic()) {
-        sendNumeric(fd, 332, std::vector<std::string>(1, channel.name()), channel.topic());
-    } else {
-        sendNumeric(fd, 331, std::vector<std::string>(1, channel.name()), "No topic is set");
+        return sendNumeric(fd, 332, std::vector<std::string>(1, channelName), channel.topic());
     }
+    return sendNumeric(fd, 331, std::vector<std::string>(1, channelName), "No topic is set");
 }
 
-void IrcApplication::sendNames(int fd, const Channel& channel) {
+bool IrcApplication::sendNames(int fd, const Channel& channel) {
+    const std::string channelName = channel.name();
     std::vector<std::string> names;
     const std::vector<int> members = channel.members();
     for (std::size_t i = 0; i < members.size(); ++i) {
@@ -135,9 +136,11 @@ void IrcApplication::sendNames(int fd, const Channel& channel) {
 
     std::vector<std::string> nameParams;
     nameParams.push_back("=");
-    nameParams.push_back(channel.name());
-    sendNumeric(fd, 353, nameParams, joinWords(names, " "));
-    sendNumeric(fd, 366, std::vector<std::string>(1, channel.name()), "End of /NAMES list");
+    nameParams.push_back(channelName);
+    if (!sendNumeric(fd, 353, nameParams, joinWords(names, " "))) {
+        return false;
+    }
+    return sendNumeric(fd, 366, std::vector<std::string>(1, channelName), "End of /NAMES list");
 }
 
 void IrcApplication::partAllChannels(int fd, const std::string& reason) {
@@ -169,18 +172,24 @@ void IrcApplication::partChannel(int fd, const std::string& channelName, const s
         params.push_back(reason);
     }
     broadcastToChannel(channelName, Replies::formatMessage(prefixFor(fd), "PART", params), -1);
+    it = _channels.find(channelName);
+    if (it == _channels.end()) {
+        return;
+    }
     it->second.removeMember(fd);
     eraseChannelIfEmpty(channelName);
 }
 
-void IrcApplication::broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg) {
+bool IrcApplication::broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg) {
+    const std::string channelName = channel.name();
     std::vector<std::string> params;
-    params.push_back(channel.name());
+    params.push_back(channelName);
     params.push_back(mode);
     if (!arg.empty()) {
         params.push_back(arg);
     }
-    broadcastToChannel(channel.name(), Replies::formatMessage(prefixFor(fd), "MODE", params), -1);
+    broadcastToChannel(channelName, Replies::formatMessage(prefixFor(fd), "MODE", params), -1);
+    return _clients.contains(fd) && _channels.find(channelName) != _channels.end();
 }
 
 void IrcApplication::broadcastToChannel(const std::string& channelName, const std::string& line, int exceptFd) {
@@ -248,19 +257,19 @@ std::string IrcApplication::prefixFor(const ClientState& client) const {
     return Replies::hostmask(client.nick, client.user, client.host);
 }
 
-void IrcApplication::sendNumeric(int fd, int numericCode, const std::vector<std::string>& params, const std::string& trailing) {
-    sendRaw(fd, Replies::numeric(_serverName, replyTarget(fd), numericCode, params, trailing));
+bool IrcApplication::sendNumeric(int fd, int numericCode, const std::vector<std::string>& params, const std::string& trailing) {
+    return sendRaw(fd, Replies::numeric(_serverName, replyTarget(fd), numericCode, params, trailing));
 }
 
-void IrcApplication::sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params) {
+bool IrcApplication::sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params) {
     std::vector<std::string> allParams;
     allParams.push_back(replyTarget(fd));
     allParams.insert(allParams.end(), params.begin(), params.end());
-    sendRaw(fd, Replies::formatMessage(_serverName, Replies::code(numericCode), allParams));
+    return sendRaw(fd, Replies::formatMessage(_serverName, Replies::code(numericCode), allParams));
 }
 
-void IrcApplication::sendRaw(int fd, const std::string& line) {
-    _server.sendTo(fd, line);
+bool IrcApplication::sendRaw(int fd, const std::string& line) {
+    return _server.sendTo(fd, line);
 }
 
 void IrcApplication::requestClose(int fd, const std::string& reason) {
diff --git a/src/ChannelCommands.cpp b/src/ChannelCommands.cpp
index 1c390cf..8199fe8 100644
--- a/src/ChannelCommands.cpp
+++ b/src/ChannelCommands.cpp
@@ -11,7 +11,6 @@ void IrcApplication::handleJoin(int fd, const IrcMessage& message) {
         sendNumeric(fd, 461, std::vector<std::string>(1, "JOIN"), "Not enough parameters");
         return;
     }
-
     if (message.params[0] == "0") {
         partAllChannels(fd, "Leaving all channels");
         return;
@@ -19,29 +18,48 @@ void IrcApplication::handleJoin(int fd, const IrcMessage& message) {
 
     const std::vector<std::string> names = splitComma(message.params[0]);
     for (std::size_t i = 0; i < names.size(); ++i) {
-        const std::string& name = names[i];
+        const std::string name = names[i];
         if (!Channel::isValidName(name)) {
-            sendNumeric(fd, 403, std::vector<std::string>(1, name), "No such channel");
+            if (!sendNumeric(fd, 403, std::vector<std::string>(1, name), "No such channel")) {
+                return;
+            }
             continue;
         }
 
         Channel& channel = ensureChannel(name);
         if (channel.hasMember(fd)) {
-            sendNames(fd, channel);
+            if (!sendNames(fd, channel)) {
+                return;
+            }
             continue;
         }
-        if (channel.isInviteOnly() && !channel.isInvited(_clients.state(fd).nick)) {
-            sendNumeric(fd, 473, std::vector<std::string>(1, name), "Cannot join channel (+i)");
+        const ClientState* client = _clients.find(fd);
+        if (client == NULL) {
+            return;
+        }
+        if (channel.isInviteOnly() && !channel.isInvited(client->nick)) {
+            if (!sendNumeric(fd, 473, std::vector<std::string>(1, name), "Cannot join channel (+i)")) {
+                return;
+            }
             continue;
         }
 
         const bool firstMember = channel.empty();
+        const std::string nick = client->nick;
         channel.addMember(fd, firstMember);
-        channel.clearInvite(_clients.state(fd).nick);
-
+        channel.clearInvite(nick);
         broadcastToChannel(name, Replies::formatMessage(prefixFor(fd), "JOIN", std::vector<std::string>(1, name)), -1);
-        sendTopicReply(fd, channel);
-        sendNames(fd, channel);
+        if (!_clients.contains(fd)) {
+            return;
+        }
+        std::map<std::string, Channel>::iterator current = _channels.find(name);
+        if (current == _channels.end() || !sendTopicReply(fd, current->second)) {
+            return;
+        }
+        current = _channels.find(name);
+        if (current == _channels.end() || !sendNames(fd, current->second)) {
+            return;
+        }
     }
 }
 
@@ -91,7 +109,6 @@ void IrcApplication::handleKick(int fd, const IrcMessage& message) {
         sendNumeric(fd, 461, std::vector<std::string>(1, "KICK"), "Not enough parameters");
         return;
     }
-
     Channel* channel = findChannelForCommand(fd, message.params[0], true);
     if (!channel) {
         return;
@@ -100,7 +117,7 @@ void IrcApplication::handleKick(int fd, const IrcMessage& message) {
         sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
         return;
     }
-
+    const std::string channelName = channel->name();
     const std::string targetNick = message.params[1];
     const int targetFd = findNick(targetNick);
     if (targetFd == -1) {
@@ -110,19 +127,22 @@ void IrcApplication::handleKick(int fd, const IrcMessage& message) {
     if (!channel->hasMember(targetFd)) {
         std::vector<std::string> params;
         params.push_back(targetNick);
-        params.push_back(channel->name());
+        params.push_back(channelName);
         sendNumeric(fd, 441, params, "They aren't on that channel");
         return;
     }
-
     const std::string comment = message.params.size() > 2 ? message.params[2] : targetNick;
     std::vector<std::string> params;
-    params.push_back(channel->name());
+    params.push_back(channelName);
     params.push_back(targetNick);
     params.push_back(comment);
-    broadcastToChannel(channel->name(), Replies::formatMessage(prefixFor(fd), "KICK", params), -1);
-    channel->removeMember(targetFd);
-    eraseChannelIfEmpty(channel->name());
+    broadcastToChannel(channelName, Replies::formatMessage(prefixFor(fd), "KICK", params), -1);
+    std::map<std::string, Channel>::iterator current = _channels.find(channelName);
+    if (current == _channels.end()) {
+        return;
+    }
+    current->second.removeMember(targetFd);
+    eraseChannelIfEmpty(channelName);
 }
 
 void IrcApplication::handleInvite(int fd, const IrcMessage& message) {
@@ -130,7 +150,6 @@ void IrcApplication::handleInvite(int fd, const IrcMessage& message) {
         sendNumeric(fd, 461, std::vector<std::string>(1, "INVITE"), "Not enough parameters");
         return;
     }
-
     const std::string targetNick = message.params[0];
     const std::string channelName = message.params[1];
     const int targetFd = findNick(targetNick);
@@ -138,35 +157,34 @@ void IrcApplication::handleInvite(int fd, const IrcMessage& message) {
         sendNumeric(fd, 401, std::vector<std::string>(1, targetNick), "No such nick/channel");
         return;
     }
-
     Channel* channel = findChannelForCommand(fd, channelName, true);
     if (!channel) {
         return;
     }
     if (channel->isInviteOnly() && !channel->isOperator(fd)) {
-        sendNumeric(fd, 482, std::vector<std::string>(1, channel->name()), "You're not channel operator");
+        sendNumeric(fd, 482, std::vector<std::string>(1, channelName), "You're not channel operator");
         return;
     }
     if (channel->hasMember(targetFd)) {
         std::vector<std::string> params;
         params.push_back(targetNick);
-        params.push_back(channel->name());
+        params.push_back(channelName);
         sendNumeric(fd, 443, params, "is already on channel");
         return;
     }
-
     channel->invite(targetNick);
-
+    const std::string sourcePrefix = prefixFor(fd);
     std::vector<std::string> invitingParams;
     invitingParams.push_back(replyTarget(fd));
     invitingParams.push_back(targetNick);
-    invitingParams.push_back(channel->name());
-    sendRaw(fd, Replies::formatMessage(_serverName, "341", invitingParams));
-
+    invitingParams.push_back(channelName);
+    const std::string acknowledgement = Replies::formatMessage(_serverName, "341", invitingParams);
     std::vector<std::string> inviteParams;
     inviteParams.push_back(targetNick);
-    inviteParams.push_back(channel->name());
-    sendRaw(targetFd, Replies::formatMessage(prefixFor(fd), "INVITE", inviteParams));
+    inviteParams.push_back(channelName);
+    const std::string invitation = Replies::formatMessage(sourcePrefix, "INVITE", inviteParams);
+    sendRaw(fd, acknowledgement);
+    sendRaw(targetFd, invitation);
 }
 
 void IrcApplication::handleMode(int fd, const IrcMessage& message) {
diff --git a/src/IrcApplication.cpp b/src/IrcApplication.cpp
index f9a54a7..8f56a2c 100644
--- a/src/IrcApplication.cpp
+++ b/src/IrcApplication.cpp
@@ -133,13 +133,12 @@ void IrcApplication::handleMessage(int fd, const IrcMessage& message) {
 }
 
 void IrcApplication::maintainClient(int fd, const MonotonicTime& now) {
-    ClientState* found = _clients.find(fd);
-    if (found == NULL) {
+    ClientState* client = _clients.find(fd);
+    if (client == NULL) {
         return;
     }
-    ClientState& client = *found;
-    if (!client.registered &&
-        now - client.connectedAt >= std::chrono::seconds(_runtime.registrationTimeoutSeconds)) {
+    if (!client->registered &&
+        now - client->connectedAt >= std::chrono::seconds(_runtime.registrationTimeoutSeconds)) {
         sendNumeric(fd, 451, std::vector<std::string>(), "Registration timeout");
         requestClose(fd, "registration timeout");
         return;
@@ -147,8 +146,8 @@ void IrcApplication::maintainClient(int fd, const MonotonicTime& now) {
     if (_runtime.idleTimeoutSeconds <= 0) {
         return;
     }
-    if (client.awaitingPong &&
-        now - client.lastPingAt >= std::chrono::seconds(_runtime.pingTimeoutSeconds)) {
+    if (client->awaitingPong &&
+        now - client->lastPingAt >= std::chrono::seconds(_runtime.pingTimeoutSeconds)) {
         ++_metrics.idleTimeouts;
         sendRaw(fd, Replies::error("Ping timeout"));
         requestClose(fd, "ping timeout");
@@ -158,14 +157,16 @@ void IrcApplication::maintainClient(int fd, const MonotonicTime& now) {
         });
         return;
     }
-    if (!client.awaitingPong &&
-        now - client.lastActivityAt >= std::chrono::seconds(_runtime.idleTimeoutSeconds)) {
+    if (!client->awaitingPong &&
+        now - client->lastActivityAt >= std::chrono::seconds(_runtime.idleTimeoutSeconds)) {
         const std::string token =
             "heartbeat-" + std::to_string(fd) + "-" + std::to_string(++_nextHeartbeatToken);
-        sendRaw(fd, Replies::formatMessage(_serverName, "PING", std::vector<std::string>(1, token)));
-        client.awaitingPong = true;
-        client.pendingPongToken = token;
-        client.lastPingAt = now;
+        client->awaitingPong = true;
+        client->pendingPongToken = token;
+        client->lastPingAt = now;
+        if (!sendRaw(fd, Replies::formatMessage(_serverName, "PING", std::vector<std::string>(1, token)))) {
+            return;
+        }
         ++_metrics.heartbeatPings;
     }
 }
diff --git a/src/IrcApplication.hpp b/src/IrcApplication.hpp
index ccd87bc..9eef128 100644
--- a/src/IrcApplication.hpp
+++ b/src/IrcApplication.hpp
@@ -82,11 +82,11 @@ private:
     void handleMetrics(int fd);
     Channel& ensureChannel(const std::string& name);
     Channel* findChannelForCommand(int fd, const std::string& name, bool requireMembership);
-    void sendTopicReply(int fd, const Channel& channel);
-    void sendNames(int fd, const Channel& channel);
+    bool sendTopicReply(int fd, const Channel& channel);
+    bool sendNames(int fd, const Channel& channel);
     void partAllChannels(int fd, const std::string& reason);
     void partChannel(int fd, const std::string& channelName, const std::string& reason);
-    void broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg);
+    bool broadcastMode(int fd, const Channel& channel, const std::string& mode, const std::string& arg);
     void broadcastToChannel(const std::string& channelName, const std::string& line, int exceptFd);
     void broadcastToCommon(int fd, const std::string& line, bool includeSelf);
     void eraseChannelIfEmpty(const std::string& channelName);
@@ -94,14 +94,14 @@ private:
     std::string replyTarget(int fd) const;
     std::string prefixFor(int fd) const;
     std::string prefixFor(const ClientState& client) const;
-    void sendNumeric(
+    bool sendNumeric(
         int fd,
         int numericCode,
         const std::vector<std::string>& params,
         const std::string& trailing
     );
-    void sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params);
-    void sendRaw(int fd, const std::string& line);
+    bool sendNumericRaw(int fd, int numericCode, const std::vector<std::string>& params);
+    bool sendRaw(int fd, const std::string& line);
     void requestClose(int fd, const std::string& reason);
     void removeClientState(int fd, const std::string& reason, bool notifyPeers);
 };
diff --git a/src/RegistrationCommands.cpp b/src/RegistrationCommands.cpp
index 3104228..c5c19e5 100644
--- a/src/RegistrationCommands.cpp
+++ b/src/RegistrationCommands.cpp
@@ -71,6 +71,9 @@ void IrcApplication::handleNick(int fd, const IrcMessage& message) {
 
     if (wasRegistered) {
         broadcastToCommon(fd, Replies::formatMessage(oldPrefix, "NICK", std::vector<std::string>(1, nextNick)), true);
+        if (!_clients.contains(fd)) {
+            return;
+        }
     }
 
     maybeRegister(fd);
@@ -121,16 +124,23 @@ void IrcApplication::handleQuit(int fd, const IrcMessage& message) {
 }
 
 void IrcApplication::maybeRegister(int fd) {
-    ClientState& client = _clients.state(fd);
-    if (client.registered || !client.passOk || !client.hasNick || !client.hasUser) {
+    ClientState* client = _clients.find(fd);
+    if (client == NULL || client->registered || !client->passOk || !client->hasNick || !client->hasUser) {
+        return;
+    }
+    client->registered = true;
+    const std::string nick = client->nick;
+    if (!sendNumeric(fd, 1, std::vector<std::string>(), "Welcome to irc-relay-server, " + nick)) {
+        return;
+    }
+    if (!sendNumeric(fd, 2, std::vector<std::string>(), "Your host is " + _serverName)) {
+        return;
+    }
+    if (!sendNumeric(fd, 3, std::vector<std::string>(), "This server is running a C++17 event backend")) {
         return;
     }
-    client.registered = true;
-    sendNumeric(fd, 1, std::vector<std::string>(), "Welcome to irc-relay-server, " + client.nick);
-    sendNumeric(fd, 2, std::vector<std::string>(), "Your host is " + _serverName);
-    sendNumeric(fd, 3, std::vector<std::string>(), "This server is running a C++17 event backend");
     logEvent("client_registered", std::vector<std::pair<std::string, std::string> >{
         std::make_pair("fd", std::to_string(fd)),
-        std::make_pair("nick", client.nick)
+        std::make_pair("nick", nick)
     });
 }


